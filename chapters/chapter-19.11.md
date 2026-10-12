# 19.11. Интеграция с WASM-библиотеками

## Обновлённая структура проекта

```
hello-wasm02/
├── src/
│   ├── fsa_api_wasm.rs          ← НОВЫЙ: обработка файлов через Rust-крейты
│   ├── app.rs                    ← ДЕМО главы 19.11 (только этот сценарий)
│   ├── lib.rs                    ← +pub mod fsa_api_wasm
│   └── … (остальные модули проекта)
├── tests/
│   ├── wasm_fsa_api_wasm.rs      ← НОВЫЙ: roundtrip PNG + self-check grayscale
│   └── … (остальные тесты)
├── pkg/                          ← артефакты wasm-pack (содержит image-крейт)
└── web/
    ├── index.html
    └── main.js
```

### Краткий комментарий

**`src/fsa_api_wasm.rs`** — обёртка над тремя движками обработки изображений: Rust-крейт `image` в WASM, нативный Canvas 2D `filter`, Canvas 2D в Worker'е. Один интерфейс — `&[u8]` PNG на входе, `Vec<u8>` PNG на выходе. Функция `generate_test_png` принимает угол градиента — благодаря этому каждый прогон даёт визуально уникальную картинку.

**`tests/wasm_fsa_api_wasm.rs`** — программная генерация PNG, проверка grayscale-результата, устойчивость к «мусорным» байтам, проверка, что разные углы дают разные изображения.

**`src/app.rs`** — демо главы 19.11: при загрузке страницы автоматически генерируется PNG и прогоняются все три движка. Каждый следующий прогон создаёт новый исходник, при этом строки таблицы **независимы** — прогон одного движка не стирает результаты других.

### Ссылки для изучения

- [Rust and WebAssembly: Reference](https://rustwasm.github.io/docs/book/) — базовая книга по WASM в Rust.
- [image crate docs](https://docs.rs/image/latest/image/) — декодирование/кодирование PNG, JPEG, WebP.
- [MDN: OffscreenCanvas](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas) — Canvas вне DOM, работает в Worker'е.
- [MDN: createImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/createImageBitmap) — декодирование Blob → ImageBitmap.
- [web.dev: OffscreenCanvas](https://web.dev/articles/offscreen-canvas) — зачем Worker'у Canvas.

---

## 19.11.1. Ключевые идеи

**1. Байты — универсальный интерфейс.** И файл из OPFS, и Blob из Canvas, и результат WASM-обработки — это всё `&[u8]` / `Vec<u8>`. Один контракт для всех источников.

**2. WASM не всегда быстрее.** Grayscale через Canvas 2D `filter = 'grayscale(1)'` использует нативный C++ движка, оптимизированный годами. WASM-версия с `image` crate честно проигрывает в этой конкретной задаче — но выигрывает в **портируемости**: тот же Rust-код работает в браузере и в CLI.

**3. Worker не ускоряет одну операцию, но не блокирует UI.** Тот же Canvas-код в Worker'е занимает столько же миллисекунд, но main thread остаётся отзывчивым.

**4. Правило «feature-agnostic».** Вызывающий код не знает, какой движок используется. Все три движка принимают `&[u8]`, возвращают `Vec<u8>`.

**5. Каждый прогон — новый источник.** Параметр `angle` в `generate_test_png` даёт визуально разную картинку. Это доказывает, что перезапуск не «переиспользует» старый результат.

**6. Строки таблицы независимы.** Прогон одного движка не стирает результаты других — WASM, Native и Worker накапливают свою уникальную историю результатов.

---

## 19.11.2. Новый модуль: `src/fsa_api_wasm.rs`

```rust
//! Интеграция с WASM-библиотеками.
//!
//! **Ключевая идея.** WASM позволяет использовать Rust-крейты
//! (`image`, `csv`, `zip`, `sha2`) прямо в браузере. Байты файла
//! попадают в Rust без промежуточных копий, обрабатываются,
//! результат возвращается как `Vec<u8>` и записывается обратно.
//!
//! **Три движка одной задачи.** Grayscale-преобразование PNG:
//! * `grayscale_wasm` — Rust-крейт `image` в WASM, main thread.
//! * `grayscale_native` — Canvas 2D `filter = 'grayscale(1)'`,
//!   main thread. Браузер оптимизирует это нативно.
//! * `grayscale_worker` — Canvas 2D в Worker'е. Main thread
//!   не блокируется.
//!
//! **Один интерфейс, три движка.** Все три принимают `&[u8]` PNG,
//! возвращают `Vec<u8>` PNG. Вызывающий код не знает деталей.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции `if/else`/
//! `match`/блока иногда выводится как `()` → E0308. Все такие
//! места обёрнуты в `format_str(format_args!(…))`.

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileSystemFileHandle, ImageBitmap, MessageEvent, OffscreenCanvas, Worker};

use crate::fsa_api_file::get_file;

// ===========================================================================
// Строковые хелперы (правило WASM-инференса)
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

// ===========================================================================
// File → Vec<u8>
// ===========================================================================

/// Прочитать файл как байты через `File.arrayBuffer()`.
///
/// `arrayBuffer()` — самый быстрый способ получить байты из Blob:
/// один вызов, одна копия. `FileReader.readAsArrayBuffer` — это
/// уже устаревший путь, `arrayBuffer()` предпочтительнее.
pub async fn read_bytes(handle: &FileSystemFileHandle) -> Result<Vec<u8>, JsValue> {
    let file = get_file(handle).await?;
    let promise = file.array_buffer();
    let value = JsFuture::from(promise).await?;
    let ab: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("arrayBuffer вернул не ArrayBuffer"))?;
    let u8arr = js_sys::Uint8Array::new(&ab);
    let len = u8arr.length() as usize;
    let mut out: Vec<u8> = vec![0u8; len];
    u8arr.copy_to(&mut out);
    Ok(out)
}

// ===========================================================================
// Движок 1: WASM (Rust-крейт image)
// ===========================================================================

/// Grayscale через Rust-крейт `image` в WASM.
///
/// Принимает байты PNG, декодирует, конвертирует в luma (один
/// канал), кодирует обратно в PNG. Всё в одном вызове — никаких
/// промежуточных копий через JS.
pub fn grayscale_wasm(png_bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    use std::io::Cursor;

    let img = image::load_from_memory(png_bytes).map_err(|e| {
        let msg = format_str(format_args!("image decode: {e}"));
        JsValue::from_str(&msg)
    })?;

    let gray = img.to_luma8();

    let mut out: Vec<u8> = Vec::new();
    {
        let mut cursor = Cursor::new(&mut out);
        gray.write_to(&mut cursor, image::ImageFormat::Png)
            .map_err(|e| {
                let msg = format_str(format_args!("image encode: {e}"));
                JsValue::from_str(&msg)
            })?;
    }
    Ok(out)
}

// ===========================================================================
// Движок 2: Native Canvas (main thread)
// ===========================================================================

/// Grayscale через Canvas 2D `filter = 'grayscale(1)'`.
///
/// Браузер выполняет фильтр нативно — как правило, быстрее, чем
/// WASM, потому что это оптимизированный C++ движка.
pub async fn grayscale_native(png_bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    let bitmap = bytes_to_bitmap(png_bytes).await?;

    let w: u32 = bitmap.width();
    let h: u32 = bitmap.height();
    let canvas = OffscreenCanvas::new(w, h)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("нет 2d контекста"))?;
    let ctx: web_sys::OffscreenCanvasRenderingContext2d = ctx_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не OffscreenCanvasRenderingContext2d"))?;

    ctx.set_filter("grayscale(1)");
    ctx.draw_image_with_image_bitmap(&bitmap, 0.0, 0.0)?;

    canvas_to_bytes(&canvas).await
}

// ===========================================================================
// Движок 3: Worker (Canvas в отдельном потоке)
// ===========================================================================

/// JS-код inline-Worker'а для grayscale через OffscreenCanvas.
const WORKER_JS: &str = r#"
    self.onmessage = async (e) => {
        const data = e.data;
        try {
            const blob = new Blob([data], { type: 'image/png' });
            const bitmap = await createImageBitmap(blob);
            const canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
            const ctx = canvas.getContext('2d');
            ctx.filter = 'grayscale(1)';
            ctx.drawImage(bitmap, 0, 0);
            const outBlob = await canvas.convertToBlob({ type: 'image/png' });
            const buf = await outBlob.arrayBuffer();
            self.postMessage({ ok: true, buf });
        } catch (err) {
            self.postMessage({ ok: false, error: String(err) });
        }
    };
"#;

/// Grayscale в Worker'е через OffscreenCanvas.
///
/// Тот же алгоритм, что и `grayscale_native`, но в отдельном
/// потоке — main thread не блокируется.
pub async fn grayscale_worker(png_bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    let parts = js_sys::Array::new();
    parts.push(&JsValue::from_str(WORKER_JS));
    let blob = Blob::new_with_str_sequence(&parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    let u8arr = js_sys::Uint8Array::from(png_bytes);
    worker.post_message(&u8arr.buffer())?;

    let worker_ref: &Worker = &worker;
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let onmessage = Closure::once(move |e: MessageEvent| {
            let _ = resolve.call1(&JsValue::UNDEFINED, &e.data());
        });
        worker_ref.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    });
    let value: JsValue = JsFuture::from(promise).await?;
    worker.terminate();

    let ok = js_sys::Reflect::get(&value, &JsValue::from_str("ok"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
    if !ok {
        let err = js_sys::Reflect::get(&value, &JsValue::from_str("error"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| "unknown worker error".to_string());
        return Err(JsValue::from_str(&err));
    }
    let buf = js_sys::Reflect::get(&value, &JsValue::from_str("buf"))?;
    let ab: js_sys::ArrayBuffer = buf
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let u8_out = js_sys::Uint8Array::new(&ab);
    let len = u8_out.length() as usize;
    let mut out: Vec<u8> = vec![0u8; len];
    u8_out.copy_to(&mut out);
    Ok(out)
}

// ===========================================================================
// Вспомогательные преобразования
// ===========================================================================

async fn bytes_to_bitmap(png_bytes: &[u8]) -> Result<ImageBitmap, JsValue> {
    let u8arr = js_sys::Uint8Array::from(png_bytes);
    let parts = js_sys::Array::new();
    parts.push(&u8arr.buffer());
    let blob = Blob::new_with_buffer_source_sequence(&parts)?;

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let promise = window.create_image_bitmap_with_blob(&blob)?;
    let value = JsFuture::from(promise).await?;
    value
        .dyn_into::<ImageBitmap>()
        .map_err(|_| JsValue::from_str("не ImageBitmap"))
}

async fn canvas_to_bytes(canvas: &OffscreenCanvas) -> Result<Vec<u8>, JsValue> {
    let promise = canvas.convert_to_blob()?;
    let value = JsFuture::from(promise).await?;
    let blob: Blob = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Blob"))?;
    let ab_value = JsFuture::from(blob.array_buffer()).await?;
    let ab: js_sys::ArrayBuffer = ab_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let u8arr = js_sys::Uint8Array::new(&ab);
    let len = u8arr.length() as usize;
    let mut out: Vec<u8> = vec![0u8; len];
    u8arr.copy_to(&mut out);
    Ok(out)
}

// ===========================================================================
// Диагностика для тестов
// ===========================================================================

/// Проверить, что изображение grayscale — все пиксели имеют R==G==B.
///
/// Сравнивать результаты разных движков попиксельно бессмысленно —
/// они используют разные цветовые пространства и разные алгоритмы
/// сведения. А вот проверить, что **результат действительно
/// grayscale** — можно и нужно.
pub fn check_grayscale(png_bytes: &[u8]) -> Result<(), JsValue> {
    let img = image::load_from_memory(png_bytes).map_err(|e| {
        let msg = format_str(format_args!("decode: {e}"));
        JsValue::from_str(&msg)
    })?;
    let rgba = img.to_rgba8();
    for (x, y, p) in rgba.enumerate_pixels() {
        let [r, g, b, _] = p.0;
        if r != g || g != b {
            let msg = format_str(format_args!(
                "pixel ({x},{y}) не grayscale: R={r} G={g} B={b}"
            ));
            return Err(JsValue::from_str(&msg));
        }
    }
    Ok(())
}

/// Сгенерировать тестовый PNG с градиентом под углом `angle` (радианы).
///
/// Рисует цветной паттерн: R — градиент вдоль оси X, повёрнутой на
/// `angle`; G — вдоль оси Y, повёрнутой на `angle`; B — константа.
///
/// Каждый вызов с новым углом даёт **визуально другую картинку**.
/// Это нужно демо: пользователь видит, что перезапуск действительно
/// пересоздал исходник, а не переиспользовал старый.
pub fn generate_test_png(
    width: u32,
    height: u32,
    angle: f64,
) -> Result<Vec<u8>, JsValue> {
    use image::{ImageBuffer, Rgb, RgbImage};

    let w = width.max(1);
    let h = height.max(1);

    let wf = w as f32;
    let hf = h as f32;
    let cx = wf / 2.0;
    let cy = hf / 2.0;
    let cos_a = (angle as f32).cos();
    let sin_a = (angle as f32).sin();

    let img: RgbImage = ImageBuffer::from_fn(w, h, |x, y| {
        let dx = x as f32 - cx;
        let dy = y as f32 - cy;

        // Проекция на повёрнутую ось X.
        let u = dx * cos_a + dy * sin_a;
        // Проекция на повёрнутую ось Y.
        let v = -dx * sin_a + dy * cos_a;

        let tr = ((u / wf) + 0.5).clamp(0.0, 1.0);
        let tg = ((v / hf) + 0.5).clamp(0.0, 1.0);

        let r = (tr * 255.0) as u8;
        let g = (tg * 255.0) as u8;
        let b = 128u8;

        Rgb([r, g, b])
    });

    let mut out: Vec<u8> = Vec::new();
    {
        let mut cursor = std::io::Cursor::new(&mut out);
        img.write_to(&mut cursor, image::ImageFormat::Png)
            .map_err(|e| {
                let msg = format_str(format_args!("encode: {e}"));
                JsValue::from_str(&msg)
            })?;
    }
    Ok(out)
}
```

### Что важно в модуле

**1. `read_bytes` через `File.arrayBuffer()`.** Самый быстрый способ получить байты: один промис, одна копия. `FileReader.readAsArrayBuffer` — устаревший путь.

**2. `grayscale_wasm` — синхронная.** `image::load_from_memory` работает с `&[u8]` и не требует await. Один вызов — весь pipeline.

**3. `grayscale_native` и `grayscale_worker` — async.** Canvas API асинхронный: `createImageBitmap` и `convertToBlob` возвращают промисы.

**4. Общий контракт.** Все три функции принимают `&[u8]` PNG, возвращают `Vec<u8>` PNG. Вызывающий код не знает, какой движок используется.

**5. `check_grayscale` для тестов.** Не пытается сравнить выходные данные движков между собой — они используют разные алгоритмы сведения (ITU-R 601 vs 709). Проверяет только инвариант: R==G==B для всех пикселей.

**6. `generate_test_png` принимает угол.** Вращение проекций `u = dx·cos + dy·sin` и `v = −dx·sin + dy·cos` даёт визуально разную картинку для каждого угла. Демо использует `Math::random() * TAU` — каждый вызов уникален.

---

## 19.11.3. Добавление модуля в `lib.rs`

```rust
// … существующие модули …
pub mod fsa_api_quota;
pub mod fsa_api_observer;
pub mod fsa_api_wasm;   // ← НОВЫЙ

// …
```

---

## 19.11.4. Тесты: `tests/wasm_fsa_api_wasm.rs`

```rust
//! Тесты `src/fsa_api_wasm.rs`.
//!
//! **Что проверяем.**
//! * Генерацию тестового PNG (детерминированный паттерн).
//! * Разные углы дают разные изображения.
//! * `check_grayscale` — принимает grayscale, отвергает цвет.
//! * Roundtrip PNG через WASM-функцию `grayscale_wasm`.
//! * Что все три движка дают grayscale-результат.
//!
//! **Что не проверяем.**
//! * Попиксельное совпадение результатов разных движков — они
//!   используют разные цветовые пространства и алгоритмы сведения.
//!   Инвариант grayscale — то, что действительно должно совпадать.

use wasm_bindgen_test::*;

use hello_wasm02::fsa_api_wasm::{
    check_grayscale, generate_test_png, grayscale_native, grayscale_wasm, grayscale_worker,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// Генерация и базовые проверки
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_generate_test_png_returns_bytes() {
    let bytes = generate_test_png(16, 16, 0.0).expect("generate");
    assert!(!bytes.is_empty(), "PNG не может быть пустым");
    // PNG всегда начинается с сигнатуры \x89PNG\r\n\x1a\n.
    assert_eq!(&bytes[..4], b"\x89PNG", "PNG-сигнатура");
}

#[wasm_bindgen_test]
fn test_check_grayscale_rejects_color() {
    // Цветной паттерн (R по X, G по Y) — не grayscale.
    let color = generate_test_png(16, 16, 0.0).expect("generate");
    let result = check_grayscale(&color);
    assert!(result.is_err(), "цветной PNG не должен проходить check_grayscale");
}

#[wasm_bindgen_test]
fn test_generate_different_angles_produce_different_images() {
    let a = generate_test_png(64, 64, 0.0).expect("a");
    let b = generate_test_png(64, 64, std::f64::consts::FRAC_PI_4).expect("b");
    assert_ne!(a, b, "разные углы дают разные PNG");
}

// ---------------------------------------------------------------------------
// WASM движок (синхронный)
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_grayscale_wasm_roundtrip() {
    let input = generate_test_png(32, 32, 0.0).expect("generate");
    let output = grayscale_wasm(&input).expect("grayscale_wasm");
    assert!(!output.is_empty(), "результат не пустой");
    assert_eq!(&output[..4], b"\x89PNG", "PNG-сигнатура результата");
    check_grayscale(&output).expect("результат grayscale");
}

#[wasm_bindgen_test]
fn test_grayscale_wasm_rejects_garbage() {
    // Мусор на входе — Err, не паника.
    let garbage: Vec<u8> = vec![0, 1, 2, 3, 4, 5];
    let result = grayscale_wasm(&garbage);
    assert!(result.is_err(), "мусор должен дать Err");
}

// ---------------------------------------------------------------------------
// Native Canvas движок
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_grayscale_native_roundtrip() {
    let input = generate_test_png(32, 32, 0.0).expect("generate");
    let output = grayscale_native(&input).await.expect("grayscale_native");
    assert!(!output.is_empty(), "результат не пустой");
    check_grayscale(&output).expect("результат grayscale");
}

// ---------------------------------------------------------------------------
// Worker движок
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_grayscale_worker_roundtrip() {
    let input = generate_test_png(32, 32, 0.0).expect("generate");
    let output = grayscale_worker(&input).await.expect("grayscale_worker");
    assert!(!output.is_empty(), "результат не пустой");
    check_grayscale(&output).expect("результат grayscale");
}

// ---------------------------------------------------------------------------
// Все три движка дают grayscale
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_all_engines_produce_grayscale() {
    let input = generate_test_png(24, 24, 0.0).expect("generate");

    let a = grayscale_wasm(&input).expect("wasm");
    check_grayscale(&a).expect("wasm — grayscale");

    let b = grayscale_native(&input).await.expect("native");
    check_grayscale(&b).expect("native — grayscale");

    let c = grayscale_worker(&input).await.expect("worker");
    check_grayscale(&c).expect("worker — grayscale");
}
```

### Почему не сравниваем результаты попиксельно

`image` crate использует **ITU-R BT.601** для luma (0.299 R + 0.587 G + 0.114 B). Canvas `filter = 'grayscale(1)'` в Chrome реализован через SVG-фильтр `feColorMatrix`, который может использовать BT.709 (0.2126 R + 0.7152 G + 0.0722 B). Числа разные — результаты после округления отличаются на 1-2 единицы на канал. Попиксельное сравнение дало бы ложные падения.

Что действительно должно совпадать — **инвариант grayscale**: R==G==B после сведения. Это и проверяем.

---

## 19.11.5. Команды запуска

```bash
# 1. Сборка WASM (включает image crate)
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты в headless Chrome
wasm-pack test --headless --chrome --test wasm_fsa_api_wasm

# 4. Запуск демо
npm run start
```

`Cargo.toml` **уже содержит** `image = { version = "0.25", default-features = false, features = ["png", "jpeg"] }` — ничего добавлять не нужно.

---

## 19.11.6. Демо-пример: `src/app.rs`

```rust
//! Демо главы 19.11 — интеграция с WASM-библиотеками.
//!
//! **Идея.** Один и тот же PNG (1024×1024) обрабатывается тремя
//! движками: WASM-крейт `image`, нативный Canvas 2D `filter`,
//! Canvas 2D в Worker'е.
//!
//! **Wow-моменты:**
//! 1. **Автозапуск.** При загрузке страницы все три движка
//!    прогоняются автоматически — превью в строках сразу заполнены.
//! 2. **Каждый прогон даёт новую картинку.** При каждом запуске
//!    исходник генерируется заново — угол градиента случайный.
//! 3. **Каждая строка независима.** Прогон одного движка не стирает
//!    результаты других.
//! 4. Canvas `filter` бьёт WASM по времени — но WASM портируем.
//! 5. **Без перерисовки layout.** Меняются только пиксели картинки.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции — через
//! `format_str(format_args!(…))`.
//!
//! **Правило snake_case.** `create_element`, `set_attribute`,
//! `append_child`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, Element, FileSystemDirectoryHandle, HtmlElement};

use crate::fsa_api_opfs::{opfs_root, write_bytes as opfs_write_bytes};
use crate::fsa_api_wasm::{
    generate_test_png, grayscale_native, grayscale_wasm, grayscale_worker,
};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[19.11]";

fn console_log(msg: &str) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::log_1(&JsValue::from_str(&line));
}

fn console_err(msg: &str, e: &JsValue) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::error_2(&JsValue::from_str(&line), e);
}

// ===========================================================================
// Строковые хелперы
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

fn fmt_ms(v: f64) -> String {
    let s: String = if v < 1.0 {
        format_str(format_args!("{:.2} мс", v))
    } else {
        format_str(format_args!("{:.0} мс", v))
    };
    s
}

fn fmt_kb(bytes: usize) -> String {
    let kb = bytes as f64 / 1024.0;
    let s: String = if kb < 1024.0 {
        format_str(format_args!("{:.1} КБ", kb))
    } else {
        format_str(format_args!("{:.2} МБ", kb / 1024.0))
    };
    s
}

// ===========================================================================
// Состояние
// ===========================================================================

struct EngineResult {
    name: &'static str,
    ms: f64,
}

struct DemoState {
    status: Element,
    log_body: Element,
    // Source preview: живут всё время, меняется только src и текст.
    source_img: Element,
    source_label: Element,
    // Строки таблицы: время, размер, превью.
    row_wasm_ms: Element,
    row_wasm_size: Element,
    row_wasm_thumb: Element,
    row_native_ms: Element,
    row_native_size: Element,
    row_native_thumb: Element,
    row_worker_ms: Element,
    row_worker_size: Element,
    row_worker_thumb: Element,
    winner: Element,
    root: RefCell<Option<FileSystemDirectoryHandle>>,
    source_png: RefCell<Option<Vec<u8>>>,
    /// URL текущего превью исходника — чтобы отзывать перед заменой.
    source_url: RefCell<Option<String>>,
    results: RefCell<Vec<EngineResult>>,
    log_lines: RefCell<Vec<String>>,
}

const TEST_W: u32 = 1024;
const TEST_H: u32 = 1024;

// ===========================================================================
// CSS
// ===========================================================================

const CSS: &str = r#"
html, body {
  margin: 0 !important; padding: 0 !important;
  background: #0d1117 !important; color: #e6edf3 !important;
  min-height: 100vh;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  font-size: 14px; line-height: 1.55;
}
header { padding: 20px 32px 4px; }
h1 { font-size: 19px; margin: 0 0 4px; font-weight: 600; }
h1 code { color: #79c0ff; font-size: 0.95em; }
.subtitle { color: #8b949e; margin: 0; font-size: 13px; }

.status-bar {
  padding: 12px 32px 4px; font-size: 12px; color: #8b949e;
  font-family: ui-monospace, monospace;
}
.status-bar.ok   { color: #3fb950; }
.status-bar.warn { color: #d29922; }
.status-bar.err  { color: #f85149; }

.section {
  margin: 16px 32px; background: #161b22;
  border: 1px solid #30363d; border-radius: 8px;
  overflow: hidden;
}
.section h3 {
  margin: 0; padding: 10px 14px;
  background: #21262d; font-size: 13px; font-weight: 600;
  border-bottom: 1px solid #30363d;
  display: flex; align-items: baseline; gap: 8px;
  cursor: help;
}
.section h3 .num { color: #79c0ff; font-weight: 700; font-size: 14px; }
.section h3 .right {
  color: #8b949e; font-weight: 400; font-size: 11px;
  margin-left: auto;
}
.section h3 .right b { color: #79c0ff; font-weight: 600; cursor: help; }
.section .body { padding: 14px 16px; }

.intro {
  margin: 12px 32px 4px;
  padding: 12px 16px;
  background: rgba(31, 111, 235, .08);
  border: 1px solid rgba(31, 111, 235, .30);
  border-radius: 8px;
  color: #8b949e; font-size: 13px; line-height: 1.6;
}
.intro b { color: #e6edf3; font-weight: 500; }
.intro code { color: #79c0ff; font-family: ui-monospace, monospace; font-size: 12px; }

.controls { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 12px; }
.btn {
  background: #21262d; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 8px 14px; font-size: 13px; cursor: pointer;
  transition: background .15s, border-color .15s;
}
.btn:hover { background: #30363d; border-color: #8b949e; }
.btn:active { background: #1f6feb; border-color: #1f6feb; }
.btn.primary { background: #1f6feb; border-color: #1f6feb; }
.btn.primary:hover { background: #388bfd; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

/* Source preview: img и label живут постоянно, layout стабилен. */
.preview-row {
  display: flex; gap: 16px; flex-wrap: wrap;
  margin-top: 8px;
}
.preview-box {
  background: #0d1117; border: 1px solid #30363d; border-radius: 6px;
  padding: 8px; text-align: center;
}
.preview-box img {
  display: block;
  width: 200px; height: 200px;
  object-fit: cover;
  image-rendering: pixelated;
  background: #21262d;
}
.preview-box .label {
  color: #8b949e; font-size: 11px;
  font-family: ui-monospace, monospace; margin-top: 4px;
  min-width: 200px;
}

.compare {
  display: flex; flex-direction: column;
  font-family: ui-monospace, monospace; font-size: 13px;
}
.compare .head {
  display: grid;
  grid-template-columns: 160px 1fr 130px 100px 100px;
  gap: 10px; align-items: center;
  padding: 6px 0;
  border-bottom: 1px solid #30363d;
  color: #6e7681; font-size: 11px;
  text-transform: uppercase; letter-spacing: .5px;
}
.compare .row {
  display: grid;
  grid-template-columns: 160px 1fr 130px 100px 100px;
  gap: 10px; align-items: center;
  padding: 8px 0;
  border-bottom: 1px solid #21262d;
}
.compare .row:last-child { border-bottom: none; }
.compare .name { color: #e6edf3; font-weight: 500; }
.compare .bar-wrap {
  background: #0d1117; border-radius: 4px;
  height: 18px; overflow: hidden;
}
.compare .bar {
  height: 100%; width: 0;
  transition: width .35s cubic-bezier(.2,.7,.2,1);
  border-radius: 4px;
}
.compare .bar.wasm { background: linear-gradient(90deg, #f0883e, #f0b73e); }
.compare .bar.native { background: linear-gradient(90deg, #1f6feb, #58a6ff); }
.compare .bar.worker { background: linear-gradient(90deg, #3fb950, #58a6ff); }
.compare .ms { color: #e6edf3; font-variant-numeric: tabular-nums; }
.compare .size { color: #8b949e; font-variant-numeric: tabular-nums; }

.compare .preview-thumb {
  width: 80px; height: 80px;
  background: #0d1117;
  border: 1px solid #30363d; border-radius: 6px;
  display: flex; align-items: center; justify-content: center;
  overflow: hidden;
  color: #30363d; font-size: 18px;
  position: relative;
}
.compare .preview-thumb img {
  display: block; width: 100%; height: 100%; object-fit: cover;
  image-rendering: pixelated;
  background: #21262d;
}
.compare .preview-thumb::before {
  content: "…";
  color: #30363d;
}
.compare .preview-thumb.has-img::before { content: ""; }

.winner {
  margin-top: 12px; padding: 10px 14px;
  background: rgba(46, 160, 67, .08);
  border: 1px solid rgba(46, 160, 67, .30);
  border-radius: 6px;
  color: #3fb950; font-family: ui-monospace, monospace; font-size: 13px;
  min-height: 20px;
}
.winner b { color: #e6edf3; font-weight: 600; }

.log-body {
  padding: 12px 16px; max-height: 200px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; white-space: pre-wrap; }
.log-line.info    { color: #8b949e; }
.log-line.request { color: #6e7681; font-style: italic; }
.log-line.ok      { color: #3fb950; }
.log-line.err     { color: #f85149; }
.log-line.bench   { color: #79c0ff; }
"#;

// ===========================================================================
// Логирование
// ===========================================================================

fn log_line(state: &Rc<DemoState>, kind: &str, msg: &str) {
    {
        let mut v = state.log_lines.borrow_mut();
        v.push(msg.to_string());
        if v.len() > 200 {
            let excess = v.len() - 200;
            v.drain(0..excess);
        }
    }

    let window = match web_sys::window() {
        Some(w) => w,
        None => return,
    };
    let doc = match window.document() {
        Some(d) => d,
        None => return,
    };
    let line = match doc.create_element("div") {
        Ok(e) => e,
        Err(_) => return,
    };
    let cls: String = format_str(format_args!("log-line {kind}"));
    line.set_class_name(&cls);
    line.set_text_content(Some(msg));
    state.log_body.append_child(&line).ok();
    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let max: f64 = el.scroll_height() as f64;
        el.set_scroll_top(max);
    }
}

fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

// ===========================================================================
// Превью
// ===========================================================================

fn set_preview(img_el: &Element, png: &[u8]) {
    if let Some(old) = img_el.get_attribute("data-url") {
        if !old.is_empty() {
            web_sys::Url::revoke_object_url(&old).ok();
        }
    }

    let u8arr = js_sys::Uint8Array::from(png);
    let parts = js_sys::Array::new();
    parts.push(&u8arr.buffer());
    let blob = match web_sys::Blob::new_with_buffer_source_sequence(&parts) {
        Ok(b) => b,
        Err(_) => return,
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(_) => return,
    };

    img_el.set_attribute("src", &url).ok();
    img_el.set_attribute("data-url", &url).ok();

    if let Some(parent) = img_el.parent_element() {
        parent.class_list().add_1("has-img").ok();
    }
}

fn clear_preview(img_el: &Element) {
    if let Some(old) = img_el.get_attribute("data-url") {
        if !old.is_empty() {
            web_sys::Url::revoke_object_url(&old).ok();
        }
    }
    img_el.remove_attribute("src").ok();
    img_el.remove_attribute("data-url").ok();
    if let Some(parent) = img_el.parent_element() {
        parent.class_list().remove_1("has-img").ok();
    }
}

/// Обновить превью исходника **без пересоздания элементов**.
fn render_source_preview(state: &Rc<DemoState>, png: &[u8], angle_rad: f64) {
    let old = state.source_url.borrow_mut().take();
    if let Some(url) = old {
        web_sys::Url::revoke_object_url(&url).ok();
    }

    let u8arr = js_sys::Uint8Array::from(png);
    let parts = js_sys::Array::new();
    parts.push(&u8arr.buffer());
    let blob = match web_sys::Blob::new_with_buffer_source_sequence(&parts) {
        Ok(b) => b,
        Err(_) => return,
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(_) => return,
    };

    state.source_img.set_attribute("src", &url).ok();

    let angle_deg = angle_rad.to_degrees();
    let label_text: String = format_str(format_args!(
        "исходник · {}×{} · угол {:.0}° · {}",
        TEST_W,
        TEST_H,
        angle_deg,
        fmt_kb(png.len())
    ));
    state.source_label.set_text_content(Some(&label_text));

    *state.source_url.borrow_mut() = Some(url);
}

// ===========================================================================
// Рендер таблицы
// ===========================================================================

fn set_row(
    state: &Rc<DemoState>,
    name: &str,
    ms: f64,
    bytes_len: usize,
    png_output: &[u8],
) {
    let (ms_el, size_el, thumb_el) = match name {
        "wasm" => (&state.row_wasm_ms, &state.row_wasm_size, &state.row_wasm_thumb),
        "native" => (
            &state.row_native_ms,
            &state.row_native_size,
            &state.row_native_thumb,
        ),
        "worker" => (
            &state.row_worker_ms,
            &state.row_worker_size,
            &state.row_worker_thumb,
        ),
        _ => return,
    };

    let ms_s: String = fmt_ms(ms);
    let size_s: String = fmt_kb(bytes_len);
    ms_el.set_text_content(Some(&ms_s));
    size_el.set_text_content(Some(&size_s));

    set_preview(thumb_el, png_output);
}

fn render_bars(state: &Rc<DemoState>) {
    let results = state.results.borrow();
    if results.is_empty() {
        return;
    }
    let mut max_ms: f64 = 0.0;
    for r in results.iter() {
        if r.ms > max_ms {
            max_ms = r.ms;
        }
    }
    if max_ms <= 0.0 {
        return;
    }
    let max_ms = max_ms.max(1.0);

    let doc = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d,
        None => return,
    };

    for r in results.iter() {
        let selector: String = format_str(format_args!(
            ".compare .row[data-engine=\"{}\"] .bar",
            r.name
        ));
        if let Ok(Some(bar)) = doc.query_selector(&selector) {
            let frac = (r.ms / max_ms).clamp(0.02, 1.0);
            let style: String =
                format_str(format_args!("width: {:.1}%", frac * 100.0));
            bar.set_attribute("style", &style).ok();
        }
    }

    let mut best: Option<&EngineResult> = None;
    for r in results.iter() {
        match best {
            None => best = Some(r),
            Some(b) if r.ms < b.ms => best = Some(r),
            _ => {}
        }
    }
    if let Some(b) = best {
        let label = match b.name {
            "wasm" => "WASM (image crate)",
            "native" => "Native Canvas",
            "worker" => "Worker Canvas",
            _ => b.name,
        };
        let html: String = format_str(format_args!(
            "🏆 Победитель: <b>{}</b> за {}",
            label,
            fmt_ms(b.ms)
        ));
        state.winner.set_inner_html(&html);
    }
}

fn reset_comparison(state: &Rc<DemoState>) {
    state.results.borrow_mut().clear();
    state.row_wasm_ms.set_text_content(Some("—"));
    state.row_native_ms.set_text_content(Some("—"));
    state.row_worker_ms.set_text_content(Some("—"));
    state.row_wasm_size.set_text_content(Some("—"));
    state.row_native_size.set_text_content(Some("—"));
    state.row_worker_size.set_text_content(Some("—"));
    clear_preview(&state.row_wasm_thumb);
    clear_preview(&state.row_native_thumb);
    clear_preview(&state.row_worker_thumb);
    state.winner.set_inner_html("—");
}

// ===========================================================================
// Действия
// ===========================================================================

async fn regenerate_source(state: Rc<DemoState>) -> Result<(), JsValue> {
    let angle = js_sys::Math::random() * std::f64::consts::TAU;

    let png = match generate_test_png(TEST_W, TEST_H, angle) {
        Ok(p) => p,
        Err(e) => {
            console_err("generate_test_png failed", &e);
            log_line(&state, "err", "не удалось сгенерировать PNG");
            return Ok(());
        }
    };

    let angle_deg = angle.to_degrees();
    log_line(
        &state,
        "info",
        &format_str(format_args!(
            "новый исходник: угол {:.0}°, {}",
            angle_deg,
            fmt_kb(png.len())
        )),
    );

    render_source_preview(&state, &png, angle);

    if let Some(root) = state.root.borrow().clone() {
        let _ = opfs_write_bytes(&root, "source_19_11.png", &png).await;
    }

    *state.source_png.borrow_mut() = Some(png);
    Ok(())
}

async fn run_one_engine(state: Rc<DemoState>, engine: &str) -> Result<(), JsValue> {
    let input = match state.source_png.borrow().clone() {
        Some(p) => p,
        None => {
            log_line(&state, "err", "[запрос] исходник не готов");
            return Ok(());
        }
    };

    let (label, engine_key) = match engine {
        "wasm" => ("WASM (image crate)", "wasm"),
        "native" => ("Native Canvas", "native"),
        "worker" => ("Worker Canvas", "worker"),
        _ => return Ok(()),
    };

    log_line(
        &state,
        "request",
        &format_str(format_args!("[запрос] grayscale через {label}…")),
    );

    let t0: f64 = js_sys::Date::now();
    let output: Vec<u8> = match engine {
        "wasm" => match grayscale_wasm(&input) {
            Ok(o) => o,
            Err(e) => {
                console_err("grayscale_wasm failed", &e);
                log_line(&state, "err", "[результат] WASM: ошибка");
                return Ok(());
            }
        },
        "native" => match grayscale_native(&input).await {
            Ok(o) => o,
            Err(e) => {
                console_err("grayscale_native failed", &e);
                log_line(&state, "err", "[результат] Native: ошибка");
                return Ok(());
            }
        },
        "worker" => match grayscale_worker(&input).await {
            Ok(o) => o,
            Err(e) => {
                console_err("grayscale_worker failed", &e);
                log_line(&state, "err", "[результат] Worker: ошибка");
                return Ok(());
            }
        },
        _ => return Ok(()),
    };
    let t1: f64 = js_sys::Date::now();
    let ms: f64 = t1 - t0;

    set_row(&state, engine_key, ms, output.len(), &output);

    {
        let mut r = state.results.borrow_mut();
        r.retain(|x| x.name != engine_key);
        r.push(EngineResult { name: engine_key, ms });
    }

    if let Some(root) = state.root.borrow().clone() {
        let name = format_str(format_args!("gray_{engine_key}.png"));
        let _ = opfs_write_bytes(&root, &name, &output).await;
    }

    let line: String = format_str(format_args!(
        "[результат] {}: {} · {}",
        label,
        fmt_ms(ms),
        fmt_kb(output.len())
    ));
    log_line(&state, "ok", &line);
    render_bars(&state);
    Ok(())
}

async fn run_all(state: Rc<DemoState>) -> Result<(), JsValue> {
    regenerate_source(state.clone()).await?;
    reset_comparison(&state);
    log_line(
        &state,
        "info",
        "прогоняю все три движка на одном новом исходнике…",
    );

    run_one_engine(state.clone(), "wasm").await?;
    run_one_engine(state.clone(), "native").await?;
    run_one_engine(state.clone(), "worker").await?;

    log_line(&state, "ok", "все три движка отработали");
    Ok(())
}

async fn run_single(state: Rc<DemoState>, engine: &str) -> Result<(), JsValue> {
    regenerate_source(state.clone()).await?;
    run_one_engine(state, engine).await?;
    Ok(())
}

// ===========================================================================
// UI-хелперы
// ===========================================================================

fn make_button<F>(
    doc: &Document,
    parent: &Element,
    label: &str,
    tooltip: &str,
    primary: bool,
    mut handler: F,
) -> Result<Element, JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    let cls: &str = if primary { "btn primary" } else { "btn" };
    btn.set_class_name(cls);
    btn.set_text_content(Some(label));
    btn.set_attribute("title", tooltip)?;
    parent.append_child(&btn)?;

    let closure = Closure::<dyn FnMut()>::new(move || handler());
    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(btn)
}

fn make_section(
    doc: &Document,
    num: &str,
    title: &str,
    hint: &str,
    tooltip: &str,
) -> Result<(Element, Element), JsValue> {
    let section = doc.create_element("div")?;
    section.set_class_name("section");

    let h3 = doc.create_element("h3")?;
    let header_html: String = format_str(format_args!(
        "<span class=\"num\">{num}</span> <span>{title}</span>\
         <span class=\"right\">{hint}</span>"
    ));
    h3.set_inner_html(&header_html);
    h3.set_attribute("title", tooltip)?;
    section.append_child(&h3)?;

    let body = doc.create_element("div")?;
    body.set_class_name("body");
    section.append_child(&body)?;

    Ok((section, body))
}

fn make_compare_row(
    doc: &Document,
    engine_key: &str,
    label: &str,
    bar_class: &str,
) -> Result<(Element, Element, Element, Element), JsValue> {
    let row = doc.create_element("div")?;
    row.set_class_name("row");
    row.set_attribute("data-engine", engine_key)?;

    let name_el = doc.create_element("div")?;
    name_el.set_class_name("name");
    name_el.set_text_content(Some(label));
    row.append_child(&name_el)?;

    let bar_wrap = doc.create_element("div")?;
    bar_wrap.set_class_name("bar-wrap");
    let bar = doc.create_element("div")?;
    let bar_cls: String = format_str(format_args!("bar {bar_class}"));
    bar.set_class_name(&bar_cls);
    bar_wrap.append_child(&bar)?;
    row.append_child(&bar_wrap)?;

    let ms_el = doc.create_element("div")?;
    ms_el.set_class_name("ms");
    ms_el.set_text_content(Some("—"));
    row.append_child(&ms_el)?;

    let size_el = doc.create_element("div")?;
    size_el.set_class_name("size");
    size_el.set_text_content(Some("—"));
    row.append_child(&size_el)?;

    let thumb_wrap = doc.create_element("div")?;
    thumb_wrap.set_class_name("preview-thumb");
    let thumb_img = doc.create_element("img")?;
    thumb_img.set_attribute("alt", label)?;
    thumb_img.set_attribute(
        "title",
        "Результат работы движка — grayscale PNG. Появится после прогона.",
    )?;
    thumb_wrap.append_child(&thumb_img)?;
    row.append_child(&thumb_wrap)?;

    Ok((row, ms_el, size_el, thumb_img))
}

fn make_compare_head(doc: &Document) -> Result<Element, JsValue> {
    let head = doc.create_element("div")?;
    head.set_class_name("head");
    head.set_inner_html(
        "<div>движок</div>\
         <div>относительное время</div>\
         <div>время</div>\
         <div>размер</div>\
         <div>результат</div>",
    );
    Ok(head)
}

fn make_source_preview(
    doc: &Document,
) -> Result<(Element, Element, Element), JsValue> {
    let box_el = doc.create_element("div")?;
    box_el.set_class_name("preview-box");

    let img = doc.create_element("img")?;
    img.set_attribute("alt", "исходное изображение")?;
    img.set_attribute(
        "title",
        "Цветной паттерн со случайным углом градиента. Меняется при каждом прогоне.",
    )?;
    box_el.append_child(&img)?;

    let label = doc.create_element("div")?;
    label.set_class_name("label");
    label.set_text_content(Some("исходник · генерация…"));
    box_el.append_child(&label)?;

    Ok((box_el, img, label))
}

// ===========================================================================
// Точка входа
// ===========================================================================

pub fn run() -> Result<(), JsValue> {
    console_log("run: starting");

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = document
        .body()
        .ok_or_else(|| JsValue::from_str("no body"))?;

    body.set_inner_html("");

    if let Ok(links) = document.query_selector_all("link[rel='stylesheet']") {
        let n: u32 = links.length();
        for i in 0..n {
            if let Some(node) = links.get(i) {
                if let Some(parent) = node.parent_node() {
                    parent.remove_child(&node).ok();
                }
            }
        }
    }
    if let Some(root_el) = document.document_element() {
        root_el
            .set_attribute(
                "style",
                "background:#0d1117;color:#e6edf3;margin:0;padding:0;min-height:100vh;",
            )
            .ok();
    }

    let style = document.create_element("style")?;
    style.set_text_content(Some(CSS));
    document
        .head()
        .ok_or_else(|| JsValue::from_str("no head"))?
        .append_child(&style)
        .ok();

    // Header.
    let h1 = document.create_element("h1")?;
    h1.set_inner_html(
        "19.11 <code>WASM-библиотеки</code>: один интерфейс — три движка",
    );
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "Rust-крейт image · Canvas 2D filter · Worker · каждая строка — своя уникальная картинка",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    // Intro.
    let intro = document.create_element("div")?;
    intro.set_class_name("intro");
    intro.set_inner_html(
        "<b>Что демонстрируется.</b> Один и тот же PNG (1024×1024) \
         обрабатывается тремя движками: Rust-крейт <code>image</code> в WASM, \
         нативный Canvas 2D через <code>filter = 'grayscale(1)'</code>, \
         тот же Canvas в Worker'е.<br><br>\
         <b>Автозапуск.</b> При загрузке страницы все три движка уже \
         отработали — превью в строках заполнены, времена замерены.<br><br>\
         <b>Wow-эффект.</b> При каждом прогоне исходник генерируется заново — \
         градиент поворачивается на случайный угол. Нажмите «Только WASM» \
         несколько раз подряд: строка WASM будет показывать <b>разные</b> \
         grayscale-картинки. То же для Native и Worker — каждая строка живёт \
         своей жизнью, прогон одного движка не стирает результаты других.<br><br>\
         <b>Прогнать все три</b> — честное сравнение: три движка на <b>одном</b> \
         новом источнике одновременно.<br><br>\
         <b>Что смотреть.</b> Canvas обычно быстрее WASM, но WASM <b>портируем</b> \
         — тот же Rust-код работает в браузере и в CLI.",
    );
    body.append_child(&intro)?;

    // Status bar.
    let status = document.create_element("div")?;
    status.set_class_name("status-bar");
    status.set_text_content(Some("инициализация…"));
    body.append_child(&status)?;

    // Section ❶.
    let (sec1, body1) = make_section(
        &document,
        "❶",
        "Исходное изображение",
        "перерисовывается при каждом прогоне",
        "Цветной паттерн 1024×1024 с градиентом под случайным углом. \
         Каждый прогон — новый угол, новая картинка.",
    )?;
    body.append_child(&sec1)?;

    let preview_row = document.create_element("div")?;
    preview_row.set_class_name("preview-row");
    body1.append_child(&preview_row)?;

    let (source_box, source_img, source_label) = make_source_preview(&document)?;
    preview_row.append_child(&source_box)?;

    // Section ❷.
    let (sec2, body2) = make_section(
        &document,
        "❷",
        "Сравнение движков",
        "время · размер · превью результата",
        "Три реализации одной задачи — grayscale PNG. Рядом с временем — \
         миниатюра результата. Прогон одного движка не стирает результаты \
         других: каждая строка обновляется независимо.",
    )?;
    body.append_child(&sec2)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("controls");
    body2.append_child(&controls)?;

    let compare = document.create_element("div")?;
    compare.set_class_name("compare");
    body2.append_child(&compare)?;

    let head = make_compare_head(&document)?;
    compare.append_child(&head)?;

    let (row_wasm, row_wasm_ms, row_wasm_size, row_wasm_thumb) =
        make_compare_row(&document, "wasm", "WASM (image crate)", "wasm")?;
    let (row_native, row_native_ms, row_native_size, row_native_thumb) =
        make_compare_row(&document, "native", "Native Canvas", "native")?;
    let (row_worker, row_worker_ms, row_worker_size, row_worker_thumb) =
        make_compare_row(&document, "worker", "Worker Canvas", "worker")?;
    compare.append_child(&row_wasm)?;
    compare.append_child(&row_native)?;
    compare.append_child(&row_worker)?;

    let winner = document.create_element("div")?;
    winner.set_class_name("winner");
    winner.set_inner_html("—");
    body2.append_child(&winner)?;

    // Section ❸.
    let (sec3, body3) = make_section(
        &document,
        "❸",
        "Журнал",
        "",
        "Запросы (что вы нажали) и результаты (что получилось). \
         Результаты сохраняются в OPFS под именами gray_<engine>.png.",
    )?;
    body.append_child(&sec3)?;
    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    body3.set_class_name("");
    body3.set_attribute("style", "padding:0")?;
    body3.append_child(&log_body)?;

    // State.
    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        log_body: log_body.clone(),
        source_img,
        source_label,
        row_wasm_ms,
        row_wasm_size,
        row_wasm_thumb,
        row_native_ms,
        row_native_size,
        row_native_thumb,
        row_worker_ms,
        row_worker_size,
        row_worker_thumb,
        winner,
        root: RefCell::new(None),
        source_png: RefCell::new(None),
        source_url: RefCell::new(None),
        results: RefCell::new(Vec::new()),
        log_lines: RefCell::new(Vec::new()),
    });

    // Кнопки.
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Прогнать все три",
            "Генерирует ОДИН новый исходник и последовательно прогоняет WASM, \
             Native и Worker на нём. Все три строки обновляются одновременно — \
             честное сравнение. Результаты других сессий стираются.",
            true,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = run_all(s).await {
                        console_err("run_all failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Только WASM",
            "Генерирует новый исходник и прогоняет ТОЛЬКО WASM. \
             Другие строки не трогает — их результаты остаются как были. \
             Многократный прогон даёт визуально разные grayscale-картинки в строке WASM.",
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = run_single(s, "wasm").await {
                        console_err("run_single wasm failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Только Native",
            "Генерирует новый исходник и прогоняет ТОЛЬКО Native Canvas 2D. \
             Другие строки не трогает.",
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = run_single(s, "native").await {
                        console_err("run_single native failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Только Worker",
            "Генерирует новый исходник и прогоняет ТОЛЬКО Worker Canvas 2D. \
             Другие строки не трогает.",
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = run_single(s, "worker").await {
                        console_err("run_single worker failed", &e);
                    }
                });
            },
        )?;
    }

    // Инициализация.
    {
        let s = state.clone();
        spawn_local(async move {
            console_log("инициализация…");
            set_status(&s, "ok", "готово");

            if let Ok(root) = opfs_root().await {
                *s.root.borrow_mut() = Some(root);
            } else {
                log_line(
                    &s,
                    "info",
                    "OPFS недоступен — результаты не сохраняются на диск",
                );
            }

            // Автозапуск: сразу генерируем исходник и прогоняем все три
            // движка — чтобы при загрузке страницы превью во всех
            // строках уже были заполнены, а не пустовали.
            log_line(
                &s,
                "info",
                "первый прогон: генерирую исходник и запускаю все три движка…",
            );
            if let Err(e) = run_all(s.clone()).await {
                console_err("auto run_all failed", &e);
            }

            log_line(
                &s,
                "info",
                "готово. Нажмите «Прогнать все три» или любой из движков.",
            );
        });
    }

    console_log("run: finished OK");
    Ok(())
}
```

---

## 19.11.7. Что показывает демо

**При загрузке страницы.** Сразу работает автозапуск: генерируется исходник, прогоняются три движка, заполняются три строки таблицы. Пользователь видит полностью работающее демо, а не пустые «—».

**Секция ❶.** Цветной градиент со случайным углом. Подпись: «исходник · 1024×1024 · угол 137° · 1.2 МБ». Каждый следующий прогон — новый угол, новая картинка.

**Секция ❷.** Таблица с пятью колонками: имя движка, относительное время, абсолютное время, размер результата, превью grayscale-миниатюры. Рядом с каждой строкой — своя уникальная grayscale-картинка.

**Секция ❸.** Журнал операций с префиксами `[запрос]` и `[результат]`.

### Wow-эффекты

**1. Автозапуск.** Открыли страницу — сразу видите работающее демо. Никаких пустых превью.

**2. Каждый прогон уникален.** Угол градиента — случайный. Нажимаете «Только WASM» три раза подряд — строка WASM показывает три разных grayscale-результата от трёх источников.

**3. Строки независимы.** Прогон одного движка не стирает другие. WASM, Native и Worker накапливают свою уникальную историю.

**4. Один интерфейс — три движка.** Все три функции принимают `&[u8]`, возвращают `Vec<u8>`. Вызывающий код не знает деталей.

**5. Честное сравнение.** «Прогнать все три» использует **один** источник для всех трёх движков. Три grayscale-превью выглядят одинаково — доказательство, что задачи решены корректно.

**6. Без перерисовки layout.** `img` и `label` созданы один раз, меняются только `src` и `textContent`. Страница не дёргается.

---

## 19.11.8. Правила проекта, выученные в этой главе

**1. Байты — универсальный интерфейс.** `&[u8]` на входе, `Vec<u8>` на выходе — один контракт для всех движков.

**2. WASM не всегда быстрее.** Браузерный Canvas оптимизирован лучше универсального Rust-крейта. Честное сравнение показывает это.

**3. WASM портируем.** Тот же Rust-код работает в браузере, на сервере, в CLI. Это главная ценность интеграции.

**4. Worker не ускоряет одну операцию.** Тот же код в Worker'е занимает столько же миллисекунд. Его ценность — не блокировать main thread.

**5. `File.arrayBuffer()` — быстрее `FileReader`.** Один промис, одна копия.

**6. `Blob::new_with_buffer_source_sequence`** для `Uint8Array.buffer` — стандартный путь создания Blob из байтов.

**7. `createImageBitmap`** — самое быстрое декодирование PNG в браузере. Не `<img>`, не `Image()`, а именно `createImageBitmap`.

**8. `OffscreenCanvas.convertToBlob()`** — экспорт изображения. Возвращает промис с Blob.

**9. Проверять инварианты, а не совпадения.** Разные движки используют разные цветовые пространства (BT.601 vs BT.709), поэтому попиксельное сравнение даёт ложные падения. Проверяем инвариант `R==G==B`.

**10. Угол в параметре генерации.** Не хардкодьте картинку — параметризуйте. `Math::random() * TAU` даёт визуально уникальный результат на каждом прогоне.

**11. Элементы создаются один раз.** Не пересоздавайте `<img>` и `<label>` — меняйте атрибуты. Layout не пересчитывается, страница не дёргается.

**12. Строки таблицы независимы.** Прогон одного движка не сбрасывает результаты других. Правильный дизайн — каждый движок сам управляет своей строкой.

**13. Автозапуск.** Демо должно быть заполнено при загрузке страницы, а не ждать первого клика.

---

## 19.11.9. Чек-лист

- [ ] Все движки принимают `&[u8]` PNG, возвращают `Vec<u8>` PNG.
- [ ] `File.arrayBuffer()` для чтения файла — не `FileReader`.
- [ ] `createImageBitmap` для декодирования PNG в Canvas.
- [ ] `OffscreenCanvas.convertToBlob()` для экспорта.
- [ ] `ctx.set_filter("grayscale(1)")` для нативного grayscale.
- [ ] Worker с inline Blob — простой способ запустить отдельный поток.
- [ ] `Closure::once` для onmessage Worker'а — резолвит Promise при первом ответе.
- [ ] Проверка `check_grayscale` — инвариант `R==G==B`, не попиксельное сравнение.
- [ ] `generate_test_png(width, height, angle)` — параметризован углом.
- [ ] Разные углы дают разные PNG (тест `test_generate_different_angles_produce_different_images`).
- [ ] Сравнение времен в UI, а не только в логах.
- [ ] Результаты сохраняются в OPFS — можно проверить в DevTools.
- [ ] `format_str(format_args!(…))` для всех строковых выражений в tail-позиции.
- [ ] Автозапуск: при загрузке страницы все три движка уже отработали.
- [ ] Строки таблицы независимы: прогон одного движка не сбрасывает другие.
- [ ] Превью и подписи — постоянные элементы, меняются только атрибуты.
- [ ] CSS: фиксированные `width`/`height` на `<img>` и `min-width` на `.label` — layout не дёргается.

