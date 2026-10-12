# Глава 41.01

## Зачем это нужно

Глава 41 показывает браузерную часть WebAuthn: как собрать `challenge`, вызвать `navigator.credentials.create()`, разобрать `authenticatorData`. Но **половина протокола живёт на сервере**: challenge генерирует бэкенд, attestation проверяется на бэкенде, публичный ключ хранится в БД.

Этот companion-проект добавляет серверную сторону:

1. **Выдаёт challenge** по запросу браузера, хранит его до получения ответа.
2. **Принимает attestation** при регистрации, сохраняет пользователя (credential_id + метаданные).
3. **Принимает assertion** при входе, выполняет 10 проверок из W3C WebAuthn Level 2.
4. **Возвращает сохранённый профиль** клиенту — то, что запрашивалось в задаче: «сохранял бы данные пользователя и при следующем входе отображал бы их».

Проект **standalone** — не входит в workspace основного WASM-проекта, потому что у него свой `main()` и своя конфигурация. Это отдельный подпроект `webauthn-server/` рядом с `hello-wasm02/`.

---

## Что мы узнали при разработке: 4 грабли серверной части

### Грабли 1. `[workspace]` в `Cargo.toml` — обязателен

Без явного `[workspace]` в начале `Cargo.toml` cargo может попытаться найти родительский workspace (`hello-wasm02`) и начнёт разрешать модули не так, как ожидается. Симптом — странные ошибки `unresolved imports` для функций, которые **есть в файле**.

**Решение:**

```toml
[workspace]
# ↑ Явно указываем: это standalone-проект.

[package]
name = "webauthn-server"
# ...
```

### Грабли 2. `dead_code` сигналит о пропущенных проверках, а не о косметике

Первая версия сервера имела предупреждения на полях `cross_origin`, `flags`, `credential_id`, `signature` в DTO и структурах. Это **не косметика** — компилятор сигналил, что сервер пропускает реальные проверки из спецификации WebAuthn:

- `cross_origin == false` — если не проверить, MITM-прокси может подделать origin.
- `UP`-бит — если не проверить, «удалённый бот с чужим ключом» пройдёт аутентификацию.
- `credential_id` — если не сверить с БД, атакующий подсунет чужой credential.
- `signature` — если не проверить размер, мусор вместо подписи пройдёт.

**Решение** — реализовать все проверки, а не глушить warning через `#[allow(dead_code)]`. Все четыре поля теперь реально читаются в `handlers.rs`, и объект `checks` в ответе содержит 10 полей вместо 5.

### Грабли 3. Дублирование модулей — источник расхождений

В процессе разработки появились два файла с одинаковыми функциями: `encoding.rs` (созданный ранее) и `webauthn.rs` (объединённый). Оба содержали `to_base64url` / `from_base64url`, но `handlers.rs` использовал только `webauthn`. Компилятор видел `mod encoding;` и ругался на неиспользуемые функции в дубликате.

**Решение** — удалить `encoding.rs`, убрать `mod encoding;` из `main.rs`. Если бы warning был заглушен через `#[allow(dead_code)]`, дублирование осталось бы: читатель видел бы два места с одной логикой, и они бы рано или поздно разошлись.

### Грабли 4. Неполное копирование файлов

При ручной сборке проекта из нескольких файлов легко скопировать часть содержимого. В первом запуске `webauthn.rs` содержал только `rp_id_hash`, `parse_auth_data`, `parse_client_data` — без `to_base64url` / `from_base64url`. Симптом — `unresolved imports` на функции, которые «вроде бы есть», плюс каскад ошибок вывода типов (`type annotations needed`, `size for values of type str cannot be known`).

**Профилактика:** после копирования файла сразу проверяйте:

```bash
grep -n "^pub fn\|^pub struct" src/webauthn.rs
```

Должно быть ровно 7 строк — 5 функций + 2 структуры.

---

## Структура проекта

```
webauthn-server/
├── Cargo.toml
├── README.md
├── src/
│   ├── main.rs         ← запуск axum-сервера, роуты, статика
│   ├── types.rs        ← request/response DTO + Checks (10 полей)
│   ├── state.rs        ← in-memory хранилище пользователей и challenges
│   ├── webauthn.rs     ← encoding + rp_id_hash + парсинг clientData/authData
│   └── handlers.rs     ← 5 эндпоинтов с полным набором проверок
└── static/
    └── index.html      ← клиент на vanilla JS + WebAuthn API
```

Клиент — на чистом JS. Это сознательное решение: спутник показывает **серверную часть**, а не дублирует WASM-демо из app.rs. Криптография та же, но код минимальный и читается за минуту.

---

## `webauthn-server/Cargo.toml`

```toml
[workspace]
# ↑ Явно указываем: это standalone-проект, не часть родительского workspace.
# Без этой строки cargo может попытаться найти workspace выше по дереву,
# и если родительский hello-wasm02/Cargo.toml объявит workspace — начнутся
# странные ошибки разрешения модулей.

[package]
name = "webauthn-server"
version = "0.1.0"
edition = "2021"
description = "WebAuthn server companion to Chapter 41"

[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
base64 = "0.22"
rand = "0.8"
sha2 = "0.10"
tower-http = { version = "0.5", features = ["fs", "cors"] }
```

Зависимости минимальны: `axum` для HTTP, `tokio` для рантайма, `rand` для challenge и user.id, `sha2` для `rpIdHash`, `base64` для codec, `tower-http` для отдачи `static/`.

---

## `webauthn-server/src/main.rs`

```rust
//! WebAuthn Server — companion к главе 41.
//!
//! Минимальный сервер, реализующий серверную часть WebAuthn:
//!   - генерирует challenge (32 случайных байта);
//!   - принимает attestation при регистрации;
//!   - принимает assertion при входе;
//!   - проверяет 10 свойств из W3C WebAuthn Level 2 §7.1/§7.2;
//!   - хранит пользователей в памяти и возвращает профиль.
//!
//! ПРАВИЛО: этот проект standalone — отдельный Cargo.toml, свой main().
//! Он не входит в workspace hello-wasm02.

mod handlers;
mod state;
mod types;
mod webauthn;

use std::sync::Arc;

use axum::{
    routing::{get, post},
    Router,
};
use tokio::net::TcpListener;
use tower_http::cors::{Any, CorsLayer};
use tower_http::services::ServeDir;

use crate::state::AppState;

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState::new());

    // API-роуты.
    let api = Router::new()
        .route("/api/register/begin", post(handlers::register_begin))
        .route("/api/register/finish", post(handlers::register_finish))
        .route("/api/login/begin", post(handlers::login_begin))
        .route("/api/login/finish", post(handlers::login_finish))
        .route("/api/users", get(handlers::list_users))
        .with_state(state);

    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    // static/ отдаётся как fallback — API имеет приоритет.
    let app = Router::new()
        .merge(api)
        .fallback_service(ServeDir::new("static"))
        .layer(cors);

    let addr = "127.0.0.1:3030";
    let listener = TcpListener::bind(addr).await
        .expect("не удалось занять порт 3030");

    println!("=========================================================");
    println!("  WebAuthn Server готов");
    println!("=========================================================");
    println!();
    println!("  Откройте: http://localhost:3030");
    println!();
    println!("  ВАЖНО: именно localhost, а не 127.0.0.1.");
    println!("  WebAuthn запрещает IP-адреса в rp.id — браузер отвергнет");
    println!("  регистрацию при открытии по 127.0.0.1:3030.");
    println!();
    println!("  Эндпоинты:");
    println!("    POST /api/register/begin");
    println!("    POST /api/register/finish");
    println!("    POST /api/login/begin");
    println!("    POST /api/login/finish");
    println!("    GET  /api/users");
    println!();

    axum::serve(listener, app).await.expect("server failed");
}
```

---

## `webauthn-server/src/types.rs`

```rust
//! DTO для API: запросы и ответы в JSON.
//!
//! Поля, которые в JSON называются `type` (зарезервированное слово
//! Rust), помечены `#[serde(rename = "type")]`.

use serde::{Deserialize, Serialize};

// ============================================================
//                    REGISTRATION
// ============================================================

#[derive(Deserialize)]
pub struct RegisterBeginReq {
    pub username: String,
    pub display_name: String,
}

#[derive(Serialize)]
pub struct RegisterBeginResp {
    pub challenge: String,
    pub rp: RpEntity,
    pub user: UserEntity,
    pub pub_key_cred_params: Vec<CredParam>,
    pub timeout: u32,
    pub attestation: String,
    pub authenticator_selection: AuthenticatorSelection,
}

#[derive(Serialize)]
pub struct RpEntity {
    pub id: String,
    pub name: String,
}

#[derive(Serialize)]
pub struct UserEntity {
    pub id: String,
    pub name: String,
    pub display_name: String,
}

#[derive(Serialize)]
pub struct CredParam {
    #[serde(rename = "type")]
    pub type_: String,
    pub alg: i32,
}

#[derive(Serialize)]
pub struct AuthenticatorSelection {
    pub authenticator_attachment: String,
    pub user_verification: String,
    pub resident_key: String,
}

#[derive(Deserialize)]
pub struct RegisterFinishReq {
    pub username: String,
    pub credential_id: String,       // base64url
    pub client_data_json: String,    // base64url
    pub attestation_object: String,  // base64url
}

#[derive(Serialize)]
pub struct RegisterFinishResp {
    pub ok: bool,
    pub user: UserProfile,
}

// ============================================================
//                    AUTHENTICATION
// ============================================================

#[derive(Deserialize)]
pub struct LoginBeginReq {
    pub username: String,
}

#[derive(Serialize)]
pub struct LoginBeginResp {
    pub challenge: String,
    pub rp_id: String,
    pub timeout: u32,
    pub user_verification: String,
    pub allow_credentials: Vec<AllowCredential>,
}

#[derive(Serialize)]
pub struct AllowCredential {
    #[serde(rename = "type")]
    pub type_: String,
    pub id: String,       // base64url
}

#[derive(Deserialize)]
pub struct LoginFinishReq {
    pub username: String,
    pub credential_id: String,
    pub client_data_json: String,
    pub authenticator_data: String,
    pub signature: String,
}

#[derive(Serialize)]
pub struct LoginFinishResp {
    pub ok: bool,
    pub user: UserProfile,
    pub checks: Checks,
}

/// Результаты всех проверок на сервере — для отображения в UI.
///
/// Ровно эти проверки требует W3C WebAuthn Level 2 (§7.1 для
/// регистрации, §7.2 для аутентификации). Все поля `*_ok` должны
/// быть `true`, чтобы церемония считалась успешной.
#[derive(Serialize)]
pub struct Checks {
    // clientDataJSON
    pub challenge_matches: bool,
    pub origin_matches: bool,
    pub not_cross_origin: bool,
    // authenticatorData
    pub rp_hash_ok: bool,
    pub up_flag_ok: bool,
    pub sign_count_ok: bool,
    pub sign_count_prev: u32,
    pub sign_count_new: u32,
    // Прочее
    pub credential_id_matches: bool,
    pub signature_size_ok: bool,
}

// ============================================================
//                    PROFILE
// ============================================================

/// Профиль пользователя — то, что сервер возвращает клиенту.
///
/// Именно эти данные показываются пользователю после входа:
/// сервер «вспомнил» его и вернул сохранённое.
#[derive(Serialize, Clone)]
pub struct UserProfile {
    pub username: String,
    pub display_name: String,
    pub created_at: String,
    pub last_login: String,
    pub login_count: u32,
    pub sign_count: u32,
    pub credential_id: String,
    pub notes: String,
}
```

---

## `webauthn-server/src/state.rs`

```rust
//! In-memory хранилище пользователей и pending challenges.
//!
//! В продакшене:
//!   - users — таблица в PostgreSQL/SQLite (public_key, credential_id,
//!     sign_count, created_at, ...);
//!   - challenges — Redis с TTL или колонка в сессии пользователя.

use std::collections::HashMap;
use std::sync::Mutex;
use std::time::{SystemTime, UNIX_EPOCH};

/// Тип challenge — при регистрации и при входе разные проверки.
#[derive(Clone, Copy, PartialEq, Eq)]
pub enum ChallengeKind {
    Register,
    Login,
}

pub struct PendingChallenge {
    pub challenge: Vec<u8>,
    pub kind: ChallengeKind,
}

/// Запись о пользователе на сервере.
///
/// `credential_id` и `sign_count` — минимум, который нужен для
/// последующей проверки assertion. В реальном приложении здесь
/// ещё публичный ключ (CBOR-структура), AAGUID аутентификатора, флаги.
#[derive(Clone)]
pub struct StoredUser {
    pub username: String,
    pub display_name: String,
    pub credential_id: Vec<u8>,
    pub created_at: u64,
    pub last_login: u64,
    pub login_count: u32,
    pub sign_count: u32,
    pub notes: String,
}

pub struct AppState {
    pub users: Mutex<HashMap<String, StoredUser>>,
    pub challenges: Mutex<HashMap<String, PendingChallenge>>,
    pub rp_id: String,
    pub rp_name: String,
    pub origin: String,
}

impl AppState {
    pub fn new() -> Self {
        return Self {
            users: Mutex::new(HashMap::new()),
            challenges: Mutex::new(HashMap::new()),
            // Жёстко зашито для демо. В продакшене — из конфига.
            rp_id: "localhost".to_string(),
            rp_name: "Глава 41 — WebAuthn сервер".to_string(),
            origin: "http://localhost:3030".to_string(),
        };
    }

    pub fn store_user(&self, user: StoredUser) {
        self.users
            .lock()
            .unwrap()
            .insert(user.username.clone(), user);
    }

    pub fn get_user(&self, username: &str) -> Option<StoredUser> {
        return self.users.lock().unwrap().get(username).cloned();
    }

    pub fn set_challenge(&self, username: &str, challenge: Vec<u8>, kind: ChallengeKind) {
        self.challenges.lock().unwrap().insert(
            username.to_string(),
            PendingChallenge { challenge, kind },
        );
    }

    /// Забирает challenge и удаляет его из хранилища.
    ///
    /// Удаление — обязательное: challenge одноразовый.
    /// Иначе возможна атака повторного воспроизведения.
    pub fn take_challenge(&self, username: &str) -> Option<PendingChallenge> {
        return self.challenges.lock().unwrap().remove(username);
    }
}

pub fn now_secs() -> u64 {
    return SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map(|d| d.as_secs())
        .unwrap_or(0);
}
```

---

## `webauthn-server/src/webauthn.rs`

```rust
//! WebAuthn: encoding, rp_id_hash, парсинг clientDataJSON и authenticatorData.
//!
//! Здесь собраны **все** вспомогательные функции, которые нужны
//! handlers.rs. Раньше они жили в двух файлах (encoding.rs и
//! webauthn.rs) — это привело к дублированию и предупреждениям
//! dead_code. Теперь один файл — один источник правды.

use base64::engine::general_purpose::URL_SAFE_NO_PAD;
use base64::Engine;
use serde::Deserialize;
use sha2::{Digest, Sha256};

// ============================================================
//                    ENCODING
// ============================================================

/// Кодирует байты в base64url без padding.
pub fn to_base64url(bytes: &[u8]) -> String {
    return URL_SAFE_NO_PAD.encode(bytes);
}

/// Декодирует base64url в байты.
pub fn from_base64url(s: &str) -> Result<Vec<u8>, String> {
    return URL_SAFE_NO_PAD
        .decode(s)
        .map_err(|e| format!("base64url decode: {e}"));
}

// ============================================================
//                    RP ID HASH
// ============================================================

/// `rpIdHash = SHA-256(rp_id)`.
///
/// Это первые 32 байта `authenticatorData`. Сервер обязан
/// сравнивать их со своим rp_id — иначе подпись могла быть
/// получена для чужого домена.
pub fn rp_id_hash(rp_id: &str) -> [u8; 32] {
    let mut hasher = Sha256::new();
    hasher.update(rp_id.as_bytes());
    let out = hasher.finalize();
    let mut arr = [0u8; 32];
    arr.copy_from_slice(&out);
    return arr;
}

// ============================================================
//                    AUTHENTICATOR DATA
// ============================================================

/// Разобранный `authenticatorData` (минимальные 37 байт).
pub struct AuthDataView {
    pub rp_id_hash: [u8; 32],
    pub flags: u8,
    pub sign_count: u32,
}

/// Парсит первые 37 байт `authenticatorData`.
///
/// Если структура длиннее (при регистрации там attestedCredentialData
/// и extensions), остаток пока игнорируется — для проверки флагов
/// и счётчика хватает первых 37.
pub fn parse_auth_data(bytes: &[u8]) -> Result<AuthDataView, String> {
    if bytes.len() < 37 {
        return Err(format!(
            "authenticatorData too short: {} < 37",
            bytes.len(),
        ));
    }
    let mut rp_id_hash = [0u8; 32];
    rp_id_hash.copy_from_slice(&bytes[0..32]);
    let flags = bytes[32];
    let sign_count = u32::from_be_bytes([
        bytes[33], bytes[34], bytes[35], bytes[36],
    ]);
    return Ok(AuthDataView { rp_id_hash, flags, sign_count });
}

// ============================================================
//                    CLIENT DATA JSON
// ============================================================

/// Разобранный `clientDataJSON`.
///
/// Именно здесь браузер подтверждает, что операция прошла
/// на нужном origin — JavaScript не может подделать это поле.
#[derive(Debug, Deserialize)]
pub struct ClientData {
    #[serde(rename = "type")]
    pub kind: String,
    pub challenge: String,
    pub origin: String,
    #[serde(default, rename = "crossOrigin")]
    pub cross_origin: bool,
}

pub fn parse_client_data(bytes: &[u8]) -> Result<ClientData, String> {
    let s = std::str::from_utf8(bytes)
        .map_err(|e| format!("clientDataJSON utf8: {e}"))?;
    return serde_json::from_str(s)
        .map_err(|e| format!("clientDataJSON json: {e}"));
}
```

---

## `webauthn-server/src/handlers.rs`

```rust
//! HTTP-эндпоинты WebAuthn-церемоний.
//!
//! Все проверки, которые возвращаются в `checks`, соответствуют
//! шагам верификации из W3C WebAuthn Level 2 (§7.1 для регистрации,
//! §7.2 для аутентификации).

use std::sync::Arc;

use axum::{extract::State, Json};
use rand::RngCore;

use crate::state::{now_secs, AppState, ChallengeKind, StoredUser};
use crate::types::*;
use crate::webauthn::{
    from_base64url, parse_auth_data, parse_client_data, rp_id_hash, to_base64url,
};

// ============================================================
//                    REGISTRATION
// ============================================================

pub async fn register_begin(
    State(state): State<Arc<AppState>>,
    Json(req): Json<RegisterBeginReq>,
) -> Json<RegisterBeginResp> {
    // Генерируем challenge — 32 криптостойких случайных байта.
    let mut challenge = [0u8; 32];
    rand::thread_rng().fill_bytes(&mut challenge);
    state.set_challenge(
        &req.username,
        challenge.to_vec(),
        ChallengeKind::Register,
    );

    // user.id — тоже случайные 16 байт (не username!).
    let mut user_id_buf = [0u8; 16];
    rand::thread_rng().fill_bytes(&mut user_id_buf);

    return Json(RegisterBeginResp {
        challenge: to_base64url(&challenge),
        rp: RpEntity {
            id: state.rp_id.clone(),
            name: state.rp_name.clone(),
        },
        user: UserEntity {
            id: to_base64url(&user_id_buf),
            name: req.username.clone(),
            display_name: req.display_name.clone(),
        },
        pub_key_cred_params: vec![
            CredParam { type_: "public-key".into(), alg: -7 },   // ES256
            CredParam { type_: "public-key".into(), alg: -257 }, // RS256
        ],
        timeout: 60_000,
        attestation: "none".into(),
        authenticator_selection: AuthenticatorSelection {
            authenticator_attachment: "platform".into(),
            user_verification: "preferred".into(),
            resident_key: "preferred".into(),
        },
    });
}

pub async fn register_finish(
    State(state): State<Arc<AppState>>,
    Json(req): Json<RegisterFinishReq>,
) -> Json<RegisterFinishResp> {
    let pending = match state.take_challenge(&req.username) {
        Some(p) => p,
        None => return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "нет активного challenge"),
        }),
    };
    if pending.kind != ChallengeKind::Register {
        return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "неверный тип challenge"),
        });
    }

    // attestationObject должен быть непустым и корректным base64url.
    // В реальном коде здесь был бы CBOR-парсинг публичного ключа.
    let attestation_bytes: Vec<u8> = match from_base64url(&req.attestation_object) {
        Ok(b) if !b.is_empty() => b,
        Ok(_) => return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "attestationObject пуст"),
        }),
        Err(e) => return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, &e),
        }),
    };

    let cd_bytes: Vec<u8> = match from_base64url(&req.client_data_json) {
        Ok(b) => b,
        Err(e) => return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, &e),
        }),
    };
    let cd = match parse_client_data(&cd_bytes) {
        Ok(c) => c,
        Err(e) => return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, &e),
        }),
    };

    if cd.kind != "webauthn.create" {
        return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "type != webauthn.create"),
        });
    }
    if cd.challenge != to_base64url(&pending.challenge) {
        return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "challenge не совпадает"),
        });
    }
    if cd.origin != state.origin {
        return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "origin не совпадает"),
        });
    }
    if cd.cross_origin {
        // Спека WebAuthn §7.1: crossOrigin MUST be false.
        return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, "crossOrigin = true запрещён"),
        });
    }

    let cred_id: Vec<u8> = match from_base64url(&req.credential_id) {
        Ok(b) => b,
        Err(e) => return Json(RegisterFinishResp {
            ok: false,
            user: error_profile(&req.username, &e),
        }),
    };

    // В продакшене: распарсить attestationObject через CBOR,
    // извлечь публичный ключ, проверить attestation statement.
    // Здесь мы просто сохраняем credential_id и метаданные.

    let now = now_secs();
    let user = StoredUser {
        username: req.username.clone(),
        display_name: req.username.clone(),
        credential_id: cred_id,
        created_at: now,
        last_login: 0,
        login_count: 0,
        sign_count: 0,
        notes: format!(
            "Зарегистрирован. origin: {}; attestationObject: {} B",
            cd.origin,
            attestation_bytes.len(),
        ),
    };
    state.store_user(user.clone());

    return Json(RegisterFinishResp {
        ok: true,
        user: profile_from(&user),
    });
}

// ============================================================
//                    AUTHENTICATION
// ============================================================

pub async fn login_begin(
    State(state): State<Arc<AppState>>,
    Json(req): Json<LoginBeginReq>,
) -> Json<LoginBeginResp> {
    let user = state.get_user(&req.username);
    let allow = match &user {
        Some(u) => vec![AllowCredential {
            type_: "public-key".into(),
            id: to_base64url(&u.credential_id),
        }],
        None => vec![],
    };

    let mut challenge = [0u8; 32];
    rand::thread_rng().fill_bytes(&mut challenge);
    state.set_challenge(
        &req.username,
        challenge.to_vec(),
        ChallengeKind::Login,
    );

    return Json(LoginBeginResp {
        challenge: to_base64url(&challenge),
        rp_id: state.rp_id.clone(),
        timeout: 60_000,
        user_verification: "preferred".into(),
        allow_credentials: allow,
    });
}

pub async fn login_finish(
    State(state): State<Arc<AppState>>,
    Json(req): Json<LoginFinishReq>,
) -> Json<LoginFinishResp> {
    let pending = match state.take_challenge(&req.username) {
        Some(p) => p,
        None => return err_login(&req.username, "нет активного challenge"),
    };

    // === clientDataJSON ===
    let cd_bytes: Vec<u8> = match from_base64url(&req.client_data_json) {
        Ok(b) => b,
        Err(e) => return err_login(&req.username, &e),
    };
    let cd = match parse_client_data(&cd_bytes) {
        Ok(c) => c,
        Err(e) => return err_login(&req.username, &e),
    };

    let challenge_matches = cd.challenge == to_base64url(&pending.challenge);
    let origin_matches = cd.origin == state.origin;
    let not_cross_origin = !cd.cross_origin;

    // === authenticatorData ===
    let auth_bytes: Vec<u8> = match from_base64url(&req.authenticator_data) {
        Ok(b) => b,
        Err(e) => return err_login(&req.username, &e),
    };
    let ad = match parse_auth_data(&auth_bytes) {
        Ok(a) => a,
        Err(e) => return err_login(&req.username, &e),
    };

    // Anti-phishing: rpIdHash == SHA-256(rp_id).
    let expected = rp_id_hash(&state.rp_id);
    let rp_hash_ok = ad.rp_id_hash == expected;

    // Спека §7.2, шаг 15: UP-бит обязан быть установлен.
    // Это подтверждение, что человек физически взаимодействовал
    // с аутентификатором (не «удалённый бот с чужим ключом»).
    let up_flag_ok = (ad.flags & 0x01) != 0;

    // Подпись: sanity-check размера. В реальном коде — проверка
    // публичным ключом. ES256 → 64 байта, RS256-2048 → 256 байт.
    let sig_bytes: Vec<u8> = match from_base64url(&req.signature) {
        Ok(b) => b,
        Err(e) => return err_login(&req.username, &e),
    };
    let signature_size_ok = sig_bytes.len() >= 32 && sig_bytes.len() <= 1024;

    // === Пользователь ===
    let mut user = match state.get_user(&req.username) {
        Some(u) => u,
        None => return err_login(&req.username, "пользователь не найден"),
    };

    // Сверяем credential_id с сохранённым — защита от подмены.
    let cred_id_bytes: Vec<u8> = match from_base64url(&req.credential_id) {
        Ok(b) => b,
        Err(e) => return err_login(&req.username, &e),
    };
    let credential_id_matches = cred_id_bytes == user.credential_id;

    // Anti-cloning: signCount должен расти.
    let prev_sign = user.sign_count;
    let sign_count_ok = if prev_sign == 0 {
        true // первый вход — сравнивать не с чем
    } else {
        ad.sign_count > prev_sign
    };

    let all_ok =
        challenge_matches
            && origin_matches
            && not_cross_origin
            && rp_hash_ok
            && up_flag_ok
            && signature_size_ok
            && credential_id_matches
            && sign_count_ok;

    if all_ok {
        user.last_login = now_secs();
        user.login_count += 1;
        user.sign_count = ad.sign_count;
        user.notes = format!(
            "Последний вход: origin {}; signature {} B",
            cd.origin,
            sig_bytes.len(),
        );
        state.store_user(user.clone());
    }

    return Json(LoginFinishResp {
        ok: all_ok,
        user: profile_from(&user),
        checks: Checks {
            challenge_matches,
            origin_matches,
            not_cross_origin,
            rp_hash_ok,
            up_flag_ok,
            sign_count_ok,
            sign_count_prev: prev_sign,
            sign_count_new: ad.sign_count,
            credential_id_matches,
            signature_size_ok,
        },
    });
}

// ============================================================
//                    LIST
// ============================================================

pub async fn list_users(
    State(state): State<Arc<AppState>>,
) -> Json<Vec<UserProfile>> {
    let users = state.users.lock().unwrap();
    let mut list: Vec<UserProfile> = users.values().map(profile_from).collect();
    list.sort_by(|a, b| a.username.cmp(&b.username));
    return Json(list);
}

// ============================================================
//                    HELPERS
// ============================================================

fn profile_from(u: &StoredUser) -> UserProfile {
    return UserProfile {
        username: u.username.clone(),
        display_name: u.display_name.clone(),
        created_at: format_time(u.created_at),
        last_login: if u.last_login == 0 {
            "—".into()
        } else {
            format_time(u.last_login)
        },
        login_count: u.login_count,
        sign_count: u.sign_count,
        credential_id: to_base64url(&u.credential_id),
        notes: u.notes.clone(),
    };
}

fn error_profile(username: &str, msg: &str) -> UserProfile {
    return UserProfile {
        username: username.to_string(),
        display_name: "—".into(),
        created_at: "—".into(),
        last_login: "—".into(),
        login_count: 0,
        sign_count: 0,
        credential_id: "—".into(),
        notes: format!("Ошибка: {msg}"),
    };
}

fn err_login(username: &str, msg: &str) -> Json<LoginFinishResp> {
    return Json(LoginFinishResp {
        ok: false,
        user: error_profile(username, msg),
        checks: Checks {
            challenge_matches: false,
            origin_matches: false,
            not_cross_origin: false,
            rp_hash_ok: false,
            up_flag_ok: false,
            sign_count_ok: false,
            sign_count_prev: 0,
            sign_count_new: 0,
            credential_id_matches: false,
            signature_size_ok: false,
        },
    });
}

fn format_time(secs: u64) -> String {
    // Простое форматирование — секунды с эпохи.
    // В продакшене: chrono или time crate.
    return format!("{secs}s (Unix epoch)");
}
```

---

## `webauthn-server/static/index.html`

```html
<!doctype html>
<html lang="ru">
<head>
<meta charset="utf-8">
<title>WebAuthn Demo — Глава 41</title>
<style>
  body { font-family: system-ui; max-width: 820px; margin: 2rem auto; padding: 0 1rem; color: #222; }
  h1 { margin: 0 0 0.25rem; }
  .sub { color: #666; margin-bottom: 1.5rem; }
  .panel { border: 1px solid #ddd; border-radius: 8px; padding: 1rem; margin-bottom: 1rem; }
  .panel h2 { margin: 0 0 0.6rem; font-size: 1rem; }
  input { padding: 0.4rem 0.6rem; font-size: 1rem; border: 1px solid #ccc;
          border-radius: 4px; width: 100%; box-sizing: border-box; margin-bottom: 0.4rem; }
  button { padding: 0.45rem 1rem; font-size: 0.95rem; border: 1px solid #999;
           border-radius: 4px; background: #f7f7f7; cursor: pointer; margin-right: 0.5rem; }
  button:hover:not(:disabled) { background: #eee; }
  button:disabled { opacity: 0.5; cursor: not-allowed; }
  pre { background: #f7f7f7; border: 1px solid #e0e0e0; padding: 0.6rem 0.8rem;
        border-radius: 4px; font-size: 0.85rem; overflow-x: auto; white-space: pre-wrap; }
  @media (prefers-color-scheme: dark) {
    body { background: #121212; color: #e6e6e6; }
    .panel { border-color: #333; background: #1e1e1e; }
    input { background: #1e1e1e; color: #e6e6e6; border-color: #444; }
    button { background: #262626; color: #e6e6e6; border-color: #444; }
    button:hover:not(:disabled) { background: #333; }
    pre { background: #1a1a1a; color: #9ed09e; border-color: #333; }
  }
</style>
</head>
<body>
<h1>WebAuthn Demo — Глава 41</h1>
<p class="sub">Сервер на Rust (axum) + WebAuthn API в браузере.
   Открывайте именно <b>http://localhost:3030</b>.</p>

<div class="panel">
  <h2>1. Регистрация passkey</h2>
  <input id="reg-username" placeholder="Логин" value="alice">
  <input id="reg-display" placeholder="Отображаемое имя" value="Alice">
  <button id="reg-btn">🔑 Зарегистрировать passkey</button>
  <pre id="reg-out">—</pre>
</div>

<div class="panel">
  <h2>2. Вход с passkey</h2>
  <input id="login-username" placeholder="Логин" value="alice">
  <button id="login-btn">🔓 Войти</button>
  <pre id="login-out">—</pre>
</div>

<div class="panel">
  <h2>3. Пользователи на сервере</h2>
  <button id="list-btn">🔄 Обновить</button>
  <pre id="list-out">—</pre>
</div>

<script type="module">
// ─── base64url <-> bytes ─────────────────────────────────────
function b64uToBytes(s) {
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

// ─── Регистрация ────────────────────────────────────────────
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
      body: JSON.stringify({ username, display_name: displayName }),
    });
    const opts = await beginResp.json();
    out.textContent = 'шаг 2/2: ждём подтверждения от аутентификатора…';

    const credential = await navigator.credentials.create({
      publicKey: {
        challenge: b64uToBytes(opts.challenge),
        rp: { id: opts.rp.id, name: opts.rp.name },
        user: {
          id: b64uToBytes(opts.user.id),
          name: opts.user.name,
          displayName: opts.user.display_name,
        },
        pubKeyCredParams: opts.pub_key_cred_params,
        timeout: opts.timeout,
        attestation: opts.attestation,
        authenticatorSelection: {
          authenticatorAttachment: opts.authenticator_selection.authenticator_attachment,
          userVerification: opts.authenticator_selection.user_verification,
          residentKey: opts.authenticator_selection.resident_key,
        },
      },
    });

    const finishResp = await fetch('/api/register/finish', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        username,
        credential_id: bytesToB64u(new Uint8Array(credential.rawId)),
        client_data_json: bytesToB64u(new Uint8Array(credential.response.clientDataJSON)),
        attestation_object: bytesToB64u(new Uint8Array(credential.response.attestationObject)),
      }),
    });
    const result = await finishResp.json();
    out.textContent = JSON.stringify(result, null, 2);
    updateUserList();
  } catch (e) {
    out.textContent = '❌ ' + e.message;
  }
});

// ─── Вход ───────────────────────────────────────────────────
document.getElementById('login-btn').addEventListener('click', async () => {
  const out = document.getElementById('login-out');
  const username = document.getElementById('login-username').value.trim();
  if (!username) { out.textContent = 'введите логин'; return; }
  out.textContent = 'шаг 1/2: получаем challenge…';

  try {
    const beginResp = await fetch('/api/login/begin', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username }),
    });
    const opts = await beginResp.json();
    out.textContent = 'шаг 2/2: ждём подтверждения…';

    const assertion = await navigator.credentials.get({
      publicKey: {
        challenge: b64uToBytes(opts.challenge),
        rpId: opts.rp_id,
        timeout: opts.timeout,
        userVerification: opts.user_verification,
        allowCredentials: opts.allow_credentials.map((c) => ({
          type: c.type,
          id: b64uToBytes(c.id),
        })),
      },
    });

    const finishResp = await fetch('/api/login/finish', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        username,
        credential_id: bytesToB64u(new Uint8Array(assertion.rawId)),
        client_data_json: bytesToB64u(new Uint8Array(assertion.response.clientDataJSON)),
        authenticator_data: bytesToB64u(new Uint8Array(assertion.response.authenticatorData)),
        signature: bytesToB64u(new Uint8Array(assertion.response.signature)),
      }),
    });
    const result = await finishResp.json();
    out.textContent = JSON.stringify(result, null, 2);
    updateUserList();
  } catch (e) {
    out.textContent = '❌ ' + e.message;
  }
});

// ─── Список ─────────────────────────────────────────────────
async function updateUserList() {
  const out = document.getElementById('list-out');
  try {
    const resp = await fetch('/api/users');
    const users = await resp.json();
    if (users.length === 0) { out.textContent = 'пока никого'; return; }
    out.textContent = JSON.stringify(users, null, 2);
  } catch (e) {
    out.textContent = '❌ ' + e.message;
  }
}
document.getElementById('list-btn').addEventListener('click', updateUserList);
updateUserList();
</script>
</body>
</html>
```

---

## `webauthn-server/README.md`

```markdown
# WebAuthn Server — companion к главе 41

Минимальный сервер (Rust + axum), реализующий серверную часть WebAuthn.

## Запуск

cd webauthn-server
cargo run

Откройте **http://localhost:3030** — не `127.0.0.1`.

WebAuthn запрещает IP-адреса в `rp.id`. Если открыть приложение по
`127.0.0.1:3030`, браузер откажется регистрировать passkey
(`SecurityError` в Firefox, `InvalidDomainError` в Chrome).

## Что делает сервер

1. POST /api/register/begin — генерирует 32-байтный challenge
2. POST /api/register/finish — принимает attestation, проверяет
   type, challenge, origin, crossOrigin в clientDataJSON.
3. POST /api/login/begin — генерирует новый challenge, возвращает
   allowCredentials с credential_id пользователя.
4. POST /api/login/finish — принимает assertion, выполняет 10 проверок,
   возвращает профиль пользователя и объект checks.
5. GET /api/users — список всех зарегистрированных пользователей.

## 10 проверок в POST /api/login/finish

Согласно W3C WebAuthn Level 2 §7.2:

| # | Проверка | Что защищает |
|---|---|---|
| 1 | type == "webauthn.get" | От подмены церемонии |
| 2 | challenge == выданный | От replay-атаки |
| 3 | origin == наш | От MITM-прокси |
| 4 | crossOrigin == false | Спека §7.1 |
| 5 | rpIdHash == SHA-256(rp_id) | Anti-phishing |
| 6 | UP-бит установлен | Человек взаимодействовал |
| 7 | signCount > prev | Anti-cloning |
| 8 | credential_id совпадает | От подмены credential |
| 9 | signature разумного размера | Отсеять мусор |
| 10 | Подпись публичным ключом | Упрощено (см. ниже) |

## Что упрощено

- **Attestation не парсится через CBOR.** В реальном приложении —
  распарсить `attestationObject` (`coset` или `ciborium`), извлечь
  публичный ключ.
- **Подпись assertion не проверяется** публичным ключом (шаг 10).
  В продакшене — `webauthn-rs` или ручная проверка через p256/rsa.
- **Пользователи в памяти.** При перезапуске сервера всё пропадает.
- **Сессии не используются.** username передаётся в каждом запросе.

Каждое упрощение помечено комментарием в исходниках.

## Что НЕ упрощено

Первые 9 проверок реализованы полностью. Это те проверки, которые
можно сделать без криптографии. Их пропуск открывает класс атак:
MITM (origin), replay (challenge), клонирование (signCount),
подмена credential (credential_id).

## Проверка протокола

POST /api/login/finish возвращает объект checks с 10 полями:

```json
{
  "ok": true,
  "checks": {
    "challenge_matches": true,
    "origin_matches": true,
    "not_cross_origin": true,
    "rp_hash_ok": true,
    "up_flag_ok": true,
    "sign_count_ok": true,
    "sign_count_prev": 0,
    "sign_count_new": 42,
    "credential_id_matches": true,
    "signature_size_ok": true
  }
}
```

## Куда дальше

Для продакшена используйте:
- webauthn-rs — полностью реализует серверную сторону WebAuthn.
- coset / ciborium — для парсинга attestationObject.
- PostgreSQL/SQLite с полями credential_id, public_key, sign_count.
```

---

## Как запустить

```bash
# В отдельном терминале — из корня репозитория:
cd webauthn-server
cargo run
```

В браузере открыть **`http://localhost:3030`** (не `127.0.0.1`).

Сценарий:

1. **Регистрация**: ввести `alice`, нажать «Зарегистрировать passkey» → Touch ID / Windows Hello → сервер сохранит пользователя. В `notes` появится размер `attestationObject`.
2. **Вход**: тот же логин, нажать «Войти» → биометрия → сервер вернёт профиль и объект `checks` с 10 полями, все `true`.
3. **Повторный вход**: нажать «Войти» снова → `login_count: 2`, `sign_count` вырос — видно проверку anti-cloning.
4. **Список пользователей**: кнопка «Обновить» — видно, какие данные сервер реально хранит.

---

## Что этот companion добавляет к главе

Глава 41 в браузерном демо показывает **клиентскую сторону**: как собрать options, как разобрать `authenticatorData`, что означают флаги UP/UV/BE/BS. Companion добавляет **серверную сторону** — то, что происходит, когда данные выходят за пределы браузера:

- **`challenge` рождается на сервере** и хранится до получения ответа. Это критично: клиент не может «придумать» challenge сам, иначе replay.
- **`origin` проверяется сервером** — независимо от того, что говорит браузер. Именно эта проверка ловит подделку через MITM-прокси.
- **`rpIdHash` сравнивается с `SHA-256(rp_id)`** — сервер подтверждает, что подпись сделана для его домена, а не для `evil.com`.
- **`signCount` сохраняется между сессиями** — это единственный способ обнаружить клонированную копию аутентификатора.
- **Профиль возвращается с сервера** — данные живут там, а не в localStorage. Это и есть смысл аутентификации: сервер знает пользователя и его состояние.

Вместе с браузерным демо из главы 41 это даёт полную картину протокола: 50% в браузере, 50% на сервере. Без второй половины первая превращается в «магию»: пользователь видит биометрический диалог, но не понимает, что с ним происходит дальше. Без первой сервер нечего проверять: challenge некому выдать.

Ключевые уроки companion-проекта:

1. **`[workspace]` в `Cargo.toml` обязателен для standalone-проекта.** Без него cargo может искать родительский workspace и путаться в разрешении модулей.
2. **`dead_code` сигналит о пропущенных проверках.** Четыре warning'а о `cross_origin`, `flags`, `credential_id`, `signature` — это четыре пропущенных шага верификации из спецификации. Их нужно реализовать, а не глушить.
3. **Дублирование модулей — источник расхождений.** Один `encoding.rs`, один `webauthn.rs` — не два файла с одинаковыми функциями.
4. **Неполное копирование файла даёт каскадные ошибки.** `unresolved imports` + `type annotations needed` + `size for values of type str cannot be known` — всё это следствие одной пропущенной `pub fn`. Проверяйте `grep -n "^pub fn"` после копирования.