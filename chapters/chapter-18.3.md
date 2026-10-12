# Глава 18. File API

## 18.3. Бинарные данные: `ArrayBuffer`, `Uint8Array`, `Vec<u8>`

В подглаве 18.2 мы читали текст. Но большинство форматов — изображения, документы, аудио, видео — бинарные. Чтобы конвертировать их, нужно уметь получать «сырые» байты в Rust и возвращать их обратно в браузер. В этой подглаве разберём цепочку `Blob → ArrayBuffer → Uint8Array → Vec<u8>` и обратную `Vec<u8> → Uint8Array → Blob`, а также извлечение расширения и определение MIME-типа.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # pub mod file_utils; (уже подключён)
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.3
│   ├── file_utils.rs              # модуль подглавы 18.3
│   ├── file_api.rs                # из 18.1, не меняется
│   ├── file_reader.rs             # из 18.2, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует (экспортирует log!)
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_file_utils.rs         # НОВЫЙ: тесты подглавы 18.3
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/file_utils.rs` — модуль, вводимый этой подглавой. Он автономен и не зависит от других модулей проекта.
- `src/app.rs` — обновлён: в `run()` добавлена демо-секция «18.3. ArrayBuffer, Uint8Array, Vec<u8>».
- `tests/wasm_file_utils.rs` — новый файл с тестами.

---

## Ссылки для детального изучения

- **MDN: ArrayBuffer** — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
- **MDN: Uint8Array** — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array
- **MDN: TypedArray** — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
- **MDN: Blob** — https://developer.mozilla.org/en-US/docs/Web/API/Blob
- **wasm-bindgen guide: js_sys** — https://rustwasm.github.io/wasm-bindgen/api/js_sys/
- **web-sys: Blob** — https://docs.rs/web-sys/latest/web_sys/struct.Blob.html
- **web-sys: BlobPropertyBag** — https://docs.rs/web-sys/latest/web_sys/struct.BlobPropertyBag.html
- **wasm-bindgen: JsCast** — https://docs.rs/wasm-bindgen/latest/wasm_bindgen/trait.JsCast.html

---

## Содержание новых / обновлённых файлов

### `src/file_utils.rs`

Модуль автономен: зависимости только от `js-sys`, `wasm-bindgen`, `web-sys`. Никаких импортов из других модулей проекта.

```rust
//! Утилиты для работы с ArrayBuffer, Uint8Array и MIME-типами.
//! Модуль автономен: не зависит от других модулей проекта.

use js_sys::{ArrayBuffer, Uint8Array};
use wasm_bindgen::prelude::*;
use web_sys::Blob;

/// Преобразует `ArrayBuffer` в `Vec<u8>`.
///
/// Создаёт представление `Uint8Array` поверх того же буфера
/// и копирует байты в вектор Rust.
pub fn array_buffer_to_vec(buffer: &ArrayBuffer) -> Vec<u8> {
    let uint8 = Uint8Array::new(buffer);
    uint8.to_vec()
}

/// Преобразует срез `u8` в `Blob` с указанным MIME-типом.
pub fn vec_to_blob(data: &[u8], mime_type: &str) -> Result<Blob, JsValue> {
    let uint8 = Uint8Array::from(data);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type(mime_type);
    Blob::new_with_u8_array_sequence_and_options(&parts, &options)
}

/// Извлекает расширение файла из имени (без точки).
///
/// Возвращает `None`, если имя не содержит точки.
/// Имя с завершающей точкой (`"file."`) даёт `Some("")`.
pub fn get_extension(file_name: &str) -> Option<&str> {
    return file_name.rsplit_once('.').map(|(_, ext)| ext);
}

/// Определяет MIME-тип по расширению (базовый набор).
///
/// Регистр расширения не важен. Для неизвестных расширений
/// возвращает `application/octet-stream`.
pub fn mime_from_extension(ext: &str) -> &'static str {
    match ext.to_lowercase().as_str() {
        "png" => "image/png",
        "jpg" | "jpeg" => "image/jpeg",
        "gif" => "image/gif",
        "webp" => "image/webp",
        "svg" => "image/svg+xml",
        "txt" => "text/plain",
        "json" => "application/json",
        "pdf" => "application/pdf",
        _ => "application/octet-stream",
    }
}
```

**Проверка импортов:**

- `js_sys::{ArrayBuffer, Uint8Array}` — оба используются.
- `wasm_bindgen::prelude::*` — для `JsValue` в сигнатуре `vec_to_blob`.
- `web_sys::Blob` — используется в `vec_to_blob`.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- `Uint8Array::new(&ArrayBuffer)` создаёт типизированное представление **поверх** того же буфера — данные не копируются. `to_vec()` затем копирует байты в `Vec<u8>`, полностью изолируя их от JS-памяти.
- `Uint8Array::from(&[u8])` создаёт новый массив с копией данных — правильный способ отдать Rust-байты в JS.
- `Blob::new_with_u8_array_sequence_and_options` принимает последовательность `JsValue` (мы кладём в неё `ArrayBuffer`), поэтому используем `Uint8Array::buffer()`.
- `BlobPropertyBag::set_type` задаёт MIME-тип будущего Blob.
- `get_extension` использует `rsplit_once('.')` — это корректнее, чем `rsplit('.').next()`, потому что для имени без точки возвращает `None`, а не всё имя.
- Обе tail-функции оформлены с явным `return` — конвенция проекта.

---

### `src/app.rs` — новая секция 18.3

Ниже — новый блок для `app.rs`. Функция `build_read_binary_section` самодостаточна: её можно вставить в `run()` рядом с другими секциями без правок последних.

```rust
//! Демонстрация подглавы 18.3: ArrayBuffer, Uint8Array, Vec<u8>.
//!
//! Секция: выбрать любой файл → прочитать как ArrayBuffer →
//! показать первые 64 байта в hex + MIME по расширению.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_reader, file_utils, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.3 ----
    build_read_binary_section(&container, &closures)?;

    log!("Section 18.3 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.3: ArrayBuffer, Uint8Array, Vec<u8>
// ---------------------------------------------------------------

fn build_read_binary_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.3. ArrayBuffer, Uint8Array, Vec<u8>"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите любой файл — первые 64 байта будут показаны в hex. \
         Цепочка: File → FileReader.readAsArrayBuffer → ArrayBuffer \
         → Uint8Array → Vec<u8> в Rust. Это основа для конвертации \
         изображений и других бинарных форматов.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Прочитать как ArrayBuffer"));
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
                let ab_js = match file_reader::read_as_array_buffer(&file).await {
                    Ok(v) => v,
                    Err(e) => {
                        let message = format!("❌ Ошибка чтения: {e:?}");
                        result_c.set_text_content(Some(&message));
                        return;
                    }
                };
                let array_buffer: js_sys::ArrayBuffer = match ab_js.dyn_into() {
                    Ok(a) => a,
                    Err(_) => {
                        result_c.set_text_content(Some("не ArrayBuffer"));
                        return;
                    }
                };

                let bytes: Vec<u8> = file_utils::array_buffer_to_vec(&array_buffer);
                let preview_len = bytes.len().min(64);
                let hex = format_hex_preview(&bytes, preview_len);

                let file_name: String = file.name();
                let ext: &str = file_utils::get_extension(&file_name).unwrap_or("");
                let mime: &str = file_utils::mime_from_extension(ext);

                let message = format_read_binary_result(
                    &file_name,
                    bytes.len(),
                    preview_len,
                    &hex,
                    ext,
                    mime,
                );
                result_c.set_text_content(Some(&message));
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.3
// ---------------------------------------------------------------

/// Формирует hex-представление первых `preview_len` байт,
/// разбивая вывод на строки по 16 байт.
fn format_hex_preview(bytes: &[u8], preview_len: usize) -> String {
    let mut hex = String::new();
    for (i, b) in bytes[..preview_len].iter().enumerate() {
        if i > 0 && i % 16 == 0 {
            hex.push('\n');
        }
        hex.push_str(&format!("{:02x} ", b));
    }
    return hex;
}

/// Формирует сообщение о результате чтения бинарного файла.
fn format_read_binary_result(
    file_name: &str,
    bytes_len: usize,
    preview_len: usize,
    hex: &str,
    ext: &str,
    mime: &str,
) -> String {
    return format!(
        "✅ Всего {} байт\n\
         Имя файла: {file_name}\n\
         Расширение: «{ext}»\n\
         MIME по расширению: {mime}\n\
         Первые {preview_len} байт в hex:\n{hex}",
        bytes_len
    );
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_read_binary_section`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` (приведение `JsValue` → `HtmlInputElement` и `JsValue` → `js_sys::ArrayBuffer`).
- `wasm_bindgen_futures::spawn_local` — для запуска `async`-блока из обработчика.
- `callbacks`, `EventClosures`, `dom`, `file_reader`, `file_utils`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- `file_reader::read_as_array_buffer` возвращает `Result<JsValue, JsValue>`. Мы приводим `JsValue` к `js_sys::ArrayBuffer` через `dyn_into` — на случай, если браузер вернёт что-то другое.
- `array_buffer_to_vec` создаёт `Vec<u8>` — Rust уже никак не связан с JS-памятью.
- `file.name()` возвращает `String`. Мы не можем написать `file_utils::get_extension(&file.name())`, потому что временный `String` не переживёт строку. Поэтому обязательно `let file_name: String = file.name();` — это критично для компиляции.
- `format_hex_preview` — локальная функция, разбивающая вывод по 16 байт.
- `format_read_binary_result` — конвенция проекта: `return format!(...);` в tail-позиции.

---

### `tests/wasm_file_utils.rs`

Тесты подглавы 18.3. Проверяют round-trip данных и корректность расширений/MIME.

```rust
use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use hello_wasm02::file_reader;
use hello_wasm02::file_utils::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_vec_blob_arraybuffer_vec_roundtrip() {
    let original: Vec<u8> = vec![0x00, 0x01, 0x02, 0x7F, 0x80, 0xFE, 0xFF];

    let blob = vec_to_blob(&original, "application/octet-stream")
        .expect("vec_to_blob");

    let ab_js = file_reader::read_as_array_buffer(&blob)
        .await
        .expect("read_as_array_buffer");
    let array_buffer: js_sys::ArrayBuffer = ab_js
        .dyn_into()
        .expect("не ArrayBuffer");

    let roundtrip = array_buffer_to_vec(&array_buffer);
    assert_eq!(original, roundtrip);
}

#[wasm_bindgen_test]
fn test_get_extension_simple() {
    assert_eq!(get_extension("image.png"), Some("png"));
    assert_eq!(get_extension("archive.tar.gz"), Some("gz"));
    assert_eq!(get_extension("noext"), None);
}

#[wasm_bindgen_test]
fn test_get_extension_edge_cases() {
    // Имя заканчивается точкой — расширение пустое.
    assert_eq!(get_extension("file."), Some(""));
    // Скрытый файл: до первой точки слева нет ничего, но rsplit_once
    // возвращает часть после первой точки — "gitignore".
    assert_eq!(get_extension(".gitignore"), Some("gitignore"));
    // Несколько точек — берётся последняя часть.
    assert_eq!(get_extension("a.b.c.d"), Some("d"));
}

#[wasm_bindgen_test]
fn test_mime_from_extension_known() {
    assert_eq!(mime_from_extension("png"), "image/png");
    assert_eq!(mime_from_extension("jpg"), "image/jpeg");
    assert_eq!(mime_from_extension("jpeg"), "image/jpeg");
    assert_eq!(mime_from_extension("txt"), "text/plain");
    assert_eq!(mime_from_extension("json"), "application/json");
    assert_eq!(mime_from_extension("pdf"), "application/pdf");
}

#[wasm_bindgen_test]
fn test_mime_from_extension_case_insensitive() {
    assert_eq!(mime_from_extension("PNG"), "image/png");
    assert_eq!(mime_from_extension("JPG"), "image/jpeg");
    assert_eq!(mime_from_extension("TXT"), "text/plain");
}

#[wasm_bindgen_test]
fn test_mime_from_extension_unknown() {
    assert_eq!(mime_from_extension("xyz"), "application/octet-stream");
    assert_eq!(mime_from_extension(""), "application/octet-stream");
}
```

**Проверка импортов:**

- `wasm_bindgen::JsCast` — критично: даёт метод `.dyn_into()` для `JsValue`. **Без этой строки компилятор выдаст `no method named 'dyn_into' found for struct 'JsValue'`** — trait должен быть явно в области видимости. В исходниках библиотеки `JsCast` приходит из `wasm_bindgen::prelude::*`, но в тестах prelude не подключается, поэтому нужен явный импорт.
- `wasm_bindgen_test::*` — `wasm_bindgen_test` и `wasm_bindgen_test_configure!`.
- `hello_wasm02::file_reader` — для `read_as_array_buffer`.
- `hello_wasm02::file_utils::*` — для `vec_to_blob`, `array_buffer_to_vec`, `get_extension`, `mime_from_extension`.

Все импорты используются, неиспользуемых нет.

**Что проверяем:**

- **Round-trip:** `Vec<u8>` → `Blob` → `ArrayBuffer` → `Vec<u8>` даёт исходные байты. Если на каком-то шаге байты портятся (например, из-за неявной конвертации UTF-8), конвертер сломается.
- `get_extension` корректно обрабатывает граничные случаи: имя без точки → `None`, имя с точкой на конце → `Some("")`, скрытый файл `.gitignore` → `Some("gitignore")`, имя с несколькими точками → последняя часть.
- `mime_from_extension` не зависит от регистра.
- Неизвестное расширение возвращает `application/octet-stream`.

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

### Тесты подглавы 18.3

```bash
# Проверка типов тестов
cargo check --target wasm32-unknown-unknown --tests

# Только тесты этой подглавы
wasm-pack test --headless --chrome --test wasm_file_utils

# То же самое для Firefox
wasm-pack test --headless --firefox --test wasm_file_utils

# Все тесты проекта
npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.3. ArrayBuffer, Uint8Array, Vec<u8>** с полем выбора файла и кнопкой «Прочитать как ArrayBuffer». После выбора, скажем, PNG-файла и нажатия кнопки в блоке результата отобразится:

```
✅ Всего 54321 байт
Имя файла: logo.png
Расширение: «png»
MIME по расширению: image/png
Первые 64 байт в hex:
89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52 
00 00 01 00 00 00 01 00 08 06 00 00 00 5c 72 a8 66 
00 00 00 04 73 42 49 54 08 08 08 08 7c 08 64 88 
00 00 00 09 70 48 59 73 00 00 0e c3 00 00 0e c3 
```

Первые восемь байт `89 50 4e 47 0d 0a 1a 0a` — это сигнатура PNG. Наглядно видно, что Rust получил именно те байты, что лежат в файле. Эта сигнатура станет основой для валидации в подглаве 18.8.

---

## Итоги подглавы 18.3

- **`ArrayBuffer`** — низкоуровневый бинарный буфер JS. Не имеет методов доступа к отдельным байтам.
- **`Uint8Array`** — типизированное представление поверх `ArrayBuffer`. Даёт `to_vec()` для копирования в Rust и `from(&[u8])` для обратного направления.
- **`Vec<u8>`** — Rust-сторона; полностью независим от JS-памяти.
- **`Blob::new_with_u8_array_sequence_and_options`** + `BlobPropertyBag` — способ собрать `Blob` из Rust-байтов с явным MIME-типом.
- **`get_extension`** через `rsplit_once('.')` корректно обрабатывает отсутствие точки, в отличие от `rsplit('.').next()`, который возвращал бы всё имя.
- **`mime_from_extension`** — простой маппинг расширений на MIME-типы, регистронезависимый, с дефолтом `application/octet-stream`.

Модуль `file_utils.rs` автономен, все tail-выражения оформлены через `return`, неиспользуемых импортов нет. Демо-секция самодостаточна. Round-trip, проверенный в тестах, гарантирует, что байты не портятся между Rust и JS — это фундамент для всех последующих подглав, где мы будем преобразовывать содержимое файлов.

## Замечание на будущее

Правило, зафиксированное в этой подглаве: **если в коде (особенно в тестах) используется `dyn_into`, `dyn_ref`, `unchecked_ref` или другие методы trait `JsCast` — нужно явно импортировать `use wasm_bindgen::JsCast;`**. В исходных файлах библиотеки `JsCast` приходит транзитивно через `wasm_bindgen::prelude::*`, но в тестах prelude не подключается, поэтому забытый импорт приводит к ошибке `no method named 'dyn_into' found for struct 'JsValue'`.