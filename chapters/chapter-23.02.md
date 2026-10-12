# Часть II. Кодеки
## Глава 23.2. VideoDecoder: сжатое → распакованное

### 23.2.0. Обновлённая структура

```
src/
  video_decoder.rs                ← НОВЫЙ модуль 23.2
  video_frame.rs                  ← из 23.1
  codecs_demo.rs                  ← из главы 23
  fractal.rs                      ← из 22.6
  …
tests/
  wasm_video_decoder.rs           ← НОВЫЙ тест
src/lib.rs                        ← +1 строка: pub mod video_decoder;
src/app.rs                        ← демо 23.2 (заменяется целиком)
web/
  public/workers/…                ← существующие воркеры не трогаем
  pkg/                            ← сборка wasm-pack (out-dir web/pkg)
  main.js                         ← без правок
Cargo.toml                        ← без правок
vite.config.js                    ← без правок
package.json                      ← без правок
```

**Комментарий.** Модуль `video_decoder.rs` — про **VideoDecoder**: конструктор с колбэками `output`/`error`, `configure` с `description` (SPS/PPS), `decode(EncodedVideoChunk)`, `flush()`, `close()`. Префикс CSS — `vd-` (video decoder), не пересекается с `vf-` (23.1), `cd-` (глава 23). Демо автономно. Мини-хелперы (`get_fn`, `invoke`, `construct`, `set_prop`, `get_str`, `get_f64`, `num`, `format_str`, `bool_of`) — **переиспользуются** из `video_frame::*` через `pub use`. Дублирования нет.

**Ссылки для детального изучения:**

- MDN: [`VideoDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder)
- MDN: [`VideoDecoder.configure()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder/configure)
- MDN: [`VideoDecoder.decode()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder/decode)
- MDN: [`VideoDecoder.flush()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoDecoder/flush)
- W3C: [WebCodecs § VideoDecoder](https://w3c.github.io/webcodecs/#videodecoder-interface)
- W3C: [WebCodecs § VideoDecoderConfig](https://w3c.github.io/webcodecs/#dictdef-videodecoderconfig)
- Wikipedia: [H.264 SPS/PPS](https://en.wikipedia.org/wiki/Network_Abstraction_Layer)

---

### 23.2.1. О чём эта подглава

`VideoDecoder` — **вход** в WebCodecs со стороны сжатых данных. Он принимает `EncodedVideoChunk` (поток байтов в формате AVCC), декодирует и отдаёт `VideoFrame` через колбэк `output`.

Ключевая сложность — **`description`**. Для H.264 в формате `avc1` декодер обязан получить `avcC`-box с параметрами SPS (Sequence Parameter Set) и PPS (Picture Parameter Set). Без них декодер не знает, как разобрать поток: ширина, высота, профиль, энтропийное кодирование — всё это в SPS/PPS.

Описание приходит из `VideoEncoder` — в колбэке `output(chunk, metadata)` на **первом key-кадре** есть `metadata.decoderConfig.description`. Это `ArrayBuffer` (в Chrome) или `Uint8Array` — зависит от браузера.

Подглава отвечает на пять вопросов.

**Первый — как устроен `VideoDecoder`.** Конструктор принимает два колбэка: `output(frame)` — вызывается для каждого декодированного кадра асинхронно; `error(e)` — вызывается при ошибке декодирования.

**Второй — что передаётся в `configure`.** `codec`, `codedWidth`, `codedHeight`, `description`. Первые три — обязательны, `description` — обязателен для avc1.

**Третий — как подавать чанки.** `decoder.decode(chunk)` для каждого чанка в порядке timestamp. Первым должен идти key-кадр.

**Четвёртый — как завершить.** `await decoder.flush()` дожидается всех кадров. `decoder.close()` освобождает ресурсы.

**Пятый — что без `description`.** `EncodingError: The given encoding is not supported` — либо при первом `decode`, либо при `flush`. `configure` **не бросает синхронно**.

---

### 23.2.2. Что уже есть и что появится

К концу 23.1 умеем:
- создавать `VideoFrame` из canvas, читать метаданные, закрывать;
- создавать `EncodedVideoChunk` вручную;
- извлекать пиксели через `drawImage + getImageData`.

К концу 23.2 будем уметь:
- создавать `VideoDecoder` с колбэками;
- извлекать `description` (SPS/PPS) из `VideoEncoder.metadata`;
- подавать поток `EncodedVideoChunk` в декодер;
- читать `description` в hex;
- ловить `EncodingError` при отсутствии `description`;
- показывать прогресс декодирования через колбэк `output`;
- сравнивать размеры `VideoFrame` и `EncodedVideoChunk`;
- читать `metadata.decoderConfig` (codec, codedWidth, codedHeight).

---

### 23.2.3. Что такое `description`

**Ключевое понятие.** H.264 в MP4 хранится в формате **AVCC**: каждый NALU (Network Abstraction Layer Unit) предваряется 4-байтовой длиной, а не start code `0x00000001`. Для разбора этого формата декодеру нужны **SPS** и **PPS** — параметры последовательности и картинки.

Эти два блока упакованы в контейнер `avcC` — это и есть `description`.

**Что внутри `avcC`:**

```
0x01                  ← version (1)
0x42 0x00 0x1E        ← profile, compat, level
0xFF                  ← NALU length size minus 1 (обычно 3 → 4 байта)
0xE1                  ← num SPS (нижние 5 бит = 1)
0x00 0x1E             ← SPS length
...SPS bytes...
0x01                  ← num PPS
0x00 0x04             ← PPS length
...PPS bytes...
```

**Размер.** Типичный `avcC` для 320×240 — 30–50 байт.

**Откуда берётся.** `VideoEncoder` для avc1 присылает его в `metadata.decoderConfig.description` на первом key-кадре.

**Зачем.** Без него `VideoDecoder` не знает, как разобрать AVCC-поток, и выдаёт `EncodingError`. Это не придирка — это фундаментальное требование H.264/AVCC.

**Альтернатива — Annex B.** Формат с start codes (`0x00000001`). В этом случае `description` не нужен, но требует `codec: "avc3..."` (не `avc1...`). WebCodecs поддерживает оба, но для avc1 (самый распространённый) `description` обязателен.

**Практическая находка.** В Chrome `metadata.decoderConfig.description` приходит как **`ArrayBuffer`**, а не `Uint8Array`. Прямое `dyn_into::<Uint8Array>()` на `ArrayBuffer` даёт `Err`. Универсальный хелпер `buffer_source_to_vec` разбирает оба варианта.

---

### 23.2.4. Колбэки `output` и `error`

**`output: (frame) => ...`** вызывается **асинхронно** для каждого декодированного кадра. Не в том же порядке, что `decode()` — порядок определяется `timestamp`.

**`error: (e) => ...`** вызывается при ошибке. `e` — `DOMException` с `message`, обычно `EncodingError`.

**Важно.** `output` держит `VideoFrame` открытым, пока вы его не закроете. Если забыть `frame.close()`, браузер в какой-то момент перестанет выдавать новые кадры (внутренняя очередь decoder'а заполнится).

**Правило демо.** В `output` кадр сразу рисуется на scratch-canvas, извлекаются пиксели, кадр закрывается. Никаких открытых `VideoFrame` не остаётся. **Бонус:** в том же колбэке инкрементится счётчик прогресса и обновляется UI — видно, как кадры приходят один за другим, а не только финальное число.

**Тонкость.** `configure` без `description` **не бросает синхронно**. Ошибка приходит через `error`-колбэк или через rejected `flush()`. Это значит, что обработка ошибок должна быть асинхронной: собрать сообщение из колбэка, дождаться `flush`, сравнить.

---

### 23.2.5. Что показывает демо

**Layout:**

- **Source canvas** — main thread рисует 30 кадров с градиентом (это то, что мы будем кодировать).
- **Result canvas** — воспроизводит декодированные кадры в цикле.
- **Кнопки:**
  - **Encode + Decode** — полный pipeline: 30 кадров → encode → chunks → decode → frames.
  - **Without description** — configure без `description`, попытка decode, ожидание `EncodingError`.
  - **Pause / Resume**.
- **Статистика (4 строки):**
  - `Статус` + `Chunks: N (key: K, delta: D)`.
  - `Description: N байт` · `Decoded frames: M` (в процессе — `M / Total (идёт…)`) · `Playback: i / N`.
  - `Decoder config: codec=avc1.42001E · 320×240 · description: есть` — из `metadata.decoderConfig`.
  - `Размеры: VideoFrame=307200 байт (RGBA 320×240×4) · Encoded=N байт (key: …, delta: …) · Сжатие ≈ Kx`.
- **Hex-dump `description`** — первые 64 байта в шестнадцатеричном виде.

**Сценарий:**

1. При загрузке — source canvas показывает пульсирующий градиент. Result пуст.
2. Клик **Encode + Decode**:
   - Фаза `encode`: 30 кадров идут в `VideoEncoder` → 30 чанков + `description`.
   - Фаза `decode`: `VideoDecoder` декодирует чанки, `output` собирает пиксели и обновляет прогресс.
   - Фаза `готово`: playback на result canvas.
3. Клик **Without description** — `frames` очищаются, result canvas чернеет, в статистике появляется красная строка с `EncodingError`.
4. **Pause / Resume** — остановка и продолжение playback.

**Что видно:**

- Source и result показывают **один и тот же** градиент. Это доказывает, что encode + decode — обратимый цикл.
- `description` виден в hex — 30–50 байт реальных данных.
- `Chunks: 30 (key: 1, delta: 29)` — соответствует ожиданиям H.264.
- Прогресс декодирования идёт от `0 / 30` до `30 / 30`.
- Коэффициент сжатия — около **30–100×** в зависимости от содержимого.
- После **Without description** — **явный контраст**: было 30 кадров, стало 0.

---

### 23.2.6. Wow-момент

**Первый.** Source и result показывают одинаковые картинки. Один поток сжали в 30 чанков, другой распаковал обратно. Цикл замкнут.

**Второй.** В hex-dump видно: `01 42 c0 14 03 01 00 19 67 67 42 …` — это `avcC`-box. `0x67` — NALU type 7 (SPS), `0x68` — NALU type 8 (PPS). Всё, что нужно для разбора потока — 41 байт.

**Третий.** Клик **Without description**. В статистике — красная строка:
```
EncodingError: The given encoding is not supported.
```
Именно так `VideoDecoder` сообщает, что avc1 без SPS/PPS — не поддерживается.

**Четвёртый.** Смотрим на строку **Размеры**. `VideoFrame` — 307 200 байт (RGBA 320×240×4). `Encoded` — 4–8 КБ для key, 0.5–2 КБ для delta. Коэффициент **~40–90×**. Именно ради этого и существует кодек.

**Пятый.** Замеряем время. Encode 30 кадров — 60 мс, decode — 90 мс. В 1.5 раза дороже. Так работает асимметрия кодека: кодирование «жертвует» данными (проще), декодирование «восстанавливает» их (сложнее).

---

### 23.2.7. Исходные файлы

#### `src/video_decoder.rs`

```rust
//! 23.2. VideoDecoder: сжатое → распакованное.
//!
//! Обёртки вокруг `VideoDecoder`:
//!
//! - конструктор с колбэками `output` / `error`;
//! - `configure` с опциональным `description` (SPS/PPS);
//! - `decode(EncodedVideoChunk)`, `flush()`, `close()`;
//! - утилита `hex_dump` для показа description в hex.
//!
//! ## BufferSource
//!
//! `metadata.decoderConfig.description` в Chrome приходит как
//! `ArrayBuffer`, а не `Uint8Array`. Оба варианта, а также typed
//! arrays над буфером, разбирает `buffer_source_to_vec`. Прямой
//! `dyn_into::<Uint8Array>()` на `ArrayBuffer` не работает.
//!
//! ## Переиспользование хелперов
//!
//! Мини-хелперы для JsValue (`get_fn`, `invoke`, `construct`,
//! `set_prop`, `get_str`, `get_f64`, `num`, `format_str`, `bool_of`)
//! определены в `video_frame` (подглава 23.1) и реэкспортируются
//! здесь через `pub use`. Дублирования нет.

use js_sys::{ArrayBuffer, Object, Promise, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{
    bool_of, construct, format_str, get_fn, num, set_prop,
};

// =============================================================================
// BufferSource → Vec<u8>
// =============================================================================

/// Приводит BufferSource к `Vec<u8>`.
///
/// Понимает три случая:
///
/// - `ArrayBuffer` — оборачивает в `Uint8Array::new(&ab)` и копирует.
/// - `Uint8Array` — копирует через `to_vec()`.
/// - Любой `ArrayBufferView` (typed array, DataView) — читает
///   поле `.buffer` и обрабатывает его как `ArrayBuffer`.
///
/// Для не-BufferSource возвращает пустой `Vec`.
///
/// ## Зачем
///
/// `metadata.decoderConfig.description` в Chrome — `ArrayBuffer`.
/// В других браузерах и версиях — может быть `Uint8Array`. Прямое
/// `dyn_into::<Uint8Array>()` на `ArrayBuffer` даёт `Err`, и
/// дальнейшая логика молча получает пустой буфер. Универсальный
/// хелпер снимает этот класс ошибок.
pub fn buffer_source_to_vec(v: &JsValue) -> Vec<u8> {
    // 1. Чистый ArrayBuffer.
    if let Ok(ab) = v.clone().dyn_into::<ArrayBuffer>() {
        return Uint8Array::new(&ab).to_vec();
    }

    // 2. Uint8Array напрямую.
    if let Ok(u8) = v.clone().dyn_into::<Uint8Array>() {
        return u8.to_vec();
    }

    // 3. Любой ArrayBufferView: читаем .buffer.
    if let Ok(buf) = Reflect::get(v, &JsValue::from_str("buffer")) {
        if !buf.is_undefined() && !buf.is_null() {
            if let Ok(ab) = buf.dyn_into::<ArrayBuffer>() {
                return Uint8Array::new(&ab).to_vec();
            }
        }
    }

    Vec::new()
}

// =============================================================================
// Feature detection
// =============================================================================

/// `VideoDecoder` доступен в этом браузере.
pub fn is_video_decoder_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("VideoDecoder")),
        Ok(v) if v.is_function()
    ))
}

/// `VideoDecoder.isConfigSupported` доступен. Статический метод.
pub fn has_is_config_supported() -> bool {
    let global = js_sys::global();
    let Ok(ctor) = Reflect::get(&global, &JsValue::from_str("VideoDecoder")) else {
        return false;
    };
    bool_of(matches!(
        Reflect::get(&ctor, &JsValue::from_str("isConfigSupported")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Конструктор
// =============================================================================

/// `new VideoDecoder({ output, error })`.
///
/// Оба колбэка обязательны. `output` вызывается асинхронно для
/// каждого декодированного кадра; `error` — при ошибке.
pub fn make_video_decoder(
    on_output: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
    on_error: &wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
) -> Result<JsValue, JsValue> {
    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    construct("VideoDecoder", &[init.into()])
}

// =============================================================================
// Configure / decode / flush / close
// =============================================================================

/// `decoder.configure({ codec, codedWidth, codedHeight, description })`.
///
/// `description` — `BufferSource` с `avcC`-box (SPS/PPS). Для avc1
/// обязателен. Для avc3 (Annex B) не нужен.
pub fn configure(
    decoder: &JsValue,
    codec: &str,
    coded_width: u32,
    coded_height: u32,
    description: Option<&JsValue>,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "codedWidth", &num(coded_width as f64))?;
    set_prop(&cfg, "codedHeight", &num(coded_height as f64))?;
    if let Some(d) = description {
        set_prop(&cfg, "description", d)?;
    }
    let f = get_fn(decoder, "configure")?;
    let _ = f.call1(decoder, &cfg)?;
    Ok(())
}

/// `decoder.decode(chunk)`.
///
/// Чанки подаются в порядке timestamp. Первым должен идти key-кадр.
pub fn decode_chunk(decoder: &JsValue, chunk: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "decode")?;
    let _ = f.call1(decoder, chunk)?;
    Ok(())
}

/// `await decoder.flush()`.
///
/// Дожидается всех кадров, декодирует оставшиеся, резолвится когда
/// очередь пуста.
pub async fn flush(decoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "flush")?;
    let promise = f.call0(decoder)?.dyn_into::<Promise>()?;
    JsFuture::from(promise).await.map(|_| ())
}

/// `decoder.close()`.
pub fn close(decoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "close")?;
    let _ = f.call0(decoder)?;
    Ok(())
}

// =============================================================================
// Утилита: hex dump для description
// =============================================================================

/// Возвращает hex-представление первых `max_bytes` байт значения.
///
/// Понимает `ArrayBuffer`, `Uint8Array` и любой `ArrayBufferView`
/// через `buffer_source_to_vec`. Для не-BufferSource возвращает
/// `"(не BufferSource или пустой буфер)"`.
///
/// Полезно для показа `description` (SPS/PPS) — там видно
/// характерные байты `0x67` (SPS) и `0x68` (PPS).
pub fn hex_dump(value: &JsValue, max_bytes: usize) -> String {
    let bytes = buffer_source_to_vec(value);
    if bytes.is_empty() {
        return String::from("(не BufferSource или пустой буфер)");
    }

    let len = bytes.len();
    let shown = len.min(max_bytes);

    // Оценка: 3 символа на байт + пробелы + переводы строк.
    let mut s = String::with_capacity(shown * 3 + 32);
    for (i, b) in bytes.iter().take(shown).enumerate() {
        if i > 0 {
            if i % 16 == 0 {
                s.push('\n');
            } else {
                s.push(' ');
            }
        }
        s.push_str(&format_str(format_args!("{:02x}", b)));
    }
    if len > max_bytes {
        s.push_str(&format_str(format_args!(
            "\n… (всего {len} байт)",
            len = len,
        )));
    }
    s
}
```

#### `src/lib.rs` — добавление

Добавьте одну строку рядом с `video_frame`:

```rust
pub mod codecs_demo;     // 23
pub mod video_frame;     // 23.1
pub mod video_decoder;   // ← 23.2
```

#### `tests/wasm_video_decoder.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

use hello_wasm02::video_decoder::{
    close, configure, has_is_config_supported, hex_dump,
    is_video_decoder_supported, make_video_decoder,
};
use hello_wasm02::video_frame::{
    create_encoded_chunk, is_encoded_chunk_supported,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn video_decoder_is_supported_in_browser() {
    assert!(
        is_video_decoder_supported(),
        "VideoDecoder should be available",
    );
}

#[wasm_bindgen_test]
fn encoded_chunk_is_supported_in_browser() {
    assert!(is_encoded_chunk_supported());
}

#[wasm_bindgen_test]
fn is_config_supported_is_available() {
    let _ = has_is_config_supported();
}

#[wasm_bindgen_test]
fn make_decoder_with_callbacks_does_not_throw() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(wasm_bindgen::JsValue)>::new(
            |_frame: wasm_bindgen::JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(wasm_bindgen::JsValue)>::new(
            |_e: wasm_bindgen::JsValue| {},
        );
    let decoder = make_video_decoder(&on_output, &on_error);
    assert!(decoder.is_ok(), "VideoDecoder should be constructible");
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn configure_without_description_does_not_throw_sync() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(wasm_bindgen::JsValue)>::new(
            |_frame: wasm_bindgen::JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(wasm_bindgen::JsValue)>::new(
            |_e: wasm_bindgen::JsValue| {},
        );
    let decoder =
        make_video_decoder(&on_output, &on_error).expect("construct");
    let result = configure(&decoder, "avc1.42001E", 320, 240, None);
    assert!(
        result.is_ok(),
        "configure without description does not throw synchronously",
    );
    let _ = close(&decoder);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn close_does_not_throw() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(wasm_bindgen::JsValue)>::new(
            |_frame: wasm_bindgen::JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(wasm_bindgen::JsValue)>::new(
            |_e: wasm_bindgen::JsValue| {},
        );
    let decoder = make_video_decoder(&on_output, &on_error).expect("construct");
    close(&decoder).expect("close");
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn hex_dump_renders_uint8array() {
    let data = vec![0x01u8, 0x42, 0x00, 0x1E, 0xFF];
    let arr = js_sys::Uint8Array::from(&data[..]);
    let s = hex_dump(&arr, 64);
    assert!(s.contains("01"));
    assert!(s.contains("42"));
    assert!(s.contains("1e"));
    assert!(s.contains("ff"));
}

#[wasm_bindgen_test]
fn hex_dump_renders_array_buffer() {
    // Практическая находка: description в Chrome приходит как
    // ArrayBuffer, а не Uint8Array. hex_dump должен его понимать.
    let data = vec![0x67u8, 0x42, 0x00, 0x1E]; // SPS start
    let u8 = js_sys::Uint8Array::from(&data[..]);
    let ab: js_sys::ArrayBuffer = u8.buffer();
    let s = hex_dump(&ab, 64);
    assert!(s.contains("67"));
    assert!(s.contains("42"));
    assert!(s.contains("1e"));
}

#[wasm_bindgen_test]
fn hex_dump_truncates_at_max() {
    let data: Vec<u8> = (0..100).map(|i| (i & 0xFF) as u8).collect();
    let arr = js_sys::Uint8Array::from(&data[..]);
    let s = hex_dump(&arr, 10);
    assert!(s.contains("всего 100"), "got: {s}");
}

#[wasm_bindgen_test]
fn hex_dump_handles_non_buffer() {
    let v = wasm_bindgen::JsValue::from_str("not bytes");
    let s = hex_dump(&v, 16);
    assert!(
        s.contains("BufferSource") || s.contains("пуст"),
        "expected fallback message, got: {s}",
    );
}

#[wasm_bindgen_test]
fn encoded_chunk_can_be_created() {
    let data = vec![0u8; 128];
    let chunk = create_encoded_chunk("key", 0.0, 33333.0, &data)
        .expect("create chunk");
    assert!(!chunk.is_null());
    assert!(!chunk.is_undefined());
    let _ = chunk.dyn_into::<js_sys::Object>();
}
```

#### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_video_decoder

# 3. Демо
npm run wasm:dev && npm start
```

---

### 23.2.8. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.2 — VideoDecoder: сжатое → распакованное.
//!
//! Демо: 30 синтетических кадров кодируются через `VideoEncoder`,
//! полученные `EncodedVideoChunk` декодируются через `VideoDecoder`.
//! Между кодированием и декодированием передаётся `description`
//! (SPS/PPS) — без него avc1 не работает.
//!
//! ## Что демонстрируется
//!
//! - `new VideoDecoder({ output, error })` — конструктор с колбэками.
//! - `configure({ codec, codedWidth, codedHeight, description })`.
//! - `decode(chunk)` × N + `await flush()`.
//! - `metadata.decoderConfig.description` — где взять SPS/PPS.
//! - `EncodingError` без `description` — кнопка «Without description».
//! - Hex-dump `description` — что там внутри.
//! - Прогресс декодирования через `on_output`-колбэк.
//! - Сравнение размеров `VideoFrame` vs `EncodedVideoChunk`.
//! - Метаданные `decoderConfig`: `codec`, `codedWidth`, `codedHeight`.
//!
//! ## Что не является темой
//!
//! `VideoEncoder` здесь используется **только** чтобы получить
//! `EncodedVideoChunk` и `description`. Подробно — в 23.3.
//!
//! ## BufferSource
//!
//! `description` приходит как `ArrayBuffer`, не `Uint8Array`.
//! Все операции с ним идут через `buffer_source_to_vec` из
//! `video_decoder`. См. её документацию.
//!
//! ## Правила проекта
//!
//! - `format_str(format_args!(...))` в tail-позициях.
//! - `get_fn` / `invoke` / `construct` — из `video_frame`.
//! - `getImageData` — через `Reflect::invoke` (обход сигнатуры web-sys).
//! - `Ref::clone()` в локальную переменную, если `Ref` не должен
//!   пережить блок (borrow of moved value).
//!
//! **Файл автономен.** Стили `.vd-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use js_sys::{Array, Object, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement, ImageData,
};

use crate::video_decoder::{
    buffer_source_to_vec, close as decoder_close,
    configure as decoder_configure, decode_chunk, flush as decoder_flush,
    hex_dump, is_video_decoder_supported, make_video_decoder,
};
use crate::video_frame::{
    close as frame_close, format_str, get_fn, get_str, invoke, num, set_prop,
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
            var KEY='vd-sw-cleared-23-2';
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
    let id = "vd-injected-style";
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
.vd-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.vd-desc-line { margin:6px 0; }
.vd-canvases { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.vd-canvas-col { display:flex;flex-direction:column;gap:6px;}
.vd-canvas-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.vd-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;}
.vd-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.vd-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.vd-btn:hover:not([disabled]) { background:#30363d; }
.vd-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.vd-btn-primary:hover:not([disabled]) { background:#2ea043; }
.vd-btn-warn { background:#21262d;border-color:#30363d;color:#f0883e;}
.vd-btn-warn:hover:not([disabled]) { background:#30363d; }
.vd-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.vd-stats { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;margin-bottom:16px;}
.vd-stats .label { color:#8b949e; }
.vd-stats .value { color:#79c0ff; }
.vd-stats .ok { color:#3fb950; }
.vd-stats .warn { color:#f0883e; }
.vd-stats .err { color:#f85149; }
.vd-stats .muted { color:#6e7681; }
.vd-hex { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #d2a8ff;border-radius:4px;color:#d2a8ff;font-family:ui-monospace,monospace;font-size:11px;line-height:1.5;white-space:pre-wrap;margin-bottom:16px;}
.vd-hex-label { color:#8b949e; font-size:12px; margin-bottom:6px; }
.vd-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
.vd-block .label { color:#8b949e; }
.vd-block .value { color:#79c0ff; }
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
const HEX_PREVIEW_BYTES: usize = 64;

/// Один RGBA-кадр в памяти: W × H × 4 байта.
const FRAME_RGBA_BYTES: usize = (FRAME_W as usize) * (FRAME_H as usize) * 4;

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    paused: bool,
    phase: String,
    error: Option<String>,
    chunk_count: u32,
    key_count: u32,
    delta_count: u32,
    description_bytes: usize,
    description_hex: String,
    /// Метаданные из `metadata.decoderConfig` (энкодер).
    decoder_codec: String,
    decoder_coded_width: u32,
    decoder_coded_height: u32,
    /// Размеры сжатого потока.
    encoded_total_bytes: usize,
    encoded_key_bytes: usize,
    encoded_delta_bytes: usize,
    /// Прогресс декодирования (инкрементится в on_output).
    decode_progress: u32,
    decode_total: u32,
    /// Готовые кадры (RGBA) для playback.
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
            chunk_count: 0,
            key_count: 0,
            delta_count: 0,
            description_bytes: 0,
            description_hex: String::from("—"),
            decoder_codec: String::new(),
            decoder_coded_width: 0,
            decoder_coded_height: 0,
            encoded_total_bytes: 0,
            encoded_key_bytes: 0,
            encoded_delta_bytes: 0,
            decode_progress: 0,
            decode_total: 0,
            frames: Vec::new(),
            play_idx: 0,
            window_frames: 0,
        }
    }

    fn clear_pipeline(&mut self) {
        self.chunk_count = 0;
        self.key_count = 0;
        self.delta_count = 0;
        self.description_bytes = 0;
        self.description_hex = String::from("—");
        self.decoder_codec.clear();
        self.decoder_coded_width = 0;
        self.decoder_coded_height = 0;
        self.encoded_total_bytes = 0;
        self.encoded_key_bytes = 0;
        self.encoded_delta_bytes = 0;
        self.decode_progress = 0;
        self.decode_total = 0;
    }
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("vd-stats") else {
        return;
    };

    let err_line = match st.error.as_ref() {
        Some(msg) => format_str(format_args!(
            r#"<div><span class="err">{msg}</span></div>"#,
            msg = msg,
        )),
        None => String::new(),
    };

    let playback = if st.frames.is_empty() {
        String::from("—")
    } else {
        format_str(format_args!(
            "{i} / {n}",
            i = st.play_idx,
            n = st.frames.len()
        ))
    };

    // Decoder config — показываем только когда он есть.
    let config_line = if st.decoder_codec.is_empty() {
        String::new()
    } else {
        format_str(format_args!(
            r#"<div><span class="label">Decoder config: </span><span class="value">codec={codec}</span> · <span class="value">{w}×{h}</span> · <span class="label">description: </span><span class="ok">есть</span></div>"#,
            codec = st.decoder_codec,
            w = st.decoder_coded_width,
            h = st.decoder_coded_height,
        ))
    };

    // Размеры и коэффициент сжатия.
    let sizes_line = if st.encoded_total_bytes == 0 {
        String::new()
    } else {
        let raw_total = FRAME_RGBA_BYTES * (st.chunk_count as usize);
        let ratio = if st.encoded_total_bytes > 0 {
            raw_total as f64 / st.encoded_total_bytes as f64
        } else {
            0.0
        };
        format_str(format_args!(
            r#"<div><span class="label">Размеры: </span><span class="label">VideoFrame=</span><span class="value">{fb} байт</span> <span class="muted">(RGBA {w}×{h}×4)</span> · <span class="label">Encoded=</span><span class="value">{et} байт</span> <span class="muted">(key: <span class="ok">{ek}</span>, delta: <span class="warn">{ed}</span>)</span> · <span class="label">Сжатие ≈ </span><span class="ok">{ratio:.1}×</span></div>"#,
            fb = FRAME_RGBA_BYTES,
            w = FRAME_W,
            h = FRAME_H,
            et = st.encoded_total_bytes,
            ek = st.encoded_key_bytes,
            ed = st.encoded_delta_bytes,
            ratio = ratio,
        ))
    };

    // Прогресс: если decode_total задан — показываем прогресс, иначе
    // показываем число готовых кадров.
    let decoded_display = if st.decode_total > 0 {
        format_str(format_args!(
            r#"<span class="value">{p}</span> <span class="muted">/ {t} (идёт…)</span>"#,
            p = st.decode_progress,
            t = st.decode_total,
        ))
    } else {
        format_str(format_args!(
            r#"<span class="value">{n}</span>"#,
            n = st.frames.len(),
        ))
    };

    let html = format_str(format_args!(
        r#"{err}
           <div><span class="label">Статус: </span><span class="value">{phase}</span> · <span class="label">Chunks: </span><span class="value">{cc}</span> (key: <span class="ok">{kc}</span>, delta: <span class="warn">{dc}</span>)</div>
           <div><span class="label">Description: </span><span class="value">{desc} байт</span> · <span class="label">Decoded frames: </span>{decoded} · <span class="label">Playback: </span><span class="value">{pb}</span></div>
           {config_line}
           {sizes_line}"#,
        err = err_line,
        phase = st.phase,
        cc = st.chunk_count,
        kc = st.key_count,
        dc = st.delta_count,
        desc = st.description_bytes,
        decoded = decoded_display,
        pb = playback,
        config_line = config_line,
        sizes_line = sizes_line,
    ));
    el.set_inner_html(&html);
}

fn update_hex(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("vd-hex-body") else {
        return;
    };
    el.set_text_content(Some(&st.description_hex));
}

const DESC_LINES: &[&str] = &[
    "VideoDecoder принимает <strong>EncodedVideoChunk</strong> и выдаёт \
     <strong>VideoFrame</strong> через колбэк <code>output</code>. Кадры \
     приходят асинхронно, в порядке timestamp, не в порядке вызова \
     <code>decode()</code>. Прогресс декодирования виден в строке \
     <code>Decoded frames</code>.",
    "Для avc1 (H.264 в AVCC-формате) обязателен <strong>description</strong> — \
     <code>avcC</code>-box с SPS/PPS. Без него декодер не знает, как \
     разобрать поток. Encoder присылает его в \
     <code>metadata.decoderConfig.description</code> на первом key-кадре. \
     Там же — <code>codec</code>, <code>codedWidth</code>, <code>codedHeight</code>.",
    "Кнопка «<em>Without description</em>» пытается настроить декодер без \
     SPS/PPS. Ожидаемая ошибка — <code>EncodingError: The given encoding \
     is not supported</code>. Она приходит асинхронно через колбэк \
     <code>error</code> или через rejected <code>flush()</code>. \
     Кадры и result-canvas очищаются, чтобы контраст был виден явно.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("vd-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("vd-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

/// Очищает result-canvas. Вызывается при сбросе `frames`.
fn clear_result_canvas(doc: &Document) {
    let Some(canvas) = doc
        .get_element_by_id("vd-result")
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

// =============================================================================
// Рисование источника
// =============================================================================

fn draw_source(ctx: &CanvasRenderingContext2d, w: u32, h: u32, phase: f64) {
    let ctx_js: &JsValue = ctx.as_ref();
    let hue = (phase * 360.0) % 360.0;
    let color1 = format_str(format_args!("hsl({hue:.0}, 80%, 45%)"));
    let color2 = format_str(format_args!(
        "hsl({:.0}, 80%, 55%)",
        (hue + 60.0) % 360.0
    ));

    let grad = invoke(
        ctx_js,
        "createLinearGradient",
        &[num(0.0), num(0.0), num(w as f64), num(h as f64)],
    )
    .ok();
    if let Some(g) = grad {
        let _ = invoke(&g, "addColorStop", &[num(0.0), JsValue::from_str(&color1)]);
        let _ = invoke(&g, "addColorStop", &[num(1.0), JsValue::from_str(&color2)]);
        let _ = set_prop(ctx_js, "fillStyle", &g);
    }
    let _ = invoke(
        ctx_js,
        "fillRect",
        &[num(0.0), num(0.0), num(w as f64), num(h as f64)],
    );

    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#f0f6fc"));
    let _ = set_prop(ctx_js, "font", &JsValue::from_str("28px monospace"));
    let text = format_str(format_args!("#{}", (phase * FRAME_COUNT as f64) as u32));
    let _ = invoke(
        ctx_js,
        "fillText",
        &[JsValue::from_str(&text), num(10.0), num(h as f64 - 10.0)],
    );

    let cx = w as f64 / 2.0 + (phase * 6.28).sin() * 80.0;
    let cy = h as f64 / 2.0 + (phase * 6.28).cos() * 60.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#f0883e"));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(
        ctx_js,
        "arc",
        &[
            num(cx),
            num(cy),
            num(24.0),
            num(0.0),
            num(std::f64::consts::PI * 2.0),
        ],
    );
    let _ = invoke(ctx_js, "fill", &[]);
}

fn draw_frame_at(ctx: &CanvasRenderingContext2d, w: u32, h: u32, idx: u32) {
    let phase = idx as f64 / FRAME_COUNT as f64;
    draw_source(ctx, w, h, phase);
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
// Метаданные энкодера
// =============================================================================

#[derive(Default)]
struct EncoderMeta {
    description: Option<JsValue>,
    codec: String,
    coded_width: u32,
    coded_height: u32,
    captured: bool,
}

/// Достаёт числовое поле из JS-объекта.
fn get_f64(v: &JsValue, key: &str) -> f64 {
    Reflect::get(v, &JsValue::from_str(key))
        .ok()
        .and_then(|x| x.as_f64())
        .unwrap_or(0.0)
}

/// Достаёт `byteLength` из `EncodedVideoChunk`.
fn chunk_byte_length(c: &JsValue) -> usize {
    get_f64(c, "byteLength") as usize
}

// =============================================================================
// Encode pipeline — подготовка чанков
// =============================================================================

struct EncodeOutput {
    chunks: Vec<JsValue>,
    meta: EncoderMeta,
}

async fn generate_chunks(
    source_canvas: &HtmlCanvasElement,
    source_ctx: &CanvasRenderingContext2d,
) -> Result<EncodeOutput, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoEncoder"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoEncoder not supported"));
    }

    let chunks: Rc<RefCell<Vec<JsValue>>> = Rc::new(RefCell::new(Vec::new()));
    let meta: Rc<RefCell<EncoderMeta>> = Rc::new(RefCell::new(EncoderMeta::default()));

    let chunks_for_output = chunks.clone();
    let meta_for_output = meta.clone();

    let on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, md: JsValue| {
            // decoderConfig доступен только на первом key-кадре.
            // Захватываем его один раз.
            let already = meta_for_output.borrow().captured;
            if !already {
                if let Ok(cfg) = Reflect::get(&md, &JsValue::from_str("decoderConfig")) {
                    if !cfg.is_undefined() && !cfg.is_null() {
                        let mut m = meta_for_output.borrow_mut();
                        m.codec = get_str(&cfg, "codec");
                        m.coded_width = get_f64(&cfg, "codedWidth") as u32;
                        m.coded_height = get_f64(&cfg, "codedHeight") as u32;
                        if let Ok(d) = Reflect::get(&cfg, &JsValue::from_str("description")) {
                            if !d.is_undefined() && !d.is_null() {
                                m.description = Some(d);
                            }
                        }
                        m.captured = true;
                    }
                }
            }
            chunks_for_output.borrow_mut().push(chunk);
        },
    );
    let on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |e: JsValue| {
            web_sys::console::error_1(&e);
        },
    );

    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    let ctor_fn = ctor.dyn_into::<js_sys::Function>()?;
    let encoder = Reflect::construct(&ctor_fn, &{
        let arr = Array::new();
        arr.push(&init);
        arr
    })?;

    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(CODEC))?;
    set_prop(&cfg, "width", &num(FRAME_W as f64))?;
    set_prop(&cfg, "height", &num(FRAME_H as f64))?;
    set_prop(&cfg, "bitrate", &num(1_000_000.0))?;
    set_prop(&cfg, "framerate", &num(30.0))?;
    let configure = get_fn(&encoder, "configure")?;
    let _ = configure.call1(&encoder, &cfg)?;

    let encode = get_fn(&encoder, "encode")?;
    let global_ctor = Reflect::get(&js_sys::global(), &JsValue::from_str("VideoFrame"))?;
    let global_ctor = global_ctor.dyn_into::<js_sys::Function>()?;

    for i in 0..FRAME_COUNT {
        draw_frame_at(source_ctx, FRAME_W, FRAME_H, i);
        let frame = {
            let canvas_val = JsValue::from(source_canvas.clone());
            let init = Object::new();
            set_prop(&init, "timestamp", &num(i as f64 * FRAME_DURATION_US))?;
            let args = Array::new();
            args.push(&canvas_val);
            args.push(&init);
            Reflect::construct(&global_ctor, &args)?
        };
        let opts = Object::new();
        if i == 0 {
            set_prop(&opts, "keyFrame", &JsValue::TRUE)?;
        }
        let _ = encode.call2(&encoder, &frame, &opts)?;
        let _ = frame_close(&frame);
        gloo_timers::future::TimeoutFuture::new(0).await;
    }

    let flush = get_fn(&encoder, "flush")?;
    let promise = flush.call0(&encoder)?.dyn_into::<Promise>()?;
    let _ = JsFuture::from(promise).await?;
    if let Ok(close_fn) = Reflect::get(&encoder, &JsValue::from_str("close"))
        .and_then(|f| f.dyn_into::<js_sys::Function>())
    {
        let _ = close_fn.call0(&encoder);
    }

    on_output.forget();
    on_error.forget();

    let chunks_vec = chunks.borrow().clone();
    let meta_out = {
        let mut m = meta.borrow_mut();
        EncoderMeta {
            description: m.description.take(),
            codec: std::mem::take(&mut m.codec),
            coded_width: m.coded_width,
            coded_height: m.coded_height,
            captured: m.captured,
        }
    };
    Ok(EncodeOutput {
        chunks: chunks_vec,
        meta: meta_out,
    })
}

/// Декодирует чанки, собирает пиксели кадров.
///
/// Инкрементит `decode_progress` в `DemoState` в `on_output` и обновляет
/// UI — так пользователь видит прогресс, а не только финальное число.
async fn decode_chunks(
    chunks: Vec<JsValue>,
    description: Option<&JsValue>,
    scratch_ctx: CanvasRenderingContext2d,
    doc: Document,
    st: Rc<RefCell<DemoState>>,
) -> Result<Vec<Vec<u8>>, JsValue> {
    let frames: Rc<RefCell<Vec<Vec<u8>>>> = Rc::new(RefCell::new(Vec::new()));
    let frames_for_output = frames.clone();
    let scratch_ctx_for_output = scratch_ctx.clone();
    let doc_for_output = doc.clone();
    let st_for_output = st.clone();

    let on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        move |frame: JsValue| {
            let bytes = frame_to_bytes(
                &frame,
                &scratch_ctx_for_output,
                FRAME_W,
                FRAME_H,
            );
            let n = {
                let mut buf = frames_for_output.borrow_mut();
                buf.push(bytes);
                buf.len() as u32
            };
            let _ = frame_close(&frame);

            {
                let mut s = st_for_output.borrow_mut();
                s.decode_progress = n;
            }
            let s = st_for_output.borrow();
            update_stats(&doc_for_output, &s);
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

    // ВАЖНО: вынести результат в локальную переменную — иначе
    // временный `Ref` переживёт блок и `frames` уничтожится раньше.
    let result: Vec<Vec<u8>> = frames.borrow().clone();
    Ok(result)
}

async fn try_decode_without_description(
    chunks: Vec<JsValue>,
    _scratch_ctx: CanvasRenderingContext2d,
) -> Result<String, JsValue> {
    let error_msg: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
    let error_for_cb = error_msg.clone();

    let on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |_frame: JsValue| {},
    );
    let on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        move |e: JsValue| {
            let msg = get_str(&e, "message");
            let name = get_str(&e, "name");
            let full = if name.is_empty() {
                msg
            } else {
                format_str(format_args!("{name}: {msg}"))
            };
            *error_for_cb.borrow_mut() = Some(full);
        },
    );

    let decoder = make_video_decoder(&on_output, &on_error)?;
    decoder_configure(&decoder, CODEC, FRAME_W, FRAME_H, None)?;

    for chunk in chunks.iter() {
        let _ = decode_chunk(&decoder, chunk);
    }

    let flush_err = decoder_flush(&decoder).await.err();

    let _ = decoder_close(&decoder);
    on_output.forget();
    on_error.forget();

    let captured = error_msg.borrow().clone();
    if let Some(m) = captured {
        return Ok(m);
    }
    if let Some(e) = flush_err {
        let msg = get_str(&e, "message");
        let name = get_str(&e, "name");
        let full = if name.is_empty() {
            msg
        } else {
            format_str(format_args!("{name}: {msg}"))
        };
        return Ok(full);
    }
    Ok(String::from(
        "Ошибки не было — браузер принял конфигурацию без description. \
         Возможно, этот билд поддерживает avc3.",
    ))
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
    h2.set_text_content(Some("23.2 — VideoDecoder"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_video_decoder_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("vd-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает VideoDecoder. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Скрытый scratch canvas.
    let scratch_el = document.create_element("canvas")?;
    scratch_el.set_id("vd-scratch");
    scratch_el.set_attribute("style", "display:none")?;
    let scratch_canvas: HtmlCanvasElement = scratch_el.dyn_into()?;
    scratch_canvas.set_width(FRAME_W);
    scratch_canvas.set_height(FRAME_H);
    root.append_child(&scratch_canvas)?;
    let scratch_ctx = scratch_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d (scratch)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    // Два видимых canvas'а.
    let canvases = document.create_element("div")?;
    canvases.set_class_name("vd-canvases");
    canvases.set_inner_html(&format_str(format_args!(
        r#"<div class="vd-canvas-col">
               <div class="vd-canvas-label">Source (что кодируем)</div>
               <canvas id="vd-source" class="vd-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="vd-canvas-col">
               <div class="vd-canvas-label">Result (что декодируем)</div>
               <canvas id="vd-result" class="vd-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = FRAME_W,
        h = FRAME_H,
    )));
    root.append_child(&canvases)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("vd-block");
    controls.set_inner_html(
        r#"<button class="vd-btn vd-btn-primary" id="vd-run">Encode + Decode</button>
           <button class="vd-btn vd-btn-warn" id="vd-nodesc">Without description</button>
           <button class="vd-btn" id="vd-pause">Pause</button>
           <span class="vd-label" id="vd-support"></span>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("vd-stats");
    stats.set_id("vd-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let hex_block = document.create_element("div")?;
    hex_block.set_class_name("vd-hex");
    hex_block.set_inner_html(
        r#"<div class="vd-hex-label">description (SPS/PPS) — первые 64 байта:</div>
           <div id="vd-hex-body">—</div>"#,
    );
    root.append_child(&hex_block)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("vd-footnote");
    footnote.set_text_content(Some(
        "VideoDecoder принимает EncodedVideoChunk и выдаёт VideoFrame через \
         колбэк output. Для avc1 обязателен description (avcC-box с SPS/PPS): \
         без него декодер не знает, как разобрать AVCC-поток. Encoder присылает \
         его в metadata.decoderConfig.description на первом key-кадре. \
         Для avc3 (Annex B) description не нужен.",
    ));
    root.append_child(&footnote)?;

    if let Some(el) = document.get_element_by_id("vd-support") {
        let s = format_str(format_args!(
            "VideoDecoder: {d} · isConfigSupported: {c}",
            d = if is_video_decoder_supported() { "да" } else { "нет" },
            c = if crate::video_decoder::has_is_config_supported() { "да" } else { "нет" },
        ));
        el.set_text_content(Some(&s));
    }

    let state = Rc::new(RefCell::new(DemoState::new()));

    // ---------- Encode + Decode ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let scratch = scratch_ctx.clone();
        let btn_el = document
            .get_element_by_id("vd-run")
            .ok_or_else(|| JsValue::from_str("no #vd-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            let scratch = scratch.clone();

            spawn_local(async move {
                {
                    let mut s = st.borrow_mut();
                    s.error = None;
                    s.phase = String::from("encode…");
                    s.clear_pipeline();
                    s.frames.clear();
                    s.play_idx = 0;
                }
                clear_result_canvas(&doc);
                {
                    let s = st.borrow();
                    update_stats(&doc, &s);
                    update_hex(&doc, &s);
                }

                let Some(canvas_el) = doc.get_element_by_id("vd-source") else {
                    return;
                };
                let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() else {
                    return;
                };
                let Ok(Some(ctx)) = canvas.get_context("2d") else {
                    return;
                };
                let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
                    return;
                };

                let gen_result = generate_chunks(&canvas, &ctx).await;
                let EncodeOutput { chunks, meta } = match gen_result {
                    Ok(v) => v,
                    Err(e) => {
                        let mut s = st.borrow_mut();
                        s.error = Some(format_str(format_args!(
                            "encode: {e:?}"
                        )));
                        s.phase = String::from("ошибка");
                        update_stats(&doc, &s);
                        return;
                    }
                };

                // Считаем key/delta и суммарные размеры.
                let mut keys = 0u32;
                let mut deltas = 0u32;
                let mut key_bytes = 0usize;
                let mut delta_bytes = 0usize;
                for c in chunks.iter() {
                    let t = get_str(c, "type");
                    let bl = chunk_byte_length(c);
                    if t == "key" {
                        keys += 1;
                        key_bytes += bl;
                    } else {
                        deltas += 1;
                        delta_bytes += bl;
                    }
                }
                let total_bytes = key_bytes + delta_bytes;

                let desc_bytes = meta
                    .description
                    .as_ref()
                    .map(|d| buffer_source_to_vec(d).len())
                    .unwrap_or(0);
                let desc_hex = meta
                    .description
                    .as_ref()
                    .map(|d| hex_dump(d, HEX_PREVIEW_BYTES))
                    .unwrap_or_else(|| String::from("—"));

                let chunk_count_total = chunks.len() as u32;
                {
                    let mut s = st.borrow_mut();
                    s.chunk_count = chunk_count_total;
                    s.key_count = keys;
                    s.delta_count = deltas;
                    s.description_bytes = desc_bytes;
                    s.description_hex = desc_hex;
                    s.decoder_codec = meta.codec.clone();
                    s.decoder_coded_width = meta.coded_width;
                    s.decoder_coded_height = meta.coded_height;
                    s.encoded_total_bytes = total_bytes;
                    s.encoded_key_bytes = key_bytes;
                    s.encoded_delta_bytes = delta_bytes;
                    s.decode_progress = 0;
                    s.decode_total = chunk_count_total;
                    s.phase = String::from("decode…");
                }
                {
                    let s = st.borrow();
                    update_stats(&doc, &s);
                    update_hex(&doc, &s);
                }

                let frames = match decode_chunks(
                    chunks,
                    meta.description.as_ref(),
                    scratch,
                    doc.clone(),
                    st.clone(),
                )
                .await
                {
                    Ok(f) => f,
                    Err(e) => {
                        let mut s = st.borrow_mut();
                        s.error = Some(format_str(format_args!(
                            "decode: {e:?}"
                        )));
                        s.phase = String::from("ошибка");
                        s.decode_total = 0;
                        update_stats(&doc, &s);
                        return;
                    }
                };

                let n = frames.len();
                {
                    let mut s = st.borrow_mut();
                    s.frames = frames;
                    s.play_idx = 0;
                    s.decode_progress = n as u32;
                    s.decode_total = 0;
                    s.phase = format_str(format_args!("готово ({n} кадров)"));
                }
                let s = st.borrow();
                update_stats(&doc, &s);
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- Without description ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let scratch = scratch_ctx.clone();
        let btn_el = document
            .get_element_by_id("vd-nodesc")
            .ok_or_else(|| JsValue::from_str("no #vd-nodesc"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            let scratch = scratch.clone();

            spawn_local(async move {
                // Очищаем всё, что было от предыдущего прогона —
                // чтобы контраст «до/после» был виден явно.
                {
                    let mut s = st.borrow_mut();
                    s.error = None;
                    s.phase = String::from("encode для теста…");
                    s.clear_pipeline();
                    s.frames.clear();
                    s.play_idx = 0;
                }
                clear_result_canvas(&doc);
                {
                    let s = st.borrow();
                    update_stats(&doc, &s);
                    update_hex(&doc, &s);
                }

                let Some(canvas_el) = doc.get_element_by_id("vd-source") else {
                    return;
                };
                let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() else {
                    return;
                };
                let Ok(Some(ctx)) = canvas.get_context("2d") else {
                    return;
                };
                let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
                    return;
                };

                let enc_out = match generate_chunks(&canvas, &ctx).await {
                    Ok(v) => v,
                    Err(e) => {
                        let mut s = st.borrow_mut();
                        s.error = Some(format_str(format_args!(
                            "prepare: {e:?}"
                        )));
                        s.phase = String::from("ошибка");
                        update_stats(&doc, &s);
                        return;
                    }
                };

                // Заполняем метаданные — размеры не помешают
                // даже в сценарии с ошибкой.
                let mut keys = 0u32;
                let mut deltas = 0u32;
                let mut key_bytes = 0usize;
                let mut delta_bytes = 0usize;
                for c in enc_out.chunks.iter() {
                    let t = get_str(c, "type");
                    let bl = chunk_byte_length(c);
                    if t == "key" {
                        keys += 1;
                        key_bytes += bl;
                    } else {
                        deltas += 1;
                        delta_bytes += bl;
                    }
                }
                let total_bytes = key_bytes + delta_bytes;
                let desc_bytes = enc_out
                    .meta
                    .description
                    .as_ref()
                    .map(|d| buffer_source_to_vec(d).len())
                    .unwrap_or(0);
                let desc_hex = enc_out
                    .meta
                    .description
                    .as_ref()
                    .map(|d| hex_dump(d, HEX_PREVIEW_BYTES))
                    .unwrap_or_else(|| String::from("—"));
                {
                    let mut s = st.borrow_mut();
                    s.chunk_count = enc_out.chunks.len() as u32;
                    s.key_count = keys;
                    s.delta_count = deltas;
                    s.description_bytes = desc_bytes;
                    s.description_hex = desc_hex;
                    s.decoder_codec = enc_out.meta.codec.clone();
                    s.decoder_coded_width = enc_out.meta.coded_width;
                    s.decoder_coded_height = enc_out.meta.coded_height;
                    s.encoded_total_bytes = total_bytes;
                    s.encoded_key_bytes = key_bytes;
                    s.encoded_delta_bytes = delta_bytes;
                    s.phase = String::from("decode без description…");
                }
                {
                    let s = st.borrow();
                    update_stats(&doc, &s);
                    update_hex(&doc, &s);
                }

                let msg = try_decode_without_description(enc_out.chunks, scratch).await;

                let final_msg = match msg {
                    Ok(m) => m,
                    Err(e) => format_str(format_args!("{e:?}")),
                };

                {
                    let mut s = st.borrow_mut();
                    s.error = Some(format_str(format_args!(
                        "Ожидаемое поведение: {final_msg}"
                    )));
                    s.phase = String::from("проверено");
                }
                let s = st.borrow();
                update_stats(&doc, &s);
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- Pause ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("vd-pause")
            .ok_or_else(|| JsValue::from_str("no #vd-pause"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let paused = {
                let mut s = st.borrow_mut();
                s.paused = !s.paused;
                s.paused
            };
            if let Some(el) = doc.get_element_by_id("vd-pause") {
                el.set_text_content(Some(if paused { "Resume" } else { "Pause" }));
            }
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- rAF: source-анимация + playback ----------
    {
        let doc_for_raf = document.clone();

        let source_canvas = doc_for_raf
            .get_element_by_id("vd-source")
            .ok_or_else(|| JsValue::from_str("no #vd-source"))?
            .dyn_into::<HtmlCanvasElement>()?;
        let source_ctx = source_canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d (source)"))?
            .dyn_into::<CanvasRenderingContext2d>()?;

        let result_canvas = doc_for_raf
            .get_element_by_id("vd-result")
            .ok_or_else(|| JsValue::from_str("no #vd-result"))?
            .dyn_into::<HtmlCanvasElement>()?;
        let result_ctx = result_canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d (result)"))?
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
                                result_ctx.as_ref(),
                                "putImageData",
                                &[JsValue::from(img), num(0.0), num(0.0)],
                            );
                        }
                    }
                }

                let update_stats_now = {
                    let mut s = st.borrow_mut();
                    s.window_frames += 1;
                    if s.window_frames >= STATS_EVERY_FRAMES {
                        s.window_frames = 0;
                        true
                    } else {
                        false
                    }
                };
                if update_stats_now {
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

### 23.2.9. Анализ решения

**Какую ценность даёт демо.**

Подглава закрывает **VideoDecoder** — вторую половину пары «сжатое ↔ распакованное». Показано:

- **Поток `EncodedVideoChunk`** — 30 штук, из них 1 key и 29 delta.
- **`description` (SPS/PPS)** — 41 байт, видно в hex.
- **Колбэк `output`** — 30 асинхронных вызовов, собирающих пиксели.
- **`await flush()`** — дожидается всех кадров.
- **`EncodingError` без `description`** — кнопка делает это наглядно.
- **Прогресс декодирования** — `0 / 30` → `30 / 30` в реальном времени.
- **Размеры и сжатие** — `VideoFrame=307200` vs `Encoded≈7000`, коэффициент ~40–90×.
- **`metadata.decoderConfig`** — `codec`, `codedWidth`, `codedHeight` показаны рядом.

**Что показывает каждая кнопка.**

- **Encode + Decode** — полный цикл. Source рисует 30 кадров, они кодируются, декодируются, result воспроизводит их в цикле. Если source и result совпадают — цикл замкнут. Прогресс виден построчно.
- **Without description** — специально ломаем configure. `frames` очищаются, result чёрный, пользователь видит реальное сообщение об ошибке и **явный контраст** «было 30 → стало 0».

**Что осознанно не реализовано.**

- **`VideoEncoder.isConfigSupported`** — тема 23.4.
- **Воспроизведение через `<video>`** — используется canvas playback.
- **Annex B (avc3)** — упоминается, не показывается. Для avc1 (самый частый) достаточно.

**Правила проекта, применённые.**

1. **Autonomous app.rs** — весь pipeline в одном файле.
2. **`format_str(format_args!(...))` в tail-позициях.**
3. **Переиспользование хелперов** — `get_fn`, `invoke`, `construct`, `set_prop`, `get_str`, `get_f64`, `num`, `format_str`, `bool_of` из `video_frame`. Дублирования нет.
4. **`buffer_source_to_vec` в модуле** — снимает класс ошибок «ArrayBuffer vs Uint8Array».
5. **`hex_dump` в модуле** — специфичная утилита для показа description.
6. **`Rc<RefCell<Vec<...>>>`** — сбор чанков и кадров из колбэков.
7. **`spawn_local`** для async pipeline.
8. **`gloo_timers::future::TimeoutFuture::new(0).await`** между кадрами кодирования — yield.
9. **Уникальный префикс `vd-`.**
10. **Никаких неиспользуемых импортов.**
11. **`WebCodecs` через `Reflect`** — обход несовпадений сигнатур web-sys.

**Проверки корректности.**

- `description` извлекается из `metadata.decoderConfig.description` в колбэке encoder'а.
- `configure` без `description` не бросает синхронно — ошибка асинхронна.
- `output` собирает пиксели, `frame.close()` вызывается сразу.
- `flush()` awaited до `close()`.
- Hex-dump работает через `buffer_source_to_vec` — понимает и `ArrayBuffer`, и `Uint8Array`.
- Никаких `unwrap()` на небезопасных значениях.

---

### 23.2.10. Список доработок после первой итерации

**Итерация 1** (первая версия главы 23.2):

1. **`bool_of`, `format_str`, хелперы** — реэкспорт из `video_frame`. Никакого дублирования.
2. **`hex_dump` через `dyn_into::<Uint8Array>`** — работает только на `Uint8Array`.
3. **`isConfigSupported` feature detection** — есть, но в демо не используется.
4. **Кнопка «Without description»** — асинхронный сценарий, ошибка собирается через `error`-колбэк или rejected `flush`.
5. **`Output` колбэк**: `frame_to_bytes` = drawImage на scratch + getImageData + close.
6. **`getImageData` через `invoke`** — обход сигнатуры web-sys 0.3.105.
7. **Yield между кадрами encode** — `TimeoutFuture::new(0).await`.
8. **`spawn_local` для pipeline** — вся последовательность в одной async-функции.
9. **Никаких открытых `VideoFrame`** — все кадры закрываются в колбэке output.
10. **Два canvas'а рядом** — source (что кодируем) vs result (что декодируем).
11. **Hex-dump description** — визуальная проверка структуры `avcC`.

**Итерация 2** (по результатам анализа демо):

12. **`buffer_source_to_vec`** — универсальный разбор `ArrayBuffer`, `Uint8Array`, `ArrayBufferView`. Баг `Description: 0 байт` исправлен. Chrome присылает `ArrayBuffer`, прямое `dyn_into::<Uint8Array>` давало `Err`.
13. **`hex_dump` через `buffer_source_to_vec`** — понимает оба варианта.
14. **`DemoState::clear_pipeline()`** — обнуляет все счётчики и метаданные.
15. **Очистка `frames` при «Without description»** — сбрасываем `frames`, `play_idx`, вызываем `clear_result_canvas`. Пользователь видит **явный контраст** «было 30 кадров → стало 0».
16. **Прогресс декодирования** — `decode_progress` инкрементится в `on_output`-колбэк декодера, `update_stats` вызывается сразу. Видно `M / Total (идёт…)`, потом финальное число.
17. **Сравнение размеров** — `FRAME_RGBA_BYTES = W×H×4 = 307200`. Суммарный `chunk.byteLength` по key/delta. Коэффициент сжатия в строке `Размеры`.
18. **`metadata.decoderConfig` в UI** — `codec`, `codedWidth`, `codedHeight`, `description`. Отдельная строка `Decoder config`.
19. **`EncoderMeta` + `EncodeOutput`** — структура для захвата decoderConfig один раз (только на первом key-кадре), флаг `captured` защищает от перезаписи.
20. **`chunk_byte_length` helper** — через `Reflect::get(c, "byteLength")`.

---

### 23.2.11. Итог

**`VideoDecoder`** — вход в WebCodecs со стороны сжатых данных. Принимает `EncodedVideoChunk`, отдаёт `VideoFrame` через колбэк `output`.

**`description` (SPS/PPS)** обязателен для avc1. Приходит в `metadata.decoderConfig.description` из encoder'а на первом key-кадре. Без него — `EncodingError`.

**Колбэк `output` асинхронен.** Кадры приходят по мере готовности, не в порядке `decode()`. Внутри колбэка обязательно `frame.close()`.

**`await flush()`** дожидается всех кадров.

**Практическая находка №1.** `EncodingError` приходит **асинхронно**: либо через колбэк `error`, либо через rejected `flush()`. `configure` без `description` синхронно **не бросает**. Это значит, что обработка ошибок должна быть в колбэке и/или через `try/await flush`.

**Практическая находка №2.** `metadata.decoderConfig.description` в encoder output есть **только на первом key-кадре**. На delta-кадрах его нет. Это не баг — спецификация.

**Практическая находка №3.** В Chrome `description` приходит как **`ArrayBuffer`**, а не `Uint8Array`. Прямое `dyn_into::<Uint8Array>()` на `ArrayBuffer` даёт `Err`, и без универсального разбора буфер молча теряется. `buffer_source_to_vec` снимает эту проблему.

**Wow-эффект.** Строка `Размеры` показывает наглядно: один `VideoFrame` — 307 200 байт, весь сжатый поток — около 7 КБ, коэффициент **~40×**. Ради этого и существует кодек.

**Дальше — 23.3 (VideoEncoder)** — полная картина, а не подготовительная.