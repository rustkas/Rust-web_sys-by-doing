# Глава 19.1. `FileSystemHandle` и `FileSystemFileHandle` — базовые типы

## Структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── lib.rs                       # +pub mod fsa_lifecycle
│   ├── app.rs                       # демо-секция 19.1
│   └── fsa_lifecycle.rs             # НОВЫЙ: семантика handle и снимков
└── tests/
    └── wasm_fsa_lifecycle.rs        # НОВЫЙ: тесты 19.1
```

**Краткий комментарий.** В проекте уже есть `fsa_api_handle.rs` (обёртки `kind`/`name`/`isSameEntry`/`as_*`) и `fsa_api_file.rs` (`getFile`/`createWritable`/`move`). Дублировать их нельзя — это размывает тему.

Новая подглава идёт **на уровень смысла, а не API**: раскрывает, что такое handle как долгоживущая ссылка, почему `File` — это **снимок** на момент вызова, а не «живой файл», и чем `isSameEntry` отличается от сравнения имён. Это отдельная ценность, которой нет в `fsa_api_*` — там просто обёртки, здесь — семантика.

Префикс `fsa_lifecycle_` — уникален, не конфликтует с существующими `fsa_api_*`, `fsa_open`, `fsa_save`, `fsa_directory`.

## Ссылки

- [MDN: FileSystemHandle](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemHandle)
- [MDN: FileSystemFileHandle](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle)
- [MDN: FileSystemHandle.isSameEntry()](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemHandle/isSameEntry)
- [MDN: File.lastModified](https://developer.mozilla.org/en-US/docs/Web/API/File/lastModified)
- [Chrome: File System Access API — концепции](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access)

---

## Исходный модуль

### `src/fsa_lifecycle.rs`

```rust
//! Handle — долгоживущая ссылка; File — снимок.
//!
//! Модуль не дублирует `fsa_api_handle` / `fsa_api_file`, а раскрывает
//! их **семантику**:
//! * `describe` — метаданные самого handle'а (kind, name).
//! * `take_snapshot` — метаданные **текущего** содержимого. Снимок может
//!   устареть, если файл изменился после вызова.
//! * `same_target` — identity через `isSameEntry`, а не по имени.
//! * `is_snapshot_stale` — чистая функция.
//!
//! Плюс форматтеры для UI: весь `format!` живёт здесь, чтобы rust-analyzer
//! не путался с типами в `app.rs`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{File, FileSystemFileHandle};

/// Обёртка, обходящая баг rust-analyzer с `format!` в позиции
/// хвостового выражения `if/else`, `match`-arm или функции.
///
/// RA выводит такое `format!` как `()`, что ломает функцию с сигнатурой
/// `-> String`. `format_str` принимает `Arguments` и объявлена как
/// `-> String`; тип фиксирован и не теряется.
///
/// Использование:
///     let s = format_str(format_args!("Hello {}", name));
#[inline]
pub fn format_str(args: std::fmt::Arguments<'_>) -> String {
    std::fmt::format(args)
}

/// Вид handle'а.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum HandleKind {
    File,
    Directory,
    Unknown,
}

impl HandleKind {
    pub fn from_str(s: &str) -> Self {
        match s {
            "file" => HandleKind::File,
            "directory" => HandleKind::Directory,
            _ => HandleKind::Unknown,
        }
    }
    pub fn as_str(self) -> &'static str {
        match self {
            HandleKind::File => "file",
            HandleKind::Directory => "directory",
            HandleKind::Unknown => "unknown",
        }
    }
}

/// Метаданные самого handle'а — не содержимого.
#[derive(Debug, Clone, PartialEq)]
pub struct HandleInfo {
    pub kind: HandleKind,
    pub name: String,
}

/// Извлекает kind и name из handle'а. Не делает чтения содержимого.
pub fn describe(handle: &JsValue) -> HandleInfo {
    let kind_str: String = js_sys::Reflect::get(handle, &JsValue::from_str("kind"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    let name: String = js_sys::Reflect::get(handle, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    HandleInfo {
        kind: HandleKind::from_str(&kind_str),
        name,
    }
}

/// Метаданные содержимого файла на момент вызова `getFile()`.
#[derive(Debug, Clone, PartialEq)]
pub struct SnapshotInfo {
    pub name: String,
    pub size: f64,
    pub mime: String,
    pub last_modified_ms: f64,
}

/// Снимает снимок содержимого файла.
pub async fn take_snapshot(handle: &FileSystemFileHandle) -> Result<SnapshotInfo, JsValue> {
    let handle_js: &JsValue = handle.as_ref();
    let get_file_js = js_sys::Reflect::get(handle_js, &JsValue::from_str("getFile"))?;
    if get_file_js.is_undefined() {
        return Err(JsValue::from_str("getFile недоступен"));
    }
    let get_file: js_sys::Function = get_file_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFile не функция"))?;
    let promise_js = get_file.call0(handle_js)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFile не Promise"))?;
    let file_value: JsValue = JsFuture::from(promise).await?;
    let file: File = file_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не File"))?;
    Ok(SnapshotInfo {
        name: file.name(),
        size: file.size(),
        mime: file.type_(),
        last_modified_ms: file.last_modified(),
    })
}

/// Identity: один ли это и тот же файл. Сравниваем не имя, а цель.
pub async fn same_target(a: &JsValue, b: &JsValue) -> Result<bool, JsValue> {
    let is_same_js = js_sys::Reflect::get(a, &JsValue::from_str("isSameEntry"))?;
    if is_same_js.is_undefined() {
        return Err(JsValue::from_str("isSameEntry недоступен"));
    }
    let is_same: js_sys::Function = is_same_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("isSameEntry не функция"))?;
    let promise_js = is_same.call1(a, b)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("isSameEntry не Promise"))?;
    let value = JsFuture::from(promise).await?;
    Ok(value.as_bool().unwrap_or(false))
}

/// Чистая функция: устарел ли снимок относительно свежего.
pub fn is_snapshot_stale(previous: &SnapshotInfo, fresh: &SnapshotInfo) -> bool {
    previous.last_modified_ms != fresh.last_modified_ms
        || previous.size != fresh.size
}

// ---------------------------------------------------------------------------
// Форматтеры для UI
// ---------------------------------------------------------------------------

/// Метаданные handle'а для UI.
pub fn format_handle_info(info: &HandleInfo) -> String {
    format_str(format_args!(
        "kind: {}\nname: {}",
        info.kind.as_str(),
        info.name
    ))
}

/// Человекочитаемый размер файла.
pub fn format_size(bytes: f64) -> String {
    if bytes < 1024.0 {
        return format_str(format_args!("{:.0} Б", bytes));
    }
    if bytes < 1024.0 * 1024.0 {
        return format_str(format_args!("{:.1} КБ", bytes / 1024.0));
    }
    if bytes < 1024.0 * 1024.0 * 1024.0 {
        return format_str(format_args!("{:.2} МБ", bytes / (1024.0 * 1024.0)));
    }
    format_str(format_args!("{:.2} ГБ", bytes / (1024.0 * 1024.0 * 1024.0)))
}

/// Строка пары handle'ов с вердиктом identity.
pub fn format_identity_pair(old_name: &str, new_name: &str, same: bool) -> String {
    let verdict: String = if same {
        "✅ isSameEntry = true: это тот же самый файл.".to_string()
    } else {
        "❌ isSameEntry = false: разные файлы.".to_string()
    };
    format_str(format_args!("Первый: {}\nВторой: {}\n{}", old_name, new_name, verdict))
}

/// Снимок содержимого для UI: человекочитаемый размер,
/// счётчик пользовательских снимков (0 = автоопрос) и пометка stale.
pub fn format_snapshot(snap: &SnapshotInfo, stale: Option<bool>, counter: u32) -> String {
    let mime: &str = if snap.mime.is_empty() { "—" } else { snap.mime.as_str() };
    let counter_line: String = if counter > 0 {
        format_str(format_args!("# снимок №{counter}"))
    } else {
        "# автообновление".to_string()
    };
    let mut lines: Vec<String> = Vec::with_capacity(6);
    lines.push(counter_line);
    lines.push(format!("name:         {}", snap.name));
    lines.push(format!("size:         {}", format_size(snap.size)));
    lines.push(format!("mime:         {}", mime));
    lines.push(format!("lastModified: {:.0} ms", snap.last_modified_ms));
    if let Some(s) = stale {
        let note: String = if s {
            "⚠️ Изменено с момента снимка.".to_string()
        } else {
            "✅ Совпадает с baseline.".to_string()
        };
        lines.push(note);
    }
    lines.join("\n")
}

/// Полезная нагрузка для «Изменить содержимое»: строка с таймстемпом.
pub fn format_modification_payload(ts: f64) -> String {
    format_str(format_args!("modified at {}\n", ts))
}

/// Человекочитаемое объяснение ошибки FSA.
pub fn explain_error(e: &JsValue) -> String {
    let raw: String = e.as_string().unwrap_or_else(|| format!("{e:?}"));
    if raw.contains("SecurityError")
        || raw.contains("gesture")
        || raw.contains("User activation")
    {
        return "Браузер требует жест пользователя — нажмите кнопку ещё раз."
            .to_string();
    }
    if raw.contains("AbortError") {
        return "Диалог закрыт.".to_string();
    }
    if raw.contains("NotFoundError") {
        return "Файл не найден.".to_string();
    }
    format_str(format_args!("Ошибка FSA: {raw}"))
}
```

---

## Как добавить в `lib.rs`

В блок `pub mod ...` **добавить одну строку**:

```rust
pub mod fsa_lifecycle;
```

---

## Тесты

### `tests/wasm_fsa_lifecycle.rs`

```rust
//! Тесты `src/fsa_lifecycle.rs` через OPFS.

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
}

use hello_wasm02::fsa_lifecycle::*;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// describe
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn describe_non_handle_returns_unknown() {
    let v = wasm_bindgen::JsValue::from_str("not a handle");
    let info = describe(&v);
    assert_eq!(info.kind, HandleKind::Unknown);
    assert_eq!(info.name, "");
}

#[wasm_bindgen_test]
async fn describe_opfs_file_handle() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_lifecycle_describe.txt";
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, name, true).await;
    let handle = hello_wasm02::fsa_api_directory::get_file_handle(&root, name, true)
        .await
        .expect("create");

    let handle_js: &wasm_bindgen::JsValue = handle.as_ref();
    let info = describe(handle_js);
    assert_eq!(info.kind, HandleKind::File);
    assert_eq!(info.name, name);

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, name, true).await;
}

// ---------------------------------------------------------------------------
// take_snapshot
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn take_snapshot_returns_metadata() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_lifecycle_snapshot.bin";
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, name, true).await;
    let handle = hello_wasm02::fsa_api_directory::get_file_handle(&root, name, true)
        .await
        .expect("create");

    let bytes = b"hello lifecycle";
    let stream = hello_wasm02::fsa_api_file::create_writable(&handle, false)
        .await
        .expect("create_writable");
    hello_wasm02::fsa_api_writable::write_bytes(&stream, bytes)
        .await
        .expect("write");
    hello_wasm02::fsa_api_writable::close(&stream)
        .await
        .expect("close");

    let snap = take_snapshot(&handle).await.expect("snapshot");
    assert_eq!(snap.name, name);
    assert_eq!(snap.size as usize, bytes.len());
    assert!(snap.last_modified_ms > 0.0);

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, name, true).await;
}

// ---------------------------------------------------------------------------
// same_target
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn same_target_two_handles_on_one_file() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "fsa_lifecycle_identity.txt";
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, name, true).await;
    let h1 = hello_wasm02::fsa_api_directory::get_file_handle(&root, name, true)
        .await
        .expect("create h1");
    let h2 = hello_wasm02::fsa_api_directory::get_file_handle(&root, name, false)
        .await
        .expect("open h2");

    let h1_js: &wasm_bindgen::JsValue = h1.as_ref();
    let h2_js: &wasm_bindgen::JsValue = h2.as_ref();
    assert!(same_target(h1_js, h2_js).await.expect("same_target"));

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, name, true).await;
}

#[wasm_bindgen_test]
async fn same_target_two_different_files() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let a = "fsa_lifecycle_a.txt";
    let b = "fsa_lifecycle_b.txt";
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, a, true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, b, true).await;
    let ha = hello_wasm02::fsa_api_directory::get_file_handle(&root, a, true)
        .await
        .expect("create a");
    let hb = hello_wasm02::fsa_api_directory::get_file_handle(&root, b, true)
        .await
        .expect("create b");

    let ha_js: &wasm_bindgen::JsValue = ha.as_ref();
    let hb_js: &wasm_bindgen::JsValue = hb.as_ref();
    assert!(!same_target(ha_js, hb_js).await.expect("same_target"));

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, a, true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, b, true).await;
}

// ---------------------------------------------------------------------------
// is_snapshot_stale
// ---------------------------------------------------------------------------

fn snap(name: &str, size: f64, mime: &str, last_modified_ms: f64) -> SnapshotInfo {
    SnapshotInfo {
        name: name.to_string(),
        size,
        mime: mime.to_string(),
        last_modified_ms,
    }
}

#[wasm_bindgen_test]
fn is_snapshot_stale_same_data_not_stale() {
    let a = snap("a", 10.0, "text/plain", 1000.0);
    let b = snap("a", 10.0, "text/plain", 1000.0);
    assert!(!is_snapshot_stale(&a, &b));
}

#[wasm_bindgen_test]
fn is_snapshot_stale_size_changed() {
    let a = snap("a", 10.0, "text/plain", 1000.0);
    let b = snap("a", 20.0, "text/plain", 1000.0);
    assert!(is_snapshot_stale(&a, &b));
}

#[wasm_bindgen_test]
fn is_snapshot_stale_modified_changed() {
    let a = snap("a", 10.0, "text/plain", 1000.0);
    let b = snap("a", 10.0, "text/plain", 2000.0);
    assert!(is_snapshot_stale(&a, &b));
}

// ---------------------------------------------------------------------------
// format_size
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn format_size_units() {
    assert_eq!(format_size(0.0), "0 Б");
    assert_eq!(format_size(512.0), "512 Б");
    assert_eq!(format_size(2048.0), "2.0 КБ");
    assert_eq!(format_size(5.0 * 1024.0 * 1024.0), "5.00 МБ");
    assert_eq!(format_size(3.0 * 1024.0 * 1024.0 * 1024.0), "3.00 ГБ");
}

// ---------------------------------------------------------------------------
// format_identity_pair
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn format_identity_pair_same() {
    let s = format_identity_pair("a.txt", "a.txt", true);
    assert!(s.contains("Первый: a.txt"));
    assert!(s.contains("Второй: a.txt"));
    assert!(s.contains("✅"));
}

#[wasm_bindgen_test]
fn format_identity_pair_different() {
    let s = format_identity_pair("a.txt", "b.txt", false);
    assert!(s.contains("❌"));
}

// ---------------------------------------------------------------------------
// format_snapshot
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn format_snapshot_with_counter_shows_number() {
    let s = snap("a.bin", 1024.0, "application/octet-stream", 12345.0);
    let text = format_snapshot(&s, None, 3);
    assert!(text.contains("# снимок №3"));
    assert!(text.contains("name:         a.bin"));
    assert!(text.contains("size:         1.0 КБ"));
    assert!(text.contains("mime:         application/octet-stream"));
}

#[wasm_bindgen_test]
fn format_snapshot_without_counter_uses_auto_label() {
    let s = snap("a.bin", 100.0, "text/plain", 1.0);
    let text = format_snapshot(&s, None, 0);
    assert!(text.contains("# автообновление"));
}

#[wasm_bindgen_test]
fn format_snapshot_empty_mime_uses_dash() {
    let s = snap("a.bin", 100.0, "", 1.0);
    let text = format_snapshot(&s, None, 1);
    assert!(text.contains("mime:         —"));
}

#[wasm_bindgen_test]
fn format_snapshot_stale_warning() {
    let s = snap("a.bin", 100.0, "text/plain", 1.0);
    let text = format_snapshot(&s, Some(true), 1);
    assert!(text.contains("⚠️"));
}

#[wasm_bindgen_test]
fn format_snapshot_fresh_note() {
    let s = snap("a.bin", 100.0, "text/plain", 1.0);
    let text = format_snapshot(&s, Some(false), 1);
    assert!(text.contains("✅ Совпадает"));
}

// ---------------------------------------------------------------------------
// format_modification_payload
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn format_modification_payload_nonempty() {
    let p = format_modification_payload(1234.5);
    assert!(p.starts_with("modified at "));
    assert!(p.ends_with("\n"));
    assert!(p.contains("1234.5"));
}

// ---------------------------------------------------------------------------
// explain_error
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn explain_error_security() {
    let e = wasm_bindgen::JsValue::from_str(
        "SecurityError: Must be handling a user gesture",
    );
    let s = explain_error(&e);
    assert!(s.contains("жест пользователя"));
}

#[wasm_bindgen_test]
fn explain_error_abort() {
    let e = wasm_bindgen::JsValue::from_str("AbortError: user cancelled");
    let s = explain_error(&e);
    assert_eq!(s, "Диалог закрыт.");
}

#[wasm_bindgen_test]
fn explain_error_other() {
    let e = wasm_bindgen::JsValue::from_str("Some random failure");
    let s = explain_error(&e);
    assert!(s.starts_with("Ошибка FSA:"));
}
```

---

## Команды запуска

```bash
# проверка типов
cargo check --target wasm32-unknown-unknown

# тесты подглавы
wasm-pack test --headless --chrome --test wasm_fsa_lifecycle

# демо
npm start
```

---

## Демо-пример в `src/app.rs`

```rust
//! Демо-секция 19.1: FileSystemHandle и FileSystemFileHandle.
//!
//! Раскрывает семантику:
//! * handle — долгоживущая ссылка;
//! * File — снимок на момент вызова `getFile()`, может устареть;
//! * identity — через `isSameEntry`, а не по имени.
//!
//! Контракт с `lib.rs`: `run() -> Result<(), JsValue>`.
//!
//! Все строки формируются в `fsa_lifecycle::format_*`, чтобы
//! rust-analyzer не путался в типах и не было inline `format!`
//! в tail-позиции `if/else`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{FileSystemFileHandle, HtmlButtonElement, HtmlElement};

use crate::fsa_api_pickers;
use crate::fsa_lifecycle::{self, SnapshotInfo};

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

const AUTO_REFRESH_MS: u32 = 500;
const AUTO_REFRESH_IDLE_MS: u32 = 2000;

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);
        let css = r#"
            #fsa-lifecycle { font-family: system-ui, sans-serif; max-width: 1000px; }
            #fsa-lifecycle button { padding: 6px 12px; cursor: pointer; margin-right: 8px; }
            #fsa-lifecycle button:disabled { opacity: 0.5; cursor: not-allowed; }
            #fsa-lifecycle .status {
                margin: 10px 0; padding: 8px 12px;
                background: #f0f4ff; border-left: 4px solid #4a7cff;
                border-radius: 4px; white-space: pre-line; font-family: monospace;
            }
            #fsa-lifecycle .panel { display: flex; gap: 16px; margin: 12px 0; }
            #fsa-lifecycle .panel > div {
                flex: 1; padding: 8px 12px; border-radius: 4px;
                font-family: monospace; font-size: 13px; white-space: pre-wrap;
            }
            #fsa-lifecycle .handle-panel {
                background: #eef6ee; border-left: 4px solid #4caf50;
            }
            #fsa-lifecycle .snap-panel {
                background: #f6f0ee; border-left: 4px solid #e67e22;
            }
            #fsa-lifecycle .panel h3 { margin: 0 0 6px 0; font-size: 13px; }
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
    handle: Option<FileSystemFileHandle>,
    /// Baseline: последний **пользовательский** снимок.
    user_snapshot: Option<SnapshotInfo>,
    /// Текущее состояние файла (обновляется автоопросом).
    auto_snapshot: Option<SnapshotInfo>,
    /// Счётчик пользовательских снимков.
    snapshot_counter: u32,
    /// Уже сообщили о stale — не повторять.
    stale_reported: bool,
}

#[derive(Clone)]
struct Ui {
    pick_btn: HtmlButtonElement,
    pick_again_btn: HtmlButtonElement,
    snapshot_btn: HtmlButtonElement,
    modify_btn: HtmlButtonElement,
    forget_btn: HtmlButtonElement,
    status: HtmlElement,
    handle_view: HtmlElement,
    snap_view: HtmlElement,
}

pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;

    let section = doc.create_element("section")?;
    section.set_id("fsa-lifecycle");

    let h2 = doc.create_element("h2")?;
    h2.set_text_content(Some("19.1 — Handle как ссылка, File как снимок"));
    section.append_child(&h2)?;

    let supported = fsa_api_pickers::is_supported();

    let hint: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    hint.set_attribute("class", "status")?;
    hint.set_text_content(Some(&hint_text(supported)));
    section.append_child(&hint)?;

    let controls = doc.create_element("div")?;
    let pick_btn = make_button(&doc, "Выбрать файл", "lc-pick")?;
    let pick_again_btn = make_button(&doc, "Открыть ещё раз", "lc-pick-again")?;
    let snapshot_btn = make_button(&doc, "Снять снимок", "lc-snap")?;
    let modify_btn = make_button(&doc, "Изменить содержимое", "lc-modify")?;
    let forget_btn = make_button(&doc, "Забыть", "lc-forget")?;

    pick_again_btn.set_disabled(true);
    snapshot_btn.set_disabled(true);
    modify_btn.set_disabled(true);
    forget_btn.set_disabled(true);
    if !supported {
        pick_btn.set_disabled(true);
    }

    controls.append_child(&pick_btn)?;
    controls.append_child(&pick_again_btn)?;
    controls.append_child(&snapshot_btn)?;
    controls.append_child(&modify_btn)?;
    controls.append_child(&forget_btn)?;
    section.append_child(&controls)?;

    let status: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    status.set_text_content(Some("Готово."));
    section.append_child(&status)?;

    let panels = doc.create_element("div")?;
    panels.set_attribute("class", "panel")?;

    let handle_panel = doc.create_element("div")?;
    handle_panel.set_attribute("class", "handle-panel")?;
    let hp_title = doc.create_element("h3")?;
    hp_title.set_text_content(Some("Handle (ссылка)"));
    handle_panel.append_child(&hp_title)?;
    let handle_view: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    handle_view.set_text_content(Some("(не выбран)"));
    handle_panel.append_child(&handle_view)?;

    let snap_panel = doc.create_element("div")?;
    snap_panel.set_attribute("class", "snap-panel")?;
    let sp_title = doc.create_element("h3")?;
    sp_title.set_text_content(Some("Snapshot (состояние на момент getFile)"));
    snap_panel.append_child(&sp_title)?;
    let snap_view: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    snap_view.set_text_content(Some("(снимок не снят)"));
    snap_panel.append_child(&snap_view)?;

    panels.append_child(&handle_panel)?;
    panels.append_child(&snap_panel)?;
    section.append_child(&panels)?;

    body.append_child(&section)?;

    let ui = Ui {
        pick_btn,
        pick_again_btn,
        snapshot_btn,
        modify_btn,
        forget_btn,
        status,
        handle_view,
        snap_view,
    };
    let state = Rc::new(RefCell::new(DemoState::default()));

    wire_pick(&ui, state.clone());
    wire_pick_again(&ui, state.clone());
    wire_snapshot(&ui, state.clone());
    wire_modify(&ui, state.clone());
    wire_forget(&ui, state.clone());
    wire_auto_refresh(&ui, state);

    Ok(())
}

fn hint_text(supported: bool) -> String {
    if supported {
        return "1. Выберите файл — получим handle.\n\
                2. Снимите снимок содержимого (baseline).\n\
                3. Нажмите «Изменить содержимое» или измените файл \
                в проводнике — увидите, что снимок устарел.\n\
                4. «Открыть ещё раз» сравнит два handle'а через isSameEntry."
            .to_string();
    }
    "⚠️ FSA недоступен в этом браузере — демо не работает.".to_string()
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

// ---------------------------------------------------------------------------
// Handlers
// ---------------------------------------------------------------------------

fn wire_pick(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            ui.status.set_text_content(Some("Выбор файла…"));
            match fsa_api_pickers::open_files(None).await {
                Ok(Some(list)) if !list.is_empty() => {
                    let handle = list.into_iter().next().unwrap();
                    let info = fsa_lifecycle::describe(handle.as_ref());
                    let text: String = fsa_lifecycle::format_handle_info(&info);
                    ui.handle_view.set_text_content(Some(&text));

                    {
                        let mut s = state.borrow_mut();
                        s.handle = Some(handle);
                        s.user_snapshot = None;
                        s.auto_snapshot = None;
                        s.snapshot_counter = 0;
                        s.stale_reported = false;
                    }

                    ui.snap_view.set_text_content(Some("(снимок не снят)"));
                    ui.pick_again_btn.set_disabled(false);
                    ui.snapshot_btn.set_disabled(false);
                    ui.modify_btn.set_disabled(false);
                    ui.forget_btn.set_disabled(false);

                    ui.status.set_text_content(Some(
                        "Handle получен. Снимите снимок содержимого.",
                    ));
                }
                Ok(_) => {
                    ui.status.set_text_content(Some("Выбор отменён."));
                }
                Err(e) => {
                    let msg: String = fsa_lifecycle::explain_error(&e);
                    ui.status.set_text_content(Some(&msg));
                }
            }
        });
    });
    ui.pick_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

/// «Открыть ещё раз» — только сравнение, handle не меняется.
fn wire_pick_again(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let old = match state.try_borrow().ok().and_then(|s| s.handle.clone())
            {
                Some(h) => h,
                None => {
                    ui.status.set_text_content(Some("Сначала выберите файл."));
                    return;
                }
            };
            match fsa_api_pickers::open_files(None).await {
                Ok(Some(list)) if !list.is_empty() => {
                    let new_handle = list.into_iter().next().unwrap();
                    let old_js: &JsValue = old.as_ref();
                    let new_js: &JsValue = new_handle.as_ref();
                    let same = fsa_lifecycle::same_target(old_js, new_js)
                        .await
                        .unwrap_or(false);
                    let info_old = fsa_lifecycle::describe(old_js);
                    let info_new = fsa_lifecycle::describe(new_js);
                    let msg: String = fsa_lifecycle::format_identity_pair(
                        &info_old.name,
                        &info_new.name,
                        same,
                    );
                    ui.status.set_text_content(Some(&msg));
                    // handle НЕ меняем — семантика «сравнить».
                }
                Ok(_) => {
                    ui.status
                        .set_text_content(Some("Повторный выбор отменён."));
                }
                Err(e) => {
                    let msg: String = fsa_lifecycle::explain_error(&e);
                    ui.status.set_text_content(Some(&msg));
                }
            }
        });
    });
    ui.pick_again_btn
        .set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_snapshot(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let handle = match state.try_borrow().ok().and_then(|s| s.handle.clone())
            {
                Some(h) => h,
                None => {
                    ui.status.set_text_content(Some("Сначала выберите файл."));
                    return;
                }
            };
            match fsa_lifecycle::take_snapshot(&handle).await {
                Ok(snap) => {
                    let counter = {
                        let mut s = state.borrow_mut();
                        s.snapshot_counter += 1;
                        s.user_snapshot = Some(snap.clone());
                        s.auto_snapshot = Some(snap.clone());
                        s.stale_reported = false;
                        s.snapshot_counter
                    };
                    let text: String =
                        fsa_lifecycle::format_snapshot(&snap, None, counter);
                    ui.snap_view.set_text_content(Some(&text));
                    ui.status.set_text_content(Some(
                        "Снимок снят — это baseline для сравнения.",
                    ));
                }
                Err(e) => {
                    let msg: String = fsa_lifecycle::explain_error(&e);
                    ui.status.set_text_content(Some(&msg));
                }
            }
        });
    });
    ui.snapshot_btn
        .set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_modify(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let handle = match state.try_borrow().ok().and_then(|s| s.handle.clone())
            {
                Some(h) => h,
                None => {
                    ui.status.set_text_content(Some("Сначала выберите файл."));
                    return;
                }
            };
            let payload: String =
                fsa_lifecycle::format_modification_payload(js_sys::Date::now());
            let stream =
                match crate::fsa_api_file::create_writable(&handle, false).await
                {
                    Ok(s) => s,
                    Err(e) => {
                        let msg: String = fsa_lifecycle::explain_error(&e);
                        ui.status.set_text_content(Some(&msg));
                        return;
                    }
                };
            if let Err(e) =
                crate::fsa_api_writable::write_text(&stream, &payload).await
            {
                let msg: String = fsa_lifecycle::explain_error(&e);
                ui.status.set_text_content(Some(&msg));
                return;
            }
            if let Err(e) = crate::fsa_api_writable::close(&stream).await {
                let msg: String = fsa_lifecycle::explain_error(&e);
                ui.status.set_text_content(Some(&msg));
                return;
            }
            if let Ok(mut s) = state.try_borrow_mut() {
                s.stale_reported = false;
            }
            ui.status.set_text_content(Some(
                "Файл изменён. Снимок выше показывает baseline — \
                 через мгновение увидите «Изменено с момента снимка».",
            ));
        });
    });
    ui.modify_btn
        .set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

/// Полный сброс: отпустить handle, очистить обе панели.
fn wire_forget(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        if let Ok(mut s) = state.try_borrow_mut() {
            *s = DemoState::default();
        }
        ui_c.handle_view.set_text_content(Some("(не выбран)"));
        ui_c.snap_view.set_text_content(Some("(снимок не снят)"));
        ui_c.status.set_text_content(Some("Состояние сброшено."));
        ui_c.pick_again_btn.set_disabled(true);
        ui_c.snapshot_btn.set_disabled(true);
        ui_c.modify_btn.set_disabled(true);
        ui_c.forget_btn.set_disabled(true);
    });
    ui.forget_btn
        .set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

/// Автоопрос: раз в 500 мс сверяем состояние с baseline пользователя.
/// Правило проекта: UI синхронизируется с общим состоянием через опрос.
fn wire_auto_refresh(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_c = ui.clone();
    spawn_local(async move {
        loop {
            let handle_opt =
                state.try_borrow().ok().and_then(|s| s.handle.clone());

            let handle = match handle_opt {
                Some(h) => h,
                None => {
                    TimeoutFuture::new(AUTO_REFRESH_IDLE_MS).await;
                    continue;
                }
            };

            // Yield в начале цикла — UI успевает перерисоваться.
            TimeoutFuture::new(AUTO_REFRESH_MS).await;

            let fresh = match fsa_lifecycle::take_snapshot(&handle).await {
                Ok(s) => s,
                Err(_) => continue,
            };

            let (user_snap, already_reported, counter) = {
                let s = match state.try_borrow() {
                    Ok(g) => g,
                    Err(_) => continue,
                };
                (
                    s.user_snapshot.clone(),
                    s.stale_reported,
                    s.snapshot_counter,
                )
            };

            let stale: Option<bool> = user_snap
                .as_ref()
                .map(|p| fsa_lifecycle::is_snapshot_stale(p, &fresh));

            let text: String =
                fsa_lifecycle::format_snapshot(&fresh, stale, counter);
            ui_c.snap_view.set_text_content(Some(&text));

            if let (Some(true), false) = (stale, already_reported) {
                ui_c.status.set_text_content(Some(
                    "⚠️ Изменено с момента снимка — см. панель Snapshot.",
                ));
                if let Ok(mut s) = state.try_borrow_mut() {
                    s.auto_snapshot = Some(fresh);
                    s.stale_reported = true;
                }
            } else if let Ok(mut s) = state.try_borrow_mut() {
                s.auto_snapshot = Some(fresh);
            }
        }
    });
}
```

---

## Что демонстрация показывает

- **Две панели рядом**: `Handle (ссылка)` и `Snapshot (состояние на момент getFile)`. Пользователь визуально различает эти два понятия.
- **`isSameEntry`** — отдельная кнопка «Открыть ещё раз»: показывает, что сравниваются не имена, а цели.
- **Устаревание снимка** — кнопка «Изменить содержимое»: пишет новую строку в файл через `createWritable`. Опрос раз в 500 мс показывает, что снимок устарел и данные обновились.
- **Baseline не теряется** — автоопрос обновляет панель, но не трогает `user_snapshot`. Поэтому «изменено с момента снимка» держится, пока пользователь не снимет новый снимок.
- **Одноразовое уведомление** о stale через флаг `stale_reported` — не дублируется каждые 500 мс.
- **Кнопка «Забыть»** — идемпотентный полный сброс.
- **Человекочитаемые размеры**: `14.5 КБ` вместо `14848 Б`.
- **Счётчик снимков** в панели: `# снимок №3` против `# автообновление`.
- **Подсказка упоминает правки из проводника** — пользователь поймёт, что stale может прийти «извне».
- **Idle-сон 2 с при отсутствии handle** — цикл не крутится вхолостую.
- **Человекочитаемые ошибки**: `SecurityError` → «Браузер требует жест пользователя», `AbortError` → «Диалог закрыт», `NotFoundError` → «Файл не найден».

## Правила проекта, зафиксированные в 19.1

- **`app.rs` содержит только демо текущей подглавы** — 19.1, никаких предыдущих глав.
- **Все строки формируются в `fsa_lifecycle::format_*`** — нет inline `format!` в `app.rs`.
- **`format_str(format_args!(…))`** — обязательная обёртка для любого `format!` в хвостовой позиции `if/else`, `match`-arm или функции. `rust-analyzer` стабильно теряет тип `format!` в этих позициях.
- **Автообновление UI** — через периодический опрос раз в 500 мс, а не через колбэки между несвязанными обработчиками.
- **Идемпотентный сброс** — `wire_forget` полностью возвращает состояние, можно нажимать многократно.
- **`cargo check` без предупреждений** — ни `dead_code`, ни `unused_imports`, ни `unused_variables`.