# Глава 41.2 Production-grade WebAuthn-сервер

## Зачем это нужно

Первая версия companion-сервера (Глава 41.01) намеренно опустила четыре вещи, чтобы показать базовую механику протокола. Этот companion — **Глава 41.02** — устраняет их и показывает **настоящее** WebAuthn-решение.

| # | Упрощение в 41.01 | Как решено в 41.02 |
|---|---|---|
| 1 | Attestation не парсится через CBOR | `webauthn-rs` + `ciborium` |
| 2 | Подпись assertion не проверяется | `finish_passkey_authentication` |
| 3 | Пользователи в памяти | SQLite через `sqlx` |
| 4 | Сессии не используются | `tower-sessions` + таблица `ceremonies` |

Проект **standalone** — свой `Cargo.toml`, свой `main()`, не входит в workspace `hello-wasm02`.

---

## Что мы узнали при разработке: 8 грабель

### Грабли 1. Типы в `webauthn-rs` 0.5 переименованы

Привычные из документации имена `PublicKeyCredentialCreationOptions` и `PublicKeyCredentialRequestOptions` в крейте **отсутствуют**. В 0.5 их заменили на:

- `CreationChallengeResponse` — options для `navigator.credentials.create()`;
- `RequestChallengeResponse` — options для `navigator.credentials.get()`.

Оба типа живут в `webauthn_rs::prelude`. Если копировать примеры из интернета, компилятор скажет `cannot find type ... in this scope`.

### Грабли 2. `danger-allow-state-serialisation` — обязательная feature

`PasskeyRegistration` и `PasskeyAuthentication` по умолчанию **не реализуют** `Serialize`/`Deserialize`. Без feature `danger-allow-state-serialisation` невозможно сохранить состояние церемонии в БД.

Название с префиксом `danger-` — не случайно: state содержит `challenge` и другие чувствительные данные. Разрешая сериализацию, вы обязуетесь **удалять state после первого использования** (у нас это делает `take_state_generic` через `DELETE`).

### Грабли 3. `Passkey::counter()` не существует

Метод `counter()` есть у `AuthenticationResult`, но не у `Passkey`. Актуальный `sign_count` после успешной аутентификации берётся так:

```rust
let auth_result = webauthn.finish_passkey_authentication(&cred, &state)?;
let new_sign_count = auth_result.counter();
```

Затем `Passkey::update_credential(&auth_result)` применяет новый счётчик и backup_state к сохранённому ключу.

### Грабли 4. `ciborium::Value` помечен `#[non_exhaustive]`

Даже если в `match v` перечислить все варианты из документации (`Bytes`, `Integer`, `Text`, `Bool`, `Array`, `Map`, `Null`, `Float`, `Tag`), компилятор потребует wildcard-arm:

```rust
_ => "unknown".to_string(),
```

Это защита от добавления новых вариантов в будущих версиях крейта.

### Грабли 5. CORS: нельзя `credentials(true)` + `headers(*)`

`tower-http` паникует **при старте**, если сконфигурировать:

```rust
CorsLayer::new()
    .allow_headers(Any)          // ← "*"
    .allow_credentials(true)     // ← "true"
```

Спецификация Fetch запрещает `Access-Control-Allow-Headers: *` вместе с `Access-Control-Allow-Credentials: true`. Нужно указывать origin и headers явно:

```rust
let allowed_origin = HeaderValue::from_static("http://localhost:3030");
CorsLayer::new()
    .allow_origin(allowed_origin)
    .allow_methods([Method::GET, Method::POST, Method::OPTIONS])
    .allow_headers([header::CONTENT_TYPE, header::COOKIE, header::AUTHORIZATION])
    .allow_credentials(true)
```

### Грабли 6. `webauthn-rs` оборачивает options в `{ publicKey: {...} }`

Согласно W3C WebAuthn Level 2, `PublicKeyCredentialCreationOptions` передаётся браузеру **вложенно**:

```js
navigator.credentials.create({ publicKey: options })
```

`webauthn-rs` строго следует спеке и сериализует `CreationChallengeResponse` как `{ publicKey: {...} }`. Значит, на клиенте нужно обращаться к `options.publicKey.challenge`, а не к `options.challenge`.

Симптом ошибки — `can't access property "replace", s is undefined`: клиент вызывает `b64uToBytes(undefined)`, и первое, что делает функция, — `undefined.replace(...)`.

### Грабли 7. Кеш браузера прячет новые версии статики

Firefox агрессивно кеширует `<script type="module">`. После правки `index.html` можно получить старую версию, даже при `Ctrl+Shift+R`. `ServeDir` не выставляет `Cache-Control`, и браузер применяет эвристику: файл без заголовка «свежий» 10% времени с последнего изменения.

Решение — добавить слой `SetResponseHeaderLayer::overriding(header::CACHE_CONTROL, "no-store")` ко всему роутеру. Требуется feature `set-header` в `tower-http`.

### Грабли 8. `[workspace]` в `Cargo.toml` — обязателен

Если проект не часть workspace, а родительский `hello-wasm02/Cargo.toml` объявит `[workspace]`, cargo начнёт искать модули не так, как ожидается, и выдаст странные ошибки `unresolved imports`. Явная секция `[workspace]` в начале файла изолирует проект.

---

## Структура проекта

```
webauthn-server-prod/
├── Cargo.toml
├── README.md
├── migrations/
│   └── 001_init.sql           ← схема БД
├── src/
│   ├── main.rs                ← запуск, роуты, CORS, no-cache
│   ├── state.rs               ← разделяемое состояние (pool + Webauthn)
│   ├── error.rs               ← AppError + IntoResponse
│   ├── db.rs                  ← sqlx: users, passkeys, ceremonies
│   ├── attestation.rs         ← ручной парсинг CBOR через ciborium
│   └── handlers.rs            ← 4 эндпоинта с webauthn-rs
└── static/
    └── index.html             ← клиент на vanilla JS
```

---

## `webauthn-server-prod/Cargo.toml`

```toml
[workspace]
# ↑ Standalone-проект, не часть workspace hello-wasm02.

[package]
name = "webauthn-server-prod"
version = "0.1.0"
edition = "2021"
description = "Production-grade WebAuthn server with webauthn-rs, SQLite, and server-side sessions"

[dependencies]
# HTTP + async runtime
axum = "0.7"
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
tower-http = { version = "0.5", features = ["fs", "cors", "set-header"] }
tower-sessions = "0.13"

# WebAuthn — реализует серверную часть протокола.
# Feature `danger-allow-state-serialisation` разрешает сериализацию
# PasskeyRegistration и PasskeyAuthentication. State содержит
# challenge и должен удаляться после первого использования.
webauthn-rs = { version = "0.5", features = ["danger-allow-state-serialisation"] }

# CBOR-парсинг для демонстрации содержимого attestationObject
ciborium = "0.2"

# SQLite
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "chrono"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Misc
uuid = { version = "1", features = ["v4", "serde"] }
url = "2"
chrono = { version = "0.4", features = ["serde"] }
base64 = "0.22"
rand = "0.8"
```

---

## `webauthn-server-prod/migrations/001_init.sql`

```sql
-- Схема БД для production WebAuthn.

-- Пользователи.
CREATE TABLE IF NOT EXISTS users (
    user_id TEXT PRIMARY KEY,       -- UUID v4
    username TEXT UNIQUE NOT NULL,
    display_name TEXT NOT NULL,
    created_at TEXT NOT NULL,
    last_login TEXT
);

-- Passkey credentials.
--
-- `passkey_json` — сериализованный `webauthn_rs::prelude::Passkey`
-- в формате JSON. Содержит публичный ключ, credential_id, sign_count,
-- флаги BE/BS. webauthn-rs сам управляет его форматом.
CREATE TABLE IF NOT EXISTS passkeys (
    credential_id TEXT PRIMARY KEY,  -- base64url от rawId
    user_id TEXT NOT NULL,
    passkey_json TEXT NOT NULL,
    created_at TEXT NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_passkeys_user ON passkeys(user_id);

-- In-flight церемонии (registration + authentication).
--
-- `state_json` — сериализованное состояние `PasskeyRegistration`
-- или `PasskeyAuthentication`. Живёт короткое время (до 5 минут).
-- Удаляется после успешного finish'а.
CREATE TABLE IF NOT EXISTS ceremonies (
    session_id TEXT NOT NULL,
    kind TEXT NOT NULL,             -- 'register' | 'authenticate'
    state_json TEXT NOT NULL,
    created_at TEXT NOT NULL,
    PRIMARY KEY (session_id, kind)
);
```

---

## `webauthn-server-prod/src/main.rs`

```rust
//! Production-grade WebAuthn Server — companion 2 к главе 41.
//!
//! Отличия от первой версии:
//!   - webauthn-rs — вся серверная часть протокола;
//!   - ручной парсинг attestationObject через ciborium;
//!   - SQLite (sqlx) — реальное хранилище;
//!   - серверные сессии через tower-sessions.

mod attestation;
mod db;
mod error;
mod handlers;
mod state;

use std::sync::Arc;

use axum::http::{header, HeaderValue, Method};
use axum::{
    routing::post,
    Router,
};
use tokio::net::TcpListener;
use tower_http::cors::CorsLayer;
use tower_http::services::ServeDir;
use tower_http::set_header::SetResponseHeaderLayer;
use tower_sessions::{MemoryStore, SessionManagerLayer};
use url::Url;
use webauthn_rs::prelude::*;

use crate::state::AppState;

#[tokio::main]
async fn main() {
    let db_url = "sqlite:webauthn.db?mode=rwc";
    let pool = match db::init_pool(db_url).await {
        Ok(p) => p,
        Err(e) => {
            eprintln!("DB init failed: {e}");
            std::process::exit(1);
        }
    };

    let rp_id = "localhost";
    let rp_origin = Url::parse("http://localhost:3030")
        .expect("invalid rp_origin");

    let mut builder = WebauthnBuilder::new(rp_id, &rp_origin)
        .expect("invalid webauthn config");
    builder = builder.rp_name("Chapter 41.02 — Production WebAuthn");

    let webauthn = builder.build().expect("invalid webauthn");

    let app_state = Arc::new(AppState { pool, webauthn });

    let session_store = MemoryStore::default();
    let session_layer = SessionManagerLayer::new(session_store)
        .with_secure(false);

    // CORS: сервер использует cookies для сессий, поэтому
    // `allow_credentials(true)` обязателен. Но браузеры запрещают
    // `Access-Control-Allow-Headers: *` вместе с credentials.
    // Указываем origin и headers явно.
    let allowed_origin = HeaderValue::from_static("http://localhost:3030");
    let cors = CorsLayer::new()
        .allow_origin(allowed_origin)
        .allow_methods([Method::GET, Method::POST, Method::OPTIONS])
        .allow_headers([
            header::CONTENT_TYPE,
            header::COOKIE,
            header::AUTHORIZATION,
        ])
        .allow_credentials(true);

    // Отключаем кеширование для всей статики и API — иначе
    // браузер может отдавать старый index.html после правок.
    let no_cache = SetResponseHeaderLayer::overriding(
        header::CACHE_CONTROL,
        HeaderValue::from_static("no-store"),
    );

    let api = Router::new()
        .route("/api/register/begin", post(handlers::register_begin))
        .route("/api/register/finish", post(handlers::register_finish))
        .route("/api/login/begin", post(handlers::login_begin))
        .route("/api/login/finish", post(handlers::login_finish))
        .with_state(app_state);

    let app = Router::new()
        .merge(api)
        .fallback_service(ServeDir::new("static"))
        .layer(cors)
        .layer(no_cache)
        .layer(session_layer);

    let addr = "127.0.0.1:3030";
    let listener = TcpListener::bind(addr)
        .await
        .expect("не удалось занять порт 3030");

    println!("=========================================================");
    println!("  Production WebAuthn Server готов");
    println!("=========================================================");
    println!();
    println!("  Откройте: http://localhost:3030");
    println!();
    println!("  Отличия от первой версии:");
    println!("    • webauthn-rs — настоящая проверка attestation");
    println!("    • CBOR-парсинг attestationObject через ciborium");
    println!("    • Криптографическая проверка подписи assertion");
    println!("    • SQLite: webauthn.db");
    println!("    • Серверные сессии: state церемоний на сервере");
    println!();

    axum::serve(listener, app).await.expect("server failed");
}
```

---

## `webauthn-server-prod/src/state.rs`

```rust
//! Разделяемое состояние приложения.

use sqlx::sqlite::SqlitePool;
use webauthn_rs::prelude::Webauthn;

pub struct AppState {
    pub pool: SqlitePool,
    pub webauthn: Webauthn,
}
```

---

## `webauthn-server-prod/src/error.rs`

```rust
//! Единый тип ошибок для всех слоёв приложения.
//!
//! Оборачивает ошибки sqlx, webauthn-rs, сессий и CBOR-парсинга
//! в один `AppError`, который реализует `IntoResponse`.
//! Это позволяет обработчикам использовать `?` без ручного маппинга.

use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;
use serde_json::json;

#[derive(Debug)]
pub enum AppError {
    /// Ошибка уровня webauthn-rs (attestation/assertion не прошли проверку).
    Webauthn(String),
    /// Ошибка БД.
    Db(String),
    /// Сессия отсутствует или просрочена.
    NoSession,
    /// Пользователь не найден.
    UserNotFound,
    /// Прочее.
    Other(String),
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return match self {
            AppError::Webauthn(m) => write!(f, "webauthn: {m}"),
            AppError::Db(m) => write!(f, "db: {m}"),
            AppError::NoSession => write!(f, "no active session"),
            AppError::UserNotFound => write!(f, "user not found"),
            AppError::Other(m) => write!(f, "{m}"),
        };
    }
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match &self {
            AppError::Webauthn(m) => (StatusCode::BAD_REQUEST, m.clone()),
            AppError::Db(m) => (StatusCode::INTERNAL_SERVER_ERROR, m.clone()),
            AppError::NoSession => (StatusCode::UNAUTHORIZED, "no active session".to_string()),
            AppError::UserNotFound => (StatusCode::NOT_FOUND, "user not found".to_string()),
            AppError::Other(m) => (StatusCode::BAD_REQUEST, m.clone()),
        };
        return (status, Json(json!({ "error": msg }))).into_response();
    }
}

impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        return AppError::Db(e.to_string());
    }
}

impl From<webauthn_rs::prelude::WebauthnError> for AppError {
    fn from(e: webauthn_rs::prelude::WebauthnError) -> Self {
        return AppError::Webauthn(format!("{e:?}"));
    }
}
```

---

## `webauthn-server-prod/src/db.rs`

```rust
//! SQLite-хранилище через `sqlx`.
//!
//! Все функции принимают `&SqlitePool` и возвращают `Result<_, sqlx::Error>`.
//! Сериализация `Passkey` и состояний церемоний — через `serde_json`,
//! потому что webauthn-rs реализует `Serialize`/`Deserialize` для этих типов.

use chrono::Utc;
use sqlx::sqlite::{SqlitePool, SqlitePoolOptions};
use webauthn_rs::prelude::{Passkey, PasskeyAuthentication, PasskeyRegistration};

/// Открывает пул соединений и создаёт таблицы, если их нет.
pub async fn init_pool(url: &str) -> Result<SqlitePool, sqlx::Error> {
    let pool = SqlitePoolOptions::new()
        .max_connections(5)
        .connect(url)
        .await?;

    // Миграции — читаем файл и выполняем через `sqlx::query`.
    let schema = include_str!("../migrations/001_init.sql");
    for stmt in schema.split(";") {
        let stmt = stmt.trim();
        if stmt.is_empty() {
            continue;
        }
        sqlx::query(stmt).execute(&pool).await?;
    }

    return Ok(pool);
}

// ============================================================
//                    USERS
// ============================================================

pub struct UserRow {
    pub user_id: String,
    pub username: String,
    pub display_name: String,
    pub created_at: String,
    pub last_login: Option<String>,
}

/// Ищет пользователя по имени.
pub async fn find_user_by_name(
    pool: &SqlitePool,
    username: &str,
) -> Result<Option<UserRow>, sqlx::Error> {
    let row = sqlx::query_as::<_, (String, String, String, String, Option<String>)>(
        "SELECT user_id, username, display_name, created_at, last_login \
         FROM users WHERE username = ?",
    )
    .bind(username)
    .fetch_optional(pool)
    .await?;

    return Ok(row.map(|r| UserRow {
        user_id: r.0,
        username: r.1,
        display_name: r.2,
        created_at: r.3,
        last_login: r.4,
    }));
}

/// Ищет пользователя по user_id.
pub async fn find_user_by_id(
    pool: &SqlitePool,
    user_id: &str,
) -> Result<Option<UserRow>, sqlx::Error> {
    let row = sqlx::query_as::<_, (String, String, String, String, Option<String>)>(
        "SELECT user_id, username, display_name, created_at, last_login \
         FROM users WHERE user_id = ?",
    )
    .bind(user_id)
    .fetch_optional(pool)
    .await?;

    return Ok(row.map(|r| UserRow {
        user_id: r.0,
        username: r.1,
        display_name: r.2,
        created_at: r.3,
        last_login: r.4,
    }));
}

/// Создаёт пользователя. Идемпотентно (INSERT OR IGNORE).
pub async fn upsert_user(
    pool: &SqlitePool,
    user_id: &str,
    username: &str,
    display_name: &str,
) -> Result<(), sqlx::Error> {
    let now = Utc::now().to_rfc3339();
    sqlx::query(
        "INSERT OR IGNORE INTO users (user_id, username, display_name, created_at) \
         VALUES (?, ?, ?, ?)",
    )
    .bind(user_id)
    .bind(username)
    .bind(display_name)
    .bind(&now)
    .execute(pool)
    .await?;
    return Ok(());
}

pub async fn update_last_login(
    pool: &SqlitePool,
    user_id: &str,
) -> Result<(), sqlx::Error> {
    let now = Utc::now().to_rfc3339();
    sqlx::query("UPDATE users SET last_login = ? WHERE user_id = ?")
        .bind(&now)
        .bind(user_id)
        .execute(pool)
        .await?;
    return Ok(());
}

// ============================================================
//                    PASSKEYS
// ============================================================

pub async fn save_passkey(
    pool: &SqlitePool,
    user_id: &str,
    passkey: &Passkey,
) -> Result<(), sqlx::Error> {
    let json = serde_json::to_string(passkey)
        .map_err(|e| sqlx::Error::Decode(Box::new(e)))?;
    let cred_id = base64_url_encode(passkey.cred_id().as_ref());
    let now = Utc::now().to_rfc3339();
    sqlx::query(
        "INSERT OR REPLACE INTO passkeys (credential_id, user_id, passkey_json, created_at) \
         VALUES (?, ?, ?, ?)",
    )
    .bind(&cred_id)
    .bind(user_id)
    .bind(&json)
    .bind(&now)
    .execute(pool)
    .await?;
    return Ok(());
}

pub async fn get_passkeys_for_user(
    pool: &SqlitePool,
    user_id: &str,
) -> Result<Vec<Passkey>, sqlx::Error> {
    let rows = sqlx::query_as::<_, (String,)>(
        "SELECT passkey_json FROM passkeys WHERE user_id = ?",
    )
    .bind(user_id)
    .fetch_all(pool)
    .await?;

    let mut out = Vec::with_capacity(rows.len());
    for (json,) in rows {
        let pk: Passkey = serde_json::from_str(&json)
            .map_err(|e| sqlx::Error::Decode(Box::new(e)))?;
        out.push(pk);
    }
    return Ok(out);
}

pub async fn update_passkey(
    pool: &SqlitePool,
    passkey: &Passkey,
) -> Result<(), sqlx::Error> {
    let json = serde_json::to_string(passkey)
        .map_err(|e| sqlx::Error::Decode(Box::new(e)))?;
    let cred_id = base64_url_encode(passkey.cred_id().as_ref());
    sqlx::query("UPDATE passkeys SET passkey_json = ? WHERE credential_id = ?")
        .bind(&json)
        .bind(&cred_id)
        .execute(pool)
        .await?;
    return Ok(());
}

// ============================================================
//                    CEREMONIES
// ============================================================

pub async fn save_registration_state(
    pool: &SqlitePool,
    session_id: &str,
    state: &PasskeyRegistration,
) -> Result<(), sqlx::Error> {
    let json = serde_json::to_string(state)
        .map_err(|e| sqlx::Error::Decode(Box::new(e)))?;
    let now = Utc::now().to_rfc3339();
    sqlx::query(
        "INSERT OR REPLACE INTO ceremonies (session_id, kind, state_json, created_at) \
         VALUES (?, 'register', ?, ?)",
    )
    .bind(session_id)
    .bind(&json)
    .bind(&now)
    .execute(pool)
    .await?;
    return Ok(());
}

pub async fn save_auth_state(
    pool: &SqlitePool,
    session_id: &str,
    state: &PasskeyAuthentication,
) -> Result<(), sqlx::Error> {
    let json = serde_json::to_string(state)
        .map_err(|e| sqlx::Error::Decode(Box::new(e)))?;
    let now = Utc::now().to_rfc3339();
    sqlx::query(
        "INSERT OR REPLACE INTO ceremonies (session_id, kind, state_json, created_at) \
         VALUES (?, 'authenticate', ?, ?)",
    )
    .bind(session_id)
    .bind(&json)
    .bind(&now)
    .execute(pool)
    .await?;
    return Ok(());
}

pub async fn take_registration_state(
    pool: &SqlitePool,
    session_id: &str,
) -> Result<Option<PasskeyRegistration>, sqlx::Error> {
    return take_state_generic(pool, session_id, "register").await;
}

pub async fn take_auth_state(
    pool: &SqlitePool,
    session_id: &str,
) -> Result<Option<PasskeyAuthentication>, sqlx::Error> {
    return take_state_generic(pool, session_id, "authenticate").await;
}

async fn take_state_generic<T: serde::de::DeserializeOwned>(
    pool: &SqlitePool,
    session_id: &str,
    kind: &str,
) -> Result<Option<T>, sqlx::Error> {
    let row = sqlx::query_as::<_, (String,)>(
        "SELECT state_json FROM ceremonies WHERE session_id = ? AND kind = ?",
    )
    .bind(session_id)
    .bind(kind)
    .fetch_optional(pool)
    .await?;

    // Удаляем запись сразу — церемония одноразовая.
    sqlx::query("DELETE FROM ceremonies WHERE session_id = ? AND kind = ?")
        .bind(session_id)
        .bind(kind)
        .execute(pool)
        .await?;

    match row {
        Some((json,)) => {
            let state: T = serde_json::from_str(&json)
                .map_err(|e| sqlx::Error::Decode(Box::new(e)))?;
            return Ok(Some(state));
        }
        None => return Ok(None),
    }
}

// ============================================================
//                    HELPERS
// ============================================================

fn base64_url_encode(bytes: &[u8]) -> String {
    use base64::engine::general_purpose::URL_SAFE_NO_PAD;
    use base64::Engine;
    return URL_SAFE_NO_PAD.encode(bytes);
}
```

---

## `webauthn-server-prod/src/attestation.rs`

```rust
//! Ручной парсинг `attestationObject` через `ciborium`.
//!
//! Зачем это нужно, если `webauthn-rs` уже всё делает?
//!
//! `webauthn-rs::finish_passkey_registration` возвращает
//! `Passkey` — но это «чёрный ящик»: библиотека проверила attestation,
//! извлекла публичный ключ и упаковала его в свою структуру.
//!
//! Для обучения полезно **увидеть**, что лежит в `attestationObject`
//! и как оттуда достаётся публичный ключ. Этот модуль делает ту же
//! работу вручную, чтобы протокол перестал быть «магией».

use ciborium::Value;
use serde::Deserialize;

/// Структура `attestationObject` в CBOR.
///
/// W3C WebAuthn Level 2 §6.5.4 определяет три обязательных поля:
///   - `fmt`: string — формат attestation ("none", "packed", "tpm", ...);
///   - `attStmt`: map — attestation statement (зависит от fmt);
///   - `authData`: bytes — authenticatorData (37+ байт, содержит
///     rpIdHash, flags, signCount, attestedCredentialData).
#[derive(Debug, Deserialize)]
pub struct AttestationObject {
    pub fmt: String,
    #[serde(rename = "attStmt")]
    pub att_stmt: Value,
    #[serde(rename = "authData")]
    pub auth_data: Vec<u8>,
}

/// Разбирает `attestationObject` из сырых байт.
pub fn parse_attestation_object(bytes: &[u8]) -> Result<AttestationObject, String> {
    let att: AttestationObject = ciborium::from_reader(bytes)
        .map_err(|e| format!("CBOR parse failed: {e}"))?;
    return Ok(att);
}

/// Извлекает из `authData` блок `attestedCredentialData`.
///
/// Формат (W3C §6.5.2):
///   ```
///   rpIdHash          : 32 байта
///   flags             : 1 байт
///   signCount         : 4 байта (BE)
///   attestedCredentialData : переменный
///     aaguid          : 16 байт
///     credentialIdLength : 2 байта (BE)
///     credentialId    : N байт
///     credentialPublicKey : CBOR-encoded COSE_Key
///   ```
pub struct AttestedCredentialData<'a> {
    pub aaguid: [u8; 16],
    pub credential_id: &'a [u8],
    pub cose_public_key: &'a [u8],
}

pub fn extract_attested_credential_data(
    auth_data: &[u8],
) -> Result<AttestedCredentialData<'_>, String> {
    if auth_data.len() < 37 {
        return Err(format!("authData too short: {} < 37", auth_data.len()));
    }

    let flags = auth_data[32];
    if (flags & 0x40) == 0 {
        return Err("AT flag not set — no attested credential data".to_string());
    }

    let mut cursor = 37;

    // aaguid: 16 байт.
    if auth_data.len() < cursor + 16 {
        return Err("authData too short for aaguid".to_string());
    }
    let mut aaguid = [0u8; 16];
    aaguid.copy_from_slice(&auth_data[cursor..cursor + 16]);
    cursor += 16;

    // credentialIdLength: 2 байта BE.
    if auth_data.len() < cursor + 2 {
        return Err("authData too short for credentialIdLength".to_string());
    }
    let cred_id_len = u16::from_be_bytes([auth_data[cursor], auth_data[cursor + 1]]) as usize;
    cursor += 2;

    // credentialId: cred_id_len байт.
    if auth_data.len() < cursor + cred_id_len {
        return Err("authData too short for credentialId".to_string());
    }
    let credential_id = &auth_data[cursor..cursor + cred_id_len];
    cursor += cred_id_len;

    // credentialPublicKey: CBOR-encoded COSE_Key.
    let cose_public_key = &auth_data[cursor..];

    return Ok(AttestedCredentialData {
        aaguid,
        credential_id,
        cose_public_key,
    });
}

/// Разобранный `credentialPublicKey` — CBOR-encoded COSE_Key.
pub struct CoseKeySummary {
    pub kty: i64,
    pub alg: i64,
    pub crv: Option<i64>,
    pub coord_x_len: usize,
    pub coord_y_len: usize,
}

pub fn summarize_cose_key(cbor: &[u8]) -> Result<CoseKeySummary, String> {
    let value: Value = ciborium::from_reader(cbor)
        .map_err(|e| format!("COSE CBOR parse failed: {e}"))?;

    let map = value.as_map().ok_or("COSE key is not a map")?;

    let mut kty = 0i64;
    let mut alg = 0i64;
    let mut crv: Option<i64> = None;
    let mut x_len = 0;
    let mut y_len = 0;

    for (k, v) in map {
        let key_id = k.as_integer().and_then(|i| i.try_into().ok());
        match key_id {
            Some(1) => kty = v.as_integer().and_then(|i| i.try_into().ok()).unwrap_or(0),
            Some(3) => alg = v.as_integer().and_then(|i| i.try_into().ok()).unwrap_or(0),
            Some(-1) => crv = v.as_integer().and_then(|i| i.try_into().ok()),
            Some(-2) => x_len = v.as_bytes().map(|b| b.len()).unwrap_or(0),
            Some(-3) => y_len = v.as_bytes().map(|b| b.len()).unwrap_or(0),
            _ => {}
        }
    }

    return Ok(CoseKeySummary { kty, alg, crv, coord_x_len: x_len, coord_y_len: y_len });
}

/// Описывает `attStmt` — attestation statement.
///
/// Для `fmt = "none"` это пустая map. Для `fmt = "packed"` —
/// `{alg, sig}`. Для `fmt = "tpm"` — набор полей по TPM-спеке.
pub fn describe_att_stmt(att_stmt: &Value) -> String {
    let map = match att_stmt.as_map() {
        Some(m) => m,
        None => return "не map".to_string(),
    };
    if map.is_empty() {
        return "пустая map (для fmt=\"none\" это норма)".to_string();
    }
    let mut parts: Vec<String> = Vec::with_capacity(map.len());
    for (k, v) in map {
        let key = match k.as_text() {
            Some(s) => s.to_string(),
            None => format!("{k:?}"),
        };
        let kind = match v {
            Value::Bytes(b) => format!("bytes({})", b.len()),
            Value::Integer(_) => "int".to_string(),
            Value::Text(s) => format!("text({})", s.len()),
            Value::Bool(_) => "bool".to_string(),
            Value::Array(a) => format!("array({})", a.len()),
            Value::Map(m) => format!("map({})", m.len()),
            Value::Null => "null".to_string(),
            Value::Float(_) => "float".to_string(),
            Value::Tag(_, _) => "tag".to_string(),
            // `Value` помечен `#[non_exhaustive]` — компилятор
            // требует wildcard-arm даже после перечисления всех
            // текущих вариантов.
            _ => "unknown".to_string(),
        };
        parts.push(format!("{key}: {kind}"));
    }
    return parts.join(", ");
}
```

---

## `webauthn-server-prod/src/handlers.rs`

```rust
//! Эндпоинты WebAuthn-церемоний.
//!
//! Все криптографические проверки выполняет webauthn-rs.

use std::sync::Arc;

use axum::extract::State;
use axum::Json;
use serde::{Deserialize, Serialize};
use tower_sessions::Session;
use uuid::Uuid;
use webauthn_rs::prelude::*;

use crate::attestation;
use crate::db;
use crate::error::AppError;
use crate::state::AppState;

// ============================================================
//                    REQUEST / RESPONSE
// ============================================================

#[derive(Deserialize)]
pub struct RegisterBeginReq {
    pub username: String,
    pub display_name: String,
}

#[derive(Serialize)]
pub struct RegisterBeginResp {
    /// `CreationChallengeResponse` — сериализуется в JSON напрямую,
    /// клиент передаёт его в `navigator.credentials.create({ publicKey: ... })`.
    ///
    /// В webauthn-rs 0.5 этот тип заменил `PublicKeyCredentialCreationOptions`.
    pub options: CreationChallengeResponse,
    pub user_id: String,
}

#[derive(Deserialize)]
pub struct RegisterFinishReq {
    pub credential: RegisterPublicKeyCredential,
}

#[derive(Serialize)]
pub struct RegisterFinishResp {
    pub ok: bool,
    pub user_id: String,
    pub username: String,
    pub attestation: AttestationInfo,
    pub cose_key: CoseKeyInfo,
}

#[derive(Serialize)]
pub struct AttestationInfo {
    pub fmt: String,
    pub auth_data_len: usize,
    pub aaguid: String,
    pub credential_id_len: usize,
    pub cose_key_len: usize,
    /// Краткое описание attestation statement.
    pub att_stmt: String,
}

#[derive(Serialize)]
pub struct CoseKeyInfo {
    pub kty: i64,
    pub alg: i64,
    pub crv: Option<i64>,
    pub x_len: usize,
    pub y_len: usize,
}

#[derive(Deserialize)]
pub struct LoginBeginReq {
    pub username: String,
}

#[derive(Serialize)]
pub struct LoginBeginResp {
    /// `RequestChallengeResponse` — заменил `PublicKeyCredentialRequestOptions`
    /// в webauthn-rs 0.5.
    pub options: RequestChallengeResponse,
    pub user_id: String,
}

#[derive(Deserialize)]
pub struct LoginFinishReq {
    pub credential: PublicKeyCredential,
}

#[derive(Serialize)]
pub struct LoginFinishResp {
    pub ok: bool,
    pub user: UserProfile,
}

#[derive(Serialize)]
pub struct UserProfile {
    pub user_id: String,
    pub username: String,
    pub display_name: String,
    pub created_at: String,
    pub last_login: Option<String>,
    pub passkey_count: usize,
    pub credential_id: String,
    pub sign_count: u32,
}

// ============================================================
//                    SESSION HELPERS
// ============================================================

async fn session_id(session: &Session) -> Result<String, AppError> {
    let id: Option<String> = session.get("sid").await
        .map_err(|e| AppError::Other(format!("session get: {e}")))?;
    return id.ok_or(AppError::NoSession);
}

async fn set_session_id(session: &Session) -> Result<String, AppError> {
    let id = Uuid::new_v4().to_string();
    session.insert("sid", &id).await
        .map_err(|e| AppError::Other(format!("session insert: {e}")))?;
    return Ok(id);
}

// ============================================================
//                    REGISTRATION
// ============================================================

pub async fn register_begin(
    State(state): State<Arc<AppState>>,
    session: Session,
    Json(req): Json<RegisterBeginReq>,
) -> Result<Json<RegisterBeginResp>, AppError> {
    let existing = db::find_user_by_name(&state.pool, &req.username).await?;
    let (user_id, display_name) = match existing {
        Some(u) => (u.user_id, u.display_name),
        None => {
            let new_id = Uuid::new_v4();
            db::upsert_user(
                &state.pool,
                &new_id.to_string(),
                &req.username,
                &req.display_name,
            ).await?;
            (new_id.to_string(), req.display_name.clone())
        }
    };

    let existing_keys = db::get_passkeys_for_user(&state.pool, &user_id).await?;
    let exclude_credentials: Option<Vec<CredentialID>> = if existing_keys.is_empty() {
        None
    } else {
        Some(existing_keys.iter().map(|p| p.cred_id().clone()).collect())
    };

    let user_uuid = Uuid::parse_str(&user_id)
        .map_err(|e| AppError::Other(format!("invalid uuid: {e}")))?;

    // webauthn-rs: возвращает (CreationChallengeResponse, PasskeyRegistration).
    let (options, reg_state) = state.webauthn.start_passkey_registration(
        user_uuid,
        &req.username,
        &display_name,
        exclude_credentials,
    )?;

    let sid = match session_id(&session).await {
        Ok(s) => s,
        Err(_) => set_session_id(&session).await?,
    };
    session.insert("reg_user_id", &user_id).await
        .map_err(|e| AppError::Other(format!("session insert: {e}")))?;
    db::save_registration_state(&state.pool, &sid, &reg_state).await?;

    return Ok(Json(RegisterBeginResp { options, user_id }));
}

pub async fn register_finish(
    State(state): State<Arc<AppState>>,
    session: Session,
    Json(req): Json<RegisterFinishReq>,
) -> Result<Json<RegisterFinishResp>, AppError> {
    let sid = session_id(&session).await?;
    let user_id: String = session.get("reg_user_id").await
        .map_err(|e| AppError::Other(format!("session get: {e}")))?
        .ok_or(AppError::NoSession)?;

    let reg_state = db::take_registration_state(&state.pool, &sid).await?
        .ok_or_else(|| AppError::Other("registration state not found".to_string()))?;

    // === Демонстрация: ручной парсинг attestationObject через ciborium. ===
    let att_bytes = req.credential.response.attestation_object.as_ref();
    let att_info = match attestation::parse_attestation_object(att_bytes) {
        Ok(att) => {
            match attestation::extract_attested_credential_data(&att.auth_data) {
                Ok(acd) => {
                    let cose_info = attestation::summarize_cose_key(acd.cose_public_key);
                    let attestation_info = AttestationInfo {
                        fmt: att.fmt.clone(),
                        auth_data_len: att.auth_data.len(),
                        aaguid: hex_encode(&acd.aaguid),
                        credential_id_len: acd.credential_id.len(),
                        cose_key_len: acd.cose_public_key.len(),
                        att_stmt: attestation::describe_att_stmt(&att.att_stmt),
                    };
                    let cose_key_info = cose_info.map(|c| CoseKeyInfo {
                        kty: c.kty,
                        alg: c.alg,
                        crv: c.crv,
                        x_len: c.coord_x_len,
                        y_len: c.coord_y_len,
                    }).unwrap_or(CoseKeyInfo {
                        kty: 0, alg: 0, crv: None, x_len: 0, y_len: 0,
                    });
                    (attestation_info, cose_key_info)
                }
                Err(e) => return Err(AppError::Other(format!("extract ACD: {e}"))),
            }
        }
        Err(e) => return Err(AppError::Other(format!("parse attestationObject: {e}"))),
    };

    // === Настоящая проверка через webauthn-rs. ===
    // Эта функция:
    //   1. Парсит attestationObject через CBOR.
    //   2. Проверяет rpIdHash, origin, challenge.
    //   3. Извлекает публичный ключ.
    //   4. Проверяет attestation statement.
    //   5. Возвращает Passkey — готовую структуру для хранения.
    let passkey = state.webauthn.finish_passkey_registration(
        &req.credential,
        &reg_state,
    )?;

    db::save_passkey(&state.pool, &user_id, &passkey).await?;

    let user = db::find_user_by_id(&state.pool, &user_id).await?
        .ok_or(AppError::UserNotFound)?;

    return Ok(Json(RegisterFinishResp {
        ok: true,
        user_id,
        username: user.username,
        attestation: att_info.0,
        cose_key: att_info.1,
    }));
}

// ============================================================
//                    AUTHENTICATION
// ============================================================

pub async fn login_begin(
    State(state): State<Arc<AppState>>,
    session: Session,
    Json(req): Json<LoginBeginReq>,
) -> Result<Json<LoginBeginResp>, AppError> {
    let user = db::find_user_by_name(&state.pool, &req.username).await?
        .ok_or(AppError::UserNotFound)?;

    let passkeys = db::get_passkeys_for_user(&state.pool, &user.user_id).await?;
    if passkeys.is_empty() {
        return Err(AppError::Other("user has no passkeys".to_string()));
    }

    let (options, auth_state) = state.webauthn.start_passkey_authentication(&passkeys)?;

    let sid = match session_id(&session).await {
        Ok(s) => s,
        Err(_) => set_session_id(&session).await?,
    };
    session.insert("auth_user_id", &user.user_id).await
        .map_err(|e| AppError::Other(format!("session insert: {e}")))?;
    db::save_auth_state(&state.pool, &sid, &auth_state).await?;

    return Ok(Json(LoginBeginResp {
        options,
        user_id: user.user_id,
    }));
}

pub async fn login_finish(
    State(state): State<Arc<AppState>>,
    session: Session,
    Json(req): Json<LoginFinishReq>,
) -> Result<Json<LoginFinishResp>, AppError> {
    let sid = session_id(&session).await?;
    let user_id: String = session.get("auth_user_id").await
        .map_err(|e| AppError::Other(format!("session get: {e}")))?
        .ok_or(AppError::NoSession)?;

    let auth_state = db::take_auth_state(&state.pool, &sid).await?
        .ok_or_else(|| AppError::Other("auth state not found".to_string()))?;

    // === Настоящая криптографическая проверка подписи. ===
    let auth_result = state.webauthn.finish_passkey_authentication(
        &req.credential,
        &auth_state,
    )?;

    // sign_count берём из результата — метода `Passkey::counter()`
    // в webauthn-rs 0.5 нет.
    let new_sign_count = auth_result.counter();

    // Обновляем Passkey: sign_count и backup_state изменились.
    let mut passkeys = db::get_passkeys_for_user(&state.pool, &user_id).await?;
    for pk in passkeys.iter_mut() {
        pk.update_credential(&auth_result);
        db::update_passkey(&state.pool, pk).await?;
    }

    db::update_last_login(&state.pool, &user_id).await?;

    let user = db::find_user_by_id(&state.pool, &user_id).await?
        .ok_or(AppError::UserNotFound)?;
    let passkey_count = passkeys.len();
    let cred_id = match passkeys.first() {
        Some(p) => base64_url_encode(p.cred_id().as_ref()),
        None => "—".to_string(),
    };

    return Ok(Json(LoginFinishResp {
        ok: true,
        user: UserProfile {
            user_id: user.user_id,
            username: user.username,
            display_name: user.display_name,
            created_at: user.created_at,
            last_login: user.last_login,
            passkey_count,
            credential_id: cred_id,
            sign_count: new_sign_count,
        },
    }));
}

// ============================================================
//                    HELPERS
// ============================================================

fn hex_encode(bytes: &[u8]) -> String {
    let mut out = String::with_capacity(bytes.len() * 2);
    for b in bytes {
        out.push_str(&format!("{b:02x}"));
    }
    return out;
}

fn base64_url_encode(bytes: &[u8]) -> String {
    use base64::engine::general_purpose::URL_SAFE_NO_PAD;
    use base64::Engine;
    return URL_SAFE_NO_PAD.encode(bytes);
}
```

---

## `webauthn-server-prod/static/index.html`

```html
<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8">
<title>Production WebAuthn — Глава 41.02</title>
<style>
  body { font-family: system-ui; max-width: 900px; margin: 2rem auto; padding: 0 1rem; color: #222; }
  h1 { margin: 0 0 0.25rem; }
  .sub { color: #666; margin-bottom: 1.5rem; }
  .sub code { background: #f0f0f0; padding: 1px 4px; border-radius: 3px; }
  .panel { border: 1px solid #ddd; border-radius: 8px; padding: 1rem; margin-bottom: 1rem; }
  .panel h2 { margin: 0 0 0.6rem; font-size: 1rem; }
  .panel.highlight { border: 2px solid #9b59b6; }
  .panel.info { border: 2px solid #16a085; }
  input { padding: 0.4rem 0.6rem; font-size: 1rem; border: 1px solid #ccc;
          border-radius: 4px; width: 100%; box-sizing: border-box; margin-bottom: 0.4rem; }
  button { padding: 0.45rem 1rem; font-size: 0.95rem; border: 1px solid #999;
           border-radius: 4px; background: #f7f7f7; cursor: pointer; margin-right: 0.5rem; }
  button:hover:not(:disabled) { background: #eee; }
  button:disabled { opacity: 0.5; cursor: not-allowed; }
  pre { background: #f7f7f7; border: 1px solid #e0e0e0; padding: 0.6rem 0.8rem;
        border-radius: 4px; font-size: 0.8rem; overflow-x: auto; white-space: pre-wrap;
        max-height: 400px; overflow-y: auto; }
  .note { font-size: 0.85rem; color: #666; margin: 0.5rem 0; }
  @media (prefers-color-scheme: dark) {
    body { background: #121212; color: #e6e6e6; }
    .panel { border-color: #333; background: #1e1e1e; }
    input { background: #1e1e1e; color: #e6e6e6; border-color: #444; }
    button { background: #262626; color: #e6e6e6; border-color: #444; }
    button:hover:not(:disabled) { background: #333; }
    pre { background: #1a1a1a; color: #9ed09e; border-color: #333; }
    .sub code { background: #262626; }
  }
</style>
</head>
<body>
<h1>Production WebAuthn — Глава 41.02</h1>
<p class="sub">Сервер на Rust: <code>webauthn-rs</code> + <code>ciborium</code> +
   <code>sqlx</code> (SQLite) + <code>tower-sessions</code>.
   Открывайте <b>http://localhost:3030</b>.</p>

<div class="panel highlight">
  <h2>1. Регистрация passkey</h2>
  <p class="note">Настоящая проверка attestation через webauthn-rs.
     Плюс ручной парсинг <code>attestationObject</code> через ciborium —
     покажем, что внутри: <code>fmt</code>, <code>aaguid</code>,
     <code>credentialIdLength</code>, COSE-ключ.</p>
  <input id="reg-username" placeholder="Логин" value="bob">
  <input id="reg-display" placeholder="Отображаемое имя" value="Bob">
  <button id="reg-btn">🔑 Зарегистрировать passkey</button>
  <pre id="reg-out">—</pre>
</div>

<div class="panel highlight">
  <h2>2. Вход с passkey</h2>
  <p class="note">Криптографическая проверка подписи через
     <code>webauthn-rs::finish_passkey_authentication</code>.
     Если подпись неверна — сервер вернёт ошибку, а не <code>ok: true</code>.</p>
  <input id="login-username" placeholder="Логин" value="bob">
  <button id="login-btn">🔓 Войти</button>
  <pre id="login-out">—</pre>
</div>

<div class="panel info">
  <h2>3. Что нового по сравнению с главой 41.01</h2>
  <ul style="font-size: 0.9rem; line-height: 1.6;">
    <li><b>Attestation парсится через CBOR</b> — в ответе видно <code>fmt</code>, <code>aaguid</code>, длину credential_id, сводку COSE-ключа.</li>
    <li><b>Подпись assertion проверяется криптографически</b> — если подделать <code>signature</code>, сервер отклонит вход.</li>
    <li><b>Пользователи и passkeys в SQLite</b> — файл <code>webauthn.db</code> переживёт перезапуск сервера.</li>
    <li><b>Сессии серверные</b> — состояние церемонии живёт в БД (<code>ceremonies</code>), а не в cookie.</li>
  </ul>
</div>

<script type="module">
// ─── base64url <-> bytes ─────────────────────────────────────
function b64uToBytes(s) {
  if (typeof s !== 'string') {
    throw new Error('b64uToBytes: ожидалась строка, получено ' + typeof s);
  }
  s = s.replace(/-/g, '+').replace(/_/g, '/');
  while (s.length % 4) s += '=';
  const bin = atob(s);
  const bytes = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i);
  return bytes;
}
function bytesToB64u(bytes) {
  let bin = '';
  for (const b of bytes) bin += String.fromCharCode(b);
  return btoa(bin).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

// ─── Регистрация ─────────────────────────────────────────────
document.getElementById('reg-btn').addEventListener('click', async () => {
  const out = document.getElementById('reg-out');
  const username = document.getElementById('reg-username').value.trim();
  const displayName = document.getElementById('reg-display').value.trim();
  if (!username) { out.textContent = 'введите логин'; return; }
  out.textContent = 'шаг 1/2: получаем challenge…';

  try {
    const beginResp = await fetch('/api/register/begin', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'same-origin',
      body: JSON.stringify({ username, display_name: displayName }),
    });
    if (!beginResp.ok) {
      const err = await beginResp.json();
      out.textContent = '❌ ' + JSON.stringify(err);
      return;
    }
    const { options } = await beginResp.json();

    // webauthn-rs оборачивает options в { publicKey: {...} },
    // как того требует спецификация WebAuthn Level 2.
    const publicKey = options.publicKey;

    publicKey.challenge = b64uToBytes(publicKey.challenge);
    publicKey.user.id = b64uToBytes(publicKey.user.id);
    if (publicKey.excludeCredentials) {
      publicKey.excludeCredentials = publicKey.excludeCredentials.map(c => ({
        ...c, id: b64uToBytes(c.id),
      }));
    }

    out.textContent = 'шаг 2/2: ждём подтверждения от аутентификатора…';

    const credential = await navigator.credentials.create({ publicKey });

    const finishBody = {
      credential: {
        id: credential.id,
        rawId: bytesToB64u(new Uint8Array(credential.rawId)),
        type: credential.type,
        response: {
          clientDataJSON: bytesToB64u(new Uint8Array(credential.response.clientDataJSON)),
          attestationObject: bytesToB64u(new Uint8Array(credential.response.attestationObject)),
          transports: credential.response.getTransports ? credential.response.getTransports() : [],
        },
      },
    };

    const finishResp = await fetch('/api/register/finish', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'same-origin',
      body: JSON.stringify(finishBody),
    });
    const result = await finishResp.json();
    out.textContent = JSON.stringify(result, null, 2);
  } catch (e) {
    out.textContent = '❌ ' + e.message;
  }
});

// ─── Вход ────────────────────────────────────────────────────
document.getElementById('login-btn').addEventListener('click', async () => {
  const out = document.getElementById('login-out');
  const username = document.getElementById('login-username').value.trim();
  if (!username) { out.textContent = 'введите логин'; return; }
  out.textContent = 'шаг 1/2: получаем challenge…';

  try {
    const beginResp = await fetch('/api/login/begin', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'same-origin',
      body: JSON.stringify({ username }),
    });
    if (!beginResp.ok) {
      const err = await beginResp.json();
      out.textContent = '❌ ' + JSON.stringify(err);
      return;
    }
    const { options } = await beginResp.json();
    const publicKey = options.publicKey;
    publicKey.challenge = b64uToBytes(publicKey.challenge);
    if (publicKey.allowCredentials) {
      publicKey.allowCredentials = publicKey.allowCredentials.map(c => ({
        ...c, id: b64uToBytes(c.id),
      }));
    }

    out.textContent = 'шаг 2/2: ждём подтверждения…';

    const assertion = await navigator.credentials.get({ publicKey });

    const finishBody = {
      credential: {
        id: assertion.id,
        rawId: bytesToB64u(new Uint8Array(assertion.rawId)),
        type: assertion.type,
        response: {
          clientDataJSON: bytesToB64u(new Uint8Array(assertion.response.clientDataJSON)),
          authenticatorData: bytesToB64u(new Uint8Array(assertion.response.authenticatorData)),
          signature: bytesToB64u(new Uint8Array(assertion.response.signature)),
          userHandle: assertion.response.userHandle
            ? bytesToB64u(new Uint8Array(assertion.response.userHandle))
            : null,
        },
      },
    };

    const finishResp = await fetch('/api/login/finish', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'same-origin',
      body: JSON.stringify(finishBody),
    });
    const result = await finishResp.json();
    out.textContent = JSON.stringify(result, null, 2);
  } catch (e) {
    out.textContent = '❌ ' + e.message;
  }
});
</script>
</body>
</html>
```

---

## `webauthn-server-prod/README.md`

```markdown
# Production WebAuthn Server — companion 2 к главе 41

Настоящая серверная реализация WebAuthn с полным набором проверок.

## Отличия от первой версии (Глава 41.01)

| Аспект | 41.01 | 41.02 |
|---|---|---|
| Attestation | Не парсится | `webauthn-rs` + `ciborium` |
| Подпись assertion | Не проверяется | Криптографическая проверка |
| Публичный ключ | Не извлекается | Извлекается и хранится |
| Пользователи | В памяти | SQLite |
| Состояние церемонии | В памяти | SQLite + серверные сессии |

## Запуск

cd webauthn-server-prod
cargo run

Откройте **http://localhost:3030** — не `127.0.0.1`.

Первый запуск создаст `webauthn.db` в текущей директории.

## Что делает сервер

1. POST /api/register/begin — генерирует options через
   `start_passkey_registration`, сохраняет `PasskeyRegistration`
   в БД.
2. POST /api/register/finish — вызывает
   `finish_passkey_registration`, которая парсит attestationObject,
   проверяет rpIdHash/origin/challenge, извлекает публичный ключ.
   Плюс вручную парсит attestation через ciborium и показывает
   содержимое (fmt, aaguid, длину credential_id, COSE-ключ, attStmt).
3. POST /api/login/begin — генерирует options через
   `start_passkey_authentication`.
4. POST /api/login/finish — вызывает `finish_passkey_authentication`,
   которая криптографически проверяет подпись, обновляет sign_count
   и возвращает профиль.

## Что НЕ упрощено

Все ключевые криптографические проверки выполняет `webauthn-rs`:
- парсинг CBOR (через `ciborium` внутри);
- проверка подписи assertion;
- хранение публичного ключа (COSE_Key);
- привязка state церемонии к серверной сессии.

## Схема БД

- `users(user_id, username, display_name, created_at, last_login)`
- `passkeys(credential_id, user_id, passkey_json, created_at)`
- `ceremonies(session_id, kind, state_json, created_at)`

## Что можно улучшить дальше

- SQLite-backed sessions вместо `MemoryStore` — сессии переживут перезапуск.
- HTTPS + Secure cookies — `with_secure(true)`.
- Attestation verification policy — требовать конкретный `fmt`.
- AAGUID whitelist.
- Rate limiting на /register/begin и /login/begin.
- Config из переменных окружения (`RP_ID`, `RP_ORIGIN`, `DATABASE_URL`, `LISTEN_ADDR`, `COOKIE_SECURE`).
```

---

## Что демонстрируется в браузере

### Регистрация

В ответе `POST /api/register/finish` появляется диагностика:

```json
{
  "ok": true,
  "user_id": "4666d059-8b97-40f9-a8c3-fb5fca218626",
  "username": "bob",
  "attestation": {
    "fmt": "none",
    "auth_data_len": 148,
    "aaguid": "ea0b8dd64d011d23ce4b6b48c5b575d4",
    "credential_id_len": 16,
    "cose_key_len": 77,
    "att_stmt": "пустая map (для fmt=\"none\" это норма)"
  },
  "cose_key": {
    "kty": 2,
    "alg": -7,
    "crv": 1,
    "x_len": 32,
    "y_len": 32
  }
}
```

Разбор:

- **`fmt: "none"`** — Chrome с `attestation: "none"` не вкладывает attestation statement.
- **`aaguid: "ea0b8dd6..."`** — идентификатор модели аутентификатора. Это **реальный** AAGUID платформенного аутентификатора Windows Hello.
- **`cose_key: kty=2, alg=-7, crv=1`** — стандарт COSE для ES256: `kty=2` (EC2), `alg=-7` (ES256), `crv=1` (P-256).
- **`x_len=32, y_len=32`** — публичный ключ P-256 это две координаты по 32 байта.
- **`att_stmt: "пустая map"`** — для `fmt="none"` это правильно: attestation statement отсутствует.

Это **тот самый ключ**, которым сервер будет проверять подпись assertion.

### Вход

Ответ `POST /api/login/finish` возвращает профиль:

```json
{
  "ok": true,
  "user": {
    "user_id": "...",
    "username": "bob",
    "display_name": "Bob",
    "created_at": "2026-10-10T08:29:52.522563000+00:00",
    "last_login": "2026-10-10T08:57:47.763019560+00:00",
    "passkey_count": 1,
    "credential_id": "tx6x0zW7gHg_ID3c_6PIw",
    "sign_count": 0
  }
}
```

`sign_count: 0` — платформенные аутентификаторы Windows Hello обычно не увеличивают счётчик, он остаётся нулевым. Это допустимо: спека разрешает `signCount == 0` для platform authenticators, анти-клонирование работает через другие механизмы (backup state, BE/BS).

Если подменить `signature` в `POST /api/login/finish` на любые байты — сервер вернёт **HTTP 400** с ошибкой от `webauthn-rs`, а не `ok: true`. Это доказывает, что подпись реально проверяется.

### Перезапуск сервера

`webauthn.db` сохраняется. После `Ctrl+C` и `cargo run` `bob` всё ещё может войти — данные пережили рестарт.

---

## Команды запуска

```bash
cd webauthn-server-prod

# Сборка (первый раз компилируется долго из-за sqlx)
cargo build

# Запуск
cargo run

# Очистка БД
rm webauthn.db
```

Открыть `http://localhost:3030` (не `127.0.0.1`).

---

## Итог

Этот companion показывает **настоящее** WebAuthn-решение. Отличия от главы 41.01 — принципиальные:

1. **`webauthn-rs` вместо ручного парсинга.** Библиотека реализует всю серверную часть спецификации: CBOR-парсинг attestationObject, извлечение публичного ключа, криптографическую проверку подписи assertion, обновление sign_count. Это то, что **не следует писать руками** — спецификация W3C WebAuthn Level 2 содержит более 200 страниц и десятки крайних случаев.

2. **`ciborium` для демонстрации.** Модуль `attestation.rs` показывает, что лежит внутри attestationObject: `fmt`, `aaguid`, `credentialIdLength`, `COSE_Key`, `attStmt`. Это «просвет» в чёрный ящик — полезно для отладки и обучения.

3. **SQLite для хранения.** `Passkey` сериализуется в JSON и сохраняется в `passkeys.passkey_json`. После перезапуска сервера пользователи остаются.

4. **Серверные сессии.** `PasskeyRegistration` и `PasskeyAuthentication` живут в БД и привязаны к `session_id` через `tower-sessions`. Cookie содержит только `session_id` — state церемонии недоступен клиенту.

Грабли, которые встретились в процессе сборки, — типичны для любого production WebAuthn-проекта на Rust:

1. **Типы переименованы** — `CreationChallengeResponse` вместо `PublicKeyCredentialCreationOptions`.
2. **Feature `danger-allow-state-serialisation`** — без неё нельзя сохранить state церемонии в БД.
3. **`AuthenticationResult::counter()`** — метода `Passkey::counter()` не существует.
4. **`ciborium::Value` не-exhaustive** — нужен wildcard в match.
5. **CORS с credentials** — нельзя `headers: *`, нужно явно.
6. **Options обёрнуты в `{ publicKey }`** — по W3C-спеке, клиент должен это учитывать.
7. **Кеш браузера** — `Cache-Control: no-store` решает проблему навсегда.

Все эти вещи не «мелочи реализации», а обязательные детали протокола и библиотек. Знание их — разница между демо, которое работает на одной машине, и сервисом, который работает в проде.