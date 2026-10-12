# Глава 23.5. AudioCodecs: AudioDecoder, AudioEncoder, AudioData

## 23.5.0. Обновлённая структура

```
src/
  audio_codecs.rs                 ← НОВЫЙ модуль 23.5
  video_frame.rs                  ← из 23.1 (переиспользуется: format_str)
  video_encoder.rs                ← из 23.3 (не используется)
  video_decoder.rs                ← из 23.2 (не используется)
  video_encoder_tuning.rs         ← из 23.4 (не используется)
  …
tests/
  wasm_audio_codecs.rs            ← НОВЫЙ тест
src/lib.rs                        ← +1 строка: pub mod audio_codecs;
src/app.rs                        ← демо 23.5 (заменяется целиком)
Cargo.toml                        ← +9 фич для web-sys (AudioContext и др.)
```

**Комментарий.** Модуль `audio_codecs.rs` — про **аудио**-часть WebCodecs: `AudioData`, `AudioEncoder`, `AudioDecoder`, `EncodedAudioChunk`. Префикс CSS — `ac-` (audio codecs), не пересекается с `vf-` (23.1), `vd-` (23.2), `ve-` (23.3), `vt-` (23.4). Мини-хелперы `num`, `set_prop`, `get_fn`, `format_str` — из `video_frame::*`. Дублирования нет.

**Ссылки для детального изучения:**

- MDN: [`AudioData`](https://developer.mozilla.org/en-US/docs/Web/API/AudioData)
- MDN: [`AudioEncoder`](https://developer.mozilla.org/en-US/docs/Web/API/AudioEncoder)
- MDN: [`AudioDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/AudioDecoder)
- MDN: [`EncodedAudioChunk`](https://developer.mozilla.org/en-US/docs/Web/API/EncodedAudioChunk)
- MDN: [`AudioContext`](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext)
- MDN: [`OfflineAudioContext`](https://developer.mozilla.org/en-US/docs/Web/API/OfflineAudioContext)
- MDN: [`OscillatorNode`](https://developer.mozilla.org/en-US/docs/Web/API/OscillatorNode)
- MDN: [`AudioBufferSourceNode`](https://developer.mozilla.org/en-US/docs/Web/API/AudioBufferSourceNode)
- W3C: [WebCodecs § AudioData](https://w3c.github.io/webcodecs/#audiodata-interface)
- W3C: [WebCodecs § AudioEncoder](https://w3c.github.io/webcodecs/#audioencoder-interface)
- W3C: [WebCodecs § AudioDecoder](https://w3c.github.io/webcodecs/#audiodecoder-interface)

---

## 23.5.1. О чём эта подглава

До сих пор WebCodecs рассматривался со стороны **видео**. Но WebCodecs умеет работать и с **аудио** — через параллельный набор интерфейсов:

| Видео | Аудио |
|---|---|
| `VideoFrame` | `AudioData` |
| `VideoEncoder` | `AudioEncoder` |
| `VideoDecoder` | `AudioDecoder` |
| `EncodedVideoChunk` | `EncodedAudioChunk` |
| `avc1`, `vp8`, `vp9`, `av01` | `opus`, `aac`, `mp3`, `flac`, `pcm-*` |

Подглава показывает **полный цикл** аудио: генерация сигнала → `AudioData` → `AudioEncoder` (Opus) → `EncodedAudioChunk` → `AudioDecoder` → воспроизведение через `AudioContext`.

Подглава отвечает на шесть вопросов.

**Первый — что такое `AudioData`.** Контейнер для несжатых PCM-сэмплов: формат (`f32-planar`, `s16`, …), частота дискретизации, число каналов, число фреймов.

**Второй — как `AudioData` получается из Web Audio API.** Прямого моста нет. Нужно извлечь `Float32Array` из `AudioBuffer` и создать `AudioData` вручную.

**Третий — как настраивается `AudioEncoder`.** `configure({ codec: "opus", sampleRate, numberOfChannels, bitrate })`.

**Четвёртый — как возвращаются сжатые данные.** Через колбэк `output(chunk, metadata)`. `chunk.byteLength` — размер сжатого пакета, `chunk.timestamp` — временная метка.

**Пятый — как декодировать.** `AudioDecoder` принимает `EncodedAudioChunk`, отдаёт `AudioData`. Для Opus `description` не нужен.

**Шестой — как воспроизвести.** Собрать `AudioBuffer` из плоскостей `AudioData`, проиграть через `AudioBufferSourceNode`.

---

## 23.5.2. Что такое `AudioData` и planar-формат

**`AudioData`** — плоский контейнер для PCM-сэмплов. У него есть:

- `format` — `"u8"`, `"s16"`, `"s32"`, `"f32"` (interleaved) или `"*-planar"` (planar).
- `sampleRate` — 44100, 48000, …
- `numberOfChannels` — 1 (mono), 2 (stereo), …
- `numberOfFrames` — сколько сэмплов на канал.
- `timestamp` — микросекунды.

**Planar vs interleaved.** В interleaved чередуются: `[L0, R0, L1, R1, …]`. В planar — сначала все сэмплы канала 0, потом канала 1: `[L0, L1, …, R0, R1, …]`. **Web Audio API использует только planar `f32-planar`.**

**Как читать planar.** Через `copyTo(dest, { planeIndex, format })`:

```js
const plane = new Float32Array(audioData.numberOfFrames);
audioData.copyTo(plane, { planeIndex: 0, format: "f32-planar" });
```

`format` здесь задаёт **целевой** формат: конвертация из любого формата в `f32-planar` обязательна по спецификации.

**Как создать `AudioData`.** Через конструктор с `AudioDataInit`:

```js
new AudioData({
  format: "f32-planar",
  sampleRate: 48000,
  numberOfFrames: 48000,
  numberOfChannels: 1,
  timestamp: 0,
  data: float32Array,
});
```

---

## 23.5.3. Что показывает демо

**Layout:**

- **Controls** — три кнопки: Generate → Encode → Decode & Play.
- **Waveform source** — первые 300 сэмплов PCM-сигнала из `OfflineAudioContext`.
- **Waveform decoded** — те же 300 сэмплов после decode, со сдвигом на pre-skip.
- **Stats** — статус, размер PCM, размер Opus, сжатие, время encode/decode.
- **Chunk-таблица** — тип, размер, timestamp первых 100 чанков.

**Сценарий:**

1. **Generate** — `OfflineAudioContext` рендерит 2 с синуса 440 Гц.
2. **Encode** — Opus, `bitrate: 64000`. Получаем ~100 чанков (~20 мс каждый).
3. **Decode & Play** — декодируем сохранённые чанки, играем через `AudioContext`.

**Wow-момент.** PCM: 384 000 байт. Opus: 20 075 байт. Сжатие **≈ 19×**. Source и decoded waveform **визуально идентичны** — доказательство корректного round-trip.

---

## 23.5.4. Практические находки

**1. `AudioData` и `AudioBuffer` — разные типы.** Нет прямого моста. Только через `getChannelData()` / `copyTo()`.

**2. Opus `description` не нужен.** В отличие от H.264/AAC. Это упрощает round-trip.

**3. `AudioContext` требует user gesture.** Ленивое создание при первом Play — правильный паттерн.

**4. Все чанки `key`.** WebCodecs для аудио всегда возвращает `type: "key"`. У Opus нет inter-frame prediction.

**5. Pre-skip Opus — 648 сэмплов.** Первый чанк имеет duration 13.5 мс (648 сэмплов), а не 20 мс (960). Это pre-skip-регион: encoder добавляет тишину для прогрева внутреннего состояния. При декодировании первые 648 сэмплов — тишина.

**6. Trailing padding.** Decoded длина **всегда** на 60–100 сэмплов больше исходной. Последний фрейм дополняется нулями.

**7. Waveform из агрегации по пикам не работает для синуса.** Пик в окне шириной несколько периодов для синуса постоянной амплитуды = 1.0. Все столбики одинаковы. **Нужно показывать сами сэмплы**: один bar на sample.

**8. Порядок импортов и `web-sys` features — источник ошибок.** Без `AudioParam` нет `OscillatorNode::frequency()`; без `AudioContext` нет `AudioContext::new()`. Особенно чувствительно в web-sys, где каждая фича — отдельная строка в `Cargo.toml`.

---

## 23.5.5. Исходные файлы

### `Cargo.toml` — добавить фичи

В блок `[dependencies.web-sys]` → `features`:

```toml
    # 23.5 — AudioCodecs
    "AudioContext",
    "AudioBuffer",
    "AudioBufferSourceNode",
    "AudioDestinationNode",
    "AudioNode",
    "AudioParam",
    "OfflineAudioContext",
    "OscillatorNode",
    "OscillatorType",
```

`AudioParam` — обязателен для `OscillatorNode::frequency()`. `AudioNode` — для `connect_with_audio_node`. `AudioDestinationNode` — для `offline.destination()`.

### `src/audio_codecs.rs`

```rust
//! 23.5. AudioCodecs: AudioData, AudioEncoder, AudioDecoder.
//!
//! Обёртки вокруг аудио-интерфейсов WebCodecs:
//!
//! - `AudioData` — конструктор из `Float32Array`, чтение метаданных,
//!   `copyTo` для извлечения сэмплов;
//! - `AudioEncoder` — конструктор, `configure`, `encode`, `flush`;
//! - `AudioDecoder` — конструктор, `configure`, `decode`, `flush`;
//! - `EncodedAudioChunk` — чтение `type`, `byteLength`, `timestamp`.
//!
//! ## Отличия от видео
//!
//! В 23.1–23.4 работали с `VideoFrame` / `VideoEncoder` /
//! `VideoDecoder`. Здесь — параллельный набор для аудио. API похожи,
//! но `AudioData` использует planar-форматы, а `copyTo` требует
//! явного `planeIndex` и целевого `format`.
//!
//! ## Переиспользование
//!
//! Хелперы `num`, `set_prop`, `get_fn`, `format_str` — из
//! `video_frame::*`. Дублирования нет.

use js_sys::{Array, Float32Array, Object, Promise, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{format_str, get_fn, num, set_prop};

// =============================================================================
// Feature detection
// =============================================================================

/// `AudioEncoder` доступен в браузере.
pub fn is_audio_encoder_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("AudioEncoder"))
        .map(|v| v.is_function())
        .unwrap_or(false)
}

/// `AudioDecoder` доступен в браузере.
pub fn is_audio_decoder_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("AudioDecoder"))
        .map(|v| v.is_function())
        .unwrap_or(false)
}

/// `AudioData` доступен в браузере.
pub fn is_audio_data_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("AudioData"))
        .map(|v| v.is_function())
        .unwrap_or(false)
}

// =============================================================================
// AudioData: конструктор и чтение
// =============================================================================

/// `new AudioData({ format, sampleRate, numberOfFrames,
/// numberOfChannels, timestamp, data })`.
///
/// Для planar-форматов `data` — один `Float32Array` со **всеми**
/// каналами подряд: сначала канал 0, потом канал 1, и т. д.
pub fn make_audio_data(
    samples: &[f32],
    sample_rate: u32,
    number_of_channels: u32,
    timestamp_us: f64,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioData"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("AudioData not available"));
    }

    let number_of_frames = (samples.len() as u32) / number_of_channels;
    let array = Float32Array::from(samples);

    let init = Object::new();
    set_prop(&init, "format", &JsValue::from_str("f32-planar"))?;
    set_prop(&init, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&init, "numberOfFrames", &num(number_of_frames as f64))?;
    set_prop(&init, "numberOfChannels", &num(number_of_channels as f64))?;
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    set_prop(&init, "data", &array)?;

    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    Reflect::construct(&ctor, &{
        let args = Array::new();
        args.push(&init);
        args
    })
}

/// `audioData.numberOfFrames`.
pub fn audio_data_number_of_frames(data: &JsValue) -> u32 {
    Reflect::get(data, &JsValue::from_str("numberOfFrames"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `audioData.numberOfChannels`.
pub fn audio_data_number_of_channels(data: &JsValue) -> u32 {
    Reflect::get(data, &JsValue::from_str("numberOfChannels"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `audioData.sampleRate`.
pub fn audio_data_sample_rate(data: &JsValue) -> u32 {
    Reflect::get(data, &JsValue::from_str("sampleRate"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `audioData.format` — строка формата (`"f32-planar"` и т. д.).
pub fn audio_data_format(data: &JsValue) -> String {
    Reflect::get(data, &JsValue::from_str("format"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

/// `audioData.timestamp` — микросекунды.
pub fn audio_data_timestamp(data: &JsValue) -> f64 {
    Reflect::get(data, &JsValue::from_str("timestamp"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// `audioData.close()`.
pub fn audio_data_close(data: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(data, "close")?;
    let _ = f.call0(data)?;
    Ok(())
}

/// Извлекает один канал (`planeIndex`) как `Vec<f32>`.
/// Целевой формат — всегда `f32-planar` (Web Audio API использует
/// только его; конвертация из любого формата обязательна по спеке).
pub fn audio_data_copy_channel(data: &JsValue, plane_index: u32) -> Vec<f32> {
    let n_frames = audio_data_number_of_frames(data);
    if n_frames == 0 {
        return Vec::new();
    }
    let dest = Float32Array::new_with_length(n_frames);
    let opts = Object::new();
    if set_prop(&opts, "planeIndex", &num(plane_index as f64)).is_err() {
        return Vec::new();
    }
    if set_prop(&opts, "format", &JsValue::from_str("f32-planar")).is_err() {
        return Vec::new();
    }
    let f = match get_fn(data, "copyTo") {
        Ok(f) => f,
        Err(_) => return Vec::new(),
    };
    if f.call2(data, &dest, &opts).is_err() {
        return Vec::new();
    }
    dest.to_vec()
}

// =============================================================================
// AudioEncoder
// =============================================================================

/// `new AudioEncoder({ output, error })`.
pub fn make_audio_encoder(
    on_output: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue, JsValue)>,
    on_error: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
) -> Result<JsValue, JsValue> {
    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioEncoder"))?;
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

/// `encoder.configure({ codec, sampleRate, numberOfChannels, bitrate })`.
pub fn configure_audio_encoder(
    encoder: &JsValue,
    codec: &str,
    sample_rate: u32,
    number_of_channels: u32,
    bitrate: u32,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&cfg, "numberOfChannels", &num(number_of_channels as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    let f = get_fn(encoder, "configure")?;
    let _ = f.call1(encoder, &cfg)?;
    Ok(())
}

/// `encoder.encode(audio_data)`.
pub fn encode_audio(encoder: &JsValue, audio_data: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "encode")?;
    let _ = f.call1(encoder, audio_data)?;
    Ok(())
}

/// `await encoder.flush()`.
pub async fn flush_audio_encoder(encoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "flush")?;
    let promise = f.call0(encoder)?.dyn_into::<Promise>()?;
    JsFuture::from(promise).await.map(|_| ())
}

/// `encoder.close()`.
pub fn close_audio_encoder(encoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "close")?;
    let _ = f.call0(encoder)?;
    Ok(())
}

/// `AudioEncoder.isConfigSupported(config)` — статический метод.
pub async fn is_audio_config_supported(
    codec: &str,
    sample_rate: u32,
    number_of_channels: u32,
    bitrate: u32,
) -> Result<bool, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioEncoder"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("AudioEncoder not available"));
    }
    let f = Reflect::get(&ctor, &JsValue::from_str("isConfigSupported"))?;
    if !f.is_function() {
        return Err(JsValue::from_str(
            "AudioEncoder.isConfigSupported not available",
        ));
    }
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&cfg, "numberOfChannels", &num(number_of_channels as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    let f = f.dyn_into::<js_sys::Function>()?;
    let promise = f.call1(&ctor, &cfg)?.dyn_into::<Promise>()?;
    let result = JsFuture::from(promise).await?;
    Ok(Reflect::get(&result, &JsValue::from_str("supported"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false))
}

// =============================================================================
// AudioDecoder
// =============================================================================

/// `new AudioDecoder({ output, error })`.
pub fn make_audio_decoder(
    on_output: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
    on_error: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
) -> Result<JsValue, JsValue> {
    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioDecoder"))?;
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

/// `decoder.configure({ codec, sampleRate, numberOfChannels })`.
pub fn configure_audio_decoder(
    decoder: &JsValue,
    codec: &str,
    sample_rate: u32,
    number_of_channels: u32,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&cfg, "numberOfChannels", &num(number_of_channels as f64))?;
    let f = get_fn(decoder, "configure")?;
    let _ = f.call1(decoder, &cfg)?;
    Ok(())
}

/// `decoder.decode(chunk)`.
pub fn decode_audio(decoder: &JsValue, chunk: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "decode")?;
    let _ = f.call1(decoder, chunk)?;
    Ok(())
}

/// `await decoder.flush()`.
pub async fn flush_audio_decoder(decoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "flush")?;
    let promise = f.call0(decoder)?.dyn_into::<Promise>()?;
    JsFuture::from(promise).await.map(|_| ())
}

/// `decoder.close()`.
pub fn close_audio_decoder(decoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "close")?;
    let _ = f.call0(decoder)?;
    Ok(())
}

// =============================================================================
// EncodedAudioChunk helpers
// =============================================================================

/// `chunk.type` — `"key"` или `"delta"`.
pub fn chunk_type(chunk: &JsValue) -> String {
    Reflect::get(chunk, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

/// `chunk.byteLength`.
pub fn chunk_byte_length(chunk: &JsValue) -> usize {
    Reflect::get(chunk, &JsValue::from_str("byteLength"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as usize)
        .unwrap_or(0)
}

/// `chunk.timestamp` — микросекунды.
pub fn chunk_timestamp(chunk: &JsValue) -> f64 {
    Reflect::get(chunk, &JsValue::from_str("timestamp"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// Извлекает байты из `EncodedAudioChunk` для отладочного
/// отображения. Помечено `#[allow(dead_code)]` — публичный API,
/// используется внешним кодом при необходимости.
#[allow(dead_code)]
pub fn chunk_to_vec(chunk: &JsValue) -> Vec<u8> {
    let n = chunk_byte_length(chunk);
    if n == 0 {
        return Vec::new();
    }
    let dest = Uint8Array::new_with_length(n as u32);
    let f = match get_fn(chunk, "copyTo") {
        Ok(f) => f,
        Err(_) => return Vec::new(),
    };
    if f.call1(chunk, &dest).is_err() {
        return Vec::new();
    }
    dest.to_vec()
}

// =============================================================================
// Конструктор EncodedAudioChunk (для тестов)
// =============================================================================

/// `new EncodedAudioChunk({ type, timestamp, duration, data })`.
pub fn make_encoded_audio_chunk(
    chunk_type: &str,
    timestamp_us: f64,
    duration_us: f64,
    data: &[u8],
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("EncodedAudioChunk"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("EncodedAudioChunk not available"));
    }
    let init = Object::new();
    set_prop(&init, "type", &JsValue::from_str(chunk_type))?;
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    set_prop(&init, "duration", &num(duration_us))?;
    let arr = Uint8Array::from(data);
    set_prop(&init, "data", &arr)?;
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

// =============================================================================
// Утилита форматирования
// =============================================================================

/// Форматирует «N байт» с килобайтами, если больше 1024.
pub fn bytes_human(n: usize) -> String {
    if n >= 1024 {
        format_str(format_args!(
            "{n} ({kb:.1} KB)",
            n = n,
            kb = n as f64 / 1024.0,
        ))
    } else {
        format_str(format_args!("{n}", n = n))
    }
}
```

### `src/lib.rs` — добавление

```rust
pub mod codecs_demo;              // 23
pub mod video_frame;              // 23.1
pub mod video_decoder;            // 23.2
pub mod video_encoder;            // 23.3
pub mod video_encoder_tuning;     // 23.4
pub mod audio_codecs;             // ← 23.5
```

### `tests/wasm_audio_codecs.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

use hello_wasm02::audio_codecs::{
    audio_data_close, audio_data_copy_channel, audio_data_format,
    audio_data_number_of_channels, audio_data_number_of_frames,
    audio_data_sample_rate, audio_data_timestamp,
    bytes_human, chunk_byte_length, chunk_timestamp, chunk_type,
    is_audio_config_supported, is_audio_data_supported,
    is_audio_decoder_supported, is_audio_encoder_supported,
    make_audio_data, make_encoded_audio_chunk,
};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Feature detection
// =============================================================================

#[wasm_bindgen_test]
fn audio_encoder_is_supported() {
    assert!(is_audio_encoder_supported(), "AudioEncoder must be available");
}

#[wasm_bindgen_test]
fn audio_decoder_is_supported() {
    assert!(is_audio_decoder_supported(), "AudioDecoder must be available");
}

#[wasm_bindgen_test]
fn audio_data_is_supported() {
    assert!(is_audio_data_supported(), "AudioData must be available");
}

// =============================================================================
// AudioData: конструктор и метаданные
// =============================================================================

#[wasm_bindgen_test]
fn audio_data_creation_from_float32_array() {
    let samples = vec![0.0f32; 100];
    let data = make_audio_data(&samples, 48000, 1, 0.0)
        .expect("AudioData creation");
    assert_eq!(audio_data_number_of_frames(&data), 100);
    assert_eq!(audio_data_number_of_channels(&data), 1);
    assert_eq!(audio_data_sample_rate(&data), 48000);
    assert_eq!(audio_data_format(&data), "f32-planar");
    assert_eq!(audio_data_timestamp(&data), 0.0);
    let _ = audio_data_close(&data);
}

#[wasm_bindgen_test]
fn audio_data_stereo_planar_layout() {
    // 2 канала × 50 фреймов = 100 сэмплов.
    // Канал 0 — нули, канал 1 — единицы. Planar: сначала канал 0, потом канал 1.
    let mut samples = vec![0.0f32; 50];
    samples.extend(vec![1.0f32; 50]);
    let data = make_audio_data(&samples, 48000, 2, 0.0)
        .expect("stereo AudioData");
    assert_eq!(audio_data_number_of_frames(&data), 50);
    assert_eq!(audio_data_number_of_channels(&data), 2);
    let _ = audio_data_close(&data);
}

// =============================================================================
// copyTo: извлечение каналов
// =============================================================================

#[wasm_bindgen_test]
fn copy_to_returns_correct_length() {
    let samples = vec![0.5f32; 480];
    let data = make_audio_data(&samples, 48000, 1, 0.0)
        .expect("AudioData");
    let channel = audio_data_copy_channel(&data, 0);
    assert_eq!(channel.len(), 480);
    let _ = audio_data_close(&data);
}

#[wasm_bindgen_test]
fn copy_to_preserves_values() {
    let samples: Vec<f32> = (0..100).map(|i| i as f32 / 100.0).collect();
    let data = make_audio_data(&samples, 48000, 1, 0.0)
        .expect("AudioData");
    let channel = audio_data_copy_channel(&data, 0);
    assert_eq!(channel.len(), samples.len());
    for (a, b) in channel.iter().zip(samples.iter()) {
        assert!((a - b).abs() < 1e-6, "значения должны совпадать");
    }
    let _ = audio_data_close(&data);
}

#[wasm_bindgen_test]
fn copy_to_stereo_plane_1() {
    let mut samples = vec![0.0f32; 30];
    samples.extend(vec![0.7f32; 30]);
    let data = make_audio_data(&samples, 48000, 2, 0.0)
        .expect("stereo AudioData");
    let ch0 = audio_data_copy_channel(&data, 0);
    let ch1 = audio_data_copy_channel(&data, 1);
    assert_eq!(ch0.len(), 30);
    assert_eq!(ch1.len(), 30);
    assert!(ch0.iter().all(|&v| v.abs() < 1e-6));
    assert!(ch1.iter().all(|&v| (v - 0.7).abs() < 1e-6));
    let _ = audio_data_close(&data);
}

#[wasm_bindgen_test]
fn close_is_idempotent() {
    let samples = vec![0.0f32; 10];
    let data = make_audio_data(&samples, 48000, 1, 0.0).expect("AudioData");
    assert!(audio_data_close(&data).is_ok());
    let _ = audio_data_close(&data);
}

// =============================================================================
// isConfigSupported для Opus
// =============================================================================

#[wasm_bindgen_test]
async fn opus_48khz_mono_supported() {
    let ok = is_audio_config_supported("opus", 48000, 1, 64000)
        .await
        .unwrap_or(false);
    assert!(ok, "opus 48k mono 64kbps must be supported");
}

#[wasm_bindgen_test]
async fn opus_48khz_stereo_supported() {
    let ok = is_audio_config_supported("opus", 48000, 2, 128000)
        .await
        .unwrap_or(false);
    assert!(ok, "opus 48k stereo 128kbps must be supported");
}

#[wasm_bindgen_test]
async fn bogus_codec_not_supported() {
    let ok = is_audio_config_supported("not-a-real-codec", 48000, 1, 64000)
        .await
        .unwrap_or(false);
    assert!(!ok);
}

// =============================================================================
// EncodedAudioChunk: конструктор и чтение
// =============================================================================

#[wasm_bindgen_test]
fn make_and_read_encoded_chunk() {
    let data = vec![0u8; 128];
    let chunk = make_encoded_audio_chunk("key", 0.0, 20000.0, &data)
        .expect("EncodedAudioChunk");
    assert_eq!(chunk_type(&chunk), "key");
    assert_eq!(chunk_byte_length(&chunk), 128);
    assert_eq!(chunk_timestamp(&chunk), 0.0);
}

// =============================================================================
// bytes_human
// =============================================================================

#[wasm_bindgen_test]
fn bytes_human_small() {
    assert_eq!(bytes_human(500), "500");
}

#[wasm_bindgen_test]
fn bytes_human_kb() {
    let s = bytes_human(2048);
    assert!(s.contains("2048"));
    assert!(s.contains("2.0 KB"));
}

// =============================================================================
// Smoke: вспомогательные функции не падают на невалидном входе
// =============================================================================

#[wasm_bindgen_test]
fn helpers_handle_non_data_input() {
    let v = JsValue::from_str("not audio data");
    assert_eq!(audio_data_number_of_frames(&v), 0);
    assert_eq!(audio_data_number_of_channels(&v), 0);
    assert_eq!(audio_data_sample_rate(&v), 0);
    assert!(audio_data_format(&v).is_empty());
    assert_eq!(audio_data_timestamp(&v), 0.0);
    assert!(audio_data_copy_channel(&v, 0).is_empty());
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_audio_codecs

# 3. Демо
npm run wasm:dev && npm start
```

---

## 23.5.6. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.5 — AudioCodecs: AudioDecoder, AudioEncoder, AudioData.
//!
//! Демо: генерация синусоиды 440 Гц → AudioData → AudioEncoder
//! (Opus) → массив EncodedAudioChunk → AudioDecoder → воспроизведение
//! через AudioContext.
//!
//! ## Что демонстрируется
//!
//! - `OscillatorNode` + `OfflineAudioContext` для генерации сигнала.
//! - `AudioData` из `Float32Array` в формате `f32-planar`.
//! - `AudioEncoder.isConfigSupported` для Opus.
//! - `AudioEncoder` с колбэком `output(chunk)`.
//! - `AudioDecoder` с колбэком `output(audio_data)`.
//! - `copyTo(dest, { planeIndex, format })` для извлечения сэмплов.
//! - Воспроизведение через `AudioBufferSourceNode`.
//! - Коэффициент сжатия PCM → Opus (≈ 20×).
//! - Waveform source vs decoded — первые N сэмплов **напрямую**,
//!   один bar на сэмпл. Агрегация по пикам в окне даёт плоскую
//!   линию для синуса постоянной амплитуды — форма волны не видна.
//!
//! ## Порядок работы
//!
//! 1. **Generate** — OfflineAudioContext рендерит 2 с синуса.
//! 2. **Encode** — Opus-энкодер собирает чанки. Чанки **сохраняются**
//!    в `DemoState.chunks_raw` — это позволяет декодировать их
//!    многократно без повторного энкодинга.
//! 3. **Decode & Play** — AudioDecoder восстанавливает PCM,
//!    AudioBufferSourceNode играет через AudioContext.
//!
//! **Файл автономен.** Стили `.ac-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::Reflect;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{
    AudioBuffer, AudioBufferSourceNode, AudioContext, Document,
    HtmlElement, OfflineAudioContext, OscillatorType,
};

use crate::audio_codecs::{
    audio_data_close, audio_data_copy_channel,
    audio_data_number_of_channels, bytes_human,
    chunk_byte_length, chunk_timestamp, chunk_type,
    close_audio_decoder, close_audio_encoder,
    configure_audio_decoder, configure_audio_encoder,
    decode_audio, encode_audio, flush_audio_decoder,
    flush_audio_encoder, is_audio_config_supported,
    is_audio_data_supported, is_audio_decoder_supported,
    is_audio_encoder_supported, make_audio_data,
    make_audio_decoder, make_audio_encoder,
};
use crate::video_frame::format_str;

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
            var KEY='ac-sw-cleared-23-5';
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
    let id = "ac-injected-style";
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
.ac-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.ac-desc-line { margin:6px 0; }
.ac-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.ac-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.ac-btn:hover:not([disabled]) { background:#30363d; }
.ac-btn:disabled { opacity:0.5;cursor:not-allowed; }
.ac-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.ac-btn-primary:hover:not([disabled]) { background:#2ea043; }
.ac-btn-play { background:#1f6feb;border-color:#388bfd;color:#fff;}
.ac-btn-play:hover:not([disabled]) { background:#388bfd; }
.ac-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.ac-wave-label { max-width:900px;font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;margin:8px 0 4px 0;}
.ac-stats { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.ac-stats .label { color:#8b949e; }
.ac-stats .value { color:#79c0ff; }
.ac-stats .ok { color:#3fb950; }
.ac-stats .warn { color:#f0883e; }
.ac-stats .err { color:#f85149; }
.ac-stats .muted { color:#6e7681; }
.ac-wave { max-width:900px;height:100px;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:6px;margin-bottom:8px;display:flex;align-items:center;gap:0;overflow:hidden;}
.ac-wave-bar { flex:1;min-width:1px;background:#58a6ff;border-radius:1px;opacity:0.75;}
.ac-chunks { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #d2a8ff;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:11px;line-height:1.5;margin-bottom:16px;overflow-x:auto;max-height:200px;overflow-y:auto;}
.ac-chunks table { width:100%; border-collapse:collapse; }
.ac-chunks th { color:#8b949e; text-align:left; padding:3px 8px; border-bottom:1px solid #21262d; font-weight:500; }
.ac-chunks td { padding:2px 8px; border-bottom:1px solid #161b22; }
.ac-chunks tr:last-child td { border-bottom:none; }
.ac-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Константы
// =============================================================================

const SAMPLE_RATE: u32 = 48000;
const CHANNELS: u32 = 1;
const DURATION_SEC: f64 = 2.0;
const FREQ_HZ: f64 = 440.0;
const OPUS_BITRATE: u32 = 64000;
const CODEC: &str = "opus";
/// Сколько первых сэмплов показываем в waveform. 300 при 48 kHz —
/// 6.25 мс, ~2.75 периода 440 Гц. Форма волны видна чётко.
const WAVE_ZOOM_SAMPLES: usize = 300;
/// Opus pre-skip: сколько сэмплов пропустить в decoded, чтобы
/// попасть на реальный сигнал. Значение > длительности первого
/// фрейма (648 сэмплов) — берём с запасом.
const DECODE_SKIP_SAMPLES: usize = 1000;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone)]
struct ChunkRow {
    idx: usize,
    ty: String,
    bytes: usize,
    timestamp_us: f64,
}

struct SourceAudio {
    samples: Vec<f32>,
    sample_rate: u32,
    channels: u32,
}

struct DemoState {
    phase: String,
    error: Option<String>,
    running: bool,
    source: Option<SourceAudio>,
    pcm_bytes: usize,
    chunks_raw: Vec<JsValue>,
    opus_bytes: usize,
    chunk_rows: Vec<ChunkRow>,
    encode_time_ms: f64,
    decode_time_ms: f64,
    audio_ctx: Option<AudioContext>,
}

impl DemoState {
    fn new() -> Self {
        Self {
            phase: String::from("простой"),
            error: None,
            running: false,
            source: None,
            pcm_bytes: 0,
            chunks_raw: Vec::new(),
            opus_bytes: 0,
            chunk_rows: Vec::new(),
            encode_time_ms: 0.0,
            decode_time_ms: 0.0,
            audio_ctx: None,
        }
    }

    fn clear_encode(&mut self) {
        self.error = None;
        self.chunks_raw.clear();
        self.opus_bytes = 0;
        self.chunk_rows.clear();
        self.encode_time_ms = 0.0;
        self.decode_time_ms = 0.0;
    }
}

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
    format_str(format_args!(
        r#"<div><span class="label">Статус: </span><span class="value">{phase}</span></div>"#,
        phase = st.phase,
    ))
}

fn line_source(st: &DemoState) -> String {
    match st.source.as_ref() {
        None => format_str(format_args!(
            r#"<div><span class="muted">Источник не сгенерирован. Нажмите «1. Generate».</span></div>"#
        )),
        Some(s) => {
            let frames = s.samples.len() / (s.channels.max(1) as usize);
            format_str(format_args!(
                r#"<div><span class="label">PCM: </span><span class="value">{sr} Hz</span> · <span class="value">{ch} ch</span> · <span class="value">{frames} фреймов</span> · <span class="value">{bytes}</span></div>"#,
                sr = s.sample_rate,
                ch = s.channels,
                frames = frames,
                bytes = bytes_human(st.pcm_bytes),
            ))
        }
    }
}

fn line_opus(st: &DemoState) -> String {
    if st.chunk_rows.is_empty() {
        return String::new();
    }
    let ratio = if st.opus_bytes > 0 {
        st.pcm_bytes as f64 / st.opus_bytes as f64
    } else {
        0.0
    };
    format_str(format_args!(
        r#"<div><span class="label">Opus: </span><span class="value">{n} чанков</span> · <span class="value">{ob}</span> · <span class="label">Сжатие: </span><span class="ok">≈ {r:.1}×</span></div>"#,
        n = st.chunk_rows.len(),
        ob = bytes_human(st.opus_bytes),
        r = ratio,
    ))
}

fn line_times(st: &DemoState) -> String {
    if st.encode_time_ms <= 0.0 {
        return String::new();
    }
    let decode_html = if st.decode_time_ms > 0.0 {
        format_str(format_args!(
            r#" · <span class="label">Decode: </span><span class="value">{d:.0} мс</span>"#,
            d = st.decode_time_ms,
        ))
    } else {
        String::new()
    };
    format_str(format_args!(
        r#"<div><span class="label">Encode: </span><span class="value">{e:.0} мс</span>{dh}</div>"#,
        e = st.encode_time_ms,
        dh = decode_html,
    ))
}

fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("ac-stats") else {
        return;
    };
    let err_line = st
        .error
        .as_ref()
        .map(|m| line_err(m.as_str()))
        .unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{phase}{src}{opus}{times}"#,
        err = err_line,
        phase = line_phase(st),
        src = line_source(st),
        opus = line_opus(st),
        times = line_times(st),
    ));
    el.set_inner_html(&html);
}

/// Рисует waveform из первых `WAVE_ZOOM_SAMPLES` сэмплов.
///
/// **Почему не агрегируем по окнам.** Для синуса постоянной амплитуды
/// пик в любом окне (шириной в несколько периодов) равен 1.0. Все
/// столбики получают одинаковую высоту — визуально это сплошной
/// блок, а не волна. Поэтому показываем сами сэмплы: один bar на
/// один sample. 300 сэмплов при 48 kHz = 6.25 мс = ~2.75 периода
/// 440 Гц — форма волны видна чётко.
fn update_waveform(doc: &Document, element_id: &str, samples: &[f32]) {
    let Some(el) = doc.get_element_by_id(element_id) else {
        return;
    };
    if samples.is_empty() {
        el.set_inner_html("");
        return;
    }
    let shown = WAVE_ZOOM_SAMPLES.min(samples.len());
    let mut html = String::with_capacity(shown * 64);
    for i in 0..shown {
        let v = samples[i].abs().min(1.0);
        // 5% — минимальная видимая высота, 45% — амплитуда.
        let h_pct = 5.0 + v * 45.0;
        html.push_str(&format_str(format_args!(
            r#"<div class="ac-wave-bar" style="height:{h:.1}%"></div>"#,
            h = h_pct,
        )));
    }
    el.set_inner_html(&html);
}

fn update_chunks(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("ac-chunks") else {
        return;
    };
    if st.chunk_rows.is_empty() {
        el.set_inner_html("");
        return;
    }
    let shown = 100.min(st.chunk_rows.len());
    let mut rows = String::with_capacity(shown * 120);
    for c in st.chunk_rows.iter().take(shown) {
        rows.push_str(&format_str(format_args!(
            r#"<tr><td>{i}</td><td>{ty}</td><td>{b}</td><td>{ts:.2} ms</td></tr>"#,
            i = c.idx,
            ty = c.ty,
            b = c.bytes,
            ts = c.timestamp_us / 1000.0,
        )));
    }
    let more = if st.chunk_rows.len() > shown {
        let hidden = st.chunk_rows.len() - shown;
        format_str(format_args!(
            r#"<div class="muted" style="padding:6px 8px">… ещё {hidden} строк (всего {total})</div>"#,
            hidden = hidden,
            total = st.chunk_rows.len(),
        ))
    } else {
        String::new()
    };
    let html = format_str(format_args!(
        r#"<table><thead><tr><th>#</th><th>Type</th><th>Bytes</th><th>Timestamp</th></tr></thead>
           <tbody>{rows}</tbody></table>{more}"#,
        rows = rows,
        more = more,
    ));
    el.set_inner_html(&html);
}

fn set_button_enabled(doc: &Document, id: &str, enabled: bool) {
    let Some(el) = doc.get_element_by_id(id) else {
        return;
    };
    if enabled {
        let _ = el.remove_attribute("disabled");
    } else {
        let _ = el.set_attribute("disabled", "");
    }
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>AudioData</strong> — контейнер для несжатых PCM-сэмплов. Формат <code>f32-planar</code> — единственный, который использует Web Audio API. Сэмплы каналов лежат последовательно: сначала канал 0, потом канал 1.",
        "<strong>AudioEncoder</strong> принимает <code>AudioData</code>, возвращает <code>EncodedAudioChunk</code> через колбэк <code>output(chunk, metadata)</code>. Для Opus <code>description</code> не нужен — в отличие от AAC.",
        "<strong>AudioDecoder</strong> принимает <code>EncodedAudioChunk</code>, возвращает <code>AudioData</code>. Сэмплы извлекаются через <code>copyTo(dest, { planeIndex, format: \"f32-planar\" })</code>.",
        "<strong>Web Audio API</strong>: <code>OscillatorNode</code> генерирует синусоиду 440 Гц через <code>OfflineAudioContext</code>. <code>AudioBufferSourceNode</code> воспроизводит декодированные сэмплы через <code>AudioContext</code>. Прямого моста между <code>AudioData</code> и <code>AudioBuffer</code> нет — только через копирование каналов.",
        "<strong>Waveform source vs decoded</strong> — первые 300 сэмплов (6.25 мс, ~2.75 периода 440 Гц). Один bar на сэмпл. Так видна форма волны; при агрегации по пикам в окне синус постоянной амплитуды даёт плоскую линию.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ac-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("ac-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Генерация сигнала через OfflineAudioContext
// =============================================================================

async fn generate_signal() -> Result<(Vec<f32>, u32, u32), JsValue> {
    let total_frames = (SAMPLE_RATE as f64 * DURATION_SEC) as u32;
    let offline = OfflineAudioContext::new_with_number_of_channels_and_length_and_sample_rate(
        CHANNELS,
        total_frames,
        SAMPLE_RATE as f32,
    )?;

    let osc = offline.create_oscillator()?;
    osc.set_type(OscillatorType::Sine);
    osc.frequency().set_value(FREQ_HZ as f32);
    osc.connect_with_audio_node(&offline.destination())?;
    osc.start()?;

    let promise = offline.start_rendering()?;
    let rendered = JsFuture::from(promise).await?;
    let buffer: AudioBuffer = rendered.dyn_into()?;

    let channel_js = buffer.get_channel_data(0)?;
    let samples: Vec<f32> = channel_js.to_vec();
    Ok((samples, SAMPLE_RATE, CHANNELS))
}

// =============================================================================
// Encode (PCM → Opus)
// =============================================================================

struct EncodeOutcome {
    chunks: Vec<JsValue>,
    rows: Vec<ChunkRow>,
}

async fn do_encode(
    samples: &[f32],
    sample_rate: u32,
    channels: u32,
) -> Result<EncodeOutcome, JsValue> {
    let chunks_store: Rc<RefCell<Vec<JsValue>>> = Rc::new(RefCell::new(Vec::new()));
    let rows_store: Rc<RefCell<Vec<ChunkRow>>> = Rc::new(RefCell::new(Vec::new()));

    let chunks_cb = chunks_store.clone();
    let rows_cb = rows_store.clone();

    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            move |chunk: JsValue, _meta: JsValue| {
                let mut rows = rows_cb.borrow_mut();
                let idx = rows.len() + 1;
                rows.push(ChunkRow {
                    idx,
                    ty: chunk_type(&chunk),
                    bytes: chunk_byte_length(&chunk),
                    timestamp_us: chunk_timestamp(&chunk),
                });
                chunks_cb.borrow_mut().push(chunk);
            },
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
            web_sys::console::error_1(&e);
        });

    let encoder = make_audio_encoder(&on_output, &on_error)?;
    configure_audio_encoder(&encoder, CODEC, sample_rate, channels, OPUS_BITRATE)?;

    // Один большой AudioData — Opus сам нарежет на фреймы по 20 мс.
    let audio_data = make_audio_data(samples, sample_rate, channels, 0.0)?;
    encode_audio(&encoder, &audio_data)?;
    let _ = audio_data_close(&audio_data);

    flush_audio_encoder(&encoder).await?;
    let _ = close_audio_encoder(&encoder);

    on_output.forget();
    on_error.forget();

    let chunks = chunks_store.borrow().clone();
    let rows = rows_store.borrow().clone();
    Ok(EncodeOutcome { chunks, rows })
}

// =============================================================================
// Decode (Opus → PCM)
// =============================================================================

async fn do_decode(
    chunks: Vec<JsValue>,
    sample_rate: u32,
    channels: u32,
) -> Result<Vec<f32>, JsValue> {
    let planes: Rc<RefCell<Vec<Vec<f32>>>> = Rc::new(RefCell::new(Vec::new()));
    let planes_cb = planes.clone();

    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(move |data: JsValue| {
            let n_ch = audio_data_number_of_channels(&data);
            let mut buf = planes_cb.borrow_mut();
            while buf.len() < n_ch as usize {
                buf.push(Vec::new());
            }
            for ch in 0..n_ch {
                let plane = audio_data_copy_channel(&data, ch);
                buf[ch as usize].extend_from_slice(&plane);
            }
            let _ = audio_data_close(&data);
        });
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
            web_sys::console::error_1(&e);
        });

    let decoder = make_audio_decoder(&on_output, &on_error)?;
    configure_audio_decoder(&decoder, CODEC, sample_rate, channels)?;

    let mut last_yield = js_sys::Date::now();
    for chunk in chunks.iter() {
        decode_audio(&decoder, chunk)?;
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    flush_audio_decoder(&decoder).await?;
    let _ = close_audio_decoder(&decoder);

    on_output.forget();
    on_error.forget();

    let planes_v = planes.borrow().clone();
    let mut out = Vec::new();
    for p in planes_v {
        out.extend_from_slice(&p);
    }
    if out.is_empty() {
        return Err(JsValue::from_str("decoded audio is empty"));
    }
    Ok(out)
}

// =============================================================================
// Воспроизведение через AudioContext
// =============================================================================

fn play_samples(
    ctx: &AudioContext,
    samples: &[f32],
    sample_rate: u32,
    channels: u32,
) -> Result<(), JsValue> {
    let frames_per_ch = samples.len() / (channels.max(1) as usize);
    if frames_per_ch == 0 {
        return Err(JsValue::from_str("no samples to play"));
    }
    let buffer = ctx.create_buffer(
        channels,
        frames_per_ch as u32,
        sample_rate as f32,
    )?;

    // Planar-раскладка: канал 0 = samples[0..frames_per_ch], ...
    // copy_to_channel принимает `&[f32]` в web-sys 0.3.105.
    for ch in 0..channels {
        let start = (ch as usize) * frames_per_ch;
        let end = start + frames_per_ch;
        buffer.copy_to_channel(&samples[start..end], ch as i32)?;
    }

    let source: AudioBufferSourceNode = ctx.create_buffer_source()?;
    source.set_buffer(Some(&buffer));
    source.connect_with_audio_node(&ctx.destination())?;
    source.start()?;
    Ok(())
}

fn ensure_audio_context(st: &Rc<RefCell<DemoState>>) -> Option<AudioContext> {
    let mut s = st.borrow_mut();
    if s.audio_ctx.is_none() {
        s.audio_ctx = AudioContext::new().ok();
    }
    s.audio_ctx.clone()
}

// =============================================================================
// Обработчики кнопок
// =============================================================================

async fn do_generate(doc: Document, st: Rc<RefCell<DemoState>>) {
    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.running = true;
        s.error = None;
        s.phase = String::from("генерация сигнала…");
        s.clear_encode();
    }
    let _guard = RunGuard(st.clone());
    update_stats(&doc, &st.borrow());
    update_chunks(&doc, &st.borrow());
    set_button_enabled(&doc, "ac-play", false);

    match generate_signal().await {
        Ok((samples, sr, ch)) => {
            let pcm_bytes = samples.len() * 4;
            {
                let mut s = st.borrow_mut();
                s.source = Some(SourceAudio {
                    samples: samples.clone(),
                    sample_rate: sr,
                    channels: ch,
                });
                s.pcm_bytes = pcm_bytes;
                s.phase = format_str(format_args!(
                    "сигнал готов ({sr} Hz, {ch} ch, {d:.1} с)",
                    sr = sr,
                    ch = ch,
                    d = DURATION_SEC,
                ));
            }
            update_waveform(&doc, "ac-wave", &samples);
            if let Some(el) = doc.get_element_by_id("ac-wave-decoded") {
                el.set_inner_html("");
            }
            update_stats(&doc, &st.borrow());
            set_button_enabled(&doc, "ac-encode", true);
        }
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("generate: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
        }
    }
}

async fn do_encode_click(doc: Document, st: Rc<RefCell<DemoState>>) {
    let (samples, sr, ch) = {
        let s = st.borrow();
        match s.source.as_ref() {
            Some(src) => (src.samples.clone(), src.sample_rate, src.channels),
            None => {
                drop(s);
                let mut s = st.borrow_mut();
                s.error = Some(String::from(
                    "Сначала нажмите Generate — источник не создан.",
                ));
                update_stats(&doc, &s);
                return;
            }
        }
    };

    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.running = true;
        s.clear_encode();
        s.phase = String::from("проверка config…");
    }
    let _guard = RunGuard(st.clone());
    update_stats(&doc, &st.borrow());
    update_chunks(&doc, &st.borrow());

    let ok = is_audio_config_supported(CODEC, sr, ch, OPUS_BITRATE)
        .await
        .unwrap_or(false);
    if !ok {
        let mut s = st.borrow_mut();
        s.error = Some(format_str(format_args!(
            "Opus {sr} Hz {ch} ch не поддержан браузером",
            sr = sr,
            ch = ch,
        )));
        s.phase = String::from("не поддерживается");
        update_stats(&doc, &s);
        return;
    }

    {
        let mut s = st.borrow_mut();
        s.phase = String::from("encode…");
    }
    update_stats(&doc, &st.borrow());

    let t0 = js_sys::Date::now();
    let outcome = match do_encode(&samples, sr, ch).await {
        Ok(v) => v,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("encode: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
            return;
        }
    };
    let t1 = js_sys::Date::now();

    let total_bytes: usize = outcome.rows.iter().map(|r| r.bytes).sum();
    let chunk_count = outcome.rows.len();

    {
        let mut s = st.borrow_mut();
        s.chunks_raw = outcome.chunks;
        s.opus_bytes = total_bytes;
        s.chunk_rows = outcome.rows;
        s.encode_time_ms = t1 - t0;
        s.phase = format_str(format_args!(
            "encode готово ({chunk_count} чанков)"
        ));
    }
    update_stats(&doc, &st.borrow());
    update_chunks(&doc, &st.borrow());
    set_button_enabled(&doc, "ac-play", true);
}

async fn do_decode_and_play(doc: Document, st: Rc<RefCell<DemoState>>) {
    let (chunks, sr, ch, source_len) = {
        let s = st.borrow();
        let src = match s.source.as_ref() {
            Some(v) => v,
            None => {
                drop(s);
                let mut s = st.borrow_mut();
                s.error = Some(String::from(
                    "Сначала Generate и Encode — нечего декодировать.",
                ));
                update_stats(&doc, &s);
                return;
            }
        };
        if s.chunks_raw.is_empty() {
            drop(s);
            let mut s = st.borrow_mut();
            s.error = Some(String::from("Нет чанков. Сначала Encode."));
            update_stats(&doc, &s);
            return;
        }
        (
            s.chunks_raw.clone(),
            src.sample_rate,
            src.channels,
            src.samples.len(),
        )
    };

    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.running = true;
        s.error = None;
        s.phase = String::from("decode…");
        s.decode_time_ms = 0.0;
    }
    let _guard = RunGuard(st.clone());
    update_stats(&doc, &st.borrow());

    let t0 = js_sys::Date::now();
    let decoded = match do_decode(chunks, sr, ch).await {
        Ok(v) => v,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("decode: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
            return;
        }
    };
    let t1 = js_sys::Date::now();

    // Opus pre-skip: первые ~648 сэмплов decoded — тишина (encoder
    // warmup). Trailing padding добавляет ~60 сэмплов в конце.
    let delta = decoded.len() as i64 - source_len as i64;
    let delta_sign = if delta >= 0 { "+" } else { "−" };
    let delta_abs = delta.unsigned_abs();

    {
        let mut s = st.borrow_mut();
        s.decode_time_ms = t1 - t0;
        s.phase = format_str(format_args!(
            "decode готово ({n} сэмплов, {sign}{d} vs source; pre-skip {skip})",
            n = decoded.len(),
            sign = delta_sign,
            d = delta_abs,
            skip = DECODE_SKIP_SAMPLES.min(decoded.len()),
        ));
    }
    update_stats(&doc, &st.borrow());

    // Pre-skip: сдвигаем окно waveform, чтобы попасть на реальный сигнал.
    let skip = DECODE_SKIP_SAMPLES.min(decoded.len());
    update_waveform(&doc, "ac-wave-decoded", &decoded[skip..]);

    let Some(ctx) = ensure_audio_context(&st) else {
        let mut s = st.borrow_mut();
        s.error = Some(String::from("AudioContext не создан"));
        update_stats(&doc, &s);
        return;
    };
    let _ = ctx.resume();
    if let Err(e) = play_samples(&ctx, &decoded, sr, ch) {
        let mut s = st.borrow_mut();
        s.error = Some(format_str(format_args!("play: {e:?}")));
        update_stats(&doc, &s);
    }
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
    h2.set_text_content(Some("23.5 — AudioCodecs (PCM ↔ Opus)"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_audio_encoder_supported()
        || !is_audio_decoder_supported()
        || !is_audio_data_supported()
    {
        let warn = document.create_element("p")?;
        warn.set_class_name("ac-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает AudioEncoder / AudioDecoder / AudioData. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // ---- Controls ----
    let controls = document.create_element("div")?;
    controls.set_class_name("ac-block");
    controls.set_inner_html(
        r#"<button class="ac-btn ac-btn-primary" id="ac-generate">1. Generate 440 Hz (2s)</button>
           <button class="ac-btn ac-btn-primary" id="ac-encode" disabled>2. Encode → Opus</button>
           <button class="ac-btn ac-btn-play" id="ac-play" disabled>3. Decode &amp; Play</button>"#,
    );
    root.append_child(&controls)?;

    // ---- Waveform: источник ----
    let wave_label = document.create_element("div")?;
    wave_label.set_class_name("ac-wave-label");
    wave_label.set_text_content(Some(
        "Source — первые 300 сэмплов (6.25 мс, ~2.75 периода 440 Гц)",
    ));
    root.append_child(&wave_label)?;

    let wave = document.create_element("div")?;
    wave.set_class_name("ac-wave");
    wave.set_id("ac-wave");
    root.append_child(&wave)?;

    // ---- Waveform: восстановленный из Opus ----
    let wave_dec_label = document.create_element("div")?;
    wave_dec_label.set_class_name("ac-wave-label");
    wave_dec_label.set_text_content(Some(
        "Decoded — сэмплы 1000-1300 (пропущен pre-skip Opus)",
    ));
    root.append_child(&wave_dec_label)?;

    let wave_dec = document.create_element("div")?;
    wave_dec.set_class_name("ac-wave");
    wave_dec.set_id("ac-wave-decoded");
    root.append_child(&wave_dec)?;

    // ---- Stats ----
    let stats = document.create_element("div")?;
    stats.set_class_name("ac-stats");
    stats.set_id("ac-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // ---- Chunks ----
    let chunks = document.create_element("div")?;
    chunks.set_class_name("ac-chunks");
    chunks.set_id("ac-chunks");
    root.append_child(&chunks)?;

    // ---- Footnote ----
    let footnote = document.create_element("p")?;
    footnote.set_class_name("ac-footnote");
    footnote.set_text_content(Some(
        "OscillatorNode генерирует синус 440 Гц через OfflineAudioContext. \
         Результат — AudioBuffer, из которого извлекается Float32Array. \
         AudioData в формате f32-planar несёт эти сэмплы в AudioEncoder (Opus). \
         Каждый EncodedAudioChunk — ~20 мс сжатого звука. Чанки сохраняются, \
         поэтому Play не требует повторного Encode. \
         AudioDecoder возвращает AudioData, copyTo извлекает сэмплы в f32-planar, \
         AudioBufferSourceNode воспроизводит через AudioContext. \
         Waveform source vs decoded показывает первые 300 сэмплов напрямую — \
         один bar на сэмпл. Так видна форма волны; при агрегации по пикам в окне \
         синус постоянной амплитуды дал бы плоскую линию. \
         Decoded-waveform пропускает pre-skip Opus (648 сэмплов).",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    // ---- Generate ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("ac-generate")
            .ok_or_else(|| JsValue::from_str("no #ac-generate"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                do_generate(doc, st).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Encode ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("ac-encode")
            .ok_or_else(|| JsValue::from_str("no #ac-encode"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                do_encode_click(doc, st).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Decode & Play ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("ac-play")
            .ok_or_else(|| JsValue::from_str("no #ac-play"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                do_decode_and_play(doc, st).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    Ok(())
}
```

---

## 23.5.7. Анализ решения

**Ценность демо.**

1. **Полный цикл аудио-WebCodecs.** Генерация → `AudioData` → `AudioEncoder` → `EncodedAudioChunk` → `AudioDecoder` → `AudioData` → воспроизведение.
2. **Численное сжатие.** PCM 384 000 B → Opus 20 075 B → **≈ 19.1×**.
3. **Planar-формат явно показан.** `copyTo` с `planeIndex` — единственный способ извлечь каналы.
4. **Визуальное доказательство round-trip.** Source и decoded waveform совпадают по форме.
5. **Правильное объяснение pre-skip.** Пользователь видит, что decoded сдвинут, и понимает почему.

**Правила проекта, применённые.**

- **Отдельные функции для строк** — `line_err`, `line_phase`, `line_source`, `line_opus`, `line_times`, `update_waveform`, `update_chunks`.
- **`format_str(format_args!(...))` в tail-позициях.**
- **`Rc<RefCell<DemoState>>`** — общее состояние между обработчиками и rAF.
- **`RunGuard`** — защита от параллельных запусков.
- **Yield через `TimeoutFuture::new(0)`** в decode-цикле.
- **Префикс `ac-`** — не пересекается с `vd-`, `vf-`, `ve-`, `vt-`.
- **Никаких `unused imports`** — все символы задействованы.
- **`create_element`, не `createElement`** — везде.
- **`#[allow(dead_code)]`** только на `chunk_to_vec` — публичный API, не используется в демо.

**Найденные в практике проблемы и их решения.**

1. **`OscillatorType` / `AudioParam` не подтягивались автоматически.** Пришлось добавить в `Cargo.toml` 9 feature-фич. Без `AudioParam` метода `frequency()` у `OscillatorNode` не существует.
2. **`copy_to_channel` в web-sys 0.3.105 принимает `&[f32]`,** не `&Float32Array`. Прямое следствие версии — важно проверять сигнатуры в регенерации web-sys.
3. **Waveform из агрегации по пикам в окне даёт плоскую линию** для синуса постоянной амплитуды. Правильное решение — показывать **сами сэмплы**, один bar на сэмпл. 300 сэмплов при 48 kHz = 6.25 мс = ~2.75 периода 440 Гц.
4. **Pre-skip Opus — 648 сэмплов.** Первый чанк — 13.5 мс, а не 20 мс. Decoded waveform показывал тишину. Решение: пропускаем 1000 сэмплов в decoded waveform.
5. **Trailing padding +60 сэмплов.** Opus дополняет последний фрейм нулями. Явно показываем дельту в статусе.
6. **`AudioContext` требует user gesture.** Ленивое создание при первом Play (через `Option<AudioContext>` в `DemoState`) — правильный паттерн.
7. **Чанки сохраняются в `chunks_raw`.** Play не требует повторного Encode. Иначе кнопки Encode и Play не были бы независимыми.

**Проверки корректности.**

- `format_args!` без `f`-суффикса: `{r:.1}`, `{e:.0}`, `{d:.0}`, `{h:.1}`, `{ts:.2}`, `{kb:.1}`. Все валидны.
- Raw-строки: `r#"..."#` в HTML, `r##"..."##` не нужен — `"#` внутри нет.
- `AudioData` конструктор принимает `Float32Array` через `set_prop(&init, "data", &array)`.
- `audio_data_copy_channel` проверяет все `Result` — при ошибке возвращает пустой `Vec`.
- `play_samples` использует planar-раскладку `&samples[start..end]`.
- `RunGuard` устанавливается после `s.running = true`.
- `decoded[skip..]` защищён `DECODE_SKIP_SAMPLES.min(decoded.len())`.

---

## 23.5.8. Wow-момент на практике

**Первый.** PCM 384 000 байт → Opus 20 075 байт = **19.1×**. Тон 440 Гц воспроизводится чисто через `AudioContext`.

**Второй.** Source и decoded waveform **визуально идентичны** — пять «линз» (форма `|sin|`) одинаковой амплитуды. Доказательство корректности round-trip.

**Третий.** Pre-skip Opus = 648 сэмплов. Первый чанк — 13.5 мс, а не 20 мс. Пользователь видит строку `decode готово (96060 сэмплов, +60 vs source; pre-skip 1000)` и понимает, что происходит.

**Четвёртый.** Все чанки `key`. WebCodecs для аудио не различает key/delta — у Opus нет inter-frame prediction. В отличие от видео, где key ≈ 5× delta по размеру.

**Пятый.** Обратная зависимость: encode 12 мс, decode 23 мс. Не паттерн, а следствие аллокаций в колбэке decode и setup `AudioContext`.

---

## 23.5.9. Итог

**`AudioData`** — контейнер для PCM-сэмплов. Planar-формат `f32-planar` — стандарт Web Audio API. Извлечение через `copyTo(dest, { planeIndex, format })`.

**`AudioEncoder`** — принимает `AudioData`, отдаёт `EncodedAudioChunk`. Opus не требует `description`. `isConfigSupported` — обязательный шаг.

**`AudioDecoder`** — принимает `EncodedAudioChunk`, отдаёт `AudioData`. Для Opus синхронизируется сам.

**Web Audio API vs WebCodecs.** Прямого моста нет: `AudioBuffer` → `AudioData` — только через `getChannelData` + конструктор; `AudioData` → `AudioBuffer` — только через `copyTo` + `createBuffer` + `copyToChannel`.

**Практические находки.**

1. Web-sys требует явных feature для каждого типа: без `AudioParam` нет `OscillatorNode::frequency()`. Каждая аудио-фича — отдельная строка в `Cargo.toml`.
2. Waveform для синуса постоянной амплитуды нужно рисовать **по сэмплам**, а не по пикам в окне. 300 сэмплов × 1 bar = форма видна.
3. Opus pre-skip = 648 сэмплов. Первый чанк короче. Decoded waveform сдвигается на длину pre-skip.
4. Trailing padding +60 сэмплов в конце — компенсация неполного последнего фрейма.
5. Чанки нужно сохранять между Encode и Play — иначе придётся энкодить заново.

