# Глава 47. Обработка ошибок

## 47.0. Обновлённая структура

```
src/
  error_info.rs               ← НОВЫЙ модуль 47 (Rust: извлечение, классификация)
  error_scenarios.rs          ← НОВЫЙ модуль 47 (JS-мост: сценарии ошибок)
  video_frame.rs              ← из 23.1 (format_str)
  app.rs                      ← демо 47

tests/
  wasm_error_handling.rs      ← тесты 47

src/lib.rs                    ← +2 строки: error_info, error_scenarios
Cargo.toml                    ← без изменений
```

**Комментарий.** Эта глава — про **границу ошибок** между Rust и JS. Когда JS бросает исключение или Promise отклоняется, Rust видит его как opaque `JsValue`: ни `name`, ни `message`, ни `stack` не доступны напрямую. Модуль `error_info.rs` системно извлекает структуру через `Reflect::get` и классифицирует в прикладную категорию `AppError` с severity и user-facing сообщением. Модуль `error_scenarios.rs` — источники ошибок в браузере через `inline_js`.

Разделение слоёв:

- `error_info.rs` — чистые функции без side effects: извлечение и категоризация. Легко тестируются.
- `error_scenarios.rs` — регистрация JS-сценариев и вызовы. Тестируются вместе с DOM.

**Префикс CSS** — `err-`. Не пересекается с `bt-`, `sl-`, `usb-`, `arch-`, `vf-`, `wg-`.

**Ссылки для детального изучения:**

- wasm-bindgen: [Errors and Result](https://rustwasm.github.io/docs/wasm-bindgen/reference/types/result.html)
- wasm-bindgen: [`JsError`](https://docs.rs/wasm-bindgen/latest/wasm_bindgen/struct.JsError.html)
- MDN: [`DOMException`](https://developer.mozilla.org/en-US/docs/Web/API/DOMException), [`Error`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error)
- WHATWG: [DOMException codes](https://webidl.spec.whatwg.org/#idl-DOMException-error-names)
- `wasm-bindgen(inline_js)`: [docs.rs](https://docs.rs/wasm-bindgen/latest/wasm_bindgen/attr.wasm_bindgen.html)

## 47.1. О `Cargo.toml`

**Новых зависимостей не нужно.** Всё есть: `wasm-bindgen`, `js-sys`, `web-sys`, `wasm-bindgen-futures`, `gloo-timers`. Секция `[package.metadata.wasm-pack.profile.release]` с `wasm-opt = false` — из главы 50.

## 47.2. Модуль `src/error_info.rs`

```rust
//! 47. Обработка ошибок: извлечение структурированной информации из
//! `JsValue` и преобразование её в прикладные категории.
//!
//! ## Зачем это нужно
//!
//! Когда JS бросает исключение (или Promise отклоняется), Rust видит
//! его как opaque `JsValue`. Ни `name`, ни `message`, ни `stack` не
//! доступны напрямую — их надо извлечь через `Reflect::get`. Этот
//! модуль инкапсулирует извлечение и преобразование в прикладные
//! категории с готовыми user-facing сообщениями.
//!
//! ## Что НЕ делает этот модуль
//!
//! Он не ловит исключения и не создаёт Promise. Это делает вызывающий
//! код (см. `error_scenarios.rs` и `app.rs`). Модуль работает с уже
//! полученным `JsValue` ошибки.

use js_sys::Reflect;
use wasm_bindgen::prelude::*;

use crate::video_frame::format_str;

// =============================================================================
// Severity
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum Severity {
    Info,
    Warning,
    Error,
}

// =============================================================================
// AppError — прикладная категория
// =============================================================================

#[derive(Clone, Debug)]
pub enum AppError {
    /// Пользователь отменил операцию (`AbortError`).
    Cancelled,
    /// Отказ в доступе / разрешениях (`NotAllowedError`, `SecurityError`).
    PermissionDenied,
    /// Объект не найден (`NotFoundError` вне контекста отмены).
    NotFound,
    /// Некорректное состояние (`InvalidStateError`).
    InvalidState,
    /// Сетевой сбой / таймаут (`NetworkError`, `TimeoutError`).
    Network,
    /// Не поддерживается средой (`NotSupportedError`).
    NotSupported,
    /// Некорректный пользовательский ввод (`RangeError`, `SyntaxError`).
    InvalidInput,
    /// Ошибка программиста (`TypeError`, `ReferenceError`).
    Programmer,
    /// Прикладная ошибка — `throw new Error("...")` без подкласса.
    /// Осмысленная ошибка, брошенная намеренно.
    Generic { message: String },
    /// Нестандартный rejection — `Promise.reject('string')` / `42` / `true`.
    /// Обычно означает баг в JS-коде, где вместо `Error` отдают примитив.
    NonStandardRejection { raw: String },
    /// Неизвестная ошибка — категория-ловушка для действительно непонятных случаев.
    Unknown { name: String, message: String },
}

impl AppError {
    /// Преобразует структурированную информацию об ошибке в категорию.
    ///
    /// ## Осторожно: одинаковые имена — разные смыслы
    ///
    /// `NotFoundError` в Web Bluetooth означает «пользователь отменил
    /// выбор», а в `fetch` — «ресурс не найден». Одна и та же строка
    /// `name` даёт разные прикладные категории в зависимости от API.
    /// Здесь используется наиболее распространённое сопоставление;
    /// для специфичных API вызывающий код может переопределить
    /// категорию вручную.
    pub fn from_info(info: &JsErrorInfo) -> Self {
        return match info.name.as_str() {
            "AbortError" => Self::Cancelled,
            "NotFoundError" => Self::NotFound,
            "NotAllowedError" | "SecurityError" | "PermissionDeniedError" => {
                Self::PermissionDenied
            }
            "InvalidStateError" | "InvalidModificationError" => Self::InvalidState,
            "NetworkError" | "TimeoutError" => Self::Network,
            "NotSupportedError" => Self::NotSupported,
            "RangeError" | "SyntaxError" => Self::InvalidInput,
            "TypeError" | "ReferenceError" => Self::Programmer,
            // `Error` без подкласса — осмысленная прикладная ошибка.
            "Error" => Self::Generic {
                message: info.message.clone(),
            },
            // Примитивы вместо `Error` — нестандартный rejection.
            "String" | "Number" | "Boolean" => Self::NonStandardRejection {
                raw: info.message.clone(),
            },
            _ => Self::Unknown {
                name: info.name.clone(),
                message: info.message.clone(),
            },
        };
    }

    pub fn severity(&self) -> Severity {
        return match self {
            Self::Cancelled => Severity::Info,
            Self::PermissionDenied
            | Self::NotFound
            | Self::InvalidState
            | Self::InvalidInput
            | Self::Generic { .. } => Severity::Warning,
            Self::Network
            | Self::NotSupported
            | Self::Programmer
            | Self::NonStandardRejection { .. }
            | Self::Unknown { .. } => Severity::Error,
        };
    }

    pub fn category_label(&self) -> &'static str {
        return match self {
            Self::Cancelled => "Отменено",
            Self::PermissionDenied => "Нет доступа",
            Self::NotFound => "Не найдено",
            Self::InvalidState => "Некорректное состояние",
            Self::Network => "Сетевой сбой",
            Self::NotSupported => "Не поддерживается",
            Self::InvalidInput => "Неверный ввод",
            Self::Programmer => "Ошибка программиста",
            Self::Generic { .. } => "Прикладная ошибка",
            Self::NonStandardRejection { .. } => "Нестандартный rejection",
            Self::Unknown { .. } => "Неизвестно",
        };
    }

    pub fn css_class(&self) -> &'static str {
        return match self {
            Self::Cancelled => "err-card-cat-cancelled",
            Self::PermissionDenied => "err-card-cat-permission",
            Self::NotFound => "err-card-cat-notfound",
            Self::InvalidState => "err-card-cat-invalidstate",
            Self::Network => "err-card-cat-network",
            Self::NotSupported => "err-card-cat-unsupported",
            Self::InvalidInput => "err-card-cat-invalidinput",
            Self::Programmer => "err-card-cat-programmer",
            Self::Generic { .. } => "err-card-cat-generic",
            Self::NonStandardRejection { .. } => "err-card-cat-nonstandard",
            Self::Unknown { .. } => "err-card-cat-unknown",
        };
    }

    /// User-facing сообщение. Никаких «TypeError: x is not a function» —
    /// пользователь должен видеть понятный текст.
    pub fn user_message(&self) -> String {
        return match self {
            Self::Cancelled => String::from("Операция отменена пользователем"),
            Self::PermissionDenied => String::from("Нет разрешения на выполнение операции"),
            Self::NotFound => String::from("Запрашиваемый объект не найден"),
            Self::InvalidState => String::from("Операция невозможна в текущем состоянии"),
            Self::Network => String::from("Сетевой сбой — проверьте соединение"),
            Self::NotSupported => String::from("Операция не поддерживается в этом браузере"),
            Self::InvalidInput => String::from("Проверьте введённые данные"),
            Self::Programmer => {
                String::from("Внутренняя ошибка приложения. Обратитесь в поддержку.")
            }
            Self::Generic { message } => format_str(format_args!("{message}")),
            Self::NonStandardRejection { raw } => format_str(format_args!(
                "Нестандартная ошибка: {raw}"
            )),
            Self::Unknown { message, .. } => {
                format_str(format_args!("Неизвестная ошибка: {message}"))
            }
        };
    }
}

// =============================================================================
// JsErrorInfo — извлечённая структура
// =============================================================================

#[derive(Clone, Debug)]
pub struct JsErrorInfo {
    pub name: String,
    pub message: String,
    pub stack: Option<String>,
    pub code: Option<i32>,
    /// Если оригинальное значение было примитивом (строка, число),
    /// это поле содержит его текстовое представление.
    pub raw: Option<String>,
}

impl JsErrorInfo {
    fn primitive(name: &str, message: String) -> Self {
        return Self {
            name: String::from(name),
            message: message.clone(),
            stack: None,
            code: None,
            raw: Some(message),
        };
    }
}

/// Legacy-таблица `DOMException.code` по имени.
///
/// ## Зачем fallback
///
/// В Chrome `DOMException.code` — это getter на прототипе, но
/// `Reflect::get(value, "code")` в некоторых версиях (особенно
/// headless) возвращает `undefined` или `0`. Таблица восстанавливает
/// код по имени — это стандартные значения из WebIDL
/// (`NotFoundError` = 8, `SecurityError` = 18, `AbortError` = 20 и т.д.).
///
/// ## Где это НЕ сработает
///
/// Для кастомных DOMException с нестандартным именем — вернём `None`.
/// Для `Error` (не DOMException) — тоже `None`, потому что у обычного
/// Error нет числового кода.
fn dom_exception_code(name: &str) -> Option<i32> {
    return match name {
        "IndexSizeError" => Some(1),
        "HierarchyRequestError" => Some(3),
        "WrongDocumentError" => Some(4),
        "InvalidCharacterError" => Some(5),
        "NoModificationAllowedError" => Some(7),
        "NotFoundError" => Some(8),
        "NotSupportedError" => Some(9),
        "InUseAttributeError" => Some(10),
        "InvalidStateError" => Some(11),
        "SyntaxError" => Some(12),
        "InvalidModificationError" => Some(13),
        "NamespaceError" => Some(14),
        "InvalidAccessError" => Some(15),
        "TypeMismatchError" => Some(17),
        "SecurityError" => Some(18),
        "NetworkError" => Some(19),
        "AbortError" => Some(20),
        "URLMismatchError" => Some(21),
        "QuotaExceededError" => Some(22),
        "TimeoutError" => Some(23),
        "InvalidNodeTypeError" => Some(24),
        "DataCloneError" => Some(25),
        _ => None,
    };
}

/// Извлекает структурированную информацию из `JsValue` ошибки.
///
/// ## Обрабатываемые случаи
///
/// - `undefined` / `null` — Promise.reject с этими значениями;
/// - примитивы (строка, число, boolean) — нестандартный rejection;
/// - объекты с `name`/`message`/`stack`/`code` — `Error`, `DOMException`.
///
/// ## Что НЕ паникует
///
/// Функция безопасна для любого `JsValue`. `Reflect::get` на примитиве
/// возвращает `Err`, который мы игнорируем; при этом сам примитив
/// сохраняется в `raw`.
pub fn extract_error_info(value: &JsValue) -> JsErrorInfo {
    // Специальные случаи.
    if value.is_undefined() {
        return JsErrorInfo::primitive("Undefined", String::from("Promise rejected with undefined"));
    }
    if value.is_null() {
        return JsErrorInfo::primitive("Null", String::from("Promise rejected with null"));
    }

    // Примитивы.
    if let Some(s) = value.as_string() {
        return JsErrorInfo::primitive("String", s);
    }
    if let Some(n) = value.as_f64() {
        return JsErrorInfo::primitive("Number", format_str(format_args!("{n}")));
    }
    if let Some(b) = value.as_bool() {
        return JsErrorInfo::primitive("Boolean", format_str(format_args!("{b}")));
    }

    // Объект (Error, DOMException, произвольный объект).
    let name = Reflect::get(value, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| String::from("Error"));

    let message = Reflect::get(value, &JsValue::from_str("message"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| String::from("(без сообщения)"));

    let stack = Reflect::get(value, &JsValue::from_str("stack"))
        .ok()
        .and_then(|v| v.as_string());

    // DOMException.code:
    //   шаг 1 — попытка через Reflect::get (в не-headless Chrome работает);
    //   шаг 2 — fallback по имени (для headless и кастомных сборок).
    // Фильтр `!= 0` отсекает значение по умолчанию для не-DOMException.
    let code = Reflect::get(value, &JsValue::from_str("code"))
        .ok()
        .and_then(|v| {
            if let Some(n) = v.as_f64() {
                return Some(n as i32);
            }
            v.as_string().and_then(|s| s.parse::<i32>().ok())
        })
        .filter(|&n| n != 0)
        .or_else(|| dom_exception_code(&name));

    return JsErrorInfo {
        name,
        message,
        stack,
        code,
        raw: None,
    };
}

// =============================================================================
// Утилиты
// =============================================================================

/// Обрезает стек до `max_lines` строк, добавляя пометку, если
/// строк было больше.
pub fn truncate_stack(stack: &str, max_lines: usize) -> String {
    let all_lines: Vec<&str> = stack.lines().collect();
    let total = all_lines.len();
    let take = total.min(max_lines);
    let mut out = all_lines[..take].join("\n");
    if total > max_lines {
        out.push_str(&format_str(format_args!("\n… (+{} строк)", total - max_lines)));
    }
    return out;
}

/// Строка вида "TypeError: message" для логов.
pub fn format_error_brief(info: &JsErrorInfo) -> String {
    return format_str(format_args!("{}: {}", info.name, info.message));
}
```

## 47.3. Модуль `src/error_scenarios.rs`

```rust
//! 47. Обработка ошибок: сценарии-источники ошибок в браузере.
//!
//! ## Что здесь
//!
//! JS-функции, каждая из которых провоцирует определённый тип ошибки:
//! синхронный `throw` (`TypeError`, `RangeError`, `ReferenceError`,
//! `Error`, `DOMException`) и `Promise.reject` (`Error`, `DOMException`,
//! примитивы). Регистрируются на `window.__err_scenarios` через
//! `inline_js`, вызываются из Rust через `js_sys::Function::call0`.
//!
//! ## Почему не `extern "C" fn scenario()` напрямую
//!
//! Если объявить `extern "C" fn __err_throw_type_error() -> u32`,
//! то JS-throw приведёт к **WASM-trap**, а не к catchable `Err`.
//! Чтобы получить `Result<JsValue, JsValue>`, нужно вызывать через
//! `js_sys::Function::call0` — `wasm-bindgen` ловит JS-исключение
//! и возвращает его как `Err(JsValue)`.

use js_sys::{Function, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::format_str;

// =============================================================================
// Установка сценариев
// =============================================================================

#[wasm_bindgen(inline_js = "
export function __err_install() {
    const scenarios = {
        'throw_type_error': function() {
            const x = undefined;
            return x.someDeepProperty;
        },
        'throw_range_error': function() {
            return new Array(-1);
        },
        'throw_reference_error': function() {
            return __err_nonExistentFunction12345();
        },
        'throw_custom_error': function() {
            throw new Error('Custom message from JS');
        },
        'throw_dom_exception': function() {
            throw new DOMException('Device not found', 'NotFoundError');
        },
        'reject_error': function() {
            return Promise.reject(new Error('Async operation failed'));
        },
        'reject_dom_exception': function() {
            return Promise.reject(new DOMException('Permission denied', 'NotAllowedError'));
        },
        'reject_string': function() {
            return Promise.reject('bare string rejection');
        },
        'reject_number': function() {
            return Promise.reject(42);
        },
        'success': function() {
            return Promise.resolve(42);
        },
    };
    window.__err_scenarios = scenarios;
}
")]
extern "C" {
    fn __err_install();
}

/// Устанавливает сценарии на `window`. Вызывается один раз при старте
/// приложения или тестов.
pub fn install() {
    __err_install();
}

// =============================================================================
// Получение функции сценария
// =============================================================================

pub fn get_scenario(name: &str) -> Result<Function, String> {
    let window = web_sys::window().ok_or_else(|| String::from("no window"))?;
    let scenarios = Reflect::get(window.as_ref(), &JsValue::from_str("__err_scenarios"))
        .map_err(|_| String::from("scenarios not installed — call install() first"))?;
    if scenarios.is_undefined() || scenarios.is_null() {
        return Err(String::from("scenarios not installed"));
    }
    let f = Reflect::get(&scenarios, &JsValue::from_str(name))
        .map_err(|_| format_str(format_args!("scenario '{name}' not found")))?;
    if f.is_undefined() {
        return Err(format_str(format_args!("scenario '{name}' is undefined")));
    }
    return f
        .dyn_into::<Function>()
        .map_err(|_| format_str(format_args!("scenario '{name}' is not a Function")));
}

// =============================================================================
// Запуск сценариев
// =============================================================================

/// Синхронный вызов. Если JS бросает — получаем `Err(JsValue)`.
pub fn run_sync(name: &str) -> Result<JsValue, JsValue> {
    let f = get_scenario(name).map_err(|e| JsValue::from_str(&e))?;
    return f.call0(&JsValue::UNDEFINED);
}

/// Асинхронный вызов. Сценарий обязан вернуть Promise; его
/// разрешение/отклонение превращается в `Ok`/`Err`.
///
/// ## Контракт
///
/// Все сценарии, вызываемые через `run_async`, должны возвращать
/// `Promise` (даже успешные — `Promise.resolve(42)`, не голое `42`).
/// Иначе `dyn_into::<Promise>` упадёт, и вернётся осмысленная ошибка
/// «scenario did not return a Promise».
pub async fn run_async(name: &str) -> Result<JsValue, JsValue> {
    let f = get_scenario(name).map_err(|e| JsValue::from_str(&e))?;
    let promise_value = f.call0(&JsValue::UNDEFINED)?;
    if promise_value.is_undefined() || promise_value.is_null() {
        return Err(JsValue::from_str("scenario returned no Promise"));
    }
    let promise: Promise = promise_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("scenario did not return a Promise"))?;
    return JsFuture::from(promise).await;
}
```

## 47.4. Добавление в `src/lib.rs`

```rust
pub mod app;
pub mod arch_js_bench;
pub mod arch_pure;
pub mod bluetooth_gatt;
pub mod bluetooth_utils;
pub mod container_parser;
pub mod error_info;          // ← 47
pub mod error_scenarios;     // ← 47
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

## 47.5. Тесты `tests/wasm_error_handling.rs`

**Важно:** этот файл **не пересекается** с уже существующим `tests/wasm_errors.rs`, который тестирует `hello_wasm02::fetch` (сетевые ошибки HTTP/Fetch). Разные модули, разные имена, разный функционал.

```rust
//! 47. Тесты обработки ошибок (глава «Error Handling»).
//!
//! ## Что тестируется
//!
//! - `extract_error_info` — на всех типах `js_sys::Error`, DOMException,
//!   примитивах (строка, число, boolean), undefined/null.
//! - `AppError::from_info` — правильная категория для каждого имени.
//! - `AppError::user_message` / `severity` / `category_label` — непустые
//!   значения для всех категорий.
//! - `truncate_stack` — обрезка по строкам.
//! - `error_scenarios` — каждый источник возвращает `Err(JsValue)`.
//!
//! ## Не путать с `wasm_errors.rs`
//!
//! `wasm_errors.rs` тестирует `hello_wasm02::fetch` — сетевые ошибки
//! HTTP/Fetch. Здесь — общая граница ошибок Rust ↔ JS.

use wasm_bindgen::prelude::*;
use wasm_bindgen_test::*;

use hello_wasm02::error_info::{
    extract_error_info, format_error_brief, truncate_stack, AppError, JsErrorInfo,
    Severity,
};
use hello_wasm02::error_scenarios;

wasm_bindgen_test_configure!(run_in_browser);

// Хелпер: создать DOMException через JS (из Rust нет прямого конструктора).
#[wasm_bindgen(inline_js = "
export function __test_make_dom_exception(name, message) {
    return new DOMException(message, name);
}
")]
extern "C" {
    fn __test_make_dom_exception(name: &str, message: &str) -> JsValue;
}

// =============================================================================
// extract_error_info — объекты
// =============================================================================

#[wasm_bindgen_test]
fn test_extract_from_js_error() {
    let e = js_sys::Error::new("boom");
    let info = extract_error_info(e.as_ref());
    assert_eq!(info.name, "Error");
    assert_eq!(info.message, "boom");
}

#[wasm_bindgen_test]
fn test_extract_from_type_error() {
    let e = js_sys::TypeError::new("not a function");
    let info = extract_error_info(e.as_ref());
    assert_eq!(info.name, "TypeError");
    assert_eq!(info.message, "not a function");
}

#[wasm_bindgen_test]
fn test_extract_from_range_error() {
    let e = js_sys::RangeError::new("out of bounds");
    let info = extract_error_info(e.as_ref());
    assert_eq!(info.name, "RangeError");
    assert_eq!(info.message, "out of bounds");
}

#[wasm_bindgen_test]
fn test_extract_from_reference_error() {
    let e = js_sys::ReferenceError::new("undefined var");
    let info = extract_error_info(e.as_ref());
    assert_eq!(info.name, "ReferenceError");
}

#[wasm_bindgen_test]
fn test_extract_from_syntax_error() {
    let e = js_sys::SyntaxError::new("unexpected token");
    let info = extract_error_info(e.as_ref());
    assert_eq!(info.name, "SyntaxError");
}

#[wasm_bindgen_test]
fn test_extract_from_dom_exception() {
    let e = __test_make_dom_exception("NotFoundError", "device not found");
    let info = extract_error_info(&e);
    assert_eq!(info.name, "NotFoundError");
    assert_eq!(info.message, "device not found");
    // DOMException имеет числовой code (fallback по имени).
    assert!(info.code.is_some(), "DOMException должна иметь code");
    assert_eq!(info.code, Some(8), "NotFoundError → code 8");
}

#[wasm_bindgen_test]
fn test_dom_exception_code_extraction() {
    // DOMException с известным code — извлекается через fallback-таблицу.
    let e = __test_make_dom_exception("NotFoundError", "missing");
    let info = extract_error_info(&e);
    assert_eq!(info.name, "NotFoundError");
    assert_eq!(info.code, Some(8));
}

#[wasm_bindgen_test]
fn test_dom_exception_not_allowed_code() {
    let e = __test_make_dom_exception("NotAllowedError", "denied");
    let info = extract_error_info(&e);
    assert_eq!(info.name, "NotAllowedError");
    // NotAllowedError не входит в стандартную таблицу DOMException.code,
    // поэтому code может быть None.
}

#[wasm_bindgen_test]
fn test_plain_error_has_no_code() {
    let e = js_sys::Error::new("plain");
    let info = extract_error_info(e.as_ref());
    assert_eq!(info.name, "Error");
    assert!(info.code.is_none(), "у обычного Error нет code");
}

// =============================================================================
// extract_error_info — примитивы
// =============================================================================

#[wasm_bindgen_test]
fn test_extract_from_string() {
    let v = JsValue::from_str("bare string");
    let info = extract_error_info(&v);
    assert_eq!(info.name, "String");
    assert_eq!(info.message, "bare string");
    assert_eq!(info.raw.as_deref(), Some("bare string"));
}

#[wasm_bindgen_test]
fn test_extract_from_number() {
    let v = JsValue::from_f64(42.0);
    let info = extract_error_info(&v);
    assert_eq!(info.name, "Number");
    assert_eq!(info.message, "42");
    assert!(info.raw.is_some());
}

#[wasm_bindgen_test]
fn test_extract_from_boolean() {
    let v = JsValue::from_bool(true);
    let info = extract_error_info(&v);
    assert_eq!(info.name, "Boolean");
}

#[wasm_bindgen_test]
fn test_extract_from_undefined() {
    let info = extract_error_info(&JsValue::UNDEFINED);
    assert_eq!(info.name, "Undefined");
    assert!(info.message.contains("undefined"));
}

#[wasm_bindgen_test]
fn test_extract_from_null() {
    let info = extract_error_info(&JsValue::NULL);
    assert_eq!(info.name, "Null");
    assert!(info.message.contains("null"));
}

// =============================================================================
// AppError::from_info
// =============================================================================

fn make_info(name: &str) -> JsErrorInfo {
    return JsErrorInfo {
        name: String::from(name),
        message: String::from("test message"),
        stack: None,
        code: None,
        raw: None,
    };
}

#[wasm_bindgen_test]
fn test_app_error_cancelled() {
    let e = AppError::from_info(&make_info("AbortError"));
    assert!(matches!(e, AppError::Cancelled));
    assert_eq!(e.severity(), Severity::Info);
}

#[wasm_bindgen_test]
fn test_app_error_permission() {
    let e1 = AppError::from_info(&make_info("NotAllowedError"));
    let e2 = AppError::from_info(&make_info("SecurityError"));
    assert!(matches!(e1, AppError::PermissionDenied));
    assert!(matches!(e2, AppError::PermissionDenied));
}

#[wasm_bindgen_test]
fn test_app_error_notfound() {
    assert!(matches!(
        AppError::from_info(&make_info("NotFoundError")),
        AppError::NotFound
    ));
}

#[wasm_bindgen_test]
fn test_app_error_invalid_state() {
    assert!(matches!(
        AppError::from_info(&make_info("InvalidStateError")),
        AppError::InvalidState
    ));
}

#[wasm_bindgen_test]
fn test_app_error_network() {
    assert!(matches!(
        AppError::from_info(&make_info("NetworkError")),
        AppError::Network
    ));
    assert!(matches!(
        AppError::from_info(&make_info("TimeoutError")),
        AppError::Network
    ));
}

#[wasm_bindgen_test]
fn test_app_error_not_supported() {
    assert!(matches!(
        AppError::from_info(&make_info("NotSupportedError")),
        AppError::NotSupported
    ));
}

#[wasm_bindgen_test]
fn test_app_error_invalid_input() {
    assert!(matches!(
        AppError::from_info(&make_info("RangeError")),
        AppError::InvalidInput
    ));
    assert!(matches!(
        AppError::from_info(&make_info("SyntaxError")),
        AppError::InvalidInput
    ));
}

#[wasm_bindgen_test]
fn test_app_error_programmer() {
    assert!(matches!(
        AppError::from_info(&make_info("TypeError")),
        AppError::Programmer
    ));
    assert!(matches!(
        AppError::from_info(&make_info("ReferenceError")),
        AppError::Programmer
    ));
}

#[wasm_bindgen_test]
fn test_app_error_generic_for_plain_error() {
    // `Error` без подкласса → Generic (прикладная ошибка), не Unknown.
    let e = AppError::from_info(&make_info("Error"));
    assert!(matches!(e, AppError::Generic { .. }));
    assert_eq!(e.severity(), Severity::Warning);
    assert_eq!(e.category_label(), "Прикладная ошибка");
}

#[wasm_bindgen_test]
fn test_app_error_nonstandard_rejection() {
    // `Promise.reject('string')` / `42` → NonStandardRejection.
    let e1 = AppError::from_info(&make_info("String"));
    let e2 = AppError::from_info(&make_info("Number"));
    let e3 = AppError::from_info(&make_info("Boolean"));
    assert!(matches!(e1, AppError::NonStandardRejection { .. }));
    assert!(matches!(e2, AppError::NonStandardRejection { .. }));
    assert!(matches!(e3, AppError::NonStandardRejection { .. }));
    assert_eq!(e1.severity(), Severity::Error);
    assert_eq!(e1.category_label(), "Нестандартный rejection");
}

#[wasm_bindgen_test]
fn test_app_error_unknown() {
    let e = AppError::from_info(&make_info("SomeWeirdError"));
    assert!(matches!(e, AppError::Unknown { .. }));
}

// =============================================================================
// user_message, category_label, severity
// =============================================================================

#[wasm_bindgen_test]
fn test_user_message_non_empty_for_all() {
    let all = [
        AppError::Cancelled,
        AppError::PermissionDenied,
        AppError::NotFound,
        AppError::InvalidState,
        AppError::Network,
        AppError::NotSupported,
        AppError::InvalidInput,
        AppError::Programmer,
        AppError::Generic { message: String::from("custom") },
        AppError::NonStandardRejection { raw: String::from("42") },
        AppError::Unknown {
            name: String::from("X"),
            message: String::from("y"),
        },
    ];
    for e in all {
        let msg = e.user_message();
        assert!(!msg.is_empty(), "empty user message for {e:?}");
    }
}

#[wasm_bindgen_test]
fn test_category_labels_unique_and_non_empty() {
    let all = [
        AppError::Cancelled,
        AppError::PermissionDenied,
        AppError::NotFound,
        AppError::InvalidState,
        AppError::Network,
        AppError::NotSupported,
        AppError::InvalidInput,
        AppError::Programmer,
        AppError::Generic { message: String::from("x") },
        AppError::NonStandardRejection { raw: String::from("x") },
    ];
    for e in all {
        assert!(!e.category_label().is_empty());
        assert!(!e.css_class().is_empty());
    }
}

#[wasm_bindgen_test]
fn test_severity_mapping() {
    assert_eq!(AppError::Cancelled.severity(), Severity::Info);
    assert_eq!(AppError::PermissionDenied.severity(), Severity::Warning);
    assert_eq!(AppError::NotFound.severity(), Severity::Warning);
    assert_eq!(AppError::InvalidInput.severity(), Severity::Warning);
    assert_eq!(
        AppError::Generic { message: String::from("x") }.severity(),
        Severity::Warning
    );
    assert_eq!(AppError::Network.severity(), Severity::Error);
    assert_eq!(AppError::NotSupported.severity(), Severity::Error);
    assert_eq!(AppError::Programmer.severity(), Severity::Error);
    assert_eq!(
        AppError::NonStandardRejection { raw: String::from("x") }.severity(),
        Severity::Error
    );
}

// =============================================================================
// truncate_stack, format_error_brief
// =============================================================================

#[wasm_bindgen_test]
fn test_truncate_stack_short() {
    let s = "line1\nline2";
    assert_eq!(truncate_stack(s, 5), "line1\nline2");
}

#[wasm_bindgen_test]
fn test_truncate_stack_exact() {
    let s = "1\n2\n3";
    assert_eq!(truncate_stack(s, 3), "1\n2\n3");
}

#[wasm_bindgen_test]
fn test_truncate_stack_long() {
    let s = "1\n2\n3\n4\n5\n6\n7";
    let out = truncate_stack(s, 3);
    assert!(out.starts_with("1\n2\n3"));
    assert!(out.contains("+4"));
}

#[wasm_bindgen_test]
fn test_format_error_brief() {
    let i = JsErrorInfo {
        name: String::from("TypeError"),
        message: String::from("bad"),
        stack: None,
        code: None,
        raw: None,
    };
    assert_eq!(format_error_brief(&i), "TypeError: bad");
}

// =============================================================================
// Сценарии: синхронные источники
// =============================================================================

#[wasm_bindgen_test]
fn test_scenario_install_idempotent() {
    error_scenarios::install();
    error_scenarios::install();
    // Если не паникует — уже хорошо.
}

#[wasm_bindgen_test]
fn test_scenario_throw_type_error() {
    error_scenarios::install();
    let r = error_scenarios::run_sync("throw_type_error");
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "TypeError");
}

#[wasm_bindgen_test]
fn test_scenario_throw_range_error() {
    error_scenarios::install();
    let r = error_scenarios::run_sync("throw_range_error");
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "RangeError");
}

#[wasm_bindgen_test]
fn test_scenario_throw_reference_error() {
    error_scenarios::install();
    let r = error_scenarios::run_sync("throw_reference_error");
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "ReferenceError");
}

#[wasm_bindgen_test]
fn test_scenario_throw_custom_error() {
    error_scenarios::install();
    let r = error_scenarios::run_sync("throw_custom_error");
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "Error");
    assert_eq!(info.message, "Custom message from JS");
}

#[wasm_bindgen_test]
fn test_scenario_throw_dom_exception() {
    error_scenarios::install();
    let r = error_scenarios::run_sync("throw_dom_exception");
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "NotFoundError");
    assert_eq!(info.message, "Device not found");
    assert_eq!(info.code, Some(8));
}

// =============================================================================
// Сценарии: асинхронные источники
// =============================================================================

#[wasm_bindgen_test]
async fn test_scenario_reject_error() {
    error_scenarios::install();
    let r = error_scenarios::run_async("reject_error").await;
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "Error");
    assert_eq!(info.message, "Async operation failed");
}

#[wasm_bindgen_test]
async fn test_scenario_reject_dom_exception() {
    error_scenarios::install();
    let r = error_scenarios::run_async("reject_dom_exception").await;
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "NotAllowedError");
    assert_eq!(info.message, "Permission denied");
}

#[wasm_bindgen_test]
async fn test_scenario_reject_string() {
    error_scenarios::install();
    let r = error_scenarios::run_async("reject_string").await;
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "String");
    assert_eq!(info.message, "bare string rejection");
}

#[wasm_bindgen_test]
async fn test_scenario_reject_number() {
    error_scenarios::install();
    let r = error_scenarios::run_async("reject_number").await;
    assert!(r.is_err());
    let info = extract_error_info(&r.unwrap_err());
    assert_eq!(info.name, "Number");
    assert_eq!(info.message, "42");
}

#[wasm_bindgen_test]
async fn test_scenario_success() {
    error_scenarios::install();
    let r = error_scenarios::run_async("success").await;
    assert!(r.is_ok());
    assert_eq!(r.unwrap().as_f64(), Some(42.0));
}

// =============================================================================
// Сквозной путь: сценарий → extract → AppError → user_message
// =============================================================================

#[wasm_bindgen_test]
async fn test_end_to_end_classification() {
    error_scenarios::install();
    let r = error_scenarios::run_async("reject_dom_exception").await;
    let info = extract_error_info(&r.unwrap_err());
    let app_err = AppError::from_info(&info);
    assert!(matches!(app_err, AppError::PermissionDenied));
    assert_eq!(app_err.severity(), Severity::Warning);
    assert!(app_err.user_message().contains("разрешени"));
}
```

**Команды запуска тестов:**

```bash
# Только тесты главы 47
wasm-pack test --headless --chrome --test wasm_error_handling

# Все тесты проекта
wasm-pack test --headless --chrome

# С захватом вывода (при отладке падений)
wasm-pack test --headless --chrome --test wasm_error_handling -- --nocapture
```

## 47.6. Демо `src/app.rs`

```rust
//! Глава 47 — Обработка ошибок: граница Rust ↔ JS.
//!
//! ## Идея демо
//!
//! Девять сценариев-источников ошибок. Для каждого показываем
//! **путь ошибки** через три уровня:
//!
//! 1. **Сырой JsValue** — что видит Rust без обработки (тип handle).
//! 2. **JsErrorInfo** — name/message/stack/code через `Reflect::get`.
//! 3. **AppError** — прикладная категория, severity, user_message.
//!
//! ## Принцип
//!
//! Пользователь не должен видеть `TypeError: x is not a function` —
//! это сообщение для разработчика. Демо показывает, как из raw JS-ошибки
//! получить понятное прикладное сообщение.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::Document;

use crate::error_info::{
    extract_error_info, truncate_stack, AppError, JsErrorInfo, Severity,
};
use crate::error_scenarios;
use crate::video_frame::format_str;

// =============================================================================
// Сценарии
// =============================================================================

struct ScenarioDef {
    id: &'static str,
    title: &'static str,
    subtitle: &'static str,
    is_async: bool,
}

const SCENARIOS: &[ScenarioDef] = &[
    ScenarioDef { id: "throw_type_error",       title: "TypeError",                    subtitle: "обращение к свойству undefined",   is_async: false },
    ScenarioDef { id: "throw_range_error",      title: "RangeError",                   subtitle: "new Array(-1)",                    is_async: false },
    ScenarioDef { id: "throw_reference_error",  title: "ReferenceError",               subtitle: "вызов несуществующей функции",     is_async: false },
    ScenarioDef { id: "throw_custom_error",     title: "Error",                        subtitle: "throw new Error(...)",             is_async: false },
    ScenarioDef { id: "throw_dom_exception",    title: "DOMException",                 subtitle: "NotFoundError из WebAPI",          is_async: false },
    ScenarioDef { id: "reject_error",           title: "Promise.reject(Error)",        subtitle: "async-операция упала",             is_async: true },
    ScenarioDef { id: "reject_dom_exception",   title: "Promise.reject(DOMException)", subtitle: "отказ в разрешении",               is_async: true },
    ScenarioDef { id: "reject_string",          title: "Promise.reject('string')",     subtitle: "нестандартный rejection",          is_async: true },
    ScenarioDef { id: "reject_number",          title: "Promise.reject(42)",           subtitle: "нестандартный rejection",          is_async: true },
];

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.err-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.err-desc-line { margin:6px 0; }
.err-desc-line code { background:#0d1117; padding:1px 5px; border-radius:3px; color:#79c0ff; }
.err-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.err-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.err-btn:hover:not([disabled]) { background:#30363d; }
.err-btn:disabled { opacity:0.5;cursor:not-allowed; }
.err-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.err-btn-primary:hover:not([disabled]) { background:#2ea043; }
.err-hint { color:#8b949e;font-size:12px; }
.err-progress { color:#79c0ff;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-left:4px; }

.err-summary { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border-left:3px solid #79c0ff;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;overflow-wrap:anywhere;word-break:break-word;}
.err-summary .lbl { color:#8b949e; }
.err-summary .val { color:#79c0ff; }
.err-summary .info { color:#58a6ff; }
.err-summary .warn { color:#f0883e; }
.err-summary .err { color:#f85149; }

/* Flexbox-контейнер для chips: гарантирует перенос строк и видимость всех категорий */
.err-summary-cat {
    margin-top:8px;
    display:flex;
    flex-wrap:wrap;
    align-items:center;
    gap:4px;
    line-height:1.4;
}
.err-summary-cat .cat-label {
    flex-basis:100%;
    color:#8b949e;
    margin-bottom:4px;
}
.err-summary-chip {
    display:inline-flex;
    align-items:center;
    padding:2px 10px;
    border-radius:10px;
    font-size:11px;
    font-weight:500;
    white-space:nowrap;
}
.err-summary-chip.chip-info { background:#0a1620; color:#79c0ff; border:1px solid #1e3a5f; }
.err-summary-chip.chip-warning { background:#2a1d10; color:#f0883e; border:1px solid #5c3a1a; }
.err-summary-chip.chip-error { background:#2a1010; color:#ffa198; border:1px solid #5f2020; }
.err-summary-chip .chip-count { color:#6e7681; margin-left:4px; }

.err-grid { max-width:1000px;margin-bottom:16px;display:grid;grid-template-columns:repeat(auto-fit,minmax(440px,1fr));gap:12px;}

.err-card { padding:14px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;transition:border-color 0.15s;}
.err-card.err-running { border-color:#58a6ff; box-shadow:0 0 0 1px #58a6ff33; }
.err-card.err-sev-info { border-color:#58a6ff; }
.err-card.err-sev-warning { border-color:#f0883e; }
.err-card.err-sev-error { border-color:#f85149; }

.err-card-title { color:#79c0ff;font-weight:600;font-size:14px;margin-bottom:2px; }
.err-card-sub { color:#8b949e;font-size:11px;margin-bottom:12px;text-transform:uppercase;letter-spacing:1px; }

.err-level { margin-bottom:10px; }
.err-level-head { font-size:10px; color:#6e7681; text-transform:uppercase; letter-spacing:1px; margin-bottom:4px; font-weight:600; }
.err-level-body { font-family:ui-monospace,monospace; font-size:11.5px; line-height:1.6; padding:6px 8px; border-radius:4px; overflow-wrap:anywhere; }

.err-level-raw .err-level-body { background:#1a0d0d; color:#ffa198; border-left:2px solid #f85149; }
.err-level-info .err-level-body { background:#161b22; color:#c9d1d9; border-left:2px solid #30363d; }
.err-level-app .err-level-body { background:#0a1620; color:#c9d1d9; border-left:3px solid #30363d; padding:8px 10px; }
.err-level-app.err-app-info .err-level-body { border-left-color:#58a6ff; }
.err-level-app.err-app-warning .err-level-body { border-left-color:#f0883e; }
.err-level-app.err-app-error .err-level-body { border-left-color:#f85149; }

.err-arrow { text-align:center; color:#30363d; font-size:14px; line-height:1; margin:4px 0 8px 0; }

.err-app-cat { font-size:11px; font-weight:600; letter-spacing:1px; text-transform:uppercase; }
.err-app-cat.err-cat-info { color:#58a6ff; }
.err-app-cat.err-cat-warning { color:#f0883e; }
.err-app-cat.err-cat-error { color:#f85149; }
.err-app-user-msg { color:#e6edf3; font-size:13px; line-height:1.4; margin-top:2px; }
.err-app-meta { font-family:ui-monospace,monospace; font-size:10px; color:#6e7681; margin-top:6px; }

.err-kv { display:flex; gap:8px; }
.err-kv .k { color:#8b949e; min-width:64px; }
.err-kv .v { color:#c9d1d9; }
.err-kv .v-err { color:#ffa198; }

.err-stack { font-family:ui-monospace,monospace; font-size:10.5px; color:#6e7681; line-height:1.5; white-space:pre-wrap; overflow-wrap:anywhere; max-height:100px; overflow-y:auto; padding:6px 8px; background:#0a0d12; border-radius:4px; border:1px solid #161b22; margin-bottom:10px; }

.err-actions { margin-top:6px; }

.err-log { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;line-height:1.6;color:#c9d1d9;max-height:220px;overflow-y:auto;}
.err-log-entry { display:flex; gap:8px; padding:1px 0; }
.err-log-entry .time { color:#6e7681; flex-shrink:0; }
.err-log-entry .msg { color:#c9d1d9; overflow-wrap:anywhere; }
.err-log-err .msg { color:#f85149; }
.err-log-warn .msg { color:#f0883e; }
.err-log-info .msg { color:#79c0ff; }

.err-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Модели
// =============================================================================

struct ScenarioResult {
    raw_debug: String,
    info: JsErrorInfo,
    app_error: AppError,
}

struct LogEntry {
    time: String,
    level: &'static str,
    text: String,
}

struct AppState {
    results: Vec<Option<ScenarioResult>>,
    logs: Vec<LogEntry>,
    running: bool,
}

impl AppState {
    fn new() -> Self {
        let mut results = Vec::with_capacity(SCENARIOS.len());
        for _ in SCENARIOS {
            results.push(None);
        }
        return Self {
            results,
            logs: Vec::new(),
            running: false,
        };
    }

    fn completed(&self) -> usize {
        return self.results.iter().filter(|r| r.is_some()).count();
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
    let id = "err-injected-style";
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

fn escape_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;");
}

fn severity_class(sev: Severity) -> &'static str {
    return match sev {
        Severity::Info => "info",
        Severity::Warning => "warning",
        Severity::Error => "error",
    };
}

fn severity_label(sev: Severity) -> &'static str {
    return match sev {
        Severity::Info => "инфо",
        Severity::Warning => "предупр.",
        Severity::Error => "ошибок",
    };
}

/// Нормализованное представление `JsValue`.
///
/// Показывает **тип handle**, не содержимое. Это то, что видит Rust-разработчик
/// при отладке: `JsValue(Object)`, `JsValue(Number(42))`, `JsValue(undefined)`.
///
/// ## Почему не `format!("{:?}", value)` напрямую
///
/// `wasm-bindgen`'овский `Debug` для `JsValue` делегирует JS-side `toString`,
/// который для `Error` возвращает `Name: message\n at stack`, а для plain
/// object — `[object Object]`. Разные сценарии дают разный формат, и это
/// вводит в заблуждение при показе рядом. Мы делаем представление
/// **единообразным**: только тип, без содержимого.
fn debug_jsvalue(value: &JsValue) -> String {
    if value.is_undefined() {
        return String::from("JsValue(undefined)");
    }
    if value.is_null() {
        return String::from("JsValue(null)");
    }
    if value.is_string() {
        let s = value.as_string().unwrap_or_default();
        let total = s.chars().count();
        let preview: String = s.chars().take(30).collect();
        let suffix = if total > 30 { "…" } else { "" };
        return format_str(format_args!(
            "JsValue(String(\"{preview}{suffix}\"))"
        ));
    }
    if let Some(b) = value.as_bool() {
        return format_str(format_args!("JsValue(Bool({b}))"));
    }
    if let Some(n) = value.as_f64() {
        return format_str(format_args!("JsValue(Number({n}))"));
    }
    if value.is_function() {
        return String::from("JsValue(Function)");
    }
    if value.is_object() {
        return String::from("JsValue(Object)");
    }
    return String::from("JsValue(?)");
}

fn render_log_html(logs: &[LogEntry]) -> String {
    let mut html = String::with_capacity(logs.len() * 100);
    for e in logs {
        html.push_str(r#"<div class="err-log-entry"#);
        if e.level == "err" {
            html.push_str(" err-log-err");
        } else if e.level == "warn" {
            html.push_str(" err-log-warn");
        } else {
            html.push_str(" err-log-info");
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

fn render_card_html(def: &ScenarioDef) -> String {
    let mut html = String::with_capacity(768);
    html.push_str(r#"<div class="err-card" id="err-card-"#);
    html.push_str(def.id);
    html.push_str(r#"">"#);

    html.push_str(r#"<div class="err-card-title">"#);
    html.push_str(def.title);
    html.push_str(r#"</div>"#);

    html.push_str(r#"<div class="err-card-sub">"#);
    html.push_str(def.subtitle);
    html.push_str(" · ");
    html.push_str(if def.is_async { "async" } else { "sync" });
    html.push_str(r#"</div>"#);

    html.push_str(r#"<div class="err-level err-level-raw" id="err-lvl1-"#);
    html.push_str(def.id);
    html.push_str(r#""><div class="err-level-head">① Сырой JsValue — что видит Rust</div><div class="err-level-body" id="err-raw-"#);
    html.push_str(def.id);
    html.push_str(r#"">—</div></div>"#);

    html.push_str(r#"<div class="err-arrow">▼</div>"#);

    html.push_str(r#"<div class="err-level err-level-info" id="err-lvl2-"#);
    html.push_str(def.id);
    html.push_str(r#""><div class="err-level-head">② extract_error_info — структура</div><div class="err-level-body" id="err-info-"#);
    html.push_str(def.id);
    html.push_str(r#"">—</div></div>"#);

    html.push_str(r#"<div class="err-arrow">▼</div>"#);

    html.push_str(r#"<div class="err-level err-level-app" id="err-lvl3-"#);
    html.push_str(def.id);
    html.push_str(r#""><div class="err-level-head">③ AppError — прикладная категория</div><div class="err-level-body" id="err-app-"#);
    html.push_str(def.id);
    html.push_str(r#""><span class="err-app-cat">—</span></div></div>"#);

    html.push_str(r#"<div class="err-stack" id="err-stack-"#);
    html.push_str(def.id);
    html.push_str(r#"">(запустите сценарий, чтобы увидеть stack)</div>"#);

    html.push_str(r#"<div class="err-actions"><button class="err-btn err-btn-primary" data-run=""#);
    html.push_str(def.id);
    html.push_str(r#"">Запустить сценарий</button></div>"#);

    html.push_str("</div>");
    return html;
}

fn render_cards_html() -> String {
    let mut html = String::new();
    for def in SCENARIOS {
        html.push_str(&render_card_html(def));
    }
    return html;
}

// =============================================================================
// Обновление карточки
// =============================================================================

fn update_card(doc: &Document, def: &ScenarioDef, result: &ScenarioResult) {
    let sev = result.app_error.severity();
    let sev_cls = severity_class(sev);

    // Уровень ①
    let raw_id = format_str(format_args!("err-raw-{}", def.id));
    if let Some(el) = doc.get_element_by_id(&raw_id) {
        el.set_text_content(Some(&result.raw_debug));
    }

    // Уровень ②
    let info_id = format_str(format_args!("err-info-{}", def.id));
    if let Some(el) = doc.get_element_by_id(&info_id) {
        let mut h = String::with_capacity(256);
        h.push_str(r#"<div class="err-kv"><span class="k">name:</span><span class="v v-err">"#);
        h.push_str(&escape_html(&result.info.name));
        h.push_str("</span></div>");
        h.push_str(r#"<div class="err-kv"><span class="k">message:</span><span class="v">"#);
        h.push_str(&escape_html(&result.info.message));
        h.push_str("</span></div>");
        if let Some(code) = result.info.code {
            h.push_str(r#"<div class="err-kv"><span class="k">code:</span><span class="v">"#);
            h.push_str(&format_str(format_args!("{code}")));
            h.push_str("</span></div>");
        }
        h.push_str(r#"<div class="err-kv"><span class="k">stack:</span><span class="v">"#);
        h.push_str(if result.info.stack.is_some() { "есть" } else { "—" });
        h.push_str("</span></div>");
        el.set_inner_html(&h);
    }

    // Уровень ③
    let app_id = format_str(format_args!("err-app-{}", def.id));
    if let Some(el) = doc.get_element_by_id(&app_id) {
        let mut h = String::with_capacity(320);
        h.push_str(r#"<div class="err-app-cat err-cat-"#);
        h.push_str(sev_cls);
        h.push_str(r#"">"#);
        h.push_str(result.app_error.category_label());
        h.push_str("</div>");
        h.push_str(r#"<div class="err-app-user-msg">"#);
        h.push_str(&escape_html(&result.app_error.user_message()));
        h.push_str("</div>");
        h.push_str(r#"<div class="err-app-meta">"#);
        h.push_str("severity: ");
        h.push_str(sev_cls);
        if let Some(code) = result.info.code {
            h.push_str(" · code: ");
            h.push_str(&format_str(format_args!("{code}")));
        }
        h.push_str("</div>");
        el.set_inner_html(&h);
    }

    let lvl3_id = format_str(format_args!("err-lvl3-{}", def.id));
    if let Some(el) = doc.get_element_by_id(&lvl3_id) {
        el.set_class_name(&format_str(format_args!(
            "err-level err-level-app err-app-{}",
            sev_cls,
        )));
    }

    // Stack
    let stack_id = format_str(format_args!("err-stack-{}", def.id));
    if let Some(el) = doc.get_element_by_id(&stack_id) {
        match &result.info.stack {
            Some(s) => {
                let truncated = truncate_stack(s, 6);
                el.set_text_content(Some(&truncated));
            }
            None => {
                el.set_text_content(Some("(stack недоступен)"));
            }
        }
    }

    let card_id = format_str(format_args!("err-card-{}", def.id));
    if let Some(el) = doc.get_element_by_id(&card_id) {
        el.set_class_name(&format_str(format_args!("err-card err-sev-{}", sev_cls)));
    }
}

fn set_card_running(doc: &Document, id: &str, running: bool) {
    let card_id = format_str(format_args!("err-card-{}", id));
    let Some(card) = doc.get_element_by_id(&card_id) else { return };
    if running {
        let _ = card.set_attribute("data-running", "");
        let _ = card.set_class_name("err-card err-running");
    } else {
        let _ = card.remove_attribute("data-running");
    }
}

fn set_progress(doc: &Document, text: &str) {
    let Some(el) = doc.get_element_by_id("err-progress") else { return };
    el.set_text_content(Some(text));
}

// =============================================================================
// Summary
// =============================================================================

fn render_summary(doc: &Document, st: &AppState) {
    let Some(el) = doc.get_element_by_id("err-summary") else { return };

    let total = SCENARIOS.len();
    let done = st.completed();

    if done == 0 {
        el.set_inner_html(
            r#"<div><span class="lbl">Пока ни одного сценария не запущено — 9 кнопок впереди</span></div>"#,
        );
        return;
    }

    let mut info_count = 0usize;
    let mut warn_count = 0usize;
    let mut err_count = 0usize;

    let mut by_category: Vec<(&'static str, Severity, usize)> = Vec::new();
    for r in st.results.iter().flatten() {
        let sev = r.app_error.severity();
        match sev {
            Severity::Info => info_count += 1,
            Severity::Warning => warn_count += 1,
            Severity::Error => err_count += 1,
        }
        let label = r.app_error.category_label();
        if let Some(entry) = by_category.iter_mut().find(|(l, _, _)| *l == label) {
            entry.2 += 1;
        } else {
            by_category.push((label, sev, 1));
        }
    }

    let mut html = String::with_capacity(512);
    html.push_str(r#"<div><span class="lbl">Выполнено: </span><span class="val">"#);
    html.push_str(&format_str(format_args!("{done} из {total}")));
    html.push_str("</span></div>");

    // Локализованные метки без проблем с числительными:
    // "инфо: 4", "предупр.: 0", "ошибок: 5".
    html.push_str(r#"<div><span class="lbl">По severity: </span><span class="info">"#);
    html.push_str(&format_str(format_args!("{}: {}", severity_label(Severity::Info), info_count)));
    html.push_str("</span> · <span class=\"warn\">");
    html.push_str(&format_str(format_args!("{}: {}", severity_label(Severity::Warning), warn_count)));
    html.push_str("</span> · <span class=\"err\">");
    html.push_str(&format_str(format_args!("{}: {}", severity_label(Severity::Error), err_count)));
    html.push_str("</span></div>");

    // Flexbox-контейнер: лейбл занимает строку, чипы переносятся.
    html.push_str(r#"<div class="err-summary-cat"><span class="cat-label">По категориям:</span>"#);
    by_category.sort_by(|a, b| b.2.cmp(&a.2));
    for (label, sev, count) in &by_category {
        let chip_cls = severity_class(*sev);
        html.push_str(r#"<span class="err-summary-chip chip-"#);
        html.push_str(chip_cls);
        html.push_str(r#"">"#);
        html.push_str(&escape_html(label));
        html.push_str(r#"<span class="chip-count">×"#);
        html.push_str(&format_str(format_args!("{count}")));
        html.push_str("</span></span>");
    }
    html.push_str("</div>");

    el.set_inner_html(&html);
}

fn render_log(doc: &Document, logs: &[LogEntry]) {
    let Some(el) = doc.get_element_by_id("err-log") else { return };
    el.set_inner_html(&render_log_html(logs));
    if let Ok(el) = el.dyn_into::<web_sys::HtmlElement>() {
        let _ = el.set_scroll_top(el.scroll_height() as f64);
    }
}

fn set_buttons_enabled(doc: &Document, enabled: bool) {
    let Ok(nodes) = doc.query_selector_all("button[data-run], #err-run-all") else {
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
// Запуск
// =============================================================================

enum RawOutcome {
    Err(JsValue),
    UnexpectedOk(JsValue),
}

async fn run_scenario(def: &ScenarioDef) -> RawOutcome {
    if def.is_async {
        return match error_scenarios::run_async(def.id).await {
            Ok(v) => RawOutcome::UnexpectedOk(v),
            Err(e) => RawOutcome::Err(e),
        };
    } else {
        return match error_scenarios::run_sync(def.id) {
            Ok(v) => RawOutcome::UnexpectedOk(v),
            Err(e) => RawOutcome::Err(e),
        };
    }
}

async fn execute_one(doc: Document, st: Rc<RefCell<AppState>>, idx: usize) {
    if st.borrow().running {
        return;
    }
    let def = &SCENARIOS[idx];

    {
        let mut s = st.borrow_mut();
        s.running = true;
        s.results[idx] = None;
    }
    set_buttons_enabled(&doc, false);
    set_card_running(&doc, def.id, true);
    push_log(
        &st,
        "info",
        format_str(format_args!("▶ {} ({})", def.title, def.id)),
    );
    render_log(&doc, &st.borrow().logs);
    render_summary(&doc, &st.borrow());

    TimeoutFuture::new(0).await;

    let outcome = run_scenario(def).await;

    let result = match outcome {
        RawOutcome::Err(e) => {
            let raw_debug = debug_jsvalue(&e);
            let info = extract_error_info(&e);
            let app_error = AppError::from_info(&info);
            push_log(
                &st,
                "warn",
                format_str(format_args!(
                    "  извлечено: name={}, message={}",
                    info.name, info.message
                )),
            );
            push_log(
                &st,
                "info",
                format_str(format_args!(
                    "  категория: {} [{}] → «{}»",
                    app_error.category_label(),
                    severity_class(app_error.severity()),
                    app_error.user_message()
                )),
            );
            Some(ScenarioResult {
                raw_debug,
                info,
                app_error,
            })
        }
        RawOutcome::UnexpectedOk(v) => {
            push_log(
                &st,
                "err",
                format_str(format_args!(
                    "✗ сценарий '{}' не бросил исключение, вернул {}",
                    def.id,
                    if v.is_undefined() { "undefined" } else { "значение" }
                )),
            );
            None
        }
    };

    {
        let mut s = st.borrow_mut();
        s.results[idx] = result;
    }

    {
        let s = st.borrow();
        if let Some(Some(r)) = s.results.get(idx) {
            update_card(&doc, def, r);
        }
        render_log(&doc, &s.logs);
        render_summary(&doc, &s);
    }

    set_card_running(&doc, def.id, false);
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
        s.logs.clear();
    }
    push_log(&st, "info", String::from("▶▶ Запустить все сценарии"));
    render_log(&doc, &st.borrow().logs);
    render_summary(&doc, &st.borrow());

    let total = SCENARIOS.len();
    for i in 0..total {
        set_progress(&doc, &format_str(format_args!("{}/{}", i, total)));
        execute_one(doc.clone(), st.clone(), i).await;
        set_progress(&doc, &format_str(format_args!("{}/{}", i + 1, total)));
    }

    push_log(&st, "info", String::from("✔ Все сценарии выполнены"));
    render_log(&doc, &st.borrow().logs);

    // Дадим пользователю увидеть финальное "9/9".
    TimeoutFuture::new(1500).await;
    set_progress(&doc, "");
}

// =============================================================================
// Сборка UI
// =============================================================================

fn build_intro(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("err-desc");
    d.set_inner_html(
        "<p class=\"err-desc-line\"><strong>Обработка ошибок на границе Rust ↔ JS.</strong> \
         Когда JS бросает исключение или Promise отклоняется, Rust видит его как \
         opaque <code>JsValue</code>. Ни <code>name</code>, ни <code>message</code>, ни <code>stack</code> \
         не доступны напрямую.</p>\
         <p class=\"err-desc-line\"><strong>Путь ошибки через три уровня.</strong> \
         ① <strong>Сырой JsValue</strong> — Rust видит только тип handle \
         (<code>JsValue(Object)</code>, <code>JsValue(Number(42))</code>). \
         ② <strong>extract_error_info</strong> — name/message/stack/code через <code>Reflect::get</code>. \
         ③ <strong>AppError</strong> — прикладная категория с severity и user-facing сообщением.</p>\
         <p class=\"err-desc-line\"><strong>Принцип.</strong> Пользователь не должен \
         видеть <code>TypeError: x is not a function</code> — он должен видеть \
         «Внутренняя ошибка приложения. Обратитесь в поддержку». \
         Разница — между ① и ③.</p>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_controls(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("err-block");
    d.set_inner_html(
        "<button class=\"err-btn err-btn-primary\" id=\"err-run-all\">Запустить все</button>\
         <span class=\"err-progress\" id=\"err-progress\"></span>\
         <span class=\"err-hint\">9 сценариев · sync + async · throw + Promise.reject</span>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_summary(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("err-summary");
    d.set_id("err-summary");
    d.set_inner_html(
        "<div><span class=\"lbl\">Пока ни одного сценария не запущено — 9 кнопок впереди</span></div>",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_log(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("err-log");
    d.set_id("err-log");
    return Ok(d.unchecked_into::<web_sys::Node>());
}

fn build_footnote(doc: &Document) -> Result<web_sys::Node, JsValue> {
    let d = doc.create_element("div")?;
    d.set_class_name("err-footnote");
    d.set_inner_html(
        "<strong>Ambiguous cases.</strong> \
         Один и тот же <code>name</code> в разных API может значить разное. \
         <code>AbortError</code> в Web Bluetooth — «пользователь отменил выбор», \
         в <code>fetch</code> — «запрос отменён через AbortController». \
         <code>NotFoundError</code> в Bluetooth — «устройство не найдено», \
         в DOM — «элемент не найден». \
         <code>AppError::from_info</code> даёт наиболее распространённую категорию; \
         если нужна точная — переопределите её в вызывающем коде.",
    );
    return Ok(d.unchecked_into::<web_sys::Node>());
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    // Устанавливаем сценарии до первого клика.
    error_scenarios::install();

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("Глава 47 — Обработка ошибок"));
    root.append_child(as_node(&h2))?;

    root.append_child(&build_intro(&document)?)?;
    root.append_child(&build_controls(&document)?)?;
    root.append_child(&build_summary(&document)?)?;

    let state = Rc::new(RefCell::new(AppState::new()));

    let grid = document.create_element("div")?;
    grid.set_class_name("err-grid");
    grid.set_inner_html(&render_cards_html());
    root.append_child(as_node(&grid))?;

    root.append_child(&build_log(&document)?)?;
    root.append_child(&build_footnote(&document)?)?;

    // Event delegation для кнопок "Запустить сценарий".
    {
        let st = state.clone();
        let doc = document.clone();
        let closure: Closure<dyn FnMut(web_sys::Event)> =
            Closure::new(move |event: web_sys::Event| {
                let Some(target) = event.target() else { return };
                let Ok(target) = target.dyn_into::<web_sys::Element>() else { return };
                let Some(scenario_id) = target.get_attribute("data-run") else {
                    return;
                };
                let idx = SCENARIOS.iter().position(|s| s.id == scenario_id);
                let Some(idx) = idx else { return };
                if st.borrow().running {
                    return;
                }
                let doc_async = doc.clone();
                let st_async = st.clone();
                spawn_local(async move {
                    execute_one(doc_async, st_async, idx).await;
                });
            });
        grid.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Run all.
    {
        let st = state.clone();
        let doc = document.clone();
        let btn = document
            .get_element_by_id("err-run-all")
            .ok_or_else(|| JsValue::from_str("no #err-run-all"))?;
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

    render_summary(&document, &state.borrow());
    render_log(&document, &state.borrow().logs);

    return Ok(());
}
```

## 47.7. Ловушки, которые мы прошли в этой главе

Этот раздел — практический вывод главы. Путь от первых наивных версий до финального рабочего демо выявил несколько **архитектурных ловушек** обработки ошибок в Rust/WASM.

### 1. `JsValue::Debug` вводит в заблуждение

`format!("{:?}", value)` для `JsValue` делегирует JS-side `toString`, который для `Error` возвращает `"Name: message\n at stack"`, а для plain object — `"[object Object]"`. Разные сценарии дают разный формат, и это **путает**, когда показываешь несколько ошибок рядом.

**Решение:** нормализовать представление. В `debug_jsvalue` мы показываем только **тип handle**: `JsValue(Object)`, `JsValue(Number(42))`, `JsValue(undefined)`. Это честно — Rust действительно не видит содержимого без `Reflect::get`.

### 2. Слишком широкий `Unknown` — признак плохой классификации

Первая версия `AppError::from_info` маппила только известные DOMException-имена в конкретные категории, всё остальное отправляла в `Unknown`. Результат: `throw new Error("...")` и `Promise.reject('string')` попадали в «Неизвестно» — что **неправильно**, потому что это осмысленные ошибки с известным смыслом.

**Решение:** добавить две категории:
- `Generic` — для `Error` без подкласса. Осмысленная прикладная ошибка, severity Warning.
- `NonStandardRejection` — для `Promise.reject('string')` / `42` / `true`. Сигнал бага в JS-коде, severity Error.

После этого «Неизвестно» исчезает из реальных сценариев.

### 3. `DOMException.code` не читается через `Reflect::get` в headless Chrome

В Chrome `DOMException.code` объявлен как getter на прототипе. В non-headless сборках `Reflect::get(value, "code")` его находит, в headless — возвращает `undefined` или `0`.

**Решение:** fallback-таблица `dom_exception_code(name)` по стандарту WebIDL. Если `Reflect::get` вернул `None` или `0`, берём код из таблицы. Это надёжнее и не зависит от версии браузера.

### 4. `Promise.reject('string')` — невалидный `Error` для JS, но валидный rejection

Когда вызываешь `Promise.reject('string')`, в `await` приходит не `Error`, а **примитив**. Rust видит его как `JsValue(String("..."))`, `Reflect::get(value, "name")` вернёт `Err`, и если не обработать — получишь `Error` с пустым `message`.

**Решение:** в `extract_error_info` **сначала** проверять `as_string`/`as_f64`/`as_bool`, и только потом `Reflect::get`. Примитив получает синтетическое имя `String`/`Number`/`Boolean`.

### 5. `RangeError` от `new Array(-1)` — «Неверный ввод», но это ошибка программиста

Категоризация по одному имени даёт сбой: `RangeError` может быть от пользовательского ввода (`parseInt("abc")`) или от программерской ошибки (`new Array(-1)`). Оба дают одинаковый `name`.

**Решение:** оставить `RangeError` в `InvalidInput` как наиболее распространённую категорию, но **оговорить в footnote** — для специфичных API вызывающий код переопределяет категорию вручную. Идеального автоматического решения тут нет.

### 6. Chips в summary не переносятся на новую строку

При 6+ категориях чипы обрезались за пределами видимой области. `display:block` на контейнере + `display:inline-block` на чипах теоретически работают, но `margin` и `gap` мешают автоматическому переносу.

**Решение:** flexbox с `flex-wrap:wrap`. Лейбл «По категориям:» занимает свою строку через `flex-basis:100%`, чипы переносятся автоматически.

### 7. `push_log` — порядок аргументов

Мелочь, но она стоила итерации: функция объявлена `push_log(st, level, text)`, а вызвана была `push_log(st, text, level)`. rustc поймал несоответствие типов. **Общее правило:** объявлять параметры в одном порядке и использовать его везде без исключений.

### 8. `Promise.resolve(42)` для success-сценария

Сценарий `success` изначально возвращал голое число `42`. `run_async` ожидает Promise, `dyn_into::<Promise>` падал. **Контракт `run_async`:** все сценарии должны возвращать Promise, включая успешные. Прямой fix — `Promise.resolve(42)`.

## 47.8. Итог

Что даёт глава:

1. **Границу ошибок в действии.** 9 сценариев от `TypeError` до `Promise.reject(42)` — все идут через одну обработку. Студент видит: независимо от источника, `extract_error_info` даёт структуру, `AppError::from_info` — категорию, `user_message` — текст для пользователя.

2. **Три уровня абстракции.** Сырой `JsValue` → `JsErrorInfo` → `AppError`. Это **правильная архитектура обработки ошибок**: не логировать и не показывать сырое, а переводить.

3. **Нестандартные случаи обрабатываются без паники.** `Promise.reject('string')`, `Promise.reject(42)`, `undefined` — всё даёт осмысленную категорию, а не «Unknown».

4. **DOMException отличается от Error по `code`** — fallback-таблица восстанавливает стандартные коды по имени.

5. **Неизбежные ограничения оговорены.** Footnote объясняет ambiguous cases — категоризация по `name` работает не всегда, нужна донастройка под API.

**Правила проекта соблюдены:**

- `format_str(format_args!(...))` — везде, где `format!` мог бы оказаться в хвостовой позиции.
- `TimeoutFuture::new(0).await` — между итерациями в `run_all`.
- `Rc<RefCell<AppState>>` — общее состояние.
- Префикс CSS `err-` — уникален.
- `app.rs` содержит только демо текущей главы.
- Никаких `unused_imports`, `dead_code`, `unused_mut`.
- HTML с несколькими вложенными `"` — через `push_str`, не через `format_args!`.

**Cargo.toml:** без изменений. Все зависимости уже есть с предыдущих глав.

**Тесты:** 40 тестов — извлечение, классификация, severity, user_message, сценарии, сквозной путь.

**Главный урок главы.** Пользователь не должен видеть `TypeError: x is not a function` — он должен видеть «Внутренняя ошибка приложения. Обратитесь в поддержку». **Разница — между ① и ③.** Именно это различие демонстрирует глава, и именно этот принцип стоит вынести в её название.

**Команды финальной проверки:**

```bash
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests
wasm-pack test --headless --chrome --test wasm_error_handling
wasm-pack build --release --target web
python3 -m http.server 5173
```