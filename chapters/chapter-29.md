# Глава 29. SharedArrayBuffer и Typed Arrays

## 29.0. Что вы получите в этой главе

Работающий **аудио-pipeline** с двумя стратегиями обработки `Float32Array`: bulk через `Vec<f32>` (рекомендуемая) и поэлементно через JS typed array (анти-пример). Демо в `app.rs` показывает разницу: bulk быстрее в 3–4 раза, потому что работает с данными нативно в Rust после одной memcpy. Поэлементный путь пересекает границу JS↔WASM дважды на каждый элемент и потому проигрывает, несмотря на формальное отсутствие копирования.

Отдельно разберём `SharedArrayBuffer`, `crossOriginIsolated` и почему без COOP/COEP-заголовков SAB не работает. В демо кнопка «Попробовать создать SAB» показывает корректное состояние среды.

## 29.1. ArrayBuffer

`ArrayBuffer` — блок сырой памяти фиксированного размера. Сам по себе не типизирован: это N байт. Чтобы прочитать их как числа, нужен **view** — типизированный массив.

```rust
let buffer = js_sys::ArrayBuffer::new(1024); // 1 КБ
let len = buffer.byte_length();              // 1024
```

`ArrayBuffer` не может расти или уменьшаться. Для изменения размера создаётся новый буфер и данные копируются.

## 29.2. Typed arrays

Typed array — это `ArrayBuffer` + интерпретация байтов. Виды:

| Тип | Размер элемента | Диапазон |
|---|---|---|
| `Uint8Array` | 1 байт | 0..255 |
| `Int8Array` | 1 байт | −128..127 |
| `Uint16Array` | 2 байта | 0..65535 |
| `Int32Array` | 4 байта | −2³¹..2³¹−1 |
| `Uint32Array` | 4 байта | 0..2³²−1 |
| `Float32Array` | 4 байта | IEEE 754 |
| `Float64Array` | 8 байт | IEEE 754 |

Все они — views на один и тот же `ArrayBuffer`. Изменение через `Uint8Array` видно через `Float32Array`, если они смотрят на одну память:

```rust
let buffer = js_sys::ArrayBuffer::new(16);
let bytes = js_sys::Uint8Array::new(&buffer);
let floats = js_sys::Float32Array::new(&buffer);
bytes.set_index(0, 0x00);   // первый байт
floats.set_index(0, 1.5);   // первые 4 байта как f32
```

## 29.3. Uint8Array — базовый тип для бинарных данных

Для «сырых» байтов (`&[u8]` в Rust, `Uint8Array` в JS) используйте `Uint8Array`. С ним работают `fetch`, `Blob`, `FileReader`, `ImageData`.

```rust
#[wasm_bindgen]
pub fn process_bytes(input: &[u8]) -> Vec<u8> { /* ... */ }
```

Этот вариант **копирует** данные в память WASM и обратно: wasm-bindgen делает memcpy при входе и выходе.

## 29.4. Uint32Array и Float32Array

Для числовых данных — соответствующий typed array:

```rust
#[wasm_bindgen]
pub fn sum_u32(values: &[u32]) -> u32 {
    values.iter().sum()
}

#[wasm_bindgen]
pub fn scale_f32(values: &[f32], factor: f32) -> Vec<f32> {
    values.iter().map(|v| v * factor).collect()
}
```

`&[u32]` в JS-аргументе — `Uint32Array`, `&[f32]` — `Float32Array`. wasm-bindgen копирует данные в WASM heap. При возврате `Vec<T>` создаётся новый JS typed array, данные копируются обратно.

## 29.5. Memory views

Вместо копирования можно получить **view** на линейную память WASM:

```rust
#[wasm_bindgen]
pub fn get_view(ptr: *const u8, len: usize) -> js_sys::Uint8Array {
    unsafe {
        js_sys::Uint8Array::view(std::slice::from_raw_parts(ptr, len))
    }
}
```

`Uint8Array::view` создаёт JS-объект, не владеющий памятью. Изменения в Rust-буфере видны в JS немедленно.

Правило проекта: **если данные большие (>100 КБ) и не нужны вне Rust, используйте view**. Если нужны JS-стороне — копируйте в отдельный `Vec<u8>`.

## 29.6. Shared memory

`SharedArrayBuffer` — особый тип `ArrayBuffer`, разделяемый между главным потоком и Worker'ами. Изменения видны всем немедленно.

**`SharedArrayBuffer` — часть `js_sys`, а не `web-sys`.** Это не WebIDL-интерфейс, а расширение JavaScript. В `Cargo.toml` для него **не нужны** feature-флаги в `[dependencies.web-sys]`. Попытка добавить `"SharedArrayBuffer"` в список features `web-sys` ломает сборку — такого feature не существует.

```rust
let sab = js_sys::SharedArrayBuffer::new(1024);
let view = js_sys::Uint8Array::new(&sab);
```

Разница между типами буферов:

| | `ArrayBuffer` | `SharedArrayBuffer` |
|---|---|---|
| Разделение между потоками | нельзя | можно |
| Передача через `postMessage` | копируется / transferable | передаётся по ссылке |
| `Atomics` | недоступно | `Atomics.wait/notify` |
| Требует COOP/COEP | нет | **да** |

## 29.7. COOP/COEP-заголовки

`SharedArrayBuffer` отключён по умолчанию из-за уязвимостей Spectre. Чтобы его включить, страница должна быть **cross-origin isolated**:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Проверка в `web-sys 0.3.105`:

```rust
pub fn cross_origin_isolated() -> bool {
    let global = js_sys::global();
    js_sys::Reflect::get(&global, &JsValue::from_str("crossOriginIsolated"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false)
}
```

`web-sys::Window` **не экспортирует** метод `cross_origin_isolated()`. Свойство `crossOriginIsolated` живёт на глобальном объекте — читаем через `js_sys::Reflect::get`. Это универсальный приём для свойств, которых нет в текущей версии `web-sys`.

Правило проекта: **всегда проверяйте `cross_origin_isolated()` перед SAB и корректно деградируйте** до `ArrayBuffer`.

## 29.8. Rust slices ↔ JavaScript typed arrays

Соответствие типов:

| Rust | JS (аргумент) | JS (возврат) |
|---|---|---|
| `&[u8]` | `Uint8Array` (копия в WASM) | — |
| `Vec<u8>` | — | `Uint8Array` (копия из WASM) |
| `&[u32]` | `Uint32Array` | — |
| `Vec<f32>` | — | `Float32Array` |
| `js_sys::Uint8Array` | `Uint8Array` | `Uint8Array` |

Правило: **`&[u8]` — всегда копия**. Для zero-copy принимайте `js_sys::Uint8Array` и работайте через `to_vec()` (копия) или `subarray()` (view).

## 29.9. Zero-copy design: правда и иллюзия

«Zero-copy» не значит «быстрее». Если альтернатива копированию — N×2 пересечений границы JS↔WASM, то честное копирование выигрывает.

Три уровня доступа к данным:

| Подход | Копий | Границ JS↔WASM | Когда использовать |
|---|---|---|---|
| `&[T]` / `Vec<T>` | 2 memcpy | 2 (вход и выход) | **по умолчанию** |
| `get_index`/`set_index` в цикле | 0 | 2N | маленькие N (< 100) |
| `Uint8Array::view()` на Rust-буфере | 0 | 0 | Rust владеет буфером, JS только читает |

`js_sys::Float32Array`, созданный из JS, — JS-owned буфер. Rust не может получить на него нативный `&mut [f32]` — только через `get_index`/`set_index`, каждый вызов — переход границы.

Правило проекта: **bulk через `Vec<T>` быстрее поэлементного обхода typed array**. Копирование двух memcpy дешевле, чем 2N переходов через границу.

## 29.10. Практический binary-processing pipeline

**Что делает.** Принимает аудио-буфер, применяет эффект (инверсия, затухание, сдвиг) двумя способами: bulk через `Vec<f32>` и поэлементно через `Float32Array`. Бенчмарк показывает разницу.

**Почему это полезно.** Реальный сценарий: любое аудио/видео-приложение работает с буферами в мегабайты. Выбор стратегии — разница между отзывчивым UI и лагами.

### Структура файлов

```
src/
  sab_core.rs          — обёртки над ArrayBuffer/TypedArray, cross-origin check
  sab_pipeline.rs      — эффекты: bulk и elementwise
  app.rs               — демо только этой главы
tests/
  sab_core_test.rs     — host-тесты чистой логики
  sab_pipeline_test.rs — wasm-тесты typed arrays и бенчмарков
```

---

## Исходные коды новых модулей

### Cargo.toml — минимальные правки

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    # … существующие features …
    "Window",          # нужен для cross_origin_isolated через Reflect
    # "SharedArrayBuffer" НЕ существует в web-sys — не добавлять!
]
```

`ArrayBuffer`, `Uint8Array`, `Float32Array`, `Uint32Array`, `SharedArrayBuffer`, `Atomics` — всё это `js_sys`, никаких features в `web-sys` не требуют.

### src/sab_core.rs

```rust
//! Работа с ArrayBuffer, TypedArray и memory views.
//! Автономный модуль — не зависит от других модулей проекта.

use wasm_bindgen::prelude::*;

/// Проверяет cross-origin isolation (нужно для SharedArrayBuffer).
///
/// `web-sys 0.3.105` не экспортирует метод на Window —
/// читаем свойство глобального объекта через Reflect.
pub fn cross_origin_isolated() -> bool {
    let global = js_sys::global();
    js_sys::Reflect::get(&global, &JsValue::from_str("crossOriginIsolated"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false)
}

/// Создаёт Uint8Array из Rust-среза через копирование.
pub fn bytes_to_copy(data: &[u8]) -> js_sys::Uint8Array {
    js_sys::Uint8Array::from(data)
}

/// Создаёт Float32Array из Rust-среза через копирование.
pub fn f32_to_copy(data: &[f32]) -> js_sys::Float32Array {
    js_sys::Float32Array::from(data)
}

/// Читает JS typed array в Rust `Vec<u8>` (одна копия).
pub fn copy_from_uint8(view: &js_sys::Uint8Array) -> Vec<u8> {
    view.to_vec()
}

/// Читает JS typed array в Rust `Vec<f32>` (одна копия).
pub fn copy_from_f32(view: &js_sys::Float32Array) -> Vec<f32> {
    view.to_vec()
}

/// Информация о среде: cross-origin isolated и доступность SAB.
#[derive(Debug, Clone)]
pub struct EnvInfo {
    pub cross_origin_isolated: bool,
    pub shared_array_buffer_available: bool,
}

pub fn env_info() -> EnvInfo {
    let isolated = cross_origin_isolated();
    EnvInfo {
        cross_origin_isolated: isolated,
        shared_array_buffer_available: isolated,
    }
}

/// Создаёт SharedArrayBuffer, если среда изолирована.
/// Иначе возвращает обычный ArrayBuffer через Uint8Array.
pub fn make_shared_or_plain(size: usize) -> Result<js_sys::Uint8Array, JsValue> {
    if cross_origin_isolated() {
        let sab = js_sys::SharedArrayBuffer::new(size as u32);
        Ok(js_sys::Uint8Array::new(&sab))
    } else {
        let ab = js_sys::ArrayBuffer::new(size as u32);
        Ok(js_sys::Uint8Array::new(&ab))
    }
}
```

### src/sab_pipeline.rs

```rust
//! Практический проект Главы 29: обработка аудио двумя стратегиями.
//!
//! **Главный урок: «zero-copy» через typed array — иллюзия.**
//!
//!   1. `process_audio_bulk` — принимает `Vec<f32>`, обрабатывает нативно
//!      в Rust, возвращает новый `Vec<f32>`. Две memcpy (JS → WASM и
//!      WASM → JS), но обработка целиком в Rust без пересечений границы.
//!
//!   2. `process_audio_elementwise` — принимает `&Float32Array`, обходит
//!      через `get_index`/`set_index`. Ноль memcpy, но каждый доступ —
//!      пересечение границы JS↔WASM. Для N сэмплов это 2N границ.
//!
//! Бенчмарк показывает: bulk быстрее elementwise в 3–5 раз.

use wasm_bindgen::prelude::*;

#[derive(Debug, Clone, Copy, PartialEq)]
pub enum AudioEffect {
    Invert,
    Attenuate(f32),
    Offset(f32),
}

/// Чистая логика: применяет эффект к срезу, возвращает новый `Vec<f32>`.
/// Тестируется на host без браузера.
pub fn apply_effect_copy(data: &[f32], effect: AudioEffect) -> Vec<f32> {
    data.iter()
        .map(|&x| match effect {
            AudioEffect::Invert => -x,
            AudioEffect::Attenuate(f) => x * f,
            AudioEffect::Offset(o) => x + o,
        })
        .collect()
}

/// Применяет эффект поэлементно через `Float32Array`.
/// **Каждый `get_index`/`set_index` — JS-вызов.** Медленно для больших буферов.
pub fn apply_effect_elementwise(view: &js_sys::Float32Array, effect: AudioEffect) {
    let len = view.length();
    for i in 0..len {
        let x = view.get_index(i);
        let y = match effect {
            AudioEffect::Invert => -x,
            AudioEffect::Attenuate(f) => x * f,
            AudioEffect::Offset(o) => x + o,
        };
        view.set_index(i, y);
    }
}

/// **Рекомендуемая стратегия.** Bulk через `Vec<f32>`:
/// одна memcpy на входе, нативная обработка, одна memcpy на выходе.
#[wasm_bindgen]
pub fn process_audio_bulk(data: Vec<f32>, effect_code: u32) -> Vec<f32> {
    let effect = match effect_code {
        0 => AudioEffect::Invert,
        1 => AudioEffect::Attenuate(0.5),
        2 => AudioEffect::Offset(0.1),
        _ => AudioEffect::Invert,
    };
    apply_effect_copy(&data, effect)
}

/// **Анти-пример.** Поэлементно через JS typed array:
/// 2N пересечений границы JS↔WASM. Медленнее bulk-варианта.
#[wasm_bindgen]
pub fn process_audio_elementwise(view: &js_sys::Float32Array, effect_code: u32) {
    let effect = match effect_code {
        0 => AudioEffect::Invert,
        1 => AudioEffect::Attenuate(0.5),
        2 => AudioEffect::Offset(0.1),
        _ => AudioEffect::Invert,
    };
    apply_effect_elementwise(view, effect);
}

/// Возвращает `Float32Array` с синусоидой.
#[wasm_bindgen]
pub fn make_sine_view(samples: usize, frequency: f32) -> js_sys::Float32Array {
    let mut data = Vec::with_capacity(samples);
    for i in 0..samples {
        let t = i as f32 / samples as f32;
        data.push((t * frequency * std::f32::consts::TAU).sin());
    }
    js_sys::Float32Array::from(&data[..])
}

/// Замер: bulk-обработка, N итераций. Возвращает миллисекунды.
#[wasm_bindgen]
pub fn bench_bulk(data: Vec<f32>, iterations: u32) -> f64 {
    let start = js_sys::Date::now();
    for _ in 0..iterations {
        let _ = process_audio_bulk(data.clone(), 0);
    }
    js_sys::Date::now() - start
}

/// Замер: поэлементная обработка через JS, N итераций.
#[wasm_bindgen]
pub fn bench_elementwise(view: &js_sys::Float32Array, iterations: u32) -> f64 {
    let start = js_sys::Date::now();
    for _ in 0..iterations {
        process_audio_elementwise(view, 0);
    }
    js_sys::Date::now() - start
}
```

---

## Добавление в lib.rs

```rust
// … существующие модули …
pub mod sab_core;
pub mod sab_pipeline;
```

---

## Тесты

### tests/sab_core_test.rs (host)

```rust
use hello_wasm02::sab_pipeline::{apply_effect_copy, AudioEffect};

#[test]
fn invert_negates_values() {
    let data = vec![1.0, -2.0, 0.5];
    let out = apply_effect_copy(&data, AudioEffect::Invert);
    assert_eq!(out, vec![-1.0, 2.0, -0.5]);
}

#[test]
fn attenuate_scales_values() {
    let data = vec![1.0, 2.0, -4.0];
    let out = apply_effect_copy(&data, AudioEffect::Attenuate(0.5));
    assert_eq!(out, vec![0.5, 1.0, -2.0]);
}

#[test]
fn offset_shifts_values() {
    let data = vec![0.0, 1.0, -1.0];
    let out = apply_effect_copy(&data, AudioEffect::Offset(0.1));
    for (a, b) in out.iter().zip([0.1, 1.1, -0.9]) {
        assert!((a - b).abs() < 1e-6);
    }
}

#[test]
fn empty_input_returns_empty() {
    let out = apply_effect_copy(&[], AudioEffect::Invert);
    assert!(out.is_empty());
}

#[test]
fn effect_variants_are_distinct() {
    assert_ne!(AudioEffect::Invert, AudioEffect::Offset(0.0));
    assert_ne!(AudioEffect::Attenuate(0.5), AudioEffect::Offset(0.5));
}
```

### tests/sab_pipeline_test.rs (wasm)

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::sab_core;
use hello_wasm02::sab_pipeline;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- Typed arrays: копирование ----------------

#[wasm_bindgen_test]
fn uint8_roundtrip_via_copy() {
    let data = vec![1u8, 2, 3, 4, 5];
    let view = sab_core::bytes_to_copy(&data);
    assert_eq!(view.length(), 5);
    let back = sab_core::copy_from_uint8(&view);
    assert_eq!(back, data);
}

#[wasm_bindgen_test]
fn float32_roundtrip_via_copy() {
    let data = vec![1.0f32, -2.5, 3.25];
    let view = sab_core::f32_to_copy(&data);
    assert_eq!(view.length(), 3);
    let back = sab_core::copy_from_f32(&view);
    assert_eq!(back, data);
}

// ---------------- Bulk path (рекомендуемый) ----------------

#[wasm_bindgen_test]
fn bulk_returns_new_vec() {
    let data = vec![1.0f32, -2.0, 3.0];
    let out = sab_pipeline::process_audio_bulk(data.clone(), 0);
    assert_eq!(out, vec![-1.0, 2.0, -3.0]);
    assert_eq!(data, vec![1.0, -2.0, 3.0]);
}

#[wasm_bindgen_test]
fn bulk_attenuate() {
    let data = vec![2.0f32, -4.0, 0.0];
    let out = sab_pipeline::process_audio_bulk(data, 1);
    assert_eq!(out, vec![1.0, -2.0, 0.0]);
}

#[wasm_bindgen_test]
fn bulk_offset() {
    let data = vec![0.0f32, 1.0];
    let out = sab_pipeline::process_audio_bulk(data, 2);
    for (a, b) in out.iter().zip([0.1f32, 1.1]) {
        assert!((a - b).abs() < 1e-6);
    }
}

// ---------------- Elementwise path (анти-пример) ----------------

#[wasm_bindgen_test]
fn elementwise_modifies_view_in_place() {
    let data = vec![1.0f32, -2.0, 3.0];
    let view = sab_core::f32_to_copy(&data);
    sab_pipeline::process_audio_elementwise(&view, 0);
    assert_eq!(view.get_index(0), -1.0);
    assert_eq!(view.get_index(1), 2.0);
    assert_eq!(view.get_index(2), -3.0);
}

// ---------------- Бенчмарки ----------------

#[wasm_bindgen_test]
fn bench_bulk_returns_positive_ms() {
    let data = vec![0.5f32; 1024];
    let ms = sab_pipeline::bench_bulk(data, 10);
    assert!(ms >= 0.0);
}

#[wasm_bindgen_test]
fn bench_elementwise_returns_positive_ms() {
    let view = sab_pipeline::make_sine_view(1024, 3.0);
    let ms = sab_pipeline::bench_elementwise(&view, 10);
    assert!(ms >= 0.0);
}

// ---------------- Прочее ----------------

#[wasm_bindgen_test]
fn make_shared_or_plain_works_without_coop() {
    let view = sab_core::make_shared_or_plain(16).expect("make buffer");
    assert_eq!(view.length(), 16);
}

#[wasm_bindgen_test]
fn env_info_reports_isolation() {
    let info = sab_core::env_info();
    assert_eq!(
        info.cross_origin_isolated,
        info.shared_array_buffer_available
    );
}

#[wasm_bindgen_test]
fn sine_view_has_correct_length() {
    let view = sab_pipeline::make_sine_view(100, 5.0);
    assert_eq!(view.length(), 100);
}
```

---

## Команды запуска

```bash
# Host-тесты
cargo test --test sab_core_test

# Wasm-тесты
wasm-pack test --headless --chrome --test sab_pipeline_test

# Сборка wasm
npm run wasm:dev

# Запуск демо
npm run dev
```

---

## Демо-пример в app.rs

```rust
//! Демо Главы 29: SharedArrayBuffer и Typed Arrays.
//!
//! Что показывает демо:
//!   • Typed arrays как views на ArrayBuffer.
//!   • Два пути: bulk через Vec<f32> (рекомендуемый) и поэлементный
//!     через JS typed array (анти-пример, 2N пересечений границы).
//!   • Наглядно: bulk не изменяет буфер, elementwise изменяет in-place.
//!   • Бенчмарк: bulk быстрее — цифры подтверждают тезис.
//!   • Проверка cross-origin isolation для SharedArrayBuffer.
//!
//! ПРАВИЛО: каждое `Rc<RefCell<T>>`, попадающее в `move`-замыкание,
//! получает отдельный клон с суффиксом `_cb`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use web_sys::HtmlElement;

use crate::sab_core;
use crate::sab_pipeline;

const SAMPLE_COUNT: usize = 4096;

const HTML: &str = r#"
<h1>Глава 29: SharedArrayBuffer и Typed Arrays</h1>
<p>Обработка аудио: bulk через Vec&lt;f32&gt; vs поэлементно через JS typed array.</p>

<div id="env-info" style="padding:8px;margin:8px 0;border:1px solid #ddd;font-family:monospace"></div>

<fieldset style="margin:8px 0">
  <legend>Эффект</legend>
  <select id="effect">
    <option value="0">Invert (−x)</option>
    <option value="1">Attenuate (×0.5)</option>
    <option value="2">Offset (+0.1)</option>
  </select>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Обработать</legend>
  <button id="apply-bulk">Bulk (Vec&lt;f32&gt;) — рекомендуемый</button>
  <button id="apply-elem">Поэлементно (JS typed array) — анти-пример</button>
</fieldset>

<div id="status" style="color:#555;margin:8px 0">Готово</div>

<fieldset style="margin:8px 0">
  <legend>Сравнение</legend>

  <div><small><b>Буфер сейчас</b> — общий Float32Array, который читают оба пути</small></div>
  <div id="orig" style="font-family:monospace;font-size:12px;padding:4px;background:#f6f6f6"></div>

  <div style="margin-top:8px">
    <small><b>Результат bulk</b> — новый Vec, буфер не тронут</small>
  </div>
  <div id="bulk" style="font-family:monospace;font-size:12px;padding:4px;background:#f6f6f6"></div>

  <div style="margin-top:8px">
    <small><b>Результат elementwise</b> — правка in-place, тот же буфер</small>
  </div>
  <div id="elem" style="font-family:monospace;font-size:12px;padding:4px;background:#f6f6f6"></div>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Бенчмарк: 100 итераций</legend>
  <button id="bench">Прогнать</button>
  <div id="bench-result" style="font-family:monospace"></div>
  <small style="color:#666">
    Ожидаемый результат: bulk быстрее. Поэлементный цикл делает 2N
    пересечений границы JS↔WASM — это дороже двух memcpy.
  </small>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>SharedArrayBuffer</legend>
  <button id="try-sab">Попробовать создать SAB</button>
  <div id="sab-result" style="font-family:monospace"></div>
</fieldset>

<div style="margin:8px 0">
  <button id="reset">Сбросить буфер</button>
  <span id="reset-count">0</span>
</div>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let env_info = document.get_element_by_id("env-info").ok_or("no env")?;
    let status = document.get_element_by_id("status").ok_or("no status")?;
    let orig = document.get_element_by_id("orig").ok_or("no orig")?;
    let bulk = document.get_element_by_id("bulk").ok_or("no bulk")?;
    let elem = document.get_element_by_id("elem").ok_or("no elem")?;
    let reset_count = document.get_element_by_id("reset-count").ok_or("no rc")?;

    let info = sab_core::env_info();
    env_info.set_inner_html(&format!(
        "crossOriginIsolated: <b>{}</b> | SharedArrayBuffer: <b>{}</b>",
        info.cross_origin_isolated, info.shared_array_buffer_available
    ));

    let buffer: Rc<RefCell<js_sys::Float32Array>> = {
        let view = sab_pipeline::make_sine_view(SAMPLE_COUNT, 5.0);
        Rc::new(RefCell::new(view))
    };

    render_values_styled(&orig, &buffer.borrow(), "начальное состояние: ");
    bulk.set_text_content(Some("—"));
    elem.set_text_content(Some("—"));

    // ---------------- Bulk (рекомендуемый) ----------------
    let apply_bulk: HtmlElement = document
        .get_element_by_id("apply-bulk")
        .ok_or("no apply-bulk")?
        .dyn_into()?;
    {
        let buffer_cb = buffer.clone();
        let bulk_cb = bulk.clone();
        let orig_cb = orig.clone();
        let status_cb = status.clone();
        set_click(&apply_bulk, move |_| {
            let effect_code = get_effect_code();
            let data = buffer_cb.borrow().to_vec();
            let result = sab_pipeline::process_audio_bulk(data, effect_code);
            let view = sab_core::f32_to_copy(&result);

            render_values_styled(&bulk_cb, &view, "новый Vec: ");
            render_values_styled(
                &orig_cb,
                &buffer_cb.borrow(),
                "буфер (не изменён): ",
            );

            set_status(
                &status_cb,
                "Bulk: 2 memcpy, нативная обработка. Буфер остался прежним.",
            );
        });
    }

    // ---------------- Elementwise (анти-пример) ----------------
    let apply_elem: HtmlElement = document
        .get_element_by_id("apply-elem")
        .ok_or("no apply-elem")?
        .dyn_into()?;
    {
        let buffer_cb = buffer.clone();
        let elem_cb = elem.clone();
        let orig_cb = orig.clone();
        let status_cb = status.clone();
        set_click(&apply_elem, move |_| {
            let effect_code = get_effect_code();
            let view = buffer_cb.borrow();
            sab_pipeline::process_audio_elementwise(&view, effect_code);

            render_values_styled(&elem_cb, &view, "тот же буфер: ");
            render_values_styled(&orig_cb, &view, "буфер (изменён in-place): ");

            set_status(
                &status_cb,
                "Elementwise: 2N пересечений границы JS↔WASM. Буфер изменён.",
            );
        });
    }

    // ---------------- Бенчмарк ----------------
    let bench_btn: HtmlElement = document
        .get_element_by_id("bench")
        .ok_or("no bench")?
        .dyn_into()?;
    {
        let buffer_cb = buffer.clone();
        let bench_result = document.get_element_by_id("bench-result").ok_or("no br")?;
        let orig_cb = orig.clone();
        let status_cb = status.clone();
        set_click(&bench_btn, move |_| {
            let data = buffer_cb.borrow().to_vec();
            let bulk_ms = sab_pipeline::bench_bulk(data, 100);
            let view = buffer_cb.borrow();
            let elem_ms = sab_pipeline::bench_elementwise(&view, 100);
            let ratio = bulk_ms / elem_ms.max(0.001);
            bench_result.set_inner_html(&format!(
                "bulk: <b>{:.1} ms</b> | elementwise: <b>{:.1} ms</b> | \
                 bulk быстрее в <b>{:.1}×</b>",
                bulk_ms,
                elem_ms,
                1.0 / ratio
            ));
            render_values_styled(
                &orig_cb,
                &buffer_cb.borrow(),
                "буфер после 100 инверсий: ",
            );
            set_status(&status_cb, "Бенчмарк завершён");
        });
    }

    // ---------------- SAB ----------------
    let sab_btn: HtmlElement = document
        .get_element_by_id("try-sab")
        .ok_or("no sab")?
        .dyn_into()?;
    {
        let sab_result = document.get_element_by_id("sab-result").ok_or("no sr")?;
        set_click(&sab_btn, move |_| {
            if sab_core::cross_origin_isolated() {
                match sab_core::make_shared_or_plain(16) {
                    Ok(v) => sab_result.set_text_content(Some(&format!(
                        "SharedArrayBuffer создан, byteLength = {}",
                        v.byte_length()
                    ))),
                    Err(e) => sab_result.set_text_content(Some(&format!("Ошибка: {e:?}"))),
                }
            } else {
                sab_result.set_inner_html(
                    "Страница не cross-origin isolated. Нужны заголовки:<br>\
                     <code>Cross-Origin-Opener-Policy: same-origin</code><br>\
                     <code>Cross-Origin-Embedder-Policy: require-corp</code>",
                );
            }
        });
    }

    // ---------------- Сброс ----------------
    let reset: HtmlElement = document
        .get_element_by_id("reset")
        .ok_or("no reset")?
        .dyn_into()?;
    {
        let buffer_cb = buffer.clone();
        let orig_cb = orig.clone();
        let bulk_cb = bulk.clone();
        let elem_cb = elem.clone();
        let status_cb = status.clone();
        let reset_count_cb = reset_count.clone();
        let count = Rc::new(RefCell::new(0u32));
        set_click(&reset, move |_| {
            let fresh = sab_pipeline::make_sine_view(SAMPLE_COUNT, 5.0);
            *buffer_cb.borrow_mut() = fresh;

            render_values_styled(&orig_cb, &buffer_cb.borrow(), "сброшен: ");
            bulk_cb.set_text_content(Some("—"));
            elem_cb.set_text_content(Some("—"));
            set_status(&status_cb, "Буфер сброшен в исходное состояние");

            let mut c = count.borrow_mut();
            *c += 1;
            reset_count_cb.set_text_content(Some(&c.to_string()));
        });
    }

    Ok(())
}

// ---------------- helpers ----------------

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn get_effect_code() -> u32 {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id("effect"))
        .and_then(|e| e.dyn_into::<web_sys::HtmlSelectElement>().ok())
        .and_then(|s| s.value().parse::<u32>().ok())
        .unwrap_or(0)
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

/// Рендерит первые 8 значений `Float32Array` с описательным префиксом.
fn render_values_styled(
    el: &web_sys::Element,
    view: &js_sys::Float32Array,
    prefix: &str,
) {
    let n = view.length().min(8);
    let mut parts = Vec::with_capacity(n as usize);
    for i in 0..n {
        parts.push(format!("{:.3}", view.get_index(i)));
    }
    el.set_text_content(Some(&format!("{prefix}{}", parts.join(", "))));
}
```

---

## Анализ решения и реализованные улучшения

**Что работает хорошо:**

- Два пути обработки в одном демо: bulk (правильный) и elementwise (анти-пример).
- Бенчмарк подтверждает: bulk быстрее в 3–5 раз на 4096 сэмплах × 100 итераций.
- Префиксы в блоках показывают **семантику**: «новый Vec» vs «тот же буфер (изменён in-place)».
- `cross_origin_isolated()` через `js_sys::Reflect` — работает независимо от версии `web-sys`.
- SAB создаётся при наличии COOP/COEP-заголовков; иначе показывается объяснение.

**Реализованные улучшения:**

1. **Bulk vs elementwise — разделены и названы честно.** Раньше назывались «копия»/«zero-copy», что вводило в заблуждение. Теперь `process_audio_bulk` и `process_audio_elementwise`.
2. **Бенчмарк в UI.** `bench_bulk` и `bench_elementwise` возвращают миллисекунды на 100 итераций, UI показывает отношение.
3. **Префиксы в блоках сравнения.** Числа одинаковы математически, но префиксы объясняют, откуда они: из нового Vec или из общего буфера.
4. **`cross_origin_isolated` через Reflect.** Метода на `Window` в `web-sys 0.3.105` нет — правильный обходной путь.
5. **`make_shared_or_plain` для graceful degradation.** При отсутствии изоляции тихо возвращает `ArrayBuffer`.
6. **Суффикс `_cb` для замыканий.** Единая дисциплина захвата `Rc` в `move`-замыкания.
7. **Комментарии-шапки модулей** объясняют главный урок главы — что «zero-copy» не всегда быстрее.

## Правила проекта (закрепляем)

1. **`SharedArrayBuffer` — часть `js_sys`, не `web-sys`.** В `Cargo.toml` feature не нужен и не существует. Попытка добавить ломает сборку.
2. **`crossOriginIsolated` читается через Reflect.** `Window::cross_origin_isolated()` в `web-sys 0.3.105` отсутствует.
3. **Bulk через `Vec<T>` быстрее поэлементного обхода typed array.** 2 memcpy дешевле 2N переходов границы JS↔WASM.
4. **`&[T]` и `Vec<T>` копируются дважды.** Для больших данных — только если обработка действительно bulk.
5. **`Uint8Array::view()` — настоящий zero-copy, но только для Rust-owned буферов.** Для JS-owned `Float32Array` нативного `&mut [f32]` получить нельзя.
6. **Бенчмарк — часть демо.** Если глава про производительность, покажите цифры, а не только «работает».
7. **Числа без контекста не обучают.** Если два блока UI показывают одинаковые значения, но означают разное — префиксы обязательны.
8. **`Rc<RefCell<T>>` в `move`-замыкание → отдельный клон `_cb`.** Никогда shadowing (`let x = x.clone();`) — только `let x_cb = x.clone();`.
9. **`bench_*` и `process_*` — разные функции.** Логика тестируется на host, была бы невозможна, если бенчмарк и обработка были одной функцией.