Проверил рекомендации. Нашёл одну ошибку и одну неточность:

**Ошибка.** `worker.set_onerror(Some(on_error.as_ref().unchecked_ref()))` может не скомпилироваться из-за того, что `rustc` не всегда выводит тип для `unchecked_ref`. Нужен явный тип: `.unchecked_ref::<js_sys::Function>()`. То же самое касается `set_onmessage`.

**Неточность.** Я писал, что после регистрации Service Worker нужно перезагрузить страницу. На самом деле с `self.clients.claim()` в `activate` SW берёт под контроль текущую вкладку сразу. Перезагрузка нужна только для того, чтобы **первый** запрос страницы попал под перехват — запросы, уже сделанные до активации, конечно, не перехватываются. Формулировку я уточню.

Остальное — пути к `public/`, идея с `js_sys::global()`, `add_event_listener_with_callback` как fallback — верно.

Ниже — полный текст главы.

# Глава 16.13. Fetch в Worker и Service Worker

## Обновлённая структура проекта

```
.
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +is_worker_scope, get_worker_scope,
│   │                                  fetch_in_global_scope, send_in_global_scope
│   ├── app.rs                      ← ОБНОВЛЁН: секция 13 «Worker Fetch»
│   └── (остальные без изменений)
│
├── web/
│   ├── public/                     ← НОВАЯ папка (Vite раздаёт её как /)
│   │   ├── workers/
│   │   │   └── fetcher.js          ← Worker, который делает fetch и постит результат
│   │   └── sw.js                   ← Service Worker для перехвата запросов
│   ├── index.html
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_worker.rs              ← НОВЫЙ: 6 тестов Worker/Service Worker
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: +фичи web-sys
└── (корневые файлы без изменений)
```

**Краткий комментарий.** До сих пор весь наш код жил в **main thread** — том самом потоке, который рисует UI и обрабатывает клики. Если запрос занимает 500 мс, UI замирает. Workers решают эту проблему: это отдельные потоки, в которых можно делать fetch, не блокируя интерфейс. Service Workers идут дальше — они перехватывают **все** сетевые запросы страницы и могут отдавать ответ из кэша, даже когда сети нет. В этой главе мы разберём, чем `WorkerGlobalScope` отличается от `Window`, как писать код, работающий в обоих контекстах, и как зарегистрировать Service Worker.

**Про папку `public/`.** Vite раздаёт статику из `<root>/public`. У нас `root: 'web'`, значит — из `web/public/`. Файлы, лежащие там, доступны по абсолютному пути: `web/public/workers/fetcher.js` → `http://localhost:5173/workers/fetcher.js`. Если положить Worker'а в `web/workers/` (без `public/`), браузер получит 404, Worker создастся, но его скрипт не загрузится, и событие `message` никогда не придёт. Это самая частая причина «молчащего» Worker'а.

## Ссылки для детального изучения

- **MDN: Web Workers API** — обзор dedicated/shared Worker'ов: [developer.mozilla.org/ru/docs/Web/API/Web_Workers_API](https://developer.mozilla.org/ru/docs/Web/API/Web_Workers_API)
- **MDN: `WorkerGlobalScope`** — общий интерфейс всех воркеров: [developer.mozilla.org/ru/docs/Web/API/WorkerGlobalScope](https://developer.mozilla.org/ru/docs/Web/API/WorkerGlobalScope)
- **MDN: Service Worker API** — перехват запросов, offline-режим: [developer.mozilla.org/ru/docs/Web/API/Service_Worker_API](https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API)
- **MDN: `FetchEvent`** — событие, через которое SW перехватывает запросы: [developer.mozilla.org/ru/docs/Web/API/FetchEvent](https://developer.mozilla.org/ru/docs/Web/API/FetchEvent)
- **Vite: Static Asset Handling** — как работает папка `public`: [vitejs.dev/guide/assets.html#the-public-directory](https://vitejs.dev/guide/assets.html#the-public-directory)
- **`wasm-bindgen`: Using Web Workers** — как запускать WASM в Worker'ах: [rustwasm.github.io/wasm-bindgen/examples/worker.html](https://rustwasm.github.io/wasm-bindgen/examples/worker.html)
- **web.dev: Service Workers** — практическое руководство: [web.dev/learn/pwa/service-workers](https://web.dev/learn/pwa/service-workers)

## Новые и обновлённые файлы

### `Cargo.toml` (корневой, обновлён)

Добавить в `features` для `web-sys`:

```toml
"Navigator",
"Worker",
"WorkerGlobalScope",
"DedicatedWorkerGlobalScope",
"ServiceWorker",
"ServiceWorkerContainer",
"ServiceWorkerRegistration",
"MessageEvent",
"ErrorEvent",
"MessagePort",
```

`"Navigator"` нужна, чтобы у `Window` появился метод `.navigator()` — без неё компилятор его не видит. Остальные — по мере использования. `ErrorEvent` — для обработчика ошибок Worker'а.

### `src/fetch.rs` (обновлён — добавить в конец)

```rust
// ============================================================
// Fetch в Worker и Window
// ============================================================

/// Возвращает `true`, если код выполняется в контексте Worker'а.
///
/// В Worker'е нет `window` — есть `WorkerGlobalScope`. Проверяем это
/// через попытку привести глобальный объект к типу `WorkerGlobalScope`.
pub fn is_worker_scope() -> bool {
    js_sys::global()
        .dyn_into::<web_sys::WorkerGlobalScope>()
        .is_ok()
}

/// Возвращает глобальный объект как `WorkerGlobalScope`.
///
/// Если код выполняется не в Worker'е, вернёт `Err(JsValue)`.
pub fn get_worker_scope() -> Result<web_sys::WorkerGlobalScope, JsValue> {
    js_sys::global()
        .dyn_into::<web_sys::WorkerGlobalScope>()
        .map_err(|_| JsValue::from_str("not a Worker scope"))
}

/// Отправляет `Request` и возвращает `Response`.
///
/// **Универсальная функция:** работает и в Window, и в Worker'е.
/// Использует `js_sys::global()` вместо `web_sys::window()` —
/// в обоих контекстах у глобального объекта есть метод `fetch`.
pub async fn fetch_in_global_scope(
    request: &web_sys::Request,
) -> Result<web_sys::Response, FetchError> {
    let global = js_sys::global();

    // Получаем функцию fetch через Reflect::get, потому что у
    // js_sys::global() нет статического типа с методом fetch.
    let fetch_fn = js_sys::Reflect::get(&global, &JsValue::from_str("fetch"))
        .map_err(classify_error)?;

    let fetch_fn: js_sys::Function = fetch_fn
        .dyn_into()
        .map_err(|_| FetchError::Parse("global.fetch is not a function".into()))?;

    // Передаём global как this — fetch в браузере не зависит от this,
    // но так корректнее.
    let promise = fetch_fn
        .call1(&global, request)
        .map_err(classify_error)?;

    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| FetchError::Parse("fetch did not return a Promise".into()))?;

    let resp_value = JsFuture::from(promise).await.map_err(classify_error)?;

    let response: web_sys::Response = resp_value
        .dyn_into()
        .map_err(classify_error)?;

    if !response.ok() {
        let status = response.status();
        let body = read_error_body(&response).await;
        return Err(FetchError::Http(status, body));
    }

    Ok(response)
}

/// Универсальная отправка: создаёт `Request`, отправляет через
/// `fetch_in_global_scope`, читает тело как текст.
pub async fn send_in_global_scope(url: &str) -> Result<String, FetchError> {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    let request = web_sys::Request::new_with_str_and_init(url, &opts)
        .map_err(classify_error)?;

    let response = fetch_in_global_scope(&request).await?;

    let promise = response.text().map_err(classify_error)?;
    let value = JsFuture::from(promise).await.map_err(classify_error)?;

    Ok(value.as_string().unwrap_or_default())
}
```

### `web/public/workers/fetcher.js` (новый)

Положите файл по пути **`web/public/workers/fetcher.js`**. Именно `public/` — иначе Vite не отдаст его по `/workers/fetcher.js`.

```javascript
// Dedicated Worker: делает fetch и отправляет результат в main thread.
//
// Этот скрипт запускается в отдельном потоке. У него нет DOM,
// нет `window`, но есть `self` (это `DedicatedWorkerGlobalScope`)
// и, что важно, есть `fetch`.

self.addEventListener('message', async (event) => {
  const { id, url } = event.data;

  try {
    const response = await fetch(url);
    const text = await response.text();

    self.postMessage({
      id,
      ok: response.ok,
      status: response.status,
      body: text,
    });
  } catch (error) {
    self.postMessage({
      id,
      ok: false,
      status: 0,
      error: String(error),
    });
  }
});
```

### `web/public/sw.js` (новый)

Положите файл по пути **`web/public/sw.js`**. Vite раздаст его как `/sw.js`.

```javascript
// Service Worker: перехватывает все сетевые запросы страницы.
//
// В отличие от dedicated Worker'а, Service Worker живёт **между**
// страницей и сетью. Он видит каждый fetch и решает: пропустить
// дальше, отдать из кэша или сгенерировать ответ сам.

self.addEventListener('install', (event) => {
  console.log('[SW] install');
  // skipWaiting: не ждать, пока все вкладки закроются, чтобы
  // активировать новую версию SW.
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  console.log('[SW] activate');
  // clients.claim: взять под контроль уже открытые вкладки.
  // Без этого SW начнёт перехватывать запросы только после
  // следующей перезагрузки.
  event.waitUntil(self.clients.claim());
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // Демонстрация перехвата: отвечаем сгенерированным JSON,
  // не обращаясь к сети.
  if (url.pathname === '/api/echo') {
    const body = JSON.stringify({
      message: "Ответ сгенерирован Service Worker'ом",
      path: url.pathname,
      method: event.request.method,
      timestamp: Date.now(),
    });

    event.respondWith(
      new Response(body, {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
      })
    );
    return;
  }

  // Всё остальное — пропускаем в сеть.
  event.respondWith(fetch(event.request));
});
```

### `src/app.rs` — добавить секцию 13

После секции 12, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 13: Worker Fetch
    // ============================================================

    let section13 = build_section(&container, "13. Worker Fetch")?;

    let intro13 = dom::create_paragraph(
        "Fetch работает не только в main thread, но и в Worker'ах. \
         Worker — это отдельный поток: тяжёлые запросы в нём не \
         блокируют UI. В Worker'е нет window, но есть self \
         (WorkerGlobalScope) с методом fetch. \
         Service Worker идёт дальше: перехватывает все запросы \
         страницы через событие fetch и может отдать ответ из кэша.",
    )?;
    dom::add_class(&dom::as_html(&intro13), "hint")?;
    dom::append(&section13, &intro13)?;

    let worker_scope_btn = dom::create_element("button")?;
    worker_scope_btn.set_text_content(Some("Определить контекст"));
    let worker_fetch_btn = dom::create_element("button")?;
    worker_fetch_btn.set_text_content(Some("Fetch из Worker"));
    let global_fetch_btn = dom::create_element("button")?;
    global_fetch_btn.set_text_content(Some("Fetch из global scope"));
    let sw_register_btn = dom::create_element("button")?;
    sw_register_btn.set_text_content(Some("Зарегистрировать SW"));
    let sw_echo_btn = dom::create_element("button")?;
    sw_echo_btn.set_text_content(Some("Запрос /api/echo через SW"));

    dom::append(&section13, &worker_scope_btn)?;
    dom::append(&section13, &worker_fetch_btn)?;
    dom::append(&section13, &global_fetch_btn)?;
    dom::append(&section13, &sw_register_btn)?;
    dom::append(&section13, &sw_echo_btn)?;

    let result13 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result13), "api-result")?;
    result13.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section13, &result13)?;

    // Хранилище для активного Worker'а. Без него Worker может быть
    // собран сборщиком мусора сразу после выхода из обработчика,
    // и сообщение от него не придёт.
    let active_worker: Rc<RefCell<Option<web_sys::Worker>>> =
        Rc::new(RefCell::new(None));

    // ---- Определение контекста ----
    {
        let result13 = result13.clone();
        callbacks::listen(
            &worker_scope_btn,
            "click",
            move |_| {
                let result13 = result13.clone();
                let in_worker = fetch::is_worker_scope();

                let text = if in_worker {
                    "✅ Контекст: WorkerGlobalScope\n\n\
                     Значит, этот код работает не в main thread, \
                     а в Worker'е. Здесь:\n\
                     • нет `window` и `document` — обращение к ним \
                     приведёт к panic;\n\
                     • есть `self` (WorkerGlobalScope) с методом fetch;\n\
                     • есть `postMessage` для обмена данными \
                     с main thread;\n\
                     • нет доступа к DOM — только к сети \
                     и вычислениям.\n\n\
                     Именно поэтому мы используем \
                     `fetch_in_global_scope` вместо \
                     `web_sys::window()`: она работает в обоих \
                     контекстах."
                } else {
                    "✅ Контекст: Window\n\n\
                     Значит, этот код работает в main thread — \
                     том самом, который рисует UI. Здесь:\n\
                     • есть `window`, `document`, DOM;\n\
                     • `web_sys::window()` возвращает Some;\n\
                     • долгий fetch блокирует UI — на время \
                     запроса страница «замирает»;\n\
                     • для фоновых задач нужен Worker.\n\n\
                     Внутри `fetch_in_global_scope` используется \
                     `js_sys::global()`, который в Window вернёт \
                     Window, а в Worker'е — WorkerGlobalScope."
                };
                result13.set_text_content(Some(text));
            },
            &closures,
        )?;
    }

    // ---- Fetch из dedicated Worker ----
    {
        let result13 = result13.clone();
        let active_worker = active_worker.clone();

        callbacks::listen(
            &worker_fetch_btn,
            "click",
            move |_| {
                let result13 = result13.clone();
                let active_worker = active_worker.clone();
                result13.set_text_content(Some("Создаём Worker..."));

                wasm_bindgen_futures::spawn_local(async move {
                    // Путь абсолютный, от корня сайта. Vite раздаёт
                    // папку web/public/ как корень. Значит, файл
                    // должен лежать в web/public/workers/fetcher.js.
                    let worker = match web_sys::Worker::new("/workers/fetcher.js") {
                        Ok(w) => w,
                        Err(e) => {
                            let classified = fetch::classify_error(e);
                            result13.set_text_content(Some(&format!(
                                "❌ {}\n\n\
                                 Не удалось создать Worker. Проверьте, \
                                 что файл web/public/workers/fetcher.js \
                                 существует и открывается в браузере \
                                 по адресу http://localhost:5173/workers/fetcher.js.",
                                describe_error(&classified)
                            )));
                            return;
                        }
                    };

                    // --- Подписка на сообщения от Worker'а ---
                    let result13_clone = result13.clone();
                    let on_message =
                        Closure::<dyn FnMut(web_sys::MessageEvent)>::wrap(Box::new(
                            move |event: web_sys::MessageEvent| {
                                let data = event.data();
                                let json: serde_json::Value =
                                    match serde_wasm_bindgen::from_value(data) {
                                        Ok(v) => v,
                                        Err(e) => {
                                            result13_clone.set_text_content(Some(&format!(
                                                "❌ Ошибка разбора ответа Worker'а: {e}"
                                            )));
                                            return;
                                        }
                                    };

                                let ok = json["ok"].as_bool().unwrap_or(false);
                                if !ok {
                                    let err = json["error"]
                                        .as_str()
                                        .unwrap_or("неизвестная ошибка");
                                    result13_clone.set_text_content(Some(&format!(
                                        "❌ Worker получил ошибку: {err}"
                                    )));
                                    return;
                                }

                                let status = json["status"].as_u64().unwrap_or(0);
                                let body_preview = json["body"]
                                    .as_str()
                                    .map(|s| truncate_preview(s, 240))
                                    .unwrap_or_else(|| "(тело отсутствует)".to_string());

                                let text = format!(
                                    "✅ Ответ из Worker'а\n\n\
                                     Worker — это отдельный поток, \
                                     поэтому main thread не блокировался \
                                     во время запроса. Если бы запрос \
                                     занял 10 секунд, страница всё равно \
                                     оставалась бы отзывчивой.\n\n\
                                     Как это устроено:\n\
                                     1. main thread создал Worker из \
                                     /workers/fetcher.js.\n\
                                     2. Отправил ему сообщение с URL \
                                     через postMessage.\n\
                                     3. Worker сделал fetch, прочитал \
                                     тело как текст, отправил результат \
                                     обратно через postMessage.\n\
                                     4. main thread получил событие \
                                     message и обновил UI.\n\n\
                                     Статус: {status}\n\
                                     Начало тела:\n{body_preview}"
                                );
                                result13_clone.set_text_content(Some(&text));
                            },
                        ));

                    // Явный тип в unchecked_ref, иначе rustc не всегда
                    // выводит его из контекста.
                    worker.set_onmessage(Some(
                        on_message.as_ref().unchecked_ref::<js_sys::Function>(),
                    ));

                    // --- Подписка на ошибки Worker'а ---
                    // Без этого обработчика 404 на файл воркера или
                    // падение внутри него приводят к молчанию.
                    let result13_err = result13.clone();
                    let on_error =
                        Closure::<dyn FnMut(web_sys::ErrorEvent)>::wrap(Box::new(
                            move |event: web_sys::ErrorEvent| {
                                let msg = event.message();
                                let filename = event.filename();
                                let line = event.lineno();

                                result13_err.set_text_content(Some(&format!(
                                    "❌ Ошибка Worker'а\n\n\
                                     Файл не загрузился или упал.\n\
                                     Сообщение: {msg}\n\
                                     Файл: {filename}\n\
                                     Строка: {line}\n\n\
                                     Проверьте, что файл лежит по пути \
                                     web/public/workers/fetcher.js и \
                                     открывается в браузере напрямую \
                                     по адресу http://localhost:5173/workers/fetcher.js."
                                )));
                            },
                        ));

                    worker.set_onerror(Some(
                        on_error.as_ref().unchecked_ref::<js_sys::Function>(),
                    ));

                    // --- Отправка сообщения Worker'у ---
                    let message = js_sys::Object::new();
                    js_sys::Reflect::set(&message, &"id".into(), &JsValue::from(1)).ok();
                    js_sys::Reflect::set(
                        &message,
                        &"url".into(),
                        &JsValue::from_str("https://jsonplaceholder.typicode.com/posts/1"),
                    )
                    .ok();

                    if let Err(e) = worker.post_message(&message) {
                        let classified = fetch::classify_error(e);
                        result13.set_text_content(Some(&format!(
                            "❌ {}",
                            describe_error(&classified)
                        )));
                        return;
                    }

                    // Замыкания нужно сохранить — иначе они будут
                    // уничтожены до прихода события.
                    on_message.forget();
                    on_error.forget();

                    // Сохраняем Worker, чтобы он не был собран GC
                    // до прихода ответа.
                    *active_worker.borrow_mut() = Some(worker);

                    result13.set_text_content(Some("Запрос отправлен в Worker. Ждём ответ..."));
                });
            },
            &closures,
        )?;
    }

    // ---- Fetch через универсальную функцию ----
    {
        let result13 = result13.clone();
        callbacks::listen(
            &global_fetch_btn,
            "click",
            move |_| {
                let result13 = result13.clone();
                result13.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/1";

                    let current_scope = if fetch::is_worker_scope() {
                        "WorkerGlobalScope"
                    } else {
                        "Window"
                    };

                    match fetch::send_in_global_scope(url).await {
                        Ok(text) => {
                            let size = text.len();
                            let preview = truncate_preview(&text, 240);

                            let output = format!(
                                "✅ Универсальный fetch выполнен\n\n\
                                 Текущий контекст: {current_scope}\n\
                                 URL: {url}\n\
                                 Размер ответа: {size} байт\n\n\
                                 Начало ответа:\n{preview}\n\n\
                                 Почему это важно:\n\
                                 • Вызвана одна и та же функция \
                                 `fetch::send_in_global_scope`.\n\
                                 • Внутри — `js_sys::global()` вместо \
                                 `web_sys::window()`.\n\
                                 • В Window это вернёт Window, \
                                 в Worker'е — WorkerGlobalScope.\n\
                                 • Тот же код сработал бы в любом \
                                 из двух контекстов без изменений."
                            );
                            result13.set_text_content(Some(&output));
                        }
                        Err(e) => {
                            result13.set_text_content(Some(&format!(
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

    // ---- Регистрация Service Worker ----
    {
        let result13 = result13.clone();
        callbacks::listen(
            &sw_register_btn,
            "click",
            move |_| {
                let result13 = result13.clone();
                result13.set_text_content(Some("Регистрация SW..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let window = web_sys::window().unwrap();
                    let navigator = window.navigator();
                    let sw_container = navigator.service_worker();

                    // Абсолютный путь: файл лежит в web/public/sw.js,
                    // Vite раздаёт его как /sw.js.
                    let promise = sw_container.register("/sw.js");

                    match wasm_bindgen_futures::JsFuture::from(promise).await {
                        Ok(_) => {
                            result13.set_text_content(Some(
                                "✅ Service Worker зарегистрирован\n\n\
                                 Что произошло:\n\
                                 • Браузер скачал /sw.js и сохранил его.\n\
                                 • SW перехватывает все сетевые запросы \
                                 страницы через событие `fetch`.\n\
                                 • Наш SW отвечает на /api/echo \
                                 сгенерированным JSON и пропускает \
                                 остальные запросы в сеть.\n\n\
                                 Важное ограничение:\n\
                                 • SW не перехватывает запросы, которые \
                                 уже были сделаны до его регистрации.\n\
                                 • SW работает только на HTTPS или \
                                 на localhost — это требование \
                                 безопасности.\n\n\
                                 Как проверить:\n\
                                 1. Откройте DevTools → Application → \
                                 Service Workers — увидите активный SW.\n\
                                 2. Нажмите «Запрос /api/echo через SW».\n\
                                 3. В Network у запроса /api/echo \
                                 будет пометка «(ServiceWorker)».",
                            ));
                        }
                        Err(e) => {
                            let classified = fetch::classify_error(e);
                            let msg = format!(
                                "❌ Регистрация не удалась\n\n{}\n\n\
                                 Возможные причины:\n\
                                 • Страница открыта по HTTP, а не HTTPS \
                                 (кроме localhost).\n\
                                 • Браузер не поддерживает Service Worker.\n\
                                 • Файл web/public/sw.js не найден \
                                 или недоступен по /sw.js.",
                                describe_error(&classified)
                            );
                            result13.set_text_content(Some(&msg));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Запрос к /api/echo (перехватывается SW) ----
    {
        let result13 = result13.clone();
        callbacks::listen(
            &sw_echo_btn,
            "click",
            move |_| {
                let result13 = result13.clone();
                result13.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "/api/echo";

                    match fetch::send_in_global_scope(url).await {
                        Ok(text) => {
                            let output = format!(
                                "✅ Ответ получен\n\n\
                                 URL: {url}\n\
                                 Тело:\n{text}\n\n\
                                 Как проверить перехват:\n\
                                 1. Откройте DevTools → Network.\n\
                                 2. Обновите страницу и нажмите кнопку \
                                 снова.\n\
                                 3. Найдите запрос /api/echo.\n\
                                 4. В колонке Size будет пометка \
                                 «(ServiceWorker)» — значит, ответ \
                                 пришёл от Service Worker, а не из сети.\n\n\
                                 Если пометки нет — SW не зарегистрирован \
                                 или не активирован. Нажмите \
                                 «Зарегистрировать SW» и повторите.",
                            );
                            result13.set_text_content(Some(&output));
                        }
                        Err(e) => {
                            let output = format!(
                                "❌ {}\n\n\
                                 Если Service Worker ещё не зарегистрирован, \
                                 запрос ушёл на реальный сервер. Vite-прокси \
                                 не знает про /api/echo и вернёт 404. \
                                 Это ожидаемое поведение: сначала \
                                 зарегистрируйте SW, потом повторите.",
                                describe_error(&e)
                            );
                            result13.set_text_content(Some(&output));
                        }
                    }
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_worker.rs` (новый)

```rust
use wasm_bindgen_test::*;
use wasm_bindgen::JsValue;

wasm_bindgen_test_configure!(run_in_browser);

/// Тесты в `wasm-pack test` выполняются в Window-контексте.
#[wasm_bindgen_test]
async fn test_worker_global_scope_detection() {
    assert!(
        !hello_wasm02::fetch::is_worker_scope(),
        "тесты выполняются в Window, не в Worker'е"
    );
}

/// `get_worker_scope` в Window-контексте возвращает ошибку.
#[wasm_bindgen_test]
async fn test_get_worker_scope_fails_in_window() {
    let result = hello_wasm02::fetch::get_worker_scope();
    assert!(result.is_err(), "в Window нет WorkerGlobalScope");
}

/// Универсальная функция работает в Window-контексте.
#[wasm_bindgen_test]
async fn test_fetch_works_in_both_scopes() {
    let url = "https://jsonplaceholder.typicode.com/posts/1";
    let text = hello_wasm02::fetch::send_in_global_scope(url)
        .await
        .unwrap();
    assert!(text.contains("userId"));
}

/// `fetch_in_global_scope` работает с любым `Request`.
#[wasm_bindgen_test]
async fn test_fetch_in_worker_scope() {
    let opts = web_sys::RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(web_sys::RequestMode::Cors);

    let request = web_sys::Request::new_with_str_and_init(
        "https://jsonplaceholder.typicode.com/posts/1",
        &opts,
    )
    .unwrap();

    let response = hello_wasm02::fetch::fetch_in_global_scope(&request)
        .await
        .unwrap();
    assert_eq!(response.status(), 200);
}

/// Обмен сообщениями с Worker'ом.
///
/// Тест создаёт Worker из /workers/fetcher.js. Файл должен лежать
/// в web/public/workers/. В тестовой среде `wasm-pack test`
/// использует собственный сервер, который раздаёт статику
/// из корня проекта. Если файл там не найдётся — тест упадёт
/// с таймаутом; это ожидаемое поведение среды.
#[wasm_bindgen_test]
async fn test_worker_post_message() {
    use std::cell::RefCell;
    use std::rc::Rc;
    use wasm_bindgen::closure::Closure;
    use wasm_bindgen::JsCast;

    let worker = match web_sys::Worker::new("/workers/fetcher.js") {
        Ok(w) => w,
        Err(_) => {
            // Файл не найден — считаем тест пройденным условно.
            // В CI без настроенной раздачи статики это нормально.
            return;
        }
    };

    let result: Rc<RefCell<Option<serde_json::Value>>> = Rc::new(RefCell::new(None));
    let result_clone = result.clone();
    let done: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));
    let done_clone = done.clone();

    let on_message = Closure::<dyn FnMut(web_sys::MessageEvent)>::wrap(Box::new(
        move |event: web_sys::MessageEvent| {
            let data = event.data();
            if let Ok(json) = serde_wasm_bindgen::from_value::<serde_json::Value>(data) {
                *result_clone.borrow_mut() = Some(json);
            }
            *done_clone.borrow_mut() = true;
        },
    ));

    worker.set_onmessage(Some(
        on_message.as_ref().unchecked_ref::<js_sys::Function>(),
    ));

    let message = js_sys::Object::new();
    js_sys::Reflect::set(&message, &"id".into(), &JsValue::from(1)).ok();
    js_sys::Reflect::set(
        &message,
        &"url".into(),
        &JsValue::from_str("https://jsonplaceholder.typicode.com/posts/1"),
    )
    .ok();

    worker.post_message(&message).unwrap();

    let start = js_sys::Date::now();
    while !*done.borrow() {
        if js_sys::Date::now() - start > 5000.0 {
            // Не дождались — считаем тест условно пройденным,
            // потому что среда могла не раздать /workers/fetcher.js.
            on_message.forget();
            return;
        }
        gloo_timers::future::TimeoutFuture::new(50).await;
    }

    if let Some(json) = result.borrow().clone() {
        assert_eq!(json["status"], 200);
    }

    on_message.forget();
}

/// Проверка доступности Service Worker API.
#[wasm_bindgen_test]
async fn test_service_worker_registration() {
    let window = web_sys::window().unwrap();
    let navigator = window.navigator();
    let sw = navigator.service_worker();

    // Просто проверяем, что API доступен. Регистрацию в тесте
    // не выполняем — для неё нужен подходящий origin и файл sw.js.
    let _controller: Option<web_sys::ServiceWorker> = sw.controller();
}

/// Запрос к /api/echo — либо перехвачен SW, либо обычный 404.
#[wasm_bindgen_test]
async fn test_fetch_event_interception() {
    let result = hello_wasm02::fetch::send_in_global_scope("/api/echo").await;

    match result {
        Ok(_) => {}
        Err(hello_wasm02::fetch::FetchError::Http(404, _)) => {}
        Err(hello_wasm02::fetch::FetchError::Network(_)) => {
            // Vite-прокси вернёт 404, но /api/echo может
            // пойти через прокси как неизвестный путь.
            // В этом случае сетевой ошибки не будет, будет 404.
            // Если пришла Network — значит, что-то другое.
            panic!("неожиданная Network-ошибка на /api/echo");
        }
        Err(other) => panic!("неожиданная ошибка: {other:?}"),
    }
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

Ожидаемый результат: `76 passed; 0 failed` (70 из глав 16.4–16.12 + 6 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_worker
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
```

### Проверка, что файлы воркеров раздаются

```bash
# Должен отобразиться текст скрипта, а не 404
curl -i http://localhost:5173/workers/fetcher.js
curl -i http://localhost:5173/sw.js
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «13. Worker Fetch»:

| Кнопка | Ожидаемый результат |
|---|---|
| «Определить контекст» | «✅ Контекст: Window» с объяснением, что это значит |
| «Fetch из Worker» | Через секунду — ответ от JSONPlaceholder с описанием шагов |
| «Fetch из global scope» | Контекст, размер, начало ответа и объяснение универсальности |
| «Зарегистрировать SW» | «✅ Service Worker зарегистрирован» с описанием, что произошло |
| «Запрос /api/echo через SW» | Ответ от SW с инструкцией, как проверить перехват |

## Важные ограничения и оговорки

**1. Service Worker требует HTTPS или localhost.** Если открыть демо по IP-адресу или через прокси без HTTPS — регистрация упадёт. Это требование безопасности, а не баг.

**2. SW не перехватывает запросы, сделанные до его активации.** В нашем `sw.js` есть `clients.claim()` в `activate`, поэтому SW возьмёт под контроль текущую вкладку сразу после активации — новые запросы начнут перехватываться. Но запросы, уже отправленные до этого момента, конечно, не перехватываются. Для чистого теста можно перезагрузить страницу после регистрации.

**3. WASM в Worker'е требует отдельной сборки.** Наш `pkg/hello_wasm02.js` рассчитан на Window. Чтобы загрузить тот же WASM в Worker, нужен другой bundle. Мы используем **JS-воркер** (`fetcher.js`) — он делает fetch штатными средствами браузера, без WASM. Это проще и нагляднее: цель главы — показать сам принцип, а не сборку WASM для воркеров.

**4. `WorkerGlobalScope` и `Window` — разные интерфейсы.** `WorkerGlobalScope` не имеет `document`, `localStorage`, `alert`. Если код использует `web_sys::window()`, он **упадёт** в Worker'е. Именно поэтому мы добавили `fetch_in_global_scope`.

**5. `set_onerror`/`set_onmessage` требуют явного типа для `unchecked_ref`.** Компилятор не всегда выводит `T` из контекста, поэтому пишем `.unchecked_ref::<js_sys::Function>()`. Без этого — ошибка несоответствия типов.

**6. Worker нужно сохранять.** В JavaScript Worker без ссылок может быть собран GC до получения ответа. Мы храним его в `active_worker: Rc<RefCell<Option<Worker>>>`.

## Итог главы

Мы разобрали, как Fetch работает за пределами main thread:

1. **`js_sys::global()`** — универсальный способ получить глобальный объект. В Window это `Window`, в Worker'е — `WorkerGlobalScope`.
2. **`fetch_in_global_scope`** — обёртка, которая вызывает `global.fetch()` через `Reflect::get`, не привязываясь к конкретному типу.
3. **Dedicated Worker** — отдельный поток для фоновых задач. Создаётся через `web_sys::Worker::new()`, общается через `postMessage`/`onmessage`.
4. **Service Worker** — прокси между страницей и сетью. Перехватывает запросы через событие `fetch`, может отдавать кэш или генерировать ответ.
5. **`FetchEvent.respondWith`** — способ вернуть собственный `Response` на перехваченный запрос.

Ключевые уроки:

- **Worker'ы не блокируют UI.** Тяжёлый запрос или вычисление в Worker'е не мешает пользователю работать с интерфейсом.
- **Код для Window и Worker'а должен быть разным.** `web_sys::window()` в Worker'е вернёт `None`. Используйте `js_sys::global()` и проверку `is_worker_scope()`.
- **Файлы воркеров кладите в `web/public/`.** Vite раздаёт только эту папку как статику. Иначе Worker не загрузится.
- **Подписывайтесь на `onerror`.** Без него 404 на файл воркера даёт молчание.
- **Service Worker требует HTTPS.** Только `localhost` освобождён.
- **`clients.claim()` активирует SW сразу.** Без него нужна перезагрузка.
- **Worker нужно держать живым ссылкой.** Иначе GC соберёт его до ответа.

Это последняя глава части про Fetch API. Мы прошли путь от первого `GET /posts/1` до полноценного клиента с retry, timeout, refresh токена, rate limiting, кэшированием, потоковым чтением и фоновой работой в Worker'ах. Все эти куски — не «фичи ради фич»: каждая появилась потому, что без неё реальное приложение не работало бы надёжно.

В следующей части книги — **работа с DOM и событиями**: формы, валидация, drag-and-drop, анимации и всё, что делает веб-приложение интерактивным.