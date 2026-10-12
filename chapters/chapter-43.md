# Часть XII. GPU и WebGL

## Глава 43. WebGL 2

---

## Введение

Глава 42 показала базовый конвейер: два шейдера, один VBO, один `drawArrays`. Но реальные GPU-приложения редко рисуют один треугольник. Они рисуют **тысячи объектов** — частицы, травы, деревья, звёзды, точки данных на графике. И здесь начинается настоящая разница между WebGL 1 и WebGL 2.

**Ключевое отличие WebGL 2 — instancing.** В WebGL 1, чтобы нарисовать 10 000 объектов, приходилось либо 10 000 раз вызывать `drawArrays` (медленно — переход JS↔GPU), либо «запекать» все 10 000 объектов в один VBO (дорого по памяти, неудобно обновлять). WebGL 2 даёт третий путь: **один draw call на N инстансов, каждый со своими атрибутами через `vertexAttribDivisor`**.

Три ключевых нововведения, которых не было в WebGL 1:

1. **Instancing** — `drawArraysInstanced` + `vertexAttribDivisor`. Рисуем N копий одной геометрии, у каждой свои параметры.
2. **Transform Feedback** — вершинный шейдер пишет результат не только в `gl_Position`, но и в произвольный буфер. Это GPGPU до compute shaders.
3. **Capabilities & extensions** — `MAX_3D_TEXTURE_SIZE`, `EXT_color_buffer_float`, `EXT_disjoint_timer_query_webgl2`. Программа адаптируется к железу.

В этой главе мы покроем 6 тем:

1. `WebGl2RenderingContext` — что нового по сравнению с WebGL 1
2. advanced buffers — VAO + divisors + `bufferSubData`
3. **instancing** — главная фича главы
4. transform feedback — GPU-вычисления
5. floating-point textures — GPGPU-хранилище (через probe расширений)
6. Rust/WASM rendering — синхронизация CPU↔GPU

---

## 1. Что мы узнали при сборке: 5 грабель WebGL 2

### Грабли 1. `get_attrib_location` возвращает `i32`, но всё остальное принимает `u32`

```rust
// get_attrib_location возвращает i32.
let loc_i32 = gl.get_attrib_location(&program, name);
// enable_vertex_attrib_array и vertex_attrib_pointer_with_i32 ждут u32.
```

Кастуем **один раз при кэшировании** в `Program::attrib_location` — не в каждой итерации draw.

### Грабли 2. `get_buffer_sub_data_*` возвращает `()`, а не `Result`

В отличие от большинства методов web-sys, `get_buffer_sub_data_with_i32_and_array_buffer_view` **не оборачивает** вызов в `Result`. Спецификация WebGL считает эту операцию «тихой»: ошибки читаются через `gl.getError()`. Значит, `.map_err(...)?` после неё — ошибка компиляции.

**Решение:** вызвать, потом явно проверить `webgl2::check_error(gl)`.

### Грабли 3. Raw-строка `r#"..."#` ломается на `"#` в HTML

Классический случай — атрибут `value="#ffffff"` у `<input type="color">`. Последовательность `"#` завершает raw-строку `r#"..."#` **прямо в середине HTML**. Парсер Rust видит CSS как код.

**Решение:** `r##"..."##` (терминатор `"##`, а `"#` внутри безопасен). Если внутри есть `"##` — использовать `r###"..."###`.

### Грабли 4. `RefCell::borrow_mut()` живёт до конца блока — не смешивайте с DOM

```rust
let mut st = app.state.borrow_mut();
let draw = st.renderer.as_mut().unwrap().draw(...);  // &mut st
st.frame_count += 1;                                  // &mut st
drop(st);                                             // ← move
st.frame_count += 1;                                  // ❌ use after move
```

**Решение:** не писать `drop(st)` вообще. Собрать всё нужное в локальную переменную, закрыть блок `{ ... }`, а DOM обновлять **после**:

```rust
let fps_line: Option<String> = {
    let mut st = app.state.borrow_mut();
    st.frame_count += 1;
    // ... собрали строку ...
    Some(format_str(format_args!(...)))
}; // ← borrow освобождён здесь автоматически

if let Some(line) = fps_line {
    set_text(&app.ui.stats, &line);
}
```

### Грабли 5. Поле `vbo_base` как «GC-якорь»

VAO хранит внутри состояние атрибутов, но wasm-bindgen не считает ссылки внутри GPU-объектов. Если `WebGlBuffer` не держится в Rust-структуре, JS-GC может удалить его, и VAO начнёт ссылаться на освобождённый буфер — драйвер молча вернёт `INVALID_OPERATION`.

**Решение:** держать буферы живыми как поля структуры. Даже если поле не читается — это намеренно. Помечаем `#[allow(dead_code)]` с комментарием.

---

## 2. Архитектура модуля

```
src/
├── lib.rs                       ← pub mod webgl2
├── app.rs                       ← демо-пример главы
└── webgl2/
    ├── mod.rs                   ← WebGl2Error, context(), Capabilities, Program, check_error()
    ├── shader.rs                ← compile + link + transform feedback varyings
    ├── instancing.rs            ← InstancedRenderer с divisors
    └── geometry.rs              ← генерация base + instance данных
tests/
└── wasm_webgl2.rs               ← интеграционные тесты
```

---

## 3. Instancing: главная фича

### Сравнение памяти

| Стратегия | Формула | N = 5000 |
|---|---|---|
| Big VBO (глава 42) | N × 3 verts × 5 float × 4 B | 300 KB |
| Instancing (глава 43) | (3 verts × 2 float + N × 6 float) × 4 B | ~120 KB |
| **Экономия** | | **60%** |

### Как работает `vertexAttribDivisor`

Без divisor'а каждый атрибут «шагает» по VBO для каждой вершины. С `vertexAttribDivisor(loc, 1)` GPU читает новое значение атрибута **раз на инстанс**, а не на вершину.

```
VBO (per-instance, 6 float × N):
[ox0, oy0, r0, g0, b0, s0]   ← инстанс 0, все 3 вершины читают эту строку
[ox1, oy1, r1, g1, b1, s1]   ← инстанс 1
[ox2, oy2, r2, g2, b2, s2]   ← инстанс 2
...
```

`drawArraysInstanced(TRIANGLES, 0, 3, N)` = «выполни всё, что делает `drawArrays(TRIANGLES, 0, 3)`, N раз, каждый раз беря следующую строку per-instance буфера».

---

## 4. Transform Feedback

Идея: **вершинный шейдер пишет результат в буфер**, а не в `gl_Position`. Растеризатор отключается (`RASTERIZER_DISCARD`), и вместо рисования получаем «GPU-вычислитель».

Шаги:

1. Собираем вершинный шейдер с `out float v_result;`.
2. Перед `link_program` вызываем `transformFeedbackVaryings([v_result], SEPARATE_ATTRIBS)`.
3. Создаём VBO для output, привязываем к `TRANSFORM_FEEDBACK_BUFFER`.
4. Создаём `WebGlTransformFeedback`, привязываем его.
5. `enable(RASTERIZER_DISCARD)` — выключаем рисование.
6. `beginTransformFeedback(POINTS); drawArrays(POINTS, 0, N); endTransformFeedback()`.
7. `getBufferSubData` читаем результат обратно.

В демо: `output[i] = input[i] * 2 + 1`. CPU присылает `[1, 2, 3, 4, 5, 6, 7, 8]`, GPU возвращает `[3, 5, 7, 9, 11, 13, 15, 17]`. Никакой растеризации не происходит — только вычисление.

Это основа GPGPU: физические симуляции, обработка частиц, сортировка, поиск минимума — всё, что не рисуется, а считается.

---

## 5. Capabilities & extensions

Программа должна адаптироваться к GPU. `gl.get_parameter(...)` даёт лимиты, `gl.get_extension(...)` — доступные расширения.

Ключевые для этой главы:

| Расширение | Что даёт |
|---|---|
| `EXT_color_buffer_float` | Рендер в `RGBA32F` текстуры (GPGPU-хранилище) |
| `EXT_disjoint_timer_query_webgl2` | Точные GPU-тайминги |
| `OES_texture_float_linear` | Линейная фильтрация FP-текстур |

Демо показывает реальные значения вашего GPU. На типичном Chrome + Linux: `MAX_VERTEX_ATTRIBS=16`, `MAX_3D_TEXTURE_SIZE=2048`, `color_buffer_float=✅`, `timer_query=❌` (зависит от GPU-профиля).

---

## 6. Rust + WebGL2

`web-sys` даёт типизированные обёртки:

- `WebGl2RenderingContext` — контекст.
- `WebGlTransformFeedback` — TF-объект (только WebGL2).
- `WebGlVertexArrayObject` — VAO (только WebGL2).
- `WebGlQuery` — для GPU-таймингов.

Типобезопасность — реальное преимущество: `WebGlProgram` не передать в `attach_shader`, `WebGlBuffer` — в `uniform1f`.

---

## 7. Исходные коды

### `Cargo.toml` (features для web-sys)

```toml
    # WebGL (глава 42–43)
    "WebGl2RenderingContext",
    "WebGlProgram",
    "WebGlShader",
    "WebGlBuffer",
    "WebGlUniformLocation",
    "WebGlTexture",
    "WebGlVertexArrayObject",
    "WebGlTransformFeedback",
    "WebGlQuery",
```

### `src/lib.rs`

```rust
//! Hello WASM02 — курс по Rust + WebAssembly.
//!
//! Глава 43: WebGL 2.

pub mod format_str;
pub mod app;
pub mod web_crypto;   // глава 38
pub mod webauthn;     // глава 41
pub mod webgl;        // глава 42
pub mod webgl2;       // глава 43
```

### `src/webgl2/mod.rs`

```rust
//! Глава 43: WebGL 2 — продвинутые возможности GPU.
//!
//! WebGL 2 основан на OpenGL ES 3.0 и добавляет к базовому
//! конвейеру (глава 42):
//!   - instancing (`drawArraysInstanced` + `vertexAttribDivisor`);
//!   - transform feedback (запись вывода вершинного шейдера в буфер);
//!   - floating-point textures (RGBA32F, R32F);
//!   - 3D-текстуры, MRT, integer attributes.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

pub mod shader;
pub mod instancing;
pub mod geometry;

use std::collections::HashMap;
use wasm_bindgen::JsValue;
use wasm_bindgen::JsCast;
use crate::format_str::format_str;

// ============================================================
//                    ОШИБКИ
// ============================================================

#[derive(Debug, Clone)]
pub struct WebGl2Error {
    pub message: String,
}

impl WebGl2Error {
    pub fn new(msg: String) -> Self {
        return Self { message: msg };
    }
}

impl std::fmt::Display for WebGl2Error {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return write!(f, "{}", self.message);
    }
}

impl From<JsValue> for WebGl2Error {
    fn from(v: JsValue) -> Self {
        let msg = v
            .as_string()
            .unwrap_or_else(|| format_str(format_args!("{v:?}")));
        return Self { message: msg };
    }
}

impl From<String> for WebGl2Error {
    fn from(s: String) -> Self {
        return Self { message: s };
    }
}

// ============================================================
//                    КОНТЕКСТ
// ============================================================

pub fn context(
    canvas: &web_sys::HtmlCanvasElement,
) -> Result<web_sys::WebGl2RenderingContext, WebGl2Error> {
    let ctx = canvas
        .get_context("webgl2")
        .map_err(WebGl2Error::from)?
        .ok_or_else(|| WebGl2Error::new("webgl2 context unavailable".to_string()))?;
    let gl: web_sys::WebGl2RenderingContext = ctx
        .dyn_into()
        .map_err(|_| WebGl2Error::new("not a WebGl2RenderingContext".to_string()))?;
    return Ok(gl);
}

// ============================================================
//                    CAPABILITIES
// ============================================================

pub struct Capabilities {
    pub max_vertex_attribs: i32,
    pub max_vertex_uniform_vectors: i32,
    pub max_texture_size: i32,
    pub max_3d_texture_size: i32,
    pub max_samples: i32,
    pub max_elements_vertices: i32,
    pub color_buffer_float: bool,
    pub timer_query: bool,
    pub float_linear: bool,
}

pub fn query_capabilities(gl: &web_sys::WebGl2RenderingContext) -> Capabilities {
    let get_i32 = |pname: u32| -> i32 {
        gl.get_parameter(pname)
            .ok()
            .and_then(|v| v.as_f64())
            .map(|f| f as i32)
            .unwrap_or(0)
    };
    let has_ext = |name: &str| -> bool {
        gl.get_extension(name).ok().flatten().is_some()
    };
    return Capabilities {
        max_vertex_attribs: get_i32(web_sys::WebGl2RenderingContext::MAX_VERTEX_ATTRIBS),
        max_vertex_uniform_vectors: get_i32(web_sys::WebGl2RenderingContext::MAX_VERTEX_UNIFORM_VECTORS),
        max_texture_size: get_i32(web_sys::WebGl2RenderingContext::MAX_TEXTURE_SIZE),
        max_3d_texture_size: get_i32(web_sys::WebGl2RenderingContext::MAX_3D_TEXTURE_SIZE),
        max_samples: get_i32(web_sys::WebGl2RenderingContext::MAX_SAMPLES),
        max_elements_vertices: get_i32(web_sys::WebGl2RenderingContext::MAX_ELEMENTS_VERTICES),
        color_buffer_float: has_ext("EXT_color_buffer_float"),
        timer_query: has_ext("EXT_disjoint_timer_query_webgl2"),
        float_linear: has_ext("OES_texture_float_linear"),
    };
}

// ============================================================
//                    PROGRAM WRAPPER
// ============================================================

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

    /// `get_attrib_location` возвращает `i32`, но все последующие
    /// вызовы принимают `u32`. Кастуем один раз при кэшировании.
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

// ============================================================
//                    CHECK ERROR
// ============================================================

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

### `src/webgl2/shader.rs`

```rust
//! Компиляция и линковка шейдеров WebGL2.

use crate::format_str::format_str;
use super::WebGl2Error;

pub fn compile_vertex(
    gl: &web_sys::WebGl2RenderingContext,
    source: &str,
) -> Result<web_sys::WebGlShader, WebGl2Error> {
    return compile(gl, source, web_sys::WebGl2RenderingContext::VERTEX_SHADER);
}

pub fn compile_fragment(
    gl: &web_sys::WebGl2RenderingContext,
    source: &str,
) -> Result<web_sys::WebGlShader, WebGl2Error> {
    return compile(gl, source, web_sys::WebGl2RenderingContext::FRAGMENT_SHADER);
}

fn compile(
    gl: &web_sys::WebGl2RenderingContext,
    source: &str,
    kind: u32,
) -> Result<web_sys::WebGlShader, WebGl2Error> {
    let shader = gl
        .create_shader(kind)
        .ok_or_else(|| WebGl2Error::new("create_shader failed".to_string()))?;
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
        return Err(WebGl2Error::new(format_str(format_args!(
            "shader compile error: {log}",
        ))));
    }
    return Ok(shader);
}

pub fn link(
    gl: &web_sys::WebGl2RenderingContext,
    vs: &web_sys::WebGlShader,
    fs: &web_sys::WebGlShader,
) -> Result<web_sys::WebGlProgram, WebGl2Error> {
    return link_with_varyings(gl, vs, fs, None);
}

/// Линкует программу с опциональным transform feedback.
///
/// `varyings` — список имён выходов вершинного шейдера, которые
/// будут записаны в `TRANSFORM_FEEDBACK_BUFFER`. Передаётся
/// **до** `link_program`.
pub fn link_with_varyings(
    gl: &web_sys::WebGl2RenderingContext,
    vs: &web_sys::WebGlShader,
    fs: &web_sys::WebGlShader,
    varyings: Option<&[&str]>,
) -> Result<web_sys::WebGlProgram, WebGl2Error> {
    let program = gl
        .create_program()
        .ok_or_else(|| WebGl2Error::new("create_program failed".to_string()))?;
    gl.attach_shader(&program, vs);
    gl.attach_shader(&program, fs);

    // transformFeedbackVaryings должен вызываться до link_program.
    if let Some(names) = varyings {
        let array = js_sys::Array::new();
        for n in names {
            array.push(&wasm_bindgen::JsValue::from_str(n));
        }
        gl.transform_feedback_varyings(
            &program,
            &array,
            web_sys::WebGl2RenderingContext::SEPARATE_ATTRIBS,
        );
    }

    gl.link_program(&program);

    let status = gl
        .get_program_parameter(&program, web_sys::WebGl2RenderingContext::LINK_STATUS)
        .as_bool()
        .unwrap_or(false);
    if !status {
        let log = gl
            .get_program_info_log(&program)
            .unwrap_or_else(|| "unknown error".to_string());
        return Err(WebGl2Error::new(format_str(format_args!(
            "program link error: {log}",
        ))));
    }
    return Ok(program);
}
```

### `src/webgl2/instancing.rs`

```rust
//! Инстансинг — главная фича WebGL 2.

use super::{check_error, shader, Program, WebGl2Error};

pub struct InstancedRenderer {
    pub program: Program,
    pub vao: web_sys::WebGlVertexArrayObject,
    /// Держим живым: даже при привязке к VAO GC может удалить
    /// буфер, если на него нет других ссылок. Это известная
    /// особенность wasm-bindgen: он не считает ссылки внутри
    /// GPU-объектов.
    #[allow(dead_code)]
    pub base_vbo: web_sys::WebGlBuffer,
    pub instance_vbo: web_sys::WebGlBuffer,
    pub loaded_instances: usize,
    pub vertex_count: i32,
}

impl InstancedRenderer {
    pub fn new(
        gl: &web_sys::WebGl2RenderingContext,
        vs_source: &str,
        fs_source: &str,
        base_data: &[f32],
    ) -> Result<Self, WebGl2Error> {
        let vs = shader::compile_vertex(gl, vs_source)?;
        let fs = shader::compile_fragment(gl, fs_source)?;
        let raw = shader::link(gl, &vs, &fs)?;
        let mut program = Program::new(raw);

        let base_vbo = gl.create_buffer()
            .ok_or_else(|| WebGl2Error::new("create_buffer failed".to_string()))?;
        gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&base_vbo));
        gl.buffer_data_with_array_buffer_view(
            web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
            &js_sys::Float32Array::from(base_data),
            web_sys::WebGl2RenderingContext::STATIC_DRAW,
        );

        let instance_vbo = gl.create_buffer()
            .ok_or_else(|| WebGl2Error::new("create_buffer failed".to_string()))?;
        gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&instance_vbo));
        gl.buffer_data_with_array_buffer_view(
            web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
            &js_sys::Float32Array::new_with_length(0),
            web_sys::WebGl2RenderingContext::DYNAMIC_DRAW,
        );

        let vao = gl.create_vertex_array()
            .ok_or_else(|| WebGl2Error::new("create_vertex_array failed".to_string()))?;
        gl.bind_vertex_array(Some(&vao));

        // a_position — per-vertex.
        let pos_loc = program.attrib_location(gl, "a_position");
        gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&base_vbo));
        gl.enable_vertex_attrib_array(pos_loc);
        gl.vertex_attrib_pointer_with_i32(
            pos_loc, 2,
            web_sys::WebGl2RenderingContext::FLOAT,
            false, 2 * 4, 0,
        );

        let stride = 6 * 4;

        let inst_loc = program.attrib_location(gl, "a_instance");
        gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&instance_vbo));
        gl.enable_vertex_attrib_array(inst_loc);
        gl.vertex_attrib_pointer_with_i32(
            inst_loc, 2,
            web_sys::WebGl2RenderingContext::FLOAT,
            false, stride, 0,
        );
        gl.vertex_attrib_divisor(inst_loc, 1);

        let color_loc = program.attrib_location(gl, "a_color");
        gl.enable_vertex_attrib_array(color_loc);
        gl.vertex_attrib_pointer_with_i32(
            color_loc, 3,
            web_sys::WebGl2RenderingContext::FLOAT,
            false, stride, 2 * 4,
        );
        gl.vertex_attrib_divisor(color_loc, 1);

        let scale_loc = program.attrib_location(gl, "a_scale");
        gl.enable_vertex_attrib_array(scale_loc);
        gl.vertex_attrib_pointer_with_i32(
            scale_loc, 1,
            web_sys::WebGl2RenderingContext::FLOAT,
            false, stride, 5 * 4,
        );
        gl.vertex_attrib_divisor(scale_loc, 1);

        gl.bind_vertex_array(None);

        let vertex_count = (base_data.len() / 2) as i32;
        return Ok(Self {
            program,
            vao,
            base_vbo,
            instance_vbo,
            loaded_instances: 0,
            vertex_count,
        });
    }

    /// Загружает per-instance данные.
    ///
    /// Если буфер уже вмещает больше — используем `bufferSubData`
    /// для инкрементального обновления.
    pub fn upload_instances(
        &mut self,
        gl: &web_sys::WebGl2RenderingContext,
        data: &[f32],
    ) -> Result<usize, WebGl2Error> {
        let count = data.len() / 6;
        gl.bind_buffer(
            web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
            Some(&self.instance_vbo),
        );
        let array = js_sys::Float32Array::from(data);

        if count <= self.loaded_instances && self.loaded_instances > 0 {
            gl.buffer_sub_data_with_i32_and_array_buffer_view(
                web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                0,
                &array,
            );
        } else {
            gl.buffer_data_with_array_buffer_view(
                web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                &array,
                web_sys::WebGl2RenderingContext::DYNAMIC_DRAW,
            );
        }
        self.loaded_instances = count;
        return Ok(count);
    }

    /// Один draw call на N инстансов.
    ///
    /// `&mut self` — чтобы использовать кэш uniform-локаций в `Program`.
    pub fn draw(
        &mut self,
        gl: &web_sys::WebGl2RenderingContext,
        time: f32,
    ) -> u32 {
        if self.loaded_instances == 0 {
            return 0;
        }
        self.program.use_program(gl);
        gl.bind_vertex_array(Some(&self.vao));

        if let Some(loc) = self.program.uniform_location(gl, "u_time") {
            gl.uniform1f(Some(&loc), time);
        }

        gl.draw_arrays_instanced(
            web_sys::WebGl2RenderingContext::TRIANGLES,
            0,
            self.vertex_count,
            self.loaded_instances as i32,
        );

        gl.bind_vertex_array(None);
        let _ = check_error(gl);
        return 1;
    }
}
```

### `src/webgl2/geometry.rs`

```rust
//! Генерация данных для инстансинга.

pub fn base_triangle() -> [f32; 6] {
    return [
        -1.0, -1.0,
         1.0, -1.0,
         0.0,  1.0,
    ];
}

/// Генерирует `n` инстансов с детерминированными параметрами.
///
/// Формат каждого инстанса (6 float):
///   offset.x, offset.y, r, g, b, scale.
pub fn generate_instances(n: usize) -> Vec<f32> {
    let mut state: u32 = 0xC0FFEE;
    let mut next = || -> f32 {
        state = state.wrapping_mul(1_664_525).wrapping_add(1_013_904_223);
        return ((state >> 8) as f32) / 16_777_216.0;
    };
    let mut out = Vec::with_capacity(n * 6);
    for _ in 0..n {
        out.push(next() * 1.8 - 0.9);
        out.push(next() * 1.8 - 0.9);
        out.push(next() * 0.75 + 0.25);
        out.push(next() * 0.75 + 0.25);
        out.push(next() * 0.75 + 0.25);
        out.push(next() * 0.016 + 0.008);
    }
    return out;
}
```

### `tests/wasm_webgl2.rs`

```rust
//! WASM-тесты главы 43: WebGL 2.
//!
//! Запуск:
//!   wasm-pack test --chrome --headless -- --test wasm_webgl2

use wasm_bindgen_test::*;
use wasm_bindgen::JsCast;

use hello_wasm02::webgl2::{
    self, geometry, instancing::InstancedRenderer, shader,
};

wasm_bindgen_test_configure!(run_in_browser);

fn make_canvas() -> web_sys::HtmlCanvasElement {
    let doc = web_sys::window().unwrap().document().unwrap();
    let canvas: web_sys::HtmlCanvasElement = doc
        .create_element("canvas").unwrap().dyn_into().unwrap();
    canvas.set_width(64);
    canvas.set_height(64);
    return canvas;
}

#[wasm_bindgen_test]
fn wasm_base_triangle_size() {
    assert_eq!(geometry::base_triangle().len(), 6);
}

#[wasm_bindgen_test]
fn wasm_generate_instances_size() {
    assert_eq!(geometry::generate_instances(100).len(), 600);
}

#[wasm_bindgen_test]
fn wasm_webgl2_context_created() {
    let canvas = make_canvas();
    let gl = webgl2::context(&canvas).expect("webgl2 должен быть доступен");
    let version = gl
        .get_parameter(web_sys::WebGl2RenderingContext::VERSION)
        .ok().and_then(|v| v.as_string()).unwrap_or_default();
    assert!(version.contains("WebGL 2.0"), "version: {version}");
}

#[wasm_bindgen_test]
fn wasm_capabilities_reasonable() {
    let canvas = make_canvas();
    let gl = webgl2::context(&canvas).unwrap();
    let caps = webgl2::query_capabilities(&gl);
    assert!(caps.max_vertex_attribs >= 16);
    assert!(caps.max_vertex_uniform_vectors >= 128);
    assert!(caps.max_texture_size >= 2048);
    assert!(caps.max_3d_texture_size >= 256);
}

const VS_INSTANCED: &str = r#"#version 300 es
in vec2 a_position;
in vec2 a_instance;
in vec3 a_color;
in float a_scale;
uniform float u_time;
out vec3 v_color;
void main() {
    vec2 p = a_position * a_scale + a_instance;
    gl_Position = vec4(p, 0.0, 1.0);
    v_color = a_color;
}
"#;

const FS_PASSTHROUGH: &str = r#"#version 300 es
precision mediump float;
in vec3 v_color;
out vec4 outColor;
void main() {
    outColor = vec4(v_color, 1.0);
}
"#;

#[wasm_bindgen_test]
fn wasm_instanced_shader_compiles() {
    let canvas = make_canvas();
    let gl = webgl2::context(&canvas).unwrap();
    assert!(shader::compile_vertex(&gl, VS_INSTANCED).is_ok());
    assert!(shader::compile_fragment(&gl, FS_PASSTHROUGH).is_ok());
}

#[wasm_bindgen_test]
fn wasm_instanced_renderer_creates() {
    let canvas = make_canvas();
    let gl = webgl2::context(&canvas).unwrap();
    let tri = geometry::base_triangle();
    let r = InstancedRenderer::new(&gl, VS_INSTANCED, FS_PASSTHROUGH, &tri)
        .expect("renderer создан");
    assert_eq!(r.vertex_count, 3);
    assert_eq!(r.loaded_instances, 0);
}

#[wasm_bindgen_test]
fn wasm_instanced_10k_ok() {
    let canvas = make_canvas();
    let gl = webgl2::context(&canvas).unwrap();
    let tri = geometry::base_triangle();
    let mut r = InstancedRenderer::new(&gl, VS_INSTANCED, FS_PASSTHROUGH, &tri).unwrap();
    let data = geometry::generate_instances(10_000);
    r.upload_instances(&gl, &data).unwrap();
    gl.clear_color(0.0, 0.0, 0.0, 1.0);
    gl.clear(web_sys::WebGl2RenderingContext::COLOR_BUFFER_BIT);
    let calls = r.draw(&gl, 0.0);
    assert_eq!(calls, 1);
    assert!(webgl2::check_error(&gl).is_none());
}
```

### `src/app.rs`

```rust
//! Глава 43: WebGL 2 — продвинутые возможности GPU.
//!
//! Демонстрирует:
//!   1. Capabilities WebGL2-контекста (limits + extensions).
//!   2. Instancing через drawArraysInstanced + vertexAttribDivisor.
//!   3. Transform Feedback: GPU считает, CPU читает.
//!   4. FPS-счётчик на разных N (100..50000).
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
use crate::webgl2::{
    self,
    geometry,
    shader,
    Capabilities, Program, WebGl2Error,
};

// ============================================================
//                    ШЕЙДЕРЫ — INSTANCING
// ============================================================

const VS_INSTANCED: &str = r#"#version 300 es
in vec2 a_position;
in vec2 a_instance;
in vec3 a_color;
in float a_scale;

uniform float u_time;
uniform float u_point_size;
out vec3 v_color;

void main() {
    float wobble = sin(u_time * 2.0 + a_instance.x * 5.0) * 0.02;
    vec2 p = a_position * a_scale + a_instance + vec2(wobble, 0.0);
    gl_Position = vec4(p, 0.0, 1.0);
    gl_PointSize = u_point_size;
    v_color = a_color;
}
"#;

const FS_INSTANCED: &str = r#"#version 300 es
precision mediump float;
in vec3 v_color;
out vec4 outColor;
void main() {
    outColor = vec4(v_color, 1.0);
}
"#;

// ============================================================
//                    ШЕЙДЕРЫ — TRANSFORM FEEDBACK
// ============================================================

const VS_TRANSFORM: &str = r#"#version 300 es
in float a_input;
out float v_result;
void main() {
    v_result = a_input * 2.0 + 1.0;
    gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
}
"#;

const FS_TRANSFORM: &str = r#"#version 300 es
precision mediump float;
out vec4 outColor;
void main() {
    outColor = vec4(0.0);
}
"#;

const MAX_INSTANCES: usize = 50_000;
const DEFAULT_COUNT: usize = 5_000;
const TF_INPUT_LEN: usize = 8;

// ============================================================
//                    HTML
// ============================================================
//
// ВНИМАНИЕ: r##"..."## вместо r#"..."# — в CSS есть "#3498db"
// и т.п. Последовательность `"#` внутри содержимого безопасна,
// пока разделитель — `"##`.

const HTML: &str = r##"
<style>
  :root {
    --g2-info: #3498db; --g2-ok: #27ae60; --g2-err: #c0392b;
    --g2-warn: #e67e22; --g2-gpu: #9b59b6; --g2-tf: #16a085;
    --g2-text: #222; --g2-muted: #666;
    --g2-bg: #fff; --g2-soft: #fafafa; --g2-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --g2-text: #e6e6e6; --g2-muted: #a0a0a0;
      --g2-bg: #1e1e1e; --g2-soft: #262626; --g2-border: #3a3a3a;
    }
    body { background: #121212; }
    .g2-out { background: #1a1a1a; color: #9ed09e; border-color: #333; }
    .g2-hint { background: #232014; color: #e6c98c; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--g2-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; max-width: 900px; margin: 0 auto; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .g2-subtitle { color: var(--g2-muted); font-size: 12px; margin: 0 0 14px; }
  .g2-subtitle a { color: var(--g2-info); }

  .g2-panel {
    border: 1px solid var(--g2-border); border-radius: 8px;
    padding: 10px 12px; background: var(--g2-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    margin-bottom: 12px;
  }
  .g2-panel-gpu { border: 2px solid var(--g2-gpu); }
  .g2-panel-tf { border: 2px solid var(--g2-tf); }
  .g2-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 8px;
    display: flex; align-items: center; gap: 6px;
  }
  .g2-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--g2-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .g2-num-gpu { background: var(--g2-gpu); }
  .g2-num-tf { background: var(--g2-tf); }

  .g2-row { display: flex; gap: 6px; flex-wrap: wrap; margin: 6px 0; align-items: center; }
  .g2-label { font-size: 11px; color: var(--g2-muted); min-width: 100px; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--g2-soft); color: var(--g2-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
    transition: background 0.15s, opacity 0.15s;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.g2-primary { background: #eaf4ff; border-color: var(--g2-info); color: #1f4e79; }
  button.g2-gpu { background: #f4ecf8; border-color: var(--g2-gpu); color: #6c3483; }
  button.g2-tf { background: #e6f4f1; border-color: var(--g2-tf); color: #0b5345; }

  .g2-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
    margin-left: auto;
    transition: background 0.2s, color 0.2s;
  }
  .g2-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .g2-badge.err { background: #fdedec; color: #922b21; }
  .g2-badge.warn { background: #fff9e6; color: #7d5a00; }

  .g2-stat {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--g2-muted);
  }
  .g2-stat b { color: var(--g2-ok); font-weight: 700; }

  .g2-out {
    background: #f6f8fa; border: 1px solid var(--g2-border);
    border-radius: 4px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #2d5a2d; overflow-wrap: anywhere;
    margin: 4px 0; min-height: 16px; white-space: pre-wrap;
  }
  .g2-out.empty { color: var(--g2-muted); font-style: italic; }

  .g2-kv { display: grid; grid-template-columns: max-content 1fr; gap: 2px 12px;
    font-family: ui-monospace, monospace; font-size: 11px; margin: 4px 0; }
  .g2-kv .k { color: var(--g2-muted); }
  .g2-kv .v { color: var(--g2-text); }
  .g2-kv .v.ok { color: var(--g2-ok); font-weight: 700; }
  .g2-kv .v.err { color: var(--g2-err); font-weight: 700; }

  .g2-hint {
    background: #fff9e6; border-left: 3px solid var(--g2-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 0 0 10px;
    color: #7d5a00;
  }
  .g2-note {
    font-size: 11px; color: var(--g2-muted); margin: 4px 0 0;
    line-height: 1.5;
  }
  .g2-note b { color: var(--g2-text); }
  .g2-note code { background: rgba(0,0,0,0.06); padding: 1px 4px; border-radius: 3px; }

  canvas {
    display: block; width: 100%; height: 300px;
    border-radius: 4px; background: #000;
  }
  .g2-canvas-wrap {
    border: 1px solid var(--g2-border); border-radius: 6px;
    overflow: hidden; margin-top: 6px;
  }

  input[type="range"] { width: 100%; }
  select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--g2-bg); color: var(--g2-text);
    font-family: inherit; cursor: pointer;
  }
</style>

<h1>Глава 43: WebGL 2</h1>
<p class="g2-subtitle">
  Instancing, transform feedback и производительность на больших N.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="g2-hint" id="g2-hint"></div>

<!-- ============ 1. Capabilities ============ -->
<div class="g2-panel">
  <h2 class="g2-panel-title">
    <span class="g2-num">1</span>Возможности WebGL2
    <span class="g2-badge" id="g2-caps-status">ожидание</span>
  </h2>
  <div class="g2-row">
    <button id="g2-caps-btn" class="g2-primary" disabled>Показать capabilities</button>
  </div>
  <div id="g2-caps-out" class="g2-kv"></div>
</div>

<!-- ============ 2. Instancing ============ -->
<div class="g2-panel g2-panel-gpu" id="g2-panel-draw">
  <h2 class="g2-panel-title">
    <span class="g2-num g2-num-gpu">2</span>Instancing
    <span class="g2-badge" id="g2-draw-status">ожидание</span>
  </h2>

  <div class="g2-row">
    <button id="g2-init-btn" class="g2-gpu" disabled>🔧 Инициализировать</button>
    <button id="g2-draw-btn" class="g2-gpu" disabled>▶ Запустить</button>
    <button id="g2-stop-btn" class="g2-primary" disabled>■ Остановить</button>
    <span class="g2-stat" id="g2-stats"></span>
  </div>

  <div class="g2-canvas-wrap">
    <canvas id="g2-canvas" width="800" height="300"></canvas>
  </div>

  <div class="g2-row">
    <div style="flex:1; min-width:200px;">
      <div class="g2-label">
        Инстансы: <span id="g2-count-label">5000</span>
      </div>
      <input type="range" id="g2-count" min="100" max="50000" step="100" value="5000">
    </div>
  </div>

  <div class="g2-row">
    <div style="min-width:200px;">
      <div class="g2-label">Примитив</div>
      <select id="g2-primitive">
        <option value="triangles">TRIANGLES</option>
        <option value="points">POINTS</option>
      </select>
    </div>
  </div>

  <div class="g2-out empty" id="g2-draw-out">—</div>

  <p class="g2-note">
    <b>Один draw call</b> на N инстансов через <code>drawArraysInstanced</code>.
    Каждый инстанс читает свои атрибуты из VBO через <code>vertexAttribDivisor(1)</code>.
  </p>
</div>

<!-- ============ 3. Transform Feedback ============ -->
<div class="g2-panel g2-panel-tf">
  <h2 class="g2-panel-title">
    <span class="g2-num g2-num-tf">3</span>Transform Feedback
    <span class="g2-badge" id="g2-tf-status">ожидание</span>
  </h2>

  <div class="g2-row">
    <button id="g2-tf-btn" class="g2-tf">⚙ Посчитать на GPU</button>
  </div>

  <div class="g2-out empty" id="g2-tf-out">—</div>

  <p class="g2-note">
    Вывод вершинного шейдера уходит в буфер, а не на экран. GPU умножает
    <code>input[i]</code> на 2 и прибавляет 1: <code>output[i] = input[i] * 2 + 1</code>.
  </p>
</div>

<!-- ============ 4. Анатомия ============ -->
<div class="g2-panel">
  <h2 class="g2-panel-title">
    <span class="g2-num">4</span>Анатомия instanced draw call
  </h2>
  <div class="g2-out" id="g2-anatomy">—</div>
</div>

<!-- ============ Чек-лист ============ -->
<div class="g2-panel">
  <h2 class="g2-panel-title">📋 Что охвачено в этой главе</h2>
  <ul class="g2-note" style="display:grid; grid-template-columns: repeat(auto-fill, minmax(220px,1fr)); gap:2px 16px; list-style:none; padding:0;">
    <li>✅ WebGl2RenderingContext</li>
    <li>✅ advanced buffers (VAO + divisors)</li>
    <li>✅ instancing (drawArraysInstanced)</li>
    <li>✅ transform feedback</li>
    <li>✅ floating-point textures — probe</li>
    <li>✅ Rust/WASM rendering</li>
  </ul>
</div>
"##;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum Primitive {
    Triangles,
    Points,
}

impl Primitive {
    fn from_str(s: &str) -> Self {
        return if s == "points" { Self::Points } else { Self::Triangles };
    }
    fn gl_mode(self) -> u32 {
        return match self {
            Self::Triangles => web_sys::WebGl2RenderingContext::TRIANGLES,
            Self::Points => web_sys::WebGl2RenderingContext::POINTS,
        };
    }
    fn point_size(self) -> f32 {
        return match self {
            Self::Triangles => 1.0,
            Self::Points => 4.0,
        };
    }
}

struct InstancingRenderer {
    program: Program,
    vao: web_sys::WebGlVertexArrayObject,
    /// Держим живым — wasm-bindgen не считает ссылки внутри GPU-объектов.
    #[allow(dead_code)]
    vbo_base: web_sys::WebGlBuffer,
    vbo_instance: web_sys::WebGlBuffer,
    loaded: usize,
    primitive: Primitive,
}

struct TfResult {
    input: Vec<f32>,
    output: Vec<f32>,
}

struct Ui {
    caps_btn: HtmlElement,
    caps_status: web_sys::Element,
    caps_out: web_sys::Element,

    init_btn: HtmlElement,
    draw_btn: HtmlElement,
    stop_btn: HtmlElement,
    draw_status: web_sys::Element,
    canvas: HtmlCanvasElement,
    draw_out: web_sys::Element,
    stats: web_sys::Element,
    count: HtmlInputElement,
    count_label: web_sys::Element,
    primitive: HtmlSelectElement,

    tf_btn: HtmlElement,
    tf_status: web_sys::Element,
    tf_out: web_sys::Element,

    anatomy: web_sys::Element,
}

struct State {
    gl: Option<web_sys::WebGl2RenderingContext>,
    capabilities: Option<Capabilities>,
    renderer: Option<InstancingRenderer>,
    all_instances: Vec<f32>,
    current_count: usize,
    running: bool,
    start_time: f64,
    last_fps_time: f64,
    frame_count: u32,
    last_frame_ms: f64,
    tf_last: Option<TfResult>,
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

    let ui = Ui {
        caps_btn: get_btn("g2-caps-btn")?,
        caps_status: get_el("g2-caps-status")?,
        caps_out: get_el("g2-caps-out")?,

        init_btn: get_btn("g2-init-btn")?,
        draw_btn: get_btn("g2-draw-btn")?,
        stop_btn: get_btn("g2-stop-btn")?,
        draw_status: get_el("g2-draw-status")?,
        canvas: get_el("g2-canvas")?.dyn_into()?,
        draw_out: get_el("g2-draw-out")?,
        stats: get_el("g2-stats")?,
        count: get_input("g2-count")?,
        count_label: get_el("g2-count-label")?,
        primitive: get_select("g2-primitive")?,

        tf_btn: get_btn("g2-tf-btn")?,
        tf_status: get_el("g2-tf-status")?,
        tf_out: get_el("g2-tf-out")?,

        anatomy: get_el("g2-anatomy")?,
    };

    let app = Rc::new(App {
        ui,
        state: RefCell::new(State {
            gl: None,
            capabilities: None,
            renderer: None,
            all_instances: geometry::generate_instances(MAX_INSTANCES),
            current_count: DEFAULT_COUNT,
            running: false,
            start_time: 0.0,
            last_fps_time: 0.0,
            frame_count: 0,
            last_frame_ms: 0.0,
            tf_last: None,
        }),
    });

    render_hint()?;
    render_anatomy(&app);
    bind_buttons(&app)?;
    bind_controls(&app)?;

    do_init_context(&app);

    Ok(())
}

// ============================================================
//                    BIND
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.caps_btn, move || {
            render_capabilities(&app_c);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.init_btn, move || {
            do_init_renderer(&app_c);
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
    {
        let app_c = app.clone();
        set_click(&app.ui.tf_btn, move || {
            do_transform_feedback(&app_c);
        });
    }
    Ok(())
}

fn bind_controls(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let n: usize = app_c.ui.count.value().parse().unwrap_or(DEFAULT_COUNT);
            app_c.state.borrow_mut().current_count = n;
            set_text(&app_c.ui.count_label, &format_str(format_args!("{n}")));
            if app_c.state.borrow().renderer.is_some() {
                let _ = upload_current_instances(&app_c);
            }
            render_anatomy(&app_c);
        });
        app.ui.count.add_event_listener_with_callback(
            "input", cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let p = Primitive::from_str(&app_c.ui.primitive.value());
            let mut st = app_c.state.borrow_mut();
            if let Some(r) = st.renderer.as_mut() {
                r.primitive = p;
            }
        });
        app.ui.primitive.add_event_listener_with_callback(
            "change", cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }
    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn do_init_context(app: &Rc<App>) {
    set_status(&app.ui.caps_status, "warn", "получаем контекст…");

    match webgl2::context(&app.ui.canvas) {
        Ok(gl) => {
            let caps = webgl2::query_capabilities(&gl);
            {
                let mut st = app.state.borrow_mut();
                st.gl = Some(gl);
                st.capabilities = Some(caps);
            }
            set_status(&app.ui.caps_status, "ok", "готово");
            set_disabled(&app.ui.caps_btn, false);
            set_disabled(&app.ui.init_btn, false);
            render_capabilities(app);
        }
        Err(e) => {
            set_status(&app.ui.caps_status, "err", "ошибка");
            set_text(&app.ui.caps_out,
                &format_str(format_args!("❌ {}", e.message)));
        }
    }
}

fn do_init_renderer(app: &Rc<App>) {
    let gl = match app.state.borrow().gl.clone() {
        Some(g) => g,
        None => {
            set_status(&app.ui.draw_status, "err", "нет контекста");
            return;
        }
    };
    set_status(&app.ui.draw_status, "warn", "компиляция шейдеров…");

    let tri = geometry::base_triangle();
    let result = build_instancing_renderer(&gl, &tri);

    match result {
        Ok(r) => {
            app.state.borrow_mut().renderer = Some(r);
            set_status(&app.ui.draw_status, "ok", "готово");
            set_disabled(&app.ui.init_btn, true);
            set_disabled(&app.ui.draw_btn, false);
            set_out(&app.ui.draw_out, "✅ Рендерер создан. Нажмите «Запустить».");
            let _ = upload_current_instances(app);
        }
        Err(e) => {
            set_status(&app.ui.draw_status, "err", "ошибка");
            set_out(&app.ui.draw_out,
                &format_str(format_args!("❌ {}", e.message)));
        }
    }
}

fn do_start_render(app: &Rc<App>) {
    if app.state.borrow().renderer.is_none() {
        set_status(&app.ui.draw_status, "err", "сначала инициализация");
        return;
    }
    {
        let mut st = app.state.borrow_mut();
        st.running = true;
        let now = js_sys::Date::now();
        st.start_time = now;
        st.last_fps_time = now;
        st.frame_count = 0;
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

fn do_transform_feedback(app: &Rc<App>) {
    let gl = match app.state.borrow().gl.clone() {
        Some(g) => g,
        None => {
            set_status(&app.ui.tf_status, "err", "нет контекста");
            return;
        }
    };

    set_status(&app.ui.tf_status, "warn", "считаем на GPU…");
    match run_transform_feedback(&gl, TF_INPUT_LEN) {
        Ok(result) => {
            let mut text = String::with_capacity(512);
            text.push_str("input  → output\n");
            for i in 0..result.input.len() {
                text.push_str(&format_str(format_args!(
                    "{:>6}  → {:>6}\n",
                    result.input[i], result.output[i],
                )));
            }
            text.push_str(&format_str(format_args!(
                "\n{} значений посчитано на GPU, прочитано обратно на CPU.",
                result.input.len(),
            )));
            set_out(&app.ui.tf_out, &text);
            set_status(&app.ui.tf_status, "ok", "готово");
            app.state.borrow_mut().tf_last = Some(result);
        }
        Err(e) => {
            set_status(&app.ui.tf_status, "err", "ошибка");
            set_out(&app.ui.tf_out,
                &format_str(format_args!("❌ {}", e.message)));
        }
    }
}

// ============================================================
//                    ПОСТРОЕНИЕ РЕНДЕРЕРА
// ============================================================

fn build_instancing_renderer(
    gl: &web_sys::WebGl2RenderingContext,
    base_data: &[f32],
) -> Result<InstancingRenderer, WebGl2Error> {
    let vs = shader::compile_vertex(gl, VS_INSTANCED)?;
    let fs = shader::compile_fragment(gl, FS_INSTANCED)?;
    let raw = shader::link(gl, &vs, &fs)?;
    let mut program = Program::new(raw);

    let vbo_base = gl.create_buffer()
        .ok_or_else(|| WebGl2Error::new("create_buffer failed".to_string()))?;
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo_base));
    gl.buffer_data_with_array_buffer_view(
        web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
        &js_sys::Float32Array::from(base_data),
        web_sys::WebGl2RenderingContext::STATIC_DRAW,
    );

    let vbo_instance = gl.create_buffer()
        .ok_or_else(|| WebGl2Error::new("create_buffer failed".to_string()))?;
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo_instance));
    gl.buffer_data_with_array_buffer_view(
        web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
        &js_sys::Float32Array::new_with_length(0),
        web_sys::WebGl2RenderingContext::DYNAMIC_DRAW,
    );

    let vao = gl.create_vertex_array()
        .ok_or_else(|| WebGl2Error::new("create_vertex_array failed".to_string()))?;
    gl.bind_vertex_array(Some(&vao));

    let pos_loc = program.attrib_location(gl, "a_position");
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo_base));
    gl.enable_vertex_attrib_array(pos_loc);
    gl.vertex_attrib_pointer_with_i32(
        pos_loc, 2,
        web_sys::WebGl2RenderingContext::FLOAT,
        false, 2 * 4, 0,
    );

    let stride = 6 * 4;

    let inst_loc = program.attrib_location(gl, "a_instance");
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo_instance));
    gl.enable_vertex_attrib_array(inst_loc);
    gl.vertex_attrib_pointer_with_i32(
        inst_loc, 2,
        web_sys::WebGl2RenderingContext::FLOAT,
        false, stride, 0,
    );
    gl.vertex_attrib_divisor(inst_loc, 1);

    let color_loc = program.attrib_location(gl, "a_color");
    gl.enable_vertex_attrib_array(color_loc);
    gl.vertex_attrib_pointer_with_i32(
        color_loc, 3,
        web_sys::WebGl2RenderingContext::FLOAT,
        false, stride, 2 * 4,
    );
    gl.vertex_attrib_divisor(color_loc, 1);

    let scale_loc = program.attrib_location(gl, "a_scale");
    gl.enable_vertex_attrib_array(scale_loc);
    gl.vertex_attrib_pointer_with_i32(
        scale_loc, 1,
        web_sys::WebGl2RenderingContext::FLOAT,
        false, stride, 5 * 4,
    );
    gl.vertex_attrib_divisor(scale_loc, 1);

    gl.bind_vertex_array(None);

    return Ok(InstancingRenderer {
        program,
        vao,
        vbo_base,
        vbo_instance,
        loaded: 0,
        primitive: Primitive::Triangles,
    });
}

fn upload_current_instances(app: &Rc<App>) -> Result<(), WebGl2Error> {
    let gl = app.state.borrow().gl.clone()
        .ok_or_else(|| WebGl2Error::new("no gl".to_string()))?;

    let slice = {
        let st = app.state.borrow();
        let n = st.current_count.min(MAX_INSTANCES);
        st.all_instances[..n * 6].to_vec()
    };
    let count = slice.len() / 6;

    {
        let mut st = app.state.borrow_mut();
        if let Some(r) = st.renderer.as_mut() {
            gl.bind_buffer(
                web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                Some(&r.vbo_instance),
            );
            let array = js_sys::Float32Array::from(&slice[..]);
            if count <= r.loaded && r.loaded > 0 {
                gl.buffer_sub_data_with_i32_and_array_buffer_view(
                    web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                    0,
                    &array,
                );
            } else {
                gl.buffer_data_with_array_buffer_view(
                    web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
                    &array,
                    web_sys::WebGl2RenderingContext::DYNAMIC_DRAW,
                );
            }
            r.loaded = count;
        }
    }
    return Ok(());
}

// ============================================================
//                    TRANSFORM FEEDBACK
// ============================================================

fn run_transform_feedback(
    gl: &web_sys::WebGl2RenderingContext,
    n: usize,
) -> Result<TfResult, WebGl2Error> {
    let vs = shader::compile_vertex(gl, VS_TRANSFORM)?;
    let fs = shader::compile_fragment(gl, FS_TRANSFORM)?;
    let raw = shader::link_with_varyings(gl, &vs, &fs, Some(&["v_result"]))?;
    let mut program = Program::new(raw);

    let mut input: Vec<f32> = Vec::with_capacity(n);
    for i in 0..n {
        input.push((i + 1) as f32);
    }

    let vbo_in = gl.create_buffer()
        .ok_or_else(|| WebGl2Error::new("create_buffer failed".to_string()))?;
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo_in));
    gl.buffer_data_with_array_buffer_view(
        web_sys::WebGl2RenderingContext::ARRAY_BUFFER,
        &js_sys::Float32Array::from(&input[..]),
        web_sys::WebGl2RenderingContext::STATIC_DRAW,
    );

    let vbo_out = gl.create_buffer()
        .ok_or_else(|| WebGl2Error::new("create_buffer failed".to_string()))?;
    gl.bind_buffer(
        web_sys::WebGl2RenderingContext::TRANSFORM_FEEDBACK_BUFFER,
        Some(&vbo_out),
    );
    gl.buffer_data_with_i32(
        web_sys::WebGl2RenderingContext::TRANSFORM_FEEDBACK_BUFFER,
        (n * 4) as i32,
        web_sys::WebGl2RenderingContext::DYNAMIC_COPY,
    );

    let vao = gl.create_vertex_array()
        .ok_or_else(|| WebGl2Error::new("create_vertex_array failed".to_string()))?;
    gl.bind_vertex_array(Some(&vao));
    gl.bind_buffer(web_sys::WebGl2RenderingContext::ARRAY_BUFFER, Some(&vbo_in));
    let input_loc = program.attrib_location(gl, "a_input");
    gl.enable_vertex_attrib_array(input_loc);
    gl.vertex_attrib_pointer_with_i32(
        input_loc, 1,
        web_sys::WebGl2RenderingContext::FLOAT,
        false, 4, 0,
    );
    gl.bind_vertex_array(None);

    let tf = gl.create_transform_feedback()
        .ok_or_else(|| WebGl2Error::new("create_transform_feedback failed".to_string()))?;
    gl.bind_transform_feedback(
        web_sys::WebGl2RenderingContext::TRANSFORM_FEEDBACK,
        Some(&tf),
    );
    gl.bind_buffer_base(
        web_sys::WebGl2RenderingContext::TRANSFORM_FEEDBACK_BUFFER,
        0,
        Some(&vbo_out),
    );

    gl.enable(web_sys::WebGl2RenderingContext::RASTERIZER_DISCARD);

    program.use_program(gl);
    gl.bind_vertex_array(Some(&vao));
    gl.begin_transform_feedback(web_sys::WebGl2RenderingContext::POINTS);
    gl.draw_arrays(web_sys::WebGl2RenderingContext::POINTS, 0, n as i32);
    gl.end_transform_feedback();
    gl.bind_vertex_array(None);

    gl.disable(web_sys::WebGl2RenderingContext::RASTERIZER_DISCARD);

    // Читаем output буфер обратно.
    gl.bind_buffer(
        web_sys::WebGl2RenderingContext::TRANSFORM_FEEDBACK_BUFFER,
        Some(&vbo_out),
    );
    let readback = js_sys::Float32Array::new_with_length(n as u32);
    // ВНИМАНИЕ: get_buffer_sub_data_with_i32_and_array_buffer_view
    // возвращает () — не Result. Ошибки читаются через check_error.
    gl.get_buffer_sub_data_with_i32_and_array_buffer_view(
        web_sys::WebGl2RenderingContext::TRANSFORM_FEEDBACK_BUFFER,
        0,
        &readback,
    );
    if let Some(err) = webgl2::check_error(gl) {
        return Err(WebGl2Error::new(err));
    }
    let output: Vec<f32> = readback.to_vec();

    gl.delete_transform_feedback(Some(&tf));

    let _ = fs;
    let _ = vbo_in;
    let _ = vbo_out;
    let _ = vao;

    return Ok(TfResult { input, output });
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
    let elapsed = (frame_start - app.state.borrow().start_time) / 1000.0;

    gl.clear_color(0.06, 0.06, 0.10, 1.0);
    gl.clear(web_sys::WebGl2RenderingContext::COLOR_BUFFER_BIT);

    // Draw: borrow_mut живёт только внутри этого блока.
    let (draw_calls, instances) = {
        let mut st = app.state.borrow_mut();
        match st.renderer.as_mut() {
            Some(r) => {
                let inst = r.loaded;
                let calls = instanced_draw(&gl, r, elapsed as f32);
                (calls, inst)
            }
            None => (0, 0),
        }
    };

    if let Some(err) = webgl2::check_error(&gl) {
        set_out(&app.ui.draw_out,
            &format_str(format_args!("⚠ {err}")));
    }

    let frame_ms = js_sys::Date::now() - frame_start;

    // FPS-счётчик: собираем строку внутри блока, выводим после.
    let fps_line: Option<String> = {
        let mut st = app.state.borrow_mut();
        st.frame_count += 1;
        st.last_frame_ms = frame_ms;
        let now = js_sys::Date::now();
        let dt = now - st.last_fps_time;
        if dt >= 1000.0 {
            let fps = st.frame_count as f64 * 1000.0 / dt;
            let ms = st.last_frame_ms;
            let total_bytes = st.all_instances.len() * 4;
            let used_bytes = instances * 6 * 4;
            let big_vbo_bytes = instances * 3 * 5 * 4;

            st.frame_count = 0;
            st.last_fps_time = now;

            Some(format_str(format_args!(
                "FPS {fps:.0} · {ms:.2} ms · {instances} inst · {draw_calls} draw \
                 · {used_bytes} B on GPU (ch42: {big_vbo_bytes} B) · total {total_bytes} B",
            )))
        } else {
            None
        }
    };

    if let Some(line) = fps_line {
        set_text(&app.ui.stats, &line);
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

fn instanced_draw(
    gl: &web_sys::WebGl2RenderingContext,
    r: &mut InstancingRenderer,
    time: f32,
) -> u32 {
    if r.loaded == 0 {
        return 0;
    }
    r.program.use_program(gl);
    gl.bind_vertex_array(Some(&r.vao));

    if let Some(loc) = r.program.uniform_location(gl, "u_time") {
        gl.uniform1f(Some(&loc), time);
    }
    if let Some(loc) = r.program.uniform_location(gl, "u_point_size") {
        gl.uniform1f(Some(&loc), r.primitive.point_size());
    }

    gl.draw_arrays_instanced(
        r.primitive.gl_mode(),
        0,
        3,
        r.loaded as i32,
    );

    gl.bind_vertex_array(None);
    return 1;
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_hint() -> Result<(), JsValue> {
    let hint = get_el("g2-hint")?;
    let html = "WebGL2 добавляет к базовому конвейеру три ключевые вещи:<br>\
                <b>1. Instancing</b> — <code>drawArraysInstanced</code> рисует N копий \
                одной геометрии за один draw call.<br>\
                <b>2. Transform Feedback</b> — вывод вершинного шейдера уходит в буфер, \
                а не на экран. GPGPU: физика, частицы, сортировка.<br>\
                <b>3. Расширения</b> — <code>color_buffer_float</code>, <code>timer_query</code> — \
                меняют класс задач, которые можно решить.<br>\
                <b>Попробуйте:</b> двигайте слайдер N, переключайте TRIANGLES ↔ POINTS, \
                запустите Transform Feedback.";
    hint.set_inner_html(html);
    Ok(())
}

fn render_capabilities(app: &Rc<App>) {
    let st = app.state.borrow();
    let caps = match &st.capabilities {
        Some(c) => c,
        None => {
            set_text(&app.ui.caps_out, "—");
            return;
        }
    };

    let mut html = String::with_capacity(512);
    push_kv(&mut html, "MAX_VERTEX_ATTRIBS", &format_str(format_args!(
        "{}", caps.max_vertex_attribs,
    )));
    push_kv(&mut html, "MAX_VERTEX_UNIFORM_VECTORS", &format_str(format_args!(
        "{}", caps.max_vertex_uniform_vectors,
    )));
    push_kv(&mut html, "MAX_TEXTURE_SIZE", &format_str(format_args!(
        "{}", caps.max_texture_size,
    )));
    push_kv(&mut html, "MAX_3D_TEXTURE_SIZE", &format_str(format_args!(
        "{}", caps.max_3d_texture_size,
    )));
    push_kv(&mut html, "MAX_SAMPLES", &format_str(format_args!(
        "{}", caps.max_samples,
    )));
    push_kv_cls(&mut html, "EXT_color_buffer_float",
        yes_no(caps.color_buffer_float),
        if caps.color_buffer_float { "ok" } else { "" });
    push_kv_cls(&mut html, "EXT_disjoint_timer_query_webgl2",
        yes_no(caps.timer_query),
        if caps.timer_query { "ok" } else { "" });
    push_kv_cls(&mut html, "OES_texture_float_linear",
        yes_no(caps.float_linear),
        if caps.float_linear { "ok" } else { "" });
    app.ui.caps_out.set_inner_html(&html);
}

fn render_anatomy(app: &Rc<App>) {
    let (n, primitive) = {
        let st = app.state.borrow();
        let p = st.renderer.as_ref()
            .map(|r| r.primitive)
            .unwrap_or(Primitive::Triangles);
        (st.current_count, p)
    };

    let instanced_floats = 6 + n * 6;
    let instanced_bytes = instanced_floats * 4;
    let big_vbo_floats = n * 3 * 5;
    let big_vbo_bytes = big_vbo_floats * 4;
    let savings_pct = if big_vbo_bytes > 0 {
        100.0 * (big_vbo_bytes as f64 - instanced_bytes as f64)
            / big_vbo_bytes as f64
    } else {
        0.0
    };

    let mode_str = match primitive {
        Primitive::Triangles => "TRIANGLES",
        Primitive::Points => "POINTS",
    };

    let mut out = String::with_capacity(768);
    out.push_str("Instanced draw call — 6 шагов:\n\n");
    out.push_str("useProgram()\n");
    out.push_str("bindVertexArray(vao)\n");
    out.push_str("  ↳ a_position    per-vertex   (divisor 0)\n");
    out.push_str("  ↳ a_instance/color/scale  per-instance (divisor 1)\n");
    out.push_str("uniform1f(u_time, ...)\n");
    out.push_str(&format_str(format_args!(
        "uniform1f(u_point_size, {})\n",
        primitive.point_size(),
    )));
    out.push_str(&format_str(format_args!(
        "drawArraysInstanced({mode_str}, 0, 3, {n})\n",
    )));
    out.push_str("  → 1 draw call\n");
    out.push_str(&format_str(format_args!(
        "  → {} вершин обработано на GPU\n\n",
        n * 3,
    )));

    out.push_str("Память VBO:\n");
    out.push_str(&format_str(format_args!(
        "  instancing (ch43): {instanced_floats} float = {instanced_bytes} B\n",
    )));
    out.push_str(&format_str(format_args!(
        "  big VBO    (ch42): {big_vbo_floats} float = {big_vbo_bytes} B\n",
    )));
    out.push_str(&format_str(format_args!(
        "  экономия: {savings_pct:.1}%\n",
    )));

    set_out(&app.ui.anatomy, &out);
}

fn yes_no(b: bool) -> &'static str {
    return if b { "✅" } else { "❌" };
}

fn push_kv(html: &mut String, k: &str, v: &str) {
    push_kv_cls(html, k, v, "");
}

fn push_kv_cls(html: &mut String, k: &str, v: &str, cls: &str) {
    html.push_str("<span class=\"k\">");
    html.push_str(&esc_html(k));
    html.push_str("</span><span class=\"v");
    if !cls.is_empty() {
        html.push(' ');
        html.push_str(cls);
    }
    html.push_str("\">");
    html.push_str(&esc_html(v));
    html.push_str("</span>");
}

fn set_text(el: &web_sys::Element, text: &str) {
    el.set_text_content(Some(text));
}

fn set_out(el: &web_sys::Element, text: &str) {
    el.set_text_content(Some(text));
    el.set_class_name("g2-out");
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "g2-badge ok",
        "err" => "g2-badge err",
        "warn" => "g2-badge warn",
        _ => "g2-badge",
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

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
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

---

## 8. Команды запуска

```bash
# Сборка
npm run wasm:dev

# Тесты главы 43
wasm-pack test --chrome --headless -- --test wasm_webgl2

# Firefox
wasm-pack test --firefox --headless -- --test wasm_webgl2

# Демо
npm run dev
# Открыть http://localhost:5173
```

**Ожидаемый результат тестов:** ~7 passed.

---

## 9. Итог

WebGL 2 — это не «WebGL 1 с плюшками», а качественно другой уровень. Три ключевых отличия определяют, почему современный GPU-рендеринг в браузере стал возможен:

1. **Instancing** — один `drawArraysInstanced` рисует N объектов. 50 000 частиц при 60 FPS без единой заминки.
2. **Transform Feedback** — GPU умеет не только рисовать, но и считать. Физика, частицы, симуляции — всё, что раньше требовало Web Workers или WebAssembly с SIMD, теперь умещается в один шейдер.
3. **Capabilities & extensions** — программа знает, что ей доступно: `MAX_3D_TEXTURE_SIZE`, `color_buffer_float`, `timer_query`. Она адаптируется к железу, а не пытается работать «в среднем».

Уроки из ошибок: **borrow checker строже, чем кажется** (`drop(st)` — плохая идея), **web-sys возвращает `Result` не везде** (`get_buffer_sub_data` — исключение), **raw-строки ломаются на `"#`** (используйте `r##"..."##`), **`get_attrib_location` — `i32`, а всё остальное — `u32`** (кастуйте один раз при кэшировании).

Эти грабли встретятся в любом WebGL-проекте на Rust. Их не нужно запоминать наизусть — достаточно знать, что они есть, и уметь быстро найти причину. Дальше — только структура данных и алгоритмы, и здесь WebGL 2 даёт полный арсенал.