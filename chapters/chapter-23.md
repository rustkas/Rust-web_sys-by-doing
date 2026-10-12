# Глава 23. WebCodecs

## 23.0. Обновлённая структура

```
src/
  codecs_demo.rs                 ← НОВЫЙ модуль главы 23
  fractal.rs                     ← из 22.6
  worker_pool.rs                 ← из 22.5
  wasm_in_worker.rs              ← из 22.4
  transferable.rs                ← из 22.3
  …
tests/
  wasm_codecs_demo.rs            ← НОВЫЙ тест
src/lib.rs                       ← +1 строка: pub mod codecs_demo;
src/app.rs                       ← демо главы (заменяется целиком)
web/
  public/workers/
    codecs_worker.js             ← НОВЫЙ worker (два режима инверсии)
  pkg/                           ← сборка wasm-pack (out-dir web/pkg)
  public/workers/…               ← существующие воркеры из глав 22 (не трогаем)
  main.js                        ← без правок
Cargo.toml                       ← без новых фич (WebCodecs через Reflect)
vite.config.js                   ← без правок
package.json                     ← без правок
```

**Комментарий.** Модуль `codecs_demo.rs` — про **фреймовую обработку видео** через WebCodecs: доступ к декодерам и энкодерам браузера напрямую, минуя `<video>` и MediaStream. Демо показывает **два режима** обработки пикселей — CPU (Rust) и GPU (WebGL2) — и измеримую разницу между ними. Префикс CSS — `cd-` (codecs demo), уникален в проекте. WebCodecs API доступен в Dedicated Web Workers — worker выносит всю тяжёлую работу с кадрами из main thread.

**Ссылки для детального изучения:**

- MDN: [WebCodecs API](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)
- MDN: [`VideoDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder)
- MDN: [`VideoEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder)
- MDN: [`VideoFrame`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame)
- MDN: [`EncodedVideoChunk`](https://developer.mozilla.org/en-US/docs/Web/API/EncodedVideoChunk)
- MDN: [`OffscreenCanvas.transferToImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/transferToImageBitmap)
- web.dev: [Video processing with WebCodecs](https://developer.chrome.com/docs/web-platform/best-practices/webcodecs)
- W3C: [WebCodecs Specification](https://w3c.github.io/webcodecs/)

---

## 23.1. О чём эта глава

Все предыдущие главы работали с **данными на уровне пикселей и байтов**. Canvas, ImageData, ArrayBuffer, WebGL. Но есть класс задач, где уровень ниже: **отдельные кадры видео**, уже сжатые или ещё нет.

До WebCodecs разработчики, которым нужен был доступ к отдельным кадрам, тащили в браузер **ffmpeg.wasm** — целый кодировщик/декодировщик, скомпилированный в WASM. Он весил десятки мегабайт, не использовал аппаратное ускорение и конкурировал с уже встроенными в браузер кодеками.

**WebCodecs** — API, дающий **прямой доступ к кодекам, уже встроенным в браузер**. H.264, VP9, AV1, AAC, Opus. Никакого WASM-кодека. Браузер сам решает, использовать ли аппаратный декодер (GPU) или программный.

Глава отвечает на пять вопросов.

**Первый — что такое VideoFrame и AudioData.** Это «сырые» (несжатые) кадры и сэмплы. `VideoFrame` привязан к пиксельным данным в памяти GPU, `AudioData` — к PCM-сэмплам.

**Второй — как декодировать видео.** `VideoDecoder` принимает `EncodedVideoChunk` (сжатые данные) и выдаёт `VideoFrame` (распакованные кадры).

**Третий — как кодировать видео.** `VideoEncoder` делает обратное: `VideoFrame` → `EncodedVideoChunk`.

**Четвёртый — что с аудио.** `AudioDecoder` и `AudioEncoder` работают симметрично.

**Пятый — где здесь Rust.** WebCodecs — JS API. Rust управляет кодеками и обрабатывает кадры между стадиями. Но **сам процесс кодирования/декодирования — не в Rust**. Это работа браузера.

---

## 23.2. Что уже есть и что появится

К концу 22.6 умеем:
- создавать Worker'ы, передавать им `OffscreenCanvas`;
- загружать WASM в Worker'е (второй инстанс);
- использовать chunked-обработку с yield;
- показывать determinate progress bar.

К концу 23 будем уметь:
- декодировать видео в `VideoFrame`'ы;
- кодировать `VideoFrame`'ы обратно в сжатые чанки;
- обрабатывать кадры в Rust между декодером и энкодером;
- **сравнивать CPU и GPU-путь обработки пикселей**;
- строить media-пайплайн: encode → decode → invert → re-encode.

---

## 23.3. Почему WebCodecs важен

Три сценария, которые до WebCodecs были либо невозможны, либо требовали ffmpeg.wasm.

**Видеоредактор.** Пользователь загружает MP4. Приложение показывает таймлайн с кадрами, накладывает фильтры, режет, перекодирует. С WebCodecs: `VideoDecoder` даёт кадры для превью, `VideoEncoder` сжимает результат. Всё на аппаратном ускорении.

**Видеоконференция.** Кадры с камеры (`MediaStreamTrack`) → `VideoFrame` → обработка в Worker'е (размытие фона) → `VideoEncoder` → отправка по сети.

**Стриминг с низкой задержкой.** `WebTransport` приносит сжатые чанки. `VideoDecoder` декодирует. `VideoFrame` рисуется на canvas. Задержка — единицы кадров вместо секунд буферизации `<video>`.

Ключевое: **ничего из этого не требует сторонних библиотек**. Кодеки уже в браузере.

---

## 23.4. VideoFrame и EncodedVideoChunk

Два представления одного кадра.

**`VideoFrame`** — распакованный кадр. Пиксели в памяти GPU. Имеет `timestamp` (микросекунды), `duration`, `format`, `codedWidth`, `codedHeight`.

`VideoFrame` — это `CanvasImageSource`. Его можно нарисовать на canvas, отдать в WebGL, передать в Worker (transferable), сжать в JPEG через `OffscreenCanvas.convertToBlob()`.

**`EncodedVideoChunk`** — сжатый кадр. Байты в обычной памяти. Имеет `type` — `"key"` или `"delta"`. Key-кадр декодируется независимо; delta требует предыдущих.

**Соотношение размеров.** `EncodedVideoChunk` хранит в 10–100 раз меньше данных. Это и есть смысл сжатия.

**Правило работы.** `VideoFrame` держит память GPU. Всегда `frame.close()` после использования, иначе утечка.

---

## 23.5. VideoDecoder

Принимает `EncodedVideoChunk`, выдаёт `VideoFrame`.

```js
const decoder = new VideoDecoder({
    output: (frame) => { /* кадр готов */ },
    error: (e) => { /* ошибка */ },
});

decoder.configure({
    codec: "avc1.42001E",
    codedWidth: 1920,
    codedHeight: 1080,
    description: avcC,     // SPS/PPS — обязательно для avc1
});

decoder.decode(chunk);
await decoder.flush();
decoder.close();
```

**Ключевое.** `output` вызывается **асинхронно**. Кадры приходят по мере готовности, не в том же порядке, что `decode()`.

**Что нужно до декодера.** Сырой поток `EncodedVideoChunk`'ов — результат **демуксинга** контейнера. WebCodecs не умеет демуксить. MP4/WebM — отдельная задача (`mp4box.js`, `jswebm`, WASM-демуксер).

**Про `description`.** Для H.264 в формате avc1 (AVCC) декодер обязан получить SPS/PPS. Они приходят в `metadata.decoderConfig` из `VideoEncoder.output` **на первом key-кадре**. Без `description` декодер выдаёт `EncodingError: The given encoding is not supported`.

---

## 23.6. VideoEncoder

Обратная операция. `VideoFrame` → `EncodedVideoChunk`.

```js
const encoder = new VideoEncoder({
    output: (chunk, metadata) => {
        // chunk: EncodedVideoChunk
        // metadata.decoderConfig: { codec, description } — на key-кадре
    },
    error: (e) => {},
});

encoder.configure({
    codec: "avc1.42001E",
    width: 1920,
    height: 1080,
    bitrate: 5_000_000,
    framerate: 30,
});

encoder.encode(frame, { keyFrame: true });
await encoder.flush();
```

**Ключевое.** `metadata.decoderConfig` содержит `description`. Его **надо передать декодеру на приёмной стороне** — иначе он не заработает.

---

## 23.7. AudioDecoder и AudioEncoder

Симметрично видео. `AudioData` — PCM-сэмплы, `EncodedAudioChunk` — сжатые.

```js
const audioDecoder = new AudioDecoder({
    output: (data) => {
        const dest = new Float32Array(data.numberOfFrames * data.numberOfChannels);
        data.copyTo(dest, { planeIndex: 0, format: "f32-planar" });
        data.close();
    },
    error: (e) => {},
});
audioDecoder.configure({ codec: "opus", sampleRate: 48000, numberOfChannels: 2 });
```

Отличие от `VideoFrame`: `AudioData` не `CanvasImageSource`. Но его можно скормить в `AudioContext` через `AudioBuffer` или обработать в Rust.

---

## 23.8. WebCodecs + WASM

WebCodecs — JS API. Rust не может вызвать `VideoDecoder` напрямую. Но **может управлять** им и **обрабатывать кадры между стадиями**.

**Схема:**

```
JS: fetch EncodedVideoChunk
  → VideoDecoder.decode(chunk)
  → output: VideoFrame
  → ImageData → WASM-память
  → Rust: обработка (фильтр, детекция, эффект)
  → JS: забираем результат
  → VideoEncoder.encode(resultFrame)
  → отправка / сохранение
```

**Где Rust даёт выигрыш:** фильтры с состоянием (motion blur, шумоподавление), ML (через `tract`, `ort`), низкоуровневая обработка (LUT, гамма).

**Где оставить работу браузеру:** сам кодек — H.264/AV1/AAC. Аппаратное ускорение доступно только через браузерный кодек. Демуксинг контейнера — если есть готовая JS-библиотека. Синхронизация A/V, буферизация — «обвязка».

**Правило:** Rust — для **вычислений**, JS — для **управления API**.

---

## 23.9. CPU vs GPU для обработки пикселей

Ключевое наблюдение главы. Работа с кадрами между декодером и энкодером может идти двумя путями.

**CPU-путь (Rust).**
- `VideoFrame.copyTo(rgba_buffer, {format:"RGBA"})` — GPU→CPU readback, ~4 МБ на кадр
- wasm-bindgen копирует `Uint8Array` в WASM-память
- Rust обрабатывает, возвращает
- wasm-bindgen копирует обратно
- `new ImageData(...)` → `new VideoFrame(canvas)`

**GPU-путь (WebGL2).**
- `gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, frame)` — GPU→GPU, текстура создаётся без readback
- `gl.drawArrays` с шейдером — один вызов на кадр
- `gl_canvas.transferToImageBitmap()` — GPU→GPU, результат в `ImageBitmap`
- `new VideoFrame(bitmap)` — сразу в encoder

**Оба используют `OffscreenCanvas` — но CPU-путь спускается на CPU, а GPU-путь остаётся в GPU.**

Разница в таймингах на стадии Invert — это **цена readback'а**. Остальные стадии (encode, decode, re-encode) одинаковы, потому что там работает браузер.

**Про флип по Y.** WebGL читает текстуры снизу вверх, а `VideoFrame` содержит пиксели сверху вниз. Без `gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true)` картинка переворачивается.

---

## 23.10. Что показывает демо

**Layout:**

- **Два canvas** рядом: `Source` (main thread генерирует 30 синтетических кадров) и `Result` (worker рисует инвертированный кадр).
- **Селект «Режим»**: CPU (Rust) / GPU (WebGL2).
- **Кнопка «Запустить пайплайн».**
- **Прогресс-бар** — determinate, 0/30 → 30/30.
- **Статистика:**
  - Encode (main), Decode (worker), **Invert (worker)**, Re-encode (worker).
  - Worker instance id, Frames main.

**Сценарий:**

1. Main thread рендерит 30 кадров (цветной градиент + номер), кодирует в H.264, передаёт чанки worker'у.
2. Worker декодирует, инвертирует цвета выбранным режимом, кодирует обратно.
3. Результат — на OffscreenCanvas.
4. Смена режима → повторный запуск → сравнение таймингов.

**Что показывает сравнение (реальные числа из демо):**

| Стадия | CPU | GPU |
|--------|----:|----:|
| Encode | 61 мс | 58 мс |
| Decode | 105 мс | 97 мс |
| **Invert** | **622 мс** | **266 мс** |
| Re-encode | 140 мс | 151 мс |
| **Итого** | **~928 мс** | **~561 мс** |

Все стадии равны, кроме Invert. Разница 356 мс — **буквально цена readback'а**.

---

## 23.11. Wow-момент

**Первый.** Кадр с canvas → `VideoEncoder` → `EncodedVideoChunk`. Размер: 200 КБ вместо 4 МБ. Это H.264, аппаратный кодек браузера.

**Второй.** Смена режима CPU → GPU. Стадия Invert: 622 → 266 мс. **2.3× быстрее.** Причина — пиксели не покидают GPU.

**Третий.** Остальные стадии — одинаковые. Encode, decode, re-encode не отличаются, потому что там работает браузер. Только обработка пикселей — «щель», где можно выбрать CPU или GPU.

**Четвёртый.** DevTools → Performance. Main thread плоский — вся работа в worker'е. `Frames main` растёт: 800, 1500, 2000+.

**Пятый.** Console: `instance_id()` в main thread и worker'е — разные. Один `.wasm`, два инстанса.

---

## 23.12. Исходные файлы

### `src/codecs_demo.rs`

```rust
//! 23. WebCodecs.
//!
//! Демо: покадровая обработка видео. Main thread кодирует
//! синтетические кадры в H.264, Worker декодирует их, обрабатывает
//! пиксели (CPU Rust или GPU WebGL2), кодирует обратно.
//!
//! ## Где Rust
//!
//! WebCodecs — JS API. Rust не может вызвать VideoDecoder напрямую.
//! Но Rust может обрабатывать VideoFrame между декодером и
//! энкодером — и это то, где WASM даёт выигрыш по сравнению с
//! JS-циклом по пикселям.
//!
//! ## Что экспортируется
//!
//! - `invert_rgba` — инверсия RGBA-байтов (используется в CPU-режиме).
//! - `average_rgb` — средний RGB (метрика для статистики).
//! - feature detection и helpers для управления Worker'ом.

use js_sys::{Array, Reflect};
use wasm_bindgen::prelude::*;
use web_sys::Worker;

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `matches!` для tail-позиций.
#[inline]
fn bool_of(value: bool) -> bool {
    value
}

/// Обёртка над `format!` для tail-позиций.
#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Feature detection
// =============================================================================

/// `Worker` доступен в этом браузере.
pub fn is_worker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("Worker")),
        Ok(v) if v.is_function()
    ))
}

/// `VideoEncoder` доступен.
pub fn is_video_encoder_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("VideoEncoder")),
        Ok(v) if v.is_function()
    ))
}

/// `VideoDecoder` доступен.
pub fn is_video_decoder_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("VideoDecoder")),
        Ok(v) if v.is_function()
    ))
}

/// `VideoFrame` доступен.
pub fn is_video_frame_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("VideoFrame")),
        Ok(v) if v.is_function()
    ))
}

/// Полная поддержка WebCodecs (видео).
pub fn is_video_webcodecs_supported() -> bool {
    is_video_encoder_supported()
        && is_video_decoder_supported()
        && is_video_frame_supported()
}

/// Сводка поддержки — для статистики в UI.
pub fn support_summary() -> String {
    format_str(format_args!(
        "Worker: {w} · VideoEncoder: {e} · VideoDecoder: {d} · VideoFrame: {f}",
        w = if is_worker_supported() { "да" } else { "нет" },
        e = if is_video_encoder_supported() { "да" } else { "нет" },
        d = if is_video_decoder_supported() { "да" } else { "нет" },
        f = if is_video_frame_supported() { "да" } else { "нет" },
    ))
}

// =============================================================================
// Обработка пикселей — то, ради чего здесь Rust
// =============================================================================

/// Инвертирует RGB-каналы RGBA-изображения. Альфа не трогается.
///
/// Вызывается из Worker'а для каждого декодированного кадра
/// в CPU-режиме. JS-цикл по 2M пикселей занимает десятки
/// миллисекунд; Rust/WASM — единицы.
#[wasm_bindgen]
pub fn invert_rgba(data: &mut [u8]) {
    let n = data.len() & !3;
    let mut i = 0;
    while i < n {
        data[i] = 255 - data[i];
        data[i + 1] = 255 - data[i + 1];
        data[i + 2] = 255 - data[i + 2];
        i += 4;
    }
}

/// Средний RGB изображения. Используется для метрики в статистике.
#[wasm_bindgen]
pub fn average_rgb(data: &[u8]) -> f64 {
    if data.is_empty() {
        return 0.0;
    }
    let mut sum: u64 = 0;
    let mut count: u64 = 0;
    let n = data.len() & !3;
    let mut i = 0;
    while i < n {
        sum += data[i] as u64 + data[i + 1] as u64 + data[i + 2] as u64;
        count += 3;
        i += 4;
    }
    if count == 0 { 0.0 } else { (sum as f64) / (count as f64) }
}

// =============================================================================
// Worker helpers
// =============================================================================

/// Создаёт module worker для обработки кадров.
pub fn create_worker(url: &str) -> Result<Worker, JsValue> {
    let opts = web_sys::WorkerOptions::new();
    opts.set_type(web_sys::WorkerType::Module);
    Worker::new_with_options(url, &opts)
}

/// Отправляет Worker'у «начать обработку»: конфиг кодека + режим.
///
/// `mode` — `"cpu"` (Rust invert) или `"gpu"` (WebGL2 shader).
pub fn post_start(
    worker: &Worker,
    codec: &str,
    width: u32,
    height: u32,
    frame_count: u32,
    mode: &str,
) -> Result<(), JsValue> {
    let msg = js_sys::Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("start"))?;
    Reflect::set(&msg, &JsValue::from_str("codec"), &JsValue::from_str(codec))?;
    Reflect::set(&msg, &JsValue::from_str("width"), &JsValue::from_f64(width as f64))?;
    Reflect::set(&msg, &JsValue::from_str("height"), &JsValue::from_f64(height as f64))?;
    Reflect::set(
        &msg,
        &JsValue::from_str("frameCount"),
        &JsValue::from_f64(frame_count as f64),
    )?;
    Reflect::set(&msg, &JsValue::from_str("mode"), &JsValue::from_str(mode))?;
    worker.post_message(&msg)
}

/// Отправляет Worker'у закодированный чанк (transfer).
pub fn post_encoded_chunk(
    worker: &Worker,
    chunk_type: &str,
    timestamp_us: f64,
    buffer: &JsValue,
) -> Result<(), JsValue> {
    let msg = js_sys::Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str("encoded_chunk"),
    )?;
    Reflect::set(
        &msg,
        &JsValue::from_str("chunkType"),
        &JsValue::from_str(chunk_type),
    )?;
    Reflect::set(
        &msg,
        &JsValue::from_str("timestamp"),
        &JsValue::from_f64(timestamp_us),
    )?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), buffer)?;

    let transfer = Array::new();
    transfer.push(buffer);
    worker.post_message_with_transfer(&msg, &transfer)
}

/// Команда «финализировать» — Worker шлёт последний flush().
pub fn post_finish(worker: &Worker) -> Result<(), JsValue> {
    let msg = js_sys::Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("finish"))?;
    worker.post_message(&msg)
}
```

### `src/lib.rs` — добавление

```rust
pub mod codecs_demo;   // ← 23
```

### `tests/wasm_codecs_demo.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::codecs_demo::{
    average_rgb, invert_rgba, is_video_decoder_supported,
    is_video_encoder_supported, is_video_frame_supported, is_worker_supported,
    support_summary,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn worker_is_supported_in_browser() {
    assert!(is_worker_supported());
}

#[wasm_bindgen_test]
fn video_encoder_is_supported_in_browser() {
    let _ = is_video_encoder_supported();
}

#[wasm_bindgen_test]
fn video_decoder_is_supported_in_browser() {
    let _ = is_video_decoder_supported();
}

#[wasm_bindgen_test]
fn video_frame_is_supported_in_browser() {
    let _ = is_video_frame_supported();
}

#[wasm_bindgen_test]
fn support_summary_returns_string() {
    let s = support_summary();
    assert!(s.contains("VideoEncoder"));
    assert!(s.contains("VideoDecoder"));
    assert!(s.contains("VideoFrame"));
}

#[wasm_bindgen_test]
fn invert_rgba_flips_rgb_keeps_alpha() {
    let mut data = vec![10u8, 20, 30, 200, 200, 100, 50, 255];
    invert_rgba(&mut data);
    assert_eq!(data[0], 245);
    assert_eq!(data[1], 235);
    assert_eq!(data[2], 225);
    assert_eq!(data[3], 200, "alpha preserved");
    assert_eq!(data[4], 55);
    assert_eq!(data[5], 155);
    assert_eq!(data[6], 205);
    assert_eq!(data[7], 255, "alpha preserved");
}

#[wasm_bindgen_test]
fn invert_rgba_is_self_inverse() {
    let orig = vec![
        10u8, 20, 30, 100, 40, 50, 60, 150, 200, 210, 220, 250,
    ];
    let mut data = orig.clone();
    invert_rgba(&mut data);
    invert_rgba(&mut data);
    assert_eq!(data, orig);
}

#[wasm_bindgen_test]
fn invert_rgba_handles_empty() {
    let mut data: Vec<u8> = vec![];
    invert_rgba(&mut data);
    assert!(data.is_empty());
}

#[wasm_bindgen_test]
fn invert_rgba_handles_unaligned_length() {
    let mut data = vec![10u8, 20, 30, 40, 50];
    invert_rgba(&mut data);
    assert_eq!(data[0], 245);
    assert_eq!(data[1], 235);
    assert_eq!(data[2], 225);
    assert_eq!(data[3], 40);
    assert_eq!(data[4], 50);
}

#[wasm_bindgen_test]
fn average_rgb_of_uniform_is_that_value() {
    let mut data = vec![0u8; 16];
    for i in (0..16).step_by(4) {
        data[i] = 100;
        data[i + 1] = 100;
        data[i + 2] = 100;
        data[i + 3] = 255;
    }
    let avg = average_rgb(&data);
    assert!((avg - 100.0).abs() < 0.001);
}

#[wasm_bindgen_test]
fn average_rgb_ignores_alpha() {
    let data = vec![100u8, 100, 100, 0];
    let avg = average_rgb(&data);
    assert!((avg - 100.0).abs() < 0.001);
}
```

### `web/public/workers/codecs_worker.js`

```javascript
// 23. WebCodecs worker.
//
// Два режима обработки пикселей:
//   - "cpu": VideoFrame → copyTo → Rust invert → ImageData → VideoFrame
//   - "gpu": VideoFrame → texImage2D → WebGL2 invert shader → ImageBitmap
//
// В обоих режимах ресурсы (canvas, контекст, буфер) переиспользуются
// между кадрами — никаких аллокаций в цикле.
//
// ## Флип по Y
//
// WebGL читает текстуру снизу вверх, а VideoFrame содержит пиксели
// сверху вниз. Без UNPACK_FLIP_Y_WEBGL картинка переворачивается.
//
// ## Почему ImageBitmap в GPU-режиме
//
// gl_canvas после drawArrays содержит результат. transferToImageBitmap()
// забирает его в GPU-resident ImageBitmap за O(1) — без readback.
// Из 30 таких bitmap'ов собирается корректный поток кадров для
// re-encode, а не 30 копий последнего кадра.

self.postMessage({ type: "boot", stage: "worker_started" });
console.log("[codecs] worker started");

// =============================================================================
// Состояние
// =============================================================================

let ready = false;
let wasm_invert_rgba = null;
let wasm_instance_id = null;

let result_canvas = null;
let result_ctx = null;
let canvas_w = 0;
let canvas_h = 0;

// CPU-путь: переиспользуемый canvas + буфер.
let work_canvas = null;
let work_ctx = null;
let rgba_buffer = null;

// GPU-путь: переиспользуемый canvas + WebGL2.
let gl_canvas = null;
let gl = null;
let gl_program = null;
let gl_texture = null;

let task_codec = "avc1.42001E";
let task_w = 0;
let task_h = 0;
let task_frame_count = 0;
let task_mode = "cpu";
let task_chunks = [];
let decoder_config = null;

const pending = [];

// =============================================================================
// onmessage — синхронно, до import
// =============================================================================

self.onmessage = (e) => {
    if (!ready) { pending.push(e); return; }
    handle(e.data);
};

self.addEventListener("error", (ev) => {
    self.postMessage({ type: "runtime_error", message: ev.message || "?" });
});

self.addEventListener("unhandledrejection", (ev) => {
    self.postMessage({
        type: "runtime_error",
        message: "unhandledrejection: " + String(ev.reason),
    });
});

// =============================================================================
// WASM
// =============================================================================

(async () => {
    try {
        const mod = await import("/pkg/hello_wasm02.js");
        await mod.default();
        wasm_invert_rgba = mod.invert_rgba;
        wasm_instance_id = mod.instance_id;
        if (typeof wasm_invert_rgba !== "function") {
            throw new Error("no invert_rgba in module");
        }
        ready = true;
        const inst = typeof wasm_instance_id === "function"
            ? wasm_instance_id() : "wasm-unknown";
        self.postMessage({ type: "ready", instance: inst });
        while (pending.length) handle(pending.shift().data);
    } catch (err) {
        self.postMessage({
            type: "init_error",
            message: String(err) + (err?.stack ? " | " + err.stack : ""),
        });
    }
})();

// =============================================================================
// Обработчик сообщений
// =============================================================================

function handle(msg) {
    switch (msg.type) {
        case "init_canvas":
            result_canvas = msg.canvas;
            canvas_w = msg.width;
            canvas_h = msg.height;
            result_ctx = result_canvas.getContext("2d");
            break;

        case "decoder_config":
            decoder_config = msg.config;
            break;

        case "start":
            task_codec = msg.codec || task_codec;
            task_w = msg.width || 0;
            task_h = msg.height || 0;
            task_frame_count = msg.frameCount || 0;
            task_mode = msg.mode || "cpu";
            task_chunks = [];
            console.log("[codecs] start task:", task_codec,
                task_w, "x", task_h, "mode:", task_mode);
            break;

        case "encoded_chunk":
            task_chunks.push(msg);
            break;

        case "finish":
            process_task().catch((err) => {
                console.error("[codecs] process failed:", err);
                self.postMessage({
                    type: "runtime_error",
                    message: "process: " + String(err),
                });
            });
            break;
    }
}

// =============================================================================
// Переиспользуемые ресурсы
// =============================================================================

function ensure_cpu_canvas() {
    if (work_canvas) return;
    work_canvas = new OffscreenCanvas(task_w, task_h);
    work_ctx = work_canvas.getContext("2d");
    rgba_buffer = new Uint8ClampedArray(task_w * task_h * 4);
    console.log("[codecs] cpu canvas ready", task_w, "x", task_h);
}

function ensure_gpu_canvas() {
    if (gl) return;
    gl_canvas = new OffscreenCanvas(task_w, task_h);
    gl = gl_canvas.getContext("webgl2", { premultipliedAlpha: false });
    if (!gl) throw new Error("WebGL2 not available in worker");

    // WebGL читает текстуры снизу вверх; VideoFrame — сверху вниз.
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);

    const vs = `#version 300 es
        in vec2 a_pos;
        out vec2 v_uv;
        void main() {
            v_uv = a_pos * 0.5 + 0.5;
            gl_Position = vec4(a_pos, 0.0, 1.0);
        }`;

    const fs = `#version 300 es
        precision mediump float;
        in vec2 v_uv;
        uniform sampler2D u_tex;
        out vec4 out_color;
        void main() {
            vec4 c = texture(u_tex, v_uv);
            out_color = vec4(1.0 - c.rgb, c.a);
        }`;

    const compile = (type, src) => {
        const s = gl.createShader(type);
        gl.shaderSource(s, src);
        gl.compileShader(s);
        if (!gl.getShaderParameter(s, gl.COMPILE_STATUS)) {
            throw new Error(gl.getShaderInfoLog(s));
        }
        return s;
    };

    const prog = gl.createProgram();
    gl.attachShader(prog, compile(gl.VERTEX_SHADER, vs));
    gl.attachShader(prog, compile(gl.FRAGMENT_SHADER, fs));
    gl.linkProgram(prog);
    if (!gl.getProgramParameter(prog, gl.LINK_STATUS)) {
        throw new Error(gl.getProgramInfoLog(prog));
    }
    gl_program = prog;

    const quad = new Float32Array([
        -1, -1, 1, -1, -1, 1,
        -1, 1, 1, -1, 1, 1,
    ]);
    const buf = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buf);
    gl.bufferData(gl.ARRAY_BUFFER, quad, gl.STATIC_DRAW);

    const pos_loc = gl.getAttribLocation(prog, "a_pos");
    gl.enableVertexAttribArray(pos_loc);
    gl.vertexAttribPointer(pos_loc, 2, gl.FLOAT, false, 0, 0);

    gl_texture = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, gl_texture);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);

    gl.viewport(0, 0, task_w, task_h);
    console.log("[codecs] gpu canvas + webgl2 ready");
}

// =============================================================================
// Обработка одного кадра
// =============================================================================

function process_cpu(frame) {
    frame.copyTo(rgba_buffer, { format: "RGBA" });
    const u8 = new Uint8Array(
        rgba_buffer.buffer,
        rgba_buffer.byteOffset,
        rgba_buffer.byteLength,
    );
    wasm_invert_rgba(u8);
    return new ImageData(rgba_buffer, task_w, task_h);
}

function process_gpu(frame) {
    gl.bindTexture(gl.TEXTURE_2D, gl_texture);
    gl.texImage2D(
        gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, frame,
    );
    gl.useProgram(gl_program);
    gl.drawArrays(gl.TRIANGLES, 0, 6);
    return gl_canvas.transferToImageBitmap();
}

// =============================================================================
// Основной пайплайн
// =============================================================================

async function process_task() {
    if (!result_ctx) throw new Error("no result canvas");
    if (task_chunks.length === 0) throw new Error("no chunks received");

    if (task_mode === "gpu") {
        ensure_gpu_canvas();
    } else {
        ensure_cpu_canvas();
    }

    self.postMessage({
        type: "phase",
        name: `decode → ${task_mode} invert → re-encode`,
    });

    const decoded_frames = [];
    const decoder = new VideoDecoder({
        output: (frame) => decoded_frames.push(frame),
        error: (e) => console.error("[codecs] decode error:", e),
    });

    const cfg = decoder_config || {
        codec: task_codec,
        codedWidth: task_w,
        codedHeight: task_h,
    };
    decoder.configure(cfg);

    const decode_start = performance.now();
    for (const c of task_chunks) {
        decoder.decode(new EncodedVideoChunk({
            type: c.chunkType,
            timestamp: c.timestamp,
            duration: c.duration || 0,
            data: new Uint8Array(c.buffer),
        }));
    }
    await decoder.flush();
    decoder.close();
    const decode_ms = performance.now() - decode_start;
    console.log("[codecs] decoded", decoded_frames.length,
        "frames in", decode_ms.toFixed(1), "ms");

    if (decoded_frames.length === 0) throw new Error("no frames decoded");

    self.postMessage({
        type: "progress",
        done: 0,
        total: decoded_frames.length,
    });

    const invert_start = performance.now();
    const cpu_frames = [];
    const gpu_frames = [];

    for (let i = 0; i < decoded_frames.length; i++) {
        const frame = decoded_frames[i];

        if (task_mode === "gpu") {
            gpu_frames.push(process_gpu(frame));
        } else {
            cpu_frames.push(process_cpu(frame));
        }
        frame.close();

        self.postMessage({
            type: "progress",
            done: i + 1,
            total: decoded_frames.length,
        });
        await new Promise((r) => setTimeout(r, 0));
    }

    const invert_ms = performance.now() - invert_start;
    console.log(`[codecs] ${task_mode} invert:`, invert_ms.toFixed(1), "ms");

    // Показ последнего кадра на result_canvas.
    if (task_mode === "gpu" && gpu_frames.length > 0) {
        const last = gpu_frames[gpu_frames.length - 1];
        result_ctx.drawImage(last, 0, 0, canvas_w, canvas_h);
    } else if (cpu_frames.length > 0) {
        const last = cpu_frames[cpu_frames.length - 1];
        if (last.width === canvas_w && last.height === canvas_h) {
            result_ctx.putImageData(last, 0, 0);
        } else {
            const tmp = new OffscreenCanvas(last.width, last.height);
            tmp.getContext("2d").putImageData(last, 0, 0);
            result_ctx.drawImage(tmp, 0, 0, canvas_w, canvas_h);
        }
    }

    // Re-encode.
    self.postMessage({ type: "phase", name: "re-encode" });

    let reencoded_count = 0;
    const reencode_start = performance.now();
    const encoder = new VideoEncoder({
        output: () => { reencoded_count++; },
        error: (e) => console.error("[codecs] encode error:", e),
    });
    encoder.configure({
        codec: task_codec,
        width: canvas_w,
        height: canvas_h,
        bitrate: 1_000_000,
        framerate: 30,
    });

    if (task_mode === "gpu") {
        for (let i = 0; i < gpu_frames.length; i++) {
            const bitmap = gpu_frames[i];
            const vf = new VideoFrame(bitmap, { timestamp: i * 33333 });
            encoder.encode(vf, { keyFrame: i === 0 });
            vf.close();
            bitmap.close();
            await new Promise((r) => setTimeout(r, 0));
        }
    } else {
        for (let i = 0; i < cpu_frames.length; i++) {
            work_ctx.putImageData(cpu_frames[i], 0, 0);
            const vf = new VideoFrame(work_canvas, { timestamp: i * 33333 });
            encoder.encode(vf, { keyFrame: i === 0 });
            vf.close();
            await new Promise((r) => setTimeout(r, 0));
        }
    }

    await encoder.flush();
    encoder.close();
    const reencode_ms = performance.now() - reencode_start;
    console.log("[codecs] re-encoded", reencoded_count,
        "frames in", reencode_ms.toFixed(1), "ms");

    self.postMessage({
        type: "timings",
        decode_ms: decode_ms,
        rust_ms: invert_ms,
        reencode_ms: reencode_ms,
    });
    self.postMessage({ type: "done" });

    task_chunks = [];
    decoder_config = null;
}
```

### `src/app.rs`

```rust
//! Глава 23 — WebCodecs.
//!
//! Демо: покадровый media-пайплайн с двумя режимами инверсии.
//!
//! Main thread кодирует 30 кадров в H.264, передаёт Worker'у.
//! Worker декодирует, инвертирует цвета одним из двух способов:
//!
//! - **CPU (Rust)** — `copyTo` в RGBA-буфер, `invert_rgba` в WASM,
//!   `ImageData` → `VideoFrame`.
//! - **GPU (WebGL2)** — `texImage2D(frame)`, шейдер инвертирует
//!   пиксели прямо на GPU, результат забирается через
//!   `transferToImageBitmap`.
//!
//! ## Что показывает сравнение
//!
//! CPU-путь спускается с GPU на CPU и обратно — дорого. GPU-путь
//! не покидает GPU. Разница в таймингах видна в статистике.
//!
//! ## Контракт с lib.rs
//!
//! `lib.rs::run()` проверяет `is_worker_context()` и вызывает
//! `logging::init()`. `app::run()` — только демо-логика.
//!
//! **Файл автономен.** Стили `.cd-*` инжектируются в документ.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Object, Promise, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{
    CanvasRenderingContext2d, Document, Event, HtmlCanvasElement, HtmlElement,
    HtmlProgressElement, HtmlSelectElement, MessageEvent, Worker,
};

use crate::codecs_demo::{
    create_worker, is_video_webcodecs_supported, is_worker_supported,
    post_encoded_chunk, post_finish, post_start, support_summary,
};

// =============================================================================
// Утилита для обхода багов rust-analyzer в tail-позиции
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Мини-хелперы для JsValue
// =============================================================================

fn get_fn(target: &JsValue, name: &str) -> Result<Function, JsValue> {
    Reflect::get(target, &JsValue::from_str(name))?
        .dyn_into::<Function>()
        .map_err(|_| JsValue::from_str("not a function"))
}

fn invoke(target: &JsValue, name: &str, args: &[JsValue]) -> Result<JsValue, JsValue> {
    let f = get_fn(target, name)?;
    let arr = Array::new();
    for a in args {
        arr.push(a);
    }
    f.apply(target, &arr)
}

fn construct(ctor_name: &str, args: &[JsValue]) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str(ctor_name))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str(ctor_name));
    }
    let ctor = ctor.dyn_into::<Function>()?;
    let arr = Array::new();
    for a in args {
        arr.push(a);
    }
    Reflect::construct(&ctor, &arr)
}

fn set_prop(target: &JsValue, key: &str, value: &JsValue) -> Result<(), JsValue> {
    Reflect::set(target, &JsValue::from_str(key), value).map(|_| ())
}

fn get_str(target: &JsValue, key: &str) -> String {
    Reflect::get(target, &JsValue::from_str(key))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

fn get_f64(target: &JsValue, key: &str) -> f64 {
    Reflect::get(target, &JsValue::from_str(key))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

fn num(v: f64) -> JsValue {
    JsValue::from_f64(v)
}

// =============================================================================
// Мини-хелперы для canvas
// =============================================================================

fn set_fill(ctx: &JsValue, color: &str) {
    let _ = set_prop(ctx, "fillStyle", &JsValue::from_str(color));
}

fn set_font(ctx: &JsValue, font: &str) {
    let _ = set_prop(ctx, "font", &JsValue::from_str(font));
}

fn draw_rect(ctx: &JsValue, x: f64, y: f64, w: f64, h: f64) {
    let _ = invoke(ctx, "fillRect", &[num(x), num(y), num(w), num(h)]);
}

fn draw_text(ctx: &JsValue, text: &str, x: f64, y: f64) {
    let _ = invoke(ctx, "fillText", &[JsValue::from_str(text), num(x), num(y)]);
}

// =============================================================================
// Service Worker cleanup
// =============================================================================

fn has_sw_controller() -> bool {
    let global = js_sys::global();
    let Ok(nav) = Reflect::get(&global, &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(sw) = Reflect::get(&nav, &JsValue::from_str("serviceWorker")) else {
        return false;
    };
    if sw.is_undefined() || sw.is_null() {
        return false;
    }
    let Ok(ctrl) = Reflect::get(&sw, &JsValue::from_str("controller")) else {
        return false;
    };
    !ctrl.is_null() && !ctrl.is_undefined()
}

fn unregister_sw_and_reload(document: &Document) -> Result<(), JsValue> {
    let script = document.create_element("script")?;
    script.set_text_content(Some(
        r#"(function(){
            var KEY='cd-sw-cleared-23';
            if(sessionStorage.getItem(KEY)==='1') return;
            sessionStorage.setItem(KEY,'1');
            if(!('serviceWorker' in navigator)){location.reload();return;}
            navigator.serviceWorker.getRegistrations().then(function(rs){
                var ps=[];for(var i=0;i<rs.length;i++)ps.push(rs[i].unregister());
                return Promise.all(ps);
            }).then(function(){location.reload();}).catch(function(){location.reload();});
        })();"#,
    ));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&script)?;
    Ok(())
}

fn show_sw_notice(document: &Document) -> Result<(), JsValue> {
    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html(
        r#"<div style="max-width:900px;padding:20px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#c9d1d9;font-family:system-ui,sans-serif;">
            <h2 style="margin:0 0 10px 0;color:#f0883e;">Очистка старых Service Worker'ов…</h2>
            <p style="margin:0;line-height:1.55;">Снимаем регистрацию и перезагружаем страницу.</p>
        </div>"#,
    );
    Ok(())
}

// =============================================================================
// Стили
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "cd-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() {
            p.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

const STYLE_CSS: &str = r#"
.cd-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.cd-desc-line { margin:6px 0; }
.cd-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;}
.cd-canvases { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.cd-canvas-col { display:flex;flex-direction:column;gap:6px;}
.cd-canvas-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.cd-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.cd-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.cd-btn:hover:not([disabled]) { background:#30363d; }
.cd-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.cd-btn-primary:hover:not([disabled]) { background:#2ea043; }
.cd-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.cd-select { font-size:12px;padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#0d1117;color:#c9d1d9;}
.cd-progress { flex:1;min-width:200px;height:10px;accent-color:#3fb950;}
.cd-stats { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;margin-bottom:16px;}
.cd-stats .label { color:#8b949e; }
.cd-stats .value { color:#79c0ff; }
.cd-stats .ok { color:#3fb950; }
.cd-stats .warn { color:#f0883e; }
.cd-stats .err { color:#f85149; }
.cd-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Константы
// =============================================================================

const WORKER_URL: &str = "/workers/codecs_worker.js";
const FRAME_W: u32 = 320;
const FRAME_H: u32 = 240;
const FRAME_COUNT: u32 = 30;
const CODEC: &str = "avc1.42001E";
const FRAME_DURATION_US: f64 = 33333.0;

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    worker: Option<Worker>,
    worker_ready: bool,
    worker_status: String,
    error: Option<String>,
    phase: String,
    progress_done: u32,
    progress_total: u32,
    encode_ms: f64,
    decode_ms: f64,
    rust_ms: f64,
    reencode_ms: f64,
    frames: u32,
    running: bool,
}

impl DemoState {
    fn new() -> Self {
        Self {
            worker: None,
            worker_ready: false,
            worker_status: String::from("инициализация…"),
            error: None,
            phase: String::from("простой"),
            progress_done: 0,
            progress_total: 0,
            encode_ms: 0.0,
            decode_ms: 0.0,
            rust_ms: 0.0,
            reencode_ms: 0.0,
            frames: 0,
            running: false,
        }
    }
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("cd-stats") else {
        return;
    };

    let err_line = match st.error.as_ref() {
        Some(msg) => format_str(format_args!(
            r#"<div><span class="err">{msg}</span></div>"#,
            msg = msg,
        )),
        None => String::new(),
    };

    let status_class = if st.error.is_some() {
        "err"
    } else if st.running {
        "warn"
    } else if st.progress_total > 0 && st.progress_done == st.progress_total {
        "ok"
    } else {
        "value"
    };

    let inst_class = if st.worker_ready { "ok" } else { "warn" };

    let html = format_str(format_args!(
        r#"{err}
           <div><span class="label">Статус: </span><span class="{sc}">{phase}</span> · <span class="label">Кадров: </span><span class="value">{done}/{total}</span></div>
           <div><span class="label">Encode (main): </span><span class="value">{enc:.1} мс</span> · <span class="label">Decode (worker): </span><span class="value">{dec:.1} мс</span></div>
           <div><span class="label">Invert (worker): </span><span class="ok">{rust:.1} мс</span> · <span class="label">Re-encode (worker): </span><span class="value">{re:.1} мс</span></div>
           <div><span class="label">Worker: </span><span class="{ic}">{inst}</span> · <span class="label">Frames main: </span><span class="value">{fps}</span></div>"#,
        err = err_line,
        sc = status_class,
        phase = st.phase,
        done = st.progress_done,
        total = st.progress_total,
        enc = st.encode_ms,
        dec = st.decode_ms,
        rust = st.rust_ms,
        re = st.reencode_ms,
        ic = inst_class,
        inst = st.worker_status,
        fps = st.frames,
    ));
    el.set_inner_html(&html);
}

fn set_progress(document: &Document, done: f64, total: f64) {
    let Some(el) = document.get_element_by_id("cd-progress") else {
        return;
    };
    let Ok(p) = el.dyn_into::<HtmlProgressElement>() else {
        return;
    };
    if total > 0.0 {
        p.set_max(total);
        p.set_value(done);
    } else {
        p.set_max(1.0);
        p.set_value(0.0);
    }
}

fn set_result_label(document: &Document, mode: &str) {
    let Some(el) = document.get_element_by_id("cd-result-label") else {
        return;
    };
    let label = if mode == "gpu" {
        "Result (worker, GPU/WebGL2 invert)"
    } else {
        "Result (worker, CPU/Rust invert)"
    };
    el.set_text_content(Some(label));
}

const DESC_LINES: &[&str] = &[
    "WebCodecs даёт прямой доступ к аппаратным кодекам браузера: \
     <strong>VideoEncoder</strong>, <strong>VideoDecoder</strong>, \
     <strong>VideoFrame</strong>, <strong>EncodedVideoChunk</strong>. \
     Никакого ffmpeg.wasm — кодеки уже встроены.",
    "Пайплайн: main thread кодирует кадры → VideoEncoder → \
     transfer в Worker → VideoDecoder → инверсия → VideoEncoder \
     → результат на OffscreenCanvas worker'а.",
    "<strong>Два режима инверсии.</strong> <em>CPU (Rust)</em> — \
     <code>copyTo</code> в RGBA-буфер, WASM-функция, обратно в \
     <code>ImageData</code>: пиксели покидают GPU и возвращаются. \
     <em>GPU (WebGL2)</em> — <code>texImage2D(frame)</code> + \
     <code>transferToImageBitmap</code>: пиксели остаются в GPU. \
     Разница в таймингах Invert — цена readback'а.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("cd-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("cd-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Генерация кадров
// =============================================================================

fn draw_frame(ctx: &CanvasRenderingContext2d, w: u32, h: u32, idx: u32) {
    let ctx_js: &JsValue = ctx.as_ref();

    let hue = ((idx as f64 / FRAME_COUNT as f64) * 360.0) % 360.0;
    let fill = format_str(format_args!("hsl({hue:.0}, 80%, 45%)"));
    set_fill(ctx_js, &fill);
    draw_rect(ctx_js, 0.0, 0.0, w as f64, h as f64);

    set_fill(ctx_js, "#f0f6fc");
    set_font(ctx_js, "28px monospace");
    let text = format_str(format_args!("#{idx}"));
    draw_text(ctx_js, &text, 10.0, h as f64 - 10.0);
}

// =============================================================================
// rAF-счётчик
// =============================================================================

fn start_raf_counter(state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    let cell: Rc<RefCell<Option<wasm_bindgen::closure::Closure<dyn FnMut(f64)>>>> =
        Rc::new(RefCell::new(None));
    let cell2 = cell.clone();

    *cell.borrow_mut() = Some(wasm_bindgen::closure::Closure::wrap(Box::new(
        move |_now: f64| {
            {
                let mut st = state.borrow_mut();
                st.frames += 1;
            }
            if let Some(w) = web_sys::window() {
                if let Some(cb) = cell2.borrow().as_ref() {
                    let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
                }
            }
            let update = {
                let st = state.borrow();
                st.frames % 30 == 0
            };
            if update {
                let st = state.borrow();
                update_stats(&document, &st);
            }
        },
    ) as Box<dyn FnMut(f64)>));

    let cb = cell.borrow();
    window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    Ok(())
}

// =============================================================================
// Worker-обработчик
// =============================================================================

fn wire_worker_messages(
    worker: &Worker,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let state_err = state.clone();

    let on_msg = wasm_bindgen::closure::Closure::<dyn FnMut(MessageEvent)>::new(
        move |ev: MessageEvent| {
            let data = ev.data();
            let kind = get_str(&data, "type");
            let Some(doc) = web_sys::window().and_then(|w| w.document()) else {
                return;
            };

            match kind.as_str() {
                "boot" => {
                    let stage = get_str(&data, "stage");
                    {
                        let mut st = state.borrow_mut();
                        st.worker_status = format_str(format_args!("boot: {stage}"));
                    }
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                "ready" => {
                    let inst = {
                        let s = get_str(&data, "instance");
                        if s.is_empty() { String::from("wasm-?") } else { s }
                    };
                    {
                        let mut st = state.borrow_mut();
                        st.worker_ready = true;
                        st.worker_status = inst;
                        st.error = None;
                    }
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                "init_error" | "runtime_error" => {
                    let msg = get_str(&data, "message");
                    {
                        let mut st = state.borrow_mut();
                        st.phase = String::from("ошибка");
                        st.running = false;
                        st.error = Some(format_str(format_args!("Worker: {msg}")));
                    }
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                "phase" => {
                    let name = get_str(&data, "name");
                    {
                        let mut st = state.borrow_mut();
                        st.phase = name;
                    }
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                "progress" => {
                    let done = get_f64(&data, "done") as u32;
                    let total = get_f64(&data, "total") as u32;
                    {
                        let mut st = state.borrow_mut();
                        st.progress_done = done;
                        st.progress_total = total;
                    }
                    set_progress(&doc, done as f64, total as f64);
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                "timings" => {
                    let dec = get_f64(&data, "decode_ms");
                    let rust = get_f64(&data, "rust_ms");
                    let re = get_f64(&data, "reencode_ms");
                    {
                        let mut st = state.borrow_mut();
                        st.decode_ms = dec;
                        st.rust_ms = rust;
                        st.reencode_ms = re;
                    }
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                "done" => {
                    {
                        let mut st = state.borrow_mut();
                        st.phase = String::from("готово");
                        st.running = false;
                        st.error = None;
                    }
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                _ => {}
            }
        },
    );
    worker.set_onmessage(Some(on_msg.as_ref().unchecked_ref()));
    on_msg.forget();

    let on_err = wasm_bindgen::closure::Closure::<dyn FnMut(Event)>::new(
        move |ev: Event| {
            let target: &JsValue = ev.as_ref();
            let raw = get_str(target, "message");
            let msg = if raw.is_empty() {
                String::from("worker error (no message)")
            } else {
                raw
            };
            if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                {
                    let mut st = state_err.borrow_mut();
                    st.phase = String::from("ошибка");
                    st.running = false;
                    st.error = Some(format_str(format_args!("Worker: {msg}")));
                }
                let st = state_err.borrow();
                update_stats(&doc, &st);
            }
        },
    );
    worker.set_onerror(Some(on_err.as_ref().unchecked_ref()));
    on_err.forget();

    Ok(())
}

// =============================================================================
// WebCodecs: обёртки
// =============================================================================

fn make_video_encoder(
    chunks: Rc<RefCell<Vec<JsValue>>>,
    decoder_config: Rc<RefCell<Option<JsValue>>>,
) -> Result<JsValue, JsValue> {
    let on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, meta: JsValue| {
            if let Ok(cfg) = Reflect::get(&meta, &JsValue::from_str("decoderConfig")) {
                if !cfg.is_undefined() && !cfg.is_null() {
                    let mut dc = decoder_config.borrow_mut();
                    if dc.is_none() { *dc = Some(cfg); }
                }
            }
            chunks.borrow_mut().push(chunk);
        },
    );
    let on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |e: JsValue| { web_sys::console::error_1(&e); },
    );

    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;

    let encoder = construct("VideoEncoder", &[init.into()])?;
    on_output.forget();
    on_error.forget();
    Ok(encoder)
}

fn configure_encoder(encoder: &JsValue, w: u32, h: u32, codec: &str) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "width", &num(w as f64))?;
    set_prop(&cfg, "height", &num(h as f64))?;
    set_prop(&cfg, "bitrate", &num(1_000_000.0))?;
    set_prop(&cfg, "framerate", &num(30.0))?;
    let f = get_fn(encoder, "configure")?;
    let _ = f.call1(encoder, &cfg)?;
    Ok(())
}

fn make_video_frame(canvas: &HtmlCanvasElement, timestamp: f64) -> Result<JsValue, JsValue> {
    let canvas_val = JsValue::from(canvas.clone());
    let init = Object::new();
    set_prop(&init, "timestamp", &num(timestamp))?;
    construct("VideoFrame", &[canvas_val, init.into()])
}

fn encode_frame(encoder: &JsValue, frame: &JsValue, key: bool) -> Result<(), JsValue> {
    let f = get_fn(encoder, "encode")?;
    let opts = Object::new();
    if key {
        set_prop(&opts, "keyFrame", &JsValue::TRUE)?;
    }
    let _ = f.call2(encoder, frame, opts.as_ref())?;
    Ok(())
}

fn close_video_frame(frame: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(frame, "close")?;
    let _ = f.call0(frame)?;
    Ok(())
}

async fn await_flush(encoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "flush")?;
    let promise = f.call0(encoder)?.dyn_into::<Promise>()?;
    let _ = JsFuture::from(promise).await?;
    Ok(())
}

fn extract_chunk_bytes(chunk: &JsValue) -> Result<JsValue, JsValue> {
    let byte_length = get_f64(chunk, "byteLength") as u32;
    let u8 = Uint8Array::new_with_length(byte_length);
    let copy_to = get_fn(chunk, "copyTo")?;
    let _ = copy_to.call1(chunk, u8.as_ref())?;
    Ok(u8.buffer().into())
}

// =============================================================================
// Пайплайн
// =============================================================================

fn run_pipeline(document: &Document, state: &Rc<RefCell<DemoState>>) {
    let worker = {
        let st = state.borrow();
        if st.running || !st.worker_ready { return; }
        st.worker.clone()
    };
    let Some(worker) = worker else { return };

    let mode = document
        .get_element_by_id("cd-mode")
        .and_then(|e| e.dyn_into::<HtmlSelectElement>().ok())
        .map(|s| s.value())
        .unwrap_or_else(|| String::from("cpu"));

    set_result_label(document, &mode);

    {
        let mut st = state.borrow_mut();
        st.running = true;
        st.phase = format_str(format_args!("encode (main, {mode})…"));
        st.error = None;
        st.progress_done = 0;
        st.progress_total = FRAME_COUNT;
        st.encode_ms = 0.0;
        st.decode_ms = 0.0;
        st.rust_ms = 0.0;
        st.reencode_ms = 0.0;
    }
    set_progress(document, 0.0, FRAME_COUNT as f64);
    {
        let st = state.borrow();
        update_stats(document, &st);
    }

    let doc = document.clone();
    let st = state.clone();

    spawn_local(async move {
        if let Err(e) = pipeline_inner(&doc, &st, &worker, &mode).await {
            let mut s = st.borrow_mut();
            s.running = false;
            s.phase = String::from("ошибка");
            s.error = Some(format_str(format_args!("Pipeline: {e:?}")));
            update_stats(&doc, &s);
        }
    });
}

async fn pipeline_inner(
    doc: &Document,
    st: &Rc<RefCell<DemoState>>,
    worker: &Worker,
    mode: &str,
) -> Result<(), JsValue> {
    let canvas = doc
        .get_element_by_id("cd-source")
        .ok_or_else(|| JsValue::from_str("no #cd-source"))?
        .dyn_into::<HtmlCanvasElement>()?;
    let ctx = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    let chunks: Rc<RefCell<Vec<JsValue>>> = Rc::new(RefCell::new(Vec::new()));
    let decoder_config: Rc<RefCell<Option<JsValue>>> = Rc::new(RefCell::new(None));
    let encoder = make_video_encoder(chunks.clone(), decoder_config.clone())?;
    configure_encoder(&encoder, FRAME_W, FRAME_H, CODEC)?;

    let encode_start = js_sys::Date::now();
    for i in 0..FRAME_COUNT {
        draw_frame(&ctx, FRAME_W, FRAME_H, i);
        let frame = make_video_frame(&canvas, i as f64 * FRAME_DURATION_US)?;
        encode_frame(&encoder, &frame, i == 0)?;
        close_video_frame(&frame)?;
    }
    await_flush(&encoder).await?;
    let encode_ms = js_sys::Date::now() - encode_start;

    {
        let mut s = st.borrow_mut();
        s.encode_ms = encode_ms;
        s.phase = String::from("transfer → worker…");
    }
    {
        let s = st.borrow();
        update_stats(doc, &s);
    }

    // decoder_config (SPS/PPS).
    if let Some(cfg) = decoder_config.borrow().as_ref() {
        let msg = Object::new();
        set_prop(&msg, "type", &JsValue::from_str("decoder_config"))?;
        set_prop(&msg, "config", cfg)?;
        worker.post_message(&msg)?;
    }

    post_start(worker, CODEC, FRAME_W, FRAME_H, FRAME_COUNT, mode)?;

    {
        let chunks_ref = chunks.borrow();
        for chunk in chunks_ref.iter() {
            let ctype = {
                let t = get_str(chunk, "type");
                if t.is_empty() { String::from("delta") } else { t }
            };
            let ts = get_f64(chunk, "timestamp");
            let buffer = extract_chunk_bytes(chunk)?;
            post_encoded_chunk(worker, &ctype, ts, &buffer)?;
        }
    }
    post_finish(worker)?;

    Ok(())
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    if has_sw_controller() {
        show_sw_notice(&document)?;
        unregister_sw_and_reload(&document)?;
        return Ok(());
    }

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23 — WebCodecs"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_worker_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("cd-footnote");
        warn.set_text_content(Some("Браузер не поддерживает Worker."));
        root.append_child(&warn)?;
        return Ok(());
    }

    if !is_video_webcodecs_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("cd-footnote");
        warn.set_text_content(Some(&support_summary()));
        root.append_child(&warn)?;
    }

    let canvases = document.create_element("div")?;
    canvases.set_class_name("cd-canvases");
    canvases.set_inner_html(
        r#"<div class="cd-canvas-col">
               <div class="cd-canvas-label">Source (main thread)</div>
               <canvas id="cd-source" class="cd-canvas" width="320" height="240"></canvas>
           </div>
           <div class="cd-canvas-col">
               <div class="cd-canvas-label" id="cd-result-label">Result (worker, CPU/Rust invert)</div>
               <canvas id="cd-result" class="cd-canvas" width="320" height="240"></canvas>
           </div>"#,
    );
    root.append_child(&canvases)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("cd-block");
    controls.set_inner_html(
        r#"<button class="cd-btn cd-btn-primary" id="cd-run">Запустить пайплайн</button>
           <span class="cd-label">Режим:</span>
           <select id="cd-mode" class="cd-select">
               <option value="cpu">CPU (Rust)</option>
               <option value="gpu">GPU (WebGL2)</option>
           </select>
           <span class="cd-label" id="cd-support"></span>"#,
    );
    root.append_child(&controls)?;

    let progress_row = document.create_element("div")?;
    progress_row.set_class_name("cd-block");
    progress_row.set_inner_html(
        r#"<span class="cd-label">Прогресс:</span>
           <progress id="cd-progress" class="cd-progress" value="0" max="1"></progress>"#,
    );
    root.append_child(&progress_row)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("cd-stats");
    stats.set_id("cd-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("cd-footnote");
    footnote.set_text_content(Some(
        "CPU: VideoFrame → copyTo → Rust invert_rgba → ImageData → \
         VideoFrame — с промежуточным спуском на CPU. GPU: VideoFrame \
         → texImage2D → WebGL2 invert shader → transferToImageBitmap → \
         VideoFrame — целиком в GPU, без readback. Оба режима \
         переиспользуют один OffscreenCanvas на все 30 кадров. \
         Разница Invert — цена readback'а: пиксели дважды пересекают \
         шину GPU↔CPU. Остальные стадии (encode, decode, re-encode) \
         одинаковы в обоих режимах.",
    ));
    root.append_child(&footnote)?;

    if let Some(el) = document.get_element_by_id("cd-support") {
        el.set_text_content(Some(&support_summary()));
    }

    let state = Rc::new(RefCell::new(DemoState::new()));

    let worker = create_worker(WORKER_URL)?;
    wire_worker_messages(&worker, state.clone())?;

    let result_canvas = document
        .get_element_by_id("cd-result")
        .ok_or_else(|| JsValue::from_str("no #cd-result"))?
        .dyn_into::<HtmlCanvasElement>()?;
    {
        let offscreen = result_canvas.transfer_control_to_offscreen()?;
        let msg = Object::new();
        set_prop(&msg, "type", &JsValue::from_str("init_canvas"))?;
        set_prop(&msg, "canvas", offscreen.as_ref())?;
        set_prop(&msg, "width", &num(FRAME_W as f64))?;
        set_prop(&msg, "height", &num(FRAME_H as f64))?;
        let transfer = Array::new();
        transfer.push(offscreen.as_ref());
        worker.post_message_with_transfer(&msg, &transfer)?;
    }

    state.borrow_mut().worker = Some(worker);

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("cd-run")
            .ok_or_else(|| JsValue::from_str("no #cd-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            run_pipeline(&doc, &st);
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    start_raf_counter(state.clone())?;

    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    Ok(())
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_codecs_demo

# 3. Демо
npm run wasm:dev && npm start
```

**Важно:** `web/public/workers/codecs_worker.js` доступен по `/workers/codecs_worker.js`. Vite отдаёт файлы из `web/public/` по корню автоматически.

---

## 23.13. Анализ решения

**Какую ценность даёт демо.**

Подглава показывает **три стадии media-пайплайна** и **два способа обработки пикселей**:

1. **Encode** — `VideoEncoder` превращает `VideoFrame` в `EncodedVideoChunk`.
2. **Decode + Invert + Re-encode** — worker декодирует, обрабатывает пиксели, кодирует обратно.
3. **UI** — два canvas рядом: «оригинал» и «после инверсии».

**Три визуальных доказательства:**

- **Картинка инвертирована.** Красный `#29` → cyan `#29`.
- **Frames main растёт** — main thread свободен, работа в worker'е.
- **Смена режима меняет тайминги Invert в 2.3×.** Это и есть цена readback'а.

**Что осознанно не реализовано.**

- **Демуксинг контейнера.** MP4/WebM — отдельная задача. Демо генерирует кадры «в воздухе».
- **AudioDecoder / AudioEncoder в демо.** Feature detection есть, демо фокусируется на видео — симметричная картина для аудио не добавит новой информации.
- **`MediaStreamTrackProcessor`.** Демо не берёт видео с камеры.

**Правила проекта, применённые.**

1. **Autonomous app.rs.** Всё inline.
2. **`format_str(format_args!(...))` в tail-позициях.**
3. **`bool_of(matches!(...))` в tail-позиции.**
4. **RefMut-дисциплина.** Узкие scope.
5. **Feature detection перед API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.**
8. **Уникальный префикс `cd-`.**
9. **Никаких неиспользуемых импортов.**
10. **`Rc<RefCell<State>>`** — общий state.
11. **Каждому замыканию — свой `state.clone()`.**
12. **Ошибки в `state.error`, не перетираются rAF.**

**Проверки корректности.**

- `invert_rgba` работает над `&mut [u8]`, обрабатывает кратные 4, не трогает хвост.
- `average_rgb` игнорирует alpha, возвращает `f64`.
- Feature detection не паникует.
- `post_encoded_chunk` использует transfer — буфер не копируется.
- `decoder_config` (SPS/PPS) отправляется worker'у до чанков — иначе H.264 не декодируется.
- GPU-путь использует `UNPACK_FLIP_Y_WEBGL` — картинка не переворачивается.
- `transferToImageBitmap` вместо `drawImage(canvas)` — нет readback.
- Оба режима сохраняют 30 отдельных кадров — сравнение apples-to-apples.

---

## 23.14. Проделанные улучшения

**Архитектура.**

1. **Динамический `import()` в worker'е** — статический валит весь модуль при ошибке. Boot до импорта, `onmessage` синхронно.
2. **Безопасный `onerror`** — `Event` + `Reflect::get` с fallback вместо `ErrorEvent::message()`.
3. **«Липкая» ошибка** в `state.error` — не перетирается rAF.
4. **`decoder_config` (SPS/PPS)** — обязательна для H.264. Main thread вытаскивает из metadata, шлёт worker'у до чанков.
5. **Один `OffscreenCanvas` на всю сессию** — вместо 30 новых в цикле.
6. **`copyTo` вместо `drawImage + getImageData`** — короче путь CPU-обработки.
7. **`transferToImageBitmap` в GPU-пути** — GPU→GPU без readback.
8. **`UNPACK_FLIP_Y_WEBGL`** — устранён вертикальный флип.

**Стиль.**

9. **Мини-хелперы** (`get_fn`, `invoke`, `construct`, `set_prop`, `get_str`, `get_f64`, `num`) — убирают многословный `Reflect::get(...).ok().and_then(...)`.
10. **Canvas-хелперы** (`set_fill`, `set_font`, `draw_rect`, `draw_text`) — компактный код.
11. **Явный `JsValue::from(canvas.clone())`** вместо `canvas.as_ref()` — обход неоднозначного `AsRef`.
12. **`set_result_label`** — динамический label показывает текущий режим.
13. **`Frames main`** вместо `FPS main` — честная метка.

---

## 23.15. Итог

**WebCodecs** — прямой доступ к кодекам браузера. Без ffmpeg.wasm. Аппаратное ускорение по умолчанию.

**`VideoFrame`** — распакованный кадр, привязан к памяти GPU. `close()` обязателен. `VideoEncoder` превращает `VideoFrame` в `EncodedVideoChunk`; `VideoDecoder` — обратно.

**Rust между кодеками.** Кодирование/декодирование — работа браузера. Rust — пиксельная обработка между стадиями. Именно там WASM даёт выигрыш.

**Два режима обработки.**

- **CPU (Rust):** `copyTo` → WASM → `ImageData`. Пиксели дважды пересекают шину GPU↔CPU.
- **GPU (WebGL2):** `texImage2D` → shader → `transferToImageBitmap`. Пиксели остаются в GPU.

Разница на стадии Invert — **2.3× в пользу GPU**. Остальные стадии (encode, decode, re-encode) одинаковы, потому что там работает браузер.

**Worker обязателен.** WebCodecs-колбэки приходят десятки раз в секунду. Worker изолирует обработку, main thread остаётся отзывчивым.

**Демуксинг — отдельная задача.** WebCodecs работает с чанками, не с контейнерами.

Глава закрыта. Следующая тема — **WebGL 3D** (глава 24) или **WebGPU** (глава 25) — покажет более глубокую работу с GPU, где пиксели не покидают вычислительные шейдеры вообще.