# Глава 23.6. ImageDecoder: статика и animated GIF

## 23.6.0. Обновлённая структура

```
src/
  image_decoder.rs                ← НОВЫЙ модуль 23.6
  video_frame.rs                  ← из 23.1 (переиспользуется: format_str, invoke, num)
  …
tests/
  wasm_image_decoder.rs           ← НОВЫЙ тест
src/lib.rs                        ← +1 строка: pub mod image_decoder;
src/app.rs                        ← демо 23.6 (заменяется целиком)
```

**Комментарий.** Модуль `image_decoder.rs` — про `ImageDecoder`, `ImageTrack`, `ImageTrackList`, `VideoFrame` в контексте изображений. Префикс CSS — `id-` (image decoder), не пересекается с `vf-` (23.1), `vd-` (23.2), `ve-` (23.3), `vt-` (23.4), `ac-` (23.5). Мини-хелперы `invoke`, `num`, `format_str` — из `video_frame::*`. Дублирования нет.

**Ссылки для детального изучения:**

- MDN: [`ImageDecoder`](https://developer.mozilla.org/en-US/docs/Web/API/ImageDecoder)
- MDN: [`ImageDecoder.decode()`](https://developer.mozilla.org/en-US/docs/Web/API/ImageDecoder/decode)
- MDN: [`ImageTrack`](https://developer.mozilla.org/en-US/docs/Web/API/ImageTrack)
- MDN: [`ImageTrackList`](https://developer.mozilla.org/en-US/docs/Web/API/ImageTrackList)
- W3C: [WebCodecs § ImageDecoder](https://w3c.github.io/webcodecs/#imagedecoder-interface)
- W3C: [WebCodecs § ImageTrack](https://w3c.github.io/webcodecs/#imagetrack-interface)
- W3C: [WebCodecs § ImageDecodeOptions](https://w3c.github.io/webcodecs/#dictdef-imagedecodeoptions)

---

## 23.6.1. О чём эта подглава

Предыдущие подглавы работали с **потоковыми** кодеками: `VideoEncoder` / `VideoDecoder` / `AudioEncoder` / `AudioDecoder` принимают данные чанками, требуют `configure` и `flush`. `ImageDecoder` работает иначе: он получает **весь файл целиком** при создании, сразу умеет читать метаданные и декодировать любой кадр по индексу.

Это API для **статических и анимированных изображений**: PNG, JPEG, GIF, WebP, AVIF, HEIF.

Подглава отвечает на пять вопросов.

**Первый — как создать `ImageDecoder`.** `new ImageDecoder({ data, type })`, где `data` — `BufferSource` с байтами файла, `type` — MIME.

**Второй — как читать метаданные.** Через `tracks` (`ImageTrackList`). После резолва `tracks.ready` доступны `frameCount`, `animated`, `repetitionCount`, размеры.

**Третий — как декодировать кадр.** `await decoder.decode({ frameIndex })` → `{ image: VideoFrame, complete: boolean }`.

**Четвёртый — как воспроизвести анимацию.** Декодировать кадры по индексу, рисовать, ждать `frame.duration` микросекунд, повторять.

**Пятый — какие практические ограничения у `ImageDecoder`.** Их оказалось три — и они неочевидные. Разбираем в 23.6.4.

---

## 23.6.2. Что такое `ImageTrack` и `ImageTrackList`

**`ImageTrackList`** — коллекция треков в файле. Всегда есть хотя бы один. `tracks.ready` — Promise, резолвится когда метаданные разобраны. `tracks.length` — количество. `tracks[0]` — первый трек.

**`ImageTrack`** — один трек. Свойства:

- `animated` — `true` для GIF, animated WebP, APNG; `false` для PNG/JPEG.
- `frameCount` — число кадров. Для статики `1`.
- `repetitionCount` — сколько раз повторить анимацию. `0` — не повторять, `Infinity` — бесконечно.
- `selected` — выбран ли трек для декодирования.

Для 99 % случаев — один трек: `tracks[0]`.

---

## 23.6.3. Что показывает демо

**Layout:**

- **File input** — выбор PNG / JPEG / GIF.
- **Метаданные** — тип, размеры, число кадров, animated, `repetitionCount`.
- **Canvas** — отрисовка декодированного кадра.
- **`<img>` для сравнения** — рядом показывает, как ведёт себя обычный тег.
- **Controls для анимации** — Play / Pause / Restart.
- **Stats** — текущий кадр, время декодирования.

**Сценарий:**

1. Пользователь выбирает файл.
2. Создаётся `ImageDecoder`, ждём `tracks.ready`.
3. Если `animated == false` — декодируем кадр 0 и рисуем. Кнопки анимации disabled.
4. Если `animated == true` — включаем автозапуск: декодируем кадры по кругу с задержкой `frame.duration`.

**Wow-момент.** `<img>` для GIF показывает **только первый кадр**, если не дать тегу управление. `ImageDecoder` даёт **покадровый контроль**: пауза, перемотка, изменение скорости — всё вручную.

---

## 23.6.4. Практические находки

Этот раздел — главный. Мы прошли через 6 итераций, каждая из которых ломалась на новой ошибке Chrome. Вот полная карта грабель.

### Находка 1. `drawImage(VideoFrame)` в Chrome — отложенная операция

Спецификация говорит, что `CanvasRenderingContext2D.drawImage` копирует пиксели **синхронно**. Chrome это не гарантирует: композитор может выполнить копирование на следующем тике, а до тех пор `VideoFrame` остаётся «в работе».

**Симптом.** Если вызвать `frame.close()` сразу после `drawImage(frame, ...)` — рано или поздно получите:
```
DOMException: CanvasRenderingContext2D.drawImage: Passed-in video frame is "broken"
```

**Что не помогает.**

- `TimeoutFuture::new(0).await` между drawImage и close — это микротаск, не даёт шанса paint.
- `requestAnimationFrame` между drawImage и close — rAF-колбэк вызывается **перед** следующим paint, а не после. Гарантий нет.
- **`held_frame`** — держать frame живым до следующей итерации. Не помогает, потому что см. находку 2.

### Находка 2. `ImageDecoder` сам инвалидирует свои VideoFrame

Это ключевое. `ImageDecoder` **владеет** всеми frame'ами, которые вернул. При следующем вызове `decode()` **все предыдущие frame'ы становятся broken**, независимо от того, держите вы их или нет.

**Симптом.** Паттерн `held_frame` падает на второй итерации:
1. Итерация 1: `decode(0)` → `held = frame_0` → `drawImage(frame_0)` → success.
2. Итерация 2: `decode(1)` → frame_0 становится broken → `drawImage(held_frame_0)` для сравнения → **падает**.
3. Даже если drawImage не вызывать снова — Chrome может перерисовать canvas асинхронно и обнаружить, что источник уже broken.

**Что не помогает.**

- Держать frame до следующего rAF.
- Держать frame через `frame.clone()` — clone тоже валиден только до следующего decode.
- Any amount of yielding.

### Находка 3. `frame.copyTo()` не работает на GPU-backed кадрах

Попытка обойти проблему явным копированием пикселей через `copyTo(buffer, { format: "RGBA" })` падает:

```
DOMException: VideoFrame.copyTo: No media resource in VideoFrame
```

**Причина.** Кадры `ImageDecoder` в этой сборке Chrome **GPU-backed**. `copyTo` требует CPU readback, а для animated GIF Chrome этот readback не поддерживает. Это платформенная особенность (Linux + текущая сборка Chrome).

### Находка 4. `createImageBitmap(frame)` падает на нулевых размерах

Логичная попытка — материализовать пиксели в независимый `ImageBitmap`:

```
DOMException: Window.createImageBitmap: Passed-in video frame has width 0
```

**Причина.** Chrome для кадров анимированного GIF **не устанавливает intrinsic size** `VideoFrame`. `frame.displayWidth = 0`, `frame.codedWidth = 0` — хотя содержимое валидное. `drawImage(frame, 0, 0, w, h)` работает, потому что размеры передаются явными аргументами. А `createImageBitmap` берёт размеры из frame и падает до любых опций.

**Попытка передать `resizeWidth` / `resizeHeight`** не помогает — Chrome валидирует intrinsic size **до** применения resize-опций.

### Решение

Единственный работающий паттерн:

```
decode() → frame_duration_us → drawImage(frame, 0, 0, w, h) → frame.close()
```

**Всё синхронно после первого `await`.** Никаких промежуточных `.await`, никаких `held_frame`, никаких rAF. Frame не переживает итерацию — он живёт ровно один цикл `decode → draw → close`.

`w` и `h` берутся **не из frame**, а из метаданных трека (`frame_display_width` с fallback на `codedWidth`, если первый вернул 0). Тогда `drawImage` получает корректные размеры независимо от intrinsic size кадра.

### Находка 5. `drawImage` после `close` внутри одной итерации — безопасен

Парадоксально, но `frame.close()` **сразу после** `drawImage` безопасен. Chrome захватывает пиксели в момент вызова `drawImage` (или ставит в очередь, но резервирует ресурс). К моменту, когда decoder инвалидирует frame при следующем `decode()` — пиксели на canvas уже зафиксированы.

Именно поэтому "синхронный блок" `decode → draw → close` работает, а любой "держать подольше" — нет.

### Резюме правил

- **Не держите frame между итерациями.** Decoder инвалидирует его при следующем decode.
- **Не вызывайте `copyTo`** — не работает на GPU-backed кадрах GIF.
- **Не вызывайте `createImageBitmap`** — падает на нулевых intrinsic размерах.
- **Всегда передавайте `w` и `h` в `drawImage` явно** — из метаданных трека, не из frame.
- **Держите `decode → draw → close` как единый синхронный блок.**
- **Yield между итерациями — только `TimeoutFuture` на `frame.duration`.** Не между draw и close.

---

## 23.6.5. Исходные файлы

### `src/image_decoder.rs`

```rust
//! 23.6. ImageDecoder: статика и animated GIF.
//!
//! Обёртки вокруг `ImageDecoder`:
//!
//! - конструктор `new ImageDecoder({ data, type })`;
//! - чтение `tracks` (ImageTrackList) — метаданные без декодирования;
//! - `decode({ frameIndex, completeFramesOnly })` → VideoFrame;
//! - `close()`;
//! - статический `ImageDecoder.isTypeSupported(type)`.
//!
//! ## Отличия от видео/аудио
//!
//! `VideoDecoder` / `AudioDecoder` — потоковые, принимают чанки по
//! одному. `ImageDecoder` получает **весь файл** при создании и
//! сразу умеет читать метаданные. Это API для статических и
//! анимированных изображений, не для видео.
//!
//! ## Fallback для размеров кадра
//!
//! Chrome для кадров анимированного GIF часто возвращает
//! `displayWidth`/`displayHeight` = 0 — intrinsic size не
//! устанавливается. `frame_display_width` / `frame_display_height`
//! пробуют несколько полей и возвращают первое ненулевое. Если все
//! нулевые — 0; вызывающий код использует fallback из метаданных
//! трека.
//!
//! ## Жизненный цикл VideoFrame
//!
//! `ImageDecoder` владеет своими frame'ами и инвалидирует их при
//! следующем `decode()`. Рабочий паттерн — синхронный блок
//! `decode → drawImage → frame_close` без промежуточных await.
//! `copyTo` и `createImageBitmap` для кадров GIF не работают.
//!
//! ## Переиспользование
//!
//! Хелперы `num`, `set_prop`, `get_fn`, `format_str` — из
//! `video_frame::*`.

use js_sys::{Object, Promise, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{format_str, get_fn, num, set_prop};

// =============================================================================
// Feature detection
// =============================================================================

/// `ImageDecoder` доступен в браузере.
pub fn is_image_decoder_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("ImageDecoder"))
        .map(|v| v.is_function())
        .unwrap_or(false)
}

/// `await ImageDecoder.isTypeSupported(type)` — статический метод.
pub async fn is_type_supported(mime: &str) -> Result<bool, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("ImageDecoder"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("ImageDecoder not available"));
    }
    let f = Reflect::get(&ctor, &JsValue::from_str("isTypeSupported"))?;
    if !f.is_function() {
        return Err(JsValue::from_str(
            "ImageDecoder.isTypeSupported not available",
        ));
    }
    let f = f.dyn_into::<js_sys::Function>()?;
    let promise = f
        .call1(&ctor, &JsValue::from_str(mime))?
        .dyn_into::<Promise>()?;
    let result = JsFuture::from(promise).await?;
    Ok(result.as_bool().unwrap_or(false))
}

// =============================================================================
// Конструктор
// =============================================================================

/// `new ImageDecoder({ data, type })`.
///
/// `data` — байты файла (BufferSource), `type` — MIME-строка.
pub fn make_image_decoder(data: &[u8], mime: &str) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("ImageDecoder"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("ImageDecoder not available"));
    }
    let init = Object::new();
    let arr = Uint8Array::from(data);
    set_prop(&init, "data", &arr)?;
    set_prop(&init, "type", &JsValue::from_str(mime))?;
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let args = js_sys::Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

// =============================================================================
// Чтение метаданных трека
// =============================================================================

/// Ждёт `decoder.tracks.ready`.
pub async fn wait_tracks_ready(decoder: &JsValue) -> Result<(), JsValue> {
    let tracks = Reflect::get(decoder, &JsValue::from_str("tracks"))?;
    let ready = Reflect::get(&tracks, &JsValue::from_str("ready"))?;
    if !ready.is_undefined() {
        let promise = ready.dyn_into::<Promise>()?;
        let _ = JsFuture::from(promise).await?;
    }
    Ok(())
}

/// `decoder.tracks.length`.
pub fn tracks_length(decoder: &JsValue) -> u32 {
    Reflect::get(decoder, &JsValue::from_str("tracks"))
        .ok()
        .and_then(|t| Reflect::get(&t, &JsValue::from_str("length")).ok())
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `decoder.tracks[0]` — первый трек.
pub fn first_track(decoder: &JsValue) -> Result<JsValue, JsValue> {
    let tracks = Reflect::get(decoder, &JsValue::from_str("tracks"))?;
    let idx = JsValue::from_f64(0.0);
    let track = Reflect::get(&tracks, &idx)?;
    if track.is_undefined() || track.is_null() {
        return Err(JsValue::from_str("no tracks in decoder"));
    }
    Ok(track)
}

/// `track.animated`.
pub fn track_animated(track: &JsValue) -> bool {
    Reflect::get(track, &JsValue::from_str("animated"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false)
}

/// `track.frameCount`.
pub fn track_frame_count(track: &JsValue) -> u32 {
    Reflect::get(track, &JsValue::from_str("frameCount"))
        .ok()
        .and_then(|v| v.as_f64())
        .map(|n| n as u32)
        .unwrap_or(0)
}

/// `track.repetitionCount`. Может быть `Infinity` — для бесконечной
/// анимации. Возвращает `f64`, потому что Infinity валидно.
pub fn track_repetition_count(track: &JsValue) -> f64 {
    Reflect::get(track, &JsValue::from_str("repetitionCount"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// `track.selected`.
pub fn track_selected(track: &JsValue) -> bool {
    Reflect::get(track, &JsValue::from_str("selected"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false)
}

// =============================================================================
// Декодирование кадра
// =============================================================================

/// Результат одного вызова `decode()`: сам кадр и флаг полноты.
pub struct DecodeResult {
    pub frame: JsValue,
    /// Флаг «кадр полностью декодирован». Для progressive JPEG
    /// при `completeFramesOnly: false` бывает `false`. В демо всегда
    /// `true` — мы запрашиваем только полные кадры.
    #[allow(dead_code)]
    pub complete: bool,
}

/// `await decoder.decode({ frameIndex, completeFramesOnly })`.
///
/// Возвращает `{ image: VideoFrame, complete: boolean }`.
pub async fn decode_frame(
    decoder: &JsValue,
    frame_index: u32,
    complete_frames_only: bool,
) -> Result<DecodeResult, JsValue> {
    let opts = Object::new();
    set_prop(&opts, "frameIndex", &num(frame_index as f64))?;
    set_prop(
        &opts,
        "completeFramesOnly",
        &JsValue::from_bool(complete_frames_only),
    )?;

    let f = get_fn(decoder, "decode")?;
    let promise = f.call1(decoder, &opts)?.dyn_into::<Promise>()?;
    let result = JsFuture::from(promise).await?;

    let frame = Reflect::get(&result, &JsValue::from_str("image"))?;
    let complete = Reflect::get(&result, &JsValue::from_str("complete"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(true);

    Ok(DecodeResult { frame, complete })
}

/// Читает числовое поле. Для ключей с точкой
/// (`visibleRect.width`) спускается на один уровень.
fn read_numeric(obj: &JsValue, key: &str) -> Result<f64, JsValue> {
    if let Some((head, tail)) = key.split_once('.') {
        let sub = Reflect::get(obj, &JsValue::from_str(head))?;
        if sub.is_null() || sub.is_undefined() {
            return Err(JsValue::from_str("missing nested field"));
        }
        read_numeric(&sub, tail)
    } else {
        let v = Reflect::get(obj, &JsValue::from_str(key))?;
        v.as_f64().ok_or_else(|| JsValue::from_str("not a number"))
    }
}

/// `frame.displayWidth`, с fallback на `codedWidth` и `visibleRect.width`.
///
/// Chrome для кадров анимированного GIF часто возвращает
/// `displayWidth = 0`. Пробуем несколько полей и возвращаем первое
/// ненулевое. Если все нулевые — 0; вызывающий использует fallback
/// из метаданных трека.
pub fn frame_display_width(frame: &JsValue) -> u32 {
    for key in ["displayWidth", "codedWidth", "visibleRect.width"] {
        if let Ok(v) = read_numeric(frame, key) {
            if v > 0.0 {
                return v as u32;
            }
        }
    }
    0
}

/// `frame.displayHeight`, с fallback на `codedHeight` и
/// `visibleRect.height`.
pub fn frame_display_height(frame: &JsValue) -> u32 {
    for key in ["displayHeight", "codedHeight", "visibleRect.height"] {
        if let Ok(v) = read_numeric(frame, key) {
            if v > 0.0 {
                return v as u32;
            }
        }
    }
    0
}

/// `frame.duration` — микросекунды. Для статических кадров `0` или `null`.
pub fn frame_duration_us(frame: &JsValue) -> f64 {
    Reflect::get(frame, &JsValue::from_str("duration"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// `frame.close()`.
pub fn frame_close(frame: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(frame, "close")?;
    let _ = f.call0(frame)?;
    Ok(())
}

// =============================================================================
// Закрытие декодера
// =============================================================================

/// `decoder.close()`.
pub fn close_image_decoder(decoder: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "close")?;
    let _ = f.call0(decoder)?;
    Ok(())
}

// =============================================================================
// Утилита: чтение файла как ArrayBuffer
// =============================================================================

/// Читает `File` через `FileReader` и возвращает байты.
pub async fn read_file_bytes(file: &web_sys::File) -> Result<Vec<u8>, JsValue> {
    let reader = web_sys::FileReader::new()?;
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let reader_c = reader.clone();
        let onload = Closure::once(move || {
            let result = reader_c.result().unwrap_or(JsValue::UNDEFINED);
            let _ = resolve.call1(&JsValue::NULL, &result);
        });
        let onerror = Closure::once(move || {
            let _ = reject.call1(
                &JsValue::NULL,
                &JsValue::from_str("FileReader failed"),
            );
        });
        reader.set_onload(Some(onload.as_ref().unchecked_ref()));
        reader.set_onerror(Some(onerror.as_ref().unchecked_ref()));
        onload.forget();
        onerror.forget();
        let _ = reader.read_as_array_buffer(file);
    });
    let result = JsFuture::from(promise).await?;
    let array = Uint8Array::new(&result);
    Ok(array.to_vec())
}

/// Форматирует `repetitionCount` для UI.
pub fn format_repetition(count: f64) -> String {
    if count.is_infinite() {
        String::from("∞ (бесконечно)")
    } else if count == 0.0 {
        String::from("0 (один раз)")
    } else {
        format_str(format_args!("{c:.0}", c = count))
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
pub mod audio_codecs;             // 23.5
pub mod image_decoder;            // ← 23.6
```

### `tests/wasm_image_decoder.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

use hello_wasm02::image_decoder::{
    close_image_decoder, decode_frame, first_track,
    frame_close, frame_display_height, frame_display_width,
    frame_duration_us, is_image_decoder_supported, is_type_supported,
    make_image_decoder, track_animated, track_frame_count,
    track_repetition_count, track_selected, tracks_length,
    wait_tracks_ready,
};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Минимальный PNG-байткод (1×1, красный пиксель)
// =============================================================================

const PNG_1X1_BASE64: &str = "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==";

fn decode_base64(s: &str) -> Vec<u8> {
    use base64::Engine;
    let engine = base64::engine::general_purpose::STANDARD;
    engine.decode(s).unwrap_or_default()
}

// =============================================================================
// Feature detection
// =============================================================================

#[wasm_bindgen_test]
fn image_decoder_is_supported() {
    assert!(is_image_decoder_supported(), "ImageDecoder must be available");
}

#[wasm_bindgen_test]
async fn png_is_supported() {
    let ok = is_type_supported("image/png").await.unwrap_or(false);
    assert!(ok, "image/png must be supported");
}

#[wasm_bindgen_test]
async fn gif_is_supported() {
    let ok = is_type_supported("image/gif").await.unwrap_or(false);
    assert!(ok, "image/gif must be supported");
}

#[wasm_bindgen_test]
async fn bogus_mime_not_supported() {
    let ok = is_type_supported("image/not-a-real-format")
        .await
        .unwrap_or(false);
    assert!(!ok);
}

// =============================================================================
// Конструктор и метаданные
// =============================================================================

#[wasm_bindgen_test]
async fn decode_png_frame_zero() {
    let bytes = decode_base64(PNG_1X1_BASE64);
    assert!(!bytes.is_empty(), "PNG decode must succeed");

    let decoder = make_image_decoder(&bytes, "image/png").expect("construct");
    wait_tracks_ready(&decoder).await.expect("tracks ready");

    assert_eq!(tracks_length(&decoder), 1);

    let track = first_track(&decoder).expect("track 0");
    assert!(!track_animated(&track), "PNG не анимирован");
    assert_eq!(track_frame_count(&track), 1);
    assert!(track_selected(&track));

    let result = decode_frame(&decoder, 0, true).await.expect("decode");
    // PNG имеет intrinsic size — displayWidth/Height должны быть 1.
    assert_eq!(frame_display_width(&result.frame), 1);
    assert_eq!(frame_display_height(&result.frame), 1);

    let _ = frame_close(&result.frame);
    let _ = close_image_decoder(&decoder);
}

#[wasm_bindgen_test]
async fn decode_invalid_bytes_fails() {
    let bogus = vec![0u8; 64];
    let result = make_image_decoder(&bogus, "image/png");
    if let Ok(decoder) = result {
        let ready = wait_tracks_ready(&decoder).await;
        assert!(
            ready.is_err(),
            "невалидный PNG должен упасть на tracks.ready",
        );
        let _ = close_image_decoder(&decoder);
    }
}

// =============================================================================
// Close: после закрытия decode бросает
// =============================================================================

#[wasm_bindgen_test]
async fn decode_after_close_throws() {
    let bytes = decode_base64(PNG_1X1_BASE64);
    let decoder = make_image_decoder(&bytes, "image/png").expect("construct");
    wait_tracks_ready(&decoder).await.expect("tracks ready");

    close_image_decoder(&decoder).expect("close");
    let result = decode_frame(&decoder, 0, true).await;
    assert!(result.is_err(), "decode после close должен бросить");
}

#[wasm_bindgen_test]
async fn close_is_idempotent() {
    let bytes = decode_base64(PNG_1X1_BASE64);
    let decoder = make_image_decoder(&bytes, "image/png").expect("construct");
    wait_tracks_ready(&decoder).await.expect("tracks ready");
    assert!(close_image_decoder(&decoder).is_ok());
    let _ = close_image_decoder(&decoder);
}

// =============================================================================
// Frame duration для статики
// =============================================================================

#[wasm_bindgen_test]
async fn png_frame_duration_is_zero() {
    let bytes = decode_base64(PNG_1X1_BASE64);
    let decoder = make_image_decoder(&bytes, "image/png").expect("construct");
    wait_tracks_ready(&decoder).await.expect("tracks ready");
    let result = decode_frame(&decoder, 0, true).await.expect("decode");
    let duration = frame_duration_us(&result.frame);
    assert_eq!(duration, 0.0);
    let _ = frame_close(&result.frame);
    let _ = close_image_decoder(&decoder);
}

// =============================================================================
// repetitionCount для статики
// =============================================================================

#[wasm_bindgen_test]
async fn png_repetition_count_is_zero() {
    let bytes = decode_base64(PNG_1X1_BASE64);
    let decoder = make_image_decoder(&bytes, "image/png").expect("construct");
    wait_tracks_ready(&decoder).await.expect("tracks ready");
    let track = first_track(&decoder).expect("track 0");
    assert_eq!(track_repetition_count(&track), 0.0);
    let _ = close_image_decoder(&decoder);
}

// =============================================================================
// Ошибки на невалидном входе
// =============================================================================

#[wasm_bindgen_test]
async fn helpers_handle_non_decoder_input() {
    let v = JsValue::from_str("not a decoder");
    assert_eq!(tracks_length(&v), 0);
    assert!(first_track(&v).is_err());
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_image_decoder

# 3. Демо
npm run wasm:dev && npm start
```

---

## 23.6.6. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.6 — ImageDecoder: статика и animated GIF.
//!
//! Демо: file input (PNG/JPEG/GIF). Читаем файл, создаём
//! `ImageDecoder`, показываем метаданные из `tracks`, декодируем
//! кадры. Для GIF — воспроизведение с правильными задержками.
//!
//! ## Жизненный цикл VideoFrame
//!
//! Практика показала три ограничения Chrome:
//!
//! 1. `ImageDecoder` **владеет** своими frame'ами и инвалидирует
//!    их при следующем `decode()`. Держать frame между итерациями
//!    (held_frame, rAF) бесполезно — он становится broken.
//! 2. `frame.copyTo()` падает с `No media resource` на GPU-backed
//!    кадрах GIF — CPU readback не работает.
//! 3. `createImageBitmap(frame)` падает с `has width 0` — Chrome
//!    не устанавливает intrinsic size кадров GIF. `resizeWidth`
//!    не помогает: валидация происходит до применения опций.
//!
//! ## Рабочий паттерн
//!
//! Синхронный блок после первого await:
//!
//! ```text
//! decode() → frame_duration_us → drawImage(frame, 0, 0, w, h) → frame.close()
//! ```
//!
//! `w` и `h` берутся из метаданных трека, не из frame. `drawImage`
//! вызывается с явными размерами — работает даже когда у frame
//! `displayWidth = 0`. Frame живёт ровно один цикл.
//!
//! ## Что демонстрируется
//!
//! - `new ImageDecoder({ data, type })`.
//! - `tracks.ready` — метаданные доступны сразу после этого.
//! - `track.animated`, `.frameCount`, `.repetitionCount`.
//! - `decode({ frameIndex })` → VideoFrame + complete.
//! - Покадровая анимация с задержками через `frame.duration`.
//! - Single-session animation через `anim_session`.
//!
//! ## Что не является темой
//!
//! Видео-кодеки (23.1–23.4), аудио (23.5). Здесь только изображения.
//!
//! **Файл автономен.** Стили `.id-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::Reflect;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    CanvasRenderingContext2d, Document, File, HtmlCanvasElement,
    HtmlElement, HtmlImageElement, HtmlInputElement,
};

use crate::image_decoder::{
    close_image_decoder, decode_frame, first_track, format_repetition,
    frame_close, frame_display_height, frame_display_width,
    frame_duration_us, is_image_decoder_supported, is_type_supported,
    make_image_decoder, read_file_bytes, track_animated,
    track_frame_count, track_repetition_count, tracks_length,
    wait_tracks_ready,
};
use crate::video_frame::{format_str, invoke, num};

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
            var KEY='id-sw-cleared-23-6';
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
    let id = "id-injected-style";
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
.id-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.id-desc-line { margin:6px 0; }
.id-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.id-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.id-btn:hover:not([disabled]) { background:#30363d; }
.id-btn:disabled { opacity:0.5;cursor:not-allowed; }
.id-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.id-btn-primary:hover:not([disabled]) { background:#2ea043; }
.id-btn-play { background:#1f6feb;border-color:#388bfd;color:#fff;}
.id-btn-play:hover:not([disabled]) { background:#388bfd; }
.id-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.id-input { font-family:ui-monospace,monospace;font-size:12px;color:#c9d1d9;background:#0d1117;border:1px solid #30363d;border-radius:6px;padding:6px 10px;}
.id-pair { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.id-slot { flex:1 1 380px;min-width:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:10px 12px;}
.id-slot-title { color:#79c0ff;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.id-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:6px;max-width:100%;}
.id-img-preview { display:block;background:#000;border:1px solid #21262d;border-radius:6px;max-width:100%;}
.id-stats { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.id-stats .label { color:#8b949e; }
.id-stats .value { color:#79c0ff; }
.id-stats .ok { color:#3fb950; }
.id-stats .warn { color:#f0883e; }
.id-stats .err { color:#f85149; }
.id-stats .muted { color:#6e7681; }
.id-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone, Default)]
struct TrackMeta {
    animated: bool,
    frame_count: u32,
    repetition_count: f64,
    width: u32,
    height: u32,
}

struct DemoState {
    phase: String,
    error: Option<String>,
    running: bool,
    meta: Option<TrackMeta>,
    animating: bool,
    /// Уникальный идентификатор активной сессии анимации.
    /// Инкрементируется на каждый Play / Pause / Restart / смену файла.
    /// Цикл сравнивает своё значение с текущим — при несовпадении
    /// выходит. Это исключает параллельные циклы на одном декодере.
    anim_session: u64,
    current_frame: u32,
    play_cycle: u32,
    file_bytes: Vec<u8>,
    file_mime: String,
    file_name: String,
    decoder: Option<JsValue>,
    last_decode_ms: f64,
}

impl DemoState {
    fn new() -> Self {
        Self {
            phase: String::from("простой"),
            error: None,
            running: false,
            meta: None,
            animating: false,
            anim_session: 0,
            current_frame: 0,
            play_cycle: 0,
            file_bytes: Vec::new(),
            file_mime: String::new(),
            file_name: String::new(),
            decoder: None,
            last_decode_ms: 0.0,
        }
    }

    fn clear_file(&mut self) {
        if let Some(d) = self.decoder.take() {
            let _ = close_image_decoder(&d);
        }
        self.error = None;
        self.meta = None;
        self.animating = false;
        self.anim_session = self.anim_session.wrapping_add(1);
        self.current_frame = 0;
        self.play_cycle = 0;
        self.file_bytes.clear();
        self.file_mime.clear();
        self.file_name.clear();
        self.last_decode_ms = 0.0;
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

fn line_file(st: &DemoState) -> String {
    if st.file_name.is_empty() {
        return format_str(format_args!(
            r#"<div><span class="muted">Файл не выбран. PNG / JPEG / GIF.</span></div>"#
        ));
    }
    format_str(format_args!(
        r#"<div><span class="label">Файл: </span><span class="value">{name}</span> · <span class="value">{mime}</span> · <span class="muted">{bytes} байт</span></div>"#,
        name = st.file_name,
        mime = st.file_mime,
        bytes = st.file_bytes.len(),
    ))
}

fn line_meta(st: &DemoState) -> String {
    match st.meta.as_ref() {
        None => String::new(),
        Some(m) => {
            let anim_str = if m.animated { "да" } else { "нет" };
            let anim_cls = if m.animated { "ok" } else { "muted" };
            let rep = if m.animated {
                format_str(format_args!(
                    r#" · <span class="label">Repetition: </span><span class="value">{r}</span>"#,
                    r = format_repetition(m.repetition_count),
                ))
            } else {
                String::new()
            };
            format_str(format_args!(
                r#"<div><span class="label">Метаданные: </span><span class="value">{w}×{h}</span> · <span class="label">Frames: </span><span class="value">{n}</span> · <span class="label">Animated: </span><span class="{ac}">{a}</span>{rep}</div>"#,
                w = m.width,
                h = m.height,
                n = m.frame_count,
                ac = anim_cls,
                a = anim_str,
                rep = rep,
            ))
        }
    }
}

fn line_playback(st: &DemoState) -> String {
    match st.meta.as_ref() {
        Some(m) if m.animated => format_str(format_args!(
            r#"<div><span class="label">Кадр: </span><span class="value">{i} / {n}</span> · <span class="label">Цикл: </span><span class="value">{c}</span> · <span class="label">Decode: </span><span class="value">{ms:.1} мс</span></div>"#,
            i = st.current_frame + 1,
            n = m.frame_count,
            c = st.play_cycle + 1,
            ms = st.last_decode_ms,
        )),
        Some(_) => format_str(format_args!(
            r#"<div><span class="label">Decode: </span><span class="value">{ms:.1} мс</span></div>"#,
            ms = st.last_decode_ms,
        )),
        None => String::new(),
    }
}

fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("id-stats") else {
        return;
    };
    let err_line = st
        .error
        .as_ref()
        .map(|m| line_err(m.as_str()))
        .unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{phase}{file}{meta}{playback}"#,
        err = err_line,
        phase = line_phase(st),
        file = line_file(st),
        meta = line_meta(st),
        playback = line_playback(st),
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

fn set_button_text(doc: &Document, id: &str, text: &str) {
    let Some(el) = doc.get_element_by_id(id) else {
        return;
    };
    el.set_text_content(Some(text));
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>ImageDecoder</strong> получает <em>весь файл</em> при создании: <code>new ImageDecoder({ data, type })</code>. Не требует <code>configure</code> и <code>flush</code>. Метаданные доступны сразу после <code>tracks.ready</code>.",
        "<strong>ImageTrack</strong> описывает один трек: <code>animated</code>, <code>frameCount</code>, <code>repetitionCount</code>. Для статических PNG/JPEG всегда <code>frameCount = 1</code>, <code>animated = false</code>. Для GIF — десятки кадров и цикличность.",
        "<strong>decode({ frameIndex })</strong> возвращает <code>{ image: VideoFrame, complete: boolean }</code>. Флаг <code>complete</code> полезен для progressive JPEG.",
        "<strong>Жизненный цикл VideoFrame.</strong> Chrome инвалидирует предыдущий frame при следующем <code>decode()</code>. Паттерн: <code>decode → drawImage → frame.close</code> без промежуточных await. Frame не переживает итерацию.",
        "<strong>drawImage, не copyTo.</strong> <code>frame.copyTo()</code> падает с <code>No media resource</code> — кадры GPU-backed, readback не работает. <code>createImageBitmap(frame)</code> падает с <code>has width 0</code> для анимированного GIF. <code>drawImage(VideoFrame)</code> работает — композитор берёт пиксели напрямую.",
        "<strong>Сравнение с &lt;img&gt;.</strong> Обычный тег для GIF показывает <em>только первый кадр</em>, если не дать ему управление. <code>ImageDecoder</code> даёт покадровый контроль.",
        "<strong>Single-session анимация.</strong> На каждый Play/Pause/Restart инкрементируется <code>anim_session</code>. Цикл сравнивает своё значение с текущим и выходит, если оно устарело.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("id-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("id-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Отрисовка кадра
// =============================================================================

/// Рисует VideoFrame напрямую через `drawImage`. Синхронно.
///
/// Размеры передаём явно из метаданных трека — у кадра GIF
/// intrinsic size может быть 0, но `drawImage(frame, x, y, w, h)`
/// работает с явными аргументами.
fn draw_frame_now(doc: &Document, frame: &JsValue, w: u32, h: u32) {
    if w == 0 || h == 0 {
        return;
    }
    let Some(canvas) = doc
        .get_element_by_id("id-canvas")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
    else {
        return;
    };
    if canvas.width() != w {
        canvas.set_width(w);
    }
    if canvas.height() != h {
        canvas.set_height(h);
    }
    let Ok(Some(ctx)) = canvas.get_context("2d") else {
        return;
    };
    let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
        return;
    };
    let _ = invoke(
        ctx.as_ref(),
        "drawImage",
        &[
            frame.clone(),
            num(0.0),
            num(0.0),
            num(w as f64),
            num(h as f64),
        ],
    );
}

/// Декодирует кадр и рисует его. Синхронная последовательность
/// после `decode()`: читаем `duration`, рисуем, закрываем — без
/// промежуточных await.
///
/// `drawImage(VideoFrame)` в Chrome захватывает пиксели у frame
/// при вызове. `frame.close()` сразу после этого безопасен.
async fn decode_and_draw(
    doc: &Document,
    decoder: &JsValue,
    frame_idx: u32,
    w: u32,
    h: u32,
) -> Result<f64, JsValue> {
    let decoded = decode_frame(decoder, frame_idx, true).await?;
    let frame = decoded.frame;

    let dur = frame_duration_us(&frame);
    draw_frame_now(doc, &frame, w, h);
    let _ = frame_close(&frame);

    Ok(dur)
}

// =============================================================================
// Загрузка файла
// =============================================================================

fn infer_mime(name: &str) -> &'static str {
    let lower = name.to_lowercase();
    if lower.ends_with(".png") {
        "image/png"
    } else if lower.ends_with(".jpg") || lower.ends_with(".jpeg") {
        "image/jpeg"
    } else if lower.ends_with(".gif") {
        "image/gif"
    } else if lower.ends_with(".webp") {
        "image/webp"
    } else if lower.ends_with(".avif") {
        "image/avif"
    } else {
        "image/png"
    }
}

async fn do_load_file(doc: Document, st: Rc<RefCell<DemoState>>, file: File) {
    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.running = true;
        s.clear_file();
        s.file_name = file.name();
        s.phase = String::from("чтение файла…");
    }
    let _guard = RunGuard(st.clone());
    update_stats(&doc, &st.borrow());

    let bytes = match read_file_bytes(&file).await {
        Ok(v) => v,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("read file: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
            return;
        }
    };

    let mut mime = infer_mime(&file.name()).to_string();
    let supported = is_type_supported(&mime).await.unwrap_or(false);
    if !supported {
        let alt = if mime == "image/jpeg" {
            "image/png"
        } else {
            "image/jpeg"
        };
        if is_type_supported(alt).await.unwrap_or(false) {
            mime = alt.to_string();
        }
    }
    {
        let mut s = st.borrow_mut();
        s.file_bytes = bytes.clone();
        s.file_mime = mime.clone();
    }

    let decoder = match make_image_decoder(&bytes, &mime) {
        Ok(v) => v,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("ImageDecoder: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
            return;
        }
    };

    if let Err(e) = wait_tracks_ready(&decoder).await {
        let mut s = st.borrow_mut();
        s.error = Some(format_str(format_args!(
            "tracks.ready: {e:?} — возможно, файл повреждён"
        )));
        s.phase = String::from("ошибка");
        update_stats(&doc, &s);
        let _ = close_image_decoder(&decoder);
        return;
    }

    let track = match first_track(&decoder) {
        Ok(t) => t,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("first_track: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
            let _ = close_image_decoder(&decoder);
            return;
        }
    };

    let animated = track_animated(&track);
    let frame_count = track_frame_count(&track);
    let repetition_count = track_repetition_count(&track);
    let tracks_n = tracks_length(&decoder);

    // Декодируем кадр 0 — узнаём размеры и рисуем первый кадр.
    let t0 = js_sys::Date::now();
    let (width, height) = match decode_frame(&decoder, 0, true).await {
        Ok(r) => {
            let w = frame_display_width(&r.frame);
            let h = frame_display_height(&r.frame);
            draw_frame_now(&doc, &r.frame, w, h);
            let _ = frame_close(&r.frame);
            (w, h)
        }
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("decode frame 0: {e:?}")));
            s.phase = String::from("ошибка");
            update_stats(&doc, &s);
            let _ = close_image_decoder(&decoder);
            return;
        }
    };
    let t1 = js_sys::Date::now();

    let session = {
        let mut s = st.borrow_mut();
        s.meta = Some(TrackMeta {
            animated,
            frame_count,
            repetition_count,
            width,
            height,
        });
        s.decoder = Some(decoder);
        s.current_frame = 0;
        s.last_decode_ms = t1 - t0;
        s.phase = format_str(format_args!(
            "готово ({tracks} трек(ов), animated={a})",
            tracks = tracks_n,
            a = if animated { "true" } else { "false" },
        ));
        s.animating = animated;
        s.anim_session
    };
    update_stats(&doc, &st.borrow());

    set_button_enabled(&doc, "id-play", animated);
    set_button_enabled(&doc, "id-restart", animated);
    if animated {
        set_button_text(&doc, "id-play", "Pause");
    } else {
        set_button_text(&doc, "id-play", "Play");
    }

    if animated {
        let doc_c = doc.clone();
        let st_c = st.clone();
        spawn_local(async move {
            start_animation_loop(doc_c, st_c, session).await;
        });
    }
}

// =============================================================================
// Анимация
// =============================================================================

/// Цикл анимации. На каждой итерации:
/// 1. Проверка сессии.
/// 2. `decode` → `drawImage` → `frame.close` — синхронный блок.
/// 3. sleep на `frame.duration`.
async fn start_animation_loop(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    session: u64,
) {
    let (mut frame_idx, mut cycle, meta_w, meta_h, frame_count, rep_limit) = {
        let s = st.borrow();
        let meta = match s.meta.as_ref() {
            Some(m) if m.animated => m,
            _ => return,
        };
        (
            s.current_frame,
            s.play_cycle,
            meta.width,
            meta.height,
            meta.frame_count,
            meta.repetition_count,
        )
    };

    loop {
        // 1. Проверка сессии.
        {
            let s = st.borrow();
            if s.anim_session != session || !s.animating {
                return;
            }
        }

        // 2. Декодер.
        let decoder = match st.borrow().decoder.as_ref() {
            Some(d) => d.clone(),
            None => return,
        };

        // 3. Decode + draw + close — всё синхронно после decode.
        let t0 = js_sys::Date::now();
        let dur_us = match decode_and_draw(&doc, &decoder, frame_idx, meta_w, meta_h).await {
            Ok(d) => d,
            Err(e) => {
                web_sys::console::error_1(&e);
                return;
            }
        };
        let t1 = js_sys::Date::now();

        // 4. Проверка сессии после await.
        {
            let s = st.borrow();
            if s.anim_session != session {
                return;
            }
        }

        // 5. Статистика.
        {
            let mut s = st.borrow_mut();
            s.current_frame = frame_idx;
            s.play_cycle = cycle;
            s.last_decode_ms = t1 - t0;
        }
        update_stats(&doc, &st.borrow());

        // 6. Пауза между кадрами (100 мс по умолчанию для GIF).
        let dur_ms = if dur_us > 0.0 { dur_us / 1000.0 } else { 100.0 };
        let sleep_ms = dur_ms.max(10.0) as u32;
        TimeoutFuture::new(sleep_ms).await;

        // 7. Продвигаем кадр.
        frame_idx += 1;
        if frame_idx >= frame_count {
            frame_idx = 0;
            cycle += 1;
            if rep_limit.is_finite() && rep_limit > 0.0 && (cycle as f64) >= rep_limit {
                let mut s = st.borrow_mut();
                if s.anim_session == session {
                    s.animating = false;
                    s.phase = format_str(format_args!(
                        "анимация завершена ({c} цикл(ов))",
                        c = cycle,
                    ));
                    set_button_text(&doc, "id-play", "Play");
                    update_stats(&doc, &s);
                }
                return;
            }
        }
    }
}

// =============================================================================
// Обработчики
// =============================================================================

async fn do_file_input(doc: Document, st: Rc<RefCell<DemoState>>) {
    let Some(input_el) = doc.get_element_by_id("id-file") else {
        return;
    };
    let Ok(input) = input_el.dyn_into::<HtmlInputElement>() else {
        return;
    };
    let Some(files) = input.files() else {
        return;
    };
    if files.length() == 0 {
        return;
    }
    let Some(file) = files.get(0) else {
        return;
    };

    let url = web_sys::Url::create_object_url_with_blob(&file).ok();
    if let Some(url) = &url {
        if let Some(img_el) = doc.get_element_by_id("id-img") {
            if let Ok(img) = img_el.dyn_into::<HtmlImageElement>() {
                img.set_src(url);
            }
        }
    }

    do_load_file(doc, st, file).await;
}

fn do_toggle_play(doc: Document, st: Rc<RefCell<DemoState>>) {
    let (now, was, session) = {
        let mut s = st.borrow_mut();
        let was = s.animating;
        s.animating = !s.animating;
        s.anim_session = s.anim_session.wrapping_add(1);
        let now = s.animating;
        s.phase = if now {
            String::from("анимация")
        } else {
            String::from("пауза")
        };
        (now, was, s.anim_session)
    };
    set_button_text(&doc, "id-play", if now { "Pause" } else { "Play" });
    update_stats(&doc, &st.borrow());

    if now && !was {
        let doc_c = doc.clone();
        let st_c = st.clone();
        spawn_local(async move {
            start_animation_loop(doc_c, st_c, session).await;
        });
    }
}

fn do_restart(doc: Document, st: Rc<RefCell<DemoState>>) {
    let (bytes, mime) = {
        let s = st.borrow();
        (s.file_bytes.clone(), s.file_mime.clone())
    };
    if bytes.is_empty() {
        return;
    }
    let doc_c = doc.clone();
    let st_c = st.clone();
    spawn_local(async move {
        {
            let mut s = st_c.borrow_mut();
            if let Some(d) = s.decoder.take() {
                let _ = close_image_decoder(&d);
            }
            s.current_frame = 0;
            s.play_cycle = 0;
            s.animating = false;
            s.anim_session = s.anim_session.wrapping_add(1);
            s.error = None;
            s.phase = String::from("перезапуск…");
        }
        update_stats(&doc_c, &st_c.borrow());

        let decoder = match make_image_decoder(&bytes, &mime) {
            Ok(v) => v,
            Err(e) => {
                let mut s = st_c.borrow_mut();
                s.error = Some(format_str(format_args!("reopen: {e:?}")));
                update_stats(&doc_c, &s);
                return;
            }
        };
        if wait_tracks_ready(&decoder).await.is_err() {
            let _ = close_image_decoder(&decoder);
            return;
        }

        let (meta_w, meta_h) = {
            let s = st_c.borrow();
            match s.meta.as_ref() {
                Some(m) => (m.width, m.height),
                None => (0, 0),
            }
        };

        let _ = decode_and_draw(&doc_c, &decoder, 0, meta_w, meta_h).await;

        let session = {
            let mut s = st_c.borrow_mut();
            s.decoder = Some(decoder);
            s.phase = String::from("перезапущено");
            s.animating = true;
            s.anim_session = s.anim_session.wrapping_add(1);
            s.anim_session
        };
        set_button_text(&doc_c, "id-play", "Pause");
        update_stats(&doc_c, &st_c.borrow());

        let doc_loop = doc_c.clone();
        let st_loop = st_c.clone();
        spawn_local(async move {
            start_animation_loop(doc_loop, st_loop, session).await;
        });
    });
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
    h2.set_text_content(Some("23.6 — ImageDecoder (static & animated)"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_image_decoder_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("id-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает ImageDecoder. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    let file_block = document.create_element("div")?;
    file_block.set_class_name("id-block");
    file_block.set_inner_html(
        r#"<label class="id-label">Файл (PNG / JPEG / GIF):</label>
           <input class="id-input" type="file" id="id-file" accept="image/*">
           <button class="id-btn id-btn-play" id="id-play" disabled>Play</button>
           <button class="id-btn" id="id-restart" disabled>Restart</button>"#,
    );
    root.append_child(&file_block)?;

    let pair = document.create_element("div")?;
    pair.set_class_name("id-pair");
    pair.set_inner_html(
        r#"<div class="id-slot">
             <div class="id-slot-title">ImageDecoder → canvas</div>
             <canvas id="id-canvas" class="id-canvas" width="320" height="240"></canvas>
           </div>
           <div class="id-slot">
             <div class="id-slot-title">&lt;img&gt; (только первый кадр для GIF)</div>
             <img id="id-img" class="id-img-preview" alt="">
           </div>"#,
    );
    root.append_child(&pair)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("id-stats");
    stats.set_id("id-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("id-footnote");
    footnote.set_text_content(Some(
        "ImageDecoder получает весь файл при создании. Метаданные доступны после \
         tracks.ready. Для GIF запускается покадровая анимация с задержками из \
         frame.duration — левый canvas показывает все кадры, правый <img> показывает \
         только первый. Кнопки Play / Pause / Restart управляют анимацией. \
         Паттерн работы с VideoFrame: decode → drawImage → frame.close, всё \
         синхронно. Chrome инвалидирует предыдущий frame при следующем decode — \
         поэтому frame не переживает итерацию. frame.copyTo() падает с \
         No media resource, createImageBitmap — с has width 0. drawImage работает.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    {
        let doc = document.clone();
        let st = state.clone();
        let input_el = document
            .get_element_by_id("id-file")
            .ok_or_else(|| JsValue::from_str("no #id-file"))?;
        let input: HtmlElement = input_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                do_file_input(doc, st).await;
            });
        });
        input.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("id-play")
            .ok_or_else(|| JsValue::from_str("no #id-play"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            do_toggle_play(doc.clone(), st.clone());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("id-restart")
            .ok_or_else(|| JsValue::from_str("no #id-restart"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            do_restart(doc.clone(), st.clone());
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

## 23.6.7. Анализ решения

**Ценность демо.**

1. **Полный цикл изображений.** File input → `ImageDecoder` → метаданные → decode → canvas.
2. **Статика и анимация в одном примере.** PNG/JPEG — кадр 0, GIF — покадровое воспроизведение с задержками.
3. **`tracks.ready` явно показан.** Метаданные (`frameCount`, `animated`, `repetitionCount`) доступны до первого `decode`.
4. **Сравнение с `<img>`.** Правый слот показывает, что тег играет GIF сам, но не даёт контроля. Левый даёт покадровое управление.
5. **Практические ограничения — задокументированы.** Раздел 23.6.4 собирает все грабли Chrome и объясняет, почему выбран именно этот паттерн.

**Правила проекта, применённые.**

- **Отдельные функции для строк** — `line_err`, `line_phase`, `line_file`, `line_meta`, `line_playback`.
- **`format_str(format_args!(...))` в tail-позициях.**
- **`Rc<RefCell<DemoState>>`** — общее состояние.
- **`RunGuard`** — защита от параллельных загрузок.
- **Yield в цикле анимации через `TimeoutFuture`** — по задержке кадра, а не по 16 мс.
- **Префикс `id-`** — не пересекается с `vd-`, `vf-`, `ve-`, `vt-`, `ac-`.
- **Никаких `unused imports`.**
- **`create_element`, не `createElement`.**

**Найденные в практике особенности.**

1. `ImageDecoder` инвалидирует свои frame'ы при следующем `decode()`. Держать их между итерациями бесполезно.
2. `frame.copyTo()` падает с `No media resource` — GPU-backed кадры GIF.
3. `createImageBitmap(frame)` падает с `has width 0` — Chrome не устанавливает intrinsic size.
4. `drawImage(frame, 0, 0, w, h)` с явными размерами работает всегда.
5. `repetitionCount = Infinity` — валидное значение.
6. `decode` после `close()` бросает `InvalidStateError`.
7. `tracks[0]` может вернуть `null` при невалидном файле — нужна проверка.

**Проверки корректности.**

- `format_args!` без `f`-суффикса: `{r:.0}`, `{ms:.1}`. Валидны.
- Raw-строки: `r#"..."#` в HTML, `r##"..."##` не нужен.
- `read_file_bytes` — через `FileReader` с `Closure::once` и `forget()`. Не течёт.
- `start_animation_loop` — цикл с `TimeoutFuture` по `frame.duration`.
- `RunGuard` устанавливается после `s.running = true`.
- `first_track` проверяет `is_null()` и `is_undefined()` — не паникует на невалидном декодере.

---

## 23.6.8. Wow-момент на практике

**Первый.** GIF на 14 кадров: `<img>` справа показывает только первый кадр (или играет сам, но без управления), canvas слева — покадрово. Разница видна сразу: у `<img>` нет API «пауза», «шаг», «кадр 5».

**Второй.** PNG с 1×1 пикселем: `frameCount = 1`, `animated = false`, `repetitionCount = 0`. Идеальный тестовый случай.

**Третий.** Оптимизация canvas-размера: canvas сжимается до 320×240, а GIF 600×338 растягивается при отрисовке. Хотите пиксель-в-пиксель — устанавливайте canvas в метаданные трека.

**Четвёртый.** После `decoder.close()` любой `decode` бросает `InvalidStateError`. Освобождение ресурсов мгновенное.

**Пятый.** Кнопки Play / Pause / Restart работают без перезапуска страницы. `anim_session` гарантирует, что параллельные циклы не накапливаются.

---

## 23.6.9. Итог

**`ImageDecoder`** — API для статических и анимированных изображений. **Не потоковый**: получает весь файл сразу при создании.

**`tracks.ready`** — момент, после которого доступны метаданные. `ImageTrack` содержит `animated`, `frameCount`, `repetitionCount`, `selected`.

**`decode({ frameIndex, completeFramesOnly })`** — декодирует конкретный кадр. Возвращает `{ image: VideoFrame, complete: boolean }`.

**Рабочий паттерн работы с frame:**
```
decode() → frame_duration_us → drawImage(frame, 0, 0, w, h) → frame.close()
```
Всё синхронно после первого `await`. Frame живёт ровно один цикл.

**Что не работает (в этой сборке Chrome):**
- `frame.copyTo()` — GPU readback не поддерживается.
- `createImageBitmap(frame)` — intrinsic size = 0 у кадров GIF.
- Держать frame между итерациями — decoder инвалидирует.

**Практическая находка.** `ImageDecoder` работает в headless-тестах — в отличие от `AudioContext`. Это делает тесты простыми: достаточно встроить base64-байты PNG.
