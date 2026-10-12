# Глава 16.11. CORS и cross-origin

## Обновлённая структура проекта

```
.
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +make_cors_request, make_same_origin_request,
│   │                                  make_request_with_credentials, send_prepared_request
│   ├── app.rs                      ← ОБНОВЛЁН: секция 11 «CORS» + helper describe_error
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_cors.rs                ← НОВЫЙ: 4 теста CORS
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: +фича web-sys RequestCredentials
└── (корневые файлы без изменений)
```

**Краткий комментарий.** CORS — тема, которая пугает новичков больше, чем все остальные в Fetch API вместе взятые. Сообщение «Cross-Origin Request Blocked» в консоли выглядит как ошибка, но на самом деле это **работающая защита браузера**. Браузер по умолчанию запрещает JavaScript читать ответы с чужого origin — иначе любой сайт мог бы дёргать ваш банк от вашего имени. CORS — это способ, которым **сервер** говорит браузеру: «этому конкретному origin можно». В этой главе мы разберём, как работает same-origin policy, что такое preflight, какие заголовки участвуют, и почему одни cross-origin запросы проходят, а другие нет — причём не из-за ошибки в коде, а из-за конфигурации сервера.

## Ссылки для детального изучения

- **MDN: CORS** — основной обзор механизма: [developer.mozilla.org/ru/docs/Web/HTTP/CORS](https://developer.mozilla.org/ru/docs/Web/HTTP/CORS)
- **MDN: Same-origin policy** — что такое «один origin» и почему это важно: [developer.mozilla.org/ru/docs/Web/Security/Same-origin_policy](https://developer.mozilla.org/ru/docs/Web/Security/Same-origin_policy)
- **MDN: `Access-Control-Allow-Origin`** — главный заголовок CORS: [developer.mozilla.org/ru/docs/Web/HTTP/Headers/Access-Control-Allow-Origin](https://developer.mozilla.org/ru/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)
- **MDN: Preflight request** — когда браузер шлёт OPTIONS и зачем: [developer.mozilla.org/ru/docs/Glossary/Preflight_request](https://developer.mozilla.org/ru/docs/Glossary/Preflight_request)
- **MDN: `Request.credentials`** — режимы работы с cookies: [developer.mozilla.org/ru/docs/Web/API/Request/credentials](https://developer.mozilla.org/ru/docs/Web/API/Request/credentials)
- **MDN: `Request.mode`** — режимы запроса (`cors`, `no-cors`, `same-origin`): [developer.mozilla.org/ru/docs/Web/API/Request/mode](https://developer.mozilla.org/ru/docs/Web/API/Request/mode)
- **Fetch Standard: CORS protocol** — официальная спецификация: [fetch.spec.whatwg.org/#http-cors-protocol](https://fetch.spec.whatwg.org/#http-cors-protocol)
- **`tower-http::cors`** — настройка CORS в Axum: [docs.rs/tower-http/latest/tower_http/cors/index.html](https://docs.rs/tower-http/latest/tower_http/cors/index.html)

## Новые и обновлённые файлы

### `Cargo.toml` (корневой, обновлён)

Добавить в `features` для `web-sys`:

```toml
"RequestCredentials",
```

`RequestMode` уже подключён — он нужен был для всех предыдущих глав.

### `src/fetch.rs` (обновлён — добавить в конец)

```rust
// ============================================================
// CORS и режимы запроса
// ============================================================

/// Создаёт GET-запрос в режиме `cors`.
///
/// Это **режим по умолчанию** для cross-origin запросов. Браузер:
/// 1. Проверяет, является ли запрос простым (simple).
/// 2. Если нет — отправляет preflight OPTIONS.
/// 3. Только после успешного preflight делает настоящий запрос.
///
/// Сервер должен ответить заголовками `Access-Control-Allow-Origin`
/// и сопутствующими, иначе браузер заблокирует ответ.
///
/// Мы используем `cors` во всех предыдущих главах, потому что
/// запросы шли на другой origin (JSONPlaceholder, наш auth-server
/// на другом порту). Если бы мы использовали `same-origin`, браузер
/// отклонил бы их сразу, не отправляя на сервер.
pub fn make_cors_request(url: &str) -> Result<web_sys::Request, JsValue> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    web_sys::Request::new_with_str_and_init(url, &opts)
}

/// Создаёт GET-запрос в режиме `same-origin`.
///
/// Браузер **отклонит запрос**, если URL ведёт на другой origin.
/// Не отправит ни preflight, ни сам запрос — просто выбросит
/// `TypeError: Failed to fetch`.
///
/// Зачем это нужно:
/// - Безопасность. Если вы точно знаете, что URL должен быть
///   локальным, режим `same-origin` гарантирует, что запрос
///   не уйдёт наружу даже при опечатке в URL.
/// - Экономия. Не отправляется preflight, что экономит один
///   round-trip.
///
/// Для нашего приложения, работающего через Vite-прокси, запросы
/// к относительным путям (`/api/me`) идут на `localhost:5173` —
/// это тот же origin, что и страница. Режим `same-origin` для них
/// работает без CORS.
pub fn make_same_origin_request(url: &str) -> Result<web_sys::Request, JsValue> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::SameOrigin);

    web_sys::Request::new_with_str_and_init(url, &opts)
}

/// Создаёт GET-запрос с credentials: `include`.
///
/// `credentials` управляет тем, отправляет ли браузер cookies
/// и HTTP-аутентификацию вместе с запросом. Три значения:
///
/// - `Omit` — никогда не отправлять cookies.
/// - `SameOrigin` — только если запрос на тот же origin. **Дефолт.**
/// - `Include` — отправлять всегда, даже на cross-origin.
///
/// **Важно.** При `include` сервер **обязан** ответить заголовком
/// `Access-Control-Allow-Credentials: true`. Просто `Allow-Origin: *`
/// недостаточно — браузер отклонит ответ, потому что wildcard
/// несовместим с credentials. Сервер должен явно перечислить
/// origin (`Access-Control-Allow-Origin: https://app.example.com`).
///
/// `CorsLayer::permissive()` в нашем auth-server **не разрешает**
/// credentials — именно поэтому такой запрос демонстративно
/// упадёт на preflight.
pub fn make_request_with_credentials(url: &str) -> Result<web_sys::Request, JsValue> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);
    opts.set_credentials(web_sys::RequestCredentials::Include);

    web_sys::Request::new_with_str_and_init(url, &opts)
}

/// Отправляет запрос и возвращает классифицированный результат.
///
/// Общая обёртка для всех трёх функций выше. Проверяет `ok()`
/// и читает тело ошибки, как `send_request_classified`.
pub async fn send_prepared_request(
    request: &web_sys::Request,
) -> Result<web_sys::Response, FetchError> {
    let window = web_sys::window().ok_or_else(|| FetchError::Network("no window object".into()))?;

    let resp_value = wasm_bindgen_futures::JsFuture::from(window.fetch_with_request(request))
        .await
        .map_err(classify_error)?;

    let response: web_sys::Response = resp_value
        .dyn_into::<web_sys::Response>()
        .map_err(classify_error)?;

    if !response.ok() {
        let status = response.status();
        let body = read_error_body(&response).await;
        return Err(FetchError::Http(status, body));
    }

    Ok(response)
}
```

### `src/app.rs` — helper `describe_error`

Добавьте в конец файла, рядом с `truncate_preview`:

```rust
/// Возвращает человекочитаемое описание ошибки с эмодзи и подсказкой.
///
/// Используется в обработчиках для замены технического `{:?}` на
/// осмысленное сообщение. Разные варианты `FetchError` превращаются
/// в короткие фразы: «🔒 Требуется вход», «🔍 Не найдено» и т.д.
///
/// Для мест, где ошибка приходит как `JsValue`, сначала вызывайте
/// `fetch::classify_error(e)`, потом передавайте результат сюда.
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

### `src/app.rs` — секция 11 «CORS»

После секции 10, перед секцией 12:

```rust
    // ============================================================
    // СЕКЦИЯ 11: CORS
    // ============================================================

    let section11 = build_section(&container, "11. CORS")?;

    let intro11 = dom::create_paragraph(
        "Same-origin policy запрещает JavaScript читать ответы \
         с чужого origin. CORS — способ, которым сервер разрешает \
         это для конкретных origin. Preflight (OPTIONS) отправляется, \
         когда запрос не «простой»: нестандартные методы, заголовки, \
         Content-Type. credentials: include требует \
         Access-Control-Allow-Credentials: true на сервере — \
         wildcard '*' не подходит.",
    )?;
    dom::add_class(&dom::as_html(&intro11), "hint")?;
    dom::append(&section11, &intro11)?;

    let cors_btn = dom::create_element("button")?;
    cors_btn.set_text_content(Some("CORS-запрос (JSONPlaceholder)"));
    let same_origin_btn = dom::create_element("button")?;
    same_origin_btn.set_text_content(Some("Same-origin (/api/me)"));
    let credentials_btn = dom::create_element("button")?;
    credentials_btn.set_text_content(Some("С credentials: include"));
    let no_cors_btn = dom::create_element("button")?;
    no_cors_btn.set_text_content(Some("Cross-origin без CORS"));

    dom::append(&section11, &cors_btn)?;
    dom::append(&section11, &same_origin_btn)?;
    dom::append(&section11, &credentials_btn)?;
    dom::append(&section11, &no_cors_btn)?;

    let result11 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result11), "api-result")?;
    result11.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section11, &result11)?;

    // ---- CORS-запрос к JSONPlaceholder ----
    {
        let result11 = result11.clone();
        callbacks::listen(
            &cors_btn,
            "click",
            move |_| {
                let result11 = result11.clone();
                result11.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/1";
                    let request = match fetch::make_cors_request(url) {
                        Ok(r) => r,
                        Err(e) => {
                            // Здесь e — JsValue: классифицируем,
                            // чтобы получить человекочитаемое сообщение.
                            let classified = fetch::classify_error(e);
                            result11.set_text_content(Some(&format!(
                                "❌ {}",
                                describe_error(&classified)
                            )));
                            return;
                        }
                    };

                    match fetch::send_prepared_request(&request).await {
                        Ok(resp) => {
                            let text = format!(
                                "✅ CORS-запрос прошёл\n\n\
                                 URL: {url}\n\
                                 Режим: cors\n\
                                 Статус: {}\n\n\
                                 Сервер JSONPlaceholder отправляет \
                                 Access-Control-Allow-Origin: * — \
                                 поэтому браузер разрешил чтение.\n\n\
                                 В DevTools → Network виден preflight \
                                 OPTIONS перед основным запросом, \
                                 потому что метод GET с заголовком \
                                 Accept не считается «простым».",
                                resp.status()
                            );
                            result11.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            // Здесь e — FetchError: describe_error напрямую.
                            result11.set_text_content(Some(&format!(
                                "❌ {}",
                                describe_error(&e)
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Same-origin через прокси ----
    {
        let result11 = result11.clone();
        callbacks::listen(
            &same_origin_btn,
            "click",
            move |_| {
                let result11 = result11.clone();
                result11.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Относительный путь — тот же origin, что и страница.
                    // Vite-прокси перенаправит его на auth-server,
                    // но для браузера это локальный запрос.
                    let url = "/api/me";
                    let request = match fetch::make_same_origin_request(url) {
                        Ok(r) => r,
                        Err(e) => {
                            let classified = fetch::classify_error(e);
                            result11.set_text_content(Some(&format!(
                                "❌ {}",
                                describe_error(&classified)
                            )));
                            return;
                        }
                    };

                    match fetch::send_prepared_request(&request).await {
                        Ok(resp) => {
                            let status = resp.status();
                            let text = format!(
                                "✅ Same-origin запрос\n\n\
                                 URL: {url}\n\
                                 Режим: same-origin\n\
                                 Статус: {status}\n\n\
                                 Preflight не отправлялся: браузер \
                                 видит, что origin совпадает с \
                                 origin страницы. CORS-заголовки \
                                 не требуются.\n\n\
                                 Обратите внимание: без авторизации \
                                 сервер вернёт 401 — но это уже \
                                 HTTP-ошибка, а не блокировка CORS.",
                                status = status
                            );
                            result11.set_text_content(Some(&text));
                        }
                        Err(fetch::FetchError::Http(status, _)) => {
                            result11.set_text_content(Some(&format!(
                                "✅ Same-origin запрос дошёл до сервера\n\n\
                                 URL: /api/me\n\
                                 Режим: same-origin\n\
                                 Статус: {status}\n\n\
                                 CORS не помешал. 401 — это уже \
                                 логика приложения: токен не передан.",
                                status = status
                            )));
                        }
                        Err(e) => {
                            result11.set_text_content(Some(&format!(
                                "❌ {}",
                                describe_error(&e)
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Запрос с credentials: include ----
    {
        let result11 = result11.clone();
        callbacks::listen(
            &credentials_btn,
            "click",
            move |_| {
                let result11 = result11.clone();
                result11.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Этот запрос пойдёт на auth-server с credentials: include.
                    // Сервер использует CorsLayer::permissive(), который
                    // НЕ ставит Access-Control-Allow-Credentials: true.
                    // Поэтому preflight провалится, и настоящий запрос
                    // не уйдёт.
                    let url = "http://127.0.0.1:3000/api/me";
                    let request = match fetch::make_request_with_credentials(url) {
                        Ok(r) => r,
                        Err(e) => {
                            let classified = fetch::classify_error(e);
                            result11.set_text_content(Some(&format!(
                                "❌ {}",
                                describe_error(&classified)
                            )));
                            return;
                        }
                    };

                    match fetch::send_prepared_request(&request).await {
                        Ok(resp) => {
                            let text = format!(
                                "⚠️ Запрос прошёл (неожиданно)\n\n\
                                 URL: {url}\n\
                                 Режим: cors, credentials: include\n\
                                 Статус: {}\n\n\
                                 Если сервер разрешает credentials, \
                                 запрос проходит. Проверьте заголовок \
                                 Access-Control-Allow-Credentials.",
                                resp.status()
                            );
                            result11.set_text_content(Some(&text));
                        }
                        Err(e) => {
                            let text = format!(
                                "❌ Запрос отклонён\n\n\
                                 URL: {url}\n\
                                 Режим: cors, credentials: include\n\
                                 Ошибка: {}\n\n\
                                 Ожидаемо: наш auth-server использует \
                                 CorsLayer::permissive(), который не \
                                 ставит Access-Control-Allow-Credentials: \
                                 true. При credentials: include браузер \
                                 требует этот заголовок. В консоли \
                                 будет «CORS request did not succeed».",
                                describe_error(&e)
                            );
                            result11.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Cross-origin без CORS ----
    {
        let result11 = result11.clone();
        callbacks::listen(
            &no_cors_btn,
            "click",
            move |_| {
                let result11 = result11.clone();
                result11.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // example.com не отправляет CORS-заголовков.
                    // Браузер отклонит чтение ответа.
                    let url = "https://example.com/";
                    let request = match fetch::make_cors_request(url) {
                        Ok(r) => r,
                        Err(e) => {
                            let classified = fetch::classify_error(e);
                            result11.set_text_content(Some(&format!(
                                "❌ {}",
                                describe_error(&classified)
                            )));
                            return;
                        }
                    };

                    match fetch::send_prepared_request(&request).await {
                        Ok(_) => {
                            result11.set_text_content(Some(
                                "⚠️ Запрос прошёл (неожиданно — проверьте, \
                                 не добавил ли example.com CORS-заголовки)",
                            ));
                        }
                        Err(e) => {
                            let text = format!(
                                "❌ Cross-origin без CORS: запрос отклонён\n\n\
                                 URL: {url}\n\
                                 Ошибка: {}\n\n\
                                 Пример ожидаемого: сервер не отправляет \
                                 Access-Control-Allow-Origin, поэтому \
                                 браузер не даёт JavaScript прочитать \
                                 ответ. В консоли будет \
                                 «Cross-Origin Request Blocked».\n\n\
                                 Это защита same-origin policy в действии, \
                                 а не ошибка кода.",
                                describe_error(&e)
                            );
                            result11.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_cors.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// CORS-запрос к JSONPlaceholder проходит.
#[wasm_bindgen_test]
async fn test_cors_request_succeeds() {
    let url = "https://jsonplaceholder.typicode.com/posts/1";
    let request = hello_wasm02::fetch::make_cors_request(url).unwrap();
    let resp = hello_wasm02::fetch::send_prepared_request(&request)
        .await
        .unwrap();
    assert_eq!(resp.status(), 200);
}

/// Cross-origin без CORS-заголовков отклоняется браузером.
#[wasm_bindgen_test]
async fn test_cross_origin_without_cors_fails() {
    let url = "https://example.com/";
    let request = hello_wasm02::fetch::make_cors_request(url).unwrap();
    let result = hello_wasm02::fetch::send_prepared_request(&request).await;
    assert!(result.is_err(), "example.com не отправляет CORS-заголовки");
}

/// Same-origin режим работает для относительных URL.
#[wasm_bindgen_test]
async fn test_same_origin_mode() {
    let url = "/api/flaky/reset";
    let request = hello_wasm02::fetch::make_same_origin_request(url).unwrap();
    let result = hello_wasm02::fetch::send_prepared_request(&request).await;
    assert!(result.is_ok());
}

/// Режим запроса устанавливается корректно.
#[wasm_bindgen_test]
async fn test_request_mode_set() {
    let url = "https://jsonplaceholder.typicode.com/posts/1";
    let request = hello_wasm02::fetch::make_cors_request(url).unwrap();
    assert_eq!(request.mode(), "cors");

    let same = hello_wasm02::fetch::make_same_origin_request("/").unwrap();
    assert_eq!(same.mode(), "same-origin");
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

Ожидаемый результат: `62 passed; 0 failed` (58 из глав 16.4–16.10 + 4 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_cors
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «11. CORS»:

| Кнопка | Ожидаемый результат |
|---|---|
| «CORS-запрос (JSONPlaceholder)» | ✅ 200 OK, в Network виден preflight OPTIONS |
| «Same-origin (/api/me)» | ✅ 401 или 200 — preflight отсутствует |
| «С credentials: include» | ❌ CORS-ошибка (expected) |
| «Cross-origin без CORS» | ❌ Cross-Origin Request Blocked (expected) |

## Как читать CORS-ошибки в консоли

Когда вы нажмёте «Cross-origin без CORS» или «С credentials: include», в DevTools появится красное сообщение. Разберём типичные формулировки:

**`Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at ...`**

Ответ пришёл, но в нём нет `Access-Control-Allow-Origin` (или он не совпадает с вашим origin). Браузер не даёт JavaScript прочитать тело. Сам запрос при этом ушёл — сервер его получил.

**`Reason: CORS header 'Access-Control-Allow-Origin' missing`**

Сервер вообще не настроил CORS. Самая частая причина — забыли добавить `tower_http::cors::CorsLayer` в роутер.

**`Reason: CORS request did not succeed`**

Preflight OPTIONS вернул что-то, что не устроило браузер. Возможно, статус 4xx, или сервер не ответил на OPTIONS вовсе, или нет `Access-Control-Allow-Headers` для нестандартного заголовка.

**`Reason: Credential is not supported if the CORS header 'Access-Control-Allow-Origin' is '*'`**

Классика: вы отправляете `credentials: include`, а сервер отвечает `Access-Control-Allow-Origin: *`. Спецификация запрещает такую комбинацию — wildcard несовместим с credentials. Сервер должен явно перечислить origin клиента.

**`Access to fetch at '...' from origin '...' has been blocked by CORS policy`**

Общая формулировка, означающая «что-то не так с CORS». Смотрите на `Reason:` — там конкретика.

## Обратите внимание на типы ошибок

В этой главе мы работаем с двумя источниками ошибок:

- `make_cors_request`, `make_same_origin_request`, `make_request_with_credentials` — возвращают `Result<_, JsValue>`. Это «сырая» ошибка от создания `Request`. Она возникает редко — например, если URL синтаксически некорректен. Чтобы получить человекочитаемое сообщение, вызываем `fetch::classify_error(e)`, а потом `describe_error`.
- `send_prepared_request` — возвращает `Result<_, FetchError>`. Это уже классифицированная ошибка после отправки. `describe_error` применяется напрямую.

Именно поэтому в обработчиках две разные ветки `Err` с разной обработкой. Если перепутать — компилятор скажет `expected &FetchError, found &JsValue`.

## Итог главы

Мы разобрали CORS — механизм, который одновременно защищает пользователя и вызывает больше всего путаницы у разработчиков:

1. **Same-origin policy** — фундамент безопасности браузера. JavaScript со страницы `app.example.com` не может читать ответы с `bank.example.com`, даже если запрос ушёл. Браузер блокирует **чтение**, а не отправку.
2. **CORS** — способ сервера сказать «этому origin можно». Без `Access-Control-Allow-Origin` в ответе JavaScript не получит данные.
3. **`RequestMode`** — `cors`, `same-origin`, `no-cors`, `navigate`. `cors` — стандарт для cross-origin. `same-origin` жёстче: запрос на другой origin вообще не уйдёт.
4. **Simple vs preflight** — простые запросы (GET, POST, HEAD с ограниченными заголовками) идут сразу. Всё остальное требует preflight OPTIONS.
5. **`Access-Control-Allow-Origin`** — главный заголовок. `*` работает только без credentials.
6. **`Access-Control-Allow-Credentials: true`** — обязателен, если клиент отправляет cookies. Несовместим с `*` в `Allow-Origin`.
7. **Credentials** — `omit` (никогда), `same-origin` (дефолт), `include` (всегда).

Ключевые уроки:

- **CORS-ошибка — это не баг вашего кода.** Это конфигурация сервера. Ваш `fetch` может быть идеален, но если сервер не отправил нужных заголовков, браузер всё равно заблокирует.
- **Запрос может уйти, а ответ — нет.** Same-origin policy блокирует **чтение**, а не отправку. Это сбивает с толку: в Network видно, что сервер ответил, но в JS — ошибка.
- **`credentials: include` несовместим с `Allow-Origin: *`.** Спецификация требует явный origin в этом случае. Это самая частая причина необъяснимых CORS-ошибок при работе с авторизацией.
- **Vite-прокси избавляет от CORS в разработке.** Запрос к `/api/...` идёт на тот же origin, что и страница, и CORS не участвует.
- **`CorsLayer::permissive()` — только для разработки.** В продакшене нужно явно перечислять разрешённые origin, методы и заголовки.

В следующей главе — **финальная сборка: API client**, объединяющий всё, что мы построили: типизированные запросы, автоматический retry, refresh токена, обработка ошибок и потоковое чтение в одном аккуратном интерфейсе.