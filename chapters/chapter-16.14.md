# Глава 16.14. Практический проект: полноценный Rust API client

## Обновлённая структура проекта

```
.
├── src/
│   ├── api_client.rs               ← используется как есть (глава 16.12)
│   ├── fetch.rs                    ← используется как есть
│   ├── auth.rs                     ← используется как есть
│   ├── app.rs                      ← ФИНАЛЬНАЯ ВЕРСИЯ: 13 секций
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_mocks.rs               ← НОВЫЙ: 11 тестов с моками через js_sys::eval
│   ├── (все тесты из глав 16.4–16.13 — 76 штук)
│   └── (остальные без изменений)
│
├── auth-server/                    ← без изменений
├── web/
│   ├── public/
│   │   ├── workers/fetcher.js      ← без изменений
│   │   └── sw.js                   ← без изменений
│   └── (остальные без изменений)
│
└── (корневые файлы без изменений)
```

**Краткий комментарий.** За тринадцать глав мы построили полноценный HTTP-клиент: от `get_json` до `ApiClient` с refresh токена, retry, timeout, rate limiting, кэшированием и потоковым чтением. Но у нас не было главы про **тестирование**. Реальный API меняется, отвечает медленно, иногда падает — и unit-тесты, которые бьют по нему вживую, становятся flaky. В этой главе мы разберём, как подменять `fetch` на управляемый мок через `js_sys::eval`, как организовать интеграционные тесты против реального JSONPlaceholder и нашего auth-server, и как изолировать тесты друг от друга.

## Ссылки для детального изучения

- **`wasm-bindgen-test`** — официальная документация: [rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/index.html](https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/index.html)
- **`js_sys::eval`** — как подменять глобальные функции в тестах: [docs.rs/js-sys/latest/js_sys/fn.eval.html](https://docs.rs/js-sys/latest/js_sys/fn.eval.html)
- **MDN: `Response` конструктор** — как создать ответ вручную: [developer.mozilla.org/ru/docs/Web/API/Response/Response](https://developer.mozilla.org/ru/docs/Web/API/Response/Response)
- **MDN: `Request`** — почему тело читается через `clone().text()`: [developer.mozilla.org/ru/docs/Web/API/Request](https://developer.mozilla.org/ru/docs/Web/API/Request)
- **MSW (Mock Service Worker)** — mock-библиотека для JS-мира: [mswjs.io](https://mswjs.io/)
- **`wasm-bindgen-test`: async tests** — как писать асинхронные тесты: [rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/usage.html](https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/usage.html)

## Три уровня тестирования

Прежде чем писать код, разберёмся со стратегией. В WASM-приложении тесты делятся на три уровня по тому, **где** они выполняются и **что** проверяют.

**Уровень 1 — Unit-тесты на чистом Rust.** Проверяют чистые функции: `calculate_backoff`, `truncate_preview`, `describe_error`, разбор `data:`-URL. Никаких запросов, никакого DOM. Работают быстро, выполняются через `cargo test` на хосте.

**Уровень 2 — WASM-тесты с моками.** Проверяют логику работы с `fetch`, но **без реальной сети**. Мы подменяем глобальный `fetch` на управляемую функцию через `js_sys::eval`. Так можно проверить обработку 401, 500, таймаутов, retry — и всё это без интернета и без зависимости от чужого аптайма.

**Уровень 3 — Интеграционные тесты против реальных серверов.** Бьют по JSONPlaceholder и по нашему `auth-server`. Проверяют, что реальный протокол совпадает с ожиданиями.

Основное внимание — второму уровню, потому что именно там самая интересная техника.

## Важное про сигнатуру `fetch`

Прежде чем писать моки, запомним ключевую деталь: **`web_sys::Request` передаётся в `fetch` одним аргументом.**

Стандартный JS-фетч может вызываться двумя способами:

```javascript
// Способ 1: URL + опции
fetch("/api/users", { method: "POST", headers: {...}, body: "..." })

// Способ 2: готовый Request
fetch(new Request("/api/users", { method: "POST", ... }))
```

`web_sys::Window::fetch_with_request(&request)` использует **второй** способ. Значит, наш мок получит **только один аргумент** — объект `Request`. Второго аргумента (`init`) не будет.

Отсюда три правила для мока:

1. **Сигнатура** — `function(request)`, а не `function(url, init)`.
2. **Заголовки** — `request.headers.get(...)`, а не `init.headers`.
3. **Тело** — `await request.clone().text()`, а не `init.body`. Потому что `request.body` — это `ReadableStream`, и после первого чтения `bodyUsed = true`.

Если мок написан по старой схеме `function(url, init)` — `init` будет `undefined`, и попытка прочитать `init.method` упадёт с «Cannot read properties of undefined». Именно эта ошибка — самая частая при написании моков для `wasm-bindgen`.

## Новый файл: `tests/wasm_mocks.rs`

```rust
//! Тесты с моками `fetch` через `js_sys::eval`.
//!
//! Идея: в JavaScript можно в любой момент заменить глобальную
//! функцию `fetch` на свою. Наш Rust-код вызывает
//! `window.fetch_with_request(&request)` — значит, он увидит
//! именно мок.
//!
//! **Важно про сигнатуру мока.** `fetch_with_request` вызывает
//! `fetch(request)` с ОДНИМ аргументом — объектом `Request`.
//! Второй аргумент (`init`) там отсутствует. Поэтому заголовки
//! и тело мок должен читать из самого `Request`:
//! `request.headers.get(...)`, `request.clone().text()`,
//! `request.method`.
//!
//! После каждого теста восстанавливаем оригинальный `fetch`.

use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Устанавливает мок `fetch`, который возвращает заданный
/// статус и тело.
///
/// Возвращает `String` — код для восстановления оригинала.
/// Вызывающий должен сохранить его и вызвать `restore_fetch()`
/// в конце теста.
fn install_mock_fetch(status: u16, body: &str, content_type: &str) -> String {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    // Экранируем тело и content-type для вставки в JS-строку.
    let body_escaped = body
        .replace('\\', "\\\\")
        .replace('"', "\\\"")
        .replace('\n', "\\n");
    let ct_escaped = content_type.replace('"', "\\\"");

    let mock_code = format!(
        r#"
        window.fetch = function(request) {{
            return Promise.resolve(new Response(
                "{body_escaped}",
                {{
                    status: {status},
                    headers: {{ "Content-Type": "{ct_escaped}" }}
                }}
            ));
        }};
        "#
    );

    js_sys::eval(&mock_code).unwrap();
    "window.fetch = window.__originalFetch;".to_string()
}

/// Восстанавливает оригинальный `fetch`.
fn restore_fetch() {
    js_sys::eval("window.fetch = window.__originalFetch;").ok();
}

// ============================================================
// Базовые сценарии
// ============================================================

/// Мок возвращает 200 OK — клиент разбирает JSON.
#[wasm_bindgen_test]
async fn test_mock_200_response() {
    let _restore = install_mock_fetch(200, r#"{"id":1,"name":"Alice"}"#, "application/json");

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com");
    let json: serde_json::Value = client.get("/users/1").await.unwrap();

    assert_eq!(json["id"], 1);
    assert_eq!(json["name"], "Alice");

    restore_fetch();
}

/// Мок возвращает 404 — клиент возвращает `Http(404)`.
#[wasm_bindgen_test]
async fn test_mock_404_response() {
    let _restore = install_mock_fetch(404, "Not Found", "text/plain");

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com");
    let result = client.get::<serde_json::Value>("/users/999").await;

    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Http(404, body) => {
            assert_eq!(body, "Not Found");
        }
        other => panic!("ожидался Http(404), получено {other:?}"),
    }

    restore_fetch();
}

/// Мок возвращает 401 — клиент возвращает `Http(401)`.
#[wasm_bindgen_test]
async fn test_mock_401_response() {
    let _restore = install_mock_fetch(401, "Unauthorized", "text/plain");

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com");
    let result = client.get::<serde_json::Value>("/protected").await;

    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Http(401, _) => {}
        other => panic!("ожидался Http(401), получено {other:?}"),
    }

    restore_fetch();
}

/// Мок возвращает пустое тело — клиент получает `Parse`.
#[wasm_bindgen_test]
async fn test_mock_empty_body() {
    let _restore = install_mock_fetch(200, "", "application/json");

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com");
    let result = client.get::<serde_json::Value>("/empty").await;

    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Parse(_) => {}
        other => panic!("ожидался Parse, получено {other:?}"),
    }

    restore_fetch();
}

// ============================================================
// Retry
// ============================================================

/// Retry повторяет запрос при 500. Три попытки, на третьей — успех.
#[wasm_bindgen_test]
async fn test_mock_retry_then_success() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    // Счётчик вызовов храним в window.__callCount.
    let mock = r#"
        window.__callCount = 0;
        window.fetch = function(request) {
            window.__callCount++;
            if (window.__callCount < 3) {
                return Promise.resolve(new Response(
                    "Server Error",
                    { status: 500, headers: { "Content-Type": "text/plain" } }
                ));
            }
            return Promise.resolve(new Response(
                '{"ok":true}',
                { status: 200, headers: { "Content-Type": "application/json" } }
            ));
        };
    "#;
    js_sys::eval(mock).unwrap();

    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 5,
        base_delay_ms: 10,
        max_delay_ms: 50,
        jitter_factor: 0.0,
    };

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com")
        .with_retry(config);

    let json: serde_json::Value = client.get("/retry").await.unwrap();
    assert_eq!(json["ok"], true);

    let calls = js_sys::eval("window.__callCount").unwrap();
    assert_eq!(calls.as_f64().unwrap() as u32, 3);

    restore_fetch();
}

/// 404 не повторяется — ровно одна попытка.
#[wasm_bindgen_test]
async fn test_mock_404_not_retried() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    let mock = r#"
        window.__callCount = 0;
        window.fetch = function(request) {
            window.__callCount++;
            return Promise.resolve(new Response(
                "Not Found",
                { status: 404, headers: { "Content-Type": "text/plain" } }
            ));
        };
    "#;
    js_sys::eval(mock).unwrap();

    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 5,
        base_delay_ms: 10,
        max_delay_ms: 50,
        jitter_factor: 0.0,
    };

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com")
        .with_retry(config);

    let _ = client.get::<serde_json::Value>("/missing").await;

    let calls = js_sys::eval("window.__callCount").unwrap();
    assert_eq!(calls.as_f64().unwrap() as u32, 1);

    restore_fetch();
}

/// Сетевой сбой классифицируется как `Network`.
#[wasm_bindgen_test]
async fn test_mock_network_failure() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    let mock = r#"
        window.fetch = function(request) {
            return Promise.reject(new TypeError("Failed to fetch"));
        };
    "#;
    js_sys::eval(mock).unwrap();

    let config = hello_wasm02::fetch::RetryConfig {
        max_attempts: 1,
        base_delay_ms: 10,
        max_delay_ms: 50,
        jitter_factor: 0.0,
    };

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com")
        .with_retry(config);

    let result = client.get::<serde_json::Value>("/data").await;
    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Network(_) => {}
        other => panic!("ожидался Network, получено {other:?}"),
    }

    restore_fetch();
}

// ============================================================
// Кэш
// ============================================================

/// Кэш: второй запрос к тому же URL не идёт в сеть.
#[wasm_bindgen_test]
async fn test_mock_cache_hits() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    let mock = r#"
        window.__callCount = 0;
        window.fetch = function(request) {
            window.__callCount++;
            return Promise.resolve(new Response(
                '{"cached":true}',
                { status: 200, headers: { "Content-Type": "application/json" } }
            ));
        };
    "#;
    js_sys::eval(mock).unwrap();

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com")
        .with_cache(60_000);

    let _: serde_json::Value = client.get("/data").await.unwrap();
    let _: serde_json::Value = client.get("/data").await.unwrap();

    let calls = js_sys::eval("window.__callCount").unwrap();
    assert_eq!(calls.as_f64().unwrap() as u32, 1, "должен быть один сетевой вызов");

    restore_fetch();
}

// ============================================================
// Заголовки
// ============================================================

/// Клиент добавляет заголовок Authorization.
///
/// `fetch_with_request` вызывает `fetch(request)` — один аргумент,
/// объект Request. Заголовки читаем из `request.headers`.
#[wasm_bindgen_test]
async fn test_mock_auth_header_sent() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    let mock = r#"
        window.__lastAuthHeader = null;
        window.fetch = function(request) {
            // request — это Request-объект, а не URL.
            // У него есть headers, method, body.
            try {
                window.__lastAuthHeader = request.headers.get("Authorization");
            } catch (e) {
                window.__lastAuthHeader = "ERROR: " + String(e);
            }
            return Promise.resolve(new Response(
                '{"ok":true}',
                { status: 200, headers: { "Content-Type": "application/json" } }
            ));
        };
    "#;
    js_sys::eval(mock).unwrap();

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com")
        .with_token("test-token-12345");

    let _: serde_json::Value = client.get("/me").await.unwrap();

    let header = js_sys::eval("window.__lastAuthHeader").unwrap();
    assert_eq!(header.as_string().as_deref(), Some("Bearer test-token-12345"));

    restore_fetch();
}

/// Клиент добавляет кастомный заголовок.
#[wasm_bindgen_test]
async fn test_mock_custom_header_sent() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    let mock = r#"
        window.__lastCustomHeader = null;
        window.fetch = function(request) {
            try {
                window.__lastCustomHeader = request.headers.get("X-Custom-Header");
            } catch (e) {
                window.__lastCustomHeader = "ERROR: " + String(e);
            }
            return Promise.resolve(new Response(
                '{}',
                { status: 200, headers: { "Content-Type": "application/json" } }
            ));
        };
    "#;
    js_sys::eval(mock).unwrap();

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com")
        .with_header("X-Custom-Header", "Rust-Wasm-Client");

    let _: serde_json::Value = client.get("/data").await.unwrap();

    let header = js_sys::eval("window.__lastCustomHeader").unwrap();
    assert_eq!(header.as_string().as_deref(), Some("Rust-Wasm-Client"));

    restore_fetch();
}

/// POST отправляет JSON-тело.
///
/// Тело в `Request` читается через `request.clone().text()`:
/// `request.body` — это `ReadableStream`, а не строка. После
/// первого чтения `Request.bodyUsed = true`, поэтому клонируем.
#[wasm_bindgen_test]
async fn test_mock_post_body_sent() {
    js_sys::eval("window.__originalFetch = window.fetch;").unwrap();

    let mock = r#"
        window.__lastBody = null;
        window.__lastMethod = null;
        window.fetch = async function(request) {
            window.__lastMethod = request.method;
            try {
                window.__lastBody = await request.clone().text();
            } catch (e) {
                window.__lastBody = "ERROR: " + String(e);
            }
            return new Response(
                '{"id":42}',
                { status: 201, headers: { "Content-Type": "application/json" } }
            );
        };
    "#;
    js_sys::eval(mock).unwrap();

    #[derive(serde::Serialize)]
    struct NewUser {
        name: String,
    }

    let client = hello_wasm02::api_client::ApiClient::new("https://mock.example.com");
    let body = NewUser { name: "Alice".into() };
    let json: serde_json::Value = client.post("/users", &body).await.unwrap();

    assert_eq!(json["id"], 42);

    let method = js_sys::eval("window.__lastMethod").unwrap();
    assert_eq!(method.as_string().as_deref(), Some("POST"));

    let sent_body = js_sys::eval("window.__lastBody").unwrap();
    let sent_str = sent_body.as_string().unwrap_or_default();
    assert!(
        sent_str.contains("Alice"),
        "тело запроса должно содержать Alice, получено: {sent_str}"
    );

    restore_fetch();
}
```

## Финальная структура `app.rs`

Файл `app.rs` уже содержит тринадцать секций из предыдущих глав. Ничего заново переписывать не нужно. Ниже — карта секций и того, что в них демонстрируется.

| Секция | Демонстрирует | Глава |
|---|---|---|
| 1. Первый GET | `get_json`, базовый fetch | 16.1 |
| 2. HTTP-методы | POST JSON, FormData, PUT, DELETE | 16.2 |
| 3. Заголовки | `get_with_headers`, чтение `Headers` | 16.3 |
| 4. Аутентификация | Bearer, 401/403, refresh | 16.4 |
| 5. Ошибки | `FetchError`, классификация | 16.5 |
| 6. Retry | backoff, jitter, `Retry-After` | 16.6 |
| 7. Timeout и отмена | `AbortController`, ручная отмена | 16.7 |
| 8. Параллельные запросы | `Promise.all`, `allSettled`, `race`, `buffer_unordered` | 16.8 |
| 9. Типы ответов | JSON, text, blob, bytes, data URL | 16.9 |
| 10. Streams | `ReadableStream`, чтение по частям | 16.10 |
| 11. CORS | режимы запроса, preflight | 16.11 |
| 12. API Client | `ApiClient` со всеми возможностями | 16.12 |
| 13. Worker Fetch | Worker, Service Worker | 16.13 |

В каждой секции — кнопки и область вывода `resultN`. Все обработчики используют общий хелпер `describe_error` для человекочитаемых ошибок и `truncate_preview` для обрезки длинных значений по `chars()`, а не по байтам.

Эта глава добавляет только тесты с моками — она не меняет `app.rs`.

## Тесты всех предыдущих глав

Все тесты, написанные в главах 16.4–16.13, продолжают работать. Сводная таблица по файлам:

| Файл | Что проверяет | Штук |
|---|---|---|
| `tests/pure_logic.rs` | Чистые функции без WASM | (часть 1) |
| `tests/wasm_async.rs` | Асинхронные примитивы | (часть 1) |
| `tests/wasm_async_utils.rs` | Утилиты async | (часть 1) |
| `tests/wasm_auth.rs` | Аутентификация | 6 |
| `tests/wasm_errors.rs` | Классификация ошибок | 8 |
| `tests/wasm_retry.rs` | Retry, backoff | 8 |
| `tests/wasm_timeout.rs` | Timeout, отмена | 6 |
| `tests/wasm_parallel.rs` | Параллельные запросы | 8 |
| `tests/wasm_response_types.rs` | Типы ответов | 12 |
| `tests/wasm_streams.rs` | Потоки | 10 |
| `tests/wasm_cors.rs` | CORS | 4 |
| `tests/wasm_api_client.rs` | ApiClient | 8 |
| `tests/wasm_worker.rs` | Worker, SW | 6 |
| `tests/wasm_mocks.rs` | Моки через eval | 11 |
| **Итого** | | **87** |

Из них 76 идут против реальных серверов (JSONPlaceholder и auth-server), 11 — против моков. Моки быстрее и не зависят от сети, реальные запросы проверяют протокол.

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

Ожидаемый результат: `87 passed; 0 failed`.

### Только тесты с моками

```bash
wasm-pack test --headless --chrome --test wasm_mocks
```

Моки не требуют ни auth-server, ни интернета. Этот набор можно запускать в CI без поднятого backend'а.

### Только интеграционные тесты

```bash
wasm-pack test --headless --chrome --test wasm_auth --test wasm_api_client
```

Требует запущенного `auth-server` на `127.0.0.1:3000`.

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

## Правила написания тестов с моками

Если решите добавлять свои тесты с моками, держите в голове пять правил.

**1. Мок принимает `Request`, а не `url, init`.** `fetch_with_request` вызывает `fetch(request)` с одним аргументом. Читайте `request.method`, `request.headers.get(...)`, `await request.clone().text()`.

**2. Всегда восстанавливайте `fetch` в конце.** Если тест упал с паникой — восстановление не выполнится. Оберните мок в RAII-обёртку:

```rust
struct MockFetchGuard;

impl MockFetchGuard {
    fn new(status: u16, body: &str) -> Self {
        // сохраняем оригинал и ставим мок
        Self
    }
}

impl Drop for MockFetchGuard {
    fn drop(&mut self) {
        js_sys::eval("window.fetch = window.__originalFetch;").ok();
    }
}
```

Теперь даже при панике `Drop` сработает и восстановит глобальную функцию.

**3. Каждый тест работает со своим `ApiClient`.** Никаких общих состояний между тестами. `wasm-bindgen-test` запускает тесты в одном потоке, но если один тест оставит клиент с кэшем — другой может неожиданно получить кэшированный ответ.

**4. Проверяйте счётчик вызовов.** Мок должен уметь ответить на вопрос «сколько раз меня вызвали». Это самый простой способ проверить retry и кэш. Счётчик храним в `window.__callCount` и увеличиваем внутри мока.

**5. Не полагайтесь на порядок тестов.** `wasm-bindgen-test` не гарантирует порядок. Если тест требует сброса состояния — делайте это в начале самого теста.

## Про MSW и моки в экосистеме

**MSW (Mock Service Worker)** — популярная библиотека для моков сетевых запросов в JavaScript-мире. Она работает через Service Worker: регистрирует SW, который перехватывает запросы и отдаёт подготовленные ответы. Плюсы: не нужно патчить `fetch`, код приложения не знает, что его тестируют. Минусы: сложнее настройка, требует Service Worker (а значит — HTTPS или localhost), для Rust+WASM менее прямолинейна.

Для нашего случая `js_sys::eval` оказался проще. Мы один раз заменяем глобальную функцию `fetch`, и весь код — включая `ApiClient`, `fetch::send_request_classified`, всё остальное — видит мок.

Когда `js_sys::eval` **не подходит**:

- **Параллельные тесты.** Если несколько тестов запускаются одновременно и каждый подменяет `fetch`, они будут мешать друг другу. `wasm-bindgen-test` по умолчанию запускает тесты последовательно в одном потоке, поэтому у нас проблемы нет.
- **Тесты реальных JS-библиотек.** Если мокать нужно не `fetch`, а, скажем, `axios` или `XMLHttpRequest`, проще использовать MSW.
- **Интеграционные тесты со сложной логикой маршрутизации.** MSW даёт декларативные хендлеры в стиле «если URL `/users/:id`, вернуть объект пользователя».

Для нашего приложения — небольшая библиотека на Rust+WASM с предсказуемым набором запросов — `js_sys::eval` оптимален. MSW оставим в бэклоге на случай, если проект вырастет.

## Итог главы

Мы собрали полноценный Rust API client и научились его тестировать:

1. **Три уровня тестирования.** Unit (на хосте), моки через `js_sys::eval`, интеграционные против реальных серверов.
2. **Моки через `js_sys::eval`.** Простейший способ подменить `fetch`: одна строка JS — и весь Rust-код видит управляемый ответ.
3. **Правильная сигнатура мока.** `function(request)` — потому что `fetch_with_request` передаёт `Request` одним аргументом. Заголовки через `request.headers.get`, тело через `await request.clone().text()`.
4. **Изоляция тестов.** Каждый тест — свой `ApiClient`, свой мок, свой счётчик вызовов.
5. **87 тестов** покрывают все аспекты клиента: от классификации ошибок до Worker'ов.

Ключевые уроки:

- **Мок — не замена реальных тестов, а дополнение.** Моками проверяем крайние случаи (500, 401, пустое тело), а реальные серверы — что протокол совпадает.
- **`js_sys::eval` — грубый, но эффективный инструмент.** Он заменяет глобальный `fetch` целиком. Это работает, потому что весь наш код использует `js_sys::global().fetch`.
- **RAII-обёртка защищает от утечек.** Если мок не восстановить, следующий тест сломается. `Drop` решает проблему.
- **Счётчик вызовов проверяет поведение.** Просто «получилось / не получилось» недостаточно. Нужно убедиться, что retry сделал ровно 3 попытки, а кэш — ровно 1 сетевой запрос.
- **`Request` — не то же самое, что `url + init`.** Тело — это `ReadableStream`, читается через `clone().text()`. Заголовки — `Headers`, а не объект.

Это последняя глава части про Fetch API. Мы прошли путь от первого `GET /posts/1` до полноценного клиента с retry, timeout, refresh токена, rate limiting, кэшированием, потоковым чтением, фоновой работой в Worker'ах и тестами с моками. Все эти куски — не «фичи ради фич»: каждая появилась потому, что без неё реальное приложение не работало бы надёжно.

В следующей части книги — **работа с DOM и событиями**: формы, валидация, drag-and-drop, анимации и всё, что делает веб-приложение интерактивным.