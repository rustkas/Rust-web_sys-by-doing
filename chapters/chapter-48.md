# Глава 48. Panic и восстановление после сбоев

## 48.0. Обновлённая структура

```
src/
  panic_guard.rs          ← НОВЫЙ модуль 48 (prevention, diagnostics, recovery)
  panic_scenarios.rs      ← НОВЫЙ модуль 48 (WASM-экспорты для Worker'а)
  video_frame.rs          ← из 23.1 (format_str)
  app.rs                  ← демо 48

tests/
  wasm_panic.rs           ← тесты 48 (не путать с wasm_errors.rs — тот про fetch)

serve.py                  ← НОВЫЙ вспомогательный файл (правильные MIME)
Cargo.toml                ← уже содержит console_error_panic_hook
src/lib.rs                ← +2 строки
```

**Комментарий.** Глава — про то, что **панику в WASM нельзя поймать**. С `panic = "abort"` (дефолт для `wasm32-unknown-unknown`) паника превращается в `unreachable`-trap, который **завершает экземпляр WASM-модуля**. `std::panic::catch_unwind` не спасает — стек не раскручивается. Единственный правильный паттерн — **не допускать панику** там, где её нельзя позволить (границы с JS), **изолировать** риск в Worker, **сохранять состояние** перед риском, **логировать** через `console_error_panic_hook`.

Модули:

- `panic_guard.rs` — prevention (`safe_*`), diagnostics (panic hook), recovery (localStorage).
- `panic_scenarios.rs` — WASM-экспорт `panic_worker_entry` для запуска в Worker'е.

**Префикс CSS** — `pn-`. Не пересекается с `bt-`, `sl-`, `usb-`, `arch-`, `err-`, `vf-`, `wg-`.

**Ссылки для детального изучения:**

- Rust Reference: [`panic = "abort"`](https://doc.rust-lang.org/rustc/codegen-options/index.html#panic), [`std::panic`](https://doc.rust-lang.org/std/panic/index.html)
- Rust/WASM Book: [Debugging](https://rustwasm.github.io/book/reference/debugging.html), [Console Error Panic Hook](https://github.com/rustwasm/console_error_panic_hook)
- MDN: [WebAssembly traps](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/RuntimeError), [Worker](https://developer.mozilla.org/en-US/docs/Web/API/Worker), [import()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import)

## 48.1. О `Cargo.toml`

**`console_error_panic_hook` уже в зависимостях.** Ничего добавлять не нужно. Секция `[package.metadata.wasm-pack.profile.release]` с `wasm-opt = false` — из главы 50.

## 48.2. Модуль `src/panic_guard.rs`

```rust
//! 48. Panic и восстановление: prevention, diagnostics, recovery.
//!
//! ## Ключевой факт
//!
//! На `wasm32-unknown-unknown` дефолтная стратегия — `panic = "abort"`.
//! Паника вызывает `unreachable`-инструкцию, что приводит к WASM-trap.
//! Модуль становится непригодным, `std::panic::catch_unwind` не работает.
//!
//! ## Что здесь
//!
//! - **Prevention** — функции `safe_*`, возвращающие `Result` вместо паники.
//! - **Diagnostics** — `install_panic_hook()` для читаемых сообщений в консоли.
//! - **Recovery** — сохранение/загрузка состояния через `localStorage`.
//! - **Classification** — категоризация panic-сообщений для UI.

use crate::video_frame::format_str;

// =============================================================================
// Diagnostics — panic hook
// =============================================================================

/// Устанавливает `console_error_panic_hook::set_once()`.
///
/// Без хука паника в WASM выглядит как `RuntimeError: unreachable` —
/// без сообщения, без стека, без места. С хуком в консоль попадает
/// полноценное сообщение: `panicked at 'msg', src/file.rs:line`.
///
/// ## Идемпотентность
///
/// `set_once()` — идемпотентна. Повторный вызов — no-op.
/// Можно безопасно вызывать в начале `run()`.
pub fn install_panic_hook() {
    console_error_panic_hook::set_once();
}

// =============================================================================
// Prevention — Result вместо panic
// =============================================================================

/// Безопасное деление. `b == 0` → `Err`, не паника.
///
/// `a / 0.0` для `f64` даёт `inf`/`NaN`, но **целочисленное** деление
/// на ноль паникует. Здесь мы явно отсекаем оба случая и возвращаем
/// `Result`, который вызывающий обработает.
pub fn safe_divide(a: f64, b: f64) -> Result<f64, String> {
    if !a.is_finite() || !b.is_finite() {
        return Err(String::from("нечисловой операнд"));
    }
    if b == 0.0 {
        return Err(String::from("деление на ноль"));
    }
    return Ok(a / b);
}

/// Безопасный доступ по индексу. `index >= len` → `Err`, не паника.
pub fn safe_at_u32(data: &[u32], index: usize) -> Result<u32, String> {
    if index >= data.len() {
        return Err(format_str(format_args!(
            "индекс {index} вне диапазона (длина {})",
            data.len()
        )));
    }
    return Ok(data[index]);
}

/// Безопасный парсинг `i32`.
pub fn safe_parse_int(s: &str) -> Result<i32, String> {
    let trimmed = s.trim();
    if trimmed.is_empty() {
        return Err(String::from("пустая строка"));
    }
    return trimmed
        .parse::<i32>()
        .map_err(|e| format_str(format_args!("не число: {e}")));
}

/// Обёртка над `Option`. `None` → `Err` с указанием, что не найдено.
pub fn safe_unwrap_or_err<T>(opt: Option<T>, what: &str) -> Result<T, String> {
    return opt.ok_or_else(|| format_str(format_args!("{what}: значение отсутствует")));
}

// =============================================================================
// Classification — категория паники по сообщению
// =============================================================================

/// Категория паники по тексту сообщения. Используется в UI,
/// чтобы показать пользователю понятную подсказку.
///
/// ## Порядок проверок важен
///
/// Более специфичные паттерны идут **раньше** общих. Например,
/// `"capacity overflow"` содержит подстроку `"overflow"` — если
/// сначала проверить `overflow`, то `"capacity overflow"` никогда
/// не дойдёт до своей ветки.
pub fn classify_panic_msg(msg: &str) -> &'static str {
    let lower = msg.to_lowercase();
    if lower.contains("index out of bounds") {
        return "IndexOutOfBounds";
    }
    if lower.contains("divide by zero") || lower.contains("division by zero") {
        return "DivideByZero";
    }
    if lower.contains("called `option::unwrap()`") {
        return "UnwrapOnNone";
    }
    if lower.contains("called `result::unwrap()`") {
        return "UnwrapOnErr";
    }
    if lower.contains("assertion failed") || lower.contains("assertion `") {
        return "AssertionFailed";
    }
    if lower.contains("unreachable") {
        return "Unreachable";
    }
    // Специфичный паттерн раньше общего: "capacity overflow"
    // содержит "overflow" как подстроку.
    if lower.contains("capacity overflow") {
        return "CapacityOverflow";
    }
    if lower.contains("overflow") {
        return "ArithmeticOverflow";
    }
    return "Explicit";
}

// =============================================================================
// Recovery — localStorage
// =============================================================================

/// Сохраняет значение по ключу в `localStorage`.
pub fn save_state(key: &str, value: &str) -> Result<(), String> {
    let window = web_sys::window().ok_or_else(|| String::from("no window"))?;
    let storage = window
        .local_storage()
        .map_err(|e| format_str(format_args!("localStorage: {e:?}")))?
        .ok_or_else(|| String::from("localStorage недоступен"))?;
    storage
        .set_item(key, value)
        .map_err(|e| format_str(format_args!("setItem: {e:?}")))?;
    return Ok(());
}

/// Загружает значение по ключу. `None`, если ключа нет или
/// `localStorage` недоступен.
pub fn load_state(key: &str) -> Option<String> {
    let window = web_sys::window()?;
    let storage = window.local_storage().ok()??;
    return storage.get_item(key).ok()?;
}

/// Удаляет ключ. Идемпотентно: удаление отсутствующего ключа — успех.
pub fn clear_state(key: &str) -> Result<(), String> {
    let window = web_sys::window().ok_or_else(|| String::from("no window"))?;
    let storage = window
        .local_storage()
        .map_err(|e| format_str(format_args!("localStorage: {e:?}")))?
        .ok_or_else(|| String::from("localStorage недоступен"))?;
    storage
        .remove_item(key)
        .map_err(|e| format_str(format_args!("removeItem: {e:?}")))?;
    return Ok(());
}
```

## 48.3. Модуль `src/panic_scenarios.rs`

```rust
//! 48. Panic-сценарии для запуска в Web Worker.
//!
//! ## Зачем отдельный модуль
//!
//! Эти функции **намеренно паникуют**. Вызывать их из main thread
//! нельзя — модуль умрёт. Единственный безопасный контекст —
//! Worker: паника убьёт Worker, но main thread продолжит работу.

use wasm_bindgen::prelude::*;

/// Намеренная паника. Запускается в Worker'е.
///
/// ## Что произойдёт
///
/// - Rust-паника → `unreachable` trap → WASM-trap.
/// - Worker получает uncaught error и завершается.
/// - Main thread получает событие `error` на объекте Worker.
/// - **Main thread продолжает работать.**
#[wasm_bindgen]
pub fn panic_worker_entry() {
    panic!("Demonstration panic inside Web Worker");
}

/// Безопасный ping для проверки, что Worker жив.
#[wasm_bindgen]
pub fn ping_worker_entry() -> u32 {
    return 42;
}
```

## 48.4. Добавление в `src/lib.rs`

```rust
pub mod app;
pub mod arch_js_bench;
pub mod arch_pure;
pub mod bluetooth_gatt;
pub mod bluetooth_utils;
pub mod container_parser;
pub mod error_info;
pub mod error_scenarios;
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod panic_guard;         // ← 48
pub mod panic_scenarios;     // ← 48
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

## 48.5. Тесты `tests/wasm_panic.rs`

```rust
//! 48. Тесты panic-guard: prevention, classification, recovery.
//!
//! ## Что тестируется
//!
//! - `safe_*` функции возвращают `Err` там, где обычный Rust паникует.
//! - `classify_panic_msg` правильно категоризирует сообщения.
//! - `save_state` / `load_state` / `clear_state` — round-trip через localStorage.
//! - `install_panic_hook` — идемпотентна.
//!
//! ## Что НЕ тестируется
//!
//! Реальная паника. С `panic = "abort"` она завершает WASM-модуль,
//! и тест не сможет продолжить. Демонстрация паники — в `app.rs`
//! через Worker.

use wasm_bindgen_test::*;

use hello_wasm02::panic_guard::{
    classify_panic_msg, clear_state, install_panic_hook, load_state,
    safe_at_u32, safe_divide, safe_parse_int, safe_unwrap_or_err, save_state,
};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Prevention — safe_divide
// =============================================================================

#[wasm_bindgen_test]
fn test_safe_divide_ok() {
    assert_eq!(safe_divide(10.0, 2.0), Ok(5.0));
    assert_eq!(safe_divide(-9.0, 3.0), Ok(-3.0));
}

#[wasm_bindgen_test]
fn test_safe_divide_by_zero() {
    let r = safe_divide(10.0, 0.0);
    assert!(r.is_err());
    assert!(r.unwrap_err().contains("ноль"));
}

#[wasm_bindgen_test]
fn test_safe_divide_nan() {
    assert!(safe_divide(f64::NAN, 1.0).is_err());
    assert!(safe_divide(1.0, f64::INFINITY).is_err());
}

// =============================================================================
// Prevention — safe_at_u32
// =============================================================================

#[wasm_bindgen_test]
fn test_safe_at_ok() {
    let data = [10u32, 20, 30];
    assert_eq!(safe_at_u32(&data, 0), Ok(10));
    assert_eq!(safe_at_u32(&data, 2), Ok(30));
}

#[wasm_bindgen_test]
fn test_safe_at_out_of_bounds() {
    let data = [10u32, 20, 30];
    let r = safe_at_u32(&data, 5);
    assert!(r.is_err());
    assert!(r.unwrap_err().contains("вне диапазона"));
}

#[wasm_bindgen_test]
fn test_safe_at_empty() {
    let data: [u32; 0] = [];
    assert!(safe_at_u32(&data, 0).is_err());
}

// =============================================================================
// Prevention — safe_parse_int
// =============================================================================

#[wasm_bindgen_test]
fn test_safe_parse_ok() {
    assert_eq!(safe_parse_int("42"), Ok(42));
    assert_eq!(safe_parse_int("  -17  "), Ok(-17));
}

#[wasm_bindgen_test]
fn test_safe_parse_err() {
    assert!(safe_parse_int("abc").is_err());
    assert!(safe_parse_int("").is_err());
    assert!(safe_parse_int("42.5").is_err());
}

// =============================================================================
// Prevention — safe_unwrap_or_err
// =============================================================================

#[wasm_bindgen_test]
fn test_safe_unwrap_ok() {
    assert_eq!(safe_unwrap_or_err(Some(7), "x"), Ok(7));
}

#[wasm_bindgen_test]
fn test_safe_unwrap_err() {
    let r: Result<i32, String> = safe_unwrap_or_err(None, "items");
    assert!(r.is_err());
    assert!(r.unwrap_err().contains("items"));
}

// =============================================================================
// Classification
// =============================================================================

#[wasm_bindgen_test]
fn test_classify_index_out_of_bounds() {
    assert_eq!(
        classify_panic_msg("index out of bounds: the len is 3 but the index is 5"),
        "IndexOutOfBounds"
    );
}

#[wasm_bindgen_test]
fn test_classify_divide_by_zero() {
    assert_eq!(classify_panic_msg("attempt to divide by zero"), "DivideByZero");
    assert_eq!(classify_panic_msg("Division by zero"), "DivideByZero");
}

#[wasm_bindgen_test]
fn test_classify_unwrap() {
    assert_eq!(
        classify_panic_msg("called `Option::unwrap()` on a `None` value"),
        "UnwrapOnNone"
    );
    assert_eq!(
        classify_panic_msg("called `Result::unwrap()` on an `Err` value: ..."),
        "UnwrapOnErr"
    );
}

#[wasm_bindgen_test]
fn test_classify_assertion() {
    assert_eq!(
        classify_panic_msg("assertion failed: x > 0"),
        "AssertionFailed"
    );
}

#[wasm_bindgen_test]
fn test_classify_unreachable() {
    assert_eq!(classify_panic_msg("entered unreachable code"), "Unreachable");
}

#[wasm_bindgen_test]
fn test_classify_overflow() {
    assert_eq!(classify_panic_msg("attempt to add with overflow"), "ArithmeticOverflow");
    assert_eq!(classify_panic_msg("capacity overflow"), "CapacityOverflow");
}

#[wasm_bindgen_test]
fn test_classify_explicit() {
    assert_eq!(classify_panic_msg("custom panic message"), "Explicit");
}

// =============================================================================
// Recovery — localStorage round-trip
// =============================================================================

#[wasm_bindgen_test]
fn test_save_load_clear_roundtrip() {
    let key = "__wasm_panic_test_key";
    let value = "recovered value 42";

    let _ = clear_state(key);

    save_state(key, value).expect("save failed");
    let loaded = load_state(key);
    assert_eq!(loaded.as_deref(), Some(value));

    clear_state(key).expect("clear failed");
    assert_eq!(load_state(key), None);
}

#[wasm_bindgen_test]
fn test_clear_idempotent() {
    let key = "__wasm_panic_test_idempotent";
    let _ = clear_state(key);
    clear_state(key).expect("second clear should succeed");
    clear_state(key).expect("third clear should succeed");
}

#[wasm_bindgen_test]
fn test_install_panic_hook_idempotent() {
    install_panic_hook();
    install_panic_hook();
    // Если не паникует — ок.
}
```

**Команды запуска тестов:**

```bash
# Только тесты главы 48
wasm-pack test --headless --chrome --test wasm_panic

# Все тесты проекта
wasm-pack test --headless --chrome

# С захватом вывода
wasm-pack test --headless --chrome --test wasm_panic -- --nocapture
```

## 48.6. Вспомогательный файл `serve.py`

**Зачем нужен.** `python3 -m http.server` в некоторых версиях отдаёт `.js` как `text/plain`. **Модульные воркеры** (`new Worker(url, {type: 'module'})`) отказываются загружать модуль с неправильным MIME-типом — и делают это **молча**, без события `error`. Именно поэтому демо может застрять в состоянии `spawning…`.

```python
#!/usr/bin/env python3
"""Локальный сервер для WASM-демо.

## Зачем не `python3 -m http.server`

`http.server` в некоторых версиях Python отдаёт `.js` как `text/plain`.
Модульные воркеры (`new Worker(url, {type:'module'})`) отказываются
загружать модуль с неправильным MIME-типом — и делают это **молча**,
без события `error` на объекте Worker. Именно поэтому демо с паникой
застревает в состоянии `spawning…`.

## Использование

    cd ~/projects/wa/hello-wasm02
    python3 serve.py
    # Откройте http://127.0.0.1:5173/

## Альтернатива

    npx serve .
"""

from __future__ import annotations

import http.server
import socketserver
import sys
from typing import ClassVar

PORT = 5173


class WasmHandler(http.server.SimpleHTTPRequestHandler):
    """Обработчик HTTP с правильными MIME-типами для WASM-демо.

    Наследует `SimpleHTTPRequestHandler`, переопределяя карту расширений
    и добавляя заголовки против кэширования. Без этого модульные
    воркеры не загружаются из-за `text/plain` для `.js`.
    """

    # Правильные MIME-типы для WASM-демо.
    # Основные правки: .js → text/javascript, .wasm → application/wasm.
    # ClassVar — чтобы линтеры не считали это мутабельным атрибутом
    # экземпляра (RUF012).
    extensions_map: ClassVar[dict[str, str]] = {
        **http.server.SimpleHTTPRequestHandler.extensions_map,
        ".js": "text/javascript",
        ".mjs": "text/javascript",
        ".wasm": "application/wasm",
        ".json": "application/json",
        ".html": "text/html",
    }

    def end_headers(self) -> None:
        """Добавляет заголовки против кэша и вызывает родителя.

        Отключаем кэш — иначе после `wasm-pack build` браузер может
        отдавать старый `.wasm`.
        """
        self.send_header("Cache-Control", "no-store, no-cache, must-revalidate")
        self.send_header("Pragma", "no-cache")
        super().end_headers()


class ReusableTCPServer(socketserver.TCPServer):
    """TCPServer с SO_REUSEADDR — перезапуск без TIME_WAIT."""

    allow_reuse_address = True


def main() -> None:
    """Запускает сервер на `PORT`. Завершается с кодом 1, если порт занят."""
    try:
        with ReusableTCPServer(("", PORT), WasmHandler) as httpd:
            print(f"serving at http://127.0.0.1:{PORT}/")
            print("Ctrl+C to stop")
            httpd.serve_forever()
    except OSError as e:
        print(f"cannot bind port {PORT}: {e}", file=sys.stderr)
        sys.exit(1)
    except KeyboardInterrupt:
        print("\nstopped")


if __name__ == "__main__":
    main()
```

## 48.7. Демо `src/app.rs`

```rust
//! Глава 48 — Panic и восстановление после сбоев.
//!
//! ## Идея демо
//!
//! Четыре зоны:
//! 1. **Prevention** — как не паниковать? `Result` вместо `unwrap`.
//! 2. **Diagnostics** — как увидеть, где упало? `console_error_panic_hook`.
//! 3. **Isolation** — как не убить main thread? Worker.
//! 4. **Recovery** — как вернуть состояние? `localStorage`.
//!
//! ## Сквозной сценарий
//!
//! Demo → pre-flight MIME-check → worker spawn → module load →
//! ping → save state → panic in worker → **onerror** → auto-load
//! state → «восстановлено: <value>». Main thread ни на секунду не
//! останавливался.
//!
//! ## Почему onmessage в воркере СИНХРОННЫЙ
//!
//! Если объявить `onmessage` как `async (e) => { panic_worker_entry(); }`,
//! исключение превратится в rejected promise. Браузер в воркере
//! триггерит `unhandledrejection`, но НЕ `worker.onerror` на main
//! thread. Наш обработчик `onerror` не сработает — демонстрация
//! изоляции сломана.
//!
//! Решение: `onmessage` синхронный. Panic вызывается синхронно →
//! uncaught error в воркере → `worker.onerror` в main thread.
//! Асинхронные операции (import, init) — в отдельной функции
//! `doInit`, вызываемой без await.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::callback::Timeout;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::Document;

use crate::panic_guard::{
    classify_panic_msg, clear_state, install_panic_hook, load_state, safe_at_u32,
    safe_divide, safe_parse_int, save_state,
};
use crate::video_frame::format_str;

// =============================================================================
// Константы
// =============================================================================

const RECOVERY_KEY: &str = "__wasm_panic_demo_state";
const WORKER_SPAWN_TIMEOUT_MS: u32 = 5000;
const WORKER_MODULE_URL: &str = "/pkg/hello_wasm02.js";

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.pn-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.pn-desc-line { margin:6px 0; }
.pn-desc-line code { background:#0d1117; padding:1px 5px; border-radius:3px; color:#79c0ff; }

.pn-zone { max-width:1000px;margin-bottom:16px;padding:14px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;}
.pn-zone-head { display:flex; align-items:baseline; gap:10px; margin-bottom:10px; flex-wrap:wrap; }
.pn-zone-num { font-size:11px; color:#6e7681; font-family:ui-monospace,monospace; font-weight:700; }
.pn-zone-title { font-size:14px; color:#79c0ff; font-weight:600; }
.pn-zone-sub { font-size:11px; color:#8b949e; text-transform:uppercase; letter-spacing:1px; margin-left:auto; }

.pn-block { display:flex; flex-wrap:wrap; gap:8px; align-items:center; margin-bottom:8px; }
.pn-btn { padding:7px 12px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.pn-btn:hover:not([disabled]) { background:#30363d; }
.pn-btn:disabled { opacity:0.5;cursor:not-allowed; }
.pn-btn-safe { background:#1a3828;border-color:#2ea043;color:#7ee2a8; }
.pn-btn-safe:hover:not([disabled]) { background:#214b32; }
.pn-btn-warn { background:#3a2810;border-color:#d29922;color:#f0c674; }
.pn-btn-warn:hover:not([disabled]) { background:#503718; }

.pn-result { font-family:ui-monospace,monospace;font-size:12px;padding:8px 10px;border-radius:4px;margin-top:4px;min-height:20px;overflow-wrap:anywhere; }
.pn-result-ok { background:#0a1f10; color:#7ee2a8; border-left:3px solid #2ea043; }
.pn-result-err { background:#1a0d0d; color:#ffa198; border-left:3px solid #f85149; }
.pn-result-idle { background:#161b22; color:#6e7681; border-left:3px solid #30363d; }

.pn-badge { display:inline-flex; align-items:center; padding:2px 8px; border-radius:10px; font-size:10px; font-weight:600; letter-spacing:0.5px; text-transform:uppercase; }
.pn-badge-ok { background:#1a3828; color:#7ee2a8; border:1px solid #2ea043; }
.pn-badge-warn { background:#3a2810; color:#f0c674; border:1px solid #d29922; }
.pn-badge-dead { background:#4a1a1a; color:#ffa198; border:1px solid #f85149; }
.pn-badge-idle { background:#21262d; color:#8b949e; border:1px solid #30363d; }

.pn-state-box { margin-top:10px; padding:8px 10px; background:#161b22; border-radius:4px; border-left:3px solid #30363d; font-family:ui-monospace,monospace; font-size:11.5px; color:#8b949e; }
.pn-state-box .k { color:#6e7681; }
.pn-state-box .v { color:#79c0ff; }
.pn-state-box.pn-state-active { border-left-color:#2ea043; }
.pn-state-box.pn-state-active .v { color:#7ee2a8; }
.pn-state-box.pn-state-empty { border-left-color:#30363d; color:#6e7681; }

.pn-log { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;line-height:1.6;color:#c9d1d9;max-height:240px;overflow-y:auto;}
.pn-log-entry { display:flex; gap:8px; padding:1px 0; }
.pn-log-entry .time { color:#6e7681; flex-shrink:0; }
.pn-log-entry .msg { color:#c9d1d9; overflow-wrap:anywhere; }
.pn-log-info .msg { color:#79c0ff; }
.pn-log-ok .msg { color:#7ee2a8; }
.pn-log-warn .msg { color:#f0c674; }
.pn-log-err .msg { color:#ffa198; }

.pn-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
.pn-footnote code { background:#0d1117; padding:1px 5px; border-radius:3px; color:#79c0ff; }
"#;

// =============================================================================
// Модели
// =============================================================================

struct LogEntry {
    time: String,
    level: &'static str,
    text: String,
}

struct AppState {
    logs: Vec<LogEntry>,
    worker: Option<web_sys::Worker>,
    worker_timeout: Option<Timeout>,
    last_recovered: Option<String>,
}

impl AppState {
    fn new() -> Self {
        return Self {
            logs: Vec::new(),
            worker: None,
            worker_timeout: None,
            last_recovered: None,
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
// UI-утилиты
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "pn-injected-style";
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

fn escape_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;");
}

fn push_log(st: &Rc<RefCell<AppState>>, level: &'static str, text: String) {
    let mut s = st.borrow_mut();
    s.logs.push(LogEntry {
        time: now_timestamp(),
        level,
        text,
    });
    if s.logs.len() > 100 {
        let drop = s.logs.len() - 100;
        s.logs.drain(0..drop);
    }
}

fn render_log_html(logs: &[LogEntry]) -> String {
    let mut html = String::with_capacity(logs.len() * 100);
    for e in logs {
        html.push_str(r#"<div class="pn-log-entry pn-log-"#);
        html.push_str(e.level);
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

fn render_log(doc: &Document, logs: &[LogEntry]) {
    let Some(el) = doc.get_element_by_id("pn-log") else { return };
    el.set_inner_html(&render_log_html(logs));
    if let Ok(el) = el.dyn_into::<web_sys::HtmlElement>() {
        let _ = el.set_scroll_top(el.scroll_height() as f64);
    }
}

fn set_result(doc: &Document, id: &str, kind: &str, text: &str) {
    let Some(el) = doc.get_element_by_id(id) else { return };
    el.set_class_name(&format_str(format_args!("pn-result pn-result-{}", kind)));
    el.set_text_content(Some(text));
}

fn set_worker_status(doc: &Document, status: &'static str, badge_text: &str) {
    let Some(el) = doc.get_element_by_id("pn-worker-status") else { return };
    let cls = match status {
        "idle" => "pn-badge pn-badge-idle",
        "running" => "pn-badge pn-badge-ok",
        "dead" => "pn-badge pn-badge-dead",
        _ => "pn-badge pn-badge-idle",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(badge_text));
}

fn update_panic_button(doc: &Document, has_worker: bool) {
    let Some(el) = doc.get_element_by_id("pn-act-panic-worker") else { return };
    let Ok(el) = el.dyn_into::<web_sys::HtmlElement>() else { return };
    if has_worker {
        let _ = el.remove_attribute("disabled");
    } else {
        let _ = el.set_attribute("disabled", "");
    }
}

fn render_state_box(doc: &Document, recovered: Option<&str>) {
    let Some(el) = doc.get_element_by_id("pn-recovery-state") else { return };
    match load_state(RECOVERY_KEY) {
        Some(v) => {
            let mut html = String::with_capacity(256);
            html.push_str(r#"<span class="k">localStorage["#);
            html.push_str(RECOVERY_KEY);
            html.push_str(r#"] = </span><span class="v">"#);
            html.push_str(&escape_html(&v));
            html.push_str("</span>");
            if let Some(r) = recovered {
                html.push_str(r#"<br><span class="k">восстановлено после паники: </span><span class="v">"#);
                html.push_str(&escape_html(r));
                html.push_str("</span>");
            }
            el.set_class_name("pn-state-box pn-state-active");
            el.set_inner_html(&html);
        }
        None => {
            el.set_class_name("pn-state-box pn-state-empty");
            el.set_text_content(Some("localStorage пуст — состояние не сохранено"));
        }
    }
}

// =============================================================================
// Действия: Safe zone
// =============================================================================

fn on_safe_divide(doc: Document, st: Rc<RefCell<AppState>>) {
    let (a, b) = (10.0, 0.0);
    push_log(&st, "info", format_str(format_args!("safe_divide({a}, {b})")));
    match safe_divide(a, b) {
        Ok(v) => set_result(&doc, "pn-res-divide", "ok", &format_str(format_args!("Ok({v})"))),
        Err(e) => set_result(
            &doc,
            "pn-res-divide",
            "err",
            &format_str(format_args!("Err(\"{e}\")")),
        ),
    }
    render_log(&doc, &st.borrow().logs);
}

fn on_safe_at(doc: Document, st: Rc<RefCell<AppState>>) {
    let data = [10u32, 20, 30];
    let idx = 5usize;
    push_log(&st, "info", format_str(format_args!("safe_at_u32(len=3, index={idx})")));
    match safe_at_u32(&data, idx) {
        Ok(v) => set_result(&doc, "pn-res-at", "ok", &format_str(format_args!("Ok({v})"))),
        Err(e) => set_result(
            &doc,
            "pn-res-at",
            "err",
            &format_str(format_args!("Err(\"{e}\")")),
        ),
    }
    render_log(&doc, &st.borrow().logs);
}

fn on_safe_parse(doc: Document, st: Rc<RefCell<AppState>>) {
    let input = "abc";
    push_log(&st, "info", format_str(format_args!("safe_parse_int(\"{input}\")")));
    match safe_parse_int(input) {
        Ok(v) => set_result(&doc, "pn-res-parse", "ok", &format_str(format_args!("Ok({v})"))),
        Err(e) => set_result(
            &doc,
            "pn-res-parse",
            "err",
            &format_str(format_args!("Err(\"{e}\")")),
        ),
    }
    render_log(&doc, &st.borrow().logs);
}

// =============================================================================
// Действия: Diagnostics
// =============================================================================

fn on_show_hook_info(doc: Document, st: Rc<RefCell<AppState>>) {
    push_log(&st, "ok", String::from("panic hook уже активен (set_once при старте)"));
    push_log(
        &st,
        "info",
        String::from("теперь паника в консоли = 'panicked at ...', а не 'unreachable'"),
    );
    set_result(
        &doc,
        "pn-res-hook",
        "ok",
        "console_error_panic_hook::set_once() — вызвано в run()",
    );
    render_log(&doc, &st.borrow().logs);
}

// =============================================================================
// Действия: Recovery
// =============================================================================

fn on_save_state(doc: Document, st: Rc<RefCell<AppState>>) {
    let value = format_str(format_args!("work-{}", now_timestamp()));
    match save_state(RECOVERY_KEY, &value) {
        Ok(()) => {
            push_log(&st, "ok", format_str(format_args!("состояние сохранено: {value}")));
            set_result(
                &doc,
                "pn-res-save",
                "ok",
                &format_str(format_args!("save_state(\"{RECOVERY_KEY}\", \"{value}\")")),
            );
        }
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("save failed: {e}")));
            set_result(&doc, "pn-res-save", "err", &e);
        }
    }
    let last_rec = st.borrow().last_recovered.clone();
    render_state_box(&doc, last_rec.as_deref());
    render_log(&doc, &st.borrow().logs);
}

fn on_clear_state(doc: Document, st: Rc<RefCell<AppState>>) {
    match clear_state(RECOVERY_KEY) {
        Ok(()) => {
            push_log(&st, "ok", String::from("состояние очищено"));
            set_result(&doc, "pn-res-save", "idle", "—");
            st.borrow_mut().last_recovered = None;
        }
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("clear failed: {e}")));
        }
    }
    let last_rec = st.borrow().last_recovered.clone();
    render_state_box(&doc, last_rec.as_deref());
    render_log(&doc, &st.borrow().logs);
}

// =============================================================================
// Действия: Worker
// =============================================================================

/// Код воркера.
///
/// ## Синхронный `onmessage` — принципиально
///
/// `onmessage` объявлен как **обычная функция** (не `async`). Это
/// критично для демонстрации паники: если обработчик async, то
/// `panic_worker_entry()` бросает внутри promise → rejected promise →
/// `unhandledrejection` вместо `worker.onerror`. Наш обработчик
/// `onerror` на main thread не сработает.
///
/// Асинхронная инициализация вынесена в отдельную `doInit()`,
/// вызываемую без `await`. После `moduleReady = true` все вызовы
/// WASM-экспортов синхронные.
///
/// ## Dynamic import
///
/// Воркер создан из Blob URL — его base URL указывает на `blob:...`.
/// Статический `import from '/pkg/...'` резолвится от этого base и
/// ведёт в никуда. Используем `import(fullUrl)` с полным URL,
/// который main thread передаёт в сообщении `init`.
const WORKER_CODE: &str = r#"
let init, panic_worker_entry, ping_worker_entry;
let moduleReady = false;

async function doInit(origin) {
    const jsUrl = new URL('/pkg/hello_wasm02.js', origin).href;
    const wasmUrl = new URL('/pkg/hello_wasm02_bg.wasm', origin).href;
    self.postMessage({ type: 'module-loading', url: jsUrl });
    try {
        const mod = await import(jsUrl);
        init = mod.default;
        panic_worker_entry = mod.panic_worker_entry;
        ping_worker_entry = mod.ping_worker_entry;
        await init(wasmUrl);
        moduleReady = true;
        self.postMessage({ type: 'spawned' });
    } catch (err) {
        self.postMessage({ type: 'module-error', message: String(err) });
    }
}

self.onmessage = (e) => {
    if (e.data.type === 'init') {
        const origin = e.data.origin || self.location.origin;
        doInit(origin);
        return;
    }

    if (!moduleReady) {
        self.postMessage({ type: 'error', message: 'module not ready' });
        return;
    }

    if (e.data.type === 'ping') {
        try {
            const v = ping_worker_entry();
            self.postMessage({ type: 'pong', value: v });
        } catch (err) {
            self.postMessage({ type: 'error', message: String(err) });
        }
        return;
    }

    if (e.data.type === 'panic') {
        self.postMessage({ type: 'about-to-panic' });
        panic_worker_entry();
        return;
    }
};
"#;

fn cancel_worker_timeout(st: &Rc<RefCell<AppState>>) {
    st.borrow_mut().worker_timeout = None;
}

fn extract_content_type(resp: &JsValue) -> Option<String> {
    let headers = js_sys::Reflect::get(resp, &JsValue::from_str("headers"))
        .ok()
        .filter(|v| !v.is_undefined() && !v.is_null())?;
    let get = js_sys::Reflect::get(&headers, &JsValue::from_str("get"))
        .ok()
        .and_then(|v| v.dyn_into::<js_sys::Function>().ok())?;
    let ct = get
        .call1(&headers, &JsValue::from_str("content-type"))
        .ok()
        .and_then(|v| v.as_string())?;
    return Some(ct);
}

fn preflight_check_mime(
    doc: Document,
    st: Rc<RefCell<AppState>>,
    on_ok: impl FnOnce(Document, Rc<RefCell<AppState>>) + 'static,
) {
    let doc_cb = doc.clone();
    let st_cb = st.clone();
    spawn_local(async move {
        let Some(window) = web_sys::window() else { return };

        let promise: js_sys::Promise = window.fetch_with_str(WORKER_MODULE_URL);

        let resp_value: JsValue = match JsFuture::from(promise).await {
            Ok(v) => v,
            Err(_) => {
                push_log(
                    &st_cb,
                    "warn",
                    String::from("pre-flight fetch не удался — пропускаем проверку"),
                );
                render_log(&doc_cb, &st_cb.borrow().logs);
                on_ok(doc_cb, st_cb);
                return;
            }
        };

        let ct = extract_content_type(&resp_value).unwrap_or_default();
        let ct_lower = ct.to_lowercase();
        let is_js = ct_lower.contains("text/javascript")
            || ct_lower.contains("application/javascript");

        if is_js {
            push_log(
                &st_cb,
                "ok",
                format_str(format_args!("pre-flight: content-type OK ({ct})")),
            );
            render_log(&doc_cb, &st_cb.borrow().logs);
            on_ok(doc_cb, st_cb);
        } else {
            push_log(
                &st_cb,
                "err",
                format_str(format_args!(
                    "pre-flight: сервер отдаёт .js как '{ct}' — модульный воркер не загрузится"
                )),
            );
            push_log(
                &st_cb,
                "warn",
                String::from(
                    "перезапустите сервер: `python3 serve.py` или `npx serve .`",
                ),
            );
            set_worker_status(&doc_cb, "dead", "bad mime");
            update_panic_button(&doc_cb, false);
            render_log(&doc_cb, &st_cb.borrow().logs);
        }
    });
}

fn spawn_worker(doc: Document, st: Rc<RefCell<AppState>>) {
    preflight_check_mime(doc, st, spawn_worker_after_preflight);
}

fn spawn_worker_after_preflight(doc: Document, st: Rc<RefCell<AppState>>) {
    let old_worker = st.borrow_mut().worker.take();
    if let Some(w) = old_worker {
        w.terminate();
    }
    cancel_worker_timeout(&st);

    let parts = js_sys::Array::new();
    parts.push(&JsValue::from_str(WORKER_CODE));
    let options = web_sys::BlobPropertyBag::new();
    options.set_type("text/javascript");
    let blob = match web_sys::Blob::new_with_str_sequence_and_options(&parts, &options) {
        Ok(b) => b,
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("Blob::new: {e:?}")));
            render_log(&doc, &st.borrow().logs);
            return;
        }
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("Url::create: {e:?}")));
            render_log(&doc, &st.borrow().logs);
            return;
        }
    };

    let worker_options = web_sys::WorkerOptions::new();
    worker_options.set_type(web_sys::WorkerType::Module);
    let worker = match web_sys::Worker::new_with_options(&url, &worker_options) {
        Ok(w) => w,
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("Worker::new: {e:?}")));
            render_log(&doc, &st.borrow().logs);
            return;
        }
    };

    // onmessage
    {
        let st_cb = st.clone();
        let doc_cb = doc.clone();
        let closure: Closure<dyn FnMut(web_sys::MessageEvent)> =
            Closure::new(move |event: web_sys::MessageEvent| {
                let data = event.data();
                let kind = js_sys::Reflect::get(&data, &JsValue::from_str("type"))
                    .ok()
                    .and_then(|v| v.as_string())
                    .unwrap_or_default();

                let get_str = |key: &str| -> String {
                    return js_sys::Reflect::get(&data, &JsValue::from_str(key))
                        .ok()
                        .and_then(|v| v.as_string())
                        .unwrap_or_default();
                };

                match kind.as_str() {
                    "module-loading" => {
                        let url = get_str("url");
                        push_log(
                            &st_cb,
                            "info",
                            format_str(format_args!("worker: загружаем модуль {url}")),
                        );
                    }
                    "module-error" => {
                        let msg = get_str("message");
                        push_log(
                            &st_cb,
                            "err",
                            format_str(format_args!(
                                "worker: не удалось загрузить модуль — {msg}"
                            )),
                        );
                        set_worker_status(&doc_cb, "dead", "module error");
                        update_panic_button(&doc_cb, false);
                        {
                            let mut s = st_cb.borrow_mut();
                            s.worker = None;
                            s.worker_timeout = None;
                        }
                    }
                    "spawned" => {
                        push_log(&st_cb, "ok", String::from("worker: spawned"));
                        set_worker_status(&doc_cb, "running", "running");
                        cancel_worker_timeout(&st_cb);
                    }
                    "pong" => {
                        let v = js_sys::Reflect::get(&data, &JsValue::from_str("value"))
                            .ok()
                            .and_then(|v| v.as_f64())
                            .unwrap_or(0.0);
                        push_log(&st_cb, "ok", format_str(format_args!("worker: pong {v}")));
                    }
                    "about-to-panic" => {
                        push_log(&st_cb, "warn", String::from("worker: about to panic"));
                    }
                    "unreachable-here" => {
                        push_log(
                            &st_cb,
                            "err",
                            String::from("worker: unexpectedly survived panic"),
                        );
                    }
                    "error" => {
                        let msg = get_str("message");
                        push_log(
                            &st_cb,
                            "err",
                            format_str(format_args!("worker: error — {msg}")),
                        );
                    }
                    _ => {
                        push_log(
                            &st_cb,
                            "info",
                            format_str(format_args!("worker: unknown message '{kind}'")),
                        );
                    }
                }
                render_log(&doc_cb, &st_cb.borrow().logs);
            });
        worker.set_onmessage(Some(closure.as_ref().unchecked_ref()));
        closure.forget();
    }

    // onerror — сюда приходит trap из panic_worker_entry().
    // Сработает только если onmessage в воркере синхронный (см.
    // комментарий к WORKER_CODE).
    {
        let st_cb = st.clone();
        let doc_cb = doc.clone();
        let closure: Closure<dyn FnMut(web_sys::ErrorEvent)> =
            Closure::new(move |event: web_sys::ErrorEvent| {
                let msg = event.message();
                let category = classify_panic_msg(&msg);
                push_log(
                    &st_cb,
                    "err",
                    format_str(format_args!("worker: died — [{category}] {msg}")),
                );
                push_log(
                    &st_cb,
                    "info",
                    String::from("main thread жив — WASM main thread не тронут"),
                );
                set_worker_status(&doc_cb, "dead", "dead");

                let recovered = load_state(RECOVERY_KEY);
                if let Some(v) = &recovered {
                    push_log(
                        &st_cb,
                        "ok",
                        format_str(format_args!("восстановлено из localStorage: {v}")),
                    );
                } else {
                    push_log(
                        &st_cb,
                        "warn",
                        String::from("localStorage пуст — нечего восстанавливать"),
                    );
                }

                {
                    let mut s = st_cb.borrow_mut();
                    s.worker = None;
                    s.worker_timeout = None;
                    s.last_recovered = recovered.clone();
                }
                update_panic_button(&doc_cb, false);
                render_state_box(&doc_cb, recovered.as_deref());
                render_log(&doc_cb, &st_cb.borrow().logs);
            });
        worker.set_onerror(Some(closure.as_ref().unchecked_ref()));
        closure.forget();
    }

    // Отправляем init с origin.
    let origin = web_sys::window()
        .and_then(|w| w.location().origin().ok())
        .unwrap_or_else(|| String::from(""));

    let init_msg = js_sys::Object::new();
    js_sys::Reflect::set(&init_msg, &JsValue::from_str("type"), &JsValue::from_str("init")).ok();
    js_sys::Reflect::set(
        &init_msg,
        &JsValue::from_str("origin"),
        &JsValue::from_str(&origin),
    )
    .ok();
    let _ = worker.post_message(&init_msg);

    push_log(
        &st,
        "info",
        format_str(format_args!("worker: spawn requested (origin={origin})")),
    );
    set_worker_status(&doc, "running", "loading…");
    update_panic_button(&doc, true);
    render_log(&doc, &st.borrow().logs);

    st.borrow_mut().worker = Some(worker);

    let doc_to = doc.clone();
    let st_to = st.clone();
    let timeout = Timeout::new(WORKER_SPAWN_TIMEOUT_MS, move || {
        let still_pending = st_to.borrow().worker.is_some();
        if !still_pending {
            return;
        }
        push_log(
            &st_to,
            "err",
            format_str(format_args!(
                "worker: не ответил за {} мс",
                WORKER_SPAWN_TIMEOUT_MS
            )),
        );
        push_log(
            &st_to,
            "warn",
            String::from("откройте DevTools → Console. Точная причина — там."),
        );
        set_worker_status(&doc_to, "dead", "no response");
        update_panic_button(&doc_to, false);
        {
            let mut s = st_to.borrow_mut();
            s.worker = None;
            s.worker_timeout = None;
        }
        render_log(&doc_to, &st_to.borrow().logs);
    });

    st.borrow_mut().worker_timeout = Some(timeout);
}

fn send_worker_panic(doc: Document, st: Rc<RefCell<AppState>>) {
    let worker = st.borrow().worker.clone();
    let Some(worker) = worker else {
        push_log(&st, "warn", String::from("worker не запущен"));
        render_log(&doc, &st.borrow().logs);
        return;
    };

    let value = format_str(format_args!("pre-panic-{}", now_timestamp()));
    let _ = save_state(RECOVERY_KEY, &value);
    push_log(
        &st,
        "info",
        format_str(format_args!("состояние сохранено до паники: {value}")),
    );
    let last_rec = st.borrow().last_recovered.clone();
    render_state_box(&doc, last_rec.as_deref());

    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("panic")).ok();
    match worker.post_message(&msg) {
        Ok(()) => {
            push_log(&st, "warn", String::from("worker: panic message sent"));
        }
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("post_message: {e:?}")));
        }
    }
    render_log(&doc, &st.borrow().logs);
}

// =============================================================================
// Сборка UI
// =============================================================================

fn build_intro(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("pn-desc");
    d.set_inner_html(
        "<p class=\"pn-desc-line\"><strong>Panic в WASM = fatal.</strong> \
         С <code>panic = \"abort\"</code> (дефолт для <code>wasm32-unknown-unknown</code>) \
         паника превращается в <code>unreachable</code>-инструкцию. WASM-trap. \
         Экземпляр модуля становится непригодным. \
         <code>catch_unwind</code> не спасает — стек не раскручивается.</p>\
         <p class=\"pn-desc-line\"><strong>Правильный паттерн — три уровня.</strong> \
         ① <strong>Prevent</strong>: <code>Result</code> на границах с JS, никаких \
         <code>unwrap()</code> в публичных функциях. \
         ② <strong>Isolate</strong>: рисковая работа — в Worker'е. \
         ③ <strong>Recover</strong>: сохраняй состояние <em>до</em> риска в \
         <code>localStorage</code>, восстанавливай <em>после</em>.</p>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_safe_zone(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("pn-zone");

    let mut html = String::with_capacity(1024);
    html.push_str(r#"<div class="pn-zone-head"><span class="pn-zone-num">①</span><span class="pn-zone-title">Prevention — Result вместо panic</span><span class="pn-zone-sub">безопасная альтернатива</span></div>"#);

    html.push_str(r#"<div class="pn-block"><button class="pn-btn pn-btn-safe" id="pn-act-divide">safe_divide(10.0, 0.0)</button></div>"#);
    html.push_str(r#"<div class="pn-result pn-result-idle" id="pn-res-divide">—</div>"#);
    html.push_str(r#"<div style="height:10px"></div>"#);

    html.push_str(r#"<div class="pn-block"><button class="pn-btn pn-btn-safe" id="pn-act-at">safe_at_u32(len=3, index=5)</button></div>"#);
    html.push_str(r#"<div class="pn-result pn-result-idle" id="pn-res-at">—</div>"#);
    html.push_str(r#"<div style="height:10px"></div>"#);

    html.push_str(r#"<div class="pn-block"><button class="pn-btn pn-btn-safe" id="pn-act-parse">safe_parse_int("abc")</button></div>"#);
    html.push_str(r#"<div class="pn-result pn-result-idle" id="pn-res-parse">—</div>"#);

    d.set_inner_html(&html);
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_diag_zone(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("pn-zone");

    let mut html = String::with_capacity(512);
    html.push_str(r#"<div class="pn-zone-head"><span class="pn-zone-num">②</span><span class="pn-zone-title">Diagnostics — panic hook</span><span class="pn-zone-sub">читаемые сообщения в консоли</span></div>"#);
    html.push_str(r#"<div class="pn-block"><button class="pn-btn" id="pn-act-hook">Показать статус hook</button></div>"#);
    html.push_str(r#"<div class="pn-result pn-result-idle" id="pn-res-hook">hook установлен автоматически в run()</div>"#);
    html.push_str(r#"<div style="margin-top:10px;font-size:12px;color:#8b949e;">Без хука: <code style="color:#f0883e">RuntimeError: unreachable</code>. С хуком: <code style="color:#7ee2a8">panicked at 'msg', src/file.rs:line</code>.</div>"#);

    d.set_inner_html(&html);
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_worker_zone(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("pn-zone");

    let mut html = String::with_capacity(1024);
    html.push_str(r#"<div class="pn-zone-head"><span class="pn-zone-num">③</span><span class="pn-zone-title">Isolation — Worker</span><span class="pn-zone-sub">паника не убивает main thread</span><span id="pn-worker-status" class="pn-badge pn-badge-idle">idle</span></div>"#);

    html.push_str(r#"<div class="pn-block"><button class="pn-btn pn-btn-safe" id="pn-act-spawn">Spawn worker</button><button class="pn-btn pn-btn-warn" id="pn-act-panic-worker" disabled>Panic in worker</button></div>"#);
    html.push_str(r#"<div style="font-size:12px;color:#8b949e;">Сценарий: нажмите <strong>Panic in worker</strong> — состояние сохраняется в localStorage <em>до</em> паники, воркер умирает, main thread продолжает работу и автоматически восстанавливает состояние.</div>"#);
    html.push_str(r#"<div style="margin-top:8px;font-size:11px;color:#6e7681;">Требуется сервер с правильными MIME: <code style="background:#0d1117;padding:1px 4px;border-radius:3px;color:#79c0ff">python3 serve.py</code> или <code style="background:#0d1117;padding:1px 4px;border-radius:3px;color:#79c0ff">npx serve .</code>.</div>"#);

    d.set_inner_html(&html);
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_recovery_zone(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("pn-zone");

    let mut html = String::with_capacity(768);
    html.push_str(r#"<div class="pn-zone-head"><span class="pn-zone-num">④</span><span class="pn-zone-title">Recovery — localStorage</span><span class="pn-zone-sub">сохраняй до, восстанавливай после</span></div>"#);
    html.push_str(r#"<div class="pn-block"><button class="pn-btn" id="pn-act-save">Сохранить состояние</button><button class="pn-btn" id="pn-act-clear">Очистить</button></div>"#);
    html.push_str(r#"<div class="pn-result pn-result-idle" id="pn-res-save">—</div>"#);
    html.push_str(r#"<div class="pn-state-box pn-state-empty" id="pn-recovery-state">localStorage пуст — состояние не сохранено</div>"#);

    d.set_inner_html(&html);
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_footnote(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("pn-footnote");
    d.set_inner_html(
        "<strong>Почему <code>catch_unwind</code> не работает.</strong> \
         В нативных сборках `panic = \"unwind\"` раскручивает стек — \
         `catch_unwind` может перехватить панику. В WASM дефолт — `panic = \"abort\"`: \
         паника превращается в `unreachable`, стек не раскручивается. \
         `catch_unwind` вернёт `Err` только если паника не пересекла границу с JS — \
         а у нас каждая паника пересекает её через сгенерированный `wasm-bindgen` shim. \
         Поэтому стратегия — не ловить, а изолировать.<br>\
         <strong>Panic в Service Worker.</strong> SW живёт между сессиями; \
         паника убивает его до следующего события. Не держите состояние в памяти SW — \
         только в IndexedDB / Cache API.<br>\
         <strong>Panic vs Result.</strong> <code>Result</code> — для ожидаемых ошибок. \
         <code>panic!</code> — для нарушенных инвариантов, которые не должны случаться. \
         Публичные функции (вызываемые из JS) — только <code>Result</code>.<br>\
         <strong>Почему onmessage в воркере СИНХРОННЫЙ.</strong> \
         Если обработчик <code>async</code>, паника превращается в rejected promise \
         и триггерит <code>unhandledrejection</code>, а не <code>worker.onerror</code>. \
         Наш <code>onerror</code> на main thread не сработает — демонстрация изоляции \
         сломается. Синхронный обработчик даёт настоящий uncaught error → \
         <code>worker.onerror</code> → авто-recovery.",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

// =============================================================================
// Точка входа
// =============================================================================

fn attach_click<F>(
    doc: &Document,
    id: &str,
    st: Rc<RefCell<AppState>>,
    mut action: F,
) -> Result<(), JsValue>
where
    F: FnMut(Document, Rc<RefCell<AppState>>) + 'static,
{
    let Some(el) = doc.get_element_by_id(id) else {
        return Ok(());
    };
    let el: web_sys::HtmlElement = el.dyn_into()?;
    let doc_clone = doc.clone();
    let closure: Closure<dyn FnMut()> = Closure::new(move || {
        action(doc_clone.clone(), st.clone());
    });
    el.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    return Ok(());
}

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    install_panic_hook();

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("Глава 48 — Panic и восстановление после сбоев"));
    root.append_child(as_node(&h2))?;

    root.append_child(&build_intro(&document)?)?;

    let state = Rc::new(RefCell::new(AppState::new()));

    if let Some(v) = load_state(RECOVERY_KEY) {
        push_log(
            &state,
            "ok",
            format_str(format_args!("найдено сохранённое состояние: {v}")),
        );
    }

    root.append_child(&build_safe_zone(&document)?)?;
    root.append_child(&build_diag_zone(&document)?)?;
    root.append_child(&build_worker_zone(&document)?)?;
    root.append_child(&build_recovery_zone(&document)?)?;

    let log = document.create_element("div")?;
    log.set_class_name("pn-log");
    log.set_id("pn-log");
    root.append_child(as_node(&log))?;

    root.append_child(&build_footnote(&document)?)?;

    attach_click(&document, "pn-act-divide", state.clone(), on_safe_divide)?;
    attach_click(&document, "pn-act-at", state.clone(), on_safe_at)?;
    attach_click(&document, "pn-act-parse", state.clone(), on_safe_parse)?;
    attach_click(&document, "pn-act-hook", state.clone(), on_show_hook_info)?;
    attach_click(&document, "pn-act-save", state.clone(), on_save_state)?;
    attach_click(&document, "pn-act-clear", state.clone(), on_clear_state)?;
    attach_click(&document, "pn-act-spawn", state.clone(), spawn_worker)?;
    attach_click(&document, "pn-act-panic-worker", state.clone(), send_worker_panic)?;

    push_log(
        &state,
        "info",
        String::from("demo ready — panic hook установлен, можно запускать сценарии"),
    );

    let last_rec = state.borrow().last_recovered.clone();
    render_state_box(&document, last_rec.as_deref());
    render_log(&document, &state.borrow().logs);

    return Ok(());
}
```

## 48.8. Уроки, которые мы прошли в этой главе

Этот раздел — практический вывод. Путь к рабочему демо прошёл через **шесть последовательных препятствий**. Каждое из них — реальная проблема в production-коде Rust/WASM.

### 1. `panic = "abort"` делает `catch_unwind` бесполезным

На `wasm32-unknown-unknown` дефолтная стратегия — `panic = "abort"`. Паника превращается в `unreachable`-инструкцию, **стек не раскручивается**, `std::panic::catch_unwind` не может перехватить. Экземпляр WASM-модуля становится непригодным. Единственный способ избежать — **не паниковать** там, где это критично.

**Решение:** `Result` на границах с JS. `safe_divide`, `safe_at_u32`, `safe_parse_int` — все возвращают `Err`, не паникуют.

### 2. `console_error_panic_hook` нужен, но не обязателен

Без хука паника в консоли выглядит как `RuntimeError: unreachable` — без файла, строки, сообщения. С хуком — `panicked at 'msg', src/file.rs:line`. Хук **не предотвращает** панику, но даёт диагностику.

**Решение:** `install_panic_hook()` в начале `run()`. Идемпотентно, безопасно вызывать повторно.

### 3. MIME-типы важны для модульных воркеров

`python3 -m http.server` в некоторых версиях отдаёт `.js` как `text/plain`. **Модульные воркеры** (`new Worker(url, {type: 'module'})`) отказываются загружать такой модуль — и делают это **молча**, без события `error` на объекте Worker.

**Решение:** `serve.py` с правильными `extensions_map`. Плюс **pre-flight проверка MIME** в самом демо — `fetch('/pkg/hello_wasm02.js')` + чтение `Content-Type` из response.

### 4. Blob URL ломает статический `import`

Воркер создаётся из Blob URL — его base URL указывает на `blob:...`. Статический `import from '/pkg/...'` резолвится от этого base и ведёт в никуда. Модуль молча не загружается.

**Решение:** Dynamic `import(fullUrl)` с полным URL, переданным из main thread. Main thread знает `location.origin()` и строит `new URL('/pkg/hello_wasm02.js', origin).href`.

### 5. Try/catch вокруг паники проглатывает trap

Если панику обернуть в `try/catch`, JS-trap от WASM будет пойман как обычное исключение и превратится в `postMessage({type: 'error'})`. `worker.onerror` на main thread **не сработает** — badge останется «running», авто-recovery не триггернется.

**Решение:** `panic_worker_entry()` вызывается **без** `try/catch`. Trap вылетает из `onmessage`, браузер триггерит `worker.onerror`. Это и есть «изоляция».

### 6. `async onmessage` даёт ту же проблему

Даже без явного `try/catch`, **`async onmessage`** превращает исключение в rejected promise. Браузер триггерит `unhandledrejection`, но **не** `worker.onerror`.

**Решение:** `onmessage` **синхронный**. Асинхронные операции (`import`, `init`) — в отдельной `async function doInit()`, вызываемой fire-and-forget. После `moduleReady = true` все вызовы WASM-экспортов синхронные.

## 48.9. Итог

Что даёт глава:

1. **Границу ошибок и паники в действии.** 4 зоны демонстрируют: `Result` на границах, hook для диагностики, Worker для изоляции, localStorage для восстановления.

2. **Сквозной сценарий resilience.** Save → panic → isolate → recover. Main thread ни разу не остановился. Всё видно в логе.

3. **Уроки про JIT/trap/async.** Шесть проблем, каждое решение — production-relevant.

4. **Самодиагностирующееся демо.** Pre-flight MIME, timeout на spawn, отдельные сообщения для `module-error`, `died`, `no response`. Любая проблема окружения даёт точное сообщение.

**Правила проекта соблюдены:**

- `format_str(format_args!(...))` — везде.
- `Rc<RefCell<AppState>>` — общее состояние.
- Префикс CSS `pn-` — уникален.
- `app.rs` содержит только демо текущей главы.
- Никаких `unused_imports`, `dead_code`, `unused_mut`.

**Тесты:** 20 тестов — prevention, classification, recovery, hook идемпотентность.

**Главный урок главы.** Паника в WASM — **fatal**. Её нельзя поймать, только **предотвратить**. `Result` на границах, Worker для изоляции, `localStorage` для recovery. И **тонкости, без которых ничего не работает**: синхронный `onmessage`, dynamic import с полным URL, правильные MIME-типы.

**Команды финальной проверки:**

```bash
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests
wasm-pack test --headless --chrome --test wasm_panic
wasm-pack build --release --target web

# Запуск через serve.py (правильные MIME + SO_REUSEADDR)
python3 serve.py
# http://127.0.0.1:5173/
```

## 48.10. Полезные команды для отладки

**Убить зависший сервер на порту 5173:**

```bash
# Самый быстрый способ
kill -9 $(lsof -t -i :5173)

# Альтернатива без lsof
fuser -k 5173/tcp

# Через ss + pgrep
kill $(pgrep -f "serve.py")
```

**Проверить MIME-тип `.js` от сервера:**

```bash
curl -I http://127.0.0.1:5173/pkg/hello_wasm02.js | grep -i content-type
# Должно быть: text/javascript
```

**Посмотреть ошибку воркера в DevTools:**

1. **Console** — там будет точное сообщение от воркера.
2. **Network** — фильтр `hello_wasm02.js`, посмотреть Status и `Content-Type`.
3. **Sources → Workers** — Blob-воркеры видны как отдельные контексты, можно поставить breakpoint.

**Если `Address already in use`** — либо предыдущий сервер не завершён, либо TIME_WAIT (30–60 секунд). `serve.py` использует `SO_REUSEADDR`, так что после `Ctrl+C` перезапуск мгновенный.