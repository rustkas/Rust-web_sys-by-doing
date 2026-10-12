# 16.3. Заголовки: чтение, запись, case-insensitivity

## Обновлённая структура проекта

```text
hello-wasm02/
├── Cargo.toml                      # без изменений (Headers уже подключён в 16.1)
├── Cargo.lock
├── package.json
├── package-lock.json
├── vite.config.js
├── webdriver.json
├── .gitignore
├── LICENSE
├── README.md
├── build.rs
├── .cargo/
│   └── config.toml
├── src/
│   ├── lib.rs
│   ├── app.rs                      # +1 секция: «Заголовки»
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── fetch.rs                    # +5 функций для работы с Headers
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs
│   ├── value.rs
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs
│   ├── wasm_js_objects.rs
│   ├── wasm_jsvalue.rs
│   ├── wasm_callbacks.rs
│   ├── wasm_async.rs
│   ├── wasm_async_utils.rs
│   ├── wasm_navigation.rs
│   ├── wasm_storage.rs
│   └── wasm_fetch.rs               # +6 тестов → 21 тест
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/fetch.rs` — **+5 функций**: `make_headers`, `make_request_with_headers`, `get_response_header`, `has_response_header`, `get_all_response_headers`. Плюс высокоуровневая `get_with_headers`.
- `src/app.rs` — **+1 секция «Заголовки»**. Отправка запроса с кастомными заголовками, чтение заголовков ответа.
- `tests/wasm_fetch.rs` — **+6 тестов**. Всего 21 тест.
- `Cargo.toml` — **без изменений**. Feature `Headers` уже подключён в 16.1.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `Headers` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Headers |
| `Headers.get()` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Headers/get |
| `Headers.set()` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Headers/set |
| `Headers.has()` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Headers/has |
| `Headers.delete()` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Headers/delete |
| HTTP-заголовки (MDN) | https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers |
| Forbidden header names (MDN) | https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name |
| CORS-safelisted response headers | https://developer.mozilla.org/en-US/docs/Glossary/CORS-safelisted_response_header |
| `web_sys::Headers` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.Headers.html |

---

## Что такое HTTP-заголовки

**HTTP-заголовки — это метаданные, которые передаются вместе с запросом и ответом.** Они не являются телом сообщения, но управляют его обработкой: указывают тип содержимого, кеширование, авторизацию, cookies, CORS-правила.

**Заголовки бывают трёх категорий:**

**1. Заголовки запроса.** Устанавливаются клиентом, сообщают серверу о желаниях и контексте:

- `Accept` — какие типы ответа принимаем.
- `Accept-Language` — предпочитаемый язык.
- `Authorization` — токен авторизации.
- `Cache-Control` — правила кеширования.
- `If-None-Match` — условный запрос, для кеша.
- `User-Agent` — информация о клиенте (управляется браузером).
- `Content-Type` — тип тела запроса.

**2. Заголовки ответа.** Устанавливаются сервером, сообщают клиенту о результате:

- `Content-Type` — тип тела ответа.
- `Content-Length` — размер тела в байтах.
- `Cache-Control` — правила кеширования ответа.
- `ETag` — идентификатор версии ресурса.
- `Set-Cookie` — установка cookie.
- `Access-Control-Allow-Origin` — CORS-правила.

**3. Общие заголовки.** Могут быть в обе стороны:

- `Cache-Control`
- `Connection`
- `Date`

---

## Case-insensitivity

**HTTP-заголовки регистронезависимы.** `Content-Type`, `content-type`, `CONTENT-TYPE` — одно и то же. Это правило зафиксировано в [RFC 7230](https://datatracker.ietf.org/doc/html/rfc7230#section-3.2) и реализовано во всех HTTP-библиотеках.

**В `Headers` API браузера это тоже работает автоматически.** Когда вы делаете `headers.get("Content-Type")`, `headers.get("content-type")` или `headers.get("CONTENT-TYPE")` — вы получаете одно и то же значение.

**Что это значит на практике:**

- Не нужно нормализовать имена заголовков перед чтением или записью.
- `headers.has("X-Custom")` и `headers.has("x-custom")` — одинаковы.
- Порядок заголовков при переборе — не гарантирован.

**Одно исключение.** Если один заголовок устанавливается несколько раз с разным регистром — они **объединяются** в один. Например, `set("Accept", "text/html")` и `set("accept", "application/json")` — второй перезапишет первый, потому что это один и тот же заголовок.

---

## Чтение заголовков ответа

**Доступ к заголовкам ответа ограничен CORS-правилами.** Клиент может прочитать только:

1. **CORS-safelisted response headers:**
   - `Cache-Control`
   - `Content-Language`
   - `Content-Length`
   - `Content-Type`
   - `Expires`
   - `Last-Modified`
   - `Pragma`

2. **Заголовки, явно разрешённые через `Access-Control-Expose-Headers`.** Сервер может добавить любые заголовки в этот список, и тогда они станут доступны.

**Что НЕ доступно:**

- **`Set-Cookie`** — никогда не читается клиентом. Cookie доступны через `document.cookie`, но не через `Headers`.
- **Любые заголовки, не входящие в safelisted и не указанные в `Access-Control-Expose-Headers`** — при cross-origin запросе просто отсутствуют в `Headers`.

**Практическое следствие.** Если вы хотите прочитать кастомный заголовок ответа (`X-Request-Id`, `X-RateLimit-Remaining`) — сервер должен явно разрешить его через `Access-Control-Expose-Headers: X-Request-Id, X-RateLimit-Remaining`.

---

## Установка заголовков запроса

**Клиент может установить большинство заголовков запроса, но не все.** Спецификация [Fetch](https://fetch.spec.whatwg.org/#forbidden-header-name) запрещает устанавливать:

- **`Accept-Charset`, `Accept-Encoding`** — браузер сам управляет.
- **`Access-Control-Request-*`** — устанавливаются браузером для preflight.
- **`Connection`, `Content-Length`** — браузер сам знает размер тела.
- **`Cookie`, `Cookie2`** — управляются браузером.
- **`Date`, `DNT`, `Expect`, `Host`** — браузер сам устанавливает.
- **`Origin`** — устанавливается браузером для CORS.
- **`Referer`** — управляется браузером (можно ограничить через `Referrer-Policy`).
- **`User-Agent`** — управляется браузером (можно переопределить с ограничениями).
- **`Sec-*`** — заголовки безопасности, только браузер.

**Что можно установить:**

- `Accept`, `Accept-Language`, `Accept-Encoding` (частично).
- `Authorization`, `Proxy-Authorization`.
- `Cache-Control`, `Pragma`, `Expires`.
- `Content-Type`, `Content-Language`, `Content-Length` (только через тело).
- `If-Match`, `If-None-Match`, `If-Modified-Since`.
- `X-*` — любые кастомные.
- Любые другие, не входящие в forbidden-список.

**Попытка установить forbidden заголовок молча игнорируется.** `Headers::set("Cookie", "...")` не вызовет ошибку, но заголовок не будет установлен.

---

## `src/fetch.rs`

Показаны **только новые части**. Существующие функции из 16.1 и 16.2 остаются.

Добавьте в раздел `use`:

```rust
use js_sys::Array;
use web_sys::Headers;
```

(Если `Headers` уже импортирован — пропустите.)

Добавьте в конец файла:

```rust
// ============================================================
// Headers: создание
// ============================================================

/// Создаёт `Headers` с заданными заголовками.
///
/// Используется для установки общих заголовков на несколько
/// запросов или для тестирования.
pub fn make_headers(headers: &[(&str, &str)]) -> Result<Headers, JsValue> {
    let result = Headers::new()?;
    for (name, value) in headers {
        result.set(name, value)?;
    }
    return Ok(result);
}

// ============================================================
// Запросы с заголовками
// ============================================================

/// Создаёт GET-запрос с кастомными заголовками.
pub fn make_request_with_headers(
    url: &str,
    headers: &[(&str, &str)],
    mode: RequestMode,
) -> Result<Request, JsValue> {
    let opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(mode);

    let request = Request::new_with_str_and_init(url, &opts)?;

    for (name, value) in headers {
        request.headers().set(name, value)?;
    }

    return Ok(request);
}

/// GET-запрос с кастомными заголовками, возвращает `Response`.
pub async fn get_with_headers(
    url: &str,
    headers: &[(&str, &str)],
) -> Result<Response, JsValue> {
    let request =
        make_request_with_headers(url, headers, RequestMode::Cors)?;
    return send_request(&request).await;
}

// ============================================================
// Headers: чтение из ответа
// ============================================================

/// Читает значение заголовка из `Response`.
///
/// Имя заголовка регистронезависимо: `"Content-Type"`,
/// `"content-type"`, `"CONTENT-TYPE"` — одно и то же.
///
/// Возвращает `None`, если заголовка нет или он недоступен
/// из-за CORS-ограничений.
pub fn get_response_header(resp: &Response, name: &str) -> Option<String> {
    return resp.headers().get(name).ok().flatten();
}

/// Проверяет наличие заголовка в `Response`.
///
/// Имя регистронезависимо. Возвращает `false`, если заголовка
/// нет или он недоступен из-за CORS.
pub fn has_response_header(resp: &Response, name: &str) -> bool {
    return resp.headers().has(name).unwrap_or(false);
}

/// Возвращает все доступные заголовки из `Response` как `Vec<(name, value)>`.
///
/// Перебирает `Headers.entries()`. Доступны только те заголовки,
/// которые разрешены CORS-правилами (safelisted + expose).
pub fn get_all_response_headers(resp: &Response) -> Vec<(String, String)> {
    let headers = resp.headers();
    let entries = match headers.entries() {
        Ok(e) => e,
        Err(_) => return Vec::new(),
    };

    let mut result = Vec::new();

    loop {
        let next = match entries.next() {
            Ok(n) => n,
            Err(_) => break,
        };

        if next.done() {
            break;
        }

        let value = next.value();

        // Каждая запись — массив из двух элементов: [name, value]
        if let Ok(arr) = value.dyn_into::<Array>() {
            if arr.length() == 2 {
                let name = arr.get(0).as_string().unwrap_or_default();
                let value = arr.get(1).as_string().unwrap_or_default();
                result.push((name, value));
            }
        }
    }

    return result;
}
```

**Ключевые моменты:**

- **`make_headers`** — создаёт `Headers` из слайса пар. Возвращает `Result`, потому что `Headers::new()` может провалиться (крайне редко).
- **`make_request_with_headers`** — создаёт GET-запрос, добавляет к нему заголовки через `request.headers().set()`. Заголовки устанавливаются **после** создания запроса, но **до** отправки.
- **`get_with_headers`** — высокоуровневая обёртка: создаёт и отправляет запрос с заголовками.
- **`get_response_header`** — читает один заголовок. `.ok().flatten()` превращает `Result<Option<String>, JsValue>` в `Option<String>`.
- **`has_response_header`** — проверяет наличие. `.unwrap_or(false)` для безопасности.
- **`get_all_response_headers`** — перебирает `Headers.entries()`. Каждая запись — `js_sys::Array` из двух элементов. Используется `dyn_into::<Array>()` для приведения.

**Почему `entries()` возвращает `Iterator`, а не `Vec`.** Это стандартный паттерн Web API: итераторы ленивы. Мы перебираем вручную через `.next()`, проверяя `.done()`.

**Почему `Array` из `js_sys`.** Каждая запись `[name, value]` — это JavaScript-массив. Приводим через `dyn_into::<js_sys::Array>()`.

**Почему `.as_string().unwrap_or_default()`.** Если значение не строка (маловероятно для заголовков) — получим пустую строку. Не паникуем.

---

## `src/app.rs`

Показаны **только новые части**. Секции 1 и 2 из 16.1 и 16.2 остаются.

Добавьте **после секции 2** и **перед** `log!("Rust API Client — часть 2 готова")`:

```rust
       // ============================================================
    // СЕКЦИЯ 3: Заголовки
    // ============================================================

    let section3 = build_section(&container, "3. Заголовки")?;

    let intro3 = dom::create_paragraph(
        "Заголовки HTTP-запроса и ответа. \
         Отправляем запрос с кастомными заголовками, \
         читаем заголовки ответа. \
         Имена заголовков регистронезависимы.",
    )?;
    dom::add_class(&dom::as_html(&intro3), "hint")?;
    dom::append(&section3, &intro3)?;

    let send_with_headers_btn = dom::create_element("button")?;
    send_with_headers_btn.set_text_content(Some("Запрос с заголовками"));
    let read_headers_btn = dom::create_element("button")?;
    read_headers_btn.set_text_content(Some("Прочитать все заголовки"));

    dom::append(&section3, &send_with_headers_btn)?;
    dom::append(&section3, &read_headers_btn)?;

    let result3 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result3), "api-result")?;
    result3.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section3, &result3)?;

    // Хранилище для последнего Response — чтобы прочитать его заголовки
    // в другом обработчике.
    let last_response: Rc<RefCell<Option<web_sys::Response>>> =
        Rc::new(RefCell::new(None));

    // ---- Обработчик «Запрос с заголовками» ----
    {
        let result3 = result3.clone();
        let last_response = last_response.clone();

        callbacks::listen(
            &send_with_headers_btn,
            "click",
            move |_event| {
                let result3 = result3.clone();
                let last_response = last_response.clone();
                result3.set_text_content(Some("Отправка..."));

                wasm_bindgen_futures::spawn_local(async move {
                    let headers = [
                        ("Accept", "application/json"),
                        ("Accept-Language", "ru,en;q=0.9"),
                        ("X-Custom-Header", "Rust-Wasm-Client"),
                    ];

                    // Result. Явный тип — чтобы компилятор не путался.
                    let response_result: Result<web_sys::Response, JsValue> =
                        fetch::get_with_headers(HEADERS_URL, &headers).await;

                    // Разбираем Result один раз, привязываем к НОВОМУ имени.
                    // http_response — уникальное имя, нигде больше не используется.
                    let http_response: web_sys::Response = match response_result {
                        Ok(r) => r,
                        Err(e) => {
                            let error_text: String = format!("❌ Ошибка: {:?}", e);
                            result3.set_text_content(Some(error_text.as_str()));
                            log!("Headers request error: {:?}", e);
                            return;
                        }
                    };

                    // ВАЖНО: Response::clone() возвращает Result<Response, JsValue>,
                    // а не Response. Это собственный метод web-sys (обёртка над
                    // Response.prototype.clone()), и он может бросить исключение,
                    // если тело уже прочитано. Трейт Clone здесь ни при чём.
                    let http_response_clone: web_sys::Response =
                        match http_response.clone() {
                            Ok(r) => r,
                            Err(e) => {
                                let error_text: String =
                                    format!("❌ Ошибка клонирования Response: {:?}", e);
                                result3.set_text_content(Some(error_text.as_str()));
                                log!("Response clone error: {:?}", e);
                                return;
                            }
                        };

                    *last_response.borrow_mut() = Some(http_response_clone);

                    let status: u16 = fetch::response_status(&http_response);

                    let content_type: String =
                        fetch::get_response_header(&http_response, "content-type")
                            .unwrap_or_else(|| "(not available)".into());

                    let cache_control: String =
                        fetch::get_response_header(&http_response, "cache-control")
                            .unwrap_or_else(|| "(not available)".into());

                    let text: String = format!(
                        "✅ GET с заголовками\n\
                         \n\
                         Отправлены:\n\
                           Accept: application/json\n\
                           Accept-Language: ru,en;q=0.9\n\
                           X-Custom-Header: Rust-Wasm-Client\n\
                         \n\
                         Получен ответ:\n\
                           Status: {}\n\
                           Content-Type: {}\n\
                           Cache-Control: {}\n\
                         \n\
                         Нажмите «Прочитать все заголовки» \
                         для полного списка.",
                        status, content_type, cache_control
                    );
                    result3.set_text_content(Some(text.as_str()));
                    log!("Headers request: status={}", status);
                });
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Прочитать все заголовки» ----
    {
        let result3 = result3.clone();
        let last_response = last_response.clone();

        callbacks::listen(
            &read_headers_btn,
            "click",
            move |_event| {
                // Клонируем Option<Response> из RefCell, чтобы не держать
                // RefCell::borrow() через весь обработчик.
                let resp_opt: Option<web_sys::Response> = last_response.borrow().clone();

                let response: web_sys::Response = match resp_opt {
                    Some(r) => r,
                    None => {
                        result3.set_text_content(Some(
                            "Сначала нажмите «Запрос с заголовками»",
                        ));
                        return;
                    }
                };

                let headers: Vec<(String, String)> =
                    fetch::get_all_response_headers(&response);

                let mut lines: Vec<String> = Vec::new();
                lines.push(format!(
                    "Все доступные заголовки ответа ({}):",
                    headers.len()
                ));
                lines.push(String::new());

                if headers.is_empty() {
                    lines.push("(заголовки недоступны или пусты)".to_string());
                } else {
                    for (name, value) in &headers {
                        // Отдельная функция — обходит проблему rust-analyzer
                        // с if/else в tail-позиции внутри замыкания.
                        let preview: String = truncate_preview(value, 60);
                        lines.push(format!("{}: {}", name, preview));
                    }
                }

                lines.push(String::new());
                lines.push(
                    "Заметьте: Set-Cookie никогда не появляется здесь. \
                     Cookie доступны только через document.cookie."
                        .to_string(),
                );

                let text: String = lines.join("\n");
                result3.set_text_content(Some(text.as_str()));

                log!("Read all headers: {} entries", headers.len());
                // Точка с запятой обязательна — иначе макрос log!
                // становится tail-выражением и замыкание вернёт (),
                // что вызовет ошибку «expected String, found ()».
            },
            &closures,
        )?;
    }
```

Добавьте **новую константу** рядом с другими URL:

```rust
const HEADERS_URL: &str = "https://jsonplaceholder.typicode.com/posts/1";
```



Добавьте в раздел вспомогательных функций (рядом с build_section):
```rust

/// Обрезает длинное значение для предпросмотра.
///
/// Вынесено в отдельную функцию, чтобы избежать проблемы
/// `rust-analyzer` с tail-выражениями `if/else`.
/// Использует `chars()`, а не байтовый срез `[..n]`, потому
/// что срез по байтам паникует на границе UTF-8 символа.
fn truncate_preview(value: &str, max_chars: usize) -> String {
    let char_count: usize = value.chars().count();
    if char_count > max_chars {
        let truncated: String = value.chars().take(max_chars).collect();
        return format!("{}...", truncated);
    }
    return value.to_string();
}
```


**Что демонстрирует секция 3:**

- **«Запрос с заголовками»** — отправляет GET-запрос с тремя кастомными заголовками (`Accept`, `Accept-Language`, `X-Custom-Header`). Показывает статус и два заголовка ответа (`Content-Type`, `Cache-Control`), прочитанных через `get_response_header`.
- **«Прочитать все заголовки»** — перебирает все доступные заголовки ответа через `get_all_response_headers`. Показывает их список. Подсказка про `Set-Cookie`.

**Почему `Response` сохраняется в `Rc<RefCell<Option<Response>>>`.** Заголовки читаются **после** отправки запроса, в другом обработчике. `Response` — не `Copy`, его нужно сохранить. `Rc<RefCell<Option<...>>>` — стандартный паттерн.

**Почему `.clone()` для `Response`.** `web_sys::Response` клонируется дёшево — создаётся новая JS-ссылка на тот же объект. Без клонирования мы бы переместили `resp` в `last_response` и не смогли бы прочитать его заголовки в том же обработчике.

**Почему `value[..60]` может паниковать.** Для Unicode-строк `value[..60]` — байтовый срез. Если 60-й байт попадает в середину многобайтового символа — panic. Для HTTP-заголовков это маловероятно (они обычно ASCII), но для надёжности можно использовать `chars().take(60).collect()`. В учебном коде байтовый срез допустим.

---

## `web/styles/main.css`

Изменений **нет**. `.api-result` из 16.1 работает и для секции 3.

---

## `tests/wasm_fetch.rs`

Добавьте в конец файла:

```rust
// ============================================================
// Headers: создание и установка
// ============================================================

#[wasm_bindgen_test]
fn test_make_headers() {
    let headers = fetch::make_headers(&[
        ("Accept", "application/json"),
        ("X-Custom", "test-value"),
    ]);

    assert!(headers.is_ok(), "Headers should be created");
    let h = headers.unwrap();

    let accept = h.get("Accept").unwrap().unwrap();
    assert_eq!(accept, "application/json");

    let custom = h.get("X-Custom").unwrap().unwrap();
    assert_eq!(custom, "test-value");
}

#[wasm_bindgen_test]
fn test_headers_case_insensitive() {
    let headers = fetch::make_headers(&[
        ("Content-Type", "application/json"),
    ])
    .unwrap();

    // Разные регистры — один и тот же заголовок
    assert_eq!(
        headers.get("Content-Type").unwrap().unwrap(),
        "application/json"
    );
    assert_eq!(
        headers.get("content-type").unwrap().unwrap(),
        "application/json"
    );
    assert_eq!(
        headers.get("CONTENT-TYPE").unwrap().unwrap(),
        "application/json"
    );
    assert_eq!(
        headers.get("CoNtEnT-TyPe").unwrap().unwrap(),
        "application/json"
    );
}

// ============================================================
// Headers: реальные запросы
// ============================================================

#[wasm_bindgen_test]
async fn test_request_with_custom_headers() {
    let result = fetch::get_with_headers(
        "https://jsonplaceholder.typicode.com/posts/1",
        &[
            ("Accept", "application/json"),
            ("X-Custom-Header", "test-value"),
        ],
    )
    .await;

    assert!(result.is_ok(), "GET with headers should succeed");
    let resp = result.unwrap();
    assert!(fetch::response_is_ok(&resp));
    assert_eq!(fetch::response_status(&resp), 200);
}

#[wasm_bindgen_test]
async fn test_response_headers_content_type() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .expect("GET should succeed");

    let content_type = fetch::get_response_header(&resp, "content-type");
    assert!(content_type.is_some(), "Content-Type should be present");
    assert!(
        content_type.unwrap().contains("application/json"),
        "Content-Type should be application/json"
    );

    // Case-insensitivity при чтении
    let ct_upper = fetch::get_response_header(&resp, "CONTENT-TYPE");
    assert!(ct_upper.is_some(), "CONTENT-TYPE should also work");
}

#[wasm_bindgen_test]
async fn test_response_headers_cache_control() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .expect("GET should succeed");

    // Cache-Control — CORS-safelisted, должен быть доступен
    let cache_control = fetch::get_response_header(&resp, "cache-control");
    assert!(
        cache_control.is_some(),
        "Cache-Control should be present for CORS request"
    );
}

#[wasm_bindgen_test]
async fn test_response_headers_has() {
    let resp = fetch::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .expect("GET should succeed");

    assert!(
        fetch::has_response_header(&resp, "content-type"),
        "Content-Type should exist"
    );
    assert!(
        fetch::has_response_header(&resp, "CONTENT-TYPE"),
        "Case-insensitive check should also work"
    );
    assert!(
        !fetch::has_response_header(&resp, "x-nonexistent-header"),
        "Non-existent header should return false"
    );
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_make_headers` | Создание `Headers` из пар |
| `test_headers_case_insensitive` | Чтение `Content-Type` в разных регистрах |
| `test_request_with_custom_headers` | Отправка GET с кастомными заголовками |
| `test_response_headers_content_type` | Чтение `Content-Type` из ответа, case-insensitivity |
| `test_response_headers_cache_control` | Чтение `Cache-Control` (CORS-safelisted) |
| `test_response_headers_has` | `has_response_header` для существующих и отсутствующих |

**Всего тестов:** 15 из 16.1–16.2 + 6 новых = **21 тест**.

**Почему `test_headers_case_insensitive` с четырьмя регистрами.** Один тест проверяет все основные варианты: канонический, нижний, верхний, смешанный. Если хотя бы один не сработает — тест упадёт.

**Почему `test_response_headers_cache_control` специфичен.** JSONPlaceholder возвращает `Cache-Control` (CORS-safelisted заголовок). Если сервер не вернёт его — тест упадёт. Это проверяет, что наш `get_response_header` работает с реальными safelisted заголовками.

---

## Команды запуска

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

Ожидаемый результат — без ошибок и предупреждений.

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Прокрутите до секции 3 «Заголовки».

**«Запрос с заголовками»:**
1. Нажмите кнопку. Отправится GET-запрос с `Accept`, `Accept-Language`, `X-Custom-Header`.
2. Увидите статус 200, `Content-Type: application/json` и `Cache-Control`.

**«Прочитать все заголовки»:**
1. Нажмите кнопку. Появится список всех доступных заголовков ответа.
2. В списке не будет `Set-Cookie` — подсказка объясняет почему.

### Тесты

```bash
# Только тесты Fetch в Chrome
wasm-pack test --headless --chrome --test wasm_fetch
```

```bash
# Только тесты Fetch в Firefox
xvfb-run -a wasm-pack test --headless --firefox \
  --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver \
  --test wasm_fetch
```

```bash
# Все тесты проекта
npm run test:all
```

**Ожидаемый результат:** 21 тест `ok` в обоих браузерах.

---

## Что важно запомнить

**1. HTTP-заголовки регистронезависимы.** `Content-Type`, `content-type`, `CONTENT-TYPE` — одно и то же. Браузер нормализует имена автоматически.

**2. `Headers` API — это интерфейс к заголовкам.** `set`, `get`, `has`, `delete`, `append`. Работает и для запроса, и для ответа.

**3. `headers.get(name)` возвращает `Option<String>`.** `None` означает «заголовка нет» или «заголовок недоступен из-за CORS».

**4. `headers.entries()` возвращает `Iterator`.** Каждая запись — массив `[name, value]`. Перебирается через `.next()` с проверкой `.done()`.

**5. CORS-safelisted заголовки доступны всегда.** `Content-Type`, `Content-Length`, `Cache-Control`, `Expires`, `Last-Modified`, `Pragma`, `Content-Language`. Остальные — только через `Access-Control-Expose-Headers`.

**6. `Set-Cookie` никогда не читается через `Headers`.** Cookie управляются отдельно — через `document.cookie` на клиенте и `Set-Cookie` в HTTP-ответе.

**7. Forbidden headers нельзя установить.** `Cookie`, `Host`, `Content-Length`, `Origin`, `Referer`, `User-Agent`, `Sec-*` — управляются браузером. Попытка установить молча игнорируется.

**8. `Accept`, `Authorization`, `X-*` — устанавливаются свободно.** Это основные заголовки, которые вы будете устанавливать вручную.

**9. Заголовки устанавливаются после создания `Request`.** Через `request.headers().set(name, value)`. Или передаются в `RequestInit` — там тоже есть `headers()`.

**10. `Rc<RefCell<Option<Response>>>` для сохранения между обработчиками.** `Response` не `Copy`, но клонируется дёшево. Клонирование создаёт новую ссылку на тот же объект.

---

## Что дальше

**16.4. Аутентификация и авторизация.** Мы разберём `Authorization: Bearer <token>`, обработку 401 и 403, обновление токена. В `fetch.rs` появятся функции `make_authenticated_request`, `with_bearer_token`, `send_with_auth_retry`. В `app.rs` — секция «Аутентификация» с эмуляцией входа, работы с токеном и автоматического обновления при 401. В тестах — 6 новых проверок на работу с токеном.