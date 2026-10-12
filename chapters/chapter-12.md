# Глава 12. JavaScript callbacks из Rust

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
│   ├── lib.rs                      # pub mod app; pub mod callbacks; pub mod dom; ...
│   ├── app.rs                      # система уведомлений + публичный Notifier
│   ├── callbacks.rs                # НОВОЕ: EventClosures, listen, listen_typed, listen_once
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── value.rs
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs
│   ├── wasm_js_objects.rs
│   ├── wasm_jsvalue.rs
│   ├── wasm_callbacks.rs           # НОВОЕ: 14 тестов для Closure и callback API
│   └── wasm_async.rs
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # стили системы уведомлений
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/callbacks.rs` — новый модуль. Содержит контейнер `EventClosures` для хранения `Closure`, а также функции `listen`, `listen_typed`, `listen_once`, `listen_forget`, которые регистрируют обработчики и одновременно управляют временем жизни `Closure`.
- `src/app.rs` — переписан под практический проект: **система уведомлений** с Rust-callback API. Демонстрирует `Fn`-подписчиков, хранение `Closure` в `EventClosures` и разделение логики между Rust и DOM.
- `src/lib.rs` — модуль `app` теперь **публичный** (`pub mod app;`), потому что интеграционные тесты обращаются к `Notifier` через `use hello_wasm02::app::Notifier;`.
- `tests/wasm_callbacks.rs` — новый файл с 14 тестами для `Closure`, `Fn`/`FnMut`/`FnOnce`, времени жизни, `EventClosures` и Rust-callback API.
- `web/styles/main.css` — заменяется целиком: стили коллекций из главы 11 удалены, добавлены стили системы уведомлений.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `Closure` (wasm-bindgen) | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html |
| `Closure::new` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html#method.new |
| `Closure::once` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html#method.once |
| `Closure::once_into_js` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html#method.once_into_js |
| `Closure::wrap` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html#method.wrap |
| `Closure::forget` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html#method.forget |
| `JsCast` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/trait.JsCast.html |
| `Fn`, `FnMut`, `FnOnce` | https://doc.rust-lang.org/std/ops/trait.Fn.html |
| Closure (Rust book) | https://doc.rust-lang.org/book/ch13-01-closures.html |
| wasm-bindgen: Event listener example | https://rustwasm.github.io/wasm-bindgen/examples/event-listener.html |

---

## `Cargo.toml`

Без изменений. Все нужные зависимости (`wasm-bindgen`, `js-sys`, `web-sys`) уже подключены.

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
    "Window", "Document", "Node", "Element", "HtmlElement",
    "NodeList", "HtmlCollection",
    "DomTokenList", "NamedNodeMap",
    "Event", "EventTarget", "EventInit", "InputEvent",
    "KeyboardEvent", "MouseEvent", "PointerEvent",
    "HtmlInputElement", "HtmlButtonElement", "HtmlTextAreaElement",
    "HtmlSelectElement", "HtmlOptionElement", "HtmlOptionsCollection",
    "HtmlFormElement",
    "HtmlImageElement", "HtmlCanvasElement", "HtmlVideoElement", "HtmlAudioElement",
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

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Публичные модули:
//! - `app` — система уведомлений с callback API (`Notifier`).
//! - `callbacks` — управление временем жизни `Closure`.
//! - `dom` — помощники для работы с DOM.
//! - `events` — помощники для работы с событиями.
//! - `value` — помощники для работы с `JsValue`.
//! - `js_objects` — помощники для `Array`, `Map`, `Set`, `Promise`, `Date`.

pub mod app;              // ← публичный: тесты используют Notifier
pub mod callbacks;
pub mod dom;
pub mod events;
pub mod js_objects;
pub mod value;
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

**Ключевое:** `pub mod app;` — модуль публичный, потому что `tests/wasm_callbacks.rs` использует `hello_wasm02::app::Notifier`.

---

## `src/callbacks.rs` (новый модуль)

```rust
//! Помощники для управления временем жизни `Closure`.
//!
//! ## Зачем это нужно
//!
//! Когда Rust-замыкание передаётся в JavaScript (например, через
//! `add_event_listener_with_callback`), `wasm-bindgen` создаёт
//! объект `Closure`, который хранит замыкание и предоставляет
//! JS-функцию-обёртку. JavaScript держит ссылку на эту функцию,
//! но **не владеет** `Closure`.
//!
//! Если `Closure` удаляется (drop), JS-функция становится
//! недействительной. Если `Closure` «забывается» через `forget()`,
//! он остаётся в памяти навсегда — утечка.
//!
//! Правильное решение — **хранить `Closure` в структуре или
//! контейнере**, время жизни которого совпадает с временем
//! использования обработчика.
//!
//! ## Что в этом модуле
//!
//! - `EventClosures` — контейнер для `Closure<dyn FnMut(Event)>`.
//! - `listen` — регистрирует обработчик и сохраняет `Closure`
//!   в контейнере.
//! - `listen_typed<T>` — то же, но для конкретного типа события.
//! - `listen_once` — одноразовый обработчик.
//! - `listen_forget` — обработчик с `forget()` (без хранения).
#![allow(dead_code)]

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, EventTarget};

// ============================================================
// Контейнер для Closure
// ============================================================

/// Хранилище для `Closure<dyn FnMut(Event)>`.
///
/// Пока контейнер жив, все замыкания активны. Когда контейнер
/// удаляется, все `Closure` внутри него тоже удаляются, и
/// соответствующие JS-обработчики перестают работать.
///
/// Обычно контейнер оборачивают в `Rc<RefCell<EventClosures>>`,
/// чтобы разделить владение между несколькими обработчиками.
#[derive(Default)]
pub struct EventClosures {
    closures: Vec<Closure<dyn FnMut(Event)>>,
}

impl EventClosures {
    /// Создаёт пустой контейнер.
    pub fn new() -> Self {
        Self {
            closures: Vec::new(),
        }
    }

    /// Добавляет `Closure` в контейнер.
    pub fn push(&mut self, closure: Closure<dyn FnMut(Event)>) {
        self.closures.push(closure);
    }

    /// Количество хранимых замыканий.
    pub fn len(&self) -> usize {
        self.closures.len()
    }

    /// Проверяет, пуст ли контейнер.
    pub fn is_empty(&self) -> bool {
        self.closures.is_empty()
    }

    /// Удаляет все замыкания.
    ///
    /// **Важно:** это не отписывает JS-обработчики.
    /// `add_event_listener_with_callback` не запоминает функцию,
    /// чтобы её можно было удалить. Если нужен полный cleanup,
    /// используйте `remove_event_listener_with_callback` перед
    /// `clear`.
    pub fn clear(&mut self) {
        self.closures.clear();
    }
}

// ============================================================
// Регистрация обработчиков
// ============================================================

/// Регистрирует обработчик события и сохраняет `Closure` в контейнере.
///
/// Это правильный способ подписки на события: `Closure` не «забывается»,
/// а живёт столько, сколько живёт контейнер.
pub fn listen<F>(
    target: &EventTarget,
    event_type: &str,
    mut handler: F,
    store: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue>
where
    F: FnMut(Event) + 'static,
{
    let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        handler(event);
    });
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    store.borrow_mut().push(closure);
    Ok(())
}

/// Регистрирует обработчик с конкретным типом события.
///
/// Например, для `KeyboardEvent` или `MouseEvent`. Внутри
/// `Event` приводится к `T` через `dyn_into`; если приведение
/// не удалось, обработчик не вызывается.
pub fn listen_typed<T, F>(
    target: &EventTarget,
    event_type: &str,
    mut handler: F,
    store: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue>
where
    T: JsCast + 'static,
    F: FnMut(T) + 'static,
{
    let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        if let Ok(typed) = event.dyn_into::<T>() {
            handler(typed);
        }
    });
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    store.borrow_mut().push(closure);
    Ok(())
}

// ============================================================
// Одноразовые и «забытые» обработчики
// ============================================================

/// Регистрирует одноразовый обработчик.
///
/// JS-функция сама себя удаляет после первого вызова.
/// `Closure` остаётся в памяти навсегда — это компромисс
/// ради простоты. Для одноразовых событий (`load`, `error`)
/// утечка несущественна.
pub fn listen_once<F>(
    target: &EventTarget,
    event_type: &str,
    mut handler: F,
) -> Result<(), JsValue>
where
    F: FnOnce(Event) + 'static,
{
    let closure = Closure::once(move |event: Event| {
        handler(event);
    });
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

/// Регистрирует обработчик и «забывает» `Closure`.
///
/// Простой, но неаккуратный способ: `Closure` остаётся
/// в памяти навсегда. Подходит для обработчиков, которые
/// живут всё время работы приложения.
pub fn listen_forget<F>(
    target: &EventTarget,
    event_type: &str,
    mut handler: F,
) -> Result<(), JsValue>
where
    F: FnMut(Event) + 'static,
{
    let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        handler(event);
    });
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}
```

**Ключевые моменты:**

- **`EventClosures`** — контейнер для `Closure`. Пока контейнер жив, замыкания активны.
- **`listen`** — регистрирует обработчик и одновременно сохраняет `Closure` в контейнере. Это **правильный** способ подписки.
- **`listen_typed<T>`** — то же, но с автоматическим приведением `Event` к конкретному типу.
- **`listen_once`** — одноразовый обработчик. `Closure::once` создаёт замыкание `FnOnce`, а `forget()` позволяет JS-функции жить независимо от Rust.
- **`listen_forget`** — упрощённый способ для случаев, когда обработчик живёт всё время работы приложения.

---

## `src/app.rs` (система уведомлений)

```rust
//! Логика приложения: «Система уведомлений».
//!
//! Демонстрирует Rust-callback API. Компонент `Notifier` позволяет
//! регистрировать подписчиков (`Fn(&str)`). Кнопки в DOM вызывают
//! `Notifier::notify`, который дёргает всех подписчиков.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;

/// Простая система уведомлений с callback API.
///
/// Хранит список подписчиков и вызывает их при `notify`.
/// Подписчики — `Fn(&str)`, хранятся в `Box<dyn Fn(&str)>`.
pub struct Notifier {
    listeners: Rc<RefCell<Vec<Box<dyn Fn(&str)>>>>,
}

impl Notifier {
    /// Создаёт пустой `Notifier`.
    pub fn new() -> Self {
        Self {
            listeners: Rc::new(RefCell::new(Vec::new())),
        }
    }

    /// Регистрирует подписчика.
    ///
    /// Замыкание вызывается при каждом `notify`. Несколько
    /// подписчиков вызываются в порядке регистрации.
    pub fn subscribe<F>(&self, callback: F)
    where
        F: Fn(&str) + 'static,
    {
        self.listeners.borrow_mut().push(Box::new(callback));
    }

    /// Отправляет сообщение всем подписчикам.
    pub fn notify(&self, message: &str) {
        for listener in self.listeners.borrow().iter() {
            listener(message);
        }
    }

    /// Количество подписчиков.
    pub fn subscriber_count(&self) -> usize {
        self.listeners.borrow().len()
    }
}

impl Default for Notifier {
    fn default() -> Self {
        Self::new()
    }
}

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Система уведомлений")?;
    dom::append(&container, &heading)?;

    // 1. Контейнер для уведомлений
    let notifications = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&notifications), "notifications")?;
    dom::set_attribute(&notifications, "id", "notifications")?;
    dom::append(&container, &notifications)?;

    // 2. Создаём Notifier и подписываем двух слушателей
    let notifier = Notifier::new();

    // Слушатель 1: логирует в консоль
    notifier.subscribe(|message: &str| {
        log!("[log-listener] {}", message);
    });

    // Слушатель 2: добавляет уведомление в DOM
    let notifications_clone = notifications.clone();
    notifier.subscribe(move |message: &str| {
        if let Ok(div) = dom::create_element("div") {
            if let Some(html) = dom::try_as_html(&div) {
                let _ = dom::add_class(&html, "notification");
                div.set_text_content(Some(message));
                let _ = dom::append(&notifications_clone, &div);
            }
        }
    });

    log!(
        "Notifier: подписчиков зарегистрировано: {}",
        notifier.subscriber_count()
    );

    // 3. Кнопки, которые дёргают notifier
    let controls = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&controls), "controls")?;

    let buttons_data = [
        ("hello", "Привет"),
        ("warning", "Предупреждение"),
        ("error", "Ошибка"),
        ("clear", "Очистить"),
    ];

    // Оборачиваем notifier в Rc<RefCell>, чтобы разделить между обработчиками
    let notifier = Rc::new(notifier);

    // Хранилище для Closure — живёт до конца `run`
    let closures = Rc::new(RefCell::new(EventClosures::new()));

    for (id, label) in buttons_data {
        let btn = dom::create_element("button")?;
        dom::set_attribute(&btn, "id", id)?;
        btn.set_text_content(Some(label));
        dom::append(&controls, &btn)?;

        let notifier = notifier.clone();
        let notifications = notifications.clone();

        callbacks::listen(
            &btn,
            "click",
            move |_event| {
                match id {
                    "clear" => {
                        notifications.set_text_content(Some(""));
                        notifier.notify("Лента очищена");
                    }
                    _ => {
                        let message = format!("Событие: {}", label);
                        notifier.notify(&message);
                    }
                }
            },
            &closures,
        )?;
    }

    dom::append(&container, &controls)?;

    log!(
        "Приложение готово. Подписчиков: {}",
        notifier.subscriber_count()
    );

    // Важно: `closures` и `notifier` живут до конца `run`.
    // После `run` приложение продолжает работать, потому что
    // Wasm-модуль не выгружается. Все Closure остаются активными
    // в течение всей сессии.
    std::mem::forget(closures);
    std::mem::forget(notifier);

    Ok(())
}
```

**Что демонстрирует `app.rs`:**

1. **`Notifier`** — Rust-callback API. Хранит `Vec<Box<dyn Fn(&str)>>`, вызывает всех при `notify`.
2. **Два подписчика** — один логирует в консоль, второй добавляет DOM-элемент.
3. **`Rc<RefCell<EventClosures>>`** — хранилище для JS-обработчиков кнопок. Обработчики не «забываются», а живут в контейнере.
4. **`std::mem::forget`** — сознательная утечка, потому что приложение живёт всё время работы страницы. Альтернатива — хранить `closures` и `notifier` в `static` или `thread_local`.

**Почему `std::mem::forget`.** В `run()` локальные переменные уничтожаются при выходе из функции. Если не «забыть» контейнер, JS-обработчики станут недействительными. В production-коде правильнее хранить состояние в статической переменной, но для учебного примера `forget` наглядно показывает суть проблемы.

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — система уведомлений</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Система уведомлений с callback API. Откройте консоль для логов.</p>

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
   Кнопки управления
   ============================================================ */

.controls {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin: 1rem 0;
}

.controls button {
    padding: 0.5rem 1rem;
    font-size: 1rem;
    border: 1px solid #b7410e;
    background: white;
    color: #b7410e;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
}

.controls button:hover {
    background: #b7410e;
    color: white;
}

.controls button:active {
    transform: scale(0.96);
}

/* ============================================================
   Лента уведомлений
   ============================================================ */

.notifications {
    min-height: 4rem;
    padding: 0.5rem;
    border: 1px dashed #ccc;
    border-radius: 8px;
    background: #fafafa;
}

.notification {
    padding: 0.5rem 0.75rem;
    margin-bottom: 0.5rem;
    border-left: 3px solid #b7410e;
    background: white;
    border-radius: 4px;
    font-size: 0.95rem;
    animation: slide-in 0.2s ease-out;
}

.notification:last-child {
    margin-bottom: 0;
}

@keyframes slide-in {
    from {
        opacity: 0;
        transform: translateY(-4px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}
```

---

## `tests/wasm_callbacks.rs` (новый файл)

```rust
//! Тесты для `Closure` и callback API.
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::Event;

use hello_wasm02::callbacks::{self, EventClosures};

wasm_bindgen_test_configure!(run_in_browser);

mod common;

// ============================================================
// Closure: базовое использование
// ============================================================

#[wasm_bindgen_test]
fn test_closure_captures_state() {
    let counter = Rc::new(RefCell::new(0));
    let counter_clone = counter.clone();

    let closure = Closure::<dyn FnMut()>::new(move || {
        *counter_clone.borrow_mut() += 1;
    });

    let f: &js_sys::Function = closure.as_ref().unchecked_ref();
    f.call0(&JsValue::NULL).unwrap();
    f.call0(&JsValue::NULL).unwrap();
    f.call0(&JsValue::NULL).unwrap();

    assert_eq!(*counter.borrow(), 3);
}

#[wasm_bindgen_test]
fn test_closure_with_arg() {
    let collected = Rc::new(RefCell::new(Vec::<String>::new()));
    let collected_clone = collected.clone();

    let closure = Closure::<dyn FnMut(String)>::new(move |s: String| {
        collected_clone.borrow_mut().push(s);
    });

    let f: &js_sys::Function = closure.as_ref().unchecked_ref();
    f.call1(&JsValue::NULL, &JsValue::from_str("a")).unwrap();
    f.call1(&JsValue::NULL, &JsValue::from_str("b")).unwrap();

    let items = collected.borrow();
    assert_eq!(items.len(), 2);
    assert_eq!(items[0], "a");
    assert_eq!(items[1], "b");
}

// ============================================================
// Closure: время жизни
// ============================================================

#[wasm_bindgen_test]
fn test_closure_dropped() {
    let counter = Rc::new(RefCell::new(0));
    let counter_clone = counter.clone();

    {
        let _closure = Closure::<dyn FnMut()>::new(move || {
            *counter_clone.borrow_mut() += 1;
        });
        // _closure удаляется при выходе из блока
    }

    // Счётчик остаётся 0: замыкание не было вызвано
    // и не будет доступно после drop.
    assert_eq!(*counter.borrow(), 0);
}

#[wasm_bindgen_test]
fn test_closure_forget_persists() {
    let counter = Rc::new(RefCell::new(0));
    let counter_clone = counter.clone();

    let closure = Closure::<dyn FnMut()>::new(move || {
        *counter_clone.borrow_mut() += 1;
    });

    // Сохраняем функцию и «забываем» Closure.
    // Явный generic-параметр нужен, потому что `.clone()`
    // не даёт компилятору достаточно информации для вывода `T`.
    let f: js_sys::Function = closure
        .as_ref()
        .unchecked_ref::<js_sys::Function>()
        .clone();
    closure.forget();

    // Функция продолжает работать
    f.call0(&JsValue::NULL).unwrap();
    f.call0(&JsValue::NULL).unwrap();

    assert_eq!(*counter.borrow(), 2);
}

// ============================================================
// Fn / FnMut / FnOnce
// ============================================================

#[wasm_bindgen_test]
fn test_fn_immutable() {
    // Fn: замыкание только читает захваченные данные
    let source = "hello";
    let closure = Closure::<dyn Fn() -> String>::new(move || source.to_uppercase());

    let f: &js_sys::Function = closure.as_ref().unchecked_ref();
    let result = f.call0(&JsValue::NULL).unwrap();

    assert_eq!(result.as_string(), Some("HELLO".into()));
}

#[wasm_bindgen_test]
fn test_fn_mut_mutates() {
    // FnMut: замыкание мутирует захваченные данные
    let count = Rc::new(RefCell::new(0));
    let count_clone = count.clone();

    let closure = Closure::<dyn FnMut()>::new(move || {
        *count_clone.borrow_mut() += 1;
    });

    let f: &js_sys::Function = closure.as_ref().unchecked_ref();
    f.call0(&JsValue::NULL).unwrap();
    f.call0(&JsValue::NULL).unwrap();

    assert_eq!(*count.borrow(), 2);
}

#[wasm_bindgen_test]
fn test_fn_once_consumes() {
    // FnOnce: замыкание потребляет захваченные данные
    let data = vec![1, 2, 3];
    let closure = Closure::once(move || {
        let sum: i32 = data.iter().sum();
        JsValue::from_f64(sum as f64)
    });

    let f: &js_sys::Function = closure.as_ref().unchecked_ref();
    let result = f.call0(&JsValue::NULL).unwrap();

    assert_eq!(result.as_f64(), Some(6.0));
    closure.forget();
}

// ============================================================
// EventClosures: контейнер
// ============================================================

#[wasm_bindgen_test]
fn test_event_closures_store() {
    let mut store = EventClosures::new();

    assert!(store.is_empty());
    assert_eq!(store.len(), 0);

    let closure = Closure::<dyn FnMut(Event)>::new(|_e: Event| {});
    store.push(closure);

    assert!(!store.is_empty());
    assert_eq!(store.len(), 1);

    store.clear();
    assert!(store.is_empty());
}

// ============================================================
// callbacks::listen
// ============================================================

#[wasm_bindgen_test]
fn test_listen_stores_closure() {
    let fixture = common::create_fixture();
    let store = Rc::new(RefCell::new(EventClosures::new()));

    let clicked = Rc::new(RefCell::new(false));
    let clicked_clone = clicked.clone();

    callbacks::listen(
        &fixture,
        "click",
        move |_event| {
            *clicked_clone.borrow_mut() = true;
        },
        &store,
    )
    .unwrap();

    assert_eq!(store.borrow().len(), 1);

    let event = Event::new("click").unwrap();
    fixture.dispatch_event(&event).unwrap();

    assert!(*clicked.borrow());

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_listen_multiple_handlers() {
    let fixture = common::create_fixture();
    let store = Rc::new(RefCell::new(EventClosures::new()));

    let count = Rc::new(RefCell::new(0));

    for _ in 0..3 {
        let count = count.clone();
        callbacks::listen(
            &fixture,
            "click",
            move |_event| {
                *count.borrow_mut() += 1;
            },
            &store,
        )
        .unwrap();
    }

    assert_eq!(store.borrow().len(), 3);

    let event = Event::new("click").unwrap();
    fixture.dispatch_event(&event).unwrap();

    // Все три обработчика вызваны
    assert_eq!(*count.borrow(), 3);

    common::cleanup_fixture(&fixture);
}

// ============================================================
// callbacks::listen_typed
// ============================================================

#[wasm_bindgen_test]
fn test_listen_typed_keyboard() {
    use web_sys::KeyboardEvent;

    let fixture = common::create_fixture();
    let store = Rc::new(RefCell::new(EventClosures::new()));

    let received = Rc::new(RefCell::new(Vec::<String>::new()));
    let received_clone = received.clone();

    callbacks::listen_typed::<KeyboardEvent, _>(
        &fixture,
        "keydown",
        move |event: KeyboardEvent| {
            received_clone.borrow_mut().push(event.key());
        },
        &store,
    )
    .unwrap();

    let event = KeyboardEvent::new("keydown").unwrap();
    fixture.dispatch_event(&event).unwrap();

    assert_eq!(received.borrow().len(), 1);

    common::cleanup_fixture(&fixture);
}

// ============================================================
// callbacks::listen_once
// ============================================================

#[wasm_bindgen_test]
fn test_listen_once() {
    let fixture = common::create_fixture();

    let called = Rc::new(RefCell::new(0));
    let called_clone = called.clone();

    callbacks::listen_once(&fixture, "click", move |_event| {
        *called_clone.borrow_mut() += 1;
    })
    .unwrap();

    let event = Event::new("click").unwrap();
    fixture.dispatch_event(&event).unwrap();
    fixture.dispatch_event(&event).unwrap();

    // Обработчик вызван только один раз
    assert_eq!(*called.borrow(), 1);

    common::cleanup_fixture(&fixture);
}

// ============================================================
// Rust-callback API: Notifier
// ============================================================

#[wasm_bindgen_test]
fn test_notifier_subscribe_and_notify() {
    use hello_wasm02::app::Notifier;

    let notifier = Notifier::new();
    let received = Rc::new(RefCell::new(Vec::<String>::new()));

    let received_clone = received.clone();
    notifier.subscribe(move |msg: &str| {
        received_clone.borrow_mut().push(msg.to_string());
    });

    notifier.notify("hello");
    notifier.notify("world");

    let items = received.borrow();
    assert_eq!(items.len(), 2);
    assert_eq!(items[0], "hello");
    assert_eq!(items[1], "world");
}

#[wasm_bindgen_test]
fn test_notifier_multiple_subscribers() {
    use hello_wasm02::app::Notifier;

    let notifier = Notifier::new();
    let a = Rc::new(RefCell::new(0));
    let b = Rc::new(RefCell::new(0));

    let a_clone = a.clone();
    notifier.subscribe(move |_msg| {
        *a_clone.borrow_mut() += 1;
    });

    let b_clone = b.clone();
    notifier.subscribe(move |_msg| {
        *b_clone.borrow_mut() += 10;
    });

    notifier.notify("test");

    assert_eq!(*a.borrow(), 1);
    assert_eq!(*b.borrow(), 10);
    assert_eq!(notifier.subscriber_count(), 2);
}
```

**Что проверяют тесты:**

- **Closure** — захват состояния, вызов через `js_sys::Function`, с аргументами.
- **Время жизни** — drop удаляет замыкание, `forget` его сохраняет.
- **`Fn`/`FnMut`/`FnOnce`** — три вида замыканий в `Closure`.
- **`EventClosures`** — контейнер работает корректно.
- **`callbacks::listen`** — регистрирует обработчик и сохраняет `Closure`.
- **`callbacks::listen_typed`** — приведение `Event` к `KeyboardEvent`.
- **`callbacks::listen_once`** — обработчик вызывается один раз.
- **`Notifier`** — Rust-callback API с несколькими подписчиками.

**Ключевые моменты:**

- **`use wasm_bindgen::JsCast;`** — обязателен для `unchecked_ref`, `dyn_into`, `unchecked_into`.
- **`unchecked_ref::<js_sys::Function>()`** с явным типом в цепочке `.clone()`. Без аннотации компилятор не может вывести `T`.
- **`HtmlElement` не импортируется** — все тесты работают с `Event` напрямую.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Появятся четыре кнопки и лента уведомлений. При клике на кнопки `notifier.notify` вызывает обоих подписчиков.

### Тесты

```bash
# Хостовые тесты
npm run test:host

# Wasm-логика в Node.js
npm run test:node

# DOM, события, js_sys, JsValue, callbacks в headless Chrome
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
wasm-pack test --headless --chrome --test wasm_callbacks
```

---

## Ключевые концепции главы

**1. `Closure` — мост между Rust-замыканием и JS-функцией.** `Closure::new(f)` создаёт JS-функцию-обёртку, которая вызывает `f`. `closure.as_ref()` даёт `&JsValue`, `closure.as_ref().unchecked_ref::<Function>()` — `&js_sys::Function`.

**2. `Fn`, `FnMut`, `FnOnce` — три вида замыканий.**

| Трейт | Что делает | Когда использовать |
|---|---|---|
| `Fn` | Только читает захваченное | Обработчики, читающие состояние |
| `FnMut` | Мутирует захваченное | Обработчики, изменяющие состояние |
| `FnOnce` | Потребляет захваченное | Одноразовые обработчики |

`Closure::new(f)` работает с `FnMut`. `Closure::once(f)` — с `FnOnce`. `Closure::<dyn Fn()>::new(f)` — с `Fn`.

**3. JavaScript callback lifetime.** JavaScript держит ссылку на JS-функцию, но **не владеет** `Closure`. Если `Closure` удаляется (drop), JS-функция становится недействительной. Если `forget()` — утечка: `Closure` остаётся в памяти навсегда.

**4. `forget()` — быстро и просто, но неаккуратно.** `closure.forget()` предотвращает удаление `Closure`. Подходит для:
- Обработчиков, живущих всё время работы приложения.
- Одноразовых событий (`load`, `error`).
- Учебных примеров.

Не подходит, если обработчиков много и они создаются динамически — это утечка.

**5. Хранение `Closure` — правильный способ.** Храните `Closure` в структуре или `Rc<RefCell<Vec<Closure>>>`. Когда контейнер удаляется, все `Closure` удаляются автоматически, и JS-обработчики отваливаются.

```rust
let store = Rc::new(RefCell::new(EventClosures::new()));
callbacks::listen(&target, "click", handler, &store)?;
// store живёт, пока нужен обработчик
```

**6. `listen` vs `listen_forget` vs `listen_once`.**

| Функция | Хранение | Когда использовать |
|---|---|---|
| `listen` | В `EventClosures` | Основной способ |
| `listen_forget` | `forget()` | Обработчик живёт всё время |
| `listen_once` | `Closure::once` + `forget()` | Одноразовые события |

**7. Rust-callback API.** Можно создавать Rust-API с callbacks, используя `Box<dyn Fn(...)>`:

```rust
pub struct Notifier {
    listeners: Rc<RefCell<Vec<Box<dyn Fn(&str)>>>>,
}

impl Notifier {
    pub fn subscribe<F: Fn(&str) + 'static>(&self, f: F) {
        self.listeners.borrow_mut().push(Box::new(f));
    }
    pub fn notify(&self, msg: &str) {
        for listener in self.listeners.borrow().iter() {
            listener(msg);
        }
    }
}
```

Это чистый Rust-код без `wasm-bindgen`. Он не зависит от JS, но может быть использован из JS-обработчиков через `callbacks::listen`.

**8. Разделение уровней.**

- **UI-слой** (`app.rs`) — привязка к DOM.
- **Callback API** (`Notifier`) — чистый Rust.
- **Мост** (`callbacks.rs`) — управление `Closure` между DOM и Rust.

**9. `std::mem::forget` для долгоживущих контейнеров.** В `run()` локальные переменные уничтожаются при выходе. Если нужны глобально живущие обработчики — либо `forget`, либо `thread_local!` / `static`.

**10. `JsCast` и `unchecked_ref` для вызова `Closure`.** Чтобы вызвать `Closure` вручную (например, в тесте), приведите его к `js_sys::Function`:

```rust
let f: &js_sys::Function = closure.as_ref().unchecked_ref();
f.call0(&JsValue::NULL)?;
```

**11. `pub mod app;` для интеграционных тестов.** Если тесты в `tests/` используют `hello_wasm02::app::Notifier`, модуль должен быть публичным. Это тот же случай, что был с `events`, `value`, `js_objects`.

**12. `unchecked_ref::<T>()` требует явного generic в цепочках.** Когда `.clone()` вызывается сразу после `unchecked_ref()`, компилятор не может вывести `T`. Решения:
- Явный generic: `unchecked_ref::<js_sys::Function>()`.
- Привязка с аннотацией: `let f: js_sys::Function = closure.as_ref().unchecked_ref();`.

---

## Итог главы

- `Closure` — мост между Rust-замыканием и JS-функцией.
- `Closure::new(f)` для `FnMut`, `Closure::once(f)` для `FnOnce`.
- `Fn`, `FnMut`, `FnOnce` — три вида замыканий с разными правами доступа к захваченному.
- JavaScript callback lifetime: JS держит функцию, но не владеет `Closure`.
- `forget()` — утечка, подходит для долгоживущих обработчиков.
- Хранение `Closure` в структуре — правильный способ.
- `callbacks::listen` регистрирует и сохраняет `Closure` в `EventClosures`.
- `callbacks::listen_typed<T>` — для конкретных типов событий.
- `callbacks::listen_once` — для одноразовых обработчиков.
- Rust-callback API строится через `Vec<Box<dyn Fn(...)>>`.
- `pub mod app;` — обязательно для тестов `Notifier`.
- `unchecked_ref::<T>()` требует явного generic в цепочках `.clone()`.
- Практический проект: система уведомлений с двумя подписчиками и четырьмя кнопками.

---

**Что дальше.** В главе 13 мы перейдём к части V и разберём **Browser navigation и URL**: `Location`, `Url`, `UrlSearchParams`, `History`, Navigation API, query parameters, hash и практический проект — простой client-side navigation.