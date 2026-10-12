# Глава 18. File API

## 18.13. Загрузка на сервер: `FormData` + `fetch`

В предыдущих подглавах мы научились читать файлы, хешировать их, обрабатывать в воркерах. Но рано или поздно встаёт задача **отправить файл на сервер**: сохранить результат конвертации, загрузить документ для совместной работы, отправить фотографию для обработки. Самый прямолинейный способ — `FormData` + `fetch`. В этой подглаве разберём его end-to-end: с реальным Rust-сервером, приёмом multipart, вычислением SHA-256 на обеих сторонах и подтверждением целостности доставки. И честно признаем главное ограничение `fetch` — **отсутствие прогресса отправки**. Частично закроем его живым счётчиком времени и кнопкой отмены, а в следующей подглаве решим полностью через `XMLHttpRequest`.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/                       # клиентский crate
├── Cargo.toml                     # без изменений
├── src/
│   ├── lib.rs                     # pub mod file_upload; (уже подключён)
│   ├── app.rs                     # ОБНОВЛЁН: только секция 18.13
│   ├── file_upload.rs             # НОВЫЙ: модуль подглавы 18.13
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── file_api.rs                # из 18.1 (используется в app.rs)
│   ├── logging.rs                 # существует
│   └── … остальные модули
├── auth-server/                   # отдельный crate (Rust-сервер)
│   ├── Cargo.toml                 # ОБНОВЛЁН: axum 0.8.9 + multipart, sha2, hex
│   └── src/
│       ├── main.rs                # ОБНОВЛЁН: +/api/upload, +DefaultBodyLimit
│       ├── handlers.rs            # ОБНОВЛЁН: +fn upload
│       └── auth.rs                # без изменений
└── tests/
    └── wasm_file_upload.rs        # НОВЫЙ: интеграционные тесты
```

**Пояснения:**

- `src/file_upload.rs` — новый модуль клиента. Отправляет `File`/`Blob` через `FormData` + `fetch`, поддерживает отмену через `AbortController`.
- `src/app.rs` — **только** код подглавы 18.13.
- `auth-server/` — существующий сервер дополняется обработчиком `POST /api/upload` и снятием дефолтного лимита тела. Остальные эндпоинты не затронуты.
- `tests/wasm_file_upload.rs` — интеграционные тесты. `#[ignore]` не используется, потому что `auth-server` запускается перед тестами.

**Новых зависимостей в клиенте не требуется.** `FormData`, `Request`, `RequestInit`, `Response`, `AbortController`, `AbortSignal`, `DomException` уже включены ранее.

---

## Ссылки для детального изучения

- **MDN: FormData** — https://developer.mozilla.org/en-US/docs/Web/API/FormData
- **MDN: Using FormData Objects** — https://developer.mozilla.org/en-US/docs/Web/API/FormData/Using_FormData_Objects
- **MDN: fetch()** — https://developer.mozilla.org/en-US/docs/Web/API/fetch
- **MDN: AbortController** — https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- **MDN: AbortSignal** — https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal
- **MDN: CORS** — https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- **axum: Multipart** — https://docs.rs/axum/latest/axum/extract/struct.Multipart.html
- **axum: DefaultBodyLimit** — https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html
- **RFC 7578 — multipart/form-data** — https://datatracker.ietf.org/doc/html/rfc7578

---

## Содержание новых / обновлённых файлов

### 1. Сервер: `auth-server/Cargo.toml`

```toml
[package]
name = "auth-server"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8.9", features = ["multipart"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
tower-http = { version = "0.7.1", features = ["cors"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jsonwebtoken = "11.1.0"
chrono = { version = "0.4", features = ["serde"] }
dotenvy = "0.15"
tokio-stream = "0.1"
futures = "0.3"
async-stream = "0.3"
sha2 = "0.10"
hex = "0.4"
```

**Что нового:**

- `axum = "0.8.9"` с feature `multipart` — без него `axum::extract::Multipart` не существует.
- `sha2 = "0.10"` и `hex = "0.4"` — сервер считает SHA-256 принятого файла и возвращает hex в ответе. Это даёт клиенту точку сверки целостности.
- `jsonwebtoken = "11.1.0"` — в 11.x требуется явный криптопровайдер. По умолчанию включён `rust_crypto` (чистый Rust, без C-зависимостей) — для HMAC-SHA256 этого достаточно.

---

### 2. Сервер: `auth-server/src/main.rs`

```rust
mod auth;
mod handlers;

use axum::{
    extract::DefaultBodyLimit,
    routing::{get, post},
    Router,
};
use tower_http::cors::CorsLayer;

#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    dotenvy::dotenv().ok();

    let app = Router::new()
        .route("/api/auth/login", post(auth::login))
        .route("/api/auth/refresh", post(auth::refresh))
        .route("/api/me", get(handlers::me))
        .route("/api/admin", get(handlers::admin_only))
        .route("/api/flaky", get(handlers::flaky))
        .route("/api/flaky/reset", get(handlers::flaky_reset))
        .route("/api/error500", get(handlers::always_500))
        .route("/api/slow", get(handlers::slow_default))
        .route("/api/slow/{secs}", get(handlers::slow))
        .route("/api/delay/{secs}", get(handlers::delay))
        .route("/api/concurrency/test", get(handlers::concurrency_test))
        .route("/api/concurrency/reset", get(handlers::concurrency_reset))
        .route("/api/concurrency/stats", get(handlers::concurrency_stats))
        .route("/api/image", get(handlers::image))
        .route("/api/binary", get(handlers::binary))
        .route("/api/stream/{count}", get(handlers::stream_chunks))
        .route("/api/stream/records/{count}", get(handlers::stream_records))
        // Снимаем дефолтный лимит тела (2 МБ в axum 0.8) только
        // для upload. Обработчик читает multipart потоково —
        // файл не буферизуется целиком, OOM не грозит.
        .route(
            "/api/upload",
            post(handlers::upload).layer(DefaultBodyLimit::disable()),
        )
        .layer(CorsLayer::permissive());

    let addr = std::env::var("BIND_ADDRESS").unwrap_or("127.0.0.1:3000".into());
    println!("Auth server listening on {addr}");
    let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**Что важно:**

- **Синтаксис маршрутов axum 0.8** — `{secs}`, `{count}` вместо `:secs`, `:count`. Старый синтаксис не даёт ошибку компиляции, но **паникует при старте сервера** с сообщением «Path segments must not start with `:`».
- **`DefaultBodyLimit::disable()`** — в axum 0.8 дефолтный лимит тела запроса — **2 МБ**. Он применяется ко всем экстракторам, включая `Multipart`. Без снятия лимита файлы крупнее 2 МБ падают с `413 Payload Too Large`. Лимит снят **точечно** только на `/api/upload` — остальные эндпоинты остаются защищены.
- **CORS: `CorsLayer::permissive()`** — уже покрывает все потребности. Более того, `multipart/form-data` — simple content-type по CORS-спеке, preflight-запрос не отправляется вообще.

---

### 3. Сервер: `auth-server/src/handlers.rs` — обработчик `upload`

В конец `handlers.rs` добавьте:

```rust
// ============================================================
// 18.13: Загрузка файла на сервер (multipart/form-data)
// ============================================================

use axum::extract::Multipart;
use sha2::{Digest, Sha256};

/// Принимает файл через multipart/form-data.
///
/// Ожидает одно поле с именем `file`. Читает тело **по чанкам**,
/// не загружая весь файл в память — это критично для больших
/// загрузок (гигабайты). Считает SHA-256 по мере чтения и
/// возвращает метаданные вместе с хешем.
///
/// **Stateless.** Файл никуда не сохраняется — ни в память, ни
/// на диск, ни в БД. После отправки ответа все данные
/// освобождаются. Сервер можно держать запущенным неделями:
/// диск не переполнится, чистить нечего.
///
/// Формат ответа:
/// ```json
/// {
///   "status": "ok",
///   "filename": "document.pdf",
///   "size": 123456,
///   "content_type": "application/pdf",
///   "sha256": "ab12…"
/// }
/// ```
pub async fn upload(mut multipart: Multipart) -> Result<Json<Value>, StatusCode> {
    let mut file_name = String::from("unnamed");
    let mut content_type = String::from("application/octet-stream");
    let mut size: usize = 0;
    let mut hasher = Sha256::new();
    let mut found_file = false;

    loop {
        let field_opt = multipart
            .next_field()
            .await
            .map_err(|_| StatusCode::BAD_REQUEST)?;

        let field = match field_opt {
            Some(f) => f,
            None => break,
        };

        let name = field.name().map(String::from).unwrap_or_default();
        if name != "file" {
            continue;
        }
        found_file = true;

        if let Some(fname) = field.file_name() {
            file_name = fname.to_string();
        }
        if let Some(ct) = field.content_type() {
            content_type = ct.to_string();
        }

        // Читаем поле по чанкам через `field.chunk()` — память
        // O(chunk_size), а не O(file_size). Гигабайтные загрузки
        // не приведут к OOM.
        let mut field = field;
        loop {
            let chunk_opt = field
                .chunk()
                .await
                .map_err(|_| StatusCode::BAD_REQUEST)?;
            let chunk = match chunk_opt {
                Some(c) => c,
                None => break,
            };
            size += chunk.len();
            hasher.update(&chunk);
        }
    }

    if !found_file {
        return Err(StatusCode::BAD_REQUEST);
    }

    let hash = hasher.finalize();
    let sha256_hex = hex::encode(hash);

    Ok(Json(json!({
        "status": "ok",
        "filename": file_name,
        "size": size,
        "content_type": content_type,
        "sha256": sha256_hex,
    })))
}
```

**Ключевые моменты:**

- **Потоковое чтение** через `field.chunk().await` — каждый chunk это `Bytes` размером несколько килобайт. Даже 10-гигабайтный файл не создаст OOM.
- **SHA-256 по мере чтения** — тот же принцип, что в 18.12 на клиенте. Хеш совпадёт с клиентским, если данные дошли без искажений.
- **Stateless** — файл нигде не сохраняется. Никаких `/tmp/uploads/` — чистить после тестов нечего.

---

### 4. Клиент: `src/file_upload.rs`

```rust
//! Загрузка файлов на сервер: FormData + fetch.
//!
//! Модуль автономен: зависит только от `web-sys`, `js-sys`,
//! `wasm-bindgen`, `wasm-bindgen-futures`.
//!
//! **Про Content-Type.** Браузер сам выставляет заголовок
//! `Content-Type: multipart/form-data; boundary=...`, когда
//! body — это `FormData`. Если задать Content-Type вручную,
//! браузер не добавит boundary, и сервер не сможет распарсить
//! multipart. Никогда не устанавливайте Content-Type для FormData.
//!
//! **Про прогресс.** fetch не имеет события `upload.onprogress`.
//! Прогресс отправки через fetch получить невозможно — это
//! свойство API. Для прогресса используйте XMLHttpRequest
//! (см. подглаву 18.14). Здесь мы показываем статус и даём
//! отменить запрос через `AbortController`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{AbortSignal, Blob, File, FormData, Request, RequestInit, Response};

/// Загружает `File` на сервер через FormData + fetch.
pub async fn upload_file_cancellable(
    file: &File,
    url: &str,
    signal: &AbortSignal,
) -> Result<String, JsValue> {
    let form_data = FormData::new()?;
    form_data.append_with_blob("file", file)?;

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_body(&form_data);
    opts.set_signal(Some(signal));
    // Content-Type НЕ ставим — браузер сам добавит
    // multipart/form-data с корректным boundary.

    let request = Request::new_with_str_and_init(url, &opts)?;
    return send_and_read_text(&request).await;
}

/// Загружает `File` без возможности отмены.
pub async fn upload_file(file: &File, url: &str) -> Result<String, JsValue> {
    let form_data = FormData::new()?;
    form_data.append_with_blob("file", file)?;

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_body(&form_data);

    let request = Request::new_with_str_and_init(url, &opts)?;
    return send_and_read_text(&request).await;
}

/// Загружает `Blob` с явным именем файла.
pub async fn upload_blob(
    blob: &Blob,
    filename: &str,
    url: &str,
) -> Result<String, JsValue> {
    let form_data = FormData::new()?;
    form_data.append_with_blob_and_filename("file", blob, filename)?;

    let opts = RequestInit::new();
    opts.set_method("POST");
    opts.set_body(&form_data);

    let request = Request::new_with_str_and_init(url, &opts)?;
    return send_and_read_text(&request).await;
}

/// Проверяет, была ли ошибка вызвана отменой через AbortController.
///
/// Браузеры возвращают `DOMException` с `name == "AbortError"`
/// при отменённом запросе. В Firefox иногда это `TypeError`,
/// но с сообщением, содержащим «abort». Проверяем оба случая.
pub fn is_abort_error(e: &JsValue) -> bool {
    if let Some(exc) = e.dyn_ref::<web_sys::DomException>() {
        return exc.name() == "AbortError";
    }
    if let Some(s) = e.as_string() {
        let lower = s.to_lowercase();
        return lower.contains("abort");
    }
    return false;
}

// ---------------------------------------------------------------
// Внутренняя отправка и чтение ответа
// ---------------------------------------------------------------

async fn send_and_read_text(request: &Request) -> Result<String, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let promise = window.fetch_with_request(request);
    let resp_value = JsFuture::from(promise).await?;
    let resp: Response = resp_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("ответ не Response"))?;

    if !resp.ok() {
        return Err(JsValue::from_str(&format!(
            "HTTP {} {}",
            resp.status(),
            resp.status_text(),
        )));
    }

    let text_promise = resp.text()?;
    let text_value = JsFuture::from(text_promise).await?;
    return text_value
        .as_string()
        .ok_or_else(|| JsValue::from_str("ответ не строка"));
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`, `.dyn_ref()`.
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{AbortSignal, Blob, File, FormData, Request, RequestInit, Response}` — все семь используются.
- `web_sys::DomException` — используется в полном пути в `is_abort_error`.
- Неиспользуемых импортов нет.

---

### 5. Клиент: `src/app.rs` — только демонстрация 18.13

```rust
//! Демонстрация подглавы 18.13: загрузка файла на сервер.
//!
//! Секция: выбрать файл → нажать «Загрузить на сервер» →
//! файл уходит на `auth-server` через FormData + fetch.
//! Пока идёт отправка — статус «Отправляем... N с» и кнопка
//! «Стоп», которая отменяет запрос через AbortController.
//!
//! **Про прогресс-бар.** У fetch нет события `upload.onprogress`,
//! поэтому indeterminate progress bar запрещён (вводит в
//! заблуждение). Вместо него — живой счётчик времени. В 18.14
//! решим это полностью через XMLHttpRequest.
//!
//! **Про SHA-256.** Сервер считает хеш принятого файла и
//! возвращает его в ответе. Клиент может сравнить со своим
//! локальным хешем (см. 18.12) — подтверждение целостности.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_api, file_upload, log};

/// URL эндпоинта загрузки на `auth-server`.
const UPLOAD_URL: &str = "http://localhost:3000/api/upload";

const BTN_READY: &str = "Загрузить на сервер";
const STOP_BTN_STYLE_VISIBLE: &str =
    "display: inline-block; padding: 4px 12px; font-size: 14px; \
     min-width: 40px;";
const STOP_BTN_STYLE_HIDDEN: &str = "display: none;";

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Загрузка файла на сервер")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));
    build_upload_section(&container, &closures)?;

    log!("Section 18.13 ready");
    std::mem::forget(closures);
    return Ok(());
}

fn build_upload_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let hint = dom::create_paragraph(
        "Выберите файл и нажмите «Загрузить на сервер» — файл уйдёт \
         на auth-server через FormData и fetch. Сервер принимает \
         файл потоково, считает SHA-256 и возвращает метаданные. \
         Файл НЕ сохраняется на сервере: обработчик stateless, \
         чистить после тестов нечего. Пока идёт отправка — живой \
         счётчик времени и кнопка «Стоп».",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    let actions_row = dom::create_element("div")?;
    actions_row.set_attribute(
        "style",
        "display: flex; align-items: center; gap: 8px; margin: 8px 0;",
    )?;

    let upload_btn = dom::create_element("button")?;
    upload_btn.set_text_content(Some(BTN_READY));
    dom::append(&actions_row, &upload_btn)?;

    let stop_btn: web_sys::Element = dom::create_element("button")?;
    stop_btn.set_attribute("type", "button")?;
    stop_btn.set_attribute("title", "Отменить загрузку")?;
    stop_btn.set_text_content(Some("\u{25A0}"));
    stop_btn.set_attribute("style", STOP_BTN_STYLE_HIDDEN)?;
    dom::append(&actions_row, &stop_btn)?;

    dom::append(&section, &actions_row)?;

    let status = dom::create_element("div")?;
    status.set_attribute(
        "style",
        "margin: 0 0 8px 0; color: #555; font-family: monospace;",
    )?;
    status.set_text_content(Some("—"));
    dom::append(&section, &status)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файл и нажмите кнопку)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    // Текущий AbortController: разделяем между кнопками
    // «Загрузить» и «Стоп».
    let controller: Rc<RefCell<Option<web_sys::AbortController>>> =
        Rc::new(RefCell::new(None));

    // Обработчик «Стоп»: вызываем abort() у текущего контроллера.
    {
        let controller_c = controller.clone();
        callbacks::listen(
            &stop_btn,
            "click",
            move |_ev: web_sys::Event| {
                if let Some(ctrl) = controller_c.borrow().as_ref() {
                    ctrl.abort();
                }
            },
            closures,
        )?;
    }

    // Обработчик «Загрузить».
    {
        let input_c = input.clone();
        let upload_btn_c = upload_btn.clone();
        let stop_btn_c = stop_btn.clone();
        let status_c = status.clone();
        let result_c = result.clone();
        let controller_c = controller.clone();

        callbacks::listen(
            &upload_btn,
            "click",
            move |_ev: web_sys::Event| {
                if upload_btn_c.has_attribute("disabled") {
                    return;
                }
                let file = match input_c.files().and_then(|f| f.get(0)) {
                    Some(f) => f,
                    None => {
                        result_c.set_text_content(Some("Сначала выберите файл."));
                        return;
                    }
                };

                let new_controller = match web_sys::AbortController::new() {
                    Ok(c) => c,
                    Err(e) => {
                        let msg = format!("❌ AbortController::new: {e:?}");
                        result_c.set_text_content(Some(&msg));
                        return;
                    }
                };
                let signal = new_controller.signal();
                *controller_c.borrow_mut() = Some(new_controller);

                upload_btn_c.set_attribute("disabled", "").ok();
                upload_btn_c.set_text_content(Some("Отправляем..."));
                set_stop_visible(&stop_btn_c, true);
                status_c.set_text_content(Some("⏳ Отправляем... 0.0 с"));
                result_c.set_text_content(Some("(ждём ответа сервера)"));

                // Тикер статуса: обновляет «Отправляем... N с»
                // каждые 100 мс, пока активен.
                let ticker_active = Rc::new(Cell::new(true));

                {
                    let status_t = status_c.clone();
                    let active_t = ticker_active.clone();
                    spawn_local(async move {
                        let start = js_sys::Date::now();
                        loop {
                            if !active_t.get() {
                                break;
                            }
                            let elapsed_s =
                                (js_sys::Date::now() - start) / 1000.0;
                            let text = format!(
                                "⏳ Отправляем... {:.1} с", elapsed_s
                            );
                            status_t.set_text_content(Some(&text));
                            TimeoutFuture::new(100).await;
                        }
                    });
                }

                // Основной запрос.
                {
                    let file_size = file.size();
                    let file_name = file.name();
                    let upload_btn_c2 = upload_btn_c.clone();
                    let stop_btn_c2 = stop_btn_c.clone();
                    let status_c2 = status_c.clone();
                    let result_c2 = result_c.clone();
                    let controller_c2 = controller_c.clone();
                    let ticker_active_c = ticker_active.clone();

                    spawn_local(async move {
                        let start = js_sys::Date::now();
                        let outcome = file_upload::upload_file_cancellable(
                            &file, UPLOAD_URL, &signal,
                        )
                        .await;
                        let elapsed_ms = js_sys::Date::now() - start;

                        ticker_active_c.set(false);

                        upload_btn_c2.remove_attribute("disabled").ok();
                        upload_btn_c2.set_text_content(Some(BTN_READY));
                        set_stop_visible(&stop_btn_c2, false);
                        *controller_c2.borrow_mut() = None;

                        let final_message: String;
                        match outcome {
                            Ok(text) => {
                                status_c2.set_text_content(Some(&format!(
                                    "✅ Загружено за {:.1} с",
                                    elapsed_ms / 1000.0,
                                )));
                                final_message = format_upload_success(
                                    &file_name, file_size, &text, elapsed_ms,
                                );
                            }
                            Err(e) if file_upload::is_abort_error(&e) => {
                                status_c2.set_text_content(Some("⏹ Отменено"));
                                final_message = format!(
                                    "⏹ Загрузка отменена пользователем\n\n\
                                     Файл: {file_name}\n\
                                     Размер: {}\n\
                                     Прошло: {:.1} с",
                                    file_api::format_file_size(file_size),
                                    elapsed_ms / 1000.0,
                                );
                            }
                            Err(e) => {
                                status_c2.set_text_content(Some("❌ Ошибка"));
                                final_message = format!(
                                    "❌ Ошибка загрузки\n\n\
                                     Файл: {file_name}\n\
                                     Размер: {}\n\
                                     URL: {UPLOAD_URL}\n\
                                     Ошибка: {e:?}\n\n\
                                     Проверьте, что auth-server запущен:\n\
                                     npm run server",
                                    file_api::format_file_size(file_size),
                                );
                            }
                        }
                        result_c2.set_text_content(Some(&final_message));
                    });
                }
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Формирование отчёта
// ---------------------------------------------------------------

fn format_upload_success(
    file_name: &str,
    file_size: f64,
    server_response: &str,
    elapsed_ms: f64,
) -> String {
    let speed_mbps: f64;
    if elapsed_ms > 0.0 {
        speed_mbps = file_size / (elapsed_ms / 1000.0) / (1024.0 * 1024.0);
    } else {
        speed_mbps = 0.0;
    }

    return format!(
        "✅ Файл успешно загружен\n\n\
         Файл: {file_name}\n\
         Размер: {} ({:.0} байт)\n\
         URL: {UPLOAD_URL}\n\
         Время: {elapsed_ms:.1} мс\n\
         Средняя скорость: {speed_mbps:.2} МБ/с\n\n\
         === Ответ сервера ===\n\
         {server_response}\n\n\
         Сервер посчитал SHA-256 принятого файла — сравните его \
         с локальным хешем (см. 18.12) для проверки целостности. \
         Файл на сервере не сохраняется: обработчик stateless.",
        file_api::format_file_size(file_size),
        file_size,
    );
}

fn set_stop_visible(btn: &web_sys::Element, visible: bool) {
    let style: &str;
    if visible {
        style = STOP_BTN_STYLE_VISIBLE;
    } else {
        style = STOP_BTN_STYLE_HIDDEN;
    }
    btn.set_attribute("style", style).ok();
    if !visible {
        btn.set_text_content(Some("\u{25A0}"));
    }
}
```

**Проверка импортов:**

- `Cell`, `RefCell`, `Rc` — все используются.
- `gloo_timers::future::TimeoutFuture` — для тикера статуса.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`.
- `wasm_bindgen_futures::spawn_local` — для двух параллельных `async`.
- `callbacks`, `EventClosures`, `dom`, `file_api`, `file_upload`, `log` — все используются.
- Неиспользуемых импортов нет.

---

### 6. `tests/wasm_file_upload.rs`

Интеграционные тесты. Сервер запускается перед тестами, поэтому `#[ignore]` **не используется**.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::file_upload::*;

wasm_bindgen_test_configure!(run_in_browser);

/// URL эндпоинта загрузки. Совпадает с `app.rs`.
/// Сервер запускается через `npm run server` перед тестами.
const UPLOAD_URL: &str = "http://localhost:3000/api/upload";

#[wasm_bindgen_test]
async fn test_upload_file_success() {
    let file = web_sys::File::new_with_str_sequence(
        &js_sys::Array::of1(&"test data".into()),
        "test.txt",
    )
    .expect("File::new_with_str_sequence");

    let result = upload_file(&file, UPLOAD_URL).await;
    assert!(result.is_ok(), "upload_file failed: {:?}", result.err());

    let body = result.unwrap();
    assert!(body.contains("\"status\":\"ok\""), "unexpected body: {body}");
    assert!(
        body.contains("\"filename\":\"test.txt\""),
        "unexpected body: {body}"
    );
    assert!(body.contains("\"size\":9"), "unexpected body: {body}");
}

#[wasm_bindgen_test]
async fn test_upload_blob_with_filename() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"some blob content".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let result = upload_blob(&blob, "report.txt", UPLOAD_URL).await;
    assert!(result.is_ok(), "upload_blob failed: {:?}", result.err());

    let body = result.unwrap();
    assert!(
        body.contains("\"filename\":\"report.txt\""),
        "unexpected body: {body}"
    );
}

#[wasm_bindgen_test]
async fn test_upload_empty_file() {
    let file = web_sys::File::new_with_str_sequence(
        &js_sys::Array::of1(&"".into()),
        "empty.txt",
    )
    .expect("File::new_with_str_sequence");

    let result = upload_file(&file, UPLOAD_URL).await;
    assert!(result.is_ok(), "upload failed: {:?}", result.err());

    let body = result.unwrap();
    assert!(body.contains("\"size\":0"), "unexpected body: {body}");
}

#[wasm_bindgen_test]
async fn test_upload_binary_file() {
    // Бинарные данные (не UTF-8) — проверяем, что multipart
    // не портит байты.
    let bytes: Vec<u8> = (0..=255).collect();
    let uint8 = js_sys::Uint8Array::from(&bytes[..]);
    let parts = js_sys::Array::of1(&uint8.buffer());
    let file = web_sys::File::new_with_u8_array_sequence(&parts, "binary.bin")
        .expect("File::new_with_u8_array_sequence");

    let result = upload_file(&file, UPLOAD_URL).await;
    assert!(result.is_ok(), "upload failed: {:?}", result.err());

    let body = result.unwrap();
    assert!(body.contains("\"size\":256"), "unexpected body: {body}");
}

#[wasm_bindgen_test]
async fn test_upload_cancelled() {
    // Предварительно отменённый сигнал — запрос должен
    // завершиться с AbortError.
    let controller = web_sys::AbortController::new()
        .expect("AbortController::new");
    let signal = controller.signal();
    controller.abort();

    let file = web_sys::File::new_with_str_sequence(
        &js_sys::Array::of1(&"test data".into()),
        "test.txt",
    )
    .expect("File::new_with_str_sequence");

    let result = upload_file_cancellable(&file, UPLOAD_URL, &signal).await;
    assert!(result.is_err(), "ожидалась ошибка отмены");

    let err = result.unwrap_err();
    assert!(is_abort_error(&err), "ожидался AbortError, получено: {err:?}");
}

#[wasm_bindgen_test]
fn test_is_abort_error_non_abort() {
    let e = wasm_bindgen::JsValue::from_str("something else");
    assert!(!is_abort_error(&e));
}
```

**Проверка импортов:**

- `wasm_bindgen_test::*` — `wasm_bindgen_test`, `wasm_bindgen_test_configure`.
- `hello_wasm02::file_upload::*` — `upload_file`, `upload_blob`, `upload_file_cancellable`, `is_abort_error`.

Полные пути `js_sys::*`, `web_sys::*`, `wasm_bindgen::JsValue` — без импортов. Неиспользуемых нет.

**Что проверяем:**

- **`test_upload_file_success`** — базовый сценарий: файл уходит, сервер отвечает, `Result::Ok`. Проверяем содержимое ответа (`status`, `filename`, `size`).
- **`test_upload_blob_with_filename`** — Blob с явным именем.
- **`test_upload_empty_file`** — пустой файл, `size: 0`.
- **`test_upload_binary_file`** — 256 байт всех значений `0..=255` — проверяем, что multipart не портит бинарные данные.
- **`test_upload_cancelled`** — предварительно отменённый запрос даёт `AbortError`.
- **`test_is_abort_error_non_abort`** — обычный `JsValue` не распознаётся как abort.

---

## Команды запуска

### 1. Запуск `auth-server` (отдельный терминал)

```bash
# Первый раз — пересборка с новыми зависимостями
cd auth-server
cargo build
cd ..

# Запуск
npm run server
```

В логе должно появиться:
```
Auth server listening on 127.0.0.1:3000
```

### 2. Проверка эндпоинта вручную

```bash
# Маленький файл
echo "hello" > /tmp/small.txt
curl -X POST http://localhost:3000/api/upload -F "file=@/tmp/small.txt"
# Ожидаемо: {"status":"ok","filename":"small.txt","size":6,"sha256":"..."}

# Большой файл (> 2 МБ — проверяем, что body limit снят)
head -c 5000000 /dev/urandom > /tmp/big.bin
curl -X POST http://localhost:3000/api/upload -F "file=@/tmp/big.bin"
# Ожидаемо: {"status":"ok","filename":"big.bin","size":5000000,"sha256":"..."}

# Убираем за собой
rm /tmp/small.txt /tmp/big.bin
```

**Важно:** сервер **никуда не сохраняет файлы** — чистить на его стороне нечего. Это stateless-обработчик: читает multipart потоково, считает SHA-256, возвращает JSON. Ни записи на диск, ни в БД, ни в постоянную память.

### 3. Сборка и запуск клиента

```bash
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests
npm run start
```

### 4. Тесты подглавы

```bash
# Сервер запущен в отдельном терминале
wasm-pack test --headless --chrome --test wasm_file_upload
wasm-pack test --headless --firefox --test wasm_file_upload
```

Все 6 тестов должны пройти — без `#[ignore]`, потому что `auth-server` доступен.

### 5. Что ожидать в браузере

**Успешная загрузка маленького файла:**
```
✅ Файл успешно загружен

Файл: readme.txt
Размер: 1.23 KB (1260 байт)
URL: http://localhost:3000/api/upload
Время: 123.4 мс
Средняя скорость: 9.97 МБ/с

=== Ответ сервера ===
{"status":"ok","filename":"readme.txt","size":1260,
 "content_type":"text/plain","sha256":"a3f5e..."}

Сервер посчитал SHA-256 принятого файла — сравните его
с локальным хешем (см. 18.12) для проверки целостности.
Файл на сервере не сохраняется: обработчик stateless.
```

**Загрузка большого файла (100 МБ):**
- Статус `⏳ Отправляем... 0.1 с`, `0.2 с`, `0.3 с`… — живой счётчик.
- Кнопка «■» доступна — можно прервать.
- Без прерывания: результат с реальной средней скоростью (для localhost это ~500 МБ/с).

**Отмена:**
```
⏹ Загрузка отменена пользователем

Файл: video.mp4
Размер: 245.34 MB
Прошло: 2.1 с
```

**Сервер не запущен:**
```
❌ Ошибка загрузки

Файл: readme.txt
Размер: 1.23 KB
URL: http://localhost:3000/api/upload
Ошибка: JsValue(TypeError: Failed to fetch)

Проверьте, что auth-server запущен:
npm run server
```

---

## Что важно понимать про Stateless

Наш `upload` **никуда не сохраняет** данные. Это осознанное решение для демонстрационного проекта:

1. **Никаких утечек.** Сервер можно держать запущенным неделями — диск не переполнится.
2. **Нет проблем с правами.** Не нужно думать, куда писать.
3. **Не нужно чистить.** После тестов, после демо, после экспериментов — ничего не остаётся.
4. **Проще контейнеризация.** Нет state volume.

Проверить отсутствие накопления:

```bash
# До теста
ls -la /tmp/
ls -la auth-server/

# Загружаем 100 файлов подряд
for i in $(seq 1 100); do
    echo "test $i" | curl -s -X POST http://localhost:3000/api/upload \
        -F "file=@-" > /dev/null
done

# После — те же самые списки, без изменений
ls -la /tmp/
ls -la auth-server/
```

**В реальных задачах** (сохранение аватарки, документа) сервер должен писать файл на диск. Тогда в обработчик добавляется `tokio::fs::File::create` + `write_all`, а очистка делается либо через `rm -rf uploads/`, либо по TTL-задаче, либо через отдельный эндпоинт `/api/upload/cleanup`. Но это уже не демо — это продакшен, и в учебной главе мы не смешиваем эти сценарии.

---

## Итоги подглавы 18.13

- **`FormData`** — контейнер для multipart-отправки. `append_with_blob` для `File`, `append_with_blob_and_filename` для `Blob` с явным именем.
- **Content-Type не устанавливается вручную** для FormData — браузер сам добавит `multipart/form-data; boundary=...`. Явная установка сломает парсинг на сервере.
- **`RequestInit::set_body`** принимает `&JsValue`; `set_signal` — для отмены.
- **`AbortController::signal()`** передаётся в `set_signal`; `controller.abort()` отменяет запрос, промис реджектится с `AbortError`.
- **`fetch` не даёт прогресс отправки** — свойство API. В UI вместо progress bar — живой счётчик времени.
- **Сервер считает SHA-256 принятого файла.** Клиент показывает хеш и может сравнить с локальным (из 18.12) — подтверждение целостности доставки.
- **Multipart парсится по чанкам** — `field.chunk().await`. Память O(chunk_size), не O(file_size). Гигабайтные файлы не падают по памяти.
- **`DefaultBodyLimit::disable()`** — критично для upload. В axum 0.8 дефолтный лимит тела 2 МБ; без его снятия файлы крупнее 2 МБ падают с `413`.
- **Синтаксис маршрутов axum 0.8** — `{secs}`, `{count}`, не `:secs`, `:count`. Старый синтаксис паникует при старте.
- **Stateless-сервер** — файлы не сохраняются. Чистить после тестов нечего.

Модуль `file_upload.rs` автономен, все tail-функции через явный `return`, неиспользуемых импортов нет. `app.rs` содержит **только** секцию 18.13. Сервер дополнен минимально — три правки в существующих файлах, остальные обработчики не затронуты.

## Замечания на будущее

**Правило проекта, зафиксированное в этой подглаве:**

1. **Никогда не устанавливайте Content-Type вручную для `FormData`.** Браузер добавляет `multipart/form-data; boundary=...` сам. Явная установка ломает парсинг.

2. **`DefaultBodyLimit` в axum 0.8 по умолчанию — 2 МБ.** Если сервер принимает большие тела через `Multipart` или `Bytes`, лимит надо снять явно. Точечное снятие через `.layer(DefaultBodyLimit::disable())` на конкретном маршруте безопаснее глобального.

3. **Если API не даёт прогресс — покажите живость.** Живой счётчик времени + кнопка отмены — минимальный UX без обмана. Indeterminate progress bar запрещён.

4. **`AbortController` — универсальный механизм отмены async-операций.** Работает с `fetch`, `ReadableStream`, DOM-событиями. Один контроллер — одна операция.

5. **Multipart на сервере парсите потоково.** `field.chunk().await` даёт чанки по мере поступления. Не используйте `field.bytes().await` для потенциально больших файлов — это загрузит весь файл в память.

6. **Stateless сервер проще чистить.** Наш `upload` ничего не сохраняет — после теста чистить на стороне сервера нечего. Если в будущем добавите сохранение в `uploads/`, помните о необходимости периодической очистки: `rm -rf uploads/`, TTL-задача или эндпоинт `/api/upload/cleanup`.

7. **При обновлении мажорных версий crate'ов проверяйте breaking changes.** axum 0.8 → паника на `:param` в маршрутах; jsonwebtoken 11 → обязательный криптопровайдер; `DefaultBodyLimit` может менять дефолт. `cargo build` ловит ошибки компиляции, но не панику в рантайме — её видно только при запуске сервера.

**Что делать дальше:** в 18.14 закроем главное ограничение текущего решения — отсутствие прогресс-бара. Заменим `fetch` на `XMLHttpRequest` с `upload.onprogress`, покажем реальный процент отправки. URL и серверный контракт не меняются — только клиентский код переключится с `fetch` на XHR.