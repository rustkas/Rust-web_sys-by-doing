# Глава 18. File API

## 18.2. Чтение текста: `FileReader`, `FileReaderSync`

В подглаве 18.1 мы научились получать метаданные файла, но не его содержимое. Теперь разберём, как прочитать текст из `File` (или `Blob`) — асинхронно через `FileReader` в основном потоке и синхронно через `FileReaderSync` внутри Web Worker. Асинхронный вариант строится на событиях `onload` / `onerror`, которые мы оборачиваем в `Promise` и используем через `async/await` в Rust.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # pub mod file_reader; (уже подключён)
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.2
│   ├── file_reader.rs             # модуль подглавы 18.2
│   ├── file_api.rs                # из подглавы 18.1, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует (экспортирует log!)
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_file_reader.rs        # тесты подглавы 18.2
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/file_reader.rs` — модуль, вводимый этой подглавой. Он автономен и не зависит от других модулей проекта, кроме `web-sys` и `wasm-bindgen`.
- `src/app.rs` — обновлён: в `run()` добавлена демо-секция «18.2. FileReader, FileReaderSync». Существующие секции не тронуты.
- `tests/wasm_file_reader.rs` — тесты, соответствующие модулю.

---

## Ссылки для детального изучения

- **MDN: FileReader** — https://developer.mozilla.org/en-US/docs/Web/API/FileReader
- **MDN: FileReaderSync** — https://developer.mozilla.org/en-US/docs/Web/API/FileReaderSync
- **MDN: FileReader.readAsText()** — https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsText
- **MDN: FileReaderSync.readAsText()** — https://developer.mozilla.org/en-US/docs/Web/API/FileReaderSync/readAsText
- **web-sys: FileReader** — https://docs.rs/web-sys/latest/web_sys/struct.FileReader.html
- **web-sys: FileReaderSync** — https://docs.rs/web-sys/latest/web_sys/struct.FileReaderSync.html
- **Спецификация W3C File API** — https://w3c.github.io/FileAPI/

---

## Содержание новых / обновлённых файлов

### `src/file_reader.rs`

Модуль автономен: зависимость только от `wasm-bindgen`, `wasm-bindgen-futures`, `js-sys`, `web-sys`. Никаких импортов из других модулей проекта.

```rust
//! FileReader и FileReaderSync — чтение содержимого файлов.
//! Асинхронный вариант работает в основном потоке,
//! синхронный — только внутри Web Workers.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileReader, FileReaderSync};

/// Асинхронно читает `Blob` как текст.
///
/// Внутри оборачивает события `onload` / `onerror` в `Promise`,
/// чтобы вызывающая сторона могла использовать `async/await`.
pub async fn read_as_text(blob: &Blob) -> Result<String, JsValue> {
    let reader = FileReader::new()?;
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let reader_clone = reader.clone();
        let onload = Closure::once_into_js(move || {
            let _ = resolve.call0(&JsValue::NULL);
        });
        let onerror = Closure::once_into_js(move |err: JsValue| {
            let _ = reject.call1(&JsValue::NULL, &err);
        });
        reader_clone.set_onload(Some(onload.unchecked_ref()));
        reader_clone.set_onerror(Some(onerror.unchecked_ref()));
    });

    reader.read_as_text(blob)?;
    JsFuture::from(promise).await?;

    return reader
        .result()?
        .as_string()
        .ok_or_else(|| JsValue::from_str("Не удалось получить текст"));
}

/// Синхронно читает `Blob` как текст.
///
/// Доступно только в контексте Web Worker — в основном потоке
/// браузер выбросит исключение.
pub fn read_as_text_sync(blob: &Blob) -> Result<String, JsValue> {
    let sync_reader = FileReaderSync::new()?;
    return sync_reader.read_as_text(blob);
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — нужен для `JsValue`, `Closure`, `JsCast` (`unchecked_ref`).
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{Blob, FileReader, FileReaderSync}` — все три типа используются.

Неиспользуемых импортов нет.

**Ключевые моменты:**

- `FileReader::new()` возвращает `Result<FileReader, JsValue>` — открываем `?`.
- Событийная модель: `set_onload` / `set_onerror` принимают `Option<&Function>`. Мы создаём замыкания через `Closure::once_into_js`, который возвращает `JsValue`; `unchecked_ref()` приводит его к `&Function`.
- `Closure::once_into_js` — правильный выбор: замыкание вызывается один раз и сразу освобождается, не утекая.
- `read_as_text(blob)?` запускает асинхронное чтение; `JsFuture::from(promise).await?` ждёт завершения.
- `reader.result()?` возвращает `JsValue` — `as_string()` даёт `Option<String>`.
- `read_as_text_sync` не имеет обёртки: `FileReaderSync::read_as_text` уже возвращает `Result<String, JsValue>` напрямую. Но вызывать её можно только внутри воркера — в основном потоке браузер выбросит `InvalidStateError`.

**Конвенция проекта:** tail-выражение, возвращающее `Result<String, JsValue>`, оформлено через `return ...;`.

---

### `src/app.rs` — новая секция 18.2

Ниже — новый блок для `app.rs`. Функция `build_read_text_section` самодостаточна: её можно вставить в `run()` рядом с другими секциями без правок последних.

```rust
//! Демонстрация подглавы 18.2: FileReader / FileReaderSync.
//!
//! Секция: `<input accept=".txt,text/plain">` → кнопка
//! «Прочитать как текст» → превью до 2000 символов.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::file_reader;
use crate::{dom, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.2 ----
    build_read_text_section(&container, &closures)?;

    log!("Section 18.2 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.2: FileReader / FileReaderSync
// ---------------------------------------------------------------

fn build_read_text_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.2. FileReader / FileReaderSync"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите текстовый файл (.txt) и нажмите «Прочитать как текст». \
         FileReader прочитает содержимое асинхронно через события \
         onload/onerror, обёрнутые в Promise. Фильтр accept ограничивает \
         выбор .txt / text/plain.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    input.set_accept(".txt,text/plain");
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Прочитать как текст"));
    dom::append(&section, &btn)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let input_c = input.clone();
    let result_c = result.clone();

    callbacks::listen(
        &btn,
        "click",
        move |_ev: web_sys::Event| {
            let file = match input_c.files().and_then(|f| f.get(0)) {
                Some(f) => f,
                None => {
                    result_c.set_text_content(Some("Сначала выберите файл."));
                    return;
                }
            };
            let result_c = result_c.clone();
            result_c.set_text_content(Some("Читаем..."));
            spawn_local(async move {
                match file_reader::read_as_text(&file).await {
                    Ok(text) => {
                        let message = format_read_text_result(&text);
                        result_c.set_text_content(Some(&message));
                    }
                    Err(e) => {
                        let message = format!("❌ Ошибка чтения: {e:?}");
                        result_c.set_text_content(Some(&message));
                    }
                }
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.2
// ---------------------------------------------------------------

/// Формирует сообщение о результате чтения текстового файла.
/// Внутри — только `let`-биндинги; финальный `format!` идёт
/// после явного `return` — конвенция проекта.
fn format_read_text_result(text: &str) -> String {
    let total = text.len();

    let preview: String;
    if total > 2000 {
        preview = format!(
            "{}...\n\n[показаны первые 2000 из {} символов]",
            &text[..2000],
            total
        );
    } else {
        preview = text.to_string();
    }

    return format!("✅ Прочитано {} символов\n\n{}", total, preview);
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_read_text_section`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` при получении `HtmlInputElement`.
- `wasm_bindgen_futures::spawn_local` — для запуска `async`-блока из обработчика.
- `callbacks`, `EventClosures`, `dom`, `file_reader`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- `input.set_accept(".txt,text/plain")` — фильтр на уровне браузера: пользователь не сможет случайно выбрать бинарный файл.
- `input_c.files().and_then(|f| f.get(0))` — берём первый файл; `FileList::get(0)` возвращает `Option<File>`.
- `spawn_local` — запускает `async`-блок на микрозадачах браузера; главный поток при этом не блокируется.
- `format_read_text_result` — та же конвенция, что в подглаве 18.1: `format!` не в tail-позиции `if/else`, а в явном `return format!(...);`.

---

### `tests/wasm_file_reader.rs`

Тесты подглавы 18.2. Используют `Blob` вместо `File`, потому что для проверки самого механизма чтения имя и MIME не нужны.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::file_reader::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_read_as_text_hello() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"Hello, WASM!".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let text = read_as_text(&blob).await.expect("read_as_text");
    assert_eq!(text, "Hello, WASM!");
}

#[wasm_bindgen_test]
async fn test_read_as_text_multiline() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"line1\nline2\nline3".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let text = read_as_text(&blob).await.expect("read_as_text");
    assert_eq!(text.lines().count(), 3);
    assert!(text.starts_with("line1"));
    assert!(text.ends_with("line3"));
}

#[wasm_bindgen_test]
async fn test_read_as_text_empty() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let text = read_as_text(&blob).await.expect("read_as_text");
    assert_eq!(text, "");
}
```

**Что проверяем:**

- `read_as_text` возвращает ровно ту строку, что была записана в `Blob`.
- Многострочный текст сохраняет переносы строк.
- Пустой `Blob` даёт пустую строку, а не ошибку.

**Про `FileReaderSync`:** тест на него в браузере не пишется — `FileReaderSync` доступен только внутри Web Worker, а `wasm_bindgen_test` выполняется в главном потоке. Проверка синхронного чтения появится в подглаве 18.11, где мы поднимаем воркер.

---

## Команды запуска

### Сборка и запуск демо

```bash
# Проверка типов без полной сборки
cargo check --target wasm32-unknown-unknown

# Dev-сборка WASM
npm run wasm:dev

# Запуск Vite (в отдельном терминале)
npm run dev

# Или одной командой — сборка WASM + запуск Vite
npm run start
```

### Тесты подглавы 18.2

```bash
# Только тесты этой подглавы
wasm-pack test --headless --chrome --test wasm_file_reader

# То же самое для Firefox
wasm-pack test --headless --firefox --test wasm_file_reader

# Все тесты проекта
npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.2. FileReader / FileReaderSync** с полем выбора файла (принимает только `.txt`) и кнопкой «Прочитать как текст». После выбора небольшого текстового файла и нажатия кнопки в блоке результата отобразится:

```
✅ Прочитано 1234 символов

Hello, world!
Это тестовый файл.
...
```

Если файл больше 2000 символов — отобразятся первые 2000 с пометкой `[показаны первые 2000 из N символов]`.

---

## Итоги подглавы 18.2

- **`FileReader`** — асинхронный интерфейс чтения; основан на событиях `onload` / `onerror`. Мы оборачиваем их в `Promise` и используем через `async/await`.
- **`read_as_text`** — читает `Blob` (и `File` как его частный случай) в `String`.
- **`Closure::once_into_js`** — правильный способ создать одноразовое замыкание для обработчика события без утечки памяти.
- **`FileReaderSync`** — синхронный интерфейс, доступный только в Web Worker. Функция `read_as_text_sync` готова к использованию, но её вызов в главном потоке приведёт к исключению — это нормально, так задумано спецификацией.
- **`accept`** — фильтр `<input>` на стороне браузера: не даёт выбрать бинарный файл в секции, предназначенной для текста.

Модуль `file_reader.rs` автономен, тайл-выражения оформлены через явный `return`, неиспользуемых импортов нет. Демо-секция самодостаточна. Синхронный вариант (`read_as_text_sync`) будет использован в подглаве 18.11 при работе с Web Workers.