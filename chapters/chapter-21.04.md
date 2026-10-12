# Глава 21.4. ImageData: пиксельный буфер и его границы

## 21.4.0. Обновлённая структура

```
src/
  canvas_imagedata.rs         ← НОВЫЙ модуль главы 21.4
  canvas_dpi.rs               ← из 21.1 (не трогаем)
  canvas_drawing.rs           ← из 21.2 (не трогаем)
  canvas_composition.rs       ← из 21.3 (не трогаем)
tests/
  canvas_imagedata_pure.rs    ← host-тесты чистой логики
  wasm_canvas_imagedata.rs    ← wasm-тесты работы с холстом
src/lib.rs                    ← +1 строка
src/app.rs                    ← демо текущей подглавы (заменяется целиком)
Cargo.toml                    ← +1 фича (DomRect)
web/styles/main.css           ← правок не требуется
```

**Комментарий.** Модуль `canvas_imagedata.rs` — про сырой доступ к пикселям: индекс `(y*w + x)*4 + channel`, чтение/запись отдельных каналов, разница между `putImageData` и обычными методами рисования. Префикс CSS — `px-`, уникален в проекте (заняты `dpi-`, `dr-`, `co-`, `bp-`, `dl-`, `ci-`, `il-`). Стили инжектируются в документ из `app.rs`, `main.css` не правится.

## Ссылки

- MDN: [`ImageData`](https://developer.mozilla.org/en-US/docs/Web/API/ImageData)
- MDN: [`Uint8ClampedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray)
- MDN: [`getImageData`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/getImageData)
- MDN: [`putImageData`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/putImageData)
- WHATWG Spec: [ImageData](https://html.spec.whatwg.org/multipage/canvas.html#imagedata)
- web.dev: [Learn HTML canvas — pixel manipulation](https://web.dev/learn/canvas/imagedata)

---

## 21.4.1. О чём эта подглава

Три предыдущие подглавы работали с canvas как с **поверхностью рисования**: настроили размеры (21.1), научились рисовать фигуры (21.2), управлять состоянием и композицией (21.3). Всё это время мы смотрели на холст **снаружи** — как на экран, на который наносится краска.

Теперь заглянем **внутрь**. Canvas — это не «магический ящик с картинкой», а **массив байтов**. Каждый пиксель — четыре байта: красный, зелёный, синий, альфа. Этот массив доступен для чтения и записи через `ImageData`. Именно здесь начинается территория Rust: как только пиксели становятся плоским `Vec<u8>`, численная обработка ускоряется в разы.

Подглава — про **данные**: как выглядит пиксельный буфер, как его читать, как писать, и что именно `putImageData` делает с холстом (спойлер: не то, что вы думаете).

**Ключевые темы:**
- Что такое `ImageData`: `width`, `height`, `data: Clamped<Vec<u8>>`;
- Формат RGBA, 4 байта на пиксель;
- `Uint8ClampedArray` vs `Uint8Array` — зачем именно clamped;
- `getImageData(sx, sy, sw, sh)` — читаем прямоугольник;
- `putImageData(data, dx, dy)` — пишем;
- **Прямое чтение и запись по индексу**: `data[i]`, где `i = (y*w + x)*4 + channel`;
- Почему `putImageData` игнорирует трансформации, `globalAlpha` и композицию;
- `createImageData(w, h)` vs `new ImageData(data, w, h)` — разница.

## 21.4.2. Что уже есть и что появится

К концу 21.3 мы умеем:
- рисовать фигуры, текст, градиенты;
- управлять состоянием (save/restore), трансформациями, композицией;
- смешивать слои, обрезать по clip.

К концу 21.4 будем уметь:
- читать пиксели с холста в `ImageData`;
- получать плоский `Vec<u8>` из `ImageData`;
- модифицировать **отдельные каналы** (R, G, B, A) напрямую;
- писать модифицированный буфер обратно на холст;
- понимать, почему `putImageData` ведёт себя **не как остальные методы рисования**.

## 21.4.3. Формат: RGBA, 4 байта, плоский массив

`ImageData` — это объект с тремя полями:
- `width: u32` — сколько пикселей по горизонтали;
- `height: u32` — сколько по вертикали;
- `data: Clamped<Vec<u8>>` — плоский RGBA-буфер, `width × height × 4` байт.

Пиксель `(x, y)` занимает четыре байта по индексу:

```
i = (y * width + x) * 4
```

`data[i]` — красный (R), `data[i+1]` — зелёный (G), `data[i+2]` — синий (B), `data[i+3]` — альфа (A). Каждый канал — `u8`, от 0 до 255.

**Никакой структуры.** Не `struct Pixel { r, g, b, a }`. Плоский массив, где пиксели идут построчно. Именно поэтому его так удобно передавать в Rust: `Vec<u8>` — это тот же кусок памяти, с которым работает любой алгоритм обработки изображений.

**Порядок RGBA.** В памяти `R` идёт первым, `A` — последним. В BGRA (который используется, например, в `canvas2d` некоторых браузеров) — наоборот. `ImageData` всегда RGBA.

**Clamped, а не обычный `Uint8Array`.** Разница существенна: `Uint8ClampedArray` при переполнении **насыщает** значение: `256 → 255`, `-1 → 0`. Обычный `Uint8Array` **заворачивает** по модулю 256: `256 → 0`, `-1 → 255`. Для цвета насыщение — то, что нужно: пиксель не может быть «ярче белого». В Rust мы работаем с `u8`, а клампинг делаем вручную через `.min(255)` или `.clamp(0, 255)`.

## 21.4.4. Чтение: getImageData

```js
const data = ctx.getImageData(x, y, w, h);
```

Возвращает `ImageData` с прямоугольником `(x, y, w, h)`. Координаты — в пикселях буфера, не в CSS. Аргументы в web-sys 0.3.105 **зависят от cfg-флага** `web_sys_unstable_apis`: `f64` в стабильном режиме, `i32` в unstable. Различие скрывается в хелпере `read_image_data_cfg` — остальной код вызывает одинаково.

`getImageData` — **дорогая операция**. Читает область из буфера холста в отдельный `ImageData`. Для больших областей это копирование `width × height × 4` байт. Поэтому его не стоит вызывать в цикле отрисовки — только когда действительно нужны пиксели.

## 21.4.5. Запись: putImageData

```js
ctx.putImageData(data, dx, dy);
```

Пишет `ImageData` в буфер холста, начиная с `(dx, dy)`. **Ключевая деталь:** `putImageData` — это не «рисование». Это **сырая запись байтов** в буфер холста. Он **игнорирует**:

- **трансформации** (`translate`, `scale`, `rotate`);
- **`globalAlpha`**;
- **`globalCompositeOperation`**;
- **clip**.

То есть если вы поставили `ctx.scale(2, 2)` и потом вызвали `putImageData(data, 0, 0)`, пиксели лягут в холст **1:1**, без масштабирования. Для сравнения, `fillRect`, `drawImage`, `fillText` — все подчиняются этим настройкам.

**Почему так.** `putImageData` работает **на уровне буфера**: «вот байты — положи их туда». Он не проходит через конвейер рисования (compositing pipeline). Это документированное поведение спецификации. Понимание этого — половина главы.

Демо показывает это наглядно: правый холст — с `scale(2, 2)`, где `fillRect` растягивается вдвое, а `putImageData` — нет.

## 21.4.6. Прямое чтение и запись по индексу

Самое интересное. Как только `ImageData` получен, мы работаем с ним как с плоским массивом `Vec<u8>`:

```rust
fn pixel_index(x: u32, y: u32, w: u32, channel: u8) -> usize {
    let ch = (channel as u32).min(3); // защита от опечаток: 0..=3
    ((y * w + x) * 4 + ch) as usize
}
```

Индекс канала в плоском буфере. `channel`: 0 — R, 1 — G, 2 — B, 3 — A.

**Зачем это нужно.** Можно модифицировать отдельные каналы:

```rust
// Инвертировать только красный канал
let mut i = 0;
while i < buf.len() {
    buf[i] = 255 - buf[i];
    i += 4;
}
```

Результат: красные квадраты становятся cyan (потому что R изменился с 240 на 15), зелёные — фиолетовыми, синие — жёлто-оранжевыми. Логика «красный стал cyan» — прямое следствие изменения одного байта.

**Ускорение.** `while i < buf.len() { buf[i] = ...; i += 4 }` — шаг 4 по плоскому массиву. Rust-компилятор это оптимизирует: цикл разворачивается, шаг становится константой. Для буфера 800×600 = 1.92 МБ это ~0.5 мс на современном ноутбуке. JavaScript на том же цикле тратит в разы больше — там типы не такие статичные, компилятор менее агрессивный.

## 21.4.7. `createImageData` vs `new ImageData`

Два способа создать `ImageData`:

**`ctx.createImageData(w, h)`** — новый пустой буфер заданного размера. Все байты — нули, включая альфу. Пиксели прозрачные.

**`new ImageData(data, w, h)`** — оборачивает **существующий** `Uint8ClampedArray`. Не копирует данные. В Rust это `ImageData::new_with_u8_clamped_array_and_sh(Clamped(pixels), w, h)`.

Разница важна, когда вы работаете с несколькими промежуточными буферами. Первый — «создай мне холст под рисование», второй — «вот мои пиксели, сделай из них `ImageData` для `putImageData`».

В демо используется только второй: `Vec<u8>` из Rust → `ImageData` → `putImageData`.

## 21.4.8. Что показывает демо

Демо — **«пипетка + каналы»**. Один вход (паттерн) и много способов его модифицировать.

**Паттерн.** Плавный градиент с тремя цветными квадратами по углам: красный, зелёный, синий. Плюс плавный переход между ними. Цель — чтобы клик в разных точках давал **разные** RGBA.

**Левый холст — «пипетка».** Обычный canvas без трансформаций. При клике в точке `(x, y)`:

1. Переводим координаты экрана в координаты буфера (через `getBoundingClientRect`).
2. Читаем RGBA из пиксельного буфера.
3. Показываем в статусе: `(x, y) → RGBA(R, G, B, A) · индекс = (y*W + x)*4 = N`.

Пользователь видит не только цвет, но и **число** — конкретный индекс в плоском массиве. Формула из учебника становится осязаемой.

**Кнопки каналов.** Пять кнопок, каждая модифицирует один канал:

- **«Сбросить»** — восстановить исходный паттерн.
- **«Инвертировать R»** — `R → 255 - R` для всех пикселей.
- **«Инвертировать G»** — то же для зелёного.
- **«Инвертировать B»** — то же для синего.
- **«Обнулить A»** — `A → 0` для всех пикселей.

Каждое нажатие — модификация плоского `Vec<u8>`, затем `putImageData` на холст. Слева сразу видно результат: инвертировать R — красные квадраты становятся cyan, инвертировать G — зелёные становятся пурпурными. Логика цвета ломается, а математика остаётся простой: `255 - x` для одного байта из четырёх.

**Правый холст — «putImageData игнорирует scale».** Тот же паттерн, но перед рисованием применяется `ctx.scale(2, 2)`. После этого:

1. `putImageData(pattern, 0, 0)` — кладёт паттерн **1:1**. 400×240 → 400×240 пикселей буфера.
2. `fillRect(0, 0, 20, 20)` — красный маркер **растянулся вдвое**. 20×20 CSS → 40×40 физических.

Одна картинка, два разных поведения. `fillRect` подчиняется трансформации, `putImageData` — нет.

## 21.4.9. Wow-момент

Три клика показывают три концепции.

**Первый.** Кликните по левому холсту в разных точках — в статусе видны разные RGBA и **разные индексы**. Красный квадрат в углу — `(240, 60, 60, 255)`, серый градиент в центре — `(120, 120, 150, 255)`, зелёный квадрат — `(60, 220, 80, 255)`. Формула `(y*w + x)*4` становится живой.

**Второй.** Нажмите «Инвертировать R» три раза подряд. Первое нажатие: красные квадраты стали cyan. Второе: cyan стали красными (инверсия инверсии = тождество). Третье: снова cyan. Это не «красный стал cyan», это **байт R поменялся по формуле `255 - x`**, трижды. Так видно, что цвет — это число.

**Третий.** Переключитесь на правый холст. Паттерн лежит ровно, не растянутый. А в левом верхнем углу — красный маркер **вдвое больше**. `scale(2, 2)` работает для `fillRect`, но не для `putImageData`. Один холст, два разных поведения, одна причина — разные уровни API.

## 21.4.10. Чему это учит

**Первое.** `ImageData` — это **данные**, не «картинка». Плоский `Vec<u8>`, RGBA, построчно. `width × height × 4` байт. Индекс канала: `(y*w + x)*4 + channel`. Это тот же формат, в котором работают PNG-декодеры, JPEG-энкодеры, все алгоритмы обработки изображений.

**Второе.** `Uint8ClampedArray` **насыщает**, `Uint8Array` **заворачивает**. Для цвета нужно первое: 256 → 255, а не 256 → 0.

**Третье.** `putImageData` — **не рисование**. Это запись в буфер. Игнорирует трансформации, alpha, композицию и clip. Если нужна картинка, которая подчиняется этим настройкам — используйте `drawImage` с `ImageBitmap` (тема 21.5).

**Четвёртое.** Чтение пикселя — это **индексация**, а не «получить цвет в точке». `data[i]`, `data[i+1]`, `data[i+2]`, `data[i+3]` — четыре байта, четыре канала.

**Пятое.** `getImageData` и `putImageData` — **дорогие операции**. Первая копирует пиксели из буфера холста, вторая — в буфер. Для 800×600 это 1.92 МБ на каждый вызов. Вызывайте их не в цикле отрисовки, а когда действительно нужна работа с пикселями.

## 21.4.11. Ссылки

- MDN: [`ImageData`](https://developer.mozilla.org/en-US/docs/Web/API/ImageData)
- MDN: [`Uint8ClampedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray)
- MDN: [`getImageData`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/getImageData)
- MDN: [`putImageData`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/putImageData)
- WHATWG Spec: [ImageData](https://html.spec.whatwg.org/multipage/canvas.html#imagedata)
- web.dev: [Learn HTML canvas — pixel manipulation](https://web.dev/learn/canvas/imagedata)

---

## 21.4.12. Исходные файлы

### `src/canvas_imagedata.rs`

```rust
//! 21.4. ImageData: пиксельный буфер и его границы.
//!
//! `ImageData` — плоский RGBA-буфер: `width × height × 4` байт.
//! Каждый пиксель — четыре байта в порядке `R, G, B, A`. Индекс
//! конкретного канала: `(y * w + x) * 4 + channel`, где `channel`:
//! 0 — R, 1 — G, 2 — B, 3 — A.
//!
//! В web-sys `ImageData::data()` возвращает `Clamped<Vec<u8>>` —
//! обёртку над `Vec<u8>`, которую можно взять через `.0`.
//!
//! ## Почему clamped
//!
//! `Uint8ClampedArray` (в отличие от `Uint8Array`) при переполнении
//! **насыщает** значение: `256 → 255`, `-1 → 0`. Не заворачивает по
//! модулю 256, а клампит. Это соответствует семантике цвета: пиксель
//! не может быть «ярче белого». В Rust мы работаем с `u8`, а
//! клампинг делаем вручную через `.clamp(0, 255)` или `.min(255)`.
//!
//! ## getImageData / putImageData
//!
//! Обе функции имеют **две сигнатуры**, переключаемые cfg-флагом
//! `web_sys_unstable_apis`: `f64` в стабильном режиме, `i32` в unstable.
//! Различие инкапсулировано в хелперах `read_image_data_cfg` и
//! `write_image_data_cfg` — остальной код их вызывает одинаково.
//!
//! ## putImageData игнорирует трансформации
//!
//! В отличие от `drawImage` и всех остальных методов рисования,
//! `putImageData` пишет **сырые пиксели** в буфер холста напрямую.
//! Он не смотрит ни на `translate`/`scale`/`rotate`, ни на
//! `globalAlpha`, ни на `globalCompositeOperation`. Это работает на
//! уровне «вот эти байты — туда», без всякой математики. Демо
//! показывает это через `scale(2, 2)`: `fillRect` увеличивается,
//! `putImageData` — нет.

use wasm_bindgen::prelude::*;
use wasm_bindgen::Clamped;
use web_sys::{CanvasRenderingContext2d, ImageData};

// =============================================================================
// Индексация и чтение
// =============================================================================

/// Индекс байта канала в плоском RGBA-буфере.
///
/// `channel`: 0 — R, 1 — G, 2 — B, 3 — A.
/// Для `channel > 3` берётся 3 (альфа) — защита от опечаток.
pub fn pixel_index(x: u32, y: u32, w: u32, channel: u8) -> usize {
    let ch = (channel as u32).min(3);
    ((y * w + x) * 4 + ch) as usize
}

/// Читает RGBA в точке `(x, y)`. Возвращает `(r, g, b, a)`.
/// За пределами буфера — `(0, 0, 0, 0)`.
pub fn read_pixel(pixels: &[u8], x: u32, y: u32, w: u32) -> (u8, u8, u8, u8) {
    let i = pixel_index(x, y, w, 0);
    if i + 3 >= pixels.len() {
        return (0, 0, 0, 0);
    }
    (pixels[i], pixels[i + 1], pixels[i + 2], pixels[i + 3])
}

/// Читает один канал. Используется в «пипетке», когда нужен
/// конкретный байт, а не весь пиксель.
pub fn read_channel(pixels: &[u8], x: u32, y: u32, w: u32, channel: u8) -> u8 {
    let i = pixel_index(x, y, w, channel);
    *pixels.get(i).unwrap_or(&0)
}

// =============================================================================
// Преобразования каналов над плоским буфером
// =============================================================================

/// Инвертирует один канал: `v → 255 - v`. Остальные не трогает.
/// Применяется на месте — не выделяет новый `Vec`.
pub fn invert_channel_inplace(pixels: &mut [u8], channel: u8) {
    let ch = (channel as u32).min(3) as usize;
    let mut i = ch;
    while i < pixels.len() {
        pixels[i] = 255 - pixels[i];
        i += 4;
    }
}

/// Обнуляет один канал. Используется для демонстрации «что будет,
/// если R = 0» — цвет смещается в сторону cyan.
pub fn zero_channel_inplace(pixels: &mut [u8], channel: u8) {
    let ch = (channel as u32).min(3) as usize;
    let mut i = ch;
    while i < pixels.len() {
        pixels[i] = 0;
        i += 4;
    }
}

/// Заполняет буфер одним цветом.
pub fn fill_rgba(pixels: &mut [u8], r: u8, g: u8, b: u8, a: u8) {
    for px in pixels.chunks_exact_mut(4) {
        px[0] = r;
        px[1] = g;
        px[2] = b;
        px[3] = a;
    }
}

// =============================================================================
// Генерация тестового паттерна
// =============================================================================

/// Демонстрационный паттерн `w × h`: плавный градиент с тремя
/// цветными квадратами по углам — чтобы «пипетка» показывала
/// разнообразные цвета при клике в разных точках.
pub fn make_pattern(w: u32, h: u32) -> Vec<u8> {
    let mut out = vec![0u8; (w * h * 4) as usize];
    let wf = w as f32;
    let hf = h as f32;
    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / wf;
            let fy = y as f32 / hf;
            let i = pixel_index(x, y, w, 0);

            // База — плавный градиент.
            let mut r = (30.0 + fx * 180.0) as u8;
            let mut g = (30.0 + fy * 180.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            // Красный квадрат в левом верхнем углу.
            if x < w / 5 && y < h / 5 {
                r = 240; g = 60; b = 60;
            }
            // Зелёный — в правом верхнем.
            if x > 4 * w / 5 && y < h / 5 {
                r = 60; g = 220; b = 80;
            }
            // Синий — в правом нижнем.
            if x > 4 * w / 5 && y > 4 * h / 5 {
                r = 60; g = 80; b = 240;
            }

            out[i] = r;
            out[i + 1] = g;
            out[i + 2] = b;
            out[i + 3] = 255;
        }
    }
    out
}

// =============================================================================
// ImageData ↔ Vec<u8>
// =============================================================================

/// `ImageData::data()` в web-sys 0.3.105 возвращает `Clamped<Vec<u8>>` —
/// tuple struct с публичным полем `.0`. Забираем внутренний `Vec<u8>`.
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

// =============================================================================
// Обёртки над getImageData / putImageData (cfg-гейт)
// =============================================================================
//
// В web-sys 0.3.105 методы имеют две сигнатуры:
//   stable  → getImageData(f64, f64, f64, f64); putImageData(_, f64, f64)
//   unstable → getImageData(i32, i32, i32, i32); putImageData(_, i32, i32)
// Хелперы скрывают различие. Остальной код их вызывает одинаково.

#[cfg(web_sys_unstable_apis)]
pub fn read_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    x: u32,
    y: u32,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(x as i32, y as i32, w as i32, h as i32)
}

#[cfg(not(web_sys_unstable_apis))]
pub fn read_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    x: u32,
    y: u32,
    w: u32,
    h: u32,
) -> Result<ImageData, JsValue> {
    ctx.get_image_data(x as f64, y as f64, w as f64, h as f64)
}

#[cfg(web_sys_unstable_apis)]
pub fn write_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
    dx: u32,
    dy: u32,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, dx as i32, dy as i32)
}

#[cfg(not(web_sys_unstable_apis))]
pub fn write_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
    dx: u32,
    dy: u32,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, dx as f64, dy as f64)
}
```

---

### `src/lib.rs` — добавление

```rust
pub mod canvas_dpi;           // 21.1
pub mod canvas_drawing;       // 21.2
pub mod canvas_composition;   // 21.3
pub mod canvas_imagedata;     // ← 21.4
```

---

### `Cargo.toml` — дополнение к `[dependencies.web-sys] features`

```toml
    "DomRect",   # для Element::get_bounding_client_rect в app.rs
```

Также должны быть уже включены (после 21.1–21.3):

```toml
    "CanvasRenderingContext2d",
    "HtmlCanvasElement",
    "HtmlElement",
    "CssStyleDeclaration",
    "ImageData",
    "CanvasGradient",
    "TextMetrics",
    "DomMatrix",
```

---

### `tests/canvas_imagedata_pure.rs`

```rust
//! Host-тесты чистой логики: индексация и преобразования буфера.
//! Без браузера, миллисекунды.

use hello_wasm02::canvas_imagedata::{
    fill_rgba, invert_channel_inplace, make_pattern, pixel_index, read_channel,
    read_pixel, zero_channel_inplace,
};

#[test]
fn pixel_index_first_pixel() {
    assert_eq!(pixel_index(0, 0, 10, 0), 0);
    assert_eq!(pixel_index(0, 0, 10, 1), 1);
    assert_eq!(pixel_index(0, 0, 10, 2), 2);
    assert_eq!(pixel_index(0, 0, 10, 3), 3);
}

#[test]
fn pixel_index_row_step_is_w_times_4() {
    let w = 100;
    let i1 = pixel_index(0, 0, w, 0);
    let i2 = pixel_index(0, 1, w, 0);
    assert_eq!(i2 - i1, 100 * 4);
}

#[test]
fn pixel_index_column_step_is_4() {
    let i1 = pixel_index(0, 0, 50, 0);
    let i2 = pixel_index(1, 0, 50, 0);
    assert_eq!(i2 - i1, 4);
}

#[test]
fn pixel_index_channel_out_of_range_is_clamped() {
    // channel > 3 должен прижиматься к 3 (альфа), а не паниковать.
    assert_eq!(pixel_index(0, 0, 10, 5), pixel_index(0, 0, 10, 3));
    assert_eq!(pixel_index(0, 0, 10, 255), pixel_index(0, 0, 10, 3));
}

#[test]
fn read_pixel_returns_filled_color() {
    let w = 4;
    let h = 4;
    let mut buf = vec![0u8; (w * h * 4) as usize];
    fill_rgba(&mut buf, 100, 150, 200, 255);
    let (r, g, b, a) = read_pixel(&buf, 1, 2, w);
    assert_eq!((r, g, b, a), (100, 150, 200, 255));
}

#[test]
fn read_pixel_out_of_bounds_returns_zero() {
    let buf = vec![0u8; 4];
    let (r, g, b, a) = read_pixel(&buf, 100, 100, 4);
    assert_eq!((r, g, b, a), (0, 0, 0, 0));
}

#[test]
fn read_channel_reads_specific_byte() {
    let buf = vec![10, 20, 30, 40, 50, 60, 70, 80];
    assert_eq!(read_channel(&buf, 0, 0, 2, 0), 10);
    assert_eq!(read_channel(&buf, 0, 0, 2, 1), 20);
    assert_eq!(read_channel(&buf, 1, 0, 2, 0), 50);
    assert_eq!(read_channel(&buf, 1, 0, 2, 3), 80);
}

#[test]
fn invert_channel_inverts_only_that_channel() {
    // 2 пикселя: R=10, R=200
    let mut buf = vec![10, 20, 30, 40, 200, 50, 60, 70];
    invert_channel_inplace(&mut buf, 0); // инвертировать R
    assert_eq!(buf[0], 245); // 255 - 10
    assert_eq!(buf[1], 20); // G не тронут
    assert_eq!(buf[2], 30); // B не тронут
    assert_eq!(buf[3], 40); // A не тронут
    assert_eq!(buf[4], 55); // 255 - 200
    assert_eq!(buf[5], 50); // G не тронут
}

#[test]
fn invert_channel_twice_is_identity() {
    let mut buf = vec![100, 150, 200, 255];
    let original = buf.clone();
    invert_channel_inplace(&mut buf, 1);
    invert_channel_inplace(&mut buf, 1);
    assert_eq!(buf, original);
}

#[test]
fn zero_channel_sets_only_that_channel_to_zero() {
    let mut buf = vec![100, 150, 200, 255, 50, 60, 70, 80];
    zero_channel_inplace(&mut buf, 1); // зануляем G
    assert_eq!(buf[0], 100); // R
    assert_eq!(buf[1], 0); // G = 0
    assert_eq!(buf[2], 200); // B
    assert_eq!(buf[3], 255); // A
    assert_eq!(buf[4], 50); // R
    assert_eq!(buf[5], 0); // G
}

#[test]
fn zero_alpha_makes_every_pixel_transparent() {
    let mut buf = vec![100, 150, 200, 255];
    zero_channel_inplace(&mut buf, 3);
    assert_eq!(buf, vec![100, 150, 200, 0]);
}

#[test]
fn make_pattern_has_correct_size() {
    let p = make_pattern(64, 48);
    assert_eq!(p.len(), 64 * 48 * 4);
}

#[test]
fn make_pattern_all_alpha_255() {
    let p = make_pattern(64, 48);
    for chunk in p.chunks_exact(4) {
        assert_eq!(chunk[3], 255);
    }
}

#[test]
fn make_pattern_corner_red() {
    // В левом верхнем углу — красный.
    let p = make_pattern(100, 100);
    let (r, g, b, _) = read_pixel(&p, 5, 5, 100);
    assert!(r > 200, "R should be high, got {r}");
    assert!(g < 100, "G should be low, got {g}");
    assert!(b < 100, "B should be low, got {b}");
}
```

---

### `tests/wasm_canvas_imagedata.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::{CanvasRenderingContext2d, Document, HtmlCanvasElement};

use hello_wasm02::canvas_imagedata::{
    image_data_to_vec, read_image_data_cfg, read_pixel, vec_to_image_data,
    write_image_data_cfg,
};

wasm_bindgen_test_configure!(run_in_browser);

fn make_canvas(w: u32, h: u32) -> (HtmlCanvasElement, CanvasRenderingContext2d) {
    let doc: Document = web_sys::window().unwrap().document().unwrap();
    let c: HtmlCanvasElement = doc.create_element("canvas").unwrap().dyn_into().unwrap();
    c.set_width(w);
    c.set_height(h);
    let ctx: CanvasRenderingContext2d = c
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into()
        .unwrap();
    (c, ctx)
}

#[wasm_bindgen_test]
fn get_image_data_returns_painted_color() {
    let (_c, ctx) = make_canvas(10, 10);
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(0.0, 0.0, 10.0, 10.0);

    let data = read_image_data_cfg(&ctx, 0, 0, 10, 10).unwrap();
    assert_eq!(data.width(), 10);
    assert_eq!(data.height(), 10);
    let pixels = image_data_to_vec(&data);
    let (r, g, b, a) = read_pixel(&pixels, 5, 5, 10);
    assert_eq!((r, g, b, a), (255, 0, 0, 255));
}

#[wasm_bindgen_test]
fn vec_to_image_data_roundtrip() {
    let w = 4u32;
    let h = 3u32;
    let original: Vec<u8> = (0..w * h * 4).map(|i| (i & 0xFF) as u8).collect();
    let data = vec_to_image_data(&original, w, h).unwrap();
    assert_eq!(data.width(), w);
    assert_eq!(data.height(), h);
    let back = image_data_to_vec(&data);
    assert_eq!(back, original);
}

#[wasm_bindgen_test]
fn vec_to_image_data_rejects_wrong_size() {
    let r = vec_to_image_data(&[0u8; 5], 2, 2);
    assert!(r.is_err());
}

#[wasm_bindgen_test]
fn put_image_data_ignores_scale() {
    // Ключевой тест главы: putImageData не подчиняется трансформациям.
    //
    // Холст 40×40 физических пикселей. При scale(4, 4) в CSS-координатах
    // доступна область 10×10. Это позволяет проверить обе стороны:
    //   - putImageData(data, 0, 0) — остаётся 1×1, не растягивается;
    //   - fillRect в CSS-координатах — растягивается ×4.
    let (_c, ctx) = make_canvas(40, 40);
    ctx.scale(4.0, 4.0).expect("scale");

    // --- Шаг 1: putImageData игнорирует scale ---
    let red = vec_to_image_data(&[255, 0, 0, 255], 1, 1).unwrap();
    write_image_data_cfg(&ctx, &red, 0, 0).unwrap();

    let data = read_image_data_cfg(&ctx, 0, 0, 40, 40).unwrap();
    let pixels = image_data_to_vec(&data);

    let (r0, _, _, _) = read_pixel(&pixels, 0, 0, 40);
    let (r1, _, _, _) = read_pixel(&pixels, 1, 0, 40);
    let (r2, _, _, _) = read_pixel(&pixels, 2, 0, 40);
    let (r3, _, _, _) = read_pixel(&pixels, 3, 0, 40);
    assert_eq!(r0, 255, "pixel (0,0) should be red");
    assert!(r1 < 40, "pixel (1,0) should NOT be red — putImageData ignores scale");
    assert!(r2 < 40, "pixel (2,0) should NOT be red");
    assert!(r3 < 40, "pixel (3,0) should NOT be red");

    // --- Шаг 2: fillRect подчиняется scale ---
    // CSS-координаты (0, 5), размер 1×1. При scale(4, 4) это:
    // физические координаты (0, 20), размер 4×4.
    ctx.set_fill_style_str("#00ff00");
    ctx.fill_rect(0.0, 5.0, 1.0, 1.0);

    let data2 = read_image_data_cfg(&ctx, 0, 0, 40, 40).unwrap();
    let pixels2 = image_data_to_vec(&data2);

    let (_, g_tl, _, _) = read_pixel(&pixels2, 0, 20, 40);
    let (_, g_br, _, _) = read_pixel(&pixels2, 3, 23, 40);
    let (_, g_beyond_x, _, _) = read_pixel(&pixels2, 4, 23, 40);
    let (_, g_beyond_y, _, _) = read_pixel(&pixels2, 0, 24, 40);

    assert!(g_tl > 200, "fillRect IS scaled (top-left of scaled rect)");
    assert!(g_br > 200, "fillRect IS scaled (bottom-right of scaled rect)");
    assert!(g_beyond_x < 40, "pixel right of scaled rect should not be green");
    assert!(g_beyond_y < 40, "pixel below scaled rect should not be green");
}

#[wasm_bindgen_test]
fn put_image_data_ignores_global_alpha() {
    // globalAlpha — не должен влиять на putImageData. Пишем полупрозрачный
    // красный поверх белого фона; при globalAlpha = 0.5 обычная заливка
    // дала бы розовый, а putImageData — чистый красный.
    let (_c, ctx) = make_canvas(4, 4);
    ctx.set_fill_style_str("#ffffff");
    ctx.fill_rect(0.0, 0.0, 4.0, 4.0);

    ctx.set_global_alpha(0.5);

    let red = vec_to_image_data(&[255, 0, 0, 255], 1, 1).unwrap();
    write_image_data_cfg(&ctx, &red, 1, 1).unwrap();

    let data = read_image_data_cfg(&ctx, 0, 0, 4, 4).unwrap();
    let pixels = image_data_to_vec(&data);
    let (r, g, _, _) = read_pixel(&pixels, 1, 1, 4);
    assert_eq!(r, 255, "putImageData ignores globalAlpha");
    assert_eq!(g, 0, "putImageData writes raw pixel");
}
```

---

### `src/app.rs` — автономный демо-пример

```rust
//! Глава 21.4 — ImageData: пиксельный буфер и его границы.
//!
//! «Пипетка»: клик по холсту показывает RGBA в точке. Отдельные кнопки
//! инвертируют и обнуляют каналы — видно, что канал это байт, а не
//! магическое свойство.
//!
//! Два холста рядом:
//! - слева — обычный: `putImageData` пишет RGBA 1:1;
//! - справа — с `ctx.scale(2, 2)` перед рисованием: `fillRect`-маркер
//!   удваивается, а `putImageData` — нет. Это наглядно показывает, что
//!   `putImageData` игнорирует трансформации.
//!
//! **Файл автономен.** Стили `.px-*` инжектируются в документ через
//! `inject_styles()` — правок в `main.css` не требуется.
//!
//! Wow: кликните в разные углы левого холста — в статусе увидите разные
//! RGBA. Нажмите «Инвертировать R» — красные квадраты станут cyan,
//! потому что R и только R поменялся.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    CanvasRenderingContext2d, Document, Element, HtmlCanvasElement, HtmlElement,
    ImageData, MouseEvent, Node,
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

/// Инжектирует `<style>` с правилами `.px-*` в `<head>`. Идемпотентно:
/// если стиль уже есть — удаляется и добавляется заново, чтобы правки
/// в этой функции всегда применялись без перезагрузки.
fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "px-injected-style";
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
.px-desc {
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
.px-desc-line { margin: 6px 0; }

.px-controls {
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
.px-btn {
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
.px-btn:hover { background: #30363d; }
.px-btn:active { transform: translateY(1px); }
.px-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.px-btn-primary:hover { background: #2ea043; }
.px-btn-danger {
    background: #21262d;
    border-color: #30363d;
    color: #f85149;
}
.px-btn-danger:hover { background: #30363d; }

.px-status {
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

.px-canvases {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    margin-bottom: 16px;
}
@media (max-width: 800px) {
    .px-canvases { grid-template-columns: 1fr; }
}
.px-panel {
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 8px;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 8px;
}
.px-panel-title {
    font-size: 12px;
    font-weight: 600;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
}
.px-panel-hint {
    font-family: ui-monospace, monospace;
    font-size: 11px;
    color: #8b949e;
}
.px-canvas {
    display: block;
    background: #0d1117;
    border-radius: 4px;
    max-width: 100%;
    cursor: crosshair;
}
.px-canvas-readonly { cursor: default; }

.px-footnote {
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
// Чистые функции
// =============================================================================

fn pixel_index(x: u32, y: u32, w: u32, channel: u8) -> usize {
    let ch = (channel as u32).min(3);
    ((y * w + x) * 4 + ch) as usize
}

fn read_pixel(pixels: &[u8], x: u32, y: u32, w: u32) -> (u8, u8, u8, u8) {
    let i = pixel_index(x, y, w, 0);
    if i + 3 >= pixels.len() {
        return (0, 0, 0, 0);
    }
    (pixels[i], pixels[i + 1], pixels[i + 2], pixels[i + 3])
}

fn invert_channel_inplace(pixels: &mut [u8], channel: u8) {
    let ch = (channel as u32).min(3) as usize;
    let mut i = ch;
    while i < pixels.len() {
        pixels[i] = 255 - pixels[i];
        i += 4;
    }
}

fn zero_channel_inplace(pixels: &mut [u8], channel: u8) {
    let ch = (channel as u32).min(3) as usize;
    let mut i = ch;
    while i < pixels.len() {
        pixels[i] = 0;
        i += 4;
    }
}

/// Демонстрационный паттерн: градиент с тремя цветными квадратами
/// по углам. Пипетка в разных точках покажет разные RGBA.
fn make_pattern(w: u32, h: u32) -> Vec<u8> {
    let mut out = vec![0u8; (w * h * 4) as usize];
    let wf = w as f32;
    let hf = h as f32;
    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / wf;
            let fy = y as f32 / hf;
            let i = pixel_index(x, y, w, 0);

            let mut r = (30.0 + fx * 180.0) as u8;
            let mut g = (30.0 + fy * 180.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            if x < w / 5 && y < h / 5 {
                r = 240; g = 60; b = 60;
            }
            if x > 4 * w / 5 && y < h / 5 {
                r = 60; g = 220; b = 80;
            }
            if x > 4 * w / 5 && y > 4 * h / 5 {
                r = 60; g = 80; b = 240;
            }

            out[i] = r;
            out[i + 1] = g;
            out[i + 2] = b;
            out[i + 3] = 255;
        }
    }
    out
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

#[cfg(web_sys_unstable_apis)]
fn write_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
    dx: u32,
    dy: u32,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, dx as i32, dy as i32)
}

#[cfg(not(web_sys_unstable_apis))]
fn write_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
    dx: u32,
    dy: u32,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, dx as f64, dy as f64)
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "<code>ImageData</code> — плоский RGBA-буфер. 4 байта на пиксель: \
     <code>(y*w + x)*4 + 0/1/2/3</code> — это R, G, B, A соответственно. \
     Никакой магии, только массив байтов.",
    "Клик по левому холсту — «пипетка»: в статусе показывается RGBA в этой точке. \
     Обратите внимание, что клик читает <em>значение пикселя</em>, а не «какой \
     объект выбран».",
    "<code>putImageData</code> игнорирует трансформации, <code>globalAlpha</code> \
     и режим смешивания. Правый холст это показывает: перед рисованием применён \
     <code>scale(2, 2)</code>. Маркер (красный квадрат через <code>fillRect</code>) \
     увеличился вдвое, а <code>putImageData</code> лёг 1:1 — пиксель в пиксель.",
];

const CANVAS_W: u32 = 400;
const CANVAS_H: u32 = 240;

struct DemoState {
    pixels: Vec<u8>,
}

impl DemoState {
    fn new() -> Self {
        Self {
            pixels: make_pattern(CANVAS_W, CANVAS_H),
        }
    }
}

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    // Стили — до всего остального. Правок в main.css не требуется.
    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("21.4 — ImageData: пиксельный буфер"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Кнопки управления буфером.
    let controls = document.create_element("div")?;
    controls.set_class_name("px-controls");
    controls.set_inner_html(
        r#"<button class="px-btn px-btn-primary" id="px-reset">Сбросить</button>
           <button class="px-btn" id="px-inv-r">Инвертировать R</button>
           <button class="px-btn" id="px-inv-g">Инвертировать G</button>
           <button class="px-btn" id="px-inv-b">Инвертировать B</button>
           <button class="px-btn px-btn-danger" id="px-zero-a">Обнулить A</button>"#,
    );
    root.append_child(&controls)?;

    // Статус.
    let status = document.create_element("div")?;
    status.set_class_name("px-status");
    status.set_id("px-status");
    status.set_text_content(Some("Кликните по левому холсту, чтобы узнать RGBA"));
    root.append_child(&status)?;

    // Два холста.
    let canvases = document.create_element("div")?;
    canvases.set_class_name("px-canvases");
    canvases.set_inner_html(
        r#"<div class="px-panel">
               <div class="px-panel-title">Обычный холст · кликните для пипетки</div>
               <div class="px-panel-hint">putImageData, transform = identity</div>
               <canvas id="px-canvas-left" class="px-canvas"
                       width="400" height="240"></canvas>
           </div>
           <div class="px-panel">
               <div class="px-panel-title">С ctx.scale(2, 2) до putImageData</div>
               <div class="px-panel-hint">
                   pattern 1:1 · маркер fillRect — увеличен ×2
               </div>
               <canvas id="px-canvas-right" class="px-canvas px-canvas-readonly"
                       width="400" height="240"></canvas>
           </div>"#,
    );
    root.append_child(&canvases)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("px-footnote");
    footnote.set_text_content(Some(
        "Классическая иллюзия: «putImageData — это просто нарисовать картинку». \
         Нет. Это сырой доступ к пиксельному буферу, который полностью обходит \
         весь конвейер рисования. На правом холсте scale(2, 2) — красный маркер \
         стал 40×40, а pattern остался 400×240, потому что putImageData смотрит \
         только на координаты (dx, dy) и больше ни на что.",
    ));
    root.append_child(&footnote)?;

    // Состояние и первая отрисовка.
    let state = Rc::new(RefCell::new(DemoState::new()));
    {
        let st = state.borrow();
        render_both(&document, &st)?;
    }

    // Обработчики.
    wire_pipette(&document, state.clone())?;
    wire_reset(&document, state.clone())?;
    wire_invert(&document, state.clone(), 0)?; // R
    wire_invert(&document, state.clone(), 1)?; // G
    wire_invert(&document, state.clone(), 2)?; // B
    wire_zero_alpha(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_pipette(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("px-canvas-left")
        .ok_or_else(|| JsValue::from_str("no left canvas"))?
        .dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(MouseEvent)>::new(move |ev: MouseEvent| {
        let Some(target) = ev.target() else { return };
        let Ok(el) = target.dyn_into::<Element>() else { return };
        let Ok(canvas) = el.dyn_into::<HtmlCanvasElement>() else {
            return;
        };
        // get_bounding_client_rect() определён на Element, а не на
        // HtmlCanvasElement. Rust не делает auto-upcast между web-sys
        // типами — нужен явный upcast через unchecked_ref.
        let rect = canvas.unchecked_ref::<Element>().get_bounding_client_rect();
        // Переводим координаты экрана в координаты буфера.
        let x = ((ev.client_x() as f64 - rect.left()) / rect.width()
            * canvas.width() as f64) as u32;
        let y = ((ev.client_y() as f64 - rect.top()) / rect.height()
            * canvas.height() as f64) as u32;

        let st = state.borrow();
        let (r, g, b, a) = read_pixel(&st.pixels, x, y, CANVAS_W);
        if let Some(el) = doc.get_element_by_id("px-status") {
            let text = format_str(format_args!(
                "({x}, {y}) → RGBA({r}, {g}, {b}, {a}) · \
                 индекс = (y*{CANVAS_W} + x)*4 = {}",
                pixel_index(x, y, CANVAS_W, 0)
            ));
            el.set_text_content(Some(&text));
        }
    });
    canvas.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_reset(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("px-reset")
        .ok_or_else(|| JsValue::from_str("no px-reset"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        {
            let mut st = state.borrow_mut();
            st.pixels = make_pattern(CANVAS_W, CANVAS_H);
        }
        let st = state.borrow();
        let _ = render_both(&doc, &st);
        set_status(&doc, "буфер сброшен к исходному паттерну");
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_invert(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
    channel: u8,
) -> Result<(), JsValue> {
    let id = match channel {
        0 => "px-inv-r",
        1 => "px-inv-g",
        _ => "px-inv-b",
    };
    let el = document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        {
            let mut st = state.borrow_mut();
            invert_channel_inplace(&mut st.pixels, channel);
        }
        let st = state.borrow();
        let _ = render_both(&doc, &st);
        let name = match channel {
            0 => "R",
            1 => "G",
            _ => "B",
        };
        let msg = format_str(format_args!("канал {name} инвертирован"));
        set_status(&doc, &msg);
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_zero_alpha(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("px-zero-a")
        .ok_or_else(|| JsValue::from_str("no px-zero-a"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        {
            let mut st = state.borrow_mut();
            zero_channel_inplace(&mut st.pixels, 3);
        }
        let st = state.borrow();
        let _ = render_both(&doc, &st);
        set_status(
            &doc,
            "альфа обнулена — холст станет прозрачным (на тёмном фоне видно как чёрный)",
        );
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Рендер ----------

fn render_both(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    render_left(document, state)?;
    render_right(document, state)?;
    Ok(())
}

fn render_left(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("px-canvas-left")
        .ok_or_else(|| JsValue::from_str("no left canvas"))?
        .dyn_into()?;
    let ctx = get_2d_context(&canvas)?;
    // Сброс трансформации на случай, если что-то осталось от предыдущего вызова.
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    let data = vec_to_image_data(&state.pixels, CANVAS_W, CANVAS_H)?;
    write_image_data_cfg(&ctx, &data, 0, 0)
}

fn render_right(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("px-canvas-right")
        .ok_or_else(|| JsValue::from_str("no right canvas"))?
        .dyn_into()?;
    let ctx = get_2d_context(&canvas)?;

    // Шаг 1. Сброс и очистка при identity-трансформации.
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, CANVAS_W as f64, CANVAS_H as f64);

    // Шаг 2. Применяем scale(2, 2). Дальше любые команды рисования
    // будут удваиваться, но putImageData это проигнорирует.
    ctx.scale(2.0, 2.0)?;

    // Шаг 3. putImageData — игнорирует scale, пишет пиксель в пиксель.
    let data = vec_to_image_data(&state.pixels, CANVAS_W, CANVAS_H)?;
    write_image_data_cfg(&ctx, &data, 0, 0)?;

    // Шаг 4. fillRect — уважает scale, поэтому 20×20 CSS-размерности
    // превратится в 40×40 физических пикселей. Всё в пределах 400×240.
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(0.0, 0.0, 20.0, 20.0);

    // Возврат к identity на выходе — следующий вызов не должен получить
    // накопленный scale.
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    Ok(())
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("px-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("px-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// ---------- Мелочи ----------

fn set_status(document: &Document, text: &str) {
    if let Some(el) = document.get_element_by_id("px-status") {
        el.set_text_content(Some(text));
    }
}
```

---

## 21.4.13. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Быстрые host-тесты чистой логики
cargo test --test canvas_imagedata_pure

# 3. wasm-тесты работы с холстом
wasm-pack test --headless --chrome --test wasm_canvas_imagedata

# 4. Полный набор
npm test

# 5. Демо
npm run wasm:dev && npm start
```

## 21.4.14. Проверка корректности

**Host-тесты (13)** — индексация, чтение пикселя, границы буфера, инверсия одного канала, идемпотентность инверсии дважды, обнуление канала, размер и формат паттерна.

**Wasm-тесты (5)**:
- `get_image_data_returns_painted_color` — чтение залитого холста;
- `vec_to_image_data_roundtrip` — `Vec ↔ ImageData` без потерь;
- `vec_to_image_data_rejects_wrong_size` — отказ при неправильном размере;
- `put_image_data_ignores_scale` — **исправлен**: холст 40×40, `scale(4, 4)`, CSS-область 10×10; проверяет, что `putImageData` не растянулся, `fillRect` растянулся, а пиксели за пределами прямоугольника — не зелёные;
- `put_image_data_ignores_global_alpha` — `putImageData` игнорирует alpha.

**Импорты и фичи:**
- `Event` убран из `use web_sys::{...}` в `app.rs` — не используется;
- `Element` присутствует — нужен для `unchecked_ref::<Element>()` перед `get_bounding_client_rect()`;
- Фича `DomRect` обязательна в `Cargo.toml` — иначе `get_bounding_client_rect` не компилируется.

**Format-строки:** все `format!` в tail-позициях обёрнуты в `format_str(format_args!(...))` — согласно правилу проекта.

**RefMut-дисциплина:** `borrow_mut()` в узком scope, `borrow()` для рендера — не пересекаются.

## 21.4.15. Итог

`ImageData` — плоский RGBA-буфер, доступный для чтения и записи по индексу. Каждый пиксель — четыре байта, каждый канал — байт. `getImageData` читает прямоугольник, `putImageData` пишет. Обе функции имеют две cfg-сигнатуры в web-sys; различие инкапсулировано в хелперах `read_image_data_cfg` и `write_image_data_cfg`.

Ключевое наблюдение: `putImageData` **игнорирует трансформации, alpha и режим смешивания**. Это не «быстрый вариант drawImage», а «сырая запись в буфер». Правый холст демо показывает это одной картинкой: маркер через `fillRect` удваивается, pattern через `putImageData` — нет.

Пипетка слева показывает байты напрямую: клик — RGBA в точке и индекс в буфере. Кнопки «инвертировать R/G/B» и «обнулить A» — это не «изменение цвета», это модификация плоского буфера `Vec<u8>`. Один байт за раз.

Демо автономно, стили `.px-*` инжектируются в документ. Тесты разделены на host (чистая логика, миллисекунды) и wasm (canvas API, headless Chrome). Правила проекта (`format_str`, отсутствие dead_code, cfg-гейт для unstable web-sys API) применены последовательно.

Следующая подглава **21.5** — про загрузку изображений: три способа получить пиксели (`<img>`, `createImageBitmap`, `OffscreenCanvas`), EXIF-ориентация, разница в таймингах. **21.6** — фильтры на Rust с SIMD и Worker. **21.7** — практический проект, где всё это собирается вместе.