# Часть XII. GPU и WebGL

## Глава 42. WebGL

---

## Введение

WebGL — низкоуровневый API для доступа к GPU из браузера. Он основан на OpenGL ES 2.0/3.0 и предоставляет **программируемый конвейер**: вы пишете два шейдера (вершинный и фрагментный), загружаете данные в буферы, настраиваете uniform-переменные — и GPU выполняет миллионы параллельных операций за один кадр.

Для Rust-разработчика WebGL — это **граница между CPU и GPU**. Всё, что в главах 38–41 работало на CPU (криптография, аутентификация), здесь переезжает на специализированный процессор. Задача программиста — правильно выстроить pipeline и передать данные. Вся тяжёлая работа — на GPU.

Ключевые отличия от других глав:

| Аспект | Web Crypto (гл. 38) | WebGL (гл. 42) |
|---|---|---|
| Где работает | CPU | GPU |
| Параллелизм | 1 поток | Тысячи ядер |
| Модель | Последовательные вызовы | Программируемый конвейер |
| Сложность | Многословность API | Многословность pipeline |
| Синхронность | Promise | Immediate + кадры |
| Ключевые понятия | AES, RSA, ECDSA | Shaders, VBO, VAO, uniforms |

В этой главе мы покроем 10 тем:

1. WebGL architecture
2. `WebGlRenderingContext` / `WebGl2RenderingContext`
3. shaders (vertex + fragment)
4. buffers (VBO + VAO)
5. textures
6. programs
7. uniforms (float / vec2 / vec3)
8. rendering pipeline
9. Rust + WebGL
10. Практический GPU application

---

## 1. Что мы узнали при сборке: 4 грабли WebGL

### Грабли 1. `get_attrib_location` возвращает `i32`, а не `u32`

Кажется логичным, что индекс атрибута — неотрицательное число. Но `WebGl2RenderingContext::get_attrib_location` возвращает **`i32`**, а все последующие вызовы (`enable_vertex_attrib_array`, `vertex_attrib_pointer_with_i32`) принимают **`u32`**. Приведение нужно делать явно:

```rust
let loc_i32 = gl.get_attrib_location(&program, name);
let loc = loc_i32 as u32;
```

Мы сделали это один раз при кэшировании в `Program::attrib_location`, чтобы не кастовать каждый кадр.

### Грабли 2. Raw-строка `r#"..."#` ломается на `"#` в HTML

Классический случай — атрибут `value="#ffffff"` у `<input type="color">`. Последовательность `"#` завершает raw-строку `r#"..."#` **прямо в середине HTML**, и парсер Rust начинает видеть CSS/HTML как Rust-код.

**Решение:** использовать более длинный разделитель:

```rust
const HTML: &str = r##"
    <input type="color" id="g-tint" value="#ffffff">
"##;
```

Терминатор теперь `"##`, и `"#` внутри содержимого безопасен. Если внутри есть `"##` — использовать `r###"..."###` и так далее.

### Грабли 3. GPU не бросает исключения — он ставит флаг ошибки

В отличие от JS, где ошибка — это `throw`, WebGL работает по модели **silent errors**: любая некорректная операция не останавливает выполнение, а устанавливает флаг, который можно прочитать через `gl.get_error()`. Если не читать — ошибки теряются молча, и вы будете искать причину «чёрного экрана» часами.

**Решение:** хелпер `check_error(gl)` вызывается после каждого draw call:

```rust
pub fn check_error(gl: &web_sys::WebGl2RenderingContext) -> Option<String> {
    let err = gl.get_error();
    if err == web_sys::WebGl2RenderingContext::NO_ERROR {
        return None;
    }
    let name = match err {
        web_sys::WebGl2RenderingContext::INVALID_ENUM => "INVALID_ENUM",
        web_sys::WebGl2RenderingContext::INVALID_VALUE => "INVALID_VALUE",
        web_sys::WebGl2RenderingContext::INVALID_OPERATION => "INVALID_OPERATION",
        // ...
    };
    return Some(format_str(format_args!("GL error: {name} (0x{err:x})")));
}
```

### Грабли 4. `Program::new` не должен зависеть от контекста

Первая версия `Program::new(gl, raw)` принимала контекст, но не использовала его. После рефакторинга параметр убрали — `Program::new(raw)`. Это тот случай, когда **уборка dead-параметра** предотвращает будущую путаницу: читатель видит, что программе нужен только `WebGlProgram`, а не весь контекст.

---

## 2. Архитектура модуля

```
src/
├── lib.rs                       ← pub mod webgl
├── app.rs                       ← демо-пример главы
└── webgl/
    ├── mod.rs                   ← WebGlError, context(), check_error()
    ├── shader.rs                ← компиляция и линковка шейдеров
    ├── buffer.rs                ← VBO, EBO, VAO
    ├── program.rs               ← Program с кэшем uniform/attrib локаций
    ├── texture.rs               ← загрузка ImageData в текстуру
    └── geometry.rs              ← генерация геометрии
```

Шесть модулей, каждый — одна ответственность. `WebGlError` оборачивает `JsValue` в человекочитаемое сообщение.

---

## 3. Rendering pipeline — что реально происходит

Каждый кадр выполняется **6 шагов**:

1. **`gl.useProgram(program)`** — выбрать активную программу.
2. **`gl.bindVertexArray(vao)`** — привязать сохранённое состояние атрибутов.
3. **`gl.uniform1f/uniform2f/uniform3f(...)`** — передать данные из JS в шейдер.
4. **`gl.drawArrays(TRIANGLES, 0, N)`** — запустить GPU-конвейер.
5. **`gl.getError()`** — проверить, не было ли ошибок.
6. **`requestAnimationFrame`** — запланировать следующий кадр.

### Программируемый конвейер

```
Вершины → вершинный шейдер → растеризация → фрагментный шейдер → пиксели
```

**Вершинный шейдер** вызывается для каждой вершины: он вычисляет финальную позицию на экране.
**Фрагментный шейдер** — для каждого пикселя внутри треугольника: вычисляет цвет.
Оба исполняются **параллельно** на сотнях ядер GPU.

### Ключевой урок — draw call overhead

Один `drawArrays` на 5000 треугольников работает в 10–100 раз быстрее, чем 5000 отдельных вызовов. Причина — **фиксированная стоимость каждого draw call** (переключение контекста JS↔GPU). Это самая важная производительностная концепция WebGL, и мы демонстрируем её прямо в UI.

---

## 4. Uniform-переменные

Uniform — это переменная, которая передаётся из JS в шейдер **один раз для всего draw call**. Она одинаково видна всем вершинам и всем фрагментам.

| Тип | Rust-вызов | Что передаёт |
|---|---|---|
| `float` | `gl.uniform1f(loc, x)` | Время, частота |
| `vec2` | `gl.uniform2f(loc, x, y)` | Сдвиг, UV |
| `vec3` | `gl.uniform3f(loc, r, g, b)` | Цвет, нормаль |
| `mat4` | `gl.uniform_matrix4fv_with_f32_array(...)` | Трансформация |

В демо мы используем три типа uniform: `u_time`, `u_frequency` (float), `u_offset` (vec2), `u_tint` (vec3). Это показывает, что «данные из Rust» — не просто число, а любой тип, поддерживаемый GLSL.

---

## 5. VAO — сохранение состояния атрибутов

Каждый атрибут вершины (позиция, цвет) должен быть «привязан» к буферу через `vertexAttribPointer`. Без VAO эту привязку нужно делать каждый кадр — медленно и неудобно.

**Vertex Array Object (VAO)** сохраняет всё состояние атрибутов один раз:

```rust
let vao = gl.create_vertex_array()?;
gl.bind_vertex_array(Some(&vao));
gl.bind_buffer(ARRAY_BUFFER, Some(&vbo));
gl.enable_vertex_attrib_array(pos_attr);
gl.vertex_attrib_pointer_with_i32(pos_attr, 2, FLOAT, false, stride, 0);
gl.enable_vertex_attrib_array(col_attr);
gl.vertex_attrib_pointer_with_i32(col_attr, 3, FLOAT, false, stride, offset);
gl.bind_vertex_array(None);
```

Теперь `draw` — это три строки:

```rust
program.use_program(gl);
gl.bind_vertex_array(Some(&vao));
gl.draw_arrays(TRIANGLES, 0, vertex_count);
```

VAO доступен только в WebGL2 / OpenGL ES 3.0. В WebGL1 (ES 2.0) его нет — отсюда и выбор `WebGl2RenderingContext` для демо.

---

## 6. Демо: два режима рендера

Главная ценность демо — **сравнение двух стратегий**:

**Single (big VBO)** — один `drawArrays` на N треугольников. Все позиции и цвета вшиты в VBO. Один вызов JS→GPU.

**CPU loop (N draw calls)** — VBO содержит один белый треугольник, а позиция и цвет каждого инстанса передаются через uniform в цикле. N вызовов JS→GPU.

Переключение режима и слайдер N (1..5000) показывают **реальную разницу в FPS**. При 5000 инстансах:

- `Single` — плавные 60 FPS.
- `CPU loop` — 20–40 FPS и падение.

Это интеллектуальная демонстрация: пользователь сам двигает слайдер и видит, как меняется производительность. Урок входит через цифры, а не через текст.

---

## 7. Rust + WebGL

`web-sys` даёт типизированные обёртки:

- `WebGl2RenderingContext` — контекст.
- `WebGlProgram`, `WebGlShader` — программа и шейдер.
- `WebGlBuffer`, `WebGlVertexArrayObject` — буферы и VAO.
- `WebGlUniformLocation` — локация uniform.

Типобезопасность — реальное преимущество: `WebGlBuffer` не спутать с `WebGlTexture`, `WebGlProgram` не передать туда, где ожидается `WebGlShader`. Компилятор ловит ошибки, которые в JS всплывут только в рантайме.

Плюс `js_sys::Float32Array::from(data)` копирует память Rust в GPU без лишних аллокаций в JS-куче.

---

## 8. Исходные коды

### `Cargo.toml` (изменения)

```toml
[dependencies.web-sys]
features = [
    # ... существующие features ...

    # WebGL (глава 42)
    "WebGl2RenderingContext",
    "WebGlProgram",
    "WebGlShader",
    "WebGlBuffer",
    "WebGlUniformLocation",
    "WebGlTexture",
    "WebGlVertexArrayObject",
]
```

### `src/lib.rs`

```rust
//! Hello WASM02 — курс по Rust + WebAssembly.
//!
//! Глава 42: WebGL.

pub mod format_str;
pub mod app;
pub mod web_crypto;   // из главы 38
pub mod webauthn;     // глава 41
pub mod webgl;        // глава 42
```

### `src/webgl/mod.rs`

```rust
//! Глава 42: WebGL — GPU-рендеринг в браузере.
//!
//! WebGL — низкоуровневый API для доступа к GPU из JavaScript.
//! Он основан на OpenGL ES 2.0/3.0 и предоставляет программируемый
//! конвейер: вершинные и фрагментные шейдеры, буферы, текстуры,
//! uniform-переменные.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

pub mod shader;
pub mod buffer;
pub mod program;
pub mod texture;
pub mod geometry;

use wasm_bindgen::JsValue;
use wasm_bindgen::JsCast;
use crate::format_str::format_str;

/// Универсальная ошибка WebGL-операций.
#[derive(Debug, Clone)]
pub struct WebGlError {
    pub message: String,
}

impl WebGlError {
    pub fn new(msg: String) -> Self {
        return Self { message: msg };
    }
}

impl std::fmt::Display for WebGlError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return write!(f, "{}", self.message);
    }
}

impl From<JsValue> for WebGlError {
    fn from(v: JsValue) -> Self {
        let msg = v
            .as_string()
            .unwrap_or_else(|| format_str(format_args!("{v:?}")));
        return Self { message: msg };
    }
}

impl From<String> for WebGlError {
    fn from(s: String) -> Self {
        return Self { message: s };
    }
}

/// Получает WebGL2-контекст из canvas.
///
/// WebGL2 предпочтительнее: он даёт VAO, instancing, 3D-текстуры
/// и другие возможности OpenGL ES 3.0.
pub fn context(
    canvas: &web_sys::HtmlCanvasElement,
) -> Result<web_sys::WebGl2RenderingContext, WebGlError> {
    let ctx = canvas
        .get_context("webgl2")
        .map_err(|e| WebGlError::from(e))?
        .ok_or_else(|| WebGlError::new("webgl2 context unavailable".to_string()))?;
    let gl: web_sys::WebGl2RenderingContext = ctx
        .dyn_into()
        .map_err(|_| WebGlError::new("not a WebGl2RenderingContext".to_string()))?;
    return Ok(gl);
}

/// Читает код ошибки из GL и возвращает читаемую строку.
///
/// `0` (NO_ERROR) — всё в порядке, возвращаем `None`.
/// Любое другое значение — ошибка, возвращаем её имя и hex-код.
///
/// ПРАВИЛО: после каждого draw-вызова проверяйте `check_error`.
/// GPU не бросает исключения, он молча ставит флаг ошибки.
pub fn check_error(gl: &web_sys::WebGl2RenderingContext) -> Option<String> {
    let err = gl.get_error();
    if err == web_sys::WebGl2RenderingContext::NO_ERROR {
        return None;
    }
    let name = match err {
        web_sys::WebGl2RenderingContext::INVALID_ENUM => "INVALID_ENUM",
        web_sys::WebGl2RenderingContext::INVALID_VALUE => "INVALID_VALUE",
        web_sys::WebGl2RenderingContext::INVALID_OPERATION => "INVALID_OPERATION",
        web_sys::WebGl2RenderingContext::OUT_OF_MEMORY => "OUT_OF_MEMORY",
        web_sys::WebGl2RenderingContext::INVALID_FRAMEBUFFER_OPERATION => {
            "INVALID_FRAMEBUFFER_OPERATION"
        }
        _ => "UNKNOWN",
    };
    return Some(format_str(format_args!("GL error: {name} (0x{err:x})")));
}
```

### `src/webgl/shader.rs`

```rust
//! Компиляция и линковка шейдеров.

use crate::format_str::format_str;
use super::WebGlError;

/// Компилирует вершинный шейдер из исходного кода.
pub fn compile_vertex(
    gl: &web_sys::WebGl2RenderingContext,
    source: &str,
) -> Result<web_sys::WebGlShader, WebGlError> {
    return compile(gl, source, web_sys::WebGl2RenderingContext::VERTEX_SHADER);
}

/// Компилирует фрагментный шейдер из исходного кода.
pub fn compile_fragment(
    gl: &web_sys::WebGl2RenderingContext,
    source: &str,
) -> Result<web_sys::WebGlShader, WebGlError> {
    return compile(gl, source, web_sys::WebGl2RenderingContext::FRAGMENT_SHADER);
}

fn compile(
    gl: &web_sys::WebGl2RenderingContext,
    source: &str,
    kind: u32,
) -> Result<web_sys::WebGlShader, WebGlError> {
    let shader = gl
        .create_shader(kind)
        .ok_or_else(|| WebGlError::new("create_shader failed".to_string()))?;
    gl.shader_source(&shader, source);
    gl.compile_shader(&shader);

    let status = gl
        .get_shader_parameter(&shader, web_sys::WebGl2RenderingContext::COMPILE_STATUS)
        .as_bool()
        .unwrap_or(false);
    if !status {
        let log = gl
            .get_shader_info_log(&shader)
            .unwrap_or_else(|| "unknown error".to_string());
        return Err(WebGlError::new(format_str(format_args!(
            "shader compile error: {log}",
        ))));
    }
    return Ok(shader);
}

/// Линкует вершинный и фрагментный шейдеры в программу.
pub fn link_program(
    gl: &web_sys::WebGl2RenderingContext,
    vs: &web_sys::WebGlShader,
    fs: &web_sys::WebGlShader,
) -> Result<web_sys::WebGlProgram, WebGlError> {
    let program = gl
        .create_program()
        .ok_or_else(|| WebGlError::new("create_program failed".to_string()))?;
    gl.attach_shader(&program, vs);
    gl.attach_shader(&program, fs);
    gl.link_program(&program);

    let status = gl
        .get_program_parameter(&program, web_sys::WebGl2RenderingContext::LINK_STATUS)
        .as_bool()
        .unwrap_or(false);
    if !status {
        let log = gl
            .get_program_info_log(&program)
            .unwrap_or_else(|| "unknown error".to_string());
        return Err(WebGlError::new(format_str(format_args!(
            "program link error: {log}",
        ))));
    }
    return Ok(program);
}
```

### `src/webgl/buffer.rs`

```rust
//! Создание и заполнение вершинных буферов (VBO) и VAO.

use super::WebGlError;

/// Создаёт буфер и загружает в него массив f32.
pub fn create_f32_buffer(
    gl: &web_sys::WebGl2RenderingContext,
    data: &[f32],
    usage: u32,
) -> Result<web_sys::WebGlBuffer, WebGlError> {
    let buffer = gl
        .create_buffer()
        .ok_or_else(|| WebGlError::new("create_buffer failed".to_string()))?;
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&buffer));

    // Создаём Float32Array поверх памяти Rust и передаём в GPU.
    let array = js_sys::Float32Array::from(data);
    gl.buffer_data_with_array_buffer_view(
        web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
        &array,
        usage,
    );
    return Ok(buffer);
}

/// Создаёт буфер для индексов (EBO).
pub fn create_index_buffer(
    gl: &web_sys::WebGl2RenderingContext,
    indices: &[u16],
    usage: u32,
) -> Result<web_sys::WebGlBuffer, WebGlError> {
    let buffer = gl
        .create_buffer()
        .ok_or_else(|| WebGlError::new("create_buffer failed".to_string()))?;
    gl.bind_buffer(
        web_sys::WebGl2RenderingContext::ELEMENT_ARRAY_BUFFER,
        Some(&buffer),
    );

    let array = js_sys::Uint16Array::from(indices);
    gl.buffer_data_with_array_buffer_view(
        web_sys::WebGl2RenderingContext::ELEMENT_ARRAY_BUFFER,
        &array,
        usage,
    );
    return Ok(buffer);
}

/// Создаёт VAO и настраивает атрибуты — состояние сохраняется
/// между draw-вызовами, не нужно переустанавливать каждый кадр.
///
/// `attrs` — срез кортежей `(location, size, stride, offset)`:
///   - `location` — индекс атрибута (`get_attrib_location`);
///   - `size`     — число компонент (1..4);
///   - `stride`   — размер шага в байтах;
///   - `offset`   — смещение внутри шага в байтах.
pub fn create_vao_with_attrs(
    gl: &web_sys::WebGl2RenderingContext,
    vbo: &web_sys::WebGlBuffer,
    attrs: &[(u32, i32, i32, i32)],
) -> Result<web_sys::WebGlVertexArrayObject, WebGlError> {
    let vao = gl
        .create_vertex_array()
        .ok_or_else(|| WebGlError::new("create_vertex_array failed".to_string()))?;

    // Привязываем VAO: все последующие вызовы enable_vertex_attrib_array
    // и vertex_attrib_pointer_with_i32 записываются в него.
    gl.bind_vertex_array(Some(&vao));
    gl.bind_buffer(
        web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
        Some(vbo),
    );

    for &(loc, size, stride, offset) in attrs {
        gl.enable_vertex_attrib_array(loc);
        gl.vertex_attrib_pointer_with_i32(
            loc,
            size,
            web_sys::WebGl2RenderingContext::FLOAT,
            false,
            stride,
            offset,
        );
    }

    // Снимаем привязку VAO, чтобы случайно не изменить его.
    gl.bind_vertex_array(None);
    return Ok(vao);
}
```

### `src/webgl/program.rs`

```rust
//! Обёртка над WebGlProgram с кэшированием uniform-локаций.

use std::collections::HashMap;

/// Программа с кэшем uniform-локаций по имени.
pub struct Program {
    pub raw: web_sys::WebGlProgram,
    uniforms: HashMap<String, Option<web_sys::WebGlUniformLocation>>,
    attributes: HashMap<String, u32>,
}

impl Program {
    pub fn new(raw: web_sys::WebGlProgram) -> Self {
        return Self {
            raw,
            uniforms: HashMap::new(),
            attributes: HashMap::new(),
        };
    }

    /// Возвращает локацию uniform-переменной (с кэшированием).
    pub fn uniform_location(
        &mut self,
        gl: &web_sys::WebGl2RenderingContext,
        name: &str,
    ) -> Option<web_sys::WebGlUniformLocation> {
        if let Some(cached) = self.uniforms.get(name) {
            return cached.clone();
        }
        let loc = gl.get_uniform_location(&self.raw, name);
        self.uniforms.insert(name.to_string(), loc.clone());
        return loc;
    }

    /// Возвращает индекс атрибута (с кэшированием).
    ///
    /// `get_attrib_location` возвращает `i32`, но все последующие
    /// вызовы `enable_vertex_attrib_array` / `vertex_attrib_pointer_with_i32`
    /// принимают `u32`. Приводим один раз при кэшировании.
    pub fn attrib_location(
        &mut self,
        gl: &web_sys::WebGl2RenderingContext,
        name: &str,
    ) -> u32 {
        if let Some(cached) = self.attributes.get(name) {
            return *cached;
        }
        let loc_i32 = gl.get_attrib_location(&self.raw, name);
        let loc = loc_i32 as u32;
        self.attributes.insert(name.to_string(), loc);
        return loc;
    }

    pub fn use_program(&self, gl: &web_sys::WebGl2RenderingContext) {
        gl.use_program(Some(&self.raw));
    }
}
```

### `src/webgl/texture.rs`

```rust
//! Загрузка текстур.

use super::WebGlError;

/// Создаёт текстуру из ImageData.
pub fn from_image_data(
    gl: &web_sys::WebGl2RenderingContext,
    image: &web_sys::ImageData,
) -> Result<web_sys::WebGlTexture, WebGlError> {
    let texture = gl
        .create_texture()
        .ok_or_else(|| WebGlError::new("create_texture failed".to_string()))?;
    gl.bind_texture(web_sys::WebGl2RenderingContext::TEXTURE_2D, Some(&texture));

    gl.tex_image_2d_with_u32_and_u32_and_image_data(
        web_sys::WebGl2RenderingContext::TEXTURE_2D,
        0,
        web_sys::WebGl2RenderingContext::RGBA as i32,
        web_sys::WebGl2RenderingContext::RGBA,
        web_sys::WebGl2RenderingContext::UNSIGNED_BYTE,
        image,
    ).map_err(|e| WebGlError::from(e))?;

    gl.tex_parameteri(
        web_sys::WebGl2RenderingContext::TEXTURE_2D,
        web_sys::WebGl2RenderingContext::TEXTURE_MIN_FILTER,
        web_sys::WebGl2RenderingContext::LINEAR as i32,
    );
    gl.tex_parameteri(
        web_sys::WebGl2RenderingContext::TEXTURE_2D,
        web_sys::WebGl2RenderingContext::TEXTURE_MAG_FILTER,
        web_sys::WebGl2RenderingContext::LINEAR as i32,
    );

    return Ok(texture);
}
```

### `src/webgl/geometry.rs`

```rust
//! Генерация геометрии.

/// Треугольник с цветами вершин (RGB).
pub fn triangle() -> Vec<f32> {
    return vec![
        // x, y, r, g, b
         0.0,  0.6, 1.0, 0.0, 0.0,
        -0.6, -0.4, 0.0, 1.0, 0.0,
         0.6, -0.4, 0.0, 0.0, 1.0,
    ];
}

/// Полноэкранный квад (2 треугольника) с UV-координатами.
pub fn quad() -> Vec<f32> {
    return vec![
        // x, y, u, v
        -1.0, -1.0, 0.0, 0.0,
         1.0, -1.0, 1.0, 0.0,
        -1.0,  1.0, 0.0, 1.0,
        -1.0,  1.0, 0.0, 1.0,
         1.0, -1.0, 1.0, 0.0,
         1.0,  1.0, 1.0, 1.0,
    ];
}
```

### `src/app.rs`

```rust
//! Глава 42: WebGL — GPU-рендеринг в браузере.
//!
//! Демонстрирует:
//!   1. Получение WebGL2-контекста.
//!   2. Компиляцию вершинного и фрагментного шейдеров (с показом исходников).
//!   3. Загрузку данных в VBO + VAO — состояние атрибутов сохраняется.
//!   4. Два режима рендера: N draw calls vs 1 draw call.
//!   5. Uniform-переменные: u_time, u_frequency (float), u_offset (vec2),
//!      u_tint (vec3).
//!   6. FPS, draw calls/frame, frame time — видно реальную разницу.
//!
//! ПРАВИЛО: app.rs содержит только демо-код текущей главы.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use web_sys::{
    HtmlCanvasElement, HtmlElement, HtmlInputElement, HtmlSelectElement,
};

use crate::format_str::format_str;
use crate::webgl::{
    self,
    buffer,
    program::Program,
    shader,
    WebGlError,
};

// ============================================================
//                    ШЕЙДЕРЫ
// ============================================================

const VERTEX_SHADER: &str = r#"#version 300 es
in vec2 a_position;
in vec3 a_color;

uniform float u_time;       // секунды с запуска
uniform float u_frequency;  // скорость покачивания
uniform vec2  u_offset;     // сдвиг для текущего треугольника
uniform vec3  u_tint;       // цветовой множитель

out vec3 v_color;

void main() {
    float wobble = sin(u_time * u_frequency + a_position.x * 3.0) * 0.1;
    gl_Position = vec4(
        a_position.x + u_offset.x,
        a_position.y + wobble + u_offset.y,
        0.0, 1.0
    );
    v_color = a_color * u_tint;
}
"#;

const FRAGMENT_SHADER: &str = r#"#version 300 es
precision mediump float;
in vec3 v_color;
out vec4 outColor;
void main() {
    outColor = vec4(v_color, 1.0);
}
"#;

const MAX_INSTANCES: u32 = 5000;

// ============================================================
//                    HTML
// ============================================================
//
// ВНИМАНИЕ: используем r##"..."## вместо r#"..."# — в HTML есть
// value="#ffffff" у color input. Последовательность `"#` завершила
// бы r#"..."# преждевременно.

const HTML: &str = r##"
<style>
  :root {
    --g-info: #3498db; --g-ok: #27ae60; --g-err: #c0392b; --g-warn: #e67e22;
    --g-gpu: #9b59b6;
    --g-text: #222; --g-muted: #666;
    --g-bg: #fff; --g-soft: #fafafa; --g-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --g-text: #e6e6e6; --g-muted: #a0a0a0;
      --g-bg: #1e1e1e; --g-soft: #262626; --g-border: #3a3a3a;
    }
    body { background: #121212; }
    .g-out { background: #1a1a1a; color: #9ed09e; border-color: #333; }
    .g-hint { background: #232014; color: #e6c98c; }
    .g-code { background: #1a1a1a; color: #c8d8f0; border-color: #333; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--g-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; max-width: 900px; margin: 0 auto; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .g-subtitle { color: var(--g-muted); font-size: 12px; margin: 0 0 14px; }
  .g-subtitle a { color: var(--g-info); }

  .g-panel {
    border: 1px solid var(--g-border); border-radius: 8px;
    padding: 10px 12px; background: var(--g-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    margin-bottom: 12px;
  }
  .g-panel-gpu { border: 2px solid var(--g-gpu); }
  .g-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 8px;
    display: flex; align-items: center; gap: 6px;
  }
  .g-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--g-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .g-num-gpu { background: var(--g-gpu); }

  .g-row { display: flex; gap: 6px; flex-wrap: wrap; margin: 6px 0; align-items: center; }
  .g-label { font-size: 11px; color: var(--g-muted); min-width: 100px; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--g-soft); color: var(--g-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
    transition: background 0.15s, opacity 0.15s;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.g-primary { background: #eaf4ff; border-color: var(--g-info); color: #1f4e79; }
  button.g-gpu { background: #f4ecf8; border-color: var(--g-gpu); color: #6c3483; }

  .g-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
    margin-left: auto;
    transition: background 0.2s, color 0.2s;
  }
  .g-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .g-badge.err { background: #fdedec; color: #922b21; }
  .g-badge.warn { background: #fff9e6; color: #7d5a00; }

  .g-stat {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--g-muted);
  }

  .g-out {
    background: #f6f8fa; border: 1px solid var(--g-border);
    border-radius: 4px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #2d5a2d; overflow-wrap: anywhere;
    margin: 4px 0; min-height: 16px; white-space: pre-wrap;
  }
  .g-out.empty { color: var(--g-muted); font-style: italic; }

  .g-code {
    background: #f6f8fa; border: 1px solid var(--g-border);
    border-radius: 4px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #1f4e79; overflow-x: auto;
    margin: 4px 0; white-space: pre;
    line-height: 1.4;
  }

  .g-hint {
    background: #fff9e6; border-left: 3px solid var(--g-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 0 0 10px;
    color: #7d5a00;
  }
  .g-note {
    font-size: 11px; color: var(--g-muted); margin: 4px 0 0;
    line-height: 1.5;
  }
  .g-note b { color: var(--g-text); }
  .g-note code { background: rgba(0,0,0,0.06); padding: 1px 4px; border-radius: 3px; }

  .g-grid2 {
    display: grid; grid-template-columns: 1fr 1fr; gap: 8px 12px;
    margin: 6px 0;
  }
  @media (max-width: 600px) { .g-grid2 { grid-template-columns: 1fr; } }

  canvas {
    display: block; width: 100%; height: 300px;
    border-radius: 4px; background: #000;
  }
  .g-canvas-wrap {
    border: 1px solid var(--g-border); border-radius: 6px;
    overflow: hidden; margin-top: 6px;
  }

  input[type="range"] { width: 100%; }
  input[type="color"] {
    width: 40px; height: 26px; padding: 0; border: 1px solid #c0c0c0;
    border-radius: 4px; cursor: pointer;
  }
  select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--g-bg); color: var(--g-text);
    font-family: inherit; cursor: pointer;
  }
</style>

<h1>Глава 42: WebGL</h1>
<p class="g-subtitle">
  GPU-рендеринг в браузере: шейдеры, буферы, uniform-переменные.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="g-hint" id="g-hint"></div>

<!-- ============ 1. WebGL context ============ -->
<div class="g-panel">
  <h2 class="g-panel-title">
    <span class="g-num">1</span>WebGL2-контекст
    <span class="g-badge" id="g-ctx-status">ожидание</span>
  </h2>
  <div class="g-row">
    <button id="g-ctx-btn" class="g-primary">Получить контекст</button>
  </div>
  <div class="g-out empty" id="g-ctx-out">—</div>
</div>

<!-- ============ 2. Шейдеры ============ -->
<div class="g-panel">
  <h2 class="g-panel-title">
    <span class="g-num">2</span>Шейдеры — что компилирует GPU
  </h2>
  <p class="g-note">
    <b>Вершинный шейдер</b> вызывается для каждой вершины: он
    вычисляет финальную позицию на экране. <b>Фрагментный</b> —
    для каждого пикселя внутри треугольника: он вычисляет цвет.
    Оба исполняются <b>параллельно</b> на сотнях ядер GPU.
  </p>
  <div class="g-grid2">
    <div>
      <div class="g-label">vertex shader</div>
      <div class="g-code" id="g-vs-src">—</div>
    </div>
    <div>
      <div class="g-label">fragment shader</div>
      <div class="g-code" id="g-fs-src">—</div>
    </div>
  </div>
</div>

<!-- ============ 3. Рендеринг + управление ============ -->
<div class="g-panel g-panel-gpu" id="g-panel-draw">
  <h2 class="g-panel-title">
    <span class="g-num g-num-gpu">3</span>Рендеринг
    <span class="g-badge" id="g-draw-status">ожидание</span>
  </h2>

  <div class="g-row">
    <button id="g-draw-btn" class="g-gpu" disabled>▶ Запустить</button>
    <button id="g-stop-btn" class="g-primary" disabled>■ Остановить</button>
    <span class="g-stat" id="g-stats"></span>
  </div>

  <div class="g-canvas-wrap">
    <canvas id="g-canvas" width="800" height="300"></canvas>
  </div>

  <div class="g-grid2">
    <div>
      <div class="g-label">Способ рендера</div>
      <select id="g-mode">
        <option value="single">Один вызов (big VBO)</option>
        <option value="loop">CPU loop (N вызовов)</option>
      </select>
    </div>
    <div>
      <div class="g-label">Цветовой множитель (u_tint, vec3)</div>
      <input type="color" id="g-tint" value="#ffffff">
    </div>
    <div>
      <div class="g-label">
        Инстансы: <span id="g-count-label">500</span>
      </div>
      <input type="range" id="g-count" min="1" max="5000" value="500">
    </div>
    <div>
      <div class="g-label">
        Частота покачивания (u_frequency): <span id="g-freq-label">2.0</span>
      </div>
      <input type="range" id="g-freq" min="0" max="10" step="0.1" value="2">
    </div>
  </div>

  <div class="g-out empty" id="g-draw-out">—</div>

  <p class="g-note">
    <b>Single</b> — один draw call на N треугольников: GPU получает
    весь массив сразу. <b>CPU loop</b> — N draw calls: каждый раз
    JS передаёт uniform и вызывает GPU заново. Разница в FPS покажет,
    почему draw call overhead — главный враг производительности.
  </p>
</div>

<!-- ============ 4. Анатомия draw call ============ -->
<div class="g-panel">
  <h2 class="g-panel-title">
    <span class="g-num">4</span>Анатомия текущего кадра
  </h2>
  <div class="g-out" id="g-anatomy">—</div>
</div>

<!-- ============ Чек-лист ============ -->
<div class="g-panel">
  <h2 class="g-panel-title">📋 Что охвачено в этой главе</h2>
  <ul class="g-note" style="display:grid; grid-template-columns: repeat(auto-fill, minmax(220px,1fr)); gap:2px 16px; list-style:none; padding:0;">
    <li>✅ WebGL architecture</li>
    <li>✅ WebGl2RenderingContext</li>
    <li>✅ shaders (vertex + fragment)</li>
    <li>✅ buffers (VBO + VAO)</li>
    <li>✅ programs</li>
    <li>✅ uniforms (float / vec2 / vec3)</li>
    <li>✅ rendering pipeline</li>
    <li>✅ Rust + WebGL</li>
    <li>✅ Практический GPU application</li>
  </ul>
</div>
"##;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum RenderMode {
    Single,
    Loop,
}

impl RenderMode {
    fn from_str(s: &str) -> Self {
        return if s == "loop" { Self::Loop } else { Self::Single };
    }
    fn as_str(self) -> &'static str {
        return match self {
            Self::Single => "single",
            Self::Loop => "loop",
        };
    }
}

struct Renderer {
    program: Program,
    vbo: web_sys::WebGlBuffer,
    vao: web_sys::WebGlVertexArrayObject,
    buffer_bytes: usize,
    vertex_count: u32,
}

struct Ui {
    ctx_btn: HtmlElement,
    ctx_status: web_sys::Element,
    ctx_out: web_sys::Element,

    vs_src: web_sys::Element,
    fs_src: web_sys::Element,

    draw_btn: HtmlElement,
    stop_btn: HtmlElement,
    draw_status: web_sys::Element,
    canvas: HtmlCanvasElement,
    draw_out: web_sys::Element,
    stats: web_sys::Element,

    mode: HtmlSelectElement,
    tint: HtmlInputElement,
    count: HtmlInputElement,
    count_label: web_sys::Element,
    freq: HtmlInputElement,
    freq_label: web_sys::Element,

    anatomy: web_sys::Element,
}

struct State {
    gl: Option<web_sys::WebGl2RenderingContext>,
    renderer: Option<Renderer>,
    positions: Vec<(f32, f32)>,
    colors: Vec<(f32, f32, f32)>,
    mode: RenderMode,
    instance_count: u32,
    tint: (f32, f32, f32),
    frequency: f32,
    running: bool,
    start_time: f64,
    last_fps_time: f64,
    frame_count: u32,
    last_draw_calls: u32,
    last_frame_ms: f64,
}

struct App {
    ui: Ui,
    state: RefCell<State>,
}

// ============================================================
//                    ЗАПУСК
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let (positions, colors) = generate_instances(MAX_INSTANCES as usize);

    let ui = Ui {
        ctx_btn: get_btn("g-ctx-btn")?,
        ctx_status: get_el("g-ctx-status")?,
        ctx_out: get_el("g-ctx-out")?,

        vs_src: get_el("g-vs-src")?,
        fs_src: get_el("g-fs-src")?,

        draw_btn: get_btn("g-draw-btn")?,
        stop_btn: get_btn("g-stop-btn")?,
        draw_status: get_el("g-draw-status")?,
        canvas: get_el("g-canvas")?.dyn_into()?,
        draw_out: get_el("g-draw-out")?,
        stats: get_el("g-stats")?,

        mode: get_select("g-mode")?,
        tint: get_input("g-tint")?,
        count: get_input("g-count")?,
        count_label: get_el("g-count-label")?,
        freq: get_input("g-freq")?,
        freq_label: get_el("g-freq-label")?,

        anatomy: get_el("g-anatomy")?,
    };

    let app = Rc::new(App {
        ui,
        state: RefCell::new(State {
            gl: None,
            renderer: None,
            positions,
            colors,
            mode: RenderMode::Single,
            instance_count: 500,
            tint: (1.0, 1.0, 1.0),
            frequency: 2.0,
            running: false,
            start_time: 0.0,
            last_fps_time: 0.0,
            frame_count: 0,
            last_draw_calls: 0,
            last_frame_ms: 0.0,
        }),
    });

    render_hint()?;
    render_shader_sources(&app);
    render_anatomy(&app);
    bind_buttons(&app)?;
    bind_controls(&app)?;

    Ok(())
}

// ============================================================
//                    BIND
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.ctx_btn, move || {
            do_init_context(&app_c);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.draw_btn, move || {
            do_start_render(&app_c);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.stop_btn, move || {
            do_stop_render(&app_c);
        });
    }
    Ok(())
}

fn bind_controls(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let mode = RenderMode::from_str(&app_c.ui.mode.value());
            app_c.state.borrow_mut().mode = mode;
            if app_c.state.borrow().renderer.is_some() {
                let _ = reupload_buffer(&app_c);
            }
            render_anatomy(&app_c);
        });
        app.ui.mode.add_event_listener_with_callback(
            "change", cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let hex = app_c.ui.tint.value();
            if let Some(rgb) = parse_hex_color(&hex) {
                app_c.state.borrow_mut().tint = rgb;
            }
        });
        app.ui.tint.add_event_listener_with_callback(
            "input", cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let n: u32 = app_c.ui.count.value().parse().unwrap_or(500);
            app_c.state.borrow_mut().instance_count = n;
            set_text(&app_c.ui.count_label,
                &format_str(format_args!("{n}")));
            if app_c.state.borrow().renderer.is_some() {
                let _ = reupload_buffer(&app_c);
            }
        });
        app.ui.count.add_event_listener_with_callback(
            "input", cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let f: f32 = app_c.ui.freq.value().parse().unwrap_or(2.0);
            app_c.state.borrow_mut().frequency = f;
            set_text(&app_c.ui.freq_label,
                &format_str(format_args!("{f:.1}")));
        });
        app.ui.freq.add_event_listener_with_callback(
            "input", cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }
    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn do_init_context(app: &Rc<App>) {
    set_status(&app.ui.ctx_status, "warn", "получаем…");

    match webgl::context(&app.ui.canvas) {
        Ok(gl) => {
            let version = gl
                .get_parameter(web_sys::WebGl2RenderingContext::VERSION)
                .ok().and_then(|v| v.as_string())
                .unwrap_or_else(|| "unknown".to_string());
            let vendor = gl
                .get_parameter(web_sys::WebGl2RenderingContext::VENDOR)
                .ok().and_then(|v| v.as_string())
                .unwrap_or_else(|| "unknown".to_string());
            let renderer = gl
                .get_parameter(web_sys::WebGl2RenderingContext::RENDERER)
                .ok().and_then(|v| v.as_string())
                .unwrap_or_else(|| "unknown".to_string());

            let mut out = String::with_capacity(256);
            out.push_str(&format_str(format_args!("version: {version}\n")));
            out.push_str(&format_str(format_args!("vendor: {vendor}\n")));
            out.push_str(&format_str(format_args!("renderer: {renderer}")));
            set_out(&app.ui.ctx_out, &out);

            app.state.borrow_mut().gl = Some(gl);
            set_status(&app.ui.ctx_status, "ok", "готово");
            set_disabled(&app.ui.draw_btn, false);
        }
        Err(e) => {
            set_status(&app.ui.ctx_status, "err", "ошибка");
            set_out(&app.ui.ctx_out,
                &format_str(format_args!("❌ {}", e.message)));
        }
    }
}

fn do_start_render(app: &Rc<App>) {
    let gl = match app.state.borrow().gl.clone() {
        Some(g) => g,
        None => {
            set_status(&app.ui.draw_status, "err", "сначала получите контекст");
            return;
        }
    };

    set_status(&app.ui.draw_status, "warn", "инициализация…");

    let mut need_upload = false;
    {
        let mut st = app.state.borrow_mut();
        if st.renderer.is_none() {
            match build_renderer(&gl) {
                Ok(r) => {
                    let bytes = r.buffer_bytes;
                    st.renderer = Some(r);
                    st.start_time = js_sys::Date::now();
                    st.last_fps_time = st.start_time;
                    st.frame_count = 0;
                    set_out(&app.ui.draw_out, "✅ Рендерер инициализирован.");
                    set_text(&app.ui.stats, &format_str(format_args!(
                        "VBO: {bytes} B",
                    )));
                    need_upload = true;
                }
                Err(e) => {
                    set_status(&app.ui.draw_status, "err", "ошибка");
                    set_out(&app.ui.draw_out,
                        &format_str(format_args!("❌ {}", e.message)));
                    return;
                }
            }
        }
        st.running = true;
    }

    if need_upload {
        let _ = reupload_buffer(app);
    }

    set_disabled(&app.ui.draw_btn, true);
    set_disabled(&app.ui.stop_btn, false);
    set_status(&app.ui.draw_status, "ok", "рендеринг");

    render_anatomy(app);
    render_loop(app.clone());
}

fn do_stop_render(app: &Rc<App>) {
    app.state.borrow_mut().running = false;
    set_disabled(&app.ui.draw_btn, false);
    set_disabled(&app.ui.stop_btn, true);
    set_status(&app.ui.draw_status, "warn", "остановлено");
}

fn build_renderer(
    gl: &web_sys::WebGl2RenderingContext,
) -> Result<Renderer, WebGlError> {
    let vs = shader::compile_vertex(gl, VERTEX_SHADER)?;
    let fs = shader::compile_fragment(gl, FRAGMENT_SHADER)?;
    let raw = shader::link_program(gl, &vs, &fs)?;
    let mut program = Program::new(raw);

    let empty: [f32; 0] = [];
    let vbo = buffer::create_f32_buffer(
        gl, &empty,
        web_sys::WebGl2RenderingContext::DYNAMIC_DRAW,
    )?;

    let pos_attr = program.attrib_location(gl, "a_position");
    let col_attr = program.attrib_location(gl, "a_color");

    let attrs = [
        (pos_attr, 2i32, 5 * 4, 0),
        (col_attr, 3i32, 5 * 4, 2 * 4),
    ];
    let vao = buffer::create_vao_with_attrs(gl, &vbo, &attrs)?;

    return Ok(Renderer {
        program,
        vbo,
        vao,
        buffer_bytes: 0,
        vertex_count: 0,
    });
}

fn reupload_buffer(app: &Rc<App>) -> Result<(), WebGlError> {
    let gl = app.state.borrow().gl.clone()
        .ok_or_else(|| WebGlError::new("no gl".to_string()))?;

    let (mode, n, positions, colors) = {
        let st = app.state.borrow();
        (st.mode, st.instance_count, st.positions.clone(), st.colors.clone())
    };
    let n = n.min(MAX_INSTANCES);

    let data: Vec<f32> = match mode {
        RenderMode::Single => build_single_data(&positions, &colors, n),
        RenderMode::Loop => build_loop_data(),
    };
    let vertex_count = (data.len() / 5) as u32;
    let buffer_bytes = data.len() * 4;

    {
        let mut st = app.state.borrow_mut();
        if let Some(r) = st.renderer.as_mut() {
            gl.bind_buffer(
                web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                Some(&r.vbo),
            );
            let array = js_sys::Float32Array::from(&data[..]);
            gl.buffer_data_with_array_buffer_view(
                web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                &array,
                web_sys::WebGl2RenderingContext::DYNAMIC_DRAW,
            );
            r.buffer_bytes = buffer_bytes;
            r.vertex_count = vertex_count;
        }
    }
    return Ok(());
}

fn build_single_data(
    positions: &[(f32, f32)],
    colors: &[(f32, f32, f32)],
    n: u32,
) -> Vec<f32> {
    let n = (n as usize).min(positions.len());
    let mut out = Vec::with_capacity(n * 15);
    for i in 0..n {
        let (px, py) = positions[i];
        let (r, g, b) = colors[i];
        let s = 0.03;
        out.extend_from_slice(&[
            px, py + s, r, g, b,
            px - s, py - s, r, g, b,
            px + s, py - s, r, g, b,
        ]);
    }
    return out;
}

fn build_loop_data() -> Vec<f32> {
    let s = 0.03;
    return vec![
        0.0, s, 1.0, 1.0, 1.0,
        -s, -s, 1.0, 1.0, 1.0,
        s, -s, 1.0, 1.0, 1.0,
    ];
}

// ============================================================
//                    АНИМАЦИОННЫЙ ЦИКЛ
// ============================================================

fn render_loop(app: Rc<App>) {
    if !app.state.borrow().running {
        return;
    }

    let frame_start = js_sys::Date::now();
    let gl = match app.state.borrow().gl.clone() {
        Some(g) => g,
        None => return,
    };

    let (elapsed, frequency, tint, mode, n) = {
        let st = app.state.borrow();
        (
            (frame_start - st.start_time) / 1000.0,
            st.frequency,
            st.tint,
            st.mode,
            st.instance_count,
        )
    };

    gl.clear_color(0.06, 0.06, 0.10, 1.0);
    gl.clear(web_sys::WebGl2RenderingContext::COLOR_BUFFER_BIT);

    let draw_calls = draw_frame(
        &gl, &app, mode, n, elapsed as f32, frequency, tint,
    );

    if let Some(err) = webgl::check_error(&gl) {
        set_out(&app.ui.draw_out,
            &format_str(format_args!("⚠ {err}")));
    }

    let frame_ms = js_sys::Date::now() - frame_start;

    {
        let mut st = app.state.borrow_mut();
        st.frame_count += 1;
        st.last_draw_calls = draw_calls;
        st.last_frame_ms = frame_ms;
        let now = js_sys::Date::now();
        let dt = now - st.last_fps_time;
        if dt >= 1000.0 {
            let fps = st.frame_count as f64 * 1000.0 / dt;
            let bytes = st.renderer.as_ref().map(|r| r.buffer_bytes).unwrap_or(0);
            let verts = st.renderer.as_ref().map(|r| r.vertex_count).unwrap_or(0);
            let ms = st.last_frame_ms;
            let calls = st.last_draw_calls;
            let mode_str = mode.as_str();
            drop(st);
            set_text(&app.ui.stats, &format_str(format_args!(
                "FPS {fps:.0} · {ms:.1} ms/frame · {calls} draws · {verts} verts · {bytes} B · mode: {mode_str}",
            )));
            let mut st2 = app.state.borrow_mut();
            st2.frame_count = 0;
            st2.last_fps_time = now;
        }
    }

    let app_next = app.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        render_loop(app_next.clone());
    });
    if let Some(window) = web_sys::window() {
        let _ = window.request_animation_frame(cb.as_ref().unchecked_ref());
    }
    cb.forget();
}

fn draw_frame(
    gl: &web_sys::WebGl2RenderingContext,
    app: &Rc<App>,
    mode: RenderMode,
    n: u32,
    time: f32,
    frequency: f32,
    tint: (f32, f32, f32),
) -> u32 {
    let mut st = app.state.borrow_mut();
    let r = match st.renderer.as_mut() {
        Some(r) => r,
        None => return 0,
    };

    r.program.use_program(gl);
    gl.bind_vertex_array(Some(&r.vao));

    let u_time = r.program.uniform_location(gl, "u_time");
    let u_freq = r.program.uniform_location(gl, "u_frequency");
    let u_offset = r.program.uniform_location(gl, "u_offset");
    let u_tint = r.program.uniform_location(gl, "u_tint");

    if let Some(loc) = &u_time {
        gl.uniform1f(Some(loc), time);
    }
    if let Some(loc) = &u_freq {
        gl.uniform1f(Some(loc), frequency);
    }

    let draw_calls = match mode {
        RenderMode::Single => {
            if let Some(loc) = &u_offset {
                gl.uniform2f(Some(loc), 0.0, 0.0);
            }
            if let Some(loc) = &u_tint {
                gl.uniform3f(Some(loc), 1.0, 1.0, 1.0);
            }
            gl.draw_arrays(
                web_sys::WebGl2RenderingContext::TRIANGLES,
                0, r.vertex_count as i32,
            );
            1
        }
        RenderMode::Loop => {
            let count = (n as usize).min(st.positions.len());
            let positions = st.positions.clone();
            let colors = st.colors.clone();
            drop(st);
            for i in 0..count {
                let (px, py) = positions[i];
                let (cr, cg, cb) = colors[i];
                if let Some(loc) = &u_offset {
                    gl.uniform2f(Some(loc), px, py);
                }
                if let Some(loc) = &u_tint {
                    gl.uniform3f(Some(loc),
                        tint.0 * cr, tint.1 * cg, tint.2 * cb);
                }
                gl.draw_arrays(
                    web_sys::WebGl2RenderingContext::TRIANGLES,
                    0, 3,
                );
            }
            count as u32
        }
    };

    gl.bind_vertex_array(None);
    return draw_calls;
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_hint() -> Result<(), JsValue> {
    let hint = get_el("g-hint")?;
    let html = "WebGL даёт доступ к GPU: тысячи параллельных вычислений \
                для каждого кадра. В отличие от Canvas 2D, где отрисовка \
                последовательна, WebGL использует <b>программируемый \
                конвейер</b>: вершинный шейдер обрабатывает геометрию, \
                фрагментный — цвет каждого пикселя.<br>\
                <b>Главный урок:</b> один <code>drawArrays</code> на \
                N треугольников в 10–100 раз быстрее, чем N вызовов. \
                Переключите режим и посмотрите на FPS.";
    hint.set_inner_html(html);
    Ok(())
}

fn render_shader_sources(app: &Rc<App>) {
    set_text(&app.ui.vs_src, VERTEX_SHADER);
    set_text(&app.ui.fs_src, FRAGMENT_SHADER);
}

fn render_anatomy(app: &Rc<App>) {
    let st = app.state.borrow();
    let mode = st.mode;
    let n = st.instance_count;
    drop(st);

    let mut out = String::with_capacity(512);
    match mode {
        RenderMode::Single => {
            out.push_str("Режим: Single (big VBO)\n\n");
            out.push_str("useProgram()\n");
            out.push_str("bindVertexArray(vao)\n");
            out.push_str("uniform1f(u_time, ...)\n");
            out.push_str("uniform1f(u_frequency, ...)\n");
            out.push_str("uniform2f(u_offset, 0, 0)\n");
            out.push_str("uniform3f(u_tint, 1, 1, 1)\n");
            out.push_str(&format_str(format_args!(
                "drawArrays(TRIANGLES, 0, {})\n",
                n * 3,
            )));
            out.push_str("  → 1 draw call\n\n");
            out.push_str("Все N треугольников уходят в GPU одним пакетом.");
        }
        RenderMode::Loop => {
            out.push_str("Режим: CPU loop (N draw calls)\n\n");
            out.push_str("useProgram()\n");
            out.push_str("bindVertexArray(vao)\n");
            out.push_str("uniform1f(u_time, ...)\n");
            out.push_str("uniform1f(u_frequency, ...)\n");
            out.push_str(&format_str(format_args!(
                "for i in 0..{n} {{\n",
            )));
            out.push_str("  uniform2f(u_offset, pos[i].x, pos[i].y)\n");
            out.push_str("  uniform3f(u_tint, ...)\n");
            out.push_str("  drawArrays(TRIANGLES, 0, 3)\n");
            out.push_str("}\n");
            out.push_str(&format_str(format_args!(
                "  → {n} draw calls\n\n",
            )));
            out.push_str("Каждый draw call — отдельный переход JS→GPU. \
                Overhead доминирует над полезной работой.");
        }
    }
    set_out(&app.ui.anatomy, &out);
}

fn set_text(el: &web_sys::Element, text: &str) {
    el.set_text_content(Some(text));
}

fn set_out(el: &web_sys::Element, text: &str) {
    el.set_text_content(Some(text));
    el.set_class_name("g-out");
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "g-badge ok",
        "err" => "g-badge err",
        "warn" => "g-badge warn",
        _ => "g-badge",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn set_disabled(el: &HtmlElement, disabled: bool) {
    if disabled {
        let _ = el.set_attribute("disabled", "");
    } else {
        let _ = el.remove_attribute("disabled");
    }
}

// ============================================================
//                    ГЕНЕРАЦИЯ ДАННЫХ
// ============================================================

fn generate_instances(max: usize) -> (Vec<(f32, f32)>, Vec<(f32, f32, f32)>) {
    let mut state: u32 = 0xDEAD_BEEF;
    let mut next = || -> f32 {
        state = state.wrapping_mul(1_664_525).wrapping_add(1_013_904_223);
        return ((state >> 8) as f32) / 16_777_216.0;
    };
    let mut positions = Vec::with_capacity(max);
    let mut colors = Vec::with_capacity(max);
    for _ in 0..max {
        positions.push((
            next() * 1.8 - 0.9,
            next() * 1.8 - 0.9,
        ));
        colors.push((
            next() * 0.75 + 0.25,
            next() * 0.75 + 0.25,
            next() * 0.75 + 0.25,
        ));
    }
    return (positions, colors);
}

fn parse_hex_color(s: &str) -> Option<(f32, f32, f32)> {
    let t = s.trim_start_matches('#');
    if t.len() != 6 {
        return None;
    }
    let r = u8::from_str_radix(&t[0..2], 16).ok()?;
    let g = u8::from_str_radix(&t[2..4], 16).ok()?;
    let b = u8::from_str_radix(&t[4..6], 16).ok()?;
    return Some((
        r as f32 / 255.0,
        g as f32 / 255.0,
        b as f32 / 255.0,
    ));
}

// ============================================================
//                    ХЕЛПЕРЫ DOM
// ============================================================

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    return web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(
            &format_str(format_args!("no #{id}")),
        ));
}

fn get_btn(id: &str) -> Result<HtmlElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlElement"));
}

fn get_input(id: &str) -> Result<HtmlInputElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlInputElement"));
}

fn get_select(id: &str) -> Result<HtmlSelectElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlSelectElement"));
}

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut() + 'static,
{
    let cb = Closure::<dyn FnMut()>::new(f);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```

### `tests/wasm_webgl.rs`

```rust
//! WASM-тесты главы 42: WebGL.
//!
//! Запуск:
//!   wasm-pack test --chrome --headless -- --test wasm_webgl
//!
//! WebGL требует реального GPU-контекста. В headless Chrome
//! используется программная растеризация (SwiftShader), но
//! API работает полностью.

use wasm_bindgen_test::*;

use hello_wasm02::webgl::{
    self,
    buffer, geometry, shader,
};

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// 1. Геометрия — чистая математика, без GPU
// ============================================================

#[wasm_bindgen_test]
fn wasm_triangle_vertex_count() {
    let data = geometry::triangle();
    assert_eq!(data.len(), 15);
}

#[wasm_bindgen_test]
fn wasm_quad_vertex_count() {
    let data = geometry::quad();
    assert_eq!(data.len(), 24);
}

#[wasm_bindgen_test]
fn wasm_triangle_bounds() {
    let data = geometry::triangle();
    for (i, v) in data.iter().enumerate() {
        if i % 5 < 2 {
            assert!(*v >= -1.0 && *v <= 1.0, "vertex out of bounds");
        } else {
            assert!(*v >= 0.0 && *v <= 1.0, "color out of range");
        }
    }
}

#[wasm_bindgen_test]
fn wasm_quad_uv_range() {
    let data = geometry::quad();
    for chunk in data.chunks(4) {
        assert!(chunk[2] >= 0.0 && chunk[2] <= 1.0, "u out of range");
        assert!(chunk[3] >= 0.0 && chunk[3] <= 1.0, "v out of range");
    }
}

// ============================================================
// 2. WebGL-контекст
// ============================================================

use wasm_bindgen::JsCast;

fn make_canvas() -> web_sys::HtmlCanvasElement {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas: web_sys::HtmlCanvasElement = document
        .create_element("canvas")
        .unwrap()
        .dyn_into()
        .unwrap();
    canvas.set_width(64);
    canvas.set_height(64);
    return canvas;
}

#[wasm_bindgen_test]
fn wasm_canvas_context_available() {
    let canvas = make_canvas();
    let ctx = canvas.get_context("webgl2").unwrap();
    assert!(ctx.is_some(), "webgl2 context должен быть доступен");
}

#[wasm_bindgen_test]
fn wasm_context_returns_valid_gl() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).expect("context() должен вернуть Ok");
    let version = gl
        .get_parameter(web_sys::WebGl2RenderingContext::VERSION)
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    assert!(!version.is_empty());
    assert!(version.contains("WebGL"), "version: {version}");
}

// ============================================================
// 3. Компиляция шейдеров
// ============================================================

const GOOD_VS: &str = r#"#version 300 es
in vec2 a_position;
void main() {
    gl_Position = vec4(a_position, 0.0, 1.0);
}
"#;

const GOOD_FS: &str = r#"#version 300 es
precision mediump float;
out vec4 outColor;
void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
"#;

const BAD_VS: &str = r#"#version 300 es
this is not valid GLSL
"#;

#[wasm_bindgen_test]
fn wasm_compile_valid_vertex_shader() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let result = shader::compile_vertex(&gl, GOOD_VS);
    assert!(result.is_ok(), "валидный шейдер должен компилироваться");
}

#[wasm_bindgen_test]
fn wasm_compile_valid_fragment_shader() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let result = shader::compile_fragment(&gl, GOOD_FS);
    assert!(result.is_ok());
}

#[wasm_bindgen_test]
fn wasm_compile_invalid_shader_fails() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let result = shader::compile_vertex(&gl, BAD_VS);
    assert!(result.is_err(), "невалидный шейдер должен дать ошибку");
}

#[wasm_bindgen_test]
fn wasm_link_program_ok() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let vs = shader::compile_vertex(&gl, GOOD_VS).unwrap();
    let fs = shader::compile_fragment(&gl, GOOD_FS).unwrap();
    let program = shader::link_program(&gl, &vs, &fs);
    assert!(program.is_ok());
}

// ============================================================
// 4. Создание буфера
// ============================================================

#[wasm_bindgen_test]
fn wasm_create_f32_buffer() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let data = geometry::triangle();
    let result = buffer::create_f32_buffer(
        &gl, &data,
        web_sys::WebGl2RenderingContext::STATIC_DRAW,
    );
    assert!(result.is_ok());
}

#[wasm_bindgen_test]
fn wasm_create_vao_with_attrs() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let data = geometry::triangle();
    let vbo = buffer::create_f32_buffer(
        &gl, &data,
        web_sys::WebGl2RenderingContext::STATIC_DRAW,
    ).unwrap();
    let attrs = [
        (0u32, 2i32, 5 * 4, 0),
        (1u32, 3i32, 5 * 4, 2 * 4),
    ];
    let result = buffer::create_vao_with_attrs(&gl, &vbo, &attrs);
    assert!(result.is_ok(), "VAO должен создаться");
}

// ============================================================
// 5. Полный pipeline — компиляция + линковка + VBO + VAO
// ============================================================

#[wasm_bindgen_test]
fn wasm_full_pipeline_init() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();

    let vs = shader::compile_vertex(&gl, GOOD_VS).unwrap();
    let fs = shader::compile_fragment(&gl, GOOD_FS).unwrap();
    let program = shader::link_program(&gl, &vs, &fs).unwrap();

    let data = geometry::triangle();
    let vbo = buffer::create_f32_buffer(
        &gl, &data,
        web_sys::WebGl2RenderingContext::STATIC_DRAW,
    ).unwrap();

    let pos_attr = gl.get_attrib_location(&program, "a_position");
    assert!(pos_attr >= 0, "a_position должен быть найден");

    let attrs = [
        (pos_attr as u32, 2i32, 5 * 4, 0),
        (pos_attr as u32 + 1, 3i32, 5 * 4, 2 * 4),
    ];
    let vao = buffer::create_vao_with_attrs(&gl, &vbo, &attrs);
    assert!(vao.is_ok(), "VAO должен создаться в полном pipeline");

    let err = webgl::check_error(&gl);
    assert!(err.is_none(), "не должно быть GL-ошибок: {err:?}");
}

// ============================================================
// 6. Проверка ошибок GL
// ============================================================

#[wasm_bindgen_test]
fn wasm_check_error_clean() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    assert!(webgl::check_error(&gl).is_none());
}

#[wasm_bindgen_test]
fn wasm_check_error_after_invalid_call() {
    let canvas = make_canvas();
    let gl = webgl::context(&canvas).unwrap();
    let _ = gl.get_parameter(0xDEADBEEF);
    let err = webgl::check_error(&gl);
    assert!(err.is_some(), "должна быть зафиксирована ошибка");
}
```

---

## 9. Команды запуска

```bash
# Сборка
npm run wasm:dev

# Тесты главы 42
wasm-pack test --chrome --headless -- --test wasm_webgl

# Firefox
wasm-pack test --firefox --headless -- --test wasm_webgl

# Демо
npm run dev
```

**Ожидаемый результат тестов:** **15 passed, 0 failed**.

---

## 10. Итог

WebGL — это **прямой доступ к GPU из браузера**. Ключевые уроки главы:

1. **Программируемый конвейер вместо фиксированного.** Вершинный шейдер обрабатывает геометрию, фрагментный — пиксели. Оба пишутся на GLSL и компилируются драйвером.
2. **Draw call overhead — главный враг.** Один `drawArrays` на 5000 треугольников в 10–100 раз быстрее, чем 5000 вызовов. Это видно в FPS-счётчике демо.
3. **VAO сохраняет состояние атрибутов один раз.** Без него `vertexAttribPointer` вызывается каждый кадр — медленно и неудобно.
4. **GPU не бросает исключения.** `gl.getError()` — обязательная проверка после каждого draw.
5. **Rust + web-sys даёт типобезопасность.** `WebGlBuffer` не спутать с `WebGlTexture`, ошибки ловятся компилятором.

Демо показывает **три принципиальных вещи**: как получить GPU-контекст, как выстроить pipeline от шейдеров до draw call, и как анимировать сцену через uniform. Всё остальное — instancing, текстуры, framebuffer — это расширения того же конвейера.