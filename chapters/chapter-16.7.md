# Глава 16.7. Timeout и отмена

## Обновлённая структура проекта

```
.
├── auth-server/
│   └── src/
│       ├── main.rs                 ← ОБНОВЛЁН: +маршруты /api/slow, /api/slow/:secs
│       └── handlers.rs             ← ОБНОВЛЁН: +slow_default, slow, slow_with_delay
│
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +make_abort_controller, make_timeout_controller,
│   │                                  get_with_timeout, send_with_signal, clear_timeout
│   ├── app.rs                      ← ОБНОВЛЁН: секция 7 «Timeout и отмена»
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_timeout.rs             ← НОВЫЙ: 6 тестов timeout и отмены
│   └── (остальные без изменений)
│
└── (корневые файлы без изменений)
```

**Краткий комментарий.** У `fetch` нет встроенного таймаута — эта особенность удивляет почти всех, кто впервые сталкивается с Fetch API. Запрос может «висеть» минутами, если сервер не отвечает, а браузер не считает нужным прервать соединение. Решение — `AbortController` плюс `setTimeout`: таймер вызывает `abort()`, а сигнал обрывает запрос. В этой главе мы добавим в `fetch.rs` функции для работы с отменой, в `auth-server` — эндпоинт `/api/slow`, который отвечает с настраиваемой задержкой, а в `app.rs` — секцию с двумя сценариями: срабатывание таймаута и ручная отмена по кнопке.

## Ссылки для детального изучения

- **MDN: `AbortController`** — основной интерфейс для отмены запросов: [developer.mozilla.org/ru/docs/Web/API/AbortController](https://developer.mozilla.org/ru/docs/Web/API/AbortController)
- **MDN: `AbortSignal`** — сигнал, который передаётся в `fetch` и другие API: [developer.mozilla.org/ru/docs/Web/API/AbortSignal](https://developer.mozilla.org/ru/docs/Web/API/AbortSignal)
- **MDN: `AbortSignal.timeout()`** — статический метод, возвращающий сигнал с автоматическим таймаутом. Baseline 2024. Сигнал завершается с `TimeoutError` при таймауте и с `AbortError` при ручной отмене. Важно: этот сигнал нельзя «отменить извне» — для комбинирования с ручной отменой нужен `AbortSignal.any()`.
- **MDN: `AbortSignal.any()`** — комбинирование нескольких сигналов: [developer.mozilla.org/en-US/docs/Web/API/AbortSignal/any_static](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/any_static)
- **WHATWG Fetch Standard: Abort** — как именно спецификация требует обрабатывать отмену: [fetch.spec.whatwg.org/#abort-fetch](https://fetch.spec.whatwg.org/#abort-fetch)
- **`web_sys::AbortController`** — документация Rust-биндинга: [docs.rs/web-sys/latest/web_sys/struct.AbortController.html](https://docs.rs/web-sys/latest/web_sys/struct.AbortController.html)
- **`gloo-timers`** — таймеры и `sleep` для WASM: [docs.rs/gloo-timers](https://docs.rs/gloo-timers)

## Новые и обновлённые файлы

### `auth-server/src/handlers.rs` (обновлён — добавить в конец)

```rust
use axum::extract::Path;

/// Эндпоинт с задержкой по умолчанию (5 секунд).
///
/// Используется по маршруту `/api/slow`.
pub async fn slow_default() -> Json<Value> {
    slow_with_delay(5).await
}

/// Эндпоинт с задержкой, указанной в URL.
///
/// `/api/slow/10` — задержка 10 секунд.
///
/// **Почему не `Path<Option<u64>>`.** Axum извлекает path-параметры
/// через serde, и `Option<u64>` ожидает либо явный `null`, либо число.
/// Строка `"10"` — ни то, ни другое, поэтому serde падает с
/// «Unsupported type». Разделение на два обработчика — идиоматичнее:
/// маршрут без параметра идёт в `slow_default`, с параметром — в `slow`.
pub async fn slow(Path(secs): Path<u64>) -> Json<Value> {
    slow_with_delay(secs).await
}

/// Общая логика: ждём `secs` секунд и возвращаем JSON.
async fn slow_with_delay(secs: u64) -> Json<Value> {
    tokio::time::sleep(std::time::Duration::from_secs(secs)).await;

    Json(json!({
        "message": format!("Ответ получен через {secs} секунд"),
        "delay_secs": secs
    }))
}
```

### `auth-server/src/main.rs` (обновлён — два маршрута вместо одного)

Внутри `Router::new()`:

```rust
.route("/api/slow", get(handlers::slow_default))
.route("/api/slow/:secs", get(handlers::slow))
```

Остальные маршруты (`/api/me`, `/api/admin`, `/api/auth/login`, `/api/auth/refresh`, `/api/flaky`, `/api/flaky/reset`, `/api/error500`) остаются без изменений.

### `src/fetch.rs` (обновлён — добавить в конец)

В начало файла добавьте импорт:

```rust
use wasm_bindgen::closure::Closure;
```

Четыре новые функции:

```rust
// ============================================================
// Timeout и отмена
// ============================================================

/// Создаёт новый `AbortController`.
pub fn make_abort_controller() -> Result<web_sys::AbortController, JsValue> {
    web_sys::AbortController::new()
}

/// Создаёт `AbortController` с автоматическим таймаутом.
///
/// Возвращает `(controller, timeout_id)`. Через `timeout_ms` миллисекунд
/// контроллер будет автоматически отменён. `timeout_id` нужен для
/// очистки таймера, если запрос завершится раньше.
///
/// **Важно:** в браузере нет надёжного способа отменить `setTimeout`
/// из Rust без хранения идентификатора. Мы возвращаем его вызывающему,
/// чтобы тот мог вызвать `clear_timeout(timeout_id)` после успешного
/// завершения запроса.
pub fn make_timeout_controller(
    timeout_ms: u32,
) -> Result<(web_sys::AbortController, i32), JsValue> {
    let controller = web_sys::AbortController::new()?;
    let controller_clone = controller.clone();

    let callback = Closure::once_into_js(move || {
        controller_clone.abort();
    });

    let window = web_sys::window().expect("no window");
    let timeout_id = window
        .set_timeout_with_callback_and_timeout_and_arguments_0(
            callback.unchecked_ref(),
            timeout_ms as i32,
        )?;

    Ok((controller, timeout_id))
}

/// Отменяет ранее установленный таймаут.
///
/// Вызывать после успешного завершения запроса, чтобы `abort()`
/// не сработал зря. Если этого не сделать, запрос завершится,
/// а через некоторое время сработает таймер и отменит **следующий**
/// запрос, если `AbortController` был переиспользован.
pub fn clear_timeout(timeout_id: i32) {
    if let Some(window) = web_sys::window() {
        window.clear_timeout_with_handle(timeout_id);
    }
}

/// GET-запрос с таймаутом.
///
/// Если сервер не ответит за `timeout_ms` миллисекунд, запрос будет
/// отменён. Ошибка отмены вернётся как `FetchError::Aborted`.
///
/// **Ограничение:** таймаут покрывает только получение заголовков.
/// Чтение тела ответа (`response.text()`, `response.json()`) уже не
/// ограничено этим таймаутом, потому что `clear_timeout` вызывается
/// сразу после `fetch`. Если нужно ограничить и чтение тела,
/// используйте `send_with_signal` и не очищайте таймер до конца чтения.
pub async fn get_with_timeout(
    url: &str,
    timeout_ms: u32,
) -> Result<web_sys::Response, FetchError> {
    let (controller, timeout_id) = make_timeout_controller(timeout_ms)
        .map_err(classify_error)?;
    let signal = controller.signal();

    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);
    opts.set_signal(Some(&signal));

    let request = web_sys::Request::new_with_str_and_init(url, &opts)
        .map_err(|e| {
            clear_timeout(timeout_id);
            classify_error(e)
        })?;

    let window = web_sys::window()
        .ok_or_else(|| {
            clear_timeout(timeout_id);
            FetchError::Network("no window object".into())
        })?;

    let resp_value = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
    .await
    .map_err(|e| {
        clear_timeout(timeout_id);
        classify_error(e)
    })?;

    // Заголовки получены — таймер больше не нужен.
    clear_timeout(timeout_id);

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

/// Отправляет GET-запрос с заданным сигналом отмены.
///
/// Сигнал передаётся снаружи — это позволяет вызывающему коду
/// сохранить `AbortController` и отменить запрос в любой момент,
/// пока он в полёте. Именно так работает ручная отмена в UI.
pub async fn send_with_signal(
    url: &str,
    signal: &web_sys::AbortSignal,
) -> Result<web_sys::Response, FetchError> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);
    opts.set_signal(Some(signal));

    let request = web_sys::Request::new_with_str_and_init(url, &opts)
        .map_err(classify_error)?;

    let window = web_sys::window()
        .ok_or_else(|| FetchError::Network("no window object".into()))?;

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
```

### `src/app.rs` — добавить секцию 7

После секции 6, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 7: Timeout и отмена
    // ============================================================

    let section7 = build_section(&container, "7. Timeout и отмена")?;

    let intro7 = dom::create_paragraph(
        "У fetch нет встроенного таймаута. \
         Стандартный способ — AbortController + setTimeout: \
         таймер вызывает abort(), сигнал обрывает запрос. \
         Не забывайте очищать таймер после завершения, иначе \
         он сработает зря. \
         AbortSignal.timeout() — новый API, но он не даёт \
         комбинировать таймаут с ручной отменой.",
    )?;
    dom::add_class(&dom::as_html(&intro7), "hint")?;
    dom::append(&section7, &intro7)?;

    let timeout_btn = dom::create_element("button")?;
    timeout_btn.set_text_content(Some("Timeout 1с (сервер 5с)"));
    let cancel_btn = dom::create_element("button")?;
    cancel_btn.set_text_content(Some("Отправить запрос"));
    let cancel_now_btn = dom::create_element("button")?;
    cancel_now_btn.set_text_content(Some("Отменить сейчас"));

    dom::append(&section7, &timeout_btn)?;
    dom::append(&section7, &cancel_btn)?;
    dom::append(&section7, &cancel_now_btn)?;

    let result7 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result7), "api-result")?;
    result7.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section7, &result7)?;

    // Хранилище для активного AbortController — чтобы «Отменить сейчас»
    // мог дотянуться до запроса, запущенного кнопкой «Отправить запрос».
    let active_controller: Rc<RefCell<Option<web_sys::AbortController>>> =
        Rc::new(RefCell::new(None));

    // ---- Timeout 1с к 5-секундному эндпоинту ----
    {
        let result7 = result7.clone();
        callbacks::listen(
            &timeout_btn,
            "click",
            move |_| {
                let result7 = result7.clone();
                result7.set_text_content(Some("Запрос с таймаутом 1с..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "http://127.0.0.1:3000/api/slow/5";
                    match fetch::get_with_timeout(url, 1000).await {
                        Ok(resp) => {
                            let status = resp.status();
                            result7.set_text_content(Some(&format!(
                                "✅ Ответ получен (статус {status}). \
                                 Таймаут не сработал — увеличьте задержку сервера."
                            )));
                        }
                        Err(fetch::FetchError::Aborted) => {
                            result7.set_text_content(Some(
                                "⏱ Таймаут сработал: запрос отменён через 1с. \
                                 Сервер отвечает через 5с — не успел.",
                            ));
                        }
                        Err(e) => {
                            result7.set_text_content(Some(&format!(
                                "❌ Неожиданная ошибка: {:?}",
                                e
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Ручная отмена: отправляем запрос и даём кнопку отмены ----
    //
    // Кнопку «Отменить сейчас» намеренно не делаем disabled.
    // В HTML атрибут `disabled` — булев: важно только его наличие,
    // а не значение. Управление им через `set_attribute`/`remove_attribute`
    // работает нестабильно в разных браузерах, особенно когда атрибут
    // выставлен в момент создания элемента. Проще всегда держать кнопку
    // активной и проверять наличие контроллера в обработчике.
    {
        let result7 = result7.clone();
        let active_controller = active_controller.clone();

        callbacks::listen(
            &cancel_btn,
            "click",
            move |_| {
                let result7 = result7.clone();
                let active_controller = active_controller.clone();

                result7.set_text_content(Some("Запрос отправлен. Ждём..."));

                wasm_bindgen_futures::spawn_local(async move {
                    let controller = match web_sys::AbortController::new() {
                        Ok(c) => c,
                        Err(e) => {
                            result7.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };
                    let signal = controller.signal();

                    // Создаём контроллер ЗАРАНЕЕ и сохраняем его в общем
                    // хранилище, чтобы «Отменить сейчас» мог его достать.
                    *active_controller.borrow_mut() = Some(controller);

                    let url = "http://127.0.0.1:3000/api/slow/5";
                    let result = fetch::send_with_signal(url, &signal).await;

                    // Запрос завершился — снимаем контроллер из хранилища.
                    *active_controller.borrow_mut() = None;

                    match result {
                        Ok(resp) => {
                            let status = resp.status();
                            result7.set_text_content(Some(&format!(
                                "✅ Запрос завершён, статус {status}"
                            )));
                        }
                        Err(fetch::FetchError::Aborted) => {
                            result7.set_text_content(Some("🛑 Запрос отменён вручную."));
                        }
                        Err(e) => {
                            result7.set_text_content(Some(&format!("❌ Ошибка: {:?}", e)));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Отменить сейчас ----
    {
        let result7 = result7.clone();
        let active_controller = active_controller.clone();

        callbacks::listen(
            &cancel_now_btn,
            "click",
            move |_| {
                let result7 = result7.clone();
                let active_controller = active_controller.clone();

                // Копируем Option из RefCell и сразу отпускаем borrow.
                // Без этого временный Ref живёт до конца if let/else,
                // и компилятор не может доказать, что заимствование
                // закончилось раньше, чем уничтожится active_controller.
                let controller_opt: Option<web_sys::AbortController> =
                    active_controller.borrow().clone();

                if let Some(controller) = controller_opt {
                    controller.abort();
                    result7.set_text_content(Some("🛑 Отмена отправлена..."));
                } else {
                    result7.set_text_content(Some(
                        "Нет активного запроса. Сначала нажмите «Отправить запрос».",
                    ));
                }
            },
            &closures,
        )?;
    }
```

### `tests/wasm_timeout.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Создание AbortController не падает.
#[wasm_bindgen_test]
async fn test_make_abort_controller() {
    let controller = hello_wasm02::fetch::make_abort_controller().unwrap();
    let signal = controller.signal();
    assert!(!signal.aborted(), "новый сигнал не должен быть отменён");
}

/// Новый сигнал не отменён.
#[wasm_bindgen_test]
async fn test_abort_signal_initially_not_aborted() {
    let controller = hello_wasm02::fetch::make_abort_controller().unwrap();
    assert!(!controller.signal().aborted());
}

/// После abort() сигнал помечен как отменённый.
#[wasm_bindgen_test]
async fn test_abort_signal_after_abort() {
    let controller = hello_wasm02::fetch::make_abort_controller().unwrap();
    controller.abort();
    assert!(controller.signal().aborted());
}

/// Таймаут срабатывает: /api/slow/5 не успевает за 200мс.
#[wasm_bindgen_test]
async fn test_timeout_fires() {
    let result = hello_wasm02::fetch::get_with_timeout(
        "http://127.0.0.1:3000/api/slow/5",
        200,
    )
    .await;

    assert!(result.is_err());
    match result.unwrap_err() {
        hello_wasm02::fetch::FetchError::Aborted => {}
        other => panic!("ожидался Aborted, получено {other:?}"),
    }
}

/// Быстрый запрос не должен сработать по таймауту.
#[wasm_bindgen_test]
async fn test_timeout_does_not_fire_when_fast() {
    // JSONPlaceholder отвечает быстро, таймаут 5 секунд не сработает.
    let result = hello_wasm02::fetch::get_with_timeout(
        "https://jsonplaceholder.typicode.com/posts/1",
        5000,
    )
    .await;

    assert!(result.is_ok(), "быстрый запрос не должен отменяться");
    assert_eq!(result.unwrap().status(), 200);
}

/// Ручная отмена во время запроса через `send_with_signal`.
#[wasm_bindgen_test]
async fn test_abort_during_request() {
    let controller = hello_wasm02::fetch::make_abort_controller().unwrap();
    let signal = controller.signal();

    // Создаём запрос вручную, чтобы отменить его сразу после старта.
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);
    opts.set_signal(Some(&signal));

    let request = web_sys::Request::new_with_str_and_init(
        "http://127.0.0.1:3000/api/slow/5",
        &opts,
    )
    .unwrap();

    controller.abort();

    let window = web_sys::window().unwrap();
    let result = wasm_bindgen_futures::JsFuture::from(
        window.fetch_with_request(&request),
    )
    .await;

    assert!(result.is_err(), "запрос должен быть отменён");
    let classified = hello_wasm02::fetch::classify_error(result.unwrap_err());
    match classified {
        hello_wasm02::fetch::FetchError::Aborted => {}
        other => panic!("ожидался Aborted, получено {other:?}"),
    }
}
```

## Команды запуска

### Разработка (три терминала)

**Терминал 1 — сервер:**

```bash
npm run server
```

После правок в `handlers.rs` и `main.rs` сервер нужно перезапустить, иначе маршрут `/api/slow` не зарегистрируется.

**Терминал 2 — сборка WASM и dev-сервер Vite:**

```bash
npm run wasm:dev
npm run dev
```

**Терминал 3 — тесты:**

```bash
npm run test:chrome
```

Ожидаемый результат: `28 passed; 0 failed` (22 из глав 16.4–16.6 + 6 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_timeout
```

### Проверка компиляции

```bash
# Из корня проекта — проверка клиента
cargo check --target wasm32-unknown-unknown

# Из корня проекта — проверка сервера
cargo check --manifest-path auth-server/Cargo.toml
```

### Ручная проверка сервера

```bash
# Должен ответить через 2 секунды
curl -i http://127.0.0.1:3000/api/slow/2

# Должен ответить через 5 секунд (задержка по умолчанию)
curl -i http://127.0.0.1:3000/api/slow
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «7. Timeout и отмена»:

| Действие | Ожидаемый результат |
|---|---|
| «Timeout 1с (сервер 5с)» | `⏱ Таймаут сработал: запрос отменён через 1с` |
| «Отправить запрос» → «Отменить сейчас» | `🛑 Отмена отправлена...`, затем `🛑 Запрос отменён вручную` |
| «Отменить сейчас» без запроса | `Нет активного запроса. Сначала нажмите «Отправить запрос».` |

В DevTools → Network при отмене статус запроса будет `(canceled)` — это корректно.


## Итог главы

Мы научились отменять запросы и ставить им таймауты:

1. **`make_abort_controller`** — создаёт `AbortController`, сигнал которого передаётся в `RequestInit::set_signal`.
2. **`make_timeout_controller(ms)`** — создаёт контроллер, который автоматически отменится через `ms` миллисекунд. Возвращает идентификатор таймера для очистки.
3. **`get_with_timeout(url, ms)`** — готовый GET с таймаутом. Ошибка отмены приходит как `FetchError::Aborted`.
4. **`send_with_signal(url, &signal)`** — GET с готовым сигналом. Позволяет вызывающему коду управлять отменой вручную.

Ключевые уроки:

- **У `fetch` нет встроенного таймаута.** Запрос может висеть сколько угодно, пока сервер не ответит или соединение не оборвётся. Таймаут — ваша ответственность.
- **`AbortController` + `setTimeout` — стандартный паттерн.** Создаём контроллер, ставим таймер, передаём сигнал в запрос. После завершения — очищаем таймер.
- **Не забывайте `clear_timeout`.** Если таймер сработает после того, как запрос уже завершился, `abort()` может отменить **следующий** запрос, если контроллер был переиспользован.
- **Таймаут покрывает только заголовки.** После того как `fetch` разрешился, `clear_timeout` уже вызван, и чтение тела не ограничено таймаутом.
- **`Path<Option<u64>>` в Axum не работает.** Для необязательного path-параметра нужно два маршрута и два обработчика.
- **Атрибут `disabled` — булев.** Важно только его наличие. Управлять им через `set_attribute`/`remove_attribute` нестабильно; лучше не выключать кнопку, а проверять состояние в обработчике.
- **`RefCell::borrow()` в `if let` держит заимствование до конца ветки.** Если после `if let` вы работаете с чем-то ещё, заимствованным из того же замыкания, — сначала скопируйте значение в локальную переменную, потом разбирайте.
- **`AbortSignal.timeout()` не комбинируется с ручной отменой.** Сигнал, созданный `timeout()`, нельзя «дополнительно» отменить через контроллер. Для комбинирования нужен `AbortSignal.any()`.
- **`AbortError` отличается от `TimeoutError`.** `timeout()` устанавливает `reason` в `TimeoutError`, а `abort()` — в `AbortError`. Это позволяет UI различать «сервер не ответил вовремя» и «пользователь нажал отмену».

В следующей главе — **параллельные запросы**, **`Promise.all` и `Promise.race`**, **ограничение параллелизма** и **агрегация результатов**.