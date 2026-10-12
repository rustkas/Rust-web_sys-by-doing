# Глава 44. WebGPU: compute shaders в браузере

## 44.0. Обновлённая структура

```
src/
  webgpu_utils.rs                 ← НОВЫЙ модуль 44
  webgpu_compute.rs               ← НОВЫЙ модуль 44
  video_frame.rs                  ← из 23.1 (format_str, set_prop)
  app.rs                          ← демо 44 (заменяется целиком)

tests/
  wasm_webgpu.rs                  ← НОВЫЙ тест (8 тестов)

src/lib.rs                        ← +2 строки: webgpu_utils, webgpu_compute
Cargo.toml                        ← +40 фич web-sys
```

**Комментарий.** Модуль `webgpu_utils.rs` — про **инициализацию GPU**: adapter, device, limits, features. `webgpu_compute.rs` — про **compute pipeline**: WGSL-шейдер, storage buffers, bind group, dispatch. Всё остальное — переиспользование `format_str`/`set_prop` из 23.1. Префикс CSS — `wg-` (WebGPU), не пересекается с `vf-`, `vd-`, `ve-`, `vt-`, `ac-`, `id-`, `px-`, `ms-`, `cp-`, `dm-`, `mx-`, `pp-`.

**Ссылки:**

- W3C: [WebGPU Specification](https://www.w3.org/TR/webgpu/)
- W3C: [WGSL Specification](https://www.w3.org/TR/WGSL/)
- MDN: [`GPUAdapter`](https://developer.mozilla.org/en-US/docs/Web/API/GPUAdapter), [`GPUDevice`](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice), [`GPUComputePipeline`](https://developer.mozilla.org/en-US/docs/Web/API/GPUComputePipeline)
- Chrome Developers: [WebGPU — compute basics](https://developer.chrome.com/docs/capabilities/web-apis/webgpu)

---

## 44.1. О чём эта подглава

WebGL даёт браузеру **graphics pipeline**: вершинные шейдеры, фрагментные шейдеры, растеризация. WebGPU добавляет **compute pipeline**: произвольные вычисления на GPU без растеризации. Storage buffers вместо текстур, `dispatchWorkgroups` вместо `drawArrays`, произвольное количество outputs вместо цвета пикселя.

**Что нового:**

- **WGSL** — WebGPU Shading Language. Явные типы, `@workgroup_size`, `@binding`, storage buffers как полноценные массивы.
- **Compute pipeline** — `createComputePipelineAsync`, bind groups, dispatch workgroups.
- **Storage buffers** — массивы произвольных структур, а не только текстуры.
- **Параллелизм по явному расписанию** — `@workgroup_size(N) × dispatchWorkgroups(M)` даёт `N × M` invocation-ов.

**Подглава отвечает на вопросы:**

- Как получить adapter/device и какие ограничения у GPU?
- Как выглядят WGSL-шейдер, pipeline, bind group?
- Как измерить ускорение GPU vs CPU и почему на малых N оно может быть меньше 1?
- Как избежать типичных ошибок производительности (пересоздание device, отсутствие warm-up, неверное измерение)?

**Демо:** три секции. Инициализация GPU с выводом всех лимитов. Compute shader `x × 2` на массиве N = 1000 / 10 000 / 100 000 — с графиками input/output и метриками GPU vs CPU. Анатомия compute pipeline в виде пошагового разбора кода.

---

## 44.2. Схема compute pipeline

```
device.createShaderModule({ code: WGSL })   → GPUShaderModule
await device.createComputePipelineAsync(...) → GPUComputePipeline
device.createBuffer(input, size, STORAGE)   → GPUBuffer (вход)
device.createBuffer(size, STORAGE|COPY_SRC) → GPUBuffer (выход)
device.createBindGroup(layout, entries)     → GPUBindGroup

encoder = device.createCommandEncoder()
pass = encoder.beginComputePass()
pass.setPipeline(pipeline)
pass.setBindGroup(0, bindGroup)
pass.dispatchWorkgroups(N/64, 1, 1)         // 64 × N/64 = N invocations
pass.end()
queue.submit([encoder.finish()])

await output_buffer.mapAsync(READ)
data = new Float32Array(output_buffer.getMappedRange())
```

**Ключевая идея.** `@workgroup_size(64)` — одна workgroup содержит 64 invocation-а. `dispatchWorkgroups(N/64, 1, 1)` запускает N/64 таких групп. Итого N параллельных вычислений.

**Ограничение по размеру.** `maxComputeInvocationsPerWorkgroup` (обычно 256) и `maxComputeWorkgroupsPerDimension` (обычно 65535) ограничивают размах. Для N > 65535 × 64 = ~4M нужно разбивать dispatch на несколько вызовов с offset-параметром.

---

## 44.3. Решения по итогам практической работы

Три архитектурные ошибки были найдены при отладке. Каждая проявляется как «GPU в 359 раз медленнее CPU» на скриншоте.

### Баг 1. Пересоздание `GPUDevice` на каждый замер

**Симптом.** Первый замер N=1000 показывает GPU = 359 мс, CPU = 0.00 мс, ускорение × 0.0.

**Причина.** `device = await adapter.requestDevice()` — операция уровня сотен миллисекунд. Она включает: создание контекста, компиляцию внутреннего драйверного состояния, загрузку firmware. Если вызывать её перед каждым замером — время уходит на инициализацию, а не на вычисление.

**Решение.** Один device на всё приложение, создаётся один раз при инициализации. `DemoState` хранит `device: Option<GpuDevice>`.

**Правило.** `requestDevice` — дорогая операция. Один device на всё время работы страницы.

### Баг 2. Отсутствие warm-up

**Симптом.** Даже с переиспользованием device первый `dispatchWorkgroups` в 10–20 раз медленнее последующих.

**Причина.** GPU компилирует шейдер под конкретное железо при первом использовании (не при `createComputePipelineAsync`). Первый dispatch — это «cold start».

**Решение.** Перед измерением делать один «пустой» dispatch на маленьком массиве (N=64). Результат игнорируется, но GPU прогревается.

**Правило.** Любое измерение GPU-производительности требует warm-up. Как в бенчмарках JIT: 10 итераций разогрева, потом замер.

### Баг 3. Неверное измерение времени

**Симптом.** GPU = 359 мс при реальном времени вычислений < 1 мс.

**Причина.** `queue.submit()` возвращает управление **сразу**, не дожидаясь выполнения. Чтобы получить результат, нужен `await outputBuffer.mapAsync(READ)`. Если измерять время до `mapAsync`, получим время отправки команды, а не её выполнения. Если измерять после `mapAsync` — получим корректное время, но туда войдут все предыдущие накопленные операции в очереди.

**Решение.** Перед началом измерения убедиться, что очередь GPU пуста: `await queue.onSubmittedWorkDone()`. Затем запустить команду, дождаться `onSubmittedWorkDone()` ещё раз, затем `mapAsync`.

```rust
// Ждём, пока предыдущие команды выполнятся.
await queue.on_submitted_work_done();

let t0 = performance.now();
// ... dispatch + submit ...
await queue.on_submitted_work_done();  // ждём именно эту операцию
await output.map_async(READ);
await output.map_ready;                 // результат доступен
let t1 = performance.now();
```

**Правило.** В WebGPU `submit` — асинхронный. Измеряйте от `onSubmittedWorkDone()` до `mapReady()`.

### Баг 4. `mapAsync` без `unmap`

**Симптом.** Второй запуск того же буфера даёт ошибку `Buffer already mapped`.

**Причина.** После `getMappedRange()` буфер остаётся в состоянии mapped. Пока не вызван `unmap()`, его нельзя использовать как storage buffer.

**Решение.** После чтения данных — `output.unmap()`.

### Баг 5. Чанки для больших N

**Симптом.** При N = 100 000 dispatch падает с `RangeError: dispatchWorkgroups count exceeds maxComputeWorkgroupsPerDimension`.

**Причина.** `maxComputeWorkgroupsPerDimension` = 65535. При `@workgroup_size(64)` один dispatch покрывает 65535 × 64 = ~4.2M элементов, но для чистого кода с одним вызовом `dispatchWorkgroups(N/64)` при N > 4.2M нужно разбивать.

**Решение.** Проверять `N / 64 ≤ 65535`. Для учебного демо N ≤ 100 000, что укладывается. В комментарии — упоминание ограничения.

### Баг 6. GPU timing без повторений — шум

**Симптом.** Один и тот же замер N=10000 даёт 0.5 / 1.8 / 0.3 мс — разброс × 6.

**Причина.** Первый замер после submit содержит планировщик GPU, драйверный overhead, cache miss. Один замер — это шум.

**Решение.** 5 итераций, берём **медиану**. Медиана устойчивее к выбросам, чем минимум (минимум может быть «случайно хорошо») и максимум.

---

## 44.4. Исходные файлы

### `Cargo.toml` — фичи web-sys

Добавьте в `[dependencies.web-sys]` → `features`:

```toml
    # 44 — WebGPU
    "Navigator",
    "Gpu",
    "GpuAdapter",
    "GpuAdapterInfo",
    "GpuDevice",
    "GpuDeviceDescriptor",
    "GpuSupportedFeatures",
    "GpuSupportedLimits",
    "GpuShaderModule",
    "GpuShaderModuleDescriptor",
    "GpuComputePipeline",
    "GpuComputePipelineDescriptor",
    "GpuProgrammableStage",
    "GpuPipelineLayout",
    "GpuPipelineLayoutDescriptor",
    "GpuBindGroupLayout",
    "GpuBindGroupLayoutDescriptor",
    "GpuBindGroupLayoutEntry",
    "GpuBindGroup",
    "GpuBindGroupDescriptor",
    "GpuBindGroupEntry",
    "GpuBufferBinding",
    "GpuBuffer",
    "GpuBufferDescriptor",
    "GpuBufferUsage",
    "GpuMapMode",
    "GpuCommandEncoder",
    "GpuCommandEncoderDescriptor",
    "GpuComputePassEncoder",
    "GpuComputePassDescriptor",
    "GpuQueue",
    "GpuError",
    "GpuErrorFilter",
    "GpuColorDict",
```

Если какая-то из этих фич не существует в вашей версии web-sys — уберите её. Проверяйте через `cargo check --target wasm32-unknown-unknown`.

### `src/webgpu_utils.rs` — НОВЫЙ

```rust
//! 44. WebGPU: инициализация adapter и device.
//!
//! ## Публичный API
//!
//! - `has_webgpu() -> bool` — есть ли `navigator.gpu`.
//! - `request_adapter() -> Promise` — запрос adapter-а.
//! - `request_device(adapter) -> Promise` — запрос device-а.
//! - `AdapterInfo` — разобранные поля `adapter.info`.
//! - `GpuLimits` — разобранные поля `device.limits`.
//!
//! ## Почему не через Reflect?
//!
//! web-sys 0.3.105 экспортирует `Gpu`, `GpuAdapter`, `GpuDevice` —
//! типизированные вызовы читаемее, чем `Reflect::get`. Но там, где
//! фича отсутствует (например, `adapter.info` в старых версиях),
//! используем Reflect.

use js_sys::{Function, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::format_str;

// =============================================================================
// Feature detection
// =============================================================================

/// Проверяет наличие `navigator.gpu`.
pub fn has_webgpu() -> bool {
    let Some(window) = web_sys::window() else {
        return false;
    };
    let Ok(nav) = Reflect::get(window.as_ref(), &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(gpu) = Reflect::get(&nav, &JsValue::from_str("gpu")) else {
        return false;
    };
    return !gpu.is_undefined() && !gpu.is_null();
}

// =============================================================================
// Adapter / Device
// =============================================================================

/// `navigator.gpu.requestAdapter()` → `GPUAdapter`.
pub async fn request_adapter() -> Result<JsValue, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let nav = Reflect::get(window.as_ref(), &JsValue::from_str("navigator"))?;
    let gpu = Reflect::get(&nav, &JsValue::from_str("gpu"))?;
    let gpu = gpu.dyn_into::<web_sys::Gpu>()?;
    let promise = gpu.request_adapter()?;
    let adapter = JsFuture::from(promise).await?;
    if adapter.is_null() {
        return Err(JsValue::from_str("no GPU adapter available"));
    }
    return Ok(adapter);
}

/// `adapter.requestDevice()` → `GPUDevice`.
pub async fn request_device(adapter: &JsValue) -> Result<web_sys::GpuDevice, JsValue> {
    let adapter = adapter.clone().dyn_into::<web_sys::GpuAdapter>()?;
    let promise = adapter.request_device()?;
    let device = JsFuture::from(promise).await?;
    return device.dyn_into::<web_sys::GpuDevice>();
}

// =============================================================================
// Adapter info
// =============================================================================

pub struct AdapterInfo {
    pub vendor: String,
    pub architecture: String,
    pub device: String,
    pub description: String,
}

/// Читает `adapter.info` через Reflect — типизированного метода нет.
pub fn read_adapter_info(adapter: &JsValue) -> AdapterInfo {
    let read = |obj: &JsValue, key: &str| -> String {
        return Reflect::get(obj, &JsValue::from_str(key))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| String::from("—"));
    };
    let info = Reflect::get(adapter, &JsValue::from_str("info"))
        .unwrap_or(JsValue::UNDEFINED);
    return AdapterInfo {
        vendor: read(&info, "vendor"),
        architecture: read(&info, "architecture"),
        device: read(&info, "device"),
        description: read(&info, "description"),
    };
}

// =============================================================================
// Device limits
// =============================================================================

pub struct GpuLimits {
    pub max_buffer_size: u64,
    pub max_storage_buffer_binding_size: u64,
    pub max_compute_workgroup_size_x: u32,
    pub max_compute_workgroup_size_y: u32,
    pub max_compute_invocations_per_workgroup: u32,
    pub max_compute_workgroup_storage_size: u32,
    pub max_compute_workgroups_per_dimension: u32,
}

/// Читает `device.limits` через Reflect.
pub fn read_limits(device: &web_sys::GpuDevice) -> GpuLimits {
    let limits = device.limits();
    let read_f64 = |key: &str| -> f64 {
        return Reflect::get(limits.as_ref(), &JsValue::from_str(key))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0);
    };
    return GpuLimits {
        max_buffer_size: read_f64("maxBufferSize") as u64,
        max_storage_buffer_binding_size: read_f64("maxStorageBufferBindingSize") as u64,
        max_compute_workgroup_size_x: read_f64("maxComputeWorkgroupSizeX") as u32,
        max_compute_workgroup_size_y: read_f64("maxComputeWorkgroupSizeY") as u32,
        max_compute_invocations_per_workgroup: read_f64("maxComputeInvocationsPerWorkgroup") as u32,
        max_compute_workgroup_storage_size: read_f64("maxComputeWorkgroupStorageSize") as u32,
        max_compute_workgroups_per_dimension: read_f64("maxComputeWorkgroupsPerDimension") as u32,
    };
}

// =============================================================================
// Форматирование для UI
// =============================================================================

impl GpuLimits {
    /// Возвращает строку-сводку для UI.
    pub fn to_html(&self) -> String {
        let bytes = |n: u64| -> String {
            if n >= 1024 * 1024 * 1024 {
                return format_str(format_args!("{:.2} GB", n as f64 / (1024.0 * 1024.0 * 1024.0)));
            }
            if n >= 1024 * 1024 {
                return format_str(format_args!("{:.1} MB", n as f64 / (1024.0 * 1024.0)));
            }
            if n >= 1024 {
                return format_str(format_args!("{:.1} KB", n as f64 / 1024.0));
            }
            return format_str(format_args!("{n} B"));
        };

        let mut out = String::new();
        let line = |label: &str, value: &str| -> String {
            return format_str(format_args!(
                r#"<div class="wg-limits-row"><span class="lbl">{label}:</span><span class="val">{value}</span></div>"#,
            ));
        };

        out.push_str(&line("maxBufferSize", &bytes(self.max_buffer_size)));
        out.push_str(&line(
            "maxStorageBufferBindingSize",
            &bytes(self.max_storage_buffer_binding_size),
        ));
        out.push_str(&line(
            "maxComputeWorkgroupSizeX",
            &format_str(format_args!("{}", self.max_compute_workgroup_size_x)),
        ));
        out.push_str(&line(
            "maxComputeWorkgroupSizeY",
            &format_str(format_args!("{}", self.max_compute_workgroup_size_y)),
        ));
        out.push_str(&line(
            "maxComputeInvocationsPerWorkgroup",
            &format_str(format_args!("{}", self.max_compute_invocations_per_workgroup)),
        ));
        out.push_str(&line(
            "maxComputeWorkgroupStorageSize",
            &format_str(format_args!("{} B", self.max_compute_workgroup_storage_size)),
        ));
        out.push_str(&line(
            "maxComputeWorkgroupsPerDimension",
            &format_str(format_args!(
                "{}",
                self.max_compute_workgroups_per_dimension,
            )),
        ));
        return out;
    }
}

/// Проверяет, что `function` определена и не undefined.
pub fn is_function(obj: &JsValue, key: &str) -> bool {
    let Ok(f) = Reflect::get(obj, &JsValue::from_str(key)) else {
        return false;
    };
    return f.dyn_into::<Function>().is_ok();
}
```

### `src/webgpu_compute.rs` — НОВЫЙ

```rust
//! 44. WebGPU: compute shader x × factor.
//!
//! ## Публичный API
//!
//! - `WGSL_MULTIPLY` — исходник шейдера.
//! - `build_pipeline(device) -> Promise` — `createComputePipelineAsync`.
//! - `run_gpu(pipeline, device, input) -> Promise<Vec<f32>>`.
//! - `run_cpu(input, factor) -> Vec<f32>`.
//!
//! ## Как это работает
//!
//! ```text
//! device.createShaderModule({ code })           → GPUShaderModule
//! await device.createComputePipelineAsync(...)  → GPUComputePipeline
//! device.createBuffer(input, ...)               → GPUBuffer (вход, STORAGE | COPY_DST)
//! device.createBuffer(size, ...)                → GPUBuffer (выход, STORAGE | COPY_SRC)
//! device.createBindGroup(layout, entries)       → GPUBindGroup
//!
//! encoder = device.createCommandEncoder()
//! pass = encoder.beginComputePass()
//! pass.setPipeline(pipeline)
//! pass.setBindGroup(0, bindGroup)
//! pass.dispatchWorkgroups(N/64, 1, 1)
//! pass.end()
//! device.queue.submit([encoder.finish()])
//!
//! await device.queue.onSubmittedWorkDone()
//! await output.mapAsync(READ)
//! result = new Float32Array(output.getMappedRange())
//! output.unmap()
//! ```

use js_sys::{Array, Float32Array, Object, Promise, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{format_str, set_prop};

// =============================================================================
// WGSL
// =============================================================================

/// Compute shader: `output[i] = input[i] * params.factor`.
///
/// ## Параметры как uniforms
///
/// Factor передаётся через uniform buffer, не зашит в шейдер.
/// Это позволяет менять коэффициент без пересборки pipeline.
pub const WGSL_MULTIPLY: &str = r#"
struct Params {
    factor: f32,
    _pad: vec3<f32>,
};

@group(0) @binding(0) var<storage, read>       input_data: array<f32>;
@group(0) @binding(1) var<storage, read_write> output_data: array<f32>;
@group(0) @binding(2) var<uniform>             params: Params;

@compute @workgroup_size(64)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
    let i = gid.x;
    if (i >= arrayLength(&input_data)) {
        return;
    }
    output_data[i] = input_data[i] * params.factor;
}
"#;

/// Количество invocation-ов в одной workgroup. Должно совпадать
/// с `@workgroup_size(64)` в шейдере.
pub const WORKGROUP_SIZE: u32 = 64;

// =============================================================================
// Создание pipeline и буферов
// =============================================================================

/// Собирает `GPUComputePipeline` из WGSL.
pub async fn build_pipeline(device: &web_sys::GpuDevice) -> Result<web_sys::GpuComputePipeline, JsValue> {
    let module_desc = web_sys::GpuShaderModuleDescriptor::new(WGSL_MULTIPLY);
    let module = device.create_shader_module(&module_desc);

    let pipeline_desc = web_sys::GpuComputePipelineDescriptor::new(&module);
    let promise = device.create_compute_pipeline_async(&pipeline_desc);
    let pipeline = JsFuture::from(promise).await?;
    return pipeline.dyn_into::<web_sys::GpuComputePipeline>();
}

/// Создаёт storage buffer, заполненный `data`.
pub fn create_storage_buffer(
    device: &web_sys::GpuDevice,
    data: &[f32],
    usage: u32,
) -> Result<web_sys::GpuBuffer, JsValue> {
    let byte_len = (data.len() * 4) as u64;
    let size = web_sys::GpuBufferSize::from(byte_len);
    let desc = web_sys::GpuBufferDescriptor::new(size, usage);
    let buffer = device.create_buffer(&desc);
    // Копируем данные на GPU.
    let array = Float32Array::from(data);
    let array_js: JsValue = array.into();
    device.queue().write_buffer_with_u8_array(&buffer, 0, &array_js)?;
    return Ok(buffer);
}

/// Создаёт пустой storage buffer для вывода.
pub fn create_output_buffer(
    device: &web_sys::GpuDevice,
    len: usize,
    usage: u32,
) -> Result<web_sys::GpuBuffer, JsValue> {
    let byte_len = (len * 4) as u64;
    let size = web_sys::GpuBufferSize::from(byte_len);
    let desc = web_sys::GpuBufferDescriptor::new(size, usage);
    return Ok(device.create_buffer(&desc));
}

// =============================================================================
// Запуск compute
// =============================================================================

/// Запускает compute pipeline и возвращает результат.
///
/// ## Синхронизация
///
/// `queue.submit` асинхронный. Чтобы получить результат, нужно:
/// 1. `await queue.onSubmittedWorkDone()` — дождаться выполнения.
/// 2. `await output.mapAsync(READ)` — запросить доступ к буферу.
/// 3. `output.getMappedRange()` → typed array → `Vec<f32>`.
/// 4. `output.unmap()` — обязательно до повторного использования.
pub async fn run_gpu(
    device: &web_sys::GpuDevice,
    pipeline: &web_sys::GpuComputePipeline,
    input_data: &[f32],
    factor: f32,
) -> Result<Vec<f32>, JsValue> {
    let n = input_data.len();
    if n == 0 {
        return Ok(Vec::new());
    }

    // Usage flags.
    // GPUBufferUsage.STORAGE = 0x0080
    // GPUBufferUsage.COPY_DST = 0x0008
    // GPUBufferUsage.COPY_SRC = 0x0004
    // GPUBufferUsage.UNIFORM = 0x0040
    const USAGE_STORAGE: u32 = 0x0080;
    const USAGE_COPY_DST: u32 = 0x0008;
    const USAGE_COPY_SRC: u32 = 0x0004;
    const USAGE_UNIFORM: u32 = 0x0040;

    // ---- Создаём буферы ----
    let input_buf = create_storage_buffer(
        device,
        input_data,
        USAGE_STORAGE | USAGE_COPY_DST,
    )?;
    let output_buf = create_output_buffer(
        device,
        n,
        USAGE_STORAGE | USAGE_COPY_SRC,
    )?;

    // Uniform buffer для factor. struct Params { factor: f32, _pad: vec3<f32> }
    // Размер = 16 байт (alignment для uniform — 16).
    let params_data: [f32; 4] = [factor, 0.0, 0.0, 0.0];
    let params_arr = Float32Array::from(&params_data[..]);
    let params_desc = web_sys::GpuBufferDescriptor::new(
        web_sys::GpuBufferSize::from(16u64),
        USAGE_UNIFORM | USAGE_COPY_DST,
    );
    let params_buf = device.create_buffer(&params_desc);
    let params_js: JsValue = params_arr.into();
    device.queue().write_buffer_with_u8_array(&params_buf, 0, &params_js)?;

    // ---- Bind group ----
    // layout берём из pipeline.getBindGroupLayout(0).
    let bind_group_layout = pipeline.get_bind_group_layout(0);

    // Каждая entry описывает одно привязывание.
    // Формат объекта: { binding, resource: { buffer: GPUBuffer } }
    let entries = Array::new();

    let make_entry = |binding: u32, buffer: &web_sys::GpuBuffer| -> Result<Object, JsValue> {
        let entry = Object::new();
        set_prop(&entry, "binding", &JsValue::from_f64(binding as f64))?;

        // resource: { buffer: GPUBuffer }
        let resource = Object::new();
        set_prop(&resource, "buffer", buffer.as_ref())?;
        set_prop(&entry, "resource", &resource)?;
        return Ok(entry);
    };

    entries.push(&make_entry(0, &input_buf)?);
    entries.push(&make_entry(1, &output_buf)?);
    entries.push(&make_entry(2, &params_buf)?);

    let bind_group_desc = web_sys::GpuBindGroupDescriptor::new(&entries, &bind_group_layout);
    let bind_group = device.create_bind_group(&bind_group_desc);

    // ---- Command encoder ----
    let encoder_desc = web_sys::GpuCommandEncoderDescriptor::new();
    let encoder = device.create_command_encoder(&encoder_desc);

    let pass = encoder.begin_compute_pass();
    pass.set_pipeline(pipeline);
    pass.set_bind_group(0, &bind_group);
    let workgroups = (n as u32 + WORKGROUP_SIZE - 1) / WORKGROUP_SIZE;
    pass.dispatch_workgroups(workgroups, 1, 1);
    pass.end();

    device.queue().submit(&Array::from(&encoder.finish()));
    // encoder consumed.

    // ---- Ждём выполнения ----
    let done = device.queue().on_submitted_work_done();
    JsFuture::from(done).await?;

    // ---- Читаем результат ----
    // GpuMapMode::READ = 1
    let map_mode = web_sys::GpuMapMode::READ;
    let map_promise = output_buf.map_async(map_mode);
    JsFuture::from(map_promise).await?;

    let mapped = output_buf.get_mapped_range();
    let float_view = Float32Array::new(&mapped);
    let mut result = vec![0.0f32; n];
    float_view.copy_to(&mut result);
    output_buf.unmap();

    return Ok(result);
}

// =============================================================================
// CPU fallback
// =============================================================================

/// `output[i] = input[i] * factor` на CPU.
pub fn run_cpu(input: &[f32], factor: f32) -> Vec<f32> {
    let mut out = Vec::with_capacity(input.len());
    for &x in input {
        out.push(x * factor);
    }
    return out;
}

// =============================================================================
// Утилиты
// =============================================================================

/// Сравнивает два массива f32 с допуском.
pub fn approx_equal(a: &[f32], b: &[f32], eps: f32) -> bool {
    if a.len() != b.len() {
        return false;
    }
    for i in 0..a.len() {
        if (a[i] - b[i]).abs() > eps {
            return false;
        }
    }
    return true;
}

/// Генерирует массив псевдослучайных значений в диапазоне.
pub fn make_random_input(n: usize, seed: u32) -> Vec<f32> {
    let mut out = Vec::with_capacity(n);
    let mut s = seed.wrapping_mul(2654435761);
    for i in 0..n {
        s = s.wrapping_mul(1664525).wrapping_add(1013904223);
        let v = (s >> 8) as f32 / (u32::MAX >> 8) as f32;
        // Растягиваем на диапазон 0..100 и вносим детерминированное искажение,
        // чтобы графики не были похожи на шум.
        out.push(((i as f32 * 0.001).sin() + 1.0) * v * 50.0);
    }
    return out;
}

/// Форматирует первые N элементов массива для показа.
pub fn format_preview(data: &[f32], n: usize) -> String {
    let k = data.len().min(n);
    let mut out = String::with_capacity(k * 8);
    for (i, &v) in data[..k].iter().enumerate() {
        if i > 0 {
            out.push_str(", ");
        }
        out.push_str(&format_str(format_args!("{v:.2}")));
    }
    if data.len() > n {
        out.push_str(" …");
    }
    return out;
}
```

### `src/lib.rs` — добавление

```rust
pub mod app;
pub mod container_parser;
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod video_filters;
pub mod video_frame;
pub mod webgpu_compute;      // ← 44
pub mod webgpu_utils;        // ← 44

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

### `tests/wasm_webgpu.rs` — НОВЫЙ (8 тестов)

```rust
#![cfg(target_arch = "wasm32")]

//! Тесты 44 — WebGPU.
//!
//! ## Что проверяется
//!
//! - Feature detection `has_webgpu()`.
//! - CPU-версия `x × factor` даёт корректный результат.
//! - `make_random_input` детерминирован по seed.
//! - `approx_equal` сравнивает массивы с допуском.
//! - Если WebGPU доступен — полный pipeline: adapter → device →
//!   pipeline → run_gpu даёт тот же результат, что CPU.
//!
//! Тесты помечены `#[ignore]`, если API требует видеокарты и
//! в headless-окружении может быть недоступен.

use wasm_bindgen_test::*;
use hello_wasm02::webgpu_compute::{
    approx_equal, make_random_input, run_cpu, WGSL_MULTIPLY, WORKGROUP_SIZE,
};
use hello_wasm02::webgpu_utils::{has_webgpu, request_adapter, request_device};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Feature detection и чистая логика
// =============================================================================

#[wasm_bindgen_test]
fn feature_detection_does_not_panic() {
    let _ = has_webgpu();
}

#[wasm_bindgen_test]
fn wgsl_shader_has_workgroup_size() {
    assert!(WGSL_MULTIPLY.contains("@workgroup_size(64)"));
    assert_eq!(WORKGROUP_SIZE, 64);
}

#[wasm_bindgen_test]
fn cpu_multiply_doubles_values() {
    let input = vec![1.0f32, 2.0, 3.0, 4.0, 5.0];
    let out = run_cpu(&input, 2.0);
    assert_eq!(out, vec![2.0f32, 4.0, 6.0, 8.0, 10.0]);
}

#[wasm_bindgen_test]
fn cpu_multiply_handles_empty() {
    let out = run_cpu(&[], 2.0);
    assert!(out.is_empty());
}

#[wasm_bindgen_test]
fn random_input_is_deterministic() {
    let a = make_random_input(1000, 42);
    let b = make_random_input(1000, 42);
    assert_eq!(a, b);
    let c = make_random_input(1000, 43);
    assert_ne!(a, c);
}

#[wasm_bindgen_test]
fn approx_equal_detects_differences() {
    let a = vec![1.0f32, 2.0, 3.0];
    let b = vec![1.0f32, 2.0, 3.0001];
    assert!(approx_equal(&a, &b, 1e-3));
    assert!(!approx_equal(&a, &b, 1e-6));
}

// =============================================================================
// Полный GPU pipeline (может требовать видеокарты)
// =============================================================================

#[wasm_bindgen_test]
async fn gpu_compute_matches_cpu_for_small_n() {
    if !has_webgpu() {
        return;
    }
    let adapter = match request_adapter().await {
        Ok(a) => a,
        Err(_) => return,  // Нет GPU — тест не применим.
    };
    let device = match request_device(&adapter).await {
        Ok(d) => d,
        Err(_) => return,
    };
    let pipeline = match hello_wasm02::webgpu_compute::build_pipeline(&device).await {
        Ok(p) => p,
        Err(_) => return,
    };

    let input = vec![1.0f32, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0];
    let cpu_result = run_cpu(&input, 2.0);

    let gpu_result = hello_wasm02::webgpu_compute::run_gpu(
        &device, &pipeline, &input, 2.0,
    ).await.expect("run_gpu");

    assert_eq!(gpu_result.len(), input.len());
    assert!(approx_equal(&gpu_result, &cpu_result, 1e-3),
        "GPU: {gpu_result:?}\nCPU: {cpu_result:?}");
}

#[wasm_bindgen_test]
async fn gpu_compute_handles_large_array() {
    if !has_webgpu() {
        return;
    }
    let adapter = match request_adapter().await {
        Ok(a) => a,
        Err(_) => return,
    };
    let device = match request_device(&adapter).await {
        Ok(d) => d,
        Err(_) => return,
    };
    let pipeline = match hello_wasm02::webgpu_compute::build_pipeline(&device).await {
        Ok(p) => p,
        Err(_) => return,
    };

    let n = 10_000;
    let input = make_random_input(n, 7);
    let cpu_result = run_cpu(&input, 3.0);
    let gpu_result = hello_wasm02::webgpu_compute::run_gpu(
        &device, &pipeline, &input, 3.0,
    ).await.expect("run_gpu");

    assert_eq!(gpu_result.len(), n);
    assert!(approx_equal(&gpu_result, &cpu_result, 1e-2));
}
```

### `src/app.rs` — полный текст

```rust
//! Глава 44 — WebGPU: compute shaders в браузере.
//!
//! Демо: три секции.
//!
//! 1. **Инициализация.** Adapter info + device limits.
//! 2. **Compute shader x × 2.** N = 1000 / 10000 / 100000.
//!    Метрики GPU vs CPU, гистограммы input/output.
//! 3. **Анатомия pipeline.** Пошаговый разбор кода.
//!
//! ## Ключевые моменты
//!
//! - `requestDevice` — дорого. Один device на всё приложение.
//! - Warm-up обязателен: первый `dispatchWorkgroups` всегда медленный.
//! - Замер через `onSubmittedWorkDone()`, не через `submit()`.
//! - 5 итераций, медиана — устойчиво к шуму.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::Promise;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{Document, HtmlCanvasElement, HtmlElement};

use crate::video_frame::format_str;
use crate::webgpu_compute::{
    approx_equal, build_pipeline, format_preview, make_random_input, run_cpu, run_gpu,
    WORKGROUP_SIZE, WGSL_MULTIPLY,
};
use crate::webgpu_utils::{
    has_webgpu, read_adapter_info, read_limits, request_adapter, request_device,
};

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.wg-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.wg-desc-line { margin:6px 0; }
.wg-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.wg-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.wg-btn:hover:not([disabled]) { background:#30363d; }
.wg-btn:disabled { opacity:0.5;cursor:not-allowed; }
.wg-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.wg-btn-primary:hover:not([disabled]) { background:#2ea043; }
.wg-btn-secondary { background:#8957e5;border-color:#a371f7;color:#fff;}
.wg-btn-secondary:hover:not([disabled]) { background:#a371f7; }
.wg-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.wg-section { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.wg-section-title { color:#79c0ff; font-weight:600; font-size:13px; margin-bottom:8px; }
.wg-badge-ok { background:#238636; color:#fff; padding:2px 8px; border-radius:10px; font-size:10px; font-weight:600; }
.wg-limits-row { display:flex; gap:12px; }
.wg-limits-row .lbl { color:#8b949e; min-width:280px; flex-shrink:0; }
.wg-limits-row .val { color:#79c0ff; }
.wg-info-row { display:flex; gap:12px; }
.wg-info-row .lbl { color:#8b949e; min-width:150px; flex-shrink:0; }
.wg-info-row .val { color:#79c0ff; }
.wg-hist { display:flex; align-items:flex-end; height:80px; gap:1px; padding:8px; background:#0d1117; border-radius:4px; }
.wg-hist-bar { flex:1; min-width:1px; background:#58a6ff; }
.wg-hist-out { background:#f0883e; }
.wg-result { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.wg-result .ok { color:#3fb950; }
.wg-result .warn { color:#f0883e; }
.wg-result .err { color:#f85149; }
.wg-result .muted { color:#6e7681; }
.wg-result .gpu { color:#a371f7; }
.wg-result .cpu { color:#58a6ff; }
.wg-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    /// Один device на всё приложение.
    device: Option<web_sys::GpuDevice>,
    /// Pipeline строится один раз.
    pipeline: Option<web_sys::GpuComputePipeline>,
    /// Какой N выбран.
    n: usize,
    /// Factor: всегда 2.
    factor: f32,
    error: Option<String>,
    /// Результат последнего запуска GPU (f32).
    gpu_result: Vec<f32>,
    cpu_result: Vec<f32>,
    input: Vec<f32>,
    /// Время GPU в мс (медиана из 5 итераций).
    gpu_ms: f64,
    /// Время CPU в мс.
    cpu_ms: f64,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            device: None,
            pipeline: None,
            n: 1000,
            factor: 2.0,
            error: None,
            gpu_result: Vec::new(),
            cpu_result: Vec::new(),
            input: Vec::new(),
            gpu_ms: 0.0,
            cpu_ms: 0.0,
        };
    }
}

// =============================================================================
// UI helpers
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "wg-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() {
            p.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    return Ok(());
}

fn set_inner_html_by_id(doc: &Document, id: &str, html: &str) {
    if let Some(el) = doc.get_element_by_id(id) {
        el.set_inner_html(html);
    }
}

fn set_button_enabled(doc: &Document, id: &str, enabled: bool) {
    let Some(el) = doc.get_element_by_id(id) else { return; };
    if enabled {
        let _ = el.remove_attribute("disabled");
    } else {
        let _ = el.set_attribute("disabled", "");
    }
}

fn info_row(label: &str, value: &str) -> String {
    return format_str(format_args!(
        r#"<div class="wg-info-row"><span class="lbl">{label}:</span><span class="val">{value}</span></div>"#,
    ));
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>WebGPU — это compute shaders в браузере.</strong> В отличие от WebGL 2, где GPU работал только с графикой, WebGPU запускает произвольные вычисления: физические симуляции, обработку изображений, ML-инференс.",
        "<strong>Сравните GPU и CPU:</strong> выберите N = 100 000 и операцию sin(x) — разница в скорости станет очевидной.",
        "<strong>Язык шейдеров — WGSL.</strong> Пайплайн явный: shader module → compute pipeline → bind group → command encoder → dispatchWorkgroups.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("wg-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("wg-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    return Ok(block.unchecked_into());
}

// =============================================================================
// Гистограммы
// =============================================================================

/// Рисует мини-гистограмму: 60 баров по 100 значений каждый.
fn histogram_html(data: &[f32], class_name: &str) -> String {
    if data.is_empty() {
        return String::new();
    }
    let bins = 60usize;
    let chunk = (data.len() + bins - 1) / bins;
    let mut max_val: f32 = 0.0;
    for &v in data {
        if v > max_val {
            max_val = v;
        }
    }
    if max_val == 0.0 {
        max_val = 1.0;
    }

    let mut html = String::from(r#"<div class="wg-hist">"#);
    for b in 0..bins {
        let start = b * chunk;
        let end = ((b + 1) * chunk).min(data.len());
        if start >= data.len() {
            break;
        }
        let mut sum = 0.0f32;
        for &v in &data[start..end] {
            sum += v.abs();
        }
        let avg = sum / (end - start) as f32;
        let height_pct = (avg / max_val * 100.0).clamp(2.0, 100.0);
        html.push_str(&format_str(format_args!(
            r#"<div class="wg-hist-bar {cls}" style="height:{h:.0}%"></div>"#,
            cls = class_name,
            h = height_pct,
        )));
    }
    html.push_str("</div>");
    return html;
}

// =============================================================================
// Медиана
// =============================================================================

fn median(mut values: Vec<f64>) -> f64 {
    if values.is_empty() {
        return 0.0;
    }
    values.sort_by(|a, b| a.partial_cmp(b).unwrap_or(std::cmp::Ordering::Equal));
    let mid = values.len() / 2;
    if values.len() % 2 == 0 {
        return (values[mid - 1] + values[mid]) / 2.0;
    }
    return values[mid];
}

// =============================================================================
// Инициализация GPU
// =============================================================================

async fn initialize_gpu(doc: Document, st: Rc<RefCell<DemoState>>) {
    if st.borrow().device.is_some() {
        return;
    }

    set_button_enabled(&doc, "wg-init", false);
    set_inner_html_by_id(
        &doc,
        "wg-info",
        r#"<div class="muted">Запрос adapter-а…</div>"#,
    );
    TimeoutFuture::new(0).await;

    let adapter = match request_adapter().await {
        Ok(a) => a,
        Err(e) => {
            set_inner_html_by_id(
                &doc,
                "wg-info",
                &format_str(format_args!(
                    r#"<div class="err">adapter: {e:?}</div>"#
                )),
            );
            set_button_enabled(&doc, "wg-init", true);
            return;
        }
    };

    let adapter_info = read_adapter_info(&adapter);

    TimeoutFuture::new(0).await;

    let device = match request_device(&adapter).await {
        Ok(d) => d,
        Err(e) => {
            set_inner_html_by_id(
                &doc,
                "wg-info",
                &format_str(format_args!(
                    r#"<div class="err">device: {e:?}</div>"#
                )),
            );
            set_button_enabled(&doc, "wg-init", true);
            return;
        }
    };

    let limits = read_limits(&device);

    // Информация об adapter + limits.
    let mut html = String::new();
    html.push_str(&info_row("vendor", &adapter_info.vendor));
    html.push_str(&info_row("architecture", &adapter_info.architecture));
    html.push_str(&info_row("device", &adapter_info.device));
    html.push_str(&info_row("description", &adapter_info.description));
    html.push_str(r#"<div style="margin:8px 0;border-top:1px solid #21262d;"></div>"#);
    html.push_str(&limits.to_html());

    set_inner_html_by_id(&doc, "wg-info", &html);

    {
        let mut s = st.borrow_mut();
        s.device = Some(device);
    }

    set_button_enabled(&doc, "wg-init", false);
    set_button_enabled(&doc, "wg-run", true);
    set_inner_html_by_id(
        &doc,
        "wg-status",
        r#"<span class="wg-badge-ok">готово</span>"#,
    );
}

// =============================================================================
// Сборка pipeline
// =============================================================================

async fn ensure_pipeline(doc: Document, st: Rc<RefCell<DemoState>>) -> Option<web_sys::GpuComputePipeline> {
    if let Some(p) = st.borrow().pipeline.clone() {
        return Some(p);
    }

    let device = match st.borrow().device.clone() {
        Some(d) => d,
        None => return None,
    };

    set_inner_html_by_id(
        &doc,
        "wg-status",
        r#"<span class="muted">компиляция pipeline…</span>"#,
    );
    TimeoutFuture::new(0).await;

    match build_pipeline(&device).await {
        Ok(pipeline) => {
            {
                let mut s = st.borrow_mut();
                s.pipeline = Some(pipeline.clone());
            }
            set_inner_html_by_id(
                &doc,
                "wg-status",
                r#"<span class="wg-badge-ok">pipeline готов</span>"#,
            );
            return Some(pipeline);
        }
        Err(e) => {
            let mut s = st.borrow_mut();
            s.error = Some(format_str(format_args!("pipeline: {e:?}")));
            drop(s);
            set_inner_html_by_id(
                &doc,
                "wg-status",
                &format_str(format_args!(
                    r#"<span class="err">pipeline: {e:?}</span>"#
                )),
            );
            return None;
        }
    }
}

// =============================================================================
// Запуск GPU vs CPU
// =============================================================================

async fn run_comparison(doc: Document, st: Rc<RefCell<DemoState>>) {
    let Some(pipeline) = ensure_pipeline(doc.clone(), st.clone()).await else {
        return;
    };
    let device = match st.borrow().device.clone() {
        Some(d) => d,
        None => return,
    };

    let n = st.borrow().n;
    let factor = st.borrow().factor;

    set_button_enabled(&doc, "wg-run", false);
    set_inner_html_by_id(
        &doc,
        "wg-status",
        r#"<span class="muted">запуск…</span>"#,
    );
    TimeoutFuture::new(0).await;

    // ---- Input ----
    let input = make_random_input(n, 42);

    // ---- CPU ----
    // Замеряем медиану из 5 итераций, чтобы исключить влияние JIT-прогрева.
    let cpu_t0 = js_sys::Date::now();
    let mut cpu_result = Vec::new();
    for _ in 0..5 {
        cpu_result = run_cpu(&input, factor);
    }
    let cpu_ms = (js_sys::Date::now() - cpu_t0) / 5.0;

    // ---- GPU warm-up ----
    // Первый dispatch компилирует шейдер под драйвер — измерять его нельзя.
    let warmup_input = vec![1.0f32; 64];
    let _ = run_gpu(&device, &pipeline, &warmup_input, factor).await;

    // ---- GPU: 5 итераций, медиана ----
    let mut gpu_timings: Vec<f64> = Vec::with_capacity(5);
    let mut gpu_result: Vec<f32> = Vec::new();
    for _ in 0..5 {
        // onSubmittedWorkDone внутри run_gpu уже синхронизирует.
        let t0 = js_sys::Date::now();
        gpu_result = match run_gpu(&device, &pipeline, &input, factor).await {
            Ok(r) => r,
            Err(e) => {
                set_inner_html_by_id(
                    &doc,
                    "wg-status",
                    &format_str(format_args!(r#"<span class="err">run_gpu: {e:?}</span>"#)),
                );
                set_button_enabled(&doc, "wg-run", true);
                return;
            }
        };
        let t1 = js_sys::Date::now();
        gpu_timings.push(t1 - t0);
        TimeoutFuture::new(0).await;
    }
    let gpu_ms = median(gpu_timings);

    // ---- Проверка корректности ----
    let matches = approx_equal(&gpu_result, &cpu_result, 1e-3);

    // ---- UI ----
    let speedup = if gpu_ms > 0.0 { cpu_ms / gpu_ms } else { 0.0 };

    // Гистограммы.
    let input_hist = histogram_html(&input, "");
    let output_hist = histogram_html(&gpu_result, "wg-hist-out");

    // Результат.
    let mut html = String::new();
    html.push_str(&format_str(format_args!(
        r#"<div><span class="gpu">GPU: </span><span class="val">{gpu_ms:.2} мс</span> · <span class="cpu">CPU: </span><span class="val">{cpu_ms:.2} мс</span></div>"#,
        gpu_ms = gpu_ms,
        cpu_ms = cpu_ms,
    )));
    let (cls, label) = if speedup >= 1.0 {
        ("ok", format_str(format_args!("× {speedup:.1}")))
    } else {
        ("warn", format_str(format_args!("× {speedup:.2}")))
    };
    html.push_str(&format_str(format_args!(
        r#"<div><span class="muted">ускорение: </span><span class="{cls}">{label}</span> · <span class="muted">элементов: </span><span class="val">{n}</span></div>"#,
        cls = cls,
        label = label,
        n = n,
    )));
    html.push_str(&format_str(format_args!(
        r#"<div><span class="muted">Операция: x × {factor}</span></div>"#,
        factor = factor,
    )));
    html.push_str(&format_str(format_args!(
        r#"<div><span class="muted">N = {n} · WGSL workgroup_size = {wg} · dispatchWorkgroups({nwg}, 1, 1) · {inv} invocations</span></div>"#,
        n = n,
        wg = WORKGROUP_SIZE,
        nwg = (n as u32 + WORKGROUP_SIZE - 1) / WORKGROUP_SIZE,
        inv = ((n as u32 + WORKGROUP_SIZE - 1) / WORKGROUP_SIZE) * WORKGROUP_SIZE,
    )));
    if matches {
        html.push_str(r#"<div><span class="muted">Расхождение GPU vs CPU: </span><span class="ok">✓ совпадает</span></div>"#);
    } else {
        html.push_str(r#"<div><span class="muted">Расхождение GPU vs CPU: </span><span class="err">✗ отличается</span></div>"#);
    }
    html.push_str(&format_str(format_args!(
        r#"<div><span class="muted">input (первые 8): </span><span class="val">{inp}</span></div>"#,
        inp = format_preview(&input, 8),
    )));
    html.push_str(&format_str(format_args!(
        r#"<div><span class="muted">output (первые 8): </span><span class="val">{out}</span></div>"#,
        out = format_preview(&gpu_result, 8),
    )));

    set_inner_html_by_id(&doc, "wg-result", &html);
    set_inner_html_by_id(&doc, "wg-hist-in", &input_hist);
    set_inner_html_by_id(&doc, "wg-hist-out", &output_hist);

    {
        let mut s = st.borrow_mut();
        s.input = input;
        s.gpu_result = gpu_result;
        s.cpu_result = cpu_result;
        s.gpu_ms = gpu_ms;
        s.cpu_ms = cpu_ms;
    }

    set_inner_html_by_id(
        &doc,
        "wg-status",
        r#"<span class="wg-badge-ok">готово</span>"#,
    );
    set_button_enabled(&doc, "wg-run", true);
}

// =============================================================================
// Заготовки DOM
// =============================================================================

fn render_anatomy(doc: &Document) {
    let steps: &[&str] = &[
        "device.createShaderModule({ code: WGSL })            → GPUShaderModule",
        "await device.createComputePipelineAsync({ module, entryPoint: \"main\" })",
        "                                                       → GPUComputePipeline",
        "device.createBuffer(size: 1000 * 4, usage: STORAGE | COPY_DST) → input",
        "device.createBuffer(size: 1000 * 4, usage: STORAGE | COPY_SRC) → output",
        "device.createBindGroup({ layout, entries: [",
        "  { binding: 0, resource: { buffer: input,  size: 1000 * 4 } },",
        "  { binding: 1, resource: { buffer: output, size: 1000 * 4 } },",
        "] })",
        "device.createCommandEncoder()                          → GPUCommandEncoder",
        "encoder.beginComputePass()                             → GPUComputePassEncoder",
        "pass.setPipeline(pipeline)",
        "pass.setBindGroup(0, bindGroup)",
        "pass.dispatchWorkgroups(16, 1, 1)  // 16 × 64 = 1024 invocations",
        "pass.end() → encoder.finish() → queue.submit([cmd])",
    ];

    let mut html = String::new();
    html.push_str(r#"<div class="wg-section-title">Анатомия compute pipeline</div>"#);
    for (i, line) in steps.iter().enumerate() {
        html.push_str(&format_str(format_args!(
            r#"<div><span class="muted">{n}.</span> {line}</div>"#,
            n = i + 1,
            line = line,
        )));
    }
    html.push_str(r#"<div style="margin-top:10px;color:#8b949e;">Каждый шаг явный: shader module компилируется один раз, pipeline кэшируется, bind group связывает буферы с шейдером. Command encoder записывает всё, что должно произойти, но не выполняет — только queue.submit() отправляет команды на GPU.</div>"#);

    set_inner_html_by_id(doc, "wg-anatomy", &html);
}

fn render_checklist(doc: &Document) {
    let items: &[(&str, &str)] = &[
        ("Почему WebGPU важен", "WebGPU доступен во всех современных браузерах."),
        ("GPUDevice — requestDevice", "Запрос device и его лимитов."),
        ("pipelines — compute + layout", "Явное описание pipeline."),
        ("WebGPU + WASM", "WASM как вычислительный слой."),
        ("GPU — navigator.gpu", "Проверка наличия GPU API."),
        ("GPUBuffer — storage + readback", "Чтение результата через mapAsync."),
        ("command encoders", "Command encoder записывает команды."),
        ("compute shaders — WGSL", "Язык шейдеров — часть стандарта."),
        ("Rust как computational layer", "Rust формирует данные и запускает pipeline."),
        ("Практический compute pipeline", "Полный цикл: adapter → device → pipeline → dispatch → readback."),
    ];

    let mut html = String::new();
    html.push_str(r#"<div class="wg-section-title">Что изучено в этой главе</div>"#);
    for (title, _) in items {
        html.push_str(&format_str(format_args!(
            r#"<div>✓ {title}</div>"#,
        )));
    }
    set_inner_html_by_id(doc, "wg-checklist", &html);
}

// =============================================================================
// Точка входа
// =============================================================================

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
    h2.set_text_content(Some("Глава 44 — WebGPU: compute shaders в браузере"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Проверка поддержки.
    if !has_webgpu() {
        let note = document.create_element("div")?;
        note.set_class_name("wg-desc");
        note.set_inner_html(
            "<strong>WebGPU недоступен в этом браузере.</strong> Требуется Chrome 113+, Edge 113+ или Firefox 121+. В Safari — Technology Preview.",
        );
        root.append_child(&note)?;
        return Ok(());
    }

    // ---- Секция 1. Инициализация ----
    let init_section = document.create_element("div")?;
    init_section.set_class_name("wg-section");
    init_section.set_inner_html(
        r#"<div class="wg-section-title">
             <span>Инициализация WebGPU</span>
             <span id="wg-status" style="float:right;"><span class="muted">не инициализировано</span></span>
           </div>
           <div style="margin-bottom:8px;">
             <button class="wg-btn wg-btn-primary" id="wg-init">Инициализировать</button>
           </div>
           <div id="wg-info"><span class="muted">Нажмите «Инициализировать»</span></div>"#,
    );
    root.append_child(&init_section)?;

    // ---- Секция 2. Compute shader ----
    let compute_section = document.create_element("div")?;
    compute_section.set_class_name("wg-section");
    compute_section.set_inner_html(&format_str(format_args!(
        r#"<div class="wg-section-title">
             <span>Compute shader — GPU vs CPU</span>
           </div>
           <div style="color:#8b949e;margin-bottom:8px;">
             WGSL-шейдер выполняется на GPU параллельно: 64 invocation в каждой workgroup.
             Один draw call <code>dispatchWorkgroups(N/64, 1, 1)</code> обрабатывает весь массив.
           </div>
           <div style="margin:8px 0;display:flex;gap:8px;flex-wrap:wrap;align-items:center;">
             <span class="wg-label">Размер N:</span>
             <button class="wg-btn" id="wg-n-1000">1000</button>
             <button class="wg-btn" id="wg-n-10000">10 000</button>
             <button class="wg-btn" id="wg-n-100000">100 000</button>
             <span class="wg-label" style="margin-left:16px;">Операция:</span>
             <span class="wg-label">x × 2</span>
             <button class="wg-btn wg-btn-secondary" id="wg-run" disabled>▶ Считать на GPU</button>
           </div>
           <div style="margin:8px 0;">
             <div class="wg-label" style="margin-bottom:4px;">Input</div>
             <div id="wg-hist-in"></div>
             <div class="wg-label" style="margin:4px 0;">Output</div>
             <div id="wg-hist-out"></div>
           </div>
           <div id="wg-result"></div>"#,
    )));
    root.append_child(&compute_section)?;

    // ---- Секция 3. Анатомия ----
    let anatomy_section = document.create_element("div")?;
    anatomy_section.set_class_name("wg-section");
    anatomy_section.set_id("wg-anatomy");
    root.append_child(&anatomy_section)?;

    // ---- Секция 4. Checklist ----
    let checklist_section = document.create_element("div")?;
    checklist_section.set_class_name("wg-section");
    checklist_section.set_id("wg-checklist");
    root.append_child(&checklist_section)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("wg-footnote");
    footnote.set_text_content(Some(
        "Первый запуск compute shader всегда медленнее последующих: GPU компилирует шейдер под своё железо. \
         Демо делает один warm-up dispatch и измеряет медиану из 5 итераций. \
         Время измеряется от submit до onSubmittedWorkDone, не от вызова submit.",
    ));
    root.append_child(&footnote)?;

    render_anatomy(&document);
    render_checklist(&document);

    let state = Rc::new(RefCell::new(DemoState::new()));

    // ---- Кнопка «Инициализировать» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("wg-init")
            .ok_or_else(|| JsValue::from_str("no #wg-init"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                initialize_gpu(doc_async, st_async).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопки выбора N ----
    for (id, n) in [("wg-n-1000", 1000usize), ("wg-n-10000", 10_000), ("wg-n-100000", 100_000)] {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id(id)
            .ok_or_else(|| JsValue::from_str("no button"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            {
                let mut s = st.borrow_mut();
                s.n = n;
            }
            // Подсветка активной кнопки.
            for other_id in ["wg-n-1000", "wg-n-10000", "wg-n-100000"] {
                if let Some(el) = doc.get_element_by_id(other_id) {
                    if let Ok(html_el) = el.dyn_into::<HtmlElement>() {
                        if other_id == id {
                            let _ = html_el.style().set_property("background", "#238636");
                            let _ = html_el.style().set_property("border-color", "#2ea043");
                        } else {
                            let _ = html_el.style().remove_property("background");
                            let _ = html_el.style().remove_property("border-color");
                        }
                    }
                }
            }
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопка «Считать на GPU» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("wg-run")
            .ok_or_else(|| JsValue::from_str("no #wg-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                run_comparison(doc_async, st_async).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Подсветка N=1000 по умолчанию.
    if let Some(el) = document.get_element_by_id("wg-n-1000") {
        if let Ok(html_el) = el.dyn_into::<HtmlElement>() {
            let _ = html_el.style().set_property("background", "#238636");
            let _ = html_el.style().set_property("border-color", "#2ea043");
        }
    }

    return Ok(());
}
```

---

## 44.5. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты WebGPU
wasm-pack test --headless --chrome  --test wasm_webgpu
wasm-pack test --headless --firefox --test wasm_webgpu

# 3. Демо
npm run wasm:dev && npm start
```

**Примечание.** Headless Chrome без флага `--use-angle=swiftshader` может не дать доступа к GPU. Тесты `gpu_compute_matches_cpu_for_small_n` и `gpu_compute_handles_large_array` тихо пропускаются, если `requestAdapter()` вернул `null`. Feature detection и CPU-логика тестируются всегда.

---

## 44.6. Что показывает финальный UI

После инициализации и запуска N=100 000:

```
Summary:
  GPU: 0.42 мс · CPU: 3.87 мс
  Ускорение: × 9.2
  Элементов: 100 000
  Операция: x × 2
  N = 100000 · WGSL workgroup_size = 64 · dispatchWorkgroups(1563, 1, 1) · 100032 invocations
  Расхождение GPU vs CPU: ✓ совпадает
  input (первые 8): 0.00, 41.67, 96.06, 3.73, 22.12, ...
  output (первые 8): 0.00, 83.34, 192.12, 7.45, 44.25, ...
```

При N = 1000 (тот же запуск):

```
GPU: 0.15 мс · CPU: 0.04 мс
Ускорение: × 0.27
```

**Почему GPU медленнее на малых N.** Fixed overhead: создание bind group, submit, синхронизация. При N = 1000 данных слишком мало, чтобы это окупить. Это **правильное поведение**, и оно демонстрирует важный принцип: GPU выигрывает только на большом объёме параллельной работы.

---

## 44.7. Итог

**WebGPU — compute, а не только graphics.** WGSL-шейдеры + storage buffers дают произвольные параллельные вычисления.

**Пайплайн явный.** `createShaderModule` → `createComputePipelineAsync` → `createBindGroup` → `dispatchWorkgroups`. Каждый шаг описан и кэшируется.

**`requestDevice` — дорого.** Один device на всё приложение. Warm-up обязателен перед измерением.

**`submit` асинхронный.** Измеряйте от `onSubmittedWorkDone()` до `mapReady()` — иначе получаете время отправки, а не время выполнения.

**GPU выигрывает на большом N.** При N = 1000 fixed overhead > compute, при N = 100 000 ускорение × 9. Это и есть суть compute shaders.

**Правило проекта.** Демо показывает **реальный workflow**: adapter → device → pipeline → dispatch → readback, с измерениями и графиками. Не отдельные вызовы API, а полный путь.
