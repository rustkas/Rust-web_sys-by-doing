# Глава 13. Async/await на границе Rust ↔ JavaScript

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
│   ├── lib.rs                      # pub mod app; pub mod async_utils; pub mod notifier; ...
│   ├── app.rs                      # асинхронный загрузчик данных
│   ├── async_utils.rs              # НОВОЕ: sleep, delay_ms, with_timeout, spawn
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── notifier.rs                 # НОВОЕ: Notifier вынесен из app.rs
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
│   ├── wasm_callbacks.rs           # обновлён: notifier::Notifier
│   ├── wasm_async.rs
│   └── wasm_async_utils.rs         # НОВОЕ: 10 тестов для async-помощников
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/async_utils.rs` — новый модуль. Содержит `delay_ms`, `sleep`, `await_promise`, `resolved`, `rejected`, `promise_race`, `with_timeout`, `spawn`, `spawn_logged`.
- `src/notifier.rs` — **вынесен из `app.rs`**. `Notifier` — стабильный переиспользуемый компонент, не должен зависеть от того, что сейчас в `app.rs`. Тесты обращаются к нему через `hello_wasm02::notifier::Notifier`.
- `src/app.rs` — переписан под практический проект: **асинхронный загрузчик** с индикатором загрузки, отменой и таймаутом.
- `src/lib.rs` — добавляются `pub mod async_utils;` и `pub mod notifier;`.
- `tests/wasm_async_utils.rs` — новый файл с 10 тестами для асинхронных помощников.
- `tests/wasm_callbacks.rs` — обновлён: путь `hello_wasm02::notifier::Notifier` вместо `hello_wasm02::app::Notifier`.
- `web/styles/main.css` — заменяется целиком: стили системы уведомлений удалены, добавлены стили загрузчика.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `wasm-bindgen-futures` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/ |
| `JsFuture` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/struct.JsFuture.html |
| `spawn_local` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/fn.spawn_local.html |
| `js_sys::Promise` | https://docs.rs/js-sys/latest/js_sys/struct.Promise.html |
| `Promise.race` | https://docs.rs/js-sys/latest/js_sys/struct.Promise.html#method.race |
| Async Rust book | https://rust-lang.github.io/async-book/ |
| wasm-bindgen: async examples | https://rustwasm.github.io/wasm-bindgen/examples/fetch.html |
| MDN: Promise | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise |
| MDN: Event loop | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop |

---

## `Cargo.toml`

Новых features `web-sys` не требуется. `wasm-bindgen-futures` уже в зависимостях с главы 6.

```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"
console_error_panic_hook = "0.1"

[dependencies.web-sys]
version = "0.3"
features = [
    # ... features из предыдущих глав ...
    "console",
]
```

**Почему не нужен `futures` crate.** Для этой главы достаточно `wasm-bindgen-futures`. `futures` понадобится позже (глава 16 — Fetch), когда будем комбинировать несколько `Future` в одном `async`-блоке.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Публичные модули:
//! - `app` — практические проекты глав.
//! - `async_utils` — помощники для асинхронной работы.
//! - `callbacks` — управление временем жизни `Closure`.
//! - `dom` — помощники для работы с DOM.
//! - `events` — помощники для работы с событиями.
//! - `value` — помощники для работы с `JsValue`.
//! - `js_objects` — помощники для `Array`, `Map`, `Set`, `Promise`, `Date`.
//! - `notifier` — система уведомлений с Rust-callback API.

pub mod app;
pub mod async_utils;
pub mod callbacks;
pub mod dom;
pub mod events;
pub mod js_objects;
pub mod notifier;
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

---

## `src/async_utils.rs` (новый модуль)

```rust
//! Помощники для асинхронной работы на Wasm.
//!
//! ## Как это работает
//!
//! JavaScript-мир асинхронный: `setTimeout`, `fetch`, `Promise` — всё
//! это работает через event loop браузера. Rust-мир тоже асинхронный:
//! `Future`, `async`, `await`. Задача этого модуля — построить мост
//! между ними.
//!
//! Мост — `wasm_bindgen_futures::JsFuture`. Он принимает `js_sys::Promise`
//! и превращает его в Rust `Future`. После этого `promise.await`
//! работает как обычно.
//!
//! `spawn_local` — способ запустить `Future` в фоне. Он регистрирует
//! задачу в event loop браузера; когда `Future` готов, она продолжает
//! выполнение с того же места.
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use js_sys::{Array, Promise};

// ============================================================
// Задержки
// ============================================================

/// Создаёт `Promise`, который разрешается через `ms` миллисекунд.
///
/// Внутри использует `setTimeout`. `Promise` не несёт значения —
/// разрешается с `undefined`.
pub fn delay_ms(ms: i32) -> Promise {
    Promise::new(&mut |resolve, _reject| {
        let window = web_sys::window().expect("no window");
        window
            .set_timeout_with_callback_and_timeout_and_arguments_0(&resolve, ms)
            .expect("setTimeout failed");
    })
}

/// Ждёт `ms` миллисекунд.
///
/// Возвращает `Result<(), JsValue>` для единообразия с другими
/// асинхронными функциями. `delay_ms` не может провалиться, но
/// сигнатура `async fn` с `Result` удобнее для `?`.
pub async fn sleep(ms: i32) -> Result<(), JsValue> {
    JsFuture::from(delay_ms(ms)).await?;
    Ok(())
}

// ============================================================
// Promise ↔ Future
// ============================================================

/// Ждёт разрешения `Promise` и возвращает результат.
///
/// Если `Promise` разрешается значением `v`, возвращает `Ok(v)`.
/// Если `Promise` отклоняется причиной `e`, возвращает `Err(e)`.
pub async fn await_promise(promise: Promise) -> Result<JsValue, JsValue> {
    JsFuture::from(promise).await
}

/// Обёртка над `Promise.resolve(value)`.
pub fn resolved(value: &JsValue) -> Promise {
    Promise::resolve(value)
}

/// Обёртка над `Promise.reject(reason)`.
pub fn rejected(reason: &JsValue) -> Promise {
    Promise::reject(reason)
}

// ============================================================
// Promise.race и таймауты
// ============================================================

/// Обёртка над `Promise.race(promises)`.
///
/// Возвращает `Promise`, который разрешается или отклоняется
/// значением первого завершившегося `Promise` из массива.
///
/// Использует готовую обёртку `js_sys::Promise::race`, которая
/// корректно вызывает JavaScript-метод с нужным `this`.
pub fn promise_race(promises: &Array) -> Promise {
    Promise::race(promises)
}

/// Ждёт `Promise` с таймаутом.
///
/// Если `Promise` разрешится за `timeout_ms`, возвращает его значение.
/// Если таймаут сработает раньше, возвращает `Err`.
///
/// **Ограничение:** если `Promise` отклоняется, `Promise.race` вернёт
/// ошибку `Promise`, но определить, что именно произошло — таймаут
/// или отклонение, — не получится. Для точного различения нужен
/// `AbortController` (глава 16).
pub async fn with_timeout(
    promise: Promise,
    timeout_ms: i32,
) -> Result<JsValue, JsValue> {
    let timeout_promise = Promise::new(&mut |_resolve, reject| {
        let window = web_sys::window().expect("no window");
        window
            .set_timeout_with_callback_and_timeout_and_arguments_0(
                &reject,
                timeout_ms,
            )
            .expect("setTimeout failed");
    });

    let promises = Array::new();
    promises.push(&promise);
    promises.push(&timeout_promise);

    let race = promise_race(&promises);
    JsFuture::from(race).await
}

// ============================================================
// spawn_local
// ============================================================

/// Запускает `Future` в фоне.
///
/// `Future` выполняется через event loop браузера. Функция возвращает
/// управление сразу, не дожидаясь завершения `Future`.
pub fn spawn<F>(future: F)
where
    F: std::future::Future<Output = ()> + 'static,
{
    spawn_local(future);
}

/// Запускает `Future`, которая возвращает `Result`.
///
/// Если `Future` завершается с `Err`, ошибка логируется в консоль.
pub fn spawn_logged<F, E>(future: F)
where
    F: std::future::Future<Output = Result<(), E>> + 'static,
    E: std::fmt::Debug,
{
    spawn_local(async move {
        if let Err(e) = future.await {
            web_sys::console::error_1(
                &format!("spawn_logged error: {:?}", e).into(),
            );
        }
    });
}
```

**Ключевые моменты:**

- **`delay_ms`** создаёт `Promise`, который разрешается через `setTimeout`.
- **`sleep`** — async-обёртка над `delay_ms`.
- **`await_promise`** — универсальная функция для ожидания любого `Promise`.
- **`promise_race`** использует **готовую обёртку `js_sys::Promise::race`**. Ручное извлечение через `Reflect::get` и вызов `Function::call1(&JsValue::NULL, ...)` не работает для статических методов: они требуют правильный `this`.
- **`with_timeout`** комбинирует исходный `Promise` и таймаут-`Promise` через `Promise.race`.
- **`spawn`** запускает `Future` в фоне через `spawn_local`.
- **`spawn_logged`** — вариант `spawn`, который логирует ошибки.

---

## `src/notifier.rs` (вынесен из `app.rs`)

```rust
//! Система уведомлений с Rust-callback API.
//!
//! Демонстрирует паттерн «observer»: подписчики регистрируются
//! через `subscribe`, уведомляются через `notify`. Подписчики —
//! `Fn(&str)`, хранятся в `Vec<Box<dyn Fn(&str)>>`.
//!
//! Этот модуль не зависит от DOM или `web_sys`. Он может
//! использоваться как в браузере, так и в нативных приложениях.
//! Именно поэтому он вынесен из `app.rs` — приложение меняется
//! от главы к главе, а `Notifier` — стабильный компонент.

use std::cell::RefCell;
use std::rc::Rc;

/// Простая система уведомлений с callback API.
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
```

**Почему вынесен из `app.rs`.**

- `app.rs` меняется от главы к главе (форма → счётчик → инспектор → коллекции → система уведомлений → асинхронный загрузчик). Если компоненты будут жить в `app.rs`, при каждой смене демонстрации придётся их переносить.
- `Notifier` — стабильный переиспользуемый компонент. Он не должен зависеть от текущего практического проекта.
- Тесты `wasm_callbacks.rs` используют `hello_wasm02::notifier::Notifier` — стабильный путь, не зависящий от демонстрации в `app.rs`.

---

## `src/app.rs` (асинхронный загрузчик данных)

```rust
//! Логика приложения: «Асинхронный загрузчик данных».
//!
//! Демонстрирует `JsFuture`, `async`/`await`, `spawn_local` и
//! работу с `Promise` на границе Rust ↔ JavaScript.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use crate::async_utils;
use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;

/// Состояние загрузки.
#[derive(Default)]
struct LoaderState {
    /// Отменена ли текущая операция.
    cancelled: bool,
    /// Идёт ли загрузка прямо сейчас.
    loading: bool,
}

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Асинхронный загрузчик")?;
    dom::append(&container, &heading)?;

    // 1. Кнопки управления
    let controls = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&controls), "controls")?;

    let load_btn = dom::create_element("button")?;
    dom::set_attribute(&load_btn, "id", "load")?;
    load_btn.set_text_content(Some("Загрузить"));
    dom::append(&controls, &load_btn)?;

    let load_timeout_btn = dom::create_element("button")?;
    dom::set_attribute(&load_timeout_btn, "id", "load-timeout")?;
    load_timeout_btn.set_text_content(Some("Загрузить с таймаутом"));
    dom::append(&controls, &load_timeout_btn)?;

    let cancel_btn = dom::create_element("button")?;
    dom::set_attribute(&cancel_btn, "id", "cancel")?;
    cancel_btn.set_text_content(Some("Отменить"));
    dom::append(&controls, &cancel_btn)?;

    dom::append(&container, &controls)?;

    // 2. Индикатор статуса
    let status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&status), "status")?;
    status.set_text_content(Some("Готов"));
    dom::append(&container, &status)?;

    // 3. Область результата
    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    dom::append(&container, &result)?;

    // 4. Состояние и хранилище Closure
    let state = Rc::new(RefCell::new(LoaderState::default()));
    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // 5. Обработчик «Загрузить»
    {
        let state = state.clone();
        let status = status.clone();
        let result = result.clone();
        callbacks::listen(
            &load_btn,
            "click",
            move |_event| {
                if state.borrow().loading {
                    log!("Уже загружаем, игнорируем клик");
                    return;
                }

                *state.borrow_mut() = LoaderState {
                    cancelled: false,
                    loading: true,
                };

                status.set_text_content(Some("Загрузка..."));
                result.set_text_content(Some(""));

                let state = state.clone();
                let status = status.clone();
                let result = result.clone();

                async_utils::spawn(async move {
                    // Эмулируем сетевой запрос: 2 секунды
                    if let Err(e) = async_utils::sleep(2000).await {
                        log!("Ошибка sleep: {:?}", e);
                    }

                    // Проверяем флаг отмены
                    if state.borrow().cancelled {
                        log!("Операция отменена, игнорируем результат");
                        return;
                    }

                    state.borrow_mut().loading = false;
                    status.set_text_content(Some("Готов"));
                    result.set_text_content(Some(
                        "Данные загружены: {\"id\": 1, \"name\": \"Rust\"}",
                    ));
                    log!("Загрузка завершена");
                });

                log!("Загрузка запущена");
            },
            &closures,
        )?;
    }

    // 6. Обработчик «Загрузить с таймаутом»
    {
        let state = state.clone();
        let status = status.clone();
        let result = result.clone();
        callbacks::listen(
            &load_timeout_btn,
            "click",
            move |_event| {
                if state.borrow().loading {
                    log!("Уже загружаем, игнорируем клик");
                    return;
                }

                *state.borrow_mut() = LoaderState {
                    cancelled: false,
                    loading: true,
                };

                status.set_text_content(Some("Загрузка с таймаутом 1 с..."));
                result.set_text_content(Some(""));

                let state = state.clone();
                let status = status.clone();
                let result = result.clone();

                async_utils::spawn(async move {
                    let slow_promise = async_utils::delay_ms(2000);

                    match async_utils::with_timeout(slow_promise, 1000).await {
                        Ok(_) => {
                            state.borrow_mut().loading = false;
                            status.set_text_content(Some("Готов"));
                            result.set_text_content(Some(
                                "Данные загружены (успели до таймаута)",
                            ));
                        }
                        Err(_) => {
                            state.borrow_mut().loading = false;
                            status.set_text_content(Some("Таймаут"));
                            result.set_text_content(Some(
                                "Превышено время ожидания (1 с)",
                            ));
                            log!("Таймаут сработал");
                        }
                    }
                });

                log!("Загрузка с таймаутом запущена");
            },
            &closures,
        )?;
    }

    // 7. Обработчик «Отменить»
    {
        let state = state.clone();
        let status = status.clone();
        let result = result.clone();
        callbacks::listen(
            &cancel_btn,
            "click",
            move |_event| {
                if !state.borrow().loading {
                    log!("Нечего отменять");
                    return;
                }

                state.borrow_mut().cancelled = true;
                state.borrow_mut().loading = false;
                status.set_text_content(Some("Отменено"));
                result.set_text_content(Some("Операция отменена пользователем"));
                log!("Операция отменена");
            },
            &closures,
        )?;
    }

    log!("Асинхронный загрузчик готов");

    // Контейнеры живут до конца сессии
    std::mem::forget(closures);
    std::mem::forget(state);

    Ok(())
}
```

**Что демонстрирует `app.rs`:**

1. **`async_utils::spawn`** — запуск асинхронной задачи в фоне. UI не блокируется.
2. **`async_utils::sleep`** — ожидание с `await`.
3. **`with_timeout`** — комбинация `Promise.race`.
4. **Флаг отмены** — простая реализация отмены через `Rc<RefCell<LoaderState>>`.

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — асинхронный загрузчик</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Асинхронный загрузчик. Откройте консоль для логов.</p>

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

#app {
    margin-top: 1.5rem;
}

/* ============================================================
   Кнопки
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
   Статус и результат
   ============================================================ */

.status {
    margin: 1rem 0;
    padding: 0.5rem 0.75rem;
    font-weight: 500;
    color: #666;
    border-left: 3px solid #b7410e;
    background: #fafafa;
    border-radius: 4px;
}

.result {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.9rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 1rem;
    border-radius: 4px;
    min-height: 3rem;
    border: 1px dashed #ccc;
}
```

---

## `tests/wasm_async_utils.rs` (новый файл)

```rust
//! Тесты для асинхронных помощников: sleep, timeout, spawn.
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use wasm_bindgen_test::*;

use hello_wasm02::async_utils;

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// delay_ms и sleep
// ============================================================

#[wasm_bindgen_test]
async fn test_delay_ms_resolves() {
    let promise = async_utils::delay_ms(10);
    let result = JsFuture::from(promise).await;

    assert!(result.is_ok());
}

#[wasm_bindgen_test]
async fn test_sleep_completes() {
    let result = async_utils::sleep(10).await;
    assert!(result.is_ok());
}

// ============================================================
// await_promise
// ============================================================

#[wasm_bindgen_test]
async fn test_await_promise_resolve() {
    let promise = async_utils::resolved(&JsValue::from_f64(42.0));
    let result = async_utils::await_promise(promise).await.unwrap();

    assert_eq!(result.as_f64(), Some(42.0));
}

#[wasm_bindgen_test]
async fn test_await_promise_reject() {
    let promise = async_utils::rejected(&JsValue::from_str("error"));
    let result = async_utils::await_promise(promise).await;

    assert!(result.is_err());
    assert_eq!(result.unwrap_err().as_string(), Some("error".into()));
}

// ============================================================
// promise_race
// ============================================================

#[wasm_bindgen_test]
async fn test_promise_race_first_wins() {
    let fast = async_utils::resolved(&JsValue::from_str("fast"));
    let slow: js_sys::Promise = async_utils::delay_ms(1000);

    let promises = js_sys::Array::new();
    promises.push(&fast);
    promises.push(&slow);

    let race = async_utils::promise_race(&promises);
    let result = JsFuture::from(race).await.unwrap();

    assert_eq!(result.as_string(), Some("fast".into()));
}

// ============================================================
// with_timeout
// ============================================================

#[wasm_bindgen_test]
async fn test_with_timeout_completes() {
    // Promise разрешается быстро (10 мс), таймаут 500 мс
    let promise = async_utils::delay_ms(10);
    let result = async_utils::with_timeout(promise, 500).await;

    assert!(result.is_ok());
}

#[wasm_bindgen_test]
async fn test_with_timeout_fires() {
    // Promise разрешается через 500 мс, таймаут 10 мс
    let promise = async_utils::delay_ms(500);
    let result = async_utils::with_timeout(promise, 10).await;

    assert!(result.is_err());
}

// ============================================================
// spawn и spawn_logged
// ============================================================

#[wasm_bindgen_test]
async fn test_spawn_runs() {
    let flag = Rc::new(RefCell::new(false));
    let flag_clone = flag.clone();

    async_utils::spawn(async move {
        async_utils::sleep(10).await.ok();
        *flag_clone.borrow_mut() = true;
    });

    // Ждём, пока флаг установится
    async_utils::sleep(50).await.ok();

    assert!(*flag.borrow(), "spawned task did not run");
}

#[wasm_bindgen_test]
async fn test_spawn_logged_catches_error() {
    let called = Rc::new(RefCell::new(false));
    let called_clone = called.clone();

    async_utils::spawn_logged(async move {
        async_utils::sleep(10).await.ok();
        *called_clone.borrow_mut() = true;
        Err::<(), String>("ignored".into())
    });

    async_utils::sleep(50).await.ok();

    assert!(*called.borrow(), "spawned task did not run");
}

// ============================================================
// Последовательность
// ============================================================

#[wasm_bindgen_test]
async fn test_sequential_sleeps() {
    let start = js_sys::Date::now();

    async_utils::sleep(20).await.ok();
    async_utils::sleep(20).await.ok();

    let elapsed = js_sys::Date::now() - start;

    // Ожидаем минимум 35 мс, но не больше 200 мс
    // (с запасом на медленные CI)
    assert!(elapsed >= 35.0, "elapsed too small: {}", elapsed);
    assert!(elapsed < 200.0, "elapsed too large: {}", elapsed);
}
```

**Обратите внимание:** импорт `std::sync::atomic::{AtomicBool, Ordering}` **отсутствует** — он был лишним. Состояние хранится в `Rc<RefCell<bool>>` (однопоточный Wasm).

---

## `tests/wasm_callbacks.rs` — фрагмент с обновлёнными импортами

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

// ... остальные тесты без изменений ...

// ============================================================
// Rust-callback API: Notifier
// ============================================================

#[wasm_bindgen_test]
fn test_notifier_subscribe_and_notify() {
    use hello_wasm02::notifier::Notifier;   // ← путь обновлён

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
    use hello_wasm02::notifier::Notifier;   // ← путь обновлён

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

**Ключевое:** `Notifier` теперь импортируется из `hello_wasm02::notifier`, а не из `hello_wasm02::app`.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Три кнопки:

- **«Загрузить»** — через 2 секунды показывает результат.
- **«Загрузить с таймаутом»** — через 1 секунду показывает таймаут (запрос идёт 2 секунды).
- **«Отменить»** — прерывает текущую загрузку, показывает «Отменено».

### Тесты

```bash
npm run test:host      # хостовые тесты
npm run test:node      # Wasm-логика в Node.js
npm run test:chrome    # DOM, события, js_sys, JsValue, callbacks, async
npm run test:firefox   # то же в headless Firefox
npm test               # host + node + chrome
npm run test:all       # полный прогон
```

### Проверка конкретного теста

```bash
wasm-pack test --headless --chrome --test wasm_async_utils
```

---

## Ключевые концепции главы

**1. `wasm-bindgen-futures` — мост между `Promise` и `Future`.** Crate предоставляет `JsFuture` и `spawn_local`.

**2. `JsFuture::from(promise).await` — базовый приём.** Превращает `js_sys::Promise` в Rust `Future`. Resolve → `Ok(value)`, reject → `Err(reason)`.

**3. `async fn` в Rust на Wasm.** Работает так же, как на нативной платформе. Отличие — реализация `Future` использует event loop браузера, а не пул потоков.

**4. `spawn_local` — запуск `Future` в фоне.** Не блокирует вызывающий код. Возвращает управление сразу.

**5. Как Rust `Future` взаимодействует с event loop.**

1. `spawn_local(future)` регистрирует задачу.
2. Event loop вызывает `future.poll(cx)`.
3. Если `Future` готова — задача завершается.
4. Если нет — `Future` сохраняет `Waker` и возвращает `Pending`.
5. Когда JS-событие (например, `setTimeout`) наступает, `Waker.wake()` ставит задачу обратно в event loop.
6. Event loop снова вызывает `poll(cx)`.

**6. `Promise` ↔ `Future` — двусторонний мост.**

| Направление | Инструмент |
|---|---|
| `Promise` → `Future` | `JsFuture::from(promise)` |
| `Future` → `Promise` | `wasm_bindgen_futures::future_to_promise` |

**7. Отмена.** В Rust нет встроенного механизма отмены `Future`. Способы:

- **Флаг отмены** (`Rc<RefCell<bool>>`) — простой, но проверяется только в точках `await`.
- **`AbortController`** — для `fetch`, даёт нативную отмену.
- **`clearTimeout`** — для `setTimeout`, если сохранить `id`.

**8. Async в event handler.** Обработчик события сам не может быть `async`. Решение — внутри обработчика вызвать `spawn_local(async { ... })`.

```rust
callbacks::listen(&target, "click", move |_event| {
    spawn_local(async move {
        // асинхронная логика
    });
}, &store)?;
```

**9. Async и `Closure`.** `Closure` сам по себе не может быть `async`. Но замыкание внутри `Closure` может запускать `spawn_local` с `async`-блоком.

**10. Ошибки и rejections.** `JsFuture` конвертирует `reject` в `Err(JsValue)`. Всегда обрабатывайте ошибки:

```rust
match JsFuture::from(promise).await {
    Ok(value) => { /* успех */ }
    Err(reason) => { /* ошибка, reason: JsValue */ }
}
```

**11. Таймауты через `Promise.race`.** Стандартный способ комбинировать `Promise` и таймаут.

**12. Статические методы JavaScript вызывайте через готовые обёртки `js_sys`.** Не извлекайте их через `Reflect::get` и не вызывайте с `JsValue::NULL`. Правильные обёртки:

| JavaScript | `js_sys` |
|---|---|
| `Promise.race(promises)` | `Promise::race(&array)` |
| `Promise.all(promises)` | `Promise::all(&array)` |
| `Promise.resolve(v)` | `Promise::resolve(&v)` |
| `Promise.reject(r)` | `Promise::reject(&r)` |

**13. Разделение уровней.**

- **UI-слой** (`app.rs`) — привязка к DOM.
- **Async-помощники** (`async_utils.rs`) — `sleep`, `with_timeout`, `spawn`.
- **Callback-мост** (`callbacks.rs`) — управление `Closure`.
- **Future-мост** (`wasm-bindgen-futures`) — управление `Future`.
- **Переиспользуемые компоненты** (`notifier.rs`) — независимы от текущей демонстрации.

**14. Вынос переиспользуемых компонентов в отдельные модули.** `Notifier` вынесен из `app.rs` в `notifier.rs`. `app.rs` меняется от главы к главе; `notifier.rs`, `dom.rs`, `events.rs`, `value.rs`, `js_objects.rs`, `async_utils.rs`, `callbacks.rs` — накапливаются и остаются стабильными.

---

## Итог главы

- `wasm-bindgen-futures` — crate, соединяющий `Promise` и `Future`.
- `JsFuture::from(promise).await` — базовый приём ожидания.
- `spawn_local` — запуск `Future` в event loop браузера.
- Event loop браузера вызывает `poll` через `Waker`.
- `Promise` → `Future` (через `JsFuture`) и `Future` → `Promise` (через `future_to_promise`).
- Отмена — через флаг, `AbortController` или `clearTimeout`.
- Async в event handler — через `spawn_local(async { ... })` внутри обработчика.
- `Closure` не может быть `async`, но может запускать `spawn_local`.
- Ошибки и rejections — `JsFuture` возвращает `Result<JsValue, JsValue>`.
- Таймауты — через `Promise.race`.
- Статические методы JavaScript — через готовые обёртки `js_sys::Promise::race` и т.д.
- Переиспользуемые компоненты выносятся в отдельные модули: `Notifier` теперь в `notifier.rs`.
- Практический проект: асинхронный загрузчик с индикатором, отменой и таймаутом.

---

**Что дальше.** В главе 14 мы перейдём к части V и разберём **Browser navigation и URL**: `Location`, `Url`, `UrlSearchParams`, `History`, Navigation API, query parameters, hash и практический проект — простой client-side navigation.