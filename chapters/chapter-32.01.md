# Глава 32.01. `ClipboardEvent` — paste без разрешений

## 32.01.1. Что вы получите в этой главе

В главе 32 мы разбирали Async Clipboard API — `navigator.clipboard.writeText()`, `navigator.clipboard.read()`, `ClipboardItem`. Это современный путь к буферу обмена. Но он **не единственный**. Существует второй, более старый и в чём-то более мощный путь — **событие `paste`** и связанный с ним `ClipboardEvent`.

За эту главу вы:

1. Разберёте, чем `paste`-событие отличается от Async API и почему в текстовых редакторах используется именно оно.
2. Напишете модуль `paste_evt`, который синхронно извлекает всё содержимое буфера обмена — plain text, HTML, URI-список, файлы — **без единого разрешения**.
3. Столкнётесь с реальной браузерной спецификой: Chromium передаёт все файлы, Firefox и Safari — только один. Разберёте, почему так и что с этим делать.
4. Соберёте демо с диагностикой `DataTransfer`, определением браузера и браузер-специфичными подсказками.
5. Поймёте, в каких сценариях `paste` незаменим, а в каких Async API — единственный вариант.

## 32.01.2. Теоретическая справка — два API для буфера обмена

Буфер обмена — системный ресурс. Web-платформа даёт к нему **два независимых пути**.

**Async Clipboard API** (`navigator.clipboard`) — про него была глава 32. Работает через промисы. Требует secure context. Для записи — user gesture, для чтения — разрешение `clipboard-read`. Возвращает `ClipboardItem[]`.

**ClipboardEvent** — события `copy`, `cut`, `paste`. Срабатывают автоматически на действия пользователя (Ctrl+C, Ctrl+X, Ctrl+V, контекстное меню). Событие содержит поле `clipboardData: DataTransfer`, из которого **синхронно** доступны все форматы буфера. Разрешение не требуется — событие и есть разрешение.

| Свойство | Async API | ClipboardEvent |
|---|---|---|
| Триггер | программный вызов | действие пользователя |
| Доступ | асинхронный (`await`) | синхронный |
| Разрешение | `clipboard-read` для чтения | не требуется |
| Форматы | `ClipboardItem` (plain, html, png) | `DataTransfer` (plain, html, uri-list, files) |
| `text/html` | с CF_HTML-обёрткой | **без** обёртки |
| Файлы ОС | недоступны | доступны через `items` |
| Инициатор | скрипт | браузер (на жест пользователя) |

**Ключевое следствие.** `paste`-событие срабатывает **только** на действие пользователя. Программно вызвать `paste` нельзя. Значит, для кнопки «Вставить из буфера» остаётся только Async API. Но при реальном Ctrl+V — `paste` даёт больше.

Почему текстовые редакторы (CodeMirror, Monaco, VS Code Web) предпочитают `paste`:

- **Синхронность.** Данные доступны внутри обработчика, без `await`.
- **Разрешения.** Чтение через `paste` не требует `clipboard-read`. Пользователь не видит диалога на каждый Ctrl+V.
- **Полнота.** `clipboardData` содержит `text/html`, `text/uri-list`, `Files` — то, чего нет в `readText()`.

## 32.01.3. Структура файлов

```
hello-wasm02/
├── src/
│   ├── paste_evt.rs          # НОВЫЙ: извлечение содержимого из paste-события
│   ├── lib.rs                # +1 модуль
│   └── app.rs                # демо — paste vs Async API
├── tests/
│   └── wasm_paste_evt.rs     # НОВЫЙ: 12 wasm-тестов
└── Cargo.toml                # без изменений
```

**Комментарий.** Модуль `paste_evt` не пересекается с `clip_core`/`clip_perms` из главы 32 ни по имени, ни по смыслу. `clip_*` работают через `navigator.clipboard`, `paste_evt` — через `ClipboardEvent.clipboardData`. Это два разных API с разными контрактами.

Все нужные `web-sys`-фичи (`DataTransfer`, `HtmlElement`, `HtmlButtonElement`, `Navigator`) уже в `Cargo.toml`. Ничего добавлять не нужно.

**Ссылки для изучения:**
- MDN, `ClipboardEvent` — https://developer.mozilla.org/en-US/docs/Web/API/ClipboardEvent
- MDN, `Element: paste event` — https://developer.mozilla.org/en-US/docs/Web/API/Element/paste_event
- MDN, `DataTransfer` — https://developer.mozilla.org/en-US/docs/Web/API/DataTransfer
- Spec, «Clipboard API and events» — https://w3c.github.io/clipboard-apis/

## 32.01.4. Исходный код `src/paste_evt.rs`

```rust
//! Событие `paste`: доступ к буферу обмена без разрешений.
//!
//! `ClipboardEvent` (события `copy`, `cut`, `paste`) — второй способ
//! доступа к буферу обмена. В отличие от Async Clipboard API:
//!
//!   • НЕ требует разрешения `clipboard-read`;
//!   • данные доступны СИНХРОННО, внутри обработчика;
//!   • сразу даёт все MIME-типы буфера: `text/plain`, `text/html`,
//!     `text/uri-list`, `Files`;
//!   • `text/html` приходит БЕЗ CF_HTML-обёртки (в отличие от Async API).
//!
//! ПРАВИЛО: `clipboardData` валиден только внутри обработчика. После
//! `return` браузер очищает `DataTransfer`. Все данные нужно извлечь
//! синхронно.
//!
//! ПРАВИЛО: файлы читаем через `clipboardData.items`, а НЕ через
//! `clipboardData.files`. На Windows Chrome часто кладёт в буфер только
//! «primary» файл; `items` может содержать больше (или тоже один —
//! см. диагностику).
//!
//! ПРАВИЛО: `web-sys` не всегда экспортирует типизированный
//! `ClipboardEvent.clipboard_data()`. Извлекаем `clipboardData` через
//! `Reflect::get` — это работает независимо от версии web-sys.

use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;

// ============================================================
//                    ПУБЛИЧНЫЕ ТИПЫ
// ============================================================

/// Метаданные файла, скопированного из ОС.
#[derive(Debug, Clone, PartialEq)]
pub struct FileMeta {
    pub name: String,
    pub size: u64,
    pub mime: String,
    pub last_modified: f64,
}

/// Один item из `clipboardData.items`.
#[derive(Debug, Clone, Default)]
pub struct ItemDump {
    pub index: u32,
    /// `"string"` или `"file"`.
    pub kind: String,
    /// Для kind=string — MIME-тип; для kind=file — обычно пусто.
    pub mime: String,
    /// Метаданные файла, если item.file.
    pub file_name: Option<String>,
    pub file_size: Option<u64>,
    pub file_mime: Option<String>,
    pub file_last_modified: Option<f64>,
}

/// Полный дамп `DataTransfer` для диагностики.
#[derive(Debug, Clone, Default)]
pub struct DataTransferDump {
    /// `clipboardData.items.length`.
    pub items_count: u32,
    /// `clipboardData.files.length`.
    pub files_count: u32,
    /// `clipboardData.types` — список MIME-типов буфера.
    pub types: Vec<String>,
    /// Все items с метаданными.
    pub items: Vec<ItemDump>,
    /// Файлы через `.files` — то, что Chrome отдаёт напрямую.
    pub files_from_files_list: Vec<FileMeta>,
    /// Файлы через `.items` с `kind === "file"` и `getAsFile()`.
    pub files_from_items: Vec<FileMeta>,
}

impl DataTransferDump {
        /// true, если `.items` содержит больше файлов, чем `.files` API.
    ///
    /// Расхождение имеет смысл **только для файлов**. Для plain-текста
    /// `items=1, files=0` — это норма: один string-item, ноль файлов.
    ///
    /// Также важно понимать: если браузер отдал один файл из трёх —
    /// расхождения **нет**. Данные согласованы, но неполны.
    /// Это ограничение браузера (см. `explain_multifile_limitation`).
    pub fn has_discrepancy(&self) -> bool {
        self.files_from_items.len() > self.files_from_files_list.len()
    }
}

/// Снимок содержимого `ClipboardEvent.clipboardData`.
#[derive(Debug, Clone, Default)]
pub struct PastePayload {
    pub plain: String,
    pub html: Option<String>,
    pub uris: Vec<String>,
    pub files: Vec<FileMeta>,
    pub types: Vec<String>,
    /// Диагностический дамп — показываем в UI для отладки.
    pub diagnostic: DataTransferDump,
}

impl PastePayload {
    pub fn is_empty(&self) -> bool {
        self.plain.is_empty()
            && self.html.as_deref().map_or(true, |s| s.is_empty())
            && self.uris.is_empty()
            && self.files.is_empty()
    }

    pub fn summary(&self) -> String {
        let mut parts: Vec<String> = Vec::new();
        if !self.plain.is_empty() {
            parts.push(format!("plain: {} симв.", self.plain.chars().count()));
        }
        if let Some(h) = &self.html {
            if !h.is_empty() {
                parts.push(format!("html: {} симв.", h.chars().count()));
            }
        }
        if !self.uris.is_empty() {
            parts.push(format!("uri: {}", self.uris.len()));
        }
        if !self.files.is_empty() {
            parts.push(format!("files: {}", self.files.len()));
        }
        if parts.is_empty() {
            return "пусто".to_string();
        }
        return parts.join(" · ");
    }
}

// ============================================================
//                    ИЗВЛЕЧЕНИЕ
// ============================================================

pub fn extract_from_event(ev: &web_sys::Event) -> Option<PastePayload> {
    let dt = get_data_transfer(ev)?;
    return Some(extract_from_data_transfer(&dt));
}

pub fn extract_from_data_transfer(dt: &web_sys::DataTransfer) -> PastePayload {
    let types = read_types(dt);
    let plain = dt.get_data("text/plain").unwrap_or_default();

    let html_raw = dt.get_data("text/html").unwrap_or_default();
    let html = if html_raw.is_empty() {
        None
    } else {
        Some(strip_cf_html_wrapper(&html_raw).to_string())
    };

    let uri_raw = dt.get_data("text/uri-list").unwrap_or_default();
    let uris = parse_uri_list(&uri_raw);

    // Основной источник файлов — items.
    let files_from_items = read_files_via_items(dt);
    // Вспомогательный источник — для диагностики.
    let files_from_files_list = read_files_via_files_list(dt);

    // Итоговый набор файлов: берём максимум из двух источников.
    // Если items дал больше — используем его. Если меньше (например,
    // Safari отдаёт files, но не items) — используем files_list.
    let files = if files_from_items.len() >= files_from_files_list.len() {
        files_from_items.clone()
    } else {
        files_from_files_list.clone()
    };

    let diagnostic = dump_data_transfer(
        dt,
        types.clone(),
        files_from_items,
        files_from_files_list,
    );

    PastePayload {
        plain,
        html,
        uris,
        files,
        types,
        diagnostic,
    }
}

/// Разбирает `text/uri-list` по RFC 2483.
pub fn parse_uri_list(s: &str) -> Vec<String> {
    let mut out = Vec::new();
    for line in s.lines() {
        let t = line.trim();
        if t.is_empty() || t.starts_with('#') {
            continue;
        }
        out.push(t.to_string());
    }
    out
}

/// Извлекает содержимое из CF_HTML-обёртки, если она есть.
pub fn strip_cf_html_wrapper(s: &str) -> &str {
    const START: &str = "<!--StartFragment-->";
    const END: &str = "<!--EndFragment-->";
    if let Some(i) = s.find(START) {
        let after = &s[i + START.len()..];
        if let Some(j) = after.find(END) {
            return after[..j].trim();
        }
    }
    s
}

// ============================================================
//                    ДИАГНОСТИКА
// ============================================================

fn dump_data_transfer(
    dt: &web_sys::DataTransfer,
    types: Vec<String>,
    files_from_items: Vec<FileMeta>,
    files_from_files_list: Vec<FileMeta>,
) -> DataTransferDump {
    let items = read_items_dump(dt);
    DataTransferDump {
        items_count: items.len() as u32,
        files_count: files_from_files_list.len() as u32,
        types,
        items,
        files_from_files_list,
        files_from_items,
    }
}

/// Читает `clipboardData.items` с полными метаданными.
fn read_items_dump(dt: &web_sys::DataTransfer) -> Vec<ItemDump> {
    let items = match js_sys::Reflect::get(dt, &JsValue::from_str("items")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return Vec::new(),
    };
    let length = js_sys::Reflect::get(&items, &JsValue::from_str("length"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u32;

    let mut out = Vec::with_capacity(length as usize);
    for i in 0..length {
        let item = match call_item_method(&items, i) {
            Some(v) if !v.is_undefined() && !v.is_null() => v,
            _ => continue,
        };

        let kind = js_sys::Reflect::get(&item, &JsValue::from_str("kind"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();
        let mime = js_sys::Reflect::get(&item, &JsValue::from_str("type"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();

        let mut dump = ItemDump {
            index: i,
            kind,
            mime,
            ..Default::default()
        };

        if dump.kind == "file" {
            if let Some(file) = call_get_as_file(&item) {
                dump.file_name = read_file_string(&file, "name");
                dump.file_size = read_file_f64(&file, "size").map(|v| v as u64);
                dump.file_mime = read_file_string(&file, "type");
                dump.file_last_modified = read_file_f64(&file, "lastModified");
            }
        }

        out.push(dump);
    }
    out
}

fn call_get_as_file(item: &JsValue) -> Option<JsValue> {
    let f: js_sys::Function = js_sys::Reflect::get(item, &JsValue::from_str("getAsFile"))
        .ok()?
        .dyn_into()
        .ok()?;
    let file = f.call0(item).ok()?;
    if file.is_null() || file.is_undefined() {
        return None;
    }
    Some(file)
}

fn read_file_string(file: &JsValue, key: &str) -> Option<String> {
    js_sys::Reflect::get(file, &JsValue::from_str(key))
        .ok()
        .and_then(|v| v.as_string())
}

fn read_file_f64(file: &JsValue, key: &str) -> Option<f64> {
    js_sys::Reflect::get(file, &JsValue::from_str(key))
        .ok()
        .and_then(|v| v.as_f64())
}

// ============================================================
//                    ВНУТРЕННЕЕ
// ============================================================

fn get_data_transfer(ev: &web_sys::Event) -> Option<web_sys::DataTransfer> {
    let cd = js_sys::Reflect::get(ev, &JsValue::from_str("clipboardData")).ok()?;
    if cd.is_undefined() || cd.is_null() {
        return None;
    }
    return cd.dyn_into::<web_sys::DataTransfer>().ok();
}

fn read_types(dt: &web_sys::DataTransfer) -> Vec<String> {
    let types = match js_sys::Reflect::get(dt, &JsValue::from_str("types")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return Vec::new(),
    };
    let length = js_sys::Reflect::get(&types, &JsValue::from_str("length"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u32;
    let mut out = Vec::with_capacity(length as usize);
    for i in 0..length {
        let item = js_sys::Reflect::get(&types, &JsValue::from_f64(i as f64))
            .unwrap_or(JsValue::UNDEFINED);
        if let Some(s) = item.as_string() {
            out.push(s);
        }
    }
    out
}

/// Читает файлы через `clipboardData.files` (FileList).
fn read_files_via_files_list(dt: &web_sys::DataTransfer) -> Vec<FileMeta> {
    let files = match js_sys::Reflect::get(dt, &JsValue::from_str("files")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return Vec::new(),
    };
    let length = js_sys::Reflect::get(&files, &JsValue::from_str("length"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u32;

    let mut out = Vec::with_capacity(length as usize);
    for i in 0..length {
        let item = js_sys::Reflect::get(&files, &JsValue::from_f64(i as f64))
            .unwrap_or(JsValue::UNDEFINED);
        if item.is_undefined() || item.is_null() {
            continue;
        }
        if let Some(meta) = extract_file_meta(&item) {
            out.push(meta);
        }
    }
    out
}

/// Читает файлы через `clipboardData.items` с `kind === "file"`.
fn read_files_via_items(dt: &web_sys::DataTransfer) -> Vec<FileMeta> {
    let items = match js_sys::Reflect::get(dt, &JsValue::from_str("items")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return Vec::new(),
    };
    let length = js_sys::Reflect::get(&items, &JsValue::from_str("length"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u32;

    let mut out = Vec::with_capacity(length as usize);
    for i in 0..length {
        let item = match call_item_method(&items, i) {
            Some(v) if !v.is_undefined() && !v.is_null() => v,
            _ => continue,
        };
        let kind = js_sys::Reflect::get(&item, &JsValue::from_str("kind"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();
        if kind != "file" {
            continue;
        }
        if let Some(file) = call_get_as_file(&item) {
            if let Some(meta) = extract_file_meta(&file) {
                out.push(meta);
            }
        }
    }
    out
}

fn extract_file_meta(file: &JsValue) -> Option<FileMeta> {
    let name = read_file_string(file, "name").unwrap_or_default();
    if name.is_empty() {
        return None;
    }
    Some(FileMeta {
        name,
        size: read_file_f64(file, "size").unwrap_or(0.0) as u64,
        mime: read_file_string(file, "type").unwrap_or_default(),
        last_modified: read_file_f64(file, "lastModified").unwrap_or(0.0),
    })
}

fn call_item_method(items: &JsValue, i: u32) -> Option<JsValue> {
    if let Ok(f) = js_sys::Reflect::get(items, &JsValue::from_str("item"))
        .and_then(|v| v.dyn_into::<js_sys::Function>())
    {
        if let Ok(v) = f.call1(items, &JsValue::from_f64(i as f64)) {
            return Some(v);
        }
    }
    js_sys::Reflect::get(items, &JsValue::from_f64(i as f64)).ok()
}
```

## 32.01.5. Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod paste_evt;
```

## 32.01.6. Тесты — `tests/wasm_paste_evt.rs`

```rust
//! Wasm-тесты `paste_evt`: извлечение содержимого из DataTransfer,
//! разбор `text/uri-list`, обработка файлов, поведение на пустом буфере.
//!
//! ПРАВИЛО: wasm-тесты называются с префиксом `wasm_` в имени файла.
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`
//! или явный `return`.
//! ПРАВИЛО: каждое имя базы содержит уникальный суффикс — но здесь
//! базы нет, работаем только с `DataTransfer` в памяти.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

use hello_wasm02::paste_evt;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- helpers ----------------

fn make_dt(entries: &[(&str, &str)]) -> web_sys::DataTransfer {
    let dt = web_sys::DataTransfer::new().expect("new DataTransfer");
    for (mime, value) in entries {
        dt.set_data(mime, value).expect("set_data");
    }
    return dt;
}

fn make_paste_event(entries: &[(&str, &str)]) -> web_sys::Event {
    let ev = web_sys::Event::new("paste").expect("new Event");
    let dt = make_dt(entries);
    js_sys::Reflect::set(&ev, &JsValue::from_str("clipboardData"), &dt)
        .expect("set clipboardData");
    ev
}

// ---------------- parse_uri_list (чистая функция) ----------------

#[wasm_bindgen_test]
fn parse_uri_list_single() {
    let v = paste_evt::parse_uri_list("https://example.com/");
    assert_eq!(v, vec!["https://example.com/"]);
}

#[wasm_bindgen_test]
fn parse_uri_list_multiple_crlf() {
    let v = paste_evt::parse_uri_list("https://a/\r\nhttps://b/\r\nhttps://c/");
    assert_eq!(v, vec!["https://a/", "https://b/", "https://c/"]);
}

#[wasm_bindgen_test]
fn parse_uri_list_skips_comments_and_blanks() {
    let raw = "#comment\n\nhttps://a/\n#another\nhttps://b/\n";
    let v = paste_evt::parse_uri_list(raw);
    assert_eq!(v, vec!["https://a/", "https://b/"]);
}

#[wasm_bindgen_test]
fn parse_uri_list_empty() {
    let v = paste_evt::parse_uri_list("");
    assert!(v.is_empty());
}

// ---------------- extract: text/plain ----------------

#[wasm_bindgen_test]
fn extract_plain_only() {
    let dt = make_dt(&[("text/plain", "hello world")]);
    let p = paste_evt::extract_from_data_transfer(&dt);
    assert_eq!(p.plain, "hello world");
    assert!(p.html.is_none());
    assert!(p.uris.is_empty());
    assert!(p.files.is_empty());   // ← исправлено
    assert!(p.types.iter().any(|t| t == "text/plain"));
}

#[wasm_bindgen_test]
fn extract_plain_and_html() {
    let dt = make_dt(&[
        ("text/plain", "hello"),
        ("text/html", "<b>hello</b>"),
    ]);
    let p = paste_evt::extract_from_data_transfer(&dt);
    assert_eq!(p.plain, "hello");
    assert_eq!(p.html.as_deref(), Some("<b>hello</b>"));
}

#[wasm_bindgen_test]
fn extract_empty_dt() {
    let dt = web_sys::DataTransfer::new().expect("new");
    let p = paste_evt::extract_from_data_transfer(&dt);
    assert!(p.is_empty(), "пустой DataTransfer → is_empty");
    assert_eq!(p.summary(), "пусто");
}

// ---------------- extract: text/uri-list ----------------

#[wasm_bindgen_test]
fn extract_uri_list() {
    let dt = make_dt(&[(
        "text/uri-list",
        "https://a/\r\nhttps://b/",
    )]);
    let p = paste_evt::extract_from_data_transfer(&dt);
    assert_eq!(p.uris, vec!["https://a/", "https://b/"]);
}

// ---------------- extract через событие ----------------

#[wasm_bindgen_test]
fn extract_from_paste_event_works() {
    let ev = make_paste_event(&[
        ("text/plain", "from paste"),
        ("text/html", "<i>from paste</i>"),
    ]);
    let p = paste_evt::extract_from_event(&ev).expect("extract");
    assert_eq!(p.plain, "from paste");
    assert_eq!(p.html.as_deref(), Some("<i>from paste</i>"));
}

#[wasm_bindgen_test]
fn extract_from_event_without_clipboard_data_returns_none() {
    let ev = web_sys::Event::new("paste").expect("event");
    let p = paste_evt::extract_from_event(&ev);
    assert!(p.is_none(), "без clipboardData — None");
}

// ---------------- summary ----------------

#[wasm_bindgen_test]
fn summary_plain_and_html() {
    let dt = make_dt(&[
        ("text/plain", "abc"),
        ("text/html", "<p>abc</p>"),
    ]);
    let p = paste_evt::extract_from_data_transfer(&dt);
    let s = p.summary();
    assert!(s.contains("plain: 3"), "summary: {s}");
    assert!(s.contains("html:"), "summary: {s}");
}

#[wasm_bindgen_test]
fn summary_uri_only() {
    let dt = make_dt(&[("text/uri-list", "https://a/")]);
    let p = paste_evt::extract_from_data_transfer(&dt);
    let s = p.summary();
    assert!(s.contains("uri: 1"), "summary: {s}");
}

// ---------------- is_empty ----------------

#[wasm_bindgen_test]
fn is_empty_false_for_plain() {
    let dt = make_dt(&[("text/plain", "x")]);
    let p = paste_evt::extract_from_data_transfer(&dt);
    assert!(!p.is_empty());
}

#[wasm_bindgen_test]
fn is_empty_true_for_empty_html_string() {
    // Пустая строка как html не должна считаться содержимым.
    let dt = web_sys::DataTransfer::new().expect("new");
    // set_data с пустой строкой иногда не сохраняет формат — используем
    // только text/plain пустой.
    let _ = dt.set_data("text/html", "");
    let p = paste_evt::extract_from_data_transfer(&dt);
    assert!(p.is_empty());
}
```

**Команды запуска:**

```bash
# Проверка типов
cargo check
cargo check --target wasm32-unknown-unknown

# Тесты
wasm-pack test --headless --chrome  --test wasm_paste_evt
wasm-pack test --headless --firefox --test wasm_paste_evt

# Группа по подстроке
wasm-pack test --headless --chrome --test wasm_paste_evt -- parse_uri_list
wasm-pack test --headless --chrome --test wasm_paste_evt -- extract_
```

## 32.01.7. Демо-пример — `src/app.rs`

Демо — **три панели**: paste event на `<div contenteditable>`, Async API для сравнения, журнал с временными метками. Под payload-карточкой — раскрывающаяся диагностика `DataTransfer` с полным дампом, бейджем браузера и кнопкой «Скопировать отчёт».

Полный код `src/app.rs` — в приложении к главе. Ключевые архитектурные решения:

- **`<div contenteditable="true">` вместо `<textarea>`.** Chrome фильтрует `clipboardData.items` для textarea — передаёт только первый файл. В editable-контексте items содержит все файлы.
- **`preventDefault()` в `on_paste`.** Останавливает вставку контента в div — payload показывается в отдельной карточке, сам div остаётся пустым.
- **Определение браузера через `user_agent()`.** Разное поведение Chromium и Firefox требует разных подсказок в UI.
- **Делегированный обработчик кликов.** Кнопка «Скопировать отчёт» создаётся динамически в `render_diagnostics`. Навешивать `onclick` на неё каждый рендер — утечка `Closure`. Один делегированный слушатель на `pe-captured` решает проблему.
- **ObjectURL не нужны.** В отличие от главы 32, здесь мы не создаём превью изображений из буфера — работаем только с метаданными.

## 32.01.8. Путь исследования и практические находки

Это самая ценная часть главы. Каждая находка стоила итерации отладки.

### Находка 1. `<textarea>` фильтрует файлы

Первая версия демо использовала `<textarea>` как приёмник paste. При копировании трёх файлов в Chrome прилетал **один** — `.txt`, не тот, который хотелось. В Firefox — тоже один.

**Причина.** Chrome на `paste` в `<textarea>` фильтрует `clipboardData.items`: оставляет только первый файл. Это ограничение контекста, а не API. В `<div contenteditable="true">` items содержит все файлы.

**Что сделано.** Заменили `<textarea>` на `<div contenteditable="true">`. Это не костыль — так делают Monaco, CodeMirror 6, VS Code Web.

**Правило.** Если приложение принимает paste с файлами — не используйте `<textarea>`, используйте `<div contenteditable>`.

### Находка 2. Chrome vs Firefox — разное поведение при multi-file paste

После перехода на contenteditable Chrome стал отдавать все три файла. Firefox — по-прежнему один.

**Причина.** Windows Explorer при multi-file copy кладёт в буфер несколько форматов: `FileNameW` (primary) и `CF_HDROP` (список путей). Chromium разворачивает `CF_HDROP` в `clipboardData.items` — JS видит все файлы. Firefox берёт только `FileNameW` — JS видит один. `CF_HDROP` для JS недоступен — это native-формат.

**Что сделано.** Демо определяет браузер через `navigator.userAgent` и показывает разные подсказки:

- Chromium → «Этот браузер передаёт все файлы из буфера. Если вы видите один — значит скопировали один.»
- Firefox/Safari → «Firefox и Safari передают только один (primary) файл. Остальные хранятся в CF_HDROP, недоступном для JS. Для получения всех файлов используйте drag-and-drop или `<input type="file" multiple>`.»

**Правило.** Браузерные различия — часть контракта. Если API ведёт себя по-разному — демо должно это явно называть.

### Находка 3. CF_HTML обёртка в `text/html`

При копировании HTML-фрагмента с веб-страницы `clipboardData.getData("text/html")` возвращал:

```
<html><body><!--StartFragment--><h2>Title</h2><!--EndFragment--></body></html>
```

Пользователь видит обёртку браузера, а не свой фрагмент.

**Причина.** Windows Clipboard Format (`CF_HTML`) — стандарт Microsoft, который добавляет маркеры выделения. Chrome и Firefox сохраняют эту обёртку в `paste`.

**Что сделано.** Функция `strip_cf_html_wrapper` вырезает содержимое между `<!--StartFragment-->` и `<!--EndFragment-->`. Результат показывается в UI без шума.

**Правило.** `text/html` в UI-превью — только после снятия CF_HTML-обёртки.

### Находка 4. URL как plain text — не всегда `text/uri-list`

Пользователь скопировал адрес из строки браузера Firefox. `plain` содержал `http://127.0.0.1:5173/`, но `uris` был пуст — `text/uri-list` не появился.

**Причина.** Firefox при копировании из адресной строки кладёт URL только как `text/plain`. При копировании ссылки через правый клик — иногда добавляет `text/uri-list`, иногда нет.

**Что сделано.** Функция `looks_like_url(s)` распознаёт URL в plain-тексте. Если payload содержит только plain, но он похож на URL — подсказка «попробуйте ссылку» подавляется, вместо неё — пояснение о поведении Firefox.

**Правило.** Не полагайтесь на MIME-тип, чтобы определить семантику. URL может прийти как `text/plain`, а rich text — как `text/html` без `text/plain`.

### Находка 5. Диагностика как инструмент исследования

Когда три итерации подряд не удавалось понять, почему файлов меньше, чем ожидалось, стало ясно: **нужна диагностика**. В демо добавлен раскрывающийся блок `DataTransfer — диагностика`:

- `items.length`, `files.length`, `file-items`
- Псевдо-MIME `Files` в списке `types`
- Полный дамп каждого item с `kind`, `mime`, метаданными
- Сравнение `items` и `.files` API — два источника правды
- Бейдж браузера с цветовым кодом
- Кнопка «Скопировать отчёт» с user-agent и всеми деталями — готовый баг-репорт

**Правило.** Когда поведение браузера расходится с ожиданиями — не гадайте, встройте диагностику. Одно раскрытие `<details>` даёт больше, чем десять попыток «а что если».

### Находка 6. Автоматическое раскрытие диагностики

Первая версия диагностики была закрыта по умолчанию. Пользователь не видел детали, пока не кликнет. Проблема: при работе с файлами он **хочет** видеть детали.

**Решение.** Auto-open по трём условиям:

1. `has_files` — в буфере есть файлы (нужны детали).
2. `has_discrepancy()` — items дал больше файлов, чем `.files` API (проблема).
3. `items_count == 0` — DataTransfer вероятно invalidated (ошибка).

**Правило.** Раскрывайте диагностику автоматически, когда она содержит ответ на вопрос «почему это выглядит странно». Не заставляйте пользователя кликать.

### Находка 7. Иконки типов файлов

Первая версия показывала файлы как `deepseek_rust_20261007_bcf2f1.rs  139.0 КБ`. При четырёх файлах читать список приходилось медленно — все строки монотонные.

**Решение.** Иконка типа на основе MIME:

- `image/*` → 🖼
- `video/*` → 🎬
- `audio/*` → 🎵
- `text/*` → 📄
- `application/pdf` → 📕
- `application/zip` → 📦
- остальное → 📎

**Правило.** Визуальная категоризация ускоряет сканирование. Особенно полезно, когда элементы однотипны на вид, но разные по сути.

### Находка 8. Суммарный размер файлов

Заголовок `files: 4 шт.` не давал представления об объёме. При копировании PDF на 1 МБ + три файла — казалось, что данных мало.

**Решение.** `files: 4 шт. · 2.03 МБ` в заголовке списка.

**Правило.** При работе с набором файлов показывайте сумму. Пользователь должен видеть масштаб с одного взгляда.

### Находка 9. Порядок файлов не детерминирован

Скриншот Chrome с 4 файлами: `.pdf → .rs → .txt → .jpg`. Скриншот Firefox с 1 файлом: `.txt`. Порядок не совпадает с порядком выделения в проводнике — его определяют браузер и ОС.

**Решение.** Сноска под списком: «порядок файлов определяется браузером и ОС, а не порядком выделения».

**Правило.** Не обещайте порядок, который не контролируете.

### Находка 10. Итог трёх итераций — «не решаемая» проблема стала понятной

Первоначальная постановка: «Получить все три файла из paste». После исследования стало ясно: **для Firefox и Safari это невозможно**. Это ограничение движка, а не наша задача. Что мы **можем**:

- В Chromium получить все файлы (работает через `items`).
- Для Firefox/Safari честно сообщить об ограничении.
- Указать альтернативы: drag-and-drop, `<input type="file" multiple>`, `showOpenFilePicker`.

**Правило.** Не бороться с ограничениями API — документировать их. Демо, которое честно говорит «в Firefox будет один файл, это не баг», полезнее демо, которое пытается обойти ограничение и не может.

## 32.01.9. Правила проекта, закреплённые в главе

1. **`<textarea>` фильтрует `clipboardData.items`.** Для полноценного доступа к буферу — `<div contenteditable="true">`. Так делают Monaco, CodeMirror 6, VS Code Web.
2. **Chrome vs Firefox — разные движки, разные ограничения.** Multi-file paste работает в Chromium и не работает в Firefox/Safari. Демо должно это называть, а не молчать.
3. **Файлы читаем через `items`, не через `files`.** `DataTransfer.files` фильтруется браузером для не-file-таргетов. `DataTransferItemList` — источник правды.
4. **`CF_HTML` обёртку снимаем перед показом.** `strip_cf_html_wrapper` — обязательная обработка html-поля в UI-превью.
5. **URL может прийти как plain text.** Firefox при копировании из адресной строки не добавляет `text/uri-list`. `looks_like_url` — эвристика для UI, не парсер.
6. **`clipboardData` валиден только внутри обработчика.** После `return` браузер очищает `DataTransfer`. Все данные извлекаем синхронно.
7. **`preventDefault()` в `paste` на contenteditable — обязателен.** Иначе браузер вставит контент в div и мы получим двойное отображение.
8. **Диагностика — не роскошь, а инструмент.** Когда поведение браузера непонятно, встройте блок с полным дампом `DataTransfer`. Это быстрее, чем десять итераций догадок.
9. **Автораскрытие — по проблеме.** Диагностика должна быть открыта, если она содержит ответ на вопрос «почему это странно». Не заставляйте кликать.
10. **`format!` и `matches!` в tail-позиции — через `return`.** rust-analyzer теряет тип в этой позиции.
11. **Делегированный обработчик для динамических кнопок.** Если кнопки создаются в `innerHTML` — навешивайте `onclick` один раз на контейнер, а не пересоздавайте `Closure` на каждый рендер.

## 32.01.10. Итоги главы

- **`ClipboardEvent` — второй путь к буферу обмена.** Не требует разрешений, работает синхронно, видит больше форматов, чем Async API.
- **`paste` — только для Ctrl+V.** Программно вызвать нельзя. Для кнопки «Вставить» — Async API.
- **`clipboardData` содержит `text/plain`, `text/html`, `text/uri-list`, `Files`.** Async `readText()` возвращает только plain.
- **Для файлов нужен `<div contenteditable>`.** `<textarea>` фильтрует список.
- **Chrome передаёт все файлы, Firefox — только primary.** Это ограничение `CF_HDROP`, не наша ошибка.
- **`text/html` в `clipboardData` не имеет CF_HTML-обёртки.** Но `read()` из главы 32 — имеет. Разное поведение двух API.
- **Диагностика в демо — самостоятельная ценность.** Раскрывающийся блок с дампом и отчётом для отправки в баг-трекер.

**Мостик к 32.02.** Мы умеем читать буфер обмена двумя способами: через Async API и через события. Дальше — **запись** через ClipboardEvent: события `copy` и `cut`, `clipboardData.setData()`, как перехватывать копирование из приложения и подменять содержимое. Те же ограничения по форматам, но обратное направление.

---

## Приложение A. Полный код `src/app.rs`

```rust
//! Демо Главы 32.01: `ClipboardEvent` — paste без разрешений.
//!
//! Три панели:
//!   1. `<div contenteditable>` с обработчиком `paste` — показывает весь
//!      payload буфера (plain, html, uri-list, files) без разрешения.
//!   2. Кнопка Async API — `navigator.clipboard.readText()`, для сравнения.
//!   3. Журнал с временными метками.
//!
//! Диагностика DataTransfer — раскрывающийся блок под payload'ом.
//! Автоматически раскрывается при наличии файлов, при расхождении
//! items/files и при items=0.
//!
//! Браузер-специфика multi-file paste:
//!   • Chromium передаёт ВСЕ файлы через `clipboardData.items`.
//!   • Firefox и Safari — ТОЛЬКО ОДИН (primary). Остальные пути
//!     хранятся в native-формате CF_HDROP, недоступном для JS.
//!
//! ПРАВИЛО: `clipboardData` валиден только внутри обработчика.
//! ПРАВИЛО: используем `<div contenteditable>`, а НЕ `<textarea>`.
//! ПРАВИЛО: `preventDefault()` в `paste` — обязательно.
//! ПРАВИЛО: файлы читаются через `items`, не через `files`.
//! ПРАВИЛО: `format!` и `matches!` в tail-позиции — через `return`.
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через `push_str`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;

use crate::format_str::format_str;
use crate::paste_evt::{self, FileMeta, PastePayload};

const PLAIN_PREVIEW_LIMIT: usize = 800;
const HTML_PREVIEW_LIMIT: usize = 2000;

const HTML: &str = r#"
<style>
  :root {
    --pe-info: #3498db; --pe-ok: #27ae60; --pe-err: #c0392b; --pe-warn: #e67e22;
    --pe-text: #222; --pe-muted: #666;
    --pe-bg: #fff; --pe-soft: #fafafa; --pe-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --pe-text: #e6e6e6; --pe-muted: #a0a0a0;
      --pe-bg: #1e1e1e; --pe-soft: #262626; --pe-border: #3a3a3a;
    }
    body { background: #121212; }
    .pe-anno { background: #1a2233; color: #9fc4ff; }
    .pe-anno code { background: rgba(52,152,219,0.2); }
    .pe-editable { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .pe-card { background: #1a1a1a; }
    .pe-pre { background: #151515; }
    .pe-log-entry { background: #1e1e1e; }
    .pe-try { background: #232014; color: #e6c98c; border-color: #5a4400; }
    .pe-try b { color: #f0d8a0; }
    .pe-try code { background: rgba(230,201,140,0.15); color: #f0d8a0; }
    .pe-legend { color: #e6c98c; border-top-color: #5a4400; }
    .pe-legend-row code { background: rgba(230,201,140,0.15); color: #f0d8a0; }
    .pe-hint { background: #232014; color: #e6c98c; border-left-color: #5a4400; }
    .pe-diag { background: #151515; }
    .pe-diag-warn { background: #2a1a1a; color: #e0b8b8; }
    .pe-diag-ok { background: #1a2a1a; color: #b8e0b8; }
    .pe-diag-limit { background: #232014; color: #e6c98c; border-left-color: #5a4400; }
    .pe-browser-note { background: #232014; color: #e6c98c; border-left-color: #e67e22; }
    .pe-browser-note b { color: #f0d8a0; }
    .pe-browser-note code { background: rgba(230,201,140,0.15); color: #f0d8a0; }
    .pe-diag-badge.browser-chromium { background: rgba(39,174,96,0.18); color: #b8e0b8; }
    .pe-diag-badge.browser-firefox  { background: rgba(230,126,34,0.2); color: #f0d8a0; }
    .pe-diag-badge.browser-safari   { background: rgba(230,126,34,0.2); color: #f0d8a0; }
    .pe-diag-badge.browser-other    { background: rgba(128,128,128,0.2); color: #aaa; }
  }
  html, body { width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important; overflow-x: hidden; }
  body {
    display: block !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--pe-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app {
    display: block !important; width: 100% !important; max-width: 100% !important;
    margin: 0 !important; padding: 12px 16px 24px !important;
    box-sizing: border-box !important;
  }
  #app, #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .pe-subtitle { color: var(--pe-muted); font-size: 12px; margin: 0 0 12px; }
  .pe-subtitle code { background: rgba(128,128,128,0.15);
    padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .pe-subtitle a { color: var(--pe-info); }

  .pe-try {
    background: #fff9e6; border: 1px solid #f0e0b0; border-radius: 6px;
    padding: 8px 12px; margin: 0 0 12px;
    font-size: 12px; line-height: 1.55; color: #5a4400;
  }
  .pe-try-head {
    display: flex; align-items: center; gap: 6px;
    font-weight: 600; margin-bottom: 4px;
  }
  .pe-try ol { margin: 4px 0 4px 18px; padding: 0; }
  .pe-try li { padding: 1px 0; }
  .pe-try code { background: rgba(128,128,128,0.15);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .pe-try b { color: #7d5a00; }

  .pe-legend {
    margin-top: 6px; padding-top: 6px;
    border-top: 1px dashed #d0c090;
    font-size: 11px; color: #7d5a00; line-height: 1.5;
  }
  .pe-legend-row {
    display: grid; grid-template-columns: 110px 1fr;
    gap: 3px 8px; align-items: baseline;
  }
  .pe-legend-row code {
    background: rgba(128,128,128,0.15);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10px;
    display: inline-block;
  }

  .pe-browser-note {
    margin-top: 8px; padding: 6px 10px;
    background: #fff9e6; border-left: 3px solid var(--pe-warn);
    border-radius: 4px; font-size: 11.5px; color: #7d5a00;
    line-height: 1.55;
  }
  .pe-browser-note b { color: #5a4400; }
  .pe-browser-note code {
    background: rgba(128,128,128,0.15); padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10.5px;
  }

  .pe-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(0, 1.1fr) minmax(0, 1.1fr) minmax(0, 0.9fr);
  }
  @media (max-width: 1100px) { .pe-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px) { .pe-grid { grid-template-columns: 1fr; } }
  .pe-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .pe-panel {
    border: 1px solid var(--pe-border); border-radius: 8px;
    padding: 10px 12px; background: var(--pe-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0;
  }
  .pe-panel-title { font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .pe-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--pe-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .pe-anno { background: #eef4ff; border-left: 3px solid var(--pe-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere; word-break: break-word; }
  .pe-anno code { background: rgba(52,152,219,0.12);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10px; }
  .pe-anno b { color: #0d3b66; }
  @media (prefers-color-scheme: dark) { .pe-anno b { color: #c0d8ff; } }

  .pe-editable {
    width: 100%; min-height: 100px; max-height: 200px;
    padding: 8px; overflow-y: auto;
    font-family: ui-monospace, monospace; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--pe-bg); color: var(--pe-text);
    outline: none; white-space: pre-wrap; word-break: break-word;
  }
  .pe-editable:focus {
    border-color: var(--pe-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }
  .pe-editable:empty::before {
    content: attr(data-placeholder);
    color: #999;
    pointer-events: none;
  }

  .pe-form-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; min-width: 0; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--pe-soft); color: var(--pe-text);
    cursor: pointer; transition: background 0.15s;
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--pe-info); outline-offset: 2px; }
  button.pe-primary { background: #eaf4ff; border-color: var(--pe-info); color: #1f4e79; }
  button.pe-danger  { background: #fdedec; border-color: var(--pe-err); color: #922b21; }

  .pe-card {
    border: 1px solid var(--pe-border); border-radius: 5px;
    background: var(--pe-soft); padding: 8px 10px; margin-top: 6px;
    font-family: ui-monospace, monospace; font-size: 11px;
    animation: pe-card-appear 0.35s ease-out;
  }
  @keyframes pe-card-appear {
    from {
      background: rgba(52,152,219,0.12);
      box-shadow: 0 0 0 2px rgba(52,152,219,0.25);
    }
    to {
      background: var(--pe-soft);
      box-shadow: none;
    }
  }
  @media (prefers-color-scheme: dark) {
    @keyframes pe-card-appear {
      from {
        background: rgba(52,152,219,0.2);
        box-shadow: 0 0 0 2px rgba(52,152,219,0.35);
      }
      to {
        background: var(--pe-soft);
        box-shadow: none;
      }
    }
  }

  .pe-card-empty {
    padding: 14px; text-align: center; color: #999;
    font-style: italic; font-size: 11px;
  }
  .pe-row { margin: 4px 0; }
  .pe-row b { color: var(--pe-muted); font-weight: 600; }
  .pe-pre {
    margin: 4px 0 0; padding: 6px 8px;
    background: #fbfbfb; border: 1px solid var(--pe-border); border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px;
    white-space: pre-wrap; word-break: break-word;
    max-height: 400px; overflow-y: auto;
    color: var(--pe-text);
  }
  .pe-truncated {
    font-size: 10px; color: var(--pe-muted); font-style: italic;
    margin-top: 2px;
  }
  .pe-list {
    margin: 4px 0 0 16px; padding: 0; list-style: disc;
  }
  .pe-list li { font-size: 11px; margin: 2px 0; }
  .pe-mime {
    display: inline-block; padding: 0 6px; border-radius: 8px;
    background: #eef4ff; color: #1f4e79; font-size: 10px; font-weight: 700;
    margin-right: 4px;
  }
  @media (prefers-color-scheme: dark) {
    .pe-mime { background: #2a3a4a; color: #cfe0ff; }
  }
  .pe-mime-files { background: #fff3cd; color: #7d5a00; }
  @media (prefers-color-scheme: dark) {
    .pe-mime-files { background: #4a3a1a; color: #ffe0a0; }
  }
  .pe-file-line {
    font-size: 11px; margin: 2px 0;
    line-height: 1.5;
  }
  .pe-file-icon {
    display: inline-block; width: 16px; margin-right: 4px;
    text-align: center;
  }
  .pe-file-name { font-weight: 700; color: var(--pe-text); }
  .pe-file-size {
    color: var(--pe-muted); font-size: 10px;
    margin-left: 8px;
  }
  .pe-file-total {
    color: var(--pe-muted); font-weight: 400;
  }
  .pe-compare {
    margin-top: 8px; padding-top: 6px;
    border-top: 1px dashed var(--pe-border);
  }
  .pe-ok   { color: var(--pe-ok); font-weight: 700; font-size: 11px; }
  .pe-diff { color: var(--pe-err); font-weight: 700; font-size: 11px; }
  .pe-warn { color: var(--pe-warn); font-weight: 700; font-size: 11px; }
  .pe-muted { color: var(--pe-muted); font-style: italic; font-size: 11px; }
  .pe-hint {
    margin-top: 4px; padding: 4px 8px;
    background: #fff9e6; border-left: 3px solid var(--pe-warn);
    border-radius: 3px; font-size: 11px; color: #7d5a00;
    line-height: 1.5;
  }
  .pe-note {
    margin-top: 4px; font-size: 10.5px; color: var(--pe-muted);
    font-style: italic; line-height: 1.5;
  }

  /* ============ Диагностика ============ */
  .pe-diag {
    margin-top: 8px; padding: 6px 10px;
    border: 1px dashed var(--pe-border); border-radius: 4px;
    background: #f8f9fb;
    font-family: ui-monospace, monospace; font-size: 10.5px;
    line-height: 1.55; color: var(--pe-text);
  }
  @media (prefers-color-scheme: dark) { .pe-diag { background: #151515; } }
  .pe-diag summary {
    cursor: pointer; font-weight: 700; user-select: none;
    color: var(--pe-info); outline: none;
    display: flex; align-items: center; gap: 6px;
    flex-wrap: wrap;
  }
  .pe-diag summary::-webkit-details-marker { display: none; }
  .pe-diag summary::before {
    content: "▸"; display: inline-block;
    transition: transform 0.15s;
  }
  .pe-diag[open] summary::before { transform: rotate(90deg); }
  .pe-diag-badge {
    padding: 0 6px; border-radius: 8px;
    font-weight: 700; font-size: 10px;
  }
  .pe-diag-badge.warn { background: rgba(230,126,34,0.15); color: #7d5a00; }
  .pe-diag-badge.browser-chromium { background: rgba(39,174,96,0.15); color: #1e8449; }
  .pe-diag-badge.browser-firefox  { background: rgba(230,126,34,0.15); color: #7d5a00; }
  .pe-diag-badge.browser-safari   { background: rgba(230,126,34,0.15); color: #7d5a00; }
  .pe-diag-badge.browser-other    { background: rgba(128,128,128,0.15); color: #666; }
  .pe-diag-section { margin: 6px 0 2px; color: var(--pe-muted); font-weight: 700; }
  .pe-diag-line { margin: 1px 0; padding-left: 10px; }
  .pe-diag-line .k { color: var(--pe-muted); }
  .pe-diag-line .v { color: var(--pe-text); font-weight: 700; }
  .pe-diag-warn {
    margin: 4px 0; padding: 4px 8px;
    background: #fdedec; border-left: 3px solid var(--pe-err);
    border-radius: 3px; color: #922b21;
  }
  .pe-diag-ok {
    margin: 4px 0; padding: 4px 8px;
    background: #e8f8ed; border-left: 3px solid var(--pe-ok);
    border-radius: 3px; color: #1e8449;
  }
  .pe-diag-limit {
    margin: 4px 0; padding: 4px 8px;
    background: #fff9e6; border-left: 3px solid var(--pe-warn);
    border-radius: 3px; color: #7d5a00; line-height: 1.55;
  }
  .pe-diag-kind-file   { color: #9b59b6; font-weight: 700; }
  .pe-diag-kind-string { color: var(--pe-warn); font-weight: 700; }
  .pe-diag-kind-other  { color: var(--pe-muted); font-weight: 700; }
  .pe-diag-copy-btn {
    margin-top: 6px; padding: 3px 10px; font-size: 11px;
    background: #eaf4ff; border-color: var(--pe-info); color: #1f4e79;
    cursor: pointer; border-radius: 4px; border-style: solid; border-width: 1px;
    font-family: inherit;
  }
  .pe-diag-copy-btn:hover { background: #d8e8fc; }
  @media (prefers-color-scheme: dark) {
    .pe-diag-copy-btn { background: #1a3a5a; border-color: #2a5a8a; color: #cfe0ff; }
    .pe-diag-copy-btn:hover { background: #234a6a; }
  }

  .pe-log {
    border: 1px solid var(--pe-border); border-radius: 5px;
    background: var(--pe-soft); max-height: 400px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .pe-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .pe-log-entry { display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent; }
  .pe-log-entry.ok    { border-left-color: var(--pe-ok); }
  .pe-log-entry.err   { border-left-color: var(--pe-err); }
  .pe-log-entry.info  { border-left-color: var(--pe-info); }
  .pe-log-entry.async { border-left-color: #9b59b6; }
  .pe-log-entry.warn  { border-left-color: var(--pe-warn); }
  .pe-log-entry .t { color: var(--pe-muted); flex-shrink: 0; }
  .pe-log-entry .m { color: var(--pe-text); flex: 1; overflow-wrap: anywhere; }
  .pe-log-entry.err .m { color: #922b21; }
  .pe-log-entry.warn .m { color: #7d5a00; }
  @media (prefers-color-scheme: dark) {
    .pe-log-entry.warn .m { color: #e6c98c; }
  }
</style>

<h1>Глава 32.01: ClipboardEvent — paste без разрешений</h1>
<p class="pe-subtitle">
  Событие <code>paste</code> даёт доступ к буферу обмена <b>без разрешения</b> и видит
  больше форматов, чем Async API.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/Element/paste_event" target="_blank" rel="noopener">MDN →</a>
</p>

<div class="pe-try">
  <div class="pe-try-head">🎯 Попробуйте все 4 сценария — они показывают разные грани</div>
  <ol>
    <li><b>Простой текст</b> — скопируйте абзац из блокнота. Оба API увидят одно и то же. <i>Скучный случай.</i></li>
    <li><b>Ссылку</b> — правый клик на ссылке → «Копировать ссылку». Paste event увидит <code>text/uri-list</code>.</li>
    <li><b>HTML-фрагмент</b> — выделите текст на веб-странице и скопируйте. Paste event увидит <code>text/html</code>; Async — только plain.</li>
    <li><b>Файл из ОС</b> — скопируйте файл в проводнике. Paste event увидит <code>Files</code> и метаданные; Async — ничего.</li>
  </ol>
  <div class="pe-legend">
    <div class="pe-legend-row">
      <code>text/plain</code><span>плоский текст, почти всегда в буфере</span>
    </div>
    <div class="pe-legend-row">
      <code>text/html</code><span>оформленная версия того же текста (заголовки, ссылки)</span>
    </div>
    <div class="pe-legend-row">
      <code>text/uri-list</code><span>список URL — когда копируют ссылку</span>
    </div>
    <div class="pe-legend-row">
      <code>Files</code><span>псевдо-MIME; копирование файлов из проводника</span>
    </div>
  </div>
  <div class="pe-browser-note">
    <b>ℹ О multi-file paste.</b>
    Chromium передаёт через <code>paste</code> <b>все</b> файлы, Firefox
    и Safari — <b>только один</b> (primary). Остальные хранятся в
    native-формате <code>CF_HDROP</code>, недоступном для JS. Для
    получения всех файлов используйте drag-and-drop или
    <code>&lt;input type="file" multiple&gt;</code>.
  </div>
</div>

<div class="pe-grid">

  <!-- ============ Колонка 1: paste event ============ -->
  <div class="pe-col">
    <div class="pe-panel">
      <h2 class="pe-panel-title">
        <span class="pe-num">1</span>Paste event
      </h2>
      <div class="pe-anno">
        Скопируйте что-нибудь в другой программе (<b>Ctrl+C</b>) и вставьте
        здесь (<b>Ctrl+V</b>). Событие <code>paste</code> отдаст всё содержимое
        буфера. <b>Без разрешения.</b>
      </div>

      <div class="pe-form-row">
        <div id="pe-target"
             class="pe-editable"
             contenteditable="true"
             data-placeholder="Нажмите сюда и вставьте (Ctrl+V)…"
             role="textbox"
             aria-multiline="true"
             spellcheck="false"></div>
      </div>

      <div class="pe-form-row">
        <button id="pe-clear" class="pe-danger" title="Очистить поле и результаты">× Очистить</button>
      </div>

      <div id="pe-captured">
        <div class="pe-card-empty">Содержимое появится после вставки</div>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 2: Async API ============ -->
  <div class="pe-col">
    <div class="pe-panel">
      <h2 class="pe-panel-title">
        <span class="pe-num">2</span>Async API (для сравнения)
      </h2>
      <div class="pe-anno">
        Тот же буфер, но через <code>navigator.clipboard.readText()</code>.
        Возвращает <b>только plain text</b> — даже если в буфере есть HTML,
        URI-список или файлы. Может потребовать разрешение
        <code>clipboard-read</code>.
      </div>

      <div class="pe-form-row">
        <button id="pe-async-read" class="pe-primary" title="navigator.clipboard.readText()">
          📥 Прочитать через Async API
        </button>
      </div>

      <div id="pe-async-result">
        <div class="pe-card-empty">Нажмите кнопку — увидите, что видит Async API</div>
      </div>
    </div>

    <div class="pe-panel">
      <h2 class="pe-panel-title">Что видит каждый подход</h2>
      <table style="width:100%;border-collapse:collapse;font-size:11px">
        <thead>
          <tr>
            <th style="text-align:left;padding:4px 8px;border-bottom:1px solid var(--pe-border);color:var(--pe-muted)">Скопировано</th>
            <th style="text-align:left;padding:4px 8px;border-bottom:1px solid var(--pe-border);color:var(--pe-muted)">paste event</th>
            <th style="text-align:left;padding:4px 8px;border-bottom:1px solid var(--pe-border);color:var(--pe-muted)">readText</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">Текст</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">plain + html</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">только plain</td>
          </tr>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">Ссылка</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">uri-list + plain</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">только текст</td>
          </tr>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">HTML</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">plain + html</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">только plain</td>
          </tr>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">Файлы ОС</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">Files + метаданные</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--pe-border)">пусто</td>
          </tr>
          <tr>
            <td style="padding:4px 8px">Разрешение</td>
            <td style="padding:4px 8px">не требуется</td>
            <td style="padding:4px 8px"><code>clipboard-read</code></td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>

  <!-- ============ Колонка 3: журнал ============ -->
  <div class="pe-col">
    <div class="pe-panel">
      <h2 class="pe-panel-title">Журнал</h2>
      <div id="pe-log" class="pe-log">
        <div class="pe-log-empty">здесь появятся события paste и чтения</div>
      </div>
    </div>
  </div>

</div>
"#;

// ============================================================
//                    БРАУЗЕР
// ============================================================

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
enum Browser {
    Chromium,
    Firefox,
    Safari,
    Other,
}

impl Browser {
    fn label(&self) -> &'static str {
        match self {
            Browser::Chromium => "Chromium (Chrome/Edge/Brave/Opera)",
            Browser::Firefox => "Firefox",
            Browser::Safari => "Safari",
            Browser::Other => "неизвестный",
        }
    }

    fn short(&self) -> &'static str {
        match self {
            Browser::Chromium => "Chromium",
            Browser::Firefox => "Firefox",
            Browser::Safari => "Safari",
            Browser::Other => "Other",
        }
    }

    fn badge_class(&self) -> &'static str {
        match self {
            Browser::Chromium => "browser-chromium",
            Browser::Firefox => "browser-firefox",
            Browser::Safari => "browser-safari",
            Browser::Other => "browser-other",
        }
    }

    /// true, если браузер передаёт ВСЕ файлы при multi-file paste.
    fn passes_all_files(&self) -> bool {
        return matches!(self, Browser::Chromium)
    }
}

fn detect_browser() -> Browser {
    let ua = user_agent().to_lowercase();
    if ua.contains("firefox") || ua.contains("fxios") {
        return Browser::Firefox;
    }
    if ua.contains("edg/")
        || ua.contains("chrome/")
        || ua.contains("chromium")
        || ua.contains("crios")
    {
        return Browser::Chromium;
    }
    if ua.contains("safari") {
        return Browser::Safari;
    }
    return Browser::Other;
}

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind {
    Ok,
    Err,
    Info,
    Async,
    Warn,
}

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

struct Ui {
    target: web_sys::HtmlElement,
    captured: web_sys::Element,
    async_result: web_sys::Element,
    log: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    last_payload: Rc<RefCell<Option<PastePayload>>>,
    ui: Ui,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        target: get_html_element("pe-target")?,
        captured: get_el("pe-captured")?,
        async_result: get_el("pe-async-result")?,
        log: get_el("pe-log")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        last_payload: Rc::new(RefCell::new(None)),
        ui,
    });

    app.render_log();
    app.push_log(
        LogKind::Info,
        "начните с любого из 4 сценариев выше".to_string(),
    );
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "браузер определён как: {}",
            detect_browser().label(),
        )),
    );

    // Обработчик paste на contenteditable.
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |ev: web_sys::Event| {
            on_paste(&app_c, &ev);
        });
        app.ui
            .target
            .add_event_listener_with_callback("paste", cb.as_ref().unchecked_ref())?;
        cb.forget();
    }

    // Делегированный обработчик кликов по карточке payload.
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::MouseEvent)>::new(
            move |ev: web_sys::MouseEvent| {
                on_captured_click(&app_c, &ev);
            },
        );
        app.ui
            .captured
            .add_event_listener_with_callback("click", cb.as_ref().unchecked_ref())?;
        cb.forget();
    }

    // Кнопка Async API.
    {
        let app_c = app.clone();
        let btn = get_button("pe-async-read")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            spawn_local(async move {
                on_async_read(app_in).await;
            });
        });
    }

    // Кнопка Clear.
    {
        let app_c = app.clone();
        let btn = get_button("pe-clear")?;
        set_click(&btn, move |_| {
            app_c.ui.target.set_inner_html("");
            app_c.ui.captured.set_inner_html(
                "<div class=\"pe-card-empty\">Содержимое появится после вставки</div>",
            );
            app_c.ui.async_result.set_inner_html(
                "<div class=\"pe-card-empty\">Нажмите кнопку — увидите, что видит Async API</div>",
            );
            *app_c.last_payload.borrow_mut() = None;
            app_c.push_log(LogKind::Info, "очищено".to_string());
        });
    }

    Ok(())
}

// ============================================================
//                    ОБРАБОТЧИКИ
// ============================================================

fn on_paste(app: &Rc<App>, ev: &web_sys::Event) {
    ev.prevent_default();

    let payload = match paste_evt::extract_from_event(ev) {
        Some(p) => p,
        None => {
            app.push_log(
                LogKind::Err,
                "не удалось извлечь clipboardData".to_string(),
            );
            return;
        }
    };

    let summary = payload.summary();
    app.push_log(
        LogKind::Ok,
        format_str(format_args!("paste event: {summary}")),
    );

    if payload.diagnostic.has_discrepancy() {
        app.push_log(
            LogKind::Warn,
            format_str(format_args!(
                "расхождение DataTransfer: items-файлов={}, files={}",
                payload.diagnostic.files_from_items.len(),
                payload.diagnostic.files_from_files_list.len(),
            )),
        );
    }

    if payload.files.len() == 1 && !detect_browser().passes_all_files() {
        app.push_log(
            LogKind::Warn,
            format_str(format_args!(
                "{} передаёт через paste только один файл — если копировали больше, используйте drag-and-drop",
                detect_browser().label(),
            )),
        );
    }

    let is_boring = !payload.is_empty()
        && payload.html.is_none()
        && payload.uris.is_empty()
        && payload.files.is_empty()
        && !looks_like_url(&payload.plain);
    if is_boring {
        app.push_log(
            LogKind::Warn,
            "только plain — попробуйте ссылку, HTML-фрагмент или файл".to_string(),
        );
    }

    let html = render_payload(&payload);
    app.ui.captured.set_inner_html(&html);

    *app.last_payload.borrow_mut() = Some(payload);
}

fn on_captured_click(app: &Rc<App>, ev: &web_sys::MouseEvent) {
    let target: web_sys::Element = match ev.target().and_then(|t| t.dyn_into().ok()) {
        Some(e) => e,
        None => return,
    };
    let action = target.get_attribute("data-action").unwrap_or_default();
    if action != "copy-report" {
        return;
    }
    ev.stop_propagation();
    ev.prevent_default();

    let payload = match app.last_payload.borrow().clone() {
        Some(p) => p,
        None => {
            app.push_log(
                LogKind::Err,
                "нечего копировать — сначала сделайте paste".to_string(),
            );
            return;
        }
    };

    let report = build_report_text(&payload);
    let len = report.chars().count();

    let app_c = app.clone();
    spawn_local(async move {
        match copy_text_to_clipboard(&report).await {
            Ok(()) => {
                app_c.push_log(
                    LogKind::Ok,
                    format_str(format_args!("отчёт скопирован ({len} символов)")),
                );
            }
            Err(e) => {
                let msg = format_str(format_args!(
                    "копирование отчёта: {}", short_error(&e),
                ));
                app_c.push_log(LogKind::Err, msg);
            }
        }
    });
}

async fn on_async_read(app: Rc<App>) {
    app.push_log(
        LogKind::Async,
        "async: navigator.clipboard.readText()…".to_string(),
    );

    match async_read_text().await {
        Ok(text) => {
            let msg = format_str(format_args!(
                "async readText: {} символов (только plain)",
                text.chars().count(),
            ));
            app.push_log(LogKind::Async, msg);

            let payload = app.last_payload.borrow().clone();
            let html = render_async_result(&text, payload.as_ref());
            app.ui.async_result.set_inner_html(&html);

            if let Some(p) = payload.as_ref() {
                let is_boring = !p.is_empty()
                    && p.html.is_none()
                    && p.uris.is_empty()
                    && p.files.is_empty()
                    && !looks_like_url(&p.plain);
                if is_boring {
                    app.push_log(
                        LogKind::Warn,
                        "результат совпал с paste — это ожидаемо для plain; \
                         попробуйте другой источник"
                            .to_string(),
                    );
                }
            }
        }
        Err(e) => {
            let msg = format_str(format_args!(
                "async readText: {}", short_error(&e),
            ));
            app.push_log(LogKind::Err, msg.clone());
            let mut html = String::with_capacity(128);
            html.push_str("<div class=\"pe-card\"><div class=\"pe-row\">");
            html.push_str("<b>Ошибка:</b> ");
            html.push_str(&esc_html(&msg));
            html.push_str("</div></div>");
            app.ui.async_result.set_inner_html(&html);
        }
    }
}

async fn async_read_text() -> Result<String, JsValue> {
    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))
        .map_err(|e| JsValue::from_str(&format!("navigator: {e:?}")))?;
    let clip = js_sys::Reflect::get(&nav, &JsValue::from_str("clipboard"))
        .map_err(|e| JsValue::from_str(&format!("clipboard: {e:?}")))?;
    if clip.is_undefined() || clip.is_null() {
        return Err(JsValue::from_str("navigator.clipboard недоступен"));
    }
    let read_fn = js_sys::Reflect::get(&clip, &JsValue::from_str("readText"))
        .map_err(|e| JsValue::from_str(&format!("readText: {e:?}")))?;
    let read_fn: js_sys::Function = read_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("readText не функция"))?;

    let promise = read_fn
        .call0(&clip)
        .map_err(|e| JsValue::from_str(&format!("readText call: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("readText не Promise"))?;

    let result = wasm_bindgen_futures::JsFuture::from(promise).await?;
    Ok(result.as_string().unwrap_or_default())
}

async fn copy_text_to_clipboard(text: &str) -> Result<(), JsValue> {
    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))
        .map_err(|e| JsValue::from_str(&format!("navigator: {e:?}")))?;
    let clip = js_sys::Reflect::get(&nav, &JsValue::from_str("clipboard"))
        .map_err(|e| JsValue::from_str(&format!("clipboard: {e:?}")))?;
    if clip.is_undefined() || clip.is_null() {
        return Err(JsValue::from_str("navigator.clipboard недоступен"));
    }
    let write_fn = js_sys::Reflect::get(&clip, &JsValue::from_str("writeText"))
        .map_err(|e| JsValue::from_str(&format!("writeText: {e:?}")))?;
    let write_fn: js_sys::Function = write_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("writeText не функция"))?;
    let promise = write_fn
        .call1(&clip, &JsValue::from_str(text))
        .map_err(|e| JsValue::from_str(&format!("writeText call: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("writeText не Promise"))?;
    wasm_bindgen_futures::JsFuture::from(promise).await?;
    Ok(())
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_payload(p: &PastePayload) -> String {
    let mut html = String::with_capacity(1024);
    html.push_str("<div class=\"pe-card\">");

    if p.is_empty() {
        html.push_str("<div class=\"pe-row\">");
        html.push_str("<b>буфер пуст</b>");
        html.push_str("</div>");
        html.push_str(&render_diagnostics(&p.diagnostic));
        html.push_str("</div>");
        return html;
    }

    html.push_str("<div class=\"pe-row\"><b>MIME:</b> ");
    if p.types.is_empty() {
        html.push_str("<i>нет</i>");
    } else {
        for t in &p.types {
            html.push_str(&mime_chip(t));
        }
    }
    html.push_str("</div>");

    if !p.plain.is_empty() {
        render_text_block(&mut html, "plain", &p.plain, PLAIN_PREVIEW_LIMIT);
    }

    if let Some(h) = &p.html {
        if !h.is_empty() {
            render_text_block(&mut html, "html", h, HTML_PREVIEW_LIMIT);
        }
    }

    if let Some(h) = &p.html {
        let plain_len = p.plain.chars().count();
        let html_len = h.chars().count();
        if plain_len > 0 && html_len > plain_len * 2 {
            let ratio = html_len as f64 / plain_len as f64;
            html.push_str(&format_str(format_args!(
                "<div class=\"pe-hint\">\
                 ℹ html в {ratio:.1}× больше plain — источник добавляет \
                 в html разметку (стили, span'ы). Async API увидит только \
                 plain и эту разметку потеряет.\
                 </div>",
            )));
        }
    }

    if !p.uris.is_empty() {
        html.push_str("<div class=\"pe-row\"><b>uri-list:</b> ");
        html.push_str(&format_str(format_args!("{} шт.", p.uris.len())));
        html.push_str("</div>");
        html.push_str("<ul class=\"pe-list\">");
        for u in &p.uris {
            html.push_str("<li><code>");
            html.push_str(&esc_html(u));
            html.push_str("</code></li>");
        }
        html.push_str("</ul>");
    }

    if !p.files.is_empty() {
        render_file_list(&mut html, &p.files);
    }

    html.push_str(&render_diagnostics(&p.diagnostic));

    html.push_str("</div>");
    html
}

fn render_text_block(out: &mut String, label: &str, text: &str, limit: usize) {
    let total = text.chars().count();
    out.push_str("<div class=\"pe-row\"><b>");
    out.push_str(label);
    out.push_str(":</b> ");
    out.push_str(&format_str(format_args!("{total} симв.")));
    out.push_str("</div>");

    let (preview, truncated) = take_chars_with_flag(text, limit);
    out.push_str("<pre class=\"pe-pre\">");
    out.push_str(&esc_html(&preview));
    out.push_str("</pre>");
    if truncated {
        out.push_str(&format_str(format_args!(
            "<div class=\"pe-truncated\">показаны первые {limit} из {total} символов</div>",
        )));
    }
}

/// Рендерит список файлов: заголовок с общим размером, строки
/// с иконкой типа, именем и размером, сноска про порядок и
/// предупреждение о браузерном ограничении (Firefox/Safari).
fn render_file_list(out: &mut String, files: &[FileMeta]) {
    let total_size: u64 = files.iter().map(|f| f.size).sum();
    out.push_str("<div class=\"pe-row\"><b>files:</b> ");
    out.push_str(&format_str(format_args!(
        "{} шт. <span class=\"pe-file-total\">· {}</span>",
        files.len(),
        fmt_bytes(total_size),
    )));
    out.push_str("</div>");

    for f in files {
        out.push_str("<div class=\"pe-file-line\">");
        out.push_str("<span class=\"pe-file-icon\" title=\"");
        out.push_str(&esc_html(&file_icon_label(&f.mime)));
        out.push_str("\">");
        out.push_str(file_icon(&f.mime));
        out.push_str("</span>");
        out.push_str("<span class=\"pe-file-name\">");
        out.push_str(&esc_html(&f.name));
        out.push_str("</span>");
        out.push_str("<span class=\"pe-file-size\">");
        out.push_str(&fmt_bytes(f.size));
        if !f.mime.is_empty() {
            out.push_str(" · ");
            out.push_str(&esc_html(&f.mime));
        }
        out.push_str("</span>");
        out.push_str("</div>");
    }

    if files.len() > 1 {
        out.push_str(
            "<div class=\"pe-note\">\
             порядок файлов определяется браузером и ОС, а не порядком выделения\
             </div>",
        );
    }

    if files.len() == 1 && !detect_browser().passes_all_files() {
        out.push_str(
            "<div class=\"pe-hint\">\
             ⚠ <b>Firefox и Safari передают через paste только один файл</b> \
             при multi-file copy из проводника. Если вы копировали больше — \
             используйте drag-and-drop или <code>&lt;input type=\"file\" multiple&gt;</code>.\
             </div>",
        );
    }
}

/// Иконка типа файла по MIME. Для пустого MIME — общая «скрепка».
fn file_icon(mime: &str) -> &'static str {
    let m = mime.to_lowercase();
    if m.starts_with("image/") {
        return "🖼";
    }
    if m.starts_with("video/") {
        return "🎬";
    }
    if m.starts_with("audio/") {
        return "🎵";
    }
    if m.starts_with("text/") {
        return "📄";
    }
    if m == "application/pdf" {
        return "📕";
    }
    if m == "application/zip" || m == "application/x-zip-compressed" {
        return "📦";
    }
    if m.starts_with("application/json") || m.starts_with("application/xml") {
        return "📋";
    }
    return "📎";
}

/// Словесная метка типа — для tooltip при наведении на иконку.
fn file_icon_label(mime: &str) -> String {
    let m = mime.to_lowercase();
    if m.starts_with("image/") { return "изображение".to_string(); }
    if m.starts_with("video/") { return "видео".to_string(); }
    if m.starts_with("audio/") { return "аудио".to_string(); }
    if m.starts_with("text/")  { return "текст".to_string(); }
    if m == "application/pdf"  { return "PDF".to_string(); }
    if m == "application/zip" || m == "application/x-zip-compressed" {
        return "архив".to_string();
    }
    if m.starts_with("application/json") { return "JSON".to_string(); }
    if m.starts_with("application/xml")  { return "XML".to_string(); }
    return "файл".to_string();
}

/// Рендерит сворачиваемый блок диагностики `DataTransfer`.
///
/// Автоматически раскрывается при:
///   • наличии файлов — пользователь хочет видеть детали;
///   • `has_discrepancy()` — items дал больше файлов, чем .files API;
///   • `items_count == 0` — DataTransfer, вероятно, invalidated.
///
/// В заголовке:
///   • бейдж браузера — цветовой код (зелёный Chromium, оранжевый Firefox/Safari);
///   • бейдж проблемы — только если она есть (расхождение или items=0).
fn render_diagnostics(d: &paste_evt::DataTransferDump) -> String {
    let mut out = String::with_capacity(2048);

    let browser = detect_browser();
    let has_discrepancy = d.has_discrepancy();
    let empty_items = d.items_count == 0;
    let has_files = !d.files_from_items.is_empty() || !d.files_from_files_list.is_empty();

    let auto_open = has_discrepancy || empty_items || has_files;
    let open_attr = if auto_open { " open" } else { "" };

    out.push_str("<details class=\"pe-diag\"");
    out.push_str(open_attr);
    out.push_str("><summary>");
    out.push_str("DataTransfer — диагностика");

    out.push_str("<span class=\"pe-diag-badge ");
    out.push_str(browser.badge_class());
    out.push_str("\" title=\"");
    out.push_str(&esc_html(browser.label()));
    out.push_str("\">");
    out.push_str(browser.short());
    out.push_str("</span>");

    if has_discrepancy {
        out.push_str("<span class=\"pe-diag-badge warn\">⚠ расхождение</span>");
    } else if empty_items {
        out.push_str("<span class=\"pe-diag-badge warn\">items=0</span>");
    }

    out.push_str(&format_str(format_args!(
        "<span style=\"margin-left:auto;color:var(--pe-muted);font-weight:400\">\
         items={} · files={} · file-items={}</span>",
        d.items_count,
        d.files_count,
        d.files_from_items.len(),
    )));

    out.push_str("</summary>");

    if has_discrepancy {
        out.push_str("<div class=\"pe-diag-warn\">");
        out.push_str(&format_str(format_args!(
            "⚠ расхождение: items-файлов={}, files.length={} — \
             читаем через items + getAsFile()",
            d.files_from_items.len(),
            d.files_count,
        )));
        out.push_str("</div>");
    } else if empty_items {
        out.push_str("<div class=\"pe-diag-warn\">");
        out.push_str(
            "⚠ items.length=0 — DataTransfer, вероятно, уже invalidated. \
             Возможно, событие обработано слишком поздно.",
        );
        out.push_str("</div>");
    } else {
        out.push_str("<div class=\"pe-diag-ok\">");
        out.push_str("✓ items и files согласованы");
        out.push_str("</div>");
    }

    if has_files {
        if browser.passes_all_files() {
            out.push_str("<div class=\"pe-diag-limit\">");
            out.push_str(
                "ℹ Этот браузер (Chromium) передаёт <b>все файлы</b> из буфера. \
                 Если вы видите один файл — значит скопировали один.",
            );
            out.push_str("</div>");
        } else {
            out.push_str("<div class=\"pe-diag-limit\">");
            out.push_str(
                "⚠ <b>Firefox и Safari передают только один (primary) файл.</b> \
                 Остальные пути хранятся в native-формате <code>CF_HDROP</code>, \
                 недоступном для JS. Это ограничение движка.",
            );
            out.push_str("</div>");
        }
    }

    out.push_str("<div class=\"pe-diag-section\">счётчики</div>");
    render_diag_line(&mut out, "items.length", &d.items_count.to_string());
    render_diag_line(&mut out, "files.length", &d.files_count.to_string());
    render_diag_line(
        &mut out,
        "file-items (getAsFile)",
        &d.files_from_items.len().to_string(),
    );

    out.push_str("<div class=\"pe-diag-section\">types — MIME-типы буфера</div>");
    if d.types.is_empty() {
        out.push_str("<div class=\"pe-diag-line\"><span class=\"k\">—</span></div>");
    } else {
        for (i, t) in d.types.iter().enumerate() {
            out.push_str("<div class=\"pe-diag-line\">");
            out.push_str(&format_str(format_args!("[{i}] ")));
            out.push_str(&esc_html(t));
            out.push_str("</div>");
        }
    }

    out.push_str("<div class=\"pe-diag-section\">items[] — что реально в списке</div>");
    if d.items.is_empty() {
        out.push_str("<div class=\"pe-diag-line\"><span class=\"k\">пусто</span></div>");
    } else {
        for it in &d.items {
            out.push_str("<div class=\"pe-diag-line\">");
            out.push_str(&format_str(format_args!("[{}] kind=", it.index)));
            let kind_cls = match it.kind.as_str() {
                "file" => "pe-diag-kind-file",
                "string" => "pe-diag-kind-string",
                _ => "pe-diag-kind-other",
            };
            out.push_str("<span class=\"");
            out.push_str(kind_cls);
            out.push_str("\">");
            out.push_str(&esc_html(&it.kind));
            out.push_str("</span>");
            if !it.mime.is_empty() {
                out.push_str(" mime=");
                out.push_str(&esc_html(&it.mime));
            }
            if let Some(name) = &it.file_name {
                out.push_str(" name=");
                out.push_str(&esc_html(name));
                if let Some(sz) = it.file_size {
                    out.push_str(" size=");
                    out.push_str(&fmt_bytes(sz));
                }
                if let Some(m) = &it.file_mime {
                    if !m.is_empty() {
                        out.push_str(" type=");
                        out.push_str(&esc_html(m));
                    }
                }
            }
            out.push_str("</div>");
        }
    }

    out.push_str("<div class=\"pe-diag-section\">files — что вернул .files API</div>");
    if d.files_from_files_list.is_empty() {
        out.push_str("<div class=\"pe-diag-line\"><span class=\"k\">пусто</span></div>");
    } else {
        for (i, f) in d.files_from_files_list.iter().enumerate() {
            let mime = if f.mime.is_empty() { "—" } else { f.mime.as_str() };
            out.push_str("<div class=\"pe-diag-line\">");
            out.push_str(&format_str(format_args!(
                "[{i}] {} · {} · {mime}",
                esc_html(&f.name),
                fmt_bytes(f.size),
            )));
            out.push_str("</div>");
        }
    }

    out.push_str("<div class=\"pe-diag-section\">file-items — через getAsFile()</div>");
    if d.files_from_items.is_empty() {
        out.push_str("<div class=\"pe-diag-line\"><span class=\"k\">пусто</span></div>");
    } else {
        for (i, f) in d.files_from_items.iter().enumerate() {
            let mime = if f.mime.is_empty() { "—" } else { f.mime.as_str() };
            out.push_str("<div class=\"pe-diag-line\">");
            out.push_str(&format_str(format_args!(
                "[{i}] {} · {} · {mime}",
                esc_html(&f.name),
                fmt_bytes(f.size),
            )));
            out.push_str("</div>");
        }
    }

    out.push_str("<div class=\"pe-diag-section\">как читать</div>");
    out.push_str(
        "<div class=\"pe-diag-line\">\
         <b>items &gt; files</b> — DataTransfer содержит больше, чем \
         .files API. Мы читаем через <code>items</code> + \
         <code>getAsFile()</code> и получаем полный набор.\
         </div>",
    );
    out.push_str(
        "<div class=\"pe-diag-line\">\
         <b>items = files = 1</b> — либо скопировали один файл, либо \
         браузер отдал только primary. Различаем по браузеру (см. выше).\
         </div>",
    );
    out.push_str(
        "<div class=\"pe-diag-line\">\
         <b>items = 1, files = 0, plain не пуст</b> — обычный текст \
         (одна string-item). Не расхождение.\
         </div>",
    );

    out.push_str(
        "<button class=\"pe-diag-copy-btn\" data-action=\"copy-report\" \
         title=\"Скопировать текстовый отчёт диагностики в буфер обмена\">\
         📋 Скопировать отчёт</button>",
    );

    out.push_str("</details>");
    out
}

fn render_diag_line(out: &mut String, key: &str, value: &str) {
    out.push_str("<div class=\"pe-diag-line\"><span class=\"k\">");
    out.push_str(&esc_html(key));
    out.push_str(":</span> <span class=\"v\">");
    out.push_str(&esc_html(value));
    out.push_str("</span></div>");
}

fn render_async_result(text: &str, last: Option<&PastePayload>) -> String {
    let mut html = String::with_capacity(512);
    html.push_str("<div class=\"pe-card\">");

    html.push_str("<div class=\"pe-row\"><b>readText:</b> ");
    html.push_str(&format_str(format_args!(
        "{} симв.", text.chars().count()
    )));
    html.push_str("</div>");

    if !text.is_empty() {
        let (preview, truncated) = take_chars_with_flag(text, PLAIN_PREVIEW_LIMIT);
        html.push_str("<pre class=\"pe-pre\">");
        html.push_str(&esc_html(&preview));
        html.push_str("</pre>");
        if truncated {
            html.push_str(&format_str(format_args!(
                "<div class=\"pe-truncated\">показаны первые {PLAIN_PREVIEW_LIMIT} из {} символов</div>",
                text.chars().count(),
            )));
        }
    }

    html.push_str("<div class=\"pe-compare\">");
    match last {
        None => {
            html.push_str(
                "<div class=\"pe-muted\">Сначала вставьте что-нибудь в поле выше — \
                 тогда сравнение станет наглядным.</div>",
            );
        }
        Some(p) => {
            let mut missed: Vec<String> = Vec::new();
            if p.html.as_deref().map(|s| !s.is_empty()).unwrap_or(false) {
                missed.push("text/html".to_string());
            }
            if !p.uris.is_empty() {
                missed.push(format!("text/uri-list ({} шт.)", p.uris.len()));
            }
            if !p.files.is_empty() {
                missed.push(format!("Files ({} шт.)", p.files.len()));
            }

            if missed.is_empty() {
                html.push_str(
                    "<div class=\"pe-muted\">Различий нет — в буфере только plain text.</div>",
                );
                if !looks_like_url(&p.plain) {
                    html.push_str(
                        "<div class=\"pe-hint\">\
                         💡 <b>Попробуйте скопировать ссылку или HTML-фрагмент</b> — \
                         тогда Async API покажет только plain, а paste event увидит \
                         uri-list и html.\
                         </div>",
                    );
                } else {
                    html.push_str(
                        "<div class=\"pe-hint\">\
                         ℹ Скопирован URL как plain text. Firefox при копировании \
                         из адресной строки не добавляет <code>text/uri-list</code> — \
                         ссылка приходит только как текст.\
                         </div>",
                    );
                }
            } else {
                html.push_str("<div class=\"pe-row\"><b>Async не увидел:</b></div>");
                html.push_str("<ul class=\"pe-list\">");
                for m in &missed {
                    html.push_str("<li class=\"pe-warn\">");
                    html.push_str(&esc_html(m));
                    html.push_str("</li>");
                }
                html.push_str("</ul>");
            }

            if p.plain == text {
                html.push_str("<div class=\"pe-ok\">✓ plain совпадает</div>");
            } else {
                html.push_str("<div class=\"pe-diff\">");
                html.push_str(
                    "≠ plain не совпадает — буфер мог измениться между вставкой и чтением",
                );
                html.push_str("</div>");
            }
        }
    }
    html.push_str("</div>");

    html.push_str("</div>");
    html
}

fn mime_chip(mime: &str) -> String {
    let class = if mime == "Files" {
        "pe-mime pe-mime-files"
    } else {
        "pe-mime"
    };
    let mut html = String::with_capacity(64);
    html.push_str("<span class=\"");
    html.push_str(class);
    html.push_str("\">");
    html.push_str(&esc_html(mime));
    html.push_str("</span>");
    html
}

// ============================================================
//                    ОТЧЁТ ДИАГНОСТИКИ
// ============================================================

fn build_report_text(payload: &PastePayload) -> String {
    let d = &payload.diagnostic;
    let mut out = String::with_capacity(2048);

    out.push_str("# Clipboard paste — отчёт диагностики\n");
    out.push_str(&format_str(format_args!(
        "timestamp: {}\n",
        fmt_now_hms(),
    )));
    out.push_str(&format_str(format_args!(
        "browser: {}\n",
        detect_browser().label(),
    )));
    out.push_str(&format_str(format_args!(
        "user-agent: {}\n\n",
        user_agent(),
    )));

    out.push_str("## счётчики\n");
    out.push_str(&format_str(format_args!(
        "items.length: {}\n",
        d.items_count,
    )));
    out.push_str(&format_str(format_args!(
        "files.length: {}\n",
        d.files_count,
    )));
    out.push_str(&format_str(format_args!(
        "file-items (getAsFile): {}\n",
        d.files_from_items.len(),
    )));
    out.push_str(&format_str(format_args!(
        "has_discrepancy: {}\n\n",
        d.has_discrepancy(),
    )));

    out.push_str("## types\n");
    if d.types.is_empty() {
        out.push_str("(пусто)\n");
    } else {
        for (i, t) in d.types.iter().enumerate() {
            out.push_str(&format_str(format_args!("[{i}] {t}\n")));
        }
    }
    out.push('\n');

    out.push_str("## items[]\n");
    if d.items.is_empty() {
        out.push_str("(пусто)\n");
    } else {
        for it in &d.items {
            out.push_str(&format_str(format_args!(
                "[{}] kind={}",
                it.index, it.kind,
            )));
            if !it.mime.is_empty() {
                out.push_str(&format_str(format_args!(" mime={}", it.mime)));
            }
            if let Some(name) = &it.file_name {
                out.push_str(&format_str(format_args!(" name={name}")));
            }
            if let Some(sz) = it.file_size {
                out.push_str(&format_str(format_args!(" size={sz}")));
            }
            if let Some(m) = &it.file_mime {
                if !m.is_empty() {
                    out.push_str(&format_str(format_args!(" type={m}")));
                }
            }
            out.push('\n');
        }
    }
    out.push('\n');

    out.push_str("## files (.files API)\n");
    if d.files_from_files_list.is_empty() {
        out.push_str("(пусто)\n");
    } else {
        for (i, f) in d.files_from_files_list.iter().enumerate() {
            out.push_str(&format_str(format_args!(
                "[{i}] {} | {} Б | {}\n",
                f.name, f.size, f.mime,
            )));
        }
    }
    out.push('\n');

    out.push_str("## file-items (getAsFile)\n");
    if d.files_from_items.is_empty() {
        out.push_str("(пусто)\n");
    } else {
        for (i, f) in d.files_from_items.iter().enumerate() {
            out.push_str(&format_str(format_args!(
                "[{i}] {} | {} Б | {}\n",
                f.name, f.size, f.mime,
            )));
        }
    }
    out.push('\n');

    out.push_str("## payload — размеры\n");
    out.push_str(&format_str(format_args!(
        "plain: {} симв.\n",
        payload.plain.chars().count(),
    )));
    if let Some(h) = &payload.html {
        out.push_str(&format_str(format_args!(
            "html: {} симв.\n",
            h.chars().count(),
        )));
    }
    out.push_str(&format_str(format_args!(
        "uris: {}\n",
        payload.uris.len(),
    )));
    out.push_str(&format_str(format_args!(
        "files (итог): {}\n\n",
        payload.files.len(),
    )));

    out.push_str("## payload — plain (первые 500 символов)\n");
    out.push_str(&take_chars(&payload.plain, 500));
    out.push('\n');

    return out;
}

fn user_agent() -> String {
    web_sys::window()
        .map(|w| w.navigator())
        .and_then(|nav| nav.user_agent().ok())
        .unwrap_or_else(|| "—".to_string())
}

// ============================================================
//                    LOG
// ============================================================

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        let time = fmt_now_hms();
        let mut v = self.log.borrow_mut();
        v.push(LogEntry { time, text, kind });
        if v.len() > 12 {
            let drop_n = v.len() - 12;
            v.drain(0..drop_n);
        }
        drop(v);
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"pe-log-empty\">здесь появятся события paste и чтения</div>",
            );
            return;
        }
        let mut html = String::with_capacity(512);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
                LogKind::Async => "async",
                LogKind::Warn => "warn",
            };
            html.push_str("<div class=\"pe-log-entry ");
            html.push_str(cls);
            html.push_str("\"><span class=\"t\">");
            html.push_str(&esc_html(&e.time));
            html.push_str("</span><span class=\"m\">");
            html.push_str(&esc_html(&e.text));
            html.push_str("</span></div>");
        }
        self.ui.log.set_inner_html(&html);
        self.ui
            .log
            .set_scroll_top(self.ui.log.scroll_height() as f64);
    }
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn looks_like_url(s: &str) -> bool {
    let t = s.trim();
    if t.is_empty() || t.contains('\n') {
        return false;
    }
    t.starts_with("http://")
        || t.starts_with("https://")
        || t.starts_with("ftp://")
        || t.starts_with("www.")
        || t.starts_with("mailto:")
        || t.starts_with("file://")
}

fn take_chars_with_flag(s: &str, n: usize) -> (String, bool) {
    let total = s.chars().count();
    if total <= n {
        return (s.to_string(), false);
    }
    let mut preview: String = s.chars().take(n).collect();
    preview.push('…');
    return (preview, true);
}

fn take_chars(s: &str, n: usize) -> String {
    let total = s.chars().count();
    if total <= n {
        return s.to_string();
    }
    let mut out: String = s.chars().take(n).collect();
    out.push('…');
    return out;
}

fn fmt_bytes(n: u64) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = 1024.0 * 1024.0;
    let f = n as f64;
    if f < KB {
        return format_str(format_args!("{n} Б"));
    }
    if f < MB {
        return format_str(format_args!("{:.1} КБ", f / KB));
    }
    return format_str(format_args!("{:.2} МБ", f / MB));
}

fn short_error(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        let msg = clamp_first_line(&dx.message());
        return format_str(format_args!("{}: {}", dx.name(), msg));
    }
    if let Some(s) = e.as_string() {
        return clamp_first_line(&s);
    }
    if e.is_object() {
        let name = js_sys::Reflect::get(e, &JsValue::from_str("name"))
            .ok()
            .and_then(|v| v.as_string());
        let message = js_sys::Reflect::get(e, &JsValue::from_str("message"))
            .ok()
            .and_then(|v| v.as_string());
        match (name, message) {
            (Some(n), Some(m)) => {
                return format_str(format_args!("{n}: {}", clamp_first_line(&m)));
            }
            (Some(n), None) => return n,
            _ => {}
        }
    }
    if let Ok(f) = js_sys::Reflect::get(e, &JsValue::from_str("toString"))
        .and_then(|v| v.dyn_into::<js_sys::Function>())
    {
        if let Ok(s) = f.call0(e) {
            if let Some(s) = s.as_string() {
                if !s.is_empty() {
                    return clamp_first_line(&s);
                }
            }
        }
    }
    return format_str(format_args!("{e:?}"));
}

fn clamp_first_line(s: &str) -> String {
    let first = s.lines().next().unwrap_or("").trim();
    let mut out: String = first.chars().take(120).collect();
    if first.chars().count() > 120 {
        out.push('…');
    }
    return out;
}

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(),
        d.get_minutes(),
        d.get_seconds(),
    ));
}

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}

fn get_html_element(id: &str) -> Result<web_sys::HtmlElement, JsValue> {
    get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не HtmlElement")))
}

fn get_button(id: &str) -> Result<web_sys::HtmlButtonElement, JsValue> {
    get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}

fn set_click<F>(el: &web_sys::HtmlButtonElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```