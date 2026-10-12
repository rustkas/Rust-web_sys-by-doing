# Глава 21.7. Image Processor: полный workflow

## 21.7.1. О чём эта подглава

Шесть предыдущих подглав изучали Canvas 2D по частям: размеры и DPR (21.1), Drawing API (21.2), состояние и композиция (21.3), пиксельный буфер `ImageData` (21.4), загрузка изображений (21.5), фильтры на Rust (21.6). Каждая из них — самостоятельная тема со своим демо.

Но реальные приложения не состоят из отдельных API. Редактор изображений не «использует `createImageBitmap`» и не «применяет градиенты» — он делает всё сразу в одном сценарии: загружает файл, обрабатывает пиксели, показывает результат, отдаёт его обратно. Эта глава — финальная сборка. Всё, что было изучено, сходится в один работающий workflow.

## 21.7.2. Сравнение с базовой главой 21

Прежде чем разбирать новую главу, стоит понять, **чем она отличается от базовой версии**. Обе работают с Canvas 2D, обе применяют фильтры, обе показывают результат на двух холстах. Но между ними есть принципиальная разница.

| Аспект | Базовая глава 21 | Глава 21.7 |
|---|---|---|
| **Модуль** | `canvas_image_processor.rs` — один модуль | `canvas_image_basic.rs` + `canvas_image_pipeline.rs` — два модуля |
| **Загрузка** | `<img>.decode()` — классика | `createImageBitmap(blob, { imageOrientation: "from-image" })` |
| **EXIF-ориентация** | Не применяется — фото с телефона может «лечь на бок» | Применяется явно — фото отображается правильно |
| **Результат загрузки** | `HtmlImageElement` → дополнительный шаг до пикселей | Сразу `Vec<u8>` — без промежуточного `<img>` |
| **Масштабирование** | `drawImage(..., dw, dh)` после декодирования | `createImageBitmap` resize на этапе декодирования |
| **Флаг `resized`** | Нет — молчит о том, что картинка уменьшена | Есть — UI честно говорит «resize: да» |
| **Экспорт в PNG** | Не было — только показ на canvas | `pixels_to_png_blob` — PNG Blob из пикселей |
| **Скачивание** | Не было | Два способа: `<a download>` + `showSaveFilePicker` |
| **Обработка `AbortError`** | Не касались | Отдельная ветка `Ok(None)` — «отменено пользователем» |
| **Источник фильтров** | Локальные функции в модуле | Делегирование в `canvas_filters` (21.6) через композицию |
| **Работа с `Rc<RefCell<State>>`** | `&DemoState` в большинстве мест | Активно, с явными scope'ами — избегает `RefCell already borrowed` |
| **Функции рендера** | Принимают `&DemoState` целиком | Обобщённые: `render_canvas(id, pixels, w, h)` — без `&State` |

**Что осталось общим:**
- чистые функции фильтров — grayscale, invert, blur, sobel;
- два холста — оригинал vs результат;
- тайминг обработки в UI;
- автономность `app.rs` — ничего не импортируется из других модулей;
- cfg-хак для `getImageData` / `putImageData`.

**Что принципиально новое:**
- **EXIF-загрузка.** Фото с телефона приходит правильно повёрнутым — одна опция `imageOrientation: "from-image"`.
- **Экспорт и скачивание.** Результат можно отдать пользователю двумя способами.
- **Реальная работа с `Rc<RefCell<State>>`.** Асинхронная загрузка, общий state между обработчиками — здесь появляется правило «каждый borrow в своём scope».
- **Два модуля вместо одного.** `basic` — минимальный пример, `pipeline` — полный workflow. Композиция вместо дублирования.

**Одно демо — для понимания. Другое — для применения.** `basic` показывает механику round-trip пикселей. `pipeline` показывает, как из этой механики собрать работающий редактор.

## 21.7.3. Что собираем

Полный конвейер от файла до скачанного результата:

```
Blob (файл пользователя)
  → createImageBitmap с EXIF
  → OffscreenCanvas
  → getImageData
  → Vec<u8> (пиксели)
  → Rust-фильтр
  → Vec<u8> (обработанные пиксели)
  → ImageData
  → HtmlCanvasElement (два холста: оригинал и результат)
  → PNG Blob
  → скачивание (<a download> или showSaveFilePicker)
```

Каждый шаг — API из соответствующей подглавы. Все шесть шагов сходятся в одном экране.

**Ключевые моменты:**

1. **EXIF-ориентация.** `createImageBitmap(blob, { imageOrientation: "from-image" })` — фото с телефона отображается правильно. Без этой опции `<img>` покажет картинку с CSS-коррекцией, но `drawImage` нарисует её «как есть» — в сырой ориентации.

2. **Оригинал сохраняется один раз.** После загрузки пиксели лежат в `Vec<u8>` в `DemoState.original_pixels`. Смена фильтра не перечитывает canvas — просто применяет функцию к сохранённому буферу.

3. **Ресайз до 800 px.** Изображения больше 800 px по длинной стороне уменьшаются при загрузке. Иначе 4000×3000 даст 48 МБ буфера и заблокирует вкладку на секунды.

4. **cfg-хак для `getImageData` / `putImageData`.** Сигнатуры зависят от флага `web_sys_unstable_apis`: `f64` в stable, `i32` в unstable. Различие инкапсулировано в трёх хелперах — остальной код не знает про cfg.

## 21.7.4. Что показывает демо

**Layout:**

- **Панель выбора файла.** Инпут + кнопка «Пример» (синтетический паттерн, работает без файла).
- **Ряд кнопок фильтров.** Original, Grayscale, Invert, Blur, Sobel.
- **Строка статуса.** Имя файла, размер, состояние resize, активный фильтр, время обработки.
- **Два холста.** Слева — оригинал (не меняется), справа — результат выбранного фильтра.
- **Кнопки скачивания.** «Скачать PNG» (тихо) и «Сохранить как…» (диалог, Chrome/Edge).

**Сценарий:**

1. Пользователь выбирает файл или нажимает «Пример».
2. Файл декодируется через `createImageBitmap` с EXIF-ориентацией.
3. Изображение ресайзится до 800 px (если больше), рисуется на OffscreenCanvas, читается `getImageData`.
4. Пиксели сохраняются в `Vec<u8>` как оригинал.
5. Пользователь кликает по фильтру. Применяется чистая функция, замеряется время, результат отображается через `putImageData`.
6. «Скачать PNG» — `<a download>` с blob-URL из `canvas.convertToBlob()`.
7. «Сохранить как…» — `showSaveFilePicker` (если браузер поддерживает).

**Обработка ошибок и границ:**

- Если браузер не поддерживает `showSaveFilePicker` — кнопка показывает tooltip и сообщает в статусе.
- Если пользователь отменяет диалог — `AbortError` распознаётся как «отменено», не как ошибка.
- Если изображение было уменьшено — в статусе `resize: да`.

## 21.7.5. Wow-момент

**Первый.** Нажмите «Пример» — увидите два холста: слева синтетический паттерн с цветными кругами, справа он же (Original).

**Второй.** Кликните «Sobel». Правый холст за 30–40 мс покажет карту границ — контуры кругов выделены, однотонные области чёрные. В статусе: `34.2 мс`.

**Третий.** Загрузите фото с телефона. Оно появится **правильно повёрнутым** — благодаря `createImageBitmap` с EXIF. Примените «Grayscale». Нажмите «Скачать PNG». Результат — PNG с обесцвеченным изображением в папке загрузок.

**Четвёртый.** Кликайте фильтры подряд: Original → Invert → Grayscale → Blur → Sobel. Правый холст обновляется мгновенно. Это интерактивный редактор, а не «демо API».

## 21.7.6. Правило проекта, зафиксированное в этой подглаве

**`Rc<RefCell<T>>`: каждый borrow в своём scope.**

`RefCell::borrow()` и `borrow_mut()` возвращают RAII-guard'ы `Ref` и `RefMut`. Они drop'аются **в конце scope**, а не там, где последнее использование. NLL это не оптимизирует. Если два borrow'а одного `RefCell` пересекаются во времени, runtime-проверка паникует с `RefCell already borrowed`.

**Правило:** каждый borrow оборачивается в явный `{ }` блок. `borrow_mut` для записи, `drop` через закрытие scope, затем `borrow` для чтения.

**Второе правило:** функции рендера не должны требовать `&State` целиком. Передача явных параметров (`pixels`, `w`, `h`) устраняет саму возможность конфликта и делает зависимости видимыми.

## 21.7.7. Исходные файлы

### Структура

```
src/
  canvas_image_basic.rs        ← переименован из canvas_image_processor.rs
  canvas_image_pipeline.rs     ← НОВЫЙ модуль 21.7
  canvas_dpi.rs                ← из 21.1
  canvas_drawing.rs            ← из 21.2
  canvas_composition.rs        ← из 21.3
  canvas_imagedata.rs          ← из 21.4
  canvas_image_loading.rs      ← из 21.5
  canvas_filters.rs            ← из 21.6
tests/
  canvas_pure.rs               ← host-тесты базового модуля
  wasm_canvas_image_processor.rs ← wasm-тесты базового модуля
  wasm_canvas_image_pipeline.rs  ← wasm-тесты pipeline
src/lib.rs                     ← регистрация модулей
src/app.rs                     ← демо 21.7
```

### `src/lib.rs`

```rust
pub mod canvas_dpi;                // 21.1
pub mod canvas_drawing;            // 21.2
pub mod canvas_composition;        // 21.3
pub mod canvas_imagedata;          // 21.4
pub mod canvas_image_loading;      // 21.5
pub mod canvas_filters;            // 21.6
pub mod canvas_image_basic;        // базовая обработка (переименован)
pub mod canvas_image_pipeline;     // ← 21.7
```

### `src/canvas_image_basic.rs`

```rust
//! Базовая обработка изображений: `<img>.decode()` + 5 фильтров + round-trip.
//!
//! Минимальный конвейер: `<img>` → canvas → `ImageData` → `Vec<u8>`
//! → Rust-фильтр → `Vec<u8>` → `ImageData` → canvas.
//!
//! Расширенная версия с EXIF-загрузкой и скачиванием — в
//! `canvas_image_pipeline` (глава 21.7).
//!
//! ## Сигнатуры getImageData / putImageData
//!
//! В web-sys 0.3.105 методы имеют две сигнатуры, переключаемые флагом
//! `web_sys_unstable_apis`: `f64` в stable, `i32` в unstable. Все вызовы
//! обёрнуты в cfg-хелперы `read_image_data_offscreen` и
//! `write_image_data_offscreen` — остальной код не знает про cfg.

use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    Blob, ImageBitmap, ImageData, OffscreenCanvas, OffscreenCanvasRenderingContext2d,
};

// =============================================================================
// Фильтры
// =============================================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Filter {
    Original,
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    pub fn label(self) -> &'static str {
        match self {
            Filter::Original => "Оригинал",
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие 3×3",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    pub fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Original => rgba.to_vec(),
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

pub fn to_grayscale(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        let r = px[0] as u32;
        let g = px[1] as u32;
        let b = px[2] as u32;
        let y = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
        out.push(y); out.push(y); out.push(y); out.push(px[3]);
    }
    out
}

pub fn invert(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        out.push(255 - px[0]); out.push(255 - px[1]);
        out.push(255 - px[2]); out.push(px[3]);
    }
    out
}

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
                    r += rgba[i] as u32; g += rgba[i + 1] as u32;
                    b += rgba[i + 2] as u32; a += rgba[i + 3] as u32;
                    n += 1;
                }
            }
            let i = (y * w + x) * 4;
            out[i] = (r / n) as u8; out[i + 1] = (g / n) as u8;
            out[i + 2] = (b / n) as u8; out[i + 3] = (a / n) as u8;
        }
    }
    out
}

pub fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];

    let mut lum = vec![0u8; w * h];
    for i in 0..w * h {
        let p = i * 4;
        let r = rgba[p] as u32;
        let g = rgba[p + 1] as u32;
        let b = rgba[p + 2] as u32;
        lum[i] = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
    }

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
                    sx += GX[k] * l; sy += GY[k] * l; k += 1;
                }
            }
            let mag = ((sx * sx + sy * sy) as f64).sqrt().min(255.0) as u8;
            let i = (y * w + x) * 4;
            out[i] = mag; out[i + 1] = mag; out[i + 2] = mag; out[i + 3] = 255;
        }
    }
    out
}

pub fn make_sample_pixels(w: u32, h: u32) -> Vec<u8> {
    let mut out = vec![0u8; (w * h * 4) as usize];
    let wf = w as f32;
    let hf = h as f32;
    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / wf;
            let fy = y as f32 / hf;
            let i = ((y * w + x) * 4) as usize;

            let mut r = (40.0 + fx * 120.0) as u8;
            let mut g = (60.0 + fy * 120.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            let cx1 = wf * 0.30; let cy1 = hf * 0.35;
            let cx2 = wf * 0.72; let cy2 = hf * 0.55;
            let cx3 = wf * 0.50; let cy3 = hf * 0.80;
            let r1 = (wf * 0.12).powi(2);
            let r2 = (wf * 0.14).powi(2);
            let r3 = (wf * 0.10).powi(2);

            let d1 = (x as f32 - cx1).powi(2) + (y as f32 - cy1).powi(2);
            let d2 = (x as f32 - cx2).powi(2) + (y as f32 - cy2).powi(2);
            let d3 = (x as f32 - cx3).powi(2) + (y as f32 - cy3).powi(2);
            if d1 < r1 { r = 240; g = 80; b = 60; }
            if d2 < r2 { r = 60; g = 200; b = 140; }
            if d3 < r3 { r = 100; g = 100; b = 240; }

            out[i] = r; out[i + 1] = g; out[i + 2] = b; out[i + 3] = 255;
        }
    }
    out
}

// =============================================================================
// ImageData ↔ Vec<u8>
// =============================================================================

pub fn image_data_to_vec(data: &ImageData) -> Vec<u8> {
    data.data().0
}

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

// =============================================================================
// cfg-хелперы: getImageData / putImageData на OffscreenCanvas
// =============================================================================

#[cfg(web_sys_unstable_apis)]
pub fn read_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0, 0, w as i32, h as i32)
}

#[cfg(not(web_sys_unstable_apis))]
pub fn read_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0.0, 0.0, w as f64, h as f64)
}

#[cfg(web_sys_unstable_apis)]
pub fn write_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
pub fn write_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

// =============================================================================
// Загрузка / экспорт / скачивание
// =============================================================================

pub async fn load_image_as_pixels(
    blob: &Blob,
    max_dim: u32,
) -> Result<(Vec<u8>, u32, u32), JsValue> {
    let bitmap = create_image_bitmap_with_exif(blob).await?;
    let iw = bitmap.width();
    let ih = bitmap.height();
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
    ctx.draw_image_with_image_bitmap_and_dw_and_dh(&bitmap, 0.0, 0.0, w as f64, h as f64)?;
    bitmap.close();

    let data = read_image_data_offscreen(&ctx, w, h)?;
    Ok((image_data_to_vec(&data), w, h))
}

async fn create_image_bitmap_with_exif(blob: &Blob) -> Result<ImageBitmap, JsValue> {
    use js_sys::{Function, Object, Reflect};
    let global = js_sys::global();
    let func_value = Reflect::get(&global, &JsValue::from_str("createImageBitmap"))?;
    if !func_value.is_function() {
        return Err(JsValue::from_str("createImageBitmap not available"));
    }
    let func: Function = func_value.dyn_into()?;

    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("imageOrientation"),
        &JsValue::from_str("from-image"),
    )?;

    let promise: js_sys::Promise = func.call2(&global, blob.as_ref(), &opts)?.dyn_into()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<ImageBitmap>()
}

pub async fn pixels_to_png_blob(pixels: &[u8], w: u32, h: u32) -> Result<Blob, JsValue> {
    let canvas = OffscreenCanvas::new(w, h)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;
    let data = vec_to_image_data(pixels, w, h)?;
    write_image_data_offscreen(&ctx, &data)?;
    let promise = canvas.convert_to_blob()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<Blob>()
}

pub fn trigger_anchor_download(blob: &Blob, filename: &str) -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let url = web_sys::Url::create_object_url_with_blob(blob)?;
    let anchor = document
        .create_element("a")?
        .dyn_into::<web_sys::HtmlAnchorElement>()?;
    anchor.set_href(&url);
    anchor.set_download(filename);
    anchor.set_attribute("style", "display: none;")?;
    let body = document.body().ok_or_else(|| JsValue::from_str("no body"))?;
    body.append_child(&anchor)?;
    anchor.click();
    body.remove_child(&anchor)?;
    let _ = web_sys::Url::revoke_object_url(&url);
    Ok(())
}

pub fn is_save_picker_supported() -> bool {
    let global = js_sys::global();
    match js_sys::Reflect::get(&global, &JsValue::from_str("showSaveFilePicker")) {
        Ok(v) => v.is_function(),
        Err(_) => false,
    }
}

pub async fn save_via_picker(
    blob: Blob,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    use js_sys::{Function, Object, Reflect};
    let global = js_sys::global();
    let picker = Reflect::get(&global, &JsValue::from_str("showSaveFilePicker"))?;
    let picker: Function = picker.dyn_into()?;

    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;

    let promise: js_sys::Promise = picker.call1(&global, &opts)?.dyn_into()?;
    let handle = match wasm_bindgen_futures::JsFuture::from(promise).await {
        Ok(h) => h,
        Err(e) if is_abort_error(&e) => return Ok(None),
        Err(e) => return Err(e),
    };

    let writable_promise = call_method(&handle, "createWritable", &[])?;
    let writable = wasm_bindgen_futures::JsFuture::from(writable_promise).await?;
    let write_promise = call_method(&writable, "write", &[blob.as_ref()])?;
    wasm_bindgen_futures::JsFuture::from(write_promise).await?;
    let close_promise = call_method(&writable, "close", &[])?;
    wasm_bindgen_futures::JsFuture::from(close_promise).await?;

    let name = Reflect::get(&handle, &JsValue::from_str("name"))?;
    Ok(Some(name.as_string().unwrap_or_else(|| suggested_name.to_string())))
}

fn call_method(
    obj: &JsValue,
    name: &str,
    args: &[&JsValue],
) -> Result<js_sys::Promise, JsValue> {
    use js_sys::{Array, Function, Reflect};
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
    match js_sys::Reflect::get(e, &JsValue::from_str("name")) {
        Ok(v) => v.as_string().as_deref() == Some("AbortError"),
        Err(_) => false,
    }
}
```

### `src/canvas_image_pipeline.rs`

```rust
//! 21.7. Image Processor Pipeline.
//!
//! Расширение базового модуля `canvas_image_basic`: полный workflow
//! от файла пользователя до скачанного результата.
//!
//! ## Чем отличается от `canvas_image_basic`
//!
//! | Возможность | `basic` | `pipeline` |
//! |---|---|---|
//! | Загрузка через `<img>.decode()` | ✅ | — |
//! | Загрузка через `createImageBitmap` с EXIF | — | ✅ |
//! | Возврат сразу `Vec<u8>` | — | ✅ |
//! | Флаг `resized` | — | ✅ |
//! | Экспорт в PNG Blob | — | ✅ |
//! | Скачивание (`<a download>` + `showSaveFilePicker`) | — | ✅ |
//!
//! Фильтры берутся из `canvas_filters` (21.6) — намеренная композиция,
//! а не дублирование.
//!
//! ## cfg-хак для `getImageData` / `putImageData`
//!
//! Сигнатуры зависят от флага `web_sys_unstable_apis`: `f64` в stable,
//! `i32` в unstable. Различие инкапсулировано в хелперах.

use std::fmt::Arguments;

use js_sys::{Array, Function, Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    Blob, HtmlAnchorElement, ImageBitmap, ImageData, OffscreenCanvas,
    OffscreenCanvasRenderingContext2d, Url,
};

use crate::canvas_filters::{box_blur_3x3, invert, sobel_edge, to_grayscale};

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Палитра фильтров
// =============================================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Filter {
    Original,
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    pub fn label(self) -> &'static str {
        match self {
            Filter::Original => "Оригинал",
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    pub fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Original => rgba.to_vec(),
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

// =============================================================================
// cfg-хелперы
// =============================================================================

#[cfg(web_sys_unstable_apis)]
fn read_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0, 0, w as i32, h as i32)
}

#[cfg(not(web_sys_unstable_apis))]
fn read_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0.0, 0.0, w as f64, h as f64)
}

#[cfg(web_sys_unstable_apis)]
fn write_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
fn write_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

// =============================================================================
// Загрузка с EXIF
// =============================================================================

pub async fn load_image_as_pixels(
    blob: &Blob,
    max_dim: u32,
) -> Result<(Vec<u8>, u32, u32, bool), JsValue> {
    let bitmap = create_image_bitmap_with_exif(blob).await?;
    let iw = bitmap.width();
    let ih = bitmap.height();
    if iw == 0 || ih == 0 {
        return Err(JsValue::from_str("image has zero size"));
    }

    let resized = iw.max(ih) > max_dim;
    let (w, h) = if resized {
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
    ctx.draw_image_with_image_bitmap_and_dw_and_dh(
        &bitmap, 0.0, 0.0, w as f64, h as f64,
    )?;
    bitmap.close();

    let data = read_image_data_offscreen(&ctx, w, h)?;
    let pixels = data.data().0;
    Ok((pixels, w, h, resized))
}

async fn create_image_bitmap_with_exif(blob: &Blob) -> Result<ImageBitmap, JsValue> {
    let global = js_sys::global();
    let func_value = Reflect::get(&global, &JsValue::from_str("createImageBitmap"))?;
    if !func_value.is_function() {
        return Err(JsValue::from_str(
            "createImageBitmap not available in this browser",
        ));
    }
    let func: Function = func_value.dyn_into()?;

    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("imageOrientation"),
        &JsValue::from_str("from-image"),
    )?;

    let promise: js_sys::Promise = func.call2(&global, blob.as_ref(), &opts)?.dyn_into()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<ImageBitmap>()
}

// =============================================================================
// Экспорт: пиксели → PNG Blob
// =============================================================================

pub async fn pixels_to_png_blob(pixels: &[u8], w: u32, h: u32) -> Result<Blob, JsValue> {
    let expected = (w as usize) * (h as usize) * 4;
    if pixels.len() != expected {
        return Err(JsValue::from_str(&format!(
            "pixel buffer size {} != {} for {}x{}",
            pixels.len(), expected, w, h,
        )));
    }

    let canvas = OffscreenCanvas::new(w, h)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;

    let data = ImageData::new_with_u8_clamped_array_and_sh(Clamped(pixels), w, h)?;
    write_image_data_offscreen(&ctx, &data)?;

    let promise = canvas.convert_to_blob()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<Blob>()
}

// =============================================================================
// Скачивание
// =============================================================================

pub fn trigger_anchor_download(blob: &Blob, filename: &str) -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let url = Url::create_object_url_with_blob(blob)?;
    let anchor = document
        .create_element("a")?
        .dyn_into::<HtmlAnchorElement>()?;
    anchor.set_href(&url);
    anchor.set_download(filename);
    anchor.set_attribute("style", "display: none;")?;

    let body = document.body().ok_or_else(|| JsValue::from_str("no body"))?;
    body.append_child(&anchor)?;
    anchor.click();
    body.remove_child(&anchor)?;
    let _ = Url::revoke_object_url(&url);
    Ok(())
}

pub fn is_save_picker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("showSaveFilePicker")),
        Ok(v) if v.is_function()
    ))
}

pub async fn save_via_picker(
    blob: Blob,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    if !is_save_picker_supported() {
        return Err(JsValue::from_str(
            "showSaveFilePicker not supported in this browser",
        ));
    }

    let global = js_sys::global();
    let picker = Reflect::get(&global, &JsValue::from_str("showSaveFilePicker"))?;
    let picker: Function = picker.dyn_into()?;

    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;

    let promise: js_sys::Promise = picker.call1(&global, &opts)?.dyn_into()?;
    let handle = match wasm_bindgen_futures::JsFuture::from(promise).await {
        Ok(h) => h,
        Err(e) if is_abort_error(&e) => return Ok(None),
        Err(e) => return Err(e),
    };

    let writable_promise = call_method(&handle, "createWritable", &[])?;
    let writable = wasm_bindgen_futures::JsFuture::from(writable_promise).await?;

    let write_promise = call_method(&writable, "write", &[blob.as_ref()])?;
    wasm_bindgen_futures::JsFuture::from(write_promise).await?;

    let close_promise = call_method(&writable, "close", &[])?;
    wasm_bindgen_futures::JsFuture::from(close_promise).await?;

    let name = Reflect::get(&handle, &JsValue::from_str("name"))?;
    Ok(Some(name.as_string().unwrap_or_else(|| suggested_name.to_string())))
}

fn call_method(
    obj: &JsValue,
    name: &str,
    args: &[&JsValue],
) -> Result<js_sys::Promise, JsValue> {
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
// Утилиты
// =============================================================================

pub fn derived_png_name(input_name: &str, suffix: &str) -> String {
    let stem = input_name
        .rsplit_once('.')
        .map(|(s, _)| s)
        .unwrap_or(input_name);
    format_str(format_args!("{stem}.{suffix}.png"))
}
```

### `tests/wasm_canvas_image_pipeline.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;
use web_sys::{Blob, BlobPropertyBag};

use image::{ImageFormat, Rgba, RgbaImage};

use hello_wasm02::canvas_image_pipeline::{
    derived_png_name, is_save_picker_supported, load_image_as_pixels,
    pixels_to_png_blob, Filter,
};

wasm_bindgen_test_configure!(run_in_browser);

fn make_png_blob(w: u32, h: u32, r: u8, g: u8, b: u8) -> Blob {
    let mut img = RgbaImage::new(w, h);
    for px in img.pixels_mut() {
        *px = Rgba([r, g, b, 255]);
    }
    let mut cursor = std::io::Cursor::new(Vec::<u8>::new());
    img.write_to(&mut cursor, ImageFormat::Png).expect("encode");
    let bytes = cursor.into_inner();

    let u8arr = js_sys::Uint8Array::from(bytes.as_slice());
    let parts = js_sys::Array::new();
    parts.push(u8arr.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type("image/png");
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag).unwrap()
}

#[wasm_bindgen_test]
async fn load_image_returns_pixels_with_exif_path() {
    let blob = make_png_blob(8, 6, 255, 0, 0);
    let (pixels, w, h, resized) =
        load_image_as_pixels(&blob, 800).await.expect("load");
    assert_eq!((w, h), (8, 6));
    assert_eq!(pixels.len(), 8 * 6 * 4);
    assert!(!resized);
    assert_eq!(pixels[0], 255);
    assert_eq!(pixels[1], 0);
}

#[wasm_bindgen_test]
async fn load_image_respects_max_dim() {
    let blob = make_png_blob(100, 50, 0, 255, 0);
    let (_pixels, w, h, resized) =
        load_image_as_pixels(&blob, 20).await.expect("load");
    assert!(resized, "should be resized from 100×50");
    assert_eq!(w, 20);
    assert_eq!(h, 10);
}

#[wasm_bindgen_test]
async fn pixels_to_png_blob_produces_valid_png() {
    let pixels = vec![255u8; 4 * 4 * 4];
    let blob = pixels_to_png_blob(&pixels, 4, 4).await.expect("png");
    assert_eq!(blob.type_(), "image/png");
    assert!(blob.size() > 0.0);
}

#[wasm_bindgen_test]
async fn pixels_to_png_rejects_wrong_size() {
    let r = pixels_to_png_blob(&[0u8; 5], 2, 2).await;
    assert!(r.is_err());
}

#[wasm_bindgen_test]
fn derived_png_name_replaces_extension() {
    assert_eq!(derived_png_name("photo.jpg", "processed"), "photo.processed.png");
    assert_eq!(derived_png_name("scan", "out"), "scan.out.png");
    assert_eq!(derived_png_name("a.b.jpeg", "sobel"), "a.b.sobel.png");
}

#[wasm_bindgen_test]
fn filter_original_is_identity() {
    let px = vec![1, 2, 3, 4, 5, 6, 7, 8];
    let out = Filter::Original.apply(&px, 2, 1);
    assert_eq!(out, px);
}

#[wasm_bindgen_test]
fn filter_grayscale_uses_rec709() {
    let px = vec![255, 0, 0, 255];
    let out = Filter::Grayscale.apply(&px, 1, 1);
    assert!(out[0] >= 53 && out[0] <= 55, "got {}", out[0]);
    assert_eq!(out[0], out[1]);
    assert_eq!(out[1], out[2]);
}

#[wasm_bindgen_test]
fn save_picker_support_is_reported() {
    let _ = is_save_picker_supported();
}
```

### `src/app.rs`

```rust
//! Глава 21.7 — Image Processor: полный конвейер.
//!
//! Единый workflow: загрузка файла с EXIF-ориентацией → обработка
//! Rust-фильтрами → отображение двух холстов → скачивание результата
//! как PNG (`<a download>` или `showSaveFilePicker`).
//!
//! Пять фильтров: Original, Grayscale, Invert, Blur, Sobel.
//! Живой тайминг обработки.
//!
//! ## Правило работы с Rc<RefCell<DemoState>>
//!
//! `borrow()` и `borrow_mut()` одного RefCell не должны пересекаться
//! во времени. `Ref`/`RefMut` guard'ы drop'аются **в конце scope**,
//! поэтому каждый borrow оборачивается в явный `{ }` блок.
//!
//! **Файл автономен.** Ничего не импортируется из других модулей проекта.
//! Стили `.pipe-*` инжектируются в документ.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    Blob, CanvasRenderingContext2d, Document, Event, HtmlAnchorElement,
    HtmlCanvasElement, HtmlElement, HtmlInputElement, ImageBitmap, ImageData,
    Node, OffscreenCanvas, OffscreenCanvasRenderingContext2d, Url,
};

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "pipe-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(parent) = old.parent_node() {
            parent.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

const STYLE_CSS: &str = r#"
.pipe-desc {
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
.pipe-desc-line { margin: 6px 0; }

.pipe-block {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
}
.pipe-file-input { display: none; }
.pipe-file-label {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 14px;
    background: #21262d;
    color: #c9d1d9;
    border: 1px dashed #30363d;
    border-radius: 6px;
    cursor: pointer;
    font-size: 13px;
    font-family: ui-monospace, monospace;
    transition: border-color 0.15s ease;
}
.pipe-file-label:hover { border-color: #58a6ff; }
.pipe-file-icon { font-size: 16px; }

.pipe-btn {
    padding: 8px 14px;
    border-radius: 6px;
    border: 1px solid #30363d;
    background: #21262d;
    color: #c9d1d9;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    transition: background 0.15s ease;
}
.pipe-btn:hover:not([disabled]) { background: #30363d; }
.pipe-btn:active:not([disabled]) { transform: translateY(1px); }
.pipe-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.pipe-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.pipe-btn-primary:hover:not([disabled]) { background: #2ea043; }
.pipe-btn-active {
    background: #1f6feb;
    border-color: #58a6ff;
    color: #fff;
}

.pipe-panels {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    margin-bottom: 16px;
}
@media (max-width: 900px) {
    .pipe-panels { grid-template-columns: 1fr; }
}
.pipe-panel {
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 8px;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}
.pipe-panel-head {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
}
.pipe-panel-title {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    font-weight: 600;
    color: #c9d1d9;
}
.pipe-panel-time {
    font-family: ui-monospace, monospace;
    font-size: 11px;
    color: #3fb950;
}
.pipe-canvas {
    display: block;
    width: 100%;
    height: 260px;
    background: #0d1117;
    border-radius: 4px;
    image-rendering: pixelated;
}

.pipe-status {
    max-width: 900px;
    padding: 10px 14px;
    background: #0d1117;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    line-height: 1.5;
    margin-bottom: 16px;
}
.pipe-status-error {
    border-left-color: #f85149;
    color: #f85149;
}

.pipe-footnote {
    max-width: 900px;
    padding: 10px 14px;
    background: #161b22;
    border-left: 3px solid #f0883e;
    border-radius: 4px;
    color: #8b949e;
    font-size: 12px;
    line-height: 1.55;
}
"#;

// =============================================================================
// Фильтры — инлайн для автономности
// =============================================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
enum Filter {
    Original,
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    fn label(self) -> &'static str {
        match self {
            Filter::Original => "Оригинал",
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    fn slug(self) -> &'static str {
        match self {
            Filter::Original => "original",
            Filter::Grayscale => "grayscale",
            Filter::Invert => "invert",
            Filter::Blur => "blur",
            Filter::Sobel => "sobel",
        }
    }

    fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Original => rgba.to_vec(),
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

const ALL_FILTERS: [Filter; 5] = [
    Filter::Original,
    Filter::Grayscale,
    Filter::Invert,
    Filter::Blur,
    Filter::Sobel,
];

fn to_grayscale(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        let r = px[0] as u32;
        let g = px[1] as u32;
        let b = px[2] as u32;
        let y = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
        out.push(y); out.push(y); out.push(y); out.push(px[3]);
    }
    out
}

fn invert(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        out.push(255 - px[0]); out.push(255 - px[1]);
        out.push(255 - px[2]); out.push(px[3]);
    }
    out
}

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
                    r += rgba[i] as u32; g += rgba[i + 1] as u32;
                    b += rgba[i + 2] as u32; a += rgba[i + 3] as u32;
                    n += 1;
                }
            }
            let i = (y * w + x) * 4;
            out[i] = (r / n) as u8; out[i + 1] = (g / n) as u8;
            out[i + 2] = (b / n) as u8; out[i + 3] = (a / n) as u8;
        }
    }
    out
}

fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];

    let mut lum = vec![0u8; w * h];
    for i in 0..w * h {
        let p = i * 4;
        let r = rgba[p] as u32;
        let g = rgba[p + 1] as u32;
        let b = rgba[p + 2] as u32;
        lum[i] = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
    }

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
                    sx += GX[k] * l; sy += GY[k] * l; k += 1;
                }
            }
            let mag = ((sx * sx + sy * sy) as f64).sqrt().min(255.0) as u8;
            let i = (y * w + x) * 4;
            out[i] = mag; out[i + 1] = mag; out[i + 2] = mag; out[i + 3] = 255;
        }
    }
    out
}

fn make_sample_pixels(w: u32, h: u32) -> Vec<u8> {
    let mut out = vec![0u8; (w * h * 4) as usize];
    let wf = w as f32;
    let hf = h as f32;
    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / wf;
            let fy = y as f32 / hf;
            let i = ((y * w + x) * 4) as usize;

            let mut r = (40.0 + fx * 120.0) as u8;
            let mut g = (60.0 + fy * 120.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            let cx1 = wf * 0.30; let cy1 = hf * 0.35;
            let cx2 = wf * 0.72; let cy2 = hf * 0.55;
            let cx3 = wf * 0.50; let cy3 = hf * 0.80;
            let r1 = (wf * 0.12).powi(2);
            let r2 = (wf * 0.14).powi(2);
            let r3 = (wf * 0.10).powi(2);

            let d1 = (x as f32 - cx1).powi(2) + (y as f32 - cy1).powi(2);
            let d2 = (x as f32 - cx2).powi(2) + (y as f32 - cy2).powi(2);
            let d3 = (x as f32 - cx3).powi(2) + (y as f32 - cy3).powi(2);
            if d1 < r1 { r = 240; g = 80; b = 60; }
            if d2 < r2 { r = 60; g = 200; b = 140; }
            if d3 < r3 { r = 100; g = 100; b = 240; }

            out[i] = r; out[i + 1] = g; out[i + 2] = b; out[i + 3] = 255;
        }
    }
    out
}

// =============================================================================
// cfg-хелперы для несовместимых сигнатур
// =============================================================================

#[cfg(web_sys_unstable_apis)]
fn read_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0, 0, w as i32, h as i32)
}

#[cfg(not(web_sys_unstable_apis))]
fn read_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(0.0, 0.0, w as f64, h as f64)
}

#[cfg(web_sys_unstable_apis)]
fn write_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
fn write_image_data_offscreen(
    ctx: &OffscreenCanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

#[cfg(web_sys_unstable_apis)]
fn write_image_data_html(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
fn write_image_data_html(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

// =============================================================================
// Web-обвязка
// =============================================================================

fn get_2d_context(
    canvas: &HtmlCanvasElement,
) -> Result<CanvasRenderingContext2d, JsValue> {
    canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?
        .dyn_into::<CanvasRenderingContext2d>()
        .map_err(|_| JsValue::from_str("context is not CanvasRenderingContext2d"))
}

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

const MAX_IMAGE_DIM: u32 = 800;

async fn load_image_as_pixels(
    blob: &Blob,
    max_dim: u32,
) -> Result<(Vec<u8>, u32, u32, bool), JsValue> {
    let bitmap = create_image_bitmap_with_exif(blob).await?;
    let iw = bitmap.width();
    let ih = bitmap.height();
    if iw == 0 || ih == 0 {
        return Err(JsValue::from_str("image has zero size"));
    }
    let resized = iw.max(ih) > max_dim;
    let (w, h) = if resized {
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
    ctx.draw_image_with_image_bitmap_and_dw_and_dh(
        &bitmap, 0.0, 0.0, w as f64, h as f64,
    )?;
    bitmap.close();

    let data = read_image_data_offscreen(&ctx, w, h)?;
    Ok((data.data().0, w, h, resized))
}

async fn create_image_bitmap_with_exif(blob: &Blob) -> Result<ImageBitmap, JsValue> {
    let global = js_sys::global();
    let func_value = Reflect::get(&global, &JsValue::from_str("createImageBitmap"))?;
    if !func_value.is_function() {
        return Err(JsValue::from_str("createImageBitmap not available"));
    }
    let func: Function = func_value.dyn_into()?;

    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("imageOrientation"),
        &JsValue::from_str("from-image"),
    )?;

    let promise: js_sys::Promise = func.call2(&global, blob.as_ref(), &opts)?.dyn_into()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<ImageBitmap>()
}

async fn pixels_to_png_blob(pixels: &[u8], w: u32, h: u32) -> Result<Blob, JsValue> {
    let canvas = OffscreenCanvas::new(w, h)?;
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_js.dyn_into()?;
    let data = vec_to_image_data(pixels, w, h)?;
    write_image_data_offscreen(&ctx, &data)?;
    let promise = canvas.convert_to_blob()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<Blob>()
}

fn trigger_anchor_download(blob: &Blob, filename: &str) -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let url = Url::create_object_url_with_blob(blob)?;
    let anchor = document
        .create_element("a")?
        .dyn_into::<HtmlAnchorElement>()?;
    anchor.set_href(&url);
    anchor.set_download(filename);
    anchor.set_attribute("style", "display: none;")?;
    let body = document.body().ok_or_else(|| JsValue::from_str("no body"))?;
    body.append_child(&anchor)?;
    anchor.click();
    body.remove_child(&anchor)?;
    let _ = Url::revoke_object_url(&url);
    Ok(())
}

fn is_save_picker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("showSaveFilePicker")),
        Ok(v) if v.is_function()
    ))
}

async fn save_via_picker(
    blob: Blob,
    suggested_name: &str,
) -> Result<Option<String>, JsValue> {
    let global = js_sys::global();
    let picker = Reflect::get(&global, &JsValue::from_str("showSaveFilePicker"))?;
    let picker: Function = picker.dyn_into()?;

    let opts = Object::new();
    Reflect::set(
        &opts,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;

    let promise: js_sys::Promise = picker.call1(&global, &opts)?.dyn_into()?;
    let handle = match wasm_bindgen_futures::JsFuture::from(promise).await {
        Ok(h) => h,
        Err(e) if is_abort_error(&e) => return Ok(None),
        Err(e) => return Err(e),
    };

    let writable_promise = call_method(&handle, "createWritable", &[])?;
    let writable = wasm_bindgen_futures::JsFuture::from(writable_promise).await?;
    let write_promise = call_method(&writable, "write", &[blob.as_ref()])?;
    wasm_bindgen_futures::JsFuture::from(write_promise).await?;
    let close_promise = call_method(&writable, "close", &[])?;
    wasm_bindgen_futures::JsFuture::from(close_promise).await?;

    let name = Reflect::get(&handle, &JsValue::from_str("name"))?;
    Ok(Some(name.as_string().unwrap_or_else(|| suggested_name.to_string())))
}

fn call_method(
    obj: &JsValue,
    name: &str,
    args: &[&JsValue],
) -> Result<js_sys::Promise, JsValue> {
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
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Полный конвейер: <code>Blob</code> → <code>createImageBitmap</code> \
     (с EXIF-ориентацией) → OffscreenCanvas → <code>getImageData</code> → \
     <code>Vec&lt;u8&gt;</code> → Rust-фильтр → <code>ImageData</code> → canvas. \
     Пять фильтров, живой тайминг, скачивание результата как PNG.",
    "Оригинал не пересчитывается при смене фильтра: он сохранён в \
     <code>Vec&lt;u8&gt;</code> один раз после загрузки. Смена фильтра — \
     это один вызов <code>apply</code> + <code>putImageData</code>.",
    "Загрузите фото с телефона: оно появится правильно повёрнутым, потому что \
     <code>createImageBitmap</code> вызывается с <code>imageOrientation: \
     \"from-image\"</code>. Скачайте результат как PNG — тихо \
     (<code>&lt;a download&gt;</code>) или с диалогом (<code>showSaveFilePicker</code>).",
];

const CANVAS_W: u32 = 400;
const CANVAS_H: u32 = 240;

struct DemoState {
    original_pixels: Vec<u8>,
    width: u32,
    height: u32,
    current: Filter,
    last_ms: f64,
    file_name: String,
    file_size: u64,
    resized: bool,
}

impl DemoState {
    fn new() -> Self {
        Self {
            original_pixels: make_sample_pixels(CANVAS_W, CANVAS_H),
            width: CANVAS_W,
            height: CANVAS_H,
            current: Filter::Original,
            last_ms: 0.0,
            file_name: String::from("sample"),
            file_size: 0,
            resized: false,
        }
    }
}

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("21.7 — Image Processor: полный конвейер"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("pipe-block");
    controls.set_inner_html(
        r#"<input type="file" id="pipe-file" class="pipe-file-input" accept="image/*">
           <label for="pipe-file" class="pipe-file-label">
               <span class="pipe-file-icon">🖼</span>
               <span id="pipe-file-name">Выбрать файл…</span>
           </label>
           <button class="pipe-btn pipe-btn-primary" id="pipe-sample">Пример</button>"#,
    );
    root.append_child(&controls)?;

    let filters_row = document.create_element("div")?;
    filters_row.set_class_name("pipe-block");
    filters_row.set_id("pipe-filters");
    let mut filters_html = String::new();
    for f in ALL_FILTERS {
        let html = format_str(format_args!(
            r#"<button class="pipe-btn" data-filter="{slug}">{label}</button>"#,
            slug = f.slug(),
            label = f.label(),
        ));
        filters_html.push_str(&html);
    }
    filters_row.set_inner_html(&filters_html);
    root.append_child(&filters_row)?;

    let status = document.create_element("div")?;
    status.set_class_name("pipe-status");
    status.set_id("pipe-status");
    status.set_text_content(Some("…"));
    root.append_child(&status)?;

    let panels = document.create_element("div")?;
    panels.set_class_name("pipe-panels");
    let w_str = CANVAS_W.to_string();
    let h_str = CANVAS_H.to_string();
    let panels_html = format_str(format_args!(
        r#"<div class="pipe-panel">
               <div class="pipe-panel-head">
                   <span class="pipe-panel-title">Оригинал</span>
               </div>
               <canvas id="pipe-canvas-original" class="pipe-canvas"
                       width="{w}" height="{h}"></canvas>
           </div>
           <div class="pipe-panel">
               <div class="pipe-panel-head">
                   <span class="pipe-panel-title" id="pipe-result-label">Результат</span>
                   <span class="pipe-panel-time" id="pipe-result-time">—</span>
               </div>
               <canvas id="pipe-canvas-result" class="pipe-canvas"
                       width="{w}" height="{h}"></canvas>
           </div>"#,
        w = w_str,
        h = h_str,
    ));
    panels.set_inner_html(&panels_html);
    root.append_child(&panels)?;

    let save_row = document.create_element("div")?;
    save_row.set_class_name("pipe-block");
    save_row.set_inner_html(
        r#"<button class="pipe-btn pipe-btn-primary" id="pipe-download">Скачать PNG</button>
           <button class="pipe-btn" id="pipe-save-as">Сохранить как…</button>"#,
    );
    root.append_child(&save_row)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("pipe-footnote");
    footnote.set_text_content(Some(
        "Изображения больше 800 px по длинной стороне масштабируются при загрузке — \
         защита от блокировки вкладки на больших файлах. Ограничение видно в статусе \
         («resize: да»). Сохранить как… доступен в Chrome/Edge — в Firefox/Safari \
         кнопка сообщит, что API нет.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));
    {
        let st = state.borrow();
        let _ = render_canvas(
            &document,
            "pipe-canvas-original",
            &st.original_pixels,
            st.width,
            st.height,
        );
        let _ = render_canvas(
            &document,
            "pipe-canvas-result",
            &st.original_pixels,
            st.width,
            st.height,
        );
        update_result_label(&document, st.current.label(), 0.0);
        update_status(&document, &st);
    }
    let _ = render_filter_buttons(&document, Filter::Original);

    wire_sample_button(&document, state.clone())?;
    wire_file_input(&document, state.clone())?;
    wire_filter_buttons(&document, state.clone())?;
    wire_download_button(&document, state.clone())?;
    wire_save_as_button(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_sample_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("pipe-sample")
        .ok_or_else(|| JsValue::from_str("no pipe-sample"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        let (w, h, label) = {
            let mut st = state.borrow_mut();
            st.original_pixels = make_sample_pixels(CANVAS_W, CANVAS_H);
            st.width = CANVAS_W;
            st.height = CANVAS_H;
            st.current = Filter::Original;
            st.last_ms = 0.0;
            st.file_name = String::from("sample");
            st.file_size = 0;
            st.resized = false;
            (st.width, st.height, st.current.label())
        };

        if let Some(el) = doc.get_element_by_id("pipe-file-name") {
            el.set_text_content(Some("Встроенный пример"));
        }

        {
            let st = state.borrow();
            let _ = render_canvas(&doc, "pipe-canvas-original", &st.original_pixels, w, h);
            let _ = render_canvas(&doc, "pipe-canvas-result", &st.original_pixels, w, h);
            update_status(&doc, &st);
        }

        update_result_label(&doc, label, 0.0);
        let _ = render_filter_buttons(&doc, Filter::Original);
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_file_input(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let input = get_input(document, "pipe-file")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(input) = doc.get_element_by_id("pipe-file") else {
            return;
        };
        let Ok(input) = input.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Some(files) = input.files() else { return };
        let Some(file) = files.get(0) else { return };

        let name = file.name();
        let size = file.size() as u64;
        if let Some(el) = doc.get_element_by_id("pipe-file-name") {
            el.set_text_content(Some(&name));
        }
        let blob: Blob = file.into();

        let doc_inner = doc.clone();
        let state_inner = state.clone();
        wasm_bindgen_futures::spawn_local(async move {
            match load_image_as_pixels(&blob, MAX_IMAGE_DIM).await {
                Ok((pixels, w, h, resized)) => {
                    let label = {
                        let mut st = state_inner.borrow_mut();
                        st.original_pixels = pixels;
                        st.width = w;
                        st.height = h;
                        st.current = Filter::Original;
                        st.last_ms = 0.0;
                        st.file_name = name;
                        st.file_size = size;
                        st.resized = resized;
                        st.current.label()
                    };

                    {
                        let st = state_inner.borrow();
                        let _ = render_canvas(
                            &doc_inner,
                            "pipe-canvas-original",
                            &st.original_pixels,
                            st.width,
                            st.height,
                        );
                        let _ = render_canvas(
                            &doc_inner,
                            "pipe-canvas-result",
                            &st.original_pixels,
                            st.width,
                            st.height,
                        );
                        update_status(&doc_inner, &st);
                    }

                    update_result_label(&doc_inner, label, 0.0);
                    let _ = render_filter_buttons(&doc_inner, Filter::Original);
                }
                Err(e) => {
                    let m = js_err_to_string(&e);
                    set_status_error(
                        &doc_inner,
                        &format_str(format_args!("не удалось загрузить: {m}")),
                    );
                }
            }
        });
    });
    input.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_filter_buttons(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let container = document
        .get_element_by_id("pipe-filters")
        .ok_or_else(|| JsValue::from_str("no pipe-filters"))?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |ev: Event| {
        let Some(target) = ev.target() else { return };
        let Ok(el) = target.dyn_into::<web_sys::Element>() else {
            return;
        };
        let Ok(Some(btn)) = el.closest("button[data-filter]") else {
            return;
        };
        let Some(slug) = btn.get_attribute("data-filter") else {
            return;
        };
        let filter = match slug.as_str() {
            "original" => Filter::Original,
            "grayscale" => Filter::Grayscale,
            "invert" => Filter::Invert,
            "blur" => Filter::Blur,
            "sobel" => Filter::Sobel,
            _ => return,
        };

        let t0 = js_sys::Date::now();

        let (filtered, w, h) = {
            let mut st = state.borrow_mut();
            st.current = filter;
            let filtered = filter.apply(&st.original_pixels, st.width, st.height);
            (filtered, st.width, st.height)
        };

        let elapsed = js_sys::Date::now() - t0;

        {
            let mut st = state.borrow_mut();
            st.last_ms = elapsed;
        }

        let _ = render_canvas(&doc, "pipe-canvas-result", &filtered, w, h);
        update_result_label(&doc, filter.label(), elapsed);

        {
            let st = state.borrow();
            update_status(&doc, &st);
        }

        let _ = render_filter_buttons(&doc, filter);
    });
    container.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_download_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("pipe-download")
        .ok_or_else(|| JsValue::from_str("no pipe-download"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        let (pixels, w, h, name) = {
            let st = state.borrow();
            let processed = st.current.apply(&st.original_pixels, st.width, st.height);
            (processed, st.width, st.height, st.file_name.clone())
        };
        let doc_inner = doc.clone();
        wasm_bindgen_futures::spawn_local(async move {
            match pixels_to_png_blob(&pixels, w, h).await {
                Ok(blob) => {
                    let filename = derived_png_name(&name, "processed");
                    let _ = trigger_anchor_download(&blob, &filename);
                }
                Err(e) => {
                    let m = js_err_to_string(&e);
                    set_status_error(
                        &doc_inner,
                        &format_str(format_args!("ошибка сборки PNG: {m}")),
                    );
                }
            }
        });
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_save_as_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("pipe-save-as")
        .ok_or_else(|| JsValue::from_str("no pipe-save-as"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    if !is_save_picker_supported() {
        let _ = btn.set_attribute(
            "title",
            "showSaveFilePicker недоступен в этом браузере (Firefox/Safari)",
        );
    }

    let closure = Closure::<dyn FnMut()>::new(move || {
        if !is_save_picker_supported() {
            set_status_error(
                &doc,
                "showSaveFilePicker недоступен — используйте «Скачать PNG»",
            );
            return;
        }
        let (pixels, w, h, name) = {
            let st = state.borrow();
            let processed = st.current.apply(&st.original_pixels, st.width, st.height);
            (processed, st.width, st.height, st.file_name.clone())
        };
        let doc_inner = doc.clone();
        wasm_bindgen_futures::spawn_local(async move {
            match pixels_to_png_blob(&pixels, w, h).await {
                Ok(blob) => {
                    let filename = derived_png_name(&name, "processed");
                    match save_via_picker(blob, &filename).await {
                        Ok(Some(saved)) => {
                            if let Some(el) = doc_inner.get_element_by_id("pipe-status") {
                                let m = format_str(format_args!("сохранено как «{saved}»"));
                                el.set_text_content(Some(&m));
                                if let Some(h) = el.dyn_ref::<HtmlElement>() {
                                    h.set_class_name("pipe-status");
                                }
                            }
                        }
                        Ok(None) => {
                            if let Some(el) = doc_inner.get_element_by_id("pipe-status") {
                                el.set_text_content(Some("отменено пользователем"));
                            }
                        }
                        Err(e) => {
                            let m = js_err_to_string(&e);
                            set_status_error(&doc_inner, &m);
                        }
                    }
                }
                Err(e) => {
                    let m = js_err_to_string(&e);
                    set_status_error(&doc_inner, &m);
                }
            }
        });
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Рендер ----------

fn render_canvas(
    document: &Document,
    canvas_id: &str,
    pixels: &[u8],
    w: u32,
    h: u32,
) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id(canvas_id)
        .ok_or_else(|| JsValue::from_str(&format!("no canvas #{canvas_id}")))?
        .dyn_into()?;
    canvas.set_width(w);
    canvas.set_height(h);
    let ctx = get_2d_context(&canvas)?;
    let data = vec_to_image_data(pixels, w, h)?;
    write_image_data_html(&ctx, &data)
}

fn update_result_label(document: &Document, label: &str, ms: f64) {
    if let Some(el) = document.get_element_by_id("pipe-result-label") {
        el.set_text_content(Some(label));
    }
    if let Some(el) = document.get_element_by_id("pipe-result-time") {
        let text = format_str(format_args!("{ms:.1} мс"));
        el.set_text_content(Some(&text));
    }
}

fn update_status(document: &Document, state: &DemoState) {
    let Some(el) = document.get_element_by_id("pipe-status") else {
        return;
    };
    let pixels = state.width as usize * state.height as usize;
    let bytes = state.original_pixels.len();
    let resized = if state.resized { "да" } else { "нет" };
    let size_kb = state.file_size / 1024;
    let text = format_str(format_args!(
        "{name} · {w}×{h} ({pixels} пкс · {bytes} байт) · файл: {size_kb} КБ · \
         resize: {resized} · фильтр: {filter} · время: {ms:.1} мс",
        name = state.file_name,
        w = state.width,
        h = state.height,
        pixels = pixels,
        bytes = bytes,
        size_kb = size_kb,
        resized = resized,
        filter = state.current.label(),
        ms = state.last_ms,
    ));
    el.set_text_content(Some(&text));
    if let Some(h) = el.dyn_ref::<HtmlElement>() {
        h.set_class_name("pipe-status");
    }
}

fn set_status_error(document: &Document, msg: &str) {
    if let Some(el) = document.get_element_by_id("pipe-status") {
        el.set_text_content(Some(msg));
        if let Some(h) = el.dyn_ref::<HtmlElement>() {
            h.set_class_name("pipe-status pipe-status-error");
        }
    }
}

fn render_filter_buttons(document: &Document, active: Filter) -> Result<(), JsValue> {
    let Some(container) = document.get_element_by_id("pipe-filters") else {
        return Ok(());
    };
    let mut html = String::new();
    for f in ALL_FILTERS {
        let cls = if f == active {
            "pipe-btn pipe-btn-active"
        } else {
            "pipe-btn"
        };
        let row = format_str(format_args!(
            r#"<button class="{cls}" data-filter="{slug}">{label}</button>"#,
            cls = cls,
            slug = f.slug(),
            label = f.label(),
        ));
        html.push_str(&row);
    }
    container.set_inner_html(&html);
    Ok(())
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("pipe-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("pipe-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// ---------- Мелочи ----------

fn derived_png_name(input_name: &str, suffix: &str) -> String {
    let stem = input_name
        .rsplit_once('.')
        .map(|(s, _)| s)
        .unwrap_or(input_name);
    format_str(format_args!("{stem}.{suffix}.png"))
}

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

fn get_input(document: &Document, id: &str) -> Result<HtmlInputElement, JsValue> {
    document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no input #{id}")))?
        .dyn_into::<HtmlInputElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} is not input")))
}
```

## 21.7.8. Команды запуска

```bash
# Проверка типов
cargo check --target wasm32-unknown-unknown

# Host-тесты базового модуля
cargo test --test canvas_pure

# wasm-тесты базового модуля
wasm-pack test --headless --chrome --test wasm_canvas_image_processor

# wasm-тесты pipeline
wasm-pack test --headless --chrome --test wasm_canvas_image_pipeline

# Демо
npm run wasm:dev && npm start
```

## 21.7.9. Итог

Глава 21.7 — это точка сходимости всей серии Canvas 2D. Шесть подглав изучали API по частям, седьмая собрала их в работающий редактор изображений:

- загрузка файла с EXIF-ориентацией из 21.5;
- OffscreenCanvas из 21.1;
- `getImageData` / `putImageData` из 21.4;
- чистые фильтры из 21.6;
- два способа скачивания из 20.3.

**Отличие от базовой главы 21** — в полноте workflow. Базовая показывала механику round-trip пикселей (`<img> → canvas → фильтр → canvas`). Текущая добавляет EXIF, экспорт, скачивание, флаг `resized` и разделение на два модуля (basic + pipeline). Одно решение — для понимания, другое — для применения.

Демо автономно, стили `.pipe-*` инжектируются в документ. Все правила проекта применены: `format_str` и `bool_of` для tail-позиций, cfg-хелперы для несовместимых сигнатур, `Rc<RefCell<State>>` с явными scope'ами, `bitmap.close()`, `is_function()` перед вызовом, обработка `AbortError` как «отменено, а не ошибка».

Глава 21 закрыта. Следующие части курса — WebGL, WebGPU, WebRTC, Web Codecs. Canvas — фундамент графики; остальное — про то, как делать её быстрее или специализированнее.