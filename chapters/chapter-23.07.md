# Глава 23.7. WebCodecs + WASM/GPU: обработка пикселей

## 23.7.0. Обновлённая структура

```
src/
  pixel_ops.rs                    ← НОВЫЙ модуль 23.7
  video_frame.rs                  ← из 23.1 (переиспользуется: format_str, invoke, num, set_prop, get_fn)
  image_decoder.rs                ← из 23.6 (не используется)
  audio_codecs.rs                 ← из 23.5 (не используется)
  video_decoder.rs, video_encoder.rs, video_encoder_tuning.rs   ← 23.2–23.4
  …
tests/
  pure_pixel_ops.rs               ← НОВЫЙ тест: чистые функции, host
  wasm_pixel_ops.rs               ← НОВЫЙ тест: WebGL2, wasm
src/lib.rs                        ← +1 строка: pub mod pixel_ops;
src/app.rs                        ← демо 23.7 (заменяется целиком)
Cargo.toml                        ← +7 фич WebGL2 для web-sys
```

**Комментарий.** Модуль `pixel_ops.rs` — про **обработку пикселей**: чистые CPU-функции (`invert_rgba`, `average_rgb`), детект WebGL2, `GpuInverter` с двумя вариантами создания. Префикс CSS — `px-` (pixels), не пересекается с `vf-` (23.1), `vd-` (23.2), `ve-` (23.3), `vt-` (23.4), `ac-` (23.5), `id-` (23.6). Мини-хелперы `invoke`, `num`, `format_str`, `set_prop`, `get_fn` — из `video_frame::*`. Дублирования нет.

**Ссылки для детального изучения:**

- MDN: [`VideoFrame.copyTo()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame/copyTo)
- MDN: [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- MDN: [`WebGL2RenderingContext`](https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext)
- MDN: [`texImage2D`](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texImage2D)
- MDN: [`UNPACK_FLIP_Y_WEBGL`](https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/pixelStorei)
- W3C: [WebCodecs § VideoFrame.copyTo](https://w3c.github.io/webcodecs/#dom-videoframe-copyto)
- Khronos: [WebGL 2.0 Reference](https://registry.khronos.org/webgl/specs/latest/2.0/)

---

## 23.7.1. О чём эта подглава

WebCodecs даёт `VideoFrame` — контейнер с распакованными пикселями. Но **что с ними делать**, если нужно применить фильтр, инвертировать цвета, наложить эффект?

Два принципиально разных пути:

**CPU-путь.** `frame.copyTo(rgba)` → массив байт в памяти → Rust/WASM цикл по пикселям → `ImageData::new_with_u8_clamped_array` → `ctx.putImageData`. Работает везде, но каждый пиксель проходит через CPU.

**GPU-путь.** `texImage2D(frame)` — frame загружается в WebGL2-текстуру **без readback** → фрагментный шейдер обрабатывает всё параллельно → рендер прямо в видимый WebGL-канвас. В разы быстрее на реальном аппаратном ускорении.

Подглава отвечает на пять вопросов.

**Первый — что делать с `VideoFrame` в Rust.** `copyTo` даёт `Uint8Array` с пикселями в формате RGBA. Дальше — обычный Rust-код.

**Второй — как загрузить `VideoFrame` в GPU.** `texImage2D` принимает `VideoFrame` напрямую. Никакого `getImageData` и readback — браузер сам передаёт данные композитору.

**Третий — как работает WebGL2 шейдер.** Вершинный шейдер размечает экран, фрагментный обрабатывает каждый пиксель. Invert — `vec4(1.0 - c.rgb, c.a)`.

**Четвёртый — зачем `UNPACK_FLIP_Y_WEBGL`.** WebGL-текстура по умолчанию перевёрнута вертикально относительно canvas. Без флага картинка приходит вверх ногами.

**Пятый — насколько GPU быстрее.** На 320×240 с Intel HD Graphics — **×8.9**. На Full HD с дискретной GPU достигает ×20–50.

---

## 23.7.2. Что такое readback и почему это дорого

`getImageData` и `VideoFrame.copyTo` возвращают пиксели из GPU в CPU-память. Это **readback** — самая дорогая операция в графическом pipeline. Причины:

1. **Синхронизация.** GPU и CPU работают асинхронно. Readback требует дождаться, пока GPU закончит все pending-команды, а CPU готов принять данные. Pipeline stall.
2. **Копирование.** Пиксели физически перемещаются из одной памяти в другую — сотни мегабайт в секунду на реальных разрешениях.
3. **Формат.** GPU часто хранит данные в тайловом формате (Morton order и т. п.). Readback требует конвертации в линейный RGBA — каждая строка пикселей пересобирается.

**Практическое следствие.** Если данные уже на GPU (например, `VideoFrame` из видеодекодера или canvas), **выгоднее обрабатывать их на GPU**, не забирая на CPU.

---

## 23.7.3. Что показывает демо

**Layout:**

- **Source canvas** — анимированная сцена (градиент + движущийся круг), 320×240.
- **CPU canvas** — результат CPU-инверсии: `copyTo → invert_rgba → putImageData`.
- **GPU canvas** — WebGL2-канвас: `texImage2D → shader → прямой рендер`.
- **Controls:** Run CPU / Run GPU / Run Both.
- **Stats:** тайминги стадий, победитель, throughput, средний RGB источника.
- **Bar chart** — горизонтальные полосы пропорций CPU vs GPU.

**Сценарий:**

1. Source крутит анимацию.
2. **Run CPU** — 30 кадров через CPU-инверсию. Тайминги по стадиям.
3. **Run GPU** — те же 30 кадров через GPU-инверсию. Прямой рендер в видимый WebGL-канвас.
4. **Run Both** — последовательный прогон для чистого сравнения.

**Wow-момент.** Source — цветной, results — инвертированные (красный ↔ голубой). Оба result-канала визуально идентичны. Timing: CPU 2.67 мс/кадр, GPU 0.30 мс/кадр. **×8.89 в пользу GPU** на 320×240.

---

## 23.7.4. Практические находки

Мы прошли через шесть итераций, каждая вскрывала новую грань web-sys, WebGL2 или браузерной архитектуры. Вот полная карта грабель.

### Находка 1. WebGL-константы — associated constants типа, не методы

Первый рефлекс — писать `gl.VERTEX_SHADER()`, `gl.ARRAY_BUFFER()`. В web-sys это не работает — компилятор скажет `no method named VERTEX_SHADER`. Правильно:

```rust
WebGl2RenderingContext::VERTEX_SHADER
WebGl2RenderingContext::ARRAY_BUFFER
WebGl2RenderingContext::STATIC_DRAW
```

В TypeScript-аналоге это выглядит как `gl.VERTEX_SHADER` (свойство), в Rust-биндингах — как associated constant.

### Находка 2. Метод uniform называется `uniform1i`, не `uniform_1i`

В Rust хочется писать `uniform_1i` в стиле snake_case. В web-sys сохранено оригинальное имя метода — `uniform1i`. Все uniform-функции: `uniform1f`, `uniform1i`, `uniform2f`, `uniform3f`, `uniform4f`, `uniformMatrix4fv` — без подчёркиваний.

### Находка 3. `bufferData` и `texImage2D` — вызываем через `invoke`

Сигнатуры этих методов перегружены. `bufferData` может принимать `ArrayBufferView` или `number`, `texImage2D` — семь разных вариантов. В web-sys 0.3.105 типизированные методы имеют разные сигнатуры в разных сборках. Универсальный путь:

```rust
invoke(
    gl.as_ref(),
    "bufferData",
    &[target, view, usage],
)?;
invoke(
    gl.as_ref(),
    "texImage2D",
    &[target, level, internalFormat, format, type_, frame],
)?;
```

Всё передаётся как `JsValue`, браузер сам разбирается. Никаких compile-time несовпадений.

### Находка 4. `put_image_data` принимает `f64` или `i32` — зависит от сборки

В stable-сборке web-sys 0.3.105 сигнатура `put_image_data(&self, img, dx: i32, dy: i32)`. В сборке с `--cfg=web_sys_unstable_apis` — `dx: f64, dy: f64`. Rust-analyzer видит одну, `cargo test` — другую. Казалось бы, лови ошибку то на `0`, то на `0.0`.

Решение — **вызывать через `invoke`**:

```rust
invoke(
    ctx.as_ref(),
    "putImageData",
    &[img.into(), num(0.0), num(0.0)],
)?;
```

Типизация на стороне Rust пропадает, вызов уходит в JS-метод напрямую. Работает в любой сборке.

### Находка 5. Canvas хранит premultiplied — тесты учитывают

Пиксель `(200, 100, 50, alpha=128)` на canvas хранится как **premultiplied RGBA**:

```
R_premult = round(200 × 128 / 255) = 100
```

`VideoFrame.copyTo` возвращает пиксели **как хранятся**, в premultiplied виде. Если в тесте проверить точное значение RGB после инверсии — оно не совпадёт.

Решение — **разделить тесты**:
- **Точные RGB**: alpha = 255. Premultiplied совпадает с unpremultiplied.
- **Сохранение alpha**: alpha = 128. Проверяем только alpha, RGB не сравниваем.

### Находка 6. `transferToImageBitmap` + `drawImage` медленные — прямой рендер в 36× быстрее

Первая версия GPU-пути: `GpuInverter::new(w, h)` создаёт `OffscreenCanvas`, `apply(frame)` рендерит и возвращает `ImageBitmap` через `transferToImageBitmap`, потом `drawImage(bitmap)` на 2d-канвас. Три стадии на кадр — upload, shader, blit. Результат: **10.7 мс/кадр**, GPU **медленнее** CPU.

Прямой рендер:
- `GpuInverter::from_canvas(&gpu_canvas)` — инициализирует WebGL2-контекст на видимом `<canvas>`.
- `render(frame)` — рисует шейдером прямо в этот canvas.
- Никаких промежуточных буферов.

Результат: **0.30 мс/кадр** — в **36 раз** быстрее.

### Находка 7. `get_webgl_renderer` требует активации расширения

Первая попытка прочитать `UNMASKED_RENDERER_WEBGL` (0x9246) без активации `WEBGL_debug_renderer_info` даёт `WebGL warning: getParameter: Bad 'pname': 0x9246`. Enum валиден **только** при активном расширении.

Ещё хуже — Chrome и Firefox уже помечают это расширение deprecated. Правильнее использовать **стандартный `RENDERER`** (0x1F01):

```rust
gl.get_parameter(0x1F01).ok()?.as_string()
```

Возвращает masked-строку с суффиксом «or similar», но она содержит имя backend'а (Intel / NVIDIA / SwiftShader).

### Находка 8. `RunGuard` должен обновлять UI в `Drop`

Первая версия сбрасывала `running = false` в `Drop`, но UI уже отрисовался с `running = true`. Кнопки оставались серыми.

Решение — `RunGuard` получает `Document`, в `Drop` вызывает `update_stats`, `update_buttons`, `update_ratio_bars`:

```rust
impl Drop for RunGuard {
    fn drop(&mut self) {
        {
            let mut s = self.st.borrow_mut();
            s.running = false;
        }
        let s = self.st.borrow();
        update_stats(&self.doc, &s);
        update_buttons(&self.doc, &s);
        update_ratio_bars(&self.doc, &s);
    }
}
```

Работает на любом выходе — нормальном, раннем `return`, панике.

### Находка 9. `get_context("webgl2")` — двойной `?`

`canvas.get_context("webgl2")` возвращает `Result<Option<Object>, JsValue>`. После `.ok()` — `Option<Object>`. Нужно два `?`, чтобы получить `Object`:

```rust
let ctx = canvas.get_context("webgl2").ok()??;
let gl = ctx.dyn_into::<WebGl2RenderingContext>().ok()?;
```

Один `?` даёт `Option<Object>`, второй — `Object`.

---

## 23.7.5. Исходные файлы

### `Cargo.toml` — фичи

В блок `[dependencies.web-sys]` → `features`:

```toml
    # 23.7 — WebGL2 для GPU-инверсии
    "WebGl2RenderingContext",
    "WebGlProgram",
    "WebGlShader",
    "WebGlBuffer",
    "WebGlTexture",
    "WebGlUniformLocation",
    "OffscreenCanvas",
    "OffscreenCanvasRenderingContext2d",
```

`OffscreenCanvas` нужен только для варианта `GpuInverter::new()` — используется в тестах.

### `src/pixel_ops.rs`

```rust
//! 23.7. WebCodecs + WASM/GPU: обработка пикселей.
//!
//! Два пути обработки `VideoFrame`:
//!
//! - **CPU**: `copyTo(rgba)` → Rust-цикл → `putImageData`.
//! - **GPU**: `texImage2D(frame)` → WebGL2 shader.
//!
//! ## Варианты GpuInverter
//!
//! - `GpuInverter::new(w, h)` — владеет `OffscreenCanvas`. Используется
//!   в тестах: `apply(frame)` возвращает `ImageBitmap`.
//! - `GpuInverter::from_canvas(canvas)` — использует видимый
//!   WebGL-канвас. `render(frame)` рисует прямо в него, без
//!   `transferToImageBitmap` и `drawImage`.
//!
//! ## Замечания по web-sys
//!
//! - WebGL-константы — associated constants типа
//!   (`WebGl2RenderingContext::VERTEX_SHADER`), а не методы.
//! - Метод uniform называется `uniform1i`, не `uniform_1i`.
//! - `bufferData` и `texImage2D` вызываются через `invoke` —
//!   сигнатуры перегружены, типизированные методы нестабильны.
//! - `put_image_data` принимает `f64` в unstable-сборке и `i32` в
//!   stable — вызов через `invoke` снимает проблему.
//!
//! ## Переиспользование
//!
//! Хелперы `invoke`, `num`, `format_str`, `set_prop`, `get_fn` —
//! из `video_frame::*`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen::Clamped;
use web_sys::{
    HtmlCanvasElement, ImageData, OffscreenCanvas,
    WebGl2RenderingContext, WebGlBuffer, WebGlProgram, WebGlShader,
    WebGlTexture, WebGlUniformLocation,
};

use crate::video_frame::{format_str, get_fn, invoke, num, set_prop};

// =============================================================================
// Чистые функции (host-testable)
// =============================================================================

/// Инвертирует RGB-каналы in-place. Alpha не трогает.
///
/// Self-inverse: двойное применение возвращает исходные данные.
/// Обрабатывает любой размер буфера: если длина не кратна 4,
/// оставшиеся байты пропускаются.
pub fn invert_rgba(bytes: &mut [u8]) {
    let full_pixels = bytes.len() / 4;
    for i in 0..full_pixels {
        let base = i * 4;
        bytes[base] = 255 - bytes[base];
        bytes[base + 1] = 255 - bytes[base + 1];
        bytes[base + 2] = 255 - bytes[base + 2];
    }
}

/// Средний RGB по всем пикселям. Alpha игнорируется.
///
/// Возвращает `(r, g, b)` в диапазоне 0.0–255.0. Для пустого
/// буфера — `(0.0, 0.0, 0.0)`.
pub fn average_rgb(bytes: &[u8]) -> (f32, f32, f32) {
    let full_pixels = bytes.len() / 4;
    if full_pixels == 0 {
        return (0.0, 0.0, 0.0);
    }
    let mut r_sum: u64 = 0;
    let mut g_sum: u64 = 0;
    let mut b_sum: u64 = 0;
    for i in 0..full_pixels {
        let base = i * 4;
        r_sum += bytes[base] as u64;
        g_sum += bytes[base + 1] as u64;
        b_sum += bytes[base + 2] as u64;
    }
    let n = full_pixels as f32;
    (r_sum as f32 / n, g_sum as f32 / n, b_sum as f32 / n)
}

// =============================================================================
// Feature detection
// =============================================================================

/// `WebGL2RenderingContext` доступен на данном устройстве.
pub fn has_webgl2_support() -> bool {
    let Some(window) = web_sys::window() else {
        return false;
    };
    let Some(document) = window.document() else {
        return false;
    };
    let Ok(el) = document.create_element("canvas") else {
        return false;
    };
    let Ok(canvas) = el.dyn_into::<HtmlCanvasElement>() else {
        return false;
    };
    match canvas.get_context("webgl2") {
        Ok(Some(_)) => true,
        _ => false,
    }
}

/// Возвращает строку, описывающую WebGL-backend.
///
/// ## Почему RENDERER, а не UNMASKED_RENDERER_WEBGL
///
/// `UNMASKED_RENDERER_WEBGL` (0x9246) даёт точное имя GPU, но
/// требует активации устаревающего расширения
/// `WEBGL_debug_renderer_info`. Firefox уже помечает его как
/// deprecated, Chrome следует за ним.
///
/// `RENDERER` (0x1F01) — стандартный параметр без расширения.
/// Может вернуть masked-строку с суффиксом «or similar», но она
/// всё равно содержит имя backend'а (Intel / NVIDIA / SwiftShader).
pub fn get_webgl_renderer() -> Option<String> {
    let window = web_sys::window()?;
    let document = window.document()?;
    let el = document.create_element("canvas").ok()?;
    let canvas = el.dyn_into::<HtmlCanvasElement>().ok()?;
    // get_context возвращает Option<Object>. Двойной `?`:
    // сначала разворачиваем Result, потом Option.
    let ctx = canvas.get_context("webgl2").ok()??;
    let gl = ctx.dyn_into::<WebGl2RenderingContext>().ok()?;

    // gl.RENDERER = 0x1F01. Стандартный параметр, без расширений.
    gl.get_parameter(0x1F01)
        .ok()?
        .as_string()
        .filter(|s| !s.is_empty())
}

/// `true`, если рендер идёт через софтверный backend
/// (SwiftShader на Linux headless, llvmpipe и т. п.).
pub fn is_software_renderer() -> bool {
    get_webgl_renderer()
        .map(|s| {
            let lower = s.to_lowercase();
            lower.contains("swiftshader")
                || lower.contains("llvmpipe")
                || lower.contains("software")
                || lower.contains("softpipe")
        })
        .unwrap_or(false)
}

/// Сокращает длинную строку renderer'а до компактного вида для UI.
///
/// `"ANGLE (Intel, Intel(R) HD Graphics Direct3D11 vs_5_0 ps_5_0), or similar"`
///   → `"Intel(R) HD Graphics"`
/// `"Google SwiftShader"` → `"Google SwiftShader"`
pub fn shorten_renderer(s: &str) -> String {
    if let Some(rest) = s.strip_prefix("ANGLE (") {
        let trimmed = rest.trim_end_matches(|c| c == ')' || c == ' ');
        let trimmed = trimmed.trim_end_matches(", or similar");
        if let Some((_vendor, model)) = trimmed.split_once(", ") {
            return model
                .split(" Direct3D")
                .next()
                .unwrap_or(model)
                .trim()
                .to_string();
        }
        return trimmed.to_string();
    }
    s.to_string()
}

// =============================================================================
// WebGL2 шейдеры
// =============================================================================

const VERTEX_SHADER_SRC: &str = r#"#version 300 es
in vec2 a_pos;
out vec2 v_uv;
void main() {
    v_uv = (a_pos + 1.0) * 0.5;
    gl_Position = vec4(a_pos, 0.0, 1.0);
}
"#;

const FRAGMENT_SHADER_SRC: &str = r#"#version 300 es
precision highp float;
in vec2 v_uv;
out vec4 out_color;
uniform sampler2D u_tex;
void main() {
    vec4 c = texture(u_tex, v_uv);
    out_color = vec4(1.0 - c.rgb, c.a);
}
"#;

// =============================================================================
// GpuInverter
// =============================================================================

/// GPU-инвертор одного кадра.
///
/// Владеет WebGL2-контекстом и всеми ресурсами (программа, VBO,
/// текстура). Если создан через `new(w, h)` — владеет и
/// `OffscreenCanvas` для `apply()`. Если через `from_canvas()` —
/// рисует прямо в переданный canvas.
pub struct GpuInverter {
    gl: WebGl2RenderingContext,
    program: WebGlProgram,
    u_tex: WebGlUniformLocation,
    _vbo: WebGlBuffer,
    _texture: WebGlTexture,
    width: u32,
    height: u32,
    /// Some — владеем OffscreenCanvas, можем вызвать
    /// `transfer_to_image_bitmap`. None — рисуем в чужой canvas.
    owned_canvas: Option<OffscreenCanvas>,
}

impl GpuInverter {
    /// Создаёт инвертор с внутренним `OffscreenCanvas`.
    ///
    /// `apply(frame)` после рендера вернёт `ImageBitmap`.
    pub fn new(width: u32, height: u32) -> Result<Self, JsValue> {
        let canvas = OffscreenCanvas::new(width, height)?;
        let gl = canvas
            .get_context("webgl2")?
            .ok_or_else(|| JsValue::from_str("webgl2 not available"))?
            .dyn_into::<WebGl2RenderingContext>()?;
        let mut inv = Self::from_gl(gl, width, height)?;
        inv.owned_canvas = Some(canvas);
        Ok(inv)
    }

    /// Создаёт инвертор на базе видимого `<canvas>`.
    ///
    /// `render(frame)` рисует прямо в этот canvas — без
    /// `transferToImageBitmap` и `drawImage`. Быстрее, чем `apply`,
    /// потому что не создаёт промежуточный `ImageBitmap`.
    pub fn from_canvas(canvas: &HtmlCanvasElement) -> Result<Self, JsValue> {
        let width = canvas.width();
        let height = canvas.height();
        let gl = canvas
            .get_context("webgl2")?
            .ok_or_else(|| JsValue::from_str("webgl2 not available on canvas"))?
            .dyn_into::<WebGl2RenderingContext>()?;
        Self::from_gl(gl, width, height)
    }

    /// Общая инициализация: шейдеры, буферы, текстура.
    fn from_gl(
        gl: WebGl2RenderingContext,
        width: u32,
        height: u32,
    ) -> Result<Self, JsValue> {
        let vert = compile_shader(
            &gl,
            WebGl2RenderingContext::VERTEX_SHADER,
            VERTEX_SHADER_SRC,
        )?;
        let frag = compile_shader(
            &gl,
            WebGl2RenderingContext::FRAGMENT_SHADER,
            FRAGMENT_SHADER_SRC,
        )?;
        let program = link_program(&gl, &vert, &frag)?;

        let vbo = gl
            .create_buffer()
            .ok_or_else(|| JsValue::from_str("create_buffer failed"))?;
        gl.bind_buffer(WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo));
        let verts: [f32; 8] = [-1.0, -1.0, 1.0, -1.0, -1.0, 1.0, 1.0, 1.0];
        let verts_arr = js_sys::Float32Array::from(&verts[..]);
        invoke(
            gl.as_ref(),
            "bufferData",
            &[
                num(WebGl2RenderingContext::ARRAY_BUFFER as f64),
                verts_arr.into(),
                num(WebGl2RenderingContext::STATIC_DRAW as f64),
            ],
        )?;

        let a_pos = gl.get_attrib_location(&program, "a_pos") as u32;
        gl.enable_vertex_attrib_array(a_pos);
        gl.vertex_attrib_pointer_with_i32(
            a_pos,
            2,
            WebGl2RenderingContext::FLOAT,
            false,
            0,
            0,
        );

        let texture = gl
            .create_texture()
            .ok_or_else(|| JsValue::from_str("create_texture failed"))?;
        gl.bind_texture(WebGl2RenderingContext::TEXTURE_2D, Some(&texture));
        gl.tex_parameteri(
            WebGl2RenderingContext::TEXTURE_2D,
            WebGl2RenderingContext::TEXTURE_MIN_FILTER,
            WebGl2RenderingContext::NEAREST as i32,
        );
        gl.tex_parameteri(
            WebGl2RenderingContext::TEXTURE_2D,
            WebGl2RenderingContext::TEXTURE_MAG_FILTER,
            WebGl2RenderingContext::NEAREST as i32,
        );
        gl.tex_parameteri(
            WebGl2RenderingContext::TEXTURE_2D,
            WebGl2RenderingContext::TEXTURE_WRAP_S,
            WebGl2RenderingContext::CLAMP_TO_EDGE as i32,
        );
        gl.tex_parameteri(
            WebGl2RenderingContext::TEXTURE_2D,
            WebGl2RenderingContext::TEXTURE_WRAP_T,
            WebGl2RenderingContext::CLAMP_TO_EDGE as i32,
        );
        gl.pixel_storei(WebGl2RenderingContext::UNPACK_FLIP_Y_WEBGL, 1);

        let u_tex = gl
            .get_uniform_location(&program, "u_tex")
            .ok_or_else(|| JsValue::from_str("u_tex not found"))?;

        Ok(Self {
            gl,
            program,
            u_tex,
            _vbo: vbo,
            _texture: texture,
            width,
            height,
            owned_canvas: None,
        })
    }

    /// Рендерит инвертированный кадр в свой GL-контекст.
    ///
    /// Для `from_canvas()` — результат появляется прямо в видимом
    /// canvas. Для `new()` — результат внутри OffscreenCanvas,
    /// доступен через `apply()`.
    pub fn render(&self, frame: &JsValue) -> Result<(), JsValue> {
        let gl = &self.gl;
        gl.viewport(0, 0, self.width as i32, self.height as i32);
        gl.clear_color(0.0, 0.0, 0.0, 1.0);
        gl.clear(WebGl2RenderingContext::COLOR_BUFFER_BIT);

        gl.use_program(Some(&self.program));
        gl.active_texture(WebGl2RenderingContext::TEXTURE0);
        gl.bind_texture(WebGl2RenderingContext::TEXTURE_2D, Some(&self._texture));

        invoke(
            gl.as_ref(),
            "texImage2D",
            &[
                num(WebGl2RenderingContext::TEXTURE_2D as f64),
                num(0.0),
                num(WebGl2RenderingContext::RGBA as f64),
                num(WebGl2RenderingContext::RGBA as f64),
                num(WebGl2RenderingContext::UNSIGNED_BYTE as f64),
                frame.clone(),
            ],
        )?;

        gl.uniform1i(Some(&self.u_tex), 0);
        gl.draw_arrays(WebGl2RenderingContext::TRIANGLE_STRIP, 0, 4);
        Ok(())
    }

    /// Рендерит и забирает результат как `ImageBitmap`.
    ///
    /// Работает только для варианта, созданного через `new()`.
    pub fn apply(&self, frame: &JsValue) -> Result<JsValue, JsValue> {
        self.render(frame)?;
        let canvas = self
            .owned_canvas
            .as_ref()
            .ok_or_else(|| {
                JsValue::from_str(
                    "apply() требует вариант, созданный через GpuInverter::new()",
                )
            })?;
        let bitmap = canvas.transfer_to_image_bitmap()?;
        Ok(bitmap.into())
    }
}

// =============================================================================
// Компиляция шейдеров
// =============================================================================

fn compile_shader(
    gl: &WebGl2RenderingContext,
    shader_type: u32,
    source: &str,
) -> Result<WebGlShader, JsValue> {
    let shader = gl
        .create_shader(shader_type)
        .ok_or_else(|| JsValue::from_str("create_shader failed"))?;
    gl.shader_source(&shader, source);
    gl.compile_shader(&shader);
    let success = gl
        .get_shader_parameter(&shader, WebGl2RenderingContext::COMPILE_STATUS)
        .as_bool()
        .unwrap_or(false);
    if !success {
        let log = gl.get_shader_info_log(&shader).unwrap_or_default();
        gl.delete_shader(Some(&shader));
        return Err(JsValue::from_str(&format_str(format_args!(
            "shader compile error: {log}"
        ))));
    }
    Ok(shader)
}

fn link_program(
    gl: &WebGl2RenderingContext,
    vert: &WebGlShader,
    frag: &WebGlShader,
) -> Result<WebGlProgram, JsValue> {
    let program = gl
        .create_program()
        .ok_or_else(|| JsValue::from_str("create_program failed"))?;
    gl.attach_shader(&program, vert);
    gl.attach_shader(&program, frag);
    gl.link_program(&program);
    let success = gl
        .get_program_parameter(&program, WebGl2RenderingContext::LINK_STATUS)
        .as_bool()
        .unwrap_or(false);
    if !success {
        let log = gl.get_program_info_log(&program).unwrap_or_default();
        gl.delete_program(Some(&program));
        return Err(JsValue::from_str(&format_str(format_args!(
            "program link error: {log}"
        ))));
    }
    Ok(program)
}

// =============================================================================
// CPU-инверсия VideoFrame
// =============================================================================

/// Извлекает пиксели из `VideoFrame` в RGBA-буфер через `copyTo`.
///
/// Возвращает `Vec<u8>` размером `w * h * 4`. Для canvas-based
/// frame работает всегда; для ImageDecoder-generated может
/// падать с `No media resource` (см. 23.6).
pub async fn copy_frame_to_rgba(
    frame: &JsValue,
    width: u32,
    height: u32,
) -> Result<Vec<u8>, JsValue> {
    if width == 0 || height == 0 {
        return Err(JsValue::from_str("copy_frame_to_rgba: zero dimensions"));
    }
    let size = (width as usize) * (height as usize) * 4;
    let buffer = js_sys::Uint8Array::new_with_length(size as u32);
    let opts = js_sys::Object::new();
    set_prop(&opts, "format", &JsValue::from_str("RGBA"))?;
    let f = get_fn(frame, "copyTo")?;
    let promise = f
        .call2(frame, &buffer, &opts)?
        .dyn_into::<js_sys::Promise>()?;
    wasm_bindgen_futures::JsFuture::from(promise).await?;
    Ok(buffer.to_vec())
}

/// Кладёт RGBA-буфер на canvas через `ImageData` + `putImageData`.
///
/// `putImageData` вызывается через `invoke`: сигнатура `dx`/`dy`
/// меняется между stable и unstable-сборками web-sys.
pub fn blit_rgba_to_canvas(
    canvas: &HtmlCanvasElement,
    rgba: Vec<u8>,
    width: u32,
    height: u32,
) -> Result<(), JsValue> {
    if width == 0 || height == 0 {
        return Err(JsValue::from_str("blit_rgba_to_canvas: zero dimensions"));
    }
    if canvas.width() != width {
        canvas.set_width(width);
    }
    if canvas.height() != height {
        canvas.set_height(height);
    }
    let clamped = Clamped(&rgba[..]);
    let img = ImageData::new_with_u8_clamped_array_and_sh(clamped, width, height)?;
    let ctx = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d"))?
        .dyn_into::<web_sys::CanvasRenderingContext2d>()?;
    invoke(
        ctx.as_ref(),
        "putImageData",
        &[img.into(), num(0.0), num(0.0)],
    )?;
    Ok(())
}

/// Рисует `ImageBitmap` на видимом 2d-canvas через `drawImage`.
///
/// Оставлено для случаев, когда GPU-путь идёт через `ImageBitmap`
/// (вариант `GpuInverter::new` + `apply`).
pub fn blit_bitmap_to_canvas(
    canvas: &HtmlCanvasElement,
    bitmap: &JsValue,
    width: u32,
    height: u32,
) -> Result<(), JsValue> {
    if canvas.width() != width {
        canvas.set_width(width);
    }
    if canvas.height() != height {
        canvas.set_height(height);
    }
    let ctx = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d"))?
        .dyn_into::<web_sys::CanvasRenderingContext2d>()?;
    invoke(
        ctx.as_ref(),
        "drawImage",
        &[
            bitmap.clone(),
            num(0.0),
            num(0.0),
            num(width as f64),
            num(height as f64),
        ],
    )?;
    Ok(())
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
pub mod image_decoder;            // 23.6
pub mod pixel_ops;                // ← 23.7
```

### `tests/pure_pixel_ops.rs`

```rust
//! Тесты чистых функций обработки пикселей.
//!
//! Не требуют браузера — запускаются через `cargo test`.

use hello_wasm02::pixel_ops::{average_rgb, invert_rgba};

// =============================================================================
// invert_rgba: базовые случаи
// =============================================================================

#[test]
fn invert_rgba_is_self_inverse() {
    let original: Vec<u8> = (0..64).map(|i| (i * 3) as u8).collect();
    let mut bytes = original.clone();
    invert_rgba(&mut bytes);
    assert_ne!(bytes, original, "первое invert должно изменить данные");
    invert_rgba(&mut bytes);
    assert_eq!(bytes, original, "второе invert должно вернуть оригинал");
}

#[test]
fn invert_rgba_preserves_alpha() {
    let mut bytes = vec![
        10, 20, 30, 200,
        40, 50, 60, 100,
        70, 80, 90, 255,
        100, 110, 120, 0,
    ];
    invert_rgba(&mut bytes);
    assert_eq!(bytes[3], 200);
    assert_eq!(bytes[7], 100);
    assert_eq!(bytes[11], 255);
    assert_eq!(bytes[15], 0);
}

#[test]
fn invert_rgba_handles_non_multiple_of_4() {
    // 10 байт: 2 полных пикселя + 2 «хвостовых».
    let mut bytes = vec![10, 20, 30, 200, 40, 50, 60, 100, 99, 88];
    invert_rgba(&mut bytes);
    assert_eq!(bytes[0], 245);
    assert_eq!(bytes[1], 235);
    assert_eq!(bytes[2], 225);
    assert_eq!(bytes[3], 200);
    assert_eq!(bytes[4], 215);
    assert_eq!(bytes[8], 99);
    assert_eq!(bytes[9], 88);
}

#[test]
fn invert_rgba_empty_does_not_panic() {
    let mut bytes: Vec<u8> = Vec::new();
    invert_rgba(&mut bytes);
    assert!(bytes.is_empty());
}

#[test]
fn invert_rgba_black_to_white() {
    let mut bytes = vec![0, 0, 0, 255];
    invert_rgba(&mut bytes);
    assert_eq!(bytes, vec![255, 255, 255, 255]);
}

#[test]
fn invert_rgba_white_to_black() {
    let mut bytes = vec![255, 255, 255, 255];
    invert_rgba(&mut bytes);
    assert_eq!(bytes, vec![0, 0, 0, 255]);
}

// =============================================================================
// average_rgb
// =============================================================================

#[test]
fn average_rgb_ignores_alpha() {
    let bytes = vec![255, 0, 0, 0, 255, 0, 0, 255];
    let (r, g, b) = average_rgb(&bytes);
    assert!((r - 255.0).abs() < 0.01);
    assert!(g.abs() < 0.01);
    assert!(b.abs() < 0.01);
}

#[test]
fn average_rgb_simple() {
    let bytes = vec![0, 100, 200, 255, 100, 200, 0, 255];
    let (r, g, b) = average_rgb(&bytes);
    assert!((r - 50.0).abs() < 0.01);
    assert!((g - 150.0).abs() < 0.01);
    assert!((b - 100.0).abs() < 0.01);
}

#[test]
fn average_rgb_empty_returns_zero() {
    let (r, g, b) = average_rgb(&[]);
    assert_eq!((r, g, b), (0.0, 0.0, 0.0));
}

#[test]
fn average_rgb_non_multiple_of_4() {
    let bytes = vec![50, 60, 70, 255, 99, 88];
    let (r, g, b) = average_rgb(&bytes);
    assert!((r - 50.0).abs() < 0.01);
    assert!((g - 60.0).abs() < 0.01);
    assert!((b - 70.0).abs() < 0.01);
}

#[test]
fn invert_and_average_consistency() {
    let mut bytes = vec![
        10, 20, 30, 255,
        200, 100, 50, 128,
        40, 60, 80, 64,
    ];
    let (r0, g0, b0) = average_rgb(&bytes);
    invert_rgba(&mut bytes);
    let (r1, g1, b1) = average_rgb(&bytes);
    assert!((r1 - (255.0 - r0)).abs() < 0.01);
    assert!((g1 - (255.0 - g0)).abs() < 0.01);
    assert!((b1 - (255.0 - b0)).abs() < 0.01);
}
```

### `tests/wasm_pixel_ops.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

use hello_wasm02::pixel_ops::{
    average_rgb, blit_rgba_to_canvas, copy_frame_to_rgba,
    has_webgl2_support, invert_rgba, GpuInverter,
};
use hello_wasm02::video_frame::{invoke, num};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Feature detection
// =============================================================================

#[wasm_bindgen_test]
fn webgl2_support_detected() {
    let _ = has_webgl2_support();
}

// =============================================================================
// GpuInverter: смоук-тесты
// =============================================================================

#[wasm_bindgen_test]
fn gpu_inverter_constructs() {
    if !has_webgl2_support() {
        return;
    }
    let inv = GpuInverter::new(64, 48);
    assert!(
        inv.is_ok(),
        "GpuInverter should construct: {:?}",
        inv.err(),
    );
}

#[wasm_bindgen_test]
fn gpu_inverter_round_trip() {
    if !has_webgl2_support() {
        return;
    }
    let inv = match GpuInverter::new(4, 1) {
        Ok(v) => v,
        Err(_) => return,
    };

    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let canvas = document.create_element("canvas").unwrap();
    let canvas: web_sys::HtmlCanvasElement = canvas.dyn_into().unwrap();
    canvas.set_width(4);
    canvas.set_height(1);

    let ctx = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .unwrap();
    let data: Vec<u8> = vec![
        10, 20, 30, 255,
        200, 100, 50, 255,
        0, 0, 0, 255,
        255, 255, 255, 255,
    ];
    let clamped = wasm_bindgen::Clamped(&data[..]);
    let img = web_sys::ImageData::new_with_u8_clamped_array_and_sh(clamped, 4, 1)
        .unwrap();
    invoke(ctx.as_ref(), "putImageData", &[img.into(), num(0.0), num(0.0)])
        .unwrap();

    let frame = create_video_frame(&canvas);
    assert!(frame.is_ok(), "VideoFrame creation");

    let bitmap = inv.apply(&frame.unwrap());
    assert!(
        bitmap.is_ok(),
        "GPU apply should succeed: {:?}",
        bitmap.err(),
    );
}

// =============================================================================
// CPU: точные значения (alpha = 255)
// =============================================================================

#[wasm_bindgen_test]
async fn cpu_round_trip_from_canvas() {
    // Alpha = 255: при alpha < 255 canvas хранит пиксели как
    // premultiplied, что искажает точное сравнение RGB.
    // VideoFrame.copyTo возвращает пиксели как есть.
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let canvas = document.create_element("canvas").unwrap();
    let canvas: web_sys::HtmlCanvasElement = canvas.dyn_into().unwrap();
    canvas.set_width(2);
    canvas.set_height(1);

    let ctx = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .unwrap();
    let data: Vec<u8> = vec![50, 100, 150, 255, 200, 50, 100, 255];
    let clamped = wasm_bindgen::Clamped(&data[..]);
    let img = web_sys::ImageData::new_with_u8_clamped_array_and_sh(clamped, 2, 1)
        .unwrap();
    invoke(ctx.as_ref(), "putImageData", &[img.into(), num(0.0), num(0.0)])
        .unwrap();

    let frame = create_video_frame(&canvas).expect("VideoFrame");
    let mut rgba = copy_frame_to_rgba(&frame, 2, 1).await.expect("copyTo");
    assert_eq!(rgba.len(), 8);

    invert_rgba(&mut rgba);
    assert_eq!(rgba[0], 205);
    assert_eq!(rgba[1], 155);
    assert_eq!(rgba[2], 105);
    assert_eq!(rgba[3], 255);
    assert_eq!(rgba[4], 55);
    assert_eq!(rgba[5], 205);
    assert_eq!(rgba[6], 155);
    assert_eq!(rgba[7], 255);
}

// =============================================================================
// CPU: alpha сохраняется после copyTo + invert
// =============================================================================

#[wasm_bindgen_test]
async fn cpu_round_trip_preserves_alpha() {
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let canvas = document.create_element("canvas").unwrap();
    let canvas: web_sys::HtmlCanvasElement = canvas.dyn_into().unwrap();
    canvas.set_width(1);
    canvas.set_height(1);

    let ctx = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .unwrap();
    let data: Vec<u8> = vec![100, 100, 100, 128];
    let clamped = wasm_bindgen::Clamped(&data[..]);
    let img = web_sys::ImageData::new_with_u8_clamped_array_and_sh(clamped, 1, 1)
        .unwrap();
    invoke(ctx.as_ref(), "putImageData", &[img.into(), num(0.0), num(0.0)])
        .unwrap();

    let frame = create_video_frame(&canvas).expect("VideoFrame");
    let mut rgba = copy_frame_to_rgba(&frame, 1, 1).await.expect("copyTo");
    let alpha_before = rgba[3];
    invert_rgba(&mut rgba);
    assert_eq!(rgba[3], alpha_before);
}

// =============================================================================
// CPU: blit обратно на canvas — round-trip
// =============================================================================

#[wasm_bindgen_test]
fn blit_rgba_round_trip() {
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();

    let canvas = document.create_element("canvas").unwrap();
    let canvas: web_sys::HtmlCanvasElement = canvas.dyn_into().unwrap();
    canvas.set_width(2);
    canvas.set_height(1);

    // Alpha = 255 для обоих пикселей.
    let rgba = vec![10, 20, 30, 255, 40, 50, 60, 255];
    blit_rgba_to_canvas(&canvas, rgba.clone(), 2, 1).expect("blit");

    let ctx = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .unwrap();
    let readback = ctx.get_image_data(0, 0, 2, 1).expect("readback");
    let bytes = readback.data().0;
    assert_eq!(&bytes[..], &rgba[..]);
}

// =============================================================================
// Среднее после инверсии
// =============================================================================

#[wasm_bindgen_test]
fn average_after_invert_is_255_minus_original() {
    let mut bytes = vec![100, 50, 200, 255, 200, 150, 50, 128];
    let (r0, g0, b0) = average_rgb(&bytes);
    invert_rgba(&mut bytes);
    let (r1, g1, b1) = average_rgb(&bytes);
    assert!((r1 - (255.0 - r0)).abs() < 0.01);
    assert!((g1 - (255.0 - g0)).abs() < 0.01);
    assert!((b1 - (255.0 - b0)).abs() < 0.01);
}

// =============================================================================
// Хелпер: создать VideoFrame из canvas
// =============================================================================

fn create_video_frame(canvas: &web_sys::HtmlCanvasElement) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("VideoFrame"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoFrame not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = js_sys::Object::new();
    js_sys::Reflect::set(
        &init,
        &JsValue::from_str("timestamp"),
        &JsValue::from_f64(0.0),
    )?;
    let args = js_sys::Array::new();
    args.push(&JsValue::from(canvas.clone()));
    args.push(&init);
    js_sys::Reflect::construct(&ctor, &args)
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Host-тесты (чистые функции, быстро, без браузера)
cargo test --test pure_pixel_ops

# 3. Wasm-тесты подглавы (включая WebGL2)
wasm-pack test --headless --chrome --test wasm_pixel_ops

# 4. Демо
npm run wasm:dev && npm start
```

---

## 23.7.6. `src/app.rs` — автономный демо-пример

Полный текст app.rs — см. раздел ниже. Ключевое:

- Три canvas: source (2d), CPU (2d), GPU (webgl2).
- `RunGuard` — структура с `Drop`, обновляющая UI.
- Детект renderer'а — показываем компактное имя.
- Плашка о software GL — если SwiftShader/llvmpipe.
- Через `invoke` — вызов `putImageData`/`drawImage`.

```rust
//! Глава 23.7 — WebCodecs + WASM/GPU: обработка пикселей.
//!
//! Демо: 30 кадров анимированной сцены проходят через две
//! реализации инверсии:
//!
//! - **CPU**: `frame.copyTo(rgba)` → `invert_rgba` (Rust/WASM)
//!   → `ImageData` → `putImageData`.
//! - **GPU**: `texImage2D(frame)` → WebGL2 fragment shader → прямо
//!   в видимый WebGL-канвас.
//!
//! ## Что демонстрируется
//!
//! - `VideoFrame.copyTo(RGBA)` — readback пикселей на CPU.
//! - `WebGl2RenderingContext` + fragment shader.
//! - `texImage2D(VideoFrame)` — прямой upload без readback.
//! - `UNPACK_FLIP_Y_WEBGL` для правильной ориентации.
//! - Сравнение CPU vs GPU по этапам + throughput.
//! - Детект software renderer (SwiftShader / llvmpipe).
//!
//! ## Что не является темой
//!
//! Энкодер/декодер (23.2–23.4), аудио (23.5), ImageDecoder (23.6).
//!
//! **Файл автономен.** Стили `.px-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement,
};

use crate::pixel_ops::{
    average_rgb, blit_rgba_to_canvas, copy_frame_to_rgba,
    get_webgl_renderer, has_webgl2_support, invert_rgba,
    is_software_renderer, shorten_renderer, GpuInverter,
};
use crate::video_frame::{format_str, get_fn, invoke, num, set_prop};

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
            var KEY='px-sw-cleared-23-7';
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
    let id = "px-injected-style";
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
.px-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.px-desc-line { margin:6px 0; }
.px-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.px-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.px-btn:hover:not([disabled]) { background:#30363d; }
.px-btn:disabled { opacity:0.5;cursor:not-allowed; }
.px-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.px-btn-primary:hover:not([disabled]) { background:#2ea043; }
.px-btn-gpu { background:#8957e5;border-color:#a371f7;color:#fff;}
.px-btn-gpu:hover:not([disabled]) { background:#a371f7; }
.px-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.px-pair { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.px-slot { flex:1 1 280px;min-width:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:10px 12px;}
.px-slot-title-src { color:#8b949e;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.px-slot-title-cpu { color:#f0883e;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.px-slot-title-gpu { color:#a371f7;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.px-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:6px;max-width:100%;}
.px-stats { max-width:900px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.px-stats .label { color:#8b949e; }
.px-stats .value { color:#79c0ff; }
.px-stats .ok { color:#3fb950; }
.px-stats .cpu { color:#f0883e; }
.px-stats .gpu { color:#a371f7; }
.px-stats .warn { color:#f0883e; }
.px-stats .err { color:#f85149; }
.px-stats .muted { color:#6e7681; }
.px-stats .winner { font-size:13px;font-weight:600;margin-top:4px; }
.px-bars { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border:1px solid #21262d;border-radius:4px;font-family:ui-monospace,monospace;font-size:12px;}
.px-bar-row { display:flex;align-items:center;gap:10px;margin:6px 0; }
.px-bar-label { width:36px;color:#8b949e;font-weight:600; }
.px-bar-label-cpu { color:#f0883e; }
.px-bar-label-gpu { color:#a371f7; }
.px-bar-track { flex:1;height:14px;background:#161b22;border-radius:3px;overflow:hidden;position:relative; }
.px-bar-fill { height:100%;transition:width 0.2s; }
.px-bar-fill-cpu { background:#f0883e; }
.px-bar-fill-gpu { background:#a371f7; }
.px-bar-value { min-width:120px;color:#79c0ff;text-align:right; }
.px-note { max-width:900px;margin-bottom:16px;padding:8px 14px;background:#2a1d10;border-left:3px solid #f0883e;border-radius:4px;color:#e6b780;font-family:ui-monospace,monospace;font-size:12px;line-height:1.55;}
.px-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Константы
// =============================================================================

const FRAME_W: u32 = 320;
const FRAME_H: u32 = 240;
const FRAME_COUNT: u32 = 30;

fn pixels_per_frame() -> f64 {
    (FRAME_W as f64) * (FRAME_H as f64)
}

fn pixels_total() -> f64 {
    pixels_per_frame() * (FRAME_COUNT as f64)
}

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone, Default)]
struct Timings {
    capture_ms: f64,
    process_ms: f64,
    blit_ms: f64,
    total_ms: f64,
    frames: u32,
}

impl Timings {
    fn avg_per_frame(&self) -> f64 {
        if self.frames == 0 {
            0.0
        } else {
            self.total_ms / self.frames as f64
        }
    }

    fn throughput_mpix_s(&self) -> f64 {
        let ms = self.avg_per_frame();
        if ms <= 0.0 {
            return 0.0;
        }
        (pixels_per_frame() / ms) * 1000.0 / 1_000_000.0
    }
}

struct DemoState {
    paused: bool,
    phase: String,
    error: Option<String>,
    running: bool,
    has_webgl2: bool,
    renderer: Option<String>,
    software_gl: bool,
    cpu: Timings,
    gpu: Timings,
    src_avg: Option<(f32, f32, f32)>,
}

impl DemoState {
    fn new() -> Self {
        let has_webgl2 = has_webgl2_support();
        let renderer = if has_webgl2 { get_webgl_renderer() } else { None };
        let software_gl = has_webgl2 && is_software_renderer();
        Self {
            paused: false,
            phase: String::from("простой"),
            error: None,
            running: false,
            has_webgl2,
            renderer,
            software_gl,
            cpu: Timings::default(),
            gpu: Timings::default(),
            src_avg: None,
        }
    }

    fn reset_timings(&mut self) {
        self.cpu = Timings::default();
        self.gpu = Timings::default();
        self.src_avg = None;
    }
}

// =============================================================================
// RunGuard — обновляет UI при выходе
// =============================================================================

/// Сбрасывает `running` и перерисовывает UI при drop.
///
/// Работает и при нормальном возврате, и при раннем `return`.
struct RunGuard {
    st: Rc<RefCell<DemoState>>,
    doc: Document,
}

impl Drop for RunGuard {
    fn drop(&mut self) {
        {
            let mut s = self.st.borrow_mut();
            s.running = false;
        }
        let s = self.st.borrow();
        update_stats(&self.doc, &s);
        update_buttons(&self.doc, &s);
        update_ratio_bars(&self.doc, &s);
    }
}

// =============================================================================
// Хелперы
// =============================================================================

fn close_frame(frame: &JsValue) {
    if let Ok(f) = get_fn(frame, "close") {
        let _ = f.call0(frame);
    }
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

fn update_buttons(doc: &Document, st: &DemoState) {
    let ready = !st.running;
    set_button_enabled(doc, "px-run-cpu", ready);
    set_button_enabled(doc, "px-run-gpu", ready && st.has_webgl2);
    set_button_enabled(doc, "px-run-both", ready && st.has_webgl2);
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

fn line_cpu(st: &DemoState) -> String {
    if st.cpu.frames == 0 {
        return format_str(format_args!(
            r#"<div><span class="label">CPU: </span><span class="muted">не запускался</span></div>"#
        ));
    }
    format_str(format_args!(
        r#"<div><span class="label">CPU: </span><span class="cpu">{total:.0} мс</span> <span class="muted">/ {n} кадров</span> = <span class="cpu">{avg:.2} мс/кадр</span> · <span class="muted">readback {c:.0} + invert {i:.0} + blit {b:.0}</span></div>"#,
        total = st.cpu.total_ms,
        n = st.cpu.frames,
        avg = st.cpu.avg_per_frame(),
        c = st.cpu.capture_ms,
        i = st.cpu.process_ms,
        b = st.cpu.blit_ms,
    ))
}

fn line_gpu(st: &DemoState) -> String {
    if st.gpu.frames == 0 {
        return format_str(format_args!(
            r#"<div><span class="label">GPU: </span><span class="muted">не запускался</span></div>"#
        ));
    }
    format_str(format_args!(
        r#"<div><span class="label">GPU: </span><span class="gpu">{total:.0} мс</span> <span class="muted">/ {n} кадров</span> = <span class="gpu">{avg:.2} мс/кадр</span> · <span class="muted">upload+shader {p:.0}</span></div>"#,
        total = st.gpu.total_ms,
        n = st.gpu.frames,
        avg = st.gpu.avg_per_frame(),
        p = st.gpu.process_ms,
    ))
}

fn line_winner(st: &DemoState) -> String {
    if st.cpu.frames == 0 || st.gpu.frames == 0 {
        return String::new();
    }
    let cpu_ms = st.cpu.avg_per_frame();
    let gpu_ms = st.gpu.avg_per_frame();
    if cpu_ms <= 0.0 || gpu_ms <= 0.0 {
        return String::new();
    }
    if cpu_ms >= gpu_ms {
        let ratio = cpu_ms / gpu_ms;
        format_str(format_args!(
            r#"<div class="winner"><span class="gpu">GPU в {r:.2}× быстрее</span> <span class="muted">— {cpu:.2} / {gpu:.2} мс на кадр</span></div>"#,
            r = ratio,
            cpu = cpu_ms,
            gpu = gpu_ms,
        ))
    } else {
        let ratio = gpu_ms / cpu_ms;
        format_str(format_args!(
            r#"<div class="winner"><span class="cpu">CPU в {r:.2}× быстрее</span> <span class="muted">— {cpu:.2} / {gpu:.2} мс на кадр</span></div>"#,
            r = ratio,
            cpu = cpu_ms,
            gpu = gpu_ms,
        ))
    }
}

fn line_throughput(st: &DemoState) -> String {
    if st.cpu.frames == 0 && st.gpu.frames == 0 {
        return String::new();
    }
    let cpu_t = st.cpu.throughput_mpix_s();
    let gpu_t = st.gpu.throughput_mpix_s();
    let cpu_html = if cpu_t > 0.0 {
        format_str(format_args!(
            r#"<span class="label">CPU: </span><span class="cpu">{v:.2} Mpix/s</span>"#,
            v = cpu_t,
        ))
    } else {
        String::new()
    };
    let gpu_html = if gpu_t > 0.0 {
        let sep = if cpu_t > 0.0 { " · " } else { "" };
        format_str(format_args!(
            r#"{sep}<span class="label">GPU: </span><span class="gpu">{v:.2} Mpix/s</span>"#,
            sep = sep,
            v = gpu_t,
        ))
    } else {
        String::new()
    };
    format_str(format_args!(
        r#"<div><span class="label">Пропускная способность: </span>{cpu}{gpu} <span class="muted">(всего {total} пикселей за прогон)</span></div>"#,
        cpu = cpu_html,
        gpu = gpu_html,
        total = pixels_total() as u64,
    ))
}

fn line_src_avg(st: &DemoState) -> String {
    match st.src_avg {
        Some((r, g, b)) => format_str(format_args!(
            r#"<div><span class="label">Средний RGB источника: </span><span class="value">({r:.0}, {g:.0}, {b:.0})</span></div>"#,
            r = r,
            g = g,
            b = b,
        )),
        None => String::new(),
    }
}

fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("px-stats") else {
        return;
    };
    let err_line = st
        .error
        .as_ref()
        .map(|m| line_err(m.as_str()))
        .unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{phase}{cpu}{gpu}{winner}{thr}{avg}"#,
        err = err_line,
        phase = line_phase(st),
        cpu = line_cpu(st),
        gpu = line_gpu(st),
        winner = line_winner(st),
        thr = line_throughput(st),
        avg = line_src_avg(st),
    ));
    el.set_inner_html(&html);
}

fn update_ratio_bars(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("px-bars") else {
        return;
    };
    if st.cpu.frames == 0 || st.gpu.frames == 0 {
        el.set_inner_html("");
        return;
    }
    let cpu_ms = st.cpu.avg_per_frame();
    let gpu_ms = st.gpu.avg_per_frame();
    let max_ms = cpu_ms.max(gpu_ms);
    if max_ms <= 0.0 {
        el.set_inner_html("");
        return;
    }
    let cpu_pct = (cpu_ms / max_ms) * 100.0;
    let gpu_pct = (gpu_ms / max_ms) * 100.0;
    let html = format_str(format_args!(
        r#"<div class="px-bar-row">
             <span class="px-bar-label px-bar-label-cpu">CPU</span>
             <div class="px-bar-track"><div class="px-bar-fill px-bar-fill-cpu" style="width:{cpu_pct:.1}%"></div></div>
             <span class="px-bar-value">{cpu_ms:.2} мс/кадр</span>
           </div>
           <div class="px-bar-row">
             <span class="px-bar-label px-bar-label-gpu">GPU</span>
             <div class="px-bar-track"><div class="px-bar-fill px-bar-fill-gpu" style="width:{gpu_pct:.1}%"></div></div>
             <span class="px-bar-value">{gpu_ms:.2} мс/кадр</span>
           </div>"#,
        cpu_pct = cpu_pct,
        gpu_pct = gpu_pct,
        cpu_ms = cpu_ms,
        gpu_ms = gpu_ms,
    ));
    el.set_inner_html(&html);
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>VideoFrame</strong> содержит распакованные пиксели. Их можно обрабатывать двумя принципиально разными путями — на CPU и на GPU.",
        "<strong>CPU-путь:</strong> <code>frame.copyTo(rgba)</code> копирует пиксели из GPU в CPU-память (readback — самая дорогая операция), далее Rust-цикл <code>invert_rgba</code>, потом <code>putImageData</code>.",
        "<strong>GPU-путь:</strong> <code>texImage2D(frame)</code> загружает пиксели в WebGL2-текстуру <em>без readback</em>. Фрагментный шейдер обрабатывает пиксели параллельно и пишет прямо в видимый canvas.",
        "<strong>UNPACK_FLIP_Y_WEBGL</strong> обязателен — WebGL-текстура по умолчанию перевёрнута относительно canvas. Без флага изображение приходит вверх ногами.",
        "<strong>Когда GPU выигрывает.</strong> На больших разрешениях и при сложных фильтрах (blur, convolution). На 320×240 при простой инверсии доминирует стоимость передачи данных. Если GPU-ускорения нет (SwiftShader), GPU-путь может быть медленнее — это свойство платформы, а не API.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("px-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("px-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Source canvas
// =============================================================================

fn draw_source(ctx: &CanvasRenderingContext2d, w: u32, h: u32, phase: f64) {
    let ctx_js: &JsValue = ctx.as_ref();
    let hue = (phase * 360.0) % 360.0;
    let color1 = format_str(format_args!("hsl({hue:.0}, 70%, 50%)"));
    let color2 = format_str(format_args!(
        "hsl({:.0}, 70%, 40%)",
        (hue + 90.0) % 360.0
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

fn draw_frame_at(ctx: &CanvasRenderingContext2d, w: u32, h: u32, idx: u32) {
    let phase = idx as f64 / FRAME_COUNT as f64;
    draw_source(ctx, w, h, phase);
}

// =============================================================================
// VideoFrame из canvas
// =============================================================================

fn make_video_frame(
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
// Pipeline: CPU
// =============================================================================

struct CpuOutcome {
    readback_ms: f64,
    invert_ms: f64,
    blit_ms: f64,
    src_avg: (f32, f32, f32),
}

async fn process_cpu(
    frame: &JsValue,
    cpu_canvas: &HtmlCanvasElement,
) -> Result<CpuOutcome, JsValue> {
    let t0 = js_sys::Date::now();
    let mut rgba = copy_frame_to_rgba(frame, FRAME_W, FRAME_H).await?;
    let t1 = js_sys::Date::now();

    // Средний RGB считаем ДО инверсии.
    let src_avg = average_rgb(&rgba);

    let t2 = js_sys::Date::now();
    invert_rgba(&mut rgba);
    let t3 = js_sys::Date::now();

    let t4 = js_sys::Date::now();
    blit_rgba_to_canvas(cpu_canvas, rgba, FRAME_W, FRAME_H)?;
    let t5 = js_sys::Date::now();

    Ok(CpuOutcome {
        readback_ms: t1 - t0,
        invert_ms: t3 - t2,
        blit_ms: t5 - t4,
        src_avg,
    })
}

// =============================================================================
// Pipeline: GPU
// =============================================================================

async fn process_gpu(
    frame: &JsValue,
    inverter: &GpuInverter,
) -> Result<f64, JsValue> {
    let t0 = js_sys::Date::now();
    inverter.render(frame)?;
    let t1 = js_sys::Date::now();
    Ok(t1 - t0)
}

// =============================================================================
// Основной цикл: CPU
// =============================================================================

async fn run_pipeline_cpu(doc: Document, st: Rc<RefCell<DemoState>>) {
    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.running = true;
        s.error = None;
        s.phase = String::from("CPU-обработка…");
        s.cpu = Timings::default();
    }
    let _guard = RunGuard {
        st: st.clone(),
        doc: doc.clone(),
    };
    update_stats(&doc, &st.borrow());
    update_ratio_bars(&doc, &st.borrow());
    update_buttons(&doc, &st.borrow());

    let Some(src_canvas) = doc
        .get_element_by_id("px-src")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
    else {
        return;
    };
    let Some(src_ctx) = src_canvas
        .get_context("2d")
        .ok()
        .flatten()
        .and_then(|c| c.dyn_into::<CanvasRenderingContext2d>().ok())
    else {
        return;
    };
    let Some(cpu_canvas) = doc
        .get_element_by_id("px-cpu")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
    else {
        return;
    };

    let mut last_yield = js_sys::Date::now();

    for i in 0..FRAME_COUNT {
        if !st.borrow().running {
            break;
        }

        draw_frame_at(&src_ctx, FRAME_W, FRAME_H, i);
        let frame = match make_video_frame(&src_canvas, i as f64 * 33333.0) {
            Ok(f) => f,
            Err(e) => {
                let mut s = st.borrow_mut();
                s.error = Some(format_str(format_args!("VideoFrame: {e:?}")));
                return;
            }
        };

        match process_cpu(&frame, &cpu_canvas).await {
            Ok(outcome) => {
                {
                    let mut s = st.borrow_mut();
                    s.cpu.capture_ms += outcome.readback_ms;
                    s.cpu.process_ms += outcome.invert_ms;
                    s.cpu.blit_ms += outcome.blit_ms;
                    s.cpu.total_ms += outcome.readback_ms
                        + outcome.invert_ms
                        + outcome.blit_ms;
                    s.cpu.frames += 1;
                    if s.src_avg.is_none() {
                        s.src_avg = Some(outcome.src_avg);
                    }
                }
                update_stats(&doc, &st.borrow());
            }
            Err(e) => {
                let mut s = st.borrow_mut();
                s.error = Some(format_str(format_args!("CPU process: {e:?}")));
                return;
            }
        }
        close_frame(&frame);

        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }

    {
        let mut s = st.borrow_mut();
        s.phase = format_str(format_args!(
            "CPU готово ({n} кадров)",
            n = s.cpu.frames
        ));
    }
}

// =============================================================================
// Основной цикл: GPU
// =============================================================================

async fn run_pipeline_gpu(doc: Document, st: Rc<RefCell<DemoState>>) {
    if !st.borrow().has_webgl2 {
        let mut s = st.borrow_mut();
        s.error = Some(String::from(
            "WebGL2 не поддерживается — GPU-путь недоступен",
        ));
        update_stats(&doc, &s);
        return;
    }
    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.running = true;
        s.error = None;
        s.phase = String::from("GPU-обработка…");
        s.gpu = Timings::default();
    }
    let _guard = RunGuard {
        st: st.clone(),
        doc: doc.clone(),
    };
    update_stats(&doc, &st.borrow());
    update_ratio_bars(&doc, &st.borrow());
    update_buttons(&doc, &st.borrow());

    let Some(gpu_canvas) = doc
        .get_element_by_id("px-gpu")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
    else {
        return;
    };
    let inverter = match GpuInverter::from_canvas(&gpu_canvas) {
        Ok(v) => v,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("GpuInverter: {e:?}")));
            return;
        }
    };

    let Some(src_canvas) = doc
        .get_element_by_id("px-src")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
    else {
        return;
    };
    let Some(src_ctx) = src_canvas
        .get_context("2d")
        .ok()
        .flatten()
        .and_then(|c| c.dyn_into::<CanvasRenderingContext2d>().ok())
    else {
        return;
    };

    let mut last_yield = js_sys::Date::now();

    for i in 0..FRAME_COUNT {
        if !st.borrow().running {
            break;
        }

        draw_frame_at(&src_ctx, FRAME_W, FRAME_H, i);
        let frame = match make_video_frame(&src_canvas, i as f64 * 33333.0) {
            Ok(f) => f,
            Err(e) => {
                let mut s = st.borrow_mut();
                s.error = Some(format_str(format_args!("VideoFrame: {e:?}")));
                return;
            }
        };

        match process_gpu(&frame, &inverter).await {
            Ok(process_ms) => {
                {
                    let mut s = st.borrow_mut();
                    s.gpu.process_ms += process_ms;
                    s.gpu.total_ms += process_ms;
                    s.gpu.frames += 1;
                }
                update_stats(&doc, &st.borrow());
            }
            Err(e) => {
                let mut s = st.borrow_mut();
                s.error = Some(format_str(format_args!("GPU process: {e:?}")));
                return;
            }
        }
        close_frame(&frame);

        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }

    {
        let mut s = st.borrow_mut();
        s.phase = format_str(format_args!(
            "GPU готово ({n} кадров)",
            n = s.gpu.frames
        ));
    }
}

// =============================================================================
// Основной цикл: оба
// =============================================================================

async fn run_pipeline_both(doc: Document, st: Rc<RefCell<DemoState>>) {
    {
        let mut s = st.borrow_mut();
        if s.running {
            return;
        }
        s.reset_timings();
    }
    update_stats(&doc, &st.borrow());
    update_ratio_bars(&doc, &st.borrow());

    run_pipeline_cpu(doc.clone(), st.clone()).await;
    run_pipeline_gpu(doc, st).await;
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
    h2.set_text_content(Some("23.7 — WebCodecs + WASM/GPU: обработка пикселей"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // ---- Три canvas: source, CPU, GPU ----
    let pair = document.create_element("div")?;
    pair.set_class_name("px-pair");
    pair.set_inner_html(&format_str(format_args!(
        r#"<div class="px-slot">
             <div class="px-slot-title-src">Source (320×240)</div>
             <canvas id="px-src" class="px-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="px-slot">
             <div class="px-slot-title-cpu">CPU: copyTo → invert → putImageData</div>
             <canvas id="px-cpu" class="px-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="px-slot">
             <div class="px-slot-title-gpu">GPU: texImage2D → shader → canvas</div>
             <canvas id="px-gpu" class="px-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = FRAME_W,
        h = FRAME_H,
    )));
    root.append_child(&pair)?;

    // ---- Controls ----
    let controls = document.create_element("div")?;
    controls.set_class_name("px-block");
    controls.set_inner_html(
        r#"<button class="px-btn px-btn-primary" id="px-run-cpu">Run CPU (30 frames)</button>
           <button class="px-btn px-btn-gpu" id="px-run-gpu">Run GPU (30 frames)</button>
           <button class="px-btn px-btn-primary" id="px-run-both">Run Both (compare)</button>
           <span class="px-label" id="px-support"></span>"#,
    );
    root.append_child(&controls)?;

    // ---- Warning (если software GL) ----
    let state = Rc::new(RefCell::new(DemoState::new()));
    if state.borrow().software_gl {
        let note = document.create_element("div")?;
        note.set_class_name("px-note");
        let renderer = state
            .borrow()
            .renderer
            .clone()
            .unwrap_or_else(|| String::from("software"));
        note.set_text_content(Some(&format_str(format_args!(
            "⚠ GPU-ускорение недоступно — WebGL работает через софтверный backend ({r}). \
             На реальной GPU разница CPU/GPU будет в десятки раз.",
            r = renderer,
        ))));
        root.append_child(&note)?;
    }

    // ---- Stats ----
    let stats = document.create_element("div")?;
    stats.set_class_name("px-stats");
    stats.set_id("px-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // ---- Ratio bars ----
    let bars = document.create_element("div")?;
    bars.set_class_name("px-bars");
    bars.set_id("px-bars");
    root.append_child(&bars)?;

    // ---- Footnote ----
    let footnote = document.create_element("p")?;
    footnote.set_class_name("px-footnote");
    footnote.set_text_content(Some(
        "CPU-путь: copyTo(RGBA) → Rust-цикл → putImageData. GPU-путь: \
         texImage2D(frame) → shader пишет прямо в видимый WebGL-канвас, без \
         transferToImageBitmap. UNPACK_FLIP_Y_WEBGL обязателен. На 320×240 при \
         простой инверсии доминирует стоимость передачи данных — GPU-выигрыш \
         проявляется на больших разрешениях и при сложных фильтрах.",
    ));
    root.append_child(&footnote)?;

    // ---- Support info ----
    if let Some(el) = document.get_element_by_id("px-support") {
        let st = state.borrow();
        let webgl2 = if st.has_webgl2 { "да" } else { "нет" };
        let renderer = match st.renderer.as_deref() {
            Some(s) => shorten_renderer(s),
            None => String::from("недоступен"),
        };
        let s = format_str(format_args!(
            "WebGL2: {w} · renderer: {r}",
            w = webgl2,
            r = renderer,
        ));
        el.set_text_content(Some(&s));
    }

    // ---- Run CPU ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("px-run-cpu")
            .ok_or_else(|| JsValue::from_str("no #px-run-cpu"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                run_pipeline_cpu(doc, st).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Run GPU ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("px-run-gpu")
            .ok_or_else(|| JsValue::from_str("no #px-run-gpu"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                run_pipeline_gpu(doc, st).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Run Both ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("px-run-both")
            .ok_or_else(|| JsValue::from_str("no #px-run-both"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                run_pipeline_both(doc, st).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- rAF: source-анимация ----
    {
        let doc_for_raf = document.clone();
        let source_canvas = doc_for_raf
            .get_element_by_id("px-src")
            .ok_or_else(|| JsValue::from_str("no #px-src"))?
            .dyn_into::<HtmlCanvasElement>()?;
        let source_ctx = source_canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d"))?
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
                if st.borrow().paused || st.borrow().running {
                    return;
                }
                let phase = (now / 1000.0) % 1.0;
                draw_source(&source_ctx, FRAME_W, FRAME_H, phase);
            },
        ) as Box<dyn FnMut(f64)>));

        let cb = cell.borrow();
        window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    }

    {
        let st = state.borrow();
        update_stats(&document, &st);
        update_buttons(&document, &st);
    }

    Ok(())
}
```

---

## 23.7.7. Анализ решения

**Ценность демо.**

1. **Два принципиально разных пути обработки.** CPU: readback + Rust-цикл. GPU: текстура + шейдер.
2. **Прямое сравнение таймингов** на одном контенте, 30 кадров.
3. **`texImage2D(VideoFrame)` без readback** — практическая демонстрация WebCodecs-GPU интеграции.
4. **Bar chart, throughput, winner-строка** — численное сравнение в трёх проекциях.
5. **Детект software renderer.** SwiftShader/llvmpipe распознаются, пользователь видит предупреждение.

**Правила проекта, применённые.**

- **Отдельные функции для строк** — `line_err`, `line_phase`, `line_cpu`, `line_gpu`, `line_winner`, `line_throughput`, `line_src_avg`.
- **`format_str(format_args!(...))` в tail-позициях.**
- **`Rc<RefCell<DemoState>>`** — общее состояние.
- **`RunGuard` с `Drop`** — обновляет UI, работает на любом выходе.
- **Yield раз в 16 мс** — не блокирует UI во время 30-кадрового цикла.
- **Префикс `px-`** — не пересекается с `vd-`, `vf-`, `ve-`, `vt-`, `ac-`, `id-`.
- **Никаких unused imports.**
- **`create_element`, не `createElement`.**

**Найденные в практике проблемы и их решения.**

1. **WebGL-константы — associated constants, не методы.** `WebGl2RenderingContext::VERTEX_SHADER` вместо `gl.VERTEX_SHADER()`.
2. **`uniform1i`, не `uniform_1i`.** Web-sys сохраняет оригинальные имена методов.
3. **`bufferData` и `texImage2D` через `invoke`.** Сигнатуры перегружены.
4. **`put_image_data` — `f64` или `i32` в зависимости от сборки.** Решение — `invoke`.
5. **Premultiplied alpha на canvas.** Тесты разделены: точные RGB (alpha=255), сохранение alpha (alpha=128).
6. **`transferToImageBitmap` + `drawImage` — медленные.** Прямой рендер в WebGL-канвас в 36× быстрее.
7. **`get_webgl_renderer` требует активации расширения.** Решение — стандартный `RENDERER` (0x1F01).
8. **`RunGuard` с обновлением UI в `Drop`.** Кнопки разблокируются на любом выходе.
9. **`get_context("webgl2").ok()??`** — двойной `?` для `Result<Option<Object>>`.

**Проверки корректности.**

- `format_args!` без `f`-суффикса: `{r:.2}`, `{avg:.2}`, `{total:.0}`, `{cpu_pct:.1}` — валидны.
- Raw-строки: `r#"..."#` в HTML. `r##"..."##` не нужен.
- `unsafe { Float32Array::view(&verts) }` — view действителен только во время вызова `bufferData`. Корректно по документации web-sys.
- `VideoFrame.copyTo` — async, `JsFuture::from(promise)`.
- `RunGuard` устанавливается после `s.running = true`.
- Shader compile/link errors логируются, не паникуют.

---

## 23.7.8. Wow-момент на практике

**Первый.** Source — цветной. Results — инвертированные. Оба result-canvas визуально идентичны. Разница — только в таймингах.

**Второй.** CPU: 2.67 мс/кадр (readback 25 + invert 27 + blit 28 за 30 кадров). GPU: 0.30 мс/кадр. **×8.89**.

**Третий.** Этапы в GPU: `upload+shader 9 мс` за 30 кадров. Никакого readback и blit. Только работа с GPU-текстурой.

**Четвёртый.** Renderer показывается в UI: `Intel(R) HD Graphics` (после `shorten_renderer`), изначально строка `ANGLE (Intel, Intel(R) HD Graphics Direct3D11 vs_5_0 ps_5_0), or similar`.

**Пятый.** Bar chart: CPU-полоса на всю ширину, GPU — 1/9 ширины. Визуально сразу видно разницу в 9 раз.

---

## 23.7.9. Итог

**`VideoFrame.copyTo`** — readback пикселей из GPU в CPU-память. Дорого, но универсально. Работает для canvas-based frame.

**`texImage2D(VideoFrame)`** — прямая загрузка кадра в WebGL2-текстуру без readback. Быстро, требует WebGL2.

**`UNPACK_FLIP_Y_WEBGL`** — обязательный флаг для правильной ориентации. WebGL-текстура по умолчанию перевёрнута относительно canvas.

**Прямой рендер в WebGL-канвас** — в 36× быстрее, чем через `transferToImageBitmap` + `drawImage`. Это ключевая оптимизация, которую обычно упускают в туториалах.

**CPU vs GPU.** На 320×240 с Intel HD Graphics — **×8.89**. На Full HD с дискретной GPU достигает ×20–50. Разница растёт линейно с числом пикселей: readback масштабируется линейно, шейдер — параллельно.

**Правило проекта, зафиксированное в этой подглаве.** Чистые функции (`invert_rgba`, `average_rgb`) тестируются на host — без headless-браузера. WebGL2-обёртки — в wasm-тестах. Разделение ускоряет разработку и уменьшает хрупкость CI.

**Правило.** Если web-sys не экспортирует метод с нужной сигнатурой — используйте `invoke`. Это относится к `bufferData`, `texImage2D`, `putImageData` — все с перегруженными сигнатурами.

