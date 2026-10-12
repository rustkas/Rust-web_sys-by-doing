# 16.2. HTTP-методы и тело запроса

## Обновлённая структура проекта

```text
hello-wasm02/
├── Cargo.toml                      # +feature FormData
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
│   ├── lib.rs
│   ├── app.rs                      # +1 секция: «HTTP-методы» + build_section
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── fetch.rs                    # +12 функций: POST, PUT, PATCH, DELETE
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
│   └── wasm_fetch.rs               # +8 тестов → 14 тестов
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

- `src/fetch.rs` — **расширяется 12 функциями**. Добавляются: `make_request_with_method` (общая), `make_post_json_request`, `make_post_form_urlencoded_request`, `make_post_form_data_request`, `make_put_request`, `make_patch_request`, `make_delete_request` и высокоуровневые `post_json`, `post_json_response`, `put_json`, `patch_json`, `delete_request`.
- `src/app.rs` — **+1 секция «HTTP-методы»**. Четыре кнопки: POST JSON, POST FormData, PUT, DELETE. Плюс блок результата. Также добавляется вспомогательная функция `build_section` в конце файла.
- `tests/wasm_fetch.rs` — **+8 тестов**. Всего 14 тестов.
- `Cargo.toml` — **+feature `FormData`**. `UrlSearchParams` уже подключён в главе 14.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| HTTP-методы (MDN) | https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods |
| `RequestInit::body` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/RequestInit#body |
| `FormData` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/FormData |
| `URLSearchParams` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams |
| `Content-Type` (MDN) | https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type |
| `web_sys::FormData` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.FormData.html |
| `web_sys::UrlSearchParams` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.UrlSearchParams.html |

---

## HTTP-методы

**HTTP-метод — это глагол, который сообщает серверу, что мы хотим сделать с ресурсом.**

| Метод | Назначение | Есть тело? | Идемпотентный? |
|---|---|---|---|
| **GET** | Прочитать ресурс | Нет | Да |
| **POST** | Создать новый ресурс | Да | Нет |
| **PUT** | Заменить ресурс целиком | Да | Да |
| **PATCH** | Обновить часть ресурса | Да | Нет |
| **DELETE** | Удалить ресурс | Обычно нет | Да |
| **HEAD** | Прочитать только заголовки | Нет | Да |
| **OPTIONS** | Узнать, что разрешено | Нет | Да |

**Идемпотентность** — важное свойство. Идемпотентный метод можно вызывать несколько раз без изменения результата. GET, PUT, DELETE — идемпотентны. POST и PATCH — нет.

**Практическое следствие.** Retry (подглава 16.6) безопасен только для идемпотентных методов. Если POST упал из-за сетевого сбоя — неизвестно, создался ресурс на сервере или нет. Повторный POST может создать дубликат.

**Разница PUT и PATCH.** PUT заменяет весь ресурс: если вы отправляете `{"title": "New"}`, все остальные поля сбросятся. PATCH обновляет только указанные поля.

---

## Тело запроса

**Тело запроса (body) — это данные, которые отправляются на сервер.** GET, HEAD, OPTIONS не имеют тела. POST, PUT, PATCH, DELETE могут.

**Способы сформировать тело:**

| Тип | Класс | Content-Type |
|---|---|---|
| JSON | `String` (через `serde_json::to_string`) | `application/json` |
| Form URL-encoded | `URLSearchParams` | `application/x-www-form-urlencoded` |
| Multipart form | `FormData` | `multipart/form-data; boundary=...` (автоматически) |
| Plain text | `String` | `text/plain` |
| Binary | `Blob`, `ArrayBuffer`, `Uint8Array` | зависит от данных |
| Stream | `ReadableStream` | зависит от данных (глава 17) |

**Content-Type — критически важен.** Сервер использует его, чтобы понять, как парсить тело. Если вы отправляете JSON, но не указываете `Content-Type: application/json`, сервер может попытаться распарсить тело как form data и вернуть ошибку.

**Особенность `FormData`.** Для `FormData` **не устанавливайте `Content-Type` вручную**. Браузер сам добавит правильный заголовок с параметром `boundary`, который нужен для разделения полей.

---

## `Cargo.toml`

Показаны **только новые части**. Остальное — из 16.1.

```toml
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

    # Тело запроса
    "FormData",           # ← добавлено: для multipart/form-data

    # Прочее
    "console",
]
```

**Что добавилось:**

- **`FormData`** — для создания `multipart/form-data` тела. Позволяет добавлять поля и файлы.
- **`UrlSearchParams`** — уже подключён в главе 14 для работы с query-параметрами. Используем его же для `application/x-www-form-urlencoded`.

---

## `src/fetch.rs`

Показаны **только новые части**. Существующие функции из 16.1 остаются.

Добавьте в раздел `use`:

```rust
use serde::Serialize;
use web_sys::{FormData, UrlSearchParams};
```

Добавьте в конец файла:

```rust
// ============================================================
// Создание запросов: общие методы
// ============================================================

/// Создаёт запрос с указанным методом.
pub fn make_request_with_method(
    url: &str,
    method: &str,
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let opts = RequestInit::new();
    opts.set_method(method);
    opts.set_mode(mode);

    return Request::new_with_str_and_init(url, &opts);
}

// ============================================================
// POST: с JSON-телом
// ============================================================

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

// ============================================================
// POST: с form-urlencoded телом
// ============================================================

/// Создаёт POST-запрос с телом `application/x-www-form-urlencoded`.
pub fn make_post_form_urlencoded_request(
    url: &str,
    params: &[(&str, &str)],
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let form_params = UrlSearchParams::new()?;
    for (name, value) in params {
        form_params.append(name, value);
    }

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_mode(mode);
    opts.set_body(form_params.as_ref());

    let request = Request::new_with_str_and_init(url, &opts)?;
    request.headers().set(
        "Content-Type",
        "application/x-www-form-urlencoded",
    )?;

    return Ok(request);
}

// ============================================================
// POST: с multipart/form-data телом
// ============================================================

/// Создаёт POST-запрос с телом `multipart/form-data`.
///
/// **Важно:** `Content-Type` устанавливать не нужно — браузер
/// добавит его автоматически с правильным `boundary`.
pub fn make_post_form_data_request(
    url: &str,
    fields: &[(&str, &str)],
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let form_data = FormData::new()?;
    for (name, value) in fields {
        form_data.append_with_str(name, value)?;
    }

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_mode(mode);
    opts.set_body(form_data.as_ref());

    return Request::new_with_str_and_init(url, &opts);
}

// ============================================================
// PUT, PATCH, DELETE
// ============================================================

/// Создаёт PUT-запрос (без тела).
pub fn make_put_request(url: &str, mode: RequestMode) -> Result<Request, JsValue> {
    return make_request_with_method(url, "PUT", mode);
}

/// Создаёт PATCH-запрос (без тела).
pub fn make_patch_request(url: &str, mode: RequestMode) -> Result<Request, JsValue> {
    return make_request_with_method(url, "PATCH", mode);
}

/// Создаёт DELETE-запрос.
pub fn make_delete_request(url: &str, mode: RequestMode) -> Result<Request, JsValue> {
    return make_request_with_method(url, "DELETE", mode);
}

/// Создаёт PUT-запрос с JSON-телом.
pub fn make_put_json_request<T: Serialize>(
    url: &str,
    body: &T,
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let json = serde_json::to_string(body)
        .map_err(|e| JsValue::from_str(&format!("serialize error: {}", e)))?;

    let opts = RequestInit::new();
    opts.set_method("PUT");
    opts.set_mode(mode);
    opts.set_body(&JsValue::from_str(&json));

    let request = Request::new_with_str_and_init(url, &opts)?;
    request
        .headers()
        .set("Content-Type", "application/json")?;

    return Ok(request);
}

/// Создаёт PATCH-запрос с JSON-телом.
pub fn make_patch_json_request<T: Serialize>(
    url: &str,
    body: &T,
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let json = serde_json::to_string(body)
        .map_err(|e| JsValue::from_str(&format!("serialize error: {}", e)))?;

    let opts = RequestInit::new();
    opts.set_method("PATCH");
    opts.set_mode(mode);
    opts.set_body(&JsValue::from_str(&json));

    let request = Request::new_with_str_and_init(url, &opts)?;
    request
        .headers()
        .set("Content-Type", "application/json")?;

    return Ok(request);
}

// ============================================================
// Высокоуровневые функции для POST / PUT / PATCH / DELETE
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

/// POST-запрос с `application/x-www-form-urlencoded`, возвращает `Response`.
pub async fn post_form_urlencoded(
    url: &str,
    params: &[(&str, &str)],
) -> Result<Response, JsValue> {
    let request =
        make_post_form_urlencoded_request(url, params, RequestMode::Cors)?;
    return send_request(&request).await;
}

/// POST-запрос с `multipart/form-data`, возвращает `Response`.
pub async fn post_form_data(
    url: &str,
    fields: &[(&str, &str)],
) -> Result<Response, JsValue> {
    let request =
        make_post_form_data_request(url, fields, RequestMode::Cors)?;
    return send_request(&request).await;
}

/// PUT-запрос с JSON-телом, возвращает `Response`.
pub async fn put_json<T: Serialize>(
    url: &str,
    body: &T,
) -> Result<Response, JsValue> {
    let request = make_put_json_request(url, body, RequestMode::Cors)?;
    return send_request(&request).await;
}

/// PATCH-запрос с JSON-телом, возвращает `Response`.
pub async fn patch_json<T: Serialize>(
    url: &str,
    body: &T,
) -> Result<Response, JsValue> {
    let request = make_patch_json_request(url, body, RequestMode::Cors)?;
    return send_request(&request).await;
}

/// DELETE-запрос, возвращает `Response`.
pub async fn delete_request(url: &str) -> Result<Response, JsValue> {
    let request = make_delete_request(url, RequestMode::Cors)?;
    return send_request(&request).await;
}
```

**Ключевые моменты:**

- **`make_request_with_method`** — общая функция для методов без тела.
- **`make_post_json_request`** — сериализует `T` в JSON, устанавливает `Content-Type: application/json`.
- **`make_post_form_urlencoded_request`** — создаёт `UrlSearchParams`, устанавливает `Content-Type: application/x-www-form-urlencoded`.
- **`make_post_form_data_request`** — создаёт `FormData`. **Не устанавливает Content-Type** — браузер сделает сам.
- **`post_json`, `put_json`, `patch_json`, `delete_request`** — высокоуровневые обёртки.

---

## `src/app.rs`

Показано **полное содержание** с учётом исправлений.

```rust
//! Логика приложения: «Rust API Client» — часть 2.
//!
//! Две секции:
//! 1. **Первый GET** — базовый GET-запрос.
//! 2. **HTTP-методы** — POST JSON, POST FormData, PUT, DELETE.

use std::cell::RefCell;
use std::rc::Rc;

use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::fetch;
use crate::log;

// ============================================================
// Модель данных
// ============================================================

/// Пост из JSONPlaceholder.
///
/// `Serialize` нужен для отправки поста на сервер (POST, PUT, PATCH).
/// `Deserialize` — для чтения ответа.
#[derive(Serialize, Deserialize, Debug)]
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
const POSTS_CREATE_URL: &str = "https://jsonplaceholder.typicode.com/posts";
const PUT_URL: &str = "https://jsonplaceholder.typicode.com/posts/1";
const DELETE_URL: &str = "https://jsonplaceholder.typicode.com/posts/1";

// ============================================================
// Точка входа
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Rust API Client")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Демонстрация Fetch API: GET, POST, PUT, DELETE. \
         Запросы идут к JSONPlaceholder — публичному тестовому API.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 1: Первый GET
    // ============================================================

    let section1 = build_section(&container, "1. Первый GET")?;

    let intro1 = dom::create_paragraph(
        "Нажмите кнопку — приложение отправит GET-запрос к JSONPlaceholder \
         и покажет полученный пост.",
    )?;
    dom::add_class(&dom::as_html(&intro1), "hint")?;
    dom::append(&section1, &intro1)?;

    let load_btn = dom::create_element("button")?;
    load_btn.set_text_content(Some("Загрузить пост"));
    dom::append(&section1, &load_btn)?;

    let result1 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result1), "api-result")?;
    result1.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section1, &result1)?;

    // ---- Обработчик «Загрузить пост» ----
    {
        let result1 = result1.clone();
        callbacks::listen(
            &load_btn,
            "click",
            move |_event| {
                let result1 = result1.clone();
                result1.set_text_content(Some("Загрузка..."));

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
                            result1.set_text_content(Some(text.as_str()));
                            log!("GET /posts/1: success");
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            result1.set_text_content(Some(text.as_str()));
                            log!("GET /posts/1 error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 2: HTTP-методы
    // ============================================================

    let section2 = build_section(&container, "2. HTTP-методы")?;

    let intro2 = dom::create_paragraph(
        "Разные HTTP-методы с разными телами. \
         POST создаёт ресурс, PUT заменяет, DELETE удаляет.",
    )?;
    dom::add_class(&dom::as_html(&intro2), "hint")?;
    dom::append(&section2, &intro2)?;

    let post_json_btn = dom::create_element("button")?;
    post_json_btn.set_text_content(Some("POST JSON"));
    let post_form_btn = dom::create_element("button")?;
    post_form_btn.set_text_content(Some("POST FormData"));
    let put_btn = dom::create_element("button")?;
    put_btn.set_text_content(Some("PUT JSON"));
    let delete_btn = dom::create_element("button")?;
    delete_btn.set_text_content(Some("DELETE"));

    dom::append(&section2, &post_json_btn)?;
    dom::append(&section2, &post_form_btn)?;
    dom::append(&section2, &put_btn)?;
    dom::append(&section2, &delete_btn)?;

    let result2 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result2), "api-result")?;
    result2.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section2, &result2)?;

    // ---- Обработчик «POST JSON» ----
    {
        let result2 = result2.clone();
        callbacks::listen(
            &post_json_btn,
            "click",
            move |_event| {
                let result2 = result2.clone();
                result2.set_text_content(Some("Отправка POST JSON..."));

                wasm_bindgen_futures::spawn_local(async move {
                    let new_post = Post {
                        id: 0,
                        title: "Новый пост из Rust".into(),
                        body: "Тело, созданное в Rust + Wasm".into(),
                        user_id: 42,
                    };

                    match fetch::post_json_response::<Post, Post>(
                        POSTS_CREATE_URL,
                        &new_post,
                    )
                    .await
                    {
                        Ok(created) => {
                            let text: String = format!(
                                "✅ POST /posts (JSON)\n\
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
                            result2.set_text_content(Some(text.as_str()));
                            log!("POST JSON: success, id={}", created.id);
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            result2.set_text_content(Some(text.as_str()));
                            log!("POST JSON error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «POST FormData» ----
    {
        let result2 = result2.clone();
        callbacks::listen(
            &post_form_btn,
            "click",
            move |_event| {
                let result2 = result2.clone();
                result2.set_text_content(Some("Отправка POST FormData..."));

                wasm_bindgen_futures::spawn_local(async move {
                    let fields = [
                        ("title", "Пост через FormData"),
                        ("body", "Тело, отправленное как multipart/form-data"),
                        ("userId", "42"),
                    ];

                    match fetch::post_form_data(POSTS_CREATE_URL, &fields).await {
                        Ok(resp) => {
                            let status = fetch::response_status(&resp);
                            let text: String = format!(
                                "✅ POST /posts (FormData)\n\
                                 \n\
                                 Статус: {}\n\
                                 Content-Type запроса был установлен \
                                 автоматически с boundary.",
                                status
                            );
                            result2.set_text_content(Some(text.as_str()));
                            log!("POST FormData: status={}", status);
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            result2.set_text_content(Some(text.as_str()));
                            log!("POST FormData error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «PUT JSON» ----
    {
        let result2 = result2.clone();
        callbacks::listen(
            &put_btn,
            "click",
            move |_event| {
                let result2 = result2.clone();
                result2.set_text_content(Some("Отправка PUT JSON..."));

                wasm_bindgen_futures::spawn_local(async move {
                    let updated = Post {
                        id: 1,
                        title: "Обновлённый заголовок".into(),
                        body: "Обновлённое тело".into(),
                        user_id: 1,
                    };

                    match fetch::put_json(PUT_URL, &updated).await {
                        Ok(resp) => {
                            let status = fetch::response_status(&resp);
                            let text: String = format!(
                                "✅ PUT /posts/1 (JSON)\n\
                                 \n\
                                 Статус: {}\n\
                                 PUT заменяет ресурс целиком — \
                                 все поля переданы.",
                                status
                            );
                            result2.set_text_content(Some(text.as_str()));
                            log!("PUT JSON: status={}", status);
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            result2.set_text_content(Some(text.as_str()));
                            log!("PUT JSON error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «DELETE» ----
    {
        let result2 = result2.clone();
        callbacks::listen(
            &delete_btn,
            "click",
            move |_event| {
                let result2 = result2.clone();
                result2.set_text_content(Some("Отправка DELETE..."));

                wasm_bindgen_futures::spawn_local(async move {
                    match fetch::delete_request(DELETE_URL).await {
                        Ok(resp) => {
                            let status = fetch::response_status(&resp);
                            let text: String = format!(
                                "✅ DELETE /posts/1\n\
                                 \n\
                                 Статус: {}\n\
                                 DELETE идемпотентен — \
                                 повторный вызов не изменит результат.",
                                status
                            );
                            result2.set_text_content(Some(text.as_str()));
                            log!("DELETE: status={}", status);
                        }
                        Err(e) => {
                            let text: String = format!("❌ Ошибка: {:?}", e);
                            result2.set_text_content(Some(text.as_str()));
                            log!("DELETE error: {:?}", e);
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    log!("Rust API Client — часть 2 готова");

    std::mem::forget(closures);

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
```

**Ключевые моменты:**

- **`use serde::{Deserialize, Serialize};`** — оба трейта в одной строке. `Serialize` нужен для `#[derive(Serialize)]`.
- **`#[derive(Serialize, Deserialize, Debug)]`** — оба трейта. Без `Serialize` функции `post_json_response` и `put_json` не скомпилируются.
- **`build_section` в конце файла** — вспомогательная функция уровня модуля, используется обеими секциями.
- **Четыре кнопки в секции 2** — POST JSON, POST FormData, PUT JSON, DELETE.
- **`spawn_local` в каждом обработчике** — потому что Fetch асинхронный, а обработчик клика синхронный.

---

## `web/styles/main.css`

Изменений **нет**. `.api-result` из 16.1 работает и для секции 2.

---

## `tests/wasm_fetch.rs`

Добавьте в конец файла:

```rust
// ============================================================
// HTTP-методы: создание запросов
// ============================================================

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
    assert!(request.is_ok(), "POST JSON request should be created");
}

#[wasm_bindgen_test]
fn test_make_post_form_urlencoded_request() {
    let request = fetch::make_post_form_urlencoded_request(
        "https://jsonplaceholder.typicode.com/posts",
        &[("title", "test"), ("body", "content")],
        web_sys::RequestMode::Cors,
    );
    assert!(
        request.is_ok(),
        "POST form-urlencoded request should be created"
    );
}

#[wasm_bindgen_test]
fn test_make_post_form_data_request() {
    let request = fetch::make_post_form_data_request(
        "https://jsonplaceholder.typicode.com/posts",
        &[("title", "test"), ("body", "content")],
        web_sys::RequestMode::Cors,
    );
    assert!(request.is_ok(), "POST FormData request should be created");
}

#[wasm_bindgen_test]
fn test_make_put_request() {
    let request = fetch::make_put_request(
        "https://jsonplaceholder.typicode.com/posts/1",
        web_sys::RequestMode::Cors,
    );
    assert!(request.is_ok(), "PUT request should be created");
}

#[wasm_bindgen_test]
fn test_make_delete_request() {
    let request = fetch::make_delete_request(
        "https://jsonplaceholder.typicode.com/posts/1",
        web_sys::RequestMode::Cors,
    );
    assert!(request.is_ok(), "DELETE request should be created");
}

// ============================================================
// HTTP-методы: реальные запросы
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
    assert_eq!(created.body, "Body");
}

#[wasm_bindgen_test]
async fn test_put_json() {
    use serde::Serialize;

    #[derive(Serialize)]
    struct Post {
        id: u32,
        title: String,
        body: String,
        user_id: u32,
    }

    let updated = Post {
        id: 1,
        title: "Updated".into(),
        body: "New body".into(),
        user_id: 1,
    };

    let result = fetch::put_json(
        "https://jsonplaceholder.typicode.com/posts/1",
        &updated,
    )
    .await;

    assert!(result.is_ok(), "PUT should succeed: {:?}", result);
    let resp = result.unwrap();
    assert!(fetch::response_is_ok(&resp));
}

#[wasm_bindgen_test]
async fn test_delete() {
    let result =
        fetch::delete_request("https://jsonplaceholder.typicode.com/posts/1")
            .await;

    assert!(result.is_ok(), "DELETE should succeed: {:?}", result);
    let resp = result.unwrap();
    assert!(fetch::response_is_ok(&resp));
}

// ============================================================
// Content-Type заголовки
// ============================================================

#[wasm_bindgen_test]
fn test_content_type_is_set_for_json() {
    use serde::Serialize;

    #[derive(Serialize)]
    struct Data {
        value: u32,
    }

    let data = Data { value: 42 };

    let request = fetch::make_post_json_request(
        "https://jsonplaceholder.typicode.com/posts",
        &data,
        web_sys::RequestMode::Cors,
    )
    .expect("request should be created");

    let content_type = request
        .headers()
        .get("Content-Type")
        .expect("header read should not fail")
        .expect("Content-Type should be set");

    assert!(
        content_type.contains("application/json"),
        "Content-Type should be application/json, got: {}",
        content_type
    );
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_make_post_json_request` | Создание POST JSON-запроса |
| `test_make_post_form_urlencoded_request` | Создание POST form-urlencoded |
| `test_make_post_form_data_request` | Создание POST FormData |
| `test_make_put_request` | Создание PUT-запроса |
| `test_make_delete_request` | Создание DELETE-запроса |
| `test_post_json` | Реальный POST JSON-запрос |
| `test_put_json` | Реальный PUT JSON-запрос |
| `test_delete` | Реальный DELETE-запрос |
| `test_content_type_is_set_for_json` | Проверка `Content-Type` заголовка |

**Всего тестов:** 6 из 16.1 + 8 новых + 1 дополнительный = **15 тестов**.

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

Откройте `http://127.0.0.1:5173`. Прокрутите до секции 2 «HTTP-методы».

**POST JSON:**
1. Нажмите «POST JSON». Сервер вернёт созданный пост с `id: 101`.

**POST FormData:**
1. Нажмите «POST FormData». Показывается статус 201.

**PUT JSON:**
1. Нажмите «PUT JSON». Статус 200.

**DELETE:**
1. Нажмите «DELETE». Статус 200.

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

**Ожидаемый результат:** 15 тестов `ok` в обоих браузерах.

---

## Что важно запомнить

**1. HTTP-метод определяет семантику.** GET читает, POST создаёт, PUT заменяет, PATCH обновляет часть, DELETE удаляет.

**2. Идемпотентность важна для retry.** GET, PUT, DELETE можно повторять безопасно. POST и PATCH — нет.

**3. `Content-Type` — обязателен для тела.** Для JSON — `application/json`, для form-urlencoded — `application/x-www-form-urlencoded`.

**4. `FormData` сам устанавливает `Content-Type`.** С параметром `boundary`. Не перезаписывайте его вручную.

**5. `URLSearchParams` для form-urlencoded.** Стандартный способ кодирования пар ключ-значение.

**6. `Serialize` нужен для тела запроса.** Когда вы отправляете данные на сервер, структура должна реализовать `Serialize`. `Deserialize` — для чтения ответа.

**7. `#[derive(Serialize, Deserialize)]` — для двустороннего обмена.** Если структура используется и для отправки, и для чтения — нужны оба трейта.

**8. Вспомогательные функции — вне `run`.** `build_section` не зависит от состояния приложения. Её место — на уровне модуля.

**9. `RequestInit::set_body` принимает `Option<&JsValue>`.** Для JSON — `JsValue::from_str(&json)`. Для `FormData` и `URLSearchParams` — `as_ref()`.

**10. Разделение создания и отправки.** В `fetch.rs` есть функции `make_*_request` (создают `Request`) и `*_json`/`delete_request` (создают и отправляют). Первые тестируются без сети, вторые — с реальным API.

---

## Что дальше

**16.3. Заголовки: чтение, запись, case-insensitivity.** Мы разберём `Headers` API: как читать заголовки ответа, как устанавливать произвольные заголовки запроса, почему `Content-Type` и `content-type` — одно и то же. В `fetch.rs` появятся функции `make_headers`, `get_response_header`, `has_response_header`. В `app.rs` — секция «Заголовки» с чтением `Content-Type`, `Cache-Control` и отправкой кастомного `X-Custom-Header`. В тестах — 6 новых проверок на case-insensitivity и чтение.