# Часть XI. Security и Cryptography

## Глава 41. Web Authentication

---

## Введение

WebAuthn — стандарт W3C для аутентификации без пароля. Вместо «секрета, который знает пользователь» (пароль) или «секрета, который знает сервер» (хеш пароля), WebAuthn использует **пару ключей, привязанную к домену**: приватный ключ живёт в аппаратном аутентификаторе (Touch ID, Windows Hello, YubiKey), публичный — на сервере.

Для Rust-разработчика WebAuthn — это **граница между мирами**. Браузер сам управляет криптографией, а WASM-код работает с высокоуровневым протоколом: генерирует `challenge`, собирает JSON-опции, разбирает `attestationObject` и `authenticatorData`. Именно на этой границе — `Rust ↔ browser authentication boundary` — и происходит всё интересное.

Ключевые отличия WebAuthn от других глав:

| Аспект | Web Crypto (гл. 38) | WebAuthn (гл. 41) |
|---|---|---|
| Что даёт | Примитивы (AES, RSA, ECDSA) | Готовый протокол аутентификации |
| Кто управляет ключом | JavaScript | Аппаратный аутентификатор |
| Защита от фишинга | Нет | Встроена (origin привязан к ключу) |
| User gesture | Не нужен | Обязателен |
| Приватный ключ | В памяти JS | Никогда не покидает устройство |

В этой главе мы покроем 7 тем:

1. `CredentialsContainer` — точка входа
2. `PublicKeyCredential` — результат операции
3. WebAuthn — create/get ceremonies
4. passkeys — платформенные аутентификаторы
5. authenticator — Touch ID / Hello / YubiKey
6. challenge / response — ядро протокола
7. Rust ↔ browser authentication boundary

---

## 1. Что мы узнали при сборке: 5 грабель WebAuthn

Прежде чем перейти к коду, зафиксируем сложности, которые выявились при разработке. Они неочевидны, и их стоит знать до начала работы.

### Грабли 1. `web-sys` не экспортирует WebAuthn-типы без флага

Попытка использовать типизированные обёртки `web_sys::CredentialsContainer`, `CredentialCreationOptions`, `CredentialRequestOptions` даёт ошибку `cannot find type ... in crate web_sys`. Эти типы помечены в `web-sys` как **unstable** и требуют глобального флага `--cfg=web_sys_unstable_apis`.

Установка флага — плохое решение: он влияет на весь проект, включая стабильные модули, и может привести к неожиданным поломкам при обновлении `web-sys`.

**Решение:** вызывать JS-методы через `js_sys::Reflect::get` + `js_sys::Function::call*`. Это универсальный способ работы с любым JS API — тот же приём мы использовали в главе 18.15 для File System Access API.

```rust
let credentials = js_sys::Reflect::get(
    &navigator,
    &JsValue::from_str("credentials"),
)?;
let create_fn: js_sys::Function = js_sys::Reflect::get(
    &credentials,
    &JsValue::from_str("create"),
)?.dyn_into()?;
let promise = create_fn.call1(&credentials, &options)?;
```

### Грабли 2. `rp.id` не может быть IP-адресом

Если открыть приложение по `http://127.0.0.1:5173`, браузер отвергнет создание passkey:

- Firefox: `SecurityError: The operation is insecure`
- Chrome: `InvalidDomainError: This is an invalid domain`

W3C WebAuthn Level 2 §5.1.3 явно запрещает IP-адреса в `rp.id`. Единственное исключение для локальной разработки — `localhost`.

**Решение:** нормализовать `rp_id` в `localhost`, если hostname — IP. И предупредить пользователя.

```rust
let raw_host = location.hostname().unwrap_or_else(|_| "localhost".to_string());
let rp_id = if is_ip_address(&raw_host) {
    "localhost".to_string()
} else {
    raw_host.clone()
};
```

При этом **сам origin страницы остаётся `127.0.0.1`** — браузер продолжит блокировать операцию. Поэтому важно не только нормализовать `rp_id`, но и показать пользователю прямую ссылку на `localhost`.

### Грабли 3. `navigator.credentials` может отсутствовать

В старых браузерах или в iframe без разрешения `navigator.credentials === undefined`. Простая проверка перед любым вызовом:

```rust
if credentials.is_undefined() || credentials.is_null() {
    return Err(WebAuthnError::new(
        WebAuthnErrorKind::Unsupported,
        "navigator.credentials unavailable".to_string(),
    ));
}
```

### Грабли 4. `isUserVerifyingPlatformAuthenticatorAvailable()` может отсутствовать

Это статический метод `PublicKeyCredential`. В некоторых браузерах (Safari < 14, Firefox без аппаратного ключа) он есть, но всегда возвращает `false`. В других — отсутствует.

**Решение:** вызывать через `Reflect::get` + проверка на `undefined`:

```rust
let func_js = js_sys::Reflect::get(&pkc, &JsValue::from_str(method))?;
if func_js.is_undefined() || func_js.is_null() {
    return Ok(false); // метод отсутствует — не ошибка, а отсутствие возможности
}
```

### Грабли 5. `clientDataJSON` и `authenticatorData` — не «чёрные ящики»

Начинающие часто сохраняют эти бинарные структуры «как есть» и отправляют на сервер. Но именно в них содержится вся информация о том, что реально произошло. Без их разбора пользователь не видит **почему** WebAuthn безопасен.

**Решение:** разбирать обе структуры прямо в браузере, показывать флаги и проверки в UI. Это ядро интеллектуальной демонстрации.

---

## 2. Архитектура модуля

```
src/
├── lib.rs                       ← pub mod webauthn
├── app.rs                       ← демо-пример главы
└── webauthn/
    ├── mod.rs                   ← WebAuthnError, WebAuthnErrorKind, credentials_container()
    ├── capability.rs            ← isAvailable, isUVPAA, isConditionalMediationAvailable
    ├── register.rs              ← navigator.credentials.create()
    ├── authenticate.rs          ← navigator.credentials.get() + conditional
    ├── encoding.rs              ← base64url ↔ bytes, hex_preview, jsvalue_to_bytes
    ├── parse.rs                 ← parse_authenticator_data, parse_client_data
    └── model.rs                 ← RpConfig, RegistrationResult, AuthResult
```

Каждый модуль — тонкая обёртка над одним аспектом WebAuthn API. `WebAuthnError` оборачивает `DOMException` в человекочитаемую структуру с классификацией ошибок.

---

## 3. Сквозной сценарий: Алиса → Боб (web-версия)

Главная идея главы — показать полный флоу, который проходят реальные сервисы:

1. **Регистрация.** Алиса нажимает «Зарегистрировать passkey». Сервер генерирует 32-байтный `challenge`. Браузер показывает диалог Touch ID. Аутентификатор создаёт пару ключей, возвращает `attestationObject` и `clientDataJSON`. Сервер сохраняет публичный ключ.

2. **Аутентификация.** Алиса нажимает «Войти с passkey». Сервер генерирует новый `challenge`. Браузер показывает диалог. Аутентификатор подписывает `authenticatorData || SHA-256(clientDataJSON)` приватным ключом. Сервер проверяет подпись публичным ключом.

3. **Anti-phishing в действии.** Passkey, зарегистрированный на `example.com`, **не сработает** на `evil.com`. Это не «фича» приложения — это физика протокола: `rpIdHash` в `authenticatorData` привязан к origin, и аутентификатор откажется подписывать чужой origin.

4. **Anti-cloning в действии.** `signCount` в `authenticatorData` растёт с каждым входом. Если счётчик не растёт, значит credential склонировали. Это видно в демо после второго входа.

---

## 4. Разбор `authenticatorData` — ядро демонстрации

`authenticatorData` — минимум 37 байт:

```
┌────────────────┬────────┬───────────┬───────────────┐
│ rpIdHash       │ flags  │ signCount │  extensions   │
│ 32 байта       │ 1 байт │ 4 байта   │  (опционально)│
│ SHA-256(rp.id) │        │ BE        │               │
└────────────────┴────────┴───────────┴───────────────┘
```

### rpIdHash

Это `SHA-256(rp.id)`. Если сервер видит другой хеш — значит подпись сделана на другом origin. В демо мы вычисляем `SHA-256("localhost")` через Web Crypto из главы 38 и сравниваем с `rpIdHash`. Это замыкает обе главы: одна даёт примитив (SHA-256), вторая — применение.

### Флаги (байт 32)

| Бит | Имя | Что значит |
|---|---|---|
| 0x01 | **UP** — User Present | Пользователь коснулся аутентификатора |
| 0x04 | **UV** — User Verified | Подтверждена личность (биометрия / PIN) |
| 0x08 | **BE** — Backup Eligible | Credential можно синхронизировать (облачный passkey) |
| 0x10 | **BS** — Backup State | Credential сейчас синхронизирован |
| 0x40 | AT — Attested Credential Data | Есть данные о публичном ключе (при регистрации) |
| 0x80 | ED — Extension Data | Есть расширения |

UP и UV — самые важные. UP показывает, что пользователь физически взаимодействовал с устройством (не «бот с украденным ключом»). UV показывает, что была биометрия или PIN, а не просто касание.

### signCount

Монотонный счётчик подписей. При каждой успешной аутентификации должен увеличиваться. Если счётчик не растёт — возможна клонированная копия аутентификатора:

- **Первый вход**: счётчик сравнивать не с чем — `sign_count = 0` в БД. Проверка невозможна.
- **Второй вход**: `prev_sign = 0` из БД, `new_sign > 0`. Если `new_sign > prev_sign` — ✅ подлинный. Если `new_sign <= prev_sign` — ❌ возможен клон.

Реальные сервисы хранят `signCount` в БД и сравнивают при каждом входе.

---

## 5. Разбор `clientDataJSON`

Структура, которую браузер формирует сам:

```json
{
  "type": "webauthn.get",
  "challenge": "5y7bD1k...",
  "origin": "http://localhost:5173",
  "crossOrigin": false
}
```

- **type** — `"webauthn.create"` при регистрации, `"webauthn.get"` при аутентификации.
- **challenge** — тот же challenge, что отправил сервер, в base64url. Совпадение доказывает свежесть запроса.
- **origin** — точный origin страницы. **JavaScript не может его подделать** — браузер сам подставляет.

Проверка `challenge` в демо — замыкание цикла challenge/response: сервер видит, что клиент ответил именно на его запрос, а не на старый (защита от replay).

---

## 6. Conditional mediation — современный UX

`mediation: "conditional"` включает autofill-режим: браузер сам предлагает подходящий passkey прямо в поле логина. Пользователь видит не кнопку «Войти с passkey», а подсказку в поле ввода.

```rust
options.publicKey = { challenge, rpId, timeout };
options.mediation = "conditional";
```

Это самый удобный UX для входа: пользователь начинает вводить логин, браузер предлагает passkey из менеджера. В демо — toggle, который переключает обычный флоу на conditional.

---

## 7. Rust cryptography vs WebAuthn

| Критерий | RustCrypto + WebAuthn в Rust | WebAuthn API из браузера |
|---|---|---|
| Размер бандла | +100–500 KB WASM | 0 (встроено в браузер) |
| Хранение ключа | Приложение решает | Аутентификатор, никогда JS |
| Защита от фишинга | Нужно реализовать | Встроена (origin) |
| Пользовательский UX | Свои диалоги | Системные (Touch ID, Hello) |
| Согласованность | Одинаково на сервере и клиенте | Только в браузере |
| Отладка | Полный контроль | Чёрный ящик аутентификатора |

**Правило выбора:** WebAuthn из браузера — всегда, если речь о веб-аутентификации. Rust на сервере нужен для **верификации** ответа аутентификатора: парсинга CBOR, проверки подписи, работы с БД. Именно поэтому реальные Rust-проекты (например, `webauthn-rs`) реализуют **серверную часть** WebAuthn, а не клиентскую.

Гибрид для веб-приложения: клиент — JS + WebAuthn API, сервер — Rust с `webauthn-rs`. В WASM на клиенте — только UI и сборка/разбор данных.

---

## 8. Исходные коды

### `Cargo.toml` (изменения)

Новые features web-sys **не нужны** — мы не используем типизированные WebAuthn-обёртки. `serde` и `serde_json` уже подключены.

### `src/lib.rs`

```rust
//! Hello WASM02 — курс по Rust + WebAssembly.
//!
//! Глава 41: Web Authentication.

pub mod format_str;
pub mod app;
pub mod web_crypto;   // из главы 38
pub mod webauthn;     // глава 41
```

### `src/webauthn/mod.rs`

```rust
//! Глава 41: Web Authentication — WebAuthn и passkeys.
//!
//! WebAuthn — расширение Credential Management API, позволяющее
//! создавать и использовать публичные ключи для аутентификации
//! без пароля.
//!
//! ПРАВИЛО: `web-sys` не экспортирует `CredentialsContainer`,
//! `CredentialCreationOptions`, `CredentialRequestOptions` без
//! флага `--cfg=web_sys_unstable_apis`. Мы вызываем JS-методы
//! через `Reflect::get` + `Function::call*` — это универсальный
//! способ работать с любым JS API.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

pub mod capability;
pub mod register;
pub mod authenticate;
pub mod encoding;
pub mod model;
pub mod parse;

use wasm_bindgen::JsValue;
use crate::format_str::format_str;

/// Универсальная ошибка WebAuthn-операций.
#[derive(Debug, Clone)]
pub struct WebAuthnError {
    pub kind: WebAuthnErrorKind,
    pub message: String,
}

/// Классификация ошибок — важна для UI: одна и та же
/// ошибка может быть «пользователь отменил» или «аппаратная
/// проблема», и реакция разная.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum WebAuthnErrorKind {
    Unsupported,
    Cancelled,
    Timeout,
    InvalidState,
    Config,
    Other,
}

impl WebAuthnError {
    pub fn new(kind: WebAuthnErrorKind, msg: String) -> Self {
        return Self { kind, message: msg };
    }

    /// Преобразует DOMException в наш тип.
    ///
    /// `NotAllowedError` покрывает два случая: отмена пользователем
    /// и таймаут. Различаем их по сообщению.
    pub fn from_js(value: &JsValue) -> Self {
        let name = js_sys::Reflect::get(value, &JsValue::from_str("name"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();
        let msg = js_sys::Reflect::get(value, &JsValue::from_str("message"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| format_str(format_args!("{value:?}")));

        let kind = match name.as_str() {
            "NotAllowedError" => {
                if msg.to_lowercase().contains("timeout") {
                    WebAuthnErrorKind::Timeout
                } else {
                    WebAuthnErrorKind::Cancelled
                }
            }
            "InvalidStateError" => WebAuthnErrorKind::InvalidState,
            "SecurityError" | "NotSupportedError" => WebAuthnErrorKind::Config,
            _ => WebAuthnErrorKind::Other,
        };
        return Self { kind, message: msg };
    }

    /// Человекочитаемое объяснение для UI.
    pub fn user_hint(&self) -> &'static str {
        return match self.kind {
            WebAuthnErrorKind::Unsupported => {
                "Браузер не поддерживает WebAuthn. Попробуйте Chrome, Firefox или Safari последних версий."
            }
            WebAuthnErrorKind::Cancelled => {
                "Операция отменена. Возможно, вы закрыли диалог или не подтвердили действие."
            }
            WebAuthnErrorKind::Timeout => {
                "Время ожидания истекло. Попробуйте ещё раз — возможно, аутентификатор не ответил вовремя."
            }
            WebAuthnErrorKind::InvalidState => {
                "Этот аутентификатор уже зарегистрирован для данного аккаунта. Используйте другой или войдите."
            }
            WebAuthnErrorKind::Config => {
                "Проблема с конфигурацией Relying Party. Проверьте, что origin совпадает с настройками сервера."
            }
            WebAuthnErrorKind::Other => {
                "Произошла неизвестная ошибка. Подробности в консоли."
            }
        };
    }
}

impl std::fmt::Display for WebAuthnError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return write!(f, "{}", self.message);
    }
}

/// Возвращает `navigator.credentials` как `JsValue`.
///
/// Не используем типизированный `web_sys::CredentialsContainer`:
/// этот тип требует `--cfg=web_sys_unstable_apis`. Работаем с
/// сырым JS-объектом через `Reflect::get` и `Function::call*`.
pub fn credentials_container() -> Result<JsValue, WebAuthnError> {
    let window = web_sys::window()
        .ok_or_else(|| WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "no window".to_string(),
        ))?;
    let navigator = window.navigator();
    let credentials = js_sys::Reflect::get(
        &navigator,
        &JsValue::from_str("credentials"),
    ).map_err(|e| WebAuthnError::from_js(&e))?;
    if credentials.is_undefined() || credentials.is_null() {
        return Err(WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "navigator.credentials unavailable".to_string(),
        ));
    }
    return Ok(credentials);
}
```

### `src/webauthn/capability.rs`

```rust
//! Проверка возможностей браузера и аутентификатора.
//!
//! Все методы вызываются через `Reflect::get` + `Function::call*`
//! из-за unstable-статуса WebAuthn-типов в web-sys.

use wasm_bindgen::JsValue;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use super::WebAuthnError;
use super::WebAuthnErrorKind;
use crate::format_str::format_str;

/// Проверяет, доступен ли WebAuthn в текущем браузере.
pub fn is_available() -> bool {
    let Some(window) = web_sys::window() else {
        return false;
    };
    let pkc = js_sys::Reflect::get(&window, &JsValue::from_str("PublicKeyCredential"))
        .ok()
        .unwrap_or(JsValue::UNDEFINED);
    return !pkc.is_undefined() && !pkc.is_null();
}

/// Возвращает `window.PublicKeyCredential` как `JsValue`.
fn pkc_object() -> Result<JsValue, WebAuthnError> {
    if !is_available() {
        return Err(WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "PublicKeyCredential not available".to_string(),
        ));
    }
    let window = web_sys::window().ok_or_else(|| {
        WebAuthnError::new(WebAuthnErrorKind::Unsupported, "no window".to_string())
    })?;
    let pkc = js_sys::Reflect::get(&window, &JsValue::from_str("PublicKeyCredential"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return Ok(pkc);
}

/// Вызывает статический метод `PublicKeyCredential.<name>()` без аргументов.
async fn call_pkc_async(method: &str) -> Result<bool, WebAuthnError> {
    let pkc = pkc_object()?;
    let func_js = js_sys::Reflect::get(&pkc, &JsValue::from_str(method))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    if func_js.is_undefined() || func_js.is_null() {
        return Err(WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            format_str(format_args!("{method}: not defined")),
        ));
    }
    let func: js_sys::Function = func_js.dyn_into().map_err(|_| {
        WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            format_str(format_args!("{method}: not a function")),
        )
    })?;
    let promise_value = func
        .call0(&pkc)
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let promise: js_sys::Promise = promise_value.dyn_into().map_err(|_| {
        WebAuthnError::new(
            WebAuthnErrorKind::Other,
            format_str(format_args!("{method}: not a promise")),
        )
    })?;
    let result = JsFuture::from(promise)
        .await
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return Ok(result.as_bool().unwrap_or(false));
}

/// Проверяет наличие platform authenticator (Touch ID, Windows Hello).
pub async fn has_platform_authenticator() -> Result<bool, WebAuthnError> {
    return call_pkc_async("isUserVerifyingPlatformAuthenticatorAvailable").await;
}

/// Проверяет поддержку conditional mediation (autofill passkey).
///
/// Возвращает `Ok(false)`, если метод отсутствует (старые браузеры) —
/// это не ошибка, а отсутствие возможности.
pub async fn is_conditional_mediation_available() -> Result<bool, WebAuthnError> {
    let pkc = match pkc_object() {
        Ok(p) => p,
        Err(e) => return Err(e),
    };
    let func_js = js_sys::Reflect::get(&pkc, &JsValue::from_str("isConditionalMediationAvailable"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    if func_js.is_undefined() || func_js.is_null() {
        return Ok(false);
    }
    let func: js_sys::Function = match func_js.dyn_into() {
        Ok(f) => f,
        Err(_) => return Ok(false),
    };
    let promise_value = match func.call0(&pkc) {
        Ok(v) => v,
        Err(e) => return Err(WebAuthnError::from_js(&e)),
    };
    let promise: js_sys::Promise = match promise_value.dyn_into() {
        Ok(p) => p,
        Err(_) => return Ok(false),
    };
    let result = JsFuture::from(promise)
        .await
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return Ok(result.as_bool().unwrap_or(false));
}

/// Возвращает строку с описанием возможностей — для отображения в UI.
pub async fn capability_summary() -> String {
    let mut parts: Vec<String> = Vec::new();
    if is_available() {
        parts.push("WebAuthn: ✅".to_string());
    } else {
        parts.push("WebAuthn: ❌".to_string());
        return parts.join(" · ");
    }
    match has_platform_authenticator().await {
        Ok(true) => parts.push("platform: ✅".to_string()),
        Ok(false) => parts.push("platform: ❌".to_string()),
        Err(_) => parts.push("platform: ?".to_string()),
    }
    match is_conditional_mediation_available().await {
        Ok(true) => parts.push("conditional: ✅".to_string()),
        Ok(false) => parts.push("conditional: ❌".to_string()),
        Err(_) => parts.push("conditional: ?".to_string()),
    }
    return parts.join(" · ");
}
```

### `src/webauthn/encoding.rs`

```rust
//! Кодирование/декодирование для WebAuthn.
//!
//! WebAuthn использует два формата для бинарных данных:
//! - `Uint8Array` — внутри JS API (challenge, rawId, signature).
//! - base64url — при передаче по сети (JSON).
//!
//! При этом в JSON-формате challenge и credential.id обычно
//! кодируются в base64url без padding.

use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;

/// Кодирует байты в base64url (без padding).
pub fn bytes_to_base64url(bytes: &[u8]) -> String {
    let arr = js_sys::Uint8Array::new_with_length(bytes.len() as u32);
    arr.copy_from(bytes);
    // btoa не умеет в base64url — заменяем символы.
    let window = web_sys::window().expect("no window");
    let btoa = js_sys::Reflect::get(&window, &JsValue::from_str("btoa"))
        .ok()
        .and_then(|f| f.dyn_into::<js_sys::Function>().ok());
    let Some(btoa) = btoa else {
        return String::new();
    };
    // Преобразуем байты в Latin-1 строку.
    let mut s = String::with_capacity(bytes.len());
    for &b in bytes {
        s.push(b as char);
    }
    let result = btoa
        .call1(&JsValue::NULL, &JsValue::from_str(&s))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    return result
        .replace('+', "-")
        .replace('/', "_")
        .trim_end_matches('=')
        .to_string();
}

/// Декодирует base64url (с или без padding) в байты.
pub fn base64url_to_bytes(s: &str) -> Result<Vec<u8>, String> {
    // Добавляем padding.
    let padded = match s.len() % 4 {
        0 => s.to_string(),
        2 => format!("{s}=="),
        3 => format!("{s}="),
        _ => return Err("invalid base64url length".to_string()),
    };
    let standard = padded.replace('-', "+").replace('_', "/");

    let window = web_sys::window().ok_or("no window")?;
    let atob = js_sys::Reflect::get(&window, &JsValue::from_str("atob"))
        .ok()
        .and_then(|f| f.dyn_into::<js_sys::Function>().ok());
    let Some(atob) = atob else {
        return Err("no atob".to_string());
    };
    let decoded = atob
        .call1(&JsValue::NULL, &JsValue::from_str(&standard))
        .map_err(|_| "atob failed".to_string())?;
    let decoded_str = decoded.as_string().ok_or("atob: not a string")?;
    return Ok(decoded_str.bytes().collect());
}

/// Копирует содержимое `ArrayBuffer`/`Uint8Array` в `Vec<u8>`.
pub fn jsvalue_to_bytes(value: &JsValue) -> Result<Vec<u8>, String> {
    if value.is_null() || value.is_undefined() {
        return Err("jsvalue_to_bytes: null/undefined".to_string());
    }
    if let Some(arr) = value.dyn_ref::<js_sys::Uint8Array>() {
        return Ok(arr.to_vec());
    }
    if let Some(buf) = value.dyn_ref::<js_sys::ArrayBuffer>() {
        let arr = js_sys::Uint8Array::new(buf);
        return Ok(arr.to_vec());
    }
    return Err("jsvalue_to_bytes: unsupported type".to_string());
}

/// Создаёт `Uint8Array` из среза байт (для передачи в JS API).
pub fn bytes_to_uint8array(bytes: &[u8]) -> js_sys::Uint8Array {
    let arr = js_sys::Uint8Array::new_with_length(bytes.len() as u32);
    arr.copy_from(bytes);
    return arr;
}

/// Hex-превью первых `max` байт — для отображения в UI.
///
/// Если байт больше `max`, в конце добавляется многоточие.
pub fn hex_preview(bytes: &[u8], max: usize) -> String {
    let n = bytes.len().min(max);
    let mut out = String::with_capacity(n * 3 + 8);
    for (i, b) in bytes.iter().take(n).enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&crate::format_str::format_str(
            format_args!("{:02x}", b),
        ));
    }
    if bytes.len() > max {
        out.push_str(" …");
    }
    return out;
}
```

### `src/webauthn/model.rs`

```rust
//! Модели данных WebAuthn-операций.

/// Конфигурация Relying Party (нашего сервиса).
///
/// `rp_id` — домен без протокола и порта (например, "example.com").
/// `origin` — полный origin (например, "https://example.com:8443").
///
/// Браузер проверяет, что origin страницы совпадает с rp_id.
/// Это встроенная защита от фишинга: passkey, зарегистрированный
/// на example.com, не сработает на evil.com.
#[derive(Debug, Clone)]
pub struct RpConfig {
    pub rp_id: String,
    pub rp_name: String,
    pub origin: String,
}

impl RpConfig {
    pub fn new(rp_id: &str, rp_name: &str) -> Self {
        // origin для rp_id: https://{rp_id}
        let origin = crate::format_str::format_str(
            format_args!("https://{rp_id}"),
        );
        return Self {
            rp_id: rp_id.to_string(),
            rp_name: rp_name.to_string(),
            origin,
        };
    }
}

/// Результат `navigator.credentials.create()`.
///
/// `attestation_object` — CBOR-структура с публичным ключом.
/// `client_data_json` — JSON с типом операции, challenge и origin.
#[derive(Debug, Clone)]
pub struct RegistrationResult {
    pub raw_id: Vec<u8>,
    pub client_data_json: Vec<u8>,
    pub attestation_object: Vec<u8>,
    /// `platform` / `cross-platform` / `unknown`.
    pub attachment: String,
}

/// Результат `navigator.credentials.get()`.
///
/// `authenticator_data` — 37+ байт: rpIdHash(32) || flags(1) ||
/// signCount(4) || extensions. `signature` — подпись над
/// `authenticatorData || SHA-256(clientDataJSON)`.
#[derive(Debug, Clone)]
pub struct AuthResult {
    pub raw_id: Vec<u8>,
    pub client_data_json: Vec<u8>,
    pub authenticator_data: Vec<u8>,
    pub signature: Vec<u8>,
    pub user_handle: Option<Vec<u8>>,
}
```

### `src/webauthn/parse.rs`

```rust
//! Разбор бинарных структур WebAuthn.
//!
//! WebAuthn возвращает две структуры, которые нужно разобрать,
//! чтобы понять, что реально произошло:
//!
//! - `authenticatorData` — минимум 37 байт:
//!   `rpIdHash(32) || flags(1) || signCount(4 BE) || [extensions]`.
//! - `clientDataJSON` — JSON с `type`, `challenge`, `origin`,
//!   `crossOrigin`. Именно здесь браузер подтверждает origin —
//!   JavaScript не может подделать это поле.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

use crate::format_str::format_str;
use serde::Deserialize;

// ============================================================
//                    AUTHENTICATOR DATA
// ============================================================

/// Байт флагов из `authenticatorData[32]`.
///
/// Расшифровка — по [W3C WebAuthn Level 2, §6.1](https://www.w3.org/TR/webauthn-2/#flags).
#[derive(Debug, Clone, Copy)]
pub struct AuthFlags(pub u8);

impl AuthFlags {
    /// User Present: пользователь коснулся аутентификатора.
    pub fn user_present(self) -> bool { return (self.0 & 0x01) != 0; }
    /// User Verified: подтверждена личность (биометрия или PIN).
    pub fn user_verified(self) -> bool { return (self.0 & 0x04) != 0; }
    /// Backup Eligible: credential может синхронизироваться (облачный passkey).
    pub fn backup_eligible(self) -> bool { return (self.0 & 0x08) != 0; }
    /// Backup State: credential сейчас синхронизирован.
    pub fn backup_state(self) -> bool { return (self.0 & 0x10) != 0; }
    /// Attested Credential Data: включены данные о новом ключе
    /// (только при регистрации).
    pub fn attested_credential(self) -> bool { return (self.0 & 0x40) != 0; }
    /// Extension Data: включены расширения.
    pub fn extension_data(self) -> bool { return (self.0 & 0x80) != 0; }
}

/// Разобранный `authenticatorData`.
#[derive(Debug, Clone, Copy)]
pub struct AuthData {
    pub rp_id_hash: [u8; 32],
    pub flags: AuthFlags,
    pub sign_count: u32,
}

/// Разбирает `authenticatorData`.
///
/// Минимальный валидный размер — 37 байт. При регистрации
/// структура длиннее (attestedCredentialData + extensions), но
/// для проверки флагов и счётчика нам достаточно первых 37.
pub fn parse_authenticator_data(bytes: &[u8]) -> Result<AuthData, String> {
    if bytes.len() < 37 {
        return Err(format_str(format_args!(
            "authenticatorData too short: {} < 37",
            bytes.len(),
        )));
    }
    let mut rp_id_hash = [0u8; 32];
    rp_id_hash.copy_from_slice(&bytes[0..32]);
    let flags = AuthFlags(bytes[32]);
    let sign_count = u32::from_be_bytes([
        bytes[33], bytes[34], bytes[35], bytes[36],
    ]);
    return Ok(AuthData { rp_id_hash, flags, sign_count });
}

// ============================================================
//                    CLIENT DATA JSON
// ============================================================

/// Разобранный `clientDataJSON`.
#[derive(Debug, Clone, Deserialize)]
pub struct ClientData {
    /// `"webauthn.create"` или `"webauthn.get"`.
    #[serde(rename = "type")]
    pub kind: String,
    /// base64url от challenge (без padding).
    pub challenge: String,
    /// Точный origin, на котором произошла операция.
    pub origin: String,
    #[serde(default, rename = "crossOrigin")]
    pub cross_origin: bool,
}

/// Разбирает `clientDataJSON`.
pub fn parse_client_data(bytes: &[u8]) -> Result<ClientData, String> {
    let s = std::str::from_utf8(bytes)
        .map_err(|e| format_str(format_args!("clientDataJSON utf8: {e}")))?;
    return serde_json::from_str(s)
        .map_err(|e| format_str(format_args!("clientDataJSON json: {e}")));
}
```

### `src/webauthn/register.rs`

```rust
//! Регистрация passkey через `navigator.credentials.create()`.
//!
//! Все опции собираются как обычные JS-объекты (`js_sys::Object`),
//! а вызов идёт через `Reflect::get("create")` + `Function::call1`.
//! Это обходит проблему отсутствия типизированных WebAuthn-обёрток
//! в стабильном web-sys.

use wasm_bindgen::JsValue;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use super::WebAuthnError;
use super::WebAuthnErrorKind;
use super::encoding::{bytes_to_uint8array, jsvalue_to_bytes};
use super::model::{RegistrationResult, RpConfig};

/// Пользователь для регистрации.
pub struct RegistrationUser<'a> {
    /// Уникальный ID (в базе данных) — 16–64 байта.
    pub id: &'a [u8],
    /// Отображаемое имя (например, "Alice").
    pub name: &'a str,
    /// Полное имя (например, "Alice Doe").
    pub display_name: &'a str,
}

/// Собирает `{ publicKey: {...} }` и вызывает `credentials.create()`.
pub async fn register_passkey(
    config: &RpConfig,
    user: &RegistrationUser<'_>,
    challenge: &[u8],
) -> Result<RegistrationResult, WebAuthnError> {
    let credentials = super::credentials_container()?;
    let options = build_creation_options(config, user, challenge)?;

    let create_fn_js = js_sys::Reflect::get(&credentials, &JsValue::from_str("create"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    if create_fn_js.is_undefined() || create_fn_js.is_null() {
        return Err(WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "credentials.create not defined".to_string(),
        ));
    }
    let create_fn: js_sys::Function = create_fn_js.dyn_into().map_err(|_| {
        WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "credentials.create is not a function".to_string(),
        )
    })?;
    let promise_value = create_fn
        .call1(&credentials, &options)
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let promise: js_sys::Promise = promise_value.dyn_into().map_err(|_| {
        WebAuthnError::new(
            WebAuthnErrorKind::Other,
            "create: not a promise".to_string(),
        )
    })?;
    let credential = JsFuture::from(promise)
        .await
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return parse_creation_result(&credential);
}

/// Собирает `CredentialCreationOptions` как JS-объект.
fn build_creation_options(
    config: &RpConfig,
    user: &RegistrationUser<'_>,
    challenge: &[u8],
) -> Result<JsValue, WebAuthnError> {
    let public_key = js_sys::Object::new();

    // rp: { id, name }
    let rp = js_sys::Object::new();
    set_prop(&rp, "id", &JsValue::from_str(&config.rp_id))?;
    set_prop(&rp, "name", &JsValue::from_str(&config.rp_name))?;
    set_prop(&public_key, "rp", &rp)?;

    // user: { id, name, displayName }
    let user_obj = js_sys::Object::new();
    set_prop(&user_obj, "id", &bytes_to_uint8array(user.id))?;
    set_prop(&user_obj, "name", &JsValue::from_str(user.name))?;
    set_prop(&user_obj, "displayName", &JsValue::from_str(user.display_name))?;
    set_prop(&public_key, "user", &user_obj)?;

    // challenge: Uint8Array
    set_prop(&public_key, "challenge", &bytes_to_uint8array(challenge))?;

    // pubKeyCredParams: ES256 = -7, RS256 = -257
    let params = js_sys::Array::new();
    params.push(&make_cred_param("public-key", -7));
    params.push(&make_cred_param("public-key", -257));
    set_prop(&public_key, "pubKeyCredParams", &params)?;

    // timeout
    set_prop(&public_key, "timeout", &JsValue::from_f64(60_000.0))?;

    // authenticatorSelection
    let selection = js_sys::Object::new();
    set_prop(
        &selection,
        "authenticatorAttachment",
        &JsValue::from_str("platform"),
    )?;
    set_prop(
        &selection,
        "userVerification",
        &JsValue::from_str("preferred"),
    )?;
    set_prop(
        &selection,
        "residentKey",
        &JsValue::from_str("preferred"),
    )?;
    set_prop(&public_key, "authenticatorSelection", &selection)?;

    // attestation
    set_prop(&public_key, "attestation", &JsValue::from_str("none"))?;

    // Финальный options: { publicKey: {...} }
    let options = js_sys::Object::new();
    set_prop(&options, "publicKey", &public_key)?;
    return Ok(options.into());
}

/// Хелпер: `Reflect::set` с понятной ошибкой.
fn set_prop(
    obj: &js_sys::Object,
    key: &str,
    value: &JsValue,
) -> Result<(), WebAuthnError> {
    js_sys::Reflect::set(obj, &JsValue::from_str(key), value)
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return Ok(());
}

/// Создаёт `{ type: "public-key", alg: -7 }`.
fn make_cred_param(type_str: &str, alg: i32) -> JsValue {
    let p = js_sys::Object::new();
    let _ = js_sys::Reflect::set(
        &p,
        &JsValue::from_str("type"),
        &JsValue::from_str(type_str),
    );
    let _ = js_sys::Reflect::set(
        &p,
        &JsValue::from_str("alg"),
        &JsValue::from_f64(alg as f64),
    );
    return p.into();
}

/// Разбирает результат `create()`.
fn parse_creation_result(
    credential: &JsValue,
) -> Result<RegistrationResult, WebAuthnError> {
    // rawId
    let raw_id_val = js_sys::Reflect::get(credential, &JsValue::from_str("rawId"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let raw_id = jsvalue_to_bytes(&raw_id_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    // response
    let response = js_sys::Reflect::get(credential, &JsValue::from_str("response"))
        .map_err(|e| WebAuthnError::from_js(&e))?;

    // clientDataJSON
    let client_data_val = js_sys::Reflect::get(&response, &JsValue::from_str("clientDataJSON"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let client_data = jsvalue_to_bytes(&client_data_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    // attestationObject
    let attestation_val = js_sys::Reflect::get(&response, &JsValue::from_str("attestationObject"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let attestation = jsvalue_to_bytes(&attestation_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    // authenticatorAttachment (опционально)
    let attachment = js_sys::Reflect::get(
        credential,
        &JsValue::from_str("authenticatorAttachment"),
    )
    .ok()
    .and_then(|v| v.as_string())
    .unwrap_or_else(|| "unknown".to_string());

    return Ok(RegistrationResult {
        raw_id,
        client_data_json: client_data,
        attestation_object: attestation,
        attachment,
    });
}
```

### `src/webauthn/authenticate.rs`

```rust
//! Аутентификация через `navigator.credentials.get()`.
//!
//! Поддерживает два режима:
//!   - обычный (`allowCredentials` со списком ID);
//!   - conditional (`mediation: "conditional"`) — passkey
//!     появляется прямо в autofill поля логина.

use wasm_bindgen::JsValue;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use super::WebAuthnError;
use super::WebAuthnErrorKind;
use super::encoding::{bytes_to_uint8array, jsvalue_to_bytes};
use super::model::{AuthResult, RpConfig};

/// Запускает `navigator.credentials.get()` со списком credential.
pub async fn authenticate_passkey(
    config: &RpConfig,
    challenge: &[u8],
    allow_credentials: &[Vec<u8>],
) -> Result<AuthResult, WebAuthnError> {
    let credentials = super::credentials_container()?;
    let options = build_request_options(config, challenge, allow_credentials)?;
    return call_get(&credentials, &options).await;
}

/// Conditional mediation — passkey в autofill.
///
/// `allowCredentials` пуст, `mediation: "conditional"`. Браузер
/// сам подставит подходящий passkey.
pub async fn authenticate_conditional(
    config: &RpConfig,
    challenge: &[u8],
) -> Result<AuthResult, WebAuthnError> {
    let credentials = super::credentials_container()?;
    let options = build_conditional_options(config, challenge)?;
    return call_get(&credentials, &options).await;
}

async fn call_get(
    credentials: &JsValue,
    options: &JsValue,
) -> Result<AuthResult, WebAuthnError> {
    let get_fn_js = js_sys::Reflect::get(credentials, &JsValue::from_str("get"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    if get_fn_js.is_undefined() || get_fn_js.is_null() {
        return Err(WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "credentials.get not defined".to_string(),
        ));
    }
    let get_fn: js_sys::Function = get_fn_js.dyn_into().map_err(|_| {
        WebAuthnError::new(
            WebAuthnErrorKind::Unsupported,
            "credentials.get is not a function".to_string(),
        )
    })?;
    let promise_value = get_fn
        .call1(credentials, options)
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let promise: js_sys::Promise = promise_value.dyn_into().map_err(|_| {
        WebAuthnError::new(
            WebAuthnErrorKind::Other,
            "get: not a promise".to_string(),
        )
    })?;
    let credential = JsFuture::from(promise)
        .await
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return parse_assertion(&credential);
}

fn build_request_options(
    config: &RpConfig,
    challenge: &[u8],
    allow_credentials: &[Vec<u8>],
) -> Result<JsValue, WebAuthnError> {
    let public_key = js_sys::Object::new();

    set_prop(&public_key, "challenge", &bytes_to_uint8array(challenge))?;
    set_prop(&public_key, "timeout", &JsValue::from_f64(60_000.0))?;
    set_prop(&public_key, "rpId", &JsValue::from_str(&config.rp_id))?;
    set_prop(
        &public_key,
        "userVerification",
        &JsValue::from_str("preferred"),
    )?;

    let allow = js_sys::Array::new();
    for cred_id in allow_credentials {
        let cred = js_sys::Object::new();
        let _ = js_sys::Reflect::set(
            &cred,
            &JsValue::from_str("type"),
            &JsValue::from_str("public-key"),
        );
        let _ = js_sys::Reflect::set(
            &cred,
            &JsValue::from_str("id"),
            &bytes_to_uint8array(cred_id),
        );
        allow.push(&cred);
    }
    set_prop(&public_key, "allowCredentials", &allow)?;

    let options = js_sys::Object::new();
    set_prop(&options, "publicKey", &public_key)?;
    return Ok(options.into());
}

fn build_conditional_options(
    config: &RpConfig,
    challenge: &[u8],
) -> Result<JsValue, WebAuthnError> {
    let public_key = js_sys::Object::new();
    set_prop(&public_key, "challenge", &bytes_to_uint8array(challenge))?;
    set_prop(&public_key, "timeout", &JsValue::from_f64(60_000.0))?;
    set_prop(&public_key, "rpId", &JsValue::from_str(&config.rp_id))?;
    set_prop(
        &public_key,
        "userVerification",
        &JsValue::from_str("preferred"),
    )?;

    let options = js_sys::Object::new();
    set_prop(&options, "publicKey", &public_key)?;
    // mediation: "conditional" — включает autofill-режим.
    set_prop(&options, "mediation", &JsValue::from_str("conditional"))?;
    return Ok(options.into());
}

fn set_prop(
    obj: &js_sys::Object,
    key: &str,
    value: &JsValue,
) -> Result<(), WebAuthnError> {
    js_sys::Reflect::set(obj, &JsValue::from_str(key), value)
        .map_err(|e| WebAuthnError::from_js(&e))?;
    return Ok(());
}

fn parse_assertion(credential: &JsValue) -> Result<AuthResult, WebAuthnError> {
    let raw_id_val = js_sys::Reflect::get(credential, &JsValue::from_str("rawId"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let raw_id = jsvalue_to_bytes(&raw_id_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    let response = js_sys::Reflect::get(credential, &JsValue::from_str("response"))
        .map_err(|e| WebAuthnError::from_js(&e))?;

    let client_data_val = js_sys::Reflect::get(&response, &JsValue::from_str("clientDataJSON"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let client_data = jsvalue_to_bytes(&client_data_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    let auth_data_val = js_sys::Reflect::get(&response, &JsValue::from_str("authenticatorData"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let auth_data = jsvalue_to_bytes(&auth_data_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    let sig_val = js_sys::Reflect::get(&response, &JsValue::from_str("signature"))
        .map_err(|e| WebAuthnError::from_js(&e))?;
    let signature = jsvalue_to_bytes(&sig_val).map_err(|e| {
        WebAuthnError::new(WebAuthnErrorKind::Other, e)
    })?;

    let user_handle = js_sys::Reflect::get(&response, &JsValue::from_str("userHandle"))
        .ok()
        .filter(|v| !v.is_null() && !v.is_undefined())
        .and_then(|v| jsvalue_to_bytes(&v).ok());

    return Ok(AuthResult {
        raw_id,
        client_data_json: client_data,
        authenticator_data: auth_data,
        signature,
        user_handle,
    });
}
```

### `tests/wasm_webauthn.rs`

```rust
//! WASM-тесты главы 41: WebAuthn.
//!
//! Запуск:
//!   wasm-pack test --chrome --headless -- --test wasm_webauthn
//!
//! WebAuthn требует secure context (HTTPS или localhost) и реального
//! user gesture. Тесты, требующие диалога аутентификатора, помечены
//! #[ignore] — их нельзя выполнить в headless-браузере.

use wasm_bindgen_test::*;

use hello_wasm02::webauthn::{
    capability,
    encoding::{base64url_to_bytes, bytes_to_base64url, hex_preview},
    model::RpConfig,
};

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// 1. base64url — кодирование/декодирование
// ============================================================

#[wasm_bindgen_test]
fn wasm_b64url_hello_roundtrip() {
    let bytes = b"hello";
    let s = bytes_to_base64url(bytes);
    assert_eq!(s, "aGVsbG8");
    let back = base64url_to_bytes(&s).unwrap();
    assert_eq!(back, bytes);
}

#[wasm_bindgen_test]
fn wasm_b64url_no_padding() {
    // 1 байт → 2 символа base64url без '='.
    let s = bytes_to_base64url(b"f");
    assert!(!s.contains('='));
    assert_eq!(s, "Zg");
}

#[wasm_bindgen_test]
fn wasm_b64url_url_safe_chars() {
    // 0xFB 0xFF в стандартном base64 → "+/8=",
    // в base64url → "-_8".
    let bytes: [u8; 2] = [0xFB, 0xFF];
    let s = bytes_to_base64url(&bytes);
    assert!(!s.contains('+'));
    assert!(!s.contains('/'));
    assert_eq!(s, "-_8");
}

#[wasm_bindgen_test]
fn wasm_b64url_empty() {
    let s = bytes_to_base64url(&[]);
    assert_eq!(s, "");
    let back = base64url_to_bytes("").unwrap();
    assert!(back.is_empty());
}

#[wasm_bindgen_test]
fn wasm_b64url_decode_with_padding() {
    // Стандартная base64 со '=' тоже должна декодироваться.
    let back = base64url_to_bytes("aGVsbG8=").unwrap();
    assert_eq!(back, b"hello");
}

#[wasm_bindgen_test]
fn wasm_b64url_decode_invalid_length() {
    // Длина %4 == 1 — невозможно.
    let r = base64url_to_bytes("abcde");
    assert!(r.is_err());
}

// ============================================================
// 2. hex_preview
// ============================================================

#[wasm_bindgen_test]
fn wasm_hex_preview_short() {
    let bytes = [0xAB, 0xCD];
    let s = hex_preview(&bytes, 4);
    assert_eq!(s, "ab cd");
}

#[wasm_bindgen_test]
fn wasm_hex_preview_truncated() {
    let bytes = [0x00, 0xAB, 0xCD, 0xEF];
    let s = hex_preview(&bytes, 2);
    assert_eq!(s, "00 ab …");
}

#[wasm_bindgen_test]
fn wasm_hex_preview_empty() {
    let s = hex_preview(&[], 4);
    assert_eq!(s, "");
}

// ============================================================
// 3. Capability detection
// ============================================================

#[wasm_bindgen_test]
async fn wasm_webauthn_is_available() {
    // На localhost (secure context) WebAuthn должен быть доступен.
    assert!(
        capability::is_available(),
        "WebAuthn должен быть доступен на localhost",
    );
}

#[wasm_bindgen_test]
async fn wasm_platform_authenticator_query() {
    // Возвращает Ok(bool) — значение зависит от окружения.
    let result = capability::has_platform_authenticator().await;
    assert!(result.is_ok(), "isUVPAA должен вернуть Result");
}

#[wasm_bindgen_test]
async fn wasm_conditional_mediation_query() {
    let result = capability::is_conditional_mediation_available().await;
    assert!(result.is_ok());
}

#[wasm_bindgen_test]
async fn wasm_capability_summary_contains_webauthn() {
    let summary = capability::capability_summary().await;
    assert!(!summary.is_empty());
    assert!(summary.contains("WebAuthn"));
}

// ============================================================
// 4. RpConfig
// ============================================================

#[wasm_bindgen_test]
fn wasm_rp_config_origin() {
    let cfg = RpConfig::new("example.com", "Example");
    assert_eq!(cfg.origin, "https://example.com");
    assert_eq!(cfg.rp_id, "example.com");
    assert_eq!(cfg.rp_name, "Example");
}

#[wasm_bindgen_test]
fn wasm_rp_config_localhost() {
    let cfg = RpConfig::new("localhost", "Local Dev");
    assert_eq!(cfg.origin, "https://localhost");
}

// ============================================================
// 5. Операции, требующие user gesture
// ============================================================

/// Реальная регистрация требует клика пользователя. В headless
/// Chrome диалог не появится — тест помечен #[ignore].
#[wasm_bindgen_test]
#[ignore]
async fn wasm_register_requires_user_gesture() {
    // Запускается вручную:
    //   wasm-pack test --chrome --headless -- --test wasm_webauthn --include-ignored
    // В headless всё равно не сработает — нужен обычный браузер.
}

/// Аналогично для аутентификации.
#[wasm_bindgen_test]
#[ignore]
async fn wasm_authenticate_requires_user_gesture() {
}

// ============================================================
// 6. Прямая проверка возвращаемых типов
// ============================================================

#[wasm_bindgen_test]
async fn wasm_credentials_container_is_object() {
    use hello_wasm02::webauthn::credentials_container;
    // На localhost container должен быть доступен.
    let result = credentials_container();
    assert!(result.is_ok(), "navigator.credentials должен быть доступен");
    let creds = result.unwrap();
    assert!(!creds.is_undefined());
    assert!(!creds.is_null());
}

// ============================================================
// 7. Проверка формата challenge
// ============================================================

#[wasm_bindgen_test]
fn wasm_challenge_base64url_length() {
    // 32 случайных байта → 43 символа base64url (без padding).
    let bytes = [0u8; 32];
    let s = bytes_to_base64url(&bytes);
    // 32 байта = 256 бит → ceil(32*8/6) = 43 символа.
    assert_eq!(s.len(), 43);
}

// ============================================================
// 8. Парсинг authenticatorData
// ============================================================

use hello_wasm02::webauthn::parse::{
    parse_authenticator_data, parse_client_data,
};

#[wasm_bindgen_test]
fn wasm_parse_auth_data_valid() {
    // Синтетический authenticatorData: 32 байта rpIdHash (нули),
    // flags = 0b0001_0101 (UP | UV | BS), signCount = 42.
    let mut data = vec![0u8; 37];
    data[32] = 0x15; // UP(0x01) | UV(0x04) | BS(0x10)
    data[33] = 0;
    data[34] = 0;
    data[35] = 0;
    data[36] = 42;

    let parsed = parse_authenticator_data(&data).unwrap();
    assert_eq!(parsed.sign_count, 42);
    assert!(parsed.flags.user_present());
    assert!(parsed.flags.user_verified());
    assert!(!parsed.flags.backup_eligible());
    assert!(parsed.flags.backup_state());
    assert!(!parsed.flags.attested_credential());
}

#[wasm_bindgen_test]
fn wasm_parse_auth_data_too_short() {
    let data = vec![0u8; 20];
    let result = parse_authenticator_data(&data);
    assert!(result.is_err());
}

#[wasm_bindgen_test]
fn wasm_parse_auth_data_minimal() {
    // Ровно 37 байт, все флаги 0, signCount 0 — минимум.
    let data = vec![0u8; 37];
    let parsed = parse_authenticator_data(&data).unwrap();
    assert_eq!(parsed.sign_count, 0);
    assert!(!parsed.flags.user_present());
    assert!(!parsed.flags.user_verified());
}

// ============================================================
// 9. Парсинг clientDataJSON
// ============================================================

#[wasm_bindgen_test]
fn wasm_parse_client_data_valid() {
    let json = br#"{
        "type":"webauthn.get",
        "challenge":"abc123",
        "origin":"http://localhost:5173",
        "crossOrigin":false
    }"#;
    let parsed = parse_client_data(json).unwrap();
    assert_eq!(parsed.kind, "webauthn.get");
    assert_eq!(parsed.challenge, "abc123");
    assert_eq!(parsed.origin, "http://localhost:5173");
    assert!(!parsed.cross_origin);
}

#[wasm_bindgen_test]
fn wasm_parse_client_data_no_cross_origin() {
    // crossOrigin опционален (default = false).
    let json = br#"{
        "type":"webauthn.create",
        "challenge":"xyz",
        "origin":"https://example.com"
    }"#;
    let parsed = parse_client_data(json).unwrap();
    assert_eq!(parsed.kind, "webauthn.create");
    assert!(!parsed.cross_origin);
}

#[wasm_bindgen_test]
fn wasm_parse_client_data_invalid_json() {
    let result = parse_client_data(b"not json");
    assert!(result.is_err());
}
```

### `src/app.rs`

```rust
//! Глава 41: Web Authentication — WebAuthn и passkeys.
//!
//! Демонстрирует:
//!   1. RP config: rp_id / origin / timeout + защита от IP-адресов.
//!   2. Возможности браузера (isAvailable, isUVPAA, conditional).
//!   3. Регистрацию passkey через navigator.credentials.create().
//!   4. Аутентификацию через navigator.credentials.get().
//!   5. Conditional mediation (autofill passkey).
//!   6. Разбор authenticatorData: rpIdHash, флаги UP/UV/BE/BS, signCount.
//!   7. Разбор clientDataJSON: type, challenge, origin.
//!   8. Проверку anti-phishing: SHA-256(rp_id) == rpIdHash.
//!   9. Обнаружение клонированного аутентификатора по signCount.
//!  10. Что делать на сервере после получения attestation/assertion.
//!
//! ПРАВИЛО: app.rs содержит только демо-код текущей главы.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;
use web_sys::{HtmlButtonElement, HtmlElement, HtmlInputElement};

use crate::format_str::format_str;
use crate::web_crypto::digest;
use crate::web_crypto::digest::HashAlg;
use crate::webauthn::{
    authenticate,
    capability,
    encoding::{bytes_to_base64url, hex_preview},
    model::RpConfig,
    parse::{parse_authenticator_data, parse_client_data, AuthData},
    register::{self, RegistrationUser},
    WebAuthnError, WebAuthnErrorKind,
};

// ============================================================
//                    HTML
// ============================================================

const HTML: &str = r#"
<style>
  :root {
    --w-info: #3498db; --w-ok: #27ae60; --w-err: #c0392b; --w-warn: #e67e22;
    --w-auth: #8e44ad; --w-server: #16a085;
    --w-text: #222; --w-muted: #666;
    --w-bg: #fff; --w-soft: #fafafa; --w-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --w-text: #e6e6e6; --w-muted: #a0a0a0;
      --w-bg: #1e1e1e; --w-soft: #262626; --w-border: #3a3a3a;
    }
    body { background: #121212; }
    input, textarea { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .w-out { background: #1a1a1a; color: #9ed09e; border-color: #333; }
    .w-hint { background: #232014; color: #e6c98c; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--w-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; max-width: 900px; margin: 0 auto; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .w-subtitle { color: var(--w-muted); font-size: 12px; margin: 0 0 14px; }
  .w-subtitle a { color: var(--w-info); }

  .w-panel {
    border: 1px solid var(--w-border); border-radius: 8px;
    padding: 10px 12px; background: var(--w-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    margin-bottom: 12px;
  }
  .w-panel-auth { border: 2px solid var(--w-auth); }
  .w-panel-server { border: 2px solid var(--w-server); }
  .w-panel-warn { border: 2px solid var(--w-err); background: #fdf3f3; }
  @media (prefers-color-scheme: dark) {
    .w-panel-warn { background: #2a1a1a; }
  }
  .w-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 8px;
    display: flex; align-items: center; gap: 6px;
  }
  .w-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--w-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .w-num-auth { background: var(--w-auth); }
  .w-num-server { background: var(--w-server); }
  .w-row { display: flex; gap: 6px; flex-wrap: wrap; margin: 6px 0; align-items: center; }
  .w-label { font-size: 11px; color: var(--w-muted); }
  .w-cb-label {
    font-size: 11px; color: var(--w-text);
    display: flex; align-items: center; gap: 4px; cursor: pointer;
  }

  input[type="text"], textarea {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--w-bg); color: var(--w-text);
    font-family: ui-monospace, monospace;
    width: 100%;
  }
  textarea { height: 40px; resize: vertical; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--w-soft); color: var(--w-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
    transition: background 0.15s, opacity 0.15s;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.w-primary { background: #eaf4ff; border-color: var(--w-info); color: #1f4e79; }
  button.w-success { background: #e8f8ed; border-color: var(--w-ok); color: #1e8449; }
  button.w-auth { background: #f4ecf8; border-color: var(--w-auth); color: #6c3483; }

  .w-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
    margin-left: auto;
    transition: background 0.2s, color 0.2s;
  }
  .w-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .w-badge.err { background: #fdedec; color: #922b21; }
  .w-badge.warn { background: #fff9e6; color: #7d5a00; }

  .w-timer {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--w-muted); margin-left: 8px;
  }
  .w-timer.warn { color: var(--w-warn); font-weight: 700; }

  .w-out {
    background: #f6f8fa; border: 1px solid var(--w-border);
    border-radius: 4px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #2d5a2d; overflow-wrap: anywhere;
    margin: 4px 0; min-height: 16px; white-space: pre-wrap;
  }
  @media (prefers-color-scheme: dark) {
    .w-out { background: #1a1a1a; color: #9ed09e; border-color: #333; }
  }
  .w-out.empty { color: var(--w-muted); font-style: italic; }

  .w-kv { display: grid; grid-template-columns: max-content 1fr; gap: 2px 12px;
    font-family: ui-monospace, monospace; font-size: 11px; margin: 4px 0; }
  .w-kv .k { color: var(--w-muted); }
  .w-kv .v { color: var(--w-text); overflow-wrap: anywhere; }
  .w-kv .v.ok { color: var(--w-ok); font-weight: 700; }
  .w-kv .v.err { color: var(--w-err); font-weight: 700; }

  .w-hint {
    background: #fff9e6; border-left: 3px solid var(--w-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 0 0 10px;
    color: #7d5a00;
  }
  .w-note {
    font-size: 11px; color: var(--w-muted); margin: 4px 0 0;
    line-height: 1.5;
  }
  .w-note b { color: var(--w-text); }
  .w-note code { background: rgba(0,0,0,0.06); padding: 1px 4px; border-radius: 3px; }

  .w-cred-list {
    border: 1px solid var(--w-border); border-radius: 4px;
    background: var(--w-soft); padding: 4px 0; margin: 6px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    max-height: 180px; overflow-y: auto;
  }
  .w-cred-item {
    padding: 4px 10px; display: flex; gap: 8px;
    border-bottom: 1px solid var(--w-border);
  }
  .w-cred-item:last-child { border-bottom: none; }
  .w-cred-item .name { color: var(--w-text); flex: 1; overflow-wrap: anywhere; }
  .w-cred-item .id { color: var(--w-muted); flex-shrink: 0; }
  .w-cred-empty {
    padding: 8px 10px; color: var(--w-muted);
    font-style: italic; text-align: center;
  }

  .w-checklist {
    list-style: none; padding: 0; margin: 0;
    display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 4px 16px; font-size: 11px;
    font-family: ui-monospace, monospace; color: var(--w-muted);
  }
  .w-checklist li { padding: 2px 0; }
</style>

<h1>Глава 41: Web Authentication</h1>
<p class="w-subtitle">
  WebAuthn и passkeys: аутентификация без пароля через публичные ключи.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="w-hint" id="w-hint"></div>

<!-- ============ Панель-предупреждение (показывается при открытии по IP) ============ -->
<div class="w-panel w-panel-warn" id="w-panel-warn" style="display:none;">
  <h2 class="w-panel-title">
    <span class="w-num" style="background:var(--w-err);">!</span>Откройте по localhost
  </h2>
  <p class="w-note" id="w-warn-text"></p>
</div>

<!-- ============ 1. RP config ============ -->
<div class="w-panel">
  <h2 class="w-panel-title">
    <span class="w-num">1</span>Relying Party — кто мы
    <span class="w-badge" id="w-rp-status">—</span>
  </h2>
  <div id="w-rp-out" class="w-kv"></div>
  <p class="w-note">
    <b>rp.id</b> — домен, к которому привяжутся все passkey. Браузер
    проверяет, что origin страницы совпадает с rp.id: passkey
    с <code>example.com</code> <b>не сработает</b> на
    <code>evil.com</code> даже если пользователь сам захочет.
    Это встроенная защита от фишинга.
  </p>
</div>

<!-- ============ 2. Возможности браузера ============ -->
<div class="w-panel">
  <h2 class="w-panel-title">
    <span class="w-num">2</span>Возможности браузера
    <span class="w-badge" id="w-cap-status">проверка…</span>
  </h2>
  <div class="w-row">
    <button id="w-cap-btn" class="w-primary">Проверить</button>
  </div>
  <div class="w-out empty" id="w-cap-out">—</div>
</div>

<!-- ============ 3. Регистрация passkey ============ -->
<div class="w-panel" id="w-panel-register">
  <h2 class="w-panel-title">
    <span class="w-num">3</span>Регистрация passkey
    <span class="w-badge" id="w-reg-status">ожидание</span>
  </h2>
  <div class="w-row">
    <label class="w-label" for="w-user-name">Логин:</label>
  </div>
  <div class="w-row">
    <input type="text" id="w-user-name" value="alice@example.com">
  </div>
  <div class="w-row">
    <label class="w-label" for="w-user-display">Отображаемое имя:</label>
  </div>
  <div class="w-row">
    <input type="text" id="w-user-display" value="Alice">
  </div>
  <div class="w-row">
    <button id="w-reg-btn" class="w-auth">🔑 Зарегистрировать passkey</button>
    <span class="w-timer" id="w-reg-timer"></span>
  </div>
  <div class="w-out empty" id="w-reg-out">—</div>
  <p class="w-note">
    Браузер покажет диалог: Touch ID / Windows Hello / вставить YubiKey.
    После подтверждения аутентификатор создаст пару ключей: приватный
    <b>никогда не покинет устройство</b>, публичный вернётся в attestation.
  </p>
</div>

<!-- ============ 4. Список credential ============ -->
<div class="w-panel" id="w-panel-credentials">
  <h2 class="w-panel-title">
    <span class="w-num">4</span>Credential в «базе данных» сервера
    <span class="w-badge" id="w-cred-count">0</span>
  </h2>
  <div id="w-cred-list" class="w-cred-list">
    <div class="w-cred-empty">пока никого</div>
  </div>
  <p class="w-note">
    В реальном приложении это таблица в БД. Публичный ключ и rawId
    сохраняются один раз при регистрации. <b>Секрета на сервере нет</b> —
    украсть нечего.
  </p>
</div>

<!-- ============ 5. Аутентификация ============ -->
<div class="w-panel w-panel-auth" id="w-panel-auth">
  <h2 class="w-panel-title">
    <span class="w-num w-num-auth">5</span>Аутентификация
    <span class="w-badge" id="w-auth-status">ожидание</span>
  </h2>
  <div class="w-row">
    <label class="w-label" for="w-auth-user">Логин:</label>
  </div>
  <div class="w-row">
    <input type="text" id="w-auth-user" value="alice@example.com">
  </div>
  <div class="w-row">
    <label class="w-cb-label">
      <input type="checkbox" id="w-conditional">
      Passkey в autofill поля ввода (conditional mediation)
    </label>
  </div>
  <div class="w-row">
    <button id="w-auth-btn" class="w-auth" disabled>🔓 Войти с passkey</button>
    <span class="w-timer" id="w-auth-timer"></span>
  </div>
  <div class="w-out empty" id="w-auth-out">—</div>
  <p class="w-note">
    Сервер присылает свежий <code>challenge</code>. Аутентификатор
    подписывает <code>authenticatorData || SHA-256(clientDataJSON)</code>.
    Ниже мы <b>разбираем</b>, что вернулось, и проверяем
    критические свойства WebAuthn: привязку к origin и счётчик.
  </p>
</div>

<!-- ============ 6. Что делать на сервере ============ -->
<div class="w-panel w-panel-server">
  <h2 class="w-panel-title">
    <span class="w-num w-num-server">6</span>Что делать на сервере
  </h2>
  <p class="w-note">
    <b>После регистрации (create):</b><br>
    1. Распарсить <code>attestationObject</code> через CBOR-парсер
       (например, <code>coset</code> или <code>ciborium</code>) — извлечь
       публичный ключ и AAGUID.<br>
    2. Проверить <code>clientDataJSON</code>: <code>type == "webauthn.create"</code>,
       <code>challenge</code> совпадает с выданным, <code>origin</code> — наш.<br>
    3. Сохранить в БД: <code>user_id</code>, <code>credential_id</code>
       (rawId), <code>public_key</code>, <code>sign_count = 0</code>.
    <br><br>
    <b>После аутентификации (get):</b><br>
    1. Найти credential по <code>rawId</code>.<br>
    2. Проверить <code>clientDataJSON</code> (то же, но
       <code>type == "webauthn.get"</code>).<br>
    3. Проверить <code>SHA-256(rp_id) == authenticatorData[0..32]</code>.<br>
    4. Проверить подпись публичным ключом над
       <code>authenticatorData || SHA-256(clientDataJSON)</code>.<br>
    5. Проверить <code>signCount</code>: должен быть больше сохранённого —
       иначе возможна клонированная копия аутентификатора.
  </p>
</div>

<!-- ============ Чек-лист ============ -->
<div class="w-panel">
  <h2 class="w-panel-title">📋 Что охвачено в этой главе</h2>
  <ul class="w-checklist">
    <li>✅ CredentialsContainer</li>
    <li>✅ PublicKeyCredential</li>
    <li>✅ WebAuthn — create/get</li>
    <li>✅ passkeys — platform authenticator</li>
    <li>✅ authenticator — Touch ID / Hello / YubiKey</li>
    <li>✅ challenge / response</li>
    <li>✅ Rust ↔ browser authentication boundary</li>
    <li>✅ Разбор authenticatorData</li>
    <li>✅ Разбор clientDataJSON</li>
    <li>✅ Anti-phishing: rpIdHash</li>
    <li>✅ Обнаружение клона по signCount</li>
    <li>✅ Что делать на сервере</li>
  </ul>
</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone)]
struct StoredCredential {
    user_name: String,
    raw_id: Vec<u8>,
    attachment: String,
    /// Максимальный signCount, который видели для этого credential.
    sign_count: u32,
    /// Сколько раз этот credential использовался.
    auth_count: u32,
}

struct Ui {
    rp_status: web_sys::Element,
    rp_out: web_sys::Element,

    panel_warn: web_sys::Element,
    warn_text: web_sys::Element,

    cap_btn: HtmlButtonElement,
    cap_status: web_sys::Element,
    cap_out: web_sys::Element,

    user_name: HtmlInputElement,
    user_display: HtmlInputElement,
    reg_btn: HtmlElement,
    reg_status: web_sys::Element,
    reg_out: web_sys::Element,
    reg_timer: web_sys::Element,

    cred_list: web_sys::Element,
    cred_count: web_sys::Element,

    auth_user: HtmlInputElement,
    auth_conditional: HtmlInputElement,
    auth_btn: HtmlButtonElement,
    auth_status: web_sys::Element,
    auth_out: web_sys::Element,
    auth_timer: web_sys::Element,
}

struct State {
    credentials: Vec<StoredCredential>,
}

struct App {
    ui: Ui,
    state: RefCell<State>,
    config: RpConfig,
    /// Открыт ли UI по IP-адресу (тогда rp_id был подменён на localhost).
    opened_by_ip: bool,
}

// ============================================================
//                    ЗАПУСК
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let location = web_sys::window().ok_or("no window")?.location();
    let raw_host = location
        .hostname()
        .unwrap_or_else(|_| "localhost".to_string());
    let protocol = location
        .protocol()
        .unwrap_or_else(|_| "https:".to_string());

    // WebAuthn НЕ допускает IP-адреса в rp.id. Если страница открыта
    // по 127.0.0.1, принудительно используем localhost — но реально
    // браузер всё равно отвергнет операцию, если сам origin = 127.0.0.1.
    let opened_by_ip = is_ip_address(&raw_host);
    let rp_id = if opened_by_ip {
        "localhost".to_string()
    } else {
        raw_host.clone()
    };
    let origin = format_str(format_args!("{protocol}//{rp_id}"));

    let config = RpConfig {
        rp_id: rp_id.clone(),
        rp_name: "Глава 41 — WebAuthn".to_string(),
        origin,
    };

    let ui = Ui {
        rp_status: get_el("w-rp-status")?,
        rp_out: get_el("w-rp-out")?,

        panel_warn: get_el("w-panel-warn")?,
        warn_text: get_el("w-warn-text")?,

        cap_btn: get_btn("w-cap-btn")?.dyn_into()?,
        cap_status: get_el("w-cap-status")?,
        cap_out: get_el("w-cap-out")?,

        user_name: get_input("w-user-name")?,
        user_display: get_input("w-user-display")?,
        reg_btn: get_btn("w-reg-btn")?,
        reg_status: get_el("w-reg-status")?,
        reg_out: get_el("w-reg-out")?,
        reg_timer: get_el("w-reg-timer")?,

        cred_list: get_el("w-cred-list")?,
        cred_count: get_el("w-cred-count")?,

        auth_user: get_input("w-auth-user")?,
        auth_conditional: get_input("w-conditional")?,
        auth_btn: get_btn("w-auth-btn")?.dyn_into()?,
        auth_status: get_el("w-auth-status")?,
        auth_out: get_el("w-auth-out")?,
        auth_timer: get_el("w-auth-timer")?,
    };

    let app = Rc::new(App {
        ui,
        state: RefCell::new(State { credentials: Vec::new() }),
        config,
        opened_by_ip,
    });

    render_hint()?;
    render_warning_if_ip(&app);
    render_rp_config(&app);
    render_cred_list(&app);
    bind_buttons(&app)?;

    let app_c = app.clone();
    spawn_local(async move { do_check_capabilities(app_c).await });

    Ok(())
}

// ============================================================
//                    BIND
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.cap_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_check_capabilities(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.reg_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_register(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.auth_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_authenticate(app).await });
        });
    }
    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

async fn do_check_capabilities(app: Rc<App>) {
    set_status(&app.ui.cap_status, "warn", "проверка…");
    set_disabled(&app.ui.cap_btn, true);
    TimeoutFuture::new(0).await;

    let summary = capability::capability_summary().await;
    set_out(&app.ui.cap_out, &summary);

    if capability::is_available() {
        set_status(&app.ui.cap_status, "ok", "готово");
    } else {
        set_status(&app.ui.cap_status, "err", "недоступно");
    }
    set_disabled(&app.ui.cap_btn, false);
}

async fn do_register(app: Rc<App>) {
    // Если страница открыта по IP — сразу объясняем, почему не сработает.
    if app.opened_by_ip {
        set_status(&app.ui.reg_status, "err", "откройте по localhost");
        set_out(&app.ui.reg_out,
            "❌ WebAuthn не принимает IP-адрес как rp.id.\n\n\
             Откройте приложение по http://localhost:5173 — \
             тогда регистрация заработает.");
        return;
    }

    set_status(&app.ui.reg_status, "warn", "ожидание подтверждения…");
    set_disabled(&app.ui.reg_btn, true);
    set_disabled(&app.ui.auth_btn, true);
    TimeoutFuture::new(0).await;

    let user_name = app.ui.user_name.value();
    let display_name = app.ui.user_display.value();

    let user_id = match crate::web_crypto::random::random_bytes(16) {
        Ok(b) => b,
        Err(e) => {
            set_status(&app.ui.reg_status, "err", &e.message);
            set_disabled(&app.ui.reg_btn, false);
            return;
        }
    };
    let challenge = match crate::web_crypto::random::random_bytes(32) {
        Ok(b) => b,
        Err(e) => {
            set_status(&app.ui.reg_status, "err", &e.message);
            set_disabled(&app.ui.reg_btn, false);
            return;
        }
    };

    let cancel = start_timer(app.ui.reg_timer.clone(), 60);

    let user = RegistrationUser {
        id: &user_id,
        name: &user_name,
        display_name: &display_name,
    };

    match register::register_passkey(&app.config, &user, &challenge).await {
        Ok(result) => {
            cancel.set(true);
            app.ui.reg_timer.set_text_content(Some(""));

            let stored = StoredCredential {
                user_name: user_name.clone(),
                raw_id: result.raw_id.clone(),
                attachment: result.attachment.clone(),
                sign_count: 0,
                auth_count: 0,
            };
            app.state.borrow_mut().credentials.push(stored);
            render_cred_list(&app);
            enable_auth_if_ready(&app);

            let out = build_register_output(
                &user_id, &challenge, &result, &app.config,
            ).await;
            set_out(&app.ui.reg_out, &out);
            set_status(&app.ui.reg_status, "ok", "зарегистрирован");
        }
        Err(e) => {
            cancel.set(true);
            app.ui.reg_timer.set_text_content(Some(""));
            show_error(&app.ui.reg_status, &app.ui.reg_out, &e, "регистрация");
        }
    }
    set_disabled(&app.ui.reg_btn, false);
    enable_auth_if_ready(&app);
}

async fn do_authenticate(app: Rc<App>) {
    if app.opened_by_ip {
        set_status(&app.ui.auth_status, "err", "откройте по localhost");
        set_out(&app.ui.auth_out,
            "❌ Аутентификация также требует доменное имя, а не IP.\n\n\
             Откройте приложение по http://localhost:5173.");
        return;
    }

    set_status(&app.ui.auth_status, "warn", "ожидание подтверждения…");
    set_disabled(&app.ui.auth_btn, true);
    TimeoutFuture::new(0).await;

    let user_name = app.ui.auth_user.value();
    let use_conditional = app.ui.auth_conditional.checked();

    let allow: Vec<Vec<u8>> = {
        let st = app.state.borrow();
        st.credentials
            .iter()
            .filter(|c| c.user_name == user_name)
            .map(|c| c.raw_id.clone())
            .collect()
    };
    if allow.is_empty() {
        set_status(&app.ui.auth_status, "err", "нет credential");
        set_out(&app.ui.auth_out,
            "Сначала зарегистрируйте passkey (блок 3).");
        set_disabled(&app.ui.auth_btn, false);
        return;
    }

    let challenge = match crate::web_crypto::random::random_bytes(32) {
        Ok(b) => b,
        Err(e) => {
            set_status(&app.ui.auth_status, "err", &e.message);
            set_disabled(&app.ui.auth_btn, false);
            return;
        }
    };

    let cancel = start_timer(app.ui.auth_timer.clone(), 60);

    let result = if use_conditional {
        authenticate::authenticate_conditional(&app.config, &challenge).await
    } else {
        authenticate::authenticate_passkey(&app.config, &challenge, &allow).await
    };

    match result {
        Ok(result) => {
            cancel.set(true);
            app.ui.auth_timer.set_text_content(Some(""));

            // Достаём предыдущий счётчик из «БД» до обновления.
            let (prev_sign, prev_auths) = update_counter_and_get_prev(&app, &result.raw_id);

            // Парсим authenticatorData один раз — используем и для
            // записи нового signCount, и для рендера.
            let parsed_ad = parse_authenticator_data(&result.authenticator_data).ok();
            if let Some(ad) = parsed_ad {
                store_new_sign_count(&app, &result.raw_id, ad.sign_count);
            }

            let out = build_auth_output(
                &challenge,
                &result,
                &app.config,
                prev_sign,
                prev_auths,
                parsed_ad,
            ).await;
            set_out(&app.ui.auth_out, &out);
            render_cred_list(&app);
            set_status(&app.ui.auth_status, "ok", "успех");
        }
        Err(e) => {
            cancel.set(true);
            app.ui.auth_timer.set_text_content(Some(""));
            show_error(&app.ui.auth_status, &app.ui.auth_out, &e, "аутентификация");
        }
    }
    set_disabled(&app.ui.auth_btn, false);
}

/// Помечает credential использованным и возвращает
/// `(предыдущий signCount, предыдущий authCount)`.
fn update_counter_and_get_prev(
    app: &Rc<App>,
    raw_id: &[u8],
) -> (u32, u32) {
    let mut st = app.state.borrow_mut();
    for c in st.credentials.iter_mut() {
        if c.raw_id == raw_id {
            let prev_sign = c.sign_count;
            let prev_auths = c.auth_count;
            c.auth_count = c.auth_count.saturating_add(1);
            return (prev_sign, prev_auths);
        }
    }
    return (0, 0);
}

/// Записывает новый signCount в «БД» после разбора authenticatorData.
fn store_new_sign_count(
    app: &Rc<App>,
    raw_id: &[u8],
    sign_count: u32,
) {
    let mut st = app.state.borrow_mut();
    for c in st.credentials.iter_mut() {
        if c.raw_id == raw_id {
            c.sign_count = sign_count;
            return;
        }
    }
}

// ============================================================
//                    ФОРМИРОВАНИЕ ВЫВОДА
// ============================================================

async fn build_register_output(
    user_id: &[u8],
    challenge: &[u8],
    result: &crate::webauthn::model::RegistrationResult,
    config: &RpConfig,
) -> String {
    let mut out = String::with_capacity(1024);
    out.push_str("✅ Passkey зарегистрирован\n\n");

    out.push_str(&format_str(format_args!(
        "user.id ({} B): {}\n\n",
        user_id.len(),
        bytes_to_base64url(user_id),
    )));

    out.push_str("— отправлено —\n");
    out.push_str(&format_str(format_args!(
        "challenge ({} B, base64url):\n  {}\n",
        challenge.len(),
        bytes_to_base64url(challenge),
    )));
    out.push_str(&format_str(format_args!("rp.id: {}\n\n", config.rp_id)));

    out.push_str("— получено —\n");
    out.push_str(&format_str(format_args!(
        "rawId ({} B, base64url):\n  {}\n",
        result.raw_id.len(),
        bytes_to_base64url(&result.raw_id),
    )));
    out.push_str(&format_str(format_args!(
        "authenticatorAttachment: {}\n",
        result.attachment,
    )));
    out.push_str(&format_str(format_args!(
        "attestationObject ({} B, hex):\n  {}…\n\n",
        result.attestation_object.len(),
        hex_preview(&result.attestation_object, 32),
    )));

    out.push_str("— разбор clientDataJSON —\n");
    match parse_client_data(&result.client_data_json) {
        Ok(cd) => {
            out.push_str(&format_str(format_args!("type: {}\n", cd.kind)));
            out.push_str(&format_str(format_args!("origin: {}\n", cd.origin)));
            out.push_str(&format_str(format_args!(
                "crossOrigin: {}\n", cd.cross_origin,
            )));
            let our_challenge = bytes_to_base64url(challenge);
            let ok = cd.challenge == our_challenge;
            out.push_str(&format_str(format_args!(
                "challenge совпадает с отправленным: {}\n",
                if ok { "✅" } else { "❌" },
            )));
        }
        Err(e) => {
            out.push_str(&format_str(format_args!("ошибка разбора: {e}\n")));
        }
    }
    return out;
}

async fn build_auth_output(
    challenge: &[u8],
    result: &crate::webauthn::model::AuthResult,
    config: &RpConfig,
    prev_sign_count: u32,
    prev_auth_count: u32,
    parsed_ad: Option<AuthData>,
) -> String {
    let mut out = String::with_capacity(2048);

    // === clientDataJSON ===
    out.push_str("— разбор clientDataJSON —\n");
    let cd = parse_client_data(&result.client_data_json).ok();
    match &cd {
        Some(cd) => {
            out.push_str(&format_str(format_args!("type: {}\n", cd.kind)));
            out.push_str(&format_str(format_args!("origin: {}\n", cd.origin)));
            let our_challenge = bytes_to_base64url(challenge);
            let ok = cd.challenge == our_challenge;
            out.push_str(&format_str(format_args!(
                "challenge совпадает: {}\n\n",
                if ok { "✅" } else { "❌" },
            )));
        }
        None => out.push_str("ошибка разбора clientDataJSON\n\n"),
    }

    // === authenticatorData ===
    out.push_str("— разбор authenticatorData —\n");
    match parsed_ad {
        Some(ad) => {
            // Проверка rpIdHash == SHA-256(rp_id).
            let expected = digest::digest(HashAlg::Sha256, config.rp_id.as_bytes())
                .await
                .ok();
            match expected {
                Some(exp) => {
                    let ok = exp.as_slice() == ad.rp_id_hash.as_slice();
                    out.push_str(&format_str(format_args!(
                        "rpIdHash == SHA-256(\"{}\"): {}\n",
                        config.rp_id,
                        if ok { "✅ origin совпадает" } else { "❌ ЧУЖОЙ ORIGIN" },
                    )));
                }
                None => out.push_str("rpIdHash: не удалось посчитать SHA-256\n"),
            }

            out.push_str(&format_str(format_args!(
                "flags: UP={} UV={} BE={} BS={}\n",
                yn(ad.flags.user_present()),
                yn(ad.flags.user_verified()),
                yn(ad.flags.backup_eligible()),
                yn(ad.flags.backup_state()),
            )));

            out.push_str(&format_str(format_args!(
                "signCount: {} (было: {})\n",
                ad.sign_count, prev_sign_count,
            )));
            let counter_ok = ad.sign_count > prev_sign_count;
            if prev_auth_count == 0 {
                out.push_str("  первый вход — счётчик проверяется со следующего раза\n");
            } else if counter_ok {
                out.push_str("  ✅ счётчик вырос — аутентификатор подлинный\n");
            } else {
                out.push_str("  ❌ счётчик НЕ вырос — возможна клонированная копия!\n");
            }
        }
        None => {
            out.push_str("ошибка разбора authenticatorData\n");
        }
    }

    // === Подпись и userHandle ===
    out.push_str(&format_str(format_args!(
        "\nsignature ({} B):\n  {}…\n",
        result.signature.len(),
        hex_preview(&result.signature, 32),
    )));
    match &result.user_handle {
        Some(h) => out.push_str(&format_str(format_args!(
            "userHandle ({} B): {}\n",
            h.len(),
            bytes_to_base64url(h),
        ))),
        None => out.push_str("userHandle: — (non-discoverable credential)\n"),
    }
    return out;
}

fn yn(b: bool) -> &'static str {
    return if b { "✅" } else { "❌" };
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_hint() -> Result<(), JsValue> {
    let hint = get_el("w-hint")?;
    let html = "WebAuthn требует <b>HTTPS или localhost</b> и \
                <b>реального user gesture</b> (клик по кнопке). \
                Аутентификатор — настоящий: Touch ID, Windows Hello, \
                YubiKey. В headless-браузере диалог не появится, \
                но в обычном Chrome всё работает.<br>\
                Приватный ключ <b>никогда не покидает аутентификатор</b>: \
                браузер отдаёт только публичную часть и подпись.";
    hint.set_inner_html(html);
    Ok(())
}

/// Показывает предупреждение, если страница открыта по IP-адресу.
fn render_warning_if_ip(app: &Rc<App>) {
    if !app.opened_by_ip {
        return;
    }
    let window = web_sys::window().expect("no window");
    let actual_host = window.location().hostname().unwrap_or_default();
    let actual_port = window.location().port().unwrap_or_default();
    let link = if actual_port.is_empty() {
        "http://localhost".to_string()
    } else {
        format_str(format_args!("http://localhost:{actual_port}"))
    };
    let mut html = String::with_capacity(256);
    html.push_str("Вы открыли приложение по <b>");
    html.push_str(&esc_html(&actual_host));
    html.push_str("</b>, но WebAuthn запрещает IP-адреса в ");
    html.push_str("<code>rp.id</code>. Откройте приложение по ");
    html.push_str("<a href=\"");
    html.push_str(&link);
    html.push_str("\"><b>");
    html.push_str(&link);
    html.push_str("</b></a> — тогда регистрация passkey заработает.");
    app.ui.warn_text.set_inner_html(&html);
    let _ = app.ui.panel_warn
        .set_attribute("style", "display:block;");
}

fn render_rp_config(app: &Rc<App>) {
    let mut html = String::with_capacity(512);
    push_kv(&mut html, "rp.id", &app.config.rp_id, "");
    push_kv(&mut html, "rp.name", &app.config.rp_name, "");
    push_kv(&mut html, "origin", &app.config.origin, "");
    push_kv(&mut html, "timeout", "60 000 мс", "");
    app.ui.rp_out.set_inner_html(&html);
    set_status(&app.ui.rp_status, "", "готово");
}

fn render_cred_list(app: &Rc<App>) {
    let st = app.state.borrow();
    let count = st.credentials.len();
    app.ui.cred_count.set_text_content(Some(
        &format_str(format_args!("{count}")),
    ));
    if count == 0 {
        app.ui.cred_list.set_inner_html(
            "<div class=\"w-cred-empty\">пока никого</div>",
        );
        return;
    }
    let mut html = String::with_capacity(512);
    for c in st.credentials.iter() {
        let short = hex_preview(&c.raw_id, 6);
        html.push_str("<div class=\"w-cred-item\"><span class=\"name\">");
        html.push_str(&esc_html(&c.user_name));
        html.push_str("</span><span class=\"id\">");
        html.push_str(&esc_html(&short));
        html.push(' ');
        html.push_str(&esc_html(&c.attachment));
        html.push_str(" · sign=");
        html.push_str(&format_str(format_args!("{}", c.sign_count)));
        html.push_str(" · auth=");
        html.push_str(&format_str(format_args!("{}", c.auth_count)));
        html.push_str("</span></div>");
    }
    app.ui.cred_list.set_inner_html(&html);
}

fn enable_auth_if_ready(app: &Rc<App>) {
    let has_any = !app.state.borrow().credentials.is_empty();
    set_disabled(&app.ui.auth_btn, !has_any);
}

fn show_error(
    status: &web_sys::Element,
    out: &web_sys::Element,
    err: &WebAuthnError,
    ceremony: &str,
) {
    set_status(status, "err", "ошибка");
    let mut text = String::with_capacity(256);
    text.push_str(&format_str(format_args!("❌ {ceremony}: ")));
    text.push_str(&err.message);
    text.push_str("\n\n");
    text.push_str(err.user_hint());
    if err.kind == WebAuthnErrorKind::Cancelled {
        text.push_str(
            "\n\nЕсли вы в headless-браузере, диалог не появится — это ожидаемо.",
        );
    }
    set_out(out, &text);
}

/// Запускает таймер обратного отсчёта и возвращает cancel-токен.
///
/// Когда вызывающий завершает операцию, он выставляет `cancel.set(true)`
/// и очищает текст — цикл увидит флаг и корректно завершится.
fn start_timer(el: web_sys::Element, seconds: u32) -> Rc<Cell<bool>> {
    let cancel = Rc::new(Cell::new(false));
    let cancel_c = cancel.clone();
    let el_c = el.clone();
    spawn_local(async move {
        for remaining in (1..=seconds).rev() {
            if cancel_c.get() {
                return;
            }
            el_c.set_text_content(Some(
                &format_str(format_args!("⏱ {remaining} с")),
            ));
            let cls = if remaining <= 10 { "w-timer warn" } else { "w-timer" };
            el_c.set_class_name(cls);
            TimeoutFuture::new(1000).await;
        }
        if !cancel_c.get() {
            el_c.set_text_content(Some(""));
        }
    });
    return cancel;
}

fn push_kv(html: &mut String, k: &str, v: &str, cls: &str) {
    html.push_str("<span class=\"k\">");
    html.push_str(&esc_html(k));
    html.push_str("</span><span class=\"v");
    if !cls.is_empty() {
        html.push(' ');
        html.push_str(cls);
    }
    html.push_str("\">");
    html.push_str(&esc_html(v));
    html.push_str("</span>");
}

fn set_out(el: &web_sys::Element, text: &str) {
    el.set_text_content(Some(text));
    el.set_class_name("w-out");
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "w-badge ok",
        "err" => "w-badge err",
        "warn" => "w-badge warn",
        _ => "w-badge",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn set_disabled(el: &HtmlElement, disabled: bool) {
    if disabled {
        let _ = el.set_attribute("disabled", "");
    } else {
        let _ = el.remove_attribute("disabled");
    }
}

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}

/// Грубая проверка: IPv4 / IPv6 / пустая строка.
///
/// WebAuthn не принимает IP-адреса в качестве RP ID.
fn is_ip_address(host: &str) -> bool {
    if host.is_empty() {
        return false;
    }
    let all_digits_dots = host
        .chars()
        .all(|c| c.is_ascii_digit() || c == '.');
    if all_digits_dots && host.contains('.') {
        return true;
    }
    if host.contains(':') {
        return true;
    }
    return false;
}

// ============================================================
//                    ХЕЛПЕРЫ DOM
// ============================================================

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    return web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(
            &format_str(format_args!("no #{id}")),
        ));
}

fn get_btn(id: &str) -> Result<HtmlElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlElement"));
}

fn get_input(id: &str) -> Result<HtmlInputElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlInputElement"));
}

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut() + 'static,
{
    let cb = Closure::<dyn FnMut()>::new(f);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```

---

## 9. Команды запуска

```bash
# Сборка
npm run wasm:dev

# Тесты главы 41
wasm-pack test --chrome --headless -- --test wasm_webauthn

# Firefox
wasm-pack test --firefox --headless -- --test wasm_webauthn

# Демо
npm run dev
# Открыть http://localhost:5173 (не 127.0.0.1)
```

Ожидаемый результат: **17 passed, 2 ignored**.

---

## 10. Итог

WebAuthn — протокол, в котором браузер и аутентификатор берут на себя всю криптографию. Разработчик на Rust/WASM работает с «церемонией»: генерирует `challenge`, собирает опции, разбирает ответ. Ключевые уроки главы:

1. **`rp.id` — это домен, а не IP.** `127.0.0.1` даст `SecurityError` в Firefox и `InvalidDomainError` в Chrome. Единственное исключение — `localhost`.
2. **`web-sys` не экспортирует WebAuthn-типы без unstable-флага.** Используйте `Reflect::get` + `Function::call*` — это работает для любого JS API и не требует глобальных cfg.
3. **Разбор `authenticatorData` и `clientDataJSON` — ядро безопасности.** `rpIdHash` привязывает подпись к origin (anti-phishing), `signCount` обнаруживает клоны (anti-cloning), `challenge` защищает от replay. Без разбора всё это остаётся «магией».
4. **`challenge` — всегда криптостойкий.** `crypto.getRandomValues(32)` из главы 38, а не `Math.random()`. Иначе — атака повторного воспроизведения.
5. **Приватный ключ никогда не покидает аутентификатор.** Сервер хранит только публичный ключ и счётчик. Даже при полной компрометации БД атакующий не сможет выдать себя за пользователя.

Именно поэтому WebAuthn — редкий случай, когда безопасность достигается не добавлением кода, а использованием правильного протокола. Rust на клиенте пишет только «клей»: собрать опции, вызвать API, разобрать ответ, показать пользователю. Вся тяжёлая работа — на устройстве пользователя и на сервере.

# Глава 41 простыми словами

## Проблема, которую решает WebAuthn

Пароли — это компромисс, который человечество заключило с собой в 1960-х и с тех пор не пересмотрело. У каждого пароля есть три встроенных дефекта:

1. **Его можно подсмотреть или подобрать.**
2. **Его можно украсть с сервера** — если БД утекла, все пароли утекли.
3. **Его можно ввести на поддельном сайте** — фишинг.

WebAuthn — это способ вообще убрать пароль из уравнения. Вместо «секрета, который знают обе стороны» — **пара ключей**: приватный остаётся у пользователя в чипе устройства, публичный лежит на сервере.

---

## Аналогия

**Пароль** — как секретное рукопожатие. Ты приходишь в офис, делаешь три взмаха рукой, охранник отвечает тем же. Любой, кто подсмотрел рукопожатие, может повторить его в другом месте.

**Passkey** — как ключ от конкретной двери. У тебя ключ (приватный), у охранника — фотография ключа (публичный). Ты приходишь, прикладываешь ключ к замку, охранник смотрит — подходит или нет. Ключ нельзя «подсмотреть» или «повторить» — он физически в твоём кармане.

Фишка в том, что **ключ сделан под конкретную дверь** — домен `example.com`. Если ты случайно зашёл в поддельный офис на `evil.com`, твой ключ не подойдёт к их замку. Даже если ты сам захочешь — физически невозможно. Это встроено в протокол.

---

## Как это работает в браузере

**Регистрация:**
1. Сервер генерирует случайное число — `challenge` — и отправляет браузеру.
2. Браузер показывает системный диалог: «Подтверди Touch ID / Windows Hello / вставь YubiKey».
3. Ты прикладываешь палец. Устройство создаёт новую пару ключей: приватный остаётся в чипе, публичный отдаётся браузеру.
4. Браузер отправляет публичный ключ на сервер. Сервер сохраняет его в БД.

**Вход:**
1. Сервер генерирует свежий `challenge`.
2. Браузер показывает диалог. Ты прикладываешь палец.
3. Устройство подписывает `challenge` своим приватным ключом.
4. Сервер проверяет подпись публичным ключом из БД.

Всё. Никаких «секретов» по сети не ходит, ничего не хранится в браузере или на сервере.

---

## Три главные выгоды

### 1. Пароль нельзя украсть, потому что его нет

На сервере хранится только публичный ключ. Публичный ключ бесполезен без парного приватного. Если завтра БД утечёт, атакующий получит набор бесполезных чисел. Ему придётся физически украсть устройство каждого пользователя — а это, согласитесь, задача другого порядка.

**До:** «Мы храним хеши паролей — если утечка, все пароли скомпрометированы, нужно заставлять всех менять».
**После:** «Мы храним публичные ключи — если утечка, ничего не произошло».

### 2. Фишинг невозможен физически

Ключ привязан к домену. Passkey для `sberbank.ru` не сработает на `sberbank-online-evil.com`. Не потому что сайт «плохо выглядит», а потому что браузер сам откажется использовать ключ на чужом origin — это проверка на уровне протокола, которую нельзя обойти через JavaScript.

Классический фишинг («введи пароль вот здесь, я из банка») перестаёт работать. Даже если пользователь сам попадётся на удочку и введёт логин на поддельном сайте, пароля нет — нечего вводить.

### 3. UX становится приятнее

Никаких «придумайте пароль от 8 до 32 символов с заглавной и цифрой». Вместо этого — приложил палец, вошёл. Один раз зарегистрировался — работают все устройства с общим аккаунтом (iCloud Keychain, Google Password Manager).

Для разработчика это тоже приятно: уходит целый пласт проблем — забыл пароль, сброс пароля, письма для восстановления, требования к сложности, блокировка после N попыток.

---

## Где это уже работает

- **Apple ID, Google Account, Microsoft Account** — passkey включён по умолчанию с 2023.
- **GitHub, Cloudflare, PayPal, Shopify** — есть кнопка «Sign in with passkey».
- **Банки** (включая Сбер, Тинькофф) — используют для подтверждения операций.
- **Корпоративный SSO** (Okta, Azure AD, Google Workspace) — WebAuthn как второй фактор.
- **1Password, Bitwarden, Dashlane** — хранят passkeys пользователя.

Если открыть `myaccount.google.com` или `github.com/settings/security`, там уже есть раздел «Passkeys».

---

## Как применять в реальном проекте

### Этап 1 — добавить как второй фактор

Самый безопасный способ зайти на новую технологию: оставить пароли как есть, но добавить опцию «Включить вход по отпечатку». После пароля пользователь проходит регистрацию passkey. При следующем входе — либо пароль, либо passkey.

Плюсы:
- Ничего не ломается у тех, кто не хочет менять привычки.
- Пользователь сам решает, когда переходить.
- Если passkey сломается — всегда есть резервный путь.

### Этап 2 — сделать passkey основным

На странице входа — кнопка «Войти с passkey» вверху, «Войти с паролем» мелким шрифтом внизу. Ещё лучше — conditional mediation: браузер сам предлагает passkey в поле логина через autofill, пользователь просто кликает по подсказке.

### Этап 3 — отказаться от пароля

Для новых пользователей пароль вообще не спрашивают — только регистрация passkey. Пароль остаётся только у старых аккаунтов и как fallback, если у пользователя нет ни одного устройства с WebAuthn.

---

## Что важно помнить при внедрении

**Localhost ≠ 127.0.0.1.** При разработке обязательно открывать приложение по `localhost`, а не по IP-адресу — WebAuthn не принимает IP в `rp.id`. Это та самая ошибка, которую мы разбирали в главе.

**Нужны recovery-коды.** Если пользователь потерял все устройства с passkey — он не войдёт. Обязательно предложить при регистрации сохранить коды восстановления (как делает Google). Это тот же принцип, что с двухфакторной аутентификацией.

**Серверная часть — не мелочь.** Клиент только собирает и отправляет данные. Вся проверка на сервере:
- Распарсить CBOR из `attestationObject`.
- Проверить `challenge` совпадает с выданным.
- Проверить `origin` в `clientDataJSON` — наш домен.
- Проверить `rpIdHash == SHA-256(rp.id)`.
- Проверить подпись публичным ключом.
- Проверить `signCount` растёт (защита от клонирования).

На Rust для этого есть готовая библиотека [`webauthn-rs`](https://github.com/kanidm/webauthn-rs) от команды Kanidm — она реализует всю серверную сторону спецификации. Писать парсер CBOR и проверку подписи руками — плохая идея.

**Cloud-synced vs hardware.** Passkey может быть «облачным» (iCloud Keychain, Google) — синхронизируется между устройствами пользователя. Или «аппаратным» (YubiKey) — только на физическом ключе. Флаги `BE` / `BS` в `authenticatorData` показывают, что именно. Для банка может быть важно требовать именно аппаратный ключ, для обычного сервиса — не важно.

---

## Ограничения

**Не работает без пользователя.** Touch ID требует живого пальца. Для сервер-к-серверу аутентификации, IoT, headless-систем — не подходит.

**Не работает в iframe.** Многие банки встраивают логин в чужой сайт — WebAuthn это запрещает без явного разрешения.

**Не везде одинаково.** Safari поддерживает с 14-й версии, Chrome с 67-й, Firefox с 60-й. Conditional mediation — только Chrome / Edge / Safari 16+.

**Восстановление — боль.** Если пользователь потерял все устройства — единственный путь назад через recovery-коды или службу поддержки. Здесь нет «забыл пароль → письмо на почту».

---

## Короткий итог

**WebAuthn — это про то, чтобы у пользователя не было секрета, который можно украсть.**

Приватный ключ — только у него, в чипе. Публичный — на сервере, бесполезен без приватного. Домен привязан к ключу — фишинг не работает физически. Биометрия вместо пароля — UX на порядок лучше.

В современном веб-приложении это уже не «экспериментальная фича», а ожидаемая функциональность. Пользователи Apple и Google привыкли к passkeys. Банки внедряют. GitHub поддерживает. Через 2–3 года отсутствие passkey будет восприниматься как отсутствие HTTPS — как признак, что сервис отстал.

Начать можно с малого: добавить passkey как второй фактор к существующим паролям. Это 3–4 дня работы и сразу закрывает целый класс атак.


Внедрение WebAuthn-аутентификации (passkeys) в новое веб-приложение — это процесс, который можно и нужно разбить на логические этапы. Главное, что стоит помнить с самого начала: в отличие от паролей, здесь **нет секрета, который можно сбросить на сервере** — приватный ключ пользователя никогда не покидает его устройство. Поэтому восстановление доступа — это не сброс пароля, а повторное доказательство владения аккаунтом, и продумать этот сценарий нужно **на старте, а не когда пользователи начнут терять доступ**.

Вот пошаговый план действий.

### 📝 Этап 1: Подготовка и планирование

Прежде чем писать код, нужно принять несколько архитектурных решений.

*   **Определите `Relying Party ID` (rp.id):** Это домен вашего приложения (например, `example.com`). **Важно:** `rp.id` не может быть IP-адресом. Для локальной разработки используйте `localhost`, а не `127.0.0.1`, так как WebAuthn не принимает IP-адреса.
*   **Выберите стратегию внедрения:** Самый безопасный путь — начать с passkeys как **дополнительного фактора**. Оставьте пароли для входа, но добавьте опцию «Войти с помощью отпечатка/лица». Это позволит пользователям привыкнуть к новой технологии без риска потерять доступ к аккаунту. Со временем, когда adoption вырастет, можно сделать passkeys основным способом входа, а пароль — резервным.
*   **Спроектируйте систему восстановления:** Это критически важный шаг. Вам нужно предложить пользователям как минимум один надежный резервный способ входа. Идеальный вариант — **зарегистрировать второй passkey** (например, на планшете или аппаратном ключе). Если это невозможно, используйте **одноразовые резервные коды**, которые пользователь сохраняет офлайн. Резервные коды должны быть хешированы на сервере (например, с помощью Argon2id) перед сохранением.

### 🖥️ Этап 2: Реализация на стороне сервера (Rust)

На бэкенде вы будете выступать в роли Relying Party, который генерирует `challenge` и проверяет ответы от аутентификатора. Для этого не нужно писать парсеры и криптографию вручную — используйте проверенную библиотеку.

*   **Библиотека:** Рекомендую **`webauthn-rs`**. Это зрелая библиотека, которая реализует всю серверную логику FIDO2 и прошла аудит безопасности.
*   **Эндпоинты для регистрации (create):**
    1.  `POST /register/begin`: Сервер генерирует `challenge` (случайные 32 байта) и сохраняет его в сессии пользователя. Возвращает JSON с опциями для `navigator.credentials.create()`.
    2.  `POST /register/finish`: Клиент отправляет `attestationObject` и `clientDataJSON`. Сервер вызывает метод библиотеки для проверки: `rp_id`, `origin`, `challenge` и подписи. При успехе сохраняет `credential_id` (rawId) и **публичный ключ** в БД, привязав его к пользователю.
*   **Эндпоинты для аутентификации (get):**
    1.  `POST /login/begin`: Сервер генерирует новый `challenge` и возвращает опции для `navigator.credentials.get()`, включая список `allowCredentials` (ID ключей пользователя).
    2.  `POST /login/finish`: Клиент отправляет `authenticatorData`, `clientDataJSON` и `signature`. Сервер проверяет подпись публичным ключом из БД и `signCount` (счетчик подписей для защиты от клонирования).
*   **Совет:** Храните в БД не только публичный ключ, но и `signCount`, а также флаги `BE` (Backup Eligible) и `BS` (Backup State), чтобы понимать, является ли ключ синхронизируемым (например, iCloud Keychain) или аппаратным (YubiKey).

### 🌐 Этап 3: Реализация на стороне клиента (WASM + JS)

Клиентская часть отвечает за вызов WebAuthn API и передачу данных на сервер.

1.  **Проверка поддержки:** Перед показом кнопки входа проверьте `window.PublicKeyCredential`. Если функция недоступна, предложите пользователю традиционный вход по паролю.
2.  **Кодирование/декодирование:** WebAuthn использует `ArrayBuffer` для бинарных данных (challenge, ID). Для передачи по JSON их нужно конвертировать в base64url. На сервере — обратно. На Rust/WASM для этого удобно использовать функции из главы.
3.  **Регистрация:**
    *   Получите опции с сервера.
    *   Преобразуйте `challenge` из base64url в `Uint8Array`.
    *   Вызовите `navigator.credentials.create({ publicKey: options })`.
    *   Отправьте результат на сервер.
4.  **Аутентификация:**
    *   Получите опции с сервера.
    *   Вызовите `navigator.credentials.get({ publicKey: options })`.
    *   Отправьте результат на сервер.
5.  **Продвинутый UX (Conditional UI):** Чтобы passkey появлялись в автозаполнении прямо в поле ввода логина, добавьте атрибут `autocomplete="webauthn"` к полю и вызывайте `get()` с опцией `mediation: 'conditional'`. Это значительно повышает конверсию входа.

### 🧪 Этап 4: Тестирование

*   **Ручное тестирование:** Проверьте оба сценария (регистрация и вход) на разных устройствах: Mac (Touch ID), Windows (Windows Hello), Android (отпечаток), iOS (Face ID), а также с аппаратным ключом (YubiKey).
*   **Автоматизированное тестирование:** Используйте `wasm-bindgen-test` для написания тестов на Rust/WASM. Тесты, требующие реального взаимодействия с пользователем (появление диалога), нужно помечать `#[ignore]` и запускать вручную.

### 🚀 Этап 5: Внедрение и мониторинг

*   **Начните с малого:** Запустите passkeys для нового потока регистрации, но оставьте пароль как fallback для существующих пользователей.
*   **Прогрессивные подсказки:** Предлагайте пользователям настроить passkey в моменты, когда они уже вошли в систему (например, в настройках безопасности или после успешного входа по паролю). Это дает хорошую конверсию.
*   **Объясняйте выгоду:** Используйте язык, понятный пользователю: «Входите быстрее», «Защита от фишинга», «Не нужно помнить пароль».
*   **Мониторинг:** Отслеживайте количество успешных регистраций и входов, а также ошибок. Собирайте обратную связь.

Начать можно с малого — добавить passkey как дополнительный фактор. Это займет несколько дней, но сразу закроет целый класс атак, связанных с кражей паролей. Если появятся вопросы на каком-то из этапов, спрашивайте.