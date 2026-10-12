# Глава 21.5. Загрузка изображений: `<img>`, `ImageBitmap`, EXIF

## 21.5.0. Обновлённая структура

```
src/
  canvas_image_loading.rs      ← НОВЫЙ модуль главы 21.5
  canvas_dpi.rs                ← из 21.1
  canvas_drawing.rs            ← из 21.2
  canvas_composition.rs        ← из 21.3
  canvas_imagedata.rs          ← из 21.4
tests/
  wasm_canvas_image_loading.rs ← wasm-тесты
src/lib.rs                     ← +1 строка
src/app.rs                     ← демо текущей подглавы (заменяется целиком)
Cargo.toml                     ← +1 фича (ImageBitmap)
web/styles/main.css            ← правок не требуется
```

**Комментарий.** Модуль `canvas_image_loading.rs` — про три способа получить пиксели из файла: `<img>`, `ImageBitmap`, `OffscreenCanvas`. Ключевая тема — EXIF-ориентация: фото с телефона приходит повёрнутым, если не применять EXIF автоматически. Префикс CSS — `il-` (image loading), уникален в проекте.

## Ссылки

- MDN: [`createImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/createImageBitmap)
- MDN: [`ImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap)
- MDN: [`HTMLImageElement.decode`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/decode)
- MDN: [`drawImage`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawImage)
- MDN: [EXIF orientation](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types#exif_orientation)
- web.dev: [Create Image Bitmap](https://web.dev/articles/createimagebitmap)
- WHATWG Spec: [`createImageBitmap`](https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#dom-createimagebitmap)

---

## 21.5.1. О чём эта подглава

До этого мы работали с пикселями, которые **уже** находятся в памяти. В 21.4 брали `ImageData` с холста, модифицировали, возвращали. Холст — источник, холст — приёмник. Всё в одной плоскости.

Но откуда холст получает пиксели? Когда пользователь загружает файл, у нас есть `Blob` — сырые байты PNG или JPEG. Их надо **декодировать** — превратить сжатые данные в RGBA-пиксели. Это делает браузер, но у нас три разных способа его попросить.

Первый — `<img>`. Классика: создаём элемент, ставим `src`, ждём загрузки. Второй — `createImageBitmap`. Прямая декодировка без DOM. Третий — `OffscreenCanvas` в комбинации с одним из первых двух, для трансформаций. У каждого свои тайминги, свои ограничения и своя роль в пайплайне.

И есть тема, без которой всё это работает **неправильно** на реальных фотографиях — **EXIF-ориентация**. Фото с телефона приходит с метаданными «этот снимок надо повернуть на 90°», и браузер не применяет их автоматически. Ни `<img>`, ни `createImageBitmap` — без явной опции. Демо показывает это на живом примере: одна и та же фотография слева «лежит на боку», справа — стоит как надо.

**Ключевые темы:**
- Три способа получить пиксели: `<img>`, `ImageBitmap`, `OffscreenCanvas`;
- `img.decode()` — Promise, ждёт декодирования; `onload` — событие;
- `createImageBitmap(blob, options)` — прямая декодировка без DOM;
- `ImageBitmap` vs `HTMLImageElement` — что быстрее, что удобнее;
- EXIF-ориентация: почему фото с телефона приходит повёрнутым;
- Масштабирование при загрузке: `resizeWidth` / `resizeHeight` / `resizeQuality`.

## 21.5.2. Что уже есть и что появится

К концу 21.4 мы умеем:
- рисовать фигуры, текст, градиенты (21.2);
- управлять стеком состояний, трансформациями, композицией (21.3);
- читать и писать пиксели через `ImageData` (21.4);
- работать с `Vec<u8>` как с плоским RGBA-буфером.

К концу 21.5 будем уметь:
- декодировать `Blob` в `<img>` через `decode()`;
- декодировать `Blob` в `ImageBitmap` через `createImageBitmap`;
- применять EXIF-ориентацию при загрузке одной опцией;
- масштабировать изображение **при декодировании**, без промежуточного canvas;
- измерять тайминги обоих способов и понимать, когда какой выбирать.

## 21.5.3. Три пути: `<img>`, `ImageBitmap`, `OffscreenCanvas`

**`<img>`** — исторически первый способ. Создаём элемент, ставим `src` (обычно `blob:` URL из `URL.createObjectURL`), ждём события `load` или вызываем `decode()` (Promise-версия). Результат — `HTMLImageElement`, у которого есть `naturalWidth`, `naturalHeight`, и который можно передавать в `ctx.drawImage(img, ...)`.

**`ImageBitmap`** — современный способ. `createImageBitmap(blob)` возвращает Promise, резолвится в `ImageBitmap` — объект без DOM-обвязки, с полями `width`, `height`. Передаётся в `ctx.drawImage(bitmap, ...)` так же, как `<img>`.

**`OffscreenCanvas`** — не способ декодирования, а способ **работы** с уже декодированным изображением. Рисуем на невидимом canvas, получаем `ImageData`, модифицируем. Используется в пайплайнах, где нужно несколько проходов рисования.

Между `<img>` и `ImageBitmap` есть практическая разница:

| | `<img>` | `ImageBitmap` |
|---|---|---|
| Тип результата | `HTMLImageElement` | `ImageBitmap` |
| DOM-элемент | Да | Нет |
| `decode()` | Promise | — |
| EXIF-ориентация | CSS `image-orientation` (по умолчанию `from-image`) | Только при `imageOrientation: "from-image"` |
| Масштабирование | — | `resizeWidth`, `resizeHeight`, `resizeQuality` |
| Закрытие | Сборка мусора | Явный `close()` |
| Скорость | Чуть медленнее (создаёт DOM-узел) | Чуть быстрее |

Разница в скорости на современных браузерах — 10–20 %, не катастрофа. Разница в **функциональности** — существеннее: `ImageBitmap` умеет resize при загрузке и работает в Worker'ах. `<img>` — нет.

## 21.5.4. `img.decode()` vs `onload`

У `<img>` два способа узнать, что картинка готова:

**`onload`** — событие. Ставим `src`, вешаем обработчик `onload`, ждём. Внутри обработчика картинка доступна для рисования. Классический способ, работает везде.

**`decode()`** — Promise. Ставим `src`, вызываем `img.decode()`, который резолвится, когда картинка полностью разобрана и **готова к синхронному рисованию без задержки**. Плюс: `drawImage` не заблокирует main thread на декодирование при первом вызове.

Разница принципиальна для UX. С `onload` браузер может показать картинку ещё не декодированной — она «появится» с небольшой задержкой при первой отрисовке. С `decode()` к моменту резолва всё готово.

Ещё один нюанс: `onload` **может не сработать**, если картинка уже загружена (из кэша). Тогда его надо проверять через `img.complete`. `decode()` этой проблемы не имеет — возвращает Promise, который резолвится в любом случае.

## 21.5.5. `createImageBitmap(blob, options)`

Сигнатура:

```js
createImageBitmap(image, options)
```

`image` — что угодно: `Blob`, `HTMLImageElement`, `HTMLCanvasElement`, `OffscreenCanvas`, `ImageData`. `options` — объект:

```js
{
    imageOrientation: "none" | "flipY" | "from-image",
    premultiplyAlpha: "none" | "premultiply" | "default",
    colorSpaceConversion: "none" | "default",
    resizeWidth: number,
    resizeHeight: number,
    resizeQuality: "pixelated" | "low" | "medium" | "high"
}
```

Возвращает `Promise<ImageBitmap>`. Резолвится после декодирования.

**Ключевые опции:**

**`imageOrientation: "from-image"`** — применяет EXIF-ориентацию из метаданных JPEG. Без неё картинка приходит «как есть», с поворотом, встроенным в EXIF.

**`resizeWidth`, `resizeHeight`** — масштабируют при декодировании. Браузер сам решает, как это сделать эффективнее — без создания промежуточного `HTMLCanvasElement`. Для больших файлов (например, 4000×3000) это критично: декодировать в 4000×3000, потом рисовать на 800×600 canvas — значит аллоцировать 48 МБ и почти всё выбросить. `resizeWidth: 800` сразу даёт маленький буфер.

**`resizeQuality`** — как именно масштабировать. `pixelated` (быстро, грубо), `low` (быстро), `medium` (по умолчанию), `high` (медленнее, качественнее).

**Важная оговорка.** `createImageBitmap` в Chrome **строже к входным данным**, чем `<img>.decode()`. То, что декодируется через `<img>`, может быть отвергнуто `createImageBitmap` с `InvalidStateError`, особенно на граничных случаях (минимальные PNG 1×1 с очень коротким IDAT). Это не баг и не «несовместимость» — просто разные декодеры. В тестах это учитывается: источник генерируется во время выполнения через `image` crate, чтобы гарантировать корректный PNG.

## 21.5.6. EXIF-ориентация: почему фото лежит на боку

Когда телефон делает снимок, аппаратный сенсор почти всегда повёрнут **фиксированно**. Программно камера решает: записать пиксели как есть и добавить в EXIF-метаданные тег `Orientation` со значением от 1 до 8, означающим «как надо повернуть для правильного отображения».

Восемь значений:

| Значение | Что означает |
|---|---|
| 1 | Ориентация нормальная |
| 2 | Отражено по горизонтали |
| 3 | Поворот 180° |
| 4 | Отражено по вертикали |
| 5 | Отражено по главной диагонали |
| 6 | Поворот 90° по часовой |
| 7 | Отражено по второй диагонали |
| 8 | Поворот 270° по часовой |

Большинство фото с телефона имеют значение **6** (или иногда 8).

**Проблема:** браузеры не смотрят в EXIF автоматически для canvas. `<img>` показывает картинку с CSS-коррекцией (по умолчанию `image-orientation: from-image`), но при `drawImage(img, ...)` на canvas поведение может отличаться. `createImageBitmap` без явной опции тоже не применяет EXIF.

**Решения:**

1. **CSS `image-orientation: from-image`** на `<img>` — работает для отображения, не всегда для рисования на canvas.

2. **`createImageBitmap(blob, { imageOrientation: "from-image" })`** — явно применяет EXIF при декодировании. Результат: `ImageBitmap` уже повёрнут правильно.

3. **Ручной разбор EXIF** + `ctx.translate` / `ctx.rotate` / `ctx.scale` перед `drawImage`. Работает всегда, но требует кода для чтения JPEG-заголовка.

**Рекомендация:** использовать `imageOrientation: "from-image"` в `createImageBitmap`. Одна строка — и фото с телефона отображаются корректно.

## 21.5.7. Что показывает демо

Демо — **«инспектор загрузки»**. Пользователь загружает файл. Два холста рядом:

- **Слева — `<img>.decode()`**, без явной EXIF-коррекции.
- **Справа — `createImageBitmap(blob, { imageOrientation: "from-image" })`**, с EXIF-коррекцией.

Оба холста рисуют картинку через `ctx.drawImage` с автоматическим масштабированием под размер холста. Оба показывают **тайминг загрузки** в миллисекундах.

**Контролы:**

- **Инпут файла.** Пользователь выбирает любое изображение — JPG, PNG, WebP.
- **Кнопка «Пример».** Генерирует синтетический PNG 300×200 в рантайме через `image` crate — чтобы демо работало без файла от пользователя.
- **Чекбокс «Применять EXIF-ориентацию».** Переключает поведение правого холста.
- **Слайдер «Масштаб».** От 100 % до ≤400. При уменьшении `createImageBitmap` получает `resizeWidth` / `resizeHeight`.

**Статус внизу** показывает:
- имя файла и его размер;
- состояние EXIF-опции и resize;
- время `img.decode()` (мс);
- время `createImageBitmap` (мс).

**Честная оговорка про встроенный пример.** Синтетический PNG **не содержит EXIF-ориентации** — PNG её не поддерживает в формате JPEG. Поэтому переключатель EXIF на встроенном примере ничего визуально не меняет. Чтобы увидеть эффект, нужно загрузить **реальное фото с телефона** (JPEG, снятое в портретном режиме). Это отражено в footnote.

## 21.5.8. Wow-момент

**Первый.** Загрузите фотографию с телефона, снятую в портретном режиме. Слева `<img>` с CSS `image-orientation: from-image` (современные браузеры) отобразит её правильно, но в canvas-отрисовке картинка может «лечь на бок». Справа `createImageBitmap` с явной опцией `imageOrientation: "from-image"` повернёт корректно.

**Второй.** Переключите чекбокс «Применять EXIF» на правом холсте. Картинка повернётся на 90° в реальном времени. Так видно, что коррекция — это **не автомат**, а явная опция.

**Третий.** Двигайте слайдер масштаба на большой фотографии. `createImageBitmap` с `resizeWidth: 400` декодирует заметно быстрее — цифры в статусе меняются. Это практическая причина использовать `resizeWidth`, а не `drawImage` с коэффициентом: экономия и по времени, и по памяти.

## 21.5.9. Чему это учит

**Первое.** `<img>` и `ImageBitmap` — **разные способы получить одно и то же**. Оба дают пиксели, оба принимаются `drawImage`. Разница в деталях.

**Второе.** `img.decode()` — **Promise, не событие**. Резолвится всегда, даже если картинка из кэша. `onload` может не сработать при повторной загрузке.

**Третье.** EXIF-ориентация — **не автомат**. `<img>` показывает картинку с CSS-коррекцией, но `drawImage(img)` может нарисовать её неверно. `createImageBitmap` без `imageOrientation: "from-image"` тоже не применяет EXIF. Единственный надёжный способ — явная опция.

**Четвёртое.** `resizeWidth` / `resizeHeight` — **экономия при загрузке**. Не декодируйте в полный размер, если нужен маленький.

**Пятое.** `ImageBitmap` надо **закрывать явно**. Если создаёте много `ImageBitmap`, вызывайте `bitmap.close()`.

**Шестое.** `createImageBitmap` **строже** к входным данным, чем `<img>.decode()`. Граничные случаи (минимальные PNG) могут отвергаться. Это причина использовать проверенные источники изображений.

## 21.5.10. Исходные файлы

### `src/canvas_image_loading.rs`

```rust
//! 21.5. Загрузка изображений: `<img>`, `ImageBitmap`, EXIF.
//!
//! Три способа получить пиксели из файла:
//! - `<img>` + `decode()` — классика, работает везде;
//! - `createImageBitmap(blob)` — прямая декодировка без DOM;
//! - `OffscreenCanvas` — для работы с уже декодированным изображением.
//!
//! ## EXIF-ориентация
//!
//! Фото с телефона приходит с метаданными «повернуть на 90°».
//! Ни `<img>`, ни `createImageBitmap` без явной опции не применяют
//! их автоматически для canvas-отрисовки. Решение —
//! `imageOrientation: "from-image"` в `createImageBitmap`.
//!
//! ## resize при загрузке
//!
//! `resizeWidth` / `resizeHeight` масштабируют при декодировании,
//! без промежуточного canvas. Для больших файлов это экономит
//! и время, и память.
//!
//! ## createImageBitmap строже к входным данным
//!
//! В отличие от `<img>.decode()`, `createImageBitmap` использует
//! другой декодер и может отвергать граничные случаи (например,
//! минимальные PNG 1×1 с очень коротким IDAT-сегментом) с ошибкой
//! `InvalidStateError: The source image could not be decoded`.
//! Это не баг — разные пути декодирования.

use js_sys::{Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Blob, HtmlImageElement, ImageBitmap, Url};

/// Опции для `createImageBitmap`.
#[derive(Clone, Copy, Debug)]
pub struct BitmapOptions {
    /// Применять EXIF-ориентацию (`"from-image"`).
    pub apply_exif: bool,
    /// Максимальный размер стороны. `None` — без resize.
    pub max_dim: Option<u32>,
    /// Качество resize: `"low"`, `"medium"`, `"high"`, `"pixelated"`.
    pub quality: &'static str,
}

impl Default for BitmapOptions {
    fn default() -> Self {
        Self {
            apply_exif: true,
            max_dim: None,
            quality: "high",
        }
    }
}

/// Загружает Blob как `<img>` и ждёт `decode()`.
///
/// `img.decode()` — Promise, резолвится, когда картинка **полностью
/// декодирована** и готова к синхронному рисованию. В отличие от
/// `onload`, работает даже если картинка уже в кэше.
///
/// Object URL отзывается до возврата — картинка уже в памяти браузера.
pub async fn load_as_image(blob: &Blob) -> Result<HtmlImageElement, JsValue> {
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

/// Декодирует Blob в `ImageBitmap` через `createImageBitmap`.
///
/// Вызываем через `Reflect`, потому что `createImageBitmap` — глобальная
/// функция, которую не всегда удобно звать через web-sys напрямую.
/// Универсальный обход. Перед вызовом проверяем, что функция существует:
/// иначе получим понятную ошибку, а не `TypeError: x is not a function`.
pub async fn load_as_bitmap(
    blob: &Blob,
    options: BitmapOptions,
) -> Result<ImageBitmap, JsValue> {
    let global = js_sys::global();
    let func_value = Reflect::get(&global, &JsValue::from_str("createImageBitmap"))?;
    if !func_value.is_function() {
        return Err(JsValue::from_str(
            "createImageBitmap not available in this browser",
        ));
    }
    let func: js_sys::Function = func_value.dyn_into()?;

    let opts = Object::new();

    if options.apply_exif {
        Reflect::set(
            &opts,
            &JsValue::from_str("imageOrientation"),
            &JsValue::from_str("from-image"),
        )?;
    }

    if let Some(max_dim) = options.max_dim {
        // resizeWidth / resizeHeight работают вместе: браузер
        // сохраняет aspect ratio сам.
        Reflect::set(
            &opts,
            &JsValue::from_str("resizeWidth"),
            &JsValue::from_f64(max_dim as f64),
        )?;
        Reflect::set(
            &opts,
            &JsValue::from_str("resizeHeight"),
            &JsValue::from_f64(max_dim as f64),
        )?;
        Reflect::set(
            &opts,
            &JsValue::from_str("resizeQuality"),
            &JsValue::from_str(options.quality),
        )?;
    }

    let promise: js_sys::Promise = func.call2(&global, blob.as_ref(), &opts)?.dyn_into()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<ImageBitmap>()
}

/// Закрывает `ImageBitmap`, освобождая память. Идемпотентно —
/// повторный вызов не паникует.
pub fn close_bitmap(bitmap: &ImageBitmap) {
    bitmap.close();
}
```

---

### `src/lib.rs` — добавление

```rust
pub mod canvas_dpi;             // 21.1
pub mod canvas_drawing;         // 21.2
pub mod canvas_composition;     // 21.3
pub mod canvas_imagedata;       // 21.4
pub mod canvas_image_loading;   // ← 21.5
```

---

### `Cargo.toml` — дополнение к `[dependencies.web-sys] features`

```toml
    "ImageBitmap",   # для типа ImageBitmap
```

Плюс всё, что было из 21.1–21.4.

---

### `tests/wasm_canvas_image_loading.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;
use web_sys::{Blob, BlobPropertyBag};

use image::{ImageFormat, Rgba, RgbaImage};

use hello_wasm02::canvas_image_loading::{
    load_as_bitmap, load_as_image, BitmapOptions,
};

wasm_bindgen_test_configure!(run_in_browser);

/// Генерирует валидный PNG 4×4 в рантайме.
///
/// Ручные 1×1 PNG (с минимальным IDAT) могут отвергаться
/// `createImageBitmap` на некоторых версиях Chrome из-за edge case
/// декодера. `image` crate гарантирует корректный файл, а 4×4 — вне
/// зоны граничных случаев.
fn tiny_png_blob() -> Blob {
    let mut img = RgbaImage::new(4, 4);
    for px in img.pixels_mut() {
        *px = Rgba([255, 0, 0, 255]); // непрозрачный красный
    }

    let mut cursor = std::io::Cursor::new(Vec::<u8>::new());
    img.write_to(&mut cursor, ImageFormat::Png)
        .expect("PNG encode");
    let png_bytes = cursor.into_inner();

    let u8arr = js_sys::Uint8Array::from(png_bytes.as_slice());
    let parts = js_sys::Array::new();
    parts.push(u8arr.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type("image/png");
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag).unwrap()
}

#[wasm_bindgen_test]
async fn img_decode_resolves_and_has_size() {
    let blob = tiny_png_blob();
    let img = load_as_image(&blob).await.expect("decode");
    assert_eq!(img.natural_width(), 4);
    assert_eq!(img.natural_height(), 4);
}

#[wasm_bindgen_test]
async fn bitmap_loads_with_correct_size() {
    let blob = tiny_png_blob();
    let options = BitmapOptions {
        apply_exif: false,
        max_dim: None,
        quality: "high",
    };
    let bitmap = load_as_bitmap(&blob, options).await.expect("bitmap");
    assert_eq!(bitmap.width(), 4);
    assert_eq!(bitmap.height(), 4);
    bitmap.close();
}

#[wasm_bindgen_test]
async fn bitmap_with_resize() {
    // 4×4 → resizeWidth: 16 даёт 16×16 (upscale сохраняет aspect ratio).
    let blob = tiny_png_blob();
    let options = BitmapOptions {
        apply_exif: false,
        max_dim: Some(16),
        quality: "medium",
    };
    let bitmap = load_as_bitmap(&blob, options).await.expect("resize");
    assert_eq!(bitmap.width(), 16);
    assert_eq!(bitmap.height(), 16);
    bitmap.close();
}

#[wasm_bindgen_test]
async fn bitmap_with_exif_option_does_not_panic() {
    // `imageOrientation: "from-image"` на PNG без EXIF-метаданных
    // должен быть просто проигнорирован — не падать.
    let blob = tiny_png_blob();
    let options = BitmapOptions {
        apply_exif: true,
        max_dim: None,
        quality: "high",
    };
    let bitmap = load_as_bitmap(&blob, options).await.expect("exif");
    assert_eq!(bitmap.width(), 4);
    assert_eq!(bitmap.height(), 4);
    bitmap.close();
}

#[wasm_bindgen_test]
async fn bitmap_close_is_safe() {
    let blob = tiny_png_blob();
    let bitmap = load_as_bitmap(&blob, BitmapOptions::default())
        .await
        .unwrap();
    // Идемпотентность: повторный close() безопасен.
    bitmap.close();
    bitmap.close();
}
```

---

### `src/app.rs` — автономный демо-пример

```rust
//! Глава 21.5 — Загрузка изображений: `<img>`, `ImageBitmap`, EXIF.
//!
//! «Инспектор загрузки»: один файл, два холста.
//! - Слева — `<img>.decode()` без явной EXIF-коррекции.
//! - Справа — `createImageBitmap(blob, { imageOrientation: "from-image" })`.
//!
//! На фото с телефона разница видна: слева картинка может «лечь на бок»,
//! справа стоит правильно. Чекбокс переключает применение EXIF.
//! Слайдер масштаба даёт `createImageBitmap` опции resize — загрузка
//! больших фото ускоряется.
//!
//! **Файл автономен.** Стили `.il-*` инжектируются в документ.
//!
//! Wow: загрузите фото с телефона и снимите галочку «Применять EXIF» —
//! картинка справа повернётся на 90° в реальном времени.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use image::{ImageFormat, Rgb, RgbImage};
use js_sys::{Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Blob, BlobPropertyBag, CanvasRenderingContext2d, Document, Event,
    HtmlCanvasElement, HtmlElement, HtmlImageElement, HtmlInputElement,
    ImageBitmap, Node, Url,
};

// =============================================================================
// Утилита обхода бага rust-analyzer: `format!` в tail-позиции теряет тип
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "il-injected-style";
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
.il-desc {
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
.il-desc-line { margin: 6px 0; }

.il-controls {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
}
.il-file-input { display: none; }
.il-file-label {
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
.il-file-label:hover { border-color: #58a6ff; }
.il-file-icon { font-size: 16px; }

.il-btn {
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
.il-btn:hover { background: #30363d; }
.il-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.il-btn-primary:hover { background: #2ea043; }

.il-check-label {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 12px;
    color: #c9d1d9;
    cursor: pointer;
}
.il-checkbox {
    accent-color: #58a6ff;
    width: 16px;
    height: 16px;
    cursor: pointer;
}

.il-slider-row {
    display: flex;
    align-items: center;
    gap: 8px;
}
.il-slider-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
}
.il-slider {
    accent-color: #58a6ff;
    width: 160px;
}
.il-value {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #79c0ff;
    min-width: 44px;
}

.il-panels {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    margin-bottom: 16px;
}
@media (max-width: 800px) {
    .il-panels { grid-template-columns: 1fr; }
}
.il-panel {
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 8px;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}
.il-panel-head {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    gap: 12px;
}
.il-panel-title {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    font-weight: 600;
    color: #c9d1d9;
}
.il-panel-time {
    font-family: ui-monospace, monospace;
    font-size: 11px;
    color: #3fb950;
}
.il-canvas {
    display: block;
    width: 100%;
    height: 260px;
    background: #0d1117;
    border-radius: 4px;
}

.il-status {
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
.il-status-error {
    border-left-color: #f85149;
    color: #f85149;
}

.il-footnote {
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
// Загрузка
// =============================================================================

#[derive(Clone, Copy)]
struct BitmapOptions {
    apply_exif: bool,
    max_dim: Option<u32>,
    quality: &'static str,
}

impl Default for BitmapOptions {
    fn default() -> Self {
        Self {
            apply_exif: true,
            max_dim: None,
            quality: "high",
        }
    }
}

async fn load_as_image(blob: &Blob) -> Result<HtmlImageElement, JsValue> {
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

async fn load_as_bitmap(
    blob: &Blob,
    options: BitmapOptions,
) -> Result<ImageBitmap, JsValue> {
    let global = js_sys::global();
    let func_value = Reflect::get(&global, &JsValue::from_str("createImageBitmap"))?;
    if !func_value.is_function() {
        return Err(JsValue::from_str(
            "createImageBitmap not available in this browser",
        ));
    }
    let func: js_sys::Function = func_value.dyn_into()?;

    let opts = Object::new();

    if options.apply_exif {
        Reflect::set(
            &opts,
            &JsValue::from_str("imageOrientation"),
            &JsValue::from_str("from-image"),
        )?;
    }

    if let Some(max_dim) = options.max_dim {
        Reflect::set(
            &opts,
            &JsValue::from_str("resizeWidth"),
            &JsValue::from_f64(max_dim as f64),
        )?;
        Reflect::set(
            &opts,
            &JsValue::from_str("resizeHeight"),
            &JsValue::from_f64(max_dim as f64),
        )?;
        Reflect::set(
            &opts,
            &JsValue::from_str("resizeQuality"),
            &JsValue::from_str(options.quality),
        )?;
    }

    let promise: js_sys::Promise =
        func.call2(&global, blob.as_ref(), &opts)?.dyn_into()?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value.dyn_into::<ImageBitmap>()
}

// =============================================================================
// Рисование
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

fn draw_image_fit(
    ctx: &CanvasRenderingContext2d,
    src_w: u32,
    src_h: u32,
    dst_w: f64,
    dst_h: f64,
) -> (f64, f64, f64, f64) {
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, dst_w, dst_h);

    if src_w == 0 || src_h == 0 {
        return (0.0, 0.0, 0.0, 0.0);
    }
    let sw = src_w as f64;
    let sh = src_h as f64;
    let scale = (dst_w / sw).min(dst_h / sh);
    let dw = sw * scale;
    let dh = sh * scale;
    let dx = (dst_w - dw) * 0.5;
    let dy = (dst_h - dh) * 0.5;
    (dx, dy, dw, dh)
}

fn draw_html_image(
    canvas: &HtmlCanvasElement,
    img: &HtmlImageElement,
) -> Result<(), JsValue> {
    let ctx = get_2d_context(canvas)?;
    let (dx, dy, dw, dh) = draw_image_fit(
        &ctx,
        img.natural_width(),
        img.natural_height(),
        canvas.width() as f64,
        canvas.height() as f64,
    );
    if dw > 0.0 && dh > 0.0 {
        ctx.draw_image_with_html_image_element_and_dw_and_dh(img, dx, dy, dw, dh)?;
    }
    Ok(())
}

fn draw_image_bitmap(
    canvas: &HtmlCanvasElement,
    bitmap: &ImageBitmap,
) -> Result<(), JsValue> {
    let ctx = get_2d_context(canvas)?;
    let (dx, dy, dw, dh) = draw_image_fit(
        &ctx,
        bitmap.width(),
        bitmap.height(),
        canvas.width() as f64,
        canvas.height() as f64,
    );
    if dw > 0.0 && dh > 0.0 {
        ctx.draw_image_with_image_bitmap_and_dw_and_dh(bitmap, dx, dy, dw, dh)?;
    }
    Ok(())
}

// =============================================================================
// Встроенный пример
// =============================================================================

/// Генерирует демонстрационный образ 300×200 в рантайме.
///
/// Синтетический PNG — без EXIF-ориентации (PNG её не поддерживает).
/// Кнопка «Пример» показывает работу пайплайна, но **не** разницу
/// «с EXIF / без EXIF». Чтобы увидеть EXIF-эффект, загрузите настоящее
/// фото с телефона (JPEG, снятое в портретном режиме).
///
/// Картинка содержит светлую горизонтальную полосу сверху — чтобы
/// при повороте было видно, где «верх».
fn make_sample_image() -> Blob {
    let (w, h) = (300u32, 200u32);
    let mut img = RgbImage::new(w, h);

    // Градиентный фон.
    for (x, y, px) in img.enumerate_pixels_mut() {
        let fx = x as f32 / w as f32;
        let fy = y as f32 / h as f32;
        let r = (60.0 + fx * 150.0) as u8;
        let g = (40.0 + fy * 150.0) as u8;
        let b = (200.0 - (fx + fy) * 80.0).max(40.0) as u8;
        *px = Rgb([r, g, b]);
    }

    // Светлая полоса сверху — «верх» картинки.
    for x in 0..w {
        for y in 0..10 {
            img.put_pixel(x, y, Rgb([240, 240, 240]));
        }
    }

    let mut cursor = std::io::Cursor::new(Vec::<u8>::new());
    img.write_to(&mut cursor, ImageFormat::Png)
        .expect("PNG encode");
    let bytes = cursor.into_inner();

    let u8arr = js_sys::Uint8Array::from(bytes.as_slice());
    let parts = js_sys::Array::new();
    parts.push(u8arr.as_ref());
    let bag = BlobPropertyBag::new();
    bag.set_type("image/png");
    Blob::new_with_u8_array_sequence_and_options(&parts, &bag)
        .expect("Blob creation should not fail")
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Один файл — два способа получить пиксели. Слева <code>&lt;img&gt;</code> \
     через <code>decode()</code>, справа <code>createImageBitmap</code>. \
     <code>decode()</code> резолвится, когда картинка полностью декодирована — \
     рисование не заблокирует main thread. <code>createImageBitmap</code> \
     быстрее и умеет resize прямо при загрузке.",
    "EXIF-ориентация — не автомат. Фото с телефона приходит с метаданными \
     «повернуть на 90°». Ни <code>&lt;img&gt;</code>, ни \
     <code>createImageBitmap</code> без опции <code>imageOrientation: \"from-image\"</code> \
     их не применяют для canvas-отрисовки. Чекбокс ниже переключает поведение \
     правого холста.",
    "Слайдер «Масштаб» даёт <code>createImageBitmap</code> опции \
     <code>resizeWidth</code> / <code>resizeHeight</code>. На больших файлах \
     это ускоряет загрузку: браузер декодирует сразу в нужный размер, \
     не аллоцируя полный буфер.",
];

const CANVAS_W: u32 = 400;
const CANVAS_H: u32 = 260;

struct DemoState {
    blob: Option<Blob>,
    file_name: String,
    file_size: u64,
    apply_exif: bool,
    max_dim: u32,
    last_img_ms: f64,
    last_bitmap_ms: f64,
}

impl Default for DemoState {
    fn default() -> Self {
        Self {
            blob: None,
            file_name: String::new(),
            file_size: 0,
            apply_exif: true,
            max_dim: 0, // 0 = без масштабирования
            last_img_ms: 0.0,
            last_bitmap_ms: 0.0,
        }
    }
}

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("21.5 — Загрузка изображений: img, ImageBitmap, EXIF"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Панель управления.
    let controls = document.create_element("div")?;
    controls.set_class_name("il-controls");
    controls.set_inner_html(
        r#"<input type="file" id="il-file" class="il-file-input" accept="image/*">
           <label for="il-file" class="il-file-label">
               <span class="il-file-icon">🖼</span>
               <span id="il-file-name">Выбрать файл…</span>
           </label>
           <button class="il-btn il-btn-primary" id="il-sample">Пример</button>
           <label class="il-check-label">
               <input type="checkbox" id="il-exif" class="il-checkbox" checked>
               применять EXIF-ориентацию (правый холст)
           </label>
           <div class="il-slider-row">
               <span class="il-slider-label">Масштаб:</span>
               <input type="range" id="il-scale" class="il-slider"
                      min="0" max="4" step="1" value="0">
               <span id="il-scale-val" class="il-value">100%</span>
           </div>"#,
    );
    root.append_child(&controls)?;

    // Статус.
    let status = document.create_element("div")?;
    status.set_class_name("il-status");
    status.set_id("il-status");
    status.set_text_content(Some("Выберите файл или нажмите «Пример»"));
    root.append_child(&status)?;

    // Два холста. Размеры задаются из констант CANVAS_W / CANVAS_H —
    // единый источник истины для разметки и кода.
    let w_str = CANVAS_W.to_string();
    let h_str = CANVAS_H.to_string();
    let panels = document.create_element("div")?;
    panels.set_class_name("il-panels");
    let panels_html = format_str(format_args!(
        r#"<div class="il-panel">
               <div class="il-panel-head">
                   <span class="il-panel-title">&lt;img&gt;.decode()</span>
                   <span class="il-panel-time" id="il-time-img">—</span>
               </div>
               <canvas id="il-canvas-img" class="il-canvas"
                       width="{w_str}" height="{h_str}"></canvas>
           </div>
           <div class="il-panel">
               <div class="il-panel-head">
                   <span class="il-panel-title">createImageBitmap</span>
                   <span class="il-panel-time" id="il-time-bitmap">—</span>
               </div>
               <canvas id="il-canvas-bitmap" class="il-canvas"
                       width="{w_str}" height="{h_str}"></canvas>
           </div>"#
    ));
    panels.set_inner_html(&panels_html);
    root.append_child(&panels)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("il-footnote");
    footnote.set_text_content(Some(
        "Встроенный пример — синтетический PNG без EXIF-метаданных, поэтому \
         переключатель EXIF на нём ничего не меняет. Чтобы увидеть разницу, \
         загрузите фотографию с телефона, снятую в портретном режиме: такие \
         снимки хранят ориентацию в EXIF и без опции imageOrientation: \
         \"from-image\" отображаются на canvas повёрнутыми на 90°.",
    ));
    root.append_child(&footnote)?;

    // Начальное состояние.
    let state = Rc::new(RefCell::new(DemoState::default()));

    // Обработчики.
    wire_file_input(&document, state.clone())?;
    wire_sample_button(&document, state.clone())?;
    wire_exif_checkbox(&document, state.clone())?;
    wire_scale_slider(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_file_input(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let input = get_input(document, "il-file")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(input) = doc.get_element_by_id("il-file") else {
            return;
        };
        let Ok(input) = input.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Some(files) = input.files() else { return };
        let Some(file) = files.get(0) else { return };

        let name = file.name();
        let size = file.size() as u64;
        if let Some(el) = doc.get_element_by_id("il-file-name") {
            el.set_text_content(Some(&name));
        }

        let blob: Blob = file.into();
        {
            let mut st = state.borrow_mut();
            st.blob = Some(blob.clone());
            st.file_name = name;
            st.file_size = size;
        }
        load_and_render(&doc, state.clone());
    });
    input.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_sample_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("il-sample")
        .ok_or_else(|| JsValue::from_str("no il-sample"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        let blob = make_sample_image();
        let size = blob.size() as u64;
        {
            let mut st = state.borrow_mut();
            st.blob = Some(blob);
            st.file_name = String::from("sample.png");
            st.file_size = size;
        }
        if let Some(el) = doc.get_element_by_id("il-file-name") {
            el.set_text_content(Some("Встроенный пример"));
        }
        load_and_render(&doc, state.clone());
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_exif_checkbox(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = get_input(document, "il-exif")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("il-exif") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        state.borrow_mut().apply_exif = s.checked();
        if state.borrow().blob.is_some() {
            load_and_render(&doc, state.clone());
        }
    });
    el.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_scale_slider(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = get_input(document, "il-scale")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("il-scale") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(idx) = s.value().parse::<usize>() else {
            return;
        };
        let max_dim = match idx {
            0 => 0,
            1 => 1024,
            2 => 800,
            3 => 512,
            _ => 400,
        };
        {
            let mut st = state.borrow_mut();
            st.max_dim = max_dim;
        }
        let label = match idx {
            0 => "100%",
            1 => "≤1024",
            2 => "≤800",
            3 => "≤512",
            _ => "≤400",
        };
        if let Some(el) = doc.get_element_by_id("il-scale-val") {
            el.set_text_content(Some(label));
        }
        if state.borrow().blob.is_some() {
            load_and_render(&doc, state.clone());
        }
    });
    el.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Загрузка и отрисовка ----------

fn load_and_render(document: &Document, state: Rc<RefCell<DemoState>>) {
    let doc = document.clone();
    let state_inner = state.clone();
    wasm_bindgen_futures::spawn_local(async move {
        let (blob, apply_exif, max_dim) = {
            let st = state_inner.borrow();
            let Some(blob) = st.blob.clone() else { return };
            (blob, st.apply_exif, st.max_dim)
        };

        // --- Левая панель: <img> ---
        let t0 = js_sys::Date::now();
        let img_result = load_as_image(&blob).await;
        let img_ms = js_sys::Date::now() - t0;

        // --- Правая панель: createImageBitmap ---
        let options = BitmapOptions {
            apply_exif,
            max_dim: if max_dim > 0 { Some(max_dim) } else { None },
            quality: "high",
        };
        let t1 = js_sys::Date::now();
        let bitmap_result = load_as_bitmap(&blob, options).await;
        let bitmap_ms = js_sys::Date::now() - t1;

        // --- Отрисовка левой панели ---
        match img_result {
            Ok(img) => {
                if let Some(canvas_el) = doc.get_element_by_id("il-canvas-img") {
                    if let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() {
                        let _ = draw_html_image(&canvas, &img);
                    }
                }
                if let Some(el) = doc.get_element_by_id("il-time-img") {
                    el.set_text_content(Some(&format_str(format_args!(
                        "{img_ms:.1} мс"
                    ))));
                }
                state_inner.borrow_mut().last_img_ms = img_ms;
            }
            Err(e) => {
                let m = js_err_to_string(&e);
                set_status_error(&doc, &format_str(format_args!("img error: {m}")));
            }
        }

        // --- Отрисовка правой панели ---
        match bitmap_result {
            Ok(bitmap) => {
                if let Some(canvas_el) = doc.get_element_by_id("il-canvas-bitmap") {
                    if let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() {
                        let _ = draw_image_bitmap(&canvas, &bitmap);
                    }
                }
                if let Some(el) = doc.get_element_by_id("il-time-bitmap") {
                    el.set_text_content(Some(&format_str(format_args!(
                        "{bitmap_ms:.1} мс"
                    ))));
                }
                // Явно закрываем bitmap — освобождаем память.
                bitmap.close();
                state_inner.borrow_mut().last_bitmap_ms = bitmap_ms;
            }
            Err(e) => {
                let m = js_err_to_string(&e);
                set_status_error(&doc, &format_str(format_args!("bitmap error: {m}")));
            }
        }

        // --- Итоговый статус ---
        let st = state_inner.borrow();
        if let Some(el) = doc.get_element_by_id("il-status") {
            let size_str: String = if st.file_size > 0 {
                format_str(format_args!("{} КБ", st.file_size / 1024))
            } else {
                String::from("—")
            };
            let exif_str = if st.apply_exif { "вкл" } else { "выкл" };
            let max_dim_str: String = if st.max_dim > 0 {
                format_str(format_args!("≤{}", st.max_dim))
            } else {
                String::from("нет")
            };
            let text = format_str(format_args!(
                "{} · {} · EXIF: {} · resize: {} · img: {:.1} мс · bitmap: {:.1} мс",
                st.file_name, size_str, exif_str, max_dim_str,
                st.last_img_ms, st.last_bitmap_ms,
            ));
            el.set_text_content(Some(&text));
            if let Some(h) = el.dyn_ref::<HtmlElement>() {
                h.set_class_name("il-status");
            }
        }
    });
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

fn set_status_error(document: &Document, msg: &str) {
    if let Some(el) = document.get_element_by_id("il-status") {
        el.set_text_content(Some(msg));
        if let Some(h) = el.dyn_ref::<HtmlElement>() {
            h.set_class_name("il-status il-status-error");
        }
    }
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("il-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("il-desc-line");
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
```

---

## 21.5.11. Команды запуска

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_canvas_image_loading
npm test
npm run wasm:dev && npm start
```

## 21.5.12. Проверка корректности

**Тесты (5 passed):**
- `img_decode_resolves_and_has_size` — `<img>.decode()` резолвится, размеры 4×4;
- `bitmap_loads_with_correct_size` — `createImageBitmap` даёт 4×4;
- `bitmap_with_resize` — `resizeWidth: 16` даёт 16×16;
- `bitmap_with_exif_option_does_not_panic` — `imageOrientation: "from-image"` не падает на PNG;
- `bitmap_close_is_safe` — идемпотентный `close()`.

**Исправления этой итерации:**
1. **Источник PNG в тестах** — генерируется в рантайме через `image` crate (4×4). Ручной 1×1 PNG отвергался `createImageBitmap` на Chrome с `InvalidStateError`.
2. **`bitmap.close()` в каждом тесте** — освобождает память, правило проекта.
3. **`make_sample_image` в `app.rs`** — синтетический PNG 300×200 вместо встроенных байтов JPEG. Работает надёжно, генерируется `image` crate.
4. **Честная оговорка в footnote** — синтетический PNG не содержит EXIF, для демонстрации нужно загрузить фото с телефона.
5. **`CANVAS_W` / `CANVAS_H` используются в HTML-шаблоне** через `format_str(format_args!(...))` — единый источник истины, нет предупреждения `dead_code`.
6. **`Event` убран из импортов** — не используется.

**Правила проекта применены:**
- `format_str(format_args!(...))` в tail-позициях;
- `is_function()` перед `Function::call` — понятная ошибка вместо `TypeError`;
- `bitmap.close()` явно — не полагаемся на GC;
- `RefMut` в узком scope — не пересекается с `borrow()`.

## 21.5.13. Итог

Загрузка изображений — это три пути с разными возможностями. `<img>.decode()` — надёжно и универсально. `createImageBitmap` — быстрее, умнее, поддерживает EXIF и resize. `OffscreenCanvas` — для пайплайнов.

Демо-инспектор показывает обе стороны на одном экране. Одна картинка, два холста, два тайминга. Синтетический пример даёт базовое представление о пайплайне; для демонстрации EXIF-эффекта нужно загрузить реальное фото с телефона — что честно сказано в footnote.

Код модуля автономен, тесты покрывают ключевые сценарии (декодирование, resize, EXIF, close). Демо инжектирует стили `.il-*` в документ — правок в `main.css` не требуется.

**Правило проекта, зафиксированное в этой подглаве:** тестовые изображения генерируются **в рантайме** через `image` crate, а не встраиваются как байтовые константы. Ручные 1×1 PNG могут отвергаться `createImageBitmap` на разных браузерах из-за edge case декодеров — генерация через проверенную библиотеку устраняет эту хрупкость.

Следующая подглава **21.6** — про фильтры на Rust: grayscale, invert, blur, sobel. С SIMD и Web Worker. Всё, что здесь декодировали, будем обрабатывать численно. **21.7** — практический проект, где загрузка, обработка и скачивание собираются вместе.