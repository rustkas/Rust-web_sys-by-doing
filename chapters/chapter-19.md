# Глава 19. File System Access API

## Что раскрывает глава

Подглава 18.15 показала три практических сценария FSA: открыть файл, сохранить файл, обойти папку. Этого достаточно для учебного примера, но не для реальной работы — остаются незакрытыми `FileSystemHandle`, `getFileHandle/getDirectoryHandle/removeEntry/resolve`, `FileSystemWritableFileStream` с `seek/truncate`, permissions, потоковое чтение больших файлов, chunked-запись с прогрессом.

Глава 19 даёт **полный обзор API** всех типов FSA, выстроенный вокруг одного практического проекта — «аудит папки с экспортом отчёта в CSV». Каждая тема — отдельный модуль `fsa_api_*.rs`, тесты — отдельные файлы, демо — единый workflow в `app.rs`.

**Что не пересекается с 18.15:**

| 18.15 | 19 |
|---|---|
| `fsa_open`, `fsa_save`, `fsa_directory` | `fsa_api_handle`, `fsa_api_file`, `fsa_api_directory`, `fsa_api_writable`, `fsa_api_pickers`, `fsa_api_permissions`, `fsa_api_reader`, `fsa_api_writer`, `fsa_api_large`, `fsa_api_project` |
| Плоские функции | Полные обёртки всех методов |
| Три отдельные кнопки | Сквозной workflow «аудит папки» |

---

## Структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                       # +фичи web-sys для FSA
├── src/
│   ├── lib.rs                       # +10 pub mod fsa_api_*
│   ├── app.rs                       # демо-секция главы 19
│   ├── fsa_api_handle.rs
│   ├── fsa_api_file.rs
│   ├── fsa_api_directory.rs
│   ├── fsa_api_writable.rs
│   ├── fsa_api_pickers.rs
│   ├── fsa_api_permissions.rs
│   ├── fsa_api_reader.rs
│   ├── fsa_api_writer.rs
│   ├── fsa_api_large.rs
│   └── fsa_api_project.rs
└── tests/
    ├── wasm_fsa_api_handle.rs
    ├── wasm_fsa_api_pickers.rs
    ├── wasm_fsa_api_reader.rs
    ├── wasm_fsa_api_writer.rs
    ├── wasm_fsa_api_directory.rs
    ├── wasm_fsa_api_large.rs
    └── wasm_fsa_api_project.rs
```

**Краткий комментарий.** Все модули используют префикс `fsa_api_`, чтобы не пересекаться с решениями 18.15. Каждый модуль покрывает одну тему из списка и может использоваться независимо. Демо в `app.rs` — отдельная секция «аудит папки», не накапливает предыдущие главы.

---

## Ссылки

- [MDN: File System API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API)
- [MDN: FileSystemHandle](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemHandle)
- [MDN: FileSystemFileHandle](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle)
- [MDN: FileSystemDirectoryHandle](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemDirectoryHandle)
- [MDN: FileSystemWritableFileStream](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemWritableFileStream)
- [Chrome: File System Access API](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access)
- [web-sys: FileSystemFileHandle](https://docs.rs/web-sys/latest/web_sys/struct.FileSystemFileHandle.html)

---

## Исходные модули

### `src/fsa_api_handle.rs` — `FileSystemHandle`

```rust
//! FileSystemHandle — базовый тип для FileSystemFileHandle и
//! FileSystemDirectoryHandle. Даёт kind, name, isSameEntry.
//! Все методы через Reflect — независимость от версии web-sys.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

pub fn kind(handle: &JsValue) -> String {
    js_sys::Reflect::get(handle, &JsValue::from_str("kind"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

pub fn name(handle: &JsValue) -> String {
    js_sys::Reflect::get(handle, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

fn call_method(handle: &JsValue, method: &str, args: &[JsValue]) -> Result<js_sys::Promise, JsValue> {
    let fn_js = js_sys::Reflect::get(handle, &JsValue::from_str(method))?;
    if fn_js.is_undefined() {
        return Err(JsValue::from_str(&format!("метод {method} недоступен")));
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

pub async fn is_same_entry(a: &JsValue, b: &JsValue) -> Result<bool, JsValue> {
    let promise = call_method(a, "isSameEntry", &[b.clone()])?;
    let value = JsFuture::from(promise).await?;
    Ok(value.as_bool().unwrap_or(false))
}

pub fn as_file_handle(v: JsValue) -> Result<FileSystemFileHandle, JsValue> {
    let kind_str = kind(&v);
    if kind_str == "directory" {
        return Err(JsValue::from_str("handle указывает на директорию"));
    }
    v.dyn_into::<FileSystemFileHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))
}

pub fn as_directory_handle(v: JsValue) -> Result<FileSystemDirectoryHandle, JsValue> {
    let kind_str = kind(&v);
    if kind_str == "file" {
        return Err(JsValue::from_str("handle указывает на файл"));
    }
    v.dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}
```

### `src/fsa_api_file.rs` — `FileSystemFileHandle`

```rust
//! FileSystemFileHandle: getFile, createWritable, move.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{File, FileSystemDirectoryHandle, FileSystemFileHandle};

fn call_promise(handle: &JsValue, method: &str, args: &[JsValue]) -> Result<js_sys::Promise, JsValue> {
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

pub async fn get_file(handle: &FileSystemFileHandle) -> Result<File, JsValue> {
    let h: &JsValue = handle.as_ref();
    let promise = call_promise(h, "getFile", &[])?;
    let value = JsFuture::from(promise).await?;
    value.dyn_into::<File>().map_err(|_| JsValue::from_str("не File"))
}

pub async fn create_writable(
    handle: &FileSystemFileHandle,
    keep_existing: bool,
) -> Result<JsValue, JsValue> {
    let h: &JsValue = handle.as_ref();
    let promise = if keep_existing {
        let options = js_sys::Object::new();
        js_sys::Reflect::set(
            &options,
            &JsValue::from_str("keepExistingData"),
            &JsValue::from_bool(true),
        )?;
        call_promise(h, "createWritable", &[options.into()])?
    } else {
        call_promise(h, "createWritable", &[])?
    };
    JsFuture::from(promise).await
}

pub async fn move_to(
    handle: &FileSystemFileHandle,
    new_name: &str,
    dir: Option<&FileSystemDirectoryHandle>,
) -> Result<(), JsValue> {
    let h: &JsValue = handle.as_ref();
    let name_js = JsValue::from_str(new_name);
    let promise = match dir {
        Some(d) => {
            let d_js: &JsValue = d.as_ref();
            call_promise(h, "move", &[name_js, d_js.clone()])?
        }
        None => call_promise(h, "move", &[name_js])?,
    };
    let _ = JsFuture::from(promise).await?;
    Ok(())
}
```

### `src/fsa_api_directory.rs` — `FileSystemDirectoryHandle`

```rust
//! FileSystemDirectoryHandle. Обход — итеративный через VecDeque.
//! Yield раз в 16 мс — правило проекта для длительных async-операций.
//! remove_entry — идемпотентна (NotFoundError → Ok).

use std::collections::VecDeque;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

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

pub async fn get_file_handle(
    dir: &FileSystemDirectoryHandle,
    name: &str,
    create: bool,
) -> Result<FileSystemFileHandle, JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("create"),
        &JsValue::from_bool(create),
    )?;
    let dir_js: &JsValue = dir.as_ref();
    let promise = call_promise(dir_js, "getFileHandle", &[
        JsValue::from_str(name),
        options.into(),
    ])?;
    let value = JsFuture::from(promise).await?;
    value
        .dyn_into::<FileSystemFileHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))
}

pub async fn get_directory_handle(
    dir: &FileSystemDirectoryHandle,
    name: &str,
    create: bool,
) -> Result<FileSystemDirectoryHandle, JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("create"),
        &JsValue::from_bool(create),
    )?;
    let dir_js: &JsValue = dir.as_ref();
    let promise = call_promise(dir_js, "getDirectoryHandle", &[
        JsValue::from_str(name),
        options.into(),
    ])?;
    let value = JsFuture::from(promise).await?;
    value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}

/// removeEntry(name, {recursive}) — идемпотентное удаление.
/// Правило проекта: удалить дважды — успех, а не 404.
pub async fn remove_entry(
    dir: &FileSystemDirectoryHandle,
    name: &str,
    recursive: bool,
) -> Result<(), JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("recursive"),
        &JsValue::from_bool(recursive),
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
                .ok()
                .and_then(|v| v.as_string())
                .unwrap_or_default();
            if err_name == "NotFoundError" {
                Ok(())
            } else {
                Err(e)
            }
        }
    }
}

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
        .dyn_into()
        .map_err(|_| JsValue::from_str("resolve: не массив"))?;
    let mut parts: Vec<String> = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        if let Some(s) = array.get(i).as_string() {
            parts.push(s);
        }
    }
    Ok(Some(parts.join("/")))
}

#[derive(Debug, Clone, PartialEq)]
pub struct Entry {
    pub path: String,
    pub kind: String,
    pub size: f64,
    pub mime: String,
}

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
        if is_cancelled() || out.len() >= max_files {
            break;
        }

        let dir_js: &JsValue = dir.as_ref();
        let values_fn_js = js_sys::Reflect::get(dir_js, &JsValue::from_str("values"))?;
        if values_fn_js.is_undefined() {
            continue;
        }
        let values_fn: js_sys::Function = values_fn_js
            .dyn_into()
            .map_err(|_| JsValue::from_str("values не функция"))?;
        let iter: JsValue = values_fn.call0(dir_js)?;

        loop {
            if is_cancelled() || out.len() >= max_files {
                break;
            }
            let entry = match iter_next(&iter).await? {
                Some(v) => v,
                None => break,
            };

            let kind = js_sys::Reflect::get(&entry, &JsValue::from_str("kind"))?
                .as_string()
                .unwrap_or_default();
            let name = js_sys::Reflect::get(&entry, &JsValue::from_str("name"))?
                .as_string()
                .unwrap_or_default();

            let full_path: String;
            if prefix.is_empty() {
                full_path = name.clone();
            } else {
                full_path = format!("{prefix}/{name}");
            }

            if kind == "file" {
                let fh: FileSystemFileHandle = entry
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))?;
                let file = crate::fsa_api_file::get_file(&fh).await?;
                out.push(Entry {
                    path: full_path,
                    kind: "file".into(),
                    size: file.size(),
                    mime: file.type_(),
                });
                on_progress(out.len());

                let now = js_sys::Date::now();
                if now - last_yield >= 16.0 {
                    TimeoutFuture::new(0).await;
                    last_yield = now;
                }
            } else if kind == "directory" {
                let dh: FileSystemDirectoryHandle = entry
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))?;
                queue.push_back((dh, full_path));
            }
        }
    }
    Ok(out)
}

/// Один шаг async-итератора: iter.next() → Promise → JsFuture.
pub async fn iter_next(iter: &JsValue) -> Result<Option<JsValue>, JsValue> {
    let next_js = js_sys::Reflect::get(iter, &JsValue::from_str("next"))?;
    let next_fn: js_sys::Function = next_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("next не функция"))?;
    let promise: js_sys::Promise = next_fn
        .call0(iter)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("next не Promise"))?;
    let result: JsValue = JsFuture::from(promise).await?;
    let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
        .as_bool()
        .unwrap_or(true);
    if done {
        return Ok(None);
    }
    Ok(Some(js_sys::Reflect::get(
        &result,
        &JsValue::from_str("value"),
    )?))
}
```

### `src/fsa_api_writable.rs` — `FileSystemWritableFileStream`

```rust
//! FileSystemWritableFileStream: write, seek, truncate, close.
//! close() обязателен — данные попадают на диск только при закрытии.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

fn call_promise(
    stream: &JsValue,
    method: &str,
    args: &[JsValue],
) -> Result<js_sys::Promise, JsValue> {
    let fn_js = js_sys::Reflect::get(stream, &JsValue::from_str(method))?;
    if fn_js.is_undefined() {
        return Err(JsValue::from_str(&format!("{method} недоступен")));
    }
    let func: js_sys::Function = fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{method} не функция")))?;
    let promise_js = match args.len() {
        0 => func.call0(stream)?,
        1 => func.call1(stream, &args[0])?,
        _ => return Err(JsValue::from_str("слишком много аргументов")),
    };
    promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{method} вернул не Promise")))
}

pub async fn write_bytes(stream: &JsValue, data: &[u8]) -> Result<(), JsValue> {
    let uint8 = js_sys::Uint8Array::from(data);
    let promise = call_promise(stream, "write", &[uint8.into()])?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}

pub async fn write_bytes_at(
    stream: &JsValue,
    data: &[u8],
    position: f64,
) -> Result<(), JsValue> {
    let uint8 = js_sys::Uint8Array::from(data);
    let payload = js_sys::Object::new();
    js_sys::Reflect::set(
        &payload,
        &JsValue::from_str("type"),
        &JsValue::from_str("write"),
    )?;
    js_sys::Reflect::set(&payload, &JsValue::from_str("position"), &JsValue::from_f64(position))?;
    js_sys::Reflect::set(&payload, &JsValue::from_str("data"), &uint8)?;
    let promise = call_promise(stream, "write", &[payload.into()])?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}

pub async fn write_blob(stream: &JsValue, blob: &web_sys::Blob) -> Result<(), JsValue> {
    let blob_js: &JsValue = blob.as_ref();
    let promise = call_promise(stream, "write", &[blob_js.clone()])?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}

pub async fn write_text(stream: &JsValue, text: &str) -> Result<(), JsValue> {
    let array = js_sys::Array::of1(&JsValue::from_str(text));
    let options = web_sys::BlobPropertyBag::new();
    options.set_type("text/plain;charset=utf-8");
    let blob = web_sys::Blob::new_with_str_sequence_and_options(&array, &options)?;
    write_blob(stream, &blob).await
}

pub async fn seek(stream: &JsValue, position: f64) -> Result<(), JsValue> {
    let promise = call_promise(stream, "seek", &[JsValue::from_f64(position)])?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}

pub async fn truncate(stream: &JsValue, size: f64) -> Result<(), JsValue> {
    let promise = call_promise(stream, "truncate", &[JsValue::from_f64(size)])?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}

pub async fn close(stream: &JsValue) -> Result<(), JsValue> {
    let promise = call_promise(stream, "close", &[])?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}
```

### `src/fsa_api_pickers.rs` — три picker'а

```rust
//! Три picker'а FSA: open, save, directory. Все вызовы через Reflect.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

pub fn is_supported() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let w: &JsValue = window.as_ref();
    js_sys::Reflect::get(w, &JsValue::from_str("showOpenFilePicker"))
        .map(|v| !v.is_undefined())
        .unwrap_or(false)
}

async fn call_window_picker(
    method: &str,
    options: Option<JsValue>,
) -> Result<Option<JsValue>, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let w: &JsValue = window.as_ref();
    let fn_js = js_sys::Reflect::get(w, &JsValue::from_str(method))?;
    if fn_js.is_undefined() {
        return Err(JsValue::from_str(&format!("{method} недоступен")));
    }
    let func: js_sys::Function = fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;
    let promise_js = match options {
        Some(opts) => func.call1(w, &opts)?,
        None => func.call0(w)?,
    };
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    match JsFuture::from(promise).await {
        Ok(v) => Ok(Some(v)),
        Err(_) => Ok(None),
    }
}

pub async fn open_files(
    options: Option<JsValue>,
) -> Result<Option<Vec<FileSystemFileHandle>>, JsValue> {
    let value = match call_window_picker("showOpenFilePicker", options).await? {
        Some(v) => v,
        None => return Ok(None),
    };
    let array: js_sys::Array = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("open_files: не массив"))?;
    let mut out: Vec<FileSystemFileHandle> = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let fh: FileSystemFileHandle = array
            .get(i)
            .dyn_into()
            .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))?;
        out.push(fh);
    }
    Ok(Some(out))
}

pub async fn save_file(
    suggested_name: &str,
) -> Result<Option<FileSystemFileHandle>, JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;
    let value = match call_window_picker("showSaveFilePicker", Some(options.into())).await? {
        Some(v) => v,
        None => return Ok(None),
    };
    value
        .dyn_into::<FileSystemFileHandle>()
        .map(Some)
        .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))
}

pub async fn pick_directory(
    mode: &str,
) -> Result<Option<FileSystemDirectoryHandle>, JsValue> {
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("mode"),
        &JsValue::from_str(mode),
    )?;
    let value = match call_window_picker("showDirectoryPicker", Some(options.into())).await? {
        Some(v) => v,
        None => return Ok(None),
    };
    value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map(Some)
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}

pub fn build_types(entries: &[(&str, &[&str])]) -> Result<JsValue, JsValue> {
    let types = js_sys::Array::new();
    for (description, extensions) in entries {
        let ext_arr = js_sys::Array::new();
        for ext in extensions.iter() {
            ext_arr.push(&JsValue::from_str(ext));
        }
        let accept = js_sys::Object::new();
        js_sys::Reflect::set(&accept, &JsValue::from_str(description), &ext_arr)?;
        let entry = js_sys::Object::new();
        js_sys::Reflect::set(
            &entry,
            &JsValue::from_str("description"),
            &JsValue::from_str(description),
        )?;
        js_sys::Reflect::set(&entry, &JsValue::from_str("accept"), &accept)?;
        types.push(&entry);
    }
    Ok(types.into())
}
```

### `src/fsa_api_permissions.rs` — Permissions

```rust
//! Permissions для FSA: handle.queryPermission({mode}) и
//! handle.requestPermission({mode}). Режимы: "read" и "readwrite".
//! Разрешения не сохраняются между сессиями.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum PermissionState {
    Granted,
    Denied,
    Prompt,
}

impl PermissionState {
    pub fn as_str(self) -> &'static str {
        match self {
            PermissionState::Granted => "granted",
            PermissionState::Denied => "denied",
            PermissionState::Prompt => "prompt",
        }
    }
    pub fn from_str(s: &str) -> Self {
        match s {
            "granted" => PermissionState::Granted,
            "denied" => PermissionState::Denied,
            _ => PermissionState::Prompt,
        }
    }
}

fn build_descriptor(mode: &str) -> Result<JsValue, JsValue> {
    let descriptor = js_sys::Object::new();
    js_sys::Reflect::set(
        &descriptor,
        &JsValue::from_str("mode"),
        &JsValue::from_str(mode),
    )?;
    Ok(descriptor.into())
}

async fn call_permission(
    handle: &JsValue,
    method: &str,
    mode: &str,
) -> Result<PermissionState, JsValue> {
    let fn_js = js_sys::Reflect::get(handle, &JsValue::from_str(method))?;
    if fn_js.is_undefined() {
        return Err(JsValue::from_str(&format!("{method} недоступен")));
    }
    let func: js_sys::Function = fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;
    let descriptor = build_descriptor(mode)?;
    let promise: js_sys::Promise = func
        .call1(handle, &descriptor)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let value = JsFuture::from(promise).await?;
    let s = value.as_string().unwrap_or_default();
    Ok(PermissionState::from_str(&s))
}

pub async fn query_permission(
    handle: &JsValue,
    mode: &str,
) -> Result<PermissionState, JsValue> {
    call_permission(handle, "queryPermission", mode).await
}

pub async fn request_permission(
    handle: &JsValue,
    mode: &str,
) -> Result<PermissionState, JsValue> {
    call_permission(handle, "requestPermission", mode).await
}
```

### `src/fsa_api_reader.rs` — чтение

```rust
//! Чтение: text, arrayBuffer, slice, потоковое с прогрессом и отменой.

use gloo_timers::future::TimeoutFuture;
use js_sys::Uint8Array;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{File, ReadableStreamDefaultReader};

pub enum ReadOutcome {
    Ok(Vec<u8>),
    Cancelled,
}

pub async fn read_text(file: &File) -> Result<String, JsValue> {
    let promise: js_sys::Promise = file.text();
    let value = JsFuture::from(promise).await?;
    value.as_string().ok_or_else(|| JsValue::from_str("не строка"))
}

pub async fn read_bytes(file: &File) -> Result<Vec<u8>, JsValue> {
    let promise: js_sys::Promise = file.array_buffer();
    let value = JsFuture::from(promise).await?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let uint8 = Uint8Array::new(&buffer);
    Ok(uint8.to_vec())
}

pub async fn read_slice(file: &File, start: f64, end: f64) -> Result<Vec<u8>, JsValue> {
    let slice = file.slice_with_f64_and_f64(start, end)?;
    let promise: js_sys::Promise = slice.array_buffer();
    let value = JsFuture::from(promise).await?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    Ok(Uint8Array::new(&buffer).to_vec())
}

pub async fn read_stream(
    file: &File,
    mut on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<ReadOutcome, JsValue> {
    let stream = file.stream();
    let reader = ReadableStreamDefaultReader::new(&stream)?;
    let total = file.size().max(1.0);
    let mut received: f64 = 0.0;
    let mut chunks: Vec<u8> = Vec::new();
    let mut last_yield = 0.0_f64;

    loop {
        if is_cancelled() {
            return Ok(ReadOutcome::Cancelled);
        }
        let result = JsFuture::from(reader.read()).await?;
        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
            .as_bool()
            .unwrap_or(true);
        if done {
            break;
        }
        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))?;
        let chunk = Uint8Array::new(&value);
        let len = chunk.length() as f64;
        chunks.extend_from_slice(&chunk.to_vec());
        received += len;

        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            on_progress(received / total);
            TimeoutFuture::new(0).await;
            last_yield = now;
        }
    }
    on_progress(1.0);
    Ok(ReadOutcome::Ok(chunks))
}
```

### `src/fsa_api_writer.rs` — запись

```rust
//! Запись: write_all_bytes, write_all_text, write_chunked.

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use web_sys::FileSystemFileHandle;

pub enum WriteOutcome {
    Ok,
    Cancelled,
}

pub async fn write_all_bytes(
    handle: &FileSystemFileHandle,
    data: &[u8],
) -> Result<String, JsValue> {
    let stream = crate::fsa_api_file::create_writable(handle, false).await?;
    crate::fsa_api_writable::write_bytes(&stream, data).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle.name())
}

pub async fn write_all_text(
    handle: &FileSystemFileHandle,
    text: &str,
) -> Result<String, JsValue> {
    let stream = crate::fsa_api_file::create_writable(handle, false).await?;
    crate::fsa_api_writable::write_text(&stream, text).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle.name())
}

pub async fn write_chunked(
    handle: &FileSystemFileHandle,
    data: &[u8],
    mut on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<WriteOutcome, JsValue> {
    let stream = crate::fsa_api_file::create_writable(handle, false).await?;
    let total = data.len().max(1);
    let chunk_size = crate::fsa_api_large::adaptive_chunk_size(data.len());
    let mut last_yield = 0.0_f64;

    for (i, chunk) in data.chunks(chunk_size).enumerate() {
        if is_cancelled() {
            let _ = crate::fsa_api_writable::close(&stream).await;
            return Ok(WriteOutcome::Cancelled);
        }
        crate::fsa_api_writable::write_bytes(&stream, chunk).await?;
        let processed = ((i + 1) * chunk_size).min(data.len());
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            on_progress(processed as f64 / total as f64);
            TimeoutFuture::new(0).await;
            last_yield = now;
        }
    }
    crate::fsa_api_writable::close(&stream).await?;
    on_progress(1.0);
    Ok(WriteOutcome::Ok)
}
```

### `src/fsa_api_large.rs` — большие файлы

```rust
//! Работа с большими файлами: адаптивный chunk и копирование.

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use web_sys::{File, FileSystemFileHandle};

pub fn adaptive_chunk_size(total: usize) -> usize {
    let target_iters = 200usize;
    (total / target_iters).clamp(64 * 1024, 4 * 1024 * 1024)
}

pub enum CopyOutcome {
    Ok(u64),
    Cancelled,
}

pub async fn copy_file(
    src: &File,
    dst: &FileSystemFileHandle,
    mut on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<CopyOutcome, JsValue> {
    let stream = crate::fsa_api_file::create_writable(dst, false).await?;
    let total = src.size().max(1.0);
    let reader = web_sys::ReadableStreamDefaultReader::new(&src.stream())?;
    let mut written: f64 = 0.0;
    let mut last_yield = 0.0_f64;

    loop {
        if is_cancelled() {
            let _ = crate::fsa_api_writable::close(&stream).await;
            return Ok(CopyOutcome::Cancelled);
        }
        let result = wasm_bindgen_futures::JsFuture::from(reader.read()).await?;
        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
            .as_bool()
            .unwrap_or(true);
        if done {
            break;
        }
        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))?;
        let chunk = js_sys::Uint8Array::new(&value);
        let bytes = chunk.to_vec();
        written += bytes.len() as f64;
        crate::fsa_api_writable::write_bytes(&stream, &bytes).await?;

        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            on_progress(written / total);
            TimeoutFuture::new(0).await;
            last_yield = now;
        }
    }
    crate::fsa_api_writable::close(&stream).await?;
    on_progress(1.0);
    Ok(CopyOutcome::Ok(written as u64))
}
```

### `src/fsa_api_project.rs` — практический проект

```rust
//! Аудит папки: обход, SHA-256 первых 64 КиБ, экспорт в CSV.
//! Yield раз в 16 мс — прогресс и отмена видны в UI.

use std::collections::VecDeque;

use gloo_timers::future::TimeoutFuture;
use sha2::{Digest, Sha256};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

#[derive(Debug, Clone, PartialEq)]
pub struct AuditRow {
    pub path: String,
    pub size: f64,
    pub mime: String,
    pub hash_prefix: String,
}

pub struct AuditReport {
    pub rows: Vec<AuditRow>,
    pub truncated: bool,
}

const HASH_PREFIX_BYTES: f64 = 64.0 * 1024.0;

async fn file_hash_prefix(file: &web_sys::File) -> Result<String, JsValue> {
    let end = file.size().min(HASH_PREFIX_BYTES);
    let slice = file.slice_with_f64_and_f64(0.0, end)?;
    let promise: js_sys::Promise = slice.array_buffer();
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let bytes = js_sys::Uint8Array::new(&buffer).to_vec();

    let mut hasher = Sha256::new();
    hasher.update(&bytes);
    let full = hex::encode(hasher.finalize());
    Ok(full[..16].to_string())
}

/// Аудит директории с лимитом файлов.
/// `on_file(count, path, size)` — колбэк для live-лога.
pub async fn audit_directory(
    root: &FileSystemDirectoryHandle,
    max_files: usize,
    mut on_file: impl FnMut(usize, &str, f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<AuditReport, JsValue> {
    let mut rows: Vec<AuditRow> = Vec::new();
    let mut queue: VecDeque<(FileSystemDirectoryHandle, String)> = VecDeque::new();
    queue.push_back((root.clone(), String::new()));
    let mut last_yield = 0.0_f64;

    while let Some((dir, prefix)) = queue.pop_front() {
        if is_cancelled() || rows.len() >= max_files {
            break;
        }

        let dir_js: &JsValue = dir.as_ref();
        let values_fn_js = js_sys::Reflect::get(dir_js, &JsValue::from_str("values"))?;
        if values_fn_js.is_undefined() {
            continue;
        }
        let values_fn: js_sys::Function = values_fn_js
            .dyn_into()
            .map_err(|_| JsValue::from_str("values не функция"))?;
        let iter: JsValue = values_fn.call0(dir_js)?;

        loop {
            if is_cancelled() || rows.len() >= max_files {
                break;
            }
            let entry = match crate::fsa_api_directory::iter_next(&iter).await? {
                Some(v) => v,
                None => break,
            };

            let kind = js_sys::Reflect::get(&entry, &JsValue::from_str("kind"))?
                .as_string()
                .unwrap_or_default();
            let name = js_sys::Reflect::get(&entry, &JsValue::from_str("name"))?
                .as_string()
                .unwrap_or_default();

            let full_path: String;
            if prefix.is_empty() {
                full_path = name.clone();
            } else {
                full_path = format!("{prefix}/{name}");
            }

            if kind == "file" {
                let fh: FileSystemFileHandle = entry
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не FileSystemFileHandle"))?;
                let file = crate::fsa_api_file::get_file(&fh).await?;
                let size = file.size();
                let hash_prefix = file_hash_prefix(&file)
                    .await
                    .unwrap_or_else(|_| "error".into());
                rows.push(AuditRow {
                    path: full_path.clone(),
                    size,
                    mime: file.type_(),
                    hash_prefix,
                });
                on_file(rows.len(), &full_path, size);

                let now = js_sys::Date::now();
                if now - last_yield >= 16.0 {
                    TimeoutFuture::new(0).await;
                    last_yield = now;
                }
            } else if kind == "directory" {
                let dh: FileSystemDirectoryHandle = entry
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))?;
                queue.push_back((dh, full_path));
            }
        }
    }

    let truncated = rows.len() >= max_files;
    Ok(AuditReport { rows, truncated })
}

pub fn report_to_csv(report: &AuditReport) -> String {
    let mut out = String::with_capacity(report.rows.len() * 96);
    out.push_str("path,size_bytes,mime,sha256_prefix\n");
    for row in &report.rows {
        out.push_str(&csv_escape(&row.path));
        out.push(',');
        out.push_str(&format!("{:.0}", row.size));
        out.push(',');
        out.push_str(&csv_escape(&row.mime));
        out.push(',');
        out.push_str(&row.hash_prefix);
        out.push('\n');
    }
    if report.truncated {
        out.push_str("# truncated by SCAN_LIMIT\n");
    }
    out
}

fn csv_escape(s: &str) -> String {
    if s.contains(',') || s.contains('"') || s.contains('\n') {
        let mut escaped = String::with_capacity(s.len() + 2);
        escaped.push('"');
        for ch in s.chars() {
            if ch == '"' {
                escaped.push('"');
            }
            escaped.push(ch);
        }
        escaped.push('"');
        escaped
    } else {
        s.to_string()
    }
}
```

---

## Тесты

### `tests/wasm_fsa_api_handle.rs`

```rust
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;
use hello_wasm02::fsa_api_handle::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_kind_and_name_on_non_handle() {
    let v = JsValue::from_str("not a handle");
    assert_eq!(kind(&v), "");
    assert_eq!(name(&v), "");
}

#[wasm_bindgen_test]
fn test_as_file_handle_rejects_non_handle() {
    let v = JsValue::from_str("x");
    assert!(as_file_handle(v).is_err());
}

#[wasm_bindgen_test]
fn test_as_directory_handle_rejects_non_handle() {
    let v = JsValue::from_str("x");
    assert!(as_directory_handle(v).is_err());
}
```

### `tests/wasm_fsa_api_pickers.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::fsa_api_pickers::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_is_supported_returns_bool() {
    let s = is_supported();
    web_sys::console::log_1(&format!("FSA supported: {s}").into());
}

#[wasm_bindgen_test]
async fn test_open_files_without_gesture_is_safe() {
    let result = open_files(None).await;
    match result {
        Ok(_) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            assert!(
                s.is_empty()
                    || s.contains("SecurityError")
                    || s.contains("gesture")
                    || s.contains("User activation")
            );
        }
    }
}

#[wasm_bindgen_test]
async fn test_pick_directory_without_gesture_is_safe() {
    let result = pick_directory("read").await;
    match result {
        Ok(_) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(&format!("pick_directory err: {s}").into());
        }
    }
}

#[wasm_bindgen_test]
fn test_build_types_constructs_array() {
    let types = build_types(&[("Text", &[".txt", ".md"])]).unwrap();
    let arr = js_sys::Array::from(&types);
    assert_eq!(arr.length(), 1);
}
```

### `tests/wasm_fsa_api_reader.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::fsa_api_reader::*;

wasm_bindgen_test_configure!(run_in_browser);

fn make_file(content: &[u8], name: &str) -> web_sys::File {
    let u8arr = js_sys::Uint8Array::from(content);
    let blob = web_sys::Blob::new_with_u8_array_sequence(&js_sys::Array::of1(&u8arr)).unwrap();
    web_sys::File::new_with_blob_sequence(&js_sys::Array::of1(&blob), name).unwrap()
}

#[wasm_bindgen_test]
async fn test_read_text_roundtrip() {
    let file = make_file("Hello, FSA!".as_bytes(), "hello.txt");
    let text = read_text(&file).await.unwrap();
    assert_eq!(text, "Hello, FSA!");
}

#[wasm_bindgen_test]
async fn test_read_bytes_roundtrip() {
    let data: Vec<u8> = (0..=255).collect();
    let file = make_file(&data, "bytes.bin");
    let bytes = read_bytes(&file).await.unwrap();
    assert_eq!(bytes, data);
}

#[wasm_bindgen_test]
async fn test_read_slice() {
    let data: Vec<u8> = (0..100).collect();
    let file = make_file(&data, "slice.bin");
    let slice = read_slice(&file, 10.0, 20.0).await.unwrap();
    assert_eq!(slice.len(), 10);
    assert_eq!(slice, &data[10..20]);
}

#[wasm_bindgen_test]
async fn test_read_stream_progress() {
    let data = vec![0xABu8; 128 * 1024];
    let file = make_file(&data, "stream.bin");
    let hits = std::rc::Rc::new(std::cell::Cell::new(0u32));
    let h = hits.clone();
    let outcome = read_stream(&file, move |_| h.set(h.get() + 1), || false).await.unwrap();
    match outcome {
        ReadOutcome::Ok(b) => assert_eq!(b, data),
        _ => panic!("expected Ok"),
    }
    assert!(hits.get() >= 1);
}

#[wasm_bindgen_test]
async fn test_read_stream_cancel() {
    let data = vec![0u8; 256 * 1024];
    let file = make_file(&data, "cancel.bin");
    let outcome = read_stream(&file, |_| {}, || true).await.unwrap();
    assert!(matches!(outcome, ReadOutcome::Cancelled));
}
```

### `tests/wasm_fsa_api_writer.rs`

```rust
//! Тесты записи через OPFS (не требует user gesture).

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use wasm_bindgen_test::*;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

use hello_wasm02::fsa_api_writer::{write_all_bytes, write_all_text, write_chunked, WriteOutcome};

wasm_bindgen_test_configure!(run_in_browser);

async fn opfs_root() -> Result<FileSystemDirectoryHandle, JsValue> {
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
    let get_dir: js_sys::Function = get_dir_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
    let promise_js = get_dir.call0(&storage_js)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getDirectory не Promise"))?;
    let value: JsValue = JsFuture::from(promise).await?;
    value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}

async fn opfs_file(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<FileSystemFileHandle, JsValue> {
    hello_wasm02::fsa_api_directory::get_file_handle(root, name, true).await
}

async fn opfs_remove(root: &FileSystemDirectoryHandle, name: &str) {
    let _ = hello_wasm02::fsa_api_directory::remove_entry(root, name, false).await;
}

#[wasm_bindgen_test]
async fn test_write_all_bytes_roundtrip() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_writer_bytes.bin";
    opfs_remove(&root, name).await;
    let handle = opfs_file(&root, name).await.expect("create file");

    let data: Vec<u8> = (0..=255).collect();
    let written = write_all_bytes(&handle, &data).await.expect("write");
    assert_eq!(written, name);

    let file = hello_wasm02::fsa_api_file::get_file(&handle)
        .await
        .expect("get_file");
    assert_eq!(file.size() as usize, data.len(), "size mismatch");

    let read = hello_wasm02::fsa_api_reader::read_bytes(&file)
        .await
        .expect("read_bytes");
    assert_eq!(read, data, "content mismatch");

    opfs_remove(&root, name).await;
}

#[wasm_bindgen_test]
async fn test_write_all_text_roundtrip() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_writer_text.txt";
    opfs_remove(&root, name).await;
    let handle = opfs_file(&root, name).await.expect("create file");

    let text = "Hello, FSA writer!\nВторая строка.\n\u{1F600} эмодзи.";
    let written = write_all_text(&handle, text).await.expect("write");
    assert_eq!(written, name);

    let file = hello_wasm02::fsa_api_file::get_file(&handle)
        .await
        .expect("get_file");
    let stored = hello_wasm02::fsa_api_reader::read_text(&file)
        .await
        .expect("read_text");
    assert_eq!(stored, text);

    opfs_remove(&root, name).await;
}

#[wasm_bindgen_test]
async fn test_write_chunked_progress_reaches_one() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_writer_chunked.bin";
    opfs_remove(&root, name).await;
    let handle = opfs_file(&root, name).await.expect("create file");

    let data: Vec<u8> = (0..2 * 1024 * 1024).map(|i| (i & 0xFF) as u8).collect();
    let seen: Rc<RefCell<Vec<f64>>> = Rc::new(RefCell::new(Vec::new()));
    let seen_c = seen.clone();

    let outcome = write_chunked(
        &handle,
        &data,
        move |f| seen_c.borrow_mut().push(f),
        || false,
    )
    .await
    .expect("write_chunked");

    assert!(matches!(outcome, WriteOutcome::Ok));

    let v = seen.borrow();
    assert!(!v.is_empty(), "progress callback must be called at least once");
    for w in v.windows(2) {
        assert!(w[1] + 1e-9 >= w[0], "progress must be monotonic");
    }
    let last = *v.last().unwrap();
    assert!((last - 1.0).abs() < 1e-6, "final progress must be 1.0, got {last}");

    let file = hello_wasm02::fsa_api_file::get_file(&handle)
        .await
        .expect("get_file");
    assert_eq!(file.size() as usize, data.len(), "written size mismatch");

    opfs_remove(&root, name).await;
}

#[wasm_bindgen_test]
async fn test_write_chunked_cancel() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_writer_cancel.bin";
    opfs_remove(&root, name).await;
    let handle = opfs_file(&root, name).await.expect("create file");

    let data = vec![0xABu8; 512 * 1024];
    let outcome = write_chunked(&handle, &data, |_| {}, || true)
        .await
        .expect("write_chunked");

    assert!(matches!(outcome, WriteOutcome::Cancelled));

    opfs_remove(&root, name).await;
}
```

### `tests/wasm_fsa_api_directory.rs`

```rust
//! Интеграционные тесты `src/fsa_api_directory.rs` через OPFS.

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
        let get_dir: js_sys::Function = get_dir_js
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
        let promise: js_sys::Promise = get_dir
            .call0(&storage_js)?
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не Promise"))?;
        let value: JsValue = JsFuture::from(promise).await?;
        value
            .dyn_into::<FileSystemDirectoryHandle>()
            .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
    }

    pub async fn cleanup(root: &FileSystemDirectoryHandle, name: &str) {
        // В интеграционном тесте `crate::` указывает на тестовый крейт,
        // а не на библиотеку. Используем полный путь.
        let _ = hello_wasm02::fsa_api_directory::remove_entry(root, name, true).await;
    }
}

use hello_wasm02::fsa_api_directory::*;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_file_handle_create_and_read() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_file.txt";
    opfs::cleanup(&root, name).await;

    let handle = get_file_handle(&root, name, true).await.expect("create");
    assert_eq!(handle.name(), name);

    let bytes = b"hello fsa_api_directory";
    let stream = hello_wasm02::fsa_api_file::create_writable(&handle, false)
        .await
        .expect("create_writable");
    hello_wasm02::fsa_api_writable::write_bytes(&stream, bytes)
        .await
        .expect("write");
    hello_wasm02::fsa_api_writable::close(&stream)
        .await
        .expect("close");

    let file = hello_wasm02::fsa_api_file::get_file(&handle)
        .await
        .expect("get_file");
    assert_eq!(file.size() as usize, bytes.len());

    opfs::cleanup(&root, name).await;
}

#[wasm_bindgen_test]
async fn test_file_handle_missing_without_create() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_missing_xyz.txt";
    opfs::cleanup(&root, name).await;

    let res = get_file_handle(&root, name, false).await;
    assert!(res.is_err(), "ожидалась ошибка для отсутствующего файла");
}

#[wasm_bindgen_test]
async fn test_directory_handle_create_and_remove() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_subdir";
    opfs::cleanup(&root, name).await;

    let subdir = get_directory_handle(&root, name, true)
        .await
        .expect("create dir");
    assert_eq!(subdir.name(), name);

    remove_entry(&root, name, true).await.expect("remove");

    // Правило проекта: повторное удаление — идемпотентно, Ok.
    remove_entry(&root, name, true)
        .await
        .expect("повторное удаление должно быть Ok (идемпотентно)");
}

#[wasm_bindgen_test]
async fn test_remove_entry_file() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_rm.txt";
    opfs::cleanup(&root, name).await;
    let _ = get_file_handle(&root, name, true).await.expect("create");

    remove_entry(&root, name, false).await.expect("remove");
    assert!(get_file_handle(&root, name, false).await.is_err());
}

#[wasm_bindgen_test]
async fn test_resolve_path_returns_relative() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_resolve.txt";
    opfs::cleanup(&root, name).await;

    let file_handle = get_file_handle(&root, name, true).await.expect("create");
    let child_js: &wasm_bindgen::JsValue = file_handle.as_ref();

    let path = resolve_path(&root, child_js).await.expect("resolve");
    assert_eq!(path.as_deref(), Some(name));

    opfs::cleanup(&root, name).await;
}

#[wasm_bindgen_test]
async fn test_scan_reports_progress() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let dir_name = "fsa_api_dir_scan";
    opfs::cleanup(&root, dir_name).await;
    let subdir = get_directory_handle(&root, dir_name, true)
        .await
        .expect("mkdir");

    for name in ["a.txt", "b.txt", "c.txt"] {
        let _ = get_file_handle(&subdir, name, true)
            .await
            .expect("create");
    }

    let hits = std::rc::Rc::new(std::cell::Cell::new(0usize));
    let hits_c = hits.clone();
    let on_progress = move |_n: usize| hits_c.set(hits_c.get() + 1);

    let entries = scan(&subdir, 100, on_progress, || false)
        .await
        .expect("scan");

    assert_eq!(entries.len(), 3, "должно быть найдено 3 файла");
    assert!(hits.get() >= 3);

    opfs::cleanup(&root, dir_name).await;
}

#[wasm_bindgen_test]
async fn test_scan_respects_limit() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let dir_name = "fsa_api_dir_limit";
    opfs::cleanup(&root, dir_name).await;
    let subdir = get_directory_handle(&root, dir_name, true)
        .await
        .expect("mkdir");

    for i in 0..5 {
        let _ = get_file_handle(&subdir, &format!("f{i}.txt"), true)
            .await
            .expect("create");
    }

    let entries = scan(&subdir, 2, |_| {}, || false)
        .await
        .expect("scan");
    assert_eq!(entries.len(), 2);

    opfs::cleanup(&root, dir_name).await;
}

#[wasm_bindgen_test]
async fn test_scan_cancels_immediately() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let entries = scan(&root, 100, |_| {}, || true).await.expect("scan");
    assert!(entries.is_empty());
}

// ---------------------------------------------------------------------------
// Идемпотентность удаления
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_remove_entry_idempotent_file() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_idem.txt";
    opfs::cleanup(&root, name).await;
    let _ = get_file_handle(&root, name, true).await.expect("create");

    remove_entry(&root, name, false).await.expect("first remove");
    remove_entry(&root, name, false).await.expect("second remove (idempotent)");
    remove_entry(&root, name, false).await.expect("third remove (idempotent)");
}

#[wasm_bindgen_test]
async fn test_remove_entry_idempotent_directory() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_idem_dir";
    opfs::cleanup(&root, name).await;
    let _ = get_directory_handle(&root, name, true).await.expect("mkdir");

    remove_entry(&root, name, true).await.expect("first remove");
    remove_entry(&root, name, true).await.expect("second remove");
    remove_entry(&root, name, true).await.expect("third remove");
}

#[wasm_bindgen_test]
async fn test_remove_entry_missing_from_start() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_api_dir_never_existed";
    opfs::cleanup(&root, name).await;

    remove_entry(&root, name, true)
        .await
        .expect("удаление отсутствующей записи — Ok (идемпотентно)");
}
```

### `tests/wasm_fsa_api_large.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::fsa_api_large::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_adaptive_chunk_bounds() {
    assert_eq!(adaptive_chunk_size(0), 64 * 1024);
    assert_eq!(adaptive_chunk_size(1), 64 * 1024);
    let huge: usize = (u32::MAX - 1) as usize;
    assert_eq!(adaptive_chunk_size(huge), 4 * 1024 * 1024);
}

#[wasm_bindgen_test]
fn test_adaptive_chunk_mid() {
    let mid = adaptive_chunk_size(200 * 1024 * 1024);
    assert!(mid >= 64 * 1024 && mid <= 4 * 1024 * 1024);
}
```

### `tests/wasm_fsa_api_project.rs`

```rust
//! Тесты `src/fsa_api_project.rs` через OPFS.

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
        let get_dir: js_sys::Function = get_dir_js
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
        let promise: js_sys::Promise = get_dir
            .call0(&storage_js)?
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не Promise"))?;
        let value: JsValue = JsFuture::from(promise).await?;
        value
            .dyn_into::<FileSystemDirectoryHandle>()
            .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
    }

    pub async fn cleanup(root: &FileSystemDirectoryHandle, name: &str) {
        let _ = hello_wasm02::fsa_api_directory::remove_entry(root, name, true).await;
    }
}

use hello_wasm02::fsa_api_directory::{get_directory_handle, get_file_handle};
use hello_wasm02::fsa_api_project::*;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// Юнит-тесты CSV
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_report_to_csv_empty() {
    let report = AuditReport { rows: vec![], truncated: false };
    let csv = report_to_csv(&report);
    assert_eq!(csv, "path,size_bytes,mime,sha256_prefix\n");
}

#[wasm_bindgen_test]
fn test_report_to_csv_with_rows() {
    let report = AuditReport {
        rows: vec![AuditRow {
            path: "a.txt".into(),
            size: 12.0,
            mime: "text/plain".into(),
            hash_prefix: "abcdef0123456789".into(),
        }],
        truncated: false,
    };
    let csv = report_to_csv(&report);
    assert!(csv.contains("a.txt,12,text/plain,abcdef0123456789"));
}

#[wasm_bindgen_test]
fn test_report_to_csv_escapes_commas() {
    let report = AuditReport {
        rows: vec![AuditRow {
            path: "a,b.txt".into(),
            size: 0.0,
            mime: "text/plain".into(),
            hash_prefix: "0".repeat(16),
        }],
        truncated: false,
    };
    let csv = report_to_csv(&report);
    assert!(csv.contains("\"a,b.txt\""));
}

#[wasm_bindgen_test]
fn test_report_to_csv_marks_truncation() {
    let report = AuditReport { rows: vec![], truncated: true };
    let csv = report_to_csv(&report);
    assert!(csv.contains("# truncated"));
}

// ---------------------------------------------------------------------------
// audit_directory через OPFS
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_audit_directory_reports_files() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let dir_name = "fsa_api_proj_audit";
    opfs::cleanup(&root, dir_name).await;
    let subdir = get_directory_handle(&root, dir_name, true).await.expect("mkdir");

    for (name, content) in [
        ("a.txt", "hello A"),
        ("b.txt", "hello B"),
        ("c.txt", "hello C"),
    ] {
        let handle = get_file_handle(&subdir, name, true).await.expect("create");
        let stream = hello_wasm02::fsa_api_file::create_writable(&handle, false)
            .await
            .expect("create_writable");
        hello_wasm02::fsa_api_writable::write_bytes(&stream, content.as_bytes())
            .await
            .expect("write");
        hello_wasm02::fsa_api_writable::close(&stream).await.expect("close");
    }

    let on_file_calls: std::rc::Rc<std::cell::RefCell<Vec<String>>> =
        std::rc::Rc::new(std::cell::RefCell::new(Vec::new()));
    let ofc = on_file_calls.clone();
    let on_file = move |_count: usize, path: &str, _size: f64| {
        ofc.borrow_mut().push(path.to_string());
    };

    let report = audit_directory(&subdir, 100, on_file, || false)
        .await
        .expect("audit");

    assert_eq!(report.rows.len(), 3);
    assert!(!report.truncated);

    for row in &report.rows {
        assert!(!row.path.is_empty());
        assert!(row.size > 0.0);
        assert_eq!(row.mime, "text/plain");
        assert_eq!(row.hash_prefix.len(), 16);
    }

    let calls = on_file_calls.borrow();
    assert_eq!(calls.len(), 3);
    let mut sorted = calls.clone();
    sorted.sort();
    assert_eq!(sorted, vec!["a.txt", "b.txt", "c.txt"]);

    opfs::cleanup(&root, dir_name).await;
}

#[wasm_bindgen_test]
async fn test_audit_directory_cancel_immediately() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let report = audit_directory(&root, 100, |_, _, _| {}, || true)
        .await
        .expect("audit");
    assert!(report.rows.is_empty());
    assert!(!report.truncated);
}

#[wasm_bindgen_test]
async fn test_audit_directory_truncated_flag() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let dir_name = "fsa_api_proj_trunc";
    opfs::cleanup(&root, dir_name).await;
    let subdir = get_directory_handle(&root, dir_name, true).await.expect("mkdir");
    for i in 0..3 {
        let _ = get_file_handle(&subdir, &format!("f{i}.txt"), true).await.expect("create");
    }

    let report = audit_directory(&subdir, 2, |_, _, _| {}, || false)
        .await
        .expect("audit");
    assert_eq!(report.rows.len(), 2);
    assert!(report.truncated);

    opfs::cleanup(&root, dir_name).await;
}
```

---

## Команды запуска

```bash
# Проверка типов
cargo check --target wasm32-unknown-unknown

# Тесты по модулям
wasm-pack test --headless --chrome --test wasm_fsa_api_handle
wasm-pack test --headless --chrome --test wasm_fsa_api_pickers
wasm-pack test --headless --chrome --test wasm_fsa_api_reader
wasm-pack test --headless --chrome --test wasm_fsa_api_writer
wasm-pack test --headless --chrome --test wasm_fsa_api_directory
wasm-pack test --headless --chrome --test wasm_fsa_api_large
wasm-pack test --headless --chrome --test wasm_fsa_api_project

# Всё вместе
npm test

# Демо
npm start
```

---

## Демо-пример в `app.rs`

```rust
//! Демо-секция главы 19: File System Access API.
//!
//! Workflow: аудит папки.
//! 1. Выбрать папку (directory picker).
//! 2. [опционально] запросить readwrite-разрешение.
//! 3. Обойти файлы с live-логом, прогрессом и отменой.
//! 4. Посчитать SHA-256 первых 64 КиБ.
//! 5. Показать таблицу с разбивкой по типам.
//! 6. Клик по строке → полный SHA-256 файла.
//! 7. Экспорт отчёта в CSV через save picker.
//! 8. «Повторно» — тот же handle, без нового диалога.
//!
//! Контракт с `lib.rs`: `run() -> Result<(), JsValue>`.

use std::cell::{Cell, RefCell};
use std::collections::HashMap;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    FileSystemDirectoryHandle, FileSystemFileHandle, HtmlButtonElement, HtmlElement,
    HtmlProgressElement,
};

use crate::fsa_api_directory;
use crate::fsa_api_permissions::{self, PermissionState};
use crate::fsa_api_pickers;
use crate::fsa_api_project::{self, AuditReport};
use crate::fsa_api_writer;

const SCAN_LIMIT: usize = 100;
const LIVE_LOG_MAX: usize = 8;

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);

        let css = r#"
            #fsa-api-demo { font-family: system-ui, sans-serif; max-width: 1000px; }
            #fsa-api-demo button { padding: 6px 12px; cursor: pointer; margin-right: 8px; }
            #fsa-api-demo button:disabled { opacity: 0.5; cursor: not-allowed; }
            #fsa-api-demo .hint { color: #555; font-size: 13px; margin: 4px 0 10px; }
            #fsa-api-demo .status {
                margin: 10px 0; padding: 8px 12px;
                background: #f0f4ff; border-left: 4px solid #4a7cff;
                border-radius: 4px; white-space: pre-line; font-family: monospace;
            }
            #fsa-api-demo .metrics {
                background: #f6f6f6; padding: 8px; border-radius: 4px;
                font-family: monospace; font-size: 13px;
                white-space: pre-wrap; margin: 8px 0;
            }
            #fsa-api-demo .live-log {
                background: #1e1e1e; color: #d4d4d4;
                padding: 8px 12px; border-radius: 4px;
                font-family: monospace; font-size: 12px;
                white-space: pre-wrap; margin: 8px 0; min-height: 40px;
            }
            #fsa-api-demo progress { width: 280px; height: 16px; vertical-align: middle; }
            #fsa-api-demo table {
                border-collapse: collapse; width: 100%; margin-top: 8px;
            }
            #fsa-api-demo th, #fsa-api-demo td {
                border: 1px solid #ddd; padding: 4px 8px;
                font-size: 13px; text-align: left;
            }
            #fsa-api-demo th { background: #f3f3f3; }
            #fsa-api-demo tr.fsa-row { cursor: pointer; }
            #fsa-api-demo tr.fsa-row:hover { background: #f0f8ff; }
            #fsa-api-demo tr.fsa-row.selected { background: #e0f0ff; }
            #fsa-api-demo .detail {
                background: #fffbe6; padding: 8px 12px;
                border-left: 4px solid #e6c300;
                border-radius: 4px; font-family: monospace;
                font-size: 12px; white-space: pre-wrap;
                margin-top: 8px;
            }
        "#;

        let doc = web_sys::window()
            .ok_or_else(|| JsValue::from_str("no window"))?
            .document()
            .ok_or_else(|| JsValue::from_str("no document"))?;
        let style = doc.create_element("style")?;
        style.set_text_content(Some(css));
        if let Some(head) = doc.head() {
            let _ = head.append_child(&style);
        }
        Ok(())
    })
}

#[derive(Default)]
struct DemoState {
    dir: Option<FileSystemDirectoryHandle>,
    report: Option<AuditReport>,
    cancel_requested: bool,
    running: bool,
    scan_ms: f64,
}

#[derive(Clone)]
struct Ui {
    pick_btn: HtmlButtonElement,
    rescan_btn: HtmlButtonElement,
    cancel_btn: HtmlButtonElement,
    perm_btn: HtmlButtonElement,
    export_btn: HtmlButtonElement,
    progress: HtmlProgressElement,
    progress_label: HtmlElement,
    status: HtmlElement,
    live_log: HtmlElement,
    metrics: HtmlElement,
    table_wrap: HtmlElement,
    detail: HtmlElement,
}

pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;

    let section = doc.create_element("section")?;
    section.set_id("fsa-api-demo");

    let h2 = doc.create_element("h2")?;
    h2.set_text_content(Some("19. File System Access API — аудит папки"));
    section.append_child(&h2)?;

    let supported = fsa_api_pickers::is_supported();
    let status_bar = doc.create_element("div")?;
    status_bar.set_attribute("class", "status")?;
    let text: String = if supported {
        "✅ FSA доступен. Выберите папку для аудита.".to_string()
    } else {
        "⚠️ FSA не поддерживается в этом браузере. Демо недоступно.".to_string()
    };
    status_bar.set_text_content(Some(&text));
    section.append_child(&status_bar)?;

    // Кнопки
    let controls = doc.create_element("div")?;
    let pick_btn = make_button(&doc, "Выбрать папку", "fsa-pick-btn")?;
    let rescan_btn = make_button(&doc, "Повторно", "fsa-rescan-btn")?;
    let cancel_btn = make_button(&doc, "Отмена", "fsa-cancel-btn")?;
    let perm_btn = make_button(&doc, "Запросить readwrite", "fsa-perm-btn")?;
    let export_btn = make_button(&doc, "Экспорт в CSV", "fsa-export-btn")?;

    rescan_btn.set_disabled(true);
    cancel_btn.set_disabled(true);
    perm_btn.set_disabled(true);
    export_btn.set_disabled(true);
    if !supported {
        pick_btn.set_disabled(true);
    }

    controls.append_child(&pick_btn)?;
    controls.append_child(&rescan_btn)?;
    controls.append_child(&cancel_btn)?;
    controls.append_child(&perm_btn)?;
    controls.append_child(&export_btn)?;
    section.append_child(&controls)?;

    // Прогресс
    let progress_row = doc.create_element("div")?;
    let progress = doc
        .create_element("progress")?
        .dyn_into::<HtmlProgressElement>()
        .map_err(|_| JsValue::from_str("не HtmlProgressElement"))?;
    progress.set_max(1.0);
    progress.set_value(0.0);

    let progress_label: HtmlElement = doc
        .create_element("span")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement (span)"))?;
    progress_label.set_text_content(Some("0%"));

    progress_row.append_child(&progress)?;
    progress_row.append_child(&progress_label)?;
    section.append_child(&progress_row)?;

    let status: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement (div)"))?;
    status.set_text_content(Some("Готово к выбору папки."));
    section.append_child(&status)?;

    let live_log: HtmlElement = doc
        .create_element("pre")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement (pre)"))?;
    live_log.set_attribute("class", "live-log")?;
    live_log.set_text_content(Some("(лог появится при сканировании)"));
    section.append_child(&live_log)?;

    let metrics: HtmlElement = doc
        .create_element("pre")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement (pre)"))?;
    metrics.set_attribute("class", "metrics")?;
    section.append_child(&metrics)?;

    let table_wrap: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement (div)"))?;
    section.append_child(&table_wrap)?;

    let detail: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement (div)"))?;
    detail.set_attribute("class", "detail")?;
    detail.set_text_content(Some("(клик по строке — полный SHA-256 файла)"));
    section.append_child(&detail)?;

    body.append_child(&section)?;

    let ui = Ui {
        pick_btn,
        rescan_btn,
        cancel_btn,
        perm_btn,
        export_btn,
        progress,
        progress_label,
        status,
        live_log,
        metrics,
        table_wrap,
        detail,
    };
    let state = Rc::new(RefCell::new(DemoState::default()));

    wire_pick(&ui, state.clone());
    wire_rescan(&ui, state.clone());
    wire_cancel(&ui, state.clone());
    wire_permission(&ui, state.clone());
    wire_export(&ui, state.clone());
    wire_table_click(&ui, state);

    Ok(())
}

fn make_button(
    doc: &web_sys::Document,
    text: &str,
    id: &str,
) -> Result<HtmlButtonElement, JsValue> {
    let b = doc.create_element("button")?.dyn_into::<HtmlButtonElement>()?;
    b.set_id(id);
    b.set_text_content(Some(text));
    Ok(b)
}

fn wire_pick(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            match fsa_api_pickers::pick_directory("read").await {
                Ok(Some(dir)) => {
                    state.borrow_mut().dir = Some(dir);
                    ui.perm_btn.set_disabled(false);
                    ui.rescan_btn.set_disabled(false);
                    run_audit(ui, state).await;
                }
                Ok(None) => {
                    ui.status.set_text_content(Some("Выбор отменён."));
                }
                Err(e) => {
                    let msg: String = format!("Ошибка выбора папки: {:?}", e);
                    ui.status.set_text_content(Some(&msg));
                }
            }
        });
    });
    ui.pick_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_rescan(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let has_dir = state.try_borrow().map(|s| s.dir.is_some()).unwrap_or(false);
            if !has_dir {
                ui.status.set_text_content(Some("Сначала выберите папку."));
                return;
            }
            run_audit(ui, state).await;
        });
    });
    ui.rescan_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_cancel(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        if let Ok(mut s) = state.try_borrow_mut() {
            s.cancel_requested = true;
        }
        ui_c.status.set_text_content(Some("Запрошена отмена…"));
        ui_c.cancel_btn.set_disabled(true);
    });
    ui.cancel_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_permission(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let dir_opt = state
                .try_borrow()
                .ok()
                .and_then(|s| s.dir.as_ref().map(|d| d.clone()));
            let dir = match dir_opt {
                Some(d) => d,
                None => {
                    ui.status
                        .set_text_content(Some("Сначала выберите папку."));
                    return;
                }
            };
            ui.status
                .set_text_content(Some("Запрос readwrite-разрешения…"));

            let dir_js: &JsValue = dir.as_ref();
            match fsa_api_permissions::request_permission(dir_js, "readwrite").await {
                Ok(PermissionState::Granted) => {
                    ui.status.set_text_content(Some(
                        "✅ Разрешение readwrite получено. Можно писать в папку.",
                    ));
                }
                Ok(PermissionState::Denied) => {
                    ui.status
                        .set_text_content(Some("❌ Пользователь отказал в readwrite."));
                }
                Ok(PermissionState::Prompt) => {
                    ui.status.set_text_content(Some(
                        "⚠️ Браузер не выдал разрешение — запрос отклонён без ответа.",
                    ));
                }
                Err(e) => {
                    let msg: String = format!("Ошибка запроса разрешения: {:?}", e);
                    ui.status.set_text_content(Some(&msg));
                }
            }
        });
    });
    ui.perm_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_export(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move { run_export(ui, state).await });
    });
    ui.export_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_table_click(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut(web_sys::MouseEvent)>::new(
        move |ev: web_sys::MouseEvent| {
            let target = match ev.target() {
                Some(t) => t,
                None => return,
            };
            let el: web_sys::Element = match target.dyn_into() {
                Ok(e) => e,
                Err(_) => return,
            };
            let row = match el.closest("tr.fsa-row").ok().flatten() {
                Some(r) => r,
                None => return,
            };
            let path = row.get_attribute("data-path").unwrap_or_default();
            if path.is_empty() {
                return;
            }

            if let Ok(list) = ui_c.table_wrap.query_selector_all("tr.fsa-row") {
                for i in 0..list.length() {
                    if let Some(node) = list.item(i) {
                        if let Ok(e) = node.dyn_into::<web_sys::Element>() {
                            let _ = e.class_list().remove_1("selected");
                        }
                    }
                }
            }
            let _ = row.class_list().add_1("selected");

            let ui = ui_c.clone();
            let state = state.clone();
            spawn_local(async move {
                ui.detail.set_text_content(Some("Чтение файла…"));
                let dir_opt = state
                    .try_borrow()
                    .ok()
                    .and_then(|s| s.dir.as_ref().map(|d| d.clone()));
                let dir = match dir_opt {
                    Some(d) => d,
                    None => {
                        ui.detail.set_text_content(Some("Папка потеряна."));
                        return;
                    }
                };
                match compute_full_hash(&dir, &path).await {
                    Ok((size, hash)) => {
                        let msg: String = format!(
                            "Путь:        {path}\n\
                             Размер:      {size} байт\n\
                             SHA-256:     {hash}",
                        );
                        ui.detail.set_text_content(Some(&msg));
                    }
                    Err(e) => {
                        let msg: String = format!("Не удалось прочитать файл: {:?}", e);
                        ui.detail.set_text_content(Some(&msg));
                    }
                }
            });
        },
    );
    ui.table_wrap
        .set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

async fn run_audit(ui: Ui, state: Rc<RefCell<DemoState>>) {
    if state.try_borrow().map(|s| s.running).unwrap_or(true) {
        return;
    }
    state.borrow_mut().running = true;
    state.borrow_mut().cancel_requested = false;

    let dir = match state
        .try_borrow()
        .ok()
        .and_then(|s| s.dir.as_ref().map(|d| d.clone()))
    {
        Some(d) => d,
        None => {
            ui.status.set_text_content(Some("Папка не выбрана."));
            state.borrow_mut().running = false;
            return;
        }
    };

    ui.pick_btn.set_disabled(true);
    ui.rescan_btn.set_disabled(true);
    ui.export_btn.set_disabled(true);
    ui.cancel_btn.set_disabled(false);
    ui.progress.set_value(0.0);
    ui.progress_label.set_text_content(Some("0%"));
    ui.metrics.set_text_content(Some(""));
    ui.live_log
        .set_text_content(Some("(лог появится при сканировании)"));
    ui.detail
        .set_text_content(Some("(клик по строке — полный SHA-256 файла)"));

    let dir_js: &JsValue = dir.as_ref();
    let perm_text: String =
        match fsa_api_permissions::query_permission(dir_js, "read").await {
            Ok(PermissionState::Granted) => "granted".to_string(),
            Ok(PermissionState::Denied) => "denied".to_string(),
            Ok(PermissionState::Prompt) => "prompt".to_string(),
            Err(e) => format!("ошибка: {:?}", e),
        };
    let dir_name = dir.name();
    let header: String = format!("Папка: {dir_name} · permission: {perm_text}");
    ui.status.set_text_content(Some(&header));

    let mut log_lines: Vec<String> = Vec::with_capacity(LIVE_LOG_MAX);
    let ui_progress = ui.progress.clone();
    let ui_progress_label = ui.progress_label.clone();
    let ui_log = ui.live_log.clone();
    let limit = SCAN_LIMIT;

    let on_file = move |count: usize, path: &str, size: f64| {
        let frac = (count as f64 / limit as f64).min(1.0);
        ui_progress.set_value(frac);
        ui_progress_label.set_text_content(Some(&format!("{:.0}%", frac * 100.0)));

        let line: String = format!("[{:>3}] {} · {:.1} КБ", count, path, size / 1024.0);
        log_lines.push(line);
        if log_lines.len() > LIVE_LOG_MAX {
            log_lines.remove(0);
        }
        let text: String = log_lines.join("\n");
        ui_log.set_text_content(Some(&text));
    };

    let state_cb = state.clone();
    let is_cancelled = move || {
        state_cb
            .try_borrow()
            .map(|s| s.cancel_requested)
            .unwrap_or(true)
    };

    ui.status
        .set_text_content(Some(&format!("{header}\nСканирование…")));
    let t0 = js_sys::Date::now();

    let report_result = fsa_api_project::audit_directory(
        &dir,
        SCAN_LIMIT,
        on_file,
        is_cancelled,
    )
    .await;

    let elapsed = js_sys::Date::now() - t0;
    state.borrow_mut().scan_ms = elapsed;

    let report = match report_result {
        Ok(r) => r,
        Err(e) => {
            let msg: String = format!("{header}\nОшибка сканирования: {:?}", e);
            finish_with_message(&ui, &state, &msg);
            return;
        }
    };

    let metrics_text: String = build_metrics(&report, elapsed);
    ui.metrics.set_text_content(Some(&metrics_text));

    if let Err(e) = render_table(&ui.table_wrap, &report) {
        let msg: String = format!("{header}\nОшибка отрисовки таблицы: {:?}", e);
        finish_with_message(&ui, &state, &msg);
        return;
    }

    state.borrow_mut().report = Some(report);
    state.borrow_mut().running = false;

    ui.progress.set_value(1.0);
    ui.progress_label.set_text_content(Some("100%"));
    let footer: String = format!(
        "{header}\nГотово за {:.2} с. Можно экспортировать CSV \
         или кликнуть по строке для полного SHA-256.",
        elapsed / 1000.0,
    );
    ui.status.set_text_content(Some(&footer));
    ui.pick_btn.set_disabled(false);
    ui.rescan_btn.set_disabled(false);
    ui.cancel_btn.set_disabled(true);
    ui.export_btn.set_disabled(false);
}

async fn run_export(ui: Ui, state: Rc<RefCell<DemoState>>) {
    let csv_opt: Option<String> = {
        match state.try_borrow() {
            Ok(s) => s.report.as_ref().map(fsa_api_project::report_to_csv),
            Err(_) => None,
        }
    };
    let csv = match csv_opt {
        Some(c) => c,
        None => {
            ui.status.set_text_content(Some("Нет отчёта для экспорта."));
            return;
        }
    };

    ui.export_btn.set_disabled(true);
    ui.status.set_text_content(Some("Сохранение отчёта…"));

    match fsa_api_pickers::save_file("audit.csv").await {
        Ok(Some(handle)) => match fsa_api_writer::write_all_text(&handle, &csv).await {
            Ok(name) => {
                let msg: String = format!("Отчёт сохранён: {name} ({} байт)", csv.len());
                ui.status.set_text_content(Some(&msg));
            }
            Err(e) => {
                let msg: String = format!("Ошибка записи: {e:?}");
                ui.status.set_text_content(Some(&msg));
            }
        },
        Ok(None) => {
            ui.status.set_text_content(Some("Сохранение отменено."));
        }
        Err(e) => {
            let msg: String = format!("Ошибка диалога: {e:?}");
            ui.status.set_text_content(Some(&msg));
        }
    }
    ui.export_btn.set_disabled(false);
}

fn finish_with_message(ui: &Ui, state: &Rc<RefCell<DemoState>>, message: &str) {
    ui.status.set_text_content(Some(message));
    ui.pick_btn.set_disabled(false);
    ui.rescan_btn.set_disabled(false);
    ui.cancel_btn.set_disabled(true);
    ui.export_btn.set_disabled(true);
    if let Ok(mut s) = state.try_borrow_mut() {
        s.running = false;
    }
}

async fn resolve_file_handle(
    root: &FileSystemDirectoryHandle,
    parts: &[&str],
) -> Result<Option<FileSystemFileHandle>, JsValue> {
    if parts.is_empty() {
        return Ok(None);
    }
    let mut dir = root.clone();
    for (i, part) in parts.iter().enumerate() {
        if i + 1 == parts.len() {
            let fh = fsa_api_directory::get_file_handle(&dir, part, false).await?;
            return Ok(Some(fh));
        }
        dir = fsa_api_directory::get_directory_handle(&dir, part, false).await?;
    }
    Ok(None)
}

async fn compute_full_hash(
    root: &FileSystemDirectoryHandle,
    path: &str,
) -> Result<(usize, String), JsValue> {
    let parts: Vec<&str> = path.split('/').collect();
    let handle = resolve_file_handle(root, &parts)
        .await?
        .ok_or_else(|| JsValue::from_str("файл не найден"))?;
    let file = crate::fsa_api_file::get_file(&handle).await?;
    let bytes = crate::fsa_api_reader::read_bytes(&file).await?;

    use sha2::{Digest, Sha256};
    let mut h = Sha256::new();
    h.update(&bytes);
    let hash_hex: String = hex::encode(h.finalize());
    Ok((bytes.len(), hash_hex))
}

/// Строка-примечание о лимите — отдельная функция,
/// чтобы rust-analyzer не выводил тип `format!` по месту.
fn format_limit_note(truncated: bool) -> String {
    if truncated {
        return format!(" · лимит {} достигнут", SCAN_LIMIT);
    }
    String::new()
}

fn build_metrics(report: &AuditReport, elapsed_ms: f64) -> String {
    let total_size: f64 = report.rows.iter().map(|r| r.size).sum();
    let seconds = (elapsed_ms / 1000.0).max(0.001);
    let files_per_sec = report.rows.len() as f64 / seconds;
    let mb_per_sec = total_size / 1_048_576.0 / seconds;

    let mut by_cat: HashMap<&'static str, (usize, f64)> = HashMap::new();
    for row in &report.rows {
        let cat = category_of_mime(&row.mime);
        let entry = by_cat.entry(cat).or_insert((0, 0.0));
        entry.0 += 1;
        entry.1 += row.size;
    }

    let limit_note: String = format_limit_note(report.truncated);

    let mut lines: Vec<String> = Vec::with_capacity(4);
    lines.push(format!(
        "Найдено файлов: {} · общий размер: {:.2} МБ{}",
        report.rows.len(),
        total_size / 1_048_576.0,
        limit_note,
    ));
    lines.push(format!(
        "Скорость:       {:.2} с · {:.1} файлов/с · {:.2} МБ/с",
        seconds, files_per_sec, mb_per_sec,
    ));

    if !by_cat.is_empty() {
        let mut cats: Vec<(&'static str, usize, f64)> = by_cat
            .into_iter()
            .map(|(k, (n, sz))| (k, n, sz))
            .collect();
        cats.sort_by(|a, b| b.2.partial_cmp(&a.2).unwrap_or(std::cmp::Ordering::Equal));
        let mut parts: Vec<String> = Vec::with_capacity(cats.len());
        for (cat, n, sz) in cats {
            parts.push(format!("{cat}: {n} ({:.1} КБ)", sz / 1024.0));
        }
        lines.push(format!("По типам:       {}", parts.join(" · ")));
    }

    let result: String = lines.join("\n");
    result
}

/// Реализована через цепочку `if { return …; }` — единственное
/// хвостовое значение, чтобы rust-analyzer не путался в типах.
fn category_of_mime(mime: &str) -> &'static str {
    if mime.starts_with("image/") {
        return "Изображения";
    }
    if mime.starts_with("video/") {
        return "Видео";
    }
    if mime.starts_with("audio/") {
        return "Аудио";
    }
    if mime.starts_with("text/") {
        return "Текст";
    }
    if mime == "application/pdf" {
        return "PDF";
    }
    if mime == "application/zip" || mime == "application/x-tar" || mime.contains("compressed") {
        return "Архивы";
    }
    if mime.is_empty() {
        return "Без типа";
    }
    "Прочее"
}

fn render_table(wrap: &HtmlElement, report: &AuditReport) -> Result<(), JsValue> {
    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    while let Some(child) = wrap.first_child() {
        let _ = wrap.remove_child(&child);
    }

    if report.rows.is_empty() {
        let empty = doc.create_element("div")?;
        empty.set_text_content(Some("Файлы не найдены."));
        wrap.append_child(&empty)?;
        return Ok(());
    }

    let table = doc.create_element("table")?;
    let thead = doc.create_element("thead")?;
    let hdr_row = doc.create_element("tr")?;
    for h in ["Путь", "Размер (Б)", "MIME", "SHA-256 prefix"] {
        let th = doc.create_element("th")?;
        th.set_text_content(Some(h));
        hdr_row.append_child(&th)?;
    }
    thead.append_child(&hdr_row)?;
    table.append_child(&thead)?;

    let tbody = doc.create_element("tbody")?;
    for row in &report.rows {
        let tr = doc.create_element("tr")?;
        tr.set_attribute("class", "fsa-row")?;
        tr.set_attribute("data-path", &row.path)?;

        let c_path: String = row.path.clone();
        let c_size: String = format!("{:.0}", row.size);
        let c_mime: String = if row.mime.is_empty() {
            "—".to_string()
        } else {
            row.mime.clone()
        };
        let c_hash: String = row.hash_prefix.clone();

        for cell in [c_path, c_size, c_mime, c_hash] {
            let td = doc.create_element("td")?;
            td.set_text_content(Some(cell.as_str()));
            tr.append_child(&td)?;
        }
        tbody.append_child(&tr)?;
    }
    table.append_child(&tbody)?;
    wrap.append_child(&table)?;
    Ok(())
}
```

---

## Дельта `Cargo.toml`

В `[dependencies.web-sys]` → `features = [...]` **добавлено**:

```toml
    # 19. File System Access API
    "FileSystemHandle",
    "FileSystemFileHandle",
    "FileSystemDirectoryHandle",
    "FileSystemWritableFileStream",
    "FileSystemCreateWritableOptions",
    "FileSystemGetFileOptions",
    "FileSystemGetDirectoryOptions",
    "FileSystemRemoveOptions",
```

## Дельта `src/lib.rs`

```rust
pub mod fsa_api_handle;
pub mod fsa_api_file;
pub mod fsa_api_directory;
pub mod fsa_api_writable;
pub mod fsa_api_pickers;
pub mod fsa_api_permissions;
pub mod fsa_api_reader;
pub mod fsa_api_writer;
pub mod fsa_api_large;
pub mod fsa_api_project;
```

---

## Что демонстрация показывает

- **Directory picker** с `mode: "read"`.
- **Permissions** через `queryPermission` + отдельная кнопка `requestPermission`.
- **Обход директории** с живым прогрессом, live-логом и отменой.
- **SHA-256 префиксов** через `File.slice` + `sha2`.
- **Отчёт в CSV** через save picker.
- **Повторный скан без picker'а** — закэшированный handle.
- **Метрики**: время, файлов/с, МБ/с.
- **Разбивка по типам MIME** — «Изображения: 42 · Текст: 3».
- **Интерактивная таблица** — клик по строке → полный SHA-256.
- **Идемпотентность удаления** — `NotFoundError` → `Ok(())`.

## Правила проекта в главе 19

1. **`app.rs` содержит только демо текущей подглавы.** Секция 19 не пересекается с 18.15.
2. **Async-вызовы FSA через `Reflect::get` + `Function::call*`** — независимость от версии web-sys.
3. **`FileSystemDirectoryHandle.values()` — `AsyncIterator`, не `Promise`.**
4. **Обход директории — итеративный через `VecDeque`**, не рекурсия.
5. **Yield 16 мс** во всех длительных async-операциях.
6. **Отмена через флаг** в общем `Rc<RefCell<State>>`, проверяется в hot loop.
7. **`close()` обязателен** для `FileSystemWritableFileStream`.
8. **Отмена picker-диалога — не ошибка** (`Ok(None)`, не `Err`).
9. **`format!` в tail-позиции `if/else` или `match`-arm** — выносить в функцию с `-> String` (`format_limit_note`).
10. **`format!` в `category_of_mime`** — цепочка `if { return …; }` с единственным хвостом.
11. **Пользователь видит имена файлов, а не handle'ы.**
12. **`SCAN_LIMIT` сообщается явно** в метриках и CSV.
13. **Идемпотентность удаления** — правило проекта: `NotFoundError` → `Ok(())`.
14. **CSS инжектится один раз** через `thread_local!`.
15. **`cargo check` без предупреждений** — ни `dead_code`, ни `unused_imports`.