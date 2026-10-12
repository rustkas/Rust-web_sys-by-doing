# Глава 16. Fetch

## Обновлённая структура проекта

```text
hello-wasm02/
├── Cargo.toml                      # +features для Fetch API
├── Cargo.lock
├── package.json
├── package-lock.json
├── vite.config.js
├── webdriver.json
├── .gitignore
├── LICENSE
├── README.md
├── build.rs
├── .cargo/
│   └── config.toml
├── src/
│   ├── lib.rs                      # +pub mod fetch;
│   ├── app.rs                      # переписан: Rust API client
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── fetch.rs                    # НОВОЕ: обёртки над Fetch API
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs
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
│   ├── wasm_callbacks.rs
│   ├── wasm_async.rs
│   ├── wasm_async_utils.rs
│   ├── wasm_navigation.rs
│   ├── wasm_storage.rs
│   └── wasm_fetch.rs               # НОВОЕ: тесты Fetch API
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +стили для API client
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/fetch.rs` — новый модуль. Обёртки над `Request`, `RequestInit`, `Response`, `Headers`, `fetch`. Функции для GET, POST, JSON, binary data, обработки HTTP-ошибок, Abort.
- `src/app.rs` — переписан под практический проект: **Rust API client**.
- `src/lib.rs` — **+`pub mod fetch;`**. Без этой строки `use crate::fetch;` в `app.rs` не скомпилируется.
- `tests/wasm_fetch.rs` — новый файл с 12 тестами Fetch API.
- `Cargo.toml` — добавлены features: `Headers`, `Request`, `RequestInit`, `RequestMode`, `Response`, `AbortController`, `AbortSignal`.

---

## Важное замечание о features

**`ArrayBuffer` и `Uint8Array` — это типы ECMAScript, а не Web API.** Они живут в crate `js-sys`, не в `web-sys`. Если добавить их в features `web-sys`, сборка упадёт с ошибкой:

```
package `hello-wasm02` depends on `web-sys` with feature `ArrayBuffer`
but `web-sys` does not have that feature.
```

**Правило разделения:**

| Crate | Что покрывает | Примеры |
|---|---|---|
| `wasm-bindgen` | Ядро | `JsValue`, `Closure`, `JsCast` |
| `js-sys` | ECMAScript | `Array`, `Object`, `Promise`, `Date`, `ArrayBuffer`, `Uint8Array` |
| `web-sys` | Web Platform | `Window`, `Document`, `Request`, `Response`, `AbortController` |

`ArrayBuffer` и `Uint8Array` используются через `js_sys::ArrayBuffer` и `js_sys::Uint8Array`. `js-sys` уже добавлен в `[dependencies]` — ничего дополнительно делать не нужно.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Fetch API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API |
| `Request` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Request |
| `RequestInit` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/RequestInit |
| `Response` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Response |
| `Headers` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Headers |
| `AbortController` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/AbortController |
| `js_sys::ArrayBuffer` | https://docs.rs/js-sys/latest/js_sys/struct.ArrayBuffer.html |
| `js_sys::Uint8Array` | https://docs.rs/js-sys/latest/js_sys/struct.Uint8Array.html |
| wasm-bindgen: fetch example | https://rustwasm.github.io/docs/wasm-bindgen/examples/fetch.html |

---

## Что такое Fetch API

**Fetch API — это современный способ делать HTTP-запросы из браузера.** Он заменил устаревший `XMLHttpRequest` и предоставляет чистый интерфейс на основе `Promise`.

**Как это работает в Rust/Wasm:**

1. Создаём `Request` через `Request::new_with_str_and_init()`.
2. Вызываем `window.fetch_with_request(&request)` — получаем `Promise`.
3. `JsFuture::from(promise).await` — ждём разрешения.
4. `Response` приходит как `JsValue`, приводится через `dyn_into::<Response>()`.
5. Читаем тело через `resp.json()?` или `resp.array_buffer()?` — снова `Promise`, снова `JsFuture`.

**Ключевая идея:** Fetch API полностью асинхронный. Каждый шаг — это `Promise`, который превращается в Rust `Future` через `JsFuture`.

**Что мы построим.** Модуль `fetch.rs`, который скрывает всю эту цепочку за простыми функциями:

```rust
let response: User = fetch::get_json("https://api.example.com/user/1").await?;
```

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
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[dependencies.web-sys]
version = "0.3"
features = [
    # ... features из предыдущих глав ...

    # Fetch API
    "Headers",
    "Request",
    "RequestInit",
    "RequestMode",
    "Response",
    "AbortController",
    "AbortSignal",

    # ВАЖНО: ArrayBuffer и Uint8Array — НЕ features web-sys.
    # Они живут в js-sys и доступны как js_sys::ArrayBuffer, js_sys::Uint8Array.

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

**Новые features `web-sys`:**

- **`Headers`** — заголовки HTTP-запроса и ответа.
- **`Request`** — объект запроса.
- **`RequestInit`** — опции запроса (метод, заголовки, тело, режим).
- **`RequestMode`** — CORS-режим (`Cors`, `NoCors`, `SameOrigin`).
- **`Response`** — объект ответа.
- **`AbortController` / `AbortSignal`** — отмена запросов.

**Почему нет `ArrayBuffer` и `Uint8Array` в этом списке.** Это ECMAScript-типы, они из `js-sys`. Если добавить их сюда, `cargo` откажется собирать проект с сообщением «web-sys does not have that feature».

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
//! - `fetch` — работа с Fetch API.
//! - `value` — помощники для работы с `JsValue`.
//! - `js_objects` — помощники для `Array`, `Map`, `Set`, `Promise`, `Date`.
//! - `navigation` — работа с URL, Location, History, Navigation API.
//! - `storage` — работа с localStorage и sessionStorage.
//! - `notifier` — система уведомлений с Rust-callback API.

pub mod app;
pub mod async_utils;
pub mod callbacks;
pub mod dom;
pub mod events;
pub mod fetch;
pub mod js_objects;
pub mod navigation;
pub mod notifier;
pub mod storage;
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

**Ключевое:** `pub mod fetch;`. Публичный, потому что используется в интеграционных тестах `tests/wasm_fetch.rs`.

---

## `src/fetch.rs`

```rust
//! Помощники для работы с Fetch API.
//!
//! ## Что в этом модуле
//!
//! - `Request`, `RequestInit` — создание запросов.
//! - `Response` — чтение ответов.
//! - `Headers` — работа с заголовками.
//! - GET, POST, JSON, binary data.
//! - Обработка HTTP-ошибок.
//! - Abort через `AbortController`.
//!
//! ## Разделение crate'ов
//!
//! - `web_sys` — Web API: `Request`, `Response`, `Headers`, `AbortController`.
//! - `js_sys` — ECMAScript: `ArrayBuffer`, `Uint8Array`.
//!
//! `ArrayBuffer` и `Uint8Array` живут в `js_sys`, а не в `web_sys`.
//! Это важно при работе с binary data.
#![allow(dead_code)]

use serde::de::DeserializeOwned;
use serde::Serialize;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use js_sys::{ArrayBuffer, Uint8Array};
use web_sys::{
    AbortController, Headers, Request, RequestInit, RequestMode, Response,
};

// ============================================================
// Создание запросов
// ============================================================

/// Создаёт GET-запрос.
pub fn make_get_request(url: &str, mode: RequestMode) -> Result<Request, JsValue> {
    let opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(mode);

    return Request::new_with_str_and_init(url, &opts);
}

/// Создаёт POST-запрос с JSON-телом.
pub fn make_post_json_request<T: Serialize>(
    url: &str,
    body: &T,
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let json = serde_json::to_string(body)
        .map_err(|e| JsValue::from_str(&format!("serialize error: {}", e)))?;

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_mode(mode);
    opts.set_body(&JsValue::from_str(&json));

    let request = Request::new_with_str_and_init(url, &opts)?;
    request
        .headers()
        .set("Content-Type", "application/json")?;

    return Ok(request);
}

/// Создаёт GET-запрос с заголовками.
pub fn make_request_with_headers(
    url: &str,
    headers: &[(&str, &str)],
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(mode);

    let request = Request::new_with_str_and_init(url, &opts)?;

    for (name, value) in headers {
        request.headers().set(name, value)?;
    }

    return Ok(request);
}

// ============================================================
// Отправка запросов
// ============================================================

/// Отправляет запрос и возвращает `Response`.
///
/// Проверяет `response.ok()`: если ответ не 2xx — `Err`.
pub async fn send_request(request: &Request) -> Result<Response, JsValue> {
    let window = web_sys::window().expect("no window");
    let resp_value = JsFuture::from(window.fetch_with_request(request)).await?;

    if !resp_value.is_instance_of::<Response>() {
        return Err(JsValue::from_str("response is not a Response object"));
    }

    let resp: Response = resp_value.dyn_into().map_err(|_| {
        JsValue::from_str("failed to cast JsValue to Response")
    })?;

    if !resp.ok() {
        let status = resp.status();
        let status_text = resp.status_text();
        return Err(JsValue::from_str(&format!(
            "HTTP {} {}",
            status, status_text
        )));
    }

    return Ok(resp);
}

/// Отправляет запрос и возвращает `Response` **без** проверки `ok()`.
pub async fn send_request_raw(request: &Request) -> Result<Response, JsValue> {
    let window = web_sys::window().expect("no window");
    let resp_value = JsFuture::from(window.fetch_with_request(request)).await?;
    let resp: Response = resp_value.dyn_into()?;
    return Ok(resp);
}

// ============================================================
// GET-запросы
// ============================================================

/// GET-запрос, возвращает `Response`.
pub async fn get(url: &str) -> Result<Response, JsValue> {
    let request = make_get_request(url, RequestMode::Cors)?;
    return send_request(&request).await;
}

/// GET-запрос, возвращает текст.
pub async fn get_text(url: &str) -> Result<String, JsValue> {
    let resp = get(url).await?;
    let text_promise = resp.text()?;
    let text_value = JsFuture::from(text_promise).await?;
    return Ok(text_value.as_string().unwrap_or_default());
}

/// GET-запрос, возвращает десериализованный JSON.
pub async fn get_json<T: DeserializeOwned>(url: &str) -> Result<T, JsValue> {
    let resp = get(url).await?;
    let json_promise = resp.json()?;
    let json_value = JsFuture::from(json_promise).await?;
    return deserialize_json(&json_value);
}

// ============================================================
// POST-запросы
// ============================================================

/// POST-запрос с JSON-телом, возвращает `Response`.
pub async fn post_json<T: Serialize>(
    url: &str,
    body: &T,
) -> Result<Response, JsValue> {
    let request = make_post_json_request(url, body, RequestMode::Cors)?;
    return send_request(&request).await;
}

/// POST-запрос с JSON-телом, возвращает десериализованный JSON.
pub async fn post_json_response<T: Serialize, R: DeserializeOwned>(
    url: &str,
    body: &T,
) -> Result<R, JsValue> {
    let resp = post_json(url, body).await?;
    let json_promise = resp.json()?;
    let json_value = JsFuture::from(json_promise).await?;
    return deserialize_json(&json_value);
}

// ============================================================
// Binary data
// ============================================================

/// GET-запрос, возвращает `ArrayBuffer` (из `js_sys`).
pub async fn get_array_buffer(url: &str) -> Result<ArrayBuffer, JsValue> {
    let resp = get(url).await?;
    let buffer_promise = resp.array_buffer()?;
    let buffer_value = JsFuture::from(buffer_promise).await?;
    let buffer: ArrayBuffer = buffer_value.dyn_into()?;
    return Ok(buffer);
}

/// GET-запрос, возвращает `Vec<u8>`.
pub async fn get_bytes(url: &str) -> Result<Vec<u8>, JsValue> {
    let buffer = get_array_buffer(url).await?;
    let uint8_array = Uint8Array::new(&buffer);
    return Ok(uint8_array.to_vec());
}

// ============================================================
// JSON helpers
// ============================================================

/// Десериализует `JsValue` в `T` через JSON.
fn deserialize_json<T: DeserializeOwned>(value: &JsValue) -> Result<T, JsValue> {
    let json_string = js_sys::JSON::stringify(value)
        .map_err(|_| JsValue::from_str("JSON.stringify failed"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("JSON.stringify result is not a string"))?;

    return serde_json::from_str(&json_string)
        .map_err(|e| JsValue::from_str(&format!("deserialize error: {}", e)));
}

// ============================================================
// Abort
// ============================================================

/// Создаёт `AbortController` для отмены запроса.
pub fn make_abort_controller() -> AbortController {
    return AbortController::new().expect("failed to create AbortController");
}

/// Создаёт GET-запрос с возможностью отмены.
pub fn make_cancellable_get_request(
    url: &str,
    controller: &AbortController,
) -> Result<Request, JsValue> {
    let opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(RequestMode::Cors);
    opts.set_signal(Some(&controller.signal()));

    return Request::new_with_str_and_init(url, &opts);
}

/// Отправляет запрос с отменой.
pub async fn get_cancellable(
    url: &str,
    controller: &AbortController,
) -> Result<Response, JsValue> {
    let request = make_cancellable_get_request(url, controller)?;
    return send_request(&request).await;
}

// ============================================================
// Headers
// ============================================================

/// Читает заголовок из `Response`.
pub fn get_response_header(resp: &Response, name: &str) -> Option<String> {
    return resp.headers().get(name).ok().flatten();
}

/// Проверяет наличие заголовка.
pub fn has_response_header(resp: &Response, name: &str) -> bool {
    return resp.headers().has(name).unwrap_or(false);
}

/// Создаёт `Headers` с заданными заголовками.
pub fn make_headers(headers: &[(&str, &str)]) -> Result<Headers, JsValue> {
    let result = Headers::new()?;
    for (name, value) in headers {
        result.set(name, value)?;
    }
    return Ok(result);
}

// ============================================================
// Свойства Response
// ============================================================

/// Возвращает HTTP-статус ответа.
pub fn response_status(resp: &Response) -> u16 {
    return resp.status();
}

/// Возвращает `true`, если статус 2xx.
pub fn response_is_ok(resp: &Response) -> bool {
    return resp.ok();
}

/// Возвращает URL ответа.
pub fn response_url(resp: &Response) -> String {
    return resp.url();
}
```

**Ключевые моменты:**

- **`use js_sys::{ArrayBuffer, Uint8Array};`** — импорт из `js-sys`, не из `web-sys`.
- **`use web_sys::{AbortController, Headers, Request, RequestInit, RequestMode, Response};`** — Web API из `web-sys`.
- **`send_request` проверяет `resp.ok()`.** При статусе не 2xx возвращает `Err`.
- **`get_json` использует `deserialize_json`.** `JsValue` → `JSON.stringify` → `serde_json::from_str`.
- **`get_bytes` возвращает `Vec<u8>`.** `ArrayBuffer` → `Uint8Array::new` → `to_vec`.

---

## `src/app.rs`

```rust
//! Логика приложения: «Rust API Client».
//!
//! Три секции:
//! 1. **GET** — запросы к JSONPlaceholder.
//! 2. **POST** — отправка данных и получение ответа.
//! 3. **Abort** — отмена запросов через `AbortController`.

use std::cell::RefCell;
use std::rc::Rc;

use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;
use web_sys::AbortController;

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::fetch;
use crate::log;

// ============================================================
// Модели данных
// ============================================================

/// Пост из JSONPlaceholder.
#[derive(Serialize, Deserialize, Clone, Debug)]
struct Post {
    #[serde(default)]
    id: u32,
    #[serde(default)]
    title: String,
    #[serde(default)]
    body: String,
    #[serde(default)]
    user_id: u32,
}

// ============================================================
// URL-ы API
// ============================================================

const POSTS_URL: &str = "https://jsonplaceholder.typicode.com/posts/1";
const POSTS_LIST_URL: &str = "https://jsonplaceholder.typicode.com/posts?_limit=5";
const CREATE_POST_URL: &str = "https://jsonplaceholder.typicode.com/posts";

// ============================================================
// Точка входа
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Rust API Client")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Демонстрация Fetch API: GET, POST, JSON, binary data, Abort. \
         Запросы идут к JSONPlaceholder — публичному тестовому API.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 1: GET-запросы
    // ============================================================

    let section1 = build_section(&container, "1. GET: загрузка данных")?;

    let get_single_btn = dom::create_element("button")?;
    get_single_btn.set_text_content(Some("GET один пост (#1)"));
    let get_list_btn = dom::create_element("button")?;
    get_list_btn.set_text_content(Some("GET список постов"));
    let clear_btn = dom::create_element("button")?;
    clear_btn.set_text_content(Some("Очистить"));

    dom::append(&section1, &get_single_btn)?;
    dom::append(&section1, &get_list_btn)?;
    dom::append(&section1, &clear_btn)?;

    let get_result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&get_result), "api-result")?;
    get_result.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section1, &get_result)?;

    // ---- Обработчик «GET один пост» ----
    {
        let get_result = get_result.clone();
        callbacks::listen(
            &get_single_btn,
            "click",
            move |_event| {
                let get_result = get_result.clone();
                get_result.set_text_content(Some("Загрузка..."));

                wasm_bindgen_futures::spawn_local(async move {
                    match fetch::get_json::<Post>(POSTS_URL).await {
                        Ok(post) => {
                            let text: String = format!(
                                "✅ GET /posts/1\n\
                                 \n\
                                 id: {}\n\
                                 userId: {}\n\
                                 title: {}\n\
                                 body: {}",
                                post.id, post.user_id, post.title, post.body
                            );
                            get_result.set_text_content(Some(text.as_str()));
                            log!("GET /posts/1: success");
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            get_result.set_text_content(Some(text.as_str()));
                            log!("GET /posts/1 error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «GET список постов» ----
    {
        let get_result = get_result.clone();
        callbacks::listen(
            &get_list_btn,
            "click",
            move |_event| {
                let get_result = get_result.clone();
                get_result.set_text_content(Some("Загрузка..."));

                wasm_bindgen_futures::spawn_local(async move {
                    match fetch::get_json::<Vec<Post>>(POSTS_LIST_URL).await {
                        Ok(posts) => {
                            let mut lines = Vec::new();
                            lines.push(format!(
                                "✅ GET /posts?_limit=5 — {} постов",
                                posts.len()
                            ));
                            lines.push(String::new());
                            for post in posts.iter().take(5) {
                                lines.push(format!(
                                    "#{}: {}",
                                    post.id,
                                    fetch_truncate(&post.title, 50)
                                ));
                            }
                            let text = lines.join("\n");
                            get_result.set_text_content(Some(text.as_str()));
                            log!("GET list: {} posts", posts.len());
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            get_result.set_text_content(Some(text.as_str()));
                            log!("GET list error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Очистить» ----
    {
        let get_result = get_result.clone();
        callbacks::listen(
            &clear_btn,
            "click",
            move |_event| {
                get_result.set_text_content(Some("(нажмите кнопку)"));
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 2: POST-запросы
    // ============================================================

    let section2 = build_section(&container, "2. POST: отправка данных")?;

    let post_btn = dom::create_element("button")?;
    post_btn.set_text_content(Some("POST создать пост"));
    let post_error_btn = dom::create_element("button")?;
    post_error_btn.set_text_content(Some("POST с HTTP-ошибкой"));
    let clear_post_btn = dom::create_element("button")?;
    clear_post_btn.set_text_content(Some("Очистить"));

    dom::append(&section2, &post_btn)?;
    dom::append(&section2, &post_error_btn)?;
    dom::append(&section2, &clear_post_btn)?;

    let post_result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&post_result), "api-result")?;
    post_result.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section2, &post_result)?;

    // ---- Обработчик «POST создать пост» ----
    {
        let post_result = post_result.clone();
        callbacks::listen(
            &post_btn,
            "click",
            move |_event| {
                let post_result = post_result.clone();
                post_result.set_text_content(Some("Отправка..."));

                wasm_bindgen_futures::spawn_local(async move {
                    let new_post = Post {
                        id: 0,
                        title: "Новый пост из Rust".into(),
                        body: "Тело, созданное в Rust + Wasm".into(),
                        user_id: 42,
                    };

                    match fetch::post_json_response::<Post, Post>(
                        CREATE_POST_URL,
                        &new_post,
                    )
                    .await
                    {
                        Ok(created) => {
                            let text: String = format!(
                                "✅ POST /posts\n\
                                 \n\
                                 Сервер вернул:\n\
                                 id: {}\n\
                                 title: {}\n\
                                 body: {}\n\
                                 userId: {}",
                                created.id,
                                created.title,
                                created.body,
                                created.user_id
                            );
                            post_result.set_text_content(Some(text.as_str()));
                            log!("POST /posts: success, id={}", created.id);
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            post_result.set_text_content(Some(text.as_str()));
                            log!("POST /posts error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «POST с HTTP-ошибкой» ----
    {
        let post_result = post_result.clone();
        callbacks::listen(
            &post_error_btn,
            "click",
            move |_event| {
                let post_result = post_result.clone();
                post_result.set_text_content(Some(
                    "Отправка на несуществующий ресурс...",
                ));

                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/999999";
                    match fetch::get_json::<Post>(url).await {
                        Ok(_) => {
                            post_result.set_text_content(Some(
                                "(неожиданно: 404 не сработал)",
                            ));
                        }
                        Err(e) => {
                            let text: String = format!(
                                "✅ HTTP-ошибка обработана корректно:\n\
                                 \n\
                                 {:?}\n\
                                 \n\
                                 Приложение не паникует — \
                                 get_json вернул Err, мы его показали.",
                                e
                            );
                            post_result.set_text_content(Some(text.as_str()));
                            log!("HTTP error handled: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Очистить» ----
    {
        let post_result = post_result.clone();
        callbacks::listen(
            &clear_post_btn,
            "click",
            move |_event| {
                post_result.set_text_content(Some("(нажмите кнопку)"));
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 3: Abort — отмена запроса
    // ============================================================

    let section3 = build_section(&container, "3. Abort: отмена запроса")?;

    let abort_hint = dom::create_paragraph(
        "Запустите долгий запрос и отмените его до завершения. \
         AbortController связан с запросом через signal. \
         При отмене fetch возвращает ошибку AbortError.",
    )?;
    dom::add_class(&dom::as_html(&abort_hint), "hint")?;
    dom::append(&section3, &abort_hint)?;

    let start_slow_btn = dom::create_element("button")?;
    start_slow_btn.set_text_content(Some("Запустить долгий запрос"));
    let abort_btn = dom::create_element("button")?;
    abort_btn.set_text_content(Some("Отменить"));
    let clear_abort_btn = dom::create_element("button")?;
    clear_abort_btn.set_text_content(Some("Очистить"));

    dom::append(&section3, &start_slow_btn)?;
    dom::append(&section3, &abort_btn)?;
    dom::append(&section3, &clear_abort_btn)?;

    let abort_result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&abort_result), "api-result")?;
    abort_result.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section3, &abort_result)?;

    // Хранилище для текущего AbortController
    let current_controller: Rc<RefCell<Option<AbortController>>> =
        Rc::new(RefCell::new(None));

    // ---- Обработчик «Запустить долгий запрос» ----
    {
        let abort_result = abort_result.clone();
        let current_controller = current_controller.clone();

        callbacks::listen(
            &start_slow_btn,
            "click",
            move |_event| {
                let abort_result = abort_result.clone();
                let current_controller = current_controller.clone();

                let controller = fetch::make_abort_controller();
                *current_controller.borrow_mut() = Some(controller.clone());

                abort_result.set_text_content(Some(
                    "Запрос выполняется (можно отменить)...",
                ));

                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://httpbin.org/delay/5";

                    match fetch::get_cancellable(url, &controller).await {
                        Ok(_) => {
                            abort_result.set_text_content(Some(
                                "✅ Запрос завершился (не был отменён)",
                            ));
                        }
                        Err(e) => {
                            let text: String = format!(
                                "✅ Запрос отменён или завершился ошибкой:\n{:?}",
                                e
                            );
                            abort_result.set_text_content(Some(text.as_str()));
                            log!("Cancellable request result: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Отменить» ----
    {
        let abort_result = abort_result.clone();
        let current_controller = current_controller.clone();

        callbacks::listen(
            &abort_btn,
            "click",
            move |_event| {
                let controller_opt = current_controller.borrow();
                if let Some(controller) = controller_opt.as_ref() {
                    controller.abort();
                    abort_result.set_text_content(Some("🛑 Запрос отменён"));
                    log!("AbortController.abort() вызван");
                } else {
                    abort_result.set_text_content(Some("(нет активного запроса)"));
                }
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Очистить» ----
    {
        let abort_result = abort_result.clone();
        let current_controller = current_controller.clone();

        callbacks::listen(
            &clear_abort_btn,
            "click",
            move |_event| {
                abort_result.set_text_content(Some("(нажмите кнопку)"));
                *current_controller.borrow_mut() = None;
            },
            &closures,
        )?;
    }

    // ============================================================
    // Финальное логирование
    // ============================================================

    log!("Rust API Client готов");

    std::mem::forget(closures);
    std::mem::forget(current_controller);

    Ok(())
}

// ============================================================
// Вспомогательные функции
// ============================================================

/// Создаёт секцию с заголовком и возвращает её.
fn build_section(
    parent: &web_sys::Element,
    title: &str,
) -> Result<web_sys::Element, JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some(title));
    dom::append(&section, &h2)?;

    dom::append(parent, &section)?;
    Ok(section)
}

/// Обрезает длинную строку для предпросмотра.
fn fetch_truncate(value: &str, max_chars: usize) -> String {
    let char_count = value.chars().count();
    if char_count <= max_chars {
        return value.to_string();
    }
    let truncated: String = value.chars().take(max_chars).collect();
    return format!("{}...", truncated);
}
```

**Импорты — обратите внимание:**

- **`use wasm_bindgen::prelude::*;`** — только он. `JsCast` здесь не нужен: мы нигде не вызываем `dyn_into` напрямую, всё делается в `fetch.rs`.
- **`use web_sys::AbortController;`** — только `AbortController`. Остальные типы Web API скрыты внутри `fetch.rs`.
- **`use crate::fetch;`** — модуль, объявленный в `lib.rs`.
- **Нет `const API_BASE`.** Удалена как неиспользуемая — URL-ы заданы полностью в константах.

---

## `tests/wasm_fetch.rs`

```rust
//! Тесты для Fetch API.
//!
//! Покрытие:
//! - Создание Request через RequestInit.
//! - Чтение Response (статус, ok, url, headers).
//! - GET-запросы к реальному API (JSONPlaceholder).
//! - Обработка HTTP-ошибок.
//! - AbortController.
use wasm_bindgen_test::*;

use hello_wasm02::fetch;

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// Создание запросов
// ============================================================

#[wasm_bindgen_test]
fn test_make_get_request() {
    let request = fetch::make_get_request(
        "https://jsonplaceholder.typicode.com/posts/1",
        web_sys::RequestMode::Cors,
    );
    assert!(request.is_ok());
}

#[wasm_bindgen_test]
fn test_make_post_json_request() {
    use serde::Serialize;

    #[derive(Serialize)]
    struct Data {
        title: String,
    }

    let data = Data {
        title: "test".into(),
    };

    let request = fetch::make_post_json_request(
        "https://jsonplaceholder.typicode.com/posts",
        &data,
        web_sys::RequestMode::Cors,
    );
    assert!(request.is_ok());
}

// ============================================================
// GET-запросы (реальные API)
// ============================================================

#[wasm_bindgen_test]
async fn test_get_json_single_post() {
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct Post {
        id: u32,
        #[serde(default)]
        title: String,
    }

    let result = fetch::get_json::<Post>(
        "https://jsonplaceholder.typicode.com/posts/1",
    )
    .await;

    assert!(result.is_ok(), "GET /posts/1 should succeed: {:?}", result);
    let post = result.unwrap();
    assert_eq!(post.id, 1);
    assert!(!post.title.is_empty());
}

#[wasm_bindgen_test]
async fn test_get_json_list() {
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct Post {
        #[serde(default)]
        id: u32,
    }

    let result = fetch::get_json::<Vec<Post>>(
        "https://jsonplaceholder.typicode.com/posts?_limit=3",
    )
    .await;

    assert!(result.is_ok());
    let posts = result.unwrap();
    assert_eq!(posts.len(), 3);

    // Проверяем, что поле id действительно десериализовано
    // и содержит осмысленные значения. JSONPlaceholder возвращает
    // id от 1, поэтому проверяем, что все id > 0.
    assert!(
        posts.iter().all(|p| p.id > 0),
        "all posts should have non-zero id: {:?}",
        posts.iter().map(|p| p.id).collect::<Vec<_>>()
    );
}

#[wasm_bindgen_test]
async fn test_get_text() {
    let result =
        fetch::get_text("https://jsonplaceholder.typicode.com/posts/1").await;

    assert!(result.is_ok());
    let text = result.unwrap();
    assert!(text.contains("userId"));
}

// ============================================================
// HTTP-ошибки
// ============================================================

#[wasm_bindgen_test]
async fn test_get_404_returns_error() {
    let result = fetch::get_json::<serde_json::Value>(
        "https://jsonplaceholder.typicode.com/posts/999999",
    )
    .await;

    assert!(result.is_err(), "404 should return Err");
}

#[wasm_bindgen_test]
async fn test_response_status_and_ok() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .unwrap();

    assert_eq!(fetch::response_status(&resp), 200);
    assert!(fetch::response_is_ok(&resp));
}

// ============================================================
// Headers
// ============================================================

#[wasm_bindgen_test]
async fn test_response_headers() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .unwrap();

    let content_type = fetch::get_response_header(&resp, "content-type");
    assert!(content_type.is_some());
    assert!(content_type.unwrap().contains("application/json"));
}

// ============================================================
// POST-запросы
// ============================================================

#[wasm_bindgen_test]
async fn test_post_json() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, Debug)]
    struct Post {
        #[serde(default)]
        id: u32,
        title: String,
        body: String,
        #[serde(default)]
        user_id: u32,
    }

    let new_post = Post {
        id: 0,
        title: "Test".into(),
        body: "Body".into(),
        user_id: 1,
    };

    let result = fetch::post_json_response::<Post, Post>(
        "https://jsonplaceholder.typicode.com/posts",
        &new_post,
    )
    .await;

    assert!(result.is_ok(), "POST should succeed: {:?}", result);
    let created = result.unwrap();
    assert_eq!(created.title, "Test");
}

// ============================================================
// Abort
// ============================================================

#[wasm_bindgen_test]
fn test_make_abort_controller() {
    let controller = fetch::make_abort_controller();
    assert!(!controller.signal().aborted());
}

#[wasm_bindgen_test]
fn test_abort_signal_initially_not_aborted() {
    let controller = fetch::make_abort_controller();
    assert!(!controller.signal().aborted());
}

#[wasm_bindgen_test]
fn test_abort_signal_after_abort() {
    let controller = fetch::make_abort_controller();
    controller.abort();
    assert!(controller.signal().aborted());
}
```

**Что проверяют тесты:**

| Тест | Что проверяет |
|---|---|
| `test_make_get_request` | Создание GET-запроса |
| `test_make_post_json_request` | Создание POST-запроса |
| `test_get_json_single_post` | Реальный GET, десериализация |
| `test_get_json_list` | GET списка, `Vec<T>` + проверка `id` |
| `test_get_text` | GET как текст |
| `test_get_404_returns_error` | HTTP-ошибка → `Err` |
| `test_response_status_and_ok` | `Response.status()` и `ok()` |
| `test_response_headers` | Чтение заголовков |
| `test_post_json` | POST-запрос |
| `test_make_abort_controller` | Создание `AbortController` |
| `test_abort_signal_initially_not_aborted` | Signal до отмены |
| `test_abort_signal_after_abort` | Signal после отмены |

**Сетевая зависимость.** Тесты делают реальные HTTP-запросы. Нужен интернет. В CI это может быть проблемой — в учебном проекте допустимо.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   API client
   ============================================================ */

.api-result {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #2563eb;
    margin: 0.75rem 0;
    min-height: 3rem;
    max-height: 400px;
    overflow-y: auto;
    word-break: break-all;
}
```

---

## Команды запуска

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

Ожидаемый результат: без ошибок и предупреждений.

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Проверьте сценарии:

**Секция 1 «GET»:**
1. «GET один пост (#1)» — загрузится пост, отобразится в блоке.
2. «GET список постов» — 5 постов.

**Секция 2 «POST»:**
1. «POST создать пост» — сервер вернёт созданный объект с новым `id`.
2. «POST с HTTP-ошибкой» — 404, приложение корректно обработает.

**Секция 3 «Abort»:**
1. «Запустить долгий запрос» — запрос к `httpbin.org/delay/5`.
2. «Отменить» — запрос прервётся.

### Тесты

```bash
wasm-pack test --headless --chrome --test wasm_fetch
```

```bash
xvfb-run -a wasm-pack test --headless --firefox \
  --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver \
  --test wasm_fetch
```

**Ожидаемый результат:** 12 тестов `ok` в обоих браузерах, без warnings.

---

## Что важно запомнить

**1. `ArrayBuffer` и `Uint8Array` — из `js-sys`, не из `web-sys`.** Это ECMAScript-типы. В `Cargo.toml` их нельзя добавлять в features `web-sys` — сборка упадёт. Используйте `js_sys::ArrayBuffer` и `js_sys::Uint8Array` в коде.

**2. `pub mod fetch;` в `lib.rs`.** Rust не обнаруживает модули автоматически. Создали `src/fetch.rs` — добавьте объявление в `lib.rs`. Без этого `use crate::fetch;` в `app.rs` не скомпилируется.

**3. `JsCast` нужен только там, где вызываются его методы.** В `app.rs` мы нигде не вызываем `dyn_into` напрямую — только в `fetch.rs`. Импорт `JsCast` в `app.rs` лишний.

**4. Удаляйте неиспользуемые константы.** `API_BASE` не пригодилась — URL-ы заданы полностью. Оставили — получили warning `dead_code`.

**5. Используйте все поля в тестовых структурах.** Если поле десериализуется, но не проверяется — компилятор выдаёт `dead_code`. Используйте поле в assertion или удалите его.

**6. Fetch API полностью асинхронный.** Каждый шаг — `Promise`, превращается в `Future` через `JsFuture`.

**7. `send_request` проверяет `ok()`.** Не-2xx статус → `Err`. Это то, что позволяет обрабатывать HTTP-ошибки как обычные ошибки Rust.

**8. `deserialize_json` через `JSON.stringify` + `serde_json`.** `JsValue` → строка → `T`.

**9. Abort через `AbortController`.** `controller.signal()` передаётся в `RequestInit::set_signal()`. `controller.abort()` отменяет запрос.

**10. `spawn_local` для асинхронных обработчиков.** Обработчик клика синхронный. `spawn_local(async { ... })` запускает `Future` в фоне.

**11. CORS.** Запросы на другой origin требуют `RequestMode::Cors`. Без этого браузер заблокирует запрос. JSONPlaceholder и httpbin.org поддерживают CORS.

---

## Итог главы

Мы прошли путь от теории Fetch API до полноценного API client. Что мы изучили:

- **`Request` и `RequestInit`** — создание запросов с методом, телом, заголовками, режимом.
- **`Response`** — чтение статуса, `ok()`, заголовков, URL.
- **`Headers`** — работа с заголовками запроса и ответа.
- **`fetch`** — `window.fetch_with_request()`, `JsFuture`, `Promise`.
- **GET, POST, JSON** — все основные сценарии.
- **Binary data** — `ArrayBuffer` + `Uint8Array` через `js-sys`.
- **HTTP-ошибки** — `response.ok()` и преобразование в `Err`.
- **Abort** — `AbortController`, `AbortSignal`, отмена запроса.
- **Практический проект** — API client с тремя секциями.

---

**Что дальше.** **Глава 17. Streams.** Мы разберём `ReadableStream`, `WritableStream`, `TransformStream`, readers, writers, backpressure и потоковую обработку данных. Fetch API возвращает `ReadableStream` через `response.body()` — это основа для больших данных, которые не помещаются в память целиком.