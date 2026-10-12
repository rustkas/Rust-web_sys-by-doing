# 19.7. `FileSystemDirectoryHandle`: обход, `resolve`, идемпотентность

## Зачем это нужно

Пока мы умеем открывать **один** файл — `showOpenFilePicker` возвращает `FileSystemFileHandle`, и мы читаем его через `getFile()`. Но реальные сценарии почти всегда про **папку**: пользователь выбрасывает на страницу директорию проекта, и мы должны её обойти, посчитать размеры, показать дерево, найти нужные файлы, дать возможность что-то удалить или перезаписать.

Для этого в File System Access API есть второй handle — `FileSystemDirectoryHandle`. Это не «файл-контейнер», а полноценный узел дерева: у него есть методы для навигации вниз (`getFileHandle`, `getDirectoryHandle`), для удаления (`removeEntry`) и для позиционирования относительно друг друга (`resolve`). Плюс — `values()`, асинхронный итератор по непосредственным детям.

В проекте уже есть базовый уровень — `src/fsa_api_directory.rs`, который закрывает:

- получить handle файла/директории,
- удалить запись идемпотентно,
- вернуть относительный путь через `resolve`,
- обойти директорию **плоским** списком через `scan`.

Этого уровня достаточно для «просто перебрать файлы». Но для реальных сценариев — рендер дерева в UI, поиск по пути, сравнение двух состояний — нужен следующий слой. Именно ему посвящена эта глава. Файлы ниже **не дублируют** существующие модули, а **опираются на них**: импортируют `iter_next`, `get_file_handle`, `get_directory_handle`, `remove_entry`, `resolve_path`, `scan`.

Мы добавляем два новых модуля и три новых тестовых файла:

| Файл | Задача |
|---|---|
| `src/fsa_dir_tree.rs` | Иерархическое дерево + навигация по путям |
| `src/fsa_dir_diff.rs` | Снимки и diff двух состояний |
| `tests/wasm_fsa_dir_tree.rs` | WASM-тесты для дерева и путей |
| `tests/pure_dir_diff.rs` | Host-тесты для чистой логики diff |
| `tests/wasm_fsa_dir_diff.rs` | WASM-тесты для `snapshot` |

Плюс — интерактивное демо в `src/app.rs`, которое показывает весь цикл «создать структуру → построить дерево → снять снимок → изменить → diff → визуализировать».

---

## 19.7.1. Анатомия `FileSystemDirectoryHandle`

Через `web-sys` тип приходит как `web_sys::FileSystemDirectoryHandle`. Прямых методов у него в биндингах почти нет — это тонкая обёртка над JS-объектом. Поэтому все операции мы делаем через `js_sys::Reflect::get` + `js_sys::Function::callN`: достаём метод по имени, проверяем, что он функция, вызываем с нужным числом аргументов.

```rust
fn call_promise(
    handle: &JsValue,
    method: &str,
    args: &[JsValue],
) -> Result<js_sys::Promise, JsValue> {
    let fn_js = js_sys::Reflect::get(handle, &JsValue::from_str(method))?;
    if fn_js.is_undefined() {
        return Err(JsValue::from_str(&format!("{method} недоступен")));
    }
    let func: js_sys::Function = fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{method} не функция")))?;
    let promise_js = match args.len() {
        0 => func.call0(handle)?,
        1 => func.call1(handle, &args[0])?,
        2 => func.call2(handle, &args[0], &args[1])?,
        _ => return Err(JsValue::from_str("слишком много аргументов")),
    };
    promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{method} вернул не Promise")))
}
```

Этот хелпер — фундамент всего модуля. Он даёт две вещи, которых нет в голом `Reflect::get`:

1. **Раннее обнаружение несовместимости окружения.** Если браузер не знает метод, мы получим внятную ошибку «`getFileHandle` недоступен», а не `undefined is not a function` где-то в недрах.
2. **Единый async-контракт.** Любой публичный метод модуля возвращает `Promise`, а значит, может быть превращён в `JsFuture` и дожидаться в `async`-контексте без сюрпризов.

> **Ловушка `dyn_into` и `JsValue`.** `JsValue` — это неопознанный JS-объект. `dyn_into::<FileSystemFileHandle>()` работает только если объект действительно является экземпляром этого интерфейса. Если браузер вернёт что-то другое (например, старую реализацию без `kind`), `dyn_into` упадёт — и это хорошо: лучше явная ошибка конвертации, чем молчаливое «работает не так».

---

## 19.7.2. Три кита: `getFileHandle`, `getDirectoryHandle`, `removeEntry`

Все три метода — это `Promise`-возвращающие операции. Оборачиваем их в async-функции.

```rust
pub async fn get_file_handle(
    dir: &FileSystemDirectoryHandle,
    name: &str,
    create: bool,
) -> Result<FileSystemFileHandle, JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options, &JsValue::from_str("create"), &JsValue::from_bool(create),
    )?;
    let dir_js: &JsValue = dir.as_ref();
    let promise = call_promise(dir_js, "getFileHandle", &[
        JsValue::from_str(name),
        options.into(),
    ])?;
    let value = JsFuture::from(promise).await?;
    value.dyn_into::<FileSystemFileHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))
}
```

`get_directory_handle` — зеркальный, только `dyn_into` идёт в `FileSystemDirectoryHandle`.

Обратите внимание на флаг `create`. Это **не** «создать, если нет, и вернуть если есть». Это «разрешить создание»:

| `create` | файла нет | файл есть |
|---|---|---|
| `false` | `NotFoundError` | `Ok(handle)` |
| `true` | `Ok(new handle)` | `Ok(existing handle)` |

То есть `create: true` — идемпотентен по факту создания, но **не** по содержимому: если файл существовал, вернётся handle на существующий файл, а не «чистый лист». Это важно помнить, когда демо показывает «создать пример»: он не должен затирать пользовательские данные.

### Удаление и правило идемпотентности

```rust
pub async fn remove_entry(
    dir: &FileSystemDirectoryHandle,
    name: &str,
    recursive: bool,
) -> Result<(), JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options, &JsValue::from_str("recursive"), &JsValue::from_bool(recursive),
    )?;
    let dir_js: &JsValue = dir.as_ref();
    let promise = call_promise(dir_js, "removeEntry", &[
        JsValue::from_str(name),
        options.into(),
    ])?;

    match JsFuture::from(promise).await {
        Ok(_) => Ok(()),
        Err(e) => {
            let err_name = js_sys::Reflect::get(&e, &JsValue::from_str("name"))
                .ok().and_then(|v| v.as_string()).unwrap_or_default();
            if err_name == "NotFoundError" { Ok(()) } else { Err(e) }
        }
    }
}
```

Здесь начинается **правило проекта**: «удалить дважды» — это успех, а не ошибка. Причина не в красоте, а в реальности:

- Chrome разных версий (и Safari) по-разному реагируют на `removeEntry` для отсутствующей записи. Кто-то кидает `NotFoundError`, кто-то молча резолвит.
- В UI кнопка «Удалить» может быть нажата дважды. Пользователь ждёт, что второе нажатие ничего не сломает.
- В распределённых сценариях (несколько вкладок правят одну OPFS-папку) удаление уже удалённого — это норма, а не ЧП.

Приводим оба варианта к `Ok(())`. Флаг `recursive` работает для директорий: `removeEntry("dir", {recursive: true})` снесёт папку со всем содержимым. Для файла `recursive` игнорируется — это нормально, мы всё равно передаём его, чтобы не плодить две почти одинаковые функции.

> **Правило проекта (повторим):** все деструктивные операции, доступные из UI, должны быть **идемпотентны**. Если операция не идемпотентна от природы (например, «увеличить счётчик»), её нельзя вешать на клик без подтверждения. Удаление — идемпотентно by design.

---

## 19.7.3. `resolve`: путь относительно корня

`resolve(child)` возвращает массив сегментов пути от `root` до `child`, либо `null`, если `child` не является потомком `root`. На практике он почти всегда используется так: «у меня есть handle, полученный откуда-то, и корень; покажи, где относительно корня он лежит».

```rust
pub async fn resolve_path(
    root: &FileSystemDirectoryHandle,
    child: &JsValue,
) -> Result<Option<String>, JsValue> {
    let root_js: &JsValue = root.as_ref();
    let promise = call_promise(root_js, "resolve", &[child.clone()])?;
    let value = JsFuture::from(promise).await?;
    if value.is_null() || value.is_undefined() {
        return Ok(None);
    }
    let array: js_sys::Array = value
        .dyn_into().map_err(|_| JsValue::from_str("resolve: не массив"))?;
    let mut parts: Vec<String> = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        if let Some(s) = array.get(i).as_string() { parts.push(s); }
    }
    Ok(Some(parts.join("/")))
}
```

Что тут стоит проговорить:

- `resolve` возвращает **не строку**, а `Array<string>`. Мы нормализуем к `String` — это дешевле для потребителя и заодно прогоняет проверку «а точно ли это массив».
- `null`/`undefined` означает «не потомок». Это не ошибка, поэтому `Ok(None)`, а не `Err`.
- `join("/")` даёт POSIX-подобный путь. Даже на Windows OPFS работает с прямыми слэшами, потому что это внутреннее представление браузера, а не файловая система ОС.

---

## 19.7.4. Почему обход итеративный, а не рекурсивный

Это, пожалуй, самая неочевидная часть главы. В обычном Rust рекурсивный обход директории — это красиво:

```rust
// Псевдокод. Так делать НЕ надо в WASM.
async fn walk(dir: &FileSystemDirectoryHandle, prefix: &str, out: &mut Vec<Entry>)
    -> Result<(), JsValue>
{
    for child in dir.children().await? {
        match child.kind {
            "file" => out.push(entry(child)),
            "dir"  => walk(&child, &full_path, out).await?, // ❌
        }
    }
    Ok(())
}
```

Проблема в `async`. Rust не умеет вычислять размер `async`-функции, если она вызывает себя. Компилятор предложит:

```rust
Box::pin(walk(&child, &full_path, out)).await?
```

Каждый уровень рекурсии добавляет `Box::pin` — аллокацию в куче. На глубине 20 папок это 20 боксов, в каждом — future, который содержит ссылки на стек предыдущего. Плюс к этому мы не можем отменить обход «между» боксами: отмена сработает только на выходе из самого глубокого вызова. А значит, кнопка «Отмена» в UI будет тормозить.

Поэтому в проекте принят **итеративный обход через `VecDeque`**:

```rust
pub async fn scan(
    root: &FileSystemDirectoryHandle,
    max_files: usize,
    mut on_progress: impl FnMut(usize),
    is_cancelled: impl Fn() -> bool,
) -> Result<Vec<Entry>, JsValue> {
    let mut out: Vec<Entry> = Vec::new();
    let mut queue: VecDeque<(FileSystemDirectoryHandle, String)> = VecDeque::new();
    queue.push_back((root.clone(), String::new()));
    let mut last_yield = 0.0_f64;

    while let Some((dir, prefix)) = queue.pop_front() {
        if is_cancelled() || out.len() >= max_files { break; }
        // ... получаем values(), итерируем ...
        // для файлов: out.push(...), on_progress(...)
        // для директорий: queue.push_back((dh, full_path))
    }
    Ok(out)
}
```

Что это даёт:

1. **Нет `Box::pin`.** Всё состояние — в `queue` и `out` на стеке `scan`. Глубина вложенности не влияет на размер future.
2. **Отмена работает в любой момент.** Мы проверяем `is_cancelled()` перед каждой итерацией внешнего и внутреннего цикла. Отмена — это просто `break`.
3. **Лимит файлов — тоже часть условия выхода.** `out.len() >= max_files` останавливает обход ровно на нужном количестве, а не «после того, как обойдём всё и обрежем».
4. **UI-friendly.** Мы можем отдавать управление браузеру между итерациями.

> **Ментальная модель.** `VecDeque` + `while let` — это «ручной стек вызовов». Мы явно храним то, что рекурсия хранила бы неявно. Да, чуть многословнее. Зато контролируемо.

**Важное правило про пути.** `scan` строит пути **относительно переданного root**, без имени самого root:

```rust
queue.push_back((root.clone(), String::new()));   // prefix = ""
// ...
let full_path: String = if prefix.is_empty() {
    name.clone()                                  // "readme.md"
} else {
    format!("{prefix}/{name}")                    // "src/util/io.rs"
};
```

То есть если мы передали `open_dir(root, "demo_19_7")`, то пути файлов внутри — `readme.md`, `src/main.rs`, а **не** `demo_19_7/readme.md`. Это критично, потому что `Snapshot`/`Diff` используют те же пути, а UI-рендер должен с ними совпадать. Мы вернёмся к этому в разделе про `fsa_dir_tree.rs`.

---

## 19.7.5. `values()` — это **не** `Promise`, и это важно

Внутри `scan` мы получаем итератор по детям директории:

```rust
let values_fn: js_sys::Function = values_fn_js
    .dyn_into().map_err(|_| JsValue::from_str("values не функция"))?;
let iter: JsValue = values_fn.call0(dir_js)?;
```

`values()` возвращает **асинхронный итератор**, а не промис. Это разные вещи:

- `Promise` резолвится один раз, `JsFuture::from(promise).await` даёт значение.
- `AsyncIterator` — это объект с методом `next()`, который **на каждом шаге** возвращает `Promise<{value, done}>`.

Поэтому обёртка:

```rust
pub async fn iter_next(iter: &JsValue) -> Result<Option<JsValue>, JsValue> {
    let next_js = js_sys::Reflect::get(iter, &JsValue::from_str("next"))?;
    let next_fn: js_sys::Function = next_js
        .dyn_into().map_err(|_| JsValue::from_str("next не функция"))?;
    let promise: js_sys::Promise = next_fn
        .call0(iter)?
        .dyn_into().map_err(|_| JsValue::from_str("next не Promise"))?;
    let result: JsValue = JsFuture::from(promise).await?;
    let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
        .as_bool().unwrap_or(true);
    if done { return Ok(None); }
    Ok(Some(js_sys::Reflect::get(&result, &JsValue::from_str("value"))?))
}
```

Частая ошибка новичка — попытаться сделать `JsFuture::from(iter)` напрямую. Компилятор не пропустит (тип не тот), но если бы пропустил — мы бы получили «ожидание промиса, которого нет». `values()` — **фабрика** асинхронного итератора, а не сам итератор-промис.

> **Замечание про `for await` в JS.** В JavaScript это выглядит естественно:
> ```js
> for await (const entry of dir.values()) { ... }
> ```
> В Rust WASM аналога `for await` нет. Мы его эмулируем вручную через `loop { iter_next(...).await? }`. Это ещё одна причина, почему обход в `scan` выглядит «циклом в цикле», а не одним `for`.

---

## 19.7.6. Yield раз в 16 мс — не микрооптимизация

Внутри цикла по файлам стоит вот такой блок:

```rust
on_progress(out.len());

let now = js_sys::Date::now();
if now - last_yield >= 16.0 {
    TimeoutFuture::new(0).await;
    last_yield = now;
}
```

Это **правило проекта**, а не деталь реализации. Смысл — отдать управление браузерному event loop.

WASM в браузере исполняется в том же потоке, что и UI. Если `scan` длится дольше ~50 мс без yield, браузер не успевает перерисовать страницу: прогресс-бар «замирает», кнопка «Отмена» не реагирует на клик (клик — это событие, которое ждёт своей очереди). Пользователь видит «зависшее приложение» и либо убивает вкладку, либо ненавидит ваш продукт.

`TimeoutFuture::new(0).await` — это `setTimeout(…, 0)`, самый дешёвый способ сказать браузеру «сделай свои дела, я подожду». 16 мс — компромисс: чаще — лишние переключения контекста, реже — UI начинает отставать от данных. Число 16 соответствует одному кадру при 60 Гц.

> **Почему именно 16 мс, а не `requestAnimationFrame`?** `rAF` приостанавливается, когда вкладка в фоне, и мы получим «застрявший» обход. `setTimeout` работает всегда. Для прогресса фонового сканирования это важнее синхронизации с кадром.

---

## 19.7.7. Новый модуль `fsa_dir_tree.rs`: иерархия и пути

### Зачем дерево, если есть плоский список

`scan` даёт `Vec<Entry>`, где `Entry.path` — `"src/fsa_api_directory.rs"`. Для рендера в UI это неудобно:

- нельзя посчитать размер подпапки без повторного прохода;
- нельзя свернуть/развернуть узел без группировки;
- нельзя отличить папку от «файла, в имени которого есть слэш».

Поэтому `fsa_dir_tree.rs` строит **иерархическое дерево**:

```rust
pub enum Node {
    File(FileInfo),
    Dir(DirInfo),
}
```

Обход остаётся **итеративным** — по той же причине, что и в `scan`. А вот **сборка** дерева из плоского списка — уже синхронная рекурсия, и здесь она безопасна: глубина ограничена реальной глубиной папок (десятки уровней), и никаких future внутри.

### Структура `fsa_dir_tree.rs`

```rust
//! Иерархическое дерево директории и навигация по путям.
//!
//! Отличие от `fsa_api_directory::scan`:
//! * `scan` даёт **плоский** список `Entry` — удобно для прогресса
//!   и подсчётов.
//! * `build_tree` даёт **вложенную** структуру `DirInfo` — удобно
//!   для рендера, рекурсивных подсчётов и diff.
//!
//! Дополнительно модуль закрывает:
//! * `find_by_path` — handle по относительному пути.
//! * `ensure_path` — идемпотентно создать цепочку директорий.
//! * `open_dir` — открыть директорию по пути без создания.
//! * `remove_path` — идемпотентное удаление по пути.
//! * `path_of` — относительный путь handle относительно корня.
//!
//! Вся работа со строками вынесена в модуль `path_str`. Правило:
//! `format!` не должен стоять в tail-позиции `if/else` — на WASM
//! компилятор может вывести такое выражение как `()` (E0308).

use std::collections::HashMap;
use std::collections::VecDeque;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

use crate::fsa_api_directory::{
    get_directory_handle, get_file_handle, iter_next, remove_entry, resolve_path,
};
use crate::fsa_api_file::get_file;

// ---------------------------------------------------------------------------
// path_str — вся работа со строками путей в одном месте
// ---------------------------------------------------------------------------

mod path_str {
    /// Склеить префикс и имя. Пустой префикс → только имя.
    pub fn join(prefix: &str, name: &str) -> String {
        if prefix.is_empty() {
            let s: String = name.to_string();
            s
        } else {
            let s: String = format!("{prefix}/{name}");
            s
        }
    }

    /// Разбить путь на непустые сегменты.
    pub fn segments(path: &str) -> Vec<String> {
        let mut out: Vec<String> = Vec::new();
        for part in path.split('/') {
            if !part.is_empty() {
                let s: String = part.to_string();
                out.push(s);
            }
        }
        out
    }

    /// Разбить путь на (родитель, имя).
    pub fn split_parent(path: &str) -> (String, String) {
        match path.rfind('/') {
            Some(i) => {
                let parent: String = path[..i].to_string();
                let name: String = path[i + 1..].to_string();
                (parent, name)
            }
            None => {
                let parent: String = String::new();
                let name: String = path.to_string();
                (parent, name)
            }
        }
    }
}

// ---------------------------------------------------------------------------
// Модель дерева
// ---------------------------------------------------------------------------

#[derive(Debug, Clone, PartialEq)]
pub enum Node {
    File(FileInfo),
    Dir(DirInfo),
}

#[derive(Debug, Clone, PartialEq)]
pub struct FileInfo {
    pub name: String,
    pub size: f64,
    pub mime: String,
}

#[derive(Debug, Clone, PartialEq)]
pub struct DirInfo {
    pub name: String,
    pub children: Vec<Node>,
}

impl Node {
    pub fn name(&self) -> &str {
        match self {
            Node::File(f) => f.name.as_str(),
            Node::Dir(d) => d.name.as_str(),
        }
    }

    pub fn is_dir(&self) -> bool {
        let is: bool = matches!(self, Node::Dir(_));
        is
    }
}

impl DirInfo {
    /// Суммарное число файлов во всём поддереве.
    pub fn total_files(&self) -> usize {
        let mut n: usize = 0;
        for c in &self.children {
            match c {
                Node::File(_) => { n += 1; }
                Node::Dir(d) => { n += d.total_files(); }
            }
        }
        n
    }

    /// Суммарный размер (байты) всех файлов во всём поддереве.
    pub fn total_bytes(&self) -> f64 {
        let mut n: f64 = 0.0;
        for c in &self.children {
            match c {
                Node::File(f) => { n += f.size; }
                Node::Dir(d) => { n += d.total_bytes(); }
            }
        }
        n
    }

    /// Рекурсивная визуализация дерева — удобно для логов и тестов.
    pub fn render(&self) -> String {
        let mut out: String = String::new();
        out.push_str(&self.name);
        out.push_str("/\n");
        render_children(&self.children, 1, &mut out);
        out
    }
}

fn render_children(children: &[Node], depth: usize, out: &mut String) {
    let indent: String = "  ".repeat(depth);
    for c in children {
        match c {
            Node::File(f) => {
                out.push_str(&indent);
                out.push_str(&f.name);
                let suffix: String = format!(" ({} B)\n", f.size as u64);
                out.push_str(&suffix);
            }
            Node::Dir(d) => {
                out.push_str(&indent);
                out.push_str(&d.name);
                out.push_str("/\n");
                render_children(&d.children, depth + 1, out);
            }
        }
    }
}
```

Опции обхода:

```rust
#[derive(Debug, Clone)]
pub struct TreeOptions {
    pub max_depth: usize,
    pub max_files: usize,
    pub skip_dirs: Vec<String>,
}

impl Default for TreeOptions {
    fn default() -> Self {
        let skip: Vec<String> = vec![
            ".git".to_string(),
            "node_modules".to_string(),
            "target".to_string(),
        ];
        Self {
            max_depth: usize::MAX,
            max_files: 10_000,
            skip_dirs: skip,
        }
    }
}
```

### Построение дерева

```rust
pub async fn build_tree(
    root: &FileSystemDirectoryHandle,
    opts: &TreeOptions,
    mut on_progress: impl FnMut(usize),
    is_cancelled: impl Fn() -> bool,
) -> Result<DirInfo, JsValue> {
    let mut flat: Vec<FlatEntry> = Vec::new();
    let mut queue: VecDeque<(FileSystemDirectoryHandle, String, usize)> = VecDeque::new();
    queue.push_back((root.clone(), String::new(), 0));
    let mut file_count: usize = 0;

    while let Some((dir, prefix, depth)) = queue.pop_front() {
        if is_cancelled() || file_count >= opts.max_files {
            break;
        }
        if depth >= opts.max_depth {
            continue;
        }

        let dir_js: &JsValue = dir.as_ref();
        let values_fn_js = js_sys::Reflect::get(dir_js, &JsValue::from_str("values"))?;
        if values_fn_js.is_undefined() {
            continue;
        }
        let values_fn: js_sys::Function = match values_fn_js.dyn_into() {
            Ok(f) => f,
            Err(_) => continue,
        };
        let iter: JsValue = values_fn.call0(dir_js)?;

        loop {
            if is_cancelled() || file_count >= opts.max_files {
                break;
            }
            let entry: JsValue = match iter_next(&iter).await? {
                Some(v) => v,
                None => break,
            };

            let kind: String = js_sys::Reflect::get(&entry, &JsValue::from_str("kind"))?
                .as_string().unwrap_or_default();
            let name: String = js_sys::Reflect::get(&entry, &JsValue::from_str("name"))?
                .as_string().unwrap_or_default();

            if kind == "file" {
                let fh: FileSystemFileHandle = match entry.dyn_into() {
                    Ok(h) => h,
                    Err(_) => continue,
                };
                let file = get_file(&fh).await?;
                let e = FlatEntry {
                    parent_path: prefix.clone(),
                    name,
                    kind: Kind::File,
                    size: file.size(),
                    mime: file.type_(),
                };
                flat.push(e);
                file_count += 1;
                on_progress(file_count);
            } else if kind == "directory" {
                let mut skip: bool = false;
                for s in &opts.skip_dirs {
                    if s == &name { skip = true; break; }
                }
                if skip { continue; }
                let dh: FileSystemDirectoryHandle = match entry.dyn_into() {
                    Ok(h) => h,
                    Err(_) => continue,
                };
                let child_path: String = path_str::join(&prefix, &name);
                let e = FlatEntry {
                    parent_path: prefix.clone(),
                    name,
                    kind: Kind::Dir,
                    size: 0.0,
                    mime: String::new(),
                };
                flat.push(e);
                queue.push_back((dh, child_path, depth + 1));
            }
        }
    }

    let tree_name: String = root.name();
    Ok(assemble(tree_name, flat))
}
```

**Что здесь важно проговорить:**

**Итеративный обход + синхронная сборка.** Внешний цикл `while let Some(...)` — тот же паттерн, что в `scan`. А вот `assemble` — уже обычная рекурсивная функция без `async`. Синхронная рекурсия в Rust не требует `Box::pin`, и её глубина ограничена глубиной папок. Это осознанное разделение: **дорогое I/O — итеративно, дешёвая сборка — рекурсивно**.

`assemble` собирает `HashMap<parent_path, Vec<FlatEntry>>` и рекурсивно строит поддеревья:

```rust
fn assemble(root_name: String, flat: Vec<FlatEntry>) -> DirInfo {
    let mut by_parent: HashMap<String, Vec<FlatEntry>> = HashMap::new();
    for e in flat {
        by_parent.entry(e.parent_path.clone()).or_default().push(e);
    }

    fn build_dir(
        name: String,
        path: String,
        by_parent: &mut HashMap<String, Vec<FlatEntry>>,
    ) -> DirInfo {
        let mut children: Vec<Node> = Vec::new();
        if let Some(entries) = by_parent.remove(&path) {
            for e in entries {
                match e.kind {
                    Kind::File => {
                        let n = Node::File(FileInfo {
                            name: e.name,
                            size: e.size,
                            mime: e.mime,
                        });
                        children.push(n);
                    }
                    Kind::Dir => {
                        let child_path: String = path_str::join(&path, &e.name);
                        let d = build_dir(e.name, child_path, by_parent);
                        children.push(Node::Dir(d));
                    }
                }
            }
        }
        // Детерминированный порядок: сначала папки, потом файлы,
        // внутри группы — по алфавиту.
        children.sort_by(|a, b| a.name().cmp(b.name()));
        children.sort_by_key(|c| match c {
            Node::Dir(_) => 0u8,
            Node::File(_) => 1u8,
        });
        DirInfo { name, children }
    }

    build_dir(root_name, String::new(), &mut by_parent)
}
```

### Навигация по путям

Три задачи, которые не закрывает `scan`:

**`open_dir` / `ensure_path` — общий хелпер `walk_dirs`.**

```rust
async fn walk_dirs(
    root: &FileSystemDirectoryHandle,
    path: &str,
    create: bool,
) -> Result<FileSystemDirectoryHandle, JsValue> {
    let segments: Vec<String> = path_str::segments(path);
    let mut current: FileSystemDirectoryHandle = root.clone();
    for seg in &segments {
        current = get_directory_handle(&current, seg, create).await?;
    }
    Ok(current)
}

pub async fn open_dir(
    root: &FileSystemDirectoryHandle,
    path: &str,
) -> Result<FileSystemDirectoryHandle, JsValue> {
    walk_dirs(root, path, false).await
}

pub async fn ensure_path(
    root: &FileSystemDirectoryHandle,
    path: &str,
) -> Result<FileSystemDirectoryHandle, JsValue> {
    walk_dirs(root, path, true).await
}
```

`ensure_path` идемпотентен **бесплатно**: `create: true` сам по себе возвращает `Ok` и для существующих сегментов. Трижды подряд `ensure_path(root, "a/b/c")` — три раза `Ok`.

**`find_by_path` — найти что угодно по относительному пути.**

```rust
pub async fn find_by_path(
    root: &FileSystemDirectoryHandle,
    path: &str,
) -> Result<Found, JsValue> {
    let (parent, name): (String, String) = path_str::split_parent(path);
    if name.is_empty() {
        return Ok(Found::Dir(root.clone()));
    }
    let dir: FileSystemDirectoryHandle = if parent.is_empty() {
        root.clone()
    } else {
        walk_dirs(root, &parent, false).await?
    };
    if let Ok(fh) = get_file_handle(&dir, &name, false).await {
        return Ok(Found::File(fh));
    }
    let dh: FileSystemDirectoryHandle = get_directory_handle(&dir, &name, false).await?;
    Ok(Found::Dir(dh))
}
```

Для последнего сегмента мы пробуем и файл, и директорию, потому что FS Access API не даёт `getEntry`, возвращающего `FileSystemHandle` без уточнения. Это стоит `NotFoundError` лишний раз, но зато вызывающий код не должен сам гадать, что там лежит.

**`remove_path` — идемпотентное удаление по пути.**

```rust
pub async fn remove_path(
    root: &FileSystemDirectoryHandle,
    path: &str,
    recursive: bool,
) -> Result<(), JsValue> {
    let (parent, name): (String, String) = path_str::split_parent(path);
    if name.is_empty() {
        return Ok(());
    }
    let dir: FileSystemDirectoryHandle = if parent.is_empty() {
        root.clone()
    } else {
        match walk_dirs(root, &parent, false).await {
            Ok(d) => d,
            Err(_) => return Ok(()), // родителя нет → удалять нечего
        }
    };
    remove_entry(&dir, &name, recursive).await
}
```

`remove_path` идемпотентен **дважды**. Если родителя нет — `Ok(())` сразу. Если родитель есть, а записи нет — `remove_entry` вернёт `Ok(())` из-за своей нормализации. Итого: `remove_path` можно безопасно вызывать сколько угодно раз с любым путём.

**`path_of` — тонкая обёртка над `resolve`.**

```rust
pub async fn path_of(
    root: &FileSystemDirectoryHandle,
    child: &JsValue,
) -> Result<Option<String>, JsValue> {
    resolve_path(root, child).await
}
```

Изолирует зависимость на `js_sys::Reflect` внутри модуля. Вызывающий код получает `Option<String>` и не знает, что под капотом `resolve` возвращает массив сегментов.

---

## 19.7.8. Новый модуль `fsa_dir_diff.rs`: снимки и diff

`scan` возвращает список того, что **есть сейчас**. Но реальный пользовательский сценарий звучит иначе: «я снял снимок до, что-то поделал, снял снимок после — покажи, что изменилось». Это уже не обход, это **сравнение**.

```rust
//! Diff двух состояний директории: снимок → изменение → снимок → diff.
//!
//! Почему не через `scan` дважды и не в лоб: `scan` возвращает
//! `Vec<Entry>` в порядке обхода, который не гарантирован между
//! вызовами. Прямое сравнение двух `Vec` даст ложные срабатывания.
//! `BTreeMap` решает это раз и навсегда.

use std::collections::BTreeMap;
use std::collections::BTreeSet;

use wasm_bindgen::prelude::*;
use web_sys::FileSystemDirectoryHandle;

use crate::fsa_api_directory::{scan, Entry};

/// Метаданные одного файла в снимке.
#[derive(Debug, Clone, PartialEq)]
pub struct FileMeta {
    pub size: f64,
    pub mime: String,
}

/// Снимок директории: путь → метаданные.
#[derive(Debug, Clone, PartialEq, Default)]
pub struct Snapshot {
    pub entries: BTreeMap<String, FileMeta>,
}

impl Snapshot {
    pub fn from_entries(entries: Vec<Entry>) -> Self {
        let mut map = BTreeMap::new();
        for e in entries {
            map.insert(e.path, FileMeta { size: e.size, mime: e.mime });
        }
        Self { entries: map }
    }
    pub fn len(&self) -> usize { self.entries.len() }
    pub fn is_empty(&self) -> bool { self.entries.is_empty() }
}

/// Разница между двумя снимками.
#[derive(Debug, Clone, PartialEq, Default)]
pub struct Diff {
    pub added: Vec<String>,
    pub removed: Vec<String>,
    pub size_changed: Vec<(String, f64, f64)>,
    pub unchanged_count: usize,
}

impl Diff {
    pub fn total_changes(&self) -> usize {
        self.added.len() + self.removed.len() + self.size_changed.len()
    }

    pub fn describe(&self) -> Vec<String> {
        let mut out = Vec::new();
        out.push(format!(
            "diff: +{} / -{} / ~{} / ={}",
            self.added.len(),
            self.removed.len(),
            self.size_changed.len(),
            self.unchanged_count
        ));
        for p in &self.added { out.push(format!("  + {p}")); }
        for p in &self.removed { out.push(format!("  - {p}")); }
        for (p, old, new) in &self.size_changed {
            out.push(format!("  ~ {p}: {old} → {new}"));
        }
        out
    }
}

/// Снять снимок директории.
pub async fn snapshot(
    root: &FileSystemDirectoryHandle,
    max_files: usize,
    on_progress: impl FnMut(usize),
    is_cancelled: impl Fn() -> bool,
) -> Result<Snapshot, JsValue> {
    let entries = scan(root, max_files, on_progress, is_cancelled).await?;
    Ok(Snapshot::from_entries(entries))
}

/// Чистая функция: сравнить два снимка. Без I/O.
pub fn diff(old: &Snapshot, new: &Snapshot) -> Diff {
    let old_keys: BTreeSet<&String> = old.entries.keys().collect();
    let new_keys: BTreeSet<&String> = new.entries.keys().collect();

    let added: Vec<String> = new_keys.difference(&old_keys)
        .map(|s| (*s).clone()).collect();
    let removed: Vec<String> = old_keys.difference(&new_keys)
        .map(|s| (*s).clone()).collect();

    let mut size_changed: Vec<(String, f64, f64)> = Vec::new();
    let mut unchanged_count = 0;
    for (path, old_meta) in &old.entries {
        if let Some(new_meta) = new.entries.get(path) {
            if (old_meta.size - new_meta.size).abs() > f64::EPSILON {
                size_changed.push((path.clone(), old_meta.size, new_meta.size));
            } else {
                unchanged_count += 1;
            }
        }
    }

    Diff { added, removed, size_changed, unchanged_count }
}
```

### Что даёт `BTreeMap`, а не `HashMap`

- **Порядок обхода.** `BTreeMap` итерируется по возрастанию ключа — значит, `diff` всегда возвращает `added`/`removed` в одном и том же порядке. Полезно для тестов и для стабильного UI.
- **Дешёвые `difference`.** `BTreeSet::difference` работает за O(n), а не за O(n·log n), как для `HashSet`, потому что оба множества отсортированы.
- **Воспроизводимость тестов.** Если бы ключи шли в случайном порядке, тест `assert_eq!(diff.added, vec!["a.txt"])` иногда падал бы. С `BTreeMap` такого нет.

### Почему `diff` — чистая функция

`diff` не принимает `&FileSystemDirectoryHandle` и не делает `await`. Это следствие осознанного дизайна:

1. **Тестируется без браузера.** Host-тесты (`cargo test`) проверяют краевые случаи: пустые снимки, полностью идентичные снимки, добавление, удаление, изменение размера, одновременное добавление + удаление.
2. **Композируется.** Тот же `diff` можно применить к снимкам, полученным не через OPFS, а через серверный API — типы не завязаны на FS Access API.
3. **Не тормозит UI.** Даже для 10 000 файлов `diff` укладывается в доли миллисекунды — его не нужно разбивать на yield-и.

---

## 19.7.9. Тесты

### `tests/pure_dir_diff.rs` — host-тесты, без браузера

```rust
use std::collections::BTreeMap;
use hello_wasm02::fsa_dir_diff::{diff, FileMeta, Snapshot};

fn snap(items: &[(&str, f64)]) -> Snapshot {
    let mut entries = BTreeMap::new();
    for (path, size) in items {
        entries.insert(
            (*path).to_string(),
            FileMeta { size: *size, mime: String::new() },
        );
    }
    Snapshot { entries }
}

#[test]
fn diff_identical_snapshots() {
    let a = snap(&[("a.txt", 10.0), ("b.txt", 20.0)]);
    let b = snap(&[("a.txt", 10.0), ("b.txt", 20.0)]);
    let d = diff(&a, &b);
    assert!(d.added.is_empty());
    assert!(d.removed.is_empty());
    assert!(d.size_changed.is_empty());
    assert_eq!(d.unchanged_count, 2);
}

#[test]
fn diff_added_and_removed() {
    let a = snap(&[("keep.txt", 1.0), ("old.txt", 2.0)]);
    let b = snap(&[("keep.txt", 1.0), ("new.txt", 3.0)]);
    let d = diff(&a, &b);
    assert_eq!(d.added, vec!["new.txt".to_string()]);
    assert_eq!(d.removed, vec!["old.txt".to_string()]);
    assert_eq!(d.unchanged_count, 1);
}

#[test]
fn diff_size_changed() {
    let a = snap(&[("f.bin", 100.0)]);
    let b = snap(&[("f.bin", 250.0)]);
    let d = diff(&a, &b);
    assert_eq!(d.size_changed, vec![("f.bin".to_string(), 100.0, 250.0)]);
}

#[test]
fn diff_order_is_deterministic() {
    // Один и тот же diff, посчитанный много раз, даёт один и тот же
    // порядок — благодаря BTreeMap/BTreeSet, а не HashMap.
    let a = snap(&[("a", 1.0), ("c", 3.0), ("e", 5.0)]);
    let b = snap(&[("b", 2.0), ("d", 4.0)]);
    let first = diff(&a, &b);
    for _ in 0..20 {
        assert_eq!(diff(&a, &b), first);
    }
}
```

### `tests/wasm_fsa_dir_tree.rs` — WASM, headless Chrome

Все операции идут через OPFS, потому что `showDirectoryPicker` в headless-режиме без user gesture недоступен:

```rust
mod opfs {
    use wasm_bindgen::prelude::*;
    use wasm_bindgen::JsCast;
    use wasm_bindgen_futures::JsFuture;
    use web_sys::FileSystemDirectoryHandle;

    pub async fn root() -> Result<FileSystemDirectoryHandle, JsValue> {
        let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
        let navigator_js = JsValue::from(window.navigator());
        let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
        if storage_js.is_undefined() {
            return Err(JsValue::from_str("navigator.storage недоступен"));
        }
        let get_dir_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str("getDirectory"))?;
        if get_dir_js.is_undefined() {
            return Err(JsValue::from_str("OPFS недоступен"));
        }
        let get_dir: js_sys::Function = get_dir_js.dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
        let promise: js_sys::Promise = get_dir.call0(&storage_js)?
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не Promise"))?;
        let value: JsValue = JsFuture::from(promise).await?;
        value.dyn_into::<FileSystemDirectoryHandle>()
            .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
    }

    pub async fn cleanup(root: &FileSystemDirectoryHandle, name: &str) {
        let _ = hello_wasm02::fsa_dir_tree::remove_path(root, name, true).await;
    }
}

use hello_wasm02::fsa_dir_tree::*;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_ensure_path_idempotent() {
    let root = match opfs::root().await { Ok(r) => r, Err(_) => return };
    let base = "fsa_dir_tree_ensure";
    opfs::cleanup(&root, base).await;

    for _ in 0..3 {
        let dir = ensure_path(&root, &format!("{base}/a/b/c"))
            .await.expect("ensure idempotent");
        assert_eq!(dir.name(), "c");
    }

    opfs::cleanup(&root, base).await;
}

#[wasm_bindgen_test]
async fn test_build_tree_nested() {
    let root = match opfs::root().await { Ok(r) => r, Err(_) => return };
    let base = "fsa_dir_tree_build";
    opfs::cleanup(&root, base).await;
    // ... создаём readme.md, src/main.rs, src/lib.rs, src/util/io.rs ...
    let dir = open_dir(&root, base).await.expect("open base");
    let tree = build_tree(&dir, &TreeOptions::default(), |_| {}, || false)
        .await.expect("build tree");
    assert_eq!(tree.total_files(), 4);
    let text = tree.render();
    assert!(text.contains("src/"));
    assert!(text.contains("main.rs"));
    opfs::cleanup(&root, base).await;
}

#[wasm_bindgen_test]
async fn test_remove_path_idempotent() {
    let root = match opfs::root().await { Ok(r) => r, Err(_) => return };
    let base = "fsa_dir_tree_remove";
    opfs::cleanup(&root, base).await;
    // ... создаём demo_19_7/dir/file.txt ...
    for _ in 0..3 {
        remove_path(&root, &format!("{base}/dir"), true)
            .await.expect("remove idempotent");
    }
    remove_path(&root, "never/existed/at/all.txt", false)
        .await.expect("missing parent → Ok");
    opfs::cleanup(&root, base).await;
}
```

Полный файл (с `test_find_by_path_file_and_dir`, `test_build_tree_skips_configured_dirs`, `test_build_tree_respects_max_depth`, `test_path_of_uses_resolve`) лежит в проекте.

### `tests/wasm_fsa_dir_diff.rs` — end-to-end цикл

```rust
#[wasm_bindgen_test]
async fn test_end_to_end_diff_cycle() {
    let root = match opfs::root().await { Ok(r) => r, Err(_) => return };
    let base = "fsa_dir_diff_e2e";
    opfs::cleanup(&root, base).await;

    write_file(&root, &format!("{base}/keep.txt"), b"same").await;
    write_file(&root, &format!("{base}/gone.txt"), b"bye").await;

    let dir = hello_wasm02::fsa_dir_tree::open_dir(&root, base)
        .await.expect("open");

    let before = snapshot(&dir, 100, |_| {}, || false).await.expect("before");

    remove_path(&root, &format!("{base}/gone.txt"), false)
        .await.expect("remove gone");
    write_file(&root, &format!("{base}/fresh.txt"), b"new").await;
    write_file(&root, &format!("{base}/grown.txt"), b"12345").await;

    let after = snapshot(&dir, 100, |_| {}, || false).await.expect("after");
    let d = diff(&before, &after);

    assert_eq!(d.removed, vec!["gone.txt".to_string()]);
    assert_eq!(d.added, vec!["fresh.txt".to_string(), "grown.txt".to_string()]);
    assert!(d.size_changed.is_empty());
    assert_eq!(d.unchanged_count, 1);

    opfs::cleanup(&root, base).await;
}
```

---

## 19.7.10. Демо: интерактивный файловый менеджер

### Что показывает

Прошлый вариант демо был `<pre>` с текстовым логом. Проблема не в том, что он работал, а в том, что он **не показывал то, ради чего существует `FileSystemDirectoryHandle`**: иерархию, изменения, идемпотентность.

Новое демо — это **интерактивный файловый менеджер** с визуализацией всех ключевых идей раздела:

| Элемент UI | Что показывает | Какая тема |
|---|---|---|
| Живая **панель прогресса** во время `build_tree` | Обход итеративный, счётчик растёт в реальном времени | Итеративный обход через `VecDeque` |
| **Настоящее DOM-дерево** с `<details>` | Иерархия читается глазами, папки сворачиваются | `DirInfo` vs плоский `Vec<Entry>` |
| **Подсчёт размера** каждой папки в заголовке | `total_bytes()` работает рекурсивно по поддереву | Композиция дерева |
| Кнопка **«Мутировать»** | Циклично: modify → add → remove | Работа с реальной ФС |
| Кнопка **«Снимок B + diff»** | Дерево **раскрашивается** зелёным/красным/жёлтым | `Snapshot` + `diff` |
| Кнопка **«Идемпотентность ×3»** | Три зелёные строки `Ok` в журнале | Правило проекта |
| Кнопка **`resolve`** | Круговой рейс: туда и обратно | `find_by_path` + `resolve` |

### Три ключевых трюка

**1. MERGE-режим для подсветки удалённых файлов.**

Простое дерево из OPFS не может показать удалённые файлы — их там **уже нет**. Классическая ошибка: взять `build_tree` и раскрасить. Она даёт только `added` и `changed`, а `removed` молча теряется.

Правильное решение — после diff'а строить дерево из **объединения снимков A и B**:

```rust
fn merge_snapshots(root_name: &str, a: &Snapshot, b: &Snapshot) -> DirInfo {
    let mut paths: BTreeMap<String, f64> = BTreeMap::new();
    for (p, m) in &a.entries { paths.insert(p.clone(), m.size); }
    for (p, m) in &b.entries { paths.insert(p.clone(), m.size); }
    let entries: Vec<(String, f64)> = paths.into_iter().collect();
    build_tree_from_paths(root_name, entries)
}
```

B приоритетнее: если файл есть в обоих — берём актуальный размер. Удалённые файлы есть в A — попадают в дерево и получают класс `removed`. Теперь все три категории видны одновременно.

**2. Пути в дереве должны совпадать с путями в `Diff`.**

`scan`/`build_tree` строят пути **относительно переданного root**, без имени root. Если передали `open_dir(root, "demo_19_7")`, то пути — `readme.md`, `src/util/io.rs`. Значит и `render_tree_html` должен начинать обход с **пустого** префикса:

```rust
fn render_tree_html(dir: &DirInfo, diff: Option<&Diff>) -> String {
    // ...
    for c in &dir.children {
        render_node_html(&mut out, c, "", 1, diff);  // ← parent_path = ""
    }
    // ...
}
```

Если начать с `dir.name` (`"demo_19_7"`), в дереве получатся `"demo_19_7/readme.md"`, а в `Diff` — `"readme.md"`. Ни одна строка не совпадёт, подсветки не будет. Это самая коварная ошибка: внешне всё работает, лог молчит, класс не навешивается.

**3. Три операции мутации по счётчику.**

Изначально `mutate_random` умел только `add` и `remove`. Из-за этого `size_changed` всегда был пуст, и жёлтая подсветка не появлялась. Добавили третью операцию — **modify** — и упорядочили по счётчику кликов:

```rust
let counter: u32 = {
    let mut c = h.mutate_counter.borrow_mut();
    *c = c.wrapping_add(1);
    *c
};
let op: u32 = counter % 3;

match op {
    1 => { /* MODIFY: перезапись длинным содержимым */ }
    2 => { /* ADD: новый src/gen_NNN.rs */ }
    _ => { /* REMOVE: один из жертвенных файлов */ }
}
```

Первый клик = modify, второй = add, третий = remove, четвёртый снова = modify. Пользователь за первые три клика гарантированно видит все три цвета. Без счётчика мы бы возвращались в случайные категории и могли долго не показать нужную.

**4. Единое визуальное поле.**

Три уровня защиты от чужого CSS (`main.css` может содержать `html { background: white }` и резать страницу на тёмную и светлую половины):

```rust
pub fn run() -> Result<(), JsValue> {
    let body = /* ... */;
    body.set_inner_html("");                       // 1) снести boilerplate
    strip_external_stylesheets(&document);          // 2) удалить все <link rel=stylesheet>
    pin_root_style(&document);                     // 3) inline-стиль на <html>
    // 4) вставить наш единственный <style>
    // ...
}
```

`strip_external_stylesheets` ищет все `link[rel='stylesheet']` и удаляет их из DOM. `pin_root_style` вешает inline-стиль прямо на `documentElement` — inline имеет приоритет над любым CSSOM-правилом без `!important`.

### Tooltips на кнопках

Каждая кнопка получает `title` с кратким описанием действия. Это не украшение — это **часть образовательной ценности демо**: студент видит не только «что нажимать», но и **что именно произойдёт**.

```rust
fn make_button<F>(
    doc: &Document,
    parent: &Element,
    label: &str,
    tooltip: &str,
    mut handler: F,
) -> Result<(), JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    btn.set_class_name("btn");
    btn.set_text_content(Some(label));
    btn.set_attribute("title", tooltip)?;   // ← тултип
    parent.append_child(&btn)?;

    let closure = Closure::<dyn FnMut()>::new(move || handler());
    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}
```

### CSS-подсветка

Чтобы подсветка была видна **всегда**, используем три сигнала одновременно: фон, левая полоса через `box-shadow: inset`, цвет имени:

```css
.tree-line.added,
details.tree-node.added > summary {
  background: rgba(46, 160, 67, .30) !important;
  box-shadow: inset 4px 0 0 0 #3fb950 !important;
}
.tree-line.added .name { color: #3fb950 !important; font-weight: 600; }

.tree-line.removed,
details.tree-node.removed > summary {
  background: rgba(248, 81, 73, .30) !important;
  box-shadow: inset 4px 0 0 0 #f85149 !important;
}
.tree-line.removed .name {
  color: #f85149 !important; font-weight: 600;
  text-decoration: line-through;
}

.tree-line.changed,
details.tree-node.changed > summary {
  background: rgba(210, 153, 34, .30) !important;
  box-shadow: inset 4px 0 0 0 #d29922 !important;
}
.tree-line.changed .name { color: #d29922 !important; font-weight: 600; }
```

`box-shadow: inset` даёт цветную полосу слева, которая **не мешает `padding`**. Просто фон `rgba(…, .22)` может сливаться с тёмным фоном панели — полоса гарантирует видимость.

---

## 19.7.11. Правила проекта, выученные в этой главе

Собраны все грабли, на которые мы наступили по ходу реализации.

**1. `format!` и `vec!` не должны стоять в tail-позиции `if/else`/`match`/блока.** На `wasm32-unknown-unknown` компилятор иногда выводит тип такого выражения как `()`, что даёт E0308. Правило:

```rust
// Плохо:
fn f() -> Vec<String> {
    vec!["a".to_string()]  // ← tail-позиция, E0308 на WASM
}

// Хорошо:
fn f() -> Vec<String> {
    let v: Vec<String> = vec!["a".to_string()];
    v
}
```

**2. `HtmlElement::style()` требует feature `CssStyleDeclaration`.** Если он не включён в `Cargo.toml`, используем `Element::set_attribute("style", "...")`. Мы контролируем inline-стиль целиком, поэтому перезапись строки безопасна.

**3. `scroll_height()` → `i32`, `set_scroll_top()` → `f64`.** Каст — только через промежуточную переменную с аннотацией:

```rust
let max: f64 = el.scroll_height() as f64;
el.set_scroll_top(max);
```

**4. Пути в `Diff` и в дереве должны совпадать.** `scan`/`build_tree` строят пути **относительно переданного root**, без имени root. `render_tree_html` тоже должен начинать с пустого префикса.

**5. Удалённых файлов нет в OPFS.** Чтобы показать их в дереве, нужно строить дерево из **объединения Snapshot A ∪ Snapshot B**, а не из текущего состояния.

**6. Правило идемпотентности.** `NotFoundError` в `remove_entry` нормализуется в `Ok(())`. `remove_path` идемпотентен дважды: родителя нет → `Ok`, записи нет → `Ok`. Любая деструктивная операция, доступная из UI, должна быть идемпотентна.

**7. `values()` — это `AsyncIterator`, не `Promise`.** Ручное разворачивание через `iter_next`: `next()` → `Promise<{value, done}>` → `JsFuture`.

**8. Итеративный обход, а не рекурсивный.** `VecDeque` + `while let` вместо `Box::pin` рекурсии. Даёт отмену в любой момент и не растёт в размере future.

**9. Yield раз в 16 мс.** `TimeoutFuture::new(0).await` даёт браузеру перерисовать UI. Без этого кнопка «Отмена» замирает.

**10. `BTreeMap` вместо `HashMap` для снимков.** Детерминированный порядок — стабильные тесты и стабильный UI.

**11. Ошибки не глотать.** `.ok()` в `spawn_local` прячет падения. Все ошибки — в `console_err` с контекстом.

**12. Логи — без двойного префикса.** `console_log` добавляет `TAG` ровно один раз. Вызывающий код не дублирует.

---

## 19.7.12. Чек-лист

- [ ] `getFileHandle` / `getDirectoryHandle` принимают `{create}`; при `false` отсутствие файла — это `NotFoundError`.
- [ ] `removeEntry(name, {recursive})` **должен** быть идемпотентным: `NotFoundError` → `Ok(())`.
- [ ] `resolve(child)` возвращает массив сегментов или `null`; не путать с «пустым путём».
- [ ] `values()` возвращает **async-iterator**, а не `Promise`. Нужен ручной `iter_next`.
- [ ] Обход — **итеративный** через `VecDeque`. Рекурсивный `async fn` требует `Box::pin` на каждом уровне.
- [ ] Внутри длительного обхода — **yield раз в 16 мс**.
- [ ] `scan` принимает `on_progress` и `is_cancelled` как замыкания — модуль не зависит от UI.
- [ ] Иерархическое дерево строится **синхронной** рекурсией из плоского списка (глубина ограничена, `Box::pin` не нужен).
- [ ] `ensure_path` идемпотентен бесплатно: `create: true` возвращает `Ok` для существующих сегментов.
- [ ] `remove_path` идемпотентен дважды: нет родителя → `Ok`, нет записи → `Ok`.
- [ ] `Snapshot` использует `BTreeMap` — детерминированный порядок для тестов и UI.
- [ ] `diff` — **чистая функция**, тестируется host-тестами без браузера.
- [ ] Diff-визуализация требует **MERGE-режима**: дерево из объединения A и B, а не из OPFS.
- [ ] Пути в дереве и в `Diff` должны совпадать: обход начинается с **пустого** префикса.
- [ ] Мутация — три операции, не две: `modify` (жёлтый), `add` (зелёный), `remove` (красный).
- [ ] Тесты FS Access API идут в OPFS (`navigator.storage.getDirectory()`), а не через `showDirectoryPicker`.
- [ ] Удаление трижды подряд — стандартный тест на идемпотентность.
- [ ] Единое визуальное поле: `<body>` очищается, внешние `<link>` удаляются, inline-стиль на `<html>`.
- [ ] Tooltips на кнопках: `title` с кратким описанием действия.
- [ ] Ошибки не глотаются: `.ok()` в `spawn_local` — только для UI-задач, критические пути пишут в `console_err`.

---

## 19.7.13. Что дальше

Демо покрывает все ключевые идеи раздела: иерархический обход, diff двух состояний, идемпотентные операции, `resolve`. Дальше — `fsa_api_project.rs`, где эти примитивы собираются в **аудит проекта**: набор правил («нет файлов > 5 МБ», «нет `.map` в проде»), обход по дереву с проверкой каждого файла, отчёт в виде `Diff`. Принципы остаются те же: итеративный обход, идемпотентные операции, чистая логика проверок отдельно от I/O.