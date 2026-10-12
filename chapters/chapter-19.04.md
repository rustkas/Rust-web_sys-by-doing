# Глава 19.4. Чтение файлов: `text`, `arrayBuffer`, `slice`, `stream`

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── lib.rs                       # +2 pub mod fsa_read_*
│   ├── app.rs                       # демо-секция 19.4 (3 блока)
│   ├── fsa_read_plan.rs             # план чтения, лимиты, magic-байты
│   └── fsa_read_ops.rs              # peek, text, bytes, slice, stream + инкрементальная обработка
└── tests/
    └── wasm_fsa_read_plan.rs        # тесты 19.4 (38 шт.)
```

**Краткий комментарий.** В проекте есть `fsa_api_reader.rs` — четыре прямые обёртки `Blob.text()` / `arrayBuffer()` / `slice()` / `stream()`. Эта подглава идёт **на уровень выбора и применения**:

1. **Peek + detect_magic.** Первое действие с незнакомым файлом — не «прочитать», а «посмотреть 8 байт». Модуль распознаёт PNG, JPEG, GIF, PDF, ZIP, WEBP, WAV, ELF, OGG, FLAC, GZIP, BZIP2, 7z, MP4, MP3, AAC.
2. **План под задачу.** `suggest_plan` + `validate_plan` — не «всегда `arrayBuffer`», а «Bytes для маленьких, Stream для больших, Text только для `text/*`».
3. **Инкрементальная обработка на потоке.** SHA-256 файла, подсчёт строк, поиск маркера — это то, для чего streaming и существует.

Префикс `fsa_read_*` — уникален; с `fsa_api_reader` не пересекается.

## Ссылки

- [MDN: Blob.text / arrayBuffer / slice / stream](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [MDN: ReadableStreamDefaultReader](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader)
- [MDN: Backpressure in streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Concepts)
- [Wikipedia: List of file signatures (magic bytes)](https://en.wikipedia.org/wiki/List_of_file_signatures)
- [Chrome: Streams — processing without loading](https://web.dev/streams/)

---

## Исходные модули

### `src/fsa_read_plan.rs`

```rust
//! Модель чтения файла: что именно мы собираемся прочитать и почему.
//!
//! Модуль не читает файл (это делает `fsa_read_ops`), а только:
//! * выбирает план под размер и MIME;
//! * проверяет план против лимитов;
//! * определяет формат по magic-байтам;
//! * объясняет выбор и ошибки пользователю.

use crate::fsa_choose_options::format_str;

#[derive(Debug, Clone, Copy)]
pub struct ReadLimits {
    pub max_buffer_bytes: u64,
    pub max_text_bytes: u64,
    pub peek_default: u64,
}

impl ReadLimits {
    pub fn demo() -> Self {
        return ReadLimits {
            max_buffer_bytes: 100 * 1024 * 1024,
            max_text_bytes: 10 * 1024 * 1024,
            peek_default: 8,
        };
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ReadPlan {
    Text,
    Bytes,
    Slice { start: u64, end: u64 },
    Stream,
}

impl ReadPlan {
    pub fn name(&self) -> &'static str {
        match self {
            ReadPlan::Text => return "Text",
            ReadPlan::Bytes => return "Bytes",
            ReadPlan::Slice { .. } => return "Slice",
            ReadPlan::Stream => return "Stream",
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
pub enum ReadError {
    TooLarge {
        size: u64,
        limit: u64,
        plan_name: &'static str,
    },
    NotApplicable {
        plan_name: &'static str,
        reason: String,
    },
    Cancelled,
    Io(String),
}

impl ReadError {
    pub fn is_cancelled(&self) -> bool {
        return matches!(self, ReadError::Cancelled);
    }
}

pub fn detect_magic_format(bytes: &[u8]) -> Option<&'static str> {
    if bytes.len() >= 4 && bytes.starts_with(&[0x89, 0x50, 0x4E, 0x47]) {
        return Some("PNG (изображение)");
    }
    if bytes.len() >= 3 && bytes.starts_with(&[0xFF, 0xD8, 0xFF]) {
        return Some("JPEG (изображение)");
    }
    if bytes.len() >= 4 && bytes.starts_with(b"GIF8") {
        return Some("GIF (изображение)");
    }
    if bytes.len() >= 4 && bytes.starts_with(b"%PDF") {
        return Some("PDF (документ)");
    }
    if bytes.len() >= 4 && bytes.starts_with(&[0x50, 0x4B, 0x03, 0x04]) {
        return Some("ZIP (архив)");
    }
    if bytes.len() >= 12 && &bytes[0..4] == b"RIFF" {
        if &bytes[8..12] == b"WEBP" {
            return Some("WEBP (изображение)");
        }
        if &bytes[8..12] == b"WAVE" {
            return Some("WAV (аудио)");
        }
    }
    if bytes.len() >= 4 && bytes.starts_with(&[0x7F, 0x45, 0x4C, 0x46]) {
        return Some("ELF (исполняемый)");
    }
    if bytes.len() >= 4 && bytes.starts_with(b"OggS") {
        return Some("OGG (аудио/видео)");
    }
    if bytes.len() >= 4 && bytes.starts_with(b"fLaC") {
        return Some("FLAC (аудио)");
    }
    if bytes.len() >= 2 && bytes.starts_with(&[0x1F, 0x8B]) {
        return Some("GZIP (архив)");
    }
    if bytes.len() >= 3 && bytes.starts_with(b"BZh") {
        return Some("BZIP2 (архив)");
    }
    if bytes.len() >= 6 && bytes.starts_with(&[0x37, 0x7A, 0xBC, 0xAF, 0x27, 0x1C]) {
        return Some("7-Zip (архив)");
    }
    if bytes.len() >= 8 && &bytes[4..8] == b"ftyp" {
        return Some("MP4 (видео)");
    }
    if bytes.len() >= 3 && bytes.starts_with(b"ID3") {
        return Some("MP3 (аудио)");
    }
    if bytes.len() >= 2 && bytes[0] == 0xFF && (bytes[1] & 0xE0) == 0xE0 {
        return Some("MP3 / AAC (аудио)");
    }
    return None;
}

pub fn format_magic_line(bytes: &[u8]) -> String {
    if bytes.is_empty() {
        return "(нет данных)".to_string();
    }
    let mut parts: Vec<String> = Vec::with_capacity(bytes.len());
    for b in bytes.iter().take(8) {
        parts.push(format_str(format_args!("{:02X}", b)));
    }
    return parts.join(" ");
}

pub fn suggest_plan(size: u64, mime: &str, limits: &ReadLimits) -> ReadPlan {
    if size == 0 {
        return ReadPlan::Bytes;
    }
    if mime.starts_with("text/") && size <= limits.max_text_bytes {
        return ReadPlan::Text;
    }
    if size <= limits.max_buffer_bytes {
        return ReadPlan::Bytes;
    }
    return ReadPlan::Stream;
}

pub fn validate_plan(
    plan: &ReadPlan,
    size: u64,
    mime: &str,
    limits: &ReadLimits,
) -> Result<(), ReadError> {
    match plan {
        ReadPlan::Text => {
            if size > limits.max_text_bytes {
                return Err(ReadError::TooLarge {
                    size,
                    limit: limits.max_text_bytes,
                    plan_name: "Text",
                });
            }
            if !mime.is_empty() && !mime.starts_with("text/") && !mime.contains("json") {
                let reason: String = format_str(format_args!(
                    "MIME '{mime}' не текстовый — text() даст кракозябры. \
                     Используйте Bytes или Slice.",
                ));
                return Err(ReadError::NotApplicable {
                    plan_name: "Text",
                    reason,
                });
            }
        }
        ReadPlan::Bytes => {
            if size > limits.max_buffer_bytes {
                return Err(ReadError::TooLarge {
                    size,
                    limit: limits.max_buffer_bytes,
                    plan_name: "Bytes",
                });
            }
        }
        ReadPlan::Slice { start, end } => {
            if end < start {
                return Err(ReadError::Io(format_str(format_args!(
                    "Slice: end ({end}) < start ({start})",
                ))));
            }
            if *end > size {
                return Err(ReadError::Io(format_str(format_args!(
                    "Slice: end ({end}) > размера файла ({size})",
                ))));
            }
        }
        ReadPlan::Stream => {}
    }
    return Ok(());
}

pub fn format_size(bytes: u64) -> String {
    if bytes < 1024 {
        return format_str(format_args!("{bytes} Б"));
    }
    if bytes < 1024 * 1024 {
        let kb: f64 = bytes as f64 / 1024.0;
        return format_str(format_args!("{kb:.1} КБ"));
    }
    if bytes < 1024 * 1024 * 1024 {
        let mb: f64 = bytes as f64 / (1024.0 * 1024.0);
        return format_str(format_args!("{mb:.2} МБ"));
    }
    let gb: f64 = bytes as f64 / (1024.0 * 1024.0 * 1024.0);
    return format_str(format_args!("{gb:.2} ГБ"));
}

pub fn explain_plan(plan: &ReadPlan, size: u64, mime: &str) -> String {
    let mime_display: &str = if mime.is_empty() { "—" } else { mime };
    match plan {
        ReadPlan::Text => {
            let s: String = format_str(format_args!(
                "Text: файл {}, MIME {mime_display}. Тип текстовый и размер в \
                 пределах лимита — декодируем как UTF-8.",
                format_size(size),
            ));
            return s;
        }
        ReadPlan::Bytes => {
            let s: String = format_str(format_args!(
                "Bytes: файл {}. Умещается в буфер памяти — читаем целиком \
                 через arrayBuffer().",
                format_size(size),
            ));
            return s;
        }
        ReadPlan::Slice { start, end } => {
            let s: String = format_str(format_args!(
                "Slice {start}..{end}: читаем только диапазон — не весь файл.",
            ));
            return s;
        }
        ReadPlan::Stream => {
            let s: String = format_str(format_args!(
                "Stream: файл {}. Больше лимита Bytes — читаем чанками, \
                 не грузим в память.",
                format_size(size),
            ));
            return s;
        }
    }
}

pub fn explain_error(err: &ReadError) -> String {
    match err {
        ReadError::TooLarge { size, limit, plan_name } => {
            let s: String = format_str(format_args!(
                "✗ Файл {} превышает лимит {} для плана {}.\n\
                 Используйте Stream — он читает чанками и не держит всё в памяти.",
                format_size(*size),
                format_size(*limit),
                plan_name,
            ));
            return s;
        }
        ReadError::NotApplicable { plan_name, reason } => {
            let s: String = format_str(format_args!(
                "✗ План {plan_name} неприменим: {reason}",
            ));
            return s;
        }
        ReadError::Cancelled => {
            return "Отменено пользователем.".to_string();
        }
        ReadError::Io(msg) => {
            let s: String = format_str(format_args!("✗ Ошибка чтения: {msg}"));
            return s;
        }
    }
}
```

### `src/fsa_read_ops.rs`

```rust
//! Реализация чтения: peek, text, bytes, slice, stream.
//!
//! Плюс три инкрементальные задачи на потоке:
//! * `hash_stream` — SHA-256 без загрузки в память;
//! * `count_lines_stream` — подсчёт строк;
//! * `find_marker_stream` — поиск подстроки с обработкой границ чанков.

use gloo_timers::future::TimeoutFuture;
use js_sys::Uint8Array;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{File, ReadableStreamDefaultReader};

use crate::fsa_choose_options::format_str;
use crate::fsa_read_plan::{
    format_size, validate_plan, ReadError, ReadLimits, ReadPlan,
};

#[derive(Debug, Clone, PartialEq)]
pub struct StreamStats {
    pub bytes_read: u64,
    pub chunks: u32,
    pub yields: u32,
    pub elapsed_ms: f64,
    pub cancelled: bool,
    pub first_chunk_sizes: Vec<usize>,
}

impl StreamStats {
    pub fn matches_size(&self, expected: u64) -> bool {
        return !self.cancelled && self.bytes_read == expected;
    }
    pub fn avg_chunk_bytes(&self) -> u64 {
        if self.chunks == 0 {
            return 0;
        }
        return self.bytes_read / self.chunks as u64;
    }
    pub fn throughput_mbps(&self) -> f64 {
        if self.elapsed_ms <= 0.0 {
            return 0.0;
        }
        let mb = self.bytes_read as f64 / (1024.0 * 1024.0);
        let seconds = self.elapsed_ms / 1000.0;
        return mb / seconds;
    }
}

pub async fn peek_bytes(file: &File, n: u64) -> Result<Vec<u8>, ReadError> {
    if n == 0 {
        return Ok(Vec::new());
    }
    let end = (file.size() as u64).min(n) as f64;
    let slice = file.slice_with_f64_and_f64(0.0, end).map_err(|e| {
        ReadError::Io(format_str(format_args!("slice: {e:?}")))
    })?;
    let promise: js_sys::Promise = slice.array_buffer();
    let value = JsFuture::from(promise).await.map_err(|e| {
        ReadError::Io(format_str(format_args!("array_buffer: {e:?}")))
    })?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| ReadError::Io("не ArrayBuffer".to_string()))?;
    return Ok(Uint8Array::new(&buffer).to_vec());
}

fn format_hex_line(bytes: &[u8]) -> String {
    let mut hex: String = String::with_capacity(16 * 3);
    let mut ascii: String = String::with_capacity(16);
    for i in 0..16 {
        if i < bytes.len() {
            let b: u8 = bytes[i];
            hex.push_str(&format_str(format_args!("{:02X} ", b)));
            let ch: char = if (0x20..0x7f).contains(&b) { b as char } else { '.' };
            ascii.push(ch);
        } else {
            hex.push_str("   ");
            ascii.push(' ');
        }
    }
    return format_str(format_args!("{hex}| {ascii}"));
}

pub fn format_hexdump(bytes: &[u8], base_offset: u64) -> String {
    if bytes.is_empty() {
        return "(нет данных)".to_string();
    }
    let mut out: String = String::with_capacity(bytes.len() * 4);
    for (i, chunk) in bytes.chunks(16).enumerate() {
        let offset: u64 = base_offset + (i * 16) as u64;
        let line: String = format_hex_line(chunk);
        if i > 0 {
            out.push('\n');
        }
        out.push_str(&format_str(format_args!("{offset:08X}: {line}")));
    }
    return out;
}

pub async fn read_text_safe(
    file: &File,
    limits: &ReadLimits,
) -> Result<String, ReadError> {
    validate_plan(&ReadPlan::Text, file.size() as u64, &file.type_(), limits)?;
    let promise: js_sys::Promise = file.text();
    let value = JsFuture::from(promise).await.map_err(|e| {
        ReadError::Io(format_str(format_args!("text: {e:?}")))
    })?;
    return value
        .as_string()
        .ok_or_else(|| ReadError::Io("не строка".to_string()));
}

pub async fn read_bytes_safe(
    file: &File,
    limits: &ReadLimits,
) -> Result<Vec<u8>, ReadError> {
    validate_plan(&ReadPlan::Bytes, file.size() as u64, &file.type_(), limits)?;
    let promise: js_sys::Promise = file.array_buffer();
    let value = JsFuture::from(promise).await.map_err(|e| {
        ReadError::Io(format_str(format_args!("array_buffer: {e:?}")))
    })?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| ReadError::Io("не ArrayBuffer".to_string()))?;
    return Ok(Uint8Array::new(&buffer).to_vec());
}

pub async fn read_slice_safe(
    file: &File,
    start: u64,
    end: u64,
    limits: &ReadLimits,
) -> Result<Vec<u8>, ReadError> {
    let plan = ReadPlan::Slice { start, end };
    validate_plan(&plan, file.size() as u64, &file.type_(), limits)?;
    let slice = file
        .slice_with_f64_and_f64(start as f64, end as f64)
        .map_err(|e| ReadError::Io(format_str(format_args!("slice: {e:?}"))))?;
    let promise: js_sys::Promise = slice.array_buffer();
    let value = JsFuture::from(promise).await.map_err(|e| {
        ReadError::Io(format_str(format_args!("array_buffer: {e:?}")))
    })?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| ReadError::Io("не ArrayBuffer".to_string()))?;
    return Ok(Uint8Array::new(&buffer).to_vec());
}

async fn stream_foreach<F>(
    file: &File,
    max_bytes: Option<u64>,
    mut on_chunk: F,
    mut on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<StreamStats, ReadError>
where
    F: FnMut(&[u8], u64) -> bool,
{
    let stream = file.stream();
    let reader = ReadableStreamDefaultReader::new(&stream)
        .map_err(|e| ReadError::Io(format_str(format_args!("reader: {e:?}"))))?;

    let total: u64 = file.size() as u64;
    let mut bytes_read: u64 = 0;
    let mut chunks: u32 = 0;
    let mut yields: u32 = 0;
    let mut last_yield: f64 = 0.0;
    let t0: f64 = js_sys::Date::now();
    let mut first_chunk_sizes: Vec<usize> = Vec::with_capacity(8);
    let mut early_exit: bool = false;

    loop {
        if is_cancelled() {
            let elapsed = js_sys::Date::now() - t0;
            return Ok(StreamStats {
                bytes_read, chunks, yields, elapsed_ms: elapsed,
                cancelled: true, first_chunk_sizes,
            });
        }

        if let Some(max) = max_bytes {
            if bytes_read >= max {
                early_exit = true;
                break;
            }
        }

        let result = JsFuture::from(reader.read()).await.map_err(|e| {
            ReadError::Io(format_str(format_args!("read: {e:?}")))
        })?;

        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))
            .ok().and_then(|v| v.as_bool()).unwrap_or(true);
        if done {
            break;
        }

        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))
            .map_err(|e| ReadError::Io(format_str(format_args!("value: {e:?}"))))?;
        let chunk = Uint8Array::new(&value);
        let bytes: Vec<u8> = chunk.to_vec();
        let len: u64 = bytes.len() as u64;

        if first_chunk_sizes.len() < 8 {
            first_chunk_sizes.push(bytes.len());
        }

        let keep_going = on_chunk(&bytes, bytes_read);
        bytes_read += len;
        chunks += 1;

        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            if total > 0 {
                on_progress(bytes_read as f64 / total as f64);
            }
            TimeoutFuture::new(0).await;
            last_yield = now;
            yields += 1;
        }

        if !keep_going {
            early_exit = true;
            break;
        }
    }

    if !early_exit {
        on_progress(1.0);
    }
    let elapsed = js_sys::Date::now() - t0;
    return Ok(StreamStats {
        bytes_read, chunks, yields, elapsed_ms: elapsed,
        cancelled: false, first_chunk_sizes,
    });
}

pub async fn read_stream_stats(
    file: &File,
    on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<StreamStats, ReadError> {
    return stream_foreach(file, None, |_, _| true, on_progress, is_cancelled).await;
}

#[derive(Debug, Clone, PartialEq)]
pub struct HashStreamResult {
    pub hex: String,
    pub stats: StreamStats,
}

pub async fn hash_stream(
    file: &File,
    on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<HashStreamResult, ReadError> {
    use sha2::{Digest, Sha256};

    let mut hasher = Sha256::new();
    let stats = stream_foreach(
        file,
        None,
        |chunk, _offset| { hasher.update(chunk); return true; },
        on_progress,
        is_cancelled,
    ).await?;

    if stats.cancelled {
        return Err(ReadError::Cancelled);
    }

    let hex: String = hex::encode(hasher.finalize());
    return Ok(HashStreamResult { hex, stats });
}

#[derive(Debug, Clone, PartialEq)]
pub struct LinesStreamResult {
    pub count: u64,
    pub stats: StreamStats,
}

pub async fn count_lines_stream(
    file: &File,
    on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<LinesStreamResult, ReadError> {
    let mut count: u64 = 0;
    let mut last_byte: Option<u8> = None;

    let stats = stream_foreach(
        file,
        None,
        |chunk, _offset| {
            for &b in chunk {
                if b == b'\n' {
                    count += 1;
                }
            }
            if let Some(&b) = chunk.last() {
                last_byte = Some(b);
            }
            return true;
        },
        on_progress,
        is_cancelled,
    ).await?;

    if !stats.cancelled {
        if let Some(b) = last_byte {
            if b != b'\n' {
                count += 1;
            }
        }
    }
    return Ok(LinesStreamResult { count, stats });
}

#[derive(Debug, Clone, PartialEq)]
pub struct MarkerStreamResult {
    pub found: bool,
    pub offset: Option<u64>,
    pub stats: StreamStats,
}

fn find_subsequence(haystack: &[u8], needle: &[u8]) -> Option<usize> {
    if needle.is_empty() || haystack.len() < needle.len() {
        return None;
    }
    let last = haystack.len() - needle.len();
    for i in 0..=last {
        if &haystack[i..i + needle.len()] == needle {
            return Some(i);
        }
    }
    return None;
}

pub async fn find_marker_stream(
    file: &File,
    marker: &[u8],
    on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<MarkerStreamResult, ReadError> {
    if marker.is_empty() {
        return Err(ReadError::Io("пустой маркер".to_string()));
    }

    let marker_vec: Vec<u8> = marker.to_vec();
    let marker_len: usize = marker_vec.len();
    let keep_tail: usize = marker_len.saturating_sub(1);

    let mut tail: Vec<u8> = Vec::with_capacity(keep_tail);
    let mut found: Option<u64> = None;

    let stats = stream_foreach(
        file,
        None,
        |chunk, offset| {
            let mut combined: Vec<u8> = Vec::with_capacity(tail.len() + chunk.len());
            combined.extend_from_slice(&tail);
            combined.extend_from_slice(chunk);

            if let Some(pos) = find_subsequence(&combined, &marker_vec) {
                let base = offset.saturating_sub(tail.len() as u64);
                found = Some(base + pos as u64);
                return false;
            }

            tail.clear();
            if keep_tail == 0 {
                return true;
            }
            if combined.len() >= keep_tail {
                let start = combined.len() - keep_tail;
                tail.extend_from_slice(&combined[start..]);
            } else {
                tail.extend_from_slice(&combined);
            }
            return true;
        },
        on_progress,
        is_cancelled,
    ).await?;

    return Ok(MarkerStreamResult {
        found: found.is_some(),
        offset: found,
        stats,
    });
}

pub fn format_stream_stats(stats: &StreamStats, expected: u64) -> String {
    let status_line: String = if stats.cancelled {
        "⚠ Отменено".to_string()
    } else if stats.bytes_read == expected {
        "✓ Прочитано полностью — совпадает с размером файла".to_string()
    } else {
        format_str(format_args!(
            "⚠ Прочитано {} из {} — расхождение",
            format_size(stats.bytes_read),
            format_size(expected),
        ))
    };

    let mut lines: Vec<String> = Vec::with_capacity(8);
    lines.push(status_line);
    lines.push(format_str(format_args!("Байт:      {}", format_size(stats.bytes_read))));
    lines.push(format_str(format_args!("Чанков:    {}", stats.chunks)));
    lines.push(format_str(format_args!("Средний:   {}", format_size(stats.avg_chunk_bytes()))));
    lines.push(format_str(format_args!("Yield'ов:  {} (раз в 16 мс)", stats.yields)));
    lines.push(format_str(format_args!(
        "Время:     {:.0} ms · {:.1} МБ/с",
        stats.elapsed_ms, stats.throughput_mbps(),
    )));
    lines.push(format_str(format_args!(
        "Размеры чанков (первые {}): {}",
        stats.first_chunk_sizes.len(),
        format_chunk_sizes(&stats.first_chunk_sizes),
    )));
    return lines.join("\n");
}

pub fn format_chunk_sizes(sizes: &[usize]) -> String {
    if sizes.is_empty() {
        return "(нет)".to_string();
    }
    let mut parts: Vec<String> = Vec::with_capacity(sizes.len());
    for s in sizes {
        parts.push(format_str(format_args!("{}", format_size(*s as u64))));
    }
    return parts.join(", ");
}

pub fn format_hash_result(r: &HashStreamResult, expected: u64) -> String {
    let mut lines: Vec<String> = Vec::with_capacity(3);
    lines.push(format_str(format_args!("SHA-256: {}", r.hex)));
    lines.push(format_str(format_args!("Ожидаемый размер: {}", format_size(expected))));
    lines.push(format_str(format_args!("Прочитано:        {}", format_size(r.stats.bytes_read))));
    return lines.join("\n");
}

pub fn format_lines_result(r: &LinesStreamResult) -> String {
    let mut lines: Vec<String> = Vec::with_capacity(2);
    lines.push(format_str(format_args!("Строк:    {}", r.count)));
    lines.push(format_str(format_args!("Прочитано: {}", format_size(r.stats.bytes_read))));
    return lines.join("\n");
}

pub fn format_marker_result(r: &MarkerStreamResult, marker: &[u8]) -> String {
    let marker_display: String = String::from_utf8_lossy(marker).to_string();
    let mut lines: Vec<String> = Vec::with_capacity(4);
    if let Some(off) = r.offset {
        lines.push("Маркер найден.".to_string());
        lines.push(format_str(format_args!("Смещение:  {off} байт (0x{off:X})")));
    } else {
        lines.push("Маркер не найден.".to_string());
    }
    lines.push(format_str(format_args!(
        "Искали:    \"{marker_display}\" ({} байт)",
        marker.len(),
    )));
    lines.push(format_str(format_args!("Прочитано: {}", format_size(r.stats.bytes_read))));
    return lines.join("\n");
}
```

---

## Как добавить в `lib.rs`

```rust
pub mod fsa_read_plan;
pub mod fsa_read_ops;
```

---

## Тесты

### `tests/wasm_fsa_read_plan.rs`

**Важно:** `File` не наследует MIME от `Blob`. Тип задаётся через `FilePropertyBag` — иначе `file.type_()` вернёт `""`, и проверки на несовместимость не сработают.

```rust
//! Тесты `fsa_read_*` — план, лимиты, magic-байты, peek, text/bytes/slice,
//! поток и три инкрементальные задачи.

use wasm_bindgen_test::*;
use web_sys::File;

use hello_wasm02::fsa_read_ops::{
    count_lines_stream, find_marker_stream, format_chunk_sizes, format_hexdump,
    format_stream_stats, hash_stream, peek_bytes, read_bytes_safe,
    read_slice_safe, read_stream_stats, read_text_safe, StreamStats,
};
use hello_wasm02::fsa_read_plan::{
    detect_magic_format, explain_error, explain_plan, format_magic_line,
    format_size, suggest_plan, validate_plan, ReadError, ReadLimits, ReadPlan,
};

wasm_bindgen_test_configure!(run_in_browser);

/// Собрать `File` с заданным MIME.
///
/// `File` **не наследует** `type` от `Blob` — конструктор берёт его
/// только из `FilePropertyBag`.
fn make_file(content: &[u8], name: &str, mime: &str) -> File {
    let u8arr = js_sys::Uint8Array::from(content);
    let blob = web_sys::Blob::new_with_u8_array_sequence(
        &js_sys::Array::of1(&u8arr),
    )
    .unwrap();

    let file_opts = web_sys::FilePropertyBag::new();
    file_opts.set_type(mime);

    web_sys::File::new_with_blob_sequence_and_options(
        &js_sys::Array::of1(&blob),
        name,
        &file_opts,
    )
    .unwrap()
}

fn small_limits() -> ReadLimits {
    return ReadLimits {
        max_buffer_bytes: 1024,
        max_text_bytes: 512,
        peek_default: 8,
    };
}

// ---------------------------------------------------------------------------
// detect_magic_format
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn magic_detects_png() {
    let bytes = [0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A];
    assert_eq!(detect_magic_format(&bytes), Some("PNG (изображение)"));
}

#[wasm_bindgen_test]
fn magic_detects_jpeg() {
    let bytes = [0xFF, 0xD8, 0xFF, 0xE0, 0x00];
    assert_eq!(detect_magic_format(&bytes), Some("JPEG (изображение)"));
}

#[wasm_bindgen_test]
fn magic_detects_pdf() {
    let bytes = b"%PDF-1.4";
    assert_eq!(detect_magic_format(bytes), Some("PDF (документ)"));
}

#[wasm_bindgen_test]
fn magic_detects_zip() {
    let bytes = [0x50, 0x4B, 0x03, 0x04, 0x14];
    assert_eq!(detect_magic_format(&bytes), Some("ZIP (архив)"));
}

#[wasm_bindgen_test]
fn magic_detects_webp() {
    let mut bytes: Vec<u8> = Vec::with_capacity(12);
    bytes.extend_from_slice(b"RIFF");
    bytes.extend_from_slice(&[0, 0, 0, 0]);
    bytes.extend_from_slice(b"WEBP");
    assert_eq!(detect_magic_format(&bytes), Some("WEBP (изображение)"));
}

#[wasm_bindgen_test]
fn magic_detects_gzip() {
    let bytes = [0x1F, 0x8B, 0x08];
    assert_eq!(detect_magic_format(&bytes), Some("GZIP (архив)"));
}

#[wasm_bindgen_test]
fn magic_unknown_returns_none() {
    assert_eq!(detect_magic_format(b"hello world"), None);
}

#[wasm_bindgen_test]
fn magic_empty_returns_none() {
    assert_eq!(detect_magic_format(&[]), None);
}

#[wasm_bindgen_test]
fn magic_line_hex() {
    let bytes = [0x89, 0x50, 0x4E, 0x47];
    assert_eq!(format_magic_line(&bytes), "89 50 4E 47");
}

// ---------------------------------------------------------------------------
// suggest_plan / validate_plan
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn suggest_text_small_is_text() {
    assert_eq!(suggest_plan(100, "text/plain", &ReadLimits::demo()), ReadPlan::Text);
}

#[wasm_bindgen_test]
fn suggest_binary_small_is_bytes() {
    assert_eq!(suggest_plan(100, "image/png", &ReadLimits::demo()), ReadPlan::Bytes);
}

#[wasm_bindgen_test]
fn suggest_huge_binary_is_stream() {
    assert_eq!(
        suggest_plan(500 * 1024 * 1024, "video/mp4", &ReadLimits::demo()),
        ReadPlan::Stream,
    );
}

#[wasm_bindgen_test]
fn validate_text_rejects_binary_mime() {
    let limits = ReadLimits::demo();
    let res = validate_plan(&ReadPlan::Text, 100, "image/png", &limits);
    assert!(matches!(res, Err(ReadError::NotApplicable { .. })));
}

#[wasm_bindgen_test]
fn validate_bytes_rejects_too_large() {
    let limits = small_limits();
    let size = limits.max_buffer_bytes + 1;
    let res = validate_plan(&ReadPlan::Bytes, size, "application/octet-stream", &limits);
    assert!(matches!(res, Err(ReadError::TooLarge { .. })));
}

#[wasm_bindgen_test]
fn validate_slice_rejects_end_above_size() {
    let limits = small_limits();
    let res = validate_plan(
        &ReadPlan::Slice { start: 0, end: 200 },
        100,
        "application/octet-stream",
        &limits,
    );
    assert!(matches!(res, Err(ReadError::Io(_))));
}

// ---------------------------------------------------------------------------
// peek_bytes
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn peek_returns_first_bytes() {
    let data: Vec<u8> = (0u8..=255).collect();
    let file = make_file(&data, "peek.bin", "application/octet-stream");
    let peeked = peek_bytes(&file, 8).await.expect("peek");
    assert_eq!(peeked, &data[..8]);
}

#[wasm_bindgen_test]
async fn peek_short_file_returns_all() {
    let data = b"abc";
    let file = make_file(data, "short.bin", "application/octet-stream");
    let peeked = peek_bytes(&file, 100).await.expect("peek");
    assert_eq!(peeked, data);
}

// ---------------------------------------------------------------------------
// format_hexdump
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn hexdump_empty() {
    assert_eq!(format_hexdump(&[], 0), "(нет данных)");
}

#[wasm_bindgen_test]
fn hexdump_single_line() {
    let bytes = &[0x89, 0x50, 0x4E, 0x47];
    let out = format_hexdump(bytes, 0);
    assert!(out.contains("00000000:"));
    assert!(out.contains("89 50 4E 47"));
}

// ---------------------------------------------------------------------------
// read_text / read_bytes / read_slice
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn read_text_roundtrip() {
    let file = make_file(b"Hello, 19.4!", "a.txt", "text/plain");
    let text = read_text_safe(&file, &ReadLimits::demo()).await.expect("text");
    assert_eq!(text, "Hello, 19.4!");
}

#[wasm_bindgen_test]
async fn read_text_rejects_binary() {
    let file = make_file(b"\x89PNG...", "a.bin", "image/png");
    let res = read_text_safe(&file, &ReadLimits::demo()).await;
    assert!(matches!(res, Err(ReadError::NotApplicable { .. })));
}

#[wasm_bindgen_test]
async fn read_bytes_roundtrip() {
    let data: Vec<u8> = (0u8..=255).collect();
    let file = make_file(&data, "a.bin", "application/octet-stream");
    let bytes = read_bytes_safe(&file, &ReadLimits::demo()).await.expect("bytes");
    assert_eq!(bytes, data);
}

#[wasm_bindgen_test]
async fn read_slice_reads_range() {
    let data: Vec<u8> = (0u8..=255).collect();
    let file = make_file(&data, "slice.bin", "application/octet-stream");
    let bytes = read_slice_safe(&file, 10, 30, &ReadLimits::demo())
        .await
        .expect("slice");
    assert_eq!(bytes, &data[10..30]);
}

// ---------------------------------------------------------------------------
// Stream
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn stream_reads_full_file() {
    let data = vec![0xABu8; 128 * 1024];
    let file = make_file(&data, "s.bin", "application/octet-stream");
    let stats = read_stream_stats(&file, |_| {}, || false).await.expect("stream");
    assert_eq!(stats.bytes_read, data.len() as u64);
    assert!(stats.matches_size(data.len() as u64));
}

#[wasm_bindgen_test]
async fn stream_cancel_immediately() {
    let data = vec![0u8; 64 * 1024];
    let file = make_file(&data, "c.bin", "application/octet-stream");
    let stats = read_stream_stats(&file, |_| {}, || true).await.expect("stream");
    assert!(stats.cancelled);
}

#[wasm_bindgen_test]
fn stream_stats_format() {
    let stats = StreamStats {
        bytes_read: 1024 * 1024,
        chunks: 16,
        yields: 2,
        elapsed_ms: 10.0,
        cancelled: false,
        first_chunk_sizes: vec![65536, 65536, 65536],
    };
    let text = format_stream_stats(&stats, 1024 * 1024);
    assert!(text.contains("✓"));
    assert!(text.contains("64.0 КБ"));
}

#[wasm_bindgen_test]
fn chunk_sizes_format() {
    assert_eq!(format_chunk_sizes(&[]), "(нет)");
    let s = format_chunk_sizes(&[1024, 2048]);
    assert!(s.contains("1.0 КБ"));
    assert!(s.contains("2.0 КБ"));
}

// ---------------------------------------------------------------------------
// hash_stream
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn hash_stream_matches_full() {
    let data: Vec<u8> = (0u8..=255).cycle().take(100_000).collect();
    let file = make_file(&data, "h.bin", "application/octet-stream");

    use sha2::{Digest, Sha256};
    let mut h = Sha256::new();
    h.update(&data);
    let expected = hex::encode(h.finalize());

    let result = hash_stream(&file, |_| {}, || false).await.expect("hash");
    assert_eq!(result.hex, expected);
    assert_eq!(result.stats.bytes_read, data.len() as u64);
}

// ---------------------------------------------------------------------------
// count_lines_stream
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn count_lines_basic() {
    let data = b"line1\nline2\nline3\n";
    let file = make_file(data, "a.txt", "text/plain");
    let r = count_lines_stream(&file, |_| {}, || false).await.expect("lines");
    assert_eq!(r.count, 3);
}

#[wasm_bindgen_test]
async fn count_lines_no_trailing_newline() {
    let data = b"a\nb\nc";
    let file = make_file(data, "a.txt", "text/plain");
    let r = count_lines_stream(&file, |_| {}, || false).await.expect("lines");
    assert_eq!(r.count, 3);
}

#[wasm_bindgen_test]
async fn count_lines_empty() {
    let file = make_file(b"", "a.txt", "text/plain");
    let r = count_lines_stream(&file, |_| {}, || false).await.expect("lines");
    assert_eq!(r.count, 0);
}

// ---------------------------------------------------------------------------
// find_marker_stream
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn find_marker_at_offset() {
    let mut data = vec![b'A'; 1000];
    data.extend_from_slice(b"MARKER");
    data.extend_from_slice(&[b'B'; 500]);
    let file = make_file(&data, "m.bin", "application/octet-stream");
    let r = find_marker_stream(&file, b"MARKER", |_| {}, || false)
        .await
        .expect("marker");
    assert!(r.found);
    assert_eq!(r.offset, Some(1000));
}

#[wasm_bindgen_test]
async fn find_marker_across_chunks() {
    let mut data = vec![b'X'; 200_000];
    data.extend_from_slice(b"NEEDLE");
    data.extend_from_slice(&[b'Y'; 200_000]);
    let file = make_file(&data, "m.bin", "application/octet-stream");
    let r = find_marker_stream(&file, b"NEEDLE", |_| {}, || false)
        .await
        .expect("marker");
    assert!(r.found);
    assert_eq!(r.offset, Some(200_000));
}

#[wasm_bindgen_test]
async fn find_marker_not_found() {
    let data = vec![b'X'; 10_000];
    let file = make_file(&data, "m.bin", "application/octet-stream");
    let r = find_marker_stream(&file, b"NEEDLE", |_| {}, || false)
        .await
        .expect("marker");
    assert!(!r.found);
    assert_eq!(r.offset, None);
}

#[wasm_bindgen_test]
async fn find_marker_empty_is_error() {
    let data = vec![0u8; 100];
    let file = make_file(&data, "m.bin", "application/octet-stream");
    let res = find_marker_stream(&file, b"", |_| {}, || false).await;
    assert!(matches!(res, Err(ReadError::Io(_))));
}

// ---------------------------------------------------------------------------
// format_* / explain_*
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn format_size_units() {
    assert_eq!(format_size(0), "0 Б");
    assert_eq!(format_size(512), "512 Б");
    assert_eq!(format_size(2048), "2.0 КБ");
}

#[wasm_bindgen_test]
fn explain_plan_nonempty() {
    for plan in [
        ReadPlan::Text,
        ReadPlan::Bytes,
        ReadPlan::Slice { start: 0, end: 16 },
        ReadPlan::Stream,
    ] {
        let text = explain_plan(&plan, 100, "application/octet-stream");
        assert!(!text.is_empty());
    }
}

#[wasm_bindgen_test]
fn explain_error_mentions_stream() {
    let e = ReadError::TooLarge {
        size: 500_000_000,
        limit: 100_000_000,
        plan_name: "Bytes",
    };
    let s = explain_error(&e);
    assert!(s.contains("Stream"));
}
```

---

## Команды запуска

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_fsa_read_plan
npm start
```

Ожидаемый результат: **38 passed, 0 failed**.

---

## Демо-пример в `src/app.rs`

Три блока, каждый — один образовательный тезис.

```rust
//! Демо-секция 19.4: четыре способа прочитать файл.
//!
//! Три независимых блока:
//! * A. «Что это за файл?» — peek 8 байт + определение формата.
//! * B. «Какой метод выбрать?» — сравнение text / bytes / slice / stream.
//! * C. «Инкрементальная обработка» — SHA-256 / строки / поиск маркера.
//!
//! Правило проекта: внутри `Closure::new(move || …)` нельзя захватывать
//! `&T` от параметра функции — `spawn_local` требует `'static`.
//! Все нужные элементы клонируются **до** `move`.
//!
//! Контракт с `lib.rs`: `run() -> Result<(), JsValue>`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{File, HtmlButtonElement, HtmlElement, HtmlProgressElement, HtmlSelectElement};

use crate::fsa_choose_exec::{self, Capabilities, PickOutcome};
use crate::fsa_choose_options::format_str;
use crate::fsa_read_ops;
use crate::fsa_read_plan::{self, ReadError, ReadLimits};

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

// ---------------------------------------------------------------------------
// CSS
// ---------------------------------------------------------------------------

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);
        let css = r#"
            #fsa-read { font-family: system-ui, sans-serif; max-width: 1000px; }
            #fsa-read button { padding: 6px 12px; cursor: pointer; margin-right: 8px; margin-bottom: 6px; }
            #fsa-read button:disabled { opacity: 0.5; cursor: not-allowed; }
            #fsa-read .unsupported {
                margin: 12px 0 0 0; padding: 16px 20px;
                background: #fdecea; border-left: 5px solid #c0392b;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px; line-height: 1.7;
                color: #5a1a13;
            }
            #fsa-read .help {
                margin: 0 0 16px 0; padding: 10px 14px;
                background: #f9f9f9; border: 1px solid #e5e5e5;
                border-radius: 6px; font-size: 13px; line-height: 1.6;
                white-space: pre-line; color: #333;
            }
            #fsa-read .block {
                margin: 0 0 20px 0; padding: 14px 18px;
                border: 1px solid #ddd; border-radius: 8px;
                background: #fdfdfd;
            }
            #fsa-read .block h3 {
                margin: 0 0 8px 0; font-size: 15px; color: #222;
                border-bottom: 1px solid #eee; padding-bottom: 6px;
            }
            #fsa-read .block .purpose {
                font-size: 12px; color: #666; margin: 0 0 12px 0;
                white-space: pre-line; line-height: 1.5;
            }
            #fsa-read .info {
                margin: 0 0 12px 0; padding: 10px 14px;
                background: #eef6ee; border-left: 4px solid #4caf50;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px;
            }
            #fsa-read .status {
                margin: 10px 0; padding: 8px 12px;
                background: #f0f4ff; border-left: 4px solid #4a7cff;
                border-radius: 4px; white-space: pre-line; font-family: monospace;
            }
            #fsa-read .status.ok      { border-left-color: #27ae60; background: #e8f7ee; }
            #fsa-read .status.blocked { border-left-color: #e6a700; background: #fffbe0; }
            #fsa-read .status.failed  { border-left-color: #c0392b; background: #fdecea; }
            #fsa-read .result {
                background: #f6f6f6; padding: 10px 14px; border-radius: 4px;
                font-family: monospace; font-size: 12px;
                white-space: pre; margin: 8px 0;
                max-height: 320px; overflow: auto;
            }
            #fsa-read progress { width: 320px; height: 14px; vertical-align: middle; }
            #fsa-read select { font-size: 13px; padding: 3px 6px; margin-right: 8px; }
            #fsa-read input[type=text] {
                font-family: monospace; font-size: 13px; padding: 3px 6px;
                margin-right: 8px;
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

// ---------------------------------------------------------------------------
// Состояние
// ---------------------------------------------------------------------------

#[derive(Default)]
struct DemoStats {
    ops: u32,
    last_ms: f64,
    last_label: String,
}

struct DemoState {
    file: Option<File>,
    cancel: Rc<Cell<bool>>,
    stats: Rc<RefCell<DemoStats>>,
}

// ---------------------------------------------------------------------------
// Точка входа
// ---------------------------------------------------------------------------

pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;

    let section = doc.create_element("section")?;
    section.set_id("fsa-read");

    let h2 = doc.create_element("h2")?;
    h2.set_text_content(Some("19.4 — Четыре способа прочитать файл"));
    section.append_child(&h2)?;

    let caps = Capabilities::detect();
    if !caps.open {
        let unsupported: HtmlElement = doc
            .create_element("div")?
            .dyn_into::<HtmlElement>()
            .map_err(|_| JsValue::from_str("не HtmlElement"))?;
        unsupported.set_attribute("class", "unsupported")?;
        unsupported.set_text_content(Some(&unsupported_text()));
        section.append_child(&unsupported)?;
        body.append_child(&section)?;
        return Ok(());
    }

    let help: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    help.set_attribute("class", "help")?;
    help.set_text_content(Some(&help_text()));
    section.append_child(&help)?;

    // Общий выбор файла.
    let pick_row = doc.create_element("div")?;
    let pick_btn = make_button(&doc, "Выбрать файл", "rd-pick")?;
    pick_row.append_child(&pick_btn)?;
    section.append_child(&pick_row)?;

    let info: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    info.set_attribute("class", "info")?;
    info.set_text_content(Some("Файл не выбран."));
    section.append_child(&info)?;

    // Три блока.
    let a = build_block_a(&doc, &section)?;
    let b = build_block_b(&doc, &section)?;
    let c = build_block_c(&doc, &section)?;

    body.append_child(&section)?;

    let state: Rc<RefCell<DemoState>> = Rc::new(RefCell::new(DemoState {
        file: None,
        cancel: Rc::new(Cell::new(false)),
        stats: Rc::new(RefCell::new(DemoStats::default())),
    }));

    wire_pick(&pick_btn, &info, &a, &b, &c, state.clone());
    wire_block_a(&a, state.clone());
    wire_block_b(&b, state.clone());
    wire_block_c(&c, state);

    Ok(())
}

// ---------------------------------------------------------------------------
// Блок A — «Что это за файл?»
// ---------------------------------------------------------------------------

struct BlockAUi {
    detect_btn: HtmlButtonElement,
    status: HtmlElement,
    result: HtmlElement,
}

fn build_block_a(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<BlockAUi, JsValue> {
    let block = make_block(doc, "A. Что это за файл?")?;
    let purpose = make_purpose(
        doc,
        "Peek 8 байт + определение формата. Первое действие с незнакомым \
         файлом — не «прочитать целиком», а «посмотреть подпись».",
    )?;
    block.append_child(&purpose)?;

    let controls = doc.create_element("div")?;
    let detect_btn = make_button(doc, "Определить формат", "rd-a-detect")?;
    detect_btn.set_disabled(true);
    controls.append_child(&detect_btn)?;
    block.append_child(&controls)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    let result = make_result(doc)?;
    block.append_child(&result)?;

    parent.append_child(&block)?;
    Ok(BlockAUi { detect_btn, status, result })
}

fn wire_block_a(ui: &BlockAUi, state: Rc<RefCell<DemoState>>) {
    // Клонируем ДО move — внутри замыкания ui недоступен.
    let status_c = ui.status.clone();
    let result_c = ui.result.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let status_el = status_c.clone();
        let result_el = result_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let file = match file_of(&state) {
                Some(f) => f,
                None => {
                    set_status(&status_el, "blocked", "Сначала выберите файл.");
                    return;
                }
            };
            let t0 = js_sys::Date::now();
            set_status(&status_el, "", "Peek 8 байт…");

            match fsa_read_ops::peek_bytes(&file, 8).await {
                Ok(bytes) => {
                    let hex: String = fsa_read_plan::format_magic_line(&bytes);
                    let fmt: &'static str =
                        fsa_read_plan::detect_magic_format(&bytes).unwrap_or("неизвестный формат");
                    let text: String = format_str(format_args!(
                        "Первые 8 байт: {hex}\nОпределённый формат: {fmt}",
                    ));
                    result_el.set_text_content(Some(&text));

                    let dt = js_sys::Date::now() - t0;
                    bump(&state, dt, "Peek");
                    set_status(&status_el, "ok", "Peek вернул подпись — без полной загрузки.");
                }
                Err(e) => {
                    let msg = fsa_read_plan::explain_error(&e);
                    set_status(&status_el, "failed", &msg);
                }
            }
        });
    });
    ui.detect_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

// ---------------------------------------------------------------------------
// Блок B — «Какой метод выбрать?»
// ---------------------------------------------------------------------------

struct BlockBUi {
    compare_btn: HtmlButtonElement,
    status: HtmlElement,
    result: HtmlElement,
}

fn build_block_b(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<BlockBUi, JsValue> {
    let block = make_block(doc, "B. Какой метод выбрать?")?;
    let purpose = make_purpose(
        doc,
        "Все четыре метода на одном файле в одной таблице: применимость \
         и время. Видно, что Bytes для больших файлов отказывает, а Peek \
         и Slice работают всегда.",
    )?;
    block.append_child(&purpose)?;

    let controls = doc.create_element("div")?;
    let compare_btn = make_button(doc, "Сравнить методы", "rd-b-compare")?;
    compare_btn.set_disabled(true);
    controls.append_child(&compare_btn)?;
    block.append_child(&controls)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    let result = make_result(doc)?;
    block.append_child(&result)?;

    parent.append_child(&block)?;
    Ok(BlockBUi { compare_btn, status, result })
}

fn wire_block_b(ui: &BlockBUi, state: Rc<RefCell<DemoState>>) {
    // Клонируем ДО move — иначе заимствованный ui убежит в spawn_local,
    // которому требуется 'static.
    let status_c = ui.status.clone();
    let result_c = ui.result.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let status_el = status_c.clone();
        let result_el = result_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let file = match file_of(&state) {
                Some(f) => f,
                None => {
                    set_status(&status_el, "blocked", "Сначала выберите файл.");
                    return;
                }
            };
            set_status(&status_el, "", "Сравнение методов…");

            let limits = ReadLimits::demo();
            let size = file.size() as u64;

            let mut lines: Vec<String> = Vec::with_capacity(8);
            lines.push(format_str(format_args!(
                "{:<22} {:<10} {:<12} {}",
                "Метод", "Применим", "Время", "Результат",
            )));
            lines.push("─".repeat(72));

            // Peek 8.
            {
                let t0 = js_sys::Date::now();
                let res = fsa_read_ops::peek_bytes(&file, 8).await;
                let dt = js_sys::Date::now() - t0;
                let cell: String = match res {
                    Ok(b) => format_str(format_args!("{} байт", b.len())),
                    Err(_) => "—".to_string(),
                };
                lines.push(format_str(format_args!(
                    "{:<22} {:<10} {:<12} {}",
                    "Peek 8", "✓", format_ms(dt), cell,
                )));
            }

            // Text.
            {
                let t0 = js_sys::Date::now();
                let res = fsa_read_ops::read_text_safe(&file, &limits).await;
                let dt = js_sys::Date::now() - t0;
                let (ok, cell): (&str, String) = match &res {
                    Ok(s) => ("✓", format_str(format_args!("{} симв.", s.chars().count()))),
                    Err(ReadError::NotApplicable { .. }) => ("✗ MIME", "не текст".to_string()),
                    Err(ReadError::TooLarge { .. }) => ("✗ размер", "> лимита".to_string()),
                    Err(_) => ("✗", "ошибка".to_string()),
                };
                lines.push(format_str(format_args!(
                    "{:<22} {:<10} {:<12} {}",
                    "Text (весь)", ok, format_ms(dt), cell,
                )));
            }

            // Bytes.
            {
                let t0 = js_sys::Date::now();
                let res = fsa_read_ops::read_bytes_safe(&file, &limits).await;
                let dt = js_sys::Date::now() - t0;
                let (ok, cell): (&str, String) = match &res {
                    Ok(b) => ("✓", fsa_read_plan::format_size(b.len() as u64)),
                    Err(ReadError::TooLarge { .. }) => ("✗ размер", "> лимита".to_string()),
                    Err(_) => ("✗", "ошибка".to_string()),
                };
                lines.push(format_str(format_args!(
                    "{:<22} {:<10} {:<12} {}",
                    "Bytes (весь)", ok, format_ms(dt), cell,
                )));
            }

            // Slice 16..64.
            {
                let start = 16u64.min(size);
                let end = 64u64.min(size);
                let t0 = js_sys::Date::now();
                let res = fsa_read_ops::read_slice_safe(&file, start, end, &limits).await;
                let dt = js_sys::Date::now() - t0;
                let cell: String = match res {
                    Ok(b) => format_str(format_args!("{} байт", b.len())),
                    Err(_) => "ошибка".to_string(),
                };
                lines.push(format_str(format_args!(
                    "{:<22} {:<10} {:<12} {}",
                    "Slice 16..64", "✓", format_ms(dt), cell,
                )));
            }

            // Stream.
            {
                let t0 = js_sys::Date::now();
                let res = fsa_read_ops::read_stream_stats(&file, |_| {}, || false).await;
                let dt = js_sys::Date::now() - t0;
                let cell: String = match res {
                    Ok(s) => format_str(format_args!("{} чанков", s.chunks)),
                    Err(_) => "ошибка".to_string(),
                };
                lines.push(format_str(format_args!(
                    "{:<22} {:<10} {:<12} {}",
                    "Stream (чанками)", "✓", format_ms(dt), cell,
                )));
            }

            result_el.set_text_content(Some(&lines.join("\n")));
            set_status(&status_el, "ok", "Сравнение завершено — см. таблицу.");
        });
    });
    ui.compare_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn format_ms(ms: f64) -> String {
    return format_str(format_args!("{ms:.1} ms"));
}

// ---------------------------------------------------------------------------
// Блок C — «Инкрементальная обработка»
// ---------------------------------------------------------------------------

struct BlockCUi {
    task_select: HtmlSelectElement,
    marker_input: web_sys::HtmlInputElement,
    run_btn: HtmlButtonElement,
    cancel_btn: HtmlButtonElement,
    progress: HtmlProgressElement,
    progress_label: HtmlElement,
    status: HtmlElement,
    result: HtmlElement,
}

fn build_block_c(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<BlockCUi, JsValue> {
    let block = make_block(doc, "C. Инкрементальная обработка на потоке")?;
    let purpose = make_purpose(
        doc,
        "Три задачи, для которых streaming существует: SHA-256 файла, \
         подсчёт строк, поиск маркера. Память O(1) — не зависит от размера файла.",
    )?;
    block.append_child(&purpose)?;

    let controls = doc.create_element("div")?;

    let task_select: HtmlSelectElement = doc
        .create_element("select")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlSelectElement"))?;
    task_select.set_id("rd-c-task");
    for (val, label) in [
        ("hash", "SHA-256 файла"),
        ("lines", "Подсчёт строк"),
        ("marker", "Поиск маркера"),
    ] {
        let opt: web_sys::HtmlOptionElement = doc
            .create_element("option")?
            .dyn_into()
            .map_err(|_| JsValue::from_str("не HtmlOptionElement"))?;
        opt.set_value(val);
        opt.set_text_content(Some(label));
        task_select.append_child(&opt)?;
    }
    controls.append_child(&task_select)?;

    let marker_input: web_sys::HtmlInputElement = doc
        .create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlInputElement"))?;
    marker_input.set_type("text");
    marker_input.set_id("rd-c-marker");
    marker_input.set_value("END");
    marker_input.set_placeholder("маркер");
    marker_input.set_disabled(true);
    controls.append_child(&marker_input)?;

    let run_btn = make_button(doc, "Запустить", "rd-c-run")?;
    run_btn.set_disabled(true);
    let cancel_btn = make_button(doc, "Отмена", "rd-c-cancel")?;
    cancel_btn.set_disabled(true);
    controls.append_child(&run_btn)?;
    controls.append_child(&cancel_btn)?;
    block.append_child(&controls)?;

    let progress_row = doc.create_element("div")?;
    let progress: HtmlProgressElement = doc
        .create_element("progress")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlProgressElement"))?;
    progress.set_max(1.0);
    progress.set_value(0.0);
    let progress_label: HtmlElement = doc
        .create_element("span")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    progress_label.set_text_content(Some("0%"));
    progress_row.append_child(&progress)?;
    progress_row.append_child(&progress_label)?;
    block.append_child(&progress_row)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    let result = make_result(doc)?;
    block.append_child(&result)?;

    parent.append_child(&block)?;
    Ok(BlockCUi {
        task_select,
        marker_input,
        run_btn,
        cancel_btn,
        progress,
        progress_label,
        status,
        result,
    })
}

fn wire_block_c(ui: &BlockCUi, state: Rc<RefCell<DemoState>>) {
    // Смена задачи → включить/выключить поле маркера.
    {
        let select_c = ui.task_select.clone();
        let marker_c = ui.marker_input.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let is_marker = select_c.value() == "marker";
            marker_c.set_disabled(!is_marker);
        });
        ui.task_select
            .set_onchange(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // Запуск.
    {
        // Клонируем ДО move.
        let task_c = ui.task_select.clone();
        let marker_c = ui.marker_input.clone();
        let status_c = ui.status.clone();
        let result_c = ui.result.clone();
        let progress_c = ui.progress.clone();
        let progress_label_c = ui.progress_label.clone();
        let cancel_c = ui.cancel_btn.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let task_el = task_c.clone();
            let marker_el = marker_c.clone();
            let status_el = status_c.clone();
            let result_el = result_c.clone();
            let progress_el = progress_c.clone();
            let progress_label_el = progress_label_c.clone();
            let cancel_el = cancel_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                let file = match file_of(&state) {
                    Some(f) => f,
                    None => {
                        set_status(&status_el, "blocked", "Сначала выберите файл.");
                        return;
                    }
                };
                let cancel_flag = state.borrow().cancel.clone();
                cancel_flag.set(false);
                cancel_el.set_disabled(false);

                let task: String = task_el.value();
                let t0 = js_sys::Date::now();

                let progress_for_cb = progress_el.clone();
                let label_for_cb = progress_label_el.clone();
                let on_progress = move |frac: f64| {
                    progress_for_cb.set_value(frac);
                    let pct: String = format_str(format_args!("{:.0}%", frac * 100.0));
                    label_for_cb.set_text_content(Some(&pct));
                };

                let result_text: String;
                let status_class: &str;

                if task == "hash" {
                    set_status(&status_el, "", "SHA-256 на потоке…");
                    let flag = cancel_flag.clone();
                    match fsa_read_ops::hash_stream(&file, on_progress, move || flag.get()).await {
                        Ok(r) => {
                            result_text = fsa_read_ops::format_hash_result(&r, file.size() as u64);
                            status_class = "ok";
                        }
                        Err(ReadError::Cancelled) => {
                            result_text = "Отменено — SHA-256 не сохранён (промежуточных значений нет).".to_string();
                            status_class = "blocked";
                        }
                        Err(e) => {
                            result_text = fsa_read_plan::explain_error(&e);
                            status_class = "failed";
                        }
                    }
                } else if task == "lines" {
                    set_status(&status_el, "", "Подсчёт строк на потоке…");
                    let flag = cancel_flag.clone();
                    match fsa_read_ops::count_lines_stream(&file, on_progress, move || flag.get()).await {
                        Ok(r) => {
                            result_text = fsa_read_ops::format_lines_result(&r);
                            status_class = "ok";
                        }
                        Err(e) => {
                            result_text = fsa_read_plan::explain_error(&e);
                            status_class = "failed";
                        }
                    }
                } else {
                    let marker: String = marker_el.value();
                    let marker_bytes: Vec<u8> = marker.as_bytes().to_vec();
                    set_status(&status_el, "", "Поиск маркера на потоке…");
                    let flag = cancel_flag.clone();
                    match fsa_read_ops::find_marker_stream(&file, &marker_bytes, on_progress, move || flag.get()).await {
                        Ok(r) => {
                            result_text = fsa_read_ops::format_marker_result(&r, &marker_bytes);
                            status_class = "ok";
                        }
                        Err(e) => {
                            result_text = fsa_read_plan::explain_error(&e);
                            status_class = "failed";
                        }
                    }
                }

                result_el.set_text_content(Some(&result_text));
                let dt = js_sys::Date::now() - t0;
                bump(&state, dt, &task);
                cancel_el.set_disabled(true);
                set_status(&status_el, status_class, "Задача завершена.");
            });
        });
        ui.run_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // Отмена.
    {
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let state = state_c.clone();
            let flag = state.borrow().cancel.clone();
            flag.set(true);
        });
        ui.cancel_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }
}

// ---------------------------------------------------------------------------
// Wire: pick file
// ---------------------------------------------------------------------------

fn wire_pick(
    pick_btn: &HtmlButtonElement,
    info: &HtmlElement,
    a: &BlockAUi,
    b: &BlockBUi,
    c: &BlockCUi,
    state: Rc<RefCell<DemoState>>,
) {
    // Клонируем ДО move.
    let info_c = info.clone();
    let a_detect_c = a.detect_btn.clone();
    let b_compare_c = b.compare_btn.clone();
    let c_run_c = c.run_btn.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let info_el = info_c.clone();
        let a_detect = a_detect_c.clone();
        let b_compare = b_compare_c.clone();
        let c_run = c_run_c.clone();
        let state = state.clone();
        spawn_local(async move {
            match pick_file().await {
                Ok(Some(file)) => {
                    let name = file.name();
                    let size = file.size() as u64;
                    let mime = file.type_();
                    let limits = ReadLimits::demo();
                    let plan = fsa_read_plan::suggest_plan(size, &mime, &limits);
                    let plan_note = fsa_read_plan::explain_plan(&plan, size, &mime);
                    let mime_display: &str = if mime.is_empty() { "—" } else { mime.as_str() };
                    let text: String = format_str(format_args!(
                        "Файл:   {name}\n\
                         Размер: {}\n\
                         MIME:   {mime_display}\n\
                         План:   {plan_note}",
                        fsa_read_plan::format_size(size),
                    ));
                    info_el.set_text_content(Some(&text));

                    state.borrow_mut().file = Some(file);
                    a_detect.set_disabled(false);
                    b_compare.set_disabled(false);
                    c_run.set_disabled(false);
                }
                Ok(None) => {
                    info_el.set_text_content(Some("Выбор отменён."));
                }
                Err(_) => {
                    info_el.set_text_content(Some("Диалог закрыт."));
                }
            }
        });
    });
    pick_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

// ---------------------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------------------

async fn pick_file() -> Result<Option<File>, JsValue> {
    let opts = fsa_choose_exec::open_files(JsValue::UNDEFINED).await;
    match opts {
        PickOutcome::Picked(handles) => {
            let h = match handles.into_iter().next() {
                Some(h) => h,
                None => return Ok(None),
            };
            let file = crate::fsa_api_file::get_file(&h).await?;
            return Ok(Some(file));
        }
        PickOutcome::Cancelled => return Ok(None),
        PickOutcome::Blocked(_) | PickOutcome::Failed(_) => return Ok(None),
    }
}

fn file_of(state: &Rc<RefCell<DemoState>>) -> Option<File> {
    return state.try_borrow().ok().and_then(|s| s.file.clone());
}

fn bump(state: &Rc<RefCell<DemoState>>, dt: f64, label: &str) {
    let stats: Rc<RefCell<DemoStats>> = state.borrow().stats.clone();
    let mut s = stats.borrow_mut();
    s.ops += 1;
    s.last_ms = dt;
    s.last_label = label.to_string();
}

fn make_button(
    doc: &web_sys::Document,
    text: &str,
    id: &str,
) -> Result<HtmlButtonElement, JsValue> {
    let b: HtmlButtonElement = doc
        .create_element("button")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlButtonElement"))?;
    b.set_id(id);
    b.set_text_content(Some(text));
    Ok(b)
}

fn make_block(doc: &web_sys::Document, title: &str) -> Result<HtmlElement, JsValue> {
    let block: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    block.set_attribute("class", "block")?;
    let h3 = doc.create_element("h3")?;
    h3.set_text_content(Some(title));
    block.append_child(&h3)?;
    Ok(block)
}

fn make_purpose(doc: &web_sys::Document, text: &str) -> Result<HtmlElement, JsValue> {
    let p: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    p.set_attribute("class", "purpose")?;
    p.set_text_content(Some(text));
    Ok(p)
}

fn make_status(doc: &web_sys::Document) -> Result<HtmlElement, JsValue> {
    let s: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    s.set_attribute("class", "status")?;
    s.set_text_content(Some("Готово."));
    Ok(s)
}

fn make_result(doc: &web_sys::Document) -> Result<HtmlElement, JsValue> {
    let r: HtmlElement = doc
        .create_element("pre")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    r.set_attribute("class", "result")?;
    r.set_text_content(Some("(результат появится здесь)"));
    Ok(r)
}

fn set_status(target: &HtmlElement, class: &str, text: &str) {
    let cls: String = if class.is_empty() {
        "status".to_string()
    } else {
        format_str(format_args!("status {class}"))
    };
    let _ = target.set_attribute("class", &cls);
    target.set_text_content(Some(text));
}

// ---------------------------------------------------------------------------
// Тексты
// ---------------------------------------------------------------------------

fn unsupported_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(6);
    lines.push("✗ showOpenFilePicker недоступен — эта подглава требует FSA.".to_string());
    lines.push(String::new());
    lines.push("Откройте демо в Chrome или Edge (86+) на HTTPS или localhost.".to_string());
    return lines.join("\n");
}

fn help_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(12);
    lines.push("Как работать с демо:".to_string());
    lines.push(String::new());
    lines.push(
        "1.  Нажмите «Выбрать файл». Демо покажет размер, MIME и \
         рекомендованный план чтения."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "A.  «Определить формат» — peek 8 байт и определение по magic. \
         Первое, что нужно сделать с незнакомым файлом."
            .to_string(),
    );
    lines.push(
        "B.  «Сравнить методы» — таблица: применимость и время для \
         text / bytes / slice / stream на одном файле. Видно, что Bytes \
         отказывает для больших файлов, а Peek и Slice работают всегда."
            .to_string(),
    );
    lines.push(
        "C.  Выберите задачу — SHA-256, подсчёт строк или поиск маркера. \
         Все три работают на потоке, память O(1). Кнопка «Отмена» \
         прерывает чтение — видно, что стриминг поддерживает отмену \
         в любой момент."
            .to_string(),
    );
    return lines.join("\n");
}
```

---

## Что демонстрация показывает

**Блок A: peek — первое действие с незнакомым файлом.**
8 байт, распознавание формата по magic, всё за доли миллисекунды. Пользователь видит не «вот вам hex», а конкретный ответ: «PNG (изображение)».

**Блок B: сравнение методов в одной таблице.**
На одном файле — `Peek` / `Text` / `Bytes` / `Slice` / `Stream`. Колонка «Применим» показывает `✗ MIME` для `Text` на бинарнике и `✗ размер` для `Bytes` на больших файлах. Колонка «Время» показывает, что `Peek` и `Slice` быстрее полной загрузки.

**Блок C: инкрементальная обработка.**
Три задачи, где streaming реально нужен: SHA-256, подсчёт строк, поиск маркера. Прогресс-бар показывает активность, отмена работает в любой момент. Обработка идёт чанк за чанком, память O(1) независимо от размера файла.

## Правила проекта, зафиксированные в 19.4

1. **`app.rs` содержит только демо 19.4** — предыдущие подглавы не накапливаются.
2. **Внутри `Closure::new(move || …)` нельзя захватывать `&T` от параметра функции.** `spawn_local` требует `'static` — все нужные элементы клонируются **до** `move`. Это правило применяется во всех wire-функциях (`wire_block_a`, `wire_block_b`, `wire_block_c`, `wire_pick`).
3. **Все строки через `format_str` или `Vec<String>.join`.**
4. **`return` в хвостах макросов** (`matches!`, `format!`) — устраняет «expected X, found ()».
5. **Лимиты — параметр.** `ReadLimits::demo()` в проде, `ReadLimits { max_buffer_bytes: 1024, ... }` в тестах.
6. **План не выполняется, если `validate_plan` вернул ошибку.** Ошибка возвращается до вызова JS.
7. **Пустой MIME не блокирует `Text`.** File может не иметь type; отказ выдаётся только на непустой несовместимый MIME.
8. **`File::new_with_blob_sequence_and_options`** в тестах — `File` не наследует MIME от `Blob`.
9. **Каждый блок демо самодостаточен** — свой статус, результат, отмена.
10. **`cargo check` без предупреждений.**

## Команды

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_fsa_read_plan
npm start
```

Ожидаемый результат тестов: **38 passed, 0 failed**.