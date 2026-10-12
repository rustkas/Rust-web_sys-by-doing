# Глава 18. File API

## 18.9. `Blob.slice`: диапазоны и большие файлы

В подглаве 18.8 мы читали только первые байты файла — чтобы проверить формат. Но что, если нужно прочитать файл **целиком**, а он большой? Загружать 500-мегабайтный файл в память целиком — плохая идея: браузер может упасть, пользователь — уйти. Правильный подход — **читать файл по частям** через `Blob.slice`. Это тот же механизм, который используют потоковые загрузчики, chunked upload и resumable download. В этой подглаве разберём `slice`, разобьём файл на чанки, склеим обратно и убедимся, что байты не портятся.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod blob_slice;
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.9
│   ├── blob_slice.rs              # НОВЫЙ: модуль подглавы 18.9
│   ├── validation.rs              # из 18.8, не меняется
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
│   └── wasm_blob_slice.rs         # НОВЫЙ: тесты подглавы 18.9
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/blob_slice.rs` — модуль, вводимый этой подглавой. Автономен: зависит только от `web-sys`, `js-sys`, `wasm-bindgen`.
- `src/lib.rs` — **обязательно** дополняется строкой `pub mod blob_slice;`.
- `src/app.rs` — обновлён: добавлена секция «18.9. Blob.slice».
- `tests/wasm_blob_slice.rs` — новый файл с тестами.

Новых фич в `Cargo.toml` не требуется: `Blob`, `BlobPropertyBag`, `FileReader` уже включены.

---

## Ссылки для детального изучения

- **MDN: Blob.slice()** — https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice
- **MDN: Blob.size** — https://developer.mozilla.org/en-US/docs/Web/API/Blob/size
- **MDN: Blob.type** — https://developer.mozilla.org/en-US/docs/Web/API/Blob/type
- **MDN: Blob() constructor** — https://developer.mozilla.org/en-US/docs/Web/API/Blob/Blob
- **web-sys: Blob** — https://docs.rs/web-sys/latest/web_sys/struct.Blob.html
- **web-sys: BlobPropertyBag** — https://docs.rs/web-sys/latest/web_sys/struct.BlobPropertyBag.html

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
pub mod validation;     // 18.8
pub mod blob_slice;     // 18.9 ← НОВОЕ
```

---

### `src/blob_slice.rs`

Модуль автономен. Содержит обёртки над `Blob::slice`, асинхронное чтение среза через `FileReader`, разбиение на чанки и обратную склейку.

```rust
//! Blob.slice — чтение частей файла без загрузки целиком.
//!
//! Модуль автономен: не зависит от других модулей проекта.
//!
//! `Blob.slice(start, end)` создаёт «окно» в исходный Blob — без
//! копирования данных. Это позволяет читать большие файлы по
//! частям и собирать их обратно без риска исчерпать память.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileReader};

/// Возвращает Blob-срез от `start` до `end`.
///
/// Семантика совпадает с `Blob.slice(start, end)`:
/// * `start` включительно, `end` не включается;
/// * если `end` больше размера Blob, срез идёт до конца;
/// * если `start >= size`, вернётся пустой Blob.
///
/// Данные не копируются: срез — это «окно» в исходный Blob.
/// Реальное копирование происходит только при чтении.
pub fn slice_blob(blob: &Blob, start: f64, end: f64) -> Result<Blob, JsValue> {
    return blob.slice_with_f64_and_f64(start, end);
}

/// Читает Blob как `Vec<u8>`.
///
/// Приватный helper для этой подглавы: превращает Blob в массив
/// байтов Rust через `FileReader::readAsArrayBuffer`.
async fn read_blob_bytes(blob: &Blob) -> Result<Vec<u8>, JsValue> {
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

    let result = reader.result()?;
    let array_buffer: js_sys::ArrayBuffer = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let uint8 = js_sys::Uint8Array::new(&array_buffer);
    return Ok(uint8.to_vec());
}

/// Читает срез `[start, end)` как `Vec<u8>`.
pub async fn read_slice(blob: &Blob, start: f64, end: f64) -> Result<Vec<u8>, JsValue> {
    let slice = slice_blob(blob, start, end)?;
    return read_blob_bytes(&slice).await;
}

/// Читает первые `n` байт из Blob.
///
/// Удобная обёртка над `read_slice(blob, 0.0, n as f64)`.
/// Если файл короче `n`, вернётся всё содержимое.
pub async fn read_first_n(blob: &Blob, n: u32) -> Result<Vec<u8>, JsValue> {
    return read_slice(blob, 0.0, n as f64).await;
}

/// Считает количество чанков для файла размера `size`
/// при размере чанка `chunk_size`.
///
/// Возвращает `0`, если `size <= 0` или `chunk_size <= 0`.
pub fn chunk_count(size: f64, chunk_size: f64) -> u32 {
    if chunk_size <= 0.0 || size <= 0.0 {
        return 0;
    }
    let n: f64 = (size / chunk_size).ceil();
    return n as u32;
}

/// Разбивает Blob на последовательность чанков по `chunk_size` байт.
///
/// Последний чанк может быть короче — он содержит остаток.
/// Если `chunk_size <= 0`, возвращается пустой вектор.
pub fn split_into_chunks(blob: &Blob, chunk_size: f64) -> Vec<Blob> {
    let mut chunks: Vec<Blob> = Vec::new();
    if chunk_size <= 0.0 {
        return chunks;
    }

    let size = blob.size();
    let mut offset = 0.0_f64;
    while offset < size {
        let end: f64 = (offset + chunk_size).min(size);
        if let Ok(chunk) = blob.slice_with_f64_and_f64(offset, end) {
            chunks.push(chunk);
        }
        offset = end;
    }
    return chunks;
}

/// Склеивает последовательность Blob в один с заданным MIME-типом.
///
/// Использует конструктор `new Blob([...parts], {type})` — тот же
/// приём, что и `vec_to_blob` из подглавы 18.3, только на входе
/// не `Uint8Array`, а готовые Blob-объекты.
pub fn concat_blobs(chunks: &[Blob], mime_type: &str) -> Result<Blob, JsValue> {
    let parts = js_sys::Array::new();
    for chunk in chunks {
        parts.push(chunk.as_ref());
    }
    let options = web_sys::BlobPropertyBag::new();
    options.set_type(mime_type);
    return Blob::new_with_blob_sequence_and_options(&parts, &options);
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — для `JsValue`, `Closure`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`, `.unchecked_ref()`, `.as_ref()`.
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{Blob, FileReader}` — оба используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- **`Blob::slice` не копирует данные.** Возвращается ссылка на тот же буфер с ограничением диапазона. Реальное чтение происходит только в `FileReader`.
- **Границы slice:** `start` включительно, `end` не включается. Если `end > size`, срез идёт до конца.
- **Отрицательные индексы:** в JS `slice(-100, -1)` отсчитывает от конца. Мы работаем через `f64`, поэтому отрицательные значения передаются «как есть» и обрабатываются браузером по спецификации. В этой подглаве мы используем только положительные значения.
- **`split_into_chunks`** создаёт N Blob-объектов, каждый из которых — «окно» в исходный файл. Память расходуется только на метаданные (обычно десятки байт на объект), а не на сами данные.
- **`concat_blobs`** использует конструктор `Blob([...parts])` — он принимает массив Blob-ов и создаёт новый Blob-контейнер, не копируя данные частей немедленно.

---

### `src/app.rs` — секция 18.9

Функция `build_blob_slice_section` самодостаточна: input, кнопка, асинхронный анализ файла.

```rust
//! Демонстрация подглавы 18.9: Blob.slice.
//!
//! Секция: выбрать файл → прочитать первые 64 КБ, разбить на
//! чанки по 1 МБ, показать первые байты каждого, склеить обратно.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{blob_slice, dom, file_api, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.9 ----
    build_blob_slice_section(&container, &closures)?;

    log!("Section 18.9 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.9: Blob.slice
// ---------------------------------------------------------------

fn build_blob_slice_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.9. Blob.slice: диапазоны и большие файлы"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите файл любого размера — прочитаем первые 64 КБ через \
         slice, разобьём файл на чанки по 1 МБ, покажем первые байты \
         каждого чанка и склеим всё обратно. Для больших файлов \
         операция почти мгновенная — данные не копируются.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Проанализировать"));
    dom::append(&section, &btn)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файл и нажмите «Проанализировать»)"));
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
            result_c.set_text_content(Some("Анализируем..."));
            spawn_local(async move {
                let message = analyze_file(&file).await;
                result_c.set_text_content(Some(&message));
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.9
// ---------------------------------------------------------------

/// Анализирует файл: читает первые 64 КБ, разбивает на чанки,
/// склеивает обратно и проверяет совпадение размеров.
async fn analyze_file(file: &web_sys::File) -> String {
    const HEADER_SIZE: u32 = 64 * 1024;      // 64 КБ
    const CHUNK_SIZE: f64 = 1024.0 * 1024.0; // 1 МБ
    const MAX_CHUNK_PREVIEWS: usize = 5;     // показываем первые байты
                                              // только для первых 5 чанков

    let total_size = file.size();
    let file_name = file.name();

    // ---- 1. Читаем первые 64 КБ. ----
    let head_start = js_sys::Date::now();
    let head = match blob_slice::read_first_n(file, HEADER_SIZE).await {
        Ok(v) => v,
        Err(e) => {
            let message = format!("❌ read_first_n: {e:?}");
            return message;
        }
    };
    let head_elapsed = js_sys::Date::now() - head_start;

    // ---- 2. Разбиваем на чанки. ----
    let chunks = blob_slice::split_into_chunks(file, CHUNK_SIZE);
    let chunk_count = chunks.len();

    // Читаем первые байты каждого из первых MAX_CHUNK_PREVIEWS чанков.
    let preview_count = chunk_count.min(MAX_CHUNK_PREVIEWS);
    let mut previews: Vec<String> = Vec::with_capacity(preview_count);
    for i in 0..preview_count {
        let chunk = &chunks[i];
        match blob_slice::read_first_n(chunk, 16).await {
            Ok(bytes) => {
                let hex = format_hex_preview(&bytes, 8);
                let line = format!(
                    "  Чанк {} ({}): {}",
                    i + 1,
                    file_api::format_file_size(chunk.size()),
                    hex,
                );
                previews.push(line);
            }
            Err(e) => {
                let line = format!("  Чанк {}: ❌ {e:?}", i + 1);
                previews.push(line);
            }
        }
    }

    // ---- 3. Склеиваем обратно. ----
    let mime = file.type_();
    let concat_result = blob_slice::concat_blobs(&chunks, &mime);
    let (concat_size, concat_ok): (f64, bool);
    match concat_result {
        Ok(blob) => {
            concat_size = blob.size();
            concat_ok = (concat_size - total_size).abs() < 0.5;
        }
        Err(_) => {
            concat_size = 0.0;
            concat_ok = false;
        }
    }

    return format_slice_report(
        &file_name,
        total_size,
        &head,
        head_elapsed,
        chunk_count,
        CHUNK_SIZE,
        &previews,
        concat_size,
        concat_ok,
    );
}

/// Формирует итоговый отчёт о разборе файла.
///
/// Ветки `if/else` присваивают в предобъявленные переменные —
/// безопасный паттерн для окружения проекта (см. подглавы 18.6–18.8).
fn format_slice_report(
    file_name: &str,
    total_size: f64,
    head: &[u8],
    head_elapsed_ms: f64,
    chunk_count: usize,
    chunk_size: f64,
    previews: &[String],
    concat_size: f64,
    concat_ok: bool,
) -> String {
    let head_hex = format_hex_preview(head, 16);
    let head_len = head.len();

    let chunk_summary: String;
    if chunk_count == 0 {
        chunk_summary = "Чанков нет (файл пуст).".to_string();
    } else {
        let full_chunks = if total_size > 0.0 {
            (total_size / chunk_size).floor() as u64
        } else {
            0
        };
        let tail_size = total_size - (full_chunks as f64) * chunk_size;

        let tail_display: String;
        if tail_size > 0.5 {
            tail_display = format!(
                ", последний — {}",
                file_api::format_file_size(tail_size)
            );
        } else {
            tail_display = String::new();
        }

        chunk_summary = format!(
            "Количество чанков: {chunk_count} (по {}{tail_display})",
            file_api::format_file_size(chunk_size),
        );
    }

    let previews_block: String;
    if previews.is_empty() {
        previews_block = "(нет данных)".to_string();
    } else {
        let mut lines = String::new();
        for line in previews {
            lines.push_str(line);
            lines.push('\n');
        }
        if chunk_count > previews.len() {
            lines.push_str(&format!(
                "  … и ещё {} чанк(ов)\n",
                chunk_count - previews.len()
            ));
        }
        previews_block = lines;
    }

    let concat_status: String;
    if concat_ok {
        concat_status = format!(
            "✅ совпадает с исходным ({})",
            file_api::format_file_size(concat_size)
        );
    } else {
        concat_status = format!(
            "❌ НЕ совпадает: получено {}, ожидалось {}",
            file_api::format_file_size(concat_size),
            file_api::format_file_size(total_size),
        );
    }

    return format!(
        "Файл: {file_name}\n\
         Полный размер: {}\n\n\
         === Первые 64 КБ ===\n\
         Прочитано: {head_len} байт за {head_elapsed_ms:.1} мс\n\
         Первые байты: {head_hex}\n\n\
         === Разбиение на чанки ===\n\
         {chunk_summary}\n\
         {previews_block}\n\
         === Склейка обратно ===\n\
         {concat_status}",
        file_api::format_file_size(total_size),
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

- `RefCell`, `Rc` — используются в `run()` и `build_blob_slice_section`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` при получении `HtmlInputElement`.
- `wasm_bindgen_futures::spawn_local` — для `async`-блока.
- `callbacks`, `EventClosures`, `blob_slice`, `dom`, `file_api`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- **`read_first_n(file, 64 * 1024)`** — для 500-мегабайтного файла это мгновенная операция: читается только 64 КБ, остальное не трогается.
- **`split_into_chunks(file, 1 MB)`** — создаёт N Blob-объектов. Данные не копируются: каждый чанк — это «окно» в исходный файл.
- **Превью первых байт каждого чанка** — для первых 5 чанков. Если чанков больше, остальные не показываем.
- **`concat_blobs`** — склеивает чанки обратно. Проверяем, что суммарный размер совпадает с исходным.
- **`format_slice_report`** — все `format!` внутри `let`-присваиваний и tail-`return`. Ветки `if/else` используют предобъявленные переменные.

---

### `tests/wasm_blob_slice.rs`

Тесты проверяют границы slice, разбиение и склейку.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::blob_slice::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Создаёт Blob из последовательности байт.
fn make_blob(bytes: &[u8]) -> web_sys::Blob {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::of1(&uint8.buffer());
    return web_sys::Blob::new_with_u8_array_sequence(&parts)
        .expect("Blob::new_with_u8_array_sequence");
}

#[wasm_bindgen_test]
fn test_slice_blob_size() {
    let blob = make_blob(&[0u8; 100]);
    let slice = slice_blob(&blob, 10.0, 30.0).expect("slice_blob");
    assert_eq!(slice.size(), 20.0);
}

#[wasm_bindgen_test]
fn test_slice_beyond_end_clamps() {
    // end больше размера — срез идёт до конца.
    let blob = make_blob(&[0u8; 50]);
    let slice = slice_blob(&blob, 40.0, 1000.0).expect("slice_blob");
    assert_eq!(slice.size(), 10.0);
}

#[wasm_bindgen_test]
fn test_slice_start_beyond_size_is_empty() {
    // start >= size — пустой Blob.
    let blob = make_blob(&[0u8; 50]);
    let slice = slice_blob(&blob, 100.0, 200.0).expect("slice_blob");
    assert_eq!(slice.size(), 0.0);
}

#[wasm_bindgen_test]
async fn test_read_first_n_exact() {
    let bytes: Vec<u8> = (0..100).collect();
    let blob = make_blob(&bytes);

    let read = read_first_n(&blob, 32).await.expect("read_first_n");
    assert_eq!(read.len(), 32);
    assert_eq!(read[0], 0);
    assert_eq!(read[31], 31);
}

#[wasm_bindgen_test]
async fn test_read_first_n_truncates() {
    let bytes: Vec<u8> = (0..10).collect();
    let blob = make_blob(&bytes);

    let read = read_first_n(&blob, 100).await.expect("read_first_n");
    assert_eq!(read.len(), 10);
}

#[wasm_bindgen_test]
async fn test_read_slice_middle() {
    let bytes: Vec<u8> = (0..100).collect();
    let blob = make_blob(&bytes);

    let read = read_slice(&blob, 20.0, 25.0).await.expect("read_slice");
    assert_eq!(read, vec![20u8, 21, 22, 23, 24]);
}

#[wasm_bindgen_test]
fn test_chunk_count() {
    assert_eq!(chunk_count(1000.0, 100.0), 10);
    assert_eq!(chunk_count(1050.0, 100.0), 11);
    assert_eq!(chunk_count(0.0, 100.0), 0);
    assert_eq!(chunk_count(100.0, 0.0), 0);
    assert_eq!(chunk_count(100.0, -1.0), 0);
}

#[wasm_bindgen_test]
fn test_split_into_chunks_exact() {
    let blob = make_blob(&[0u8; 300]);
    let chunks = split_into_chunks(&blob, 100.0);
    assert_eq!(chunks.len(), 3);
    for chunk in &chunks {
        assert_eq!(chunk.size(), 100.0);
    }
}

#[wasm_bindgen_test]
fn test_split_into_chunks_with_tail() {
    let blob = make_blob(&[0u8; 250]);
    let chunks = split_into_chunks(&blob, 100.0);
    assert_eq!(chunks.len(), 3);
    assert_eq!(chunks[0].size(), 100.0);
    assert_eq!(chunks[1].size(), 100.0);
    assert_eq!(chunks[2].size(), 50.0);
}

#[wasm_bindgen_test]
fn test_split_into_chunks_invalid_size() {
    let blob = make_blob(&[0u8; 100]);
    let chunks = split_into_chunks(&blob, 0.0);
    assert!(chunks.is_empty());
}

#[wasm_bindgen_test]
async fn test_concat_roundtrip() {
    let original: Vec<u8> = (0..100u8).collect();
    let blob = make_blob(&original);
    let chunks = split_into_chunks(&blob, 30.0);
    assert_eq!(chunks.len(), 4);

    let recombined = concat_blobs(&chunks, "application/octet-stream")
        .expect("concat_blobs");
    assert_eq!(recombined.size(), 100.0);

    // Читаем результат и сравниваем побайтово.
    let bytes = read_first_n(&recombined, 100).await.expect("read");
    assert_eq!(bytes, original);
}

#[wasm_bindgen_test]
async fn test_chunks_preserve_content() {
    // Проверяем, что содержимое чанков не портится.
    let original: Vec<u8> = (0..60u8).collect();
    let blob = make_blob(&original);
    let chunks = split_into_chunks(&blob, 20.0);

    let c0 = read_first_n(&chunks[0], 20).await.expect("c0");
    let c1 = read_first_n(&chunks[1], 20).await.expect("c1");
    let c2 = read_first_n(&chunks[2], 20).await.expect("c2");

    assert_eq!(c0, (0..20u8).collect::<Vec<_>>());
    assert_eq!(c1, (20..40u8).collect::<Vec<_>>());
    assert_eq!(c2, (40..60u8).collect::<Vec<_>>());
}
```

**Проверка импортов:**

- `wasm_bindgen_test::*` — `wasm_bindgen_test`, `wasm_bindgen_test_configure`.
- `hello_wasm02::blob_slice::*` — `slice_blob`, `read_first_n`, `read_slice`, `chunk_count`, `split_into_chunks`, `concat_blobs`.

Полные пути `js_sys::Uint8Array`, `js_sys::Array`, `web_sys::Blob` — без импортов. Неиспользуемых нет.

**Что проверяем:**

- **Границы slice** — точный размер, обрезка `end > size`, пустой результат при `start >= size`.
- **`read_first_n` и `read_slice`** — точное чтение и усечение для коротких файлов.
- **`chunk_count`** — кратные, некратные, нулевые, отрицательные случаи.
- **`split_into_chunks`** — точное разбиение, с хвостом, с невалидным размером.
- **Round-trip** — содержимое чанков не портится: оригинал 100 байт → 4 чанка → склейка → чтение 100 байт даёт исходный массив.

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

### Тесты подглавы 18.9

```bash
cargo check --target wasm32-unknown-unknown --tests

wasm-pack test --headless --chrome --test wasm_blob_slice
wasm-pack test --headless --firefox --test wasm_blob_slice

npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.9. Blob.slice: диапазоны и большие файлы** с полем выбора файла и кнопкой «Проанализировать».

**Сценарий 1 — маленький файл (300 КБ):**
```
Файл: small.bin
Полный размер: 300.00 KB

=== Первые 64 КБ ===
Прочитано: 65536 байт за 3.2 мс
Первые байты: 89 50 4E 47 0D 0A 1A 0A 00 00 00 0D 49 48 44 52 …

=== Разбиение на чанки ===
Количество чанков: 1 (по 1.00 MB)
  Чанк 1 (300.00 KB): 89 50 4E 47 0D 0A 1A 0A …

=== Склейка обратно ===
✅ совпадает с исходным (300.00 KB)
```

**Сценарий 2 — большой файл (25 МБ):**
```
Файл: video.mp4
Полный размер: 25.00 MB

=== Первые 64 КБ ===
Прочитано: 65536 байт за 2.8 мс
Первые байты: 00 00 00 18 66 74 79 70 69 73 6F 6D 00 00 02 00 …

=== Разбиение на чанки ===
Количество чанков: 25 (по 1.00 MB)
  Чанк 1 (1.00 MB): 00 00 00 18 66 74 79 70 …
  Чанк 2 (1.00 MB): 4D 44 41 54 00 02 00 00 …
  Чанк 3 (1.00 MB): 9C 1F A3 E2 88 01 CC DD …
  Чанк 4 (1.00 MB): 15 8F 04 A2 00 00 00 00 …
  Чанк 5 (1.00 MB): 6A C3 88 01 00 01 02 03 …
  … и ещё 20 чанк(ов)

=== Склейка обратно ===
✅ совпадает с исходным (25.00 MB)
```

Обратите внимание: **первые 64 КБ большого файла прочитались за 2–3 миллисекунды**. Если бы мы читали весь 25-мегабайтный файл целиком, это заняло бы сотни миллисекунд и заблокировало UI.

---

## Итоги подглавы 18.9

- **`Blob::slice(start, end)`** — создаёт «окно» в исходный Blob без копирования данных.
- **Границы slice** — `start` включительно, `end` не включается. `end > size` обрезается до конца. `start >= size` даёт пустой Blob.
- **`read_first_n`** — быстрое чтение заголовка большого файла.
- **`chunk_count` / `split_into_chunks`** — разбиение на чанки фиксированного размера, последний содержит остаток.
- **`concat_blobs`** — обратная склейка через конструктор `new Blob([...parts])`.
- **Данные не копируются** до момента чтения: чанки — это лёгкие метаданные, а не копии байтов.

Модуль `blob_slice.rs` автономен, все tail-функции оформлены через `return`, неиспользуемых импортов нет. Демо-секция самодостаточна и демонстрирует три ключевые операции: slice, split, concat.

## Замечания на будущее

Три правила, зафиксированные в этой подглаве:

1. **`Blob::slice` не копирует данные.** Это «окно» в исходный буфер. Реальное чтение происходит только в `FileReader`. Поэтому разбиение 25-мегабайтного файла на 25 чанков — мгновенная операция, а не копирование 25 МБ.

2. **Все границы slice — `f64`.** В web-sys методы `slice_with_f64_and_f64` принимают `f64`. Целочисленные индексы нужно приводить: `n as f64`. Отрицательные значения работают по спецификации JS (отсчёт от конца), но в этой подглаве мы используем только положительные.

3. **`concat_blobs` использует `Blob::new_with_blob_sequence_and_options`** — конструктор, принимающий массив Blob-объектов. Не путайте с `new_with_u8_array_sequence`, который принимает массив `Uint8Array` (использовали в подглаве 18.3 для `vec_to_blob`).

В следующей подглаве (18.10) мы перейдём к **`Blob.stream()`** — потоковому чтению, где мы не только не копируем данные, но и не буферизуем их целиком.