# 19.10. `FileSystemObserver`: реакция на изменения

## Обновлённая структура проекта

```
hello-wasm02/
├── src/
│   ├── fsa_api_observer.rs       ← НОВЫЙ: подписка на изменения файлов/папок
│   ├── app.rs                     ← ДЕМО главы 19.10 (только этот сценарий)
│   ├── lib.rs                     ← +pub mod fsa_api_observer
│   └── … (остальные модули проекта)
├── tests/
│   ├── wasm_fsa_api_observer.rs   ← НОВЫЙ: feature detection + lifecycle
│   └── … (остальные тесты)
├── pkg/                           ← артефакты wasm-pack
└── web/
    ├── index.html
    └── main.js
```

### Краткий комментарий

**`src/fsa_api_observer.rs`** — обёртка над `FileSystemObserver` (Chrome 133+, февраль 2025). Изолирует всю работу с нестабильным API внутри одного модуля, использует `js_sys::Reflect` для вызова методов — потому что в `web-sys` этого интерфейса пока нет.

**`tests/wasm_fsa_api_observer.rs`** — feature detection, жизненный цикл observer'а. Полная проверка «срабатывает ли на изменения» возможна только вручную: в headless Chrome нет внешних источников изменений.

**`src/app.rs`** — демо главы 19.10: три секции, кнопки программно меняют файлы в OPFS, а live-лог показывает, как observer реагирует **событиями**, а не polling'ом.

### Ссылки для изучения

- [MDN: FileSystemObserver](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemObserver) — справочник по API.
- [Chrome: File System Observer API](https://developer.chrome.com/blog/file-system-observer) — анонс, примеры, ограничения.
- [WHATWG File System spec: FileSystemObserver](https://fs.spec.whatwg.org/#api-filesystemobserver) — спецификация.
- [web.dev: Observe file system changes](https://web.dev/articles/file-system-observer) — практическое руководство.

---

## 19.10.1. Зачем нужен observer, если есть polling

**Polling** — это «спрашивай каждые N миллисекунд: не изменилось ли?». Он работает везде, но имеет три недостатка:

1. **Постоянная нагрузка.** Даже в покое `setInterval` срабатывает — на большом дереве это заметно.
2. **Задержка.** Изменение обнаруживается не раньше следующего тика. Интервал 1 секунда → пользователь видит изменение через секунду.
3. **Дорого на объёме.** Проверка «что изменилось» — это обход директории + сравнение. На 10 000 файлов это сотни миллисекунд каждую секунду.

**Observer** — это «скажи мне, когда что-то изменится». Браузер сам отслеживает изменения в файловой системе и присылает событие **в момент** их появления. Zero-cost в покое: нет событий — нет работы.

Разница видна на одном примере:

| Сценарий | Polling (раз в 1 сек) | Observer |
|---|---|---|
| Ничего не происходит 10 минут | 600 проверок, все пустые | 0 работы |
| Изменение в момент T | Обнаружено в T + [0, 1с] | Обнаружено в T + ~1 мс |
| Дерево 10 000 файлов | Каждая проверка ~ 100 мс | Zero-cost |

Единственная проблема observer'а — **поддержка**: Chrome 133+ (февраль 2025), Firefox и Safari пока не поддерживают. Поэтому первое, что делает код — **feature detection**.

---

## 19.10.2. Feature detection — первым делом

```rust
pub fn is_supported() -> bool {
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("FileSystemObserver"))
        .ok()
        .unwrap_or(JsValue::UNDEFINED);
    ctor.is_function()
}
```

Проверка не паникует, не бросает исключений. Просто спрашивает у глобального объекта: есть ли конструктор `FileSystemObserver`? В Chrome 133+ — есть. В Firefox/Safari — нет, вернёт `false`.

Это **правило проекта**: не полагайтесь на то, что API доступен. Если его нет — сообщите пользователю понятным текстом, а не падайте с `TypeError`.

---

## 19.10.3. Правило DOM-конструктора

Первая наивная реализация вызывает `FileSystemObserver` как обычную функцию:

```rust
// ❌ НЕ РАБОТАЕТ
let observer: JsValue = ctor
    .call1(&JsValue::NULL, callback.as_ref().unchecked_ref())?
    .into();
```

Браузер отвечает:

```
TypeError: Failed to construct 'FileSystemObserver':
Please use the 'new' operator, this DOM object constructor
cannot be called as a function.
```

`FileSystemObserver` — **DOM-конструктор**. Его надо вызывать через `new`. В `js_sys` это делается через `Reflect::construct`:

```rust
// ✅ РАБОТАЕТ
let args = js_sys::Array::new();
args.push(callback.as_ref());
let observer: JsValue = js_sys::Reflect::construct(&ctor, &args)?;
```

`Reflect::construct(&ctor, &args)` — прямая обёртка над `Reflect.construct()` в JS. Эквивалент `new FileSystemObserver(callback)`.

**Правило:** если API-метод — конструктор (начинается с заглавной буквы, вызывается через `new`), используйте `Reflect::construct`, а не `Function::call`.

---

## 19.10.4. Новый модуль `src/fsa_api_observer.rs`

```rust
//! `FileSystemObserver`: подписка на изменения файлов и папок.
//!
//! **Что это.** Интерфейс, который позволяет получать **события**
//! при изменении файла/папки — вместо того чтобы опрашивать
//! (polling) состояние каждые N миллисекунд.
//!
//! **Чем отличается от polling.**
//! * Observer: zero-cost в покое — браузер сам присылает уведомление,
//!   когда что-то реально изменилось. Нет задержки на интервал опроса.
//! * Polling: постоянная нагрузка, даже когда ничего не происходит.
//!   Задержка между изменением и обнаружением равна интервалу опроса.
//!   Дорого на больших деревьях.
//!
//! **Поддержка.** Chrome 133+ (февраль 2025). Firefox/Safari — пока нет.
//! Поэтому модуль строится по правилу «feature detection»: если
//! `FileSystemObserver` недоступен, `is_supported()` вернёт `false`,
//! а `observe` — `Err` с внятным сообщением.
//!
//! **Почему через Reflect.** В `web-sys` интерфейс `FileSystemObserver`
//! пока не экспортируется. Все методы вызываем через `js_sys::Reflect`.
//! Это общий приём для нестабильных API — он не зависит от версии
//! `web-sys`.
//!
//! **Правило DOM-конструктора.** `FileSystemObserver` вызывается
//! через `new` — в Rust это `js_sys::Reflect::construct`. Обычный
//! `Function::call1` падает с TypeError «Please use the 'new' operator».

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

// ===========================================================================
// Строковые хелперы (правило WASM-инференса)
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

// ===========================================================================
// Модель событий
// ===========================================================================

/// Тип изменения, полученного от observer'а.
#[derive(Debug, Clone, PartialEq)]
pub enum ChangeKind {
    /// Запись появилась.
    Appeared,
    /// Запись исчезла.
    Disappeared,
    /// Содержимое изменилось (размер, mtime).
    Modified,
    /// Запись перемещена (переименована или в другую папку).
    Moved,
    /// Что-то ещё — на случай будущих категорий.
    Unknown,
}

impl ChangeKind {
    pub fn from_str(s: &str) -> Self {
        match s {
            "appeared" => ChangeKind::Appeared,
            "disappeared" => ChangeKind::Disappeared,
            "modified" => ChangeKind::Modified,
            "moved" => ChangeKind::Moved,
            _ => ChangeKind::Unknown,
        }
    }

    pub fn label_ru(&self) -> &'static str {
        match self {
            ChangeKind::Appeared => "появился",
            ChangeKind::Disappeared => "удалён",
            ChangeKind::Modified => "изменён",
            ChangeKind::Moved => "перемещён",
            ChangeKind::Unknown => "изменение",
        }
    }
}

/// Одно событие об изменении.
#[derive(Debug, Clone)]
pub struct ChangeRecord {
    pub kind: ChangeKind,
    /// Имя изменённой записи (без пути — API не даёт полный путь).
    pub name: String,
    /// Был ли это файл (иначе директория).
    pub is_file: bool,
}

// ===========================================================================
// Feature detection
// ===========================================================================

/// Проверка: доступен ли `FileSystemObserver` в этом браузере.
///
/// Не паникует, не бросает исключений — возвращает `bool`.
/// Chrome 133+ → `true`. Firefox/Safari/старый Chrome → `false`.
pub fn is_supported() -> bool {
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("FileSystemObserver"))
        .ok()
        .unwrap_or(JsValue::UNDEFINED);
    let is_fn: bool = ctor.is_function();
    is_fn
}

// ===========================================================================
// Handle observer'а
// ===========================================================================

/// Handle активной подписки. `disconnect()` идемпотентен.
///
/// Держит `FileSystemObserver` и `Closure` колбэка — оба должны жить
/// столько же, сколько и подписка.
pub struct ObserverHandle {
    observer: JsValue,
    _callback: Closure<dyn FnMut(JsValue, JsValue)>,
}

impl ObserverHandle {
    /// Отписаться от наблюдения. Идемпотентно: повторный вызов — no-op.
    pub fn disconnect(&self) {
        let disconnect_fn = match js_sys::Reflect::get(
            &self.observer,
            &JsValue::from_str("disconnect"),
        ) {
            Ok(f) => f,
            Err(_) => return,
        };
        if !disconnect_fn.is_function() {
            return;
        }
        let func: js_sys::Function = match disconnect_fn.dyn_into() {
            Ok(f) => f,
            Err(_) => return,
        };
        // Игнорируем ошибку — disconnect() не должен падать.
        let _ = func.call0(&self.observer);
    }
}

// ===========================================================================
// Создание observer'а (общий хелпер)
// ===========================================================================

/// Создать `FileSystemObserver` через `new`.
///
/// **Правило DOM-конструктора.** `FileSystemObserver` — DOM-конструктор,
/// вызывать его надо через `Reflect::construct`, а не `Function::call1`.
/// Иначе браузер вернёт `TypeError: Please use the 'new' operator`.
fn construct_observer(
    callback: &Closure<dyn FnMut(JsValue, JsValue)>,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor_js = js_sys::Reflect::get(&global, &JsValue::from_str("FileSystemObserver"))?;
    let ctor: js_sys::Function = ctor_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("FileSystemObserver не функция-конструктор"))?;

    // `Reflect::construct` — аналог `new FileSystemObserver(callback)`.
    let args = js_sys::Array::new();
    args.push(callback.as_ref());
    let observer: JsValue = js_sys::Reflect::construct(&ctor, &args)?;
    Ok(observer)
}

/// Вызвать метод `observer.<name>` и вернуть результат.
fn call_method(
    observer: &JsValue,
    name: &str,
    args: &js_sys::Array,
) -> Result<JsValue, JsValue> {
    let method_js = js_sys::Reflect::get(observer, &JsValue::from_str(name))?;
    if !method_js.is_function() {
        let msg = format_str(format_args!("observer.{name} не функция"));
        return Err(JsValue::from_str(&msg));
    }
    let method: js_sys::Function = method_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Function"))?;
    method.apply(observer, args)
}

// ===========================================================================
// observe
// ===========================================================================

/// Подписаться на изменения директории.
///
/// `on_change` вызывается для каждой пачки изменений — браузер
/// группирует события и присылает массив `FileSystemObserverRecord`.
pub fn observe_dir(
    dir: &FileSystemDirectoryHandle,
    mut on_change: impl FnMut(Vec<ChangeRecord>) + 'static,
) -> Result<ObserverHandle, JsValue> {
    if !is_supported() {
        return Err(JsValue::from_str(
            "FileSystemObserver не поддерживается браузером (нужен Chrome 133+)",
        ));
    }

    let callback = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |records: JsValue, _observer: JsValue| {
            let parsed = parse_records(&records);
            if !parsed.is_empty() {
                on_change(parsed);
            }
        },
    );

    let observer: JsValue = construct_observer(&callback)?;

    // `observer.observe(dir, { recursive: true })`.
    let options = js_sys::Object::new();
    js_sys::Reflect::set(&options, &JsValue::from_str("recursive"), &JsValue::TRUE)?;

    let dir_js: &JsValue = dir.as_ref();
    let args = js_sys::Array::new();
    args.push(dir_js);
    args.push(&options);
    let _ = call_method(&observer, "observe", &args)?;

    Ok(ObserverHandle {
        observer,
        _callback: callback,
    })
}

/// Подписаться на изменения одного файла.
pub fn observe_file(
    file: &FileSystemFileHandle,
    mut on_change: impl FnMut(Vec<ChangeRecord>) + 'static,
) -> Result<ObserverHandle, JsValue> {
    if !is_supported() {
        return Err(JsValue::from_str(
            "FileSystemObserver не поддерживается браузером (нужен Chrome 133+)",
        ));
    }

    let callback = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |records: JsValue, _observer: JsValue| {
            let parsed = parse_records(&records);
            if !parsed.is_empty() {
                on_change(parsed);
            }
        },
    );

    let observer: JsValue = construct_observer(&callback)?;

    let file_js: &JsValue = file.as_ref();
    let args = js_sys::Array::new();
    args.push(file_js);
    let _ = call_method(&observer, "observe", &args)?;

    Ok(ObserverHandle {
        observer,
        _callback: callback,
    })
}

// ===========================================================================
// Разбор records
// ===========================================================================

/// Разбирает массив `FileSystemObserverRecord` в доменную модель.
fn parse_records(records: &JsValue) -> Vec<ChangeRecord> {
    let mut out: Vec<ChangeRecord> = Vec::new();

    if !js_sys::Array::is_array(records) {
        return out;
    }
    let arr: js_sys::Array = records.clone().into();
    let len: u32 = arr.length();
    for i in 0..len {
        let rec = arr.get(i);
        let kind_str: String = js_sys::Reflect::get(&rec, &JsValue::from_str("type"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();
        let kind = ChangeKind::from_str(&kind_str);

        let handle = js_sys::Reflect::get(&rec, &JsValue::from_str("changedHandle"))
            .ok()
            .unwrap_or(JsValue::UNDEFINED);

        let name: String = js_sys::Reflect::get(&handle, &JsValue::from_str("name"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();

        let kind_field: String = js_sys::Reflect::get(&handle, &JsValue::from_str("kind"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();
        let is_file: bool = kind_field == "file";

        out.push(ChangeRecord {
            kind,
            name,
            is_file,
        });
    }

    out
}

// ===========================================================================
// Публичный API для демо
// ===========================================================================

/// Создать `Rc<RefCell<Vec<String>>>` — накопитель лога.
pub fn make_log_store() -> Rc<RefCell<Vec<String>>> {
    Rc::new(RefCell::new(Vec::new()))
}

/// Утилита для форматирования одного события в строку лога.
pub fn format_record(rec: &ChangeRecord) -> String {
    let kind = rec.kind.label_ru();
    let icon: &str = match rec.kind {
        ChangeKind::Appeared => "+",
        ChangeKind::Disappeared => "−",
        ChangeKind::Modified => "~",
        ChangeKind::Moved => "→",
        ChangeKind::Unknown => "?",
    };
    let what: &str = if rec.is_file { "файл" } else { "папка" };
    let s: String = format_str(format_args!("{icon} {kind} {what}: {}", rec.name));
    s
}
```

### Что важно в модуле

**1. Feature detection — первым делом.** `is_supported()` вызывается до всех операций. Если API нет — возвращаем `Err` с внятным текстом, а не падаем с `TypeError`.

**2. Всё через `js_sys::Reflect`.** `FileSystemObserver` в `web-sys` не экспортируется. `Reflect::get` + `Reflect::construct` работают с любой версией крейта.

**3. `construct_observer` — единственное место с `new`.** Правило DOM-конструктора изолировано в одном хелпере, чтобы не размножать ошибку по коду.

**4. `Closure<dyn FnMut(JsValue, JsValue)>` — постоянный колбэк.** `FileSystemObserver` присылает два аргумента: массив `records` и сам observer. Обработчик живёт всё время подписки, поэтому хранится в `ObserverHandle`.

**5. `disconnect()` идемпотентен.** Вызываем метод, игнорируем ошибку. Двойной вызов — no-op.

**6. `parse_records` устойчив к мусору.** Если `changedHandle` не имеет `name` — берём пустую строку. Если `type` не распознан — `Unknown`. Никогда не паникуем.

---

## 19.10.5. Добавление модуля в `lib.rs`

```rust
// … существующие модули …
pub mod fsa_api_quota;
pub mod fsa_api_observer;   // ← НОВЫЙ

// …
```

---

## 19.10.6. Тесты: `tests/wasm_fsa_api_observer.rs`

```rust
//! Тесты `src/fsa_api_observer.rs`.
//!
//! **Что можно проверить в headless Chrome.**
//! * `is_supported()` — feature detection (возвращает bool).
//! * Создание observer'а — не падает, если API есть.
//! * `disconnect()` идемпотентен — два вызова подряд — ОК.
//!
//! **Что проверить нельзя.**
//! * Реальное срабатывание observer'а на изменения. В headless
//!   Chrome нет внешнего источника изменений (пользователь не
//!   редактирует файлы из проводника ОС во время теста).
//!   Полная проверка — только вручную через демо.
//!
//! Все тесты мягко пропускаются, если `FileSystemObserver`
//! не поддерживается браузером.

use wasm_bindgen_test::*;

use hello_wasm02::fsa_api_observer::{is_supported, observe_dir, ChangeKind};
use hello_wasm02::fsa_api_opfs::opfs_root;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// Feature detection
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_is_supported_returns_bool() {
    // Не паникует в любом окружении. Если API нет — false.
    let ok = is_supported();
    let msg: String = format!("FileSystemObserver supported: {ok}");
    web_sys::console::log_1(&wasm_bindgen::JsValue::from_str(&msg));
}

// ---------------------------------------------------------------------------
// Создание observer'а
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_observe_dir_creates_observer() {
    if !is_supported() {
        return; // мягкий skip: браузер не поддерживает
    }

    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return, // мягкий skip: OPFS недоступен
    };

    // Создаём observer на корне OPFS.
    let handle = observe_dir(&root, |_records| {}).expect("observe_dir");
    // Сразу отписываемся — в тесте не нужен постоянный наблюдатель.
    handle.disconnect();
}

#[wasm_bindgen_test]
async fn test_disconnect_is_idempotent() {
    if !is_supported() {
        return;
    }

    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };

    let handle = observe_dir(&root, |_| {}).expect("observe_dir");

    // Три disconnect() подряд — все Ok (правило проекта).
    handle.disconnect();
    handle.disconnect();
    handle.disconnect();
}

// ---------------------------------------------------------------------------
// Доменная модель
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_change_kind_from_str() {
    assert_eq!(ChangeKind::from_str("appeared"), ChangeKind::Appeared);
    assert_eq!(ChangeKind::from_str("disappeared"), ChangeKind::Disappeared);
    assert_eq!(ChangeKind::from_str("modified"), ChangeKind::Modified);
    assert_eq!(ChangeKind::from_str("moved"), ChangeKind::Moved);
    // Неизвестное значение → Unknown, не паника.
    assert_eq!(ChangeKind::from_str("garbage"), ChangeKind::Unknown);
    assert_eq!(ChangeKind::from_str(""), ChangeKind::Unknown);
}

#[wasm_bindgen_test]
fn test_change_kind_labels() {
    // Проверяем, что ярлыки непустые и не паникуют.
    assert!(!ChangeKind::Appeared.label_ru().is_empty());
    assert!(!ChangeKind::Disappeared.label_ru().is_empty());
    assert!(!ChangeKind::Modified.label_ru().is_empty());
    assert!(!ChangeKind::Moved.label_ru().is_empty());
    assert!(!ChangeKind::Unknown.label_ru().is_empty());
}
```

### Почему тесты такие минималистичные

**Feature detection — единственное, что можно проверить в headless.** Создание observer'а — можно (не падает). Реальное срабатывание — только вручную.

**`test_change_kind_from_str`** — проверяет, что разбор строк устойчив к мусору. Это важно: спецификация может добавить новые типы, а мы не должны падать.

**`test_disconnect_is_idempotent`** — три вызова подряд. Если бы `disconnect()` кидал исключение на втором вызове, тест бы упал. Правило проекта — все деструктивные операции идемпотентны.

---

## 19.10.7. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты observer'а в headless Chrome
wasm-pack test --headless --chrome --test wasm_fsa_api_observer

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** новых features. `.cargo/config.toml` **не требует** `--cfg=web_sys_unstable_apis`. Всё общение с `FileSystemObserver` идёт через `js_sys::Reflect`.

---

## 19.10.8. Демо-пример: `src/app.rs`

### Идея

**Задача.** Показать, что observer реагирует **на события**, а не на опрос.

**Как показать в браузере.** В OPFS файлы меняет только сам код — внешнего пользователя нет. Значит, демо должно **само** менять файлы по кнопкам, и рядом показывать:

- **Мгновенную реакцию observer'а** — событие в журнале сразу после действия.
- **Счётчик polling'а** — сколько пустых проверок он бы сделал за это же время.
- **Счётчик «observer молчит N секунд»** — наглядно видно, что observer не крутится вхолостую.

### Три секции

**❶ Изменить что-нибудь в OPFS.** Кнопки «Создать файл», «Изменить размер», «Удалить файл», «Очистить всё». Каждая меняет реальный OPFS.

**❷ События от observer'а.** Live-журнал. В нём два типа строк: `[запрос]` — что вы нажали, `[observer]` — что увидел observer. В заголовке секции — два счётчика: «событий: N» и «observer молчит: N сек».

**❸ Сравнение observer vs polling.** Две колонки: события observer'а против проверок polling'а. Счётчик polling'а тикает вхолостую каждую секунду — наглядная цена опроса.

### Wow-моменты

**Wow №1.** Жмёшь «Создать файл» — в журнале **сразу** появляется `[observer] + появился файл: observed_0.txt`. Никакой задержки на интервал опроса.

**Wow №2.** Счётчик «observer молчит N сек» растёт, пока ничего не происходит. Observer **спит** — это визуально отличается от счётчика polling'а, который тикает вхолостую.

**Wow №3.** Через минуту бездействия: «observer молчит 60 секунд, 0 событий» против «polling: 60 проверок, все пустые». Разница в эффективности видна сразу.

**Wow №4.** В Firefox — вместо журнала сообщение «нужен Chrome 133+». Не падение, а объяснение.

### Секция «Как читать это демо»

Перед секциями — плашка с тремя шагами:

1. Нажмите любую кнопку в **❶**, чтобы увидеть мгновенную реакцию в **❷**.
2. Понаблюдайте за счётчиком **«observer молчит»** — пока нет действий, он растёт.
3. Сравните с **❸**: счётчик polling'а растёт каждую секунду, даже когда ничего не происходит.

### Tooltips

На всех заголовках секций и счётчиках — атрибут `title`. Наведение мыши даёт объяснение:

- Заголовок ❶ → «Кнопки этой секции меняют файлы в наблюдаемой папке OPFS…»
- Счётчик «событий: N» → «Сколько событий observer прислал с момента подписки…»
- Счётчик «observer молчит: N с» → «Сколько секунд прошло с последнего события…»
- Заголовок ❸ → «Наглядная цена опроса…»

### Полный текст `src/app.rs`

```rust
//! Демо главы 19.10 — FileSystemObserver: события вместо polling'а.
//!
//! **Идея.** В OPFS файлы меняет только сам код — внешнего пользователя
//! нет. Поэтому демо **программно** меняет файлы по кнопкам, а observer
//! реагирует **мгновенно**. Рядом — счётчик polling'а: сколько пустых
//! проверок он сделал бы, пока observer ждал событий.
//!
//! **Wow-моменты:**
//! 1. Нажал «Создать» — событие в логе **сразу**, без интервала опроса.
//! 2. Счётчик polling'а тикает вхолостую — наглядная цена опроса.
//! 3. «Observer молчит N секунд» — видно, что он не крутится вхолостую.
//! 4. В Firefox — сообщение «нужен Chrome 133+», без падения.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>` —
//! вызывается из `#[wasm_bindgen(start)]`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции `if/else`/
//! `match`/блока иногда выводится как `()` → E0308. Все такие
//! места обёрнуты в `format_str(format_args!(…))`.
//!
//! **Правило snake_case.** `web-sys` использует snake_case:
//! `create_element`, `set_attribute`, `append_child`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, Element, FileSystemDirectoryHandle, HtmlElement};

use crate::fsa_api_observer::{
    format_record, is_supported, observe_dir, ChangeKind, ChangeRecord, ObserverHandle,
};
use crate::fsa_api_opfs::{opfs_root, remove_file, write_text};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[19.10]";

fn console_log(msg: &str) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::log_1(&JsValue::from_str(&line));
}

fn console_err(msg: &str, e: &JsValue) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::error_2(&JsValue::from_str(&line), e);
}

// ===========================================================================
// Строковые хелперы (правило проекта)
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

fn fmt_count(n: u64) -> String {
    let s: String = format_str(format_args!("{n}"));
    s
}

fn fmt_secs(n: u64) -> String {
    let s: String = format_str(format_args!("{n} с"));
    s
}

// ===========================================================================
// Состояние
// ===========================================================================

struct DemoState {
    status: Element,
    log_body: Element,
    events_count: Element,
    polls_count: Element,
    empty_polls: Element,
    silence_secs: Element,
    controls: Element,
    root: RefCell<Option<FileSystemDirectoryHandle>>,
    observer: RefCell<Option<ObserverHandle>>,
    events: Cell<u64>,
    polls: Cell<u64>,
    empty_polls_count: Cell<u64>,
    /// Секунды с последнего события observer'а.
    seconds_since_event: Cell<u64>,
    log_lines: RefCell<Vec<String>>,
    polling_on: Cell<bool>,
    next_index: Cell<u64>,
}

// ===========================================================================
// CSS
// ===========================================================================

const CSS: &str = r#"
html, body {
  margin: 0 !important; padding: 0 !important;
  background: #0d1117 !important; color: #e6edf3 !important;
  min-height: 100vh;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  font-size: 14px; line-height: 1.55;
}
header { padding: 20px 32px 4px; }
h1 { font-size: 19px; margin: 0 0 4px; font-weight: 600; }
h1 code { color: #79c0ff; font-size: 0.95em; }
.subtitle { color: #8b949e; margin: 0; font-size: 13px; }

.status-bar {
  padding: 12px 32px 4px; font-size: 12px; color: #8b949e;
  font-family: ui-monospace, monospace;
}
.status-bar.ok   { color: #3fb950; }
.status-bar.warn { color: #d29922; }
.status-bar.err  { color: #f85149; }

.section {
  margin: 16px 32px; background: #161b22;
  border: 1px solid #30363d; border-radius: 8px;
  overflow: hidden;
}
.section h3 {
  margin: 0; padding: 10px 14px;
  background: #21262d; font-size: 13px; font-weight: 600;
  border-bottom: 1px solid #30363d;
  display: flex; align-items: baseline; gap: 8px;
  cursor: help;
}
.section h3 .num { color: #79c0ff; font-weight: 700; font-size: 14px; }
.section h3 .right {
  color: #8b949e; font-weight: 400; font-size: 11px;
  margin-left: auto;
}
.section h3 .right b { color: #79c0ff; font-weight: 600; cursor: help; }
.section .body { padding: 14px 16px; }

.intro {
  margin: 12px 32px 4px;
  padding: 12px 16px;
  background: rgba(31, 111, 235, .08);
  border: 1px solid rgba(31, 111, 235, .30);
  border-radius: 8px;
  color: #8b949e; font-size: 13px; line-height: 1.6;
}
.intro b { color: #e6edf3; font-weight: 500; }
.intro code { color: #79c0ff; font-family: ui-monospace, monospace; font-size: 12px; }

.howto {
  margin: 12px 32px 4px;
  padding: 14px 18px;
  background: rgba(46, 160, 67, .08);
  border: 1px solid rgba(46, 160, 67, .30);
  border-radius: 8px;
  color: #8b949e; font-size: 13px; line-height: 1.7;
}
.howto .howto-title {
  color: #3fb950; font-weight: 600; font-size: 13px;
  margin-bottom: 6px;
}
.howto ol { margin: 4px 0 0 20px; padding: 0; }
.howto li { margin-bottom: 4px; }
.howto b { color: #e6edf3; font-weight: 500; }
.howto code { color: #79c0ff; font-family: ui-monospace, monospace; font-size: 12px; }

.controls { display: flex; flex-wrap: wrap; gap: 8px; }
.btn {
  background: #21262d; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 8px 14px; font-size: 13px; cursor: pointer;
  transition: background .15s, border-color .15s;
}
.btn:hover { background: #30363d; border-color: #8b949e; }
.btn:active { background: #1f6feb; border-color: #1f6feb; }
.btn.primary { background: #1f6feb; border-color: #1f6feb; }
.btn.primary:hover { background: #388bfd; }
.btn.danger { background: #3d1418; border-color: #6e2a2e; }
.btn.danger:hover { background: #5a1f24; border-color: #f85149; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.log-body {
  padding: 12px 16px; max-height: 240px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; white-space: pre-wrap; }
.log-line.info    { color: #8b949e; }
.log-line.request { color: #6e7681; font-style: italic; }
.log-line.add     { color: #3fb950; }
.log-line.del     { color: #f85149; }
.log-line.mod     { color: #d29922; }
.log-line.move    { color: #79c0ff; }
.log-line.empty   { color: #6e7681; font-style: italic; }
.log-line.ok      { color: #3fb950; }

.compare {
  display: flex; gap: 24px; flex-wrap: wrap;
  font-family: ui-monospace, monospace; font-size: 13px;
}
.compare .col { flex: 1; min-width: 240px; }
.compare .col-title {
  color: #8b949e; font-size: 11px;
  text-transform: uppercase; letter-spacing: .5px;
  margin-bottom: 6px; cursor: help;
}
.compare .col-value {
  font-size: 22px; font-weight: 600;
}
.compare .col-value.obs { color: #3fb950; }
.compare .col-value.poll { color: #d29922; }
.compare .col-sub {
  color: #8b949e; font-size: 11px; margin-top: 4px;
}
.compare .col-sub b { color: #e6edf3; font-weight: 500; cursor: help; }
"#;

// ===========================================================================
// Логирование
// ===========================================================================

fn log_line(state: &Rc<DemoState>, kind: &str, msg: &str) {
    {
        let mut v = state.log_lines.borrow_mut();
        v.push(msg.to_string());
        if v.len() > 300 {
            let excess = v.len() - 300;
            v.drain(0..excess);
        }
    }

    let window = match web_sys::window() {
        Some(w) => w,
        None => return,
    };
    let doc = match window.document() {
        Some(d) => d,
        None => return,
    };
    let line = match doc.create_element("div") {
        Ok(e) => e,
        Err(_) => return,
    };
    let cls: String = format_str(format_args!("log-line {kind}"));
    line.set_class_name(&cls);
    line.set_text_content(Some(msg));
    state.log_body.append_child(&line).ok();
    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let max: f64 = el.scroll_height() as f64;
        el.set_scroll_top(max);
    }
}

fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

// ===========================================================================
// Рендер счётчиков
// ===========================================================================

fn update_counters(state: &Rc<DemoState>) {
    let ev = fmt_count(state.events.get());
    let po = fmt_count(state.polls.get());
    let em = fmt_count(state.empty_polls_count.get());
    let sil = fmt_secs(state.seconds_since_event.get());

    state.events_count.set_text_content(Some(&ev));
    state.polls_count.set_text_content(Some(&po));
    state.empty_polls.set_text_content(Some(&em));
    state.silence_secs.set_text_content(Some(&sil));
}

// ===========================================================================
// Callback observer'а
// ===========================================================================

fn handle_records(state: &Rc<DemoState>, records: Vec<ChangeRecord>) {
    for rec in &records {
        let line: String = format_record(rec);
        let kind: &str = match rec.kind {
            ChangeKind::Appeared => "add",
            ChangeKind::Disappeared => "del",
            ChangeKind::Modified => "mod",
            ChangeKind::Moved => "move",
            ChangeKind::Unknown => "info",
        };
        // Префикс подчёркивает: это от observer'а, а не наш запрос.
        let prefixed: String = format_str(format_args!("[observer] {line}"));
        log_line(state, kind, &prefixed);
        state.events.set(state.events.get() + 1);
    }
    // Сбрасываем счётчик молчания при любом событии.
    state.seconds_since_event.set(0);
    update_counters(state);
}

// ===========================================================================
// Polling-эмуляция для сравнения
// ===========================================================================

fn start_polling_counter(state: Rc<DemoState>) {
    if state.polling_on.get() {
        return;
    }
    state.polling_on.set(true);

    spawn_local(async move {
        while state.polling_on.get() {
            TimeoutFuture::new(1000).await;
            if !state.polling_on.get() {
                break;
            }
            // Polling сделал бы одну проверку.
            state.polls.set(state.polls.get() + 1);
            state
                .empty_polls_count
                .set(state.empty_polls_count.get() + 1);
            // И «молчание observer'а» увеличивается каждую секунду
            // без события.
            state
                .seconds_since_event
                .set(state.seconds_since_event.get() + 1);
            update_counters(&state);
        }
    });
}

// ===========================================================================
// Действия с OPFS
// ===========================================================================

async fn create_file(state: Rc<DemoState>) -> Result<(), JsValue> {
    if !is_supported() {
        log_line(
            &state,
            "request",
            "[запрос] создать файл — пропущено (observer не поддерживается)",
        );
        return Ok(());
    }
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return Ok(()),
    };
    let n = state.next_index.get();
    state.next_index.set(n + 1);

    let name: String = format_str(format_args!("observed_{n}.txt"));
    let body: String = format_str(format_args!("создан в итерации {n}\n"));

    let req: String = format_str(format_args!("[запрос] создать файл {name}"));
    log_line(&state, "request", &req);

    if let Err(e) = write_text(&root, &name, &body).await {
        console_err("write_text failed", &e);
        return Ok(());
    }
    Ok(())
}

async fn modify_file(state: Rc<DemoState>) -> Result<(), JsValue> {
    if !is_supported() {
        log_line(
            &state,
            "request",
            "[запрос] изменить файл — пропущено (observer не поддерживается)",
        );
        return Ok(());
    }
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return Ok(()),
    };
    let current = state.next_index.get();
    if current == 0 {
        log_line(
            &state,
            "empty",
            "[запрос] нечего менять — сначала создайте файл",
        );
        return Ok(());
    }
    let n = current - 1;
    let name: String = format_str(format_args!("observed_{n}.txt"));
    let filler: String = "x".repeat(120);
    let body: String =
        format_str(format_args!("изменён в итерации {n}\n{filler}\n"));

    let req: String = format_str(format_args!("[запрос] изменить размер {name}"));
    log_line(&state, "request", &req);

    if let Err(e) = write_text(&root, &name, &body).await {
        console_err("write_text failed", &e);
        return Ok(());
    }
    Ok(())
}

async fn delete_file(state: Rc<DemoState>) -> Result<(), JsValue> {
    if !is_supported() {
        log_line(
            &state,
            "request",
            "[запрос] удалить файл — пропущено (observer не поддерживается)",
        );
        return Ok(());
    }
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return Ok(()),
    };
    let current = state.next_index.get();
    if current == 0 {
        log_line(
            &state,
            "empty",
            "[запрос] нечего удалять — сначала создайте файл",
        );
        return Ok(());
    }
    let n = current - 1;
    let name: String = format_str(format_args!("observed_{n}.txt"));

    let req: String = format_str(format_args!("[запрос] удалить {name}"));
    log_line(&state, "request", &req);

    // Идемпотентно.
    if let Err(e) = remove_file(&root, &name).await {
        console_err("remove_file failed", &e);
        return Ok(());
    }
    Ok(())
}

async fn clear_all(state: Rc<DemoState>) -> Result<(), JsValue> {
    if !is_supported() {
        log_line(
            &state,
            "request",
            "[запрос] очистить — пропущено (observer не поддерживается)",
        );
        return Ok(());
    }
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return Ok(()),
    };

    let req: String = format_str(format_args!("[запрос] очистить все observed_*.txt"));
    log_line(&state, "request", &req);

    let current = state.next_index.get();
    for n in 0..current {
        let name: String = format_str(format_args!("observed_{n}.txt"));
        // Идемпотентно: отсутствие файла — Ok.
        let _ = remove_file(&root, &name).await;
    }
    state.next_index.set(0);
    Ok(())
}

// ===========================================================================
// UI-хелперы
// ===========================================================================

fn make_button<F>(
    doc: &Document,
    parent: &Element,
    label: &str,
    tooltip: &str,
    primary: bool,
    danger: bool,
    mut handler: F,
) -> Result<Element, JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    let cls: &str = if primary {
        "btn primary"
    } else if danger {
        "btn danger"
    } else {
        "btn"
    };
    btn.set_class_name(cls);
    btn.set_text_content(Some(label));
    btn.set_attribute("title", tooltip)?;
    parent.append_child(&btn)?;

    let closure = Closure::<dyn FnMut()>::new(move || handler());
    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(btn)
}

/// Создать секцию с заголовком. `tooltip` — всплывающая подсказка
/// на `<h3>`, чтобы читатель мог навести мышь и узнать, что секция
/// делает.
fn make_section(
    doc: &Document,
    num: &str,
    title: &str,
    right_html: &str,
    tooltip: &str,
) -> Result<(Element, Element), JsValue> {
    let section = doc.create_element("div")?;
    section.set_class_name("section");

    let h3 = doc.create_element("h3")?;
    let header_html: String = format_str(format_args!(
        "<span class=\"num\">{num}</span> <span>{title}</span>\
         <span class=\"right\">{right_html}</span>"
    ));
    h3.set_inner_html(&header_html);
    h3.set_attribute("title", tooltip)?;
    section.append_child(&h3)?;

    let body = doc.create_element("div")?;
    body.set_class_name("body");
    section.append_child(&body)?;

    Ok((section, body))
}

// ===========================================================================
// Точка входа
// ===========================================================================

pub fn run() -> Result<(), JsValue> {
    console_log("run: starting");

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = document
        .body()
        .ok_or_else(|| JsValue::from_str("no body"))?;

    body.set_inner_html("");

    // Единое визуальное поле.
    if let Ok(links) = document.query_selector_all("link[rel='stylesheet']") {
        let n: u32 = links.length();
        for i in 0..n {
            if let Some(node) = links.get(i) {
                if let Some(parent) = node.parent_node() {
                    parent.remove_child(&node).ok();
                }
            }
        }
    }
    if let Some(root_el) = document.document_element() {
        root_el
            .set_attribute(
                "style",
                "background:#0d1117;color:#e6edf3;margin:0;padding:0;min-height:100vh;",
            )
            .ok();
    }

    let style = document.create_element("style")?;
    style.set_text_content(Some(CSS));
    document
        .head()
        .ok_or_else(|| JsValue::from_str("no head"))?
        .append_child(&style)
        .ok();

    // Header
    let h1 = document.create_element("h1")?;
    h1.set_inner_html("19.10 <code>FileSystemObserver</code>: события вместо polling'а");
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "подписка на изменения файлов и папок · мгновенная реакция без опроса",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    // Вступительная плашка
    let intro = document.create_element("div")?;
    intro.set_class_name("intro");
    intro.set_inner_html(
        "<b>Что такое FileSystemObserver.</b> Механизм подписки на изменения \
         файлов и папок. Вместо опроса каждые N миллисекунд (<code>polling</code>) \
         вы получаете <b>событие</b> в момент, когда изменение реально произошло.<br><br>\
         <b>Почему это важно.</b> Polling тратит ресурсы постоянно, даже когда \
         ничего не меняется. Observer в покое не стоит ничего. Плюс нет задержки \
         на интервал опроса — событие приходит сразу.<br><br>\
         <b>Поддержка.</b> Chrome 133+ (февраль 2025). Firefox и Safari — пока нет.",
    );
    body.append_child(&intro)?;

    // Секция «Как читать это демо»
    let howto = document.create_element("div")?;
    howto.set_class_name("howto");
    howto.set_inner_html(
        "<div class=\"howto-title\">🎯 Как читать это демо</div>\
         <ol>\
           <li>Нажмите любую кнопку в секции <b>❶</b> — она программно изменит \
               файлы в OPFS. Секция <b>❷</b> покажет, как observer реагирует \
               <b>мгновенно</b>: событие появляется в тот же момент, без задержки \
               на интервал опроса.</li>\
           <li>Посмотрите на счётчик <b>«observer молчит»</b> в секции ❷. Пока вы \
               не нажимаете кнопки, он растёт — observer <b>спит</b> и не тратит \
               ресурсы. Это его главное преимущество перед polling'ом.</li>\
           <li>Сравните с секцией <b>❸</b>: счётчик polling'а растёт \
               <b>каждую секунду</b>, даже когда ничего не происходит. Большинство \
               этих проверок — пустые. В этом и разница: <b>событие</b> против \
               <b>опроса</b>.</li>\
         </ol>",
    );
    body.append_child(&howto)?;

    // Status bar
    let status = document.create_element("div")?;
    status.set_class_name("status-bar");
    status.set_text_content(Some("проверяю поддержку API…"));
    body.append_child(&status)?;

    // Section ❶
    let (sec1, body1) = make_section(
        &document,
        "❶",
        "Изменить что-нибудь в OPFS",
        "observer среагирует мгновенно",
        "Кнопки этой секции меняют файлы в наблюдаемой папке OPFS: \
         создают, изменяют размер, удаляют. Каждое действие вызовет \
         событие в observer'е — увидите его в секции ❷.",
    )?;
    body.append_child(&sec1)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("controls");
    body1.append_child(&controls)?;

    // Section ❷
    let events_count = document.create_element("b")?;
    events_count.set_id("ev-num");
    events_count.set_text_content(Some("0"));
    events_count.set_attribute(
        "title",
        "Сколько событий observer прислал с момента подписки. \
         Каждое событие — реакция на реальное изменение в OPFS.",
    )?;

    let silence_secs = document.create_element("b")?;
    silence_secs.set_id("sil-num");
    silence_secs.set_text_content(Some("0 с"));
    silence_secs.set_attribute(
        "title",
        "Сколько секунд прошло с последнего события. Если число растёт — \
         observer молчит и не тратит ресурсы. Это отличает его от polling'а.",
    )?;

    let (sec2, body2) = make_section(
        &document,
        "❷",
        "События от observer'а",
        "",
        "Живой лог событий от FileSystemObserver. Здесь появляется \
         по строке на каждое изменение в OPFS. Строки с префиксом \
         [observer] — реальные события; [запрос] — что вы сделали кнопкой.",
    )?;
    body.append_child(&sec2)?;

    // Счётчики в заголовок секции ❷
    {
        let h3 = sec2
            .query_selector("h3")?
            .ok_or_else(|| JsValue::from_str("no h3"))?;
        let right = h3
            .query_selector(".right")?
            .ok_or_else(|| JsValue::from_str("no right"))?;
        let label_ev = document.create_element("span")?;
        label_ev.set_text_content(Some("событий: "));
        right.append_child(&label_ev)?;
        right.append_child(&events_count)?;

        let sep = document.create_element("span")?;
        sep.set_text_content(Some(" · observer молчит: "));
        right.append_child(&sep)?;
        right.append_child(&silence_secs)?;
    }

    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    log_body.set_inner_html(
        "<div class=\"log-line empty\">(пока пусто — нажмите кнопку в секции ❶)</div>",
    );
    body2.append_child(&log_body)?;

    // Section ❸
    let (sec3, body3) = make_section(
        &document,
        "❸",
        "Сравнение: observer vs polling",
        "polling раз в секунду, вхолостую",
        "Наглядная цена опроса. Слева — сколько событий observer прислал \
         (он молчит, когда нет изменений). Справа — сколько проверок сделал бы \
         polling за то же время. Почти все они — пустые.",
    )?;
    body.append_child(&sec3)?;

    let compare = document.create_element("div")?;
    compare.set_class_name("compare");

    // Колонка observer
    let col_obs = document.create_element("div")?;
    col_obs.set_class_name("col");
    let title_obs = document.create_element("div")?;
    title_obs.set_class_name("col-title");
    title_obs.set_text_content(Some("Observer"));
    title_obs.set_attribute(
        "title",
        "Счётчик реальных событий от observer'а. Он растёт только тогда, \
         когда что-то действительно изменилось. В покое — стоит на месте.",
    )?;
    col_obs.append_child(&title_obs)?;
    let value_obs = document.create_element("div")?;
    value_obs.set_class_name("col-value obs");
    value_obs.set_id("obs-count");
    value_obs.set_text_content(Some("0"));
    value_obs.set_attribute(
        "title",
        "Количество событий от observer'а с момента подписки.",
    )?;
    col_obs.append_child(&value_obs)?;
    let sub_obs = document.create_element("div")?;
    sub_obs.set_class_name("col-sub");
    sub_obs.set_text_content(Some("событий — приходят мгновенно, без опроса"));
    col_obs.append_child(&sub_obs)?;
    compare.append_child(&col_obs)?;

    // Колонка polling
    let col_poll = document.create_element("div")?;
    col_poll.set_class_name("col");
    let title_poll = document.create_element("div")?;
    title_poll.set_class_name("col-title");
    title_poll.set_text_content(Some("Polling (раз в секунду)"));
    title_poll.set_attribute(
        "title",
        "Сколько проверок сделал бы polling за это же время. Растёт \
         каждую секунду — независимо от того, есть ли изменения.",
    )?;
    col_poll.append_child(&title_poll)?;
    let value_poll = document.create_element("div")?;
    value_poll.set_class_name("col-value poll");
    value_poll.set_id("poll-count");
    value_poll.set_text_content(Some("0"));
    value_poll.set_attribute(
        "title",
        "Общее число проверок, которые сделал бы polling. \
         Большинство из них — пустые (ничего не изменилось).",
    )?;
    col_poll.append_child(&value_poll)?;

    let sub_poll = document.create_element("div")?;
    sub_poll.set_class_name("col-sub");
    sub_poll.set_inner_html(
        "проверок всего · <b id=\"empty-count\" title=\"Сколько проверок \
         оказались бы пустыми — то есть за это время ничего не менялось.\
         \">0</b> из них пустых",
    );
    col_poll.append_child(&sub_poll)?;
    compare.append_child(&col_poll)?;

    body3.append_child(&compare)?;

    let polls_count = compare
        .query_selector("#poll-count")?
        .ok_or_else(|| JsValue::from_str("no poll-count"))?;
    let empty_polls = compare
        .query_selector("#empty-count")?
        .ok_or_else(|| JsValue::from_str("no empty-count"))?;

    // State
    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        log_body: log_body.clone(),
        events_count,
        polls_count,
        empty_polls,
        silence_secs,
        controls: controls.clone(),
        root: RefCell::new(None),
        observer: RefCell::new(None),
        events: Cell::new(0),
        polls: Cell::new(0),
        empty_polls_count: Cell::new(0),
        seconds_since_event: Cell::new(0),
        log_lines: RefCell::new(Vec::new()),
        polling_on: Cell::new(false),
        next_index: Cell::new(0),
    });

    // Кнопки секции ❶
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Создать файл",
            "Записывает новый observed_N.txt в OPFS. Observer получит событие 'appeared'.",
            true,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = create_file(s).await {
                        console_err("create_file failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Изменить размер",
            "Перезаписывает последний observed_N.txt содержимым другого размера. Observer получит 'modified'.",
            false,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = modify_file(s).await {
                        console_err("modify_file failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Удалить файл",
            "Удаляет последний observed_N.txt. Observer получит 'disappeared'. Идемпотентно.",
            false,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = delete_file(s).await {
                        console_err("delete_file failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Очистить всё",
            "Удаляет все observed_*.txt. Повторное нажатие — тоже успех (идемпотентность).",
            false,
            true,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = clear_all(s).await {
                        console_err("clear_all failed", &e);
                    }
                });
            },
        )?;
    }

    // Инициализация
    {
        let s = state.clone();
        spawn_local(async move {
            console_log("инициализация…");

            if !is_supported() {
                set_status(
                    &s,
                    "err",
                    "FileSystemObserver не поддерживается — нужен Chrome 133+",
                );
                log_line(
                    &s,
                    "info",
                    "FileSystemObserver недоступен. Демо работает только в Chrome 133+ (февраль 2025).",
                );

                s.controls
                    .set_attribute("style", "opacity:.4;pointer-events:none")
                    .ok();
                return;
            }

            let root = match opfs_root().await {
                Ok(r) => r,
                Err(e) => {
                    console_err("opfs_root failed", &e);
                    set_status(&s, "err", "OPFS недоступен");
                    log_line(&s, "info", "OPFS недоступен — демо не может работать.");
                    return;
                }
            };

            set_status(
                &s,
                "ok",
                "FileSystemObserver поддерживается · OPFS доступен",
            );
            log_line(&s, "ok", "[observer] FileSystemObserver поддерживается.");
            log_line(&s, "info", "[observer] Подписка на изменения корня OPFS…");
            console_log("subscribing to OPFS root");

            let s_for_cb = s.clone();
            let handle = match observe_dir(&root, move |records| {
                handle_records(&s_for_cb, records);
            }) {
                Ok(h) => h,
                Err(e) => {
                    console_err("observe_dir failed", &e);
                    set_status(&s, "err", "не удалось подписаться");
                    let msg: String = format_str(format_args!(
                        "[observer] Ошибка подписки: {e:?}"
                    ));
                    log_line(&s, "info", &msg);
                    return;
                }
            };

            *s.root.borrow_mut() = Some(root);
            *s.observer.borrow_mut() = Some(handle);

            log_line(
                &s,
                "ok",
                "[observer] Подписка активна. Нажмите кнопку в секции ❶.",
            );

            s.log_body.set_inner_html("");

            start_polling_counter(s.clone());
        });
    }

    console_log("run: finished OK");
    Ok(())
}
```

---

## 19.10.9. Правила проекта, выученные в этой главе

**1. Feature detection — первым делом.** `is_supported()` вызывается до всех операций. Если API нет — `Err` с внятным сообщением, а не падение.

**2. DOM-конструкторы требуют `Reflect::construct`.** `FileSystemObserver` — не функция, а конструктор. `Function::call1` падает с `TypeError: Please use the 'new' operator`. Правильный вызов — `js_sys::Reflect::construct(&ctor, &args)`.

**3. Нестабильные API — через `js_sys::Reflect`.** В `web-sys` интерфейс пока не экспортируется. `Reflect::get` + `Reflect::construct` + `Function::apply` работают с любой версией крейта.

**4. `Closure<dyn FnMut(...)>` — постоянный колбэк.** Обработчик, переданный в браузерный API, должен жить дольше вызова. Храните его в handle рядом с самим объектом.

**5. `disconnect()` идемпотентен.** Три вызова подряд — все `Ok`. Правило проекта.

**6. Observer в покое — zero-cost.** Это главное отличие от polling'а. Счётчик «observer молчит N сек» показывает это визуально.

**7. `format_str` объявлена в начале файла.** Первое, что читатель видит после doc-comment, — функции работы со строками.

**8. Все методы `web-sys` — в snake_case.** `createElement` → `create_element`, `appendChild` → `append_child`, `setAttribute` → `set_attribute`.

**9. Импорты — по имени.** Если тип используется в теле функции, он должен быть в `use`. Компилятор подсказывает: «consider importing this struct».

**10. Tooltips — часть образовательной ценности.** На всех заголовках секций и счётчиках — атрибут `title`. Читатель наводит мышь и понимает, что перед ним.

---

## 19.10.10. Чек-лист

- [ ] `is_supported()` — feature detection, возвращает `bool`, не падает.
- [ ] `FileSystemObserver` создаётся через `Reflect::construct`, а не `Function::call`.
- [ ] Колбэк хранится в `ObserverHandle` — иначе `Closure` будет собран и события перестанут приходить.
- [ ] `disconnect()` идемпотентен — три вызова подряд, всё `Ok`.
- [ ] `parse_records` устойчив к мусору — неизвестный `type` → `Unknown`, отсутствие `name` → пустая строка.
- [ ] Импорты включают все типы, используемые в теле функций (`ChangeRecord`, `ObserverHandle`, `ChangeKind`).
- [ ] `format_str(format_args!(…))` вместо `format!` в tail-позиции `if/else`/`match`/блока.
- [ ] Все методы `web-sys` — snake_case.
- [ ] Секции и счётчики имеют tooltips.
- [ ] Демо работает в Chrome 133+, а в Firefox/Safari показывает понятное сообщение, не падает.
- [ ] Счётчик polling'а тикает вхолостую — наглядная демонстрация цены опроса.
- [ ] Счётчик «observer молчит N сек» сбрасывается при каждом событии.

---

## 19.10.11. Что дальше

Мы научились подписываться на изменения файловой системы **событиями**, а не опросом. Это последний «сетевой» слой работы с OPFS:

- **19.7** — структура: дерево, пути, diff.
- **19.8** — производительность: OPFS, sync access handle.
- **19.9** — устойчивость: квота, persist, usageDetails.
- **19.10** — реактивность: события вместо polling'а.

Дальше — либо `fsa_api_project.rs` (аудит проекта поверх дерева с проверками), либо знакомство с **SQLite WASM**, где OPFS используется как файловая система для полноценной SQL-базы в браузере. Принципы остаются те же: feature detection, идемпотентность, единое визуальное поле, явные ошибки в консоль.