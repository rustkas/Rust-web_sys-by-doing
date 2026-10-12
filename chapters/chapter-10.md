# Глава 10. `JsValue` как универсальный тип границы

## Обновлённая структура проекта

```
hello-wasm02/
├── Cargo.toml
├── Cargo.lock
├── package.json
├── package-lock.json
├── vite.config.js
├── webdriver.json
├── .gitignore
├── LICENSE
├── README.md
├── src/
│   ├── lib.rs                      # pub mod dom; pub mod events; pub mod value; ...
│   ├── app.rs                      # инспектор JsValue
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── value.rs                    # помощники для работы с JsValue
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs
│   ├── wasm_jsvalue.rs             # 15 тестов для JsValue
│   └── wasm_async.rs
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # стили инспектора
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/value.rs` — модуль помощников для работы с `JsValue`. Публичный (`pub mod value;`), потому что интеграционные тесты обращаются к нему через `use hello_wasm02::value;`.
- `src/app.rs` — переписан под практический проект: **инспектор `JsValue`** с секциями «Примитивы», «Массив», «Объект», «Функция».
- `tests/wasm_jsvalue.rs` — 15 тестов, разбитых по категориям: примитивы, массивы, объекты, функции, `describe`, `JsCast`.
- `web/styles/main.css` — заменяется целиком: стили счётчика из главы 9 удалены, добавлены стили секций инспектора.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `JsValue` (docs.rs) | https://docs.rs/wasm-bindgen/latest/wasm_bindgen/struct.JsValue.html |
| `JsValue` (wasm-bindgen guide) | https://wasm-bindgen.github.io/wasm-bindgen/api/wasm_bindgen/struct.JsValue.html |
| `js_sys::Array` | https://docs.rs/js-sys/latest/js_sys/struct.Array.html |
| `js_sys::Object` | https://docs.rs/js-sys/latest/js_sys/struct.Object.html |
| `js_sys::Reflect` | https://docs.rs/js-sys/latest/js_sys/struct.Reflect.html |
| `js_sys::Function` | https://docs.rs/js-sys/latest/js_sys/struct.Function.html |
| `js_sys::Map` | https://docs.rs/js-sys/latest/js_sys/struct.Map.html |
| Type Translations (wasm-bindgen) | https://rustwasm.github.io/wasm-bindgen/guide/types.html |
| Accessing Properties of Untyped JS Values | https://wasm-bindgen.netlify.app/docs/reference/accessing-properties-of-untyped-js-values.html |

---

## `Cargo.toml`

```toml
[package]
name = "hello-wasm02"
version = "0.1.0"
edition = "2021"
description = "Rust + web_sys starter"
license = "MIT"
repository = "https://github.com/rustkas/hello-wasm02"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"
console_error_panic_hook = "0.1"

[dependencies.web-sys]
version = "0.3"
features = [
    # Основные типы DOM
    "Window",
    "Document",
    "Node",
    "Element",
    "HtmlElement",

    # Поиск и создание
    "NodeList",
    "HtmlCollection",

    # Атрибуты и классы
    "DomTokenList",
    "NamedNodeMap",

    # События (базовые)
    "Event",
    "EventTarget",
    "EventInit",
    "InputEvent",

    # События (специализированные)
    "KeyboardEvent",
    "MouseEvent",
    "PointerEvent",

    # Элементы формы
    "HtmlInputElement",
    "HtmlButtonElement",
    "HtmlTextAreaElement",
    "HtmlSelectElement",
    "HtmlOptionElement",
    "HtmlOptionsCollection",
    "HtmlFormElement",

    # Медиа-элементы
    "HtmlImageElement",
    "HtmlCanvasElement",
    "HtmlVideoElement",
    "HtmlAudioElement",

    # Прочее
    "console",
]

[dev-dependencies]
wasm-bindgen-test = "0.3"

[profile.release]
opt-level = "s"
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

**Что изменилось:** новых features в `web-sys` не добавлено. Работа с `JsValue`, `js_sys::Array`, `js_sys::Object`, `js_sys::Reflect` идёт через crate `js-sys`, который уже в зависимостях.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Модуль `dom` публичен: переиспользуемые помощники для работы с DOM.
//! Модуль `events` публичен: интеграционные тесты обращаются к нему
//! через `use hello_wasm02::events;`.
//! Модуль `value` публичен: помощники для работы с `JsValue`
//! и тесты для них.

pub mod dom;
pub mod events;
pub mod value;
mod app;
mod form;
mod logging;

use wasm_bindgen::prelude::*;

/// Точка входа. Вызывается автоматически при инициализации Wasm-модуля.
#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    logging::init();
    app::run()
}
```

**Ключевое:** `pub mod value;` — модуль публичный, чтобы интеграционные тесты в `tests/wasm_jsvalue.rs` могли обращаться к нему через `use hello_wasm02::value;`.

---

## `src/value.rs`

```rust
//! Помощники для работы с `JsValue`.
//!
//! `JsValue` — универсальный тип на границе Rust ↔ JavaScript.
//! Он может представлять любое JavaScript-значение: примитив,
//! массив, объект, функцию. Этот модуль содержит функции для
//! определения типа `JsValue` и безопасного преобразования
//! в Rust-типы.
//!
//! Модуль — переиспользуемая библиотека. Часть функций пока
//! не используется в `app.rs`, но понадобится в следующих главах.
//!
//! ## Замечание о `return`
//!
//! В функции `describe` последняя строка использует явный `return`.
//! Это защищает от редких случаев, когда `rust-analyzer` неверно
//! интерпретирует tail-выражение и выдаёт ошибку типа.
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use js_sys::{Array, Function, Object, Reflect};

// ============================================================
// Создание JsValue из Rust-значений
// ============================================================

/// Создаёт `JsValue` из строки.
pub fn from_str(s: &str) -> JsValue {
    JsValue::from_str(s)
}

/// Создаёт `JsValue` из числа.
pub fn from_f64(n: f64) -> JsValue {
    JsValue::from_f64(n)
}

/// Создаёт `JsValue` из булева значения.
pub fn from_bool(b: bool) -> JsValue {
    JsValue::from_bool(b)
}

/// Возвращает `JsValue`, представляющий `null`.
pub fn null() -> JsValue {
    JsValue::NULL
}

/// Возвращает `JsValue`, представляющий `undefined`.
pub fn undefined() -> JsValue {
    JsValue::UNDEFINED
}

// ============================================================
// Определение типа
// ============================================================

/// Возвращает строковое имя типа `JsValue`.
///
/// Использует JavaScript-оператор `typeof`. Для `null` возвращает
/// `"object"` — это особенность JS, которую нужно помнить.
pub fn type_name(value: &JsValue) -> String {
    value
        .js_typeof()
        .as_string()
        .unwrap_or_else(|| "unknown".into())
}

/// Проверяет, является ли значение строкой.
pub fn is_string(value: &JsValue) -> bool {
    value.is_string()
}

/// Проверяет, является ли значение числом.
pub fn is_number(value: &JsValue) -> bool {
    value.as_f64().is_some()
}

/// Проверяет, является ли значение булевым.
pub fn is_bool(value: &JsValue) -> bool {
    value.as_bool().is_some()
}

/// Проверяет, является ли значение `null`.
pub fn is_null(value: &JsValue) -> bool {
    value.is_null()
}

/// Проверяет, является ли значение `undefined`.
pub fn is_undefined(value: &JsValue) -> bool {
    value.is_undefined()
}

/// Проверяет, является ли значение массивом.
pub fn is_array(value: &JsValue) -> bool {
    value.is_array()
}

/// Проверяет, является ли значение объектом.
///
/// В JavaScript `[] instanceof Object === true` — массив наследуется
/// от `Object`, поэтому `is_object()` возвращает `true` и для массивов,
/// и для объектов. Функции возвращают `false`: у них `typeof === "function"`.
/// `null` тоже возвращает `false`: это единственный случай, когда
/// `typeof null === "object"`, но `null instanceof Object === false`.
///
/// Для точного определения массива используйте `is_array()`,
/// для функции — `is_function()`.
pub fn is_object(value: &JsValue) -> bool {
    value.is_object()
}

/// Проверяет, является ли значение функцией.
pub fn is_function(value: &JsValue) -> bool {
    value.is_function()
}

// ============================================================
// Безопасное преобразование
// ============================================================

/// Пытается получить `String` из `JsValue`.
pub fn as_string(value: &JsValue) -> Option<String> {
    value.as_string()
}

/// Пытается получить `f64` из `JsValue`.
pub fn as_f64(value: &JsValue) -> Option<f64> {
    value.as_f64()
}

/// Пытается получить `bool` из `JsValue`.
pub fn as_bool(value: &JsValue) -> Option<bool> {
    value.as_bool()
}

/// Описывает `JsValue` в читаемом виде для логирования.
///
/// Порядок проверок важен: `is_array()` проверяется **до** `is_object()`,
/// потому что массив проходит обе проверки. Без этого массив был бы
/// описан как `"object"`, что менее точно.
pub fn describe(value: &JsValue) -> String {
    if value.is_null() {
        return "null".into();
    }
    if value.is_undefined() {
        return "undefined".into();
    }
    if let Some(s) = value.as_string() {
        return format!("string: {:?}", s);
    }
    if let Some(n) = value.as_f64() {
        return format!("number: {}", n);
    }
    if let Some(b) = value.as_bool() {
        return format!("boolean: {}", b);
    }
    if value.is_array() {
        let arr: &Array = value.unchecked_ref();
        return format!("array: len={}", arr.length());
    }
    if value.is_function() {
        return "function".into();
    }
    if value.is_object() {
        return "object".into();
    }
    return format!("unknown ({})", type_name(value));
}

// ============================================================
// Работа с массивами
// ============================================================

/// Создаёт пустой массив.
pub fn new_array() -> Array {
    Array::new()
}

/// Создаёт массив из Rust-слайса строк.
pub fn array_of_strings(items: &[&str]) -> Array {
    let arr = Array::new();
    for s in items {
        arr.push(&JsValue::from_str(s));
    }
    arr
}

/// Читает элемент массива по индексу.
pub fn array_get(arr: &Array, index: u32) -> Option<JsValue> {
    let value = arr.get(index);
    if value.is_undefined() {
        None
    } else {
        Some(value)
    }
}

// ============================================================
// Работа с объектами
// ============================================================

/// Создаёт пустой объект.
pub fn new_object() -> Object {
    Object::new()
}

/// Устанавливает свойство объекта.
pub fn object_set(obj: &Object, key: &str, value: &JsValue) -> Result<(), JsValue> {
    Reflect::set(obj.as_ref(), &JsValue::from_str(key), value)
        .map(|_| ())
        .map_err(|e| JsValue::from_str(&format!("Reflect::set failed: {:?}", e)))
}

/// Читает свойство объекта.
pub fn object_get(obj: &Object, key: &str) -> Result<JsValue, JsValue> {
    Reflect::get(obj.as_ref(), &JsValue::from_str(key))
}

/// Проверяет наличие свойства в объекте.
pub fn object_has(obj: &Object, key: &str) -> bool {
    Reflect::has(obj.as_ref(), &JsValue::from_str(key)).unwrap_or(false)
}

/// Создаёт объект из пар ключ-значение.
pub fn object_from_pairs(pairs: &[(&str, &str)]) -> Result<Object, JsValue> {
    let obj = Object::new();
    for (key, value) in pairs {
        object_set(&obj, key, &JsValue::from_str(value))?;
    }
    Ok(obj)
}

// ============================================================
// Работа с функциями
// ============================================================

/// Вызывает `Function` без аргументов.
pub fn call_function(f: &Function) -> Result<JsValue, JsValue> {
    f.call0(&JsValue::NULL)
}

/// Вызывает `Function` с одним аргументом.
pub fn call_function_1(f: &Function, arg: &JsValue) -> Result<JsValue, JsValue> {
    f.call1(&JsValue::NULL, arg)
}

// ============================================================
// Проверка на вложенность
// ============================================================

/// Проверяет, является ли `JsValue` «примитивом»:
/// string, number, boolean, null или undefined.
pub fn is_primitive(value: &JsValue) -> bool {
    is_string(value)
        || is_number(value)
        || is_bool(value)
        || is_null(value)
        || is_undefined(value)
}
```

**Ключевые моменты:**

- **`describe` содержит явный `return` в последней строке.** Это защищает от редкой ошибки `rust-analyzer`, когда tail-выражение интерпретируется неверно.
- **`is_object` имеет подробный doc-комментарий** о том, что массивы проходят эту проверку, а функции и `null` — нет.
- **Порядок проверок в `describe` важен:** `is_array()` раньше `is_object()`.
- **`describe` использует `format!("string: {:?}", s)`** — `{:?}` добавляет кавычки, что делает вывод однозначнее.

---

## `src/app.rs` (инспектор `JsValue`)

```rust
//! Логика приложения: «Инспектор JsValue».
//!
//! Демонстрирует работу с разными типами JavaScript:
//! примитивами, массивами, объектами и функциями.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use js_sys::Function;

use crate::dom;
use crate::log;
use crate::value;

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Инспектор JsValue")?;
    dom::append(&container, &heading)?;

    // 1. Примитивы
    inspect_primitive(&container, "Строка", value::from_str("Hello, JsValue!"))?;
    inspect_primitive(&container, "Число", value::from_f64(42.5))?;
    inspect_primitive(&container, "Булево true", value::from_bool(true))?;
    inspect_primitive(&container, "Булево false", value::from_bool(false))?;
    inspect_primitive(&container, "null", value::null())?;
    inspect_primitive(&container, "undefined", value::undefined())?;

    // 2. Массив
    let array = value::array_of_strings(&["один", "два", "три"]);
    inspect_collection(&container, "Массив строк", array.as_ref())?;

    // 3. Объект
    let obj = value::object_from_pairs(&[
        ("name", "Rust"),
        ("version", "1.75"),
        ("wasm", "true"),
    ])?;
    inspect_collection(&container, "Объект", obj.as_ref())?;

    let name = value::object_get(&obj, "name")?;
    log!("Object.name = {}", value::describe(&name));

    // 4. Функция
    inspect_function(&container)?;

    log!("Инспектор JsValue завершён");
    Ok(())
}

/// Создаёт секцию с описанием примитива.
fn inspect_primitive(container: &web_sys::Element, label: &str, val: JsValue) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;

    let title = dom::create_element("h2")?;
    title.set_text_content(Some(label));
    dom::append(&section, &title)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;

    let description = format!(
        "describe: {}\ntypeof: {}\nis_string: {}\nis_number: {}\nis_bool: {}\nis_null: {}\nis_undefined: {}",
        value::describe(&val),
        value::type_name(&val),
        value::is_string(&val),
        value::is_number(&val),
        value::is_bool(&val),
        value::is_null(&val),
        value::is_undefined(&val),
    );

    result.set_text_content(Some(&description));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("{}: {}", label, value::describe(&val));

    Ok(())
}

/// Создаёт секцию с описанием массива или объекта.
fn inspect_collection(
    container: &web_sys::Element,
    label: &str,
    val: &JsValue,
) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;

    let title = dom::create_element("h2")?;
    title.set_text_content(Some(label));
    dom::append(&section, &title)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;

    let mut lines = Vec::new();
    lines.push(format!("describe: {}", value::describe(val)));
    lines.push(format!("typeof: {}", value::type_name(val)));
    lines.push(format!("is_array: {}", value::is_array(val)));
    lines.push(format!("is_object: {}", value::is_object(val)));
    lines.push(format!("is_function: {}", value::is_function(val)));

    if value::is_array(val) {
        let arr: &js_sys::Array = val.unchecked_ref();
        lines.push(format!("length: {}", arr.length()));
        for i in 0..arr.length() {
            if let Some(item) = value::array_get(arr, i) {
                lines.push(format!("  [{}] = {}", i, value::describe(&item)));
            }
        }
    }

    if value::is_object(val) {
        let obj: &js_sys::Object = val.unchecked_ref();
        for key in ["name", "version", "wasm"] {
            if value::object_has(obj, key) {
                let v = value::object_get(obj, key)?;
                lines.push(format!("  .{} = {}", key, value::describe(&v)));
            }
        }
    }

    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("{}: {}", label, value::describe(val));

    Ok(())
}

/// Создаёт секцию с описанием функции.
fn inspect_function(container: &web_sys::Element) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;

    let title = dom::create_element("h2")?;
    title.set_text_content(Some("Функция"));
    dom::append(&section, &title)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;

    let f: Function = js_sys::eval("(function(x) { return x * 2; })")
        .map_err(|_| JsValue::from_str("eval failed"))?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not a function"))?;

    let f_value: &JsValue = f.as_ref();
    let mut lines = Vec::new();
    lines.push(format!("describe: {}", value::describe(f_value)));
    lines.push(format!("typeof: {}", value::type_name(f_value)));
    lines.push(format!("is_function: {}", value::is_function(f_value)));
    lines.push(format!("is_object: {}", value::is_object(f_value)));

    let arg = value::from_f64(21.0);
    let result_value = value::call_function_1(&f, &arg)?;
    lines.push(format!(
        "f(21) = {} ({})",
        value::describe(&result_value),
        value::as_f64(&result_value).unwrap_or(0.0)
    ));

    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("Функция: f(21) = {:?}", value::as_f64(&result_value));

    Ok(())
}
```

**Что демонстрирует `app.rs`:**

1. **Примитивы** — строка, число, `true`, `false`, `null`, `undefined`.
2. **Массив строк** — длина, элементы.
3. **Объект** — `is_object: true`, свойства через `Reflect`.
4. **Функция** — `is_function: true`, `is_object: false`, вызов `f(21) = 42`.

**Обратите внимание:** в секции «Массив строк» будет `is_object: true`. Это правильное поведение JavaScript — массив наследуется от `Object`.

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — инспектор JsValue</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Инспектор JsValue. Откройте консоль для логов.</p>

    <div id="app"></div>

    <script type="module" src="/main.js"></script>
</body>
</html>
```

---

## `web/styles/main.css`

```css
/* ============================================================
   Базовые стили
   ============================================================ */

:root {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    line-height: 1.6;
    color: #222;
}

body {
    max-width: 720px;
    margin: 2rem auto;
    padding: 0 1rem;
}

h1 {
    color: #b7410e;
}

/* ============================================================
   Контейнер #app
   ============================================================ */

#app {
    margin-top: 1.5rem;
}

/* ============================================================
   Секции инспектора
   ============================================================ */

.section {
    margin-bottom: 1rem;
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
    background: #fafafa;
}

.section h2 {
    margin: 0 0 0.5rem 0;
    font-size: 1.1rem;
    color: #b7410e;
}

.result {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.9rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    min-height: 2rem;
}
```

**Что изменилось относительно главы 9.** Стили счётчика (`.counter-display`, `.controls`, `.hint`) удалены. Добавлены стили секций (`.section`, `.result`). `#app` без рамки — рамку несут секции.

---

## `tests/wasm_jsvalue.rs`

```rust
//! Тесты для `JsValue`: примитивы, массивы, объекты, функции.
use wasm_bindgen_test::*;
use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;
use js_sys::{Array, Function, Object};

use hello_wasm02::value;

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// Примитивы
// ============================================================

#[wasm_bindgen_test]
fn test_string_value() {
    let v = value::from_str("hello");

    assert!(value::is_string(&v));
    assert!(!value::is_number(&v));
    assert!(!value::is_bool(&v));
    assert!(!value::is_null(&v));
    assert!(!value::is_undefined(&v));
    assert!(!value::is_array(&v));
    assert!(!value::is_object(&v));
    assert!(!value::is_function(&v));
    assert!(value::is_primitive(&v));

    assert_eq!(value::as_string(&v), Some("hello".into()));
    assert_eq!(value::type_name(&v), "string");
}

#[wasm_bindgen_test]
fn test_number_value() {
    let v = value::from_f64(42.5);

    assert!(value::is_number(&v));
    assert!(!value::is_string(&v));
    assert!(!value::is_bool(&v));
    assert!(value::is_primitive(&v));

    assert_eq!(value::as_f64(&v), Some(42.5));
    assert_eq!(value::type_name(&v), "number");
}

#[wasm_bindgen_test]
fn test_bool_value() {
    let t = value::from_bool(true);
    let f = value::from_bool(false);

    assert!(value::is_bool(&t));
    assert!(value::is_bool(&f));
    assert!(value::is_primitive(&t));

    assert_eq!(value::as_bool(&t), Some(true));
    assert_eq!(value::as_bool(&f), Some(false));
    assert_eq!(value::type_name(&t), "boolean");
}

#[wasm_bindgen_test]
fn test_null_value() {
    let v = value::null();

    assert!(value::is_null(&v));
    assert!(!value::is_undefined(&v));
    assert!(!value::is_object(&v));
    assert!(value::is_primitive(&v));

    assert_eq!(value::as_string(&v), None);
    assert_eq!(value::as_f64(&v), None);
}

#[wasm_bindgen_test]
fn test_undefined_value() {
    let v = value::undefined();

    assert!(value::is_undefined(&v));
    assert!(!value::is_null(&v));
    assert!(value::is_primitive(&v));

    assert_eq!(value::as_string(&v), None);
}

// ============================================================
// Массивы
// ============================================================

#[wasm_bindgen_test]
fn test_empty_array() {
    let arr = value::new_array();

    assert_eq!(arr.length(), 0);

    // Явная аннотация, чтобы `as_ref()` вывел целевой тип.
    // `Array` реализует `AsRef<T>` для нескольких `T`,
    // и без аннотации компилятор не может выбрать.
    let js: &JsValue = arr.as_ref();
    assert!(js.is_array());
    // ВАЖНО: массив проходит и проверку is_object,
    // потому что в JS `[] instanceof Object === true`.
    assert!(js.is_object());
}

#[wasm_bindgen_test]
fn test_array_of_strings() {
    let arr = value::array_of_strings(&["один", "два", "три"]);

    assert_eq!(arr.length(), 3);

    let first = value::array_get(&arr, 0).unwrap();
    assert_eq!(value::as_string(&first), Some("один".into()));

    let second = value::array_get(&arr, 1).unwrap();
    assert_eq!(value::as_string(&second), Some("два".into()));

    let out_of_range = value::array_get(&arr, 99);
    assert!(out_of_range.is_none());
}

// ============================================================
// Объекты
// ============================================================

#[wasm_bindgen_test]
fn test_empty_object() {
    let obj = value::new_object();

    let js: &JsValue = obj.as_ref();
    assert!(js.is_object());
    assert!(!js.is_array());
    assert!(!js.is_function());
}

#[wasm_bindgen_test]
fn test_object_set_get() {
    let obj = value::new_object();

    value::object_set(&obj, "name", &value::from_str("Rust")).unwrap();
    value::object_set(&obj, "version", &value::from_f64(1.75)).unwrap();

    let name = value::object_get(&obj, "name").unwrap();
    assert_eq!(value::as_string(&name), Some("Rust".into()));

    let version = value::object_get(&obj, "version").unwrap();
    assert_eq!(value::as_f64(&version), Some(1.75));

    assert!(value::object_has(&obj, "name"));
    assert!(value::object_has(&obj, "version"));
    assert!(!value::object_has(&obj, "missing"));
}

#[wasm_bindgen_test]
fn test_object_from_pairs() {
    let obj = value::object_from_pairs(&[
        ("language", "Rust"),
        ("target", "wasm"),
    ])
    .unwrap();

    let lang = value::object_get(&obj, "language").unwrap();
    assert_eq!(value::as_string(&lang), Some("Rust".into()));

    let target = value::object_get(&obj, "target").unwrap();
    assert_eq!(value::as_string(&target), Some("wasm".into()));
}

// ============================================================
// Функции
// ============================================================

#[wasm_bindgen_test]
fn test_function_detection() {
    let f: Function = js_sys::eval("(function() { return 1; })")
        .unwrap()
        .dyn_into()
        .unwrap();

    let v: &JsValue = f.as_ref();

    assert!(value::is_function(v));
    assert!(!value::is_object(v)); // функции ≠ object по is_object
    assert!(!value::is_array(v));
    assert!(!value::is_primitive(v));
    assert_eq!(value::type_name(v), "function");
}

#[wasm_bindgen_test]
fn test_function_call() {
    let f: Function = js_sys::eval("(function(x) { return x * 2; })")
        .unwrap()
        .dyn_into()
        .unwrap();

    let arg = value::from_f64(21.0);
    let result = value::call_function_1(&f, &arg).unwrap();

    assert_eq!(value::as_f64(&result), Some(42.0));
}

// ============================================================
// describe
// ============================================================

#[wasm_bindgen_test]
fn test_describe() {
    assert_eq!(value::describe(&value::null()), "null");
    assert_eq!(value::describe(&value::undefined()), "undefined");
    assert!(value::describe(&value::from_str("x")).starts_with("string:"));
    assert!(value::describe(&value::from_f64(1.0)).starts_with("number:"));
    assert!(value::describe(&value::from_bool(true)).starts_with("boolean:"));

    // Привязка с явной аннотацией — иначе `as_ref()` неоднозначен.
    let arr = value::new_array();
    let arr_js: &JsValue = arr.as_ref();
    assert_eq!(value::describe(arr_js), "array: len=0");

    let obj = value::new_object();
    let obj_js: &JsValue = obj.as_ref();
    assert_eq!(value::describe(obj_js), "object");
}

// ============================================================
// JsCast: unchecked_ref и dyn_into
// ============================================================

#[wasm_bindgen_test]
fn test_unchecked_ref_to_array() {
    let arr = Array::new();
    arr.push(&JsValue::from_str("a"));

    let generic: &JsValue = arr.as_ref();
    assert!(generic.is_array());

    let back: &Array = generic.unchecked_ref();
    assert_eq!(back.length(), 1);
}

#[wasm_bindgen_test]
fn test_dyn_into_object() {
    let obj = Object::new();
    value::object_set(&obj, "k", &JsValue::from_str("v")).unwrap();

    let generic: JsValue = obj.clone().into();
    assert!(generic.is_object());

    let back: Object = generic.dyn_into().unwrap();
    let v = value::object_get(&back, "k").unwrap();
    assert_eq!(value::as_string(&v), Some("v".into()));
}
```

**Ключевые моменты тестов:**

- **`test_empty_array`** проверяет `js.is_object()` = `true` — массив наследуется от `Object`.
- **`test_function_detection`** проверяет `!value::is_object(v)` — функция **не** является объектом по этой проверке.
- **Все `as_ref()` имеют явную аннотацию `let js: &JsValue = ...`** — иначе компилятор не может вывести целевой тип.
- **`test_describe`** подтверждает, что массив описывается как `"array: len=0"`, а не `"object"` — благодаря порядку проверок в `describe`.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. В контейнере `#app` появятся секции: «Примитивы» (6 подсекций), «Массив строк», «Объект», «Функция». В каждой — описание типа, `typeof` и результаты проверок.

### Тесты

```bash
npm run test:host      # хостовые тесты
npm run test:node      # Wasm-логика в Node.js
npm run test:chrome    # DOM, события, JsValue в headless Chrome
npm run test:firefox   # то же в headless Firefox
npm test               # host + node + chrome
npm run test:all       # полный прогон
```

### Проверка конкретного теста

```bash
wasm-pack test --headless --chrome --test wasm_jsvalue
```

---

## Ключевые концепции главы

**1. `JsValue` — универсальный тип границы.** Может представлять любое JavaScript-значение. Это индекс в таблице `externref`, которой управляет JavaScript-обвязка `wasm-bindgen`.

**2. Создание `JsValue`.** `JsValue::from_str`, `from_f64`, `from_bool`, `JsValue::NULL`, `JsValue::UNDEFINED`.

**3. Проверка типа.** `is_string()`, `is_number()`, `is_bool()`, `is_null()`, `is_undefined()`, `is_array()`, `is_object()`, `is_function()`, `js_typeof()`.

**4. Особенность `is_object()` — критическая.** В JavaScript:

| Значение | `typeof` | `instanceof Object` | `is_object()` |
|---|---|---|---|
| `"hello"` | `"string"` | `false` | `false` |
| `42` | `"number"` | `false` | `false` |
| `true` | `"boolean"` | `false` | `false` |
| `null` | `"object"` | `false` | `false` |
| `undefined` | `"undefined"` | `false` | `false` |
| `[]` | `"object"` | **`true`** | **`true`** |
| `{}` | `"object"` | `true` | `true` |
| `function(){}` | `"function"` | `true` | **`false`** |

Массив проходит и `is_array()`, и `is_object()`. Функция проходит `is_function()`, но **не** `is_object()`. `null` не проходит `is_object()`, хотя `typeof null === "object"`.

**5. Безопасное преобразование.** `as_string()`, `as_f64()`, `as_bool()` возвращают `Option`.

**6. `unchecked_ref` и `dyn_into`.** После проверки типа можно привести `JsValue` к конкретному типу.

**7. `js_sys::Array`, `Object`, `Function`.** Специализированные обёртки. `Array::new`, `push`, `get`, `length`. `Object::new` + `Reflect::set/get/has`. `Function::call0`, `call1`.

**8. `Reflect` — доступ к свойствам.** Стандартный способ работы с произвольными свойствами объектов в `js_sys`.

**9. `describe` с явным порядком проверок.** `is_array()` раньше `is_object()` — иначе массив описывался бы как `"object"`.

**10. Явная аннотация для `as_ref()`.** Когда `Array` или `Object` вызывают `.as_ref()`, компилятор не может выбрать целевой `T` без подсказки. `let js: &JsValue = arr.as_ref();` решает проблему.

**11. Явный `return` в `describe`.** Tail-выражение с `format!` в конце цепочки `if-ов` иногда вызывает ошибку `rust-analyzer`. Явный `return` — практическое решение.

---

## Итог главы

- `JsValue` — универсальный тип для любого JavaScript-значения.
- Создание: `from_str`, `from_f64`, `from_bool`, `NULL`, `UNDEFINED`.
- Проверка типа: `is_string`, `is_number`, `is_bool`, `is_null`, `is_undefined`, `is_array`, `is_object`, `is_function`, `js_typeof`.
- **Массивы — объекты** в JavaScript: `is_array()` и `is_object()` обе возвращают `true`.
- **Функции — не объекты** по `is_object()`: `typeof === "function"`.
- **`null` — не объект** по `is_object()`, хотя `typeof null === "object"`.
- Безопасное преобразование: `as_string`, `as_f64`, `as_bool`.
- `js_sys::Array`, `Object`, `Function` — специализированные обёртки.
- `Reflect::set/get/has` — работа со свойствами объектов.
- `unchecked_ref` и `dyn_into` — приведение к конкретным типам после проверки.
- Порядок проверок в `describe` важен: `is_array()` раньше `is_object()`.
- Явная аннотация `let js: &JsValue = ...` нужна для `as_ref()` на `Array` и `Object`.
- Практический проект: инспектор `JsValue` с секциями для примитивов, массивов, объектов и функций.

---

**Что дальше.** В главе 11 мы продолжим работу с `js_sys` и разберём JavaScript-объекты глубже: `Array`, `Object`, `Function`, `Map`, `Set`, `Promise`, `Date`, типизированные массивы. Практический проект — «коллекции JavaScript в Rust».