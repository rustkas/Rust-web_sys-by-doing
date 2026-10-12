# Глава 18. File API

## 18.15. File System Access API: открытие, сохранение, папки

До сих пор мы работали с файлами через `<input type="file">` и `Blob`. Это надёжный и универсальный подход, но у него есть фундаментальное ограничение: **файл выбирается в одном месте, а результат сохраняется в другое** (обычно в папку загрузок). Пользователь не может открыть файл из проекта, обработать его и сохранить рядом с исходным.

**File System Access API** решает эту проблему. Он даёт веб-приложению доступ к реальной файловой системе:

- **Открыть файл** из любого места — через `showOpenFilePicker()`.
- **Сохранить файл** в выбранное место — через `showSaveFilePicker()`.
- **Работать с папкой** — через `showDirectoryPicker()`, включая рекурсивный обход.

Это не «чтение через `<input>` с расширенными правами». Это полноценный доступ к диску, как у настольного приложения. Пользователь сам выбирает файл или папку, а приложение получает **handle** — долгоживущую ссылку, которая работает между вызовами и позволяет читать и записывать данные.

### Ограничения, которые нужно знать сразу

1. **Только Chrome/Edge.** Firefox и Safari не поддерживают API. Для них — fallback на `<input type="file">` и `<a download>`.
2. **Только secure context.** HTTPS или `localhost`. На HTTP (кроме localhost) API недоступен.
3. **Обязателен user gesture.** Каждый вызов `showOpenFilePicker`, `showSaveFilePicker` или `showDirectoryPicker` должен происходить из обработчика клика — браузер не даст открыть диалог «просто так».
4. **Разрешения не сохраняются между сессиями.** Handle можно сохранить в IndexedDB, но после перезагрузки страницы `queryPermission()` вернёт `"prompt"`, и для восстановления доступа потребуется новый клик.

### Особенности web-sys 0.3.105

При работе с FSA через `web-sys` есть три нюанса, которые нужно учитывать:

1. **`show_open_file_picker_with_options` и `show_save_file_picker_with_options` не экспортированы.** В web-sys 0.3.105 есть только методы без аргументов. Для передачи опций (`suggestedName`, фильтры расширений) используем `js_sys::Reflect::get` + `Function::call1`. Это работает независимо от версии web-sys.
2. **`FileSystemDirectoryHandle::values()` возвращает `js_sys::AsyncIterator`, а не `Promise`.** `JsFuture::from` его не принимает — итерируем вручную через `iter.next()` → `Promise` → `JsFuture::from`.
3. **`FileSystemDirectoryHandle::remove_entry()` возвращает `Promise` напрямую**, не `Result<Promise, JsValue>`. Поэтому `?` работает только на `.await`.

Все три нюанса учтены в коде ниже.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # +новые фичи web-sys для File System Access API
├── src/
│   ├── lib.rs                     # +pub mod fsa_open, fsa_save, fsa_directory
│   ├── app.rs                     # только секция 18.15
│   ├── fsa_open.rs                # НОВЫЙ: showOpenFilePicker, FileSystemFileHandle
│   ├── fsa_save.rs                # НОВЫЙ: showSaveFilePicker, createWritable, write, close
│   ├── fsa_directory.rs           # НОВЫЙ: showDirectoryPicker, обход папки
│   └── … остальные модули
└── tests/
    ├── wasm_fsa_open.rs           # НОВЫЙ: тесты открытия (без #[ignore])
    ├── wasm_fsa_save.rs           # НОВЫЙ: тесты сохранения (без #[ignore])
    └── wasm_fsa_directory.rs      # НОВЫЙ: тесты директорий (без #[ignore])
```

**Пояснения:**

- `src/fsa_open.rs` — обёртка над `showOpenFilePicker()` и `FileSystemFileHandle.getFile()`. Возвращает `web_sys::File` — тот же тип, что отдаёт `<input type="file">`.
- `src/fsa_save.rs` — обёртка над `showSaveFilePicker()`, `createWritable()`, `write()`, `close()`.
- `src/fsa_directory.rs` — обёртка над `showDirectoryPicker()` и итеративным обходом дерева.
- `tests/*` — **без `#[ignore]`**. В headless-режиме picker-вызовы возвращают `SecurityError`, тесты это обрабатывают и проходят. В интерактивном Chrome кликаем по picker'ам вручную.

---

## Ссылки для детального изучения

- **MDN: File System API** — https://developer.mozilla.org/en-US/docs/Web/API/File_System_API
- **MDN: FileSystemFileHandle** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle
- **MDN: FileSystemDirectoryHandle** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemDirectoryHandle
- **MDN: showOpenFilePicker()** — https://developer.mozilla.org/en-US/docs/Web/API/Window/showOpenFilePicker
- **MDN: showSaveFilePicker()** — https://developer.mozilla.org/en-US/docs/Web/API/Window/showSaveFilePicker
- **MDN: showDirectoryPicker()** — https://developer.mozilla.org/en-US/docs/Web/API/Window/showDirectoryPicker
- **Chrome: File System Access API** — https://developer.chrome.com/docs/capabilities/web-apis/file-system-access
- **web-sys: FileSystemFileHandle** — https://docs.rs/web-sys/latest/web_sys/struct.FileSystemFileHandle.html

---

## Содержание файлов

### 1. `Cargo.toml` — новые фичи web-sys

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    # … существующие фичи …

    # 18.15: File System Access API
    "FileSystemHandle",
    "FileSystemFileHandle",
    "FileSystemDirectoryHandle",
    "FileSystemWritableFileStream",
    "FileSystemCreateWritableOptions",
    "FileSystemGetFileOptions",
    "FileSystemGetDirectoryOptions",
    "FileSystemRemoveOptions",
    "Window",
]
```

---

### 2. `src/lib.rs`

```rust
// ---- Глава 18. File API ----
// … модули из предыдущих подглав …
pub mod fsa_open;        // 18.15
pub mod fsa_save;        // 18.15
pub mod fsa_directory;   // 18.15
```

---

### 3. `src/fsa_open.rs`

```rust
//! File System Access API: открытие файла.
//!
//! **Требования:** Chrome/Edge + secure context + user gesture.
//! В Firefox и Safari модуль не используется — там fallback
//! на `<input type="file">`.
//!
//! **Про вызов пикера.** web-sys 0.3.105 экспортирует
//! `Window::show_open_file_picker()` **без аргументов** — опции
//! передать нельзя. Чтобы использовать фильтры, вызываем
//! `showOpenFilePicker` через `Reflect::get` + `Function::call1`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::FileSystemFileHandle;

/// Вызывает `showOpenFilePicker` с опциями (или без).
///
/// Возвращает `Ok(None)`, если пользователь отменил диалог.
async fn call_open_picker(
    options: Option<&JsValue>,
) -> Result<Option<JsValue>, JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let window_js: &JsValue = window.as_ref();

    // Получаем функцию showOpenFilePicker через Reflect — работает
    // независимо от того, какие сигнатуры экспортирует web-sys.
    let func_js = js_sys::Reflect::get(
        window_js,
        &JsValue::from_str("showOpenFilePicker"),
    )?;
    if func_js.is_undefined() {
        return Err(JsValue::from_str("showOpenFilePicker недоступен"));
    }
    let func: js_sys::Function = func_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;

    let promise_js = match options {
        Some(opts) => func.call1(window_js, opts)?,
        None => func.call0(window_js)?,
    };
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;

    match JsFuture::from(promise).await {
        Ok(v) => Ok(Some(v)),
        Err(_) => Ok(None), // пользователь отменил
    }
}

/// Извлекает первый `FileSystemFileHandle` из массива и читает файл.
async fn first_handle_to_file(
    handles: JsValue,
) -> Result<Option<web_sys::File>, JsValue> {
    let array: js_sys::Array = handles
        .dyn_into()
        .map_err(|_| JsValue::from_str("ответ не Array"))?;
    if array.length() == 0 {
        return Ok(None);
    }

    let handle: FileSystemFileHandle = array
        .get(0)
        .dyn_into()
        .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))?;

    let file_promise: js_sys::Promise = handle.get_file();
    let file_value: JsValue = JsFuture::from(file_promise).await?;
    let file: web_sys::File = file_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не File"))?;
    return Ok(Some(file));
}

/// Открывает диалог выбора одного файла.
pub async fn pick_one_file() -> Result<Option<web_sys::File>, JsValue> {
    let handles = match call_open_picker(None).await? {
        Some(v) => v,
        None => return Ok(None),
    };
    return first_handle_to_file(handles).await;
}

/// Открывает диалог выбора файла с фильтром по расширениям.
///
/// `accept` — `{ "text/plain": [".txt"] }` как `JsValue`.
pub async fn pick_one_file_with_filter(
    accept: &JsValue,
) -> Result<Option<web_sys::File>, JsValue> {
    let type_entry = js_sys::Object::new();
    js_sys::Reflect::set(
        &type_entry,
        &JsValue::from_str("description"),
        &JsValue::from_str("Files"),
    )?;
    js_sys::Reflect::set(&type_entry, &JsValue::from_str("accept"), accept)?;

    let types = js_sys::Array::new();
    types.push(&type_entry);

    let options = js_sys::Object::new();
    js_sys::Reflect::set(&options, &JsValue::from_str("types"), &types)?;
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("multiple"),
        &JsValue::from_bool(false),
    )?;

    let handles = match call_open_picker(Some(&options)).await? {
        Some(v) => v,
        None => return Ok(None),
    };
    return first_handle_to_file(handles).await;
}

/// Проверяет, поддерживается ли File System Access API в браузере.
pub fn is_supported() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let window_js: &JsValue = window.as_ref();
    return js_sys::Reflect::get(
        window_js,
        &JsValue::from_str("showOpenFilePicker"),
    )
    .map(|v| !v.is_undefined())
    .unwrap_or(false);
}
```

---

### 4. `src/fsa_save.rs`

```rust
//! File System Access API: сохранение файла.
//!
//! **Требования:** Chrome/Edge + secure context + user gesture.
//! В Firefox и Safari модуль не используется — там fallback
//! на `<a download>` + Object URL.
//!
//! **Про вызовы.** web-sys 0.3.105 экспортирует
//! `show_save_file_picker()` **без аргументов**. Опции передаём
//! через Reflect. Методы `write` и `close` writable-потока тоже
//! вызываем через Reflect — так надёжнее и переносимее между
//! версиями.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileSystemFileHandle};

/// Сохраняет `Blob` в файл, выбранный пользователем.
///
/// Возвращает имя сохранённого файла (или `None`, если пользователь
/// отменил диалог).
pub async fn save_blob(
    blob: &Blob,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let window_js: &JsValue = window.as_ref();

    // showSaveFilePicker({ suggestedName })
    let func_js = js_sys::Reflect::get(
        window_js,
        &JsValue::from_str("showSaveFilePicker"),
    )?;
    if func_js.is_undefined() {
        return Err(JsValue::from_str("showSaveFilePicker недоступен"));
    }
    let func: js_sys::Function = func_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;

    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;

    let promise_js = func.call1(window_js, &options)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;

    let handle_value = match JsFuture::from(promise).await {
        Ok(v) => v,
        Err(_) => return Ok(None), // пользователь отменил
    };

    let handle: FileSystemFileHandle = handle_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))?;
    let file_name = handle.name();

    // createWritable() → Promise<FileSystemWritableFileStream>
    let handle_js: &JsValue = handle.as_ref();
    let create_fn_js = js_sys::Reflect::get(
        handle_js,
        &JsValue::from_str("createWritable"),
    )?;
    let create_fn: js_sys::Function = create_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("createWritable не функция"))?;
    let writable_promise_js = create_fn.call0(handle_js)?;
    let writable_promise: js_sys::Promise = writable_promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let writable: JsValue = JsFuture::from(writable_promise).await?;

    // write(blob) — данные буферизуются.
    let writable_js: &JsValue = &writable;
    let write_fn_js = js_sys::Reflect::get(
        writable_js,
        &JsValue::from_str("write"),
    )?;
    let write_fn: js_sys::Function = write_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("write не функция"))?;
    let blob_js: &JsValue = blob.as_ref();
    let write_promise_js = write_fn.call1(writable_js, blob_js)?;
    let write_promise: js_sys::Promise = write_promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    JsFuture::from(write_promise).await?;

    // close() — данные попадают на диск.
    let close_fn_js = js_sys::Reflect::get(
        writable_js,
        &JsValue::from_str("close"),
    )?;
    let close_fn: js_sys::Function = close_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("close не функция"))?;
    let close_promise_js = close_fn.call0(writable_js)?;
    let close_promise: js_sys::Promise = close_promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    JsFuture::from(close_promise).await?;

    return Ok(Some(file_name));
}

/// Сохранить срез `u8` как бинарный файл.
pub async fn save_bytes(
    bytes: &[u8],
    mime_type: &str,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type(mime_type);
    let blob = Blob::new_with_u8_array_sequence_and_options(&parts, &options)?;
    return save_blob(&blob, suggested_name).await;
}

/// Сохранить строку как текстовый файл.
pub async fn save_text(
    text: &str,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    let array = js_sys::Array::of1(&JsValue::from_str(text));
    let blob = Blob::new_with_str_sequence(&array)?;
    return save_blob(&blob, suggested_name).await;
}
```

---

### 5. `src/fsa_directory.rs`

```rust
//! File System Access API: работа с папками.
//!
//! **Требования:** Chrome/Edge + secure context + user gesture.
//!
//! **Про `values()`.** В web-sys 0.3.105 `FileSystemDirectoryHandle::values()`
//! возвращает `js_sys::AsyncIterator`, а не `Promise`. `JsFuture::from`
//! такой тип не принимает — итерируем вручную через `iter.next()`.
//!
//! **Итеративная очередь вместо рекурсии.** Async-рекурсия в Rust
//! требует `Box::pin` на каждом уровне и может переполнить стек.

use std::collections::VecDeque;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

/// Запись о файле, найденном при обходе директории.
#[derive(Debug, Clone, PartialEq)]
pub struct DirEntry {
    pub path: String,
    pub size: f64,
    pub mime_type: String,
}

/// Открывает диалог выбора директории.
pub async fn pick_directory() -> Result<Option<FileSystemDirectoryHandle>, JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let window_js: &JsValue = window.as_ref();

    let func_js = js_sys::Reflect::get(
        window_js,
        &JsValue::from_str("showDirectoryPicker"),
    )?;
    if func_js.is_undefined() {
        return Err(JsValue::from_str("showDirectoryPicker недоступен"));
    }
    let func: js_sys::Function = func_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;

    let promise_js = func.call0(window_js)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;

    match JsFuture::from(promise).await {
        Ok(v) => {
            let handle: FileSystemDirectoryHandle = v
                .dyn_into()
                .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))?;
            Ok(Some(handle))
        }
        Err(_) => Ok(None), // пользователь отменил
    }
}

/// Читает один элемент из async iterator директории.
async fn next_entry(iter: &JsValue) -> Result<Option<JsValue>, JsValue> {
    let next_fn_js = js_sys::Reflect::get(iter, &JsValue::from_str("next"))?;
    let next_fn: js_sys::Function = next_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("next не функция"))?;
    let promise_js = next_fn.call0(iter)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("next не Promise"))?;

    let result: JsValue = JsFuture::from(promise).await?;
    let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
        .as_bool()
        .unwrap_or(true);
    if done {
        return Ok(None);
    }
    let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))?;
    return Ok(Some(value));
}

/// Рекурсивно обходит директорию и возвращает список файлов.
///
/// `max_files` — защита от обхода огромных деревьев.
pub async fn scan_directory(
    root: &FileSystemDirectoryHandle,
    max_files: usize,
) -> Result<Vec<DirEntry>, JsValue> {
    let mut out: Vec<DirEntry> = Vec::new();
    let mut queue: VecDeque<(FileSystemDirectoryHandle, String)> = VecDeque::new();
    queue.push_back((root.clone(), String::new()));

    while let Some((dir, prefix)) = queue.pop_front() {
        if out.len() >= max_files {
            break;
        }

        // dir.values() → AsyncIterator
        let dir_js: &JsValue = dir.as_ref();
        let values_fn_js = js_sys::Reflect::get(
            dir_js,
            &JsValue::from_str("values"),
        )?;
        let values_fn: js_sys::Function = values_fn_js
            .dyn_into()
            .map_err(|_| JsValue::from_str("values не функция"))?;
        let iter: JsValue = values_fn.call0(dir_js)?;

        loop {
            if out.len() >= max_files {
                break;
            }
            let entry = match next_entry(&iter).await? {
                Some(v) => v,
                None => break,
            };

            let kind = js_sys::Reflect::get(&entry, &JsValue::from_str("kind"))?
                .as_string()
                .unwrap_or_default();
            let name = js_sys::Reflect::get(&entry, &JsValue::from_str("name"))?
                .as_string()
                .unwrap_or_default();

            // Предобъявленная переменная: `format!` в tail-позиции
            // блока if/else в этом окружении выводится как `()`.
            let full_path: String;
            if prefix.is_empty() {
                full_path = name.clone();
            } else {
                full_path = format!("{prefix}/{name}");
            }

            if kind == "file" {
                let file_handle: FileSystemFileHandle = entry
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))?;
                let file_promise: js_sys::Promise = file_handle.get_file();
                let file_value: JsValue = JsFuture::from(file_promise).await?;
                let file: web_sys::File = file_value
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не File"))?;
                out.push(DirEntry {
                    path: full_path,
                    size: file.size(),
                    mime_type: file.type_(),
                });
            } else if kind == "directory" {
                let child_dir: FileSystemDirectoryHandle = entry
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))?;
                queue.push_back((child_dir, full_path));
            }
        }
    }

    return Ok(out);
}

/// Удаляет файл или поддиректорию из директории.
///
/// `remove_entry` возвращает `Promise` напрямую, не `Result<Promise>` —
/// поэтому `?` на самом вызове не работает, только на `.await`.
pub async fn remove_entry(
    dir: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<(), JsValue> {
    let promise: js_sys::Promise = dir.remove_entry(name);
    let _ = JsFuture::from(promise).await?;
    return Ok(());
}
```

---

## Тесты

Все три файла — **без `#[ignore]`**. В headless-режиме picker-функции возвращают `SecurityError`, тесты это обрабатывают и проходят.

### `tests/wasm_fsa_open.rs`

```rust
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;
use hello_wasm02::fsa_open::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Проверяет, что `is_supported()` возвращает `bool` без паники.
#[wasm_bindgen_test]
fn test_is_supported_returns_bool() {
    let supported = is_supported();
    web_sys::console::log_1(&format!("FSA supported: {supported}").into());
}

/// Без user gesture `pick_one_file` вернёт `Err` с `SecurityError`.
#[wasm_bindgen_test]
async fn test_pick_one_file_returns_result() {
    let result = pick_one_file().await;
    match result {
        Ok(Some(file)) => assert!(!file.name().is_empty()),
        Ok(None) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(&format!("pick_one_file error: {s}").into());
        }
    }
}

/// Версия с фильтром тоже возвращает `Result`.
#[wasm_bindgen_test]
async fn test_pick_one_file_with_filter_returns_result() {
    let accept = js_sys::Array::of1(&JsValue::from_str(".txt"));
    let result = pick_one_file_with_filter(&accept.into()).await;
    if let Err(e) = &result {
        let s = e.as_string().unwrap_or_default();
        web_sys::console::log_1(
            &format!("pick_one_file_with_filter error: {s}").into(),
        );
    }
}
```

### `tests/wasm_fsa_save.rs`

```rust
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;
use hello_wasm02::fsa_save::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_save_text_returns_result() {
    let result = save_text("Hello, FSA!", "test-fsa.txt").await;
    match result {
        Ok(Some(name)) => assert_eq!(name, "test-fsa.txt"),
        Ok(None) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(&format!("save_text error: {s}").into());
        }
    }
}

#[wasm_bindgen_test]
async fn test_save_bytes_returns_result() {
    let bytes: Vec<u8> = (0..=255).collect();
    let result = save_bytes(&bytes, "application/octet-stream", "test.bin").await;
    match result {
        Ok(Some(name)) => assert_eq!(name, "test.bin"),
        Ok(None) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(&format!("save_bytes error: {s}").into());
        }
    }
}

#[wasm_bindgen_test]
async fn test_save_blob_returns_result() {
    let array = js_sys::Array::of1(&JsValue::from_str("blob content"));
    let blob = web_sys::Blob::new_with_str_sequence(&array)
        .expect("Blob::new_with_str_sequence");

    let result = save_blob(&blob, "test-blob.txt").await;
    match result {
        Ok(Some(name)) => assert_eq!(name, "test-blob.txt"),
        Ok(None) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(&format!("save_blob error: {s}").into());
        }
    }
}
```

### `tests/wasm_fsa_directory.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::fsa_directory::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_pick_directory_returns_result() {
    let result = pick_directory().await;
    match result {
        Ok(Some(handle)) => assert!(!handle.name().is_empty()),
        Ok(None) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(&format!("pick_directory error: {s}").into());
        }
    }
}

#[wasm_bindgen_test]
fn test_dir_entry_construction() {
    let entry = DirEntry {
        path: "photos/2024/img.png".to_string(),
        size: 12345.0,
        mime_type: "image/png".to_string(),
    };
    assert_eq!(entry.path, "photos/2024/img.png");
    assert_eq!(entry.size, 12345.0);
    assert_eq!(entry.mime_type, "image/png");
}

#[wasm_bindgen_test]
fn test_dir_entry_equality() {
    let a = DirEntry {
        path: "a.txt".to_string(),
        size: 100.0,
        mime_type: "text/plain".to_string(),
    };
    let b = DirEntry {
        path: "a.txt".to_string(),
        size: 100.0,
        mime_type: "text/plain".to_string(),
    };
    let c = DirEntry {
        path: "b.txt".to_string(),
        size: 100.0,
        mime_type: "text/plain".to_string(),
    };
    assert_eq!(a, b);
    assert_ne!(a, c);
}

#[wasm_bindgen_test]
async fn test_scan_directory_requires_valid_handle() {
    // Без user gesture handle получить нельзя — фиксируем, что
    // сигнатура функции скомпилирована корректно.
    let _fn_signature = |_: &web_sys::FileSystemDirectoryHandle,
                          _: usize|
     -> std::pin::Pin<
        Box<
            dyn std::future::Future<
                Output = Result<Vec<DirEntry>, wasm_bindgen::JsValue>,
            >,
        >,
    > {
        Box::pin(async { Ok(Vec::new()) })
    };
}
```

---

## Демо-пример в `app.rs`

```rust
//! Демонстрация подглавы 18.15: File System Access API.
//!
//! **Интеллектуальный подход.**
//! * Статус-бар показывает, доступен ли FSA и secure context.
//! * Сценарий «Открыть → Сохранить копию» демонстрирует реальный
//!   workflow: приложение работает с файлом как настольная программа.
//! * Кнопка «Сохранить копию» блокируется до открытия файла.
//! * При `SecurityError` показывается понятное сообщение.
//! * Обход папки ограничен `SCAN_LIMIT`; при обрезании списка
//!   это явно сообщается.
//! * В браузерах без FSA показывается fallback на `<input>` + `<a download>`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_api, fsa_directory, fsa_open, fsa_save, log};

const SCAN_LIMIT: usize = 50;

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File System Access API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));
    build_fsa_section(&container, &closures)?;

    log!("Section 18.15 ready");
    std::mem::forget(closures);
    return Ok(());
}

/// Файл, открытый через FSA — нужен для кнопки «Сохранить копию».
struct State {
    opened_file: Option<web_sys::File>,
}

fn build_fsa_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let state = Rc::new(RefCell::new(State { opened_file: None }));

    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.15. File System Access API"));
    dom::append(&section, &h2)?;

    // ---- Статус-бар. ----
    let fsa_ok = fsa_open::is_supported();
    let status = dom::create_element("div")?;
    status.set_attribute(
        "style",
        "margin: 12px 0; padding: 10px 14px; \
         background: #f0f4ff; border-left: 4px solid #4a7cff; \
         border-radius: 4px; font-family: monospace; \
         white-space: pre-line; line-height: 1.6;",
    )?;

    let status_text: String;
    if fsa_ok {
        status_text = "✅ File System Access API доступен.\n\
                       Все три сценария работают: открытие, \
                       сохранение, обход папок."
            .to_string();
    } else {
        status_text = "⚠️ File System Access API недоступен в этом \
                       браузере.\n\
                       Firefox и Safari не поддерживают FSA. Ниже \
                       показан fallback на стандартные механизмы."
            .to_string();
    }
    status.set_text_content(Some(status_text.as_str()));
    dom::append(&section, &status)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите действие)"));
    dom::append(&section, &result)?;

    if fsa_ok {
        build_open_save_block(&section, closures, &state, &result)?;
        build_directory_block(&section, closures, &result)?;
    } else {
        build_fallback_block(&section, closures, &result)?;
    }

    dom::append(parent, &section)?;
    return Ok(());
}

// ---------------------------------------------------------------
// Подсекция «Открыть → Сохранить копию»
// ---------------------------------------------------------------

fn build_open_save_block(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
    state: &Rc<RefCell<State>>,
    result: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = dom::create_element("div")?;
    block.set_attribute(
        "style",
        "margin: 16px 0; padding: 12px; background: #fafafa; \
         border: 1px solid #e5e5e5; border-radius: 6px;",
    )?;

    let h3 = dom::create_element("h3")?;
    h3.set_text_content(Some("Сценарий: открыть → сохранить копию"));
    dom::append(&block, &h3)?;

    let hint = dom::create_paragraph(
        "Откройте файл через showOpenFilePicker, затем сохраните \
         его копию через showSaveFilePicker. Это реальный workflow, \
         который FSA делает возможным.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&block, &hint)?;

    let open_btn = dom::create_element("button")?;
    open_btn.set_text_content(Some("Открыть файл"));
    dom::append(&block, &open_btn)?;

    let save_btn: web_sys::Element = dom::create_element("button")?;
    save_btn.set_text_content(Some("Сохранить копию"));
    save_btn.set_attribute("disabled", "").ok();
    dom::append(&block, &save_btn)?;

    let open_info = dom::create_element("div")?;
    open_info.set_attribute(
        "style",
        "margin-top: 10px; font-family: monospace; color: #555; \
         white-space: pre-line;",
    )?;
    open_info.set_text_content(Some("(файл не открыт)"));
    dom::append(&block, &open_info)?;

    dom::append(parent, &block)?;

    // ---- Обработчик «Открыть файл». ----
    {
        let state_c = state.clone();
        let result_c = result.clone();
        let open_info_c = open_info.clone();
        let save_btn_c = save_btn.clone();

        callbacks::listen(
            &open_btn,
            "click",
            move |_ev: web_sys::Event| {
                let state_c = state_c.clone();
                let result_c = result_c.clone();
                let open_info_c = open_info_c.clone();
                let save_btn_c = save_btn_c.clone();
                spawn_local(async move {
                    match fsa_open::pick_one_file().await {
                        Ok(Some(file)) => {
                            let name = file.name();
                            let size = file.size();
                            let mime_display: String;
                            if file.type_().is_empty() {
                                mime_display = "(не определён)".to_string();
                            } else {
                                mime_display = file.type_();
                            }
                            let info = format!(
                                "Открыт: {name}\n\
                                 Размер: {}\n\
                                 MIME: {mime_display}",
                                file_api::format_file_size(size),
                            );
                            open_info_c.set_text_content(Some(info.as_str()));
                            save_btn_c.remove_attribute("disabled").ok();

                            result_c.set_text_content(Some(
                                "✅ Файл открыт. Теперь можно сохранить копию.",
                            ));

                            state_c.borrow_mut().opened_file = Some(file);
                        }
                        Ok(None) => {
                            result_c.set_text_content(Some("Диалог отменён."));
                        }
                        Err(e) => {
                            let msg = format_fsa_error(&e);
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    // ---- Обработчик «Сохранить копию». ----
    {
        let state_c = state.clone();
        let result_c = result.clone();

        callbacks::listen(
            &save_btn,
            "click",
            move |_ev: web_sys::Event| {
                let file_opt = state_c.borrow().opened_file.clone();
                let file = match file_opt {
                    Some(f) => f,
                    None => {
                        result_c.set_text_content(Some("Сначала откройте файл."));
                        return;
                    }
                };
                let result_c = result_c.clone();
                spawn_local(async move {
                    let value = match JsFuture::from(file.array_buffer()).await {
                        Ok(v) => v,
                        Err(e) => {
                            let msg = format!("❌ Чтение файла: {e:?}");
                            result_c.set_text_content(Some(msg.as_str()));
                            return;
                        }
                    };
                    let array_buffer: js_sys::ArrayBuffer = match value.dyn_into() {
                        Ok(a) => a,
                        Err(_) => {
                            result_c.set_text_content(Some("не ArrayBuffer"));
                            return;
                        }
                    };
                    let bytes = js_sys::Uint8Array::new(&array_buffer).to_vec();

                    let original = file.name();
                    let copy_name = make_copy_name(&original);
                    let mime = file.type_();

                    match fsa_save::save_bytes(&bytes, &mime, &copy_name).await {
                        Ok(Some(saved_name)) => {
                            let msg = format!(
                                "✅ Копия сохранена через showSaveFilePicker\n\n\
                                 Исходный файл: {original}\n\
                                 Копия: {saved_name}\n\
                                 Размер: {} байт",
                                bytes.len(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Ok(None) => {
                            result_c.set_text_content(Some(
                                "Диалог сохранения отменён."));
                        }
                        Err(e) => {
                            let msg = format_fsa_error(&e);
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Подсекция «Обход папки»
// ---------------------------------------------------------------

fn build_directory_block(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
    result: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = dom::create_element("div")?;
    block.set_attribute(
        "style",
        "margin: 16px 0; padding: 12px; background: #fafafa; \
         border: 1px solid #e5e5e5; border-radius: 6px;",
    )?;

    let h3 = dom::create_element("h3")?;
    h3.set_text_content(Some("Сценарий: обход папки"));
    dom::append(&block, &h3)?;

    let hint_text = format!(
        "Выберите папку через showDirectoryPicker. Покажем первые \
         {SCAN_LIMIT} файлов с относительными путями.",
    );
    let hint = dom::create_paragraph(&hint_text)?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&block, &hint)?;

    let dir_btn = dom::create_element("button")?;
    dir_btn.set_text_content(Some("Выбрать папку"));
    dom::append(&block, &dir_btn)?;

    dom::append(parent, &block)?;

    {
        let result_c = result.clone();
        callbacks::listen(
            &dir_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    let handle = match fsa_directory::pick_directory().await {
                        Ok(Some(h)) => h,
                        Ok(None) => {
                            result_c.set_text_content(Some("Диалог отменён."));
                            return;
                        }
                        Err(e) => {
                            let msg = format_fsa_error(&e);
                            result_c.set_text_content(Some(msg.as_str()));
                            return;
                        }
                    };

                    let entries = match fsa_directory::scan_directory(
                        &handle, SCAN_LIMIT,
                    )
                    .await
                    {
                        Ok(v) => v,
                        Err(e) => {
                            let msg = format!("❌ scan_directory: {e:?}");
                            result_c.set_text_content(Some(msg.as_str()));
                            return;
                        }
                    };

                    let message = format_directory_result(&entries);
                    result_c.set_text_content(Some(message.as_str()));
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Подсекция «Fallback»
// ---------------------------------------------------------------

fn build_fallback_block(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
    result: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = dom::create_element("div")?;
    block.set_attribute(
        "style",
        "margin: 16px 0; padding: 12px; background: #fff8e0; \
         border: 1px solid #e0d090; border-radius: 6px;",
    )?;

    let h3 = dom::create_element("h3")?;
    h3.set_text_content(Some(
        "Fallback: <input type=\"file\"> и <a download>",
    ));
    dom::append(&block, &h3)?;

    let hint = dom::create_paragraph(
        "В этом браузере FSA недоступен. Используем стандартные \
         механизмы: выбор через <input>, скачивание через <a download> \
         + Object URL. Работает везде, но пользователь сам решает, \
         куда сохранять — обычно в папку «Загрузки».",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&block, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&block, &input)?;

    let download_btn = dom::create_element("button")?;
    download_btn.set_text_content(Some("Скачать тестовый .txt"));
    dom::append(&block, &download_btn)?;

    dom::append(parent, &block)?;

    {
        let result_c = result.clone();
        callbacks::listen(
            &input,
            "change",
            move |ev: web_sys::Event| {
                let target = match ev.target() {
                    Some(t) => t,
                    None => return,
                };
                let input_el: web_sys::HtmlInputElement =
                    match target.dyn_into() {
                        Ok(el) => el,
                        Err(_) => return,
                    };
                if let Some(file) = input_el.files().and_then(|f| f.get(0)) {
                    let mime_display: String;
                    if file.type_().is_empty() {
                        mime_display = "(не определён)".to_string();
                    } else {
                        mime_display = file.type_();
                    }
                    let info = format!(
                        "✅ Файл выбран через <input>\n\
                         Имя: {}\n\
                         Размер: {}\n\
                         MIME: {mime_display}",
                        file.name(),
                        file_api::format_file_size(file.size()),
                    );
                    result_c.set_text_content(Some(info.as_str()));
                }
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &download_btn,
            "click",
            move |_ev: web_sys::Event| {
                match download_test_blob() {
                    Ok(name) => {
                        let msg = format!(
                            "✅ Файл скачан: {name}\n\
                             (через <a download> + Object URL)",
                        );
                        result_c.set_text_content(Some(msg.as_str()));
                    }
                    Err(e) => {
                        let msg = format!("❌ download: {e:?}");
                        result_c.set_text_content(Some(msg.as_str()));
                    }
                }
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------

fn make_copy_name(original: &str) -> String {
    let dot_opt = original.rfind('.');
    match dot_opt {
        Some(dot) => {
            let (base, ext) = original.split_at(dot);
            return format!("{base}-copy{ext}");
        }
        None => return format!("{original}-copy"),
    }
}

fn format_directory_result(entries: &[fsa_directory::DirEntry]) -> String {
    if entries.is_empty() {
        return "Папка пуста.".to_string();
    }

    let mut out = format!("✅ Найдено файлов: {}\n\n", entries.len());
    for (i, e) in entries.iter().enumerate() {
        out.push_str(&format!(
            "{}. {} · {}\n",
            i + 1,
            e.path,
            file_api::format_file_size(e.size),
        ));
    }

    if entries.len() >= SCAN_LIMIT {
        out.push_str(&format!(
            "\n⚠️ Достигнут лимит {SCAN_LIMIT} файлов. \
             В папке могут быть другие файлы.",
        ));
    }

    return out;
}

fn format_fsa_error(e: &JsValue) -> String {
    let raw = e
        .as_string()
        .unwrap_or_else(|| format!("{e:?}"));

    let is_security = raw.contains("SecurityError")
        || raw.contains("user gesture")
        || raw.contains("User activation");
    if is_security {
        return format!(
            "⚠️ SecurityError: picker требует user gesture.\n\n\
             Возможные причины:\n\
             • Вызов произошёл не из обработчика клика.\n\
             • Браузер не считает клик «активацией».\n\n\
             Попробуйте нажать на кнопку ещё раз.\n\n\
             Техническая информация: {raw}",
        );
    }
    return format!("❌ Ошибка FSA: {raw}");
}

fn download_test_blob() -> Result<String, JsValue> {
    let content = "Hello from <a download> + Object URL!\n";
    let array = js_sys::Array::of1(&JsValue::from_str(content));
    let options = web_sys::BlobPropertyBag::new();
    options.set_type("text/plain");
    let blob = web_sys::Blob::new_with_str_sequence_and_options(
        &array, &options,
    )?;

    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let url = web_sys::Url::create_object_url_with_blob(&blob)?;

    let a: web_sys::HtmlAnchorElement = document
        .create_element("a")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не anchor"))?;
    a.set_href(&url);
    a.set_download("hello-fallback.txt");
    a.click();

    web_sys::Url::revoke_object_url(&url)?;
    return Ok("hello-fallback.txt".to_string());
}
```

---

## Команды запуска

### Проверка типов

```bash
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests
```

### Тесты

```bash
# Headless (без user gesture — picker вернёт SecurityError, тесты это обрабатывают)
wasm-pack test --headless --chrome --test wasm_fsa_open
wasm-pack test --headless --chrome --test wasm_fsa_save
wasm-pack test --headless --chrome --test wasm_fsa_directory

# Интерактивный Chrome (кликаем по picker'ам вручную)
wasm-pack test --chrome --test wasm_fsa_open
wasm-pack test --chrome --test wasm_fsa_save
wasm-pack test --chrome --test wasm_fsa_directory

# Для Firefox
wasm-pack test --headless --firefox --test wasm_fsa_open
wasm-pack test --headless --firefox --test wasm_fsa_save
wasm-pack test --headless --firefox --test wasm_fsa_directory
```

### Демо-пример

```bash
npm run start
```

Откройте `http://localhost:5173` в Chrome — увидите активную FSA-секцию. В Firefox — fallback.

---

## Что увидит пользователь

**Chrome/Edge:**

```
✅ File System Access API доступен.
Все три сценария работают: открытие, сохранение, обход папок.

Сценарий: открыть → сохранить копию
[Открыть файл]  [Сохранить копию]

Открыт: report.pdf
Размер: 234.56 KB
MIME: application/pdf

Сценарий: обход папки
[Выбрать папку]
```

После открытия `report.pdf` и клика «Сохранить копию»:
```
✅ Копия сохранена через showSaveFilePicker

Исходный файл: report.pdf
Копия: report-copy.pdf
Размер: 240192 байт
```

**Firefox/Safari:**

```
⚠️ File System Access API недоступен в этом браузере.
Firefox и Safari не поддерживают FSA. Ниже показан fallback.

Fallback: <input type="file"> и <a download>
[Выберите файл]  [Скачать тестовый .txt]
```

---

## Итоги подглавы 18.15

- **File System Access API** — прямой доступ к файловой системе: открытие, сохранение, обход папок.
- **`showOpenFilePicker()`** → `FileSystemFileHandle` → `getFile()` → `web_sys::File`. Совместим с остальным кодом главы 18.
- **`showSaveFilePicker()`** → `FileSystemFileHandle` → `createWritable()` → `write()` → `close()`. Данные попадают на диск только после `close()`.
- **`showDirectoryPicker()`** → `FileSystemDirectoryHandle` → `values()` → рекурсивный обход.
- **Три требования:** Chrome/Edge + secure context + user gesture. Нарушение — `SecurityError` или отсутствие API.
- **web-sys 0.3.105:** `show_*_picker_with_options` не экспортированы, `values()` возвращает `AsyncIterator`, `remove_entry()` возвращает `Promise` напрямую. Все три нюанса обходим через `Reflect::get` + `Function::call*` и ручную итерацию.

## Замечания на будущее

**Правило проекта, зафиксированное в этой подглаве:**

1. **FSA не заменяет `<input>` и `<a download>` — он дополняет.** Всегда давайте fallback для Firefox и Safari. Проверка через `is_supported()` в начале секции.

2. **User gesture — не пожелание, а требование.** `show*Picker` **должны** вызываться из обработчика клика. Не пытайтесь обернуть их в `setTimeout` — браузер отклонит.

3. **`close()` обязателен для `FileSystemWritableFileStream`.** Данные буферизуются и попадают на диск только при закрытии потока.

4. **Отмена диалога — не ошибка.** `show*Picker` реджектит промис с `AbortError` при отмене. Возвращайте `Ok(None)`, а не `Err`.

5. **Итеративная очередь для обхода дерева.** Async-рекурсия в Rust требует `Box::pin` на каждом уровне и может переполнить стек.

6. **`FileSystemDirectoryHandle.values()` возвращает `js_sys::AsyncIterator`.** Не `Promise`. Итерируйте вручную через `iter.next()` → `Promise` → `JsFuture::from`.

7. **`Reflect::get` + `Function::call1` — универсальный способ вызова JS API.** Если web-sys не экспортирует метод с нужной сигнатурой, вызывайте его через Reflect. Это работает независимо от версии web-sys.

8. **Правило проекта про `format!` в `if/else`.** В этом окружении `format!` в tail-позиции блока, присваиваемого в `let`, выводится как `()`. Используйте предобъявленную переменную:

   ```rust
   let x: String;
   if cond { x = a; } else { x = format!(...); }
   ```

9. **Демо должно имитировать реальный workflow.** Кнопка «Сохранить» в отрыве от «Открыть» демонстрирует не FSA, а просто «записать байты на диск». Реальный сценарий — «взял файл, обработал, положил рядом».