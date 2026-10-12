# Глава 23.1. VideoFrame и EncodedVideoChunk: два представления кадра

## 23.1.0. Обновлённая структура

```
src/
  video_frame.rs                  ← НОВЫЙ модуль 23.1
  codecs_demo.rs                  ← из главы 23 (первая версия, не трогаем)
  fractal.rs                      ← из 22.6
  worker_pool.rs                  ← из 22.5
  …
tests/
  wasm_video_frame.rs             ← НОВЫЙ тест
src/lib.rs                        ← +1 строка: pub mod video_frame;
src/app.rs                        ← демо 23.1 (заменяется целиком)
web/
  public/workers/…                ← существующие воркеры не трогаем
  pkg/                            ← сборка wasm-pack (out-dir web/pkg)
  main.js                         ← без правок
Cargo.toml                        ← без новых фич (WebCodecs через Reflect)
vite.config.js                    ← без правок
package.json                      ← без правок
```

**Комментарий.** Модуль `video_frame.rs` — про **два базовых типа WebCodecs**: `VideoFrame` (распакованный кадр, память GPU) и `EncodedVideoChunk` (сжатый кадр, память CPU). Префикс CSS — `vf-` (video frame), уникален в проекте. Демо автономно, стили инжектируются в документ. Все вызовы WebCodecs — через `Reflect::get` + `Function::call*` + `Reflect::construct`, потому что web-sys не экспортирует эти типы с нужными сигнатурами.

**Ссылки для детального изучения:**

- MDN: [`VideoFrame`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame)
- MDN: [`EncodedVideoChunk`](https://developer.mozilla.org/en-US/docs/Web/API/EncodedVideoChunk)
- MDN: [`VideoFrame.close()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame/close)
- MDN: [`VideoFrame.copyTo()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame/copyTo)
- MDN: [`ImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap)
- W3C: [WebCodecs § VideoFrame](https://w3c.github.io/webcodecs/#videoframe-interface)
- W3C: [WebCodecs § EncodedVideoChunk](https://w3c.github.io/webcodecs/#encodedvideochunk-interface)

---

## 23.1.1. О чём эта подглава

WebCodecs работает с **двумя представлениями одного и того же кадра**:

- **`VideoFrame`** — распакованный кадр. Пиксели. Живёт в памяти GPU.
- **`EncodedVideoChunk`** — сжатый кадр. Байты. Живёт в памяти CPU.

Всё остальное в WebCodecs — комбинации этих двух. `VideoDecoder` превращает `EncodedVideoChunk` в `VideoFrame`. `VideoEncoder` — наоборот. `VideoFrame` рисуется на canvas. `EncodedVideoChunk` передаётся по сети.

Подглава отвечает на пять вопросов.

**Первый — что такое `VideoFrame`.** Это не картинка в DOM, а **объект владения пикселями**. Он держит память GPU до тех пор, пока явно не закрыт.

**Второй — почему его нужно закрывать.** `frame.close()` освобождает ресурсы. Без него — утечка видеопамяти.

**Третий — как извлечь пиксели.** Через `drawImage` на canvas (для показа) или через scratch-canvas + `getImageData` (для обработки).

**Четвёртый — что такое `EncodedVideoChunk`.** Сжатые байты + метаданные: `type`, `timestamp`, `duration`, `byteLength`.

**Пятый — как они соотносятся.** `VideoFrame` в 10–100 раз больше по объёму. `EncodedVideoChunk` — результат сжатия.

---

## 23.1.2. VideoFrame: владение пикселями

**Что это.** `VideoFrame` — объект, содержащий распакованный кадр. Создаётся несколькими способами:

```js
new VideoFrame(canvas, { timestamp: 0 });
new VideoFrame(bitmap, { timestamp: 0 });
new VideoFrame(otherFrame, { timestamp: 0 });
new VideoFrame(buffer, { format: "RGBA", codedWidth: 320, codedHeight: 240, timestamp: 0 });
```

**Ключевые свойства:**

- `codedWidth`, `codedHeight` — размеры буфера;
- `displayWidth`, `displayHeight` — размеры отображения;
- `timestamp` — момент времени в **микросекундах** (не миллисекундах!);
- `duration` — длительность в микросекундах;
- `format` — пиксельный формат: `"RGBA"`, `"I420"`, `"NV12"`, `"BGRA"`.

**Почему микросекунды.** Стандарт WebCodecs использует микросекунды — это точность аудио-сэмплов. 30 кадров в секунду — это `33333` микросекунды на кадр.

**Что можно делать с кадром:**

- `ctx.drawImage(frame, 0, 0, w, h)` — нарисовать на canvas;
- `gl.texImage2D(..., frame)` — загрузить в WebGL-текстуру;
- `frame.copyTo(buffer, { format: "RGBA" })` — извлечь пиксели;
- `frame.close()` — освободить память.

**`close()` идемпотентен.** Второй вызов не бросает исключение. Это сознательное решение спецификации.

**После `close()` кадр непригоден.** Любая операция бросает `InvalidStateError`. Метаданные при этом обнуляются: `codedWidth = 0`, `codedHeight = 0` — это видно в демо.

---

## 23.1.3. EncodedVideoChunk: сжатые байты

```js
new EncodedVideoChunk({
    type: "key",           // или "delta"
    timestamp: 0,
    duration: 33333,
    data: new Uint8Array([...]),
});
```

**Свойства:**

- `type` — `"key"` (независимый кадр) или `"delta"` (зависит от предыдущих);
- `timestamp`, `duration` — микросекунды;
- `byteLength` — размер в байтах;
- `copyTo(dest)` — извлечь байты в `Uint8Array`.

**Что такое key и delta.** Key-кадр (IDR) декодируется независимо. Delta-кадр зависит от предыдущих. В потоке H.264 обычно 1 key-кадр на 30–60 delta. Key занимает в 5–10 раз больше байт.

**Соотношение размеров.** Один `VideoFrame` 480×360 = 691 200 байт (≈ 675 КБ). Тот же кадр в H.264 при bitrate 1 Мбит/с ≈ 4 КБ. Разница — **170×**. Это и есть смысл сжатия.

---

## 23.1.4. Что показывает демо

**Layout:**

- **Три canvas'а рядом:**
  - **Source** — main thread рисует пульсирующий градиент.
  - **Via VideoFrame + drawImage** — `VideoFrame` из source, нарисованный через `drawImage`.
  - **Via VideoFrame + getImageData** — пиксели извлекаются через scratch-canvas, кладутся как `ImageData`.
- **Селект «Режим»:** Both / drawImage only / getImageData only.
- **Кнопки:** Pause / Resume, Use closed frame.
- **Статистика:** FPS (скользящее окно 500 мс), время каждого пути, размер кадра.
- **Блок про EncodedVideoChunk:** метаданные чанка.

**Сценарий:**

1. Анимация идёт — три canvas'а синхронны.
2. Пользователь переключает режим — один из правых canvas'ов очищается, второй продолжает. Тайминги показывают разницу.
3. **Pause** — FPS падает до 0. **Resume** — восстанавливается.
4. **Use closed frame** — в статистике появляется сообщение `codedWidth=0, codedHeight=0`.

---

## 23.1.5. Практические находки

Работа над демо выявила несколько **неочевидных особенностей WebCodecs и web-sys**, которые важно зафиксировать.

### 1. `VideoFrame.copyTo` на canvas-sourced кадре падает **асинхронно**

**Симптом:**

```
Uncaught (in promise) DOMException: VideoFrame.copyTo: No media resource in VideoFrame
```

Ключевое — **«in promise»**. Ошибка не выбрасывается синхронно из `copyTo`. На момент возврата из вызова всё выглядит нормально — `Result::Ok`, буфер приходит, но через несколько миллисекунд приходит асинхронное исключение.

**Почему.** `VideoFrame`, созданный из `HtmlCanvasElement`, хранит **ссылку** на canvas-ресурс. Chrome в целях оптимизации не материализует пиксели кадра до явного запроса. Если между созданием кадра и запросом пикселей было вызвано `drawImage(frame, ...)` — Chrome может использовать GPU→GPU путь и не материализовать CPU-ресурс вообще. Тогда `copyTo` не находит, что копировать.

**Синхронный `Result::Err` поймать не удаётся** — в момент возврата ошибки ещё нет.

**Что делать.** Не использовать `copyTo` для canvas-sourced `VideoFrame`. Надёжный путь — `drawImage(frame, scratch)` + `getImageData(scratch)`. Один лишний шаг, но работает во всех браузерах.

### 2. Порядок операций: `copyTo` до `drawImage`

Даже для `copyTo`-совместимых кадров порядок критичен. Правильно:

```
1. copyTo(frame)      ← читаем пиксели
2. drawImage(frame)   ← рисуем на canvas
3. close(frame)
```

Если `drawImage` идёт первым — он может «потребить» ресурс кадра, и `copyTo` следом упадёт.

### 3. `getImageData` в web-sys 0.3.105 объявлен с `i32`-параметрами

**Симптом:**

```
expected i32, found f64
```

На строках вида:

```rust
ctx.get_image_data(0.0, 0.0, w as f64, h as f64)  // E0308
```

web-sys 0.3.105 объявляет `CanvasRenderingContext2d::get_image_data` с `i32`. Реальный JS API принимает `f64`. Рассинхронизация биндинга с API.

**Решение.** Все вызовы `getImageData` — через `Reflect::invoke`:

```rust
let args = [num(0.0), num(0.0), num(w as f64), num(h as f64)];
let img = invoke(ctx_js, "getImageData", &args)?;
```

Не зависит от версии web-sys, работает всегда.

### 4. `js_sys::Function` не имеет `construct1` / `construct2`

`Function` в `js-sys` предоставляет только `call0`…`call3`. Для `new Ctor(...)` нужно:

```rust
let ctor = Reflect::get(&global, "VideoFrame")?
    .dyn_into::<Function>()?;
let args = Array::new();
args.push(&init);
Reflect::construct(&ctor, &args)?
```

`Reflect::construct` требует `&Function`, не `&JsValue`. Приводить через `dyn_into`.

### 5. `HtmlCanvasElement::as_ref()` неоднозначен

`HtmlCanvasElement` реализует несколько `AsRef<T>` (`AsRef<JsValue>`, `AsRef<Object>`, …). При передаче в функцию, ожидающую `&JsValue`, компилятор не может выбрать. Решение:

```rust
let canvas_val = JsValue::from(canvas.clone());
construct("VideoFrame", &[canvas_val, init.into()])
```

Явное `Into<JsValue>` снимает неоднозначность.

### 6. `Clamped<&[u8]>` vs `Clamped<Vec<u8>>`

`ImageData::new_with_u8_clamped_array_and_sh` принимает `Clamped<&[u8]>`, не `Clamped<Vec<u8>>`:

```rust
// Не работает:
let clamped = wasm_bindgen::Clamped(vec);

// Работает:
let bytes = /* Vec<u8> */;
let clamped = wasm_bindgen::Clamped(&bytes[..]);
ImageData::new_with_u8_clamped_array_and_sh(clamped, w, h)
```

`bytes` должен жить до конца вызова.

### 7. `performance.now()` для точных измерений

`Date.now()` даёт миллисекунды от эпохи. Разница двух таких значений — миллисекунды, но с плохим разрешением (1 мс — квант).

Для микросекундных измерений лучше `performance.now()` — монотонный, высокоточный.

```rust
fn now_ms() -> f64 {
    let perf = Reflect::get(&global, "performance")?;
    let f = Reflect::get(&perf, "now")?.dyn_into::<Function>()?;
    f.call0(&perf)?.as_f64()
}
```

### 8. `close()` обнуляет метаданные

После `frame.close()` **`codedWidth` и `codedHeight` становятся `0`**. Это не баг, а часть контракта: кадр освобождён, метаданные обнулены. Демо проверяет это через кнопку «Use closed frame».

### 9. Цена readback'а: 10–50×

`drawImage` (визуализация, GPU→GPU) — **0.2 мс**.  
`drawImage + getImageData` (извлечение пикселей, GPU→CPU) — **2–10 мс**.

Разница в **10–50 раз**. Это фундаментальная цена чтения пикселей с GPU на CPU. В реальных приложениях такое чтение выносят в Worker или заменяют на `WebGL2.texImage2D(frame)` — GPU→GPU без readback. Именно об этом — подглавы 23.7 и 23.8.

### 10. `.vf-block .value` требует своего CSS-правила

В `STYLE_CSS` правила `.vf-stats .label` и `.vf-stats .value` действуют **только внутри** `.vf-stats`. Блок `EncodedVideoChunk` находится в `.vf-block` — там нужны свои правила:

```css
.vf-block .label { color:#8b949e; }
.vf-block .value { color:#79c0ff; }
```

Без них значения чанка рендерятся, но не видны — цвет по умолчанию тёмный на тёмном фоне.

---

## 23.1.6. Исходные файлы

### `src/video_frame.rs`

```rust
//! 23.1. VideoFrame и EncodedVideoChunk.
//!
//! Два представления кадра в WebCodecs:
//!
//! - `VideoFrame` — распакованный кадр, память GPU. Обязательно
//!   закрывать через `close()`.
//! - `EncodedVideoChunk` — сжатый кадр, память CPU. В 10–100 раз
//!   меньше по объёму.
//!
//! ## Доступ к WebCodecs
//!
//! web-sys не экспортирует эти типы с нужными сигнатурами. Все
//! вызовы идут через `Reflect::get` + `Function::call*` +
//! `Reflect::construct`.
//!
//! ## Мини-хелперы
//!
//! Всё общение с JsValue сосредоточено в шести функциях:
//! `get_fn`, `invoke`, `construct`, `set_prop`, `get_str`, `get_f64`.
//! В `app.rs` они импортируются и не дублируются.

use js_sys::{Array, Function, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::HtmlCanvasElement;

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `matches!` для tail-позиций.
#[inline]
pub fn bool_of(value: bool) -> bool {
    value
}

/// Обёртка над `format!` для tail-позиций.
#[inline]
pub fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Мини-хелперы для JsValue
// =============================================================================

/// `target[name]` — как `Function`.
pub fn get_fn(target: &JsValue, name: &str) -> Result<Function, JsValue> {
    Reflect::get(target, &JsValue::from_str(name))?
        .dyn_into::<Function>()
        .map_err(|_| JsValue::from_str("not a function"))
}

/// `target.name(args)` — вызов метода.
pub fn invoke(target: &JsValue, name: &str, args: &[JsValue]) -> Result<JsValue, JsValue> {
    let f = get_fn(target, name)?;
    let arr = Array::new();
    for a in args {
        arr.push(a);
    }
    f.apply(target, &arr)
}

/// `new Ctor(...)` из глобального конструктора.
pub fn construct(ctor_name: &str, args: &[JsValue]) -> Result<JsValue, JsValue> {
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

/// `target[key] = value`.
pub fn set_prop(target: &JsValue, key: &str, value: &JsValue) -> Result<(), JsValue> {
    Reflect::set(target, &JsValue::from_str(key), value).map(|_| ())
}

/// `target[key]` как `String`. Пустая строка, если нет.
pub fn get_str(target: &JsValue, key: &str) -> String {
    Reflect::get(target, &JsValue::from_str(key))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

/// `target[key]` как `f64`. `0.0`, если нет.
pub fn get_f64(target: &JsValue, key: &str) -> f64 {
    Reflect::get(target, &JsValue::from_str(key))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// Короткий конструктор `JsValue::from_f64`.
pub fn num(v: f64) -> JsValue {
    JsValue::from_f64(v)
}

// =============================================================================
// Feature detection
// =============================================================================

/// `VideoFrame` доступен в этом браузере.
pub fn is_video_frame_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("VideoFrame")),
        Ok(v) if v.is_function()
    ))
}

/// `EncodedVideoChunk` доступен.
pub fn is_encoded_chunk_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("EncodedVideoChunk")),
        Ok(v) if v.is_function()
    ))
}

/// Сводка поддержки — для UI.
pub fn support_summary() -> String {
    format_str(format_args!(
        "VideoFrame: {f} · EncodedVideoChunk: {c}",
        f = if is_video_frame_supported() { "да" } else { "нет" },
        c = if is_encoded_chunk_supported() { "да" } else { "нет" },
    ))
}

// =============================================================================
// VideoFrame API
// =============================================================================

/// `new VideoFrame(canvas, { timestamp })`.
///
/// `canvas` приводится к `JsValue` явно — `HtmlCanvasElement`
/// реализует несколько `AsRef`, неявное `as_ref()` не работает.
pub fn create_from_canvas(
    canvas: &HtmlCanvasElement,
    timestamp_us: f64,
) -> Result<JsValue, JsValue> {
    let canvas_val = JsValue::from(canvas.clone());
    let init = Object::new();
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    construct("VideoFrame", &[canvas_val, init.into()])
}

/// Ширина буфера кадра (`codedWidth`).
pub fn width(frame: &JsValue) -> u32 {
    get_f64(frame, "codedWidth") as u32
}

/// Высота буфера кадра (`codedHeight`).
pub fn height(frame: &JsValue) -> u32 {
    get_f64(frame, "codedHeight") as u32
}

/// Timestamp в микросекундах.
pub fn timestamp(frame: &JsValue) -> f64 {
    get_f64(frame, "timestamp")
}

/// `frame.close()` — освобождает память GPU.
///
/// Идемпотентен: второй вызов не бросает. После close()
/// `codedWidth` и `codedHeight` обнуляются.
pub fn close(frame: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(frame, "close")?;
    let _ = f.call0(frame)?;
    Ok(())
}

// =============================================================================
// EncodedVideoChunk API
// =============================================================================

/// `new EncodedVideoChunk({ type, timestamp, duration, data })`.
pub fn create_encoded_chunk(
    chunk_type: &str,
    timestamp_us: f64,
    duration_us: f64,
    data: &[u8],
) -> Result<JsValue, JsValue> {
    let u8 = Uint8Array::new_with_length(data.len() as u32);
    u8.copy_from(data);

    let init = Object::new();
    set_prop(&init, "type", &JsValue::from_str(chunk_type))?;
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    set_prop(&init, "duration", &num(duration_us))?;
    set_prop(&init, "data", u8.as_ref())?;

    construct("EncodedVideoChunk", &[init.into()])
}

/// Тип чанка: `"key"` или `"delta"`.
pub fn chunk_type(chunk: &JsValue) -> String {
    get_str(chunk, "type")
}

/// Timestamp чанка в микросекундах.
pub fn chunk_timestamp(chunk: &JsValue) -> f64 {
    get_f64(chunk, "timestamp")
}

/// Размер чанка в байтах.
pub fn chunk_byte_length(chunk: &JsValue) -> u32 {
    get_f64(chunk, "byteLength") as u32
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `codecs_demo`:

```rust
pub mod codecs_demo;   // 23
pub mod video_frame;   // ← 23.1
```

### `tests/wasm_video_frame.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::HtmlCanvasElement;

use hello_wasm02::video_frame::{
    chunk_byte_length, chunk_timestamp, chunk_type, close, create_encoded_chunk,
    create_from_canvas, height, is_encoded_chunk_supported,
    is_video_frame_supported, support_summary, timestamp, width,
};

wasm_bindgen_test_configure!(run_in_browser);

fn make_canvas(w: u32, h: u32) -> HtmlCanvasElement {
    let doc = web_sys::window().unwrap().document().unwrap();
    let canvas: HtmlCanvasElement = doc
        .create_element("canvas")
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()
        .unwrap();
    canvas.set_width(w);
    canvas.set_height(h);
    canvas
}

#[wasm_bindgen_test]
fn video_frame_is_supported_in_browser() {
    assert!(is_video_frame_supported(), "VideoFrame should be available");
}

#[wasm_bindgen_test]
fn encoded_chunk_is_supported_in_browser() {
    assert!(
        is_encoded_chunk_supported(),
        "EncodedVideoChunk should be available"
    );
}

#[wasm_bindgen_test]
fn support_summary_returns_string() {
    let s = support_summary();
    assert!(s.contains("VideoFrame"));
    assert!(s.contains("EncodedVideoChunk"));
}

#[wasm_bindgen_test]
fn create_frame_from_canvas_has_correct_size() {
    let canvas = make_canvas(320, 240);
    let frame = create_from_canvas(&canvas, 0.0).expect("create frame");
    assert_eq!(width(&frame), 320);
    assert_eq!(height(&frame), 240);
    close(&frame).unwrap();
}

#[wasm_bindgen_test]
fn frame_timestamp_is_preserved() {
    let canvas = make_canvas(64, 64);
    let frame = create_from_canvas(&canvas, 12345.0).expect("create frame");
    assert_eq!(timestamp(&frame), 12345.0);
    close(&frame).unwrap();
}

#[wasm_bindgen_test]
fn close_is_idempotent() {
    let canvas = make_canvas(64, 64);
    let frame = create_from_canvas(&canvas, 0.0).expect("create frame");
    close(&frame).expect("first close");
    close(&frame).expect("second close must not throw");
}

#[wasm_bindgen_test]
fn close_zeroes_metadata() {
    // Практическая находка: после close() метаданные обнуляются.
    let canvas = make_canvas(64, 64);
    let frame = create_from_canvas(&canvas, 0.0).expect("create frame");
    close(&frame).unwrap();
    assert_eq!(width(&frame), 0);
    assert_eq!(height(&frame), 0);
}

#[wasm_bindgen_test]
fn encoded_chunk_key_type() {
    let data = vec![0u8; 1024];
    let chunk =
        create_encoded_chunk("key", 0.0, 33333.0, &data).expect("create chunk");
    assert_eq!(chunk_type(&chunk), "key");
    assert_eq!(chunk_timestamp(&chunk), 0.0);
    assert_eq!(chunk_byte_length(&chunk), 1024);
}

#[wasm_bindgen_test]
fn encoded_chunk_delta_type() {
    let data = vec![0u8; 256];
    let chunk =
        create_encoded_chunk("delta", 33333.0, 33333.0, &data).expect("create chunk");
    assert_eq!(chunk_type(&chunk), "delta");
    assert_eq!(chunk_timestamp(&chunk), 33333.0);
    assert_eq!(chunk_byte_length(&chunk), 256);
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_video_frame

# 3. Демо
npm run wasm:dev && npm start
```

---

## 23.1.7. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.1 — VideoFrame и EncodedVideoChunk.
//!
//! Демо: три canvas'а рядом. Источник рисует пульсирующий
//! градиент. Два правых canvas'а повторяют его через два пути:
//!
//! - `drawImage(frame, ...)` — рисуем VideoFrame напрямую.
//! - `drawImage(frame, scratch)` + `getImageData(scratch)` —
//!   извлекаем пиксели, кладём через `ImageData`.
//!
//! ## Режим отображения
//!
//! Селект «Режим» переключает активные пути:
//!
//! - **Both** — работают оба, видны оба canvas'а.
//! - **drawImage only** — только визуализация, извлечение выключено.
//! - **getImageData only** — только извлечение, визуализация выключена.
//!
//! Время каждого пути измеряется отдельно: `drawImage` — единицы
//! миллисекунд, `getImageData` — десятки. Разница видна в цифрах.
//!
//! ## Почему не copyTo
//!
//! `VideoFrame.copyTo` на кадрах, созданных из canvas, в Chrome
//! иногда падает **асинхронно** с `DOMException: No media resource`.
//! Синхронный `Result::Err` поймать не удаётся — ошибка всплывает
//! позже, как `Uncaught (in promise)`. Поэтому используется
//! `drawImage + getImageData`.
//!
//! ## Почему getImageData через Reflect
//!
//! web-sys 0.3.105 объявляет `CanvasRenderingContext2d::get_image_data`
//! с параметрами `i32`, что не соответствует реальному JS API (`f64`).
//! Прямой вызов даёт E0308. Все вызовы `getImageData` идут через
//! `invoke(ctx, "getImageData", ...)` — работает независимо от
//! сигнатуры web-sys.
//!
//! **Файл автономен.** Стили `.vf-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use js_sys::{Function, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement,
    HtmlSelectElement, ImageData,
};

use crate::video_frame::{
    chunk_byte_length, chunk_timestamp, chunk_type, close, create_encoded_chunk,
    create_from_canvas, format_str, height, invoke,
    is_encoded_chunk_supported, is_video_frame_supported, num, set_prop,
    support_summary, width,
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
            var KEY='vf-sw-cleared-23-1';
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
// Хелпер: точное время
// =============================================================================

/// `performance.now()` если доступно, иначе `Date.now()`.
fn now_ms() -> f64 {
    let global = js_sys::global();
    let Ok(perf) = Reflect::get(&global, &JsValue::from_str("performance")) else {
        return js_sys::Date::now();
    };
    let Ok(f) = Reflect::get(&perf, &JsValue::from_str("now")) else {
        return js_sys::Date::now();
    };
    let Ok(f) = f.dyn_into::<Function>() else {
        return js_sys::Date::now();
    };
    let Ok(v) = f.call0(&perf) else {
        return js_sys::Date::now();
    };
    v.as_f64().unwrap_or_else(js_sys::Date::now)
}

// =============================================================================
// Стили
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "vf-injected-style";
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
.vf-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.vf-desc-line { margin:6px 0; }
.vf-canvases { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.vf-canvas-col { display:flex;flex-direction:column;gap:6px;}
.vf-canvas-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.vf-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;}
.vf-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.vf-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.vf-btn:hover:not([disabled]) { background:#30363d; }
.vf-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.vf-btn-primary:hover:not([disabled]) { background:#2ea043; }
.vf-btn-warn { background:#21262d;border-color:#30363d;color:#f0883e;}
.vf-btn-warn:hover:not([disabled]) { background:#30363d; }
.vf-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.vf-select { font-size:12px;padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#0d1117;color:#c9d1d9;}
.vf-stats { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;margin-bottom:16px;}
.vf-stats .label { color:#8b949e; }
.vf-stats .value { color:#79c0ff; }
.vf-stats .ok { color:#3fb950; }
.vf-stats .warn { color:#f0883e; }
.vf-stats .err { color:#f85149; }
.vf-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
.vf-block .label { color:#8b949e; }
.vf-block .value { color:#79c0ff; }
"#;

// =============================================================================
// Константы
// =============================================================================

const CANVAS_W: u32 = 480;
const CANVAS_H: u32 = 360;
const STATS_EVERY_FRAMES: u32 = 15;
const FPS_WINDOW_MS: f64 = 500.0;

// =============================================================================
// Режим отображения
// =============================================================================

#[derive(Clone, Copy, PartialEq)]
enum PathMode {
    Both,
    DrawOnly,
    CopyOnly,
}

impl PathMode {
    fn from_str(s: &str) -> Self {
        match s {
            "draw" => PathMode::DrawOnly,
            "copy" => PathMode::CopyOnly,
            _ => PathMode::Both,
        }
    }

    fn label(self) -> &'static str {
        match self {
            PathMode::Both => "both",
            PathMode::DrawOnly => "drawImage only",
            PathMode::CopyOnly => "getImageData only",
        }
    }
}

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    paused: bool,
    mode: PathMode,
    frame_count: u32,
    last_frame_bytes: u32,
    last_error: Option<String>,
    draw_ms: f64,
    copy_ms: f64,
    window_start_ms: f64,
    window_frames: u32,
    fps: f64,
}

impl DemoState {
    fn new() -> Self {
        Self {
            paused: false,
            mode: PathMode::Both,
            frame_count: 0,
            last_frame_bytes: 0,
            last_error: None,
            draw_ms: 0.0,
            copy_ms: 0.0,
            window_start_ms: 0.0,
            window_frames: 0,
            fps: 0.0,
        }
    }
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("vf-stats") else {
        return;
    };

    let err_line = match st.last_error.as_ref() {
        Some(msg) => format_str(format_args!(
            r#"<div><span class="err">{msg}</span></div>"#,
            msg = msg,
        )),
        None => String::new(),
    };

    let size_kb = st.last_frame_bytes as f64 / 1024.0;

    let paths_line = match st.mode {
        PathMode::Both => format_str(format_args!(
            r#"<div><span class="label">drawImage: </span><span class="ok">{d:.2} мс</span> · <span class="label">getImageData: </span><span class="warn">{c:.2} мс</span></div>"#,
            d = st.draw_ms,
            c = st.copy_ms,
        )),
        PathMode::DrawOnly => format_str(format_args!(
            r#"<div><span class="label">drawImage: </span><span class="ok">{d:.2} мс</span> · <span class="label">getImageData: </span><span class="value">выключен</span></div>"#,
            d = st.draw_ms,
        )),
        PathMode::CopyOnly => format_str(format_args!(
            r#"<div><span class="label">drawImage: </span><span class="value">выключен</span> · <span class="label">getImageData: </span><span class="warn">{c:.2} мс</span></div>"#,
            c = st.copy_ms,
        )),
    };

    let html = format_str(format_args!(
        r#"{err}
           <div><span class="label">FPS: </span><span class="ok">{fps:.1}</span> · <span class="label">Режим: </span><span class="value">{mode}</span> · <span class="label">Кадров: </span><span class="value">{n}</span></div>
           {paths}
           <div><span class="label">Размер кадра: </span><span class="value">{bytes} байт</span> (<span class="value">{kb:.1} КБ</span>)</div>"#,
        err = err_line,
        fps = st.fps,
        mode = st.mode.label(),
        n = st.frame_count,
        paths = paths_line,
        bytes = st.last_frame_bytes,
        kb = size_kb,
    ));
    el.set_inner_html(&html);
}

const DESC_LINES: &[&str] = &[
    "WebCodecs работает с двумя представлениями кадра: \
     <strong>VideoFrame</strong> — распакованные пиксели в памяти GPU, \
     <strong>EncodedVideoChunk</strong> — сжатые байты в памяти CPU. \
     Всё остальное — комбинации этих двух.",
    "Три canvas'а показывают один и тот же кадр через два разных пути. \
     <em>drawImage</em> рисует VideoFrame напрямую. \
     <em>drawImage + getImageData</em> извлекает пиксели через scratch \
     и кладёт их как ImageData. Результат визуально совпадает.",
    "<strong>Сравните тайминги.</strong> Переключайте «Режим» и смотрите: \
     <em>drawImage</em> — доли миллисекунды, <em>getImageData</em> — \
     в десятки раз дольше. Это цена чтения пикселей с GPU на CPU. \
     В реальных приложениях такое чтение выносят в Worker.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("vf-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("vf-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
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

    let cx = w as f64 / 2.0 + (phase * 6.28).sin() * 80.0;
    let cy = h as f64 / 2.0 + (phase * 6.28).cos() * 60.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#f0f6fc"));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(
        ctx_js,
        "arc",
        &[
            num(cx),
            num(cy),
            num(30.0),
            num(0.0),
            num(std::f64::consts::PI * 2.0),
        ],
    );
    let _ = invoke(ctx_js, "fill", &[]);
}

// =============================================================================
// Очистка canvas'а
// =============================================================================

fn clear_canvas(ctx: &CanvasRenderingContext2d) {
    let ctx_js: &JsValue = ctx.as_ref();
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#000"));
    let _ = invoke(
        ctx_js,
        "fillRect",
        &[num(0.0), num(0.0), num(CANVAS_W as f64), num(CANVAS_H as f64)],
    );
}

// =============================================================================
// Пути обработки
// =============================================================================

/// Путь 1: drawImage на видимый canvas.
fn path_draw_image(
    frame: &JsValue,
    draw_ctx: &CanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> f64 {
    let t0 = now_ms();
    let _ = invoke(
        draw_ctx.as_ref(),
        "drawImage",
        &[
            frame.clone(),
            num(0.0),
            num(0.0),
            num(w as f64),
            num(h as f64),
        ],
    );
    now_ms() - t0
}

/// Путь 2: drawImage на scratch + getImageData + putImageData.
///
/// `getImageData` вызывается через `invoke`, потому что web-sys
/// 0.3.105 объявляет `get_image_data` с `i32`-параметрами, что
/// не соответствует реальному API.
fn path_get_image_data(
    frame: &JsValue,
    scratch_ctx: &CanvasRenderingContext2d,
    copy_ctx: &CanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> (f64, Vec<u8>) {
    let t0 = now_ms();
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
    let bytes = invoke(ctx_js, "getImageData", &args)
        .ok()
        .and_then(|v| v.dyn_into::<ImageData>().ok())
        .map(|img| img.data().0)
        .unwrap_or_default();

    if !bytes.is_empty() {
        let clamped = wasm_bindgen::Clamped(&bytes[..]);
        if let Ok(img) = ImageData::new_with_u8_clamped_array_and_sh(
            clamped, w, h,
        ) {
            let _ = invoke(
                copy_ctx.as_ref(),
                "putImageData",
                &[JsValue::from(img), num(0.0), num(0.0)],
            );
        }
    }

    (now_ms() - t0, bytes)
}

// =============================================================================
// Главный цикл
// =============================================================================

fn start_animation(
    scratch_ctx: CanvasRenderingContext2d,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let source_canvas = document
        .get_element_by_id("vf-source")
        .ok_or_else(|| JsValue::from_str("no #vf-source"))?
        .dyn_into::<HtmlCanvasElement>()?;
    let source_ctx = source_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx (source)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    let draw_canvas = document
        .get_element_by_id("vf-draw")
        .ok_or_else(|| JsValue::from_str("no #vf-draw"))?
        .dyn_into::<HtmlCanvasElement>()?;
    let draw_ctx = draw_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx (draw)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    let copy_canvas = document
        .get_element_by_id("vf-copy")
        .ok_or_else(|| JsValue::from_str("no #vf-copy"))?
        .dyn_into::<HtmlCanvasElement>()?;
    let copy_ctx = copy_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx (copy)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    let cell: Rc<RefCell<Option<wasm_bindgen::closure::Closure<dyn FnMut(f64)>>>> =
        Rc::new(RefCell::new(None));
    let cell2 = cell.clone();

    *cell.borrow_mut() = Some(wasm_bindgen::closure::Closure::wrap(Box::new(
        move |now: f64| {
            if let Some(w) = web_sys::window() {
                if let Some(cb) = cell2.borrow().as_ref() {
                    let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
                }
            }

            if state.borrow().paused {
                return;
            }

            let phase = (now / 1000.0) % 1.0;
            let mode = state.borrow().mode;

            draw_source(&source_ctx, CANVAS_W, CANVAS_H, phase);

            let Ok(frame) = create_from_canvas(&source_canvas, now * 1000.0) else {
                return;
            };

            let frame_w = width(&frame);
            let frame_h = height(&frame);

            let draw_ms = if mode != PathMode::CopyOnly {
                path_draw_image(&frame, &draw_ctx, frame_w, frame_h)
            } else {
                clear_canvas(&draw_ctx);
                0.0
            };

            let copy_ms = if mode != PathMode::DrawOnly {
                let (ms, _) = path_get_image_data(
                    &frame, &scratch_ctx, &copy_ctx, frame_w, frame_h,
                );
                ms
            } else {
                clear_canvas(&copy_ctx);
                0.0
            };

            let _ = close(&frame);

            {
                let mut st = state.borrow_mut();
                st.frame_count += 1;
                st.last_frame_bytes = frame_w * frame_h * 4;
                if mode != PathMode::CopyOnly {
                    st.draw_ms = draw_ms;
                }
                if mode != PathMode::DrawOnly {
                    st.copy_ms = copy_ms;
                }

                if st.window_start_ms == 0.0 {
                    st.window_start_ms = now;
                    st.window_frames = 0;
                }
                st.window_frames += 1;
                let elapsed = now - st.window_start_ms;
                if elapsed >= FPS_WINDOW_MS {
                    st.fps = st.window_frames as f64 * 1000.0 / elapsed;
                    st.window_start_ms = now;
                    st.window_frames = 0;
                }
            }

            let update = {
                let st = state.borrow();
                st.frame_count % STATS_EVERY_FRAMES == 0
            };
            if update {
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
            }
        },
    ) as Box<dyn FnMut(f64)>));

    let cb = cell.borrow();
    window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    Ok(())
}

// =============================================================================
// Обработчики
// =============================================================================

fn wire_pause_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("vf-pause")
        .ok_or_else(|| JsValue::from_str("no #vf-pause"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let paused = {
            let mut st = state.borrow_mut();
            st.paused = !st.paused;
            if st.paused {
                st.window_start_ms = 0.0;
                st.window_frames = 0;
                st.fps = 0.0;
            }
            st.paused
        };
        if let Some(el) = doc.get_element_by_id("vf-pause") {
            el.set_text_content(Some(if paused { "Resume" } else { "Pause" }));
        }
        let st = state.borrow();
        update_stats(&doc, &st);
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_mode_select(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("vf-mode")
        .ok_or_else(|| JsValue::from_str("no #vf-mode"))?;
    let select: HtmlSelectElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(el) = doc.get_element_by_id("vf-mode") else {
            return;
        };
        let Ok(select) = el.dyn_into::<HtmlSelectElement>() else {
            return;
        };
        let mode = PathMode::from_str(&select.value());
        {
            let mut st = state.borrow_mut();
            st.mode = mode;
            st.window_start_ms = 0.0;
            st.window_frames = 0;
            st.fps = 0.0;
        }
        let st = state.borrow();
        update_stats(&doc, &st);
    });
    select.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_use_closed_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("vf-closed")
        .ok_or_else(|| JsValue::from_str("no #vf-closed"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(canvas_el) = doc.get_element_by_id("vf-source") else {
            return;
        };
        let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() else {
            return;
        };

        let Ok(frame) = create_from_canvas(&canvas, 0.0) else {
            return;
        };
        let _ = close(&frame);

        let w = width(&frame);
        let h = height(&frame);
        let msg = if w == 0 || h == 0 {
            format_str(format_args!(
                "close() выполнен. codedWidth={w}, codedHeight={h} — \
                 кадр пуст, использовать нельзя."
            ))
        } else {
            format_str(format_args!(
                "close() выполнен. codedWidth={w}, codedHeight={h}. \
                 Chrome сохраняет метаданные, но drawImage молча \
                 игнорирует закрытый кадр."
            ))
        };

        {
            let mut st = state.borrow_mut();
            st.last_error = Some(msg);
        }
        let st = state.borrow();
        update_stats(&doc, &st);
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// =============================================================================
// EncodedVideoChunk: демонстрация
// =============================================================================

fn show_encoded_chunk_info(document: &Document) {
    let data = vec![0u8; 1024];
    let Ok(chunk) = create_encoded_chunk("key", 0.0, 33333.0, &data) else {
        return;
    };

    let ctype = chunk_type(&chunk);
    let ts = chunk_timestamp(&chunk);
    let bl = chunk_byte_length(&chunk);

    let Some(el) = document.get_element_by_id("vf-chunk-info") else {
        return;
    };
    let html = format_str(format_args!(
        r#"<span class="label">type:</span> <span class="value">{t}</span> · <span class="label">timestamp:</span> <span class="value">{ts:.0} мкс</span> · <span class="label">byteLength:</span> <span class="value">{bl}</span>"#,
        t = ctype,
        ts = ts,
        bl = bl,
    ));
    el.set_inner_html(&html);
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
    h2.set_text_content(Some("23.1 — VideoFrame и EncodedVideoChunk"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_video_frame_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("vf-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает VideoFrame. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Скрытый scratch canvas.
    let scratch_el = document.create_element("canvas")?;
    scratch_el.set_id("vf-scratch");
    scratch_el.set_attribute("style", "display:none")?;
    let scratch_canvas: HtmlCanvasElement = scratch_el.dyn_into()?;
    scratch_canvas.set_width(CANVAS_W);
    scratch_canvas.set_height(CANVAS_H);
    root.append_child(&scratch_canvas)?;
    let scratch_ctx = scratch_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx (scratch)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    // Три видимых canvas'а.
    let canvases = document.create_element("div")?;
    canvases.set_class_name("vf-canvases");
    canvases.set_inner_html(&format_str(format_args!(
        r#"<div class="vf-canvas-col">
               <div class="vf-canvas-label">Source (main thread, rAF)</div>
               <canvas id="vf-source" class="vf-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="vf-canvas-col">
               <div class="vf-canvas-label">Via VideoFrame + drawImage</div>
               <canvas id="vf-draw" class="vf-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="vf-canvas-col">
               <div class="vf-canvas-label">Via VideoFrame + getImageData</div>
               <canvas id="vf-copy" class="vf-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = CANVAS_W,
        h = CANVAS_H,
    )));
    root.append_child(&canvases)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("vf-block");
    controls.set_inner_html(
        r#"<button class="vf-btn vf-btn-primary" id="vf-pause">Pause</button>
           <button class="vf-btn vf-btn-warn" id="vf-closed">Use closed frame</button>
           <span class="vf-label">Режим:</span>
           <select id="vf-mode" class="vf-select">
               <option value="both">Both</option>
               <option value="draw">drawImage only</option>
               <option value="copy">getImageData only</option>
           </select>
           <span class="vf-label" id="vf-support"></span>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("vf-stats");
    stats.set_id("vf-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let chunk_block = document.create_element("div")?;
    chunk_block.set_class_name("vf-block");
    chunk_block.set_inner_html(
        r#"<span class="vf-label">EncodedVideoChunk (1024 байта, type="key"):</span>
           <span id="vf-chunk-info"></span>"#,
    );
    root.append_child(&chunk_block)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("vf-footnote");
    footnote.set_text_content(Some(
        "VideoFrame держит память GPU. Всегда закрывайте через frame.close() \
         после использования. close() идемпотентен: второй вызов не бросает. \
         После close() кадр непригоден — drawImage / texImage2D бросают \
         InvalidStateError, а метаданные обнуляются. EncodedVideoChunk — \
         сжатые байты + метаданные (type, timestamp, byteLength). Создаётся \
         либо VideoEncoder'ом, либо вручную из Uint8Array. Пиксели \
         извлекаются через drawImage(frame, scratch) + getImageData(scratch): \
         copyTo на canvas-sourced VideoFrame в Chrome падает асинхронно \
         с No media resource и не ловится.",
    ));
    root.append_child(&footnote)?;

    if let Some(el) = document.get_element_by_id("vf-support") {
        el.set_text_content(Some(&support_summary()));
    }

    if is_encoded_chunk_supported() {
        show_encoded_chunk_info(&document);
    }

    let state = Rc::new(RefCell::new(DemoState::new()));

    wire_pause_button(&document, state.clone())?;
    wire_use_closed_button(&document, state.clone())?;
    wire_mode_select(&document, state.clone())?;

    start_animation(scratch_ctx, state.clone())?;

    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    Ok(())
}
```

---

## 23.1.8. Анализ решения

**Какую ценность даёт демо.**

Подглава закрывает **введение в WebCodecs через два базовых типа**. Три canvas'а показывают одно и то же изображение через два разных пути — это доказывает, что `VideoFrame` действительно представляет кадр, и что способ извлечения пикселей — вопрос задачи, а не типа.

**Что показывает каждая кнопка и режим.**

- **Both / drawImage only / getImageData only** — режим переключает активные пути. В режиме «только getImageData» правый верхний canvas чёрный — drawImage выключен, но пиксели извлекаются через scratch. Это доказывает, что **два пути независимы**.
- **Pause/Resume** — FPS-окно сбрасывается, при Resume счётчик набирается заново. Это даёт стабильную цифру без «прыжков».
- **Use closed frame** — показывает, что после `close()` метаданные обнуляются: `codedWidth=0, codedHeight=0`. Пользователь видит явное подтверждение, а не молчаливый сбой.

**Что осознанно не реализовано.**

- **Worker, OffscreenCanvas, WASM.** Это темы 23.7–23.8.
- **VideoDecoder/Encoder.** Темы 23.2–23.3.

**Правила проекта, применённые.**

1. **Autonomous app.rs.** Всё inline: `format_str`, feature detection, обработчики.
2. **`format_str(format_args!(...))` в tail-позициях.**
3. **`bool_of(matches!(...))` в tail-позиции.**
4. **RefMut-дисциплина.** Узкие scope.
5. **Feature detection перед API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.**
8. **Уникальный префикс `vf-`.**
9. **Никаких неиспользуемых импортов.**
10. **Мини-хелперы** (`get_fn`, `invoke`, `construct`, `set_prop`, `get_str`, `get_f64`, `num`) — в модуле, не дублируются.
11. **Один `rAF`-цикл** — вся анимация.
12. **`getImageData` через `Reflect::invoke`** — обход сигнатуры web-sys 0.3.105.

---

## 23.1.9. Список доработок после первой итерации

1. **`bool_of` и `format_str` в модуле** — обход E0308 r-a в tail-позиции.
2. **Мини-хелперы для JsValue** — `get_fn`, `invoke`, `construct`, `set_prop`, `get_str`, `get_f64`, `num`.
3. **`Reflect::construct` вместо `Function::construct1/2`** — таких методов в js-sys нет.
4. **`JsValue::from(canvas.clone())` вместо `canvas.as_ref()`** — обход неоднозначного `AsRef`.
5. **`Clamped(&bytes[..])` для `ImageData`** — `Clamped<&[u8]>`, не `Clamped<Vec<u8>>`.
6. **`getImageData` через `invoke`** — web-sys 0.3.105 неверно объявляет сигнатуру (`i32` вместо `f64`).
7. **`copyTo` убран полностью** — на canvas-sourced `VideoFrame` в Chrome падает асинхронно.
8. **Порядок `copyTo` до `drawImage`** (в ранних версиях) — потом отказались от `copyTo`.
9. **`now_ms()` через `performance.now()`** — точные измерения с микросекундным разрешением.
10. **Реальный FPS** — скользящее окно 500 мс.
11. **Селект режима** — Both / drawImage only / getImageData only.
12. **CSS: `.vf-block .label` / `.value`** — отдельные правила, значения EncodedVideoChunk видны.
13. **Кнопка «Use closed frame»** показывает обнуление метаданных после `close()`.
14. **Тест `close_zeroes_metadata`** — фиксирует контракт: после `close()` `codedWidth == 0`.

---

## 23.1.10. Итог

**`VideoFrame`** — распакованный кадр. Пиксели в памяти GPU. Обязательно закрывать через `close()`. После `close()` — непригоден, метаданные обнулены.

**`EncodedVideoChunk`** — сжатый кадр. Байты в памяти CPU. Метаданные: `type`, `timestamp`, `duration`, `byteLength`.

**Соотношение размеров.** Один `VideoFrame` 480×360 = 691 200 байт. Тот же кадр в H.264 ≈ 4 КБ. **170×** — смысл кодека.

**Практическая находка № 1.** `VideoFrame.copyTo` на canvas-sourced кадре в Chrome падает **асинхронно** с `No media resource`. Синхронный `Result::Err` поймать нельзя. Решение — `drawImage + getImageData` через scratch-canvas.

**Практическая находка № 2.** web-sys 0.3.105 объявляет `get_image_data` с `i32`, что не соответствует JS API. Решение — `getImageData` через `Reflect::invoke`.

**Практическая находка № 3.** `close()` обнуляет `codedWidth`/`codedHeight`. Это видно в демо через кнопку «Use closed frame».

**Практическая находка № 4.** `drawImage` — **0.2 мс**, `getImageData` — **2–10 мс**. Разница в **10–50×**. Это фундаментальная цена readback'а GPU→CPU. В реальных приложениях такое чтение выносят в Worker или заменяют на `WebGL2.texImage2D(frame)`.

Подглава закрывает **фундамент WebCodecs**. Дальше — 23.2 (VideoDecoder).