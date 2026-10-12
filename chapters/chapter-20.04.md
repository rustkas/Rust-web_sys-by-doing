# Глава 20.4. Практический проект: Blob → PNG + JSON

## 20.4.1. Зачем эта глава

Три подглавы подряд изучали Blob с разных сторон. **20.2** — жизненный цикл Object URL: как create/revoke связаны с памятью, почему «забыть ссылку» — не то же самое, что «освободить». **20.3** — три пути отдать Blob пользователю: тихий `<a download>`, диалог `showSaveFilePicker`, системный шер `navigator.share`. **20.4** собирает всё это в один практический workflow.

Пользователь даёт файл. Мы его читаем кусками, считаем статистику, рисуем гистограмму, формируем JSON-метаданные. На выходе — два независимых артефакта с автогенерированными именами. Один вход, два выхода.

Что студент видит своими глазами:

**Blob двусторонний.** До этого мы только создавали Blob из байтов. Здесь — читаем `Blob.arrayBuffer()` обратно в байты, `Blob.text()` — как строку, `Blob.slice()` — по частям. Ментальная модель замыкается: Blob не «конечный файл», а буфер с обеими сторонами.

**`Blob.slice()` — рабочий инструмент.** Файл читается кусками, а не целиком в память. Размер куска подобран под файл так, чтобы число чанков было около 128 — независимо от того, 1 КБ это или 1 ГБ.

**Yield через `TimeoutFuture::new(0)`** — то самое правило проекта. Без него UI обновится только в конце. С ним — прогресс-бар, ETA и живая гистограмма обновляются 60 раз в секунду.

**Отмена — первоклассная операция.** `Rc<Cell<bool>>` выставляется из обработчика кнопки и мгновенно виден в цикле обработки. Никаких borrow-конфликтов: `Cell` даёт внутреннюю изменяемость без заимствования.

**Два пути скачивания из 20.3 в деле.** `<a download>` — тихий, везде. `showSaveFilePicker` — диалог «Сохранить как…», Chrome/Edge. Рядом на одном экране.

**Wow.** Загрузите 50 МБ — увидите, как гистограмма строится в реальном времени, ETA уменьшается, скорость колеблется. Нажмите «Отменить» — процесс остановится мгновенно.

---

## 20.4.2. Обновлённая структура

```
src/
  blob_processor.rs        ← модуль главы 20.4 (для тестов и внешнего использования)
  download_paths.rs        ← из 20.3 (используется в других главах)
  object_url_lifecycle.rs  ← из 20.2 (используется косвенно)
  format_str.rs            ← из 20.2 (используется)
tests/
  wasm_blob_processor.rs   ← тест модуля
src/lib.rs                 ← +1 строка
src/app.rs                 ← полностью автономный демо-пример главы
```

**Ключевое правило проекта.** `app.rs` содержит **только** демо текущей подглавы. Все зависимости инкапсулированы **внутри** файла: утилита `format_str`, функции чтения Blob, обработка, рисование, скачивание. Студент открывает `app.rs` — и видит ровно то, что относится к теме. Никаких `use crate::…` из соседних модулей, которые могут «пролиться» деталями других глав.

Модуль `blob_processor.rs` при этом **существует отдельно** — он используется тестами `wasm_blob_processor`, а также другими главами, если им понадобится этот же конвейер. Демо дублирует его интерфейс, а не импортирует. Это не копипаст ради копипаста: `app.rs` — самодостаточная страница для изучения, а модуль — переиспользуемый инструмент для тестов и других контекстов.

## Ссылки

- MDN: [`Blob.arrayBuffer()`](https://developer.mozilla.org/en-US/docs/Web/API/Blob/arrayBuffer)
- MDN: [`Blob.slice()`](https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice)
- MDN: [`OffscreenCanvas.convertToBlob()`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/convertToBlob)
- web.dev: [Async functions: yielding to the browser](https://web.dev/articles/async-functions)
- HTML Spec: [`<a download>`](https://html.spec.whatwg.org/multipage/links.html#attr-hyperlink-download)
- MDN: [`FileSystemWritableFileStream`](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemWritableFileStream)

---

# Глава 20.4 — Полные исходные коды

## 1. `src/blob_processor.rs` — модуль

```rust
//! 20.4. Blob → PNG + JSON: практический проект.
//!
//! Один вход (Blob), два выхода (PNG + JSON).
//! Показывает три вещи:
//! 1. Blob двусторонний (`arrayBuffer()` возвращает байты обратно).
//! 2. `Blob.slice()` — чтение по кускам, а не «всё в память».
//! 3. Yield через `TimeoutFuture::new(0)` — иначе UI обновится в конце.
//!
//! Размер куска **не задаётся константой**. Он вычисляется под размер
//! файла, чтобы число кусков было ~128 независимо от того, 1 КБ это или 1 ГБ.

use std::cell::Cell;
use std::rc::Rc;

use js_sys::{Array, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Blob, BlobPropertyBag, CanvasRenderingContext2d,
    OffscreenCanvas, OffscreenCanvasRenderingContext2d,
};

/// Строковый маркер отмены. Возвращается из `process_blob` как `Err`,
/// когда пользователь нажал «Отменить». UI проверяет через `js_err_to_string`.
pub const CANCELLED: &str = "cancelled-by-user";

// ---------- Публичные типы ----------

pub struct ProcessedArtifacts {
    pub png: Blob,
    pub json: Blob,
    pub metadata: Metadata,
}

/// Метрики файла. Считаются инкрементально по кускам.
///
/// `Default` реализован вручную: массив `[u32; 256]` не реализует `Default`
/// в стабильном Rust (derive работает только до `[T; 32]`).
#[derive(Clone, Debug)]
pub struct Metadata {
    pub total_bytes: usize,
    pub chunks: usize,
    pub chunk_size: usize,
    pub sha256: String,
    pub unique_bytes: usize,
    pub sum: u64,
    pub min_byte: u8,
    pub max_byte: u8,
    pub histogram: [u32; 256],
}

impl Default for Metadata {
    fn default() -> Self {
        Self {
            total_bytes: 0,
            chunks: 0,
            chunk_size: 0,
            sha256: String::new(),
            unique_bytes: 0,
            sum: 0,
            min_byte: 0,
            max_byte: 0,
            histogram: [0; 256],
        }
    }
}

impl Metadata {
    /// Ручная сериализация в JSON. Не используем serde, чтобы модуль
    /// оставался автономным (никаких derive-обвязок).
    pub fn to_json(&self) -> String {
        let mut hist = String::with_capacity(256 * 8);
        for (i, v) in self.histogram.iter().enumerate() {
            if i > 0 {
                hist.push(',');
            }
            hist.push_str(&v.to_string());
        }
        let s: String = format!(
            "{{\n\
             \"total_bytes\": {},\n\
             \"chunks\": {},\n\
             \"chunk_size\": {},\n\
             \"sha256\": \"{}\",\n\
             \"unique_bytes\": {},\n\
             \"sum\": {},\n\
             \"min_byte\": {},\n\
             \"max_byte\": {},\n\
             \"histogram\": [{}]\n\
             }}\n",
            self.total_bytes,
            self.chunks,
            self.chunk_size,
            self.sha256,
            self.unique_bytes,
            self.sum,
            self.min_byte,
            self.max_byte,
            hist,
        );
        s
    }
}

// ---------- Адаптивный размер куска ----------

/// Подбирает размер куска так, чтобы число кусков было около 128.
///
/// Границы:
/// - минимум 32 КБ — иначе накладные расходы на `arrayBuffer()` съедят выгоду;
/// - максимум 8 МБ — иначе прогресс становится «дёрганым» (один кусок
///   занимает сотни миллисекунд).
///
/// Примеры:
/// - 64 КБ → 32 КБ (минимум), 2 чанка;
/// - 4 МБ → 32 КБ, ровно 128 чанков;
/// - 64 МБ → 512 КБ, 128 чанков;
/// - 1 ГБ → 8 МБ (максимум), 128 чанков.
pub fn choose_chunk_size(total_bytes: usize) -> usize {
    const TARGET_CHUNKS: usize = 128;
    const MIN_CHUNK: usize = 32 * 1024;
    const MAX_CHUNK: usize = 8 * 1024 * 1024;
    if total_bytes == 0 {
        return MIN_CHUNK;
    }
    let ideal = total_bytes / TARGET_CHUNKS;
    ideal.clamp(MIN_CHUNK, MAX_CHUNK)
}

// ---------- Чтение ----------

/// Читает **весь** Blob как байты через `.arrayBuffer()`.
pub async fn read_blob_bytes(blob: &Blob) -> Result<Vec<u8>, JsValue> {
    let promise = blob.array_buffer();
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    let buffer: js_sys::ArrayBuffer = value.dyn_into()?;
    let u8arr = Uint8Array::new(&buffer);
    Ok(u8arr.to_vec())
}

/// Читает один кусок `blob[start..end]` как байты.
pub async fn read_slice_bytes(blob: &Blob, start: f64, end: f64) -> Result<Vec<u8>, JsValue> {
    let slice = blob.slice_with_f64_and_f64(start, end)?;
    read_blob_bytes(&slice).await
}

// ---------- Обработка ----------

/// Обрабатывает Blob кусками.
///
/// **Контракт `on_progress`:** минимум два вызова — `(0, total, _)` в начале
/// и `(total, total, &hist)` в конце. Промежуточные вызовы троттлятся до 60 Гц.
///
/// **Контракт `cancel`:** если `cancel.get() == true` на входе в новую
/// итерацию — функция возвращает `Err(JsValue::from_str(CANCELLED))`.
/// Идемпотентно: повторный вызов с уже установленным флагом сразу отменит.
pub async fn process_blob(
    blob: &Blob,
    chunk_size: usize,
    cancel: Rc<Cell<bool>>,
    mut on_progress: impl FnMut(usize, usize, &[u32; 256]),
) -> Result<Metadata, JsValue> {
    let total = blob.size() as usize;
    let mut meta = Metadata {
        total_bytes: total,
        chunk_size,
        min_byte: 255,
        max_byte: 0,
        ..Default::default()
    };

    on_progress(0, total, &meta.histogram);

    // Простой не-криптографический хеш — FNV-1a. Для демо достаточно отпечатка.
    let mut hash: u64 = 0xcbf2_9ce4_8422_2325;
    let fnv_prime: u64 = 0x0000_0100_0000_01b3;
    let fnv_mix = |h: u64, b: u8| -> u64 { (h ^ (b as u64)).wrapping_mul(fnv_prime) };

    let mut last_yield_ms = now_ms();
    let mut offset = 0usize;

    while offset < total {
        if cancel.get() {
            return Err(JsValue::from_str(CANCELLED));
        }

        let end = (offset + chunk_size).min(total);
        let bytes = read_slice_bytes(blob, offset as f64, end as f64).await?;

        for &b in &bytes {
            meta.histogram[b as usize] = meta.histogram[b as usize].saturating_add(1);
            meta.sum = meta.sum.wrapping_add(b as u64);
            if b < meta.min_byte {
                meta.min_byte = b;
            }
            if b > meta.max_byte {
                meta.max_byte = b;
            }
            hash = fnv_mix(hash, b);
        }

        meta.chunks += 1;
        offset = end;

        // Yield не каждый кусок, а раз в 16 мс — иначе UI не успеет отрисоваться.
        if now_ms() - last_yield_ms >= 16.0 {
            gloo_timers::future::TimeoutFuture::new(0).await;
            last_yield_ms = now_ms();
            on_progress(offset, total, &meta.histogram);
        }
    }

    // Явный yield перед финальным callback'ом: даём браузеру отрисовать
    // последний промежуточный кадр и гарантированно показать 100%.
    gloo_timers::future::TimeoutFuture::new(0).await;
    on_progress(total, total, &meta.histogram);

    meta.unique_bytes = meta.histogram.iter().filter(|&&v| v > 0).count();
    if total == 0 {
        meta.min_byte = 0;
    }
    meta.sha256 = format!("{:016x}", hash);
    Ok(meta)
}

// ---------- Рисование гистограммы ----------

/// Общий «скелет» рисования. `CanvasRenderingContext2d` и
/// `OffscreenCanvasRenderingContext2d` — разные типы в web-sys, но их API
/// идентичен (Canvas 2D API). Макрос разворачивает одну и ту же логику
/// в два биндера без дублирования кода.
macro_rules! histogram_draw_body {
    ($ctx:expr, $hist:expr, $w:expr, $h:expr) => {{
        $ctx.set_fill_style_str("#0d1117");
        $ctx.fill_rect(0.0, 0.0, $w, $h);

        $ctx.set_stroke_style_str("#21262d");
        $ctx.set_line_width(1.0);
        $ctx.begin_path();
        $ctx.move_to(0.0, $h - 0.5);
        $ctx.line_to($w, $h - 0.5);
        $ctx.stroke();

        let max = $hist.iter().copied().max().unwrap_or(1).max(1) as f64;
        let bar_w = $w / 256.0;
        $ctx.set_fill_style_str("#58a6ff");
        for (i, &count) in $hist.iter().enumerate() {
            if count == 0 {
                continue;
            }
            let bh = (count as f64 / max) * ($h - 20.0);
            let x = (i as f64) * bar_w;
            let y = $h - 10.0 - bh;
            $ctx.fill_rect(x, y, bar_w.max(1.0), bh);
        }

        $ctx.set_fill_style_str("#8b949e");
        $ctx.set_font("11px monospace");
        $ctx.fill_text("byte histogram (0..255)", 8.0, 14.0)?;
    }};
}

/// Рисует гистограмму на обычном `CanvasRenderingContext2d`
/// (используется для live-канваса в DOM).
pub fn draw_histogram(
    ctx: &CanvasRenderingContext2d,
    histogram: &[u32; 256],
    width: f64,
    height: f64,
) -> Result<(), JsValue> {
    histogram_draw_body!(ctx, histogram, width, height);
    Ok(())
}

/// Рисует гистограмму на `OffscreenCanvasRenderingContext2d`
/// (используется для PNG-генерации).
fn draw_histogram_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    histogram: &[u32; 256],
    width: f64,
    height: f64,
) -> Result<(), JsValue> {
    histogram_draw_body!(ctx, histogram, width, height);
    Ok(())
}

/// Генерирует PNG-гистограмму через `OffscreenCanvas`.
///
/// `convert_to_blob()` — Promise-версия (не callback). Именно она
/// экспортируется web-sys 0.3.105 под этим именем; у `HtmlCanvasElement`
/// `to_blob` — callback-API с другим контрактом.
pub async fn render_histogram_png(
    histogram: &[u32; 256],
    width: u32,
    height: u32,
) -> Result<Blob, JsValue> {
    let canvas = OffscreenCanvas::new(width, height)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;

    draw_histogram_offscreen(&ctx, histogram, width as f64, height as f64)?;

    let promise = canvas.convert_to_blob()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<Blob>()
}

// ---------- Конвейер ----------

/// Полный конвейер: обработка + генерация обоих артефактов.
pub async fn run_pipeline(
    input: &Blob,
    chunk_size: usize,
    cancel: Rc<Cell<bool>>,
    mut on_progress: impl FnMut(usize, usize, &[u32; 256]),
) -> Result<ProcessedArtifacts, JsValue> {
    let metadata = process_blob(input, chunk_size, cancel, &mut on_progress).await?;
    let png = render_histogram_png(&metadata.histogram, 820, 200).await?;
    let json = make_json_blob(&metadata)?;
    Ok(ProcessedArtifacts { png, json, metadata })
}

/// Оборачивает JSON-строку в Blob.
pub fn make_json_blob(meta: &Metadata) -> Result<Blob, JsValue> {
    let text = meta.to_json();
    let bytes = Uint8Array::from(text.as_bytes());
    let parts = Array::new();
    parts.push(bytes.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type("application/json");
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag)
}

fn now_ms() -> f64 {
    js_sys::Date::now()
}
```

---

## 2. `src/lib.rs` — добавление строки

Найдите блок `pub mod cache_storage;` и добавьте одну строку:

```rust
pub mod cache_storage;
pub mod idb_store;
pub mod opfs;
pub mod object_url_lifecycle;
pub mod format_str;
pub mod download_paths;
pub mod blob_processor;   // ← глава 20.4
```

---

## 3. `tests/wasm_blob_processor.rs` — тесты

```rust
#![cfg(target_arch = "wasm32")]

use std::cell::Cell;
use std::rc::Rc;

use wasm_bindgen_test::*;

use hello_wasm02::blob_processor::{
    choose_chunk_size, make_json_blob, process_blob, read_blob_bytes, read_slice_bytes,
    render_histogram_png, Metadata, CANCELLED,
};

wasm_bindgen_test_configure!(run_in_browser);

fn blob_from_bytes(bytes: &[u8], mime: &str) -> web_sys::Blob {
    use js_sys::{Array, Uint8Array};
    use web_sys::{Blob, BlobPropertyBag};
    let u8arr = Uint8Array::from(bytes);
    let parts = Array::new();
    parts.push(u8arr.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type(mime);
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag).unwrap()
}

fn no_cancel() -> Rc<Cell<bool>> {
    Rc::new(Cell::new(false))
}

#[wasm_bindgen_test]
fn choose_chunk_size_is_bounded() {
    assert_eq!(choose_chunk_size(1024), 32 * 1024);
    assert_eq!(choose_chunk_size(0), 32 * 1024);
    assert_eq!(choose_chunk_size(4 * 1024 * 1024), 32 * 1024);
    assert_eq!(choose_chunk_size(64 * 1024 * 1024), 512 * 1024);
    // 1 ГБ: верхняя граница клампинга, влезает в usize даже на wasm32.
    let gb1: usize = 1024 * 1024 * 1024;
    assert_eq!(choose_chunk_size(gb1), 8 * 1024 * 1024);
}

#[wasm_bindgen_test]
async fn read_blob_returns_same_bytes() {
    let bytes = b"hello, blob!";
    let blob = blob_from_bytes(bytes, "text/plain");
    let roundtrip = read_blob_bytes(&blob).await.expect("read");
    assert_eq!(roundtrip, bytes);
}

#[wasm_bindgen_test]
async fn read_slice_extracts_subrange() {
    let blob = blob_from_bytes(b"0123456789", "text/plain");
    let mid = read_slice_bytes(&blob, 3.0, 7.0).await.expect("slice");
    assert_eq!(mid, b"3456");
}

#[wasm_bindgen_test]
async fn process_empty_blob_is_safe() {
    let blob = blob_from_bytes(&[], "application/octet-stream");
    let meta = process_blob(&blob, 1024, no_cancel(), |_, _, _| {})
        .await
        .expect("empty");
    assert_eq!(meta.total_bytes, 0);
    assert_eq!(meta.chunks, 0);
    assert_eq!(meta.unique_bytes, 0);
    assert_eq!(meta.min_byte, 0);
}

#[wasm_bindgen_test]
async fn process_counts_bytes_correctly() {
    let bytes: Vec<u8> = (0u16..1024).map(|i| (i & 0xFF) as u8).collect();
    let blob = blob_from_bytes(&bytes, "application/octet-stream");

    let meta = process_blob(&blob, 128, no_cancel(), |_, _, _| {})
        .await
        .expect("process");
    assert_eq!(meta.total_bytes, 1024);
    assert_eq!(meta.chunks, 8);
    assert_eq!(meta.sum, bytes.iter().map(|&b| b as u64).sum::<u64>());
    assert_eq!(meta.min_byte, 0);
    assert_eq!(meta.max_byte, 255);
    assert!(meta.unique_bytes > 0 && meta.unique_bytes <= 256);
    // В файле из 0..255 ровно по 4 вхождения каждого байта (1024/256).
    assert_eq!(meta.histogram[0], 4);
    assert_eq!(meta.histogram[255], 4);
}

#[wasm_bindgen_test]
async fn process_reports_progress() {
    let bytes = vec![0u8; 64 * 1024];
    let blob = blob_from_bytes(&bytes, "application/octet-stream");

    let mut calls: Vec<(usize, usize)> = Vec::new();
    process_blob(&blob, 4096, no_cancel(), |done, total, _hist| {
        calls.push((done, total));
    })
    .await
    .expect("process");

    assert!(calls.len() >= 2, "expected start + end, got {}", calls.len());
    assert_eq!(calls.first(), Some(&(0, 64 * 1024)));
    assert_eq!(calls.last(), Some(&(64 * 1024, 64 * 1024)));
    for (done, total) in &calls {
        assert!(*done <= *total);
        assert_eq!(*total, 64 * 1024);
    }
}

#[wasm_bindgen_test]
async fn process_provides_histogram_in_callback() {
    let bytes = vec![0u8; 512];
    let blob = blob_from_bytes(&bytes, "application/octet-stream");

    let mut final_hist = [0u32; 256];
    process_blob(&blob, 256, no_cancel(), |_, _, hist| {
        final_hist = *hist;
    })
    .await
    .expect("process");

    // Все байты нулевые → histogram[0] == 512.
    assert_eq!(final_hist[0], 512);
}

#[wasm_bindgen_test]
async fn cancel_stops_processing() {
    let bytes = vec![0u8; 1_000_000];
    let blob = blob_from_bytes(&bytes, "application/octet-stream");

    let cancel = Rc::new(Cell::new(false));
    let cancel_for_cb = cancel.clone();
    let mut tick = 0u32;

    let result = process_blob(&blob, 4096, cancel, move |_, _, _| {
        tick += 1;
        if tick == 1 {
            cancel_for_cb.set(true);
        }
    })
    .await;

    assert!(result.is_err());
    let msg = result.unwrap_err().as_string().unwrap_or_default();
    assert_eq!(msg, CANCELLED);
}

#[wasm_bindgen_test]
async fn render_histogram_returns_png_blob() {
    let mut hist = [0u32; 256];
    hist[0] = 10;
    hist[128] = 5;
    hist[255] = 7;
    let blob = render_histogram_png(&hist, 400, 200).await.expect("png");
    assert_eq!(blob.type_(), "image/png");
    assert!(blob.size() > 0.0);
}

#[wasm_bindgen_test]
async fn json_blob_has_expected_content_type() {
    let meta = Metadata::default();
    let blob = make_json_blob(&meta).expect("json blob");
    assert_eq!(blob.type_(), "application/json");
    assert!(blob.size() > 0.0);
}

#[wasm_bindgen_test]
fn metadata_json_is_parseable_shape() {
    let mut meta = Metadata::default();
    meta.total_bytes = 42;
    meta.chunks = 1;
    meta.chunk_size = 64;
    meta.sha256 = String::from("deadbeefcafebabe");
    meta.histogram[0] = 42;

    let text = meta.to_json();
    assert!(text.contains("\"total_bytes\": 42"));
    assert!(text.contains("\"chunks\": 1"));
    assert!(text.contains("\"sha256\": \"deadbeefcafebabe\""));
    assert!(text.contains("\"histogram\": ["));
}
```

---

## 4. `src/app.rs` — полностью автономный демо-пример

```rust
//! Глава 20.4 — Практический проект: Blob → PNG + JSON.
//!
//! Один вход (файл), два выхода (PNG-гистограмма и JSON-метаданные).
//! Демонстрирует:
//! - Blob двусторонний (`arrayBuffer()` / `text()` — обратное чтение);
//! - `Blob.slice()` + yield через `TimeoutFuture::new(0)` — потоковая
//!   обработка с живым прогрессом, ETA и скоростью;
//! - адаптивный размер куска (`choose_chunk_size` — ~128 чанков);
//! - отмену через `Rc<Cell<bool>>`;
//! - оба пути скачивания: `<a download>` и `showSaveFilePicker`.
//!
//! **Файл автономен.** Все зависимости инкапсулированы здесь: утилита
//! `format_str`, чтение Blob, обработка, рисование гистограммы,
//! генерация PNG через `OffscreenCanvas`, скачивание через anchor и
//! Save Picker. Ничего не импортируется из других модулей проекта,
//! кроме `web-sys` и `js-sys`.
//!
//! Wow: загрузите 50 МБ — гистограмма строится в реальном времени, ETA
//! уменьшается, скорость колеблется. «Отменить» останавливает мгновенно.

use std::cell::{Cell, RefCell};
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Blob, BlobPropertyBag, CanvasRenderingContext2d, Document, Element, Event,
    HtmlAnchorElement, HtmlCanvasElement, HtmlElement, HtmlInputElement, Node,
    OffscreenCanvas, OffscreenCanvasRenderingContext2d, Url,
};

// =============================================================================
// Утилита обхода бага rust-analyzer: `format!` в tail-позиции теряет тип
// =============================================================================

/// Возвращает `args` как `String`. Единственная цель — дать rust-analyzer
/// точку, где тип результата жёстко зафиксирован сигнатурой.
#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Feature detection и утилиты JS-ошибок (инлайн из download_paths)
// =============================================================================

fn global_has_function(name: &str) -> bool {
    let global = js_sys::global();
    let Ok(v) = Reflect::get(&global, &JsValue::from_str(name)) else {
        return false;
    };
    v.is_function()
}

fn is_save_picker_supported() -> bool {
    global_has_function("showSaveFilePicker")
}

fn js_err_to_string(e: &JsValue) -> String {
    if e.is_undefined() {
        return String::from("(браузер вернул undefined)");
    }
    if e.is_null() {
        return String::from("(null)");
    }
    if let Some(s) = e.as_string() {
        return s;
    }
    let name = Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string());
    let message = Reflect::get(e, &JsValue::from_str("message"))
        .ok()
        .and_then(|v| v.as_string());
    match (name, message) {
        (Some(n), Some(m)) => {
            let s: String = format!("{n}: {m}");
            s
        }
        (Some(n), None) => n,
        (None, Some(m)) => m,
        (None, None) => {
            let s: String = format!("{e:?}");
            s
        }
    }
}

fn now_ms() -> f64 {
    js_sys::Date::now()
}

// =============================================================================
// Скачивание: anchor и Save Picker (инлайн из download_paths)
// =============================================================================

/// Программный клик по невидимому `<a download href=url>`. URL должен быть жив.
fn trigger_anchor_download(url: &str, filename: &str) -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let anchor = document
        .create_element("a")?
        .dyn_into::<HtmlAnchorElement>()?;
    anchor.set_href(url);
    anchor.set_download(filename);
    anchor.set_attribute("style", "display: none;")?;

    let body = document.body().ok_or_else(|| JsValue::from_str("no body"))?;
    body.append_child(&anchor)?;
    anchor.click();
    body.remove_child(&anchor)?;
    Ok(())
}

/// Достаёт `global[name]` как `Function`.
fn global_get_function(name: &str) -> Option<Function> {
    let global = js_sys::global();
    let f = Reflect::get(&global, &JsValue::from_str(name)).ok()?;
    f.dyn_into::<Function>().ok()
}

/// «Сохранить как…». Возвращает:
/// - `Ok(Some(name))` — файл записан;
/// - `Ok(None)` — пользователь отменил (AbortError);
/// - `Err(msg)` — ошибка или API недоступен.
async fn save_via_picker(
    blob: Blob,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    let Some(picker) = global_get_function("showSaveFilePicker") else {
        return Err(JsValue::from_str(
            "showSaveFilePicker не поддерживается этим браузером (Firefox/Safari)",
        ));
    };

    let window = js_sys::global();
    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;

    let promise: js_sys::Promise = picker.call1(&window, &opts)?.dyn_into()?;

    let handle = match wasm_bindgen_futures::JsFuture::from(promise).await {
        Ok(h) => h,
        Err(e) if is_abort_error(&e) => return Ok(None),
        Err(e) => return Err(clarify_picker_error(e)),
    };

    let writable_promise = call_method(&handle, "createWritable", &[])?;
    let writable = wasm_bindgen_futures::JsFuture::from(writable_promise).await?;

    let write_promise = call_method(&writable, "write", &[blob.as_ref()])?;
    wasm_bindgen_futures::JsFuture::from(write_promise).await?;

    let close_promise = call_method(&writable, "close", &[])?;
    wasm_bindgen_futures::JsFuture::from(close_promise).await?;

    let name = Reflect::get(&handle, &JsValue::from_str("name"))?;
    let name_str: String = name
        .as_string()
        .unwrap_or_else(|| suggested_name.to_string());
    Ok(Some(name_str))
}

fn clarify_picker_error(e: JsValue) -> JsValue {
    if e.is_undefined() || e.is_null() {
        return JsValue::from_str(
            "диалог не открылся: браузер требует user gesture. Кликните кнопку ещё раз \
             и не переключайте окно во время показа диалога",
        );
    }
    e
}

fn call_method(obj: &JsValue, name: &str, args: &[&JsValue]) -> Result<js_sys::Promise, JsValue> {
    let func_js = Reflect::get(obj, &JsValue::from_str(name))?;
    if !func_js.is_function() {
        return Err(JsValue::from_str(&format!("{name} is not a function")));
    }
    let func: Function = func_js.dyn_into()?;
    let args_arr = Array::new();
    for a in args {
        args_arr.push(a);
    }
    let ret = func.apply(obj, &args_arr)?;
    ret.dyn_into::<js_sys::Promise>()
}

fn is_abort_error(e: &JsValue) -> bool {
    match Reflect::get(e, &JsValue::from_str("name")) {
        Ok(v) => v.as_string().as_deref() == Some("AbortError"),
        Err(_) => false,
    }
}

// =============================================================================
// Blob → PNG + JSON (инлайн из blob_processor)
// =============================================================================

/// Маркер отмены — возвращается из `process_blob` как `Err`, когда
/// пользователь нажал «Отменить».
const CANCELLED: &str = "cancelled-by-user";

pub struct ProcessedArtifacts {
    pub png: Blob,
    pub json: Blob,
    pub metadata: Metadata,
}

#[derive(Clone, Debug)]
pub struct Metadata {
    pub total_bytes: usize,
    pub chunks: usize,
    pub chunk_size: usize,
    pub sha256: String,
    pub unique_bytes: usize,
    pub sum: u64,
    pub min_byte: u8,
    pub max_byte: u8,
    pub histogram: [u32; 256],
}

impl Default for Metadata {
    fn default() -> Self {
        Self {
            total_bytes: 0,
            chunks: 0,
            chunk_size: 0,
            sha256: String::new(),
            unique_bytes: 0,
            sum: 0,
            min_byte: 0,
            max_byte: 0,
            histogram: [0; 256],
        }
    }
}

impl Metadata {
    pub fn to_json(&self) -> String {
        let mut hist = String::with_capacity(256 * 8);
        for (i, v) in self.histogram.iter().enumerate() {
            if i > 0 {
                hist.push(',');
            }
            hist.push_str(&v.to_string());
        }
        let s: String = format!(
            "{{\n\
             \"total_bytes\": {},\n\
             \"chunks\": {},\n\
             \"chunk_size\": {},\n\
             \"sha256\": \"{}\",\n\
             \"unique_bytes\": {},\n\
             \"sum\": {},\n\
             \"min_byte\": {},\n\
             \"max_byte\": {},\n\
             \"histogram\": [{}]\n\
             }}\n",
            self.total_bytes,
            self.chunks,
            self.chunk_size,
            self.sha256,
            self.unique_bytes,
            self.sum,
            self.min_byte,
            self.max_byte,
            hist,
        );
        s
    }
}

/// Адаптивный размер куска: цель ~128 чанков.
/// Границы — 32 КБ … 8 МБ.
fn choose_chunk_size(total_bytes: usize) -> usize {
    const TARGET_CHUNKS: usize = 128;
    const MIN_CHUNK: usize = 32 * 1024;
    const MAX_CHUNK: usize = 8 * 1024 * 1024;
    if total_bytes == 0 {
        return MIN_CHUNK;
    }
    let ideal = total_bytes / TARGET_CHUNKS;
    ideal.clamp(MIN_CHUNK, MAX_CHUNK)
}

async fn read_blob_bytes(blob: &Blob) -> Result<Vec<u8>, JsValue> {
    let promise = blob.array_buffer();
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    let buffer: js_sys::ArrayBuffer = value.dyn_into()?;
    let u8arr = Uint8Array::new(&buffer);
    Ok(u8arr.to_vec())
}

async fn read_slice_bytes(blob: &Blob, start: f64, end: f64) -> Result<Vec<u8>, JsValue> {
    let slice = blob.slice_with_f64_and_f64(start, end)?;
    read_blob_bytes(&slice).await
}

/// Обрабатывает Blob кусками.
///
/// **Контракт `on_progress`:** минимум два вызова — `(0, total, _)` в начале
/// и `(total, total, &hist)` в конце. Промежуточные троттлятся до 60 Гц.
///
/// **Контракт `cancel`:** если `cancel.get() == true` на входе в итерацию —
/// возвращает `Err(CANCELLED)`.
async fn process_blob(
    blob: &Blob,
    chunk_size: usize,
    cancel: Rc<Cell<bool>>,
    mut on_progress: impl FnMut(usize, usize, &[u32; 256]),
) -> Result<Metadata, JsValue> {
    let total = blob.size() as usize;
    let mut meta = Metadata {
        total_bytes: total,
        chunk_size,
        min_byte: 255,
        max_byte: 0,
        ..Default::default()
    };

    on_progress(0, total, &meta.histogram);

    let mut hash: u64 = 0xcbf2_9ce4_8422_2325;
    let fnv_prime: u64 = 0x0000_0100_0000_01b3;
    let fnv_mix = |h: u64, b: u8| -> u64 { (h ^ (b as u64)).wrapping_mul(fnv_prime) };

    let mut last_yield_ms = now_ms();
    let mut offset = 0usize;

    while offset < total {
        if cancel.get() {
            return Err(JsValue::from_str(CANCELLED));
        }
        let end = (offset + chunk_size).min(total);
        let bytes = read_slice_bytes(blob, offset as f64, end as f64).await?;

        for &b in &bytes {
            meta.histogram[b as usize] = meta.histogram[b as usize].saturating_add(1);
            meta.sum = meta.sum.wrapping_add(b as u64);
            if b < meta.min_byte {
                meta.min_byte = b;
            }
            if b > meta.max_byte {
                meta.max_byte = b;
            }
            hash = fnv_mix(hash, b);
        }

        meta.chunks += 1;
        offset = end;

        if now_ms() - last_yield_ms >= 16.0 {
            gloo_timers::future::TimeoutFuture::new(0).await;
            last_yield_ms = now_ms();
            on_progress(offset, total, &meta.histogram);
        }
    }

    gloo_timers::future::TimeoutFuture::new(0).await;
    on_progress(total, total, &meta.histogram);

    meta.unique_bytes = meta.histogram.iter().filter(|&&v| v > 0).count();
    if total == 0 {
        meta.min_byte = 0;
    }
    meta.sha256 = format!("{:016x}", hash);
    Ok(meta)
}

/// Общий «скелет» рисования. `CanvasRenderingContext2d` и
/// `OffscreenCanvasRenderingContext2d` — разные типы в web-sys, но их API
/// идентичен (Canvas 2D API). Макрос разворачивает одну и ту же логику
/// в два биндера без дублирования кода.
macro_rules! histogram_draw_body {
    ($ctx:expr, $hist:expr, $w:expr, $h:expr) => {{
        $ctx.set_fill_style_str("#0d1117");
        $ctx.fill_rect(0.0, 0.0, $w, $h);

        $ctx.set_stroke_style_str("#21262d");
        $ctx.set_line_width(1.0);
        $ctx.begin_path();
        $ctx.move_to(0.0, $h - 0.5);
        $ctx.line_to($w, $h - 0.5);
        $ctx.stroke();

        let max = $hist.iter().copied().max().unwrap_or(1).max(1) as f64;
        let bar_w = $w / 256.0;
        $ctx.set_fill_style_str("#58a6ff");
        for (i, &count) in $hist.iter().enumerate() {
            if count == 0 {
                continue;
            }
            let bh = (count as f64 / max) * ($h - 20.0);
            let x = (i as f64) * bar_w;
            let y = $h - 10.0 - bh;
            $ctx.fill_rect(x, y, bar_w.max(1.0), bh);
        }

        $ctx.set_fill_style_str("#8b949e");
        $ctx.set_font("11px monospace");
        $ctx.fill_text("byte histogram (0..255)", 8.0, 14.0)?;
    }};
}

/// Рисует гистограмму на обычном `CanvasRenderingContext2d`
/// (используется для live-канваса в DOM).
fn draw_histogram(
    ctx: &CanvasRenderingContext2d,
    histogram: &[u32; 256],
    width: f64,
    height: f64,
) -> Result<(), JsValue> {
    histogram_draw_body!(ctx, histogram, width, height);
    Ok(())
}

/// Рисует гистограмму на `OffscreenCanvasRenderingContext2d`
/// (используется для PNG-генерации).
fn draw_histogram_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    histogram: &[u32; 256],
    width: f64,
    height: f64,
) -> Result<(), JsValue> {
    histogram_draw_body!(ctx, histogram, width, height);
    Ok(())
}

/// Генерирует PNG-гистограмму через `OffscreenCanvas`.
async fn render_histogram_png(
    histogram: &[u32; 256],
    width: u32,
    height: u32,
) -> Result<Blob, JsValue> {
    let canvas = OffscreenCanvas::new(width, height)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;

    draw_histogram_offscreen(&ctx, histogram, width as f64, height as f64)?;

    let promise = canvas.convert_to_blob()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<Blob>()
}

fn make_json_blob(meta: &Metadata) -> Result<Blob, JsValue> {
    let text = meta.to_json();
    let bytes = Uint8Array::from(text.as_bytes());
    let parts = Array::new();
    parts.push(bytes.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type("application/json");
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag)
}

/// Полный конвейер: обработка + генерация обоих артефактов.
async fn run_pipeline(
    input: &Blob,
    chunk_size: usize,
    cancel: Rc<Cell<bool>>,
    mut on_progress: impl FnMut(usize, usize, &[u32; 256]),
) -> Result<ProcessedArtifacts, JsValue> {
    let metadata = process_blob(input, chunk_size, cancel, &mut on_progress).await?;
    let png = render_histogram_png(&metadata.histogram, 820, 200).await?;
    let json = make_json_blob(&metadata)?;
    Ok(ProcessedArtifacts { png, json, metadata })
}

// =============================================================================
// Демо
// =============================================================================

const LIVE_CANVAS_ID: &str = "bp-live-histogram";
const LIVE_CANVAS_W: f64 = 820.0;
const LIVE_CANVAS_H: f64 = 200.0;
const JSON_PREVIEW_CHARS: usize = 500;

const DESC_LINES: &[&str] = &[
    "Практический проект: пользователь даёт файл — получает PNG-гистограмму \
     и JSON-метаданные. Размер куска чтения подобран <em>автоматически</em>: \
     целевое число чанков ≈ 128, независимо от того, 1 КБ это или 1 ГБ.",
    "Во время обработки <code>Blob.slice(a, b).arrayBuffer()</code> читает куски, \
     а каждые 16 мс цикл отдаёт управление браузеру через \
     <code>TimeoutFuture::new(0)</code> — поэтому гистограмма строится в реальном \
     времени, а ETA и скорость обновляются плавно.",
    "Оба артефакта — независимые Blob. Имена выводятся из имени входа: \
     <code>dataset.csv</code> → <code>dataset.csv.histogram.png</code> \
     и <code>dataset.csv.metadata.json</code>.",
];

struct SelectedFile {
    blob: Blob,
    name: String,
    size: u64,
}

struct Artifact {
    label: &'static str,
    filename: String,
    blob: Blob,
    url: String,
    size: u64,
    mime: String,
    text_preview: Option<String>,
}

struct DemoState {
    artifacts: Vec<Artifact>,
    last_meta_summary: Option<String>,
}

impl DemoState {
    fn new() -> Self {
        Self {
            artifacts: Vec::new(),
            last_meta_summary: None,
        }
    }
    fn clear(&mut self) {
        for a in self.artifacts.drain(..) {
            let _ = Url::revoke_object_url(&a.url);
        }
        self.last_meta_summary = None;
    }
}

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("20.4 — Практический проект: Blob → PNG + JSON"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Панель управления
    let input_panel = document.create_element("div")?;
    input_panel.set_class_name("bp-block");
    input_panel.set_inner_html(
        r#"<div class="bp-input-row">
               <input type="file" id="bp-input-file" class="bp-file-input"
                      accept=".png,.jpg,.jpeg,.gif,.webp,.bmp,.tiff,.svg,\
                              .pdf,.zip,.gz,.tar,.7z,\
                              .wasm,.bin,.dat,.exe,\
                              .txt,.csv,.tsv,.json,.xml,.yaml,.yml,.md,.log">
               <label for="bp-input-file" class="bp-file-label">
                   <span class="bp-file-icon">📄</span>
                   <span id="bp-file-name">Выбрать файл…</span>
               </label>
               <button class="bp-btn bp-btn-primary" id="bp-run" disabled>Обработать</button>
               <button class="bp-btn bp-btn-danger" id="bp-cancel" disabled>Отменить</button>
           </div>
           <p class="bp-input-hint">
               Фильтр в диалоге — рекомендация, а не ограничение: подойдёт любой файл.
               Самую выразительную гистограмму дают изображения, архивы и исполняемые
               файлы — в них байты распределены по всему диапазону 0…255. Текстовые
               файлы дают более узкую картину, но тоже считаются корректно.
           </p>
           <div class="bp-progress-wrap">
               <div class="bp-progress">
                   <div class="bp-progress-fill idle" id="bp-progress-fill" style="width:0%"></div>
               </div>
               <div class="bp-progress-label" id="bp-progress-label">ожидание файла</div>
           </div>
           <canvas id="bp-live-histogram" class="bp-live-histogram"
                   width="820" height="200"></canvas>"#,
    );
    root.append_child(&input_panel)?;

    // Результаты
    let results_panel = document.create_element("div")?;
    results_panel.set_class_name("bp-block");
    results_panel.set_attribute("id", "bp-results")?;
    results_panel.set_inner_html(
        r#"<h3>Результаты</h3>
           <div class="bp-cards" id="bp-cards">
               <div class="bp-empty" id="bp-empty">Пока ничего не обработано.</div>
           </div>"#,
    );
    root.append_child(&results_panel)?;

    // Footnote
    let footnote = document.create_element("p")?;
    footnote.set_class_name("bp-footnote");
    footnote.set_text_content(Some(
        "Решение построено так, чтобы пользователь не принимал технических решений. \
         Размер куска чтения определяется автоматически, оставшееся время оценивается \
         на лету, имена выходных файлов наследуются от входного. Отмена опирается на \
         общий флаг Rc<Cell<bool>>: значение, выставленное обработчиком кнопки, \
         немедленно становится видимым в цикле обработки — Cell обеспечивает \
         внутреннюю изменяемость без конфликтов заимствования.",
    ));
    root.append_child(&footnote)?;

    // Состояние
    let state = Rc::new(RefCell::new(DemoState::new()));
    let selected: Rc<RefCell<Option<SelectedFile>>> = Rc::new(RefCell::new(None));
    let cancel_flag: Rc<Cell<bool>> = Rc::new(Cell::new(false));

    // Живой canvas — один контекст, переиспользуется в callback'е
    let live_canvas = get_canvas(&document, LIVE_CANVAS_ID)?;
    let live_ctx: CanvasRenderingContext2d = live_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx"))?
        .dyn_into()?;
    let _ = draw_histogram(&live_ctx, &[0u32; 256], LIVE_CANVAS_W, LIVE_CANVAS_H);

    wire_file_input(&document, selected.clone())?;
    wire_run_button(
        &document,
        selected.clone(),
        state.clone(),
        cancel_flag.clone(),
        live_ctx,
    )?;
    wire_cancel_button(&document, cancel_flag)?;
    wire_cards_delegation(&document, state)?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_file_input(
    document: &Document,
    selected: Rc<RefCell<Option<SelectedFile>>>,
) -> Result<(), JsValue> {
    let input = get_input(document, "bp-input-file")?;
    let doc = document.clone();

    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(input) = doc.get_element_by_id("bp-input-file") else {
            return;
        };
        let Ok(input) = input.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Some(files) = input.files() else { return };
        let Some(file) = files.get(0) else { return };

        let name = file.name();
        let size = file.size() as u64;
        let blob: Blob = file.into();

        let label = format_str(format_args!("{} ({})", name, human_bytes(size as usize)));
        if let Some(el) = doc.get_element_by_id("bp-file-name") {
            el.set_text_content(Some(&label));
        }
        *selected.borrow_mut() = Some(SelectedFile { blob, name, size });

        set_progress(
            &doc,
            ProgressState::Idle,
            0.0,
            "файл выбран — нажмите «Обработать»",
        );
        set_btn_enabled(&doc, "bp-run", true);
        set_btn_enabled(&doc, "bp-cancel", false);
    });
    input.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_run_button(
    document: &Document,
    selected: Rc<RefCell<Option<SelectedFile>>>,
    state: Rc<RefCell<DemoState>>,
    cancel_flag: Rc<Cell<bool>>,
    live_ctx: CanvasRenderingContext2d,
) -> Result<(), JsValue> {
    let btn = get_button(document, "bp-run")?;
    let doc = document.clone();

    wire_click(&btn, move || {
        let doc = doc.clone();
        let state = state.clone();
        let cancel = cancel_flag.clone();
        let live_ctx = live_ctx.clone();

        // Сброс флага отмены перед новым запуском — идемпотентность.
        cancel.set(false);

        let Some(sel) = selected.borrow_mut().take() else { return };
        let SelectedFile { blob, name, size } = sel;
        // Возвращаем, чтобы можно было обработать повторно.
        *selected.borrow_mut() = Some(SelectedFile {
            blob: blob.clone(),
            name: name.clone(),
            size,
        });

        let chunk_size = choose_chunk_size(size as usize);
        let chunk_human = human_bytes(chunk_size);
        let est_chunks = if chunk_size == 0 {
            0
        } else {
            (size as usize).div_ceil(chunk_size)
        };

        set_progress(
            &doc,
            ProgressState::Working,
            0.0,
            &format_str(format_args!(
                "подготовка · чанк {chunk_human} · ~{est_chunks} шт"
            )),
        );
        set_btn_enabled(&doc, "bp-run", false);
        set_btn_enabled(&doc, "bp-cancel", true);

        let doc_outer = doc.clone();
        let state_outer = state.clone();
        let cancel_outer = cancel.clone();
        let name_for_run = name.clone();

        wasm_bindgen_futures::spawn_local(async move {
            let t0 = now_ms();

            let doc_cb = doc_outer.clone();
            let cancel_cb = cancel_outer.clone();
            let live_ctx_cb = live_ctx.clone();
            let progress = move |done: usize, total: usize, hist: &[u32; 256]| {
                let _ = draw_histogram(&live_ctx_cb, hist, LIVE_CANVAS_W, LIVE_CANVAS_H);

                if total == 0 {
                    set_progress(&doc_cb, ProgressState::Working, 100.0, "пустой файл");
                    return;
                }
                let pct = (done as f64 / total as f64) * 100.0;
                let elapsed_ms = now_ms() - t0;
                let elapsed_s = elapsed_ms / 1000.0;
                let speed_bps = if elapsed_s > 0.0 {
                    done as f64 / elapsed_s
                } else {
                    0.0
                };
                let eta_s = if speed_bps > 0.0 {
                    (total - done) as f64 / speed_bps
                } else {
                    0.0
                };
                let msg = format_str(format_args!(
                    "{:.0}% · {} из {} · {}/с · ~{}",
                    pct,
                    human_bytes(done),
                    human_bytes(total),
                    human_bytes(speed_bps as usize),
                    human_duration(eta_s),
                ));
                let _ = &cancel_cb;
                set_progress(&doc_cb, ProgressState::Working, pct, &msg);
            };

            let result = run_pipeline(&blob, chunk_size, cancel_outer, progress).await;
            let elapsed_ms = now_ms() - t0;

            match result {
                Ok(art) => {
                    let json_text = read_text_from_blob(&art.json).await.unwrap_or_default();
                    let json_preview = truncate_string(&json_text, JSON_PREVIEW_CHARS);

                    {
                        let mut st = state_outer.borrow_mut();
                        st.clear();

                        let png_url = match Url::create_object_url_with_blob(&art.png) {
                            Ok(u) => u,
                            Err(e) => {
                                let m = js_err_to_string(&e);
                                set_progress(&doc_outer, ProgressState::Error, 0.0, &m);
                                set_btn_enabled(&doc_outer, "bp-run", true);
                                set_btn_enabled(&doc_outer, "bp-cancel", false);
                                return;
                            }
                        };
                        st.artifacts.push(Artifact {
                            label: "PNG-гистограмма",
                            filename: derived_name(&name_for_run, "histogram", "png"),
                            blob: art.png.clone(),
                            url: png_url,
                            size: art.png.size() as u64,
                            mime: art.png.type_(),
                            text_preview: None,
                        });

                        let json_url = match Url::create_object_url_with_blob(&art.json) {
                            Ok(u) => u,
                            Err(e) => {
                                let m = js_err_to_string(&e);
                                set_progress(&doc_outer, ProgressState::Error, 0.0, &m);
                                set_btn_enabled(&doc_outer, "bp-run", true);
                                set_btn_enabled(&doc_outer, "bp-cancel", false);
                                return;
                            }
                        };
                        st.artifacts.push(Artifact {
                            label: "JSON-метаданные",
                            filename: derived_name(&name_for_run, "metadata", "json"),
                            blob: art.json.clone(),
                            url: json_url,
                            size: art.json.size() as u64,
                            mime: art.json.type_(),
                            text_preview: Some(json_preview),
                        });

                        st.last_meta_summary = Some(format_str(format_args!(
                            "{} байт · {} чанков по {} · unique={} · sum={} · hash={}",
                            art.metadata.total_bytes,
                            art.metadata.chunks,
                            human_bytes(art.metadata.chunk_size),
                            art.metadata.unique_bytes,
                            art.metadata.sum,
                            art.metadata.sha256,
                        )));
                    }

                    let done_msg = format_str(format_args!("готово за {:.0} мс", elapsed_ms));
                    set_progress(&doc_outer, ProgressState::Done, 100.0, &done_msg);
                    set_btn_enabled(&doc_outer, "bp-run", true);
                    set_btn_enabled(&doc_outer, "bp-cancel", false);
                    render_cards(&doc_outer, &state_outer.borrow());

                    if let Some(res) = doc_outer.get_element_by_id("bp-results") {
                        res.scroll_into_view();
                    }
                }
                Err(e) => {
                    let msg = js_err_to_string(&e);
                    if msg == CANCELLED {
                        set_progress(
                            &doc_outer,
                            ProgressState::Cancelled,
                            0.0,
                            "отменено пользователем",
                        );
                    } else {
                        let text = format_str(format_args!("ошибка: {msg}"));
                        set_progress(&doc_outer, ProgressState::Error, 0.0, &text);
                    }
                    set_btn_enabled(&doc_outer, "bp-run", true);
                    set_btn_enabled(&doc_outer, "bp-cancel", false);
                }
            }
        });
    })
}

fn wire_cancel_button(
    document: &Document,
    cancel_flag: Rc<Cell<bool>>,
) -> Result<(), JsValue> {
    let btn = get_button(document, "bp-cancel")?;
    let doc = document.clone();
    wire_click(&btn, move || {
        cancel_flag.set(true);
        set_progress(&doc, ProgressState::Cancelling, 0.0, "…отмена");
    })
}

fn wire_cards_delegation(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let container = document
        .get_element_by_id("bp-cards")
        .ok_or_else(|| JsValue::from_str("no #bp-cards"))?;
    let doc = document.clone();

    let closure = Closure::<dyn FnMut(Event)>::new(move |ev: Event| {
        let Some(target) = ev.target() else { return };
        let Ok(el) = target.dyn_into::<Element>() else { return };
        let Ok(Some(btn)) = el.closest("button[data-action]") else { return };
        let Some(action) = btn.get_attribute("data-action") else { return };
        let Some(idx_str) = btn.get_attribute("data-index") else { return };
        let Ok(idx) = idx_str.parse::<usize>() else { return };

        let (filename, blob, url) = {
            let st = state.borrow();
            let Some(a) = st.artifacts.get(idx) else { return };
            (a.filename.clone(), a.blob.clone(), a.url.clone())
        };

        match action.as_str() {
            "anchor" => {
                let _ = trigger_anchor_download(&url, &filename);
            }
            "picker" => {
                let doc_inner = doc.clone();
                let fname = filename.clone();
                wasm_bindgen_futures::spawn_local(async move {
                    match save_via_picker(blob, &fname).await {
                        Ok(Some(name)) => {
                            let m = format_str(format_args!("сохранено как «{name}»"));
                            set_progress(&doc_inner, ProgressState::Done, 100.0, &m);
                        }
                        Ok(None) => set_progress(
                            &doc_inner,
                            ProgressState::Cancelled,
                            100.0,
                            "отменено пользователем",
                        ),
                        Err(e) => {
                            let m = js_err_to_string(&e);
                            set_progress(&doc_inner, ProgressState::Error, 100.0, &m);
                        }
                    }
                });
            }
            _ => {}
        }
    });

    container.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- UI helpers ----------

#[derive(Clone, Copy)]
enum ProgressState {
    Idle,
    Working,
    Cancelling,
    Cancelled,
    Done,
    Error,
}

impl ProgressState {
    fn class(&self) -> &'static str {
        match self {
            ProgressState::Idle => "bp-progress-fill idle",
            ProgressState::Working => "bp-progress-fill working",
            ProgressState::Cancelling => "bp-progress-fill cancelling",
            ProgressState::Cancelled => "bp-progress-fill cancelled",
            ProgressState::Done => "bp-progress-fill done",
            ProgressState::Error => "bp-progress-fill error",
        }
    }
}

fn set_progress(document: &Document, state: ProgressState, pct: f64, label: &str) {
    if let Some(fill) = document.get_element_by_id("bp-progress-fill") {
        if let Some(el) = fill.dyn_ref::<HtmlElement>() {
            let style = format_str(format_args!("width:{}%", pct.clamp(0.0, 100.0)));
            el.set_attribute("style", &style).ok();
            el.set_class_name(state.class());
        }
    }
    if let Some(el) = document.get_element_by_id("bp-progress-label") {
        el.set_text_content(Some(label));
    }
}

fn set_btn_enabled(document: &Document, id: &str, enabled: bool) {
    let Some(btn) = document.get_element_by_id(id) else { return };
    let Ok(btn) = btn.dyn_into::<HtmlElement>() else { return };
    if enabled {
        let _ = btn.remove_attribute("disabled");
    } else {
        let _ = btn.set_attribute("disabled", "");
    }
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("bp-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("bp-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

fn get_input(document: &Document, id: &str) -> Result<HtmlInputElement, JsValue> {
    document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no input #{id}")))?
        .dyn_into::<HtmlInputElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} is not input")))
}

fn get_button(document: &Document, id: &str) -> Result<HtmlElement, JsValue> {
    document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no button #{id}")))?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} is not button")))
}

fn get_canvas(document: &Document, id: &str) -> Result<HtmlCanvasElement, JsValue> {
    document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no canvas #{id}")))?
        .dyn_into::<HtmlCanvasElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} is not canvas")))
}

fn wire_click<F>(btn: &HtmlElement, f: F) -> Result<(), JsValue>
where
    F: FnMut() + 'static,
{
    let closure = Closure::<dyn FnMut()>::new(f);
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Рендер ----------

fn render_cards(document: &Document, state: &DemoState) {
    let Some(container) = document.get_element_by_id("bp-cards") else { return };
    if state.artifacts.is_empty() {
        container.set_inner_html(
            r#"<div class="bp-empty">Пока ничего не обработано.</div>"#,
        );
        return;
    }
    let mut html = String::new();
    for (i, a) in state.artifacts.iter().enumerate() {
        html.push_str(&render_card(i, a));
    }
    if let Some(sum) = &state.last_meta_summary {
        let esc = html_escape(sum);
        html.push_str(&format_str(format_args!(
            r#"<div class="bp-summary">{esc}</div>"#
        )));
    }
    container.set_inner_html(&html);
}

fn render_card(index: usize, a: &Artifact) -> String {
    let url_esc = html_escape(&a.url);
    let name_esc = html_escape(&a.filename);
    let mime_esc = html_escape(&a.mime);
    let label_esc = html_escape(a.label);
    let size_str = human_bytes(a.size as usize);

    let preview = render_preview(a, &url_esc, &name_esc);

    let picker_btn: String = if is_save_picker_supported() {
        format_str(format_args!(
            r#"<button class="bp-card-btn bp-card-btn-secondary" data-action="picker" data-index="{index}">Сохранить как…</button>"#
        ))
    } else {
        String::from(
            r#"<span class="bp-card-hint">showSaveFilePicker недоступен в этом браузере</span>"#,
        )
    };

    format_str(format_args!(
        r#"<div class="bp-card">
            <div class="bp-card-head">
                <div class="bp-card-title">{label_esc}</div>
                <div class="bp-card-meta">{name_esc} · {size_str} · <code>{mime_esc}</code></div>
            </div>
            {preview}
            <div class="bp-card-actions">
                <button class="bp-card-btn bp-card-btn-primary" data-action="anchor" data-index="{index}">Скачать</button>
                {picker_btn}
            </div>
           </div>"#
    ))
}

fn render_preview(a: &Artifact, url_esc: &str, name_esc: &str) -> String {
    if a.mime.starts_with("image/") {
        return format_str(format_args!(
            r#"<img class="bp-card-preview" src="{url_esc}" alt="{name_esc}">"#
        ));
    }
    if let Some(text) = &a.text_preview {
        let esc = html_escape(text);
        return format_str(format_args!(
            r#"<pre class="bp-card-preview bp-card-preview-json">{esc}</pre>"#
        ));
    }
    String::from(r#"<div class="bp-card-preview bp-card-preview-text">—</div>"#)
}

// ---------- Утилиты ----------

async fn read_text_from_blob(blob: &Blob) -> Result<String, JsValue> {
    let promise = blob.text();
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value
        .as_string()
        .ok_or_else(|| JsValue::from_str("blob.text() did not return a string"))
}

fn truncate_string(s: &str, max: usize) -> String {
    let chars: Vec<char> = s.chars().collect();
    if chars.len() <= max {
        return s.to_string();
    }
    let head: String = chars[..max].iter().collect();
    format_str(format_args!("{head}…"))
}

/// Имя выходного файла на основе имени входа:
/// `dataset.csv` + `histogram` + `png` → `dataset.csv.histogram.png`.
fn derived_name(input: &str, kind: &str, ext: &str) -> String {
    format_str(format_args!("{input}.{kind}.{ext}"))
}

fn human_bytes(n: usize) -> String {
    let b = n as f64;
    let s: String = if b < 1024.0 {
        format_str(format_args!("{:.0} Б", b))
    } else if b < 1024.0 * 1024.0 {
        format_str(format_args!("{:.1} КБ", b / 1024.0))
    } else if b < 1024.0 * 1024.0 * 1024.0 {
        format_str(format_args!("{:.2} МБ", b / (1024.0 * 1024.0)))
    } else {
        format_str(format_args!("{:.2} ГБ", b / (1024.0 * 1024.0 * 1024.0)))
    };
    s
}

fn human_duration(seconds: f64) -> String {
    let s: String = if seconds < 1.0 {
        String::from("<1с")
    } else if seconds < 60.0 {
        format_str(format_args!("{:.0}с", seconds))
    } else if seconds < 3600.0 {
        format_str(format_args!(
            "{:.0}м{:.0}с",
            (seconds / 60.0).floor(),
            seconds % 60.0
        ))
    } else {
        format_str(format_args!(
            "{:.0}ч{:.0}м",
            (seconds / 3600.0).floor(),
            ((seconds % 3600.0) / 60.0).floor()
        ))
    };
    s
}

fn html_escape(s: &str) -> String {
    let mut out = String::with_capacity(s.len());
    for c in s.chars() {
        match c {
            '&' => out.push_str("&amp;"),
            '<' => out.push_str("&lt;"),
            '>' => out.push_str("&gt;"),
            '"' => out.push_str("&quot;"),
            '\'' => out.push_str("&#39;"),
            _ => out.push(c),
        }
    }
    out
}
```

---

## 5. CSS — дополнение к `web/styles/main.css`

```css
/* ---------- 20.4 Blob processor ---------- */
.bp-desc {
    max-width: 900px;
    margin: 0 0 20px 0;
    padding: 12px 16px;
    background: #161b22;
    border-left: 3px solid #58a6ff;
    border-radius: 4px;
    color: #c9d1d9;
    font-size: 14px;
    line-height: 1.55;
}
.bp-desc-line { margin: 6px 0; }
.bp-block { margin-bottom: 24px; }
.bp-block h3 { color: #c9d1d9; margin: 20px 0 12px; }

.bp-input-row {
    display: flex;
    gap: 12px;
    align-items: center;
    margin-bottom: 8px;
    flex-wrap: wrap;
}
.bp-file-input { display: none; }
.bp-file-label {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 14px;
    background: #161b22;
    color: #c9d1d9;
    border: 1px dashed #30363d;
    border-radius: 6px;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;
    font-size: 13px;
    font-family: ui-monospace, monospace;
}
.bp-file-label:hover {
    border-color: #58a6ff;
    background: #1c2128;
}
.bp-file-icon { font-size: 18px; }

.bp-input-hint {
    max-width: 820px;
    margin: 4px 0 16px;
    color: #8b949e;
    font-size: 12px;
    line-height: 1.55;
}

.bp-btn {
    padding: 8px 18px;
    border-radius: 6px;
    border: 1px solid #30363d;
    background: #21262d;
    color: #c9d1d9;
    cursor: pointer;
    font-size: 13px;
    font-weight: 500;
    transition: background 0.15s ease;
}
.bp-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.bp-btn-primary:hover:not([disabled]) { background: #2ea043; }
.bp-btn[disabled] { opacity: 0.5; cursor: not-allowed; }
.bp-btn-danger {
    background: #21262d;
    border-color: #30363d;
    color: #f85149;
}
.bp-btn-danger:hover:not([disabled]) {
    background: #30363d;
    border-color: #f85149;
}
.bp-btn-danger[disabled] { opacity: 0.4; cursor: not-allowed; }

.bp-progress-wrap { display: flex; flex-direction: column; gap: 6px; }
.bp-progress {
    height: 6px;
    background: #21262d;
    border-radius: 3px;
    overflow: hidden;
}
.bp-progress-fill { transition: width 0.08s linear, background 0.2s ease; }
.bp-progress-fill.idle       { background: #21262d; }
.bp-progress-fill.working    { background: linear-gradient(90deg, #58a6ff 0%, #3fb950 100%); }
.bp-progress-fill.cancelling { background: #d29922; }
.bp-progress-fill.cancelled  { background: #d29922; }
.bp-progress-fill.done       { background: #3fb950; }
.bp-progress-fill.error      { background: #f85149; }
.bp-progress-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
}

.bp-live-histogram {
    display: block;
    width: 100%;
    max-width: 820px;
    height: 200px;
    margin-top: 12px;
    border: 1px solid #21262d;
    border-radius: 6px;
    background: #0d1117;
}

.bp-cards {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
    gap: 16px;
}
.bp-empty {
    padding: 24px;
    text-align: center;
    color: #6e7681;
    font-size: 13px;
    border: 1px dashed #30363d;
    border-radius: 8px;
}
.bp-card {
    background: linear-gradient(180deg, #161b22 0%, #0d1117 100%);
    border: 1px solid #30363d;
    border-radius: 10px;
    padding: 16px;
    display: flex;
    flex-direction: column;
    transition: transform 0.15s ease, box-shadow 0.15s ease, border-color 0.15s ease;
}
.bp-card:hover {
    transform: translateY(-3px);
    border-color: #58a6ff;
    box-shadow: 0 8px 24px rgba(88, 166, 255, 0.12);
}
.bp-card-head { margin-bottom: 12px; }
.bp-card-title {
    font-size: 15px;
    font-weight: 600;
    color: #c9d1d9;
    margin-bottom: 4px;
}
.bp-card-meta {
    font-family: ui-monospace, monospace;
    font-size: 11px;
    color: #8b949e;
}
.bp-card-meta code { color: #79c0ff; }
.bp-card-preview {
    width: 100%;
    height: 180px;
    object-fit: contain;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 6px;
    margin-bottom: 12px;
}
.bp-card-preview-json {
    font-family: ui-monospace, monospace;
    font-size: 11px;
    line-height: 1.4;
    color: #c9d1d9;
    padding: 10px 12px;
    white-space: pre;
    overflow: hidden;
    max-height: 180px;
    height: auto;
}
.bp-card-preview-text {
    display: flex;
    align-items: center;
    justify-content: center;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    text-align: center;
    padding: 12px;
}
.bp-card-actions {
    display: flex;
    gap: 8px;
    margin-top: auto;
    flex-wrap: wrap;
}
.bp-card-btn {
    flex: 1;
    padding: 8px 12px;
    border-radius: 6px;
    border: 1px solid #30363d;
    background: #21262d;
    color: #c9d1d9;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    transition: background 0.15s ease;
    white-space: nowrap;
}
.bp-card-btn:active { transform: translateY(1px); }
.bp-card-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.bp-card-btn-primary:hover { background: #2ea043; }
.bp-card-btn-secondary:hover { background: #30363d; }
.bp-card-hint {
    font-size: 11px;
    color: #6e7681;
    font-style: italic;
    padding: 8px 0;
}

.bp-summary {
    grid-column: 1 / -1;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    padding: 12px;
    background: #161b22;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
}

.bp-footnote {
    padding: 10px 14px;
    background: #161b22;
    border-left: 3px solid #f0883e;
    border-radius: 4px;
    color: #8b949e;
    font-size: 12px;
    line-height: 1.55;
    margin-top: 20px;
}
```

---

## 6. Команды запуска

```bash
# 1. Сборка wasm
npm run wasm:dev

# 2. Только тест главы 20.4
wasm-pack test --headless --chrome --test wasm_blob_processor

# 3. Полный набор тестов
npm test

# 4. Демо
npm start
```

## Требования к `Cargo.toml`

Убедитесь, что в `[dependencies.web-sys] features` присутствуют:

```toml
"CanvasRenderingContext2d",
"HtmlCanvasElement",
"OffscreenCanvas",
"OffscreenCanvasRenderingContext2d",
```

Все четыре используются в коде. `OffscreenCanvas::convert_to_blob` — Promise-версия, работает без callback-обёрток.

---

## Итог

Глава 20.4 состоит из пяти артефактов:

| Файл | Назначение |
|---|---|
| `src/blob_processor.rs` | Модуль: обработка Blob, гистограмма, PNG, JSON. Для тестов и внешнего использования. |
| `src/lib.rs` | Одна строка `pub mod blob_processor;`. |
| `tests/wasm_blob_processor.rs` | 11 тестов: адаптивный размер, чтение, пустой файл, статистика, прогресс, отмена, PNG, JSON. |
| `src/app.rs` | Автономный демо-пример. Всё инлайн: `format_str`, скачивание, обработка, рисование. |
| CSS | Стили `.bp-*` — описание, прогресс, карточки, гистограмма. |

Демо работает в браузере: пользователь выбирает файл (с фильтром по типам), видит живой прогресс с процентами, скоростью и ETA, наблюдает, как гистограмма строится в реальном времени, может отменить операцию в любой момент. На выходе — два независимых артефакта с автогенерированными именами и двумя путями скачивания.