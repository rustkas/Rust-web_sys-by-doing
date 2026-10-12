# Глава 16.5. Ошибки: различение и классификация

## Обновлённая структура проекта

```
.
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +FetchError, classify_error, send_request_classified
│   ├── app.rs                      ← ОБНОВЛЁН: секция 5 «Ошибки»
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_errors.rs              ← НОВЫЙ: 8 тестов классификации ошибок
│   └── (остальные без изменений)
│
└── (корневые файлы без изменений)
```

**Краткий комментарий.** Глава не требует новых зависимостей и не трогает серверную часть. Всё сосредоточено в двух файлах: `fetch.rs` получает `enum FetchError` и функции классификации, а `app.rs` — пятую секцию с демонстрацией четырёх типов ошибок. Добавляется один новый файл тестов.

## Ссылки для детального изучения

- **MDN: Fetch API — Handling errors**: [developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#checking_that_the_fetch_was_successful](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#checking_that_the_fetch_was_successful)
- **MDN: `AbortController`** — как отменять запросы и почему приходит `AbortError`: [developer.mozilla.org/ru/docs/Web/API/AbortController](https://developer.mozilla.org/ru/docs/Web/API/AbortController)
- **MDN: `DOMException`** — список подтипов (`AbortError`, `NetworkError`, `TimeoutError`): [developer.mozilla.org/ru/docs/Web/API/DOMException](https://developer.mozilla.org/ru/docs/Web/API/DOMException)
- **MDN: `TypeError`** — почему сетевая ошибка в `fetch` приходит именно как `TypeError`: [developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/TypeError](https://developer.mozilla.org/ru/docs/Web/JavaScript/Reference/Global_Objects/TypeError)
- **WHATWG Fetch Standard** — разделы «Errors» и «Abort»: [fetch.spec.whatwg.org](https://fetch.spec.whatwg.org/)
- **`js_sys::Reflect::get`** — как читать свойства JS-объекта из Rust: [docs.rs/js-sys/latest/js_sys/Reflect/fn.get.html](https://docs.rs/js-sys/latest/js_sys/Reflect/fn.get.html)
- **`wasm_bindgen::JsCast`** — динамические приведения типов (`dyn_ref`, `dyn_into`, `is_instance_of`): [docs.rs/wasm-bindgen/latest/wasm_bindgen/trait.JsCast.html](https://docs.rs/wasm-bindgen/latest/wasm_bindgen/trait.JsCast.html)

## Новые и обновлённые файлы

### `src/fetch.rs` — добавить в конец файла

```rust
// ============================================================
// Классификация ошибок
// ============================================================

/// Классифицированная ошибка fetch-запроса.
///
/// Отличается от «сырого» `JsValue` тем, что каждый вариант несёт
/// информацию, достаточную для принятия решения: повторять запрос,
/// показать сообщение пользователю или молча игнорировать.
#[derive(Debug, Clone)]
pub enum FetchError {
    /// Сетевая ошибка: DNS не разрешился, CORS заблокировал,
    /// соединение оборвалось. `fetch` не вернул `Response` вообще.
    /// В JS это обычно `TypeError`.
    Network(String),

    /// Запрос дошёл до сервера, но тот ответил 4xx или 5xx.
    /// Первое поле — статус, второе — тело ответа (может быть пустым).
    Http(u16, String),

    /// Запрос отменён через `AbortController`.
    /// В JS это `DOMException` с `name === "AbortError"`.
    Aborted,

    /// Ответ получен, но не удалось разобрать тело (JSON-парсинг и т.п.).
    Parse(String),
}

impl FetchError {
    /// Человекочитаемое описание — для UI.
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
            FetchError::Aborted => return "Запрос отменён.".to_string(),
            FetchError::Parse(msg) => return format!("Ошибка разбора ответа: {msg}"),
        }
    }

    /// Стоит ли повторять запрос.
    ///
    /// Сетевые ошибки и 5xx — временные, повтор имеет смысл.
    /// 4xx и abort — постоянные, повтор только навредит.
    pub fn is_retryable(&self) -> bool {
        match self {
            FetchError::Network(_) => true,
            FetchError::Http(status, _) => *status >= 500,
            FetchError::Aborted => false,
            FetchError::Parse(_) => false,
        }
    }
}

/// Определяет, является ли `JsValue` ошибкой типа `AbortError`.
///
/// `AbortError` — это `DOMException` с `name === "AbortError"`.
/// Проверяем через `Reflect::get`, потому что у `JsValue` нет
/// прямого доступа к свойствам.
fn is_abort_error(value: &JsValue) -> bool {
    let name = js_sys::Reflect::get(value, &JsValue::from_str("name"));
    return matches!(name, Ok(n) if n.as_string().as_deref() == Some("AbortError"));
}

/// Классифицирует «сырую» ошибку от `fetch` в `FetchError`.
///
/// Сюда попадают только те ошибки, которые возникают **до** получения
/// `Response` — сетевые сбои и отмены. HTTP-ошибки классифицируются
/// отдельно, по статусу ответа.
pub fn classify_error(err: JsValue) -> FetchError {
    if is_abort_error(&err) {
        return FetchError::Aborted;
    }

    // TypeError в JS — это почти всегда сетевая ошибка:
    // DNS, CORS, обрыв соединения. Браузер намеренно не даёт
    // JavaScript-коду отличить эти ситуации друг от друга.
    if err.is_instance_of::<js_sys::TypeError>() {
        let msg = err
            .dyn_ref::<js_sys::Error>()
            .and_then(|e| e.message().as_string())
            .unwrap_or_else(|| "unknown network error".to_string());
        return FetchError::Network(msg);
    }

    // Всё остальное — тоже сетевая ошибка, но без деталей.
    let msg = format!("{err:?}");
    FetchError::Network(msg)
}

/// Отправляет GET-запрос и возвращает классифицированный результат.
///
/// В отличие от `get_json`, эта функция **не** пытается разобрать тело
/// как JSON. Но при HTTP-статусе ≥ 400 она **читает тело как текст**
/// и кладёт его во второй элемент `FetchError::Http`.
pub async fn send_request_classified(url: &str) -> Result<web_sys::Response, FetchError> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    let request = web_sys::Request::new_with_str_and_init(url, &opts)
        .map_err(classify_error)?;

    let window = web_sys::window().ok_or_else(|| {
        FetchError::Network("no window object".to_string())
    })?;

    let resp_value = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
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

/// Асинхронно читает тело `Response` как текст.
///
/// Используется только для ошибочных статусов: 4xx и 5xx обычно
/// отдают `text/html` или `text/plain`, а не JSON. Возвращает пустую
/// строку, если тело прочитать не удалось.
pub async fn read_error_body(response: &web_sys::Response) -> String {
    if let Ok(promise) = response.text() {
        if let Ok(value) = wasm_bindgen_futures::JsFuture::from(promise).await {
            if let Some(s) = value.as_string() {
                return s;
            }
        }
    }
    String::new()
}
```

**Замечание про `.map_err(classify_error)?`.** Идиоматичнее, чем `.map_err(|e| classify_error(e))?`. Обе формы эквивалентны, но первая короче и читается лучше.

### `src/app.rs` — добавить секцию 5

После секции 4, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 5: Ошибки
    // ============================================================

    let section5 = build_section(&container, "5. Ошибки")?;

    let intro5 = dom::create_paragraph(
        "Разные типы ошибок требуют разной реакции. \
         Сетевая ошибка — повторить. 5xx — повторить. \
         4xx — показать пользователю. Abort — игнорировать. \
         Внимание: при сетевой ошибке браузер пишет в консоль \
         «CORS request did not succeed» — это нормально. \
         JavaScript не различает DNS-сбой, обрыв соединения и \
         CORS-блокировку: все три дают TypeError: Failed to fetch.",
    )?;
    dom::add_class(&dom::as_html(&intro5), "hint")?;
    dom::append(&section5, &intro5)?;

    let network_btn = dom::create_element("button")?;
    network_btn.set_text_content(Some("Сетевая ошибка"));
    let http_404_btn = dom::create_element("button")?;
    http_404_btn.set_text_content(Some("HTTP 404"));
    let http_500_btn = dom::create_element("button")?;
    http_500_btn.set_text_content(Some("HTTP 500"));
    let abort_btn = dom::create_element("button")?;
    abort_btn.set_text_content(Some("Отменить запрос"));

    dom::append(&section5, &network_btn)?;
    dom::append(&section5, &http_404_btn)?;
    dom::append(&section5, &http_500_btn)?;
    dom::append(&section5, &abort_btn)?;

    let result5 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result5), "api-result")?;
    result5.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section5, &result5)?;

    // ---- Сетевая ошибка: несуществующий домен ----
    {
        let result5 = result5.clone();
        callbacks::listen(
            &network_btn,
            "click",
            move |_| {
                let result5 = result5.clone();
                result5.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://nonexistent-domain-xyz-42.invalid/api/test";
                    match fetch::send_request_classified(url).await {
                        Ok(_) => result5.set_text_content(Some("✅ (неожиданно)")),
                        Err(e) => {
                            let text = format!(
                                "❌ {:?}\n\n\
                                 Сообщение: {}\n\
                                 Повторять: {}\n\n\
                                 Примечание: браузер пишет в консоль \
                                 «CORS request did not succeed» — это \
                                 ожидаемо. JavaScript не различает \
                                 DNS-сбой, обрыв соединения и CORS-блокировку: \
                                 все три дают TypeError: Failed to fetch.",
                                e,
                                e.user_message(),
                                e.is_retryable()
                            );
                            result5.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- HTTP 404 ----
    {
        let result5 = result5.clone();
        callbacks::listen(
            &http_404_btn,
            "click",
            move |_| {
                let result5 = result5.clone();
                result5.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/999999";
                    match fetch::send_request_classified(url).await {
                        Ok(_) => result5.set_text_content(Some("✅ (неожиданно)")),
                        Err(e) => {
                            let text = format!(
                                "❌ {:?}\n\nСообщение: {}\nПовторять: {}",
                                e,
                                e.user_message(),
                                e.is_retryable()
                            );
                            result5.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- HTTP 500 ----
    {
        let result5 = result5.clone();
        callbacks::listen(
            &http_500_btn,
            "click",
            move |_| {
                let result5 = result5.clone();
                result5.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://httpstat.us/500";
                    match fetch::send_request_classified(url).await {
                        Ok(_) => result5.set_text_content(Some("✅ (неожиданно)")),
                        Err(e) => {
                            let text = format!(
                                "❌ {:?}\n\nСообщение: {}\nПовторять: {}",
                                e,
                                e.user_message(),
                                e.is_retryable()
                            );
                            result5.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Abort: отменяем запрос через AbortController ----
    {
        let result5 = result5.clone();
        callbacks::listen(
            &abort_btn,
            "click",
            move |_| {
                let result5 = result5.clone();
                result5.set_text_content(Some("Отправка и отмена..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let controller = web_sys::AbortController::new().unwrap();
                    let signal = controller.signal();

                    let opts = web_sys::RequestInit::new();
                    opts.set_method("GET");
                    opts.set_mode(web_sys::RequestMode::Cors);
                    opts.set_signal(Some(&signal));

                    let request = web_sys::Request::new_with_str_and_init(
                        "https://jsonplaceholder.typicode.com/posts/1",
                        &opts,
                    )
                    .unwrap();

                    // Отменяем сразу — запрос не успеет завершиться.
                    controller.abort();

                    let window = web_sys::window().unwrap();
                    let result = wasm_bindgen_futures::JsFuture::from(
                        window.fetch_with_request(&request),
                    )
                    .await;

                    match result {
                        Ok(_) => result5.set_text_content(Some("✅ (неожиданно)")),
                        Err(e) => {
                            let classified = fetch::classify_error(e);
                            let text = format!(
                                "❌ {:?}\n\nСообщение: {}\nПовторять: {}",
                                classified,
                                classified.user_message(),
                                classified.is_retryable()
                            );
                            result5.set_text_content(Some(&text));
                        }
                    }
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_errors.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Сетевая ошибка: несуществующий домен → FetchError::Network.
#[wasm_bindgen_test]
async fn test_classify_network_error() {
    let url = "https://nonexistent-domain-xyz-42.invalid/api/test";
    let result = hello_wasm02::fetch::send_request_classified(url).await;
    assert!(result.is_err(), "должна быть ошибка");

    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Network(_) => {}
        other => panic!("ожидался Network, получено {other:?}"),
    }
}

/// HTTP 404: ресурс не найден → FetchError::Http(404, _).
#[wasm_bindgen_test]
async fn test_classify_http_404() {
    let url = "https://jsonplaceholder.typicode.com/posts/999999";
    let result = hello_wasm02::fetch::send_request_classified(url).await;
    assert!(result.is_err());

    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Http(404, _) => {}
        other => panic!("ожидался Http(404), получено {other:?}"),
    }
}

/// HTTP 500: ошибка сервера → FetchError::Http(500, _).
#[wasm_bindgen_test]
async fn test_classify_http_500() {
    let url = "https://httpstat.us/500";
    let result = hello_wasm02::fetch::send_request_classified(url).await;
    assert!(result.is_err());

    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Http(500, _) => {}
        other => panic!("ожидался Http(500), получено {other:?}"),
    }
}

/// AbortError: отмена через AbortController → FetchError::Aborted.
#[wasm_bindgen_test]
async fn test_classify_abort_error() {
    let controller = web_sys::AbortController::new().unwrap();
    let signal = controller.signal();

    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);
    opts.set_signal(Some(&signal));

    let request = web_sys::Request::new_with_str_and_init(
        "https://jsonplaceholder.typicode.com/posts/1",
        &opts,
    )
    .unwrap();

    controller.abort();

    let window = web_sys::window().unwrap();
    let result = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
    .await;

    assert!(result.is_err());
    let classified = hello_wasm02::fetch::classify_error(result.unwrap_err());
    match classified {
        hello_wasm02::fetch::FetchError::Aborted => {}
        other => panic!("ожидался Aborted, получено {other:?}"),
    }
}

/// Ошибка парсинга → FetchError::Parse.
///
/// Проверяем вручную, потому что `send_request_classified` не парсит тело.
#[wasm_bindgen_test]
async fn test_classify_parse_error() {
    let err = hello_wasm02::fetch::FetchError::Parse("bad json".to_string());
    match err {
        hello_wasm02::fetch::FetchError::Parse(msg) => {
            assert_eq!(msg, "bad json");
        }
        _ => panic!("ожидался Parse"),
    }
}

/// Проверяет текст сообщения для HTTP-ошибки.
#[wasm_bindgen_test]
async fn test_http_error_message() {
    let err = hello_wasm02::fetch::FetchError::Http(500, "".to_string());
    let msg = err.user_message();
    assert!(msg.contains("500"), "сообщение должно содержать статус");
    assert!(err.is_retryable(), "5xx должен быть retryable");

    let err404 = hello_wasm02::fetch::FetchError::Http(404, "".to_string());
    assert!(!err404.is_retryable(), "404 не должен быть retryable");
}

/// Сетевая ошибка: проверяем сообщение и retryable-флаг.
///
/// Важно: сначала проверяем `is_retryable()` по ссылке, **потом**
/// разбираем enum через `match` — иначе `err` окажется частично
/// перемещённым, и обращение к нему не скомпилируется.
#[wasm_bindgen_test]
async fn test_network_error_no_response() {
    let err = hello_wasm02::fetch::FetchError::Network("timeout".to_string());

    // Сначала — то, что берёт &self.
    assert!(err.is_retryable(), "сетевая ошибка должна быть retryable");

    // И только потом — разбор с перемещением владения.
    match err {
        hello_wasm02::fetch::FetchError::Network(msg) => assert_eq!(msg, "timeout"),
        _ => panic!("ожидался Network"),
    }
}

/// Успешный запрос не должен возвращать ошибку.
#[wasm_bindgen_test]
async fn test_ok_response_no_error() {
    let url = "https://jsonplaceholder.typicode.com/posts/1";
    let result = hello_wasm02::fetch::send_request_classified(url).await;
    assert!(result.is_ok(), "успешный запрос не должен возвращать ошибку");
    assert_eq!(result.unwrap().status(), 200);
}
```

## Команды запуска

### Разработка (три терминала)

**Терминал 1 — сервер аутентификации (нужен для секции 4):**

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

Ожидаемый результат: `14 passed; 0 failed` (6 из главы 16.4 + 8 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_errors
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «5. Ошибки» и нажмите каждую кнопку:

| Кнопка | Ожидаемый вывод |
|---|---|
| «Сетевая ошибка» | `FetchError::Network(...)`, «Повторять: true», примечание про CORS |
| «HTTP 404» | `FetchError::Http(404, "<тело>")`, «Повторять: false» |
| «HTTP 500» | `FetchError::Http(500, "<тело>")`, «Повторять: true» |
| «Отменить запрос» | `FetchError::Aborted`, «Повторять: false» |

**Про сообщение в консоли.** При нажатии «Сетевая ошибка» браузер напишет в DevTools:

```
Cross-Origin Request Blocked: The Same Origin Policy disallows reading
the remote resource at https://nonexistent-domain-xyz-42.invalid/api/test.
(Reason: CORS request did not succeed). Status code: (null).
```

Это **нормально**. Браузер не различает три принципиально разные ситуации:

- **DNS не разрешил домен** (наш случай — `.invalid` гарантированно не резолвится);
- **TCP-соединение не установилось** (сервер не слушает порт);
- **CORS-префлайт провалился** или ответ не содержит `Access-Control-Allow-Origin`.

С точки зрения JavaScript-кода все три выглядят одинаково: `fetch` реджектится с `TypeError: Failed to fetch`. Это защита от утечки информации о внутренней сети пользователя: если бы скрипт мог отличить «домен не существует» от «CORS-заголовки не пришли», он получил бы канал для сканирования локальной сети.

Для нашей классификации это неважно: все три ситуации относятся к категории `Network` и считаются retryable. Различить их можно только на стороне сервера (логи) или через DevTools → Network, где видно, был ли вообще установлен TCP-коннект.


## Итог главы

Мы научились различать четыре типа ошибок, которые могут возникнуть при работе с `fetch`:

1. **`FetchError::Network`** — `fetch` не вернул `Response`. Причина — DNS, CORS, обрыв соединения. В JS это `TypeError`. **Повторяем.**
2. **`FetchError::Http(u16, String)`** — запрос дошёл, сервер ответил 4xx или 5xx. **5xx повторяем, 4xx нет.**
3. **`FetchError::Aborted`** — запрос отменён через `AbortController`. В JS это `DOMException` с `name === "AbortError"`. **Игнорируем.**
4. **`FetchError::Parse`** — тело получено, но не разбирается. **Не повторяем, показываем пользователю.**

Ключевые уроки:

- «Сырой» `JsValue` не несёт информации о том, что произошло. `classify_error` превращает его в осмысленный enum.
- `is_retryable()` — не украшение, а рабочий инструмент. Сетевые ошибки и 5xx — временные, их имеет смысл повторить. 4xx и abort — постоянные, повтор только усугубит ситуацию.
- `AbortError` определяется через `Reflect::get(value, "name")` — у `JsValue` нет прямого доступа к свойствам.
- HTTP-статус проверяется через `response.ok()`, а не через ручное сравнение с 200.
- **Тело ошибки — это тоже `async`.** Даже на статусе 500 ответ нужно читать через `.await`. Синхронных «обходных путей» тут нет.
- **Браузер не различает DNS, TCP и CORS.** Все три дают `TypeError: Failed to fetch`. Это защита, а не баг, и её нужно принять как данность.
- **`match` по значению перемещает поля.** Если после `match` вы обращаетесь к тому же значению — матчите по ссылке (`match &err`) или меняйте порядок операций.

В следующей главе — **Retry с экспоненциальной задержкой**, **таймауты через `AbortController` + `setTimeout`**, **Circuit Breaker** и **очередь запросов с ограничением параллелизма**.