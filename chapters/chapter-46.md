# Глава 46. Где должен находиться Rust

## 46.0. Обновлённая структура

```
src/
  arch_pure.rs                    ← НОВЫЙ модуль 46 (чистые вычисления на Rust)
  arch_js_bench.rs                ← НОВЫЙ модуль 46 (JS-эквиваленты для сравнения)
  video_frame.rs                  ← из 23.1 (format_str)
  app.rs                          ← демо 46

tests/
  wasm_arch.rs                    ← тесты 46

index.html                        ← точка входа для браузера
Cargo.toml                        ← секция [package.metadata.wasm-pack...]
src/lib.rs                        ← +2 строки: arch_pure, arch_js_bench
```

**Комментарий.** Эта глава — архитектурная, а не про новый API. Мы показываем **границу** между Rust и JS и даём студенту инструмент для принятия решения: **что оставить в Rust, что отдать JavaScript**.

Модуль `arch_pure.rs` — чистые вычисления и обработка данных (алгоритмы, бинарные протоколы, строки, CRC). Модуль `arch_js_bench.rs` — JS-эквиваленты тех же задач, вызываемые через `js_sys::Function::new_with_args`, чтобы можно было сравнить **честно, на одних и тех же данных, на одной и той же машине**.

**Ключевой принцип главы.** Rust берёт на себя **всё, что связано с байтами, типами, алгоритмами и протоколами**. JS берёт на себя **всё, что связано с DOM, пользовательскими событиями и мостом к браузерным API**. Между ними — тонкая граница: `JsValue`-объекты, `Uint8Array`, строки. Каждый переход через границу стоит денег (копирование, сериализация), поэтому **границу надо пересекать крупными кусками**, а не по байту за раз.

**Префикс CSS** — `arch-` (architecture). Не пересекается с `bt-`, `sl-`, `usb-`, `wg-`, `vf-`.

**Ссылки для детального изучения:**

- Rust/WASM Book: [Design Patterns](https://rustwasm.github.io/book/game-of-life/design-patterns.html)
- `wasm-bindgen`: [Design](https://rustwasm.github.io/wasm-bindgen/contributing/design.html)
- Lin Clark: [JavaScript to Rust and back again](https://hacks.mozilla.org/2018/04/javascript-to-rust-and-back-again-a-wasm-bindgen-tale/)
- WebAssembly Design: [Use Cases](https://github.com/WebAssembly/design/blob/main/UseCases.md)
- `js_sys::Function::new_with_args`: [docs.rs](https://docs.rs/js-sys/latest/js_sys/struct.Function.html#method.new_with_args)
- `wasm-bindgen(inline_js)`: [docs.rs](https://docs.rs/wasm-bindgen/latest/wasm_bindgen/attr.wasm_bindgen.html)

---

## 46.1. О `Cargo.toml`

**Новых фич web-sys добавлять не нужно.** Достаточно того, что уже есть: `js-sys`, `wasm-bindgen`, `web-sys`, `gloo-timers`, `wasm-bindgen-futures`.

**Однако нужно добавить одну секцию.** `wasm-pack build --release` вызывает `wasm-opt` из `binaryen`. Если у вас установлена версия `binaryen` старше 110 (типичная ситуация для системных пакетов Ubuntu/Debian), она не знает про современные WASM-инструкции (`i32.trunc_sat_*` из nontrapping-float-to-int, `memory.copy`/`memory.fill` из bulk memory, `v128.*` из SIMD) и падает с сотнями `wasm-validator error`. Отключаем:

```toml
[package.metadata.wasm-pack.profile.release]
wasm-opt = false
```

**Почему это правильно.** `wasm-opt` оптимизирует размер, но не скорость CPU-кода. Для главы 46, где мы измеряем производительность, отключение `wasm-opt` не влияет на цифры. Если всё же хочется оптимизации по размеру — обновите `binaryen` до 123+ или уберите системный `wasm-opt` из `$PATH`, тогда `wasm-pack` сам скачает актуальную версию.

---

## 46.2. Модуль `src/arch_pure.rs`

Чистые вычисления. Пять задач-представителей: алгоритмы (Fibonacci), данные (сортировка), бинарная обработка (CRC-32), строки (UTF-8 code points), протоколы (парсинг кадров). Плюс детерминированные генераторы данных для бенчмарков.

```rust
//! 46. Архитектура: чистые вычисления, которые должны жить в Rust.
//!
//! ## Пять задач-представителей
//!
//! - **Алгоритмы** — Fibonacci (рекурсия).
//! - **Данные** — сортировка `u32`.
//! - **Бинарная обработка** — CRC-32.
//! - **Строки** — подсчёт code points UTF-8.
//! - **Протоколы** — парсинг кадров `55 AA len payload CRC16`.
//!
//! Все функции — чистые: на входе байты / числа / строки, на выходе
//! простые данные. Никакого DOM, никаких JsValue. Именно поэтому их
//! стоит держать в Rust — они не пересекают границу на каждый элемент.
//!
//! ## `black_box` не здесь
//!
//! `std::hint::black_box` вызывается на стороне `app.rs` — вокруг
//! входов и результатов бенчмарка. Сами функции должны оставаться
//! чистыми, без бенчмарочной обвязки.

// =============================================================================
// 1. Алгоритмы: Fibonacci
// =============================================================================

/// Рекурсивный Fibonacci. Медленный и предсказуемый для сравнения.
/// На n = 30 делает ~2.7M вызовов — на этом и строится бенчмарк.
pub fn fib_recursive(n: u32) -> u64 {
    if n < 2 {
        return n as u64;
    }
    return fib_recursive(n - 1) + fib_recursive(n - 2);
}

/// Итеративный Fibonacci — эталон для проверки рекурсивного.
pub fn fib_iterative(n: u32) -> u64 {
    let mut a: u64 = 0;
    let mut b: u64 = 1;
    for _ in 0..n {
        let next = a + b;
        a = b;
        b = next;
    }
    return a;
}

// =============================================================================
// 2. Данные: сортировка
// =============================================================================

/// In-place сортировка `u32`. `sort_unstable` — pattern-defeating
/// quicksort из std. Без компаратора-замыкания, чтобы JS-сравнение
/// не превращалось в сравнение «Rust quicksort vs V8 Array.sort».
pub fn sort_u32(data: &mut [u32]) {
    data.sort_unstable();
}

// =============================================================================
// 3. Бинарная обработка: CRC-32 (IEEE 802.3, полином 0xEDB88320)
// =============================================================================

pub fn crc32(data: &[u8]) -> u32 {
    let mut crc: u32 = 0xFFFF_FFFF;
    for &b in data {
        crc ^= b as u32;
        for _ in 0..8 {
            let mask = (crc & 1).wrapping_neg();
            crc = (crc >> 1) ^ (0xEDB8_8320 & mask);
        }
    }
    return !crc;
}

// =============================================================================
// 4. Строки: подсчёт code points
// =============================================================================

/// Число Unicode code points (не байт и не UTF-16 units).
pub fn count_code_points(text: &str) -> usize {
    return text.chars().count();
}

// =============================================================================
// 5. Протоколы: разбор кадра `55 AA len payload CRC16`
// =============================================================================

pub const FRAME_HEADER: [u8; 2] = [0x55, 0xAA];

pub struct Frame {
    pub payload: Vec<u8>,
    pub crc_ok: bool,
}

/// CRC-16/CCITT-FALSE, полином 0x1021, init 0xFFFF.
pub fn crc16_ccitt(data: &[u8]) -> u16 {
    let mut crc: u16 = 0xFFFF;
    for &b in data {
        crc ^= (b as u16) << 8;
        for _ in 0..8 {
            if (crc & 0x8000) != 0 {
                crc = (crc << 1) ^ 0x1021;
            } else {
                crc <<= 1;
            }
        }
    }
    return crc;
}

/// Собирает кадр: `55 AA len payload crc_lo crc_hi`.
pub fn build_frame(payload: &[u8]) -> Vec<u8> {
    let mut frame = Vec::with_capacity(payload.len() + 5);
    frame.push(FRAME_HEADER[0]);
    frame.push(FRAME_HEADER[1]);
    frame.push(payload.len() as u8);
    frame.extend_from_slice(payload);
    let crc = crc16_ccitt(payload);
    frame.push((crc & 0xFF) as u8);
    frame.push((crc >> 8) as u8);
    return frame;
}

/// Разбирает поток кадров с копированием payload. Используется
/// в тестах и в демонстрации структуры; в бенчмарке не участвует.
pub fn parse_frames(data: &[u8]) -> Vec<Frame> {
    let mut out = Vec::new();
    let mut i = 0;
    while i + 5 <= data.len() {
        if data[i] != FRAME_HEADER[0] || data[i + 1] != FRAME_HEADER[1] {
            break;
        }
        let len = data[i + 2] as usize;
        if i + 5 + len > data.len() {
            break;
        }
        let payload = data[i + 3..i + 3 + len].to_vec();
        let crc_expected = crc16_ccitt(&payload);
        let crc_actual =
            (data[i + 3 + len] as u16) | ((data[i + 4 + len] as u16) << 8);
        out.push(Frame {
            payload,
            crc_ok: crc_expected == crc_actual,
        });
        i += 5 + len;
    }
    return out;
}

/// Только счётчик валидных кадров — без копирования payload.
/// Семантика совпадает с JS-версией
/// в `arch_js_bench::compile_parse_frames`. Именно это используется
/// в бенчмарке — обе стороны делают одну и ту же работу (считают
/// CRC-16 и число кадров).
pub fn parse_frames_count(data: &[u8]) -> u32 {
    let mut count: u32 = 0;
    let mut i = 0;
    while i + 5 <= data.len() {
        if data[i] != FRAME_HEADER[0] || data[i + 1] != FRAME_HEADER[1] {
            break;
        }
        let len = data[i + 2] as usize;
        if i + 5 + len > data.len() {
            break;
        }
        let crc_expected = crc16_ccitt(&data[i + 3..i + 3 + len]);
        let crc_actual =
            (data[i + 3 + len] as u16) | ((data[i + 4 + len] as u16) << 8);
        if crc_expected == crc_actual {
            count += 1;
        }
        i += 5 + len;
    }
    return count;
}

// =============================================================================
// Генераторы данных
// =============================================================================

pub fn xorshift32(state: &mut u32) -> u32 {
    let mut x = *state;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    *state = x;
    return x;
}

pub fn gen_u32_array(n: usize, seed: u32) -> Vec<u32> {
    let mut state = seed;
    let mut out = Vec::with_capacity(n);
    for _ in 0..n {
        out.push(xorshift32(&mut state));
    }
    return out;
}

pub fn gen_bytes(n: usize, seed: u32) -> Vec<u8> {
    let mut state = seed;
    let mut out = Vec::with_capacity(n);
    for _ in 0..n {
        out.push((xorshift32(&mut state) & 0xFF) as u8);
    }
    return out;
}

pub fn gen_text(count: usize, seed: u32) -> String {
    let mut state = seed;
    let mut s = String::new();
    let alphabet: &[char] = &[
        'A', 'b', 'я', 'Ж', '☃', '∑', '中', '🦀',
    ];
    for _ in 0..count {
        let idx = (xorshift32(&mut state) as usize) % alphabet.len();
        s.push(alphabet[idx]);
    }
    return s;
}

pub fn simulate_stream(frame_count: usize, payload_size: usize, seed: u32) -> Vec<u8> {
    let mut state = seed;
    let mut out = Vec::with_capacity(frame_count * (payload_size + 5));
    for _ in 0..frame_count {
        let mut payload = Vec::with_capacity(payload_size);
        for _ in 0..payload_size {
            payload.push((xorshift32(&mut state) & 0xFF) as u8);
        }
        out.extend_from_slice(&build_frame(&payload));
    }
    return out;
}
```

---

## 46.3. Модуль `src/arch_js_bench.rs`

JS-эквиваленты задач. Ключевое правило: **одинаковый алгоритм с обеих сторон**. JS-сортировка — собственный quicksort, не `Array.sort` (встроенный вызывает C++ `rqsort` V8, и это не JS-бенчмарк). JS-парсер кадров считает CRC-16/CCITT, как Rust.

```rust
//! 46. Архитектура: JS-эквиваленты задач из `arch_pure.rs`.
//!
//! ## Принципы честного сравнения
//!
//! - **Одинаковый алгоритм.** JS-сортировка — собственный quicksort,
//!   а не `Array.prototype.sort`. Встроенный sort вызывает C++ код
//!   V8, и это не сравнение JS с Rust.
//! - **Одинаковая работа.** JS-парсер кадров считает CRC-16/CCITT
//!   так же, как Rust, а не просто ищет заголовки.
//! - **Прогрев.** Каждую JS-функцию вызываем несколько раз перед
//!   замером, чтобы V8 успел JIT-компилировать горячие пути.

use js_sys::{Function, JsString, Uint8Array, Uint32Array};
use wasm_bindgen::prelude::*;

// =============================================================================
// Компиляторы JS-функций
// =============================================================================

/// Компилирует рекурсивный Fibonacci.
///
/// `new Function("n", body)` создаёт анонимную функцию — `fib`
/// нужно объявить через function declaration внутри тела.
pub fn compile_fib() -> Function {
    return Function::new_with_args(
        "n",
        "function fib(x) { if (x < 2) return x; return fib(x - 1) + fib(x - 2); } return fib(n);",
    );
}

/// Компилирует quicksort для `Uint32Array`. Не использует
/// `Array.prototype.sort` — это важно для честного замера.
pub fn compile_sort_u32() -> Function {
    return Function::new_with_args(
        "arr",
        "function qs(a, lo, hi) {\
           if (lo >= hi) return;\
           const pivot = a[(lo + hi) >>> 1];\
           let i = lo, j = hi;\
           while (i <= j) {\
             while (a[i] < pivot) i++;\
             while (a[j] > pivot) j--;\
             if (i <= j) {\
               const t = a[i]; a[i] = a[j]; a[j] = t;\
               i++; j--;\
             }\
           }\
           qs(a, lo, j);\
           qs(a, i, hi);\
         }\
         qs(arr, 0, arr.length - 1);\
         return arr.length;",
    );
}

pub fn compile_crc32() -> Function {
    return Function::new_with_args(
        "bytes",
        "let crc = 0xFFFFFFFF;\
         for (let i = 0; i < bytes.length; i++) {\
           crc ^= bytes[i];\
           for (let j = 0; j < 8; j++) {\
             crc = (crc >>> 1) ^ (0xEDB88320 & -(crc & 1));\
           }\
         }\
         return (crc ^ 0xFFFFFFFF) >>> 0;",
    );
}

pub fn compile_count_code_points() -> Function {
    return Function::new_with_args(
        "text",
        "let n = 0; for (const _ of text) n++; return n;",
    );
}

/// Компилирует парсер кадров `55 AA len payload CRC16` с проверкой
/// CRC-16/CCITT-FALSE — ровно так же, как `parse_frames_count`
/// в `arch_pure.rs`. Без извлечения payload (нам нужен только
/// счётчик валидных кадров).
pub fn compile_parse_frames() -> Function {
    return Function::new_with_args(
        "bytes",
        "function crc16(arr, start, len) {\
           let crc = 0xFFFF;\
           for (let i = 0; i < len; i++) {\
             crc ^= arr[start + i] << 8;\
             for (let j = 0; j < 8; j++) {\
               if (crc & 0x8000) crc = ((crc << 1) ^ 0x1021) & 0xFFFF;\
               else crc = (crc << 1) & 0xFFFF;\
             }\
           }\
           return crc & 0xFFFF;\
         }\
         let count = 0;\
         let i = 0;\
         while (i + 5 <= bytes.length) {\
           if (bytes[i] !== 0x55 || bytes[i + 1] !== 0xAA) break;\
           const len = bytes[i + 2];\
           if (i + 5 + len > bytes.length) break;\
           const crcExpected = crc16(bytes, i + 3, len);\
           const crcActual = bytes[i + 3 + len] | (bytes[i + 4 + len] << 8);\
           if (crcExpected === crcActual) count++;\
           i += 5 + len;\
         }\
         return count;",
    );
}

// =============================================================================
// Типизированные обёртки для вызова
// =============================================================================

pub fn call_fib(f: &Function, n: u32) -> Result<u64, String> {
    let result = f
        .call1(&JsValue::NULL, &JsValue::from_f64(n as f64))
        .map_err(|e| crate::video_frame::format_str(format_args!("call_fib: {e:?}")))?;
    let val = result
        .as_f64()
        .ok_or_else(|| String::from("call_fib: результат не число"))?;
    return Ok(val as u64);
}

pub fn call_sort_u32(f: &Function, data: &[u32]) -> Result<Uint32Array, String> {
    let js_arr = Uint32Array::from(data);
    let _ = f
        .call1(&JsValue::NULL, js_arr.as_ref())
        .map_err(|e| {
            crate::video_frame::format_str(format_args!("call_sort_u32: {e:?}"))
        })?;
    return Ok(js_arr);
}

pub fn call_crc32(f: &Function, data: &[u8]) -> Result<u32, String> {
    let js_arr = Uint8Array::from(data);
    let arg: JsValue = js_arr.into();
    let result = f
        .call1(&JsValue::NULL, &arg)
        .map_err(|e| crate::video_frame::format_str(format_args!("call_crc32: {e:?}")))?;
    let val = result
        .as_f64()
        .ok_or_else(|| String::from("call_crc32: результат не число"))?;
    return Ok(val as u32);
}

pub fn call_count_code_points(f: &Function, text: &str) -> Result<u32, String> {
    let js_str = JsString::from(text);
    let arg: JsValue = js_str.into();
    let result = f
        .call1(&JsValue::NULL, &arg)
        .map_err(|e| {
            crate::video_frame::format_str(format_args!("call_count_code_points: {e:?}"))
        })?;
    let val = result
        .as_f64()
        .ok_or_else(|| String::from("call_count_code_points: результат не число"))?;
    return Ok(val as u32);
}

pub fn call_parse_frames(f: &Function, data: &[u8]) -> Result<u32, String> {
    let js_arr = Uint8Array::from(data);
    let arg: JsValue = js_arr.into();
    let result = f
        .call1(&JsValue::NULL, &arg)
        .map_err(|e| crate::video_frame::format_str(format_args!("call_parse_frames: {e:?}")))?;
    let val = result
        .as_f64()
        .ok_or_else(|| String::from("call_parse_frames: результат не число"))?;
    return Ok(val as u32);
}
```

---

## 46.4. Обновление `src/lib.rs`

```rust
pub mod app;
pub mod arch_js_bench;       // ← 46
pub mod arch_pure;           // ← 46
pub mod bluetooth_gatt;
pub mod bluetooth_utils;
pub mod container_parser;
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod serial_streams;
pub mod serial_utils;
pub mod usb_transfers;
pub mod usb_utils;
pub mod video_filters;
pub mod video_frame;
pub mod webgpu_compute;
pub mod webgpu_utils;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

---

## 46.5. Тесты `tests/wasm_arch.rs`

```rust
//! 46. Тесты архитектурных решений.
//!
//! ## Что тестируется
//!
//! - Чистые алгоритмы: `fib_recursive`, `fib_iterative`.
//! - Сортировка: порядок, длина, устойчивость к дубликатам.
//! - CRC-32 и CRC-16/CCITT-FALSE на эталонных векторах.
//! - UTF-8 code points: ASCII, кириллица, emoji.
//! - Протокол кадров: build/parse roundtrip, обрезанные данные.
//! - Генераторы данных: детерминированность.
//! - JS-обёртки: `compile_*` возвращает функцию, вызов даёт
//!   то же значение, что Rust-версия.

use wasm_bindgen_test::*;

use hello_wasm02::arch_js_bench::{
    call_count_code_points, call_crc32, call_fib, call_parse_frames,
    compile_count_code_points, compile_crc32, compile_fib, compile_parse_frames,
};
use hello_wasm02::arch_pure::{
    build_frame, count_code_points, crc16_ccitt, crc32, fib_iterative,
    fib_recursive, gen_bytes, gen_text, gen_u32_array, parse_frames,
    simulate_stream, sort_u32,
};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// 1. Fibonacci
// =============================================================================

#[wasm_bindgen_test]
fn test_fib_small() {
    assert_eq!(fib_recursive(0), 0);
    assert_eq!(fib_recursive(1), 1);
    assert_eq!(fib_recursive(2), 1);
    assert_eq!(fib_recursive(10), 55);
    assert_eq!(fib_recursive(20), 6765);
}

#[wasm_bindgen_test]
fn test_fib_recursive_matches_iterative() {
    for n in 0..25 {
        assert_eq!(
            fib_recursive(n),
            fib_iterative(n),
            "fib({n}) mismatch"
        );
    }
}

// =============================================================================
// 2. Сортировка
// =============================================================================

#[wasm_bindgen_test]
fn test_sort_basic() {
    let mut data = vec![5, 1, 4, 2, 8, 0, 3];
    sort_u32(&mut data);
    assert_eq!(data, vec![0, 1, 2, 3, 4, 5, 8]);
}

#[wasm_bindgen_test]
fn test_sort_with_duplicates() {
    let mut data = vec![3, 1, 3, 1, 3, 1];
    sort_u32(&mut data);
    assert_eq!(data, vec![1, 1, 1, 3, 3, 3]);
}

#[wasm_bindgen_test]
fn test_sort_empty() {
    let mut data: Vec<u32> = Vec::new();
    sort_u32(&mut data);
    assert!(data.is_empty());
}

#[wasm_bindgen_test]
fn test_sort_generated_data() {
    let mut data = gen_u32_array(1000, 0xDEAD_BEEF);
    sort_u32(&mut data);
    for i in 1..data.len() {
        assert!(data[i - 1] <= data[i], "not sorted at index {i}");
    }
}

// =============================================================================
// 3. CRC-32
// =============================================================================

#[wasm_bindgen_test]
fn test_crc32_empty() {
    assert_eq!(crc32(&[]), 0);
}

#[wasm_bindgen_test]
fn test_crc32_known_vector_quick_fox() {
    let data = b"The quick brown fox jumps over the lazy dog";
    assert_eq!(crc32(data), 0x414F_A339);
}

#[wasm_bindgen_test]
fn test_crc32_known_vector_123456789() {
    let data = b"123456789";
    assert_eq!(crc32(data), 0xCBF4_3926);
}

// =============================================================================
// 4. Code points
// =============================================================================

#[wasm_bindgen_test]
fn test_count_code_points_ascii() {
    assert_eq!(count_code_points("hello"), 5);
    assert_eq!(count_code_points(""), 0);
}

#[wasm_bindgen_test]
fn test_count_code_points_cyrillic() {
    assert_eq!(count_code_points("Привет"), 6);
}

#[wasm_bindgen_test]
fn test_count_code_points_emoji() {
    assert_eq!(count_code_points("🦀"), 1);
    assert_eq!(count_code_points("a🦀b"), 3);
}

// =============================================================================
// 5. CRC-16/CCITT-FALSE
// =============================================================================

#[wasm_bindgen_test]
fn test_crc16_known_vector() {
    let data = b"123456789";
    assert_eq!(crc16_ccitt(data), 0x29B1);
}

#[wasm_bindgen_test]
fn test_crc16_empty() {
    assert_eq!(crc16_ccitt(&[]), 0xFFFF);
}

// =============================================================================
// 6. Протокол кадров
// =============================================================================

#[wasm_bindgen_test]
fn test_build_frame_layout() {
    let frame = build_frame(b"hi");
    assert_eq!(frame.len(), 7);
    assert_eq!(frame[0], 0x55);
    assert_eq!(frame[1], 0xAA);
    assert_eq!(frame[2], 2);
    assert_eq!(frame[3], b'h');
    assert_eq!(frame[4], b'i');
}

#[wasm_bindgen_test]
fn test_parse_frames_roundtrip() {
    let stream = simulate_stream(10, 16, 42);
    let frames = parse_frames(&stream);
    assert_eq!(frames.len(), 10);
    for f in &frames {
        assert_eq!(f.payload.len(), 16);
        assert!(f.crc_ok, "CRC должен быть валиден в сгенерированном потоке");
    }
}

#[wasm_bindgen_test]
fn test_parse_frames_empty() {
    assert!(parse_frames(&[]).is_empty());
}

#[wasm_bindgen_test]
fn test_parse_frames_truncated() {
    let mut stream = simulate_stream(3, 8, 7);
    stream.truncate(stream.len() - 3);
    let frames = parse_frames(&stream);
    assert_eq!(frames.len(), 2);
}

#[wasm_bindgen_test]
fn test_parse_frames_bad_header() {
    let mut stream = simulate_stream(1, 4, 1);
    stream[0] = 0x00;
    let frames = parse_frames(&stream);
    assert!(frames.is_empty());
}

#[wasm_bindgen_test]
fn test_parse_frames_crc_mismatch() {
    let mut stream = simulate_stream(1, 8, 3);
    let n = stream.len();
    stream[n - 1] ^= 0xFF;
    let frames = parse_frames(&stream);
    assert_eq!(frames.len(), 1);
    assert!(!frames[0].crc_ok, "CRC должен не совпасть после порчи байта");
}

// =============================================================================
// 7. Генераторы данных
// =============================================================================

#[wasm_bindgen_test]
fn test_gen_bytes_deterministic() {
    let a = gen_bytes(100, 12345);
    let b = gen_bytes(100, 12345);
    assert_eq!(a, b);
}

#[wasm_bindgen_test]
fn test_gen_bytes_different_seeds() {
    let a = gen_bytes(100, 111);
    let b = gen_bytes(100, 222);
    assert_ne!(a, b);
}

#[wasm_bindgen_test]
fn test_gen_u32_array_deterministic() {
    let a = gen_u32_array(50, 999);
    let b = gen_u32_array(50, 999);
    assert_eq!(a, b);
}

#[wasm_bindgen_test]
fn test_gen_text_code_points() {
    let s = gen_text(1000, 7);
    assert_eq!(count_code_points(&s), 1000);
}

// =============================================================================
// 8. JS-обёртки совпадают с Rust-версиями
// =============================================================================

#[wasm_bindgen_test]
fn test_js_fib_matches_rust() {
    let f = compile_fib();
    for n in 0..25 {
        let js = call_fib(&f, n).expect("js fib failed");
        assert_eq!(js, fib_iterative(n), "fib({n}) mismatch");
    }
}

#[wasm_bindgen_test]
fn test_js_crc32_matches_rust() {
    let f = compile_crc32();
    let data = gen_bytes(1024, 42);
    let rust = crc32(&data);
    let js = call_crc32(&f, &data).expect("js crc32 failed");
    assert_eq!(js, rust);
}

#[wasm_bindgen_test]
fn test_js_count_code_points_matches_rust() {
    let f = compile_count_code_points();
    for text in ["hello", "Привет", "🦀", "a🦀b", ""] {
        let rust = count_code_points(text) as u32;
        let js = call_count_code_points(&f, text).expect("js count failed");
        assert_eq!(js, rust, "count mismatch for {text:?}");
    }
}

#[wasm_bindgen_test]
fn test_js_parse_frames_matches_rust() {
    let f = compile_parse_frames();
    let stream = simulate_stream(20, 16, 100);
    let rust_count = parse_frames(&stream).len() as u32;
    let js_count = call_parse_frames(&f, &stream).expect("js parse failed");
    assert_eq!(js_count, rust_count);
}
```

**Команды запуска тестов:**

```bash
# Все тесты проекта
wasm-pack test --headless --chrome

# Только тесты главы 46
wasm-pack test --headless --chrome --test wasm_arch

# С захватом вывода (при отладке падений)
wasm-pack test --headless --chrome --test wasm_arch -- --nocapture
```

---

## 46.6. Демо: `index.html`

`wasm-pack build --target web` генерирует только `.wasm` + JS-обёртку + `.d.ts`. HTML-страницы нет — её нужно создать вручную в корне проекта.

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Глава 46 — Где должен находиться Rust</title>
    <style>
        html, body {
            margin: 0;
            padding: 0;
            background: #0d1117;
            color: #c9d1d9;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            min-height: 100vh;
        }
        #app {
            max-width: 1200px;
            margin: 0 auto;
            padding: 24px 20px 60px 20px;
        }
        #loading {
            padding: 40px 20px;
            text-align: center;
            color: #8b949e;
            font-family: ui-monospace, monospace;
            font-size: 13px;
        }
    </style>
</head>
<body>
    <div id="app">
        <div id="loading">Загрузка WASM-модуля…</div>
    </div>

    <script type="module">
        import init, { run } from './pkg/hello_wasm02.js';

        try {
            await init();
            document.getElementById('loading')?.remove();
            run();
        } catch (err) {
            console.error('Ошибка инициализации WASM:', err);
            document.getElementById('app').innerHTML =
                '<pre style="color:#f85149;font-family:ui-monospace,monospace;font-size:12px;">' +
                'Ошибка загрузки: ' + String(err) + '</pre>';
        }
    </script>
</body>
</html>
```

**Что важно:**

- `<script type="module">` — вывод `wasm-pack --target web` это ES-модуль.
- Путь `./pkg/hello_wasm02.js` работает только если HTML лежит рядом с папкой `pkg/`.
- `await init()` — инициализирует WASM. Без него `run()` упадёт.
- `try/catch` — если модуль не загрузится, ошибка появится на странице.

**Команды запуска демо:**

```bash
# Сборка в release (для честных цифр)
wasm-pack build --release --target web

# Запуск из корня проекта (не из pkg/)
python3 -m http.server 5173
# Откройте http://127.0.0.1:5173/

# Альтернатива: npx serve (корректно отдаёт MIME application/wasm)
npx serve .
```

---

## 46.7. Демо: `src/app.rs`

Это ядро главы. Здесь собраны все уроки: двойной барьер против JIT (LLVM + V8), data chaining, геометрическое среднее, sanity check.

```rust
//! Глава 46 — Где должен находиться Rust: живое сравнение Rust/WASM и JS.
//!
//! ## Честность замера
//!
//! - **Прогрев + калибровка**: подбираем `iters` так, чтобы суммарное
//!   время было ≥ `BENCH_TARGET_MS`.
//! - **Медиана из 5 прогонов**: устойчиво к GC-pause и JIT-deopt.
//! - **Двойной барьер против JIT**:
//!   - `opaque()` — Rust-side. `AtomicU32::store` + `fence(SeqCst)` +
//!     `AtomicU32::load` в `#[inline(never)]`. LLVM не может удалить
//!     атомики и не может вынести зависящее от них вычисление из цикла.
//!   - `__arch_chain()` — V8-side. `inline_js`-функция, вызываемая
//!     через границу WASM↔JS. TurboFan не инлайнит через границу и
//!     не может удалить вызов как dead code.
//! - **Sanity check**: если per-iteration время < 10 мкс — карточка
//!   помечается как подозрительная.

use std::cell::RefCell;
use core::sync::atomic::{fence, AtomicU32, Ordering};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::Document;

use crate::arch_js_bench::{
    call_count_code_points, call_crc32, call_fib, call_parse_frames,
    compile_count_code_points, compile_crc32, compile_fib, compile_parse_frames,
    compile_sort_u32,
};
use crate::arch_pure::{
    count_code_points, crc32, fib_recursive, gen_bytes, gen_text,
    gen_u32_array, parse_frames_count, simulate_stream, sort_u32,
};
use crate::video_frame::format_str;

// =============================================================================
// Rust-барьер против LLVM
// =============================================================================

/// LLVM-барьер. Реальная atomic-операция — LLVM не может удалить её
/// как dead store: атомики часть контракта LLVM IR и не устраняются
/// оптимизатором. `#[inline(never)]` защищает от развёртывания вызова,
/// а значит зависящее от `v` вычисление не может быть вынесено из цикла
/// (иначе аргумент станет константой — но он зависит от `SLOT.load`,
/// а load идёт после fence).
///
/// ## Зачем нужен отдельный Rust-барьер, если есть `__arch_chain`
///
/// `__arch_chain` объявлена как `extern "C"` через `inline_js`. Для
/// LLVM её тело неизвестно, поэтому он **формально мог бы** считать
/// её `readnone` и вынести вычисление аргумента из цикла — что и
/// происходило: `fib_recursive(30)` вычислялся один раз, а в цикле
/// крутился только shim. `opaque` решает эту проблему на уровне LLVM.
#[inline(never)]
fn opaque(v: u32) -> u32 {
    static SLOT: AtomicU32 = AtomicU32::new(0);
    SLOT.store(v, Ordering::SeqCst);
    fence(Ordering::SeqCst);
    return SLOT.load(Ordering::SeqCst);
}

// =============================================================================
// V8-барьер против TurboFan
// =============================================================================

/// JS-функция, объявленная через `inline_js`. Читает
/// `window.__arch_chain_state`, XOR-ит с аргументом, записывает обратно,
/// возвращает новое значение. Side effect: мутация свойства `window` —
/// наблюдаемое состояние, V8 не может удалить как dead store.
///
/// `inline_js` создаёт вызов **через границу** WASM↔JS. TurboFan не
/// инлайнит через эту границу и не может доказать отсутствие side
/// effects у чужой JS-функции. Это принципиально отличает `__arch_chain`
/// от `Reflect::set` — семантику последнего TurboFan знает.
#[wasm_bindgen(inline_js = "
export function __arch_chain(v) {
    window.__arch_chain_state = ((window.__arch_chain_state || 0) ^ v) >>> 0;
    return window.__arch_chain_state;
}
")]
extern "C" {
    fn __arch_chain(v: u32) -> u32;
}

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.arch-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.arch-desc-line { margin:6px 0; }
.arch-note-critical { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#3a1010;border-left:4px solid #f85149;border-radius:4px;color:#ffa198;font-size:13px;line-height:1.6;}
.arch-note-critical code { background:#1a0505; color:#ffd0d0; padding:1px 5px; border-radius:3px; }
.arch-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.arch-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.arch-btn:hover:not([disabled]) { background:#30363d; }
.arch-btn:disabled { opacity:0.5;cursor:not-allowed; }
.arch-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.arch-btn-primary:hover:not([disabled]) { background:#2ea043; }
.arch-hint { color:#8b949e;font-size:12px; }

.arch-cards { max-width:1000px;margin-bottom:16px;display:grid;grid-template-columns:repeat(auto-fit,minmax(340px,1fr));gap:12px;}
.arch-card { padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;transition:border-color 0.15s;}
.arch-card.arch-win { border-color:#2ea043; }
.arch-card.arch-close { border-color:#a371f7; }
.arch-card.arch-loss { border-color:#f0883e; }
.arch-card.arch-suspect { border-color:#f85149; background:#1a0d0d; }
.arch-card.arch-running { border-color:#58a6ff; box-shadow:0 0 0 1px #58a6ff33; }
.arch-card-title { color:#79c0ff;font-weight:600;font-size:14px;margin-bottom:2px; }
.arch-card-sub { color:#8b949e;font-size:11px;margin-bottom:10px;text-transform:uppercase;letter-spacing:1px; }
.arch-card-timings { font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;color:#c9d1d9; }
.arch-card-timings .lbl { color:#8b949e; }
.arch-card-timings .rust { color:#3fb950;font-weight:600; }
.arch-card-timings .js { color:#f0883e;font-weight:600; }
.arch-card-out { font-family:ui-monospace,monospace;font-size:11px;color:#8b949e;margin-top:6px;overflow-wrap:anywhere; }
.arch-card-ratio { margin-top:8px;font-family:ui-monospace,monospace;font-size:13px;font-weight:600; }
.arch-card-ratio.arch-win { color:#3fb950; }
.arch-card-ratio.arch-close { color:#a371f7; }
.arch-card-ratio.arch-loss { color:#f0883e; }
.arch-card-suspect-msg { color:#f85149;font-size:11px;margin-top:6px; }
.arch-card-btn { margin-top:10px; }

.arch-summary { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border-left:3px solid #79c0ff;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:13px;line-height:1.8;}
.arch-summary .lbl { color:#8b949e; }
.arch-summary .val { color:#79c0ff; }
.arch-summary .rust { color:#3fb950; font-weight:600; }
.arch-summary .js { color:#f0883e; font-weight:600; }
.arch-summary .close { color:#a371f7; font-weight:600; }

.arch-log { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;line-height:1.6;color:#c9d1d9;max-height:220px;overflow-y:auto;}
.arch-log-entry { display:flex; gap:8px; padding:1px 0; }
.arch-log-entry .time { color:#6e7681; flex-shrink:0; }
.arch-log-entry .msg { color:#c9d1d9; overflow-wrap:anywhere; }
.arch-log-err .msg { color:#f85149; }

.arch-framework { max-width:1000px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-size:13px;line-height:1.65;}
.arch-framework h3 { margin:0 0 10px 0;color:#79c0ff;font-size:14px;font-weight:600; }
.arch-framework table { width:100%; border-collapse:collapse; }
.arch-framework th { text-align:left; padding:6px 8px; color:#8b949e; border-bottom:1px solid #21262d; font-size:11px; text-transform:uppercase; letter-spacing:1px; }
.arch-framework td { padding:6px 8px; border-bottom:1px solid #161b22; vertical-align:top; }
.arch-framework td.usb-rust { color:#3fb950; font-family:ui-monospace,monospace; }
.arch-framework td.usb-js { color:#f0883e; font-family:ui-monospace,monospace; }
.arch-framework td.usb-both { color:#a371f7; font-family:ui-monospace,monospace; }
"#;

// =============================================================================
// Определения задач
// =============================================================================

struct TaskDef {
    id: &'static str,
    title: &'static str,
    subtitle: &'static str,
}

const TASKS: &[TaskDef] = &[
    TaskDef { id: "fib",    title: "Fibonacci(30)",             subtitle: "алгоритмы" },
    TaskDef { id: "sort",   title: "Сортировка 100k",           subtitle: "данные" },
    TaskDef { id: "crc32",  title: "CRC-32 · 1 МБ",             subtitle: "бинарная обработка" },
    TaskDef { id: "utf8",   title: "UTF-8 · 100k code points",  subtitle: "строки" },
    TaskDef { id: "frames", title: "Парсинг 10k кадров",        subtitle: "протоколы" },
];

// =============================================================================
// Модели
// =============================================================================

struct TaskResult {
    rust_ms: f64,
    js_ms: f64,
    rust_iters: u32,
    js_iters: u32,
    out: String,
    err: Option<String>,
    suspect: bool,
}

impl TaskResult {
    fn speedup(&self) -> f64 {
        if self.js_ms <= 0.0 {
            return 0.0;
        }
        return self.js_ms / self.rust_ms.max(1e-9);
    }
}

struct LogEntry {
    time: String,
    text: String,
    is_err: bool,
}

struct AppState {
    results: Vec<Option<TaskResult>>,
    logs: Vec<LogEntry>,
    running: bool,
}

impl AppState {
    fn new() -> Self {
        let mut results: Vec<Option<TaskResult>> = Vec::with_capacity(TASKS.len());
        for _ in TASKS {
            results.push(None);
        }
        return Self {
            results,
            logs: Vec::new(),
            running: false,
        };
    }
}

// =============================================================================
// Хелперы DOM
// =============================================================================

fn as_node(el: &web_sys::Element) -> &web_sys::Node {
    return el.unchecked_ref::<web_sys::Node>();
}

// =============================================================================
// Бенчмарк
// =============================================================================

const BENCH_TARGET_MS: f64 = 100.0;
const BENCH_RUNS: u32 = 5;
const BENCH_WARMUP: u32 = 3;
const BENCH_MAX_ITERS: u32 = 1_000_000;

/// Порог «замер недостоверен»: 10 мкс. Если per-iteration время меньше,
/// значит либо debug, либо JIT всё-таки выбросил тело цикла.
const SUSPECT_THRESHOLD_MS: f64 = 0.01;

fn measure_total<F: FnMut()>(f: &mut F, iters: u32) -> f64 {
    let t0 = perf_now();
    for _ in 0..iters {
        f();
    }
    return perf_now() - t0;
}

fn adaptive_measure<F: FnMut()>(f: &mut F, target_ms: f64) -> (f64, u32) {
    let mut iters: u32 = 1;
    loop {
        let total = measure_total(f, iters);
        if total >= target_ms || iters >= BENCH_MAX_ITERS {
            return (total, iters);
        }
        let factor = if total < 0.1 {
            100.0
        } else if total < target_ms / 10.0 {
            10.0
        } else {
            2.0
        };
        let next = ((iters as f64) * factor).min(BENCH_MAX_ITERS as f64) as u32;
        iters = next.max(iters + 1);
    }
}

fn median_measure<F: FnMut()>(f: &mut F, iters: u32, runs: u32) -> f64 {
    let mut samples: Vec<f64> = Vec::with_capacity(runs as usize);
    for _ in 0..runs {
        let total = measure_total(f, iters);
        samples.push(total / iters as f64);
    }
    samples.sort_by(|a, b| a.partial_cmp(b).unwrap_or(std::cmp::Ordering::Equal));
    return samples[samples.len() / 2];
}

fn bench_pair<F1, F2>(mut rust_fn: F1, mut js_fn: F2) -> (f64, f64, u32, u32)
where
    F1: FnMut(),
    F2: FnMut(),
{
    for _ in 0..BENCH_WARMUP {
        rust_fn();
        js_fn();
    }

    let (_rust_total, rust_iters) = adaptive_measure(&mut rust_fn, BENCH_TARGET_MS);
    let (_js_total, js_iters) = adaptive_measure(&mut js_fn, BENCH_TARGET_MS);

    let rust_ms = median_measure(&mut rust_fn, rust_iters, BENCH_RUNS);
    let js_ms = median_measure(&mut js_fn, js_iters, BENCH_RUNS);

    return (rust_ms, js_ms, rust_iters, js_iters);
}

// =============================================================================
// Утилиты времени и форматирования
// =============================================================================

fn perf_now() -> f64 {
    if let Some(w) = web_sys::window() {
        if let Some(p) = w.performance() {
            return p.now();
        }
    }
    return js_sys::Date::now();
}

fn fmt_ms(ms: f64) -> String {
    if ms < 0.001 {
        return format_str(format_args!("{:.0} нс", ms * 1_000_000.0));
    }
    if ms < 1.0 {
        return format_str(format_args!("{:.1} мкс", ms * 1000.0));
    }
    if ms < 100.0 {
        return format_str(format_args!("{:.2} мс", ms));
    }
    return format_str(format_args!("{:.1} мс", ms));
}

fn fmt_speedup(s: f64) -> String {
    if s >= 100.0 {
        return format_str(format_args!("{:.0}×", s));
    }
    if s >= 10.0 {
        return format_str(format_args!("{:.1}×", s));
    }
    return format_str(format_args!("{:.2}×", s));
}

fn fmt_iters(n: u32) -> String {
    if n <= 1 {
        return String::new();
    }
    return format_str(format_args!(" ×{n}"));
}

fn escape_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;");
}

// =============================================================================
// UI-утилиты
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "arch-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() {
            p.remove_child(as_node(&old))?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(as_node(&style))?;
    return Ok(());
}

fn now_timestamp() -> String {
    let d = js_sys::Date::new_0();
    let h = d.get_hours();
    let m = d.get_minutes();
    let s = d.get_seconds();
    let ms = d.get_milliseconds();
    return format_str(format_args!("{h:02}:{m:02}:{s:02}.{ms:03}"));
}

fn push_log(st: &Rc<RefCell<AppState>>, text: String, is_err: bool) {
    let mut s = st.borrow_mut();
    s.logs.push(LogEntry {
        time: now_timestamp(),
        text,
        is_err,
    });
    if s.logs.len() > 100 {
        let drop = s.logs.len() - 100;
        s.logs.drain(0..drop);
    }
}

fn render_log_html(logs: &[LogEntry]) -> String {
    let mut html = String::with_capacity(logs.len() * 100);
    for e in logs {
        html.push_str(r#"<div class="arch-log-entry"#);
        if e.is_err {
            html.push_str(" arch-log-err");
        }
        html.push_str(r#"">"#);
        html.push_str(r#"<span class="time">"#);
        html.push_str(&e.time);
        html.push_str(r#"</span>"#);
        html.push_str(r#"<span class="msg">"#);
        html.push_str(&escape_html(&e.text));
        html.push_str("</span></div>");
    }
    return html;
}

// =============================================================================
// Разметка карточек
// =============================================================================

fn render_card_html(task: &TaskDef) -> String {
    let mut html = String::with_capacity(512);
    html.push_str(r#"<div class="arch-card" id="arch-card-"#);
    html.push_str(task.id);
    html.push_str(r#"">"#);

    html.push_str(r#"<div class="arch-card-title">"#);
    html.push_str(task.title);
    html.push_str(r#"</div>"#);

    html.push_str(r#"<div class="arch-card-sub">"#);
    html.push_str(task.subtitle);
    html.push_str(r#"</div>"#);

    html.push_str(r#"<div class="arch-card-timings">"#);
    html.push_str(r#"<span class="lbl">Rust:</span> <span class="rust" id="arch-rust-"#);
    html.push_str(task.id);
    html.push_str(r#"">—</span><br>"#);
    html.push_str(r#"<span class="lbl">JS:  </span> <span class="js" id="arch-js-"#);
    html.push_str(task.id);
    html.push_str(r#"">—</span>"#);
    html.push_str("</div>");

    html.push_str(r#"<div class="arch-card-out" id="arch-out-"#);
    html.push_str(task.id);
    html.push_str(r#""></div>"#);

    html.push_str(r#"<div class="arch-card-ratio" id="arch-ratio-"#);
    html.push_str(task.id);
    html.push_str(r#"">—</div>"#);

    html.push_str(r#"<div class="arch-card-suspect-msg" id="arch-suspect-"#);
    html.push_str(task.id);
    html.push_str(r#""></div>"#);

    html.push_str(r#"<button class="arch-btn arch-card-btn" data-run=""#);
    html.push_str(task.id);
    html.push_str(r#"">Запустить</button>"#);

    html.push_str("</div>");
    return html;
}

fn render_cards_html() -> String {
    let mut html = String::new();
    for task in TASKS {
        html.push_str(&render_card_html(task));
    }
    return html;
}

// =============================================================================
// Обновление карточки
// =============================================================================

fn update_card(doc: &Document, task: &TaskDef, result: &TaskResult) {
    let rust_id = format_str(format_args!("arch-rust-{}", task.id));
    if let Some(el) = doc.get_element_by_id(&rust_id) {
        let txt = format_str(format_args!(
            "{}{}",
            fmt_ms(result.rust_ms),
            fmt_iters(result.rust_iters),
        ));
        el.set_text_content(Some(&txt));
    }

    let js_id = format_str(format_args!("arch-js-{}", task.id));
    if let Some(el) = doc.get_element_by_id(&js_id) {
        let txt = format_str(format_args!(
            "{}{}",
            fmt_ms(result.js_ms),
            fmt_iters(result.js_iters),
        ));
        el.set_text_content(Some(&txt));
    }

    let out_id = format_str(format_args!("arch-out-{}", task.id));
    if let Some(el) = doc.get_element_by_id(&out_id) {
        if let Some(err) = &result.err {
            el.set_text_content(Some(err));
        } else {
            el.set_text_content(Some(&result.out));
        }
    }

    let ratio_id = format_str(format_args!("arch-ratio-{}", task.id));
    let card_id = format_str(format_args!("arch-card-{}", task.id));
    let suspect_id = format_str(format_args!("arch-suspect-{}", task.id));

    let speedup = result.speedup();
    let (ratio_text, class_suffix) = if result.err.is_some() {
        (String::from("ошибка"), "arch-loss")
    } else if result.suspect {
        (String::from("замер недостоверен"), "arch-suspect")
    } else if speedup >= 1.3 {
        (format_str(format_args!("Rust быстрее в {}", fmt_speedup(speedup))), "arch-win")
    } else if speedup >= 0.8 {
        (format_str(format_args!("сравнимо (Rust/JS = {})", fmt_speedup(speedup))), "arch-close")
    } else {
        (format_str(format_args!("JS быстрее в {}", fmt_speedup(1.0 / speedup.max(1e-9)))), "arch-loss")
    };

    if let Some(el) = doc.get_element_by_id(&ratio_id) {
        el.set_text_content(Some(&ratio_text));
        el.set_class_name(&format_str(format_args!("arch-card-ratio {}", class_suffix)));
    }
    if let Some(el) = doc.get_element_by_id(&card_id) {
        el.set_class_name(&format_str(format_args!("arch-card {}", class_suffix)));
    }
    if let Some(el) = doc.get_element_by_id(&suspect_id) {
        if result.suspect {
            el.set_text_content(Some(
                "⚠ измеренное время < 10 мкс — вероятно debug-сборка или ошибка таймера",
            ));
        } else {
            el.set_text_content(Some(""));
        }
    }
}

fn set_card_running(doc: &Document, task_id: &str, running: bool) {
    let card_id = format_str(format_args!("arch-card-{}", task_id));
    let Some(card) = doc.get_element_by_id(&card_id) else { return };
    if running {
        let _ = card.set_attribute("data-running", "");
        let _ = card.set_class_name("arch-card arch-running");
    } else {
        let _ = card.remove_attribute("data-running");
    }
}

fn render_summary(doc: &Document, results: &[Option<TaskResult>]) {
    let Some(el) = doc.get_element_by_id("arch-summary") else { return };

    let mut count = 0usize;
    let mut rust_wins = 0usize;
    let mut js_wins = 0usize;
    let mut close = 0usize;
    let mut log_sum = 0.0f64;
    let mut max_s = 0.0f64;
    let mut min_s = f64::INFINITY;

    for r in results.iter().flatten() {
        if r.err.is_some() || r.suspect {
            continue;
        }
        let s = r.speedup().max(1e-9);
        count += 1;
        log_sum += s.ln();
        if s > max_s { max_s = s; }
        if s < min_s { min_s = s; }
        if s >= 1.3 { rust_wins += 1; }
        else if s < 0.8 { js_wins += 1; }
        else { close += 1; }
    }

    if count == 0 {
        el.set_inner_html(
            r#"<div><span class="lbl">Пока ни одной задачи не запущено</span></div>"#,
        );
        return;
    }

    let geo_mean = (log_sum / (count as f64)).exp();

    let mut html = String::with_capacity(512);
    html.push_str("<div><span class=\"lbl\">Выполнено: </span><span class=\"val\">");
    html.push_str(&format_str(format_args!("{count} из {}", TASKS.len())));
    html.push_str("</span></div>");

    html.push_str("<div><span class=\"rust\">Rust быстрее: ");
    html.push_str(&format_str(format_args!("{rust_wins}")));
    html.push_str("</span> · <span class=\"close\">сравнимо: ");
    html.push_str(&format_str(format_args!("{close}")));
    html.push_str("</span> · <span class=\"js\">JS быстрее: ");
    html.push_str(&format_str(format_args!("{js_wins}")));
    html.push_str("</span></div>");

    html.push_str("<div><span class=\"lbl\">Геом. среднее Rust/JS: </span><span class=\"val\">");
    html.push_str(&fmt_speedup(geo_mean));
    html.push_str("</span> · <span class=\"lbl\">макс: </span><span class=\"val\">");
    html.push_str(&fmt_speedup(max_s));
    html.push_str("</span> · <span class=\"lbl\">мин: </span><span class=\"val\">");
    html.push_str(&fmt_speedup(min_s));
    html.push_str("</span></div>");

    el.set_inner_html(&html);
}

fn render_log(doc: &Document, logs: &[LogEntry]) {
    let Some(el) = doc.get_element_by_id("arch-log") else { return };
    el.set_inner_html(&render_log_html(logs));
    let _ = el.set_scroll_top(el.scroll_height() as f64);
}

fn set_buttons_enabled(doc: &Document, enabled: bool) {
    let Ok(nodes) = doc.query_selector_all("button[data-run], #arch-run-all") else {
        return;
    };
    for i in 0..nodes.length() {
        let Some(node) = nodes.get(i) else { continue };
        let Ok(el) = node.dyn_into::<web_sys::Element>() else { continue };
        if enabled {
            let _ = el.remove_attribute("disabled");
        } else {
            let _ = el.set_attribute("disabled", "");
        }
    }
}

// =============================================================================
// Задачи
// =============================================================================

fn run_task(task_id: &str) -> Result<TaskResult, String> {
    return match task_id {
        "fib" => run_fib(),
        "sort" => run_sort(),
        "crc32" => run_crc32(),
        "utf8" => run_utf8(),
        "frames" => run_frames(),
        _ => Err(format_str(format_args!("неизвестная задача: {task_id}"))),
    };
}

fn run_fib() -> Result<TaskResult, String> {
    let n: u32 = 30;

    let rust_v = fib_recursive(n);
    let f = compile_fib();
    let js_v = call_fib(&f, n)?;

    let out = if rust_v == js_v {
        format_str(format_args!("fib({n}) = {rust_v}"))
    } else {
        format_str(format_args!("MISMATCH: Rust {rust_v} vs JS {js_v}"))
    };

    // Двойной барьер: opaque для LLVM, __arch_chain для V8.
    // state становится функцией от результата fib, что делает
    // невозможным ни hoisting fib_recursive, ни DCE всего цикла.
    let mut rust_state: u32 = 12345;
    let rust_fn = move || {
        let nn = 30 + (rust_state & 0x1);
        let v = fib_recursive(nn) as u32;
        rust_state = __arch_chain(opaque(v ^ rust_state));
    };

    let mut js_state: u32 = 12345;
    let js_fn = move || {
        let nn = 30 + (js_state & 0x1);
        let v = call_fib(&f, nn).unwrap_or(0);
        js_state = __arch_chain((v as u32) ^ js_state);
    };

    let (rust_ms, js_ms, rust_iters, js_iters) = bench_pair(rust_fn, js_fn);

    let suspect = rust_ms < SUSPECT_THRESHOLD_MS || js_ms < SUSPECT_THRESHOLD_MS;

    return Ok(TaskResult {
        rust_ms, js_ms, rust_iters, js_iters, out, err: None, suspect,
    });
}

fn run_sort() -> Result<TaskResult, String> {
    let n: usize = 100_000;
    let seed: u32 = 0xC0FFEE;
    let data = gen_u32_array(n, seed);

    let mut rust_check = data.clone();
    sort_u32(&mut rust_check);

    let f_sort = compile_sort_u32();
    let js_check = js_sys::Uint32Array::from(&data[..]);
    let _ = f_sort
        .call1(&JsValue::NULL, js_check.as_ref())
        .map_err(|e| format_str(format_args!("js sort check: {e:?}")))?;
    let mut js_check_vec: Vec<u32> = vec![0; n];
    js_check.copy_to(&mut js_check_vec);

    let out = if js_check_vec == rust_check {
        format_str(format_args!("{n} элементов, отсортировано идентично"))
    } else {
        String::from("MISMATCH: Rust и JS дали разные массивы")
    };

    // state определяет модификацию data[0]; LLVM не может выкинуть
    // sort_u32, потому что результат уходит в opaque→__arch_chain,
    // а следующий state зависит от него.
    let data_for_rust = data.clone();
    let mut state_r: u32 = 0;
    let rust_fn = move || {
        let mut d = data_for_rust.clone();
        d[0] = d[0].wrapping_add(state_r);
        sort_u32(&mut d);
        let v = d[0] ^ d[d.len() - 1];
        state_r = __arch_chain(opaque(v ^ state_r));
    };

    let data_for_js = data.clone();
    let f_for_js = f_sort.clone();
    let mut state_j: u32 = 0;
    let js_fn = move || {
        let arr = js_sys::Uint32Array::from(&data_for_js[..]);
        let len = arr.length();
        if len > 0 {
            let first = arr.get_index(0);
            arr.set_index(0, first.wrapping_add(state_j));
        }
        let _ = f_for_js.call1(&JsValue::NULL, arr.as_ref());
        let v = if len > 0 {
            arr.get_index(0) ^ arr.get_index(len - 1)
        } else {
            0
        };
        state_j = __arch_chain(v ^ state_j);
    };

    let (rust_ms, js_ms, rust_iters, js_iters) = bench_pair(rust_fn, js_fn);

    let suspect = rust_ms < SUSPECT_THRESHOLD_MS || js_ms < SUSPECT_THRESHOLD_MS;

    return Ok(TaskResult {
        rust_ms, js_ms, rust_iters, js_iters, out, err: None, suspect,
    });
}

fn run_crc32() -> Result<TaskResult, String> {
    let n: usize = 1_048_576;
    let data = gen_bytes(n, 0xBEEF);

    let rust_v = crc32(&data);
    let f = compile_crc32();
    let js_v = call_crc32(&f, &data)?;

    let out = if rust_v == js_v {
        format_str(format_args!("CRC = {:08X}", rust_v))
    } else {
        format_str(format_args!("MISMATCH: Rust {:08X} vs JS {:08X}", rust_v, js_v))
    };

    // Модифицируем первый байт буфера на каждом шаге; CRC-32 чувствителен
    // ко всем битам, поэтому каждая итерация — реальный полный пересчёт.
    let mut data_for_rust = data.clone();
    let mut state_r: u32 = 0;
    let rust_fn = move || {
        data_for_rust[0] = data_for_rust[0].wrapping_add(state_r as u8);
        let v = crc32(&data_for_rust);
        state_r = __arch_chain(opaque(v ^ state_r));
    };

    let mut data_for_js = data.clone();
    let f_for_js = f.clone();
    let mut state_j: u32 = 0;
    let js_fn = move || {
        data_for_js[0] = data_for_js[0].wrapping_add(state_j as u8);
        let v = call_crc32(&f_for_js, &data_for_js).unwrap_or(0);
        state_j = __arch_chain(v ^ state_j);
    };

    let (rust_ms, js_ms, rust_iters, js_iters) = bench_pair(rust_fn, js_fn);

    let suspect = rust_ms < SUSPECT_THRESHOLD_MS || js_ms < SUSPECT_THRESHOLD_MS;

    return Ok(TaskResult {
        rust_ms, js_ms, rust_iters, js_iters, out, err: None, suspect,
    });
}

fn run_utf8() -> Result<TaskResult, String> {
    let count: usize = 100_000;
    let text = gen_text(count, 0xCAFE);

    let rust_v = count_code_points(&text);
    let f = compile_count_code_points();
    let js_v = call_count_code_points(&f, &text)?;

    let out = if rust_v as u32 == js_v {
        format_str(format_args!("{} code points", rust_v))
    } else {
        format_str(format_args!("MISMATCH: Rust {rust_v} vs JS {js_v}"))
    };

    // UTF-8 и так честно измеряется, но двойной барьер не помешает.
    let text_for_rust = text.clone();
    let mut state_r: u32 = 0;
    let rust_fn = move || {
        let v = count_code_points(&text_for_rust) as u32;
        state_r = __arch_chain(opaque(v ^ state_r));
    };

    let text_for_js = text.clone();
    let f_for_js = f.clone();
    let mut state_j: u32 = 0;
    let js_fn = move || {
        let v = call_count_code_points(&f_for_js, &text_for_js).unwrap_or(0);
        state_j = __arch_chain(v ^ state_j);
    };

    let (rust_ms, js_ms, rust_iters, js_iters) = bench_pair(rust_fn, js_fn);

    let suspect = rust_ms < SUSPECT_THRESHOLD_MS || js_ms < SUSPECT_THRESHOLD_MS;

    return Ok(TaskResult {
        rust_ms, js_ms, rust_iters, js_iters, out, err: None, suspect,
    });
}

fn run_frames() -> Result<TaskResult, String> {
    let frame_count: usize = 10_000;
    let payload_size: usize = 32;
    let stream = simulate_stream(frame_count, payload_size, 0xABBA);

    let rust_count = parse_frames_count(&stream);
    let f = compile_parse_frames();
    let js_count = call_parse_frames(&f, &stream)?;

    let out = if rust_count == js_count {
        format_str(format_args!("{} кадров, {} байт", rust_count, stream.len()))
    } else {
        format_str(format_args!("MISMATCH: Rust {rust_count} vs JS {js_count} кадров"))
    };

    // Меняем байт payload первого кадра (offset 3 — не заголовок,
    // не length). Парсер каждый раз пересчитывает CRC — иначе parity с JS.
    let mut stream_for_rust = stream.clone();
    let mut state_r: u32 = 0;
    let rust_fn = move || {
        stream_for_rust[3] = stream_for_rust[3].wrapping_add(state_r as u8);
        let v = parse_frames_count(&stream_for_rust);
        state_r = __arch_chain(opaque(v ^ state_r));
    };

    let mut stream_for_js = stream.clone();
    let f_for_js = f.clone();
    let mut state_j: u32 = 0;
    let js_fn = move || {
        stream_for_js[3] = stream_for_js[3].wrapping_add(state_j as u8);
        let v = call_parse_frames(&f_for_js, &stream_for_js).unwrap_or(0);
        state_j = __arch_chain(v ^ state_j);
    };

    let (rust_ms, js_ms, rust_iters, js_iters) = bench_pair(rust_fn, js_fn);

    let suspect = rust_ms < SUSPECT_THRESHOLD_MS || js_ms < SUSPECT_THRESHOLD_MS;

    return Ok(TaskResult {
        rust_ms, js_ms, rust_iters, js_iters, out, err: None, suspect,
    });
}

// =============================================================================
// Асинхронный запуск
// =============================================================================

async fn run_single(doc: Document, st: Rc<RefCell<AppState>>, idx: usize) {
    if st.borrow().running {
        return;
    }
    let task = &TASKS[idx];

    {
        let mut s = st.borrow_mut();
        s.running = true;
        s.results[idx] = None;
    }
    set_buttons_enabled(&doc, false);
    set_card_running(&doc, task.id, true);
    push_log(
        &st,
        format_str(format_args!("▶ {} · {}", task.title, task.subtitle)),
        false,
    );
    render_log(&doc, &st.borrow().logs);

    TimeoutFuture::new(0).await;

    let result = run_task(task.id);

    {
        let mut s = st.borrow_mut();
        match result {
            Ok(r) => { s.results[idx] = Some(r); }
            Err(e) => {
                s.results[idx] = Some(TaskResult {
                    rust_ms: 0.0, js_ms: 0.0,
                    rust_iters: 0, js_iters: 0,
                    out: String::new(),
                    err: Some(e.clone()),
                    suspect: false,
                });
            }
        }
    }

    let failed = match &st.borrow().results[idx] {
        Some(r) => r.err.clone(),
        None => None,
    };
    if let Some(msg) = failed {
        push_log(&st, format_str(format_args!("✗ {msg}")), true);
    }

    {
        let s = st.borrow();
        if let Some(Some(r)) = s.results.get(idx) {
            update_card(&doc, task, r);
        }
        render_summary(&doc, &s.results);
        render_log(&doc, &s.logs);
    }

    set_card_running(&doc, task.id, false);
    {
        let mut s = st.borrow_mut();
        s.running = false;
    }
    set_buttons_enabled(&doc, true);

    TimeoutFuture::new(0).await;
}

async fn run_all(doc: Document, st: Rc<RefCell<AppState>>) {
    if st.borrow().running {
        return;
    }
    {
        let mut s = st.borrow_mut();
        for r in s.results.iter_mut() {
            *r = None;
        }
    }
    push_log(&st, String::from("▶▶ Запустить все"), false);
    render_log(&doc, &st.borrow().logs);

    for i in 0..TASKS.len() {
        run_single(doc.clone(), st.clone(), i).await;
    }

    push_log(&st, String::from("✔ Все задачи выполнены"), false);
    render_log(&doc, &st.borrow().logs);
}

// =============================================================================
// Сборка UI
// =============================================================================

fn build_intro(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("arch-desc");
    d.set_inner_html(
        "<p class=\"arch-desc-line\"><strong>Где должен находиться Rust?</strong> \
         Пять задач-представителей: алгоритмы, данные, бинарная обработка, строки, \
         протоколы. Каждая запускается дважды на одних данных — в Rust/WASM и в чистом JavaScript.</p>\
         <p class=\"arch-desc-line\"><strong>Правило границы.</strong> Всё, что связано с байтами, \
         типами, алгоритмами и парсингом, живёт в Rust. Всё, что связано с DOM, событиями и \
         браузерными API, живёт в JS.</p>\
         <p class=\"arch-desc-line\"><strong>Крупные куски, а не байты.</strong> Rust готовит массив \
         или строку один раз и отдаёт её одним объектом. JS возвращает одно число.</p>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_critical_note(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("arch-note-critical");
    d.set_inner_html(
        "<strong>⚙ Для честных цифр соберите демо в release:</strong> \
         <code>wasm-pack build --release --target web</code>. \
         Замер использует <strong>двойной барьер против JIT</strong>: \
         <code>opaque()</code> (atomic store+load в Rust) не даёт LLVM вынести \
         вычисление из цикла, а JS-функция <code>__arch_chain</code> через \
         границу WASM↔JS не даёт TurboFan выбросить тело цикла. \
         Карточки с временем &lt; 10 мкс помечаются как <em>«замер недостоверен»</em>.",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_controls(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("arch-block");
    d.set_inner_html(
        "<button class=\"arch-btn arch-btn-primary\" id=\"arch-run-all\">Запустить все</button>\
         <span class=\"arch-hint\">Одни данные · одинаковые алгоритмы · двойной барьер · медиана из 5</span>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_summary(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("arch-summary");
    d.set_id("arch-summary");
    d.set_inner_html(
        "<div><span class=\"lbl\">Пока ни одной задачи не запущено</span></div>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_log(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("arch-log");
    d.set_id("arch-log");
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_framework(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("arch-framework");

    let mut html = String::with_capacity(2048);
    html.push_str("<h3>Фреймворк решений</h3>");
    html.push_str("<table>");
    html.push_str("<tr><th>Область</th><th>Где</th><th>Почему</th></tr>");

    html.push_str("<tr><td>UI orchestration</td><td class=\"usb-js\">JS</td>");
    html.push_str("<td>События, промисы, анимация — родная среда браузера</td></tr>");

    html.push_str("<tr><td>DOM manipulation</td><td class=\"usb-js\">JS</td>");
    html.push_str("<td>Каждое обращение к DOM — граница; делать их из Rust дорого</td></tr>");

    html.push_str("<tr><td>Application state</td><td class=\"usb-both\">Rust</td>");
    html.push_str("<td>Типизированные структуры, enum'ы, инварианты — компилятор ловит ошибки</td></tr>");

    html.push_str("<tr><td>Data processing</td><td class=\"usb-rust\">Rust</td>");
    html.push_str("<td>Массивы, сортировки, преобразования без бокса и полиморфизма</td></tr>");

    html.push_str("<tr><td>Binary processing</td><td class=\"usb-rust\">Rust</td>");
    html.push_str("<td>Побайтовые циклы по <code>&amp;[u8]</code> — без накладных расходов</td></tr>");

    html.push_str("<tr><td>Algorithms</td><td class=\"usb-rust\">Rust</td>");
    html.push_str("<td>Чистый CPU-bound; JIT проигрывает AOT-компиляции на рекурсии</td></tr>");

    html.push_str("<tr><td>Protocols</td><td class=\"usb-rust\">Rust</td>");
    html.push_str("<td>Фрейминг, CRC, парсинг полей — типы и <code>Result</code> вместо исключений</td></tr>");

    html.push_str("<tr><td>Cryptography</td><td class=\"usb-both\">Rust + Web Crypto</td>");
    html.push_str("<td>Примитивы — <code>SubtleCrypto</code> (timing-safe). Логика — Rust</td></tr>");

    html.push_str("<tr><td>Media processing</td><td class=\"usb-rust\">Rust</td>");
    html.push_str("<td>Демуксинг, кодеки, покадровая обработка — главы 20–23</td></tr>");

    html.push_str("<tr><td>Device protocols</td><td class=\"usb-rust\">Rust</td>");
    html.push_str("<td>Bluetooth GATT, Serial, USB — главы 45–47; парсинг и CRC — в Rust</td></tr>");

    html.push_str("</table>");
    d.set_inner_html(&html);
    return Ok(d.unchecked_into::<web_sys::Node>());
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("Глава 46 — Где должен находиться Rust"));
    root.append_child(as_node(&h2))?;

    root.append_child(&build_intro(&document)?)?;
    root.append_child(&build_critical_note(&document)?)?;
    root.append_child(&build_controls(&document)?)?;

    let state = Rc::new(RefCell::new(AppState::new()));
    let cards = document.create_element("div")?;
    cards.set_class_name("arch-cards");
    cards.set_inner_html(&render_cards_html());
    root.append_child(as_node(&cards))?;

    root.append_child(&build_summary(&document)?)?;
    root.append_child(&build_log(&document)?)?;
    root.append_child(&build_framework(&document)?)?;

    {
        let st = state.clone();
        let doc = document.clone();
        let closure: Closure<dyn FnMut(web_sys::Event)> =
            Closure::new(move |event: web_sys::Event| {
                let Some(target) = event.target() else { return };
                let Ok(target) = target.dyn_into::<web_sys::Element>() else { return };
                let Some(task_id) = target.get_attribute("data-run") else {
                    return;
                };
                let idx = TASKS.iter().position(|t| t.id == task_id);
                let Some(idx) = idx else { return };
                if st.borrow().running {
                    return;
                }
                let doc_async = doc.clone();
                let st_async = st.clone();
                spawn_local(async move {
                    run_single(doc_async, st_async, idx).await;
                });
            });
        cards
            .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let st = state.clone();
        let doc = document.clone();
        let btn = document
            .get_element_by_id("arch-run-all")
            .ok_or_else(|| JsValue::from_str("no #arch-run-all"))?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                run_all(doc_async, st_async).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    render_summary(&document, &state.borrow().results);
    render_log(&document, &state.borrow().logs);

    return Ok(());
}
```

---

## 46.8. Ловушки бенчмарков в браузере

Этот раздел — практический вывод главы. Путь, который мы прошли в разработке этой главы, показал, что **честный бенчмарк — это отдельная инженерная задача**, а не «написал цикл, замерил, поделил». Ниже — шесть ловушек, с которыми мы столкнулись, и способы их нейтрализации.

### 1. Constant folding (LLVM)

**Проявление.** `fib_recursive(30)` — компилятор знает `n = 30` на этапе компиляции, вычисляет результат и заменяет вызов константой. Замер показывает 12 нс вместо 4 мс.

**Как обходить.** Аргумент должен читаться из места, которое LLVM не может проанализировать. Простейший способ — чтение через `Reflect::get(window, ...)`: значение приходит из JS-мира, LLVM не может его свести к константе.

### 2. Loop-invariant code motion (LLVM)

**Проявление.** Если аргумент функции не меняется между итерациями, LLVM выносит вычисление из цикла, оставив внутри только инкремент счётчика. Замер показывает время пустого цикла.

**Как обходить.** Сделать аргумент i-й итерации зависимым от результата (i-1)-й итерации через барьер, который LLVM уважает. В нашем случае — `opaque()` с atomic store+load и `fence(SeqCst)`. Атомарные операции — часть контракта LLVM IR, их нельзя удалить или переупорядочить.

### 3. Dead store elimination (TurboFan)

**Проявление.** Если результат вычисления идёт в `Reflect::set(window, "__sink", v)`, а значение по этому ключу нигде в hot path не читается, TurboFan удаляет store. А с ним — и весь loop body.

**Как обходить.** Вызов через **границу WASM↔JS** (`inline_js`). TurboFan не инлайнит через эту границу и не может доказать отсутствие side effects у чужой функции. Возвращаемое значение обязано использоваться в следующей итерации — иначе сам вызов можно удалить как pure.

### 4. JIT tier-up (V8)

**Проявление.** Первые итерации выполняются в интерпретаторе Liftoff (медленно, но честно). После N вызовов V8 поднимает код до TurboFan, который оптимизирует и **выбрасывает тело цикла**. Замер, снятый после tier-up, в 500× меньше реального.

**Как обходить.** (a) Комбинировать `opaque` + `__arch_chain` — двойной барьер, каждый на своём уровне. (b) Прогревать функцию перед замером (`BENCH_WARMUP = 3`), чтобы tier-up случился до начала измерения. (c) **Sanity check**: если измеренное время < 10 мкс — пометить карточку как подозрительную.

### 5. Разрешение `performance.now()`

**Проявление.** Chrome с 89-й версии загрубляет точность `performance.now()` (Spectre mitigation) до 5–100 мкс. Если операция занимает меньше, таймер возвращает 0 или один квант. Замер «одной итерации» даёт абсурдные значения.

**Как обходить.** **Адаптивные итерации**: калибровкой подбираем число `iters` так, чтобы суммарное время было ≥ 100 мс. Тогда точность таймера (0.1 мс) даёт относительную погрешность < 0.1%. Медиана из 5 прогонов сглаживает остаточный шум.

### 6. `std::hint::black_box` не работает в WASM

**Проявление.** В нативном Rust `black_box(v)` — гарантия, что компилятор не тронет `v`. В WASM-таргете `black_box` компилируется в `ReadVolatile` без реального barrier-эффекта. LLVM и V8 его игнорируют.

**Как обходить.** Использовать `opaque()` с atomic-операциями и `#[inline(never)]`. Не доверять `black_box` в WASM.

### Итоговое правило

**Если ваш замер в браузере даёт значение, которое физически невозможно (микросекунды на миллисекундную работу) — вы измеряете пустой цикл.** Единственный способ убедиться, что замер честный — сравнить **калибровочное время** (когда JIT ещё не тронул код) с **медианой** (когда JIT уже поработал). Если разница в 1.5–2 раза — замер честный. Если в 100–500 раз — нет.

---

## 46.9. Итог

Что даёт глава:

1. **Реальные цифры на машине студента.** Не «Rust быстрее», а «Fibonacci: 4.10 мс против 10.69 мс (2.61×)». Это меняет интуицию.
2. **Пять категорий** — студент видит, что Rust выигрывает по-разному: где-то в 2.6× (алгоритмы), где-то в 36× (UTF-8).
3. **Правило границы** — данные готовятся один раз в Rust, передаются одним объектом в JS. Это конкретная техника, а не абстрактный совет.
4. **Урок про бенчмарки** — важнее любого вывода о «скорости Rust». Наивный замер врёт; чтобы получить честную цифру, нужно понимать работу JIT и ставить барьеры на двух уровнях (LLVM + TurboFan).

**Правила проекта соблюдены:**

- `format_str(format_args!(...))` — везде, где `format!` мог бы оказаться в хвостовой позиции.
- `TimeoutFuture::new(0).await` — между задачами в `run_all`.
- `Rc<RefCell<AppState>>` — общее состояние.
- Префикс CSS `arch-` — уникален.
- `app.rs` содержит только демо текущей главы.
- Никаких `unused_imports`, `dead_code`, `unused_mut`.
- HTML с несколькими вложенными `"` — через `push_str`, не через `format_args!`.

**Cargo.toml:** добавлена секция `[package.metadata.wasm-pack.profile.release]` с `wasm-opt = false` — иначе сборка падает на старом `binaryen`. Никаких новых фич web-sys не нужно.

**Тесты:** 28 тестов — эталонные CRC-вектора, round-trip кадров, детерминированные генераторы, проверка совпадения JS-обёрток с Rust-реализациями.

**Главный урок главы, который стоит вынести в её название:** **«Rust быстрее» — это не аксиома, а результат (алгоритм × сборка × честность измерения).** Именно это показывает демо и раздел 46.8.

**Команды финальной проверки:**

```bash
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests
wasm-pack test --headless --chrome --test wasm_arch
wasm-pack build --release --target web
python3 -m http.server 5173
```

**Ожидаемые результаты после `--release`:**

| Задача | Rust | JS | Ratio |
|---|---|---|---|
| Fibonacci(30) | ~4 мс | ~11 мс | 2.6× |
| Сортировка 100k | ~1.8 мс | ~21 мс | 11.8× |
| CRC-32 · 1 МБ | ~8.6 мс | ~40 мс | 4.7× |
| UTF-8 · 100k | ~40 мкс | ~1.45 мс | 36.5× |
| Парсинг 10k кадров | ~3.8 мс | ~11 мс | 3.0× |

**Геом. среднее: ~6.9×.**