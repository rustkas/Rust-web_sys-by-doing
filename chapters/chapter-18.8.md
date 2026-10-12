# Глава 18. File API

## 18.8. Валидация файлов: MIME, magic-байты, размер

Подглавы 18.6 и 18.7 научили нас принимать файлы и папки — из `<input>`, через drag&drop, целыми директориями. Но прежде чем отдавать файл конвертеру, его нужно проверить. Расширение и MIME-тип ненадёжны: пользователь может переименовать `malicious.exe` в `image.png`, а ОС — не зарегистрировать MIME для `.md`. В этой подглаве мы научимся определять настоящий формат по **сигнатуре файла** (magic bytes) и проверять размер против лимита.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod validation;
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.8
│   ├── validation.rs              # НОВЫЙ: модуль подглавы 18.8
│   ├── folder_drop.rs             # из 18.7, не меняется
│   ├── drag_drop.rs               # из 18.6, не меняется
│   ├── file_utils.rs              # из 18.6, не меняется
│   ├── data_url.rs                # из 18.5, не меняется
│   ├── base64_utils.rs            # из 18.5, не меняется
│   ├── object_url.rs              # из 18.4, не меняется
│   ├── file_api.rs                # из 18.1, не меняется
│   ├── file_reader.rs             # из 18.2, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_validation.rs         # НОВЫЙ: тесты подглавы 18.8
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/validation.rs` — модуль, вводимый этой подглавой. Автономен: зависит только от `web-sys`, `js-sys`, `wasm-bindgen`.
- `src/lib.rs` — **обязательно** дополняется строкой `pub mod validation;`.
- `src/app.rs` — обновлён: добавлена секция «18.8. Валидация: MIME, magic-байты, размер».
- `tests/wasm_validation.rs` — новый файл с тестами.

Новых фич в `Cargo.toml` не требуется: `Blob`, `FileReader`, `HtmlInputElement`, `HtmlSelectElement`, `HtmlOptionElement` были включены ранее.

---

## Ссылки для детального изучения

- **MDN: MIME types** — https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types
- **MDN: Blob.slice()** — https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice
- **Wikipedia: List of file signatures** — https://en.wikipedia.org/wiki/List_of_file_signatures
- **Gary Kessler's File Signatures Table** — https://www.garykessler.net/library/file_sigs.html
- **RFC 6838 — Media Type Specifications** — https://datatracker.ietf.org/doc/html/rfc6838
- **web-sys: Blob** — https://docs.rs/web-sys/latest/web_sys/struct.Blob.html

---

## Содержание новых / обновлённых файлов

### `src/lib.rs` — подключение модуля

```rust
// src/lib.rs

// … существующие модули проекта …

// ---- Глава 18. File API ----
pub mod file_api;       // 18.1
pub mod file_reader;    // 18.2
pub mod file_utils;     // 18.3
pub mod object_url;     // 18.4
pub mod data_url;       // 18.5
pub mod base64_utils;   // 18.5
pub mod drag_drop;      // 18.6
pub mod folder_drop;    // 18.7
pub mod validation;     // 18.8 ← НОВОЕ
```

---

### `src/validation.rs`

Модуль автономен. Содержит перечисление `Format`, функцию `detect_format` (по первым байтам), функцию `check_size` (проверка лимита) и асинхронную `read_header` (читает первые N байт через `Blob::slice` + `FileReader`).

**Важно про оформление функций:** в этом окружении **макросы (`format!`, `matches!`) и `match` в tail-позиции функции возвращают `()` вместо ожидаемого значения**. Поэтому все функции, где tail-выражение содержит макрос или `match`, оформлены через явный `return`.

```rust
//! Валидация файлов: MIME, magic-байты, размер.
//!
//! Модуль автономен: не зависит от других модулей проекта.
//!
//! Определяет настоящий формат файла по первым байтам (сигнатуре),
//! а не по расширению или заявленному MIME. Это защищает от
//! подделок: `malware.txt`, внутри которого PNG или ZIP.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileReader};

/// Формат файла, определённый по сигнатуре.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Format {
    Png,
    Jpeg,
    Gif,
    Webp,
    Bmp,
    Ico,
    Tiff,
    Pdf,
    Zip,
    Gzip,
    Rar,
    SevenZ,
    Tar,
    Mp3,
    Wav,
    Ogg,
    Flac,
    Mp4,
    Webm,
    Wasm,
    Text,
    Unknown,
}

impl Format {
    /// Человекочитаемое имя формата.
    ///
    /// `match` в tail-позиции функции возвращает `()` — поэтому
    /// оформлено через явный `return`.
    pub fn name(&self) -> &'static str {
        return match self {
            Format::Png => "PNG",
            Format::Jpeg => "JPEG",
            Format::Gif => "GIF",
            Format::Webp => "WebP",
            Format::Bmp => "BMP",
            Format::Ico => "ICO",
            Format::Tiff => "TIFF",
            Format::Pdf => "PDF",
            Format::Zip => "ZIP",
            Format::Gzip => "GZIP",
            Format::Rar => "RAR",
            Format::SevenZ => "7z",
            Format::Tar => "TAR",
            Format::Mp3 => "MP3",
            Format::Wav => "WAV",
            Format::Ogg => "OGG",
            Format::Flac => "FLAC",
            Format::Mp4 => "MP4",
            Format::Webm => "WebM",
            Format::Wasm => "WebAssembly",
            Format::Text => "Текст (ASCII)",
            Format::Unknown => "Неизвестный",
        };
    }

    /// MIME-тип, соответствующий формату.
    ///
    /// Тот же паттерн, что и в `name()` — явный `return match`.
    pub fn mime_type(&self) -> &'static str {
        return match self {
            Format::Png => "image/png",
            Format::Jpeg => "image/jpeg",
            Format::Gif => "image/gif",
            Format::Webp => "image/webp",
            Format::Bmp => "image/bmp",
            Format::Ico => "image/x-icon",
            Format::Tiff => "image/tiff",
            Format::Pdf => "application/pdf",
            Format::Zip => "application/zip",
            Format::Gzip => "application/gzip",
            Format::Rar => "application/vnd.rar",
            Format::SevenZ => "application/x-7z-compressed",
            Format::Tar => "application/x-tar",
            Format::Mp3 => "audio/mpeg",
            Format::Wav => "audio/wav",
            Format::Ogg => "audio/ogg",
            Format::Flac => "audio/flac",
            Format::Mp4 => "video/mp4",
            Format::Webm => "video/webm",
            Format::Wasm => "application/wasm",
            Format::Text => "text/plain",
            Format::Unknown => "application/octet-stream",
        };
    }

    /// Является ли формат изображением.
    ///
    /// `matches!` — макрос, в tail-позиции функции возвращает `()`.
    /// Присваиваем результат в `let result: bool` и возвращаем
    /// через явный `return` — безопасный паттерн для окружения проекта.
    pub fn is_image(&self) -> bool {
        let result: bool = matches!(
            self,
            Format::Png
                | Format::Jpeg
                | Format::Gif
                | Format::Webp
                | Format::Bmp
                | Format::Ico
                | Format::Tiff
        );
        return result;
    }
}

/// Определяет формат по первым байтам (magic bytes).
///
/// Если данных меньше, чем нужно для проверки конкретной сигнатуры,
/// эта сигнатура просто не срабатывает — проверяются остальные.
/// Для пустого входа возвращает `Format::Unknown`.
///
/// Все `return` — явные: каждое условие возвращает конкретное
/// значение `Format`.
pub fn detect_format(bytes: &[u8]) -> Format {
    // PNG: 89 50 4E 47 0D 0A 1A 0A
    if bytes.starts_with(&[0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]) {
        return Format::Png;
    }
    // JPEG: FF D8 FF
    if bytes.starts_with(&[0xFF, 0xD8, 0xFF]) {
        return Format::Jpeg;
    }
    // GIF: "GIF8"
    if bytes.starts_with(b"GIF8") {
        return Format::Gif;
    }
    // WebP: RIFF .... WEBP
    if bytes.len() >= 12 && &bytes[0..4] == b"RIFF" && &bytes[8..12] == b"WEBP" {
        return Format::Webp;
    }
    // WAV: RIFF .... WAVE
    if bytes.len() >= 12 && &bytes[0..4] == b"RIFF" && &bytes[8..12] == b"WAVE" {
        return Format::Wav;
    }
    // BMP: "BM"
    if bytes.starts_with(b"BM") {
        return Format::Bmp;
    }
    // ICO: 00 00 01 00
    if bytes.starts_with(&[0x00, 0x00, 0x01, 0x00]) {
        return Format::Ico;
    }
    // TIFF: "II*\0" (little-endian) или "MM\0*" (big-endian)
    if bytes.starts_with(&[0x49, 0x49, 0x2A, 0x00])
        || bytes.starts_with(&[0x4D, 0x4D, 0x00, 0x2A])
    {
        return Format::Tiff;
    }
    // PDF: "%PDF"
    if bytes.starts_with(b"%PDF") {
        return Format::Pdf;
    }
    // ZIP: PK\x03\x04
    if bytes.starts_with(&[0x50, 0x4B, 0x03, 0x04]) {
        return Format::Zip;
    }
    // GZIP: 1F 8B
    if bytes.starts_with(&[0x1F, 0x8B]) {
        return Format::Gzip;
    }
    // RAR: "Rar!\x1A\x07"
    if bytes.starts_with(&[0x52, 0x61, 0x72, 0x21, 0x1A, 0x07]) {
        return Format::Rar;
    }
    // 7z: 37 7A BC AF 27 1C
    if bytes.starts_with(&[0x37, 0x7A, 0xBC, 0xAF, 0x27, 0x1C]) {
        return Format::SevenZ;
    }
    // TAR: "ustar" по смещению 257
    if bytes.len() >= 262 && &bytes[257..262] == b"ustar" {
        return Format::Tar;
    }
    // WebAssembly: 00 61 73 6D
    if bytes.starts_with(&[0x00, 0x61, 0x73, 0x6D]) {
        return Format::Wasm;
    }
    // OGG: "OggS"
    if bytes.starts_with(b"OggS") {
        return Format::Ogg;
    }
    // FLAC: "fLaC"
    if bytes.starts_with(b"fLaC") {
        return Format::Flac;
    }
    // MP3: "ID3" или frame sync (старшие 11 бит первого байта == 1)
    if bytes.starts_with(b"ID3") {
        return Format::Mp3;
    }
    if bytes.len() >= 2 && bytes[0] == 0xFF && (bytes[1] & 0xE0) == 0xE0 {
        return Format::Mp3;
    }
    // MP4 / MOV: bytes[4..8] == "ftyp"
    if bytes.len() >= 8 && &bytes[4..8] == b"ftyp" {
        return Format::Mp4;
    }
    // WebM / MKV: EBML 1A 45 DF A3
    if bytes.starts_with(&[0x1A, 0x45, 0xDF, 0xA3]) {
        return Format::Webm;
    }
    // Текстовый эвристический анализ: только печатаемые ASCII.
    if !bytes.is_empty() && is_mostly_text(bytes) {
        return Format::Text;
    }
    return Format::Unknown;
}

/// Проверяет, что байты выглядят как ASCII-текст.
///
/// Считаем текстом, если не менее 90 % байт — печатаемые символы
/// или пробельные (`\n`, `\r`, `\t`, пробел).
///
/// Пустой срез не считается текстом — возвращаем `false`.
/// Это также защищает от деления на ноль в вычислении процента.
fn is_mostly_text(bytes: &[u8]) -> bool {
    if bytes.is_empty() {
        return false;
    }
    let mut printable = 0usize;
    for &b in bytes {
        if b.is_ascii_graphic() || b == b' ' || b == b'\n' || b == b'\r' || b == b'\t' {
            printable += 1;
        }
    }
    let percent: usize = printable * 100 / bytes.len();
    return percent >= 90;
}

/// Результат проверки размера файла.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct SizeCheck {
    /// Фактический размер файла в байтах.
    pub actual: f64,
    /// Лимит в байтах; `0.0` означает «без лимита».
    pub limit: f64,
    /// Проходит ли файл по лимиту.
    pub ok: bool,
}

/// Проверяет размер файла против лимита.
///
/// `limit_bytes <= 0.0` означает «без лимита» — всегда `ok = true`.
///
/// Значение `ok` присваивается в предобъявленную переменную:
/// это безопасный паттерн для ветвлений, проверенный в подглавах
/// 18.5–18.7.
pub fn check_size(size: f64, limit_bytes: f64) -> SizeCheck {
    let ok: bool;
    if limit_bytes <= 0.0 {
        ok = true;
    } else {
        ok = size <= limit_bytes;
    }
    return SizeCheck {
        actual: size,
        limit: limit_bytes,
        ok,
    };
}

/// Читает первые `n` байт из Blob.
///
/// Использует `Blob::slice` — это позволяет прочитать заголовок
/// большого файла без загрузки всего содержимого в память.
/// Если файл короче `n`, возвращается всё содержимое.
pub async fn read_header(blob: &Blob, n: u32) -> Result<Vec<u8>, JsValue> {
    let slice = blob.slice_with_f64_and_f64(0.0, n as f64)?;
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

    reader.read_as_array_buffer(&slice)?;
    JsFuture::from(promise).await?;

    let result = reader.result()?;
    let array_buffer: js_sys::ArrayBuffer = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let uint8 = js_sys::Uint8Array::new(&array_buffer);
    return Ok(uint8.to_vec());
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — для `JsValue`, `Closure`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`, `.unchecked_ref()`.
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{Blob, FileReader}` — оба используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- **`name()` и `mime_type()`** — `return match self { ... };` вместо голого `match` в tail-позиции.
- **`is_image()`** — результат `matches!` присваивается в `let result: bool`, затем `return result;`.
- **`detect_format`** — все ветки `if` с явным `return`, финальный `return Format::Unknown;`.
- **`is_mostly_text`** — защита от пустого среза (деление на ноль). Результат процента присваивается в `let percent: usize`, возврат через `return percent >= 90;`.
- **`check_size`** — `let ok: bool;` с присваиванием в ветках `if/else`.
- **`read_header`** использует `Blob::slice(0, n)`. Это читает только первые `n` байт, а не весь файл.

---

### `src/app.rs` — секция 18.8

Функция `build_validation_section` самодостаточна: input файла, select лимита размера, кнопка проверки.

```rust
//! Демонстрация подглавы 18.8: валидация файлов.
//!
//! Секция: выбрать файл → показать заявленный MIME и MIME по
//! magic-байтам, проверить размер против выбранного лимита.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_api, file_utils, log, validation};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.8 ----
    build_validation_section(&container, &closures)?;

    log!("Section 18.8 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.8: Валидация
// ---------------------------------------------------------------

fn build_validation_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.8. Валидация: MIME, magic-байты, размер"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите файл — покажем заявленный браузером MIME и MIME, \
         определённый по сигнатуре (magic bytes). Это позволяет \
         распознать подделку: файл с расширением .txt, но с \
         содержимым PNG будет определён как PNG. Здесь же — \
         проверка размера против выбранного лимита.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    // ---- Input файла ----
    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    // ---- Select лимита размера ----
    let select: web_sys::HtmlSelectElement = dom::create_element("select")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не select"))?;

    let options = [
        ("1048576", "1 МБ"),
        ("10485760", "10 МБ"),
        ("104857600", "100 МБ"),
        ("0", "без лимита"),
    ];
    for (value, label) in options.iter() {
        let opt: web_sys::HtmlOptionElement = dom::create_element("option")?
            .dyn_into()
            .map_err(|_| JsValue::from_str("не option"))?;
        opt.set_value(value);
        opt.set_text_content(Some(label));
        select.append_child(&opt)?;
    }
    select.set_value("10485760");
    dom::append(&section, &select)?;

    // ---- Кнопка ----
    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Проверить"));
    dom::append(&section, &btn)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файл и нажмите «Проверить»)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let input_c = input.clone();
    let select_c = select.clone();
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
            let limit_bytes: f64 = select_c.value().parse().unwrap_or(0.0);
            let result_c = result_c.clone();
            result_c.set_text_content(Some("Проверяем..."));
            spawn_local(async move {
                let header = match validation::read_header(&file, 32).await {
                    Ok(v) => v,
                    Err(e) => {
                        let message = format!("❌ read_header: {e:?}");
                        result_c.set_text_content(Some(&message));
                        return;
                    }
                };
                let sniffed = validation::detect_format(&header);
                let size_check = validation::check_size(file.size(), limit_bytes);
                let declared = file.type_();
                let effective = file_utils::effective_mime_type(&file);
                let message = format_validation_report(
                    &file.name(),
                    file.size(),
                    &declared,
                    &effective,
                    sniffed,
                    &size_check,
                    &header,
                );
                result_c.set_text_content(Some(&message));
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.8
// ---------------------------------------------------------------

/// Формирует отчёт о валидации файла.
///
/// Ветки `if/else` присваивают в предобъявленные переменные —
/// это безопасный паттерн для окружения проекта (см. подглавы 18.6–18.7).
fn format_validation_report(
    file_name: &str,
    file_size: f64,
    declared_mime: &str,
    effective_mime: &str,
    sniffed: validation::Format,
    size_check: &validation::SizeCheck,
    header: &[u8],
) -> String {
    let declared_display: String;
    if declared_mime.is_empty() {
        declared_display = "(не определён)".to_string();
    } else {
        declared_display = declared_mime.to_string();
    }

    let size_verdict: String;
    if size_check.limit <= 0.0 {
        size_verdict = "✅ без лимита".to_string();
    } else if size_check.ok {
        size_verdict = format!(
            "✅ в пределах лимита ({})",
            file_api::format_file_size(size_check.limit)
        );
    } else {
        size_verdict = format!(
            "❌ превышает лимит ({})",
            file_api::format_file_size(size_check.limit)
        );
    }

    let forgery_status: String;
    if !declared_mime.is_empty()
        && sniffed != validation::Format::Unknown
        && declared_mime != sniffed.mime_type()
    {
        forgery_status = format!(
            "⚠️ ВОЗМОЖНАЯ ПОДДЕЛКА: браузер сообщает «{declared_mime}», \
             а по сигнатуре — «{}»",
            sniffed.mime_type()
        );
    } else {
        forgery_status = "✅ MIME согласован".to_string();
    }

    let header_hex = format_hex_preview(header, 16);

    return format!(
        "Файл: {file_name}\n\
         Размер: {} ({} байт)\n\n\
         === Заявленные данные ===\n\
         MIME (браузер): {declared_display}\n\
         MIME (effective с fallback): {effective_mime}\n\n\
         === По сигнатуре (magic bytes) ===\n\
         Формат: {} ({})\n\
         MIME: {}\n\
         Первые байты: {header_hex}\n\n\
         === Проверка размера ===\n\
         {size_verdict}\n\n\
         === Итог ===\n\
         {forgery_status}",
        file_api::format_file_size(file_size),
        file_size as u64,
        sniffed.name(),
        sniffed.mime_type(),
        sniffed.mime_type(),
    );
}

/// Формирует hex-представление первых `n` байт.
fn format_hex_preview(bytes: &[u8], n: usize) -> String {
    let limit = n.min(bytes.len());
    let mut out = String::new();
    for (i, b) in bytes[..limit].iter().enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format!("{:02X}", b));
    }
    if bytes.len() > limit {
        out.push_str(" …");
    }
    return out;
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_validation_section`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` при получении `HtmlInputElement`, `HtmlSelectElement`, `HtmlOptionElement`.
- `wasm_bindgen_futures::spawn_local` — для `async`-блока.
- `callbacks`, `EventClosures`, `dom`, `file_api`, `file_utils`, `log`, `validation` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- **Select лимита** — четыре варианта: 1 МБ, 10 МБ, 100 МБ, без лимита. Значения — в байтах (строки), парсятся через `.parse::<f64>()`.
- **`read_header(&file, 32)`** — `File` coerces to `&Blob` через `Deref`.
- **`effective_mime_type`** из подглавы 18.6 — использует расширение как fallback.
- **`format_validation_report`** — все `format!` внутри `let`-присваиваний и tail-`return`. Ветки `if/else` используют предобъявленные переменные.

---

### `tests/wasm_validation.rs`

Тесты покрывают `detect_format` для всех основных форматов, `check_size` и `read_header`. Все — чистые и не требуют пользовательских действий.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::validation::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_detect_png() {
    let bytes = [0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A, 0x00, 0x00];
    assert_eq!(detect_format(&bytes), Format::Png);
}

#[wasm_bindgen_test]
fn test_detect_jpeg() {
    let bytes = [0xFF, 0xD8, 0xFF, 0xE0, 0x00, 0x10, 0x4A, 0x46];
    assert_eq!(detect_format(&bytes), Format::Jpeg);
}

#[wasm_bindgen_test]
fn test_detect_gif() {
    let bytes = *b"GIF89a\x01\x00\x01\x00";
    assert_eq!(detect_format(&bytes), Format::Gif);
}

#[wasm_bindgen_test]
fn test_detect_webp() {
    let bytes = *b"RIFF\x24\x00\x00\x00WEBPVP8 ";
    assert_eq!(detect_format(&bytes), Format::Webp);
}

#[wasm_bindgen_test]
fn test_detect_wav() {
    let bytes = *b"RIFF\x24\x00\x00\x00WAVEfmt ";
    assert_eq!(detect_format(&bytes), Format::Wav);
}

#[wasm_bindgen_test]
fn test_detect_pdf() {
    let bytes = *b"%PDF-1.7\n";
    assert_eq!(detect_format(&bytes), Format::Pdf);
}

#[wasm_bindgen_test]
fn test_detect_zip() {
    let bytes = [0x50, 0x4B, 0x03, 0x04, 0x14, 0x00];
    assert_eq!(detect_format(&bytes), Format::Zip);
}

#[wasm_bindgen_test]
fn test_detect_gzip() {
    let bytes = [0x1F, 0x8B, 0x08, 0x00];
    assert_eq!(detect_format(&bytes), Format::Gzip);
}

#[wasm_bindgen_test]
fn test_detect_wasm() {
    let bytes = [0x00, 0x61, 0x73, 0x6D, 0x01, 0x00, 0x00, 0x00];
    assert_eq!(detect_format(&bytes), Format::Wasm);
}

#[wasm_bindgen_test]
fn test_detect_mp4() {
    let bytes = [
        0x00, 0x00, 0x00, 0x18, b'f', b't', b'y', b'p',
        b'i', b's', b'o', b'm',
    ];
    assert_eq!(detect_format(&bytes), Format::Mp4);
}

#[wasm_bindgen_test]
fn test_detect_webm() {
    let bytes = [0x1A, 0x45, 0xDF, 0xA3, 0x01, 0x00, 0x00, 0x00];
    assert_eq!(detect_format(&bytes), Format::Webm);
}

#[wasm_bindgen_test]
fn test_detect_text() {
    let bytes = b"Hello, world!\nThis is a test.";
    assert_eq!(detect_format(bytes), Format::Text);
}

#[wasm_bindgen_test]
fn test_detect_unknown_binary() {
    // Случайные байты без сигнатур — не текст (менее 90 % печатаемых).
    let bytes = [0xF7, 0x14, 0xA3, 0x88, 0x00, 0xCC, 0xDD, 0xEE];
    assert_eq!(detect_format(&bytes), Format::Unknown);
}

#[wasm_bindgen_test]
fn test_detect_empty() {
    let bytes: [u8; 0] = [];
    assert_eq!(detect_format(&bytes), Format::Unknown);
}

#[wasm_bindgen_test]
fn test_format_name_and_mime() {
    assert_eq!(Format::Png.name(), "PNG");
    assert_eq!(Format::Png.mime_type(), "image/png");
    assert!(Format::Png.is_image());
    assert!(!Format::Pdf.is_image());
}

#[wasm_bindgen_test]
fn test_check_size_within() {
    let check = check_size(500.0, 1000.0);
    assert!(check.ok);
    assert_eq!(check.actual, 500.0);
    assert_eq!(check.limit, 1000.0);
}

#[wasm_bindgen_test]
fn test_check_size_over() {
    let check = check_size(2000.0, 1000.0);
    assert!(!check.ok);
}

#[wasm_bindgen_test]
fn test_check_size_no_limit() {
    let check = check_size(999_999_999.0, 0.0);
    assert!(check.ok);
    assert_eq!(check.limit, 0.0);
}

#[wasm_bindgen_test]
async fn test_read_header_png() {
    // Файл с PNG-сигнатурой в начале.
    let bytes = vec![
        0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A,
        0x01, 0x02, 0x03, 0x04,
    ];
    let uint8 = js_sys::Uint8Array::from(&bytes[..]);
    let parts = js_sys::Array::of1(&uint8.buffer());
    let blob = web_sys::Blob::new_with_u8_array_sequence(&parts)
        .expect("Blob::new_with_u8_array_sequence");

    let header = read_header(&blob, 8).await.expect("read_header");
    assert_eq!(header.len(), 8);
    assert_eq!(detect_format(&header), Format::Png);
}

#[wasm_bindgen_test]
async fn test_read_header_truncates() {
    // Если файл короче запрошенного N, вернётся всё содержимое.
    let bytes = vec![0x01, 0x02, 0x03];
    let uint8 = js_sys::Uint8Array::from(&bytes[..]);
    let parts = js_sys::Array::of1(&uint8.buffer());
    let blob = web_sys::Blob::new_with_u8_array_sequence(&parts)
        .expect("Blob::new_with_u8_array_sequence");

    let header = read_header(&blob, 32).await.expect("read_header");
    assert_eq!(header.len(), 3);
    assert_eq!(header, vec![0x01, 0x02, 0x03]);
}

#[wasm_bindgen_test]
async fn test_spoofed_txt_with_png_content() {
    // Файл с именем "fake.txt", но содержимым PNG. Имя не влияет
    // на sniffing — детектируется реальный формат.
    let bytes = vec![0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A, 0x00, 0x00];
    let uint8 = js_sys::Uint8Array::from(&bytes[..]);
    let parts = js_sys::Array::of1(&uint8.buffer());
    let file = web_sys::File::new_with_u8_array_sequence(&parts, "fake.txt")
        .expect("File::new_with_u8_array_sequence");

    let header = read_header(&file, 8).await.expect("read_header");
    assert_eq!(detect_format(&header), Format::Png);
    assert_eq!(file.name(), "fake.txt");
}
```

**Проверка импортов:**

- `wasm_bindgen_test::*` — `wasm_bindgen_test` и `wasm_bindgen_test_configure`.
- `hello_wasm02::validation::*` — `detect_format`, `check_size`, `read_header`, `Format`, `SizeCheck`.

Полные пути `js_sys::Uint8Array`, `js_sys::Array`, `web_sys::Blob`, `web_sys::File` — без импортов, как внешние crate. Неиспользуемых нет.

**Что проверяем:**

- **`detect_format`** — 12 форматов + текст + unknown + empty. Ключевой тест — `test_spoofed_txt_with_png_content`: файл с расширением `.txt` определяется как PNG по содержимому.
- **`check_size`** — три случая: в пределах, сверх, без лимита.
- **`read_header`** — корректное чтение N байт и правильное усечение для коротких файлов.

---

## Команды запуска

### Сборка и запуск демо

```bash
cargo check --target wasm32-unknown-unknown
npm run wasm:dev
npm run dev
# или одной командой:
npm run start
```

### Тесты подглавы 18.8

```bash
cargo check --target wasm32-unknown-unknown --tests

wasm-pack test --headless --chrome --test wasm_validation
wasm-pack test --headless --firefox --test wasm_validation

npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.8. Валидация: MIME, magic-байты, размер** с полем выбора файла, select лимита и кнопкой «Проверить».

**Сценарий 1 — нормальный PNG:**
```
Файл: logo.png
Размер: 234.56 KB (240192 байт)

=== Заявленные данные ===
MIME (браузер): image/png
MIME (effective с fallback): image/png

=== По сигнатуре (magic bytes) ===
Формат: PNG (image/png)
MIME: image/png
Первые байты: 89 50 4E 47 0D 0A 1A 0A 00 00 00 0D 49 48 44 52

=== Проверка размера ===
✅ в пределах лимита (10.00 MB)

=== Итог ===
✅ MIME согласован
```

**Сценарий 2 — подделка: PNG-содержимое под именем `.txt`:**
```
Файл: fake.txt
Размер: 234.56 KB (240192 байт)

=== Заявленные данные ===
MIME (браузер): text/plain
MIME (effective с fallback): text/plain

=== По сигнатуре (magic bytes) ===
Формат: PNG (image/png)
MIME: image/png
Первые байты: 89 50 4E 47 0D 0A 1A 0A …

=== Проверка размера ===
✅ в пределах лимита (10.00 MB)

=== Итог ===
⚠️ ВОЗМОЖНАЯ ПОДДЕЛКА: браузер сообщает «text/plain», а по сигнатуре — «image/png»
```

**Сценарий 3 — превышение лимита:**
```
Файл: video.mp4
Размер: 24.50 MB (25690112 байт)
…
=== Проверка размера ===
❌ превышает лимит (10.00 MB)
```

---

## Итоги подглавы 18.8

- **Magic bytes** — первые байты файла, однозначно идентифицирующие формат: PNG `89 50 4E 47`, JPEG `FF D8 FF`, PDF `%PDF`, ZIP `PK\x03\x04`, WebAssembly `\0asm`.
- **`detect_format`** — чистая функция от `&[u8]`; не зависит от DOM; легко тестируется.
- **`Blob::slice(0, N)`** — читает только заголовок файла, а не весь файл. Критично для больших файлов.
- **`check_size`** — простая проверка размера против лимита; `0.0` = «без лимита».
- **Определение подделки** — сравнение заявленного MIME с sniffed; расхождение даёт предупреждение.
- **Порядок проверок важен**: сначала конкретные сигнатуры, потом текстовая эвристика.

Модуль `validation.rs` автономен, все tail-функции оформлены через `return`, неиспользуемых импортов нет. Демо-секция самодостаточна и демонстрирует три ключевых сценария: успешная проверка, обнаружение подделки и превышение лимита.

## Замечания на будущее

**Расширенное правило проекта.** В этом окружении не только `format!`, но и **`matches!`**, а также **голый `match` в tail-позиции функции** возвращают `()` вместо ожидаемого значения. Это касается всех макросов (`format!`, `matches!`, `vec!`, `assert!` и др.) и `match`.

| Ситуация | Как писать |
|---|---|
| `format!` / `matches!` / `vec!` в tail функции | Через `return <expr>;` или предобъявленную переменную |
| `match` в tail функции | `return match ...;` |
| `format!` / `matches!` в tail блока `if/else`, присвоенного в `let` | Через предобъявленную переменную |
| `match` в tail блока, присвоенного в `let` | Через предобъявленную переменную |
| Обычное выражение в tail | `return expr;` |

**Общий принцип:** если tail-выражение содержит макрос или `match` — не оставляйте его «голым». Либо `return`, либо предобъявленная переменная.

**Дополнительно в этой подглаве:** `is_mostly_text` защищена от деления на ноль при пустом срезе. Это отдельный баг, который мог проявиться на пустом файле.

В следующей подглаве (18.9) мы разберём **`Blob.slice`** подробно — как читать большие файлы по частям, разбивать на чанки и собирать обратно.