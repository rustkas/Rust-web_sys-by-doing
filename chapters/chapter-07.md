# Глава 7. `Window`, `Document` и DOM

## Обновлённая структура проекта

```
hello-wasm02/
├── Cargo.toml
├── package.json
├── vite.config.js
├── webdriver.json
├── .gitignore
├── LICENSE                         # текст лицензии (MIT)
├── src/
│   ├── lib.rs                      # pub mod dom; mod app; mod logging;
│   ├── app.rs                      # логика приложения (Rust управляет HTML)
│   ├── dom.rs                      # помощники для работы с DOM
│   └── logging.rs                  # инициализация логирования
├── tests/
│   ├── common.rs                   # общие помощники
│   ├── pure_logic.rs               # хостовые тесты
│   ├── wasm_logic.rs               # Wasm-тесты логики (Node.js)
│   ├── wasm_dom.rs                 # DOM-тесты (headless-браузер)
│   ├── wasm_events.rs              # тесты событий
│   └── wasm_async.rs               # асинхронные тесты
├── web/
│   ├── index.html                  # с контейнером <div id="app">
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # стили для #app, кнопок, списка
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/dom.rs` — библиотека помощников для работы с DOM. Расширяется функциями поиска, создания, работы с атрибутами и классами.
- `src/app.rs` — практический проект «Rust управляет HTML»: строит заголовок, список и кнопку с классами внутри контейнера `#app`.
- `src/lib.rs` — объявляет `pub mod dom;`, чтобы doc-тесты и внешние интеграционные тесты могли обращаться к модулю.
- `Cargo.toml` дополняется features `DomTokenList`, `NamedNodeMap`, `NodeList`.
- `web/index.html` содержит контейнер `<div id="app"></div>` — точку монтирования для Rust.
- `tests/wasm_dom.rs` дополняется тестами `test_set_attribute`, `test_class_list`, `test_query_selector`.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `web_sys` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/ |
| `Window` | https://docs.rs/web-sys/latest/web_sys/struct.Window.html |
| `Document` | https://docs.rs/web-sys/latest/web_sys/struct.Document.html |
| `Node` | https://docs.rs/web-sys/latest/web_sys/struct.Node.html |
| `Element` | https://docs.rs/web-sys/latest/web_sys/struct.Element.html |
| `HtmlElement` | https://docs.rs/web-sys/latest/web_sys/struct.HtmlElement.html |
| `DomTokenList` | https://docs.rs/web-sys/latest/web_sys/struct.DomTokenList.html |
| Наследование в `web_sys` | https://rustwasm.github.io/wasm-bindgen/guide/web-sys/inheritance.html |
| DOM hello world (пример) | https://rustwasm.github.io/docs/wasm-bindgen/examples/dom.html |
| MDN: DOM | https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model |

---

## `Cargo.toml` (обновлённый)

```toml
[package]
name = "hello-wasm02"
version = "0.1.0"
edition = "2021"
description = "Rust + web_sys starter"
license = "MIT"

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

    # Поиск и создание элементов
    "NodeList",           # query_selector_all
    "HtmlCollection",     # Element::children()

    # Атрибуты и классы
    "DomTokenList",       # classList
    "NamedNodeMap",       # attributes

    # События
    "Event",
    "EventTarget",

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

**Почему добавлены features:**

- **`NodeList`** — `Document::query_selector_all()` возвращает `NodeList`.
- **`DomTokenList`** — `HtmlElement::class_list()` возвращает `DomTokenList`.
- **`NamedNodeMap`** — `Element::attributes()` возвращает `NamedNodeMap`.
- **`EventTarget`** — явно указан, потому что `add_event_listener_with_callback` живёт в `EventTarget`.

Правило `web_sys`: **если метод возвращает тип `X`, нужна feature `X`**, даже если вы не используете `X` напрямую.

### `LICENSE`

Файл с текстом лицензии MIT. Стандартный шаблон:

```
MIT License

Copyright (c) 2026 <Ваше имя>

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Наличие файла `LICENSE` убирает предупреждение `wasm-pack`: *«License key is set in Cargo.toml but no LICENSE file(s) were found»*.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Точка входа — функция `run`, которая вызывается автоматически
//! при инициализации Wasm-модуля.
//!
//! Модуль `dom` публичен: он содержит переиспользуемые помощники
//! для работы с DOM. Публичность нужна для doc-тестов и внешних
//! интеграционных тестов.

pub mod dom;
mod app;
mod logging;

use wasm_bindgen::prelude::*;

/// Точка входа. Вызывается автоматически при инициализации Wasm-модуля.
#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    logging::init();
    app::run()
}
```

**Ключевой момент:** `pub mod dom;` — модуль публичен, потому что его функции используются в doc-тестах и могут использоваться из внешних тестов. `app` и `logging` остаются приватными: они — внутренняя кухня приложения.

---

## `src/dom.rs`

```rust
//! Помощники для работы с DOM.
//!
//! Каждая функция возвращает `Result` или `Option`, повторяя сигнатуры
//! `web_sys`. Это позволяет вызывающему коду решать, как обрабатывать
//! ошибки — через `?`, `unwrap()` или `match`.
//!
//! Этот модуль — переиспользуемая библиотека. В текущем `app.rs`
//! используется только часть функций; остальные понадобятся
//! в следующих главах. Подавляем `dead_code`, чтобы предупреждения
//! не засоряли вывод сборки.
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Document, Element, HtmlElement, Node, Window};

// ============================================================
// Получение базовых объектов
// ============================================================

/// Возвращает глобальный объект `window`.
///
/// # Panics
///
/// Паникует, если код выполняется вне браузерного контекста
/// (например, в Node.js без `jsdom`).
///
/// # Пример
///
/// ```no_run
/// use hello_wasm02::dom;
/// let w = dom::window();
/// assert!(w.document().is_some());
/// ```
///
/// Атрибут `no_run` нужен потому, что doc-тесты выполняются
/// на хосте, а `web_sys::window()` доступен только в браузере.
pub fn window() -> Window {
    web_sys::window().expect("no global `window` exists")
}

/// Возвращает `document` из `window`.
///
/// # Panics
///
/// Паникует, если у `window` нет `document`.
pub fn document() -> Document {
    window()
        .document()
        .expect("should have a document on window")
}

/// Возвращает `<body>` из `document`.
///
/// # Panics
///
/// Паникует, если у документа нет `<body>`.
pub fn body() -> HtmlElement {
    document()
        .body()
        .expect("document should have a body")
        .unchecked_into()
}

// ============================================================
// Поиск элементов
// ============================================================

/// Ищет элемент по CSS-селектору. Возвращает первый совпавший.
pub fn query_selector(selector: &str) -> Result<Option<Element>, JsValue> {
    document().query_selector(selector)
}

/// Ищет все элементы по CSS-селектору.
pub fn query_selector_all(selector: &str) -> Result<web_sys::NodeList, JsValue> {
    document().query_selector_all(selector)
}

/// Ищет элемент по `id`. Возвращает `None`, если не найден.
pub fn get_element_by_id(id: &str) -> Option<Element> {
    document().get_element_by_id(id)
}

// ============================================================
// Создание элементов
// ============================================================

/// Создаёт элемент с заданным тегом.
pub fn create_element(tag: &str) -> Result<Element, JsValue> {
    document().create_element(tag)
}

/// Создаёт `<p>` с заданным текстом.
pub fn create_paragraph(text: &str) -> Result<Element, JsValue> {
    let p = create_element("p")?;
    p.set_text_content(Some(text));
    Ok(p)
}

/// Создаёт `<h1>`–`<h6>` с заданным текстом.
pub fn create_heading(level: u8, text: &str) -> Result<Element, JsValue> {
    let tag = format!("h{}", level.clamp(1, 6));
    let h = create_element(&tag)?;
    h.set_text_content(Some(text));
    Ok(h)
}

/// Создаёт `<li>` с заданным текстом.
pub fn create_list_item(text: &str) -> Result<Element, JsValue> {
    let li = create_element("li")?;
    li.set_text_content(Some(text));
    Ok(li)
}

// ============================================================
// Вложенность DOM
// ============================================================

/// Добавляет дочерний узел к родителю.
pub fn append(parent: &Node, child: &Node) -> Result<(), JsValue> {
    parent.append_child(child)?;
    Ok(())
}

/// Удаляет элемент из DOM.
///
/// Принимает `&Element`, а не `&Node`: метод `remove` определён
/// в WebIDL-миксине `ChildNode`, который реализует `Element`,
/// а не `Node`.
pub fn remove(el: &Element) {
    el.remove();
}

/// Удаляет произвольный узел из DOM через родителя.
///
/// Полезно, когда узел не является `Element` (например, текстовый узел).
pub fn remove_node(node: &Node) -> Result<(), JsValue> {
    if let Some(parent) = node.parent_node() {
        parent.remove_child(node)?;
    }
    Ok(())
}

// ============================================================
// Атрибуты
// ============================================================

/// Устанавливает атрибут на элементе.
pub fn set_attribute(el: &Element, name: &str, value: &str) -> Result<(), JsValue> {
    el.set_attribute(name, value)
}

/// Читает атрибут. Возвращает `None`, если атрибута нет.
pub fn get_attribute(el: &Element, name: &str) -> Option<String> {
    el.get_attribute(name)
}

/// Удаляет атрибут.
pub fn remove_attribute(el: &Element, name: &str) -> Result<(), JsValue> {
    el.remove_attribute(name)
}

// ============================================================
// Классы
// ============================================================

/// Добавляет CSS-класс элементу.
pub fn add_class(el: &HtmlElement, class: &str) -> Result<(), JsValue> {
    el.class_list().add_1(class)
}

/// Удаляет CSS-класс.
pub fn remove_class(el: &HtmlElement, class: &str) -> Result<(), JsValue> {
    el.class_list().remove_1(class)
}

/// Проверяет наличие CSS-класса.
///
/// Возвращает `bool`, а не `Result`: `DomTokenList::contains` —
/// чистая проверка без возможности ошибки.
pub fn has_class(el: &HtmlElement, class: &str) -> bool {
    el.class_list().contains(class)
}

// ============================================================
// Приведение типов
// ============================================================

/// Приводит `Element` к `HtmlElement` без проверки типа.
///
/// # Panics
///
/// Паникует, если элемент не является `HtmlElement`
/// (например, это SVG-элемент).
pub fn as_html(el: &Element) -> HtmlElement {
    el.clone().unchecked_into()
}

/// Безопасно пытается привести `Element` к `HtmlElement`.
pub fn try_as_html(el: &Element) -> Option<HtmlElement> {
    el.clone().dyn_into::<HtmlElement>().ok()
}
```

**Ключевые моменты:**

- **`window()` содержит doc-тест с `no_run`** — компилируется, но не выполняется на хосте, потому что `web_sys::window()` использует статики `js-sys`, которых на нативном таргете нет.
- **`#![allow(dead_code)]`** подавляет warnings о неиспользуемых функциях. Модуль — библиотека на весь проект.
- **`body()` возвращает `HtmlElement`** через `unchecked_into()` — `<body>` всегда HTML-элемент, проверка типа не нужна.
- **`remove` принимает `&Element`**, а не `&Node`. Метод `remove` определён в mix-in `ChildNode`, который реализует `Element`.
- **`remove_node` — альтернатива для произвольных `Node`** через родителя.
- **`has_class` возвращает `bool`**, не `Result`: `DomTokenList::contains` не может провалиться.
- **`as_html` использует `unchecked_into`** (быстро, но паникует при неверном типе), `try_as_html` — `dyn_into` (безопасно, возвращает `Option`).

---

## `src/app.rs`

```rust
//! Логика приложения: «Rust управляет HTML».
//!
//! Строим небольшую структуру DOM внутри контейнера `#app`:
//!
//! ```text
//! <div id="app">
//!   <h1>Rust + web_sys</h1>
//!   <p>Демонстрация работы с DOM из Rust.</p>
//!   <ul>
//!     <li>Window</li>
//!     <li>Document</li>
//!     <li>Element</li>
//!     <li>HtmlElement</li>
//!   </ul>
//!   <button class="primary" type="button">Click me</button>
//! </div>
//! ```

use wasm_bindgen::prelude::*;

use crate::dom;
use crate::log;

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    // 1. Находим контейнер, в который будем строить DOM.
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    // 2. Заголовок
    let heading = dom::create_heading(1, "Rust + web_sys")?;
    dom::append(&container, &heading)?;

    // 3. Описание
    let description = dom::create_paragraph("Демонстрация работы с DOM из Rust.")?;
    dom::append(&container, &description)?;

    // 4. Список
    let list = dom::create_element("ul")?;
    for item_text in ["Window", "Document", "Element", "HtmlElement"] {
        let item = dom::create_list_item(item_text)?;
        dom::append(&list, &item)?;
    }
    dom::append(&container, &list)?;

    // 5. Кнопка с атрибутами и классами
    let button = dom::create_element("button")?;
    button.set_text_content(Some("Click me"));
    dom::set_attribute(&button, "type", "button")?;
    dom::set_attribute(&button, "data-role", "action")?;

    let button_html = dom::as_html(&button);
    dom::add_class(&button_html, "primary")?;
    dom::add_class(&button_html, "large")?;

    assert!(dom::has_class(&button_html, "primary"));
    assert!(dom::has_class(&button_html, "large"));
    assert!(!dom::has_class(&button_html, "secondary"));

    // 6. Читаем атрибут
    let role = dom::get_attribute(&button, "data-role")
        .unwrap_or_else(|| "(no role)".into());
    log!("Rust: кнопка имеет data-role = {}", role);

    // 7. Демонстрируем remove: создаём временный элемент и сразу удаляем.
    let temp = dom::create_element("div")?;
    temp.set_text_content(Some("temporary"));
    dom::append(&container, &temp)?;
    log!("Rust: до удаления элементов: {}", container.child_element_count());

    dom::remove(&temp);
    log!("Rust: после удаления элементов: {}", container.child_element_count());

    // 8. Демонстрируем query_selector.
    let found = dom::query_selector("button[data-role='action']")?;
    match found {
        Some(el) => log!("Rust: найден button по селектору: <{}>", el.tag_name()),
        None => log!("Rust: button по селектору не найден"),
    }

    // 9. Демонстрируем query_selector_all.
    let items = dom::query_selector_all("li")?;
    log!("Rust: найдено <li>: {}", items.length());

    // 10. Демонстрируем remove_attribute и remove_class.
    dom::remove_attribute(&button, "data-role")?;
    dom::remove_class(&button_html, "large")?;
    log!(
        "Rust: после очистки — data-role = {:?}, has large = {}",
        dom::get_attribute(&button, "data-role"),
        dom::has_class(&button_html, "large")
    );

    // 11. Демонстрируем body().
    log!("Rust: body tag = <{}>", dom::body().tag_name());

    // 12. Добавляем кнопку в контейнер.
    dom::append(&container, &button)?;

    log!(
        "Rust: DOM построен. Элементов в контейнере: {}",
        container.child_element_count()
    );

    Ok(())
}
```

**Что демонстрирует `app.rs`:**

- Поиск контейнера по `id`.
- Создание заголовка, абзаца, списка, кнопки.
- Установка атрибутов (`type`, `data-role`).
- Добавление и удаление классов.
- Чтение атрибута через `get_attribute`.
- Удаление элемента через `remove`.
- Поиск по CSS-селектору через `query_selector` и `query_selector_all`.
- Получение `body()`.
- Логирование результатов в консоль браузера.

---

## `src/logging.rs`

```rust
/// Инициализация инфраструктуры логирования.
///
/// Устанавливает panic hook, чтобы паники в Rust отображались
/// в консоли браузера с нормальным сообщением и стеком.
pub fn init() {
    console_error_panic_hook::set_once();
}

/// Логирование в консоль браузера.
#[macro_export]
macro_rules! log {
    ($($t:tt)*) => {
        web_sys::console::log_1(&format!($($t)*).into())
    };
}
```

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — Rust управляет HTML</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Откройте консоль разработчика, чтобы увидеть вывод Rust.</p>

    <!-- Контейнер, в который Rust будет строить DOM -->
    <div id="app"></div>

    <script type="module" src="/main.js"></script>
</body>
</html>
```

**Что важно:** контейнер `<div id="app"></div>` — точка монтирования. Rust ищет его по `id` и наполняет. HTML служит «скелетом»; структура — на стороне Rust.

---

## `web/styles/main.css`

```css
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

#app {
    margin-top: 1.5rem;
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
}

button.primary {
    background: #b7410e;
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    border-radius: 4px;
    cursor: pointer;
}

button.primary:hover {
    background: #8f300b;
}

button.large {
    font-size: 1.1rem;
}

ul {
    padding-left: 1.5rem;
}
```

---

## `tests/wasm_dom.rs` (дополненный)

```rust
//! Тесты, требующие реального браузера (DOM).
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

mod common;

use wasm_bindgen::JsCast;
use web_sys::HtmlElement;

#[wasm_bindgen_test]
fn test_document_exists() {
    let document = web_sys::window()
        .expect("no window")
        .document()
        .expect("no document");
    assert!(document.body().is_some());
}

#[wasm_bindgen_test]
fn test_create_paragraph() {
    let fixture = common::create_fixture();

    let p = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("p")
        .unwrap();
    p.set_text_content(Some("test paragraph"));
    fixture.append_child(&p).unwrap();

    common::assert_has_child(&fixture, "p");

    let text = fixture
        .first_child()
        .unwrap()
        .dyn_into::<HtmlElement>()
        .unwrap()
        .text_content()
        .unwrap();
    assert_eq!(text, "test paragraph");

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_set_attribute() {
    let fixture = common::create_fixture();

    fixture.set_attribute("data-role", "test").unwrap();
    assert_eq!(fixture.get_attribute("data-role").unwrap(), "test");

    fixture.remove_attribute("data-role").unwrap();
    assert!(fixture.get_attribute("data-role").is_none());

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_class_list() {
    let fixture = common::create_fixture();
    let html: HtmlElement = fixture.clone().unchecked_into();

    html.class_list().add_1("foo").unwrap();
    html.class_list().add_1("bar").unwrap();

    assert!(html.class_list().contains("foo"));
    assert!(html.class_list().contains("bar"));
    assert!(!html.class_list().contains("baz"));

    html.class_list().remove_1("foo").unwrap();
    assert!(!html.class_list().contains("foo"));
    assert!(html.class_list().contains("bar"));

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_query_selector() {
    let fixture = common::create_fixture();

    let child = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("span")
        .unwrap();
    child.set_attribute("data-test-child", "yes").unwrap();
    fixture.append_child(&child).unwrap();

    let found = fixture
        .query_selector("[data-test-child='yes']")
        .unwrap()
        .unwrap();
    assert_eq!(found.tag_name(), "SPAN");

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
async fn test_promise_resolve() {
    use wasm_bindgen_futures::JsFuture;

    let promise = js_sys::Promise::resolve(&wasm_bindgen::JsValue::from(42));
    let result = JsFuture::from(promise).await.unwrap();
    assert_eq!(result.as_f64().unwrap(), 42.0);
}
```

**Обратите внимание на `test_class_list`:**

- `add_1` и `remove_1` возвращают `Result<(), JsValue>` → `.unwrap()`.
- `contains` возвращает `bool` → никакого `?` или `unwrap`.

---

## Команды запуска

### Разработка

```bash
# Debug-сборка Wasm + Vite dev server
npm start
```

Откройте `http://127.0.0.1:5173`. В контейнере `#app` появится структура, построенная Rust: заголовок, описание, список из четырёх пунктов и кнопка с классами.

### Тесты

```bash
# Хостовые тесты (чистый Rust)
npm run test:host

# Только doc-тесты
npm run test:doc

# Wasm-тесты логики в Node.js
npm run test:node

# DOM-тесты в headless Chrome
npm run test:chrome

# DOM-тесты в headless Firefox (WSL2, требует xvfb)
npm run test:firefox

# Основной набор: host + node + chrome
npm test

# Полный прогон: host + node + chrome + firefox
npm run test:all
```

### Проверка конкретного теста

```bash
# Запустить только тесты DOM в Chrome
wasm-pack test --headless --chrome --test wasm_dom

# Запустить с подробным выводом
RUST_LOG=wasm_bindgen_test_runner wasm-pack test --headless --chrome
```

### Проверка сборки

```bash
# Компиляция без запуска
cargo check --target wasm32-unknown-unknown

# Только doc-тесты (проверяют, что примеры компилируются)
cargo test --doc
```

---

## Что видно в консоли

После `npm start` в DevTools:

```
Rust: кнопка имеет data-role = action
Rust: до удаления элементов: 4
Rust: после удаления элементов: 3
Rust: найден button по селектору: <BUTTON>
Rust: найдено <li>: 4
Rust: после очистки — data-role = None, has large = false
Rust: body tag = <BODY>
Rust: DOM построен. Элементов в контейнере: 4
```

На странице: заголовок «Rust + web_sys», описание, список из четырёх пунктов, кнопка «Click me» с классом `primary`. Класс `large` и атрибут `data-role` удалены в процессе.

---

## Ключевые концепции главы

**1. Иерархия DOM в `web_sys`.** `Window` → `Document` → `Node` → `Element` → `HtmlElement` → конкретные HTML-элементы. Каждый уровень добавляет методы. `Node` даёт `append_child`, `remove_child`, `parent_node`. `Element` — `set_attribute`, `get_attribute`, `query_selector`, `children`, `remove`. `HtmlElement` — `class_list`, `style`, `inner_text`.

**2. `Deref` для наследования.** `web_sys` реализует `Deref` к родительскому классу. Если у вас есть `&HtmlElement`, вы можете передать его туда, где ожидается `&Element` или `&Node` — компилятор применит `Deref` автоматически.

**3. Не все методы есть на всех типах.** `remove` определён в mix-in `ChildNode`, который реализует `Element`, но не `Node`. Если метод не найден — проверьте документацию типа на docs.rs.

**4. `unchecked_into` vs `dyn_into`.** Первое — статическое приведение без проверки, быстрее, но паникует при неверном типе. Второе — динамическая проверка через `instanceof`, безопаснее, возвращает `Result`.

**5. Features для возвращаемых типов.** `query_selector_all` возвращает `NodeList` → нужна feature `NodeList`. `class_list` возвращает `DomTokenList` → нужна feature `DomTokenList`. `attributes` возвращает `NamedNodeMap` → нужна feature `NamedNodeMap`.

**6. Возвращаемые типы важны.** `class_list().contains()` возвращает `bool`, а не `Result`. Обёртка `Result` вокруг «чистых» проверок избыточна: если функция не может провалиться — она не должна возвращать `Result`.

**7. Doc-тесты для браузерозависимого кода — только с `no_run`.** Doc-тесты выполняются на хосте, где нет `web_sys`. Атрибут `no_run` компилирует пример, но не запускает. Реальные проверки — в `tests/wasm_dom.rs` с `#[wasm_bindgen_test]`.

---

## Итог главы

- `Window` — глобальный объект, даёт доступ к `document`, `alert`, `set_timeout`.
- `Document` — корень DOM, методы `create_element`, `query_selector`, `get_element_by_id`.
- `Node` — базовый узел, методы `append_child`, `remove_child`, `parent_node`.
- `Element` — узел-элемент, методы `set_attribute`, `get_attribute`, `remove`, `query_selector`.
- `HtmlElement` — HTML-элемент, методы `class_list`, `style`, `inner_text`, `dataset`.
- Наследование выражено через `Deref`: `HtmlElement` можно использовать там, где ожидается `Element` или `Node`.
- `unchecked_into` — быстрое статическое приведение; `dyn_into` — безопасное динамическое.
- Features `NodeList`, `DomTokenList`, `NamedNodeMap` нужны для методов, возвращающих соответствующие типы.
- Doc-тесты для браузерозависимого кода — с `no_run`; модуль должен быть `pub mod dom;`.
- Практический проект: Rust строит DOM внутри контейнера `#app` — заголовок, список, кнопку с классами.

---

**Что дальше.** В главе 8 мы разберём **типизированный DOM**: `HtmlInputElement`, `HtmlButtonElement`, `HtmlTextAreaElement`, `HtmlSelectElement`, `HtmlOptionElement`, `dyn_into`, `JsCast`, проверку типа и практический проект — «форма на Rust».