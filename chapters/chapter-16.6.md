# Глава 16.6. Retry и восстановление

## Обновлённая структура проекта

```
.
├── auth-server/
│   └── src/
│       ├── main.rs                 ← ОБНОВЛЁН: +маршруты /api/flaky, /api/flaky/reset, /api/error500
│       └── handlers.rs             ← ОБНОВЛЁН: +flaky, flaky_reset, always_500
│
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +RetryConfig, send_with_retry, is_retriable, calculate_backoff
│   ├── app.rs                      ← ОБНОВЛЁН: секция 6 «Retry», заменён URL в секции 5
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_retry.rs               ← НОВЫЙ: 8 тестов retry и backoff
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: +gloo-timers (для sleep в WASM)
└── (корневые файлы без изменений)
```

**Краткий комментарий.** Retry — это не «повторить запрос», а целая стратегия. Нужно решить, какие ошибки повторять, как долго ждать между попытками, сколько раз пробовать и что делать, если сервер сам сказал, когда вернуться. В этой главе мы добавим в `fetch.rs` конфигурируемый retry с экспоненциальным backoff и jitter, научимся читать заголовок `Retry-After`, а для демонстрации расширим `auth-server` двумя эндпоинтами: `/api/flaky` (падает первые два раза, потом успех) и `/api/error500` (всегда 500). Второй заодно заменит ненадёжный внешний сервис в секции 5.

## Ссылки для детального изучения

- **AWS: Exponential Backoff and Jitter** — классическая статья, откуда пошла формула с jitter: [aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- **MDN: `Retry-After`** — заголовок, которым сервер сообщает клиенту, когда возвращаться: [developer.mozilla.org/ru/docs/Web/HTTP/Headers/Retry-After](https://developer.mozilla.org/ru/docs/Web/HTTP/Headers/Retry-After)
- **RFC 9110: Idempotent Methods** — какие методы можно повторять безопасно: [httpwg.org/specs/rfc9110.html#idempotent.methods](https://httpwg.org/specs/rfc9110.html#idempotent.methods)
- **IETF: The Idempotency-Key HTTP Header Field** — как сделать POST идемпотентным: [datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- **`gloo-timers`** — таймеры и `sleep` для WASM: [docs.rs/gloo-timers](https://docs.rs/gloo-timers)
- **`backon`** — популярный крейт retry с поддержкой WASM (если не хочется писать свой): [docs.rs/backon](https://docs.rs/backon)

## Новые и обновлённые файлы

### `Cargo.toml` (корневой, обновлён)

Добавить в `[dependencies]`:

```toml
gloo-timers = { version = "0.3", features = ["futures"] }
```

Без `gloo-timers` в WASM нет асинхронного `sleep`. `std::thread::sleep` не работает в браузере — он заблокировал бы главный поток. `TimeoutFuture` из `gloo-timers` использует `setTimeout` и корректно уступает управление.

### `auth-server/src/handlers.rs` (обновлён — добавить в конец)

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static FLAKY_COUNTER: AtomicUsize = AtomicUsize::new(0);

/// Эндпоинт для демонстрации retry.
///
/// Первые два запроса возвращают 500, третий — 200.
/// Счётчик сбрасывается через `/api/flaky/reset`.
pub async fn flaky() -> Result<Json<Value>, StatusCode> {
    let n = FLAKY_COUNTER.fetch_add(1, Ordering::SeqCst);

    if n < 2 {
        return Err(StatusCode::INTERNAL_SERVER_ERROR);
    }

    Ok(Json(json!({
        "message": "Успех после двух неудач",
        "attempt": n + 1
    })))
}

/// Сбрасывает счётчик flaky-эндпоинта.
pub async fn flaky_reset() -> Json<Value> {
    FLAKY_COUNTER.store(0, Ordering::SeqCst);
    Json(json!({ "reset": true }))
}

/// Эндпоинт, который всегда возвращает 500.
///
/// Используется для демонстрации и тестов: собственный сервер
/// отвечает мгновенно и не зависит от аптайма внешних сервисов
/// вроде `httpstat.us`.
pub async fn always_500() -> Result<Json<Value>, StatusCode> {
    Err(StatusCode::INTERNAL_SERVER_ERROR)
}
```

### `auth-server/src/main.rs` (обновлён — добавить маршруты)

Внутри `Router::new()`:

```rust
.route("/api/flaky", get(handlers::flaky))
.route("/api/flaky/reset", get(handlers::flaky_reset))
.route("/api/error500", get(handlers::always_500))
```

### `src/fetch.rs` (обновлён — добавить в конец)

```rust
// ============================================================
// Retry и восстановление
// ============================================================

/// Конфигурация стратегии retry.
#[derive(Debug, Clone)]
pub struct RetryConfig {
    /// Максимальное число попыток, включая первую.
    pub max_attempts: u32,
    /// Базовая задержка в миллисекундах.
    pub base_delay_ms: u64,
    /// Верхняя граница задержки.
    pub max_delay_ms: u64,
    /// Доля jitter: 0.0 — без случайности, 0.5 — до +50% к задержке.
    pub jitter_factor: f64,
}

impl Default for RetryConfig {
    fn default() -> Self {
        Self {
            max_attempts: 3,
            base_delay_ms: 100,
            max_delay_ms: 5000,
            jitter_factor: 0.3,
        }
    }
}

/// Стоит ли повторять запрос при данной ошибке.
///
/// Сетевые ошибки, 5xx, 429 (Too Many Requests) и 408 (Request Timeout)
/// считаются временными. 4xx (кроме 408 и 429) — постоянными,
/// повтор только навредит.
pub fn is_retriable(error: &FetchError) -> bool {
    match error {
        FetchError::Network(_) => true,
        FetchError::Http(status, _) => {
            *status >= 500 || *status == 429 || *status == 408
        }
        FetchError::Aborted => false,
        FetchError::Parse(_) => false,
    }
}

/// Вычисляет задержку перед следующей попыткой.
///
/// Формула: `min(base * 2^attempt, max) + jitter`.
/// `attempt` нумеруется с нуля: первая задержка равна `base`,
/// вторая — `2 * base`, третья — `4 * base` и так далее.
pub fn calculate_backoff(attempt: u32, config: &RetryConfig) -> u64 {
    let exponential = config.base_delay_ms.saturating_mul(1u64 << attempt.min(20));
    let clamped = exponential.min(config.max_delay_ms);

    if config.jitter_factor <= 0.0 {
        return clamped;
    }

    let jitter_range = (clamped as f64) * config.jitter_factor;
    let jitter = (js_sys::Math::random() * jitter_range) as u64;
    clamped + jitter
}

/// Читает заголовок `Retry-After` и переводит его в миллисекунды.
///
/// Поддерживается только числовая форма (`Retry-After: 120`).
/// HTTP-дата игнорируется — для учебного примера это избыточно.
fn parse_retry_after(response: &web_sys::Response) -> Option<u64> {
    let value = response
        .headers()
        .get("Retry-After")
        .ok()
        .flatten()?;
    value.trim().parse::<u64>().ok().map(|s| s * 1000)
}

/// Одна попытка запроса. Возвращает `Err` с флагом `Retry-After`.
async fn send_once(url: &str) -> Result<web_sys::Response, (FetchError, Option<u64>)> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    let request = web_sys::Request::new_with_str_and_init(url, &opts)
        .map_err(|e| (classify_error(e), None))?;

    let window = web_sys::window()
        .ok_or_else(|| (FetchError::Network("no window object".into()), None))?;

    let resp_value = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
    .await
    .map_err(|e| (classify_error(e), None))?;

    let response: web_sys::Response = resp_value
        .dyn_into::<web_sys::Response>()
        .map_err(|e| (classify_error(e), None))?;

    if response.ok() {
        return Ok(response);
    }

    let status = response.status();
    let retry_after = parse_retry_after(&response);
    let body = read_error_body(&response).await;
    Err((FetchError::Http(status, body), retry_after))
}

/// Отправляет GET-запрос с автоматическим retry.
///
/// Повторяет запрос при временных ошибках, используя экспоненциальный
/// backoff с jitter. Если сервер прислал `Retry-After`, его значение
/// имеет приоритет над вычисленной задержкой.
pub async fn send_with_retry(
    url: &str,
    config: &RetryConfig,
) -> Result<web_sys::Response, FetchError> {
    let mut last_error: Option<FetchError> = None;

    for attempt in 0..config.max_attempts {
        match send_once(url).await {
            Ok(resp) => return Ok(resp),
            Err((error, retry_after)) => {
                let is_last = attempt + 1 >= config.max_attempts;

                if !is_retriable(&error) || is_last {
                    return Err(error);
                }

                let delay = retry_after.unwrap_or_else(|| {
                    calculate_backoff(attempt, config)
                });

                gloo_timers::future::TimeoutFuture::new(
                    delay.min(u32::MAX as u64) as u32,
                )
                .await;

                last_error = Some(error);
            }
        }
    }

    Err(last_error.unwrap_or_else(|| {
        FetchError::Network("retry loop exited unexpectedly".into())
    }))
}
```

### `src/app.rs` — заменить URL в секции 5

В обработчике «HTTP 500» из главы 16.5 замените внешний сервис на собственный:

```rust
// было:
// let url = "https://httpstat.us/500";

// стало:
let url = "http://127.0.0.1:3000/api/error500";
```

**Почему.** `httpstat.us` то работает, то нет, и зависит от CORS-политики чужого сервера. Мы не должны ставить работоспособность демо в зависимость от чужого аптайма. Собственный эндпоинт отвечает мгновенно, всегда и без сюрпризов.

### `src/app.rs` — добавить секцию 6

После секции 5, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 6: Retry
    // ============================================================

    let section6 = build_section(&container, "6. Retry")?;

    let intro6 = dom::create_paragraph(
        "Повтор запроса при временных ошибках. \
         Экспоненциальный backoff: 100мс → 200мс → 400мс. \
         Jitter добавляет случайность, чтобы тысячи клиентов \
         не били по серверу одновременно. \
         Повторяются только временные ошибки: сеть, 5xx, 429. \
         4xx (кроме 429 и 408) не повторяются.",
    )?;
    dom::add_class(&dom::as_html(&intro6), "hint")?;
    dom::append(&section6, &intro6)?;

    let retry_btn = dom::create_element("button")?;
    retry_btn.set_text_content(Some("Retry-запрос (падает 2 раза)"));
    let retry_reset_btn = dom::create_element("button")?;
    retry_reset_btn.set_text_content(Some("Сбросить счётчик сервера"));
    let retry_404_btn = dom::create_element("button")?;
    retry_404_btn.set_text_content(Some("404 — без повторов"));

    dom::append(&section6, &retry_btn)?;
    dom::append(&section6, &retry_reset_btn)?;
    dom::append(&section6, &retry_404_btn)?;

    let result6 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result6), "api-result")?;
    result6.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section6, &result6)?;

    // ---- Retry с логированием попыток ----
    {
        let result6 = result6.clone();
        callbacks::listen(
            &retry_btn,
            "click",
            move |_| {
                let result6 = result6.clone();
                result6.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "http://127.0.0.1:3000/api/flaky";
                    let config = fetch::RetryConfig {
                        max_attempts: 4,
                        base_delay_ms: 200,
                        max_delay_ms: 2000,
                        jitter_factor: 0.2,
                    };

                    let mut log_lines: Vec<String> = Vec::new();

                    for attempt in 1..=config.max_attempts {
                        log_lines.push(format!("Попытка {attempt}..."));

                        match fetch::send_request_classified(url).await {
                            Ok(resp) => {
                                let status = resp.status();
                                log_lines.push(format!(
                                    "Попытка {attempt}: ✅ {status}"
                                ));
                                result6.set_text_content(Some(&log_lines.join("\n")));
                                return;
                            }
                            Err(e) => {
                                let retriable = fetch::is_retriable(&e);
                                log_lines.push(format!(
                                    "Попытка {attempt}: ❌ {:?} (повторять: {})",
                                    e, retriable
                                ));

                                if !retriable || attempt == config.max_attempts {
                                    result6.set_text_content(Some(&log_lines.join("\n")));
                                    return;
                                }

                                let delay = fetch::calculate_backoff(attempt - 1, &config);
                                log_lines.push(format!("  ждём {delay}мс..."));
                                result6.set_text_content(Some(&log_lines.join("\n")));

                                gloo_timers::future::TimeoutFuture::new(
                                    delay as u32
                                )
                                .await;
                            }
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Сброс счётчика flaky на сервере ----
    {
        let result6 = result6.clone();
        callbacks::listen(
            &retry_reset_btn,
            "click",
            move |_| {
                let result6 = result6.clone();
                result6.set_text_content(Some("Сброс..."));
                wasm_bindgen_futures::spawn_local(async move {
                    match fetch::send_request_classified(
                        "http://127.0.0.1:3000/api/flaky/reset",
                    )
                    .await
                    {
                        Ok(_) => result6.set_text_content(Some("✅ Счётчик сброшен")),
                        Err(e) => result6.set_text_content(Some(&format!("❌ {:?}", e))),
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- 404: не повторяем ----
    {
        let result6 = result6.clone();
        callbacks::listen(
            &retry_404_btn,
            "click",
            move |_| {
                let result6 = result6.clone();
                result6.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/999999";
                    let config = fetch::RetryConfig::default();

                    match fetch::send_with_retry(url, &config).await {
                        Ok(_) => result6.set_text_content(Some("✅ (неожиданно)")),
                        Err(e) => {
                            let text = format!(
                                "❌ {:?}\n\n\
                                 is_retriable = {}\n\
                                 Повторов не было — 404 постоянная ошибка.",
                                e,
                                fetch::is_retriable(&e)
                            );
                            result6.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_retry.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Retry succeeds after failures: /api/flaky падает 2 раза, потом 200.
#[wasm_bindgen_test]
async fn test_retry_succeeds_after_failures() {
    // Сбрасываем счётчик сервера перед тестом.
    let _ = hello_wasm02::fetch::send_request_classified(
        "http://127.0.0.1:3000/api/flaky/reset",
    )
    .await;

    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 4,
        base_delay_ms: 50,
        max_delay_ms: 500,
        jitter_factor: 0.0,
    };

    let result = hello_wasm02::fetch::send_with_retry(
        "http://127.0.0.1:3000/api/flaky",
        &config,
    )
    .await;

    assert!(result.is_ok(), "третья попытка должна быть успешной");
    assert_eq!(result.unwrap().status(), 200);
}

/// Retry gives up after max: /api/error500 всегда отвечает 500.
///
/// Используем собственный сервер, а не внешний `httpstat.us`:
/// внешний сервис может упасть или изменить CORS-политику, и тест
/// превратится в проверку чужого аптайма.
#[wasm_bindgen_test]
async fn test_retry_gives_up_after_max() {
    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 2,
        base_delay_ms: 30,
        max_delay_ms: 100,
        jitter_factor: 0.0,
    };

    let result = hello_wasm02::fetch::send_with_retry(
        "http://127.0.0.1:3000/api/error500",
        &config,
    )
    .await;

    assert!(result.is_err(), "все попытки должны вернуть ошибку");
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Http(500, _) => {}
        other => panic!("ожидался Http(500), получено {other:?}"),
    }
}

/// Network-ошибка retriable.
#[wasm_bindgen_test]
async fn test_is_retriable_network() {
    let err = hello_wasm02::fetch::FetchError::Network("timeout".into());
    assert!(hello_wasm02::fetch::is_retriable(&err));
}

/// 500 retriable.
#[wasm_bindgen_test]
async fn test_is_retriable_500() {
    let err = hello_wasm02::fetch::FetchError::Http(500, "".into());
    assert!(hello_wasm02::fetch::is_retriable(&err));
}

/// 404 не retriable.
#[wasm_bindgen_test]
async fn test_is_not_retriable_404() {
    let err = hello_wasm02::fetch::FetchError::Http(404, "".into());
    assert!(!hello_wasm02::fetch::is_retriable(&err));
}

/// 429 retriable.
#[wasm_bindgen_test]
async fn test_is_retriable_429() {
    let err = hello_wasm02::fetch::FetchError::Http(429, "".into());
    assert!(hello_wasm02::fetch::is_retriable(&err));
}

/// Backoff растёт экспоненциально.
#[wasm_bindgen_test]
async fn test_backoff_increases() {
    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 5,
        base_delay_ms: 100,
        max_delay_ms: 10_000,
        jitter_factor: 0.0,
    };

    let d0 = hello_wasm02::fetch::calculate_backoff(0, &config);
    let d1 = hello_wasm02::fetch::calculate_backoff(1, &config);
    let d2 = hello_wasm02::fetch::calculate_backoff(2, &config);
    let d3 = hello_wasm02::fetch::calculate_backoff(3, &config);

    assert_eq!(d0, 100);
    assert_eq!(d1, 200);
    assert_eq!(d2, 400);
    assert_eq!(d3, 800);
}

/// Backoff с jitter даёт разные значения.
#[wasm_bindgen_test]
async fn test_backoff_has_jitter() {
    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 3,
        base_delay_ms: 1000,
        max_delay_ms: 10_000,
        jitter_factor: 0.5,
    };

    let mut values = std::collections::HashSet::new();
    for _ in 0..20 {
        values.insert(hello_wasm02::fetch::calculate_backoff(0, &config));
    }

    assert!(
        values.len() > 1,
        "jitter должен давать разные значения, получено {} уникальных",
        values.len()
    );

    for v in &values {
        assert!(*v >= 1000, "jitter не должен уменьшать задержку");
    }
}
```

## Команды запуска

### Разработка (три терминала)

**Терминал 1 — сервер аутентификации, flaky и error500:**

```bash
npm run server
```

Сервер должен быть перезапущен после правок в `handlers.rs` и `main.rs` — иначе новые маршруты не зарегистрируются.

**Терминал 2 — сборка WASM и dev-сервер Vite:**

```bash
npm run wasm:dev
npm run dev
```

**Терминал 3 — тесты:**

```bash
npm run test:chrome
```

Ожидаемый результат: `22 passed; 0 failed` (14 из глав 16.4–16.5 + 8 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_retry
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
cargo check --manifest-path auth-server/Cargo.toml
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «6. Retry» и нажмите каждую кнопку:

| Кнопка | Ожидаемый вывод |
|---|---|
| «Retry-запрос (падает 2 раза)» | Лог: Попытка 1 ❌, ждём 200мс, Попытка 2 ❌, ждём 400мс, Попытка 3 ✅ 200 |
| «Сбросить счётчик сервера» | «✅ Счётчик сброшен» |
| «404 — без повторов» | `Http(404, ...)`, `is_retriable = false`, одна попытка |

В секции 5 кнопка «HTTP 500» теперь тоже бьёт по собственному серверу и стабильно возвращает `Http(500, "<тело>")`.

## Замечание про внешние сервисы

При написании тестов и демо мы держимся простого принципа: **внешние сервисы — только те, которые заведомо стабильны**. JSONPlaceholder работает годами, к нему претензий нет. А вот `httpstat.us` и ему подобные то отвечают, то нет, то меняют CORS-политику, то требуют API-ключ. В результате тест `test_retry_gives_up_after_max` падал не потому, что наш retry сломан, а потому, что внешний сервис подвёл.

Всё, что нужно для тестов и демо, мы теперь умеем поднимать сами. `auth-server` — это не только JWT, но и полигон для сценариев с ошибками. Если в будущих главах понадобится 404, 429, 503, таймаут — добавляйте маршруты в `handlers.rs`, а не ищите подходящий внешний сервис.

## Итог главы

Мы добавили в проект полноценную стратегию retry:

1. **`RetryConfig`** — конфигурация: сколько попыток, базовая задержка, верхняя граница, доля jitter.
2. **`is_retriable`** — сетевые ошибки, 5xx, 429 и 408 считаются временными. 4xx (кроме этих двух) — постоянными, повтор не поможет.
3. **`calculate_backoff`** — экспоненциальный рост с jitter: `min(base * 2^attempt, max) + random(0, jitter * delay)`.
4. **`send_with_retry`** — готовая функция, которая сама делает retry, ждёт между попытками и учитывает заголовок `Retry-After`, если сервер его прислал.

Ключевые уроки:

- **Retry — это не «повторить запрос».** Это стратегия: какие ошибки повторять, как долго ждать, сколько раз пробовать.
- **Экспоненциальный backoff без jitter опасен.** Если тысяча клиентов одновременно получит 500 и все начнут ждать ровно 100мс, они ударят по серверу синхронно. Jitter разносит их во времени.
- **`Retry-After` — сигнал от сервера.** Если сервер сказал «вернись через 120 секунд», не надо умничать с экспонентой. Уважайте его.
- **POST не повторяем вслепую.** Без `Idempotency-Key` повтор POST создаст дубликат. Для GET, PUT, DELETE — можно.
- **`gloo-timers` — единственный правильный `sleep` в WASM.** `std::thread::sleep` заблокирует браузер.
- **Тесты не должны зависеть от чужого аптайма.** Собственный `auth-server` с эндпоинтами `/api/flaky` и `/api/error500` — надёжнее любого публичного сервиса.

В следующей главе — **Timeouts через `AbortController` + `setTimeout`**, **Circuit Breaker** и **очередь запросов с ограничением параллелизма**.