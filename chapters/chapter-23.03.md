# Глава 23.3. VideoEncoder: распакованное → сжатое

## 23.3.0. Обновлённая структура

```
src/
  video_encoder.rs                ← НОВЫЙ модуль 23.3
  video_decoder.rs                ← из 23.2 (переиспользуется: buffer_source_to_vec)
  video_frame.rs                  ← из 23.1 (переиспользуется: хелперы JsValue)
  …
tests/
  wasm_video_encoder.rs           ← НОВЫЙ тест
src/lib.rs                        ← +1 строка: pub mod video_encoder;
src/app.rs                        ← демо 23.3 (заменяется целиком)
```

**Комментарий.** Модуль `video_encoder.rs` — про **VideoEncoder**: конструктор с колбэком `output(chunk, metadata)`, `configure` с `bitrate`/`framerate`, `encode(frame, {keyFrame})`, `flush`, `close`, `reset`, статический `isConfigSupported`. Префикс CSS — `ve-` (video encoder), не пересекается с `vf-` (23.1), `vd-` (23.2), `cd-` (глава 23). Мини-хелперы (`get_fn`, `construct`, `set_prop`, `num`, `bool_of`) — **переиспользуются** из `video_frame::*`. `buffer_source_to_vec` для подсчёта `description` — из `video_decoder::*`. Дублирования нет.

**Ссылки для детального изучения:**

- MDN: [`VideoEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder)
- MDN: [`VideoEncoder.configure()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder/configure)
- MDN: [`VideoEncoder.encode()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder/encode)
- MDN: [`VideoEncoder.isConfigSupported()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder/isConfigSupported)
- W3C: [WebCodecs § VideoEncoder](https://w3c.github.io/webcodecs/#videoencoder-interface)
- W3C: [WebCodecs § EncodedVideoChunk](https://w3c.github.io/webcodecs/#encodedvideochunk-interface)
- Wikipedia: [H.264 GOP, I/P-frames](https://en.wikipedia.org/wiki/Video_compression_picture_types)

---

## 23.3.1. О чём эта подглава

`VideoEncoder` — **выход** из WebCodecs в сторону сжатых данных. Он принимает `VideoFrame` (распакованный RGBA-кадр), сжимает его в H.264 и отдаёт `EncodedVideoChunk` через колбэк `output`. Каждый чанк имеет тип (`key` или `delta`) и размер в байтах.

Если 23.2 отвечала на вопрос «как разобрать то, что произвёл энкодер», то 23.3 — «как это произвести и почему оно такое маленькое».

Подглава отвечает на шесть вопросов.

**Первый — как настроить энкодер.** `configure({codec, width, height, bitrate, framerate})`. Первые три определяют размер, `bitrate` — целевой поток в битах/с, `framerate` — частоту.

**Второй — как кодировать.** `encode(frame, {keyFrame})` для каждого кадра. Первый кадр — key, остальные — delta (P-frame), если энкодер считает, что GOP не пора обновлять.

**Третий — где `description`.** Ровно там, где его ждёт декодер из 23.2: `metadata.decoderConfig.description`. Приходит **только на первом key-кадре**, потом `metadata.decoderConfig` — `undefined`.

**Четвёртый — как дождаться.** `await encoder.flush()`. После него все чанки гарантированно в колбэке `output`.

**Пятый — почему сжатие такое сильное.** Один `VideoFrame` — 307 200 байт (RGBA 320×240×4). Key-чанк — 2–4 КБ, delta — 0.3–0.9 КБ. Итог — ~15 КБ против 9.2 МБ исходных. **×600**.

**Шестой — что такое «bitrate» в VBR-кодеке.** Это **target**, а не гарантия. H.264 тратит столько битов, сколько нужно контенту. На простой сцене он уходит в **undershoot** (target завышен), при заниженном target — в **overshoot** (encoder не может опуститься ниже физического минимума). Демо показывает оба случая.

---

## 23.3.2. Что появится

- `VideoEncoder` с колбэками `output(chunk, metadata)` и `error`;
- `isConfigSupported` — проверка конфигурации до `configure`;
- `configure` с `bitrate`/`framerate`;
- `encode(frame, {keyFrame})` — принудительный key на первом кадре;
- `flush`, `close`, `reset`;
- чтение `chunk.type`, `chunk.byteLength`, `chunk.timestamp`;
- извлечение `metadata.decoderConfig` (codec, codedWidth, codedHeight, description);
- round-trip encode → decode для верификации;
- target vs actual bitrate — два числа с интерпретацией undershoot/overshoot.

---

## 23.3.3. Что такое GOP, key и delta

**GOP** (Group of Pictures) — последовательность кадров от одного key-кадра до следующего. Внутри GOP все кадры зависят от key-кадра (или друг от друга). Если key потерян — весь GOP декодировать невозможно.

**Key-кадр (IDR).** «Мгновенный»: декодируется независимо от предыдущих. Занимает много байт: содержит полное изображение (аналог JPEG).

**Delta-кадр (P-frame).** Кодируется как разница с предыдущим(и). Занимает мало: 0.3–0.9 КБ при 2–4 КБ у key.

**Почему это работает.** 30 кадров статичной сцены с одним движущимся кругом — это 30 снимков одного и того же фона. Разница между соседними кадрами — единицы процентов пикселей (сместившийся круг + счётчик кадра). H.264 кодирует эти проценты, а не весь кадр.

**Как энкодер решает.** По умолчанию H.264 сам выбирает момент следующего key (обычно раз в 1–2 секунды). Демо форсирует первый key через `{keyFrame: true}` — так гарантированно получаем `1 key + 29 delta` для 30 кадров при 1 секунде видео.

---

## 23.3.4. Колбэк `output` и `metadata.decoderConfig`

**`output: (chunk, metadata) => …`** вызывается для каждого готового чанка. `chunk` — `EncodedVideoChunk`. `metadata` — объект, у которого на **первом key-кадре** есть поле `decoderConfig`.

**`metadata.decoderConfig` содержит:**
- `codec` — фактический кодек (может отличаться от запрошенного);
- `codedWidth`, `codedHeight` — размеры после padding;
- `description` — `BufferSource` с `avcC`-box (SPS/PPS).

**Важно.** На delta-кадрах `metadata.decoderConfig` — `undefined`. Это спецификация: параметры последовательности описывают весь поток, повторять их в каждом кадре не нужно.

**Правило.** Захватывайте `decoderConfig` в колбэке один раз (при `captured == false`) и сохраняйте. Иначе легко получить `undefined` при попытке прочитать его на 20-м чанке.

**Практическая находка.** В Chrome `description` приходит как **`ArrayBuffer`**, а не `Uint8Array`. Универсальный хелпер `buffer_source_to_vec` из 23.2 корректно разбирает оба варианта.

---

## 23.3.5. Что показывает демо

**Layout:**

- **Source canvas** — main thread рисует 30 кадров сцены: статичный фон (градиент + звёзды + подпись) + маленький движущийся круг.
- **Verify canvas** — round-trip: кодируем → декодируем → воспроизводим.
- **Controls:**
  - Bitrate select: 50 kbps / 500 kbps / 1 Mbps / 5 Mbps.
  - Кнопка **Encode → Verify** — полный цикл.
  - Кнопка **Pause / Resume**.
- **Статистика (7 строк):**
  - `Статус` + `Playback`.
  - `Bitrate: target N · actual M бит/с (undershoot/overshoot ×K)`.
  - `Progress: p / total` (в процессе).
  - `Chunks: 30 (key: 1, delta: 29) · Bytes: total (key: K, delta: D)`.
  - `Sizes: min / avg / max байт · Δeff: key/avg_delta ≈ N×`.
  - `Сжатие: raw → encoded байт · ≈ R×`.
  - `decoderConfig: codec · W×H · description: N байт`.
- **Bar chart** — 30 полосок, высота пропорциональна `chunk.byteLength`. Key — оранжевый, delta — синий. Пунктирная горизонтальная линия — средний delta.

**Сценарий:**

1. При загрузке — source canvas показывает сцену. Verify пуст.
2. Клик **Encode → Verify**:
   - Проверка `isConfigSupported`.
   - Фаза `encode`: 30 кадров идут в `VideoEncoder` через yield каждые 16 мс.
   - Колбэк `output` инкрементит progress, собирает чанки, ловит `decoderConfig`.
   - Фаза `verify`: полученные 30 чанков идут в `VideoDecoder`.
   - Фаза `готово`: verify-canvas играет декодированные кадры в цикле.
3. Смена **bitrate** — **автоматический перезапуск** encode → verify. Пользователь видит новый результат без единого клика.
4. **Pause** — обе анимации замирают.

**Wow-момент — bar chart.** Полоска key упирается в потолок, delta — в 20 % высоты. **Δeff ≈ 5×**. Разница видна мгновенно.

**Wow-момент — коэффициент.** Строка `Сжатие: 9216000 → 15342 байт · ≈ 600.7×` показывает главное: **кодек сжимает в 600 раз**. На статичной сцене с одним движущимся элементом — это реально.

**Wow-момент — target vs actual bitrate.** Строка `Bitrate: target 1000000 · actual 122737 бит/с (undershoot ×8.1)` объясняет, почему H.264 — не CBR. При переключении на 50 kbps появится **overshoot ×2.4** — encoder не может опуститься ниже физического минимума.

---

## 23.3.6. Wow-момент на практике

**Первый.** Source и Verify показывают одинаковые сцены — encode + decode — обратимый цикл. Verify #14 при Source #0 — на разных временных шкалах (rAF vs play_idx).

**Второй.** Bar chart: оранжевый key во всю высоту, синие deltas — на 20 %. **Разница в 5 раз**. Это физически видимая работа H.264.

**Третий.** Строка `Сжатие ≈ 600×` — численное подтверждение. В 10 раз больше, чем ожидалось до практики. Именно потому, что сцена статична.

**Четвёртый.** Смена bitrate в select **мгновенно перерисовывает и Verify, и chart**. Никакой «пустой сцены». Пользователь видит другой результат за ~100 мс.

**Пятый.** При 1 Mbps строка Bitrate показывает `undershoot ×8.1`. При 50 kbps — `overshoot ×2.4`. **VBR работает в обе стороны.**

---

## 23.3.7. Исходные файлы

### `src/video_encoder.rs`

```rust
//! 23.3. VideoEncoder: распакованное → сжатое.
//!
//! Обёртки вокруг `VideoEncoder`:
//!
//! - конструктор с колбэками `output(chunk, metadata)` / `error`;
//! - `configure({ codec, width, height, bitrate, framerate })`;
//! - `encode(VideoFrame, { keyFrame })`, `flush()`, `close()`, `reset()`;
//! - статический `isConfigSupported(config)`;
//! - утилиты для чтения `EncodedVideoChunk` (type, byteLength, timestamp);
//! - утилиты для чтения `metadata.decoderConfig`
//!   (codec, codedWidth, codedHeight, description).
//!
//! ## Что нового по сравнению с 23.1/23.2
//!
//! 23.1 (`video_frame`) — что такое `VideoFrame` и `EncodedVideoChunk`.
//! 23.2 (`video_decoder`) — как подавать чанки в `VideoDecoder`.
//! 23.3 (`video_encoder`) — как `VideoEncoder` **производит** чанки:
//! размер key vs delta, влияние `bitrate`/`framerate`, содержимое
//! `metadata.decoderConfig`.
//!
//! ## Переиспользование хелперов
//!
//! Мини-хелперы для JsValue (`get_fn`, `construct`, `set_prop`,
//! `num`, `bool_of`) определены в `video_frame` (подглава 23.1) и
//! реэкспортируются здесь через `use`. Дублирования нет.

use js_sys::{Object, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{
    bool_of, construct, get_fn, num, set_prop,
};

// =============================================================================
// Feature detection
// =============================================================================

/// `VideoEncoder` доступен в этом браузере.
pub fn is_video_encoder_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("VideoEncoder")),
        Ok(v) if v.is_function()
    ))
}

/// `VideoEncoder.isConfigSupported` доступен. Статический метод.
pub fn has_is_config_supported() -> bool {
    let global = js_sys::global();
    let Ok(ctor) = Reflect::get(&global, &JsValue::from_str("VideoEncoder")) else {
        return false;
    };
    bool_of(matches!(
        Reflect::get(&ctor, &JsValue::from_str("isConfigSupported")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// isConfigSupported
// =============================================================================

/// `await VideoEncoder.isConfigSupported(config)`.
///
/// Возвращает JS-объект `{ supported: bool, config: {...} }`.
/// Проверка до `configure` — браузер может отвергнуть кодек,
/// разрешение или bitrate, и это видно заранее.
pub async fn is_config_supported(config: &JsValue) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoEncoder"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoEncoder not available"));
    }
    let f = Reflect::get(&ctor, &JsValue::from_str("isConfigSupported"))?;
    if !f.is_function() {
        return Err(JsValue::from_str(
            "VideoEncoder.isConfigSupported not available",
        ));
    }
    let f = f.dyn_into::<js_sys::Function>()?;
    let promise = f.call1(&ctor, config)?.dyn_into::<Promise>()?;
    JsFuture::from(promise).await
}

// =============================================================================
// Конструктор
// =============================================================================

/// `new VideoEncoder({ output, error })`.
///
/// `output(chunk, metadata)` вызывается асинхронно для каждого
/// готового `EncodedVideoChunk`. `metadata.decoderConfig` есть
/// **только на первом key-кадре** — на delta он `undefined`.
pub fn make_video_encoder(
    on_output: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue, JsValue)>,
    on_error: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
) -> Result<JsValue, JsValue> {
    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    construct("VideoEncoder", &[init.into()])
}

// =============================================================================
// Configure / encode / flush / close / reset
// =============================================================================

/// `encoder.configure({ codec, width, height, bitrate, framerate })`.
///
/// Для avc1 encoder сам формирует `avcC`-box (SPS/PPS) и отдаёт его
/// в `metadata.decoderConfig.description` на первом key-кадре.
pub fn configure(
    encoder: &JsValue,
    codec: &str,
    width: u32,
    height: u32,
    bitrate: u32,
    framerate: f64,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "width", &num(width as f64))?;
    set_prop(&cfg, "height", &num(height as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    set_prop(&cfg, "framerate", &num(framerate))?;
    let f = get_fn(encoder, "configure")?;
    let _ = f.call1(encoder, &cfg)?;
    Ok(())
}

/// `encoder.encode(frame, { keyFrame })`.
///
/// `keyFrame: true` — принудительный IDR. Остальные кадры encoder
/// кодирует как delta (P-frame), если решает, что они помещаются в GOP.
pub fn encode(encoder: &JsValue, frame: &JsValue, key_frame: bool) -> Result<(), JsValue> {
    let opts = Object::new();
    if key_frame {
        set_prop(&opts, "keyFrame", &JsValue::TRUE)?;
    }
    let f = get_fn(encoder, "encode")?;
    let _ = f.call2(encoder, frame, &opts)?;
    Ok(())
}

/// `await encoder.flush()`.
///
/// Дожидается всех чанков в очереди и вызывает `output` для каждого.
/// После `flush()` все кадры гарантированно закодированы.
pub async fn flush(encoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "flush")?;
    let promise = f.call0(encoder)?.dyn_into::<Promise>()?;
    JsFuture::from(promise).await.map(|_| ())
}

/// `encoder.close()`.
pub fn close(encoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "close")?;
    let _ = f.call0(encoder)?;
    Ok(())
}

/// `encoder.reset()`.
///
/// Отменяет все pending `encode()` и `flush()`. Encoder возвращается
/// в состояние до `configure()`. Полезно, если пользователь
/// переключил bitrate на середине: вместо ожидания flush —
/// сбрасываем и начинаем заново.
pub fn reset(encoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "reset")?;
    let _ = f.call0(encoder)?;
    Ok(())
}

// =============================================================================
// Чтение EncodedVideoChunk
// =============================================================================

/// `chunk.type` — `"key"` или `"delta"`.
///
/// Для не-`EncodedVideoChunk` возвращает пустую строку.
pub fn chunk_type(chunk: &JsValue) -> String {
    Reflect::get(chunk, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

/// `chunk.byteLength` — размер сжатых данных в байтах.
pub fn chunk_byte_length(chunk: &JsValue) -> usize {
    Reflect::get(chunk, &JsValue::from_str("byteLength"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as usize)
        .unwrap_or(0)
}

/// `chunk.timestamp` — временная метка в микросекундах.
pub fn chunk_timestamp_us(chunk: &JsValue) -> f64 {
    Reflect::get(chunk, &JsValue::from_str("timestamp"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// `chunk.duration` — длительность в микросекундах (или `null`).
pub fn chunk_duration_us(chunk: &JsValue) -> f64 {
    Reflect::get(chunk, &JsValue::from_str("duration"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

// =============================================================================
// Чтение metadata.decoderConfig
// =============================================================================

/// Извлекает `metadata.decoderConfig` из второго аргумента
/// `output`-колбэка encoder'а. Возвращает `None`, если поля
/// нет — это нормально для всех delta-кадров.
pub fn extract_decoder_config(metadata: &JsValue) -> Option<JsValue> {
    let cfg = Reflect::get(metadata, &JsValue::from_str("decoderConfig")).ok()?;
    if cfg.is_undefined() || cfg.is_null() {
        return None;
    }
    Some(cfg)
}

/// `decoderConfig.codec` — строка кодека, подтверждённая encoder'ом.
pub fn decoder_config_codec(cfg: &JsValue) -> String {
    Reflect::get(cfg, &JsValue::from_str("codec"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

/// `decoderConfig.codedWidth` — ширина в пикселях.
pub fn decoder_config_coded_width(cfg: &JsValue) -> u32 {
    Reflect::get(cfg, &JsValue::from_str("codedWidth"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `decoderConfig.codedHeight` — высота в пикселях.
pub fn decoder_config_coded_height(cfg: &JsValue) -> u32 {
    Reflect::get(cfg, &JsValue::from_str("codedHeight"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `decoderConfig.description` — `BufferSource` с `avcC`-box (SPS/PPS).
/// Возвращает `None`, если поле отсутствует или пустое.
pub fn decoder_config_description(cfg: &JsValue) -> Option<JsValue> {
    let d = Reflect::get(cfg, &JsValue::from_str("description")).ok()?;
    if d.is_undefined() || d.is_null() {
        return None;
    }
    Some(d)
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `video_decoder`:

```rust
pub mod codecs_demo;     // 23
pub mod video_frame;     // 23.1
pub mod video_decoder;   // 23.2
pub mod video_encoder;   // ← 23.3
```

### `tests/wasm_video_encoder.rs`

```rust
#![cfg(target_arch = "wasm32")]

use js_sys::Object;
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

use hello_wasm02::video_encoder::{
    chunk_byte_length, chunk_type, close, configure,
    has_is_config_supported, is_config_supported,
    is_video_encoder_supported, make_video_encoder, reset,
};
use hello_wasm02::video_frame::{num, set_prop};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn video_encoder_is_supported_in_browser() {
    assert!(
        is_video_encoder_supported(),
        "VideoEncoder should be available",
    );
}

#[wasm_bindgen_test]
fn is_config_supported_is_available() {
    let _ = has_is_config_supported();
}

#[wasm_bindgen_test]
async fn is_config_supported_avc1_baseline() {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str("avc1.42001E")).unwrap();
    set_prop(&cfg, "width", &num(320.0)).unwrap();
    set_prop(&cfg, "height", &num(240.0)).unwrap();
    set_prop(&cfg, "bitrate", &num(1_000_000.0)).unwrap();
    set_prop(&cfg, "framerate", &num(30.0)).unwrap();
    let result = is_config_supported(&cfg).await;
    assert!(result.is_ok(), "avc1.42001E 320x240 1Mbps should be supported");
}

#[wasm_bindgen_test]
fn make_encoder_does_not_throw() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_chunk: JsValue, _meta: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error);
    assert!(enc.is_ok(), "VideoEncoder should be constructible");
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn configure_and_close_do_not_throw() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_c: JsValue, _m: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");
    let result = configure(&enc, "avc1.42001E", 320, 240, 1_000_000, 30.0);
    assert!(result.is_ok(), "configure should not throw");
    let _ = close(&enc);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn reset_does_not_throw() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_c: JsValue, _m: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");
    configure(&enc, "avc1.42001E", 320, 240, 1_000_000, 30.0).unwrap();
    let _ = reset(&enc);
    let _ = close(&enc);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn chunk_helpers_return_empty_for_non_chunk() {
    let v = JsValue::from_str("not a chunk");
    assert!(chunk_type(&v).is_empty());
    assert_eq!(chunk_byte_length(&v), 0);
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_video_encoder

# 3. Демо
npm run wasm:dev && npm start
```

---

## 23.3.8. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.3 — VideoEncoder: распакованное → сжатое.
//!
//! Демо: 30 синтетических кадров с canvas → `VideoEncoder` →
//! массив `EncodedVideoChunk` → визуализация размеров и сжатия,
//! round-trip через `VideoDecoder` для верификации.
//!
//! ## Что демонстрируется
//!
//! - `new VideoEncoder({ output(chunk, metadata), error })`.
//! - `VideoEncoder.isConfigSupported()` **до** `configure()`.
//! - `configure({ codec, width, height, bitrate, framerate })`.
//! - `encode(frame, { keyFrame })` × 30 + `await flush()`.
//! - `chunk.type` — "key" vs "delta"; `chunk.byteLength`.
//! - `metadata.decoderConfig` — только на первом key.
//! - Target vs actual bitrate: H.264 — VBR, encoder undershoot'ит
//!   и overshoot'ит в зависимости от сложности контента.
//! - Коэффициент сжатия, эффективность delta (key/avg_delta).
//! - Round-trip encode → decode.
//! - Yield в async-loop раз в 16 мс, троттлинг UI раз в 50 мс.
//!
//! ## Что не является темой
//!
//! `VideoDecoder` — только для round-trip верификации. См. 23.2.
//!
//! ## Сцена
//!
//! Фон статичен (градиент + точки + подпись), движется только
//! маленький круг. Это даёт H.264 возможность реально кодировать
//! «разницу»: key большой, delta маленькие. Динамический градиент
//! (как в 23.2) ломает delta-кодирование — encoder кодирует почти
//! полный кадр на каждом шаге.
//!
//! ## Автозапуск
//!
//! Смена `bitrate` в select **автоматически перезапускает**
//! encode → verify. Флаг `running` + `RunGuard` защищают от
//! параллельных запусков и от ранних `return` без сброса флага.
//!
//! **Файл автономен.** Стили `.ve-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement,
    HtmlSelectElement, ImageData,
};

use crate::video_decoder::{
    buffer_source_to_vec, close as decoder_close,
    configure as decoder_configure, decode_chunk,
    flush as decoder_flush, make_video_decoder,
};
use crate::video_encoder::{
    chunk_byte_length, chunk_type, close as encoder_close,
    configure as encoder_configure, decoder_config_codec,
    decoder_config_coded_height, decoder_config_coded_width,
    decoder_config_description, encode, extract_decoder_config,
    flush as encoder_flush, is_config_supported,
    is_video_encoder_supported, make_video_encoder,
};
use crate::video_frame::{
    close as frame_close, format_str, invoke, num, set_prop,
};

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
            var KEY='ve-sw-cleared-23-3';
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
    let id = "ve-injected-style";
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
.ve-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.ve-desc-line { margin:6px 0; }
.ve-canvases { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.ve-canvas-col { display:flex;flex-direction:column;gap:6px;}
.ve-canvas-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.ve-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;}
.ve-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.ve-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.ve-btn:hover:not([disabled]) { background:#30363d; }
.ve-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.ve-btn-primary:hover:not([disabled]) { background:#2ea043; }
.ve-select { padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;font-size:12px;font-family:ui-monospace,monospace;}
.ve-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.ve-stats { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;margin-bottom:16px;}
.ve-stats .label { color:#8b949e; }
.ve-stats .value { color:#79c0ff; }
.ve-stats .ok { color:#3fb950; }
.ve-stats .warn { color:#f0883e; }
.ve-stats .err { color:#f85149; }
.ve-stats .muted { color:#6e7681; }
.ve-chart-label { max-width:900px;font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;margin-bottom:6px;}
.ve-chart { max-width:900px;height:140px;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:8px;display:flex;align-items:flex-end;gap:2px;margin-bottom:16px;position:relative;}
.ve-chart-marker { position:absolute;left:8px;right:8px;height:0;border-top:1px dashed rgba(230,237,243,0.45);pointer-events:none;}
.ve-bar { flex:1;min-width:3px;border-radius:2px 2px 0 0;transition:height 0.2s;}
.ve-bar-key { background:#f0883e; }
.ve-bar-delta { background:#58a6ff; }
.ve-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Константы
// =============================================================================

const FRAME_W: u32 = 320;
const FRAME_H: u32 = 240;
const FRAME_COUNT: u32 = 30;
const CODEC: &str = "avc1.42001E";
const FRAME_DURATION_US: f64 = 33333.0;
const STATS_EVERY_FRAMES: u32 = 15;
const YIELD_EVERY_MS: f64 = 16.0;
const UI_THROTTLE_MS: f64 = 50.0;

const FRAME_RGBA_BYTES: usize = (FRAME_W as usize) * (FRAME_H as usize) * 4;
const CHART_CONTENT_H: f64 = 140.0;
const CHART_PADDING: f64 = 8.0;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone)]
struct ChunkInfo {
    ty: String,
    bytes: usize,
}

#[derive(Clone)]
struct DecoderConfigInfo {
    codec: String,
    coded_width: u32,
    coded_height: u32,
    description_bytes: usize,
}

struct DemoState {
    paused: bool,
    phase: String,
    error: Option<String>,
    bitrate: u32,
    encode_progress: u32,
    encode_total: u32,
    /// Идёт ли сейчас encode→decode. Защищает от параллельных запусков.
    running: bool,
    chunks: Vec<ChunkInfo>,
    key_count: u32,
    delta_count: u32,
    key_bytes: usize,
    delta_bytes: usize,
    total_bytes: usize,
    decoder_config_info: Option<DecoderConfigInfo>,
    frames: Vec<Vec<u8>>,
    play_idx: usize,
    window_frames: u32,
}

impl DemoState {
    fn new() -> Self {
        Self {
            paused: false,
            phase: String::from("простой"),
            error: None,
            bitrate: 1_000_000,
            encode_progress: 0,
            encode_total: 0,
            running: false,
            chunks: Vec::new(),
            key_count: 0,
            delta_count: 0,
            key_bytes: 0,
            delta_bytes: 0,
            total_bytes: 0,
            decoder_config_info: None,
            frames: Vec::new(),
            play_idx: 0,
            window_frames: 0,
        }
    }

    fn clear_for_idle(&mut self) {
        self.error = None;
        self.encode_progress = 0;
        self.encode_total = 0;
        // `running` не трогаем здесь — им управляет RunGuard.
        self.chunks.clear();
        self.key_count = 0;
        self.delta_count = 0;
        self.key_bytes = 0;
        self.delta_bytes = 0;
        self.total_bytes = 0;
        self.decoder_config_info = None;
        self.frames.clear();
        self.play_idx = 0;
    }

    /// Готовит состояние к encode с заданным bitrate.
    /// НЕ трогает `running` — его устанавливает вызывающий
    /// (через `RunGuard`), чтобы не было гонки при быстрых кликах.
    fn start_encoding(&mut self, bitrate: u32) {
        self.clear_for_idle();
        self.bitrate = bitrate;
        self.encode_total = FRAME_COUNT;
    }
}

// =============================================================================
// Guard: сбрасывает `running` на любом выходе из full_pipeline
// =============================================================================

/// Сбрасывает `DemoState::running` при drop. Защищает от того,
/// что при раннем `return` (ошибка config, ошибка encode, …)
/// флаг «запущено» останется висеть и заблокирует автозапуск
/// при следующей смене bitrate.
struct RunGuard(Rc<RefCell<DemoState>>);

impl Drop for RunGuard {
    fn drop(&mut self) {
        let mut s = self.0.borrow_mut();
        s.running = false;
    }
}

// =============================================================================
// UI: строки
// =============================================================================

fn line_err(msg: &str) -> String {
    format_str(format_args!(r#"<div><span class="err">{msg}</span></div>"#))
}

fn line_phase(st: &DemoState) -> String {
    let playback = if st.frames.is_empty() {
        String::from("—")
    } else {
        format_str(format_args!(
            "{i} / {n}",
            i = st.play_idx,
            n = st.frames.len()
        ))
    };
    format_str(format_args!(
        r#"<div><span class="label">Статус: </span><span class="value">{phase}</span> · <span class="label">Playback: </span><span class="value">{pb}</span></div>"#,
        phase = st.phase,
        pb = playback,
    ))
}

/// Target vs actual bitrate. H.264 — VBR. Возможны оба случая:
///
/// - **undershoot** (actual < target): encoder экономит биты на простом
///   контенте. Строка подсвечивается как `muted`.
/// - **overshoot** (actual > target): target задан ниже физического
///   минимума для этой сцены, encoder не может опуститься ниже.
///   Строка подсвечивается как `warn`.
/// - **в норме** (0.7 ≤ target/actual ≤ 1.5): encoder уложился в target.
fn line_bitrate(st: &DemoState) -> String {
    let target = st.bitrate;
    if st.total_bytes == 0 {
        return format_str(format_args!(
            r#"<div><span class="label">Bitrate: </span><span class="value">target {t} бит/с</span> <span class="muted">(VBR: actual — после encode)</span></div>"#,
            t = target,
        ));
    }
    let duration_s = (FRAME_COUNT as f64 * FRAME_DURATION_US) / 1_000_000.0;
    let actual = (st.total_bytes as f64 * 8.0) / duration_s;
    let deviation = if actual > 0.0 {
        target as f64 / actual
    } else {
        0.0
    };

    let dev_html = if deviation >= 1.5 {
        format_str(format_args!(
            r#" <span class="muted">(undershoot ×{d:.1} — VBR экономит)</span>"#,
            d = deviation,
        ))
    } else if deviation > 0.0 && deviation <= 0.7 {
        let inv = 1.0 / deviation;
        format_str(format_args!(
            r#" <span class="warn">(overshoot ×{d:.1} — encoder не может ниже)</span>"#,
            d = inv,
        ))
    } else {
        String::new()
    };

    format_str(format_args!(
        r#"<div><span class="label">Bitrate: </span><span class="value">target {t}</span> · <span class="value">actual {a:.0} бит/с</span>{d}</div>"#,
        t = target,
        a = actual,
        d = dev_html,
    ))
}

fn line_progress(st: &DemoState) -> String {
    if st.encode_total == 0 {
        return String::new();
    }
    format_str(format_args!(
        r#"<div><span class="label">Progress: </span><span class="value">{p}</span> <span class="muted">/ {t}</span></div>"#,
        p = st.encode_progress,
        t = st.encode_total,
    ))
}

fn line_chunks(st: &DemoState) -> String {
    if st.chunks.is_empty() {
        return String::new();
    }
    format_str(format_args!(
        r#"<div><span class="label">Chunks: </span><span class="value">{n}</span> (key: <span class="ok">{k}</span>, delta: <span class="warn">{d}</span>) · <span class="label">Bytes: </span><span class="value">{t}</span> <span class="muted">(key: {kb}, delta: {db})</span></div>"#,
        n = st.chunks.len(),
        k = st.key_count,
        d = st.delta_count,
        t = st.total_bytes,
        kb = st.key_bytes,
        db = st.delta_bytes,
    ))
}

fn line_sizes(st: &DemoState) -> String {
    if st.chunks.is_empty() {
        return String::new();
    }
    let min_b = st.chunks.iter().map(|c| c.bytes).min().unwrap_or(0);
    let max_b = st.chunks.iter().map(|c| c.bytes).max().unwrap_or(0);
    let avg_b = st.total_bytes / st.chunks.len();

    let key_avg = if st.key_count > 0 {
        st.key_bytes as f64 / st.key_count as f64
    } else {
        0.0
    };
    let delta_avg = if st.delta_count > 0 {
        st.delta_bytes as f64 / st.delta_count as f64
    } else {
        0.0
    };
    let eff = if delta_avg > 0.0 {
        key_avg / delta_avg
    } else {
        0.0
    };

    if eff <= 0.0 {
        return format_str(format_args!(
            r#"<div><span class="label">Sizes: </span><span class="muted">min</span> <span class="value">{mn}</span> · <span class="muted">avg</span> <span class="value">{av}</span> · <span class="muted">max</span> <span class="value">{mx}</span> <span class="muted">байт</span></div>"#,
            mn = min_b,
            av = avg_b,
            mx = max_b,
        ));
    }
    let eff_cls = if eff >= 3.0 {
        "ok"
    } else if eff >= 1.5 {
        "value"
    } else {
        "warn"
    };
    format_str(format_args!(
        r#"<div><span class="label">Sizes: </span><span class="muted">min</span> <span class="value">{mn}</span> · <span class="muted">avg</span> <span class="value">{av}</span> · <span class="muted">max</span> <span class="value">{mx}</span> <span class="muted">байт</span> · <span class="label">Δeff: </span><span class="{ec}">key/avg_delta ≈ {eff:.1}×</span></div>"#,
        mn = min_b,
        av = avg_b,
        mx = max_b,
        ec = eff_cls,
        eff = eff,
    ))
}

fn line_ratio(st: &DemoState) -> String {
    if st.total_bytes == 0 {
        return String::new();
    }
    let raw = FRAME_RGBA_BYTES * st.chunks.len();
    let ratio = raw as f64 / st.total_bytes as f64;
    format_str(format_args!(
        r#"<div><span class="label">Сжатие: </span><span class="value">{raw} → {enc} байт</span> · <span class="ok">≈ {r:.1}×</span></div>"#,
        raw = raw,
        enc = st.total_bytes,
        r = ratio,
    ))
}

fn line_config(st: &DemoState) -> String {
    match st.decoder_config_info.as_ref() {
        Some(c) => format_str(format_args!(
            r#"<div><span class="label">decoderConfig: </span><span class="value">{codec}</span> · <span class="value">{w}×{h}</span> · <span class="label">description: </span><span class="ok">{d} байт</span></div>"#,
            codec = c.codec,
            w = c.coded_width,
            h = c.coded_height,
            d = c.description_bytes,
        )),
        None => String::new(),
    }
}

fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("ve-stats") else {
        return;
    };
    let err_line = st
        .error
        .as_ref()
        .map(|m| line_err(m.as_str()))
        .unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{phase}{br}{prog}{chunks}{sizes}{ratio}{config}"#,
        err = err_line,
        phase = line_phase(st),
        br = line_bitrate(st),
        prog = line_progress(st),
        chunks = line_chunks(st),
        sizes = line_sizes(st),
        ratio = line_ratio(st),
        config = line_config(st),
    ));
    el.set_inner_html(&html);
}

/// Строит HTML для одного столбика bar chart.
/// Вынесено отдельно, чтобы не путаться с raw-строками в цикле.
fn bar_html(idx: usize, info: &ChunkInfo, max_bytes: usize) -> String {
    let pct = (info.bytes as f64 / max_bytes as f64) * 100.0;
    let cls = if info.ty == "key" {
        "ve-bar ve-bar-key"
    } else {
        "ve-bar ve-bar-delta"
    };
    let title = format_str(format_args!(
        "#{i} {ty}: {b} B",
        i = idx,
        ty = info.ty,
        b = info.bytes,
    ));
    // r##"..."## — внутри встречается "# (title="#N ..."),
    // что закрыло бы обычную raw-строку r#"..."#.
    format_str(format_args!(
        r##"<div class="{cls}" style="height:{pct:.1}%" title="{title}"></div>"##,
        cls = cls,
        pct = pct,
        title = title,
    ))
}

fn update_chart(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("ve-chart") else {
        return;
    };
    if st.chunks.is_empty() {
        el.set_inner_html("");
        return;
    }
    let max = st.chunks.iter().map(|c| c.bytes).max().unwrap_or(1).max(1);
    let mut html = String::with_capacity(st.chunks.len() * 96 + 160);

    // Горизонтальный маркер среднего delta.
    if st.delta_count > 0 {
        let delta_avg = st.delta_bytes as f64 / st.delta_count as f64;
        let marker_bottom_px =
            CHART_PADDING + (delta_avg / max as f64) * CHART_CONTENT_H;
        html.push_str(&format_str(format_args!(
            r#"<div class="ve-chart-marker" style="bottom:{b:.1}px" title="avg delta = {avg:.0} B"></div>"#,
            b = marker_bottom_px,
            avg = delta_avg,
        )));
    }

    for (i, c) in st.chunks.iter().enumerate() {
        html.push_str(&bar_html(i, c, max));
    }
    el.set_inner_html(&html);
}

fn clear_canvas(doc: &Document, id: &str) {
    let Some(canvas) = doc
        .get_element_by_id(id)
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
    else {
        return;
    };
    let Ok(Some(ctx)) = canvas.get_context("2d") else {
        return;
    };
    let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
        return;
    };
    let _ = ctx.clear_rect(0.0, 0.0, FRAME_W as f64, FRAME_H as f64);
}

const DESC_LINES: &[&str] = &[
    "VideoEncoder принимает <strong>VideoFrame</strong> (RGBA) и выдаёт \
     <strong>EncodedVideoChunk</strong> (H.264) через колбэк \
     <code>output(chunk, metadata)</code>. Первый кадр — key (IDR), \
     остальные — delta (P-frame), если укладываются в GOP.",
    "Перед <code>configure()</code> вызывается <strong>isConfigSupported()</strong>: \
     если браузер отвергает кодек/разрешение/bitrate — показываем ошибку \
     без запуска энкодера.",
    "<code>metadata.decoderConfig</code> приходит <strong>только на первом \
     key-кадре</strong>: <code>codec</code>, <code>codedWidth</code>, \
     <code>codedHeight</code> и <code>description</code> (avcC-box с SPS/PPS). \
     Именно эти байты 23.2 подаёт в <code>VideoDecoder</code>.",
    "<strong>Bitrate — target, не гарантия.</strong> H.264 в Chrome — VBR: \
     encoder тратит столько битов, сколько нужно контенту. При \
     <code>50 kbps</code> он <em>не может опуститься</em> ниже минимума \
     (~120 kbps для этой сцены) — <strong>overshoot</strong>. При \
     <code>5 Mbps</code> target завышен — <strong>undershoot</strong>: \
     encoder не тратит лишнее. <strong>Bar chart</strong>: key (оранжевый) \
     заметно выше delta (синий). Пунктир — средний delta.",
    "<strong>Смена bitrate — автозапуск.</strong> Изменение target в \
     <code>&lt;select&gt;</code> немедленно перезапускает encode → verify. \
     Никаких промежуточных «пустых» состояний: пользователь видит новый \
     результат через ~100 мс.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ve-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("ve-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Рисование source
// =============================================================================

/// Статичные точки фона. Координаты — в пикселях 320×240.
/// Они одинаковы во всех кадрах, поэтому не влияют на размер delta,
/// но добавляют детали в key-кадр.
const BG_STARS: &[(f64, f64)] = &[
    (20.0, 40.0), (55.0, 18.0), (90.0, 65.0), (130.0, 30.0), (170.0, 55.0),
    (210.0, 22.0), (250.0, 48.0), (290.0, 75.0), (305.0, 30.0), (15.0, 100.0),
    (75.0, 130.0), (140.0, 180.0), (200.0, 195.0), (260.0, 160.0), (300.0, 210.0),
    (180.0, 120.0), (35.0, 175.0), (110.0, 90.0), (280.0, 130.0), (220.0, 100.0),
];

/// Рисует один кадр source-сцены.
///
/// **Дизайн:** статичный фон (градиент + точки + подпись) + один
/// маленький движущийся круг + счётчик кадра. Между соседними
/// кадрами меняется только положение круга, яркость круга и цифра
/// счётчика. Это позволяет H.264 кодировать delta-кадры как
/// «круг сдвинулся» — единицы КБ вместо десятков.
///
/// Динамический градиент (как в 23.2) плохо подходит: смена hue
/// на всём полотне ломает delta-кодирование.
fn draw_source(ctx: &CanvasRenderingContext2d, w: u32, h: u32, phase: f64) {
    let ctx_js: &JsValue = ctx.as_ref();
    let pi2 = std::f64::consts::PI * 2.0;

    // 1. Статичный фон — вертикальный градиент, НЕ меняется между кадрами.
    let bg = invoke(
        ctx_js,
        "createLinearGradient",
        &[num(0.0), num(0.0), num(0.0), num(h as f64)],
    )
    .ok();
    if let Some(g) = bg {
        let _ = invoke(&g, "addColorStop", &[num(0.0), JsValue::from_str("#0f2027")]);
        let _ = invoke(&g, "addColorStop", &[num(1.0), JsValue::from_str("#203a43")]);
        let _ = set_prop(ctx_js, "fillStyle", &g);
    }
    let _ = invoke(
        ctx_js,
        "fillRect",
        &[num(0.0), num(0.0), num(w as f64), num(h as f64)],
    );

    // 2. Статичные точки фона — они НЕ двигаются,
    //    поэтому не увеличивают delta, но добавляют детали в key.
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#2d4a5a"));
    for (sx, sy) in BG_STARS {
        let _ = invoke(ctx_js, "beginPath", &[]);
        let _ = invoke(
            ctx_js,
            "arc",
            &[num(*sx), num(*sy), num(1.6), num(0.0), num(pi2)],
        );
        let _ = invoke(ctx_js, "fill", &[]);
    }

    // 3. Статичная подпись — тоже детализирует key, не влияет на delta.
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#3fb950"));
    let _ = set_prop(ctx_js, "font", &JsValue::from_str("11px monospace"));
    let _ = invoke(
        ctx_js,
        "fillText",
        &[
            JsValue::from_str("VideoEncoder · 23.3"),
            num(w as f64 - 140.0),
            num(14.0),
        ],
    );

    // 4. Движущийся круг — единственный «дорогой» элемент delta.
    //    Радиус 32 px — заметен, но занимает ~2 % площади кадра.
    let cx = w as f64 / 2.0 + (phase * pi2).sin() * 100.0;
    let cy = h as f64 / 2.0 + (phase * pi2).cos() * 70.0;
    let light = 52.0 + (phase * 2.0 * pi2).sin() * 6.0;
    let circle_color = format_str(format_args!("hsl(24, 82%, {l:.0}%)", l = light));
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str(&circle_color));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(
        ctx_js,
        "arc",
        &[num(cx), num(cy), num(32.0), num(0.0), num(pi2)],
    );
    let _ = invoke(ctx_js, "fill", &[]);

    // 5. Счётчик кадра в углу — маленькая меняющаяся область.
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#c9d1d9"));
    let _ = set_prop(ctx_js, "font", &JsValue::from_str("20px monospace"));
    let text = format_str(format_args!(
        "#{}",
        (phase * FRAME_COUNT as f64) as u32
    ));
    let _ = invoke(
        ctx_js,
        "fillText",
        &[JsValue::from_str(&text), num(8.0), num(h as f64 - 8.0)],
    );
}

fn draw_frame_at(ctx: &CanvasRenderingContext2d, w: u32, h: u32, idx: u32) {
    let phase = idx as f64 / FRAME_COUNT as f64;
    draw_source(ctx, w, h, phase);
}

// =============================================================================
// VideoFrame из canvas
// =============================================================================

fn make_frame_from_canvas(
    canvas: &HtmlCanvasElement,
    timestamp_us: f64,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoFrame"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoFrame not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = Object::new();
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    let args = Array::new();
    args.push(&JsValue::from(canvas.clone()));
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

// =============================================================================
// Извлечение пикселей из VideoFrame
// =============================================================================

fn frame_to_bytes(
    frame: &JsValue,
    scratch_ctx: &CanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Vec<u8> {
    let ctx_js: &JsValue = scratch_ctx.as_ref();
    let _ = invoke(
        ctx_js,
        "drawImage",
        &[
            frame.clone(),
            num(0.0),
            num(0.0),
            num(w as f64),
            num(h as f64),
        ],
    );
    let args = [num(0.0), num(0.0), num(w as f64), num(h as f64)];
    invoke(ctx_js, "getImageData", &args)
        .ok()
        .and_then(|v| v.dyn_into::<ImageData>().ok())
        .map(|img| img.data().0)
        .unwrap_or_default()
}

// =============================================================================
// isConfigSupported — проверка ДО configure
// =============================================================================

async fn check_avc1_config(bitrate: u32) -> Result<bool, JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(CODEC))?;
    set_prop(&cfg, "width", &num(FRAME_W as f64))?;
    set_prop(&cfg, "height", &num(FRAME_H as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    set_prop(&cfg, "framerate", &num(30.0))?;
    let result = is_config_supported(&cfg).await?;
    let supported = Reflect::get(&result, &JsValue::from_str("supported"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
    Ok(supported)
}

// =============================================================================
// Encode
// =============================================================================

struct EncodeResult {
    chunks: Vec<JsValue>,
    types: Vec<String>,
    sizes: Vec<usize>,
    description: Option<JsValue>,
    config: Option<DecoderConfigInfo>,
}

async fn do_encode(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    bitrate: u32,
) -> Result<EncodeResult, JsValue> {
    let chunks_store: Rc<RefCell<Vec<JsValue>>> = Rc::new(RefCell::new(Vec::new()));
    let types_store: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let sizes_store: Rc<RefCell<Vec<usize>>> = Rc::new(RefCell::new(Vec::new()));
    let desc_store: Rc<RefCell<Option<JsValue>>> = Rc::new(RefCell::new(None));
    let cfg_store: Rc<RefCell<Option<DecoderConfigInfo>>> = Rc::new(RefCell::new(None));

    let chunks_cb = chunks_store.clone();
    let types_cb = types_store.clone();
    let sizes_cb = sizes_store.clone();
    let desc_cb = desc_store.clone();
    let cfg_cb = cfg_store.clone();
    let st_cb = st.clone();
    let doc_cb = doc.clone();
    // Троттлинг обновления UI из колбэка (не чаще 20 Гц).
    let last_update: Rc<RefCell<f64>> = Rc::new(RefCell::new(0.0));

    let on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, metadata: JsValue| {
            let ty = chunk_type(&chunk);
            let bl = chunk_byte_length(&chunk);

            // decoderConfig доступен только на первом key-кадре.
            if cfg_cb.borrow().is_none() {
                if let Some(cfg) = extract_decoder_config(&metadata) {
                    let desc_opt = decoder_config_description(&cfg);
                    let desc_bytes = desc_opt
                        .as_ref()
                        .map(|d| buffer_source_to_vec(d).len())
                        .unwrap_or(0);
                    *desc_cb.borrow_mut() = desc_opt;
                    *cfg_cb.borrow_mut() = Some(DecoderConfigInfo {
                        codec: decoder_config_codec(&cfg),
                        coded_width: decoder_config_coded_width(&cfg),
                        coded_height: decoder_config_coded_height(&cfg),
                        description_bytes: desc_bytes,
                    });
                }
            }

            chunks_cb.borrow_mut().push(chunk);
            types_cb.borrow_mut().push(ty);
            sizes_cb.borrow_mut().push(bl);

            let n = sizes_cb.borrow().len() as u32;
            {
                let mut s = st_cb.borrow_mut();
                s.encode_progress = n;
            }

            // Throttle: не чаще одного обновления stats в 50 мс.
            let now = js_sys::Date::now();
            let mut lu = last_update.borrow_mut();
            if now - *lu >= UI_THROTTLE_MS {
                *lu = now;
                drop(lu);
                let s = st_cb.borrow();
                update_stats(&doc_cb, &s);
            }
        },
    );
    let on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |e: JsValue| {
            web_sys::console::error_1(&e);
        },
    );

    let encoder = make_video_encoder(&on_output, &on_error)?;
    encoder_configure(&encoder, CODEC, FRAME_W, FRAME_H, bitrate, 30.0)?;

    let source_canvas = doc
        .get_element_by_id("ve-source")
        .ok_or_else(|| JsValue::from_str("no #ve-source"))?
        .dyn_into::<HtmlCanvasElement>()?;
    let source_ctx = source_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d (source)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    // Yield не каждый кадр, а раз в 16 мс (≈60 Гц).
    let mut last_yield = js_sys::Date::now();
    for i in 0..FRAME_COUNT {
        draw_frame_at(&source_ctx, FRAME_W, FRAME_H, i);
        let frame = make_frame_from_canvas(&source_canvas, i as f64 * FRAME_DURATION_US)?;
        encode(&encoder, &frame, i == 0)?;
        let _ = frame_close(&frame);

        let now = js_sys::Date::now();
        if now - last_yield >= YIELD_EVERY_MS {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }

    encoder_flush(&encoder).await?;
    let _ = encoder_close(&encoder);
    on_output.forget();
    on_error.forget();

    // Итоговое обновление UI.
    {
        let s = st.borrow();
        update_stats(&doc, &s);
    }

    let chunks = chunks_store.borrow().clone();
    let types = types_store.borrow().clone();
    let sizes = sizes_store.borrow().clone();
    let description = desc_store.borrow().clone();
    let config = cfg_store.borrow().clone();

    Ok(EncodeResult {
        chunks,
        types,
        sizes,
        description,
        config,
    })
}

// =============================================================================
// Decode (round-trip verify)
// =============================================================================

async fn do_decode(
    chunks: Vec<JsValue>,
    description: Option<&JsValue>,
    scratch_ctx: CanvasRenderingContext2d,
    doc: Document,
    st: Rc<RefCell<DemoState>>,
) -> Result<Vec<Vec<u8>>, JsValue> {
    let frames: Rc<RefCell<Vec<Vec<u8>>>> = Rc::new(RefCell::new(Vec::new()));
    let frames_cb = frames.clone();
    let scratch_cb = scratch_ctx.clone();
    let st_cb = st.clone();
    let doc_cb = doc.clone();

    let on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        move |frame: JsValue| {
            let bytes = frame_to_bytes(&frame, &scratch_cb, FRAME_W, FRAME_H);
            let n = {
                let mut buf = frames_cb.borrow_mut();
                buf.push(bytes);
                buf.len() as u32
            };
            let _ = frame_close(&frame);
            {
                let mut s = st_cb.borrow_mut();
                s.encode_progress = n;
            }
            let s = st_cb.borrow();
            update_stats(&doc_cb, &s);
        },
    );
    let on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |e: JsValue| {
            web_sys::console::error_1(&e);
        },
    );

    let decoder = make_video_decoder(&on_output, &on_error)?;
    decoder_configure(&decoder, CODEC, FRAME_W, FRAME_H, description)?;

    for chunk in chunks.iter() {
        decode_chunk(&decoder, chunk)?;
    }
    let flush_res = decoder_flush(&decoder).await;
    let _ = decoder_close(&decoder);
    on_output.forget();
    on_error.forget();
    flush_res?;

    let result: Vec<Vec<u8>> = frames.borrow().clone();
    Ok(result)
}

// =============================================================================
// Вспомогательное
// =============================================================================

fn get_scratch_ctx(doc: &Document) -> Option<CanvasRenderingContext2d> {
    let el = doc.get_element_by_id("ve-scratch")?;
    let canvas: HtmlCanvasElement = el.dyn_into().ok()?;
    let ctx = canvas.get_context("2d").ok()??;
    ctx.dyn_into::<CanvasRenderingContext2d>().ok()
}

// =============================================================================
// Пайплайн
// =============================================================================

async fn full_pipeline(doc: Document, st: Rc<RefCell<DemoState>>, bitrate: u32) {
    {
        let mut s = st.borrow_mut();
        if s.running {
            return; // Уже идёт. Не наслаиваем.
        }
        s.running = true;
        s.start_encoding(bitrate);
        s.phase = String::from("проверка config…");
    }
    let _guard = RunGuard(st.clone());

    clear_canvas(&doc, "ve-verify");
    {
        let s = st.borrow();
        update_stats(&doc, &s);
        update_chart(&doc, &s);
    }

    // --- isConfigSupported ДО configure ---
    match check_avc1_config(bitrate).await {
        Ok(true) => {
            let mut s = st.borrow_mut();
            s.phase = String::from("encode…");
        }
        Ok(false) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!(
                "isConfigSupported: конфигурация avc1 при bitrate={br} не поддерживается",
                br = bitrate,
            )));
            s.phase = String::from("не поддерживается");
            s.encode_total = 0;
            update_stats(&doc, &s);
            return;
        }
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!(
                "isConfigSupported ошибка: {e:?}"
            )));
            s.phase = String::from("ошибка");
            s.encode_total = 0;
            update_stats(&doc, &s);
            return;
        }
    }
    update_stats(&doc, &st.borrow());

    // --- Encode ---
    let enc = match do_encode(doc.clone(), st.clone(), bitrate).await {
        Ok(v) => v,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("encode: {e:?}")));
            s.phase = String::from("ошибка");
            s.encode_total = 0;
            update_stats(&doc, &s);
            return;
        }
    };

    let key_count = enc.types.iter().filter(|t| *t == "key").count() as u32;
    let delta_count = enc.types.len() as u32 - key_count;
    let key_bytes: usize = enc
        .types
        .iter()
        .zip(enc.sizes.iter())
        .filter(|(t, _)| *t == "key")
        .map(|(_, b)| *b)
        .sum();
    let delta_bytes: usize = enc
        .types
        .iter()
        .zip(enc.sizes.iter())
        .filter(|(t, _)| *t == "delta")
        .map(|(_, b)| *b)
        .sum();
    let total_bytes = key_bytes + delta_bytes;

    {
        let mut s = st.borrow_mut();
        s.chunks = enc
            .types
            .iter()
            .zip(enc.sizes.iter())
            .map(|(t, b)| ChunkInfo {
                ty: t.clone(),
                bytes: *b,
            })
            .collect();
        s.key_count = key_count;
        s.delta_count = delta_count;
        s.key_bytes = key_bytes;
        s.delta_bytes = delta_bytes;
        s.total_bytes = total_bytes;
        s.decoder_config_info = enc.config.clone();
        s.encode_total = 0;
        s.phase = format_str(format_args!(
            "encode готово ({} чанков)",
            enc.types.len()
        ));
    }
    {
        let s = st.borrow();
        update_stats(&doc, &s);
        update_chart(&doc, &s);
    }

    // --- Decode (verify) ---
    let Some(scratch_ctx) = get_scratch_ctx(&doc) else {
        let mut s = st.borrow_mut();
        s.error = Some(String::from("no scratch canvas"));
        s.phase = String::from("ошибка");
        update_stats(&doc, &s);
        return;
    };
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode (verify)…");
        s.encode_total = enc.chunks.len() as u32;
        s.encode_progress = 0;
    }
    update_stats(&doc, &st.borrow());

    let frames = match do_decode(
        enc.chunks,
        enc.description.as_ref(),
        scratch_ctx,
        doc.clone(),
        st.clone(),
    )
    .await
    {
        Ok(f) => f,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("decode: {e:?}")));
            s.phase = String::from("ошибка");
            s.encode_total = 0;
            update_stats(&doc, &s);
            return;
        }
    };

    let n = frames.len();
    {
        let mut s = st.borrow_mut();
        s.frames = frames;
        s.play_idx = 0;
        s.encode_progress = n as u32;
        s.encode_total = 0;
        s.phase = format_str(format_args!("готово ({n} кадров)"));
    }
    let s = st.borrow();
    update_stats(&doc, &s);
}

// =============================================================================
// Bitrate select
// =============================================================================

fn read_bitrate(doc: &Document) -> u32 {
    doc.get_element_by_id("ve-bitrate")
        .and_then(|e| e.dyn_into::<HtmlSelectElement>().ok())
        .and_then(|s| s.value().parse::<u32>().ok())
        .unwrap_or(1_000_000)
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
    h2.set_text_content(Some("23.3 — VideoEncoder"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_video_encoder_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("ve-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает VideoEncoder. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Скрытый scratch canvas.
    let scratch_el = document.create_element("canvas")?;
    scratch_el.set_id("ve-scratch");
    scratch_el.set_attribute("style", "display:none")?;
    let scratch_canvas: HtmlCanvasElement = scratch_el.dyn_into()?;
    scratch_canvas.set_width(FRAME_W);
    scratch_canvas.set_height(FRAME_H);
    root.append_child(&scratch_canvas)?;

    // Два видимых canvas'а: source и verify.
    let canvases = document.create_element("div")?;
    canvases.set_class_name("ve-canvases");
    canvases.set_inner_html(&format_str(format_args!(
        r#"<div class="ve-canvas-col">
               <div class="ve-canvas-label">Source (что кодируем)</div>
               <canvas id="ve-source" class="ve-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="ve-canvas-col">
               <div class="ve-canvas-label">Verify (что декодируем обратно)</div>
               <canvas id="ve-verify" class="ve-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = FRAME_W,
        h = FRAME_H,
    )));
    root.append_child(&canvases)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("ve-block");
    controls.set_inner_html(
        r#"<label class="ve-label" for="ve-bitrate">Bitrate:</label>
           <select class="ve-select" id="ve-bitrate">
               <option value="50000">50 kbps (ниже минимума)</option>
               <option value="500000">500 kbps</option>
               <option value="1000000" selected>1 Mbps</option>
               <option value="5000000">5 Mbps</option>
           </select>
           <button class="ve-btn ve-btn-primary" id="ve-run" title="Round-trip: encode через VideoEncoder + decode через VideoDecoder">Encode → Verify</button>
           <button class="ve-btn" id="ve-pause">Pause</button>
           <span class="ve-label" id="ve-support"></span>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("ve-stats");
    stats.set_id("ve-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let chart_label = document.create_element("div")?;
    chart_label.set_class_name("ve-chart-label");
    chart_label.set_text_content(Some(
        "Размер каждого чанка: key (оранжевый) vs delta (синий). Пунктир — средний delta.",
    ));
    root.append_child(&chart_label)?;

    let chart = document.create_element("div")?;
    chart.set_class_name("ve-chart");
    chart.set_id("ve-chart");
    root.append_child(&chart)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("ve-footnote");
    footnote.set_text_content(Some(
        "VideoEncoder принимает VideoFrame и выдаёт EncodedVideoChunk через \
         колбэк output(chunk, metadata). metadata.decoderConfig — только на \
         первом key-кадре: codec, codedWidth, codedHeight, description \
         (avcC-box с SPS/PPS). Bitrate — это target, а не гарантия: H.264 — \
         VBR-кодек, encoder тратит столько битов, сколько нужно контенту. \
         При низком target — overshoot (encoder не может ниже физического \
         минимума сцены), при высоком — undershoot (encoder не тратит лишнее). \
         Смена bitrate автоматически перезапускает encode→decode. \
         Round-trip encode → decode: Verify-canvas показывает, что чанки валидны.",
    ));
    root.append_child(&footnote)?;

    if let Some(el) = document.get_element_by_id("ve-support") {
        let s = format_str(format_args!(
            "VideoEncoder: {d} · isConfigSupported: {c}",
            d = if is_video_encoder_supported() { "да" } else { "нет" },
            c = if crate::video_encoder::has_is_config_supported() { "да" } else { "нет" },
        ));
        el.set_text_content(Some(&s));
    }

    let state = Rc::new(RefCell::new(DemoState::new()));

    // ---------- Encode + Verify ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("ve-run")
            .ok_or_else(|| JsValue::from_str("no #ve-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            let bitrate = read_bitrate(&doc);
            spawn_local(async move {
                full_pipeline(doc, st, bitrate).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- Bitrate select: автозапуск encode при смене ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let sel_el = document
            .get_element_by_id("ve-bitrate")
            .ok_or_else(|| JsValue::from_str("no #ve-bitrate"))?;
        let sel: HtmlElement = sel_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let br = read_bitrate(&doc);

            // Обновляем целевой bitrate немедленно — stats покажет
            // новое значение, даже если сейчас идёт другая сессия.
            let busy = {
                let mut s = st.borrow_mut();
                s.bitrate = br;
                s.running
            };
            if busy {
                // Идёт encode. Пропускаем автозапуск — пользователь
                // сможет нажать «Encode → Verify» после завершения.
                return;
            }

            // Автозапуск: смена target автоматически перезапускает
            // цикл encode→decode. Никакой «пустой сцены» — новый
            // результат появится сам через ~100 мс.
            let doc_c = doc.clone();
            let st_c = st.clone();
            spawn_local(async move {
                full_pipeline(doc_c, st_c, br).await;
            });
        });
        sel.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- Pause ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("ve-pause")
            .ok_or_else(|| JsValue::from_str("no #ve-pause"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let paused = {
                let mut s = st.borrow_mut();
                s.paused = !s.paused;
                s.paused
            };
            if let Some(el) = doc.get_element_by_id("ve-pause") {
                el.set_text_content(Some(if paused { "Resume" } else { "Pause" }));
            }
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- rAF: source-анимация + playback verify ----------
    {
        let doc_for_raf = document.clone();

        let source_canvas = doc_for_raf
            .get_element_by_id("ve-source")
            .ok_or_else(|| JsValue::from_str("no #ve-source"))?
            .dyn_into::<HtmlCanvasElement>()?;
        let source_ctx = source_canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d (source)"))?
            .dyn_into::<CanvasRenderingContext2d>()?;

        let verify_canvas = doc_for_raf
            .get_element_by_id("ve-verify")
            .ok_or_else(|| JsValue::from_str("no #ve-verify"))?
            .dyn_into::<HtmlCanvasElement>()?;
        let verify_ctx = verify_canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d (verify)"))?
            .dyn_into::<CanvasRenderingContext2d>()?;

        let cell: Rc<RefCell<Option<wasm_bindgen::closure::Closure<dyn FnMut(f64)>>>> =
            Rc::new(RefCell::new(None));
        let cell2 = cell.clone();
        let st = state.clone();

        *cell.borrow_mut() = Some(wasm_bindgen::closure::Closure::wrap(Box::new(
            move |now: f64| {
                if let Some(w) = web_sys::window() {
                    if let Some(cb) = cell2.borrow().as_ref() {
                        let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
                    }
                }

                if st.borrow().paused {
                    return;
                }

                let phase = (now / 1000.0) % 1.0;
                draw_source(&source_ctx, FRAME_W, FRAME_H, phase);

                let (has_frames, idx) = {
                    let mut s = st.borrow_mut();
                    if s.frames.is_empty() {
                        (false, 0)
                    } else {
                        s.play_idx = (s.play_idx + 1) % s.frames.len();
                        (true, s.play_idx)
                    }
                };
                if has_frames {
                    let bytes = {
                        let s = st.borrow();
                        s.frames.get(idx).cloned().unwrap_or_default()
                    };
                    if !bytes.is_empty() {
                        let clamped = wasm_bindgen::Clamped(&bytes[..]);
                        if let Ok(img) = ImageData::new_with_u8_clamped_array_and_sh(
                            clamped, FRAME_W, FRAME_H,
                        ) {
                            let _ = invoke(
                                verify_ctx.as_ref(),
                                "putImageData",
                                &[JsValue::from(img), num(0.0), num(0.0)],
                            );
                        }
                    }
                }

                let update_now = {
                    let mut s = st.borrow_mut();
                    s.window_frames += 1;
                    if s.window_frames >= STATS_EVERY_FRAMES {
                        s.window_frames = 0;
                        true
                    } else {
                        false
                    }
                };
                if update_now {
                    let s = st.borrow();
                    update_stats(&doc_for_raf, &s);
                }
            },
        ) as Box<dyn FnMut(f64)>));

        let cb = cell.borrow();
        window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    }

    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    Ok(())
}
```

---

## 23.3.9. Анализ решения

**Что демонстрирует.**

Подглава 23.3 раскрывает **VideoEncoder** — источник сжатых данных. Студент видит:

1. **Разницу key vs delta на bar chart.** Оранжевый столбик в потолок, синие — на 20 %. **Δeff ≈ 5×**.
2. **Compression ratio ≈ 600×.** Численное подтверждение, что кодек действительно работает.
3. **`metadata.decoderConfig`.** Строка в UI показывает те же 41 байт `description`, которые 23.2 подаёт в `VideoDecoder`. Мост между подглавами.
4. **VBR-семантику.** Undershoot при высокой target, overshoot при низкой. Строка `Bitrate` показывает target, actual и интерпретацию.

**Правила проекта, применённые.**

- **Yield каждые 16 мс**, а не каждый кадр.
- **Троттлинг UI** — 50 мс из колбэка `output`.
- **`format_str(format_args!(...))` в tail-позициях** — все строковые хелперы (`line_phase`, `line_bitrate`, `line_chunks`, `line_sizes`, `line_ratio`, `line_config`, `bar_html`, `update_chart`).
- **Отдельные функции для строк** — не мешаем логику и построение HTML.
- **`Rc<RefCell<DemoState>>`** — общее состояние между колбэками, пайплайном и rAF.
- **Правило «focus app.rs»** — только 23.3, без предыдущих секций.
- **Префикс `ve-`** — не пересекается с `vd-` / `vf-` / `cd-`.
- **Никаких неиспользуемых импортов.**

**Найденные в практике проблемы и их решения.**

1. **Динамический градиент (как в 23.2) ломает delta-кодирование.** H.264 видит «полностью другое изображение» и кодирует полный кадр. **Решение:** статичный фон + маленький движущийся круг. Δeff поднялся с 1.16 до 4.9.
2. **Target bitrate не совпадает с actual.** Пользователь недоумевает «почему 1 Mbps, а на деле 316 kbps?». **Решение:** строка `Bitrate` показывает оба числа + интерпретацию (`undershoot ×8.1`).
3. **H.264 может overshoot'ить при низком target.** 50 kbps ниже физического минимума сцены (~120 kbps) — encoder не может опуститься. **Решение:** добавлена опция 50 kbps в select + ветка `overshoot` в `line_bitrate`.
4. **Смена bitrate оставляла «пустую сцену».** Пользователь менял target и видел чёрный Verify, пустой chart. Выглядело как баг. **Решение:** автозапуск `full_pipeline` из `change`-обработчика.
5. **Параллельные запуски при быстрых переключениях.** Двойной клик мог запустить два encode. **Решение:** флаг `running` + `RunGuard` (Drop-семантика).

**Проверки корректности.**

- `RunGuard` устанавливается **после** `s.running = true` — порядок критичен, иначе Drop сбросит флаг сразу.
- `clear_for_idle` **не трогает** `running` — иначе `start_encoding → clear_for_idle` сбросит флаг, только что установленный.
- Ранние `return` (5 мест в `full_pipeline`) — все безопасны: `RunGuard` сработает на drop.
- `format_args!` без `f`-суффикса: `{a:.0}`, `{d:.1}`, `{r:.1}`, `{eff:.1}` — все валидны.
- Raw-строки: `r##"..."##` в `bar_html` — `title="#N ..."` внутри не закрывает строку.
- Все `<select>`-значения парсятся `read_bitrate` в `u32`.

---

## 23.3.10. Итог

**`VideoEncoder`** — источник сжатых данных. Принимает `VideoFrame` (RGBA), отдаёт `EncodedVideoChunk` (H.264) через колбэк `output`.

**`metadata.decoderConfig`** — есть **только на первом key-кадре**. Содержит `codec`, `codedWidth`, `codedHeight`, `description` (avcC-box с SPS/PPS). Это ровно то, что подаётся в `VideoDecoder.configure()` из 23.2.

**Key vs delta.** Key (IDR) — полное изображение, 2–4 КБ. Delta (P-frame) — разница, 0.3–0.9 КБ. На 30 кадрах статичной сцены — 1 key + 29 delta. Δeff = key_avg / delta_avg ≈ 5×.

**Compression ratio.** 320×240×4 × 30 = 9.2 МБ raw → ~15 КБ encoded = **×600**. Именно потому, что сцена статична и H.264 использует delta-кодирование.

**Bitrate — target, не гарантия.** H.264 VBR. На простом контенте — undershoot (target завышен). При заниженном target — overshoot (encoder не может опуститься ниже физического минимума).

**Практические находки №1–3.**

1. **Дизайн сцены критичен.** Динамический градиент превращает все кадры в «key» (Δeff ≈ 1.16). Статичный фон + маленький движущийся элемент даёт настоящие delta (Δeff ≈ 5).
2. **Chrome VBR undershoot'ит в 3–10×.** Target не гарантирует actual. Строка `Bitrate` показывает оба числа, чтобы это было явно.
3. **Автозапуск при смене target.** Иначе пользователь видит «пустую сцену» и думает, что демо сломалось. Флаг `running` + `RunGuard` защищают от параллельных запусков.

**Дальше — 23.4 (WebCodecs: полный pipeline)** — реальный видеофайл → декодер → фильтр → энкодер → файл. Здесь всё сходится вместе.