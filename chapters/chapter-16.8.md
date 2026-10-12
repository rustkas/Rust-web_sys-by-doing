# Глава 16.8. Параллельные запросы

## Обновлённая структура проекта

```
.
├── auth-server/
│   └── src/
│       ├── main.rs                 ← ОБНОВЛЁН: +/api/delay/:secs, /api/concurrency/*
│       └── handlers.rs             ← ОБНОВЛЁН: +delay, concurrency_test, concurrency_reset, concurrency_stats
│
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +get_all, get_all_settled, get_race, get_all_limited
│   ├── app.rs                      ← ОБНОВЛЁН: секция 8 «Параллельные запросы»
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_parallel.rs            ← НОВЫЙ: 8 тестов параллельных запросов
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: +futures = "0.3"
└── (корневые файлы без изменений)
```

**Краткий комментарий.** До сих пор все запросы в нашем приложении шли последовательно: нажали кнопку — дождались ответа — показали результат. Но в реальных приложениях часто нужно загрузить несколько независимых ресурсов одновременно. Если делать это последовательно, время ожидания суммируется: три запроса по 500 мс — это полторы секунды. Если запустить их параллельно — те же полторы секунды превращаются в 500 мс. JavaScript предоставляет для этого `Promise.all`, `Promise.allSettled` и `Promise.race`; мы будем вызывать их через `js_sys::Promise`. Также разберём, как ограничить параллельность, чтобы не отправить сто запросов разом.

## Ссылки для детального изучения

- **MDN: `Promise.all()`** — ждёт все промисы, падает при первой ошибке: [developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise/all](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise/all)
- **MDN: `Promise.allSettled()`** — ждёт все промисы, не падает: [developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled)
- **MDN: `Promise.race()`** — первый завершившийся побеждает: [developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise/race](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)
- **`js_sys::Promise`** — документация Rust-биндинга: [docs.rs/js-sys/latest/js_sys/struct.Promise.html](https://docs.rs/js-sys/latest/js_sys/struct.Promise.html)
- **`futures::stream::buffer_unordered`** — ограничение параллельности: [docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffer_unordered](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.buffer_unordered)
- **Chrome: Connection pools per origin** — почему `localhost` и `127.0.0.1` ведут себя по-разному: [developer.chrome.com/blog/http-connection-pools](https://developer.chrome.com/blog/http-connection-pools)

## Новые и обновлённые файлы

### `Cargo.toml` (корневой, обновлён)

Добавить в `[dependencies]`:

```toml
futures = "0.3"
```

### `auth-server/src/handlers.rs` (обновлён — добавить в конец)

```rust
use axum::extract::Path;
use std::sync::atomic::{AtomicUsize, Ordering};

/// Эндпоинт с задержкой, указанной в URL.
///
/// `/api/delay/2` — ответ через 2 секунды.
pub async fn delay(Path(secs): Path<u64>) -> Json<Value> {
    tokio::time::sleep(std::time::Duration::from_secs(secs)).await;

    Json(json!({
        "message": format!("Ответ через {secs} секунд"),
        "delay_secs": secs
    }))
}

// ============================================================
// Счётчик одновременных запросов
// ============================================================

static CONCURRENT: AtomicUsize = AtomicUsize::new(0);
static MAX_CONCURRENT: AtomicUsize = AtomicUsize::new(0);

/// Эндпоинт с задержкой 300 мс, считающий одновременные запросы.
pub async fn concurrency_test() -> Json<Value> {
    let current = CONCURRENT.fetch_add(1, Ordering::SeqCst) + 1;
    MAX_CONCURRENT.fetch_max(current, Ordering::SeqCst);

    tokio::time::sleep(std::time::Duration::from_millis(300)).await;

    CONCURRENT.fetch_sub(1, Ordering::SeqCst);
    Json(json!({ "ok": true }))
}

/// Сбрасывает счётчик максимальной одновременности.
pub async fn concurrency_reset() -> Json<Value> {
    MAX_CONCURRENT.store(0, Ordering::SeqCst);
    Json(json!({ "reset": true }))
}

/// Возвращает максимальное количество одновременных запросов.
pub async fn concurrency_stats() -> Json<Value> {
    Json(json!({
        "max_concurrent": MAX_CONCURRENT.load(Ordering::SeqCst)
    }))
}
```

### `auth-server/src/main.rs` (обновлён — добавить маршруты)

```rust
.route("/api/delay/:secs", get(handlers::delay))
.route("/api/concurrency/test", get(handlers::concurrency_test))
.route("/api/concurrency/reset", get(handlers::concurrency_reset))
.route("/api/concurrency/stats", get(handlers::concurrency_stats))
```

### `src/fetch.rs` (обновлён — добавить в конец)

В начало файла добавьте импорты:

```rust
use js_sys::Promise;
use futures::stream::{self, StreamExt};
```

**Внимание:** `use futures::future::join_all;` **не нужен** — мы его не используем. `Promise::all` живёт в `js_sys`, а `buffer_unordered` — в `StreamExt`. Лишний импорт вызовет предупреждение.

Четыре новые функции:

```rust
// ============================================================
// Параллельные запросы
// ============================================================

/// Отправляет несколько GET-запросов параллельно через `Promise.all`.
///
/// Возвращает `Vec<Response>` в том же порядке, что и URL-ы.
/// При **HTTP-статусе ≥ 400** хотя бы у одного запроса весь вызов
/// падает с `FetchError::Http`. Сетевые ошибки тоже приводят к `Err`.
///
/// **Важно:** `fetch` резолвит промис даже при 404 и 500 — ошибкой
/// считается только сетевой сбой. `Promise.all` смотрит на промисы,
/// а не на HTTP-статусы, и видит три «успешно завершившихся» промиса.
/// Поэтому проверку `response.ok()` выполняем мы сами.
pub async fn get_all(urls: &[&str]) -> Result<Vec<web_sys::Response>, FetchError> {
    let promises = js_sys::Array::new();
    for url in urls {
        let promise = fetch_as_promise(url);
        promises.push(&promise);
    }

    let all_promise = Promise::all(&promises);
    let result = JsFuture::from(all_promise).await.map_err(classify_error)?;

    let array: js_sys::Array = result.dyn_into().map_err(|_| {
        FetchError::Parse("Promise.all result is not an array".into())
    })?;

    let mut responses = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let value = array.get(i);
        let response: web_sys::Response = value.dyn_into().map_err(|_| {
            FetchError::Parse(format!("element {i} is not a Response"))
        })?;

        // fetch резолвит промис даже при 404/500 — проверяем ok() сами.
        if !response.ok() {
            let status = response.status();
            let body = read_error_body(&response).await;
            return Err(FetchError::Http(status, body));
        }

        responses.push(response);
    }

    Ok(responses)
}

/// Отправляет несколько GET-запросов параллельно и не падает при ошибках.
///
/// Возвращает `Vec<Result<Response, FetchError>>` — по одному результату
/// на каждый URL. HTTP-статусы ≥ 400 попадают в `Err(FetchError::Http)`,
/// сетевые сбои — в `Err(FetchError::Network)`, успешные — в `Ok`.
pub async fn get_all_settled(
    urls: &[&str],
) -> Result<Vec<Result<web_sys::Response, FetchError>>, FetchError> {
    let promises = js_sys::Array::new();
    for url in urls {
        let promise = fetch_as_promise(url);
        promises.push(&promise);
    }

    let all_promise = Promise::all_settled(&promises);
    let result = JsFuture::from(all_promise).await.map_err(classify_error)?;

    let array: js_sys::Array = result.dyn_into().map_err(|_| {
        FetchError::Parse("Promise.allSettled result is not an array".into())
    })?;

    let mut results = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let entry = array.get(i);
        let status = js_sys::Reflect::get(&entry, &JsValue::from_str("status"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();

        if status == "fulfilled" {
            let value = js_sys::Reflect::get(&entry, &JsValue::from_str("value"))
                .unwrap_or(JsValue::UNDEFINED);
            let response: web_sys::Response = value.dyn_into().map_err(|_| {
                FetchError::Parse(format!("fulfilled element {i} is not a Response"))
            })?;

            // Promise.allSettled считает fulfilled-промис успехом,
            // даже если HTTP-статус 404 или 500. Проверяем ok() сами.
            if !response.ok() {
                let status = response.status();
                let body = read_error_body(&response).await;
                results.push(Err(FetchError::Http(status, body)));
            } else {
                results.push(Ok(response));
            }
        } else {
            // rejected — сетевой сбой или abort.
            let reason = js_sys::Reflect::get(&entry, &JsValue::from_str("reason"))
                .unwrap_or(JsValue::UNDEFINED);
            results.push(Err(classify_error(reason)));
        }
    }

    Ok(results)
}

/// Отправляет несколько GET-запросов и возвращает первый завершившийся.
///
/// Аналог `Promise.race`. Если первый завершившийся запрос упал,
/// вернётся ошибка — даже если остальные ещё в полёте.
pub async fn get_race(urls: &[&str]) -> Result<web_sys::Response, FetchError> {
    let promises = js_sys::Array::new();
    for url in urls {
        let promise = fetch_as_promise(url);
        promises.push(&promise);
    }

    let race_promise = Promise::race(&promises);
    let result = JsFuture::from(race_promise).await.map_err(classify_error)?;

    let response: web_sys::Response = result.dyn_into().map_err(|_| {
        FetchError::Parse("Promise.race result is not a Response".into())
    })?;

    if !response.ok() {
        let status = response.status();
        let body = read_error_body(&response).await;
        return Err(FetchError::Http(status, body));
    }

    Ok(response)
}

/// Отправляет несколько GET-запросов с ограничением параллельности.
///
/// `limit` — максимальное количество одновременных запросов.
/// Если URL-ов больше, они будут запускаться по мере освобождения слотов.
pub async fn get_all_limited(
    urls: &[&str],
    limit: usize,
) -> Vec<Result<web_sys::Response, FetchError>> {
    let futures = urls.iter().map(|url| {
        let url = url.to_string();
        async move {
            let opts = web_sys::RequestInit::new();
            opts.set_method("GET");
            opts.set_mode(web_sys::RequestMode::Cors);

            let request = web_sys::Request::new_with_str_and_init(&url, &opts)
                .map_err(classify_error)?;

            send_request(&request).await.map_err(classify_error)
        }
    });

    stream::iter(futures)
        .buffer_unordered(limit)
        .collect::<Vec<_>>()
        .await
}

/// Вспомогательная функция: превращает fetch-запрос в JS Promise.
fn fetch_as_promise(url: &str) -> Promise {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    let request = web_sys::Request::new_with_str_and_init(url, &opts)
        .expect("failed to create request");

    let window = web_sys::window().expect("no window");
    window.fetch_with_request(&request)
}
```

### `src/app.rs` — добавить секцию 8

После секции 7, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 8: Параллельные запросы
    // ============================================================

    let section8 = build_section(&container, "8. Параллельные запросы")?;

    let intro8 = dom::create_paragraph(
        "Три запроса по 1 секунде последовательно — 3 секунды. \
         Те же три запроса параллельно — 1 секунда. \
         Promise.all ждёт все, падает при первой ошибке. \
         Promise.allSettled ждёт все, не падает: успешные результаты \
         сохраняются, упавшие помечаются как ошибки. \
         Promise.race — первый завершившийся. \
         Для ограничения параллельности используйте buffer_unordered.",
    )?;
    dom::add_class(&dom::as_html(&intro8), "hint")?;
    dom::append(&section8, &intro8)?;

    let parallel_btn = dom::create_element("button")?;
    parallel_btn.set_text_content(Some("3 запроса параллельно"));
    let sequential_btn = dom::create_element("button")?;
    sequential_btn.set_text_content(Some("3 запроса последовательно"));
    let race_btn = dom::create_element("button")?;
    race_btn.set_text_content(Some("Race: 1с vs 3с vs 5с"));
    let settled_btn = dom::create_element("button")?;
    settled_btn.set_text_content(Some("AllSettled: 2 успеха, 1 ошибка"));
    let limited_btn = dom::create_element("button")?;
    limited_btn.set_text_content(Some("5 запросов, лимит 2"));

    dom::append(&section8, &parallel_btn)?;
    dom::append(&section8, &sequential_btn)?;
    dom::append(&section8, &race_btn)?;
    dom::append(&section8, &settled_btn)?;
    dom::append(&section8, &limited_btn)?;

    let result8 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result8), "api-result")?;
    result8.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section8, &result8)?;

    // ---- Параллельно: 3 запроса по 1с ----
    {
        let result8 = result8.clone();
        callbacks::listen(
            &parallel_btn,
            "click",
            move |_| {
                let result8 = result8.clone();
                result8.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let urls = [
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://localhost:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                    ];

                    let start = js_sys::Date::now();
                    let result = fetch::get_all(&urls).await;
                    let elapsed = js_sys::Date::now() - start;

                    match result {
                        Ok(responses) => {
                            let text = format!(
                                "✅ Promise.all: {} запросов\n\
                                 Время: {:.0} мс\n\
                                 Статусы: {:?}\n\n\
                                 Последовательно было бы ~3000 мс.",
                                responses.len(),
                                elapsed,
                                responses.iter().map(|r| r.status()).collect::<Vec<_>>()
                            );
                            result8.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result8.set_text_content(Some(&format!("❌ {:?}", e)));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Последовательно: 3 запроса по 1с ----
    {
        let result8 = result8.clone();
        callbacks::listen(
            &sequential_btn,
            "click",
            move |_| {
                let result8 = result8.clone();
                result8.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let start = js_sys::Date::now();
                    let mut statuses = Vec::new();

                    for url in [
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                    ] {
                        match fetch::send_request_classified(url).await {
                            Ok(resp) => statuses.push(resp.status()),
                            Err(e) => {
                                result8.set_text_content(Some(&format!("❌ {:?}", e)));
                                return;
                            }
                        }
                    }

                    let elapsed = js_sys::Date::now() - start;
                    let text = format!(
                        "✅ Последовательно: {} запросов\n\
                         Время: {:.0} мс\n\
                         Статусы: {:?}\n\n\
                         Параллельно было бы ~1000 мс.",
                        statuses.len(),
                        elapsed,
                        statuses
                    );
                    result8.set_text_content(Some(&text));
                });
            },
            &closures,
        )?;
    }

    // ---- Race: кто быстрее ----
    {
        let result8 = result8.clone();
        callbacks::listen(
            &race_btn,
            "click",
            move |_| {
                let result8 = result8.clone();
                result8.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let urls = [
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://localhost:3000/api/delay/3",
                        "http://127.0.0.1:3000/api/delay/5",
                    ];

                    let start = js_sys::Date::now();
                    let result = fetch::get_race(&urls).await;
                    let elapsed = js_sys::Date::now() - start;

                    match result {
                        Ok(resp) => {
                            let text = format!(
                                "✅ Promise.race: победил первый\n\
                                 Время: {:.0} мс\n\
                                 Статус: {}\n\n\
                                 Остальные запросы всё ещё в полёте, \
                                 но мы их игнорируем.",
                                elapsed,
                                resp.status()
                            );
                            result8.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            result8.set_text_content(Some(&format!("❌ {:?}", e)));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- AllSettled: один упал, остальные живы ----
    {
        let result8 = result8.clone();
        callbacks::listen(
            &settled_btn,
            "click",
            move |_| {
                let result8 = result8.clone();
                result8.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let urls = [
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://localhost:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/error500",
                    ];

                    let result = fetch::get_all_settled(&urls).await;

                    match result {
                        Ok(results) => {
                            let mut lines = vec!["✅ Promise.allSettled:".to_string()];
                            for (i, r) in results.iter().enumerate() {
                                match r {
                                    Ok(resp) => lines.push(format!(
                                        "  [{}] ✅ статус {}",
                                        i,
                                        resp.status()
                                    )),
                                    Err(e) => lines.push(format!(
                                        "  [{}] ❌ {:?}",
                                        i, e
                                    )),
                                }
                            }
                            lines.push(String::new());
                            lines.push(
                                "Promise.all упал бы на третьем запросе \
                                 и мы не увидели бы первые два."
                                    .to_string(),
                            );
                            result8.set_text_content(Some(&lines.join("\n")));
                        }
                        Err(e) => {
                            result8.set_text_content(Some(&format!("❌ {:?}", e)));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Ограничение параллельности: 5 URL-ов, лимит 2 ----
    {
        let result8 = result8.clone();
        callbacks::listen(
            &limited_btn,
            "click",
            move |_| {
                let result8 = result8.clone();
                result8.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let urls = [
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                        "http://127.0.0.1:3000/api/delay/1",
                    ];

                    let start = js_sys::Date::now();
                    let results = fetch::get_all_limited(&urls, 2).await;
                    let elapsed = js_sys::Date::now() - start;

                    let ok_count = results.iter().filter(|r| r.is_ok()).count();
                    let text = format!(
                        "✅ buffer_unordered(2): 5 запросов\n\
                         Время: {:.0} мс\n\
                         Успешных: {}\n\n\
                         Без лимита было бы ~1000 мс, \
                         с лимитом 2 — ~3000 мс \
                         (по 2 запроса за раз).",
                        elapsed,
                        ok_count
                    );
                    result8.set_text_content(Some(&text));
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_parallel.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Проверяет, что запросы действительно идут одновременно.
///
/// Считаем максимальное количество одновременных запросов на сервере.
/// Если параллелизм работает, счётчик поднимется выше 1.
///
/// **Почему `localhost` и `127.0.0.1` чередуются.** Headless Chrome
/// сериализует запросы к одному origin: браузер открывает одно
/// соединение и шлёт запросы по очереди. Но `localhost` и `127.0.0.1`
/// Chrome трактует как **разные origins** — у них разные пулы
/// соединений. Это позволяет проверить параллельность в headless-среде,
/// не отключая headless-режим.
///
/// В обычном Chrome такого трюка не требуется — там пул на origin
/// вмещает 6 соединений.
#[wasm_bindgen_test]
async fn test_parallel_requests_are_concurrent() {
    let _ = hello_wasm02::fetch::send_request_classified(
        "http://127.0.0.1:3000/api/concurrency/reset",
    )
    .await;

    let urls = [
        "http://127.0.0.1:3000/api/concurrency/test",
        "http://localhost:3000/api/concurrency/test",
        "http://127.0.0.1:3000/api/concurrency/test",
    ];
    let _ = hello_wasm02::fetch::get_all(&urls).await;

    let resp = hello_wasm02::fetch::send_request_classified(
        "http://127.0.0.1:3000/api/concurrency/stats",
    )
    .await
    .unwrap();

    let json: serde_json::Value = hello_wasm02::fetch::response_json(&resp)
        .await
        .unwrap();
    let max = json["max_concurrent"].as_u64().unwrap_or(0);

    assert!(
        max >= 2,
        "параллельные запросы должны идти одновременно, \
         max_concurrent = {max}. Если 1 — браузер сериализует \
         запросы к одному origin, и это ограничение среды, \
         а не нашего кода."
    );
}

/// Promise.all сохраняет порядок результатов.
#[wasm_bindgen_test]
async fn test_parallel_preserves_order() {
    let urls = [
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/2",
        "https://jsonplaceholder.typicode.com/posts/3",
    ];

    let responses = hello_wasm02::fetch::get_all(&urls).await.unwrap();
    assert_eq!(responses.len(), 3);

    for r in &responses {
        assert_eq!(r.status(), 200);
    }
}

/// Promise.all падает при первой HTTP-ошибке.
#[wasm_bindgen_test]
async fn test_parallel_fails_fast() {
    let urls = [
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/999999",
        "https://jsonplaceholder.typicode.com/posts/2",
    ];

    let result = hello_wasm02::fetch::get_all(&urls).await;
    assert!(result.is_err(), "Promise.all должен упасть на 404");
}

/// AllSettled возвращает результаты даже при ошибках.
#[wasm_bindgen_test]
async fn test_all_settled_partial_failure() {
    let urls = [
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/999999",
    ];

    let results = hello_wasm02::fetch::get_all_settled(&urls).await.unwrap();
    assert_eq!(results.len(), 2);

    assert!(results[0].is_ok());
    assert!(results[1].is_err());
}

/// Race: побеждает первый завершившийся.
#[wasm_bindgen_test]
async fn test_race_first_wins() {
    let urls = [
        "http://127.0.0.1:3000/api/delay/2",
        "http://localhost:3000/api/delay/1",
        "http://127.0.0.1:3000/api/delay/3",
    ];

    let start = js_sys::Date::now();
    let result = hello_wasm02::fetch::get_race(&urls).await;
    let elapsed = js_sys::Date::now() - start;

    assert!(result.is_ok());
    assert!(elapsed < 2000.0, "race должен завершиться быстро, получено {elapsed} мс");
}

/// Параллельные запросы с AbortController.
#[wasm_bindgen_test]
async fn test_parallel_with_abort() {
    let controller = hello_wasm02::fetch::make_abort_controller().unwrap();
    let signal = controller.signal();

    controller.abort();

    let result = hello_wasm02::fetch::send_with_signal(
        "http://127.0.0.1:3000/api/delay/1",
        &signal,
    )
    .await;

    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Aborted => {}
        other => panic!("ожидался Aborted, получено {other:?}"),
    }
}

/// Ограничение параллельности: 5 запросов, лимит 2.
#[wasm_bindgen_test]
async fn test_parallel_limited_concurrency() {
    let urls = [
        "http://127.0.0.1:3000/api/delay/1",
        "http://localhost:3000/api/delay/1",
        "http://127.0.0.1:3000/api/delay/1",
        "http://localhost:3000/api/delay/1",
        "http://127.0.0.1:3000/api/delay/1",
    ];

    let start = js_sys::Date::now();
    let results = hello_wasm02::fetch::get_all_limited(&urls, 2).await;
    let elapsed = js_sys::Date::now() - start;

    assert_eq!(results.len(), 5);
    assert!(results.iter().all(|r| r.is_ok()));

    assert!(
        elapsed > 2000.0,
        "с лимитом 2 должно быть медленнее 2000 мс, получено {elapsed} мс"
    );
}

/// Три параллельных запроса возвращают три результата.
#[wasm_bindgen_test]
async fn test_parallel_three_requests() {
    let urls = [
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/2",
        "https://jsonplaceholder.typicode.com/posts/3",
    ];

    let responses = hello_wasm02::fetch::get_all(&urls).await.unwrap();
    assert_eq!(responses.len(), 3);

    for (i, r) in responses.iter().enumerate() {
        assert_eq!(r.status(), 200, "запрос {i} вернул {}", r.status());
    }
}
```

## Команды запуска

### Разработка (три терминала)

**Терминал 1 — сервер:**

```bash
npm run server
```

После правок в `handlers.rs` и `main.rs` сервер нужно перезапустить.

**Терминал 2 — сборка WASM и dev-сервер Vite:**

```bash
npm run wasm:dev
npm run dev
```

**Терминал 3 — тесты:**

```bash
npm run test:chrome
```

Ожидаемый результат: `36 passed; 0 failed` (28 из глав 16.4–16.7 + 8 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_parallel
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
cargo check --manifest-path auth-server/Cargo.toml
```

### Ручная проверка сервера

```bash
curl -i http://127.0.0.1:3000/api/delay/2
curl -i http://127.0.0.1:3000/api/concurrency/reset
curl -i http://127.0.0.1:3000/api/concurrency/stats
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «8. Параллельные запросы»:

| Кнопка | Ожидаемый результат |
|---|---|
| «3 запроса параллельно» | Время: ~1000 мс |
| «3 запроса последовательно» | Время: ~3000 мс |
| «Race: 1с vs 3с vs 5с» | Время: ~1000 мс |
| «AllSettled: 2 успеха, 1 ошибка» | Три результата: два 200, один 500 |
| «5 запросов, лимит 2» | Время: ~3000 мс |

## Итог главы

Мы научились отправлять несколько запросов одновременно и управлять параллелизмом:

1. **`get_all`** — обёртка над `Promise.all`. Ждёт все запросы, возвращает `Vec<Response>` в исходном порядке, падает при первой HTTP-ошибке.
2. **`get_all_settled`** — обёртка над `Promise.allSettled`. Ждёт все запросы, возвращает `Vec<Result<Response, FetchError>>`, не падает.
3. **`get_race`** — обёртка над `Promise.race`. Возвращает первый завершившийся запрос, игнорирует остальные.
4. **`get_all_limited`** — ограничение параллельности через `futures::stream::buffer_unordered`.

Ключевые уроки:

- **Параллельные запросы экономят время.** Три запроса по 500 мс параллельно — 500 мс, а не 1500 мс.
- **`fetch` резолвит промис даже при 404 и 500.** Ошибкой считается только сетевой сбой. Значит, `response.ok()` мы проверяем сами — иначе `Promise.all` не заметит HTTP-ошибку.
- **`Promise.all` — «всё или ничего».** Один упавший запрос обнуляет результаты всех. Если нужны все результаты даже при частичных ошибках — `allSettled`.
- **`Promise.race` — для таймаутов и fallback.** Primary и secondary серверы, кто первый ответит — того и результат.
- **`js_sys::Promise::all` / `race` / `allSettled` — вызовы JavaScript-методов.** Не переписывайте их на Rust: браузерные реализации надёжнее и быстрее.
- **Ограничение параллельности — ваша ответственность.** Сто запросов разом браузер начнёт душить, а сервер ответит 429. `buffer_unordered` решает проблему.
- **Headless Chrome сериализует запросы к одному origin.** Это ограничение тестовой среды, а не кода. Обходится чередованием `127.0.0.1` и `localhost` — Chrome трактует их как разные origins с разными пулами соединений.
- **`buffer_unordered` возвращает результаты в порядке завершения.** Если нужен исходный порядок — используйте `join_all`, но он не ограничивает параллельность.

В следующей главе — **типы ответов и бинарные данные**: `text()`, `json()`, `blob()`, `arrayBuffer()`, `ReadableStream` и работа с файлами.