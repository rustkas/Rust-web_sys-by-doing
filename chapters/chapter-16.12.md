# Глава 16.12. API client: паттерны и архитектура

## Обновлённая структура проекта

```
.
├── src/
│   ├── api_client.rs               ← НОВЫЙ: ApiClient с refresh, timeout, logging,
│   │                                  rate limiting, кэшированием
│   ├── lib.rs                      ← ОБНОВЛЁН: +pub mod api_client;
│   ├── fetch.rs                    ← ОБНОВЛЁН: +FetchError::Timeout + is_retryable метод
│   ├── app.rs                      ← ОБНОВЛЁН: секция 12 + helper describe_error
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_api_client.rs          ← НОВЫЙ: 8 тестов ApiClient
│   └── (остальные без изменений)
│
└── (корневые файлы без изменений)
```

**Краткий комментарий.** За одиннадцать глав мы накопили солидный арсенал: `get_json`, `post_json`, классификация ошибок, retry с backoff, refresh токенов, timeout, параллельные запросы, чтение потока. Но каждый вызов в демо-приложении всё ещё собирает запрос вручную — повторяет `RequestInit`, `set_mode`, `set_method`, проверки статуса. Это нормально для учебных примеров, но не для реального кода. В этой главе мы соберём всё в одну структуру — `ApiClient` — которая инкапсулирует base URL, общие заголовки, токен с прозрачным refresh, retry, timeout через `AbortController`, логирование, обработку 429 с `Retry-After` и кэширование GET-ответов. Каждый запрос станет одной строкой, а обработка ошибок — единой точкой.

## Ссылки для детального изучения

- **Rust API Guidelines: Builder pattern** — как правильно делать конфигурируемые структуры: [rust-lang.github.io/api-guidelines/type-safety.html#builders-enable-construction-of-complex-values-c-builder](https://rust-lang.github.io/api-guidelines/type-safety.html#builders-enable-construction-of-complex-values-c-builder)
- **Rust Design Patterns: Newtype** — зачем оборачивать типы: [rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html](https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html)
- **`reqwest::Client`** — как устроен production HTTP-клиент в Rust (для сравнения): [docs.rs/reqwest/latest/reqwest/struct.Client.html](https://docs.rs/reqwest/latest/reqwest/struct.Client.html)
- **`gloo-net`** — готовый HTTP-клиент для WASM (альтернатива нашему `ApiClient`): [docs.rs/gloo-net](https://docs.rs/gloo-net)
- **MDN: `Retry-After`** — заголовок, которым сервер сообщает, когда вернуться: [developer.mozilla.org/ru/docs/Web/HTTP/Headers/Retry-After](https://developer.mozilla.org/ru/docs/Web/HTTP/Headers/Retry-After)
- **MDN: `AbortController`** — как реализован timeout в клиенте: [developer.mozilla.org/ru/docs/Web/API/AbortController](https://developer.mozilla.org/ru/docs/Web/API/AbortController)
- **AWS: Exponential Backoff and Jitter** — почему retry-задержка рандомизируется: [aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- **Martin Fowler: Parameter Object** — почему длинные списки параметров плохи: [refactoring.com/catalog/introduceParameterObject.html](https://refactoring.com/catalog/introduceParameterObject.html)

## Новые и обновлённые файлы

### `src/fetch.rs` — добавить вариант `Timeout`

Найдите `pub enum FetchError` и добавьте вариант:

```rust
#[derive(Debug, Clone)]
pub enum FetchError {
    Network(String),
    Http(u16, String),
    Aborted,
    /// Таймаут: сервер не ответил за указанное количество миллисекунд.
    Timeout(u32),
    Parse(String),
}
```

Обновите `impl FetchError`: метод `user_message` — добавьте ветку для `Timeout`, а метод `is_retryable` **добавьте как метод**, обернув свободную функцию:

```rust
impl FetchError {
    pub fn user_message(&self) -> String {
        match self {
            FetchError::Network(msg) => {
                return format!("Сетевая ошибка: нет соединения с сервером. ({msg})")
            }
            FetchError::Http(status, _) if *status >= 500 => {
                return format!("Ошибка сервера: {status}. Попробуйте позже.")
            }
            FetchError::Http(status, _) if *status == 404 => {
                return format!("Ресурс не найден: {status}.")
            }
            FetchError::Http(status, _) => {
                return format!("Запрос отклонён: {status}.")
            }
            FetchError::Aborted => "Запрос отменён.".to_string(),
            FetchError::Timeout(ms) => format!("Таймаут: сервер не ответил за {ms} мс."),
            FetchError::Parse(msg) => format!("Ошибка разбора ответа: {msg}"),
        }
    }

    /// Метод-обёртка над свободной функцией `is_retriable`.
    ///
    /// Позволяет писать `error.is_retryable()` вместо
    /// `fetch::is_retriable(&error)`. Обратите внимание на написание:
    /// метод — `is_retryable` (через «а»), функция — `is_retriable`
    /// (через «и»). Это исторически сложилось: функция появилась
    /// раньше, а метод добавлен для удобства.
    pub fn is_retryable(&self) -> bool {
        is_retriable(self)
    }
}
```

Свободная функция `is_retriable` — добавьте ветку `Timeout`:

```rust
pub fn is_retriable(error: &FetchError) -> bool {
    match error {
        FetchError::Network(_) => true,
        FetchError::Http(status, _) => *status >= 500 || *status == 429 || *status == 408,
        FetchError::Aborted => false,
        FetchError::Timeout(_) => true, // таймаут временный — повторяем
        FetchError::Parse(_) => false,
    }
}
```

### `src/api_client.rs` (новый)

```rust
//! API-клиент: единая точка для всех HTTP-запросов.
//!
//! Инкапсулирует:
//! - базовый URL;
//! - общие заголовки (Accept, Authorization, Content-Type);
//! - хранилище токенов с прозрачным refresh при 401;
//! - конфигурацию retry и timeout (через AbortController);
//! - кэш GET-ответов с TTL;
//! - обработку 429 с заголовком Retry-After;
//! - опциональное логирование запросов.

use std::cell::{Cell, RefCell};
use std::collections::HashMap;
use std::rc::Rc;

use serde::de::DeserializeOwned;
use serde::Serialize;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::auth::TokenStoreHandle;
use crate::fetch::{self, FetchError, RetryConfig};
use crate::log;

/// Запись в кэше GET-ответов.
#[derive(Clone)]
struct CachedEntry {
    body: String,
    expires_at: f64,
}

/// HTTP-клиент с состоянием.
#[derive(Clone)]
pub struct ApiClient {
    base_url: String,
    default_headers: Vec<(String, String)>,
    token_store: Option<TokenStoreHandle>,
    retry_config: RetryConfig,
    timeout_ms: Option<u32>,
    verbose: bool,
    cache: Rc<RefCell<HashMap<String, CachedEntry>>>,
    cache_ttl_ms: u64,
    rate_limit_until: Rc<Cell<f64>>,
}

impl ApiClient {
    /// Создаёт клиент с базовым URL и настройками по умолчанию.
    pub fn new(base_url: &str) -> Self {
        Self {
            base_url: base_url.trim_end_matches('/').to_string(),
            default_headers: vec![("Accept".to_string(), "application/json".to_string())],
            token_store: None,
            retry_config: RetryConfig::default(),
            timeout_ms: None,
            verbose: false,
            cache: Rc::new(RefCell::new(HashMap::new())),
            cache_ttl_ms: 0,
            rate_limit_until: Rc::new(Cell::new(0.0)),
        }
    }

    /// Устанавливает токен напрямую (без хранилища).
    ///
    /// При 401 клиент **не** сможет обновить токен — для этого нужен
    /// `with_token_store`.
    pub fn with_token(mut self, token: &str) -> Self {
        let store = crate::auth::TokenStore::new();
        store.borrow_mut().set_tokens(token.to_string(), None);
        self.token_store = Some(store);
        self
    }

    /// Передаёт клиенту хранилище токенов.
    ///
    /// Если задано, клиент при получении 401 попробует обновить токен
    /// через `auth::try_refresh` и повторит запрос один раз.
    pub fn with_token_store(mut self, store: TokenStoreHandle) -> Self {
        self.token_store = Some(store);
        self
    }

    /// Убирает токен (logout).
    pub fn clear_token(&mut self) {
        if let Some(store) = &self.token_store {
            store.borrow_mut().clear();
        }
    }

    /// Добавляет общий заголовок ко всем запросам.
    pub fn with_header(mut self, name: &str, value: &str) -> Self {
        self.default_headers
            .push((name.to_string(), value.to_string()));
        self
    }

    /// Устанавливает заголовок, заменяя прежние с таким именем.
    pub fn set_header(&mut self, name: &str, value: &str) {
        self.default_headers
            .retain(|(n, _)| !n.eq_ignore_ascii_case(name));
        self.default_headers
            .push((name.to_string(), value.to_string()));
    }

    /// Устанавливает конфигурацию retry.
    pub fn with_retry(mut self, config: RetryConfig) -> Self {
        self.retry_config = config;
        self
    }

    /// Устанавливает таймаут в миллисекундах.
    ///
    /// Таймаут реализован через `AbortController` — запрос **реально
    /// отменяется**, а не просто прекращается ожидание. Ошибка отмены
    /// возвращается как `FetchError::Timeout`.
    pub fn with_timeout(mut self, ms: u32) -> Self {
        self.timeout_ms = Some(ms);
        self
    }

    /// Включает логирование запросов через `crate::log!`.
    pub fn with_logging(mut self) -> Self {
        self.verbose = true;
        self
    }

    /// Включает кэширование GET-ответов.
    ///
    /// `ttl_ms` — время жизни записи в миллисекундах. `0` отключает
    /// кэш. Кэшируются только успешные GET-ответы. Ключ — полный URL.
    pub fn with_cache(mut self, ttl_ms: u64) -> Self {
        self.cache_ttl_ms = ttl_ms;
        self
    }

    /// Базовый URL.
    pub fn base_url(&self) -> &str {
        &self.base_url
    }

    // ============================================================
    // Сборка URL и запроса
    // ============================================================

    fn full_url(&self, path: &str) -> String {
        if path.starts_with("http://") || path.starts_with("https://") {
            return path.to_string();
        }

        let path_with_slash: String = if path.starts_with('/') {
            path.to_string()
        } else {
            return format!("/{path}");
        };

        return format!("{}{}", self.base_url, path_with_slash);
    }

    /// Читает Retry-After из ответа (в миллисекундах).
    fn parse_retry_after(response: &web_sys::Response) -> Option<u64> {
        let value = response.headers().get("Retry-After").ok().flatten()?;
        value.trim().parse::<u64>().ok().map(|s| s * 1000)
    }

    /// Ждёт, если активен rate limit.
    async fn wait_for_rate_limit(&self) {
        let now = js_sys::Date::now();
        let until = self.rate_limit_until.get();
        if now < until {
            let wait_ms = (until - now).max(0.0) as u32;
            if self.verbose {
                log!("ApiClient: rate limit активен, ждём {wait_ms} мс");
            }
            gloo_timers::future::TimeoutFuture::new(wait_ms).await;
        }
    }

    // ============================================================
    // Отправка одной попытки
    // ============================================================

    async fn send_once(
        &self,
        method: &str,
        path: &str,
        body: Option<&str>,
        extra_headers: &[(&str, &str)],
    ) -> Result<web_sys::Response, FetchError> {
        let url = self.full_url(path);

        let opts = web_sys::RequestInit::new();
        opts.set_method(method);
        opts.set_mode(web_sys::RequestMode::Cors);

        if let Some(b) = body {
            opts.set_body(&JsValue::from_str(b));
        }

        // --- Timeout через AbortController ---
        let controller = if self.timeout_ms.is_some() {
            Some(
                web_sys::AbortController::new()
                    .map_err(fetch::classify_error)?,
            )
        } else {
            None
        };

        let timeout_id: Option<i32> = match (self.timeout_ms, &controller) {
            (Some(ms), Some(ctrl)) => {
                let ctrl_clone = ctrl.clone();
                let callback = Closure::once_into_js(move || {
                    ctrl_clone.abort();
                });
                let window = web_sys::window()
                    .ok_or_else(|| FetchError::Network("no window".into()))?;
                let id = window
                    .set_timeout_with_callback_and_timeout_and_arguments_0(
                        callback.unchecked_ref(),
                        ms as i32,
                    )
                    .map_err(fetch::classify_error)?;
                Some(id)
            }
            _ => None,
        };

        if let Some(ctrl) = &controller {
            opts.set_signal(Some(&ctrl.signal()));
        }

        let request = web_sys::Request::new_with_str_and_init(&url, &opts)
            .map_err(fetch::classify_error)?;

        // Общие заголовки.
        for (name, value) in &self.default_headers {
            request
                .headers()
                .set(name, value)
                .map_err(fetch::classify_error)?;
        }

        // Authorization из хранилища.
        if let Some(store) = &self.token_store {
            if let Some(token) = store.borrow().access_token.clone() {
                request
                    .headers()
                    .set("Authorization", &format!("Bearer {token}"))
                    .map_err(fetch::classify_error)?;
            }
        }

        // Заголовки конкретного запроса (перезаписывают общие).
        for (name, value) in extra_headers {
            request
                .headers()
                .set(name, value)
                .map_err(fetch::classify_error)?;
        }

        if self.verbose {
            log!("ApiClient: {method} {url}");
        }

        // --- Отправка ---
        let window = web_sys::window()
            .ok_or_else(|| FetchError::Network("no window".into()))?;

        let result = JsFuture::from(window.fetch_with_request(&request)).await;

        // Снимаем таймер — запрос завершился (успешно или нет).
        if let Some(id) = timeout_id {
            if let Some(w) = web_sys::window() {
                w.clear_timeout_with_handle(id);
            }
        }

        let value = match result {
            Ok(v) => v,
            Err(e) => {
                // Если контроллер отменён и у нас был таймаут — значит, это timeout.
                if let (Some(ms), Some(ctrl)) = (self.timeout_ms, &controller) {
                    if ctrl.signal().aborted() {
                        if self.verbose {
                            log!("ApiClient: timeout {ms} мс для {url}");
                        }
                        return Err(FetchError::Timeout(ms));
                    }
                }
                return Err(fetch::classify_error(e));
            }
        };

        let response: web_sys::Response = value
            .dyn_into()
            .map_err(fetch::classify_error)?;

        if self.verbose {
            log!("ApiClient: {} {} → {}", method, url, response.status());
        }

        // --- Rate limit ---
        if response.status() == 429 {
            if let Some(ms) = Self::parse_retry_after(&response) {
                let now = js_sys::Date::now();
                self.rate_limit_until.set(now + ms as f64);
                if self.verbose {
                    log!("ApiClient: 429, Retry-After = {ms} мс");
                }
            }
        }

        Ok(response)
    }

    // ============================================================
    // Retry-цикл
    // ============================================================

    async fn try_with_retry(
        &self,
        method: &str,
        path: &str,
        body: Option<&str>,
        extra_headers: &[(&str, &str)],
    ) -> Result<web_sys::Response, FetchError> {
        let is_idempotent = matches!(method, "GET" | "HEAD" | "PUT" | "DELETE");
        let mut last_error: Option<FetchError> = None;

        for attempt in 0..self.retry_config.max_attempts {
            match self.send_once(method, path, body, extra_headers).await {
                Ok(resp) => {
                    if resp.ok() {
                        return Ok(resp);
                    }

                    let status = resp.status();
                    let is_retriable_status = status >= 500 || status == 429 || status == 408;

                    if !is_idempotent || !is_retriable_status {
                        return Ok(resp);
                    }

                    let is_last = attempt + 1 >= self.retry_config.max_attempts;
                    if is_last {
                        return Ok(resp);
                    }

                    let delay = Self::parse_retry_after(&resp)
                        .unwrap_or_else(|| fetch::calculate_backoff(attempt, &self.retry_config));

                    // Поглощаем тело, чтобы не оставлять висеть ответ.
                    let _ = fetch::read_error_body(&resp).await;

                    gloo_timers::future::TimeoutFuture::new(delay.min(u32::MAX as u64) as u32)
                        .await;
                }
                Err(error) => {
                    let is_last = attempt + 1 >= self.retry_config.max_attempts;
                    let should_retry = is_idempotent && fetch::is_retriable(&error);

                    if !should_retry || is_last {
                        return Err(error);
                    }

                    let delay = fetch::calculate_backoff(attempt, &self.retry_config);
                    gloo_timers::future::TimeoutFuture::new(delay.min(u32::MAX as u64) as u32)
                        .await;

                    last_error = Some(error);
                }
            }
        }

        Err(last_error.unwrap_or_else(|| FetchError::Network("retry loop exhausted".into())))
    }

    // ============================================================
    // Верхний уровень: rate limit + refresh + HTTP-ошибки
    // ============================================================

    async fn send(
        &self,
        method: &str,
        path: &str,
        body: Option<&str>,
        extra_headers: &[(&str, &str)],
    ) -> Result<web_sys::Response, FetchError> {
        self.wait_for_rate_limit().await;

        let resp = self
            .try_with_retry(method, path, body, extra_headers)
            .await?;

        // Прозрачный refresh токена при 401.
        if resp.status() == 401 {
            if let Some(store) = &self.token_store {
                let refresh_token_present = store.borrow().refresh_token.is_some();
                if refresh_token_present {
                    if self.verbose {
                        log!("ApiClient: 401, пробуем refresh");
                    }
                    match crate::auth::try_refresh(store).await {
                        Ok(true) => {
                            let resp2 = self
                                .try_with_retry(method, path, body, extra_headers)
                                .await?;
                            return self.finalize(resp2).await;
                        }
                        Ok(false) => {
                            if self.verbose {
                                log!("ApiClient: refresh не удался");
                            }
                        }
                        Err(e) => {
                            if self.verbose {
                                log!("ApiClient: ошибка refresh: {:?}", e);
                            }
                        }
                    }
                }
            }
        }

        self.finalize(resp).await
    }

    /// Преобразует `!ok` в `FetchError::Http`, `ok` — возвращает как есть.
    async fn finalize(&self, resp: web_sys::Response) -> Result<web_sys::Response, FetchError> {
        if resp.ok() {
            return Ok(resp);
        }
        let status = resp.status();
        let body = fetch::read_error_body(&resp).await;
        Err(FetchError::Http(status, body))
    }

    // ============================================================
    // Разбор тела
    // ============================================================

    async fn parse_json<T: DeserializeOwned>(body: &str) -> Result<T, FetchError> {
        serde_json::from_str(body).map_err(|e| FetchError::Parse(format!("deserialize error: {e}")))
    }

    // ============================================================
    // Публичные методы
    // ============================================================

    /// GET-запрос с десериализацией JSON.
    pub async fn get<T: DeserializeOwned>(&self, path: &str) -> Result<T, FetchError> {
        let url = self.full_url(path);

        // Проверяем кэш.
        if self.cache_ttl_ms > 0 {
            let now = js_sys::Date::now();
            let cache = self.cache.borrow();
            if let Some(entry) = cache.get(&url) {
                if now < entry.expires_at {
                    if self.verbose {
                        log!("ApiClient: cache hit для {url}");
                    }
                    return Self::parse_json::<T>(&entry.body).await;
                }
            }
        }

        let resp = self.send("GET", path, None, &[]).await?;
        let body = fetch::read_error_body(&resp).await;

        if self.cache_ttl_ms > 0 {
            let now = js_sys::Date::now();
            let entry = CachedEntry {
                body: body.clone(),
                expires_at: now + self.cache_ttl_ms as f64,
            };
            self.cache.borrow_mut().insert(url, entry);
        }

        Self::parse_json::<T>(&body).await
    }

    /// GET-запрос без разбора тела.
    pub async fn get_raw(&self, path: &str) -> Result<web_sys::Response, FetchError> {
        self.send("GET", path, None, &[]).await
    }

    /// POST-запрос с JSON-телом.
    pub async fn post<T: DeserializeOwned, B: Serialize>(
        &self,
        path: &str,
        body: &B,
    ) -> Result<T, FetchError> {
        let body_str = serde_json::to_string(body)
            .map_err(|e| FetchError::Parse(format!("serialize error: {e}")))?;

        let resp = self
            .send(
                "POST",
                path,
                Some(&body_str),
                &[("Content-Type", "application/json")],
            )
            .await?;

        let body_text = fetch::read_error_body(&resp).await;
        Self::parse_json::<T>(&body_text).await
    }

    /// PUT-запрос с JSON-телом.
    pub async fn put<T: DeserializeOwned, B: Serialize>(
        &self,
        path: &str,
        body: &B,
    ) -> Result<T, FetchError> {
        let body_str = serde_json::to_string(body)
            .map_err(|e| FetchError::Parse(format!("serialize error: {e}")))?;

        let resp = self
            .send(
                "PUT",
                path,
                Some(&body_str),
                &[("Content-Type", "application/json")],
            )
            .await?;

        let body_text = fetch::read_error_body(&resp).await;
        Self::parse_json::<T>(&body_text).await
    }

    /// DELETE-запрос.
    pub async fn delete(&self, path: &str) -> Result<web_sys::Response, FetchError> {
        self.send("DELETE", path, None, &[]).await
    }

    /// Сбрасывает кэш.
    pub fn clear_cache(&self) {
        self.cache.borrow_mut().clear();
    }
}
```

### `src/lib.rs` (обновлён)

Добавить одну строку:

```rust
pub mod api_client;
```

### `src/app.rs` — импорт и helper `describe_error`

**1. Обновите импорт:**

```rust
use crate::{api_client, auth, dom, fetch, log};
```

**2. Добавьте helper в конец файла, рядом с `truncate_preview`:**

```rust
/// Возвращает человекочитаемое описание ошибки с эмодзи и подсказкой.
///
/// Используется в обработчиках для замены технического `{:?}` на
/// осмысленное сообщение. Для мест, где ошибка приходит как `JsValue`,
/// сначала вызывайте `fetch::classify_error(e)`, потом передавайте
/// результат сюда.
fn describe_error(e: &fetch::FetchError) -> String {
    match e {
        fetch::FetchError::Network(msg) => {
            return format!("🌐 Сеть: {msg}")
        }
        fetch::FetchError::Http(401, _) => "🔒 Требуется вход".to_string(),
        fetch::FetchError::Http(403, _) => "🚫 Доступ запрещён".to_string(),
        fetch::FetchError::Http(404, _) => "🔍 Не найдено".to_string(),
        fetch::FetchError::Http(status, _) if *status >= 500 => {
            return format!("💥 Сервер сломался (HTTP {status})")
        }
        fetch::FetchError::Http(status, _) => format!("⚠️ HTTP {status}"),
        fetch::FetchError::Aborted => "🛑 Отменено".to_string(),
        fetch::FetchError::Timeout(ms) => format!("⏱ Таймаут {ms} мс"),
        fetch::FetchError::Parse(msg) => format!("📄 Ошибка разбора: {msg}"),
    }
}
```

### `src/app.rs` — секция 12 «API Client»

После секции 11:

```rust
    // ============================================================
    // СЕКЦИЯ 12: API Client
    // ============================================================

    let section12 = build_section(&container, "12. API Client")?;

    let intro12 = dom::create_paragraph(
        "API-клиент инкапсулирует base URL, общие заголовки, токен \
         с прозрачным refresh, retry, timeout через AbortController, \
         логирование, обработку 429 и кэширование GET-ответов. \
         Каждый запрос — одна строка: client.get::<T>(\"/path\").await. \
         Вызывающий код занимается бизнес-логикой, а не сборкой \
         RequestInit.",
    )?;
    dom::add_class(&dom::as_html(&intro12), "hint")?;
    dom::append(&section12, &intro12)?;

    let client_json_btn = dom::create_element("button")?;
    client_json_btn.set_text_content(Some("GET JSON (JSONPlaceholder)"));
    let client_post_btn = dom::create_element("button")?;
    client_post_btn.set_text_content(Some("POST JSON"));
    let client_auth_btn = dom::create_element("button")?;
    client_auth_btn.set_text_content(Some("GET /api/me с токеном"));
    let client_error_btn = dom::create_element("button")?;
    client_error_btn.set_text_content(Some("Ошибка 404"));
    let client_retry_btn = dom::create_element("button")?;
    client_retry_btn.set_text_content(Some("Retry через клиент"));
    let client_cache_btn = dom::create_element("button")?;
    client_cache_btn.set_text_content(Some("Кэш: 2 запроса"));

    dom::append(&section12, &client_json_btn)?;
    dom::append(&section12, &client_post_btn)?;
    dom::append(&section12, &client_auth_btn)?;
    dom::append(&section12, &client_error_btn)?;
    dom::append(&section12, &client_retry_btn)?;
    dom::append(&section12, &client_cache_btn)?;

    let result12 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result12), "api-result")?;
    result12.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section12, &result12)?;

    // ---- GET JSON ----
    {
        let result12 = result12.clone();
        callbacks::listen(
            &client_json_btn,
            "click",
            move |_| {
                let result12 = result12.clone();
                result12.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let client = api_client::ApiClient::new("https://jsonplaceholder.typicode.com");

                    match client.get::<serde_json::Value>("/posts/1").await {
                        Ok(json) => {
                            let text = format!(
                                "✅ client.get::<Value>(\"/posts/1\")\n\n\
                                 Base URL: {}\n\
                                 Ответ:\n{}",
                                client.base_url(),
                                json
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result12.set_text_content(Some(&format!(
                                "❌ {}\n\n(Технические детали: {e:?})",
                                describe_error(&e)
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- POST JSON ----
    {
        let result12 = result12.clone();
        callbacks::listen(
            &client_post_btn,
            "click",
            move |_| {
                let result12 = result12.clone();
                result12.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    #[derive(serde::Serialize)]
                    struct NewPost<'a> {
                        title: &'a str,
                        body: &'a str,
                        user_id: u32,
                    }

                    let client = api_client::ApiClient::new("https://jsonplaceholder.typicode.com");

                    let body = NewPost {
                        title: "Создан через ApiClient",
                        body: "Это тело сериализовано автоматически",
                        user_id: 42,
                    };

                    match client.post::<serde_json::Value, _>("/posts", &body).await {
                        Ok(json) => {
                            let text = format!(
                                "✅ client.post(\"/posts\", &body)\n\n\
                                 Тело сериализовано клиентом.\n\
                                 Ответ:\n{}",
                                json
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result12.set_text_content(Some(&format!(
                                "❌ {}\n\n(Технические детали: {e:?})",
                                describe_error(&e)
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- GET /api/me с токеном ----
    {
        let result12 = result12.clone();
        let store = token_store.clone();
        callbacks::listen(
            &client_auth_btn,
            "click",
            move |_| {
                let result12 = result12.clone();
                let store = store.clone();
                result12.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let has_token = store.borrow().access_token.is_some();

                    let client = api_client::ApiClient::new("http://127.0.0.1:3000")
                        .with_token_store(store.clone())
                        .with_logging();

                    match client.get::<serde_json::Value>("/api/me").await {
                        Ok(json) => {
                            let text = format!(
                                "✅ Авторизованный запрос прошёл\n\n\
                                 Эндпоинт: GET /api/me\n\
                                 Ответ сервера:\n{}\n\n\
                                 API-клиент сам добавил заголовок \
                                 Authorization: Bearer … — вызывающий \
                                 код про токен ничего не знает.",
                                serde_json::to_string_pretty(&json).unwrap_or_default()
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Http(401, _)) => {
                            let text = if has_token {
                                "🔒 Токен не принят (HTTP 401)\n\n\
                                 Токен был в хранилище, но сервер его \
                                 отклонил. Скорее всего, срок действия \
                                 истёк (по умолчанию — 5 минут).\n\n\
                                 Что делать:\n\
                                 1. Вернитесь в секцию «4. Аутентификация».\n\
                                 2. Нажмите «Войти как user» — получите новый токен.\n\
                                 3. Вернитесь сюда и повторите запрос.\n\n\
                                 Примечание: API-клиент попытался обновить \
                                 токен через refresh, но refresh-токена в \
                                 хранилище нет — поэтому запрос завершился \
                                 как есть."
                                    .to_string()
                            } else {
                                "🔒 Нужна авторизация (HTTP 401)\n\n\
                                 В хранилище нет access-токена, поэтому \
                                 запрос ушёл без заголовка Authorization. \
                                 Сервер ответил 401 — это стандартное \
                                 «представьтесь».\n\n\
                                 Что делать:\n\
                                 1. Прокрутите вверх до секции \
                                 «4. Аутентификация».\n\
                                 2. Нажмите «Войти как user».\n\
                                 3. Вернитесь сюда и повторите запрос."
                                    .to_string()
                            };
                            result12.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Http(403, _)) => {
                            result12.set_text_content(Some(
                                "🚫 Доступ запрещён (HTTP 403)\n\n\
                                 Токен валиден, но у пользователя нет \
                                 прав на этот ресурс. Проверьте роль: \
                                 для /api/admin нужен пользователь «admin».",
                            ));
                        }
                        Err(fetch::FetchError::Network(msg)) => {
                            let text = format!(
                                "🌐 Сервер недоступен\n\n\
                                 Детали: {msg}\n\n\
                                 Проверьте, что `npm run server` запущен \
                                 в отдельном терминале. Auth-server должен \
                                 слушать http://127.0.0.1:3000."
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Timeout(ms)) => {
                            let text = format!(
                                "⏱ Таймаут {ms} мс\n\n\
                                 Сервер не ответил вовремя. Проверьте, \
                                 не завис ли auth-server."
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result12.set_text_content(Some(&format!(
                                "❌ {}\n\n\
                                 (Технические детали: {e:?})",
                                e.user_message()
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- 404 ----
    {
        let result12 = result12.clone();
        callbacks::listen(
            &client_error_btn,
            "click",
            move |_| {
                let result12 = result12.clone();
                result12.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let client = api_client::ApiClient::new("https://jsonplaceholder.typicode.com");

                    match client.get::<serde_json::Value>("/posts/999999").await {
                        Ok(json) => {
                            let text = format!(
                                "⚠️ Неожиданный успех\n\n\
                                 Запрос к /posts/999999 вернул 200 OK. \
                                 JSONPlaceholder такого не должен делать — \
                                 проверьте URL или состояние сервера.\n\n\
                                 Ответ:\n{}",
                                serde_json::to_string_pretty(&json).unwrap_or_default()
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Http(404, body)) => {
                            let body_display = if body.trim().is_empty() {
                                "(тело пустое)".to_string()
                            } else {
                                truncate_preview(body.trim(), 200)
                            };

                            let text = format!(
                                "✅ Ожидаемая ошибка: HTTP 404 Not Found\n\n\
                                 Запрошенный URL: /posts/999999\n\
                                 Сервер: jsonplaceholder.typicode.com\n\
                                 Тело ответа: {body_display}\n\n\
                                 Что произошло:\n\
                                 • API-клиент отправил GET-запрос.\n\
                                 • Сервер ответил статусом 404.\n\
                                 • Клиент вернул вызывающему коду \
                                 FetchError::Http(404, body).\n\n\
                                 Почему это правильно:\n\
                                 • HTTP-статус сохранён как число — \
                                 легко сравнить и обработать.\n\
                                 • Тело ответа не потеряно — можно \
                                 показать пользователю или залогировать.\n\
                                 • Тип ошибки единый: любой HTTP-сбой, \
                                 сетевой сбой или таймаут приходит как \
                                 FetchError, а не как загадочный JsValue.\n\n\
                                 В реальном приложении здесь вы бы показали \
                                 «Страница не найдена» и кнопку «Назад»."
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Http(status, body)) => {
                            let body_display = if body.trim().is_empty() {
                                "(тело пустое)".to_string()
                            } else {
                                truncate_preview(body.trim(), 100)
                            };
                            let text = format!(
                                "⚠️ Получен статус {status}, а не 404\n\n\
                                 Ожидался HTTP 404, но сервер ответил \
                                 другим кодом. Возможно, URL изменился \
                                 или сервер вернул ошибку по другой причине.\n\n\
                                 Тело: {body_display}"
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Network(msg)) => {
                            let text = format!(
                                "🌐 Сетевая ошибка\n\n\
                                 Детали: {msg}\n\n\
                                 Скорее всего, нет доступа к \
                                 jsonplaceholder.typicode.com — проверьте \
                                 интернет или корпоративный прокси."
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result12.set_text_content(Some(&format!(
                                "❌ {}\n\n(Технические детали: {e:?})",
                                e.user_message()
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Retry через клиент ----
    {
        let result12 = result12.clone();
        callbacks::listen(
            &client_retry_btn,
            "click",
            move |_| {
                let result12 = result12.clone();
                result12.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Сброс счётчика flaky на сервере.
                    let _ = fetch::send_request_classified(
                        "http://127.0.0.1:3000/api/flaky/reset",
                    )
                    .await;

                    let config = fetch::RetryConfig {
                        max_attempts: 5,
                        base_delay_ms: 100,
                        max_delay_ms: 1000,
                        jitter_factor: 0.0,
                    };

                    let client = api_client::ApiClient::new("http://127.0.0.1:3000")
                        .with_retry(config)
                        .with_logging();

                    match client.get::<serde_json::Value>("/api/flaky").await {
                        Ok(json) => {
                            let text = format!(
                                "✅ Retry сработал\n\n\
                                 Эндпоинт /api/flaky падает первые два раза, \
                                 потом отвечает 200. API-клиент сам повторил \
                                 запрос с экспоненциальным backoff.\n\n\
                                 Ответ:\n{}",
                                serde_json::to_string_pretty(&json).unwrap_or_default()
                            );
                            result12.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result12.set_text_content(Some(&format!(
                                "❌ {}\n\n(Технические детали: {e:?})",
                                describe_error(&e)
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Кэш: два запроса подряд ----
    {
        let result12 = result12.clone();
        callbacks::listen(
            &client_cache_btn,
            "click",
            move |_| {
                let result12 = result12.clone();
                result12.set_text_content(Some("Запрос 1..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Один и тот же клиент переиспользуется между запросами.
                    let client = api_client::ApiClient::new("https://jsonplaceholder.typicode.com")
                        .with_cache(30_000) // 30 секунд
                        .with_logging();

                    // Первый запрос — пойдёт по сети.
                    let start1 = js_sys::Date::now();
                    let json1: serde_json::Value = match client.get("/posts/1").await {
                        Ok(v) => v,
                        Err(e) => {
                            result12.set_text_content(Some(&format!("❌ {}", describe_error(&e))));
                            return;
                        }
                    };
                    let time1 = js_sys::Date::now() - start1;

                    result12.set_text_content(Some(&format!(
                        "Запрос 1: {:.0} мс (по сети)\nЗапрос 2...",
                        time1
                    )));

                    // Второй запрос — из кэша.
                    let start2 = js_sys::Date::now();
                    let json2: serde_json::Value = match client.get("/posts/1").await {
                        Ok(v) => v,
                        Err(e) => {
                            result12.set_text_content(Some(&format!("❌ {}", describe_error(&e))));
                            return;
                        }
                    };
                    let time2 = js_sys::Date::now() - start2;

                    let text = format!(
                        "✅ Кэш GET-ответов\n\n\
                         Запрос 1 (по сети): {:.0} мс\n\
                         Запрос 2 (из кэша): {:.0} мс\n\n\
                         TTL кэша: 30 секунд.\n\
                         Оба ответа идентичны: {}\n\n\
                         В реальном приложении кэш помогает \
                         не дёргать сервер повторно, если данные \
                         не изменились. Сброс — через client.clear_cache().",
                        time1,
                        time2,
                        json1 == json2
                    );
                    result12.set_text_content(Some(&text));
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_api_client.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Создание клиента.
#[wasm_bindgen_test]
async fn test_api_client_new() {
    let client = hello_wasm02::api_client::ApiClient::new("https://example.com");
    assert_eq!(client.base_url(), "https://example.com");
}

/// Базовый URL нормализуется (убирается trailing slash).
#[wasm_bindgen_test]
async fn test_api_client_base_url() {
    let client = hello_wasm02::api_client::ApiClient::new("https://example.com/");
    assert_eq!(client.base_url(), "https://example.com");
}

/// GET-запрос через клиент.
#[wasm_bindgen_test]
async fn test_api_client_get() {
    let client =
        hello_wasm02::api_client::ApiClient::new("https://jsonplaceholder.typicode.com");
    let json: serde_json::Value = client.get("/posts/1").await.unwrap();
    assert_eq!(json["id"], 1);
    assert!(json["title"].is_string());
}

/// POST-запрос через клиент.
#[wasm_bindgen_test]
async fn test_api_client_post() {
    use serde::Serialize;

    #[derive(Serialize)]
    struct Body {
        title: String,
        body: String,
        #[serde(rename = "userId")]
        user_id: u32,
    }

    let client =
        hello_wasm02::api_client::ApiClient::new("https://jsonplaceholder.typicode.com");

    let body = Body {
        title: "Test".into(),
        body: "Test body".into(),
        user_id: 1,
    };

    let json: serde_json::Value = client.post("/posts", &body).await.unwrap();
    assert_eq!(json["title"], "Test");
    assert_eq!(json["userId"], 1);
}

/// Токен добавляется в заголовки.
#[wasm_bindgen_test]
async fn test_api_client_with_token() {
    let store = hello_wasm02::auth::TokenStore::new();
    hello_wasm02::auth::login(&store, "user", "secret")
        .await
        .unwrap();

    let client = hello_wasm02::api_client::ApiClient::new("http://127.0.0.1:3000")
        .with_token_store(store);

    let json: serde_json::Value = client.get("/api/me").await.unwrap();
    assert_eq!(json["username"], "user");
}

/// Пользовательские заголовки.
#[wasm_bindgen_test]
async fn test_api_client_custom_headers() {
    let client =
        hello_wasm02::api_client::ApiClient::new("https://jsonplaceholder.typicode.com")
            .with_header("X-Custom-Header", "Rust-Wasm-Client");

    let json: serde_json::Value = client.get("/posts/1").await.unwrap();
    assert_eq!(json["id"], 1);
}

/// Ошибка HTTP возвращается как FetchError::Http.
#[wasm_bindgen_test]
async fn test_api_client_error_handling() {
    let client =
        hello_wasm02::api_client::ApiClient::new("https://jsonplaceholder.typicode.com");

    let result = client.get::<serde_json::Value>("/posts/999999").await;

    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Http(404, _) => {}
        other => panic!("ожидался Http(404), получено {other:?}"),
    }
}

/// Retry-конфиг применяется.
#[wasm_bindgen_test]
async fn test_api_client_retry() {
    let _ = hello_wasm02::fetch::send_request_classified(
        "http://127.0.0.1:3000/api/flaky/reset",
    )
    .await;

    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 5,
        base_delay_ms: 50,
        max_delay_ms: 500,
        jitter_factor: 0.0,
    };

    let client = hello_wasm02::api_client::ApiClient::new("http://127.0.0.1:3000")
        .with_retry(config);

    let json: serde_json::Value = client.get("/api/flaky").await.unwrap();
    assert!(json["message"].is_string());
}
```

## Команды запуска

### Разработка (три терминала)

**Терминал 1 — сервер:**

```bash
npm run server
```

**Терминал 2 — сборка WASM и dev-сервер Vite:**

```bash
npm run wasm:dev
npm run dev
```

**Терминал 3 — тесты:**

```bash
npm run test:chrome
```

Ожидаемый результат: `70 passed; 0 failed` (62 из глав 16.4–16.11 + 8 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_api_client
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «12. API Client»:

| Кнопка | Ожидаемый результат |
|---|---|
| «GET JSON (JSONPlaceholder)» | `client.get::<Value>("/posts/1")` → JSON |
| «POST JSON» | Ответ с полями `id`, `title`, `userId` |
| «GET /api/me с токеном» | Сначала войдите в секции 4, потом сюда |
| «Ошибка 404» | Человекочитаемое сообщение с объяснением |
| «Retry через клиент» | Успех после 2 неудач, автоматический backoff |
| «Кэш: 2 запроса» | Запрос 1 — по сети, запрос 2 — мгновенно |

## Что даёт ApiClient

До этой главы каждый запрос выглядел так:

```rust
let opts = RequestInit::new();
opts.set_method("POST");
opts.set_mode(RequestMode::Cors);
opts.set_body(&JsValue::from_str(&json));

let request = Request::new_with_str_and_init(url, &opts)?;
request.headers().set("Content-Type", "application/json")?;
request.headers().set("Authorization", &format!("Bearer {token}"))?;
request.headers().set("Accept", "application/json")?;

let window = web_sys::window().unwrap();
let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
let response: Response = resp_value.dyn_into()?;

if !response.ok() { ... }
if response.status() == 401 { try_refresh... }
```

После — так:

```rust
let json: serde_json::Value = client.post("/posts", &body).await?;
```

Разница в трёх вещах:

**1. Меньше кода.** Двадцать строк превратились в одну. Это не косметика — это меньше мест, где можно ошибиться.

**2. Единая точка изменения.** Нужно добавить ещё один заголовок ко всем запросам? Правится `default_headers` в `ApiClient::new`. В старом коде — двадцать мест.

**3. Консистентная обработка.** Все ошибки приходят как `FetchError`. Все успешные ответы разбираются через `parse_json::<T>`. Retry применяется одинаково. Токен добавляется везде. Refresh срабатывает везде.

## Что добавлено по сравнению с первой версией

| Что | Как реализовано | Зачем |
|---|---|---|
| **Refresh токена** | `with_token_store(store)` + логика в `send` | При 401 клиент сам обновляет токен и повторяет запрос |
| **Timeout с настоящей отменой** | `AbortController` + `set_timeout` | Запрос реально прерывается, а не просто «перестаёт ждать» |
| **Логирование** | `with_logging()` + `crate::log!` | Видно каждый запрос: метод, URL, статус |
| **Rate limiting (429)** | `parse_retry_after` + `rate_limit_until` | После 429 клиент выдерживает паузу до следующего запроса |
| **Кэширование GET** | `cache` + `cache_ttl_ms` + `with_cache(ms)` | Повторные GET-запросы к тому же URL берутся из кэша |

## Итог главы

Мы собрали всё, что делали в предыдущих главах, в один паттерн:

1. **`ApiClient` инкапсулирует состояние.** Base URL, заголовки, токен, retry, timeout, кэш — всё хранится в структуре, а не в аргументах каждой функции.
2. **Методы `get`, `post`, `put`, `delete`** — обёртки над общим `send`, который делает всю работу.
3. **Единая точка обработки ответов.** `parse_json::<T>` разбирает тело только если статус 2xx.
4. **Единая точка обработки ошибок.** `FetchError` — единственный тип, с которым работает вызывающий код.
5. **Retry применяется осознанно.** Только для идемпотентных методов (`GET`, `PUT`, `DELETE`, `HEAD`) и только для временных ошибок (`Network`, `Timeout`, 5xx, 429, 408).
6. **Refresh токена прозрачен.** Вызывающий код просто вызывает `client.get("/api/me")` — клиент сам разберётся с 401.
7. **Timeout — настоящий.** `AbortController` прерывает запрос, а не оставляет его висеть в фоне.
8. **Rate limiting автоматический.** После 429 клиент ждёт `Retry-After` секунд, прежде чем делать следующий запрос.
9. **Кэш опционален.** `with_cache(ms)` включает кэширование GET-ответов с указанным TTL.

Ключевые уроки:

- **Дублирование кода — источник ошибок.** Если сборка `RequestInit` копипастится в двадцати местах, рано или поздно в одном из них забудут заголовок или режим.
- **Builder pattern — идиоматично для конфигурации.** `with_token`, `with_retry`, `with_timeout` читаются как предложения.
- **Инкапсуляция ≠ сокрытие.** `ApiClient` не скрывает, что под капотом `fetch`. Он организует код так, что вызывающему не нужно об этом думать.
- **Retry не должен быть агрессивным.** POST не повторяем, потому что без `Idempotency-Key` это создаст дубликат на сервере.
- **Timeout через `AbortController` лучше, чем через `Promise.race`.** Первый реально отменяет запрос, второй просто прекращает ожидание.
- **Кэш — не только оптимизация.** Он снижает нагрузку на сервер и делает UI отзывчивее. Но требует аккуратности: TTL, инвалидация, учёт методов.

Это последняя глава части про Fetch API. Мы прошли путь от первого `GET /posts/1` до полноценного клиента с retry, timeout, refresh токена, rate limiting, кэшированием и потоковым чтением. Все эти куски — не «фичи ради фич»: каждая появилась потому, что без неё реальное приложение не работало бы надёжно.