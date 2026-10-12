# Часть VI. Файлы — фундамент нашего конвертера

## Глава 18. File API

В этой главе мы познакомимся с File API — набором интерфейсов, позволяющих веб-приложениям читать содержимое файлов, выбранных пользователем через `<input type="file">` или перетаскиванием (drag & drop). Это основа для нашего будущего конвертера: без умения получать бинарные данные из файла невозможно ни конвертировать изображения, ни парсить документы.

Мы рассмотрим ключевые типы: `File`, `Blob`, `FileList`, `FileReader`, `FileReaderSync`, разберёмся с MIME-типами, метаданными, работой с `ArrayBuffer` и `Uint8Array`, а также реализуем практический проект — загрузку файла на Rust-сервер.

---

## 18.1. Обновлённая структура файлов и папок

После добавления модулей для работы с файлами структура проекта примет следующий вид (показаны только новые и изменённые элементы):

```
hello-wasm02/
├── Cargo.toml                     # + фичи web-sys для File API
├── src/
│   ├── lib.rs                     # + подключение новых модулей
│   ├── app.rs                     # демо-пример главы 18
│   ├── file_api.rs                # НОВЫЙ: File, Blob, FileList, метаданные
│   ├── file_reader.rs             # НОВЫЙ: FileReader, FileReaderSync
│   ├── file_utils.rs              # НОВЫЙ: ArrayBuffer, Uint8Array, MIME
│   ├── drag_drop.rs               # НОВЫЙ: Drag & Drop API
│   └── file_upload.rs             # НОВЫЙ: практический проект — загрузка
├── tests/
│   ├── wasm_file_api.rs           # НОВЫЙ: тесты File, Blob, FileList
│   ├── wasm_file_reader.rs        # НОВЫЙ: тесты FileReader
│   ├── wasm_drag_drop.rs          # НОВЫЙ: тесты Drag & Drop
│   └── wasm_file_upload.rs        # НОВЫЙ: интеграционный тест загрузки
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- Все новые модули изолированы: они не зависят от существующих `app.rs`, `fetch.rs` и т.д., поэтому могут использоваться независимо.
- `lib.rs` дополняется только объявлениями `pub mod`, существующий код не затрагивается.
- `app.rs` полностью заменяется новым демо-примером — предыдущий пример (Streams в Service Worker) удаляется.
- В `Cargo.toml` добавляются фичи `web-sys`, необходимые для File API.

---

## 18.2. Ссылки для детального изучения

- **MDN File API** — обзорное руководство: https://developer.mozilla.org/en-US/docs/Web/API/File_API
- **MDN File** — интерфейс файла: https://developer.mozilla.org/en-US/docs/Web/API/File
- **MDN Blob** — бинарные объекты: https://developer.mozilla.org/en-US/docs/Web/API/Blob
- **MDN FileReader** — асинхронное чтение: https://developer.mozilla.org/en-US/docs/Web/API/FileReader
- **MDN FileReaderSync** — синхронное чтение в воркерах: https://developer.mozilla.org/en-US/docs/Web/API/FileReaderSync
- **MDN Drag and Drop API**: https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API
- **web-sys документация** (File, FileList, FileReader): https://docs.rs/web-sys/latest/web_sys/struct.File.html
- **wasm-bindgen guide** — работа с `js_sys::Uint8Array`: https://rustwasm.github.io/wasm-bindgen/api/js_sys/struct.Uint8Array.html

---

## 18.3. Обновление `Cargo.toml`

В секцию `[dependencies.web-sys] features` добавляем фичи для File API:

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    # … существующие фичи …

    # File API
    "File",
    "FileList",
    "FileReader",
    "FileReaderSync",
    "FilePropertyBag",   # для File::new_with_str_sequence_and_options
    "DataTransfer",
    "DragEvent",
    "DragEventInit",     # для DragEvent::new_with_event_init_dict
    "ProgressEvent",

    # … остальные существующие фичи …
]
```

**Пояснения:**

- `File` и `FileList` — для получения выбранных файлов.
- `FileReader` — асинхронное чтение в основном потоке.
- `FileReaderSync` — синхронное чтение (только в Web Workers).
- `FilePropertyBag` — словарь опций для `File` (позволяет задать MIME-тип).
- `DataTransfer` и `DragEvent` — для обработки drag & drop.
- `DragEventInit` — словарь опций для конструирования `DragEvent` (в тестах).
- `ProgressEvent` — для отслеживания прогресса чтения.

Проверьте также, что уже присутствуют: `HtmlInputElement`, `HtmlAnchorElement`, `Url`, `Blob`, `BlobPropertyBag`.

---

## 18.4. Новые модули

### 18.4.1. `src/file_api.rs` — File, Blob, FileList, метаданные

Безопасные обёртки над `web_sys::File` и `web_sys::FileList`.

```rust
//! Работа с File, Blob, FileList и метаданными файлов.
//! Модуль автономен: не зависит от других модулей проекта.

use web_sys::{File, FileList};

/// Метаданные файла: имя, размер, MIME-тип, дата изменения.
#[derive(Debug, Clone, PartialEq)]
pub struct FileMetadata {
    pub name: String,
    pub size: f64,
    pub mime_type: String,
    pub last_modified: f64,
}

impl FileMetadata {
    /// Извлекает метаданные из `web_sys::File`.
    pub fn from_file(file: &File) -> Self {
        Self {
            name: file.name(),
            size: file.size(),
            mime_type: file.type_(),
            last_modified: file.last_modified(),
        }
    }
}

/// Возвращает количество файлов в списке.
pub fn file_list_length(list: &FileList) -> u32 {
    list.length()
}

/// Возвращает файл по индексу, если он существует.
pub fn file_list_get(list: &FileList, index: u32) -> Option<File> {
    list.get(index)
}

/// Преобразует `FileList` в вектор метаданных.
pub fn file_list_metadata(list: &FileList) -> Vec<FileMetadata> {
    (0..list.length())
        .filter_map(|i| list.get(i))
        .map(|f| FileMetadata::from_file(&f))
        .collect()
}

/// Проверяет, является ли файл изображением (по MIME-типу).
pub fn is_image(file: &File) -> bool {
    file.type_().starts_with("image/")
}

/// Форматирует размер файла в человекочитаемый вид.
pub fn format_file_size(bytes: f64) -> String {
    const UNITS: &[&str] = &["B", "KB", "MB", "GB", "TB"];
    let mut size = bytes;
    let mut unit_idx: usize = 0;
    while size >= 1024.0 && unit_idx < UNITS.len() - 1 {
        size /= 1024.0;
        unit_idx += 1;
    }
    return format!("{:.2} {}", size, UNITS[unit_idx]);
}
```

**Ключевое:**

- `File` предоставляет методы `name()`, `size()`, `type_()`, `last_modified()` — доступ к метаданным.
- `FileList::get(index)` возвращает `Option<File>` — безопаснее прямого индексирования.
- `format_file_size` использует явный `return format!(...);` — конвенция проекта.

---

### 18.4.2. `src/file_reader.rs` — FileReader и FileReaderSync

Асинхронное и синхронное чтение содержимого файлов.

```rust
//! FileReader и FileReaderSync — чтение содержимого файлов.
//! Асинхронный вариант работает в основном потоке,
//! синхронный — только внутри Web Workers.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileReader, FileReaderSync};

/// Асинхронно читает файл как ArrayBuffer.
pub async fn read_as_array_buffer(blob: &Blob) -> Result<JsValue, JsValue> {
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
    reader.read_as_array_buffer(blob)?;
    JsFuture::from(promise).await?;
    reader.result()
}

/// Асинхронно читает файл как текст.
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

/// Синхронно читает Blob как ArrayBuffer.
/// Доступно только в контексте Web Worker.
pub fn read_as_array_buffer_sync(blob: &Blob) -> Result<js_sys::ArrayBuffer, JsValue> {
    let sync_reader = FileReaderSync::new()?;
    sync_reader.read_as_array_buffer(blob)
}
```

**Пояснения:**

- `FileReader` использует события `onload` / `onerror`, обёрнутые в `Promise` — так мы получаем `async/await` в Rust.
- `FileReaderSync` — синхронный интерфейс, доступный только в Web Workers.
- `return reader.result()?...;` — соблюдена конвенция явного `return` для tail-выражений.

---

### 18.4.3. `src/file_utils.rs` — ArrayBuffer, Uint8Array, MIME

Вспомогательные функции для работы с бинарными данными.

```rust
//! Утилиты для работы с ArrayBuffer, Uint8Array и MIME-типами.

use js_sys::{ArrayBuffer, Uint8Array};
use wasm_bindgen::prelude::*;
use web_sys::Blob;

/// Преобразует `ArrayBuffer` в `Vec<u8>`.
pub fn array_buffer_to_vec(buffer: &ArrayBuffer) -> Vec<u8> {
    let uint8 = Uint8Array::new(buffer);
    uint8.to_vec()
}

/// Преобразует `Vec<u8>` в `Blob` с указанным MIME-типом.
pub fn vec_to_blob(data: &[u8], mime_type: &str) -> Result<Blob, JsValue> {
    let uint8 = Uint8Array::from(data);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type(mime_type);
    Blob::new_with_u8_array_sequence_and_options(&parts, &options)
}

/// Извлекает расширение файла из имени.
pub fn get_extension(file_name: &str) -> Option<&str> {
    file_name.rsplit('.').next()
}

/// Определяет MIME-тип по расширению (базовый набор).
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

**Ключевое:**

- `Uint8Array::new(&array_buffer)` создаёт типизированный массив из `ArrayBuffer`.
- `Uint8Array::to_vec()` копирует данные в `Vec<u8>`.
- `Blob` создаётся из последовательности `Uint8Array` с `BlobPropertyBag`.

---

### 18.4.4. `src/drag_drop.rs` — Drag & Drop

Обработка перетаскивания файлов.

```rust
//! Drag & Drop — получение файлов через DataTransfer.

use web_sys::{DataTransfer, DragEvent, FileList};
use crate::file_api::FileMetadata;

/// Извлекает список файлов из события `drop`.
/// `DataTransfer::files()` возвращает `Option<FileList>`.
pub fn files_from_drop_event(event: &DragEvent) -> Option<FileList> {
    let dt: DataTransfer = event.data_transfer()?;
    dt.files()
}

/// Извлекает метаданные всех файлов из события `drop`.
pub fn metadata_from_drop_event(event: &DragEvent) -> Vec<FileMetadata> {
    files_from_drop_event(event)
        .map(|list| crate::file_api::file_list_metadata(&list))
        .unwrap_or_default()
}

/// Предотвращает стандартное поведение браузера при drag&drop.
pub fn prevent_default(event: &DragEvent) {
    event.prevent_default();
    event.stop_propagation();
}
```

**Важное замечание по версии `web-sys`:**

В актуальной версии `web-sys` `DataTransfer::files()` возвращает `Option<FileList>`, а не `FileList`. Поэтому мы возвращаем его напрямую, без `Some(...)`. Это тонкий момент, который ломает сборку, если его упустить.

---

### 18.4.5. `src/file_upload.rs` — практический проект: загрузка файла в Rust

Полный цикл: выбор файла → чтение → отправка на сервер.

```rust
//! Практический проект: загрузка файла на Rust-сервер.
//! Использует fetch с FormData.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, File, FormData, Request, RequestInit, Response};

/// Загружает файл на сервер по указанному URL.
/// Возвращает текст ответа сервера.
pub async fn upload_file(file: &File, url: &str) -> Result<String, JsValue> {
    let form_data = FormData::new()?;
    form_data.append_with_blob("file", file)?;

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_body(&form_data);

    let request = Request::new_with_str_and_init(url, &opts)?;
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("Нет window"))?;
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;

    if !resp.ok() {
        return Err(JsValue::from_str(&format!("HTTP ошибка: {}", resp.status())));
    }

    let text = JsFuture::from(resp.text()?).await?;
    return text
        .as_string()
        .ok_or_else(|| JsValue::from_str("Ответ не является строкой"));
}

/// Загружает Blob (например, результат конвертации) на сервер.
pub async fn upload_blob(blob: &Blob, filename: &str, url: &str) -> Result<String, JsValue> {
    let form_data = FormData::new()?;
    form_data.append_with_blob_and_filename("file", blob, filename)?;

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_body(&form_data);

    let request = Request::new_with_str_and_init(url, &opts)?;
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("Нет window"))?;
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;

    if !resp.ok() {
        return Err(JsValue::from_str(&format!("HTTP ошибка: {}", resp.status())));
    }

    let text = JsFuture::from(resp.text()?).await?;
    return text
        .as_string()
        .ok_or_else(|| JsValue::from_str("Ответ не является строкой"));
}
```

**Что важно понимать:**

- В актуальной версии `web-sys` методы `RequestInit::set_method` и `set_body` принимают `&self` — `mut` не нужен, только `let opts = RequestInit::new();`.
- Устаревшие `method()` / `body()` использовать не стоит — они помечены deprecated.
- `return text.as_string().ok_or_else(...)` в конце функции — конвенция явного `return`.

---

### 18.4.6. `src/lib.rs` — подключение модулей

```rust
// … существующие модули …
pub mod file_api;
pub mod file_reader;
pub mod file_utils;
pub mod drag_drop;
pub mod file_upload;
```

---

## 18.5. Тесты

### `tests/wasm_file_api.rs`

```rust
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;
use hello_wasm02::file_api::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_format_file_size() {
    assert_eq!(format_file_size(512.0), "512.00 B");
    assert_eq!(format_file_size(2048.0), "2.00 KB");
    assert_eq!(format_file_size(1024.0 * 1024.0), "1.00 MB");
}

#[wasm_bindgen_test]
fn test_file_metadata_from_file() {
    // FilePropertyBag позволяет задать MIME-тип явно.
    let parts = js_sys::Array::of1(&JsValue::from_str("hello"));
    let bag = web_sys::FilePropertyBag::new();
    bag.set_type("text/plain");

    let file = web_sys::File::new_with_str_sequence_and_options(
        &parts,
        "test.txt",
        &bag,
    )
    .expect("File::new_with_str_sequence_and_options");

    let meta = FileMetadata::from_file(&file);
    assert_eq!(meta.name, "test.txt");
    assert_eq!(meta.mime_type, "text/plain");
    assert!(meta.size > 0.0);
}
```

**Ключевое:**

- `FilePropertyBag::set_type` устанавливает MIME-тип, иначе `file.type_()` вернёт пустую строку.

### `tests/wasm_file_reader.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::file_reader::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_read_as_text() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"Hello, WASM!".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let text = read_as_text(&blob).await.expect("read_as_text");
    assert_eq!(text, "Hello, WASM!");
}
```

### `tests/wasm_drag_drop.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::drag_drop::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_prevent_default() {
    // По умолчанию DragEvent не cancelable — prevent_default() не сработает.
    // Создаём событие с cancelable = true через DragEventInit.
    let init = web_sys::DragEventInit::new();
    init.set_bubbles(true);
    init.set_cancelable(true);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    prevent_default(&event);
    assert!(event.default_prevented());
}

#[wasm_bindgen_test]
fn test_files_from_drop_event_no_data_transfer() {
    // Событие без DataTransfer: должно вернуться None, без паники.
    let init = web_sys::DragEventInit::new();
    init.set_cancelable(true);
    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    assert!(files_from_drop_event(&event).is_none());
    assert!(metadata_from_drop_event(&event).is_empty());
}
```

**Ключевое:**

- `DragEventInit::set_cancelable(true)` — обязательное условие для того, чтобы `default_prevented()` вернул `true`.
- Тесты требуют фичи `DragEventInit` в `Cargo.toml`.

### `tests/wasm_file_upload.rs`

Тест интеграционный — требует запущенного `auth-server`. Помечен `#[ignore]`.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::file_upload::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
#[ignore = "требует запущенного auth-server на localhost:3000"]
async fn test_upload_file() {
    let file = web_sys::File::new_with_str_sequence(
        &js_sys::Array::of1(&"test data".into()),
        "test.txt",
    )
    .expect("File::new_with_str_sequence");

    let result = upload_file(&file, "http://localhost:3000/upload").await;
    assert!(result.is_ok(), "upload_file failed: {:?}", result.err());
}
```

---

## 18.6. Демонстрация в браузере — `src/app.rs`

Полностью новый пример: пять секций с разными сценариями File API.

```rust
//! Демонстрация главы 18: File API.
//!
//! Секции:
//! 1. `<input type="file">` — метаданные выбранных файлов.
//! 2. Drag & Drop — перетаскивание файлов в drop-зону.
//! 3. FileReader::readAsText — чтение файла как текст (только .txt).
//! 4. FileReader::readAsArrayBuffer — бинарный предпросмотр (hex).
//! 5. Blob — создание и скачивание через Object URL.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, drag_drop, file_api, file_reader, file_utils, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Демонстрация File API: выбор файлов, Drag & Drop, чтение \
         бинарных данных через FileReader, создание и скачивание Blob.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    build_input_section(&container, &closures)?;
    build_drag_drop_section(&container, &closures)?;
    build_read_text_section(&container, &closures)?;
    build_read_binary_section(&container, &closures)?;
    build_blob_section(&container, &closures)?;

    log!("File API demo ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 1: <input type="file">
// ---------------------------------------------------------------

fn build_input_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = build_section(parent, "1. Выбор файла через <input type=\"file\">")?;

    let hint = dom::create_paragraph(
        "Выберите один или несколько файлов — отобразятся их \
         метаданные (имя, размер, MIME-тип, дата изменения). \
         Фильтр не задан — можно выбрать любой файл.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    input.set_multiple(true);
    dom::append(&section, &input)?;

    let result = make_result(&section)?;
    let result_c = result.clone();

    callbacks::listen(
        &input,
        "change",
        move |ev: web_sys::Event| {
            let target = match ev.target() {
                Some(t) => t,
                None => {
                    result_c.set_text_content(Some("нет target"));
                    return;
                }
            };
            let input_el: web_sys::HtmlInputElement = match target.dyn_into() {
                Ok(el) => el,
                Err(_) => {
                    result_c.set_text_content(Some("target не input"));
                    return;
                }
            };
            let files = match input_el.files() {
                Some(f) => f,
                None => {
                    result_c.set_text_content(Some("файлы не выбраны"));
                    return;
                }
            };
            let meta = file_api::file_list_metadata(&files);
            let message = format_metadata_list(&meta);
            result_c.set_text_content(Some(&message));
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Секция 2: Drag & Drop
// ---------------------------------------------------------------

fn build_drag_drop_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = build_section(parent, "2. Drag & Drop — перетащите файлы")?;

    let hint = dom::create_paragraph(
        "Перетащите файлы из проводника в область ниже. \
         Обработчики dragover и drop вызывают prevent_default, \
         чтобы браузер не открывал файл вместо нашей обработки.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    const STYLE_IDLE: &str = "border: 2px dashed #888; padding: 30px; \
        text-align: center; margin: 10px 0; color: #555; cursor: pointer;";
    const STYLE_HOVER: &str = "border: 2px dashed #2b8; padding: 30px; \
        text-align: center; margin: 10px 0; color: #2b8; cursor: pointer;";

    let drop_zone = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&drop_zone), "drop-zone")?;
    drop_zone.set_attribute("style", STYLE_IDLE)?;
    drop_zone.set_text_content(Some("Перетащите файлы сюда"));
    dom::append(&section, &drop_zone)?;

    let result = make_result(&section)?;

    {
        let drop_zone_c = drop_zone.clone();
        callbacks::listen(
            &drop_zone,
            "dragover",
            move |ev: web_sys::Event| {
                if let Ok(drag_ev) = ev.dyn_into::<web_sys::DragEvent>() {
                    drag_drop::prevent_default(&drag_ev);
                }
                drop_zone_c.set_attribute("style", STYLE_HOVER).ok();
            },
            closures,
        )?;
    }

    {
        let drop_zone_c = drop_zone.clone();
        callbacks::listen(
            &drop_zone,
            "dragleave",
            move |_ev: web_sys::Event| {
                drop_zone_c.set_attribute("style", STYLE_IDLE).ok();
            },
            closures,
        )?;
    }

    {
        let drop_zone_c = drop_zone.clone();
        let result_c = result.clone();
        callbacks::listen(
            &drop_zone,
            "drop",
            move |ev: web_sys::Event| {
                let drag_ev: web_sys::DragEvent = match ev.dyn_into() {
                    Ok(e) => e,
                    Err(_) => {
                        result_c.set_text_content(Some("событие не DragEvent"));
                        return;
                    }
                };
                drag_drop::prevent_default(&drag_ev);
                drop_zone_c.set_attribute("style", STYLE_IDLE).ok();

                let meta = drag_drop::metadata_from_drop_event(&drag_ev);
                if meta.is_empty() {
                    result_c.set_text_content(Some("В drop-событии нет файлов."));
                } else {
                    let message = format_metadata_list(&meta);
                    result_c.set_text_content(Some(&message));
                }
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Секция 3: чтение как текст
// ---------------------------------------------------------------

fn build_read_text_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = build_section(parent, "3. FileReader::readAsText")?;

    let hint = dom::create_paragraph(
        "Выберите текстовый файл (.txt) и нажмите «Прочитать как текст». \
         FileReader прочитает содержимое асинхронно через \
         события onload/onerror, обёрнутые в Promise. \
         Фильтр accept ограничивает выбор .txt / text/plain.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    // Фильтр: только текстовые файлы.
    input.set_accept(".txt,text/plain");
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Прочитать как текст"));
    dom::append(&section, &btn)?;

    let result = make_result(&section)?;
    let result_c = result.clone();
    let input_c = input.clone();

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
// Секция 4: чтение как ArrayBuffer
// ---------------------------------------------------------------

fn build_read_binary_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = build_section(parent, "4. FileReader::readAsArrayBuffer")?;

    let hint = dom::create_paragraph(
        "Выберите любой файл — первые 64 байта будут показаны в hex. \
         ArrayBuffer → Uint8Array → Vec<u8> в Rust. Это основа для \
         конвертации изображений и других бинарных форматов. \
         Фильтр не задан — принимается любой файл.",
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

    let result = make_result(&section)?;
    let result_c = result.clone();
    let input_c = input.clone();

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
                let bytes = file_utils::array_buffer_to_vec(&array_buffer);
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
// Секция 5: Blob — создание и скачивание
// ---------------------------------------------------------------

fn build_blob_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = build_section(parent, "5. Blob: создание и скачивание")?;

    let hint = dom::create_paragraph(
        "Создаёт Blob из байтов и скачивает его через Object URL. \
         Это базовый приём для экспорта результатов конвертации.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Создать Blob и скачать"));
    dom::append(&section, &btn)?;

    let result = make_result(&section)?;
    let result_c = result.clone();

    callbacks::listen(
        &btn,
        "click",
        move |_ev: web_sys::Event| {
            match create_and_download_blob() {
                Ok(name) => {
                    let message = format!("✅ Blob создан и скачан как {name}");
                    result_c.set_text_content(Some(&message));
                }
                Err(e) => {
                    let message = format!("❌ Ошибка: {e:?}");
                    result_c.set_text_content(Some(&message));
                }
            }
        },
        closures,
    )?;

    return Ok(());
}

fn create_and_download_blob() -> Result<String, JsValue> {
    let content = "Hello, Blob!\n\
                   Это содержимое сгенерировано в Rust и упаковано в Blob.\n\
                   Такой приём используется для экспорта результатов.\n";
    let blob = file_utils::vec_to_blob(content.as_bytes(), "text/plain")?;

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let url = web_sys::Url::create_object_url_with_blob(&blob)?;

    let a: web_sys::HtmlAnchorElement = document
        .create_element("a")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не anchor"))?;
    a.set_href(&url);
    a.set_download("hello-blob.txt");
    a.click();

    web_sys::Url::revoke_object_url(&url)?;

    return Ok("hello-blob.txt".to_string());
}

// ---------------------------------------------------------------
// Вспомогательные функции
// ---------------------------------------------------------------

fn build_section(
    parent: &web_sys::Element,
    title: &str,
) -> Result<web_sys::Element, JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some(title));
    dom::append(&section, &h2)?;

    dom::append(parent, &section)?;
    return Ok(section);
}

fn make_result(parent: &web_sys::Element) -> Result<web_sys::Element, JsValue> {
    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    dom::append(parent, &result)?;
    return Ok(result);
}

/// Формирует сообщение о результате чтения текстового файла.
/// Внутри использует только `let`-биндинги; финальный `format!`
/// всегда идёт после явного `return` — конвенция проекта.
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
         MIME по расширению «{ext}»: {mime}\n\
         Первые {preview_len} байт в hex:\n{hex}",
        bytes_len
    );
}

/// Формирует человекочитаемый список метаданных файлов.
fn format_metadata_list(items: &[file_api::FileMetadata]) -> String {
    if items.is_empty() {
        return "Список файлов пуст.".to_string();
    }
    let mut out = format!("Выбрано файлов: {}\n\n", items.len());
    for (i, m) in items.iter().enumerate() {
        out.push_str(&format!(
            "{}. {}\n   Размер: {}\n   MIME: {}\n   Изменён: {}\n\n",
            i + 1,
            m.name,
            file_api::format_file_size(m.size),
            if m.mime_type.is_empty() {
                "(не определён)"
            } else {
                &m.mime_type
            },
            m.last_modified,
        ));
    }
    return out;
}
```

---

## 18.7. Команды запуска

### Сборка и запуск

```bash
# Сборка WASM (dev)
npm run wasm:dev

# Запуск Vite + сборка WASM
npm run start

# Запуск только Rust-сервера (для тестов загрузки)
npm run server

# Запуск полного стека (сервер + Vite)
npm run dev:full

# Проверка типов без полной сборки
cargo check --target wasm32-unknown-unknown
```

### Тесты

```bash
# Отдельные новые тесты главы 18
wasm-pack test --headless --chrome --test wasm_file_api
wasm-pack test --headless --chrome --test wasm_file_reader
wasm-pack test --headless --chrome --test wasm_drag_drop

# Интеграционный тест загрузки (требует запущенного auth-server)
npm run server   # в отдельном терминале
wasm-pack test --headless --chrome --test wasm_file_upload -- --ignored

# Все сразу — три первых
wasm-pack test --headless --chrome \
  --test wasm_file_api \
  --test wasm_file_reader \
  --test wasm_drag_drop

# Через npm-скрипты (все браузеры)
npm run test:chrome
npm run test:firefox
npm run test:node
npm run test
```

---

## 18.8. Конвенции проекта, зафиксированные в этой главе

| Правило | Где применяется |
|---|---|
| `return format!(...)` в tail-выражении функции, возвращающей `String` | `format_file_size`, `format_read_text_result`, `format_read_binary_result`, `format_metadata_list` |
| `return Ok(...)` в tail-выражении функции, возвращающей `Result<_, JsValue>` | `run`, `build_*_section`, `build_section`, `make_result`, `create_and_download_blob` |
| `format!` не передаётся напрямую в `set_text_content` | везде через `let message = ...; Some(&message)` |
| Сложное форматирование — в отдельной синхронной функции, не в `async`-блоке | `format_read_text_result`, `format_read_binary_result`, `format_hex_preview` |
| `RequestInit::set_method` / `set_body` вместо deprecated `method` / `body` | `file_upload.rs` |
| `let opts = RequestInit::new();` без `mut` | `file_upload.rs` |
| Временные `String` не пересекают границы выражений | `let file_name: String = file.name();` перед `get_extension` |
| `<input accept="...">` для фильтрации форматов в текстовых секциях | секция 3 в `app.rs` — `.txt,text/plain` |

---

## 18.9. Итоги главы

Мы изучили File API в контексте Rust + WebAssembly:

- **File** — метаданные (имя, размер, MIME, дата).
- **Blob** — бинарные данные; `File` является наследником `Blob`.
- **FileList** — список выбранных файлов; `DataTransfer::files()` возвращает `Option<FileList>`.
- **FileReader** — асинхронное чтение (ArrayBuffer, текст, Data URL).
- **FileReaderSync** — синхронное чтение в Web Workers.
- **MIME-типы** и определение по расширению.
- **ArrayBuffer / Uint8Array** — работа с бинарными данными.
- **Drag & Drop** — получение файлов через `DataTransfer`.
- **Практический проект** — загрузка файла на Rust-сервер через `FormData` и `fetch`.

Все модули автономны и не ломают существующие решения. Демо-пример в `app.rs` демонстрирует пять секций работы с File API прямо в браузере. В следующей главе мы применим эти знания для конвертации изображений.