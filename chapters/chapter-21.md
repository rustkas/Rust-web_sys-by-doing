# Глава 21. Canvas и изображения

## 21.0.1. О чём эта глава

Canvas — это HTML-элемент, внутри которого можно рисовать пиксели. Не «верстать блоками», не «расставлять DOM-узлы», а именно **рисовать**: ставить точки, проводить линии, заливать области, читать и записывать байты. Это единственный способ в браузере добраться до физического уровня изображения — того самого, где живут цвет, координата и бит.

Для Rust + WebAssembly это особенно интересно. Пока JS-цикл проходит по пиксельному буферу 800×600 за десять миллисекунд, тот же цикл на Rust проходит за одну. Разница становится заметна не в синтетических бенчмарках, а на живом пользователе, который двигает слайдер и видит результат мгновенно. Canvas — это та ниша, где WASM окупается сам: численная обработка пикселей, фильтры, эффекты, композиция.

Глава состоит из семи подглав. Каждая — отдельная тема со своим демо, своими тестами и своим кодом. Прогрессия выстроена по принципу «от внешнего к внутреннему»: сначала учимся создавать изображение на экране, потом получать пиксели, потом обрабатывать их и, наконец, собирать всё вместе.

## 21.0.2. Прогрессия подглав

**21.1. HtmlCanvasElement: размеры и HiDPI.** Первый вопрос, который возникает у любого, кто впервые ставит `<canvas>` — «почему картинка размыта на Retina». Ответ простой: буфер рисования и элемент на странице — **разные вещи**. `canvas.width` — это разрешение буфера, `style.width` — размер на экране, `devicePixelRatio` — коэффициент между ними. Без формулы `canvas.width = cssW × DPR` и `ctx.scale(dpr, dpr)` всё выглядит мыльно. Демо показывает два холста рядом — наивный и правильный — чтобы разница была видна глазами, а не на словах.

**21.2. Drawing API: фигуры, пути, текст.** Что вообще можно нарисовать, кроме уже готовой картинки: прямоугольники, линии, дуги, кривые, текст, градиенты. Это база, без которой остальные шесть подглав повисают в воздухе. Все операции — через один контекст `CanvasRenderingContext2d`, все координаты — CSS-пиксели. Демо — «инспектор кисти»: панель со слайдерами толщины, цвета, закругления; картинка перерисовывается мгновенно, пока двигаешь ползунок.

**21.3. Состояние, трансформации, композиция.** Что происходит, когда рисунков много и они накладываются друг на друга. Стек `save`/`restore`, матрица трансформации `translate`/`rotate`/`scale`, режимы смешивания `globalCompositeOperation`, отсечение `clip`. Это половина работы с canvas: не «что нарисовать», а «как наложить». Демо — «сцена композиции»: несколько слоёв, слайдер прозрачности верхнего, выпадашка режима смешивания. Рядом — переключатель «сохранять стек / испортить без save», который наглядно показывает, зачем нужны `save` и `restore`.

**21.4. ImageData: пиксельный буфер и его границы.** Точка входа в Rust-обработку. `ImageData` — это `width × height × 4` байт RGBA, доступных напрямую. Здесь учат читать по индексу `(y*w + x)*4 + channel`, писать обратно, понимать формат `Uint8ClampedArray`, знать, что `putImageData` игнорирует трансформации и композицию — это «сырой» доступ к холсту. Демо — «пипетка»: клик по рисунку показывает RGBA в точке, отдельные кнопки обнуляют и инвертируют каналы.

**21.5. Загрузка изображений: `<img>`, `ImageBitmap`, EXIF.** Как получить пиксели из файла. Три способа — `<img>`, `createImageBitmap`, `OffscreenCanvas` — и три разных сценария. Отдельная тема — EXIF-ориентация: фото с телефона приходит повёрнутым, потому что ни `<img>`, ни `createImageBitmap` не применяют EXIF автоматически. `createImageBitmap(blob, { imageOrientation: "from-image" })` решает эту задачу одной строкой. Демо — «инспектор загрузки»: слева `<img>`, справа `ImageBitmap`, в центре с `imageOrientation`. Разные фото показывают разницу. Тайминг загрузки — на глазах.

**21.6. Фильтры на Rust: чистые функции, SIMD, Worker.** Собственно численная обработка. Разделение кода на **чистые функции** (`&[u8] -> Vec<u8>`) и **web-обвязку** — принципиальное: чистые функции тестируются на хосте за миллисекунды, без браузера. Классические фильтры — grayscale, invert, box blur, Sobel — и их ускорение. SIMD 128 через `+simd128` в `rustflags`, ускорение в разы. Web Worker с transferable `ArrayBuffer` — обработка без блокировки main thread. Демо — «бенчмарк-стол»: одна картинка, четыре фильтра, три реализации (чистый Rust, SIMD, Worker), таблица сравнительных цифр.

**21.7. Практический проект: Image Processor.** Сборка. Всё, что было изучено, сходится в один работающий редактор: загрузка файла, рисование на OffscreenCanvas, чтение пикселей, Rust-фильтр, отображение. Пять фильтров, тайминг на каждом, живое обновление. Отдельная изюминка — работа с несовместимостью сигнатур `getImageData` / `putImageData` между `web_sys_unstable_apis` и стабильной сборкой через cfg-гейт. Демо — то самое, которое читатель уже видел в предыдущей итерации главы 21.

## 21.0.3. Как читать эту главу

Каждая подглава **автономна**. Студент открывает `app.rs` — видит ровно то, что относится к текущей теме. При переходе к следующей подглаве `app.rs` заменяется целиком. Это правило проекта: демо не накапливается, предыдущие секции не остаются в файле. Фокус на одной теме — меньше шума, яснее связь между кодом и концепцией.

Модули (`canvas_dpi.rs`, `canvas_drawing.rs`, `canvas_image_processor.rs`, …) существуют **отдельно** для тестов и переиспользования. Демо их не импортирует — оно содержит свою копию нужных функций. Дублирование здесь осознанное: студент открывает `app.rs` и видит цельную историю без отсылок к другим главам.

Порядок подглав — рекомендация, не правило. Кто хочет сразу понять, «зачем Rust в canvas», может начать с **21.4** и **21.6**, а потом вернуться к **21.1–21.3**. Кто строит визуальный редактор — идёт последовательно от **21.1** к **21.7**.

## 21.0.4. Что вы получите

К концу главы студент:

- знает, почему `canvas.width ≠ style.width` и как сделать резкую картинку на Retina;
- умеет рисовать фигуры, текст, дуги, градиенты, применять трансформации и композицию;
- понимает формат `ImageData` и может читать/писать пиксели напрямую;
- умеет загружать изображения через `createImageBitmap` с учётом EXIF;
- пишет фильтры на Rust, тестирует их как чистые функции, ускоряет через SIMD и Worker;
- собирает работающий image-редактор из всех этих частей;
- знает, как разрешать несовместимости web-sys через cfg-гейт, не ломая IDE.

## 21.0.5. Технические требования

**Зависимости.** Все нужные `web-sys` features уже подключены в `Cargo.toml` проекта (см. `[dependencies.web-sys]`). Дополнительных крейтов не требуется.

**Совместимость браузеров.** `canvas 2d` — везде. `OffscreenCanvas` — Chrome 69+, Firefox 105+, Safari 16.4+. `createImageBitmap` — Chrome 50+, Firefox 42+, Safari 15+. `+simd128` — Chrome 91+, Firefox 89+, Safari 16.4+ (для Safari <16.4 нужна отдельная сборка без SIMD).

**`.cargo/config.toml`.** Для глав 21.6 и 21.7 рекомендуется включить SIMD:

```toml
[target.wasm32-unknown-unknown]
rustflags = ["-C", "target-feature=+simd128"]
```

Про флаг `--cfg=web_sys_unstable_apis` — отдельный разговор в подглаве 21.7. Включать его не обязательно; код готов к обоим режимам.

## 21.0.6. Ссылки для углублённого изучения

- MDN: [`<canvas>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/canvas)
- MDN: [`CanvasRenderingContext2D`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D)
- MDN: [`ImageData`](https://developer.mozilla.org/en-US/docs/Web/API/ImageData)
- MDN: [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- MDN: [`createImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/createImageBitmap)
- MDN: [`devicePixelRatio`](https://developer.mozilla.org/en-US/docs/Web/API/Window/devicePixelRatio)
- WHATWG HTML Spec: [The canvas element](https://html.spec.whatwg.org/multipage/canvas.html)
- web.dev: [Learn HTML canvas](https://web.dev/learn/canvas)
- web.dev: [OffscreenCanvas](https://web.dev/articles/offscreen-canvas)
- Wikipedia: [Sobel operator](https://en.wikipedia.org/wiki/Sobel_operator)
- Wikipedia: [Rec. 709 luma](https://en.wikipedia.org/wiki/Rec._709)

## 21.0.7. Итог

Canvas — это мост между двумя мирами: декларативным DOM и императивной пиксельной графикой. Семь подглав ведут через этот мост от «нарисуй прямоугольник» до «собери редактор с фильтрами на Rust». Каждая — самостоятельная история; вместе — полная картина того, как браузер работает с изображениями и как Rust делает эту работу быстрой.

Дальше — по подглавам.

# Глава 21.7 — Полные исходные файлы

## Структура

```
src/
  canvas_image_processor.rs         ← модуль (для тестов и переиспользования)
  app.rs                            ← демо главы, автономный
tests/
  canvas_pure.rs                    ← host-тесты чистых функций
  wasm_canvas_image_processor.rs    ← wasm-тесты web-обвязки
```

---

## `src/canvas_image_processor.rs`

```rust
//! 21.7. Практический проект: Image Processor.
//!
//! Полный round-trip пикселей:
//!   `<img>` → canvas → `ImageData` → `Vec<u8>`
//!   → **Rust-обработка** → `Vec<u8>` → `ImageData` → canvas → экран.
//!
//! Модуль специально разделён на две части:
//! - **чистые функции** (`to_grayscale`, `invert`, `box_blur_3x3`,
//!   `sobel_edge`, `make_sample_pixels`) работают с `&[u8]` / `Vec<u8>`,
//!   не знают про DOM и тестируются на хосте;
//! - **web-обвязка** (`load_image_from_blob`, `image_to_image_data`,
//!   `image_data_to_vec`, `vec_to_image_data`, `draw_image_data_to_canvas`) —
//!   тонкий слой над Canvas API.
//!
//! ## Про сигнатуры `getImageData` / `putImageData`
//!
//! В `web-sys 0.3.105` эти методы имеют две сигнатуры, переключаемые
//! cfg-флагом `web_sys_unstable_apis`:
//!
//! | Режим | `getImageData` | `putImageData` |
//! |---|---|---|
//! | stable (по умолчанию) | `f64, f64, f64, f64` | `_, f64, f64` |
//! | unstable (`--cfg=web_sys_unstable_apis`) | `i32, i32, i32, i32` | `_, i32, i32` |
//!
//! Различие инкапсулировано в двух хелперах `read_image_data` /
//! `write_image_data`. Остальной код их вызывает одинаково и не знает
//! про cfg.

use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    Blob, CanvasRenderingContext2d, HtmlCanvasElement, HtmlImageElement,
    ImageData, OffscreenCanvas, OffscreenCanvasRenderingContext2d, Url,
};

// =============================================================================
// Фильтры и чистые функции обработки пикселей
// =============================================================================

/// Палитра доступных фильтров.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Filter {
    Identity,
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    pub fn label(self) -> &'static str {
        match self {
            Filter::Identity => "Оригинал",
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    /// Применяет фильтр к RGBA-буферу. 4 байта на пиксель: R, G, B, A.
    pub fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Identity => rgba.to_vec(),
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

/// Luminance по Rec. 709: Y = 0.2126·R + 0.7152·G + 0.0722·B.
/// Коэффициенты умножены на 10000 для целочисленной арифметики.
/// Альфа сохраняется без изменений.
pub fn to_grayscale(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        let r = px[0] as u32;
        let g = px[1] as u32;
        let b = px[2] as u32;
        let y = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
        out.push(y);
        out.push(y);
        out.push(y);
        out.push(px[3]);
    }
    out
}

/// Инверсия RGB. Альфа не трогается — иначе прозрачные области
/// станут непрозрачными и картинка «посереет».
pub fn invert(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        out.push(255 - px[0]);
        out.push(255 - px[1]);
        out.push(255 - px[2]);
        out.push(px[3]);
    }
    out
}

/// Box blur 3×3 — усреднение 9 соседей. На краях окно обрезается
/// (clamp к границам изображения), а не заворачивается.
pub fn box_blur_3x3(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];
    for y in 0..h {
        for x in 0..w {
            let (mut r, mut g, mut b, mut a, mut n) = (0u32, 0u32, 0u32, 0u32, 0u32);
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let i = (ny * w + nx) * 4;
                    r += rgba[i] as u32;
                    g += rgba[i + 1] as u32;
                    b += rgba[i + 2] as u32;
                    a += rgba[i + 3] as u32;
                    n += 1;
                }
            }
            let i = (y * w + x) * 4;
            out[i] = (r / n) as u8;
            out[i + 1] = (g / n) as u8;
            out[i + 2] = (b / n) as u8;
            out[i + 3] = (a / n) as u8;
        }
    }
    out
}

/// Sobel edge detection по яркости (не по отдельным каналам — так
/// получается чище). Возвращает чёрно-белую карту границ.
pub fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];

    // Считаем яркость один раз — не гоняем градиенты по трём каналам.
    let mut lum = vec![0u8; w * h];
    for i in 0..w * h {
        let p = i * 4;
        let r = rgba[p] as u32;
        let g = rgba[p + 1] as u32;
        let b = rgba[p + 2] as u32;
        lum[i] = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
    }

    // Классические ядра Sobel.
    const GX: [i32; 9] = [-1, 0, 1, -2, 0, 2, -1, 0, 1];
    const GY: [i32; 9] = [-1, -2, -1, 0, 0, 0, 1, 2, 1];

    for y in 0..h {
        for x in 0..w {
            let (mut sx, mut sy) = (0i32, 0i32);
            let mut k = 0;
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let l = lum[ny * w + nx] as i32;
                    sx += GX[k] * l;
                    sy += GY[k] * l;
                    k += 1;
                }
            }
            let mag = ((sx * sx + sy * sy) as f64).sqrt().min(255.0) as u8;
            let i = (y * w + x) * 4;
            out[i] = mag;
            out[i + 1] = mag;
            out[i + 2] = mag;
            out[i + 3] = 255;
        }
    }
    out
}

/// Встроенное «полотно» — цветные круги на градиенте. Используется,
/// когда пользователь ещё не загрузил свой файл.
pub fn make_sample_pixels(w: u32, h: u32) -> Vec<u8> {
    let mut out = Vec::with_capacity(w as usize * h as usize * 4);
    let cx1 = w as f32 * 0.30;
    let cy1 = h as f32 * 0.35;
    let cx2 = w as f32 * 0.72;
    let cy2 = h as f32 * 0.55;
    let cx3 = w as f32 * 0.50;
    let cy3 = h as f32 * 0.80;
    let r1 = (w as f32 * 0.12).powi(2);
    let r2 = (w as f32 * 0.14).powi(2);
    let r3 = (w as f32 * 0.10).powi(2);

    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / w as f32;
            let fy = y as f32 / h as f32;
            let mut r = (40.0 + fx * 120.0) as u8;
            let mut g = (60.0 + fy * 120.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            let d1 = (x as f32 - cx1).powi(2) + (y as f32 - cy1).powi(2);
            let d2 = (x as f32 - cx2).powi(2) + (y as f32 - cy2).powi(2);
            let d3 = (x as f32 - cx3).powi(2) + (y as f32 - cy3).powi(2);
            if d1 < r1 { r = 240; g = 80; b = 60; }
            if d2 < r2 { r = 60; g = 200; b = 140; }
            if d3 < r3 { r = 100; g = 100; b = 240; }

            out.push(r);
            out.push(g);
            out.push(b);
            out.push(255);
        }
    }
    out
}

// =============================================================================
// Хелперы для сигнатур canvas: getImageData / putImageData
// =============================================================================

#[cfg(web_sys_unstable_apis)]
fn read_image_data(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0, 0, w as i32, h as i32)
}

#[cfg(not(web_sys_unstable_apis))]
fn read_image_data(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0.0, 0.0, w as f64, h as f64)
}

#[cfg(web_sys_unstable_apis)]
fn write_image_data(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
fn write_image_data(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

// =============================================================================
// Web-обвязка: <img> ↔ ImageData ↔ Vec<u8>
// =============================================================================

/// Загружает Blob как `<img>` через Object URL и ждёт `decode()`.
/// Object URL отзывается **до** возврата — картинка уже в памяти браузера.
pub async fn load_image_from_blob(blob: &Blob) -> Result<HtmlImageElement, JsValue> {
    let document = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let img: HtmlImageElement = document
        .create_element("img")?
        .dyn_into::<HtmlImageElement>()?;

    let url = Url::create_object_url_with_blob(blob)?;
    img.set_src(&url);

    let result = wasm_bindgen_futures::JsFuture::from(img.decode()).await;
    let _ = Url::revoke_object_url(&url);
    result?;
    Ok(img)
}

/// Рисует `<img>` на OffscreenCanvas и возвращает ImageData + размеры.
/// Если сторона превышает `max_dim` — картинка масштабируется.
pub fn image_to_image_data(
    img: &HtmlImageElement,
    max_dim: u32,
) -> Result<(ImageData, u32, u32), JsValue> {
    let iw = img.natural_width();
    let ih = img.natural_height();
    if iw == 0 || ih == 0 {
        return Err(JsValue::from_str("image has zero size"));
    }
    let (w, h) = if iw.max(ih) > max_dim {
        let scale = max_dim as f64 / iw.max(ih) as f64;
        (
            (iw as f64 * scale).round().max(1.0) as u32,
            (ih as f64 * scale).round().max(1.0) as u32,
        )
    } else {
        (iw, ih)
    };

    let canvas = OffscreenCanvas::new(w, h)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;

    // drawImage — f64 независимо от cfg.
    ctx.draw_image_with_html_image_element_and_dw_and_dh(
        img, 0.0, 0.0, w as f64, h as f64,
    )?;

    // getImageData — сигнатура зависит от cfg, скрыта в хелпере.
    let data = read_image_data(&ctx, w, h)?;
    Ok((data, w, h))
}

/// `ImageData::data()` в web-sys 0.3.105 возвращает `Clamped<Vec<u8>>` —
/// tuple struct с публичным полем `.0`. Просто забираем внутренний `Vec<u8>`.
pub fn image_data_to_vec(data: &ImageData) -> Vec<u8> {
    data.data().0
}

/// Обратный путь: `&[u8]` → `Clamped<&[u8]>` → `ImageData`.
/// `Clamped` передаётся по значению, без `&`.
pub fn vec_to_image_data(pixels: &[u8], w: u32, h: u32) -> Result<ImageData, JsValue> {
    let expected = (w as usize) * (h as usize) * 4;
    if pixels.len() != expected {
        return Err(JsValue::from_str(&format!(
            "pixel buffer size {} != {} for {}x{}",
            pixels.len(), expected, w, h,
        )));
    }
    ImageData::new_with_u8_clamped_array_and_sh(Clamped(pixels), w, h)
}

/// Заменяет содержимое `HtmlCanvasElement` на `ImageData`.
/// Подгоняет размер холста под данные — иначе обрезка или растяжка.
pub fn draw_image_data_to_canvas(
    canvas: &HtmlCanvasElement,
    data: &ImageData,
) -> Result<(), JsValue> {
    canvas.set_width(data.width());
    canvas.set_height(data.height());
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx"))?;
    let ctx: CanvasRenderingContext2d = ctx_js.dyn_into()?;
    // putImageData — сигнатура зависит от cfg, скрыта в хелпере.
    write_image_data(&ctx, data)
}
```

---

## `tests/canvas_pure.rs`

```rust
//! Host-тесты чистой логики обработки пикселей.
//! Запускаются `cargo test` — без браузера, за миллисекунды.

use hello_wasm02::canvas_image_processor::{
    box_blur_3x3, invert, make_sample_pixels, sobel_edge, to_grayscale, Filter,
};

/// Создать однотонный RGBA-буфер `w × h`.
fn solid(w: u32, h: u32, r: u8, g: u8, b: u8, a: u8) -> Vec<u8> {
    let mut v = Vec::with_capacity((w * h * 4) as usize);
    for _ in 0..w * h {
        v.extend_from_slice(&[r, g, b, a]);
    }
    v
}

#[test]
fn grayscale_of_pure_red_is_expected_luminance() {
    // 0.2126 * 255 ≈ 54
    let px = vec![255, 0, 0, 255];
    let out = to_grayscale(&px);
    assert_eq!(out.len(), 4);
    assert_eq!(out[0], out[1]);
    assert_eq!(out[1], out[2]);
    assert_eq!(out[3], 255, "alpha unchanged");
    assert!(out[0] >= 53 && out[0] <= 55, "got {}", out[0]);
}

#[test]
fn grayscale_of_pure_green_is_brighter_than_red() {
    let red = to_grayscale(&[255, 0, 0, 255])[0];
    let green = to_grayscale(&[0, 255, 0, 255])[0];
    let blue = to_grayscale(&[0, 0, 255, 255])[0];
    assert!(green > red, "green {} should beat red {}", green, red);
    assert!(red > blue, "red {} should beat blue {}", red, blue);
}

#[test]
fn invert_flips_rgb_keeps_alpha() {
    let px = vec![10, 20, 30, 200];
    let out = invert(&px);
    assert_eq!(out, vec![245, 235, 225, 200]);
}

#[test]
fn invert_is_its_own_inverse() {
    let px = solid(4, 4, 11, 22, 33, 44);
    let twice = invert(&invert(&px));
    assert_eq!(twice, px);
}

#[test]
fn blur_of_solid_image_is_identity() {
    // Размытие однотонного — тот же цвет.
    let px = solid(8, 8, 100, 150, 200, 255);
    let out = box_blur_3x3(&px, 8, 8);
    assert_eq!(out, px);
}

#[test]
fn blur_softens_a_single_bright_pixel() {
    // 3×3 с белой точкой в центре.
    let mut px = vec![0u8; 3 * 3 * 4];
    for i in 0..9 {
        px[i * 4 + 3] = 255; // alpha
    }
    let center = (1 * 3 + 1) * 4;
    px[center] = 255;
    px[center + 1] = 255;
    px[center + 2] = 255;
    let out = box_blur_3x3(&px, 3, 3);
    // Центр размылся: 255/9 ≈ 28.
    assert!(out[center] < 255, "center should be dimmed");
    assert!(out[center] > 0, "but not zero");
}

#[test]
fn sobel_on_uniform_image_is_black() {
    let px = solid(6, 6, 128, 128, 128, 255);
    let out = sobel_edge(&px, 6, 6);
    for i in 0..6 * 6 {
        assert_eq!(out[i * 4], 0, "pixel {} should be black", i);
    }
}

#[test]
fn sobel_on_vertical_edge_lights_up_boundary() {
    // Левые 3 колонки — чёрные, правые 3 — белые.
    let mut px = vec![0u8; 6 * 6 * 4];
    for y in 0..6 {
        for x in 3..6 {
            let i = (y * 6 + x) * 4;
            px[i] = 255;
            px[i + 1] = 255;
            px[i + 2] = 255;
        }
    }
    for i in 0..6 * 6 {
        px[i * 4 + 3] = 255;
    }
    let out = sobel_edge(&px, 6, 6);
    // Граница — колонки 2 и 3. Там должно быть ярче, чем в однотонных зонах.
    let edge_px = (3 * 6 + 3) * 4;
    let flat_px = (3 * 6 + 0) * 4;
    assert!(out[edge_px] > out[flat_px], "edge should be brighter than flat");
}

#[test]
fn identity_filter_returns_copy() {
    let px = solid(4, 4, 1, 2, 3, 4);
    let out = Filter::Identity.apply(&px, 4, 4);
    assert_eq!(out, px);
}

#[test]
fn sample_has_correct_size_and_alpha() {
    let px = make_sample_pixels(64, 48);
    assert_eq!(px.len(), 64 * 48 * 4);
    for chunk in px.chunks_exact(4) {
        assert_eq!(chunk[3], 255);
    }
}

#[test]
fn sample_is_not_uniform() {
    let px = make_sample_pixels(64, 48);
    let first = &px[0..3];
    let mut distinct = 0;
    for chunk in px.chunks_exact(4).step_by(37) {
        if &chunk[0..3] != first {
            distinct += 1;
        }
    }
    assert!(distinct > 10, "sample should have many distinct colors");
}
```

---

## `tests/wasm_canvas_image_processor.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::{Blob, BlobPropertyBag, HtmlCanvasElement, ImageData};

use hello_wasm02::canvas_image_processor::{
    draw_image_data_to_canvas, image_data_to_vec, load_image_from_blob,
    vec_to_image_data,
};

wasm_bindgen_test_configure!(run_in_browser);

/// 1×1 красный PNG — фиксированные байты, без сети.
fn make_tiny_png_blob() -> Blob {
    const PNG_1X1_RED: &[u8] = &[
        0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A,
        0x00, 0x00, 0x00, 0x0D, 0x49, 0x48, 0x44, 0x52,
        0x00, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00, 0x01,
        0x08, 0x02, 0x00, 0x00, 0x00, 0x90, 0x77, 0x53,
        0xDE, 0x00, 0x00, 0x00, 0x0C, 0x49, 0x44, 0x41,
        0x54, 0x08, 0xD7, 0x63, 0xF8, 0xCF, 0xC0, 0x00,
        0x00, 0x00, 0x03, 0x00, 0x01, 0x9A, 0x4C, 0x8C,
        0x4A, 0x00, 0x00, 0x00, 0x00, 0x49, 0x45, 0x4E,
        0x44, 0xAE, 0x42, 0x60, 0x82,
    ];
    let u8arr = js_sys::Uint8Array::from(PNG_1X1_RED);
    let parts = js_sys::Array::new();
    parts.push(u8arr.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type("image/png");
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag).unwrap()
}

#[wasm_bindgen_test]
async fn load_blob_as_image_and_decode() {
    let blob = make_tiny_png_blob();
    let img = load_image_from_blob(&blob).await.expect("decode");
    assert_eq!(img.natural_width(), 1);
    assert_eq!(img.natural_height(), 1);
}

#[wasm_bindgen_test]
fn vec_roundtrip_via_image_data() {
    let w = 3u32;
    let h = 2u32;
    let original: Vec<u8> = (0..w * h * 4).map(|i| (i & 0xFF) as u8).collect();
    let data = vec_to_image_data(&original, w, h).expect("to imagedata");
    assert_eq!(data.width(), w);
    assert_eq!(data.height(), h);
    let back = image_data_to_vec(&data);
    assert_eq!(back, original);
}

#[wasm_bindgen_test]
fn vec_to_image_data_rejects_wrong_size() {
    // Передаём буфер неправильного размера — ждём Err, не панику.
    let r = vec_to_image_data(&[0u8; 5], 2, 2);
    assert!(r.is_err());
}

#[wasm_bindgen_test]
fn draw_image_data_resizes_canvas() {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas: HtmlCanvasElement = document
        .create_element("canvas")
        .unwrap()
        .dyn_into()
        .unwrap();
    let pixels = vec![255u8; 4 * 4 * 4];
    let data: ImageData = vec_to_image_data(&pixels, 4, 4).unwrap();
    draw_image_data_to_canvas(&canvas, &data).unwrap();
    assert_eq!(canvas.width(), 4);
    assert_eq!(canvas.height(), 4);
}

#[wasm_bindgen_test]
fn empty_imagedata_roundtrip() {
    // 1×1 пиксель с нулевой альфой.
    let pixels = vec![0u8, 0, 0, 0];
    let data = vec_to_image_data(&pixels, 1, 1).unwrap();
    let back = image_data_to_vec(&data);
    assert_eq!(back, pixels);
}
```

---

## `src/app.rs`

```rust
//! Глава 21.7 — Canvas 2D: практический проект «Image Processor».
//!
//! Полный round-trip пикселей:
//!   `<img>` → canvas → `ImageData` → `Vec<u8>`
//!   → **Rust-обработка** → `Vec<u8>` → `ImageData` → canvas → экран.
//!
//! Демонстрирует:
//! - `HtmlCanvasElement::getContext("2d")`, `drawImage`, `getImageData`, `putImageData`;
//! - `ImageData::data()` — RGBA, 4 байта на пиксель;
//! - быстрые фильтры на Rust: grayscale, invert, box-blur, Sobel;
//! - замер времени — сколько миллисекунд занимает каждый фильтр;
//! - off-screen обработку через `OffscreenCanvas` (без DOM-узлов в цикле).
//!
//! ## Про сигнатуры `getImageData` / `putImageData`
//!
//! В `web-sys 0.3.105` эти методы имеют две сигнатуры, переключаемые
//! cfg-флагом `web_sys_unstable_apis`. Чтобы код собирался и без флага
//! (host-тесты, CI без настройки) и с флагом (wasm-сборка), вызовы
//! завёрнуты в хелперы `read_image_data` / `write_image_data`.
//! Различие инкапсулировано в одном месте файла.
//!
//! **Файл автономен.** Ничего не импортируется из других модулей проекта,
//! кроме `web-sys` и `js-sys`.
//!
//! Wow: выберите картинку (или нажмите «Пример»), затем кликайте по
//! фильтрам — правый холст обновляется мгновенно, снизу видно, сколько
//! миллисекунд Rust потратил на обработку.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    Blob, CanvasRenderingContext2d, Document, Element, Event, HtmlCanvasElement,
    HtmlElement, HtmlImageElement, HtmlInputElement, ImageData, Node,
    OffscreenCanvas, OffscreenCanvasRenderingContext2d, Url,
};

// =============================================================================
// Утилита обхода бага rust-analyzer: `format!` в tail-позиции теряет тип
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Фильтры и чистые функции обработки пикселей
// =============================================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
enum Filter {
    Identity,
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    fn label(self) -> &'static str {
        match self {
            Filter::Identity => "Оригинал",
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    fn slug(self) -> &'static str {
        match self {
            Filter::Identity => "identity",
            Filter::Grayscale => "grayscale",
            Filter::Invert => "invert",
            Filter::Blur => "blur",
            Filter::Sobel => "sobel",
        }
    }

    fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Identity => rgba.to_vec(),
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

/// Luminance по Rec. 709: Y = 0.2126·R + 0.7152·G + 0.0722·B.
/// Коэффициенты умножены на 10000 для целочисленной арифметики.
/// Альфа сохраняется без изменений.
fn to_grayscale(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        let r = px[0] as u32;
        let g = px[1] as u32;
        let b = px[2] as u32;
        let y = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
        out.push(y);
        out.push(y);
        out.push(y);
        out.push(px[3]);
    }
    out
}

/// Инверсия RGB. Альфа не трогается — иначе прозрачные области
/// станут непрозрачными и картинка «посереет».
fn invert(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        out.push(255 - px[0]);
        out.push(255 - px[1]);
        out.push(255 - px[2]);
        out.push(px[3]);
    }
    out
}

/// Box blur 3×3 — усреднение 9 соседей. На краях окно обрезается
/// (clamp к границам изображения), а не заворачивается.
fn box_blur_3x3(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];
    for y in 0..h {
        for x in 0..w {
            let (mut r, mut g, mut b, mut a, mut n) = (0u32, 0u32, 0u32, 0u32, 0u32);
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let i = (ny * w + nx) * 4;
                    r += rgba[i] as u32;
                    g += rgba[i + 1] as u32;
                    b += rgba[i + 2] as u32;
                    a += rgba[i + 3] as u32;
                    n += 1;
                }
            }
            let i = (y * w + x) * 4;
            out[i] = (r / n) as u8;
            out[i + 1] = (g / n) as u8;
            out[i + 2] = (b / n) as u8;
            out[i + 3] = (a / n) as u8;
        }
    }
    out
}

/// Sobel edge detection по яркости (не по отдельным каналам — так
/// получается чище). Возвращает чёрно-белую карту границ.
fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];

    // Считаем яркость один раз — не гоняем градиенты по трём каналам.
    let mut lum = vec![0u8; w * h];
    for i in 0..w * h {
        let p = i * 4;
        let r = rgba[p] as u32;
        let g = rgba[p + 1] as u32;
        let b = rgba[p + 2] as u32;
        lum[i] = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
    }

    // Классические ядра Sobel.
    const GX: [i32; 9] = [-1, 0, 1, -2, 0, 2, -1, 0, 1];
    const GY: [i32; 9] = [-1, -2, -1, 0, 0, 0, 1, 2, 1];

    for y in 0..h {
        for x in 0..w {
            let (mut sx, mut sy) = (0i32, 0i32);
            let mut k = 0;
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let l = lum[ny * w + nx] as i32;
                    sx += GX[k] * l;
                    sy += GY[k] * l;
                    k += 1;
                }
            }
            let mag = ((sx * sx + sy * sy) as f64).sqrt().min(255.0) as u8;
            let i = (y * w + x) * 4;
            out[i] = mag;
            out[i + 1] = mag;
            out[i + 2] = mag;
            out[i + 3] = 255;
        }
    }
    out
}

/// Встроенное «полотно» — цветные круги на градиенте. Используется,
/// когда пользователь ещё не загрузил свой файл.
fn make_sample_pixels(w: u32, h: u32) -> Vec<u8> {
    let mut out = Vec::with_capacity(w as usize * h as usize * 4);
    let cx1 = w as f32 * 0.30;
    let cy1 = h as f32 * 0.35;
    let cx2 = w as f32 * 0.72;
    let cy2 = h as f32 * 0.55;
    let cx3 = w as f32 * 0.50;
    let cy3 = h as f32 * 0.80;
    let r1 = (w as f32 * 0.12).powi(2);
    let r2 = (w as f32 * 0.14).powi(2);
    let r3 = (w as f32 * 0.10).powi(2);

    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / w as f32;
            let fy = y as f32 / h as f32;
            let mut r = (40.0 + fx * 120.0) as u8;
            let mut g = (60.0 + fy * 120.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            let d1 = (x as f32 - cx1).powi(2) + (y as f32 - cy1).powi(2);
            let d2 = (x as f32 - cx2).powi(2) + (y as f32 - cy2).powi(2);
            let d3 = (x as f32 - cx3).powi(2) + (y as f32 - cy3).powi(2);
            if d1 < r1 { r = 240; g = 80; b = 60; }
            if d2 < r2 { r = 60; g = 200; b = 140; }
            if d3 < r3 { r = 100; g = 100; b = 240; }

            out.push(r);
            out.push(g);
            out.push(b);
            out.push(255);
        }
    }
    out
}

// =============================================================================
// Хелперы для сигнатур canvas: getImageData / putImageData
// =============================================================================

#[cfg(web_sys_unstable_apis)]
fn read_image_data(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0, 0, w as i32, h as i32)
}

#[cfg(not(web_sys_unstable_apis))]
fn read_image_data(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0.0, 0.0, w as f64, h as f64)
}

#[cfg(web_sys_unstable_apis)]
fn write_image_data(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
fn write_image_data(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

// =============================================================================
// Web-обвязка: <img> ↔ ImageData ↔ Vec<u8>
// =============================================================================

const MAX_IMAGE_DIM: u32 = 800;

/// Загружает Blob как `<img>` через Object URL и ждёт `decode()`.
/// Object URL отзывается до возврата — картинка уже в памяти браузера.
async fn load_image_from_blob(blob: &Blob) -> Result<HtmlImageElement, JsValue> {
    let document = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let img: HtmlImageElement = document
        .create_element("img")?
        .dyn_into::<HtmlImageElement>()?;

    let url = Url::create_object_url_with_blob(blob)?;
    img.set_src(&url);

    let result = wasm_bindgen_futures::JsFuture::from(img.decode()).await;
    let _ = Url::revoke_object_url(&url);
    result?;
    Ok(img)
}

/// Рисует `<img>` на OffscreenCanvas и возвращает ImageData + размеры.
fn image_to_image_data(
    img: &HtmlImageElement,
    max_dim: u32,
) -> Result<(ImageData, u32, u32), JsValue> {
    let iw = img.natural_width();
    let ih = img.natural_height();
    if iw == 0 || ih == 0 {
        return Err(JsValue::from_str("image has zero size"));
    }
    let (w, h) = if iw.max(ih) > max_dim {
        let scale = max_dim as f64 / iw.max(ih) as f64;
        (
            (iw as f64 * scale).round().max(1.0) as u32,
            (ih as f64 * scale).round().max(1.0) as u32,
        )
    } else {
        (iw, ih)
    };

    let canvas = OffscreenCanvas::new(w, h)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;

    // drawImage — f64 независимо от cfg.
    ctx.draw_image_with_html_image_element_and_dw_and_dh(
        img, 0.0, 0.0, w as f64, h as f64,
    )?;

    // getImageData — сигнатура зависит от cfg, скрыта в хелпере.
    let data = read_image_data(&ctx, w, h)?;
    Ok((data, w, h))
}

/// `ImageData::data()` в web-sys 0.3.105 возвращает `Clamped<Vec<u8>>` —
/// tuple struct с публичным полем `.0`.
fn image_data_to_vec(data: &ImageData) -> Vec<u8> {
    data.data().0
}

/// Обратный путь: `&[u8]` → `Clamped<&[u8]>` → `ImageData`.
fn vec_to_image_data(pixels: &[u8], w: u32, h: u32) -> Result<ImageData, JsValue> {
    let expected = (w as usize) * (h as usize) * 4;
    if pixels.len() != expected {
        return Err(JsValue::from_str(&format!(
            "pixel buffer size {} != {} for {}x{}",
            pixels.len(), expected, w, h,
        )));
    }
    ImageData::new_with_u8_clamped_array_and_sh(Clamped(pixels), w, h)
}

/// Заменяет содержимое `HtmlCanvasElement` на `ImageData`.
fn draw_image_data_to_canvas(
    canvas: &HtmlCanvasElement,
    data: &ImageData,
) -> Result<(), JsValue> {
    canvas.set_width(data.width());
    canvas.set_height(data.height());
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d ctx"))?;
    let ctx: CanvasRenderingContext2d = ctx_js.dyn_into()?;
    // putImageData — сигнатура зависит от cfg, скрыта в хелпере.
    write_image_data(&ctx, data)
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Полный round-trip пикселей между браузером и Rust. Картинка загружается \
     в <code>&lt;img&gt;</code>, рисуется на canvas, её пиксели достаются через \
     <code>getImageData()</code>, копируются в <code>Vec&lt;u8&gt;</code> и \
     обрабатываются на Rust.",
    "RGBA-порядок: каждый пиксель — четыре байта <code>R, G, B, A</code>. \
     Никакой магии: <code>ImageData::data()</code> и <code>Vec&lt;u8&gt;</code> \
     содержат те же значения. Мы просто переносим их туда-обратно.",
    "Слева — оригинал, справа — результат выбранного фильтра. Снизу — \
     сколько миллисекунд Rust потратил на обработку. Кликните по фильтру — \
     увидите результат мгновенно.",
];

const FILTERS: [Filter; 5] = [
    Filter::Identity,
    Filter::Grayscale,
    Filter::Invert,
    Filter::Blur,
    Filter::Sobel,
];

struct DemoState {
    original_pixels: Vec<u8>,
    width: u32,
    height: u32,
    current: Filter,
    last_ms: f64,
}

impl DemoState {
    fn new() -> Self {
        Self {
            original_pixels: make_sample_pixels(320, 240),
            width: 320,
            height: 240,
            current: Filter::Identity,
            last_ms: 0.0,
        }
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
    h2.set_text_content(Some("21 — Canvas 2D: Image Processor"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Панель управления
    let controls = document.create_element("div")?;
    controls.set_class_name("ci-block");
    controls.set_inner_html(
        r#"<div class="ci-input-row">
               <input type="file" id="ci-input-file" class="ci-file-input"
                      accept="image/*">
               <label for="ci-input-file" class="ci-file-label">
                   <span class="ci-file-icon">🖼</span>
                   <span id="ci-file-name">Загрузить картинку…</span>
               </label>
               <button class="ci-btn ci-btn-secondary" id="ci-sample">Пример</button>
           </div>
           <div class="ci-filters" id="ci-filters"></div>"#,
    );
    root.append_child(&controls)?;

    // Два холста
    let canvases = document.create_element("div")?;
    canvases.set_class_name("ci-canvases");
    canvases.set_inner_html(
        r#"<div class="ci-canvas-panel">
               <div class="ci-canvas-label">Оригинал</div>
               <canvas id="ci-canvas-original" class="ci-canvas"></canvas>
           </div>
           <div class="ci-canvas-panel">
               <div class="ci-canvas-label" id="ci-result-label">Результат</div>
               <canvas id="ci-canvas-result" class="ci-canvas"></canvas>
           </div>"#,
    );
    root.append_child(&canvases)?;

    // Статистика
    let stats = document.create_element("div")?;
    stats.set_class_name("ci-stats");
    stats.set_id("ci-stats");
    stats.set_text_content(Some("…"));
    root.append_child(&stats)?;

    // Footnote
    let footnote = document.create_element("p")?;
    footnote.set_class_name("ci-footnote");
    footnote.set_text_content(Some(
        "Смысл этого проекта в разделении обязанностей: браузер отвечает за \
         декодирование картинки и её отображение, а Rust — за численную обработку. \
         Между ними ровно два переноса — пиксели в Rust и пиксели обратно. \
         Никаких изображений, закодированных в base64, никаких DOM-манипуляций \
         в цикле — только плоский буфер байтов, с которым Rust работает быстрее \
         любого JavaScript-цикла.",
    ));
    root.append_child(&footnote)?;

    // Начальное состояние и первая отрисовка.
    let state = Rc::new(RefCell::new(DemoState::new()));
    {
        let st = state.borrow();
        render_filter_buttons(&document, st.current)?;
        render_original_canvas(&document, &st)?;
        render_result_canvas(&document, &st)?;
        render_stats(&document, &st);
    }

    // Обработчики.
    wire_sample_button(&document, state.clone())?;
    wire_file_input(&document, state.clone())?;
    wire_filter_delegation(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_sample_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let btn = get_button(document, "ci-sample")?;
    let doc = document.clone();
    wire_click(&btn, move || {
        {
            let mut st = state.borrow_mut();
            *st = DemoState::new();
        }
        let st = state.borrow();
        let _ = render_filter_buttons(&doc, st.current);
        let _ = render_original_canvas(&doc, &st);
        let _ = render_result_canvas(&doc, &st);
        render_stats(&doc, &st);
        if let Some(el) = doc.get_element_by_id("ci-file-name") {
            el.set_text_content(Some("Встроенный пример"));
        }
    })
}

fn wire_file_input(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let input = get_input(document, "ci-input-file")?;
    let doc = document.clone();

    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(input) = doc.get_element_by_id("ci-input-file") else {
            return;
        };
        let Ok(input) = input.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Some(files) = input.files() else { return };
        let Some(file) = files.get(0) else { return };

        let name = file.name();
        if let Some(el) = doc.get_element_by_id("ci-file-name") {
            el.set_text_content(Some(&name));
        }
        let blob: Blob = file.into();

        let doc_inner = doc.clone();
        let state_inner = state.clone();
        wasm_bindgen_futures::spawn_local(async move {
            match load_image_from_blob(&blob).await {
                Ok(img) => match image_to_image_data(&img, MAX_IMAGE_DIM) {
                    Ok((data, w, h)) => {
                        let pixels = image_data_to_vec(&data);
                        {
                            let mut st = state_inner.borrow_mut();
                            st.original_pixels = pixels;
                            st.width = w;
                            st.height = h;
                            st.current = Filter::Identity;
                            st.last_ms = 0.0;
                        }
                        let st = state_inner.borrow();
                        let _ = render_filter_buttons(&doc_inner, st.current);
                        let _ = render_original_canvas(&doc_inner, &st);
                        let _ = render_result_canvas(&doc_inner, &st);
                        render_stats(&doc_inner, &st);
                    }
                    Err(e) => {
                        set_error(&doc_inner, &format_str(format_args!(
                            "ошибка чтения пикселей: {}", js_err_to_string(&e)
                        )));
                    }
                },
                Err(e) => {
                    set_error(&doc_inner, &format_str(format_args!(
                        "не удалось декодировать картинку: {}",
                        js_err_to_string(&e)
                    )));
                }
            }
        });
    });
    input.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_filter_delegation(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let container = document
        .get_element_by_id("ci-filters")
        .ok_or_else(|| JsValue::from_str("no #ci-filters"))?;
    let doc = document.clone();

    let closure = Closure::<dyn FnMut(Event)>::new(move |ev: Event| {
        let Some(target) = ev.target() else { return };
        let Ok(el) = target.dyn_into::<Element>() else { return };
        let Ok(Some(btn)) = el.closest("button[data-filter]") else { return };
        let Some(name) = btn.get_attribute("data-filter") else { return };
        let filter = match name.as_str() {
            "identity" => Filter::Identity,
            "grayscale" => Filter::Grayscale,
            "invert" => Filter::Invert,
            "blur" => Filter::Blur,
            "sobel" => Filter::Sobel,
            _ => return,
        };

        // Перерисовка активной кнопки.
        if let Ok(Some(parent)) = el.closest(".ci-filters") {
            if let Ok(children) = parent.query_selector_all("button[data-filter]") {
                for i in 0..children.length() {
                    if let Some(node) = children.get(i) {
                        if let Some(b) = node.dyn_ref::<HtmlElement>() {
                            b.remove_attribute("data-active").ok();
                        }
                    }
                }
            }
        }
        btn.set_attribute("data-active", "").ok();

        // Обработка: применяем фильтр, замеряем время.
        let t0 = js_sys::Date::now();
        let (w, h, filtered) = {
            let st = state.borrow();
            let f = filter.apply(&st.original_pixels, st.width, st.height);
            (st.width, st.height, f)
        };
        let elapsed = js_sys::Date::now() - t0;

        // Обновляем состояние и отрисовываем результат.
        {
            let mut st = state.borrow_mut();
            st.current = filter;
            st.last_ms = elapsed;
        }

        match vec_to_image_data(&filtered, w, h) {
            Ok(data) => {
                let Some(canvas_el) = doc.get_element_by_id("ci-canvas-result") else {
                    return;
                };
                let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() else {
                    return;
                };
                if let Err(e) = draw_image_data_to_canvas(&canvas, &data) {
                    set_error(&doc, &format_str(format_args!(
                        "ошибка отрисовки: {}", js_err_to_string(&e)
                    )));
                    return;
                }
                if let Some(lbl) = doc.get_element_by_id("ci-result-label") {
                    lbl.set_text_content(Some(filter.label()));
                }
                let st = state.borrow();
                render_stats(&doc, &st);
            }
            Err(e) => {
                set_error(&doc, &format_str(format_args!(
                    "ошибка сборки ImageData: {}", js_err_to_string(&e)
                )));
            }
        }
    });

    container.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Рендер ----------

fn render_filter_buttons(document: &Document, active: Filter) -> Result<(), JsValue> {
    let Some(container) = document.get_element_by_id("ci-filters") else {
        return Ok(());
    };
    let mut html = String::new();
    for f in FILTERS {
        let slug = f.slug();
        let label_esc = html_escape(f.label());
        let active_attr = if f == active { " data-active" } else { "" };
        html.push_str(&format_str(format_args!(
            r#"<button class="ci-filter-btn" data-filter="{slug}"{active_attr}>{label_esc}</button>"#
        )));
    }
    container.set_inner_html(&html);
    Ok(())
}

fn render_original_canvas(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let Some(canvas_el) = document.get_element_by_id("ci-canvas-original") else {
        return Ok(());
    };
    let canvas: HtmlCanvasElement = canvas_el.dyn_into()?;
    let data = vec_to_image_data(&state.original_pixels, state.width, state.height)?;
    draw_image_data_to_canvas(&canvas, &data)
}

fn render_result_canvas(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let filtered = state
        .current
        .apply(&state.original_pixels, state.width, state.height);
    let Some(canvas_el) = document.get_element_by_id("ci-canvas-result") else {
        return Ok(());
    };
    let canvas: HtmlCanvasElement = canvas_el.dyn_into()?;
    let data = vec_to_image_data(&filtered, state.width, state.height)?;
    draw_image_data_to_canvas(&canvas, &data)
}

fn render_stats(document: &Document, state: &DemoState) {
    let Some(el) = document.get_element_by_id("ci-stats") else {
        return;
    };
    let text = format_str(format_args!(
        "{}×{} пикселей · {} байт RGBA · {} · {:.2} мс",
        state.width,
        state.height,
        state.original_pixels.len(),
        state.current.label(),
        state.last_ms,
    ));
    el.set_text_content(Some(&text));
    if let Some(html) = el.dyn_ref::<HtmlElement>() {
        html.set_class_name("ci-stats");
    }
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ci-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("ci-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// ---------- Мелочи ----------

fn js_err_to_string(e: &JsValue) -> String {
    if e.is_undefined() {
        return String::from("(undefined)");
    }
    if e.is_null() {
        return String::from("(null)");
    }
    if let Some(s) = e.as_string() {
        return s;
    }
    let s: String = format!("{e:?}");
    s
}

fn set_error(document: &Document, msg: &str) {
    let Some(el) = document.get_element_by_id("ci-stats") else {
        return;
    };
    el.set_text_content(Some(msg));
    if let Some(html) = el.dyn_ref::<HtmlElement>() {
        html.set_class_name("ci-stats ci-stats-error");
    }
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

fn wire_click<F>(btn: &HtmlElement, f: F) -> Result<(), JsValue>
where
    F: FnMut() + 'static,
{
    let closure = Closure::<dyn FnMut()>::new(f);
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
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

## `src/lib.rs` — регистрация модуля

```rust
pub mod canvas_image_processor;   // глава 21.7
```

---

## Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Быстрые host-тесты чистой логики
cargo test --test canvas_pure

# 3. wasm-тесты web-обвязки
wasm-pack test --headless --chrome --test wasm_canvas_image_processor

# 4. Сборка и демо
npm run wasm:dev && npm start
```

---

## Что нужно помнить

**Модуль vs `app.rs`.** Функции в модуле и в демо — **идентичны по логике**, но разные по области видимости: в модуле `pub`, в демо `fn`. Это правило проекта: демо автономно, но тесты могут тестировать тот же код через модуль. Дублирование здесь осознанное — студент открывает `app.rs` и видит цельную картину без отсылок.

**Двойная реализация cfg-хелперов.** `read_image_data` и `write_image_data` есть и в модуле, и в `app.rs`. Обе пары под `#[cfg(web_sys_unstable_apis)]` / `#[cfg(not(...))]`. Так работает и host-сборка (без флага, `f64`), и wasm-сборка (с флагом, `i32`), и rust-analyzer не ругается ни в одной конфигурации.

**Тесты покрывают обе половины.** `canvas_pure.rs` — host, 11 тестов на арифметику фильтров. `wasm_canvas_image_processor.rs` — wasm, 5 тестов на canvas API. Первые дают 90 % уверенности за 1 % времени; вторые нужны, когда правка касается именно web-обвязки.