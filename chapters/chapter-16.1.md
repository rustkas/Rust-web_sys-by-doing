# 16.1. Первый GET: Request, Response, JsFuture

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
│   ├── app.rs                      # демо: один GET-запрос
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── fetch.rs                    # НОВОЕ: базовые обёртки Fetch
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
│   └── wasm_fetch.rs               # НОВОЕ: 6 тестов базового Fetch
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +1 правило для .api-result
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/fetch.rs` — **новый модуль**. В этой подглаве содержит **только базовые функции**: `make_get_request`, `send_request`, `get`, `get_json`, `get_text` и три утилиты для чтения свойств `Response`. Всё остальное (POST, заголовки, retry, timeout) появится в следующих подглавах.
- `src/app.rs` — **переписан с нуля** под минимальное демо. Одна секция, одна кнопка, один блок результата.
- `src/lib.rs` — **+`pub mod fetch;`**. Без этой строки `use crate::fetch;` в `app.rs` не скомпилируется.
- `tests/wasm_fetch.rs` — **новый файл** с 6 тестами.
- `Cargo.toml` — **добавлены features**: `Headers`, `Request`, `RequestInit`, `RequestMode`, `Response`. Остальные (AbortController, AbortSignal) появятся в 16.7.

**Что важно.** `ArrayBuffer` и `Uint8Array` **не добавляются** в features `web-sys`. Они живут в `js-sys` и появятся в подглаве 16.9. В этой подглаве мы работаем только с JSON и текстом.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Fetch API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API |
| `Request` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Request |
| `RequestInit` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/RequestInit |
| `RequestMode` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Request/mode |
| `Response` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Response |
| `Window.fetch()` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/fetch |
| `Promise` (MDN) | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise |
| `JsFuture` (docs.rs) | https://docs.rs/wasm-bindgen-futures/latest/wasm_bindgen_futures/struct.JsFuture.html |
| wasm-bindgen: fetch example | https://rustwasm.github.io/docs/wasm-bindgen/examples/fetch.html |

---

## Механика Fetch API

Прежде чем писать код, разберём **что происходит при GET-запросе** шаг за шагом.

**Шаг 1. Создание запроса.** Мы собираем `Request` из URL и `RequestInit` (опций). В опциях указываем метод (`GET`), режим (`Cors`), заголовки, тело. Пока это только описание — запрос ещё не отправлен.

**Шаг 2. Отправка.** Вызываем `window.fetch_with_request(&request)`. Браузер **немедленно возвращает** `Promise<Response>` — обещание, что когда-то придёт ответ. Сам HTTP-запрос уходит в фоне.

**Шаг 3. Ожидание.** В JavaScript это `await promise`. В Rust — `JsFuture::from(promise).await`. `JsFuture` превращает JS-`Promise` в Rust-`Future`, который можно ожидать через `await`.

**Шаг 4. Получение ответа.** Когда сервер ответил, `Promise` разрешается значением типа `Response`. В Rust это `JsValue`, который приводится к `Response` через `dyn_into::<Response>()`.

**Шаг 5. Проверка статуса.** `response.ok()` — `true` для 2xx. Если не 2xx — возвращаем `Err`.

**Шаг 6. Чтение тела.** `response.json()` снова возвращает `Promise` — на этот раз `Promise<JsValue>` с распарсенным JSON. Снова `JsFuture`, снова `await`. Получаем `JsValue`, который десериализуем через `serde_json`.

**Что важно понять.** Каждый шаг после отправки — это асинхронная операция. Между шагами управление возвращается в event loop, UI остаётся отзывчивым. Именно поэтому Fetch API не блокирует браузер, в отличие от синхронного `XMLHttpRequest`.

---

## `Cargo.toml`

Показаны **только новые части**. Остальное — из главы 15.

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    # ... features из предыдущих глав ...

    # Fetch API (базовые)
    "Headers",
    "Request",
    "RequestInit",
    "RequestMode",
    "Response",

    # Прочее
    "console",
]
```

**Что добавилось:**

- **`Headers`** — заголовки HTTP-запроса и ответа.
- **`Request`** — объект запроса.
- **`RequestInit`** — опции запроса (метод, режим, тело).
- **`RequestMode`** — CORS-режим.
- **`Response`** — объект ответа.

**Что НЕ добавилось и почему:**

- **`AbortController` / `AbortSignal`** — появятся в 16.7 (Timeout и отмена).
- **`Blob`** — появится в 16.9 (Типы ответов).
- **`ArrayBuffer` / `Uint8Array`** — это features `js-sys`, а не `web-sys`. Появятся в 16.9.

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
//! - `make_get_request` — создание GET-запроса.
//! - `send_request` — отправка и проверка `ok()`.
//! - `get`, `get_json`, `get_text` — высокоуровневые функции.
//! - Утилиты для чтения свойств `Response`.
//!
//! ## Что будет дальше
//!
//! В следующих подглавах модуль расширится:
//! - 16.2 — POST, PUT, DELETE, тело запроса.
//! - 16.3 — заголовки, case-insensitivity.
//! - 16.4 — аутентификация.
//! - 16.5 — классификация ошибок.
//! - 16.6 — retry.
//! - 16.7 — timeout и Abort.
//! - 16.8 — параллельные запросы.
//! - 16.9 — типы ответов, binary data.
//! - 16.10 — Streams.
//! - 16.11 — CORS.
//! - 16.12 — API client.
#![allow(dead_code)]

use serde::de::DeserializeOwned;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use web_sys::{Request, RequestInit, RequestMode, Response};

// ============================================================
// Создание запросов
// ============================================================

/// Создаёт GET-запрос.
///
/// `mode` — CORS-режим. Для запросов на другой origin используйте
/// `RequestMode::Cors`. Для запросов на тот же origin —
/// `RequestMode::SameOrigin`.
pub fn make_get_request(url: &str, mode: RequestMode) -> Result<Request, JsValue> {
    let opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(mode);

    return Request::new_with_str_and_init(url, &opts);
}

// ============================================================
// Отправка запросов
// ============================================================

/// Отправляет запрос и возвращает `Response`.
///
/// Проверяет `response.ok()`: если ответ не 2xx — возвращает `Err`
/// с информацией о статусе.
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

// ============================================================
// Высокоуровневые функции
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
// JSON helpers
// ============================================================

/// Десериализует `JsValue` в `T` через JSON.
///
/// Промежуточный шаг: `JsValue` → JSON-строка → `T`.
fn deserialize_json<T: DeserializeOwned>(value: &JsValue) -> Result<T, JsValue> {
    let json_string = js_sys::JSON::stringify(value)
        .map_err(|_| JsValue::from_str("JSON.stringify failed"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("JSON.stringify result is not a string"))?;

    return serde_json::from_str(&json_string)
        .map_err(|e| JsValue::from_str(&format!("deserialize error: {}", e)));
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

- **`make_get_request`** — создаёт `Request` через `RequestInit`. Устанавливает метод и режим CORS. Возвращает `Result` — создание может провалиться, если URL невалиден.
- **`send_request`** — отправляет запрос через `window.fetch_with_request()`. Ждёт `JsFuture`. Проверяет, что ответ — `Response` через `is_instance_of`. Приводит к `Response` через `dyn_into`. Проверяет `ok()` и возвращает `Err` при не-2xx.
- **`get_json`** — цепочка: `get` → `resp.json()` → `JsFuture` → `deserialize_json`. `deserialize_json` делает `JSON.stringify` через `js_sys::JSON`, затем `serde_json::from_str`.
- **`response_status`, `response_is_ok`, `response_url`** — утилиты для чтения свойств `Response`. Нужны, потому что `Response` имеет методы, но мы хотим обернуть их в функции для тестируемости.

**Почему `is_instance_of::<Response>()` перед `dyn_into`.** `dyn_into` сам проверяет тип и возвращает `Err` при неверном. Но если `Promise` разрешился не `Response` (например, из-за ошибки в JS-обёртке), мы получим невнятную ошибку. `is_instance_of` даёт явное сообщение.

**Почему `RequestMode::Cors`.** JSONPlaceholder — другой origin. Без `Cors` браузер заблокирует запрос. Подробнее — в 16.11.

---

## `src/app.rs`

```rust
//! Логика приложения: «Первый GET-запрос».
//!
//! Одна секция: кнопка «Загрузить пост» — отправляет GET-запрос
//! к JSONPlaceholder, десериализует ответ в структуру Post
//! и показывает результат в блоке.

use std::cell::RefCell;
use std::rc::Rc;

use serde::Deserialize;
use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::fetch;
use crate::log;

// ============================================================
// Модель данных
// ============================================================

/// Пост из JSONPlaceholder.
#[derive(Deserialize, Debug)]
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

const POST_URL: &str = "https://jsonplaceholder.typicode.com/posts/1";

// ============================================================
// Точка входа
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Первый GET-запрос")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Нажмите кнопку — приложение отправит GET-запрос к JSONPlaceholder \
         и покажет полученный пост.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let load_btn = dom::create_element("button")?;
    load_btn.set_text_content(Some("Загрузить пост"));
    dom::append(&container, &load_btn)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&container, &result)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Обработчик «Загрузить пост» ----
    {
        let result = result.clone();
        callbacks::listen(
            &load_btn,
            "click",
            move |_event| {
                let result = result.clone();
                result.set_text_content(Some("Загрузка..."));

                wasm_bindgen_futures::spawn_local(async move {
                    match fetch::get_json::<Post>(POST_URL).await {
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
                            result.set_text_content(Some(text.as_str()));
                            log!("GET /posts/1: success");
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            result.set_text_content(Some(text.as_str()));
                            log!("GET /posts/1 error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    log!("Демо «Первый GET» готово");

    std::mem::forget(closures);

    Ok(())
}
```

**Что демонстрирует `app.rs`:**

- **Одна кнопка** — «Загрузить пост».
- **Один блок результата** — показывает ответ или ошибку.
- **Один GET-запрос** — к `jsonplaceholder.typicode.com/posts/1`.

**Что важно понять:**

- **`spawn_local`** — обязателен. Обработчик клика синхронный, а `get_json` асинхронный. `spawn_local` запускает `Future` в фоне, UI не блокируется.
- **Импорт `JsCast` не нужен.** Все приведения типов (`dyn_into`, `is_instance_of`) происходят внутри `fetch.rs`. В `app.rs` мы работаем только с высокоуровневыми функциями.
- **`#[serde(default)]` на полях.** Если JSONPlaceholder вернёт JSON без какого-то поля, десериализация не провалится — поле получит значение по умолчанию.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   API client (секции Fetch)
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

**Что делает:** блок результата с синей полосой слева — визуально отличается от оранжевых блоков Storage. Пользователь сразу видит, что это раздел Fetch.

---

## `tests/wasm_fetch.rs`

```rust
//! Тесты для Fetch API: базовые GET-запросы.
//!
//! Покрытие:
//! - Создание Request через RequestInit.
//! - Отправка запроса, чтение Response (статус, ok, url).
//! - GET JSON и GET text.
//! - Механика Promise → JsFuture.
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
    assert!(request.is_ok(), "GET request should be created");
}

// ============================================================
// GET-запросы
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
async fn test_get_text() {
    let result =
        fetch::get_text("https://jsonplaceholder.typicode.com/posts/1").await;

    assert!(result.is_ok(), "GET text should succeed: {:?}", result);
    let text = result.unwrap();
    assert!(text.contains("userId"));
    assert!(text.contains("title"));
}

// ============================================================
// Response: свойства
// ============================================================

#[wasm_bindgen_test]
async fn test_response_status_and_ok() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .expect("GET should succeed");

    assert_eq!(fetch::response_status(&resp), 200);
    assert!(fetch::response_is_ok(&resp));
}

#[wasm_bindgen_test]
async fn test_response_url() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .expect("GET should succeed");

    let url = fetch::response_url(&resp);
    assert!(
        url.contains("jsonplaceholder.typicode.com"),
        "URL should contain host: {}",
        url
    );
    assert!(
        url.contains("posts/1"),
        "URL should contain path: {}",
        url
    );
}

// ============================================================
// Механика Promise → JsFuture
// ============================================================

#[wasm_bindgen_test]
async fn test_jsfuture_promise_resolution() {
    use wasm_bindgen_futures::JsFuture;

    // Создаём Promise, который разрешается значением 42
    let promise = js_sys::Promise::resolve(&wasm_bindgen::JsValue::from(42));

    // Превращаем в Future и ждём
    let value = JsFuture::from(promise)
        .await
        .expect("Promise should resolve");

    assert_eq!(value.as_f64(), Some(42.0));
}
```

**Что проверяют тесты:**

| Тест | Что проверяет |
|---|---|
| `test_make_get_request` | Создание `Request` через `RequestInit` |
| `test_get_json_single_post` | Реальный GET, десериализация в структуру |
| `test_get_text` | GET как текст |
| `test_response_status_and_ok` | `Response.status() == 200` и `ok() == true` |
| `test_response_url` | `Response.url()` содержит host и path |
| `test_jsfuture_promise_resolution` | Прямой тест механики `Promise` → `JsFuture` |

**Тест `test_jsfuture_promise_resolution` — самый важный.** Он проверяет не Fetch, а **фундамент**, на котором Fetch работает. Если этот тест проходит, читатель точно понимает механику `JsFuture`.

**Всего тестов: 6.**

---

## Команды запуска

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

Ожидаемый результат — без ошибок и предупреждений.

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Нажмите «Загрузить пост». Появится:

```
✅ GET /posts/1

id: 1
userId: 1
title: sunt aut facere repellat provident occaecati excepturi optio reprehenderit
body: quia et suscipit...
```

### Тесты

```bash
# Только тесты Fetch в Chrome
wasm-pack test --headless --chrome --test wasm_fetch
```

```bash
# Только тесты Fetch в Firefox
xvfb-run -a wasm-pack test --headless --firefox \
  --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver \
  --test wasm_fetch
```

```bash
# Все тесты проекта
npm run test:all
```

**Ожидаемый результат:** 6 тестов `ok` в обоих браузерах, без warnings.

---

## Что важно запомнить

**1. Fetch API — асинхронный по своей природе.** `window.fetch_with_request()` возвращает `Promise<Response>`, который мы ждём через `JsFuture::from(promise).await`.

**2. `Request` создаётся через `RequestInit`.** `RequestInit::new()` → `set_method("GET")` → `set_mode(RequestMode::Cors)` → `Request::new_with_str_and_init(url, &opts)`.

**3. `RequestMode` определяет CORS-поведение.** `Cors` — для cross-origin, `SameOrigin` — для same-origin, `NoCors` — для публичных ресурсов без CORS.

**4. `Response.ok()` — проверка статуса.** `true` для 2xx, `false` для остальных. Возвращаем `Err` при `false`.

**5. `response.json()` снова возвращает `Promise`.** Читаем через `JsFuture`, получаем `JsValue`. Десериализуем через `serde_json`.

**6. `JsValue` → `String` → `T`.** `serde_json` работает со строками, а `JsValue` — это JavaScript-объект. `js_sys::JSON::stringify` — мост.

**7. `spawn_local` для асинхронных обработчиков.** Обработчик клика синхронный. `spawn_local(async { ... })` запускает `Future` в фоне.

**8. `dyn_into` для приведения `JsValue` к `Response`.** Требует `JsCast`, который импортируется в `fetch.rs`, не в `app.rs`.

**9. `#[serde(default)]` для устойчивой десериализации.** Если поле отсутствует в JSON — берётся значение по умолчанию. Полезно для внешних API, где схема может меняться.

**10. Разделение уровней.** `fetch.rs` знает про `Request`, `Response`, `JsValue`. `app.rs` работает только с высокоуровневыми функциями — `get_json::<Post>`, `get_text`. Граница между модулями скрывает всю асинхронную цепочку.

---

## Что дальше

**16.2. HTTP-методы и тело запроса.** Мы расширим `fetch.rs` функциями для POST, PUT, PATCH, DELETE. Разберём разные типы тела: JSON, form data, `URLSearchParams`. В `app.rs` появится секция «HTTP-методы» с четырьмя кнопками. В тестах — 8 новых проверок на разные методы и типы тела.