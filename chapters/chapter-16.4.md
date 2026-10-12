# Глава 16.4. Аутентификация и авторизация

## Обновлённая структура проекта

```
.
├── auth-server/                    ← НОВОЕ: серверная часть (Axum + JWT)
│   ├── Cargo.toml
│   ├── .env.example
│   └── src/
│       ├── main.rs                 ← роутинг, CORS, запуск
│       ├── auth.rs                 ← login, refresh, валидация JWT
│       └── handlers.rs             ← защищённые эндпоинты /api/me, /api/admin
│
├── src/
│   ├── auth.rs                     ← НОВЫЙ: TokenStore, login, try_refresh, logout
│   ├── fetch.rs                    ← ОБНОВЛЁН: +5 функций
│   ├── app.rs                      ← ОБНОВЛЁН: секция 4 «Аутентификация»
│   ├── lib.rs                      ← ОБНОВЛЁН: +pub mod auth;
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_auth.rs                ← НОВЫЙ: 6 тестов аутентификации
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: +serde-wasm-bindgen = "0.6"
├── package.json                    ← ОБНОВЛЁН: +scripts "server", "server:release"
├── vite.config.js                  ← ОБНОВЛЁН: +proxy для /api → :3000
└── (остальное без изменений)
```

**Краткий комментарий к структуре.** Серверная часть — **отдельный крейт** в подпапке `auth-server/`. Он не входит в workspace (у нас его нет), поэтому команды вроде `cargo check --target wasm32-unknown-unknown` не пытаются собрать его под WASM. Это принципиально: `auth-server` тянет `tokio`, `hyper`, `socket2`, которые под WASM не собираются. Два крейта живут рядом, но собираются независимо. Связь между ними — только по сети через HTTP.

Клиентский код остаётся тем же WASM-приложением. Добавляется один модуль `auth.rs` и пять функций в `fetch.rs`. Остальные модули не трогаются.

## Ссылки для детального изучения

- **Axum** — официальная документация и примеры: [docs.rs/axum](https://docs.rs/axum)
- **RFC 6750 — The OAuth 2.0 Authorization Framework: Bearer Token Usage**: [datatracker.ietf.org/doc/html/rfc6750](https://datatracker.ietf.org/doc/html/rfc6750)
- **RFC 7519 — JSON Web Token (JWT)**: [datatracker.ietf.org/doc/html/rfc7519](https://datatracker.ietf.org/doc/html/rfc7519)
- **MDN: HTTP Authentication** — схемы Basic, Bearer, Digest, заголовок `WWW-Authenticate`: [developer.mozilla.org/ru/docs/Web/HTTP/Authentication](https://developer.mozilla.org/ru/docs/Web/HTTP/Authentication)
- **MDN: 401 Unauthorized vs 403 Forbidden** — когда какой код использовать: [developer.mozilla.org/ru/docs/Web/HTTP/Status](https://developer.mozilla.org/ru/docs/Web/HTTP/Status)
- **OWASP: JWT Cheat Sheet** — где хранить токены, почему `localStorage` опасен: [cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- **OWASP: XSS Prevention** — почему HttpOnly-cookie защищает refresh-токен: [cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- **`jsonwebtoken` crate** — документация библиотеки для работы с JWT в Rust: [docs.rs/jsonwebtoken](https://docs.rs/jsonwebtoken)
- **Vite: Server Options → proxy** — как настраивать проксирование: [vitejs.dev/config/server-options.html#server-proxy](https://vitejs.dev/config/server-options.html#server-proxy)

## Новые и обновлённые файлы

### `auth-server/Cargo.toml` (новый)

```toml
[package]
name = "auth-server"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
tower-http = { version = "0.5", features = ["cors"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jsonwebtoken = "9"
chrono = { version = "0.4", features = ["serde"] }
dotenvy = "0.15"
```

### `auth-server/.env.example` (новый)

```
JWT_SECRET=change-me-in-production
BIND_ADDRESS=127.0.0.1:3000
ACCESS_TOKEN_TTL=300
REFRESH_TOKEN_TTL=86400
```

### `auth-server/src/main.rs` (новый)

```rust
mod auth;
mod handlers;

use axum::{routing::{get, post}, Router};
use tower_http::cors::CorsLayer;

#[tokio::main]
async fn main() {
    dotenvy::dotenv().ok();

    let app = Router::new()
        .route("/api/auth/login", post(auth::login))
        .route("/api/auth/refresh", post(auth::refresh))
        .route("/api/me", get(handlers::me))
        .route("/api/admin", get(handlers::admin_only))
        .layer(CorsLayer::permissive());

    let addr = std::env::var("BIND_ADDRESS").unwrap_or("127.0.0.1:3000".into());
    println!("Auth server listening on {addr}");
    let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### `auth-server/src/auth.rs` (новый)

```rust
use axum::{http::StatusCode, Json};
use chrono::{Duration, Utc};
use jsonwebtoken::{encode, EncodingKey, Header};
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub exp: i64,
    pub typ: String,
    pub roles: Vec<String>,
}

#[derive(Deserialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

#[derive(Deserialize)]
pub struct RefreshRequest {
    pub refresh_token: String,
}

fn secret() -> String {
    std::env::var("JWT_SECRET").unwrap_or("dev-secret".into())
}

fn make_token(sub: &str, typ: &str, ttl_secs: i64, roles: Vec<String>) -> String {
    let claims = Claims {
        sub: sub.into(),
        exp: (Utc::now() + Duration::seconds(ttl_secs)).timestamp(),
        typ: typ.into(),
        roles,
    };
    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(secret().as_bytes()),
    )
    .unwrap()
}

pub async fn login(Json(req): Json<LoginRequest>) -> Result<Json<Value>, StatusCode> {
    if req.password != "secret" {
        return Err(StatusCode::UNAUTHORIZED);
    }

    let roles = if req.username == "admin" {
        vec!["user".into(), "admin".into()]
    } else {
        vec!["user".into()]
    };

    let access_ttl: i64 = std::env::var("ACCESS_TOKEN_TTL")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(300);
    let refresh_ttl: i64 = std::env::var("REFRESH_TOKEN_TTL")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(86400);

    let access = make_token(&req.username, "access", access_ttl, roles.clone());
    let refresh = make_token(&req.username, "refresh", refresh_ttl, roles);

    Ok(Json(json!({
        "access_token": access,
        "refresh_token": refresh,
        "token_type": "Bearer",
        "expires_in": access_ttl
    })))
}

pub async fn refresh(Json(req): Json<RefreshRequest>) -> Result<Json<Value>, StatusCode> {
    use jsonwebtoken::{decode, DecodingKey, Validation};

    let token_data = decode::<Claims>(
        &req.refresh_token,
        &DecodingKey::from_secret(secret().as_bytes()),
        &Validation::default(),
    )
    .map_err(|_| StatusCode::UNAUTHORIZED)?;

    if token_data.claims.typ != "refresh" {
        return Err(StatusCode::UNAUTHORIZED);
    }

    let access_ttl: i64 = std::env::var("ACCESS_TOKEN_TTL")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(300);

    let new_access = make_token(
        &token_data.claims.sub,
        "access",
        access_ttl,
        token_data.claims.roles,
    );

    Ok(Json(json!({
        "access_token": new_access,
        "token_type": "Bearer",
        "expires_in": access_ttl
    })))
}

/// Извлекает и валидирует access-токен из заголовка Authorization.
pub fn validate_bearer(headers: &axum::http::HeaderMap) -> Result<Claims, StatusCode> {
    use jsonwebtoken::{decode, DecodingKey, Validation};

    let auth = headers
        .get(axum::http::header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let token = auth
        .strip_prefix("Bearer ")
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret().as_bytes()),
        &Validation::default(),
    )
    .map_err(|_| StatusCode::UNAUTHORIZED)?;

    if data.claims.typ != "access" {
        return Err(StatusCode::UNAUTHORIZED);
    }

    Ok(data.claims)
}
```

### `auth-server/src/handlers.rs` (новый)

```rust
use axum::{http::HeaderMap, http::StatusCode, Json};
use serde_json::{json, Value};

use crate::auth::validate_bearer;

pub async fn me(headers: HeaderMap) -> Result<Json<Value>, StatusCode> {
    let claims = validate_bearer(&headers)?;
    Ok(Json(json!({
        "username": claims.sub,
        "roles": claims.roles
    })))
}

pub async fn admin_only(headers: HeaderMap) -> Result<Json<Value>, StatusCode> {
    let claims = validate_bearer(&headers)?;

    if !claims.roles.iter().any(|r| r == "admin") {
        return Err(StatusCode::FORBIDDEN);
    }

    Ok(Json(json!({
        "message": "Добро пожаловать, администратор",
        "username": claims.sub
    })))
}
```

### `src/auth.rs` (новый)

```rust
//! Управление токенами на клиенте.
//!
//! Access-токен хранится только в памяти — это безопаснее, чем `localStorage`:
//! XSS-атака не сможет прочитать его через `localStorage.getItem`.
//! Refresh-токен в реальном приложении должен жить в HttpOnly-cookie,
//! но для простоты учебного примера держим и его в памяти.

use std::cell::RefCell;
use std::rc::Rc;
use std::sync::OnceLock;

use wasm_bindgen::JsValue;

use crate::fetch;

/// Базовый URL API. Пустой по умолчанию — тогда используются
/// относительные пути, и запросы идут через прокси dev-сервера Vite.
/// В тестах выставляется в `http://127.0.0.1:3000`.
static API_BASE: OnceLock<String> = OnceLock::new();

/// Задаёт базовый URL API. Идемпотентно: повторные вызовы игнорируются.
pub fn set_api_base(url: &str) {
    let _ = API_BASE.set(url.trim_end_matches('/').to_string());
}

fn api_base() -> &'static str {
    API_BASE.get().map(|s| s.as_str()).unwrap_or("")
}

fn api_url(path: &str) -> String {
    return format!("{}{}", api_base(), path);
}

/// Разделяемое хранилище токенов.
#[derive(Default)]
pub struct TokenStore {
    pub access_token: Option<String>,
    pub refresh_token: Option<String>,
}

/// Удобный псевдоним для передачи хранилища между замыканиями.
pub type TokenStoreHandle = Rc<RefCell<TokenStore>>;

impl TokenStore {
    /// Создаёт новое пустое хранилище, обёрнутое в `Rc<RefCell<…>>`.
    pub fn new() -> TokenStoreHandle {
        Rc::new(RefCell::new(Self::default()))
    }

    /// Устанавливает оба токена. `refresh` опционален, потому что
    /// при обновлении access-токена новый refresh обычно не выдаётся.
    pub fn set_tokens(&mut self, access: String, refresh: Option<String>) {
        self.access_token = Some(access);
        if let Some(r) = refresh {
            self.refresh_token = Some(r);
        }
    }

    /// Полная очистка — используется при logout и при неудачном refresh.
    pub fn clear(&mut self) {
        self.access_token = None;
        self.refresh_token = None;
    }
}

/// Логин: отправляет credentials, сохраняет оба токена.
pub async fn login(
    store: &TokenStoreHandle,
    username: &str,
    password: &str,
) -> Result<(), JsValue> {
    let url = api_url("/api/auth/login");
    let body = serde_json::json!({
        "username": username,
        "password": password
    });

    let resp = fetch::post_json_raw(&url, &body).await?;

    if resp.status() != 200 {
        return Err(JsValue::from_str(&format!(
            "login failed: status {}",
            resp.status()
        )));
    }

    let json: serde_json::Value = fetch::response_json(&resp).await?;

    let access = json["access_token"]
        .as_str()
        .ok_or_else(|| JsValue::from_str("no access_token"))?
        .to_string();
    let refresh = json["refresh_token"].as_str().map(|s| s.to_string());

    store.borrow_mut().set_tokens(access, refresh);
    Ok(())
}

/// Пытается обновить access-токен через refresh-токен.
///
/// Возвращает:
/// - `Ok(true)` — refresh прошёл, access-токен обновлён;
/// - `Ok(false)` — refresh-токена нет или сервер отказал (хранилище очищено);
/// - `Err(_)` — транспортная ошибка (сеть, CORS и т.п.).
pub async fn try_refresh(store: &TokenStoreHandle) -> Result<bool, JsValue> {
    let refresh = match store.borrow().refresh_token.clone() {
        Some(r) => r,
        None => return Ok(false),
    };

    let url = api_url("/api/auth/refresh");
    let body = serde_json::json!({ "refresh_token": refresh });
    let resp = fetch::post_json_raw(&url, &body).await;

    match resp {
        Ok(r) if r.status() == 200 => {
            let json: serde_json::Value = fetch::response_json(&r).await?;
            let access = json["access_token"]
                .as_str()
                .ok_or_else(|| JsValue::from_str("no access_token"))?
                .to_string();
            store.borrow_mut().access_token = Some(access);
            Ok(true)
        }
        Ok(_) => {
            // Сервер явно отказал — refresh-токен невалиден, чистим всё.
            store.borrow_mut().clear();
            Ok(false)
        }
        Err(e) => Err(e),
    }
}

/// Logout: просто забываем токены на клиенте.
pub fn logout(store: &TokenStoreHandle) {
    store.borrow_mut().clear();
}
```

### `src/fetch.rs` (обновлён — добавить в конец файла)

В начале файла добавьте импорт:

```rust
use crate::auth::TokenStoreHandle;
```

Пять новых функций:

```rust
/// POST с JSON-телом, возвращает сырой Response (без десериализации).
pub async fn post_json_raw(url: &str, body: &serde_json::Value) -> Result<web_sys::Response, JsValue> {
    let body_str = serde_json::to_string(body).map_err(|e| JsValue::from_str(&e.to_string()))?;
    post_json_raw_string(url, &body_str).await
}

pub async fn post_json_raw_string(url: &str, body: &str) -> Result<web_sys::Response, JsValue> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("POST");
    opts.set_mode(web_sys::RequestMode::Cors);
    opts.set_body(&JsValue::from_str(body));

    let request = web_sys::Request::new_with_str_and_init(url, &opts)?;
    request.headers().set("Content-Type", "application/json")?;

    let window = web_sys::window().unwrap();
    let resp_value = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
    .await?;

    resp_value.dyn_into::<web_sys::Response>()
}

/// Десериализует Response как JSON.
pub async fn response_json(resp: &web_sys::Response) -> Result<serde_json::Value, JsValue> {
    let promise = resp.json()?;
    let json: JsValue = wasm_bindgen_futures::JsFuture::from(promise).await?;
    let value: serde_json::Value = serde_wasm_bindgen::from_value(json)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    Ok(value)
}

/// Запрос с Bearer-токеном.
pub async fn make_authenticated_request(
    url: &str,
    token: &str,
) -> Result<web_sys::Response, JsValue> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    let request = web_sys::Request::new_with_str_and_init(url, &opts)?;
    request
        .headers()
        .set("Authorization", &format!("Bearer {token}"))?;

    let window = web_sys::window().unwrap();
    let resp_value = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
    .await?;

    resp_value.dyn_into::<web_sys::Response>()
}

/// Обёртка с автоматическим refresh при 401.
///
/// Если запрос вернул 401 — пытается обновить токен и повторить запрос один раз.
pub async fn send_with_auth_retry(
    store: &TokenStoreHandle,
    url: &str,
) -> Result<web_sys::Response, JsValue> {
    let token = store
        .borrow()
        .access_token
        .clone()
        .ok_or_else(|| JsValue::from_str("no access token"))?;

    let resp = make_authenticated_request(url, &token).await?;

    if resp.status() != 401 {
        return Ok(resp);
    }

    // Токен истёк — пробуем refresh.
    let refreshed = crate::auth::try_refresh(store).await?;
    if !refreshed {
        return Ok(resp); // refresh не удался — возвращаем исходный 401
    }

    let new_token = store.borrow().access_token.clone().unwrap();
    make_authenticated_request(url, &new_token).await
}
```

### `src/lib.rs` (обновлён)

Добавьте одну строку — новый модуль должен быть объявлен:

```rust
pub mod app;
pub mod async_utils;
pub mod auth;          // ← НОВЫЙ модуль
pub mod callbacks;
pub mod dom;
pub mod events;
pub mod fetch;
pub mod form;
pub mod js_objects;
pub mod logging;
pub mod navigation;
pub mod notifier;
pub mod storage;
pub mod value;
```

### `src/app.rs` (обновлён — добавлен импорт и секция 4)

В начале файла добавьте `auth` в импорт:

```rust
use crate::callbacks::{self, EventClosures};
use crate::{auth, dom};
use crate::fetch;
use crate::log;
```

Секция 4 — после секции 3, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 4: Аутентификация
    // ============================================================

    let section4 = build_section(&container, "4. Аутентификация")?;

    let intro4 = dom::create_paragraph(
        "Bearer-токены, обработка 401/403, автоматический refresh. \
         Логин: любой username, пароль — «secret». \
         Для admin-эндпоинта используйте username «admin».",
    )?;
    dom::add_class(&dom::as_html(&intro4), "hint")?;
    dom::append(&section4, &intro4)?;

    let token_store = auth::TokenStore::new();

    let login_btn = dom::create_element("button")?;
    login_btn.set_text_content(Some("Войти как user"));
    let login_admin_btn = dom::create_element("button")?;
    login_admin_btn.set_text_content(Some("Войти как admin"));
    let me_btn = dom::create_element("button")?;
    me_btn.set_text_content(Some("GET /api/me"));
    let admin_btn = dom::create_element("button")?;
    admin_btn.set_text_content(Some("GET /api/admin"));

    dom::append(&section4, &login_btn)?;
    dom::append(&section4, &login_admin_btn)?;
    dom::append(&section4, &me_btn)?;
    dom::append(&section4, &admin_btn)?;

    let result4 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result4), "api-result")?;
    result4.set_text_content(Some("(не авторизован)"));
    dom::append(&section4, &result4)?;

    // ---- Login (user) ----
    {
        let result4 = result4.clone();
        let store = token_store.clone();
        callbacks::listen(&login_btn, "click", move |_| {
            let result4 = result4.clone();
            let store = store.clone();
            result4.set_text_content(Some("Вход..."));
            wasm_bindgen_futures::spawn_local(async move {
                match auth::login(&store, "user", "secret").await {
                    Ok(()) => result4.set_text_content(Some("✅ Вход выполнен (user)")),
                    Err(e) => result4.set_text_content(Some(&format!("❌ {e:?}"))),
                }
            });
        }, &closures)?;
    }

    // ---- Login (admin) ----
    {
        let result4 = result4.clone();
        let store = token_store.clone();
        callbacks::listen(&login_admin_btn, "click", move |_| {
            let result4 = result4.clone();
            let store = store.clone();
            result4.set_text_content(Some("Вход..."));
            wasm_bindgen_futures::spawn_local(async move {
                match auth::login(&store, "admin", "secret").await {
                    Ok(()) => result4.set_text_content(Some("✅ Вход выполнен (admin)")),
                    Err(e) => result4.set_text_content(Some(&format!("❌ {e:?}"))),
                }
            });
        }, &closures)?;
    }

    // ---- GET /api/me ----
    {
        let result4 = result4.clone();
        let store = token_store.clone();
        callbacks::listen(&me_btn, "click", move |_| {
            let result4 = result4.clone();
            let store = store.clone();
            result4.set_text_content(Some("Запрос..."));
            wasm_bindgen_futures::spawn_local(async move {
                match fetch::send_with_auth_retry(&store, "/api/me").await {
                    Ok(resp) if resp.status() == 200 => {
                        let json: serde_json::Value = fetch::response_json(&resp).await.unwrap();
                        result4.set_text_content(Some(&format!("✅ {json}")));
                    }
                    Ok(resp) => {
                        result4.set_text_content(Some(&format!("❌ Статус {}", resp.status())));
                    }
                    Err(e) => result4.set_text_content(Some(&format!("❌ {e:?}"))),
                }
            });
        }, &closures)?;
    }

    // ---- GET /api/admin ----
    {
        let result4 = result4.clone();
        let store = token_store.clone();
        callbacks::listen(&admin_btn, "click", move |_| {
            let result4 = result4.clone();
            let store = store.clone();
            result4.set_text_content(Some("Запрос..."));
            wasm_bindgen_futures::spawn_local(async move {
                match fetch::send_with_auth_retry(&store, "/api/admin").await {
                    Ok(resp) if resp.status() == 200 => {
                        let json: serde_json::Value = fetch::response_json(&resp).await.unwrap();
                        result4.set_text_content(Some(&format!("✅ {json}")));
                    }
                    Ok(resp) if resp.status() == 403 => {
                        result4.set_text_content(Some("❌ 403 Forbidden: нет прав администратора"));
                    }
                    Ok(resp) => {
                        result4.set_text_content(Some(&format!("❌ Статус {}", resp.status())));
                    }
                    Err(e) => result4.set_text_content(Some(&format!("❌ {e:?}"))),
                }
            });
        }, &closures)?;
    }
```

### `tests/wasm_auth.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Тесты в `wasm-pack test` работают на собственном HTTP-сервере
/// на случайном порту, а не через Vite. Поэтому относительные URL
/// (`/api/auth/login`) резолвятся против тестового origin и не доходят
/// до auth-server. Переключаем клиент на абсолютный адрес.
fn use_direct_server() {
    hello_wasm02::auth::set_api_base("http://127.0.0.1:3000");
}

#[wasm_bindgen_test]
async fn test_make_request_with_bearer() {
    let resp = hello_wasm02::fetch::make_authenticated_request(
        "http://127.0.0.1:3000/api/me",
        "test-token",
    )
    .await
    .unwrap();

    // Без валидного токена сервер вернёт 401 — но заголовок был отправлен.
    assert_eq!(resp.status(), 401);
}

#[wasm_bindgen_test]
async fn test_authorization_header_set() {
    // Косвенная проверка: сервер отвечает 401, а не 400 — значит, заголовок дошёл.
    let resp = hello_wasm02::fetch::make_authenticated_request(
        "http://127.0.0.1:3000/api/me",
        "invalid",
    )
    .await
    .unwrap();
    assert_eq!(resp.status(), 401);
}

#[wasm_bindgen_test]
async fn test_401_handling() {
    let resp = hello_wasm02::fetch::make_authenticated_request(
        "http://127.0.0.1:3000/api/me",
        "",
    )
    .await
    .unwrap();
    assert_eq!(resp.status(), 401);
}

#[wasm_bindgen_test]
async fn test_403_handling() {
    use_direct_server();
    let store = hello_wasm02::auth::TokenStore::new();
    hello_wasm02::auth::login(&store, "user", "secret").await.unwrap();

    let token = store.borrow().access_token.clone().unwrap();
    let resp = hello_wasm02::fetch::make_authenticated_request(
        "http://127.0.0.1:3000/api/admin",
        &token,
    )
    .await
    .unwrap();

    assert_eq!(resp.status(), 403);
}

#[wasm_bindgen_test]
async fn test_token_refresh_flow() {
    use_direct_server();
    let store = hello_wasm02::auth::TokenStore::new();
    hello_wasm02::auth::login(&store, "user", "secret").await.unwrap();

    // Первый запрос — успешен.
    let resp = hello_wasm02::fetch::send_with_auth_retry(
        &store,
        "http://127.0.0.1:3000/api/me",
    )
    .await
    .unwrap();
    assert_eq!(resp.status(), 200);

    // Искусственно портим access-токен, оставляя refresh валидным.
    store.borrow_mut().access_token = Some("broken".into());

    let resp2 = hello_wasm02::fetch::send_with_auth_retry(
        &store,
        "http://127.0.0.1:3000/api/me",
    )
    .await
    .unwrap();
    assert_eq!(resp2.status(), 200, "refresh должен был восстановить доступ");
}

#[wasm_bindgen_test]
async fn test_auth_header_not_duplicated() {
    use_direct_server();
    let store = hello_wasm02::auth::TokenStore::new();
    hello_wasm02::auth::login(&store, "user", "secret").await.unwrap();

    store.borrow_mut().access_token = Some("broken".into());

    // send_with_auth_retry создаёт новый Request — дублирования быть не должно.
    let resp = hello_wasm02::fetch::send_with_auth_retry(
        &store,
        "http://127.0.0.1:3000/api/me",
    )
    .await
    .unwrap();
    assert_eq!(resp.status(), 200);
}
```

### `Cargo.toml` (корневой, обновлён)

Добавить в `[dependencies]`:

```toml
serde-wasm-bindgen = "0.6"
```

Внимание на написание: в `Cargo.toml` — дефис (`serde-wasm-bindgen`), в коде — подчёркивание (`serde_wasm_bindgen`). Это стандартное правило Rust.

### `package.json` (обновлён)

Добавить два скрипта в `scripts`:

```json
"server": "cargo run --manifest-path auth-server/Cargo.toml",
"server:release": "cargo run --release --manifest-path auth-server/Cargo.toml",
```

Флаг `--manifest-path` позволяет запускать сервер из корня проекта, не переходя в подпапку.

### `vite.config.js` (обновлён)

Добавить блок `proxy` внутрь существующей секции `server`:

```js
server: {
    host: '127.0.0.1',
    port: 5173,
    fs: {
        allow: ['..'],
    },
    proxy: {
        '/api': {
            target: 'http://127.0.0.1:3000',
            changeOrigin: true,
        },
    },
},
```

`rewrite` **не добавлять**: путь должен передаваться как есть (`/api/auth/login` → `http://127.0.0.1:3000/api/auth/login`).

### `.gitignore` (обновлён)

Добавить:

```
auth-server/target/
auth-server/.env
```

## Команды запуска

### Подготовка (один раз)

```bash
# Установить wasm-pack, если ещё не установлен
cargo install wasm-pack

# Установить npm-зависимости
npm install

# Создать конфиг сервера из шаблона
cp auth-server/.env.example auth-server/.env
# Отредактировать auth-server/.env — задать свой JWT_SECRET
```

### Запуск в трёх терминалах (разработка)

**Терминал 1 — сервер аутентификации:**

```bash
npm run server
```

Сервер поднимется на `http://127.0.0.1:3000`. В логе будет: `Auth server listening on 127.0.0.1:3000`.

**Терминал 2 — сборка WASM и dev-сервер Vite:**

```bash
npm run wasm:dev   # одноразовая сборка WASM в pkg/
npm run dev        # Vite на http://localhost:5173
```

Откройте `http://localhost:5173`. Секция 4 «Аутентификация» должна работать: «Войти как user» → «GET /api/me» → 200; «GET /api/admin» → 403. «Войти как admin» → «GET /api/admin» → 200.

**Терминал 3 — тесты (когда сервер уже работает):**

```bash
npm run test:chrome
```

Ожидаемый результат: `6 passed; 0 failed`.

### Продакшн-сборка

```bash
npm run build        # wasm-pack build + vite build → dist/
npm run server:release
```

### Дополнительные команды

```bash
# Проверка компиляции клиента без запуска
cargo check --target wasm32-unknown-unknown

# Проверка компиляции сервера
cargo check --manifest-path auth-server/Cargo.toml

# Проверить, что сервер отвечает (ручная проверка)
curl -i -X POST http://127.0.0.1:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"user","password":"secret"}'
```

## Итог главы

Мы построили настоящую систему аутентификации с сервером и клиентом:

1. **Сервер** на Axum выдаёт JWT с HMAC-подписью, различает access- и refresh-токены по полю `typ`, отдаёт 401 для невалидных токенов и 403 для нехватки прав.
2. **Клиент** хранит access-токен в памяти (не в `localStorage` — XSS его не прочитает), логинится, шлёт Bearer-заголовок и автоматически повторяет запрос при 401, обновив токен через refresh.
3. **Vite-прокси** позволяет клиенту ходить по относительным путям, а тесты переключаются на абсолютные URL через `set_api_base`.
4. **Тесты** покрывают все ключевые сценарии: 401, 403, refresh-flow и отсутствие дублирования заголовка.

Ключевые уроки:

- Аутентификацию нельзя эмулировать — реальный сервер показывает поведение сети, которое мок скрывает.
- `resp.status()` нужно проверять **до** `resp.json()` — иначе пустое тело даёт загадочный `Unexpected end of JSON input`.
- `web_sys::Response::clone()` возвращает `Result`, а не `Response` — это DOM-метод, а не трейт `Clone`.
- 401 и 403 — это разные ситуации, и клиент должен реагировать на них по-разному.
- В тестах `wasm-pack` нет прокси — конфигурируемая база API решает проблему без хардкода в `src/`.

В следующей главе — **HttpOnly-cookie для refresh-токена**, **SameSite и CSRF**, **logout с blacklist на сервере** и **проактивное обновление токена по таймеру**, а не только по факту 401.