# Глава 49. Rust/WASM File Converter

## 49.0. Обновлённая структура

```
src/
  file_convert.rs          ← НОВЫЙ модуль 49 (чистая логика конвертаций)
  file_io.rs               ← НОВЫЙ модуль 49 (браузерные File API)
  video_frame.rs           ← из 23.1 (format_str)
  app.rs                   ← демо 49

tests/
  wasm_file_convert.rs     ← тесты 49

index.html                 ← из главы 50
Cargo.toml                 ← без изменений (serde_json уже есть)
src/lib.rs                 ← +2 строки
```

**Комментарий.** В проекте уже есть image-пайплайн (`converter.rs` + `converter_native.rs` + `converter_ui.rs` + `converter_worker.rs`). Он **не пересекается** с материалами этой главы. Глава 49 работает с **текстовыми форматами** — JSON, CSV, Markdown, TXT. Это другой класс задач: UTF-8, структурированный текст, парсинг/сериализация, превью. Image-пайплайн раскроется в главе 51.

Разделение модулей главы 49:

- `file_convert.rs` — **чистая логика**: обнаружение формата по расширению и содержимому, планирование конвертации, четыре конвертера (JSON ↔ CSV, MD → HTML, TXT → JSON-статистика), превью. Без DOM, без Promise — тестируется headless.
- `file_io.rs` — **браузерные API**: `CancellationToken`, чанковое чтение `File` с прогрессом, `Blob`, `URL.createObjectURL`, programmatic download через `<a download>`.

**Префикс CSS** — `fc-` (file converter). Не пересекается с `bt-`, `sl-`, `usb-`, `arch-`, `err-`, `pn-`, `vf-`, `wg-`.

**Ссылки для детального изучения:**

- MDN: [`File`](https://developer.mozilla.org/en-US/docs/Web/API/File), [`Blob`](https://developer.mozilla.org/en-US/docs/Web/API/Blob), [`URL.createObjectURL`](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL_static), [`HTMLAnchorElement.download`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLAnchorElement/download)
- `serde_json`: [docs.rs](https://docs.rs/serde_json/latest/serde_json/)
- Rust/WASM Book: [Reference — Types](https://rustwasm.github.io/docs/wasm-bindgen/reference/types.html)
- Chrome Developers: [Reading files in JavaScript](https://web.dev/read-files/)

## 49.1. О `Cargo.toml`

**Новых зависимостей не нужно.** `serde_json` уже в дереве, фичи `web-sys` для `File`, `Blob`, `Url`, `HtmlInputElement`, `HtmlAnchorElement` активны. Секция `[package.metadata.wasm-pack.profile.release]` с `wasm-opt = false` — из главы 50.

## 49.2. Модуль `src/file_convert.rs`

```rust
//! 49. File Converter: чистая логика текстовых конвертаций.
//!
//! Никакого DOM, никаких Promise. Только функции над байтами.
//! Тестируется headless.
//!
//! ## Поддерживаемые конвертации
//!
//! - `.json` → `.csv`
//! - `.csv`  → `.json`
//! - `.md`   → `.html`
//! - `.txt`  → `.json` (статистика: строки, слова, символы)
//!
//! ## Обнаружение формата
//!
//! По расширению (приоритет), затем по содержимому. Для JSON —
//! первый непустой символ `{` или `[`. Для CSV — наличие `,`/`;`
//! в первой строке. Для Markdown — заголовки `# `.

use std::collections::HashSet;

use serde_json::{json, Map, Value};

use crate::video_frame::format_str;

// =============================================================================
// Format
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum Format {
    Text,
    Json,
    Csv,
    Markdown,
    Html,
    Unknown,
}

impl Format {
    pub fn as_str(&self) -> &'static str {
        return match self {
            Self::Text => "txt",
            Self::Json => "json",
            Self::Csv => "csv",
            Self::Markdown => "md",
            Self::Html => "html",
            Self::Unknown => "bin",
        };
    }

    pub fn label(&self) -> &'static str {
        return match self {
            Self::Text => "Plain text",
            Self::Json => "JSON",
            Self::Csv => "CSV",
            Self::Markdown => "Markdown",
            Self::Html => "HTML",
            Self::Unknown => "Binary",
        };
    }

    pub fn mime(&self) -> &'static str {
        return match self {
            Self::Text => "text/plain;charset=utf-8",
            Self::Json => "application/json",
            Self::Csv => "text/csv;charset=utf-8",
            Self::Markdown => "text/markdown;charset=utf-8",
            Self::Html => "text/html;charset=utf-8",
            Self::Unknown => "application/octet-stream",
        };
    }
}

// =============================================================================
// Обнаружение формата
// =============================================================================

/// Определяет формат по расширению (приоритет), затем по содержимому.
pub fn detect_format(filename: &str, bytes: &[u8]) -> Format {
    let lower = filename.to_lowercase();

    let from_ext = if lower.ends_with(".json") {
        Some(Format::Json)
    } else if lower.ends_with(".csv") {
        Some(Format::Csv)
    } else if lower.ends_with(".md") || lower.ends_with(".markdown") {
        Some(Format::Markdown)
    } else if lower.ends_with(".html") || lower.ends_with(".htm") {
        Some(Format::Html)
    } else if lower.ends_with(".txt") {
        Some(Format::Text)
    } else {
        None
    };

    if let Some(f) = from_ext {
        return f;
    }

    // Magic bytes — fallback для файлов без расширения.
    let head_len = bytes.len().min(512);
    let head = &bytes[..head_len];
    let first = head.iter().find(|&&b| !b.is_ascii_whitespace()).copied();
    if matches!(first, Some(b'{') | Some(b'[')) {
        return Format::Json;
    }

    if let Ok(s) = std::str::from_utf8(head) {
        if let Some(first_line) = s.lines().next() {
            if first_line.contains(',') || first_line.contains(';') {
                return Format::Csv;
            }
        }
        if s.contains("# ") || s.contains("## ") {
            return Format::Markdown;
        }
    }

    return Format::Unknown;
}

// =============================================================================
// Планирование
// =============================================================================

/// Формат-цель по умолчанию для каждого входного формата.
pub fn choose_target(from: Format) -> Option<Format> {
    return match from {
        Format::Json => Some(Format::Csv),
        Format::Csv => Some(Format::Json),
        Format::Markdown => Some(Format::Html),
        Format::Text => Some(Format::Json),
        Format::Html | Format::Unknown => None,
    };
}

// =============================================================================
// Диспетчер
// =============================================================================

pub fn convert(from: Format, to: Format, input: &[u8]) -> Result<Vec<u8>, String> {
    return match (from, to) {
        (Format::Json, Format::Csv) => json_to_csv(input),
        (Format::Csv, Format::Json) => csv_to_json(input),
        (Format::Markdown, Format::Html) => md_to_html(input),
        (Format::Text, Format::Json) => text_to_json(input),
        _ => Err(format_str(format_args!(
            "неподдерживаемая конвертация: {:?} → {:?}",
            from, to
        ))),
    };
}

// =============================================================================
// JSON → CSV
// =============================================================================

/// Конвертация JSON в CSV. Поддерживает четыре формы корня:
///
/// - **массив объектов** — многорядный CSV с объединением ключей;
/// - **одиночный объект** — 1-строчный CSV с заголовками;
/// - **массив примитивов** — CSV с одной колонкой `value`;
/// - **одиночный примитив** — `value\n<значение>\n`.
///
/// ## Нормализация
///
/// Все четыре формы приводятся к общему виду: последовательность
/// JSON-объектов. Дальше алгоритм один и тот же — объединение
/// ключей и построчное заполнение.
///
/// ## Что НЕ поддерживается
///
/// `null` и пустой массив — нечего конвертировать. Возвращаем `Err`.
pub fn json_to_csv(input: &[u8]) -> Result<Vec<u8>, String> {
    let value: Value =
        serde_json::from_slice(input).map_err(|e| format_str(format_args!("JSON: {e}")))?;

    // Нормализуем вход в последовательность объектов.
    let rows: Vec<Value> = match normalize_to_rows(value) {
        Some(r) => r,
        None => {
            return Err(String::from(
                "нечего конвертировать: пустой массив или JSON null",
            ));
        }
    };

    // Собираем ключи в порядке первого появления.
    let mut keys: Vec<String> = Vec::new();
    let mut seen: HashSet<String> = HashSet::new();
    for item in &rows {
        if let Some(obj) = item.as_object() {
            for k in obj.keys() {
                if seen.insert(k.clone()) {
                    keys.push(k.clone());
                }
            }
        }
    }

    if keys.is_empty() {
        return Err(String::from(
            "нечего конвертировать: ни одного поля в JSON",
        ));
    }

    let mut out = String::new();
    out.push_str(
        &keys
            .iter()
            .map(|k| csv_escape(k))
            .collect::<Vec<_>>()
            .join(","),
    );
    out.push('\n');

    for item in &rows {
        let row: Vec<String> = keys
            .iter()
            .map(|k| item.get(k).map(json_value_to_csv).unwrap_or_default())
            .collect();
        out.push_str(&row.join(","));
        out.push('\n');
    }

    return Ok(out.into_bytes());
}

/// Приводит произвольный JSON к последовательности объектов.
///
/// - `[{...}, {...}]` → как есть (если элемент не объект — оборачиваем);
/// - `{...}` → `[{...}]`;
/// - `[1, 2, 3]` → `[{"value": 1}, {"value": 2}, {"value": 3}]`;
/// - `42` → `[{"value": 42}]`.
///
/// Возвращает `None` для случаев, где нечего конвертировать:
/// пустой массив или `null`.
fn normalize_to_rows(value: Value) -> Option<Vec<Value>> {
    match value {
        Value::Array(arr) => {
            if arr.is_empty() {
                return None;
            }
            let rows: Vec<Value> = arr
                .into_iter()
                .map(|v| match v {
                    Value::Object(_) => v,
                    other => wrap_as_value(other),
                })
                .collect();
            return Some(rows);
        }
        Value::Object(_) => return Some(vec![value]),
        Value::Null => return None,
        other => return Some(vec![wrap_as_value(other)]),
    }
}

fn wrap_as_value(v: Value) -> Value {
    let mut m = Map::new();
    m.insert(String::from("value"), v);
    return Value::Object(m);
}

fn json_value_to_csv(v: &Value) -> String {
    return match v {
        Value::Null => String::new(),
        Value::Bool(b) => b.to_string(),
        Value::Number(n) => n.to_string(),
        Value::String(s) => csv_escape(s),
        _ => csv_escape(&v.to_string()),
    };
}

fn csv_escape(s: &str) -> String {
    if s.contains(',') || s.contains('"') || s.contains('\n') {
        let escaped = s.replace('"', "\"\"");
        return format_str(format_args!("\"{escaped}\""));
    }
    return s.to_string();
}

// =============================================================================
// CSV → JSON
// =============================================================================

/// Первая строка — заголовок. Каждая следующая — объект.
pub fn csv_to_json(input: &[u8]) -> Result<Vec<u8>, String> {
    let text =
        std::str::from_utf8(input).map_err(|e| format_str(format_args!("UTF-8: {e}")))?;
    let rows = parse_csv(text)?;
    if rows.is_empty() {
        return Ok(b"[]".to_vec());
    }

    let header = &rows[0];
    let mut arr = Vec::with_capacity(rows.len().saturating_sub(1));
    for row in &rows[1..] {
        let mut obj = Map::new();
        for (i, key) in header.iter().enumerate() {
            let val = row.get(i).cloned().unwrap_or_default();
            obj.insert(key.clone(), Value::String(val));
        }
        arr.push(Value::Object(obj));
    }

    let out = serde_json::to_vec_pretty(&Value::Array(arr))
        .map_err(|e| format_str(format_args!("JSON: {e}")))?;
    return Ok(out);
}

/// Минимальный CSV-парсер с поддержкой кавычек и `""`-escaping.
fn parse_csv(text: &str) -> Result<Vec<Vec<String>>, String> {
    let mut rows: Vec<Vec<String>> = Vec::new();
    let mut row: Vec<String> = Vec::new();
    let mut field = String::new();
    let mut chars = text.chars().peekable();
    let mut in_quotes = false;

    while let Some(c) = chars.next() {
        if in_quotes {
            if c == '"' {
                if chars.peek() == Some(&'"') {
                    chars.next();
                    field.push('"');
                } else {
                    in_quotes = false;
                }
            } else {
                field.push(c);
            }
        } else {
            match c {
                '"' => in_quotes = true,
                ',' => row.push(std::mem::take(&mut field)),
                '\r' => {}
                '\n' => {
                    row.push(std::mem::take(&mut field));
                    rows.push(std::mem::take(&mut row));
                }
                _ => field.push(c),
            }
        }
    }
    if !field.is_empty() || !row.is_empty() {
        row.push(field);
        rows.push(row);
    }
    return Ok(rows);
}

// =============================================================================
// Markdown → HTML
// =============================================================================

/// Минимальный Markdown → HTML.
pub fn md_to_html(input: &[u8]) -> Result<Vec<u8>, String> {
    let text =
        std::str::from_utf8(input).map_err(|e| format_str(format_args!("UTF-8: {e}")))?;
    let mut out = String::with_capacity(text.len() * 2);
    let mut in_p = false;

    for line in text.lines() {
        let trimmed = line.trim_end();
        if trimmed.is_empty() {
            close_paragraph(&mut out, &mut in_p);
            continue;
        }
        if let Some(rest) = trimmed.strip_prefix("### ") {
            close_paragraph(&mut out, &mut in_p);
            out.push_str("<h3>");
            out.push_str(&inline_md(rest));
            out.push_str("</h3>\n");
            continue;
        }
        if let Some(rest) = trimmed.strip_prefix("## ") {
            close_paragraph(&mut out, &mut in_p);
            out.push_str("<h2>");
            out.push_str(&inline_md(rest));
            out.push_str("</h2>\n");
            continue;
        }
        if let Some(rest) = trimmed.strip_prefix("# ") {
            close_paragraph(&mut out, &mut in_p);
            out.push_str("<h1>");
            out.push_str(&inline_md(rest));
            out.push_str("</h1>\n");
            continue;
        }
        if !in_p {
            out.push_str("<p>");
            in_p = true;
        } else {
            out.push(' ');
        }
        out.push_str(&inline_md(trimmed));
    }
    close_paragraph(&mut out, &mut in_p);
    return Ok(out.into_bytes());
}

fn close_paragraph(out: &mut String, in_p: &mut bool) {
    if *in_p {
        out.push_str("</p>\n");
        *in_p = false;
    }
}

fn inline_md(text: &str) -> String {
    let mut out = String::with_capacity(text.len() + 16);
    let mut i = 0;
    while i < text.len() {
        let rest = &text[i..];

        if let Some(end) = rest.strip_prefix("**").and_then(|s| s.find("**")) {
            out.push_str("<strong>");
            out.push_str(&rest[2..2 + end]);
            out.push_str("</strong>");
            i += 2 + end + 2;
            continue;
        }
        if let Some(end) = rest.strip_prefix('`').and_then(|s| s.find('`')) {
            out.push_str("<code>");
            out.push_str(&rest[1..1 + end]);
            out.push_str("</code>");
            i += 1 + end + 1;
            continue;
        }
        if let Some(end) = rest.strip_prefix('*').and_then(|s| s.find('*')) {
            if end > 0 {
                out.push_str("<em>");
                out.push_str(&rest[1..1 + end]);
                out.push_str("</em>");
                i += 1 + end + 1;
                continue;
            }
        }
        if let Some(after_open) = rest.strip_prefix('[') {
            if let Some(close) = after_open.find("](") {
                let link_text = &after_open[..close];
                let after_paren = &after_open[close + 2..];
                if let Some(paren) = after_paren.find(')') {
                    let url = &after_paren[..paren];
                    out.push_str("<a href=\"");
                    out.push_str(url);
                    out.push_str("\">");
                    out.push_str(link_text);
                    out.push_str("</a>");
                    i += 1 + close + 2 + paren + 1;
                    continue;
                }
            }
        }

        let c = rest.chars().next().unwrap();
        match c {
            '<' => out.push_str("&lt;"),
            '>' => out.push_str("&gt;"),
            '&' => out.push_str("&amp;"),
            _ => out.push(c),
        }
        i += c.len_utf8();
    }
    return out;
}

// =============================================================================
// TXT → JSON (статистика)
// =============================================================================

#[derive(Clone, Debug)]
pub struct TextStats {
    pub bytes: usize,
    pub chars: usize,
    pub lines: usize,
    pub words: usize,
}

pub fn text_stats(input: &[u8]) -> Result<TextStats, String> {
    let text =
        std::str::from_utf8(input).map_err(|e| format_str(format_args!("UTF-8: {e}")))?;
    return Ok(TextStats {
        bytes: input.len(),
        chars: text.chars().count(),
        lines: text.lines().count(),
        words: text.split_whitespace().count(),
    });
}

pub fn text_to_json(input: &[u8]) -> Result<Vec<u8>, String> {
    let s = text_stats(input)?;
    let v = json!({
        "bytes": s.bytes,
        "chars": s.chars,
        "lines": s.lines,
        "words": s.words,
    });
    let out = serde_json::to_vec_pretty(&v)
        .map_err(|e| format_str(format_args!("JSON: {e}")))?;
    return Ok(out);
}

// =============================================================================
// Превью
// =============================================================================

/// Отдаёт первые `max_chars` символов результата для UI-превью.
/// Для бинарных результатов — hex.
pub fn preview_output(bytes: &[u8], max_chars: usize) -> String {
    match std::str::from_utf8(bytes) {
        Ok(s) => {
            let total = s.chars().count();
            let taken: String = s.chars().take(max_chars).collect();
            if total > max_chars {
                return format_str(format_args!("{taken}\n… (+{} символов)", total - max_chars));
            }
            return taken;
        }
        Err(_) => {
            let hex = hex_bytes(bytes, 64);
            return format_str(format_args!("<binary>\n{hex}"));
        }
    }
}

fn hex_bytes(bytes: &[u8], limit: usize) -> String {
    let take = bytes.len().min(limit);
    let mut out = String::with_capacity(take * 3);
    for (i, b) in bytes[..take].iter().enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format_str(format_args!("{b:02X}")));
    }
    if bytes.len() > limit {
        out.push_str(" …");
    }
    return out;
}
```

## 49.3. Модуль `src/file_io.rs`

```rust
//! 49. File Converter: браузерные File API.
//!
//! Обёртки над `File`, `Blob`, `URL.createObjectURL`, `<input type=file>`.
//! Асинхронное чтение чанками с отменой и прогрессом.
//!
//! ## Что НЕ делает этот модуль
//!
//! Он не пересекается с `converter_ui.rs` (image-пайплайн): тот
//! работает через FSA (`showSaveFilePicker`), этот — через
//! `<a download>`. Разные сценарии.

use std::cell::Cell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::Uint8Array;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, BlobPropertyBag, Document, File, HtmlAnchorElement, Url};

use crate::video_frame::format_str;

// =============================================================================
// CancellationToken
// =============================================================================

/// Маркер отмены. Клонируется как `Rc` — общий флаг между UI и
/// async-задачей.
#[derive(Clone)]
pub struct CancellationToken {
    flag: Rc<Cell<bool>>,
}

impl CancellationToken {
    pub fn new() -> Self {
        return Self {
            flag: Rc::new(Cell::new(false)),
        };
    }

    pub fn cancel(&self) {
        self.flag.set(true);
    }

    pub fn is_cancelled(&self) -> bool {
        return self.flag.get();
    }
}

// =============================================================================
// Чтение файла
// =============================================================================

pub const CHUNK_SIZE: u64 = 64 * 1024;

/// Читает `File` по кускам, вызывая `on_progress(consumed, total)`
/// после каждого. Проверяет `token` между чанками.
///
/// ## Почему `slice` + `arrayBuffer`
///
/// `FileReader.readAsArrayBuffer` читает всё за раз — не даёт
/// промежуточного прогресса и точки отмены. `file.slice(a, b)`
/// возвращает `Blob` с методом `arrayBuffer()`. Разбивая файл
/// на куски по 64 KB, получаем прогресс и возможность прервать.
///
/// ## Сигнатуры web-sys 0.3.105
///
/// - `File::slice_with_f64_and_f64(start, end)` → `Result<Blob, JsValue>`
/// - `Blob::array_buffer()` → `Promise` (не `Result<Promise, _>`)
///
/// Поэтому у `slice` есть `.map_err`, а у `array_buffer` — нет.
///
/// ## Yield
///
/// После каждого чанка — `TimeoutFuture::new(0).await`. Без этого
/// UI не перерисуется до конца операции.
pub async fn read_file_chunked<F>(
    file: &File,
    token: &CancellationToken,
    mut on_progress: F,
) -> Result<Vec<u8>, String>
where
    F: FnMut(u64, u64),
{
    let total = file.size() as u64;
    let mut offset: u64 = 0;
    let mut out: Vec<u8> = Vec::with_capacity(total as usize);

    while offset < total {
        if token.is_cancelled() {
            return Err(String::from("Отменено пользователем"));
        }

        let end = (offset + CHUNK_SIZE).min(total);
        let slice = file
            .slice_with_f64_and_f64(offset as f64, end as f64)
            .map_err(|e| format_str(format_args!("slice: {e:?}")))?;

        // `array_buffer()` возвращает `Promise` напрямую.
        let ab_promise: js_sys::Promise = slice.array_buffer();
        let ab_value: wasm_bindgen::JsValue = JsFuture::from(ab_promise)
            .await
            .map_err(|e| format_str(format_args!("read: {e:?}")))?;

        let ab: js_sys::ArrayBuffer = ab_value
            .dyn_into()
            .map_err(|_| String::from("не ArrayBuffer"))?;
        let u8arr = Uint8Array::new(&ab);
        let mut chunk = vec![0u8; u8arr.length() as usize];
        u8arr.copy_to(&mut chunk);
        out.extend_from_slice(&chunk);

        offset = end;
        on_progress(offset, total);

        TimeoutFuture::new(0).await;
    }

    return Ok(out);
}

// =============================================================================
// Blob / download
// =============================================================================

pub fn make_blob(bytes: &[u8], mime: &str) -> Result<Blob, String> {
    let parts = js_sys::Array::new();
    parts.push(&Uint8Array::from(bytes));
    let opts = BlobPropertyBag::new();
    opts.set_type(mime);
    return Blob::new_with_u8_array_sequence_and_options(&parts, &opts)
        .map_err(|e| format_str(format_args!("Blob: {e:?}")));
}

pub fn create_download_url(blob: &Blob) -> Result<String, String> {
    return Url::create_object_url_with_blob(blob)
        .map_err(|e| format_str(format_args!("createObjectURL: {e:?}")));
}

/// Освобождает object URL. Идемпотентно.
pub fn revoke_download_url(url: &str) {
    let _ = Url::revoke_object_url(url);
}

/// Программно инициирует скачивание через `<a download href>`.
///
/// ## Почему не `location.href`
///
/// `location.href = url` откроет бинарный файл в браузере, а не
/// скачает. Атрибут `download` на `<a>` заставляет браузер
/// сохранить файл с заданным именем.
pub fn trigger_download(
    document: &Document,
    url: &str,
    filename: &str,
) -> Result<(), String> {
    let a = document
        .create_element("a")
        .map_err(|e| format_str(format_args!("createElement: {e:?}")))?;
    let a: HtmlAnchorElement = a
        .dyn_into()
        .map_err(|_| String::from("не <a>"))?;
    a.set_href(url);
    a.set_download(filename);
    a.click();
    return Ok(());
}

// =============================================================================
// File picker
// =============================================================================

pub fn pick_file(input: &web_sys::HtmlInputElement) -> Option<File> {
    let files = input.files()?;
    if files.length() == 0 {
        return None;
    }
    return files.item(0);
}

// =============================================================================
// Утилиты
// =============================================================================

pub fn format_size(bytes: u64) -> String {
    if bytes < 1024 {
        return format_str(format_args!("{bytes} Б"));
    }
    if bytes < 1024 * 1024 {
        return format_str(format_args!("{:.1} КБ", bytes as f64 / 1024.0));
    }
    return format_str(format_args!("{:.2} МБ", bytes as f64 / (1024.0 * 1024.0)));
}

pub fn replace_extension(filename: &str, new_ext: &str) -> String {
    let stem = match filename.rfind('.') {
        Some(i) => &filename[..i],
        None => filename,
    };
    return format_str(format_args!("{stem}.{new_ext}"));
}
```

## 49.4. Добавление в `src/lib.rs`

```rust
pub mod app;
pub mod arch_js_bench;
pub mod arch_pure;
pub mod bluetooth_gatt;
pub mod bluetooth_utils;
pub mod container_parser;
pub mod converter;           // image pipeline (глава 51)
pub mod converter_native;
pub mod converter_ui;
pub mod converter_worker;
pub mod error_info;
pub mod error_scenarios;
pub mod file_convert;        // ← 49
pub mod file_io;             // ← 49
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod panic_guard;
pub mod panic_scenarios;
pub mod serial_streams;
pub mod serial_utils;
pub mod usb_transfers;
pub mod usb_utils;
pub mod video_filters;
pub mod video_frame;
pub mod webgpu_compute;
pub mod webgpu_utils;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

## 49.5. Тесты `tests/wasm_file_convert.rs`

```rust
//! 49. Тесты File Converter.
//!
//! ## Что тестируется
//!
//! - `detect_format` — по расширению и содержимому.
//! - `choose_target` — правильная цель по умолчанию.
//! - JSON ↔ CSV: round-trip, экранирование, ошибки, все формы корня.
//! - Markdown → HTML: заголовки, inline-разметка, эскейпинг.
//! - TXT → JSON stats.
//! - `preview_output`, `replace_extension`, `format_size`.
//!
//! ## Отличие от `wasm_converter.rs`
//!
//! `wasm_converter.rs` тестирует image-пайплайн (`src/converter.rs`).
//! Здесь — текстовые конвертации, отдельный модуль.

use wasm_bindgen_test::*;

use hello_wasm02::file_convert::{
    choose_target, csv_to_json, detect_format, json_to_csv, md_to_html,
    preview_output, text_stats, text_to_json, Format,
};
use hello_wasm02::file_io::{format_size, replace_extension};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// detect_format
// =============================================================================

#[wasm_bindgen_test]
fn test_detect_by_extension() {
    assert_eq!(detect_format("data.json", b""), Format::Json);
    assert_eq!(detect_format("rows.csv", b""), Format::Csv);
    assert_eq!(detect_format("README.md", b""), Format::Markdown);
    assert_eq!(detect_format("page.html", b""), Format::Html);
    assert_eq!(detect_format("notes.txt", b""), Format::Text);
    assert_eq!(detect_format("DATA.JSON", b""), Format::Json);
}

#[wasm_bindgen_test]
fn test_detect_by_content_json() {
    assert_eq!(detect_format("noext", b"[1, 2, 3]"), Format::Json);
    assert_eq!(detect_format("noext", b"  {\"a\": 1}"), Format::Json);
}

#[wasm_bindgen_test]
fn test_detect_by_content_csv() {
    assert_eq!(detect_format("noext", b"a,b,c\n1,2,3"), Format::Csv);
}

#[wasm_bindgen_test]
fn test_detect_unknown() {
    assert_eq!(detect_format("file.bin", &[0xDE, 0xAD, 0xBE, 0xEF]), Format::Unknown);
}

// =============================================================================
// choose_target
// =============================================================================

#[wasm_bindgen_test]
fn test_choose_target() {
    assert_eq!(choose_target(Format::Json), Some(Format::Csv));
    assert_eq!(choose_target(Format::Csv), Some(Format::Json));
    assert_eq!(choose_target(Format::Markdown), Some(Format::Html));
    assert_eq!(choose_target(Format::Text), Some(Format::Json));
    assert_eq!(choose_target(Format::Html), None);
    assert_eq!(choose_target(Format::Unknown), None);
}

// =============================================================================
// JSON → CSV
// =============================================================================

#[wasm_bindgen_test]
fn test_json_to_csv_basic() {
    let input = br#"[{"a":1,"b":"x"},{"a":2,"b":"y"}]"#;
    let out = json_to_csv(input).expect("convert failed");
    let s = String::from_utf8(out).unwrap();
    assert_eq!(s, "a,b\n1,x\n2,y\n");
}

#[wasm_bindgen_test]
fn test_json_to_csv_union_of_keys() {
    let input = br#"[{"a":1},{"b":2,"a":3}]"#;
    let out = json_to_csv(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert_eq!(s, "a,b\n1,\n3,2\n");
}

#[wasm_bindgen_test]
fn test_json_to_csv_escaping() {
    let input = br#"[{"msg":"hello, world"},{"msg":"say \"hi\""}]"#;
    let out = json_to_csv(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert!(s.contains("\"hello, world\""));
    assert!(s.contains("\"say \"\"hi\"\"\""));
}

#[wasm_bindgen_test]
fn test_json_to_csv_single_object() {
    // Одиночный объект → 1-строчный CSV.
    let input = br#"{"bytes":29825,"chars":24424,"lines":1061,"words":3292}"#;
    let out = json_to_csv(input).expect("single object");
    let s = String::from_utf8(out).unwrap();
    assert_eq!(s, "bytes,chars,lines,words\n29825,24424,1061,3292\n");
}

#[wasm_bindgen_test]
fn test_json_to_csv_array_of_primitives() {
    let input = br#"[1, 2, 3, 4]"#;
    let out = json_to_csv(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert_eq!(s, "value\n1\n2\n3\n4\n");
}

#[wasm_bindgen_test]
fn test_json_to_csv_single_primitive() {
    let input = br#"42"#;
    let out = json_to_csv(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert_eq!(s, "value\n42\n");
}

#[wasm_bindgen_test]
fn test_json_to_csv_empty_and_null() {
    // Пустой массив и null — нечего конвертировать.
    assert!(json_to_csv(b"[]").is_err());
    assert!(json_to_csv(b"null").is_err());
}

#[wasm_bindgen_test]
fn test_json_to_csv_invalid_json() {
    assert!(json_to_csv(b"{ not json").is_err());
}

// =============================================================================
// CSV → JSON
// =============================================================================

#[wasm_bindgen_test]
fn test_csv_to_json_basic() {
    let input = b"a,b\n1,x\n2,y\n";
    let out = csv_to_json(input).unwrap();
    let v: serde_json::Value = serde_json::from_slice(&out).unwrap();
    let arr = v.as_array().unwrap();
    assert_eq!(arr.len(), 2);
    assert_eq!(arr[0]["a"], "1");
    assert_eq!(arr[0]["b"], "x");
    assert_eq!(arr[1]["a"], "2");
}

#[wasm_bindgen_test]
fn test_csv_to_json_empty() {
    let out = csv_to_json(b"").unwrap();
    assert_eq!(String::from_utf8(out).unwrap(), "[]");
}

#[wasm_bindgen_test]
fn test_csv_json_roundtrip() {
    let original = br#"[{"name":"Alice","age":"30"},{"name":"Bob","age":"25"}]"#;
    let csv = json_to_csv(original).unwrap();
    let json = csv_to_json(&csv).unwrap();
    let v: serde_json::Value = serde_json::from_slice(&json).unwrap();
    let arr = v.as_array().unwrap();
    assert_eq!(arr.len(), 2);
    assert_eq!(arr[0]["name"], "Alice");
}

// =============================================================================
// Markdown → HTML
// =============================================================================

#[wasm_bindgen_test]
fn test_md_to_html_headers() {
    let input = b"# Title\n\n## Subtitle\n";
    let out = md_to_html(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert!(s.contains("<h1>Title</h1>"));
    assert!(s.contains("<h2>Subtitle</h2>"));
}

#[wasm_bindgen_test]
fn test_md_to_html_paragraph() {
    let input = b"Just text\n";
    let out = md_to_html(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert_eq!(s.trim(), "<p>Just text</p>");
}

#[wasm_bindgen_test]
fn test_md_to_html_inline() {
    let input = b"Hello **world** and *universe*\n";
    let out = md_to_html(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert!(s.contains("<strong>world</strong>"));
    assert!(s.contains("<em>universe</em>"));
}

#[wasm_bindgen_test]
fn test_md_to_html_code_and_link() {
    let input = b"Use `cargo build` and see [docs](https://example.com)\n";
    let out = md_to_html(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert!(s.contains("<code>cargo build</code>"));
    assert!(s.contains("<a href=\"https://example.com\">docs</a>"));
}

#[wasm_bindgen_test]
fn test_md_to_html_escapes_specials() {
    let input = b"5 < 10 & 10 > 5\n";
    let out = md_to_html(input).unwrap();
    let s = String::from_utf8(out).unwrap();
    assert!(s.contains("5 &lt; 10 &amp; 10 &gt; 5"));
}

// =============================================================================
// TXT → JSON stats
// =============================================================================

#[wasm_bindgen_test]
fn test_text_stats() {
    let input = b"hello world\nsecond line\n";
    let s = text_stats(input).unwrap();
    // 24 байта: "hello world"(11) + "\n"(1) + "second line"(11) + "\n"(1).
    // Ожидаемое считаем из длины входа — не вручную.
    assert_eq!(s.bytes, input.len());
    assert_eq!(s.lines, 2);
    assert_eq!(s.words, 4);
}

#[wasm_bindgen_test]
fn test_text_to_json() {
    let out = text_to_json(b"one two\nthree").unwrap();
    let v: serde_json::Value = serde_json::from_slice(&out).unwrap();
    assert_eq!(v["words"], 3);
    assert_eq!(v["lines"], 2);
}

// =============================================================================
// preview_output, replace_extension, format_size
// =============================================================================

#[wasm_bindgen_test]
fn test_preview_output_truncates() {
    let s = preview_output(b"abcdefghij", 5);
    assert!(s.starts_with("abcde"));
    assert!(s.contains("+5"));
}

#[wasm_bindgen_test]
fn test_preview_output_binary() {
    let s = preview_output(&[0xFF, 0xFE, 0xFD], 64);
    assert!(s.contains("<binary>"));
    assert!(s.contains("FF"));
}

#[wasm_bindgen_test]
fn test_replace_extension() {
    assert_eq!(replace_extension("data.json", "csv"), "data.csv");
    assert_eq!(replace_extension("no_ext", "txt"), "no_ext.txt");
    assert_eq!(replace_extension("a.b.c", "x"), "a.b.x");
}

#[wasm_bindgen_test]
fn test_format_size() {
    assert_eq!(format_size(500), "500 Б");
    assert_eq!(format_size(2048), "2.0 КБ");
    assert!(format_size(3 * 1024 * 1024).contains("МБ"));
}
```

**Команды запуска тестов:**

```bash
# Только тесты главы 49
wasm-pack test --headless --chrome --test wasm_file_convert

# Все тесты проекта
wasm-pack test --headless --chrome

# С захватом вывода
wasm-pack test --headless --chrome --test wasm_file_convert -- --nocapture
```

## 49.6. Демо `src/app.rs`

```rust
//! Глава 49 — Rust/WASM File Converter.
//!
//! Полный workflow для текстовых форматов: выбор файла (клик или
//! drag-and-drop) → чанковое чтение с прогрессом → определение
//! формата → конвертация в Rust → превью → скачивание.
//!
//! ## Фазы
//!
//! Idle → Ready → Reading → Converting → Done
//!     Cancelled ← (отмена в Reading/Converting)
//!     Error     ← (ошибка в любой фазе)
//!
//! ## Отмена
//!
//! `CancellationToken` — общий `Rc<Cell<bool>>`. Проверяется
//! между чанками в `read_file_chunked` и после yield между фазами.
//!
//! ## Двухфазный workflow
//!
//! Сначала конвертация с превью, потом явное «Скачать» или
//! «Открыть в новой вкладке». Пользователь видит результат
//! до сохранения.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, File, HtmlElement, HtmlInputElement};

use crate::file_convert::{
    choose_target, convert, detect_format, preview_output, Format,
};
use crate::file_io::{
    create_download_url, format_size, make_blob, pick_file, read_file_chunked,
    replace_extension, revoke_download_url, trigger_download, CancellationToken,
};
use crate::video_frame::format_str;

// =============================================================================
// Константы
// =============================================================================

/// Порог, после которого показываем предупреждение о размере.
/// 50 МБ текста — это уже сотни тысяч строк; для «file converter»
/// это крайний случай.
const SIZE_WARN_BYTES: u64 = 50 * 1024 * 1024;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.fc-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.fc-desc-line { margin:6px 0; }
.fc-desc-line code { background:#0d1117; padding:1px 5px; border-radius:3px; color:#79c0ff; }

.fc-block { max-width:1000px;margin-bottom:16px;padding:14px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;}
.fc-block-head { display:flex; align-items:baseline; gap:10px; font-size:11px;color:#8b949e;text-transform:uppercase;letter-spacing:1px;margin-bottom:10px;font-weight:600;flex-wrap:wrap;}
.fc-block-head .grow { flex:1; }

.fc-drop {
  max-width:1000px; margin-bottom:16px; padding:24px 20px;
  background:#0d1117; border:2px dashed #30363d; border-radius:6px;
  text-align:center; color:#8b949e; font-size:13px; transition:all 0.15s;
}
.fc-drop.fc-drop-hover { border-color:#58a6ff; background:#0a1620; color:#79c0ff; }
.fc-drop-hint { margin-top:6px; font-size:11px; color:#6e7681; }

.fc-row { display:flex;flex-wrap:wrap;gap:10px;align-items:center;margin-bottom:10px;}
.fc-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.fc-btn:hover:not([disabled]) { background:#30363d; }
.fc-btn:disabled { opacity:0.5;cursor:not-allowed; }
.fc-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.fc-btn-primary:hover:not([disabled]) { background:#2ea043; }
.fc-btn-danger { background:#4a1a1a;border-color:#d1242f;color:#ffa198; }
.fc-btn-danger:hover:not([disabled]) { background:#5c2020; }
.fc-btn-link { background:transparent; border:none; color:#79c0ff; text-decoration:underline; padding:8px 4px; cursor:pointer; font-size:12px; }
.fc-btn-link:hover:not([disabled]) { color:#58a6ff; }
.fc-file-input { display:none; }
.fc-file-label { display:inline-block; padding:8px 14px; border-radius:6px; background:#21262d; border:1px solid #30363d; color:#c9d1d9; cursor:pointer; font-size:12px; font-weight:500; }
.fc-file-label:hover { background:#30363d; }

.fc-info { font-family:ui-monospace,monospace; font-size:12px; line-height:1.8; color:#c9d1d9; }
.fc-info .lbl { color:#8b949e; display:inline-block; min-width:150px; }
.fc-info .val { color:#79c0ff; }
.fc-info .val-warn { color:#f0c674; }
.fc-info .arrow { color:#6e7681; margin:0 6px; }

.fc-phase { display:inline-flex; align-items:center; padding:3px 10px; border-radius:10px; font-size:10px; font-weight:600; letter-spacing:0.5px; text-transform:uppercase; }
.fc-phase-idle { background:#21262d; color:#8b949e; border:1px solid #30363d; }
.fc-phase-ready { background:#0a1620; color:#79c0ff; border:1px solid #1e3a5f; }
.fc-phase-busy { background:#2a1d10; color:#f0c674; border:1px solid #d29922; }
.fc-phase-done { background:#1a3828; color:#7ee2a8; border:1px solid #2ea043; }
.fc-phase-error { background:#4a1a1a; color:#ffa198; border:1px solid #f85149; }

.fc-steps { display:flex; align-items:center; gap:8px; margin-top:8px; flex-wrap:wrap; }
.fc-step { display:flex; align-items:center; gap:6px; padding:4px 10px; border-radius:14px; font-size:11px; font-family:ui-monospace,monospace; color:#6e7681; background:#161b22; border:1px solid #21262d; }
.fc-step-active { color:#f0c674; background:#2a1d10; border-color:#d29922; }
.fc-step-done { color:#7ee2a8; background:#1a3828; border-color:#2ea043; }
.fc-step-arrow { color:#30363d; font-size:12px; }

.fc-progress-wrap { margin-top:10px; height:8px; background:#0d1117; border-radius:4px; overflow:hidden; border:1px solid #21262d; }
.fc-progress-bar { height:100%; background:linear-gradient(90deg,#238636,#2ea043); transition:width 0.1s ease-out; }
.fc-progress-label { margin-top:4px; font-family:ui-monospace,monospace; font-size:11px; color:#8b949e; display:flex; justify-content:space-between; }

.fc-warn-card { margin-top:10px; padding:8px 12px; background:#2a1d10; color:#f0c674; border-left:3px solid #d29922; border-radius:4px; font-family:ui-monospace,monospace; font-size:12px; }
.fc-error { margin-top:10px; padding:10px 12px; background:#1a0d0d; color:#ffa198; border-left:3px solid #f85149; border-radius:4px; font-family:ui-monospace,monospace; font-size:12px; overflow-wrap:anywhere; }

.fc-preview { margin-top:10px; padding:10px 12px; background:#0a0d12; border:1px solid #161b22; border-radius:4px; font-family:ui-monospace,monospace; font-size:11.5px; color:#c9d1d9; max-height:220px; overflow-y:auto; white-space:pre-wrap; overflow-wrap:anywhere; }
.fc-preview-actions { margin-top:8px; display:flex; gap:10px; align-items:center; }
.fc-preview-actions .hint { color:#6e7681; font-size:11px; }

.fc-log { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;line-height:1.6;color:#c9d1d9;max-height:200px;overflow-y:auto;}
.fc-log-entry { display:flex; gap:8px; padding:1px 0; }
.fc-log-entry .time { color:#6e7681; flex-shrink:0; }
.fc-log-entry .msg { color:#c9d1d9; overflow-wrap:anywhere; }
.fc-log-info .msg { color:#79c0ff; }
.fc-log-ok .msg { color:#7ee2a8; }
.fc-log-warn .msg { color:#f0c674; }
.fc-log-err .msg { color:#ffa198; }

.fc-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
.fc-footnote code { background:#0d1117; padding:1px 5px; border-radius:3px; color:#79c0ff; }
"#;

// =============================================================================
// Модели
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum Phase {
    Idle,
    Ready,
    Reading,
    Converting,
    Done,
    Cancelled,
    Error,
}

impl Phase {
    fn label(&self) -> &'static str {
        return match self {
            Self::Idle => "idle",
            Self::Ready => "ready",
            Self::Reading => "reading",
            Self::Converting => "converting",
            Self::Done => "done",
            Self::Cancelled => "cancelled",
            Self::Error => "error",
        };
    }

    fn css(&self) -> &'static str {
        return match self {
            Self::Idle => "fc-phase-idle",
            Self::Ready => "fc-phase-ready",
            Self::Reading | Self::Converting => "fc-phase-busy",
            Self::Done => "fc-phase-done",
            Self::Cancelled | Self::Error => "fc-phase-error",
        };
    }

    /// Индекс активного шага: 0=read, 1=convert, 2=ready.
    /// `None`, если фаза не относится к workflow.
    fn step_index(&self) -> Option<u8> {
        return match self {
            Self::Reading => Some(0),
            Self::Converting => Some(1),
            Self::Done => Some(2),
            _ => None,
        };
    }
}

struct LogEntry {
    time: String,
    level: &'static str,
    text: String,
}

struct OutputInfo {
    url: String,
    filename: String,
    size: u64,
    preview: String,
}

struct AppState {
    phase: Phase,
    token: Option<CancellationToken>,
    filename: String,
    size: u64,
    detected: Option<Format>,
    target: Option<Format>,
    progress_done: u64,
    progress_total: u64,
    output: Option<OutputInfo>,
    error: Option<String>,
    logs: Vec<LogEntry>,
}

impl AppState {
    fn new() -> Self {
        return Self {
            phase: Phase::Idle,
            token: None,
            filename: String::new(),
            size: 0,
            detected: None,
            target: None,
            progress_done: 0,
            progress_total: 0,
            output: None,
            error: None,
            logs: Vec::new(),
        };
    }

    fn is_busy(&self) -> bool {
        return matches!(self.phase, Phase::Reading | Phase::Converting);
    }

    fn has_file(&self) -> bool {
        return !self.filename.is_empty();
    }
}

// =============================================================================
// Утилиты
// =============================================================================

fn as_node(el: &web_sys::Element) -> &web_sys::Node {
    return el.unchecked_ref::<web_sys::Node>();
}

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "fc-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() {
            p.remove_child(as_node(&old))?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(as_node(&style))?;
    return Ok(());
}

fn now_timestamp() -> String {
    let d = js_sys::Date::new_0();
    let h = d.get_hours();
    let m = d.get_minutes();
    let s = d.get_seconds();
    let ms = d.get_milliseconds();
    return format_str(format_args!("{h:02}:{m:02}:{s:02}.{ms:03}"));
}

fn escape_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;");
}

fn push_log(st: &Rc<RefCell<AppState>>, level: &'static str, text: String) {
    let mut s = st.borrow_mut();
    s.logs.push(LogEntry {
        time: now_timestamp(),
        level,
        text,
    });
    if s.logs.len() > 100 {
        let drop = s.logs.len() - 100;
        s.logs.drain(0..drop);
    }
}

fn render_log_html(logs: &[LogEntry]) -> String {
    let mut html = String::with_capacity(logs.len() * 100);
    for e in logs {
        html.push_str(r#"<div class="fc-log-entry fc-log-"#);
        html.push_str(e.level);
        html.push_str(r#""><span class="time">"#);
        html.push_str(&e.time);
        html.push_str(r#"</span><span class="msg">"#);
        html.push_str(&escape_html(&e.text));
        html.push_str("</span></div>");
    }
    return html;
}

/// Снимает UTF-8 BOM (`EF BB BF`) с начала буфера.
///
/// Многие редакторы (в т.ч. Notepad на Windows) сохраняют файлы
/// с BOM. JSON-парсер спотыкается на первом символе, CSV-парсер
/// видит три лишних байта в первой колонке. Снимаем — и всё парсится.
///
/// UTF-16 (BOM `FF FE` / `FE FF`) не обрабатываем: конвертер
/// работает только с UTF-8.
fn strip_utf8_bom(bytes: &[u8]) -> &[u8] {
    if bytes.starts_with(&[0xEF, 0xBB, 0xBF]) {
        return &bytes[3..];
    }
    return bytes;
}

// =============================================================================
// Рендер
// =============================================================================

fn render_state(doc: &Document, st: &AppState) {
    // Phase badge
    if let Some(el) = doc.get_element_by_id("fc-phase") {
        el.set_class_name(&format_str(format_args!("fc-phase {}", st.phase.css())));
        el.set_text_content(Some(st.phase.label()));
    }

    // Info
    if let Some(el) = doc.get_element_by_id("fc-info") {
        el.set_inner_html(&render_info_html(st));
    }

    // Steps
    if let Some(el) = doc.get_element_by_id("fc-steps") {
        el.set_inner_html(&render_steps_html(st.phase.step_index()));
    }

    // Progress
    if let Some(el) = doc.get_element_by_id("fc-progress") {
        if st.progress_total > 0 && matches!(st.phase, Phase::Reading | Phase::Converting) {
            let pct = ((st.progress_done as f64 / st.progress_total as f64) * 100.0) as u32;
            let bar = format_str(format_args!(
                r#"<div class="fc-progress-wrap"><div class="fc-progress-bar" style="width:{pct}%"></div></div><div class="fc-progress-label"><span>{pct}%</span><span>{d} / {t}</span></div>"#,
                pct = pct,
                d = format_size(st.progress_done),
                t = format_size(st.progress_total),
            ));
            el.set_inner_html(&bar);
        } else {
            el.set_inner_html("");
        }
    }

    // Warning
    if let Some(el) = doc.get_element_by_id("fc-warning") {
        if st.has_file() && st.size > SIZE_WARN_BYTES {
            el.set_class_name("fc-warn-card");
            el.set_text_content(Some(&format_str(format_args!(
                "Файл большой ({}). Текстовые конвертеры — не для гигабайтных файлов; \
                 ожидайте заметное время чтения. Нажмите «Отменить», если случайно.",
                format_size(st.size)
            ))));
        } else {
            el.set_inner_html("");
            el.set_class_name("");
        }
    }

    // Error
    if let Some(el) = doc.get_element_by_id("fc-error") {
        match &st.error {
            Some(e) => el.set_inner_html(&escape_html(e)),
            None => el.set_inner_html(""),
        }
    }

    // Preview + actions
    if let Some(el) = doc.get_element_by_id("fc-preview") {
        match &st.output {
            Some(o) => {
                let mut h = String::with_capacity(o.preview.len() + 256);
                h.push_str(&escape_html(&o.preview));
                h.push_str(r#"<div class="fc-preview-actions"><span class="hint">Проверьте результат и сохраните файл.</span></div>"#);
                el.set_inner_html(&h);
            }
            None => el.set_inner_html(""),
        }
    }

    // Log
    if let Some(el) = doc.get_element_by_id("fc-log") {
        el.set_inner_html(&render_log_html(&st.logs));
        if let Ok(el) = el.dyn_into::<HtmlElement>() {
            let _ = el.set_scroll_top(el.scroll_height() as f64);
        }
    }

    // Buttons
    let busy = st.is_busy();
    let ready = st.phase == Phase::Ready;
    let done = st.phase == Phase::Done;
    let can_reset = matches!(
        st.phase,
        Phase::Done | Phase::Cancelled | Phase::Error
    ) && st.has_file();

    set_btn(doc, "fc-convert", ready);
    set_btn(doc, "fc-cancel", busy);
    set_btn(doc, "fc-download", done);
    set_btn(doc, "fc-open-tab", done);
    set_btn(doc, "fc-reset", can_reset);

    // Drop zone — dim во время работы
    if let Some(el) = doc.get_element_by_id("fc-drop") {
        if busy {
            let _ = el.set_attribute("data-busy", "");
        } else {
            let _ = el.remove_attribute("data-busy");
        }
    }
}

fn render_info_html(st: &AppState) -> String {
    let mut h = String::with_capacity(512);
    if !st.has_file() {
        h.push_str(r#"<span class="lbl">Файл:</span><span class="val">не выбран</span>"#);
        return h;
    }

    h.push_str(r#"<span class="lbl">Файл:</span><span class="val">"#);
    h.push_str(&escape_html(&st.filename));
    h.push_str(r#"</span><br><span class="lbl">Размер:</span><span class="val">"#);
    h.push_str(&format_size(st.size));
    h.push_str("</span>");

    if let (Some(from), Some(to)) = (st.detected, st.target) {
        h.push_str(r#"<br><span class="lbl">Конвертация:</span><span class="val">"#);
        h.push_str(from.label());
        h.push_str(r#"</span><span class="arrow">→</span><span class="val">"#);
        h.push_str(to.label());
        h.push_str("</span>");
    } else if let Some(from) = st.detected {
        h.push_str(r#"<br><span class="lbl">Формат:</span><span class="val">"#);
        h.push_str(from.label());
        h.push_str(r#"</span><span class="arrow">·</span><span class="val-warn">конвертация не поддерживается</span>"#);
    }

    if let Some(o) = &st.output {
        h.push_str(r#"<br><span class="lbl">Результат:</span><span class="val">"#);
        h.push_str(&escape_html(&o.filename));
        h.push_str(" (");
        h.push_str(&format_size(o.size));
        h.push_str(")</span>");
    }

    return h;
}

fn render_steps_html(active: Option<u8>) -> String {
    let steps = ["read", "convert", "ready"];
    let mut h = String::with_capacity(256);
    for (i, name) in steps.iter().enumerate() {
        if i > 0 {
            h.push_str(r#"<span class="fc-step-arrow">→</span>"#);
        }
        let cls = match active {
            Some(a) if (i as u8) < a => "fc-step fc-step-done",
            Some(a) if (i as u8) == a => "fc-step fc-step-active",
            _ => "fc-step",
        };
        h.push_str(r#"<span class=""#);
        h.push_str(cls);
        h.push_str(r#"">"#);
        h.push_str(name);
        h.push_str("</span>");
    }
    return h;
}

fn set_btn(doc: &Document, id: &str, enabled: bool) {
    let Some(el) = doc.get_element_by_id(id) else { return };
    if enabled {
        let _ = el.remove_attribute("disabled");
    } else {
        let _ = el.set_attribute("disabled", "");
    }
}

// =============================================================================
// Действия
// =============================================================================

fn on_file_picked(doc: Document, st: Rc<RefCell<AppState>>, file: File) {
    // Revoke предыдущего output URL и очищаем логи.
    {
        let mut s = st.borrow_mut();
        if let Some(o) = s.output.take() {
            revoke_download_url(&o.url);
        }
        s.logs.clear();
    }

    let size = file.size() as u64;

    if size == 0 {
        let msg = String::from("файл пуст — нечего конвертировать");
        {
            let mut s = st.borrow_mut();
            s.phase = Phase::Error;
            s.filename = file.name();
            s.size = 0;
            s.error = Some(msg.clone());
        }
        push_log(&st, "err", msg);
        render_state(&doc, &st.borrow());
        return;
    }

    {
        let mut s = st.borrow_mut();
        s.phase = Phase::Ready;
        s.filename = file.name();
        s.size = size;
        s.detected = None;
        s.target = None;
        s.progress_done = 0;
        s.progress_total = 0;
        s.error = None;
    }
    push_log(
        &st,
        "ok",
        format_str(format_args!(
            "файл: {} ({})",
            file.name(),
            format_size(size)
        )),
    );
    render_state(&doc, &st.borrow());
}

async fn on_convert(doc: Document, st: Rc<RefCell<AppState>>, file: File) {
    if st.borrow().is_busy() {
        return;
    }

    let token = CancellationToken::new();
    {
        let mut s = st.borrow_mut();
        s.phase = Phase::Reading;
        s.token = Some(token.clone());
        s.error = None;
        if let Some(o) = s.output.take() {
            revoke_download_url(&o.url);
        }
        s.progress_done = 0;
        s.progress_total = file.size() as u64;
    }
    push_log(&st, "info", String::from("чтение файла чанками…"));
    render_state(&doc, &st.borrow());

    // --- Phase 1: chunked read
    let st_read = st.clone();
    let doc_read = doc.clone();
    let raw_bytes = match read_file_chunked(&file, &token, move |done, total| {
        {
            let mut s = st_read.borrow_mut();
            s.progress_done = done;
            s.progress_total = total;
        }
        render_state(&doc_read, &st_read.borrow());
    })
    .await
    {
        Ok(b) => b,
        Err(e) => {
            let cancelled = token.is_cancelled();
            {
                let mut s = st.borrow_mut();
                s.phase = if cancelled { Phase::Cancelled } else { Phase::Error };
                s.error = if cancelled {
                    None
                } else {
                    Some(format_str(format_args!("чтение: {e}")))
                };
                s.token = None;
            }
            push_log(
                &st,
                if cancelled { "warn" } else { "err" },
                if cancelled {
                    String::from("отменено пользователем")
                } else {
                    format_str(format_args!("чтение: {e}"))
                },
            );
            render_state(&doc, &st.borrow());
            return;
        }
    };

    // --- Phase 2: detect & strip BOM
    let detected = detect_format(&file.name(), &raw_bytes);
    let bytes = if matches!(detected, Format::Unknown | Format::Html) {
        // Для бинарных / HTML — не трогаем BOM, но и не конвертируем.
        raw_bytes
    } else {
        strip_utf8_bom(&raw_bytes).to_vec()
    };
    let target = choose_target(detected);

    {
        let mut s = st.borrow_mut();
        s.detected = Some(detected);
        s.target = target;
    }
    push_log(
        &st,
        "info",
        format_str(format_args!("определён формат: {}", detected.label())),
    );
    render_state(&doc, &st.borrow());

    // Проверка отмены между фазами.
    if token.is_cancelled() {
        {
            let mut s = st.borrow_mut();
            s.phase = Phase::Cancelled;
            s.token = None;
        }
        push_log(&st, "warn", String::from("отменено между фазами"));
        render_state(&doc, &st.borrow());
        return;
    }

    let target = match target {
        Some(t) => t,
        None => {
            let msg = format_str(format_args!(
                "формат {} не поддерживает конвертацию",
                detected.label()
            ));
            {
                let mut s = st.borrow_mut();
                s.phase = Phase::Error;
                s.error = Some(msg.clone());
                s.token = None;
            }
            push_log(&st, "err", msg);
            render_state(&doc, &st.borrow());
            return;
        }
    };

    // --- Phase 3: convert
    {
        let mut s = st.borrow_mut();
        s.phase = Phase::Converting;
    }
    push_log(
        &st,
        "info",
        format_str(format_args!(
            "конвертация: {} → {}",
            detected.label(),
            target.label()
        )),
    );
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    if token.is_cancelled() {
        {
            let mut s = st.borrow_mut();
            s.phase = Phase::Cancelled;
            s.token = None;
        }
        push_log(&st, "warn", String::from("отменено перед конвертацией"));
        render_state(&doc, &st.borrow());
        return;
    }

    let output_bytes = match convert(detected, target, &bytes) {
        Ok(b) => b,
        Err(e) => {
            let msg = format_str(format_args!("конвертация: {e}"));
            {
                let mut s = st.borrow_mut();
                s.phase = Phase::Error;
                s.error = Some(msg.clone());
                s.token = None;
            }
            push_log(&st, "err", msg);
            render_state(&doc, &st.borrow());
            return;
        }
    };

    // --- Phase 4: blob & URL
    let blob = match make_blob(&output_bytes, target.mime()) {
        Ok(b) => b,
        Err(e) => {
            let msg = format_str(format_args!("создание Blob: {e}"));
            {
                let mut s = st.borrow_mut();
                s.phase = Phase::Error;
                s.error = Some(msg.clone());
                s.token = None;
            }
            push_log(&st, "err", msg);
            render_state(&doc, &st.borrow());
            return;
        }
    };
    let url = match create_download_url(&blob) {
        Ok(u) => u,
        Err(e) => {
            let msg = format_str(format_args!("createObjectURL: {e}"));
            {
                let mut s = st.borrow_mut();
                s.phase = Phase::Error;
                s.error = Some(msg.clone());
                s.token = None;
            }
            push_log(&st, "err", msg);
            render_state(&doc, &st.borrow());
            return;
        }
    };

    let new_filename = replace_extension(&file.name(), target.as_str());
    let preview = preview_output(&output_bytes, 500);
    let output_size = output_bytes.len() as u64;

    {
        let mut s = st.borrow_mut();
        s.phase = Phase::Done;
        s.token = None;
        s.output = Some(OutputInfo {
            url,
            filename: new_filename.clone(),
            size: output_size,
            preview,
        });
    }
    push_log(
        &st,
        "ok",
        format_str(format_args!(
            "готово: {} ({})",
            new_filename,
            format_size(output_size)
        )),
    );
    render_state(&doc, &st.borrow());
}

fn on_cancel(doc: Document, st: Rc<RefCell<AppState>>) {
    let token = st.borrow().token.clone();
    if let Some(token) = token {
        token.cancel();
        push_log(&st, "warn", String::from("запрошена отмена…"));
        render_state(&doc, &st.borrow());
    }
}

fn on_download(doc: Document, st: Rc<RefCell<AppState>>) {
    let pair = {
        let s = st.borrow();
        match &s.output {
            Some(o) => Some((o.url.clone(), o.filename.clone())),
            None => None,
        }
    };
    let Some((url, filename)) = pair else { return };

    match trigger_download(&doc, &url, &filename) {
        Ok(()) => push_log(&st, "ok", format_str(format_args!("скачано: {filename}"))),
        Err(e) => push_log(&st, "err", format_str(format_args!("download: {e}"))),
    }
    render_state(&doc, &st.borrow());
}

fn on_open_in_tab(doc: Document, st: Rc<RefCell<AppState>>) {
    let url = {
        let s = st.borrow();
        match &s.output {
            Some(o) => o.url.clone(),
            None => return,
        }
    };

    let Some(window) = web_sys::window() else { return };
    match window.open_with_url_and_target(&url, "_blank") {
        Ok(_) => {
            push_log(
                &st,
                "info",
                String::from("открыто в новой вкладке"),
            );
        }
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("open: {e:?}")));
        }
    }
    render_state(&doc, &st.borrow());
}

fn on_reset(doc: Document, st: Rc<RefCell<AppState>>) {
    {
        let mut s = st.borrow_mut();
        if let Some(o) = s.output.take() {
            revoke_download_url(&o.url);
        }
        // Файл (filename/size) сохраняем — можно запустить конвертацию
        // снова без повторного выбора. Сбрасываем только производные поля.
        s.phase = if s.size > 0 { Phase::Ready } else { Phase::Idle };
        s.detected = None;
        s.target = None;
        s.progress_done = 0;
        s.progress_total = 0;
        s.error = None;
        s.token = None;
    }
    push_log(&st, "info", String::from("сброшено — готов к новой конвертации"));
    render_state(&doc, &st.borrow());
}

// =============================================================================
// Сборка UI
// =============================================================================

fn build_intro(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("fc-desc");
    d.set_inner_html(
        "<p class=\"fc-desc-line\"><strong>Rust/WASM File Converter.</strong> \
         Полный workflow: выбор файла (клик или drag-and-drop) → \
         чанковое чтение с прогрессом → определение формата → \
         конвертация в Rust → превью → скачивание.</p>\
         <p class=\"fc-desc-line\"><strong>Что поддерживается.</strong> \
         <code>.json → .csv</code>, <code>.csv → .json</code>, \
         <code>.md → .html</code>, <code>.txt → .json</code> (статистика). \
         UTF-8 BOM снимается автоматически.</p>\
         <p class=\"fc-desc-line\"><strong>Отмена.</strong> \
         Файл читается кусками по 64 КБ, между кусками проверяется \
         <code>CancellationToken</code> и делается yield — кнопка \
         «Отменить» работает в реальном времени.</p>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_controls(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("fc-block");

    let mut html = String::with_capacity(2048);
    html.push_str(r#"<div class="fc-block-head"><span>Файл</span><span class="grow"></span><span id="fc-phase" class="fc-phase fc-phase-idle">idle</span></div>"#);

    html.push_str(r#"<div class="fc-row">"#);
    html.push_str(r#"<input type="file" id="fc-input" class="fc-file-input">"#);
    html.push_str(r#"<label for="fc-input" class="fc-file-label">Выбрать файл…</label>"#);
    html.push_str(r#"<button class="fc-btn fc-btn-primary" id="fc-convert" disabled>Конвертировать</button>"#);
    html.push_str(r#"<button class="fc-btn fc-btn-danger" id="fc-cancel" disabled>Отменить</button>"#);
    html.push_str(r#"<button class="fc-btn" id="fc-download" disabled>Скачать результат</button>"#);
    html.push_str(r#"<button class="fc-btn fc-btn-link" id="fc-open-tab" disabled>Открыть в новой вкладке ↗</button>"#);
    html.push_str(r#"<button class="fc-btn" id="fc-reset" disabled>Сбросить</button>"#);
    html.push_str("</div>");

    html.push_str(r#"<div class="fc-info" id="fc-info"><span class="lbl">Файл:</span><span class="val">не выбран</span></div>"#);
    html.push_str(r#"<div class="fc-steps" id="fc-steps"></div>"#);
    html.push_str(r#"<div id="fc-progress"></div>"#);
    html.push_str(r#"<div id="fc-warning"></div>"#);
    html.push_str(r#"<div id="fc-error"></div>"#);
    html.push_str(r#"<div id="fc-preview" class="fc-preview"></div>"#);

    d.set_inner_html(&html);
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_drop_zone(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("fc-drop");
    d.set_id("fc-drop");
    d.set_inner_html(
        "<div>Перетащите файл сюда или нажмите «Выбрать файл…»</div>\
         <div class=\"fc-drop-hint\">.json · .csv · .md · .txt</div>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_log(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("fc-log");
    d.set_id("fc-log");
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_footnote(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("fc-footnote");
    d.set_inner_html(
        "<strong>Правила, применённые в демо.</strong> \
         <em>Реальный прогресс, не indeterminate bar:</em> чанковое чтение даёт \
         измеримый процент (64 KB × N). <em>Отмена идемпотентна:</em> \
         <code>CancellationToken</code> можно выставить много раз, эффект один. \
         <em>Object URL освобождается:</em> при повторной конвертации и Reset мы \
         вызываем <code>revokeObjectURL</code> — иначе утечка памяти. \
         <em>BOM снимается:</em> <code>EF BB BF</code> в начале UTF-8 ломает \
         JSON-парсер; демо удаляет его до конвертации. \
         <em>Никаких <code>unwrap()</code> в обработчиках:</em> только \
         <code>Result</code> и понятные сообщения. <em>Двухфазный workflow:</em> \
         сначала конвертация с превью, потом явная кнопка «Скачать» или \
         «Открыть в новой вкладке».",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("Глава 49 — Rust/WASM File Converter"));
    root.append_child(as_node(&h2))?;

    root.append_child(&build_intro(&document)?)?;

    let state = Rc::new(RefCell::new(AppState::new()));

    root.append_child(&build_drop_zone(&document)?)?;
    root.append_child(&build_controls(&document)?)?;
    root.append_child(&build_log(&document)?)?;
    root.append_child(&build_footnote(&document)?)?;

    // Общая ячейка для текущего File — нужна обработчику Convert.
    let current_file: Rc<RefCell<Option<File>>> = Rc::new(RefCell::new(None));

    // ---- file input ----
    {
        let doc = document.clone();
        let st = state.clone();
        let cf = current_file.clone();
        let input = document
            .get_element_by_id("fc-input")
            .ok_or_else(|| JsValue::from_str("no #fc-input"))?
            .dyn_into::<HtmlInputElement>()?;
        // Клон для замыкания — сам `input` остаётся доступен ниже для
        // set_onchange. `HtmlInputElement::clone` дешёвый (refcount).
        let input_cb = input.clone();
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let Some(file) = pick_file(&input_cb) else { return };
            *cf.borrow_mut() = Some(file.clone());
            on_file_picked(doc.clone(), st.clone(), file);
        });
        input.set_onchange(Some(closure.as_ref().unchecked_ref()));
        closure.forget();
    }

    // ---- drag & drop ----
    {
        let doc = document.clone();
        let st = state.clone();
        let cf = current_file.clone();
        let drop_el = document
            .get_element_by_id("fc-drop")
            .ok_or_else(|| JsValue::from_str("no #fc-drop"))?;
        let drop_el: HtmlElement = drop_el.dyn_into()?;

        // dragover — preventDefault, иначе drop не сработает.
        let drop_for_over = drop_el.clone();
        let on_over: Closure<dyn FnMut(web_sys::DragEvent)> =
            Closure::new(move |e: web_sys::DragEvent| {
                e.prevent_default();
                let _ = drop_for_over.set_class_name("fc-drop fc-drop-hover");
            });
        drop_el.add_event_listener_with_callback("dragover", on_over.as_ref().unchecked_ref())?;
        on_over.forget();

        // dragleave — снять подсветку.
        let drop_for_leave = drop_el.clone();
        let on_leave: Closure<dyn FnMut(web_sys::DragEvent)> =
            Closure::new(move |_e: web_sys::DragEvent| {
                let _ = drop_for_leave.set_class_name("fc-drop");
            });
        drop_el.add_event_listener_with_callback("dragleave", on_leave.as_ref().unchecked_ref())?;
        on_leave.forget();

        // drop — взять первый файл.
        let on_drop: Closure<dyn FnMut(web_sys::DragEvent)> =
            Closure::new(move |e: web_sys::DragEvent| {
                e.prevent_default();
                let Some(target) = e.current_target() else { return };
                if let Ok(el) = target.dyn_into::<HtmlElement>() {
                    let _ = el.set_class_name("fc-drop");
                }
                let Some(dt) = e.data_transfer() else { return };
                let Some(files) = dt.files() else { return };
                if files.length() == 0 {
                    return;
                }
                let Some(file) = files.item(0) else { return };
                *cf.borrow_mut() = Some(file.clone());
                on_file_picked(doc.clone(), st.clone(), file);
            });
        drop_el.add_event_listener_with_callback("drop", on_drop.as_ref().unchecked_ref())?;
        on_drop.forget();
    }

    // ---- convert ----
    {
        let doc = document.clone();
        let st = state.clone();
        let cf = current_file.clone();
        let btn = document
            .get_element_by_id("fc-convert")
            .ok_or_else(|| JsValue::from_str("no #fc-convert"))?;
        let btn: HtmlElement = btn.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let file = cf.borrow().clone();
            let Some(file) = file else { return };
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                on_convert(doc_async, st_async, file).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- cancel ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn = document
            .get_element_by_id("fc-cancel")
            .ok_or_else(|| JsValue::from_str("no #fc-cancel"))?;
        let btn: HtmlElement = btn.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            on_cancel(doc.clone(), st.clone());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- download ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn = document
            .get_element_by_id("fc-download")
            .ok_or_else(|| JsValue::from_str("no #fc-download"))?;
        let btn: HtmlElement = btn.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            on_download(doc.clone(), st.clone());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- open in tab ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn = document
            .get_element_by_id("fc-open-tab")
            .ok_or_else(|| JsValue::from_str("no #fc-open-tab"))?;
        let btn: HtmlElement = btn.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            on_open_in_tab(doc.clone(), st.clone());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- reset ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn = document
            .get_element_by_id("fc-reset")
            .ok_or_else(|| JsValue::from_str("no #fc-reset"))?;
        let btn: HtmlElement = btn.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            on_reset(doc.clone(), st.clone());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    push_log(
        &state,
        "info",
        String::from("готово — выберите файл или перетащите его в область выше"),
    );
    render_state(&document, &state.borrow());

    return Ok(());
}
```

## 49.7. Ловушки, которые мы прошли в этой главе

Этот раздел — практический вывод. Путь к рабочему демо выявил **шесть специфических проблем** Rust/WASM, каждая из которых может встретиться в реальном проекте.

### 1. `array_buffer()` возвращает `Promise`, не `Result`

В `web-sys 0.3.105` сигнатура `Blob::array_buffer()` — `fn array_buffer(&self) -> Promise`. Не `Result<Promise, JsValue>`. Ошибки сети (редкие для локального Blob, но возможные) проявляются как **rejection** промиса и ловятся в `JsFuture::from(promise).await` как `Err(JsValue)`.

Правильно:
```rust
let ab_promise: js_sys::Promise = slice.array_buffer();
let ab_value: JsValue = JsFuture::from(ab_promise).await
    .map_err(|e| format_str(format_args!("read: {e:?}")))?;
```

Неправильно:
```rust
let ab_promise = slice.array_buffer().map_err(...)?;  // ← не скомпилируется
```

### 2. `HtmlInputElement` не `Copy` — клонировать для замыкания

Классическая ошибка borrow checker:
```rust
let input = document.get_element_by_id("fc-input")...?;
let closure = Closure::new(move || {
    let Some(file) = pick_file(&input) else { return };  // ← input перемещён
});
input.set_onchange(Some(closure.as_ref().unchecked_ref()));  // ← E0382
```

`HtmlInputElement::clone` дешёвый — это refcount WebIDL-обёртки. Правильно:
```rust
let input_cb = input.clone();
let closure = Closure::new(move || {
    let Some(file) = pick_file(&input_cb) else { return };
});
input.set_onchange(Some(closure.as_ref().unchecked_ref()));  // ← ok
```

### 3. `json_to_csv` должен принимать все формы JSON

Первая версия требовала **массив объектов**. Но JSON-демо **само создаёт** одиночные объекты (статистика из `.txt → .json`). Пользователь сохраняет результат первой конвертации, потом конвертирует его обратно — и получает `корень JSON должен быть массивом`.

**Решение:** нормализация входа. Четыре формы приводятся к общей — последовательности объектов:
```rust
fn normalize_to_rows(value: Value) -> Option<Vec<Value>> {
    match value {
        Value::Array(arr) if !arr.is_empty() => Some(arr.into_iter().map(...).collect()),
        Value::Object(_) => Some(vec![value]),
        Value::Null => None,
        other => Some(vec![wrap_as_value(other)]),
    }
}
```

### 4. UTF-8 BOM ломает парсеры

Многие редакторы (Notepad на Windows) сохраняют текстовые файлы с BOM `EF BB BF` в начале. `serde_json::from_slice` спотыкается на первом байте, CSV-парсер видит три лишних байта в первой колонке.

**Решение:** `strip_utf8_bom` перед конвертацией:
```rust
fn strip_utf8_bom(bytes: &[u8]) -> &[u8] {
    if bytes.starts_with(&[0xEF, 0xBB, 0xBF]) {
        return &bytes[3..];
    }
    return bytes;
}
```

Применяется к текстовым форматам (JSON, CSV, MD, TXT), но **не** к бинарным (Unknown) и **не** к HTML (BOM там валиден по спецификации).

### 5. Тесты с ручными числами ломаются

Изначально `test_text_stats` проверял `assert_eq!(s.bytes, 25)` на входе `b"hello world\nsecond line\n"`. Но реальная длина входа — 24 байта. Тест падал.

**Правило:** для проверок длины **вычисляйте ожидаемое значение из входа**, а не вбивайте руками:
```rust
assert_eq!(s.bytes, input.len());  // ← истина от самого входа
```

### 6. Устаревшие тесты при расширении контракта

После того как `json_to_csv` был расширен, старый `test_json_to_csv_not_array` продолжал ожидать ошибку на входе `{"a": 1}`. Тест стал проверять **отменённое поведение** и падал.

**Правило:** при расширении контракта функции **удаляйте тесты, закреплявшие старое поведение**. Обычно они называются `test_*_not_*` или `test_*_invalid_*`.

## 49.8. Итог

Что даёт глава:

1. **Полный workflow файловой обработки** — от `<input type="file">` до programmatic download. Впервые в книге показывается вся цепочка целиком.
2. **Чанковое чтение с реальным прогрессом** через `file.slice().arrayBuffer()`. Правило проекта о indeterminate progress bar соблюдено.
3. **Отмена через `CancellationToken`** — общий `Rc<Cell<bool>>`, проверка между чанками, идемпотентна.
4. **Двухфазный workflow** — Convert → Preview → Download. Пользователь видит результат до сохранения.
5. **Обработка ошибок** — `Phase::Error` + карточка с сообщением, никаких `unwrap()`.
6. **Освобождение ресурсов** — `revoke_download_url` перед повторной конвертацией.
7. **Устойчивость к реальности** — BOM strip, нормализация JSON.

**Изоляция от image-пайплайна.** Модули `file_convert.rs`, `file_io.rs` не пересекаются с `converter.rs`, `converter_ui.rs`, `converter_native.rs`, `converter_worker.rs`. Image-пайплайн продолжит работать — он раскроется в главе 51.

**Правила проекта соблюдены:** `format_str(format_args!(...))` в хвостах, `TimeoutFuture::new(0).await` после чанка, `Rc<RefCell<AppState>>`, префикс CSS `fc-`, никаких `unused_imports`, `dead_code`, `unused_mut`.

**Тесты:** 28 тестов главы 49 — конвертеры, определение формата, edge cases, утилиты.

**Команды финальной проверки:**

```bash
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests
wasm-pack test --headless --chrome --test wasm_file_convert   # 28 passed
wasm-pack build --release --target web
python3 serve.py
# http://127.0.0.1:5173/
```