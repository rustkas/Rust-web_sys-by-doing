# Глава 11. `js_sys` и JavaScript objects

## Обновлённая структура проекта

```text
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
│   ├── lib.rs                      # pub mod dom; pub mod events; pub mod js_objects; pub mod value; ...
│   ├── app.rs                      # демонстрация коллекций JavaScript
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs               # НОВОЕ: помощники для Array, Map, Set, Promise, Date, Uint8Array
│   ├── value.rs
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs
│   ├── wasm_js_objects.rs          # НОВОЕ: 17 тестов для js_sys
│   ├── wasm_jsvalue.rs
│   └── wasm_async.rs
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # стили демонстрации коллекций
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/js_objects.rs` — новый модуль. Содержит помощники для работы с коллекциями и встроенными объектами JavaScript: `Array`, `Object`, `Function`, `Map`, `Set`, `Promise`, `Date`, типизированные массивы. Вынесен отдельно от `value.rs`, потому что здесь речь о **конкретных типах**, а не об универсальном `JsValue`. Модуль публичный (`pub mod js_objects;`), потому что интеграционные тесты обращаются к нему через `use hello_wasm02::js_objects;`.
- `src/app.rs` — демонстрация работы с `Array`, `Map`, `Set`, `Date`, `Uint8Array` и `Promise`.
- `src/lib.rs` — добавляется `pub mod js_objects;`.
- `tests/wasm_js_objects.rs` — 17 тестов, разбитых по категориям.
- `web/styles/main.css` — заменяется целиком: стили инспектора `JsValue` из главы 10 удалены, добавлены стили секций демонстрации коллекций.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `js-sys` (docs.rs) | https://docs.rs/js-sys |
| `js-sys` (wasm-bindgen guide) | https://rustwasm.github.io/docs/wasm-bindgen/contributing/js-sys/index.html |
| `js_sys::Array` | https://docs.rs/js-sys/latest/js_sys/struct.Array.html |
| `js_sys::Object` | https://docs.rs/js-sys/latest/js_sys/struct.Object.html |
| `js_sys::Function` | https://docs.rs/js-sys/latest/js_sys/struct.Function.html |
| `js_sys::Map` | https://docs.rs/js-sys/latest/js_sys/struct.Map.html |
| `js_sys::Set` | https://docs.rs/js-sys/latest/js_sys/struct.Set.html |
| `js_sys::Promise` | https://docs.rs/js-sys/latest/js_sys/struct.Promise.html |
| `js_sys::Date` | https://docs.rs/js-sys/latest/js_sys/struct.Date.html |
| `js_sys::Uint8Array` | https://docs.rs/js-sys/latest/js_sys/struct.Uint8Array.html |
| `js_sys::Reflect` | https://docs.rs/js-sys/latest/js_sys/struct.Reflect.html |
| `JsCast` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/trait.JsCast.html |
| MDN: Map | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map |
| MDN: Set | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Set |
| MDN: Typed Arrays | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray |

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

    # События
    "Event",
    "EventTarget",
    "EventInit",
    "InputEvent",
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

**Что изменилось:** ничего. Все типы `js_sys` (`Array`, `Map`, `Set`, `Promise`, `Date`, `Uint8Array`) доступны через уже подключённый crate `js-sys`. Features `web-sys` не нужны — это ECMAScript-типы, а не Web API.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Публичные модули:
//! - `dom` — помощники для работы с DOM.
//! - `events` — помощники для работы с событиями.
//! - `value` — помощники для работы с `JsValue`.
//! - `js_objects` — помощники для работы с `Array`, `Map`, `Set`,
//!   `Promise`, `Date`, типизированными массивами.

pub mod dom;
pub mod events;
pub mod js_objects;
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

---

## `src/js_objects.rs` (новый модуль)

```rust
//! Помощники для работы с встроенными объектами JavaScript.
//!
//! `js_sys` предоставляет Rust-обёртки для всех глобальных объектов,
//! гарантированных стандартом ECMAScript: `Array`, `Object`, `Function`,
//! `Map`, `Set`, `Promise`, `Date`, типизированные массивы.
//!
//! Этот модуль — переиспользуемая библиотека. Часть функций пока
//! не используется в `app.rs`, но понадобится в следующих главах.
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use js_sys::{Array, Date, Function, Map, Object, Promise, Reflect, Set, Uint8Array};

// ============================================================
// Array
// ============================================================

/// Создаёт массив из слайса `JsValue`.
pub fn array_from_values(values: &[JsValue]) -> Array {
    let arr = Array::new();
    for v in values {
        arr.push(v);
    }
    arr
}

/// Создаёт массив чисел.
pub fn array_of_numbers(values: &[f64]) -> Array {
    let arr = Array::new();
    for n in values {
        arr.push(&JsValue::from_f64(*n));
    }
    arr
}

/// Создаёт массив из `Vec<T>` через `JsValue::from`.
pub fn array_from_vec<T: Into<JsValue> + Clone>(items: &[T]) -> Array {
    let arr = Array::new();
    for item in items {
        arr.push(&item.clone().into());
    }
    arr
}

/// Сортирует массив.
pub fn array_sort(arr: &Array) -> Array {
    arr.sort()
}

/// Объединяет массивы.
pub fn array_concat(a: &Array, b: &Array) -> Array {
    a.concat(b)
}

/// Ищет индекс элемента.
pub fn array_index_of(arr: &Array, value: &JsValue) -> i32 {
    arr.index_of(value, 0)
}

// ============================================================
// Object
// ============================================================

/// Создаёт объект из пар ключ-значение.
///
/// Использует `Reflect::set` для установки свойств. Возвращает
/// `Result`, потому что `Reflect::set` может провалиться, если
/// объект заморожен (`Object.freeze`) или свойство недоступно
/// для записи.
pub fn object_from_entries(entries: &[(&str, JsValue)]) -> Result<Object, JsValue> {
    let obj = Object::new();
    for (key, value) in entries {
        Reflect::set(obj.as_ref(), &JsValue::from_str(key), value)?;
    }
    Ok(obj)
}

/// Возвращает ключи объекта.
pub fn object_keys(obj: &Object) -> Array {
    Object::keys(obj)
}

/// Возвращает значения объекта.
pub fn object_values(obj: &Object) -> Array {
    Object::values(obj)
}

// ============================================================
// Function
// ============================================================

/// Создаёт `Function` из `js_sys::eval`.
///
/// Возвращает `Result`, потому что `eval` может провалиться
/// при синтаксической ошибке в строке.
pub fn eval_function(code: &str) -> Result<Function, JsValue> {
    js_sys::eval(code)?.dyn_into::<Function>()
}

/// Создаёт `Function` из строки аргументов и тела.
///
/// `args` — список имён аргументов, `body` — тело функции.
/// Возвращает `Function` напрямую: `Function::new_with_args`
/// не может провалиться.
pub fn function_with_args(args: &[&str], body: &str) -> Function {
    Function::new_with_args(&args.join(","), body)
}

// ============================================================
// Map
// ============================================================

/// Создаёт `Map`.
pub fn new_map() -> Map {
    Map::new()
}

/// Устанавливает пару ключ-значение.
pub fn map_set(map: &Map, key: &JsValue, value: &JsValue) -> Map {
    map.set(key, value)
}

/// Читает значение по ключу.
///
/// Возвращает `Option`: `Map::get` возвращает `undefined` для
/// отсутствующих ключей. Если ключ есть, но значение — `undefined`,
/// поведение неотличимо. Это особенность JavaScript-`Map`.
pub fn map_get(map: &Map, key: &JsValue) -> Option<JsValue> {
    let value = map.get(key);
    if value.is_undefined() {
        None
    } else {
        Some(value)
    }
}

/// Проверяет наличие ключа.
pub fn map_has(map: &Map, key: &JsValue) -> bool {
    map.has(key)
}

/// Удаляет пару.
pub fn map_delete(map: &Map, key: &JsValue) -> bool {
    map.delete(key)
}

/// Возвращает размер.
pub fn map_size(map: &Map) -> u32 {
    map.size()
}

// ============================================================
// Set
// ============================================================

/// Создаёт `Set` с пустым массивом в качестве начального значения.
pub fn new_set() -> Set {
    Set::new(&Array::new().into())
}

/// Добавляет значение.
///
/// Возвращает сам `Set` для цепочек. Дубликаты игнорируются.
pub fn set_add(set: &Set, value: &JsValue) -> Set {
    set.add(value)
}

/// Проверяет наличие значения.
pub fn set_has(set: &Set, value: &JsValue) -> bool {
    set.has(value)
}

/// Удаляет значение. Возвращает `true`, если значение было в `Set`.
pub fn set_delete(set: &Set, value: &JsValue) -> bool {
    set.delete(value)
}

/// Возвращает размер.
pub fn set_size(set: &Set) -> u32 {
    set.size()
}

// ============================================================
// Promise
// ============================================================

/// Создаёт зарезолвленный `Promise`.
pub fn promise_resolve(value: &JsValue) -> Promise {
    Promise::resolve(value)
}

/// Создаёт зареджекченный `Promise`.
pub fn promise_reject(reason: &JsValue) -> Promise {
    Promise::reject(reason)
}

/// Создаёт `Promise` через `Promise::new`.
///
/// `executor` — замыкание, принимающее `resolve` и `reject`.
/// Rust коэрцирует `&mut F` в `&mut dyn FnMut(Function, Function)`
/// автоматически, без `.into()`.
pub fn promise_new<F>(mut executor: F) -> Promise
where
    F: FnMut(Function, Function) + 'static,
{
    Promise::new(&mut executor)
}

// ============================================================
// Date
// ============================================================

/// Текущее время в миллисекундах (как `Date.now()`).
pub fn date_now() -> f64 {
    Date::now()
}

/// Создаёт `Date` из строки.
pub fn date_from_string(s: &str) -> Date {
    Date::new(&JsValue::from_str(s))
}

/// Возвращает год.
pub fn date_full_year(d: &Date) -> f64 {
    d.get_full_year() as f64
}

/// Возвращает месяц (0–11).
pub fn date_month(d: &Date) -> f64 {
    d.get_month() as f64
}

/// Возвращает день месяца.
pub fn date_date(d: &Date) -> f64 {
    d.get_date() as f64
}

/// Форматирует `Date` в ISO-строку.
pub fn date_to_iso(d: &Date) -> String {
    d.to_iso_string().as_string().unwrap_or_default()
}

// ============================================================
// Typed Arrays
// ============================================================

/// Создаёт `Uint8Array` из `Vec<u8>`.
pub fn uint8_array_from_vec(data: &[u8]) -> Uint8Array {
    Uint8Array::from(data)
}

/// Возвращает длину `Uint8Array`.
pub fn uint8_array_length(arr: &Uint8Array) -> u32 {
    arr.length()
}

/// Возвращает `Vec<u8>` из `Uint8Array`.
pub fn uint8_array_to_vec(arr: &Uint8Array) -> Vec<u8> {
    arr.to_vec()
}

/// Создаёт `Uint8Array` заданной длины, заполненный нулями.
pub fn uint8_array_zeroed(len: u32) -> Uint8Array {
    Uint8Array::new_with_length(len)
}

/// Читает элемент по индексу.
pub fn uint8_array_get(arr: &Uint8Array, index: u32) -> u8 {
    arr.get_index(index)
}

/// Устанавливает элемент по индексу.
pub fn uint8_array_set(arr: &Uint8Array, index: u32, value: u8) {
    arr.set_index(index, value);
}

/// Возвращает срез `Uint8Array`.
pub fn uint8_array_slice(arr: &Uint8Array, start: u32, end: u32) -> Uint8Array {
    arr.slice(start, end)
}
```

**Ключевые моменты:**

- **`Array`** — `new`, `push`, `concat`, `sort`, `index_of`.
- **`Object`** — `Object::keys`, `Object::values` возвращают `Array`. `Reflect::set` — установка свойств.
- **`Function`** — `function_with_args` возвращает `Function` напрямую: `Function::new_with_args` не может провалиться. **Не оборачивайте в `Result` без причины.**
- **`Map`** — `set`, `get`, `has`, `delete`, `size`.
- **`Set`** — `add`, `has`, `delete`, `size`. `Set::new` принимает `&JsValue`, `Array` приводится через `.into()`.
- **`Promise`** — `resolve`, `reject`, `new`. **`Promise::new` принимает `&mut F`, коэрция происходит автоматически, `.into()` не нужен.**
- **`Date`** — `now()`, `new()`, геттеры, `to_iso_string()`.
- **`Uint8Array`** — `from`, `to_vec`, `new_with_length`, `get_index`, `set_index`, `slice`.

---

## `src/app.rs` (демонстрация коллекций)

```rust
//! Логика приложения: «Коллекции JavaScript в Rust».
//!
//! Демонстрирует работу с `Array`, `Map`, `Set`, `Date`, `Uint8Array`
//! и `Promise`.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;

use crate::dom;
use crate::js_objects;
use crate::log;

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Коллекции JavaScript в Rust")?;
    dom::append(&container, &heading)?;

    inspect_array(&container)?;
    inspect_map(&container)?;
    inspect_set(&container)?;
    inspect_date(&container)?;
    inspect_typed_array(&container)?;
    spawn_promise_demo(&container);

    log!("Демонстрация коллекций завершена");
    Ok(())
}

fn inspect_array(container: &web_sys::Element) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;
    let title = dom::create_element("h2")?;
    title.set_text_content(Some("Array"));
    dom::append(&section, &title)?;

    let arr = js_objects::array_of_numbers(&[3.0, 1.0, 4.0, 1.0, 5.0]);
    let sorted = js_objects::array_sort(&arr);

    let extra = js_objects::array_from_values(&[JsValue::from_str("доп")]);
    let concatenated = js_objects::array_concat(&arr, &extra);

    let index_of_4 = js_objects::array_index_of(&arr, &JsValue::from_f64(4.0));

    let mut lines = Vec::new();
    lines.push(format!("Исходный: [3, 1, 4, 1, 5]"));
    lines.push(format!("Длина: {}", arr.length()));
    lines.push(format!("Сортированный: длина {}", sorted.length()));
    lines.push(format!("Конкатенация: длина {}", concatenated.length()));
    lines.push(format!("indexOf(4) = {}", index_of_4));

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("Array: length={}, indexOf(4)={}", arr.length(), index_of_4);
    Ok(())
}

fn inspect_map(container: &web_sys::Element) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;
    let title = dom::create_element("h2")?;
    title.set_text_content(Some("Map"));
    dom::append(&section, &title)?;

    let map = js_objects::new_map();
    js_objects::map_set(&map, &JsValue::from_str("name"), &JsValue::from_str("Rust"));
    js_objects::map_set(&map, &JsValue::from_str("version"), &JsValue::from_f64(1.75));
    js_objects::map_set(&map, &JsValue::from_str("wasm"), &JsValue::from_bool(true));

    let name = js_objects::map_get(&map, &JsValue::from_str("name"));
    let has_version = js_objects::map_has(&map, &JsValue::from_str("version"));
    let has_missing = js_objects::map_has(&map, &JsValue::from_str("missing"));
    let size = js_objects::map_size(&map);

    let mut lines = Vec::new();
    lines.push(format!("size: {}", size));
    lines.push(format!(
        "has(\"name\"): {}",
        js_objects::map_has(&map, &JsValue::from_str("name"))
    ));
    lines.push(format!("has(\"version\"): {}", has_version));
    lines.push(format!("has(\"missing\"): {}", has_missing));
    lines.push(format!(
        "get(\"name\"): {}",
        name.as_ref().and_then(|v| v.as_string()).unwrap_or_default()
    ));

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("Map: size={}", size);
    Ok(())
}

fn inspect_set(container: &web_sys::Element) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;
    let title = dom::create_element("h2")?;
    title.set_text_content(Some("Set"));
    dom::append(&section, &title)?;

    let set = js_objects::new_set();
    js_objects::set_add(&set, &JsValue::from_str("a"));
    js_objects::set_add(&set, &JsValue::from_str("b"));
    js_objects::set_add(&set, &JsValue::from_str("a")); // дубликат

    let has_a = js_objects::set_has(&set, &JsValue::from_str("a"));
    let has_c = js_objects::set_has(&set, &JsValue::from_str("c"));
    let size = js_objects::set_size(&set);

    let mut lines = Vec::new();
    lines.push(format!("size после add(a), add(b), add(a): {}", size));
    lines.push(format!("has(\"a\"): {}", has_a));
    lines.push(format!("has(\"c\"): {}", has_c));

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("Set: size={}", size);
    Ok(())
}

fn inspect_date(container: &web_sys::Element) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;
    let title = dom::create_element("h2")?;
    title.set_text_content(Some("Date"));
    dom::append(&section, &title)?;

    let now = js_objects::date_now();
    let date = js_objects::date_from_string("2024-01-15T12:00:00Z");

    let mut lines = Vec::new();
    lines.push(format!("Date.now(): {:.0}", now));
    lines.push(format!("ISO: {}", js_objects::date_to_iso(&date)));
    lines.push(format!("Год: {:.0}", js_objects::date_full_year(&date)));
    lines.push(format!("Месяц: {:.0} (0-11)", js_objects::date_month(&date)));
    lines.push(format!("День: {:.0}", js_objects::date_date(&date)));

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("Date: {}", js_objects::date_to_iso(&date));
    Ok(())
}

fn inspect_typed_array(container: &web_sys::Element) -> Result<(), JsValue> {
    let section = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&section), "section")?;
    let title = dom::create_element("h2")?;
    title.set_text_content(Some("Uint8Array"));
    dom::append(&section, &title)?;

    let data = vec![10u8, 20, 30, 40, 50];
    let arr = js_objects::uint8_array_from_vec(&data);

    let mut lines = Vec::new();
    lines.push(format!("Длина: {}", js_objects::uint8_array_length(&arr)));
    lines.push(format!("[0] = {}", js_objects::uint8_array_get(&arr, 0)));
    lines.push(format!("[2] = {}", js_objects::uint8_array_get(&arr, 2)));
    lines.push(format!("[4] = {}", js_objects::uint8_array_get(&arr, 4)));

    let slice = js_objects::uint8_array_slice(&arr, 1, 4);
    lines.push(format!(
        "slice(1, 4): длина {}",
        js_objects::uint8_array_length(&slice)
    ));

    let back = js_objects::uint8_array_to_vec(&arr);
    lines.push(format!("to_vec(): {:?}", back));

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    result.set_text_content(Some(&lines.join("\n")));
    dom::append(&section, &result)?;
    dom::append(container, &section)?;

    log!("Uint8Array: len={}, to_vec={:?}", arr.length(), back);
    Ok(())
}

fn spawn_promise_demo(container: &web_sys::Element) {
    let container = container.clone();
    wasm_bindgen_futures::spawn_local(async move {
        let section = dom::create_element("div").unwrap();
        dom::add_class(&dom::as_html(&section), "section").unwrap();
        let title = dom::create_element("h2").unwrap();
        title.set_text_content(Some("Promise"));
        dom::append(&section, &title).unwrap();

        let result = dom::create_element("div").unwrap();
        dom::add_class(&dom::as_html(&result), "result").unwrap();

        // Promise.resolve(42)
        let promise = js_objects::promise_resolve(&JsValue::from_f64(42.0));
        let value = JsFuture::from(promise).await;

        let text = match value {
            Ok(v) => format!("resolve(42) → {}", v.as_f64().unwrap_or(0.0)),
            Err(e) => format!("reject: {:?}", e),
        };

        result.set_text_content(Some(&text));
        dom::append(&section, &result).unwrap();
        dom::append(&container, &section).unwrap();

        log!("Promise: {}", text);
    });
}
```

**Что демонстрирует `app.rs`:**

1. **Array** — создание, сортировка, конкатенация, поиск индекса.
2. **Map** — установка, чтение, проверка наличия, размер.
3. **Set** — добавление (дубликаты игнорируются), проверка наличия, размер.
4. **Date** — `now()`, создание из строки, геттеры, ISO-формат.
5. **Uint8Array** — создание из `Vec<u8>`, чтение, срез, обратное преобразование.
6. **Promise** — `resolve` через `JsFuture` в `spawn_local`.

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — коллекции JavaScript</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Коллекции JavaScript в Rust. Откройте консоль для логов.</p>

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
   Секции демонстрации
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

**Что изменилось относительно главы 10.** Стили инспектора `JsValue` (те же `.section`, `.result`) сохранены — они одинаково применимы. Никаких изменений в CSS не требуется.

---

## `tests/wasm_js_objects.rs` (новый файл)

```rust
//! Тесты для `js_sys`: Array, Map, Set, Date, Uint8Array.
use wasm_bindgen_test::*;
use wasm_bindgen::{JsCast, JsValue};   // JsCast нужен для unchecked_ref
use js_sys::{Array, Map, Uint8Array};  // Set и Date не используются напрямую

use hello_wasm02::js_objects;

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// Array
// ============================================================

#[wasm_bindgen_test]
fn test_array_of_numbers() {
    let arr = js_objects::array_of_numbers(&[1.0, 2.0, 3.0]);
    assert_eq!(arr.length(), 3);
    assert_eq!(arr.get(0).as_f64(), Some(1.0));
    assert_eq!(arr.get(2).as_f64(), Some(3.0));
}

#[wasm_bindgen_test]
fn test_array_concat() {
    let a = js_objects::array_of_numbers(&[1.0, 2.0]);
    let b = js_objects::array_of_numbers(&[3.0, 4.0]);
    let c = js_objects::array_concat(&a, &b);

    assert_eq!(c.length(), 4);
    assert_eq!(c.get(0).as_f64(), Some(1.0));
    assert_eq!(c.get(3).as_f64(), Some(4.0));
}

#[wasm_bindgen_test]
fn test_array_index_of() {
    let arr = js_objects::array_of_numbers(&[10.0, 20.0, 30.0]);

    assert_eq!(js_objects::array_index_of(&arr, &JsValue::from_f64(20.0)), 1);
    assert_eq!(js_objects::array_index_of(&arr, &JsValue::from_f64(99.0)), -1);
}

// ============================================================
// Map
// ============================================================

#[wasm_bindgen_test]
fn test_map_basic() {
    let map = js_objects::new_map();

    js_objects::map_set(&map, &JsValue::from_str("a"), &JsValue::from_f64(1.0));
    js_objects::map_set(&map, &JsValue::from_str("b"), &JsValue::from_f64(2.0));

    assert_eq!(js_objects::map_size(&map), 2);
    assert!(js_objects::map_has(&map, &JsValue::from_str("a")));
    assert!(!js_objects::map_has(&map, &JsValue::from_str("c")));

    let val = js_objects::map_get(&map, &JsValue::from_str("a")).unwrap();
    assert_eq!(val.as_f64(), Some(1.0));
}

#[wasm_bindgen_test]
fn test_map_delete() {
    let map = js_objects::new_map();
    js_objects::map_set(&map, &JsValue::from_str("x"), &JsValue::from_f64(1.0));

    assert!(js_objects::map_delete(&map, &JsValue::from_str("x")));
    assert_eq!(js_objects::map_size(&map), 0);
    assert!(!js_objects::map_has(&map, &JsValue::from_str("x")));
}

// ============================================================
// Set
// ============================================================

#[wasm_bindgen_test]
fn test_set_basic() {
    let set = js_objects::new_set();

    js_objects::set_add(&set, &JsValue::from_str("a"));
    js_objects::set_add(&set, &JsValue::from_str("b"));
    js_objects::set_add(&set, &JsValue::from_str("a")); // дубликат

    assert_eq!(js_objects::set_size(&set), 2);
    assert!(js_objects::set_has(&set, &JsValue::from_str("a")));
    assert!(js_objects::set_has(&set, &JsValue::from_str("b")));
    assert!(!js_objects::set_has(&set, &JsValue::from_str("c")));
}

#[wasm_bindgen_test]
fn test_set_delete() {
    let set = js_objects::new_set();
    js_objects::set_add(&set, &JsValue::from_str("x"));

    assert!(js_objects::set_delete(&set, &JsValue::from_str("x")));
    assert_eq!(js_objects::set_size(&set), 0);
}

// ============================================================
// Date
// ============================================================

#[wasm_bindgen_test]
fn test_date_now() {
    let now = js_objects::date_now();
    assert!(now > 1_700_000_000_000.0); // после 2023-11-14
}

#[wasm_bindgen_test]
fn test_date_from_string() {
    let d = js_objects::date_from_string("2024-01-15T12:00:00Z");

    assert_eq!(js_objects::date_full_year(&d), 2024.0);
    assert_eq!(js_objects::date_month(&d), 0.0); // январь = 0
    assert_eq!(js_objects::date_date(&d), 15.0);

    let iso = js_objects::date_to_iso(&d);
    assert!(iso.starts_with("2024-01-15"));
}

// ============================================================
// Uint8Array
// ============================================================

#[wasm_bindgen_test]
fn test_uint8_array_from_vec() {
    let data = vec![10u8, 20, 30, 40, 50];
    let arr = js_objects::uint8_array_from_vec(&data);

    assert_eq!(js_objects::uint8_array_length(&arr), 5);
    assert_eq!(js_objects::uint8_array_get(&arr, 0), 10);
    assert_eq!(js_objects::uint8_array_get(&arr, 4), 50);
}

#[wasm_bindgen_test]
fn test_uint8_array_to_vec() {
    let data = vec![1u8, 2, 3];
    let arr = js_objects::uint8_array_from_vec(&data);
    let back = js_objects::uint8_array_to_vec(&arr);

    assert_eq!(back, data);
}

#[wasm_bindgen_test]
fn test_uint8_array_slice() {
    let data = vec![1u8, 2, 3, 4, 5];
    let arr = js_objects::uint8_array_from_vec(&data);
    let slice = js_objects::uint8_array_slice(&arr, 1, 4);

    assert_eq!(js_objects::uint8_array_length(&slice), 3);
    assert_eq!(js_objects::uint8_array_to_vec(&slice), vec![2u8, 3, 4]);
}

#[wasm_bindgen_test]
fn test_uint8_array_zeroed() {
    let arr = js_objects::uint8_array_zeroed(4);
    assert_eq!(js_objects::uint8_array_length(&arr), 4);
    assert_eq!(js_objects::uint8_array_to_vec(&arr), vec![0u8, 0, 0, 0]);
}

#[wasm_bindgen_test]
fn test_uint8_array_set_get() {
    let arr = js_objects::uint8_array_zeroed(3);

    js_objects::uint8_array_set(&arr, 0, 100);
    js_objects::uint8_array_set(&arr, 1, 200);

    assert_eq!(js_objects::uint8_array_get(&arr, 0), 100);
    assert_eq!(js_objects::uint8_array_get(&arr, 1), 200);
    assert_eq!(js_objects::uint8_array_get(&arr, 2), 0);
}

// ============================================================
// Проверка типов через JsCast
// ============================================================

#[wasm_bindgen_test]
fn test_array_is_js_sys_type() {
    let arr = Array::new();
    arr.push(&JsValue::from_f64(1.0));

    let generic: &JsValue = arr.as_ref();
    assert!(generic.is_array());

    // `unchecked_ref` — метод трейта `JsCast`,
    // который должен быть в области видимости.
    let back: &Array = generic.unchecked_ref();
    assert_eq!(back.length(), 1);
}

#[wasm_bindgen_test]
fn test_map_is_object() {
    let map = Map::new();
    js_objects::map_set(&map, &JsValue::from_str("k"), &JsValue::from_f64(1.0));

    let generic: &JsValue = map.as_ref();
    assert!(generic.is_object());
    // Map не является массивом
    assert!(!generic.is_array());
}

#[wasm_bindgen_test]
fn test_uint8_array_is_object() {
    let arr = Uint8Array::new_with_length(4);
    let generic: &JsValue = arr.as_ref();

    assert!(generic.is_object());
    assert!(!generic.is_array());
}
```

**Что проверяют тесты:**

- **Array:** создание, конкатенация, поиск индекса.
- **Map:** установка, чтение, проверка наличия, удаление, размер.
- **Set:** добавление (с дубликатами), проверка наличия, удаление.
- **Date:** `now()`, создание из строки, геттеры, ISO-формат.
- **Uint8Array:** создание из `Vec<u8>`, обратное преобразование, срез, установка/чтение элементов.
- **JsCast:** проверка, что `Array`, `Map`, `Uint8Array` — это объекты JavaScript.

**Ключевые моменты:**

- **`use wasm_bindgen::JsCast;`** — обязателен для `unchecked_ref`, `dyn_into`, `unchecked_into`. Без импорта трейта методы недоступны, даже если тип его реализует.
- **`Set` и `Date` не импортируются**, потому что работа с ними идёт через функции `js_objects::set_*` и `js_objects::date_*`.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. В контейнере `#app` появятся секции: «Array», «Map», «Set», «Date», «Uint8Array», «Promise».

### Тесты

```bash
# Хостовые тесты
npm run test:host

# Wasm-логика в Node.js
npm run test:node

# DOM, события, js_sys, JsValue в headless Chrome
npm run test:chrome

# То же в headless Firefox
npm run test:firefox

# Основной набор: host + node + chrome
npm test

# Полный прогон
npm run test:all
```

### Проверка конкретного теста

```bash
wasm-pack test --headless --chrome --test wasm_js_objects
```

---

## Ключевые концепции главы

**1. `js_sys` vs `web_sys` vs `wasm_bindgen`.**

| Crate | Что покрывает | Пример |
|---|---|---|
| `wasm-bindgen` | Ядро: `JsValue`, `Closure`, `JsCast` | `JsValue`, `Closure` |
| `js_sys` | ECMAScript: `Array`, `Map`, `Set`, `Promise`, `Date`, типизированные массивы | `js_sys::Array` |
| `web_sys` | Web Platform: `Window`, `Document`, `Element`, `fetch`, `Canvas` | `web_sys::HtmlElement` |

`js_sys` не содержит привязок к Web API — только к глобальным объектам, гарантированным стандартом ECMAScript.

**2. `Array`.** `Array::new()` создаёт пустой массив. `push`, `concat`, `sort`, `index_of` — основные методы. `arr.get(i)` возвращает `JsValue`.

**3. `Map` и `Set`.** `Map` хранит пары ключ-значение, `Set` — уникальные значения. Методы `set`/`add` возвращают сам объект для цепочек. `size()` возвращает `u32`.

**4. `Promise`.** `Promise::resolve(v)`, `Promise::reject(reason)`, `Promise::new(executor)`. Для интеграции с Rust `Future` используйте `wasm_bindgen_futures::JsFuture::from(promise).await`.

**5. `Date`.** `Date::now()` — текущее время в миллисекундах. `Date::new(&JsValue::from_str(...))` — создание из строки. Геттеры: `get_full_year()`, `get_month()` (0–11), `get_date()`, `get_hours()`. `to_iso_string()` возвращает ISO-8601.

**6. Типизированные массивы.** `Uint8Array` — для бинарных данных. `Uint8Array::from(&[u8])`, `to_vec()`, `new_with_length()`, `get_index()`, `set_index()`, `slice()`. Ключевой тип для работы с `ArrayBuffer` и бинарными данными.

**7. `JsCast` для `js_sys`-типов.** `Array`, `Map`, `Set` реализуют `JsCast`. `JsValue` можно привести к конкретному типу через `unchecked_ref` или `dyn_into` после проверки.

**8. Когда использовать `js_sys`.**

- Работа с ECMAScript-объектами: `Array`, `Map`, `Set`, `Promise`, `Date`, `RegExp`.
- Типизированные массивы: `Uint8Array`, `Float32Array`, `Int32Array`.
- `Reflect` для доступа к свойствам произвольных объектов.
- `JSON`, `Math`, `Number`, `String` — стандартные встроенные объекты.

**9. Когда использовать `web_sys`.**

- Работа с DOM: `Window`, `Document`, `Element`, `Node`.
- События: `EventTarget`, `MouseEvent`, `KeyboardEvent`.
- Сетевые API: `fetch`, `WebSocket`, `Request`, `Response`.
- Медиа, Storage, Canvas, WebGL, WebGPU.

**10. Когда достаточно `wasm_bindgen`.**

- Базовые типы: `JsValue`, `Closure`, `JsCast`.
- Макросы: `#[wasm_bindgen]`, `#[wasm_bindgen(start)]`.
- Атрибуты: `js_name`, `extends`, `catch`.
- `JsValue::from`, `JsValue::from_str`, `JsValue::NULL`, `JsValue::UNDEFINED`.

**11. Возвращаемые типы должны соответствовать методам `js_sys`.** Если метод не может провалиться, обёртка в `Result` не нужна. `Function::new_with_args` возвращает `Function`, значит и наша обёртка должна возвращать `Function`.

**12. `.into()` требует однозначности.** Когда целевой тип не выводится из контекста, компилятор ошибётся. `Promise::new(&mut executor)` использует коэрцию `&mut F` → `&mut dyn FnMut(...)`, которой хватает без `.into()`.

**13. Трейты должны быть в области видимости.** Метод `unchecked_ref` — часть трейта `JsCast`. Без `use wasm_bindgen::JsCast;` его не видно, даже если тип его реализует.

---

## Итог главы

- `js_sys` — привязки к ECMAScript-объектам: `Array`, `Object`, `Function`, `Map`, `Set`, `Promise`, `Date`, типизированные массивы.
- `Array` — `new`, `push`, `concat`, `sort`, `index_of`, `length`, `get`.
- `Map` — `new`, `set`, `get`, `has`, `delete`, `size`.
- `Set` — `new`, `add`, `has`, `delete`, `size`. Дубликаты игнорируются.
- `Promise` — `resolve`, `reject`, `new`. Интеграция с `Future` через `JsFuture`.
- `Date` — `now()`, `new()`, геттеры, `to_iso_string()`.
- `Uint8Array` — `from`, `to_vec`, `new_with_length`, `get_index`, `set_index`, `slice`.
- `js_sys` не содержит Web API — только ECMAScript.
- `web_sys` содержит Web API — DOM, события, сеть, медиа.
- `wasm_bindgen` — ядро: `JsValue`, `Closure`, `JsCast`.
- `JsCast` должен быть в области видимости для `unchecked_ref`, `dyn_into`.
- Возвращаемые типы должны соответствовать методам `js_sys`: не оборачивайте в `Result`, если метод не может провалиться.
- `Promise::new` принимает `&mut F` — `.into()` не нужен.
- Практический проект: демонстрация `Array`, `Map`, `Set`, `Date`, `Uint8Array`, `Promise` в браузере.

---

**Что дальше.** В главе 12 мы разберём **JavaScript callbacks из Rust** глубже: `Closure`, `Fn`/`FnMut`/`FnOnce`, время жизни callback, `forget`, хранение callback в структуре, event handlers, callback с аргументами и возвращаемым значением. Практический проект — «Rust-компонент с callback API».