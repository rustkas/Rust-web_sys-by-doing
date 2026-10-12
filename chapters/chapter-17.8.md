# 17.8. Streams в Service Worker

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← без изменений
│   ├── streams_wasm.rs             ← без изменений
│   ├── fetch.rs                    ← без изменений (все нужные функции уже есть)
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.8
│   └── (остальные без изменений)
│
├── web/
│   └── public/
│       └── sw.js                   ← ОБНОВЛЁН: +/api/logs/stream, +/api/proxy-stream/*
│
├── tests/
│   ├── wasm_sw_streams.rs          ← НОВЫЙ: 4 теста (#[ignore])
│   └── (остальные без изменений)
│
├── package.json                    ← без изменений
└── (корневые файлы без изменений)
```

**Краткий комментарий.** Service Worker — это прокси между страницей и сетью. Он перехватывает все HTTP-запросы страницы через событие `fetch` и может ответить **не только готовым `Response`**, но и `Response` с потоковым телом. Это открывает два новых сценария: SW **сам генерирует** поток данных (например, лог-записи) или **модифицирует** ответ из сети на лету (добавляет префиксы, логирует содержимое, дополняет из кэша). Клиентский код не меняется: он видит обычный `Response` с `ReadableStream` в `body` и работает с ним через уже знакомые инструменты. Эта глава — последняя в части про Streams API. Она опирается на инфраструктуру главы 16.13 (`web/public/sw.js`, регистрация SW), но **не переписывает** её — только дополняет двумя новыми обработчиками.

**Правило архитектуры.** `streams.rs`, `streams_wasm.rs`, `fetch.rs`, `lib.rs` не трогаются. Все нужные функции уже есть: `get_stream` из 16.10, `read_all` из 17.1, `read_stream_with_callback` из 16.10, `send_request_classified` из 16.5, `get_response_header` из 16.3. Эта глава дополняет `web/public/sw.js` двумя обработчиками, заменяет `app.rs` на секцию 17.8 и создаёт новый файл тестов.

## Ссылки для детального изучения

- **MDN: Service Worker API** — основной обзор: [developer.mozilla.org/ru/docs/Web/API/Service_Worker_API](https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API)
- **MDN: `FetchEvent.respondWith()`** — как SW отвечает на запрос: [developer.mozilla.org/ru/docs/Web/API/FetchEvent/respondWith](https://developer.mozilla.org/ru/docs/Web/API/FetchEvent/respondWith)
- **MDN: `Response()` конструктор** — принимает `ReadableStream` как тело: [developer.mozilla.org/ru/docs/Web/API/Response/Response](https://developer.mozilla.org/ru/docs/Web/API/Response/Response)
- **MDN: `Response.clone()`** — почему нельзя прочитать body дважды: [developer.mozilla.org/ru/docs/Web/API/Response/clone](https://developer.mozilla.org/ru/docs/Web/API/Response/clone)
- **MDN: `TextDecoder`** — декодирование байтов в строки: [developer.mozilla.org/ru/docs/Web/API/TextDecoder](https://developer.mozilla.org/ru/docs/Web/API/TextDecoder)
- **MDN: `TextEncoder`** — кодирование строк в байты: [developer.mozilla.org/ru/docs/Web/API/TextEncoder](https://developer.mozilla.org/ru/docs/Web/API/TextEncoder)
- **web.dev: Service Workers** — практическое руководство: [web.dev/learn/pwa/service-workers](https://web.dev/learn/pwa/service-workers)
- **`tower-http::cors`** — почему в главе 16.13 SW работает на localhost: [docs.rs/tower-http/latest/tower_http/cors/index.html](https://docs.rs/tower-http/latest/tower_http/cors/index.html)
- **Chrome: Compression Streams API** — встроенные потоки gzip/deflate для SW: [developer.chrome.com/docs/web-platform/compression-streams](https://developer.chrome.com/docs/web-platform/compression-streams)

## Теоретическая часть

### Как SW возвращает поток

Service Worker перехватывает запрос через событие `fetch`. Внутри `event.respondWith(responseOrPromise)` можно передать:

- **готовый `Response`** — обычный случай;
- **`Promise<Response>`** — если ответ надо сначала получить или сгенерировать;
- **`Response` с потоковым телом** — если данные генерируются на лету.

`Response` принимает `ReadableStream` как `body`:

```js
const stream = new ReadableStream({...});
event.respondWith(new Response(stream, {
    status: 200,
    headers: { 'Content-Type': 'text/plain' },
}));
```

Клиент видит это как обычный HTTP-ответ. `fetch()` возвращает `Response`, у которого `body` — `ReadableStream`. Дальше мы работаем с ним через уже знакомые инструменты: `get_stream`, `js_to_async_read`, `read_all`.

### Модификация ответа из сети на лету

SW может пропустить запрос в сеть, получить `Response`, но не отдавать его клиенту как есть, а обернуть в свой поток:

```js
event.respondWith((async () => {
    const upstream = await fetch(event.target);
    const reader = upstream.body.getReader();
    const stream = new ReadableStream({
        async pull(controller) {
            const { done, value } = await reader.read();
            if (done) return controller.close();
            controller.enqueue(value);
        },
    });
    return new Response(stream, upstream);
})());
```

Здесь `upstream.body` — тоже `ReadableStream`. Мы читаем его через default reader и **переупаковываем** в новый поток. По ходу можем:

- добавлять префиксы;
- заменять фрагменты;
- считать статистику;
- логировать содержимое;
- прерывать передачу при ошибке.

### Зачем это нужно на практике

- **Сжатие на лету.** Оборачиваем ответ в `CompressionStream('gzip')` и отдаём клиенту сжатый поток. Клиент даже не знает, что данные пришли сжатыми.
- **Логирование.** SW перехватывает все ответы, копирует их в `IndexedDB` для аналитики, но клиенту отдаёт оригинал без задержки.
- **Модификация HTML.** Внедряем в HTML-ответ баннер или скрипт, не загружая весь HTML в память.
- **Offline-fallback на лету.** Если сеть оборвалась на середине ответа, SW может дополнить его данными из кэша.
- **Замена API-ответа.** Разработчик отлаживает UI, а бэкенд ещё не готов. SW отдаёт заранее заготовленный поток тестовых данных.
- **Аутентификация.** SW добавляет `Authorization` к исходящим запросам или проверяет права на входящие ответы.

### Ограничения и подводные камни

- **SW живёт в отдельном JS-контексте.** У него нет `window`, `document`, `localStorage` (есть `IndexedDB` и `CacheStorage`).
- **HTTPS или localhost.** На HTTP SW не регистрируется, кроме `localhost`.
- **Активация.** SW не перехватывает запросы до первой активации. Запросы, сделанные до регистрации, уходят в сеть напрямую.
- **Одна ошибка — всё падает.** Если SW выбросит исключение в обработчике `fetch`, клиент получит сетевую ошибку. Оборачивайте логику в `try/catch`.
- **Body можно прочитать один раз.** `Response.body` — поток. Если SW прочитал его для логирования, клиенту уже ничего не останется. Используйте `response.clone()` до чтения, если нужно и то, и другое.
- **`upstream.body` может быть `null`.** Если сервер вернул ответ без тела (204, 304) или статус с ошибкой, `body` будет пустым. Проверяйте перед `getReader()`.
- **Не пишите бесконечные потоки.** Запрос, который никогда не закрывается, «зависнет» на уровне сети и не освободит ресурсы.

## `web/public/sw.js` — полное содержание

Файл заменяется целиком. Он содержит три обработчика: `/api/echo` (из 16.13), новый `/api/logs/stream` и новый `/api/proxy-stream/*`.

```js
// Service Worker: перехват запросов и потоковые ответы.
//
// Три сценария:
// 1. /api/echo — сгенерированный JSON-ответ (из главы 16.13).
// 2. /api/logs/stream — поток NDJSON-логов, генерируется в SW.
// 3. /api/proxy-stream/* — прокси с модификацией ответа на лету.

self.addEventListener('install', () => {
    console.log('[SW] install');
    // Активируем новую версию сразу, не дожидаясь закрытия вкладок.
    self.skipWaiting();
});

self.addEventListener('activate', (event) => {
    console.log('[SW] activate');
    // Берём под контроль уже открытые вкладки без перезагрузки.
    event.waitUntil(self.clients.claim());
});

self.addEventListener('fetch', (event) => {
    const url = new URL(event.request.url);

    // ---- 1. /api/echo: сгенерированный ответ (16.13) ----
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

    // ---- 2. /api/logs/stream: поток NDJSON, генерируется в SW ----
    if (url.pathname === '/api/logs/stream') {
        const encoder = new TextEncoder();
        const stream = new ReadableStream({
            async start(controller) {
                for (let i = 0; i < 10; i++) {
                    const line = JSON.stringify({
                        seq: i,
                        timestamp: Date.now(),
                        level: i % 3 === 0 ? 'warn' : 'info',
                        message: `Событие номер ${i}`,
                    }) + '\n';
                    controller.enqueue(encoder.encode(line));
                    await new Promise(r => setTimeout(r, 200));
                }
                controller.close();
            },
        });

        event.respondWith(new Response(stream, {
            status: 200,
            headers: {
                'Content-Type': 'application/x-ndjson',
                'X-Generated-By': 'ServiceWorker',
            },
        }));
        return;
    }

    // ---- 3. /api/proxy-stream/*: прокси с модификацией ----
    if (url.pathname.startsWith('/api/proxy-stream/')) {
        // /api/proxy-stream/stream/5 → /api/stream/5
        const target = url.pathname.replace('/api/proxy-stream/', '/api/');

        event.respondWith((async () => {
            try {
                const upstream = await fetch(target);

                // Если у ответа нет тела (204, 304, ошибки) — проксируем как есть.
                if (!upstream.body) {
                    return new Response(await upstream.text(), {
                        status: upstream.status,
                        headers: upstream.headers,
                    });
                }

                const reader = upstream.body.getReader();
                const decoder = new TextDecoder();
                const encoder = new TextEncoder();

                const stream = new ReadableStream({
                    async pull(controller) {
                        const { done, value } = await reader.read();
                        if (done) {
                            controller.close();
                            return;
                        }

                        // Декодируем чанк как поток текста (stream: true).
                        const text = decoder.decode(value, { stream: true });

                        // Добавляем префикс к каждой непустой строке.
                        const modified = text
                            .split('\n')
                            .map(line => line ? `[SW] ${line}` : line)
                            .join('\n');

                        controller.enqueue(encoder.encode(modified));
                    },
                });

                return new Response(stream, {
                    status: upstream.status,
                    headers: upstream.headers,
                });
            } catch (e) {
                console.error('[SW] proxy-stream error:', e);
                return new Response(`Service Worker error: ${e}`, { status: 502 });
            }
        })());
        return;
    }

    // ---- Всё остальное — пропускаем в сеть ----
    event.respondWith(fetch(event.request));
});
```

## `src/app.rs` — только секция 17.8

```rust
//! Демонстрация главы 17.8: Streams в Service Worker.
//!
//! Одна секция с тремя кнопками:
//! 1. SW генерирует поток — `/api/logs/stream`.
//! 2. SW модифицирует поток из сети — `/api/proxy-stream/stream/5`.
//! 3. SW: инкрементальное чтение с прогрессом.
//!
//! **Требует активированного SW.** Зарегистрируйте его в секции 13
//! и перезагрузите страницу.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::{dom, fetch, log, streams};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Streams в Service Worker")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Service Worker перехватывает запросы через событие fetch и \
         может ответить Response с потоковым телом. Клиент видит \
         обычный HTTP-ответ с ReadableStream в body. Ниже — три \
         сценария: SW генерирует поток, SW модифицирует поток из \
         сети, SW отдаёт поток с задержкой.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 8: Streams в Service Worker
    // ============================================================

    let s8 = build_section(&container, "8. Streams в Service Worker")?;

    let s8_hint = dom::create_paragraph(
        "SW регистрируется в секции 13. Если он не активирован, \
         запросы уйдут в сеть напрямую, и вы увидите 404 — эти \
         пути обслуживает только SW.",
    )?;
    dom::add_class(&dom::as_html(&s8_hint), "hint")?;
    dom::append(&s8, &s8_hint)?;

    let generate_btn = dom::create_element("button")?;
    generate_btn.set_text_content(Some("SW генерирует поток"));
    let modify_btn = dom::create_element("button")?;
    modify_btn.set_text_content(Some("SW модифицирует поток из сети"));
    let progress_btn = dom::create_element("button")?;
    progress_btn.set_text_content(Some("SW: стрим с задержкой"));

    dom::append(&s8, &generate_btn)?;
    dom::append(&s8, &modify_btn)?;
    dom::append(&s8, &progress_btn)?;

    // Прогресс-бар — нужен только третьей кнопке, но создадим
    // заранее, чтобы не пересоздавать DOM.
    let progress = dom::create_element("progress")?;
    progress.set_attribute("value", "0")?;
    progress.set_attribute("max", "10")?;
    progress.set_attribute("style", "width: 100%; height: 20px;")?;
    dom::append(&s8, &progress)?;

    let s8_result = make_result(&s8)?;

    // ---- Кнопка 1: SW генерирует поток ----
    {
        let s8_result = s8_result.clone();
        callbacks::listen(
            &generate_btn,
            "click",
            move |_| {
                let s8_result = s8_result.clone();
                s8_result.set_text_content(Some("Читаем поток..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "/api/logs/stream";

                    // Проверяем заголовок X-Generated-By.
                    let header_value = match fetch::send_request_classified(url).await {
                        Ok(resp) => {
                            fetch::get_response_header(&resp, "x-generated-by")
                                .unwrap_or_else(|| "(нет)".to_string())
                        }
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ Запрос не удался: {}\n\n\
                                 Возможно, Service Worker не активирован. \
                                 Зарегистрируйте его в секции 13 и \
                                 перезагрузите страницу.",
                                e.user_message()
                            )));
                            return;
                        }
                    };

                    // Читаем поток целиком.
                    let stream = match fetch::get_stream(url).await {
                        Ok(s) => s,
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ get_stream: {}",
                                e.user_message()
                            )));
                            return;
                        }
                    };

                    match streams::read_all(&stream).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            let lines: Vec<&str> = text.lines().collect();

                            s8_result.set_text_content(Some(&format!(
                                "✅ Ответ от Service Worker\n\n\
                                 Заголовок X-Generated-By: {header_value}\n\
                                 Строк получено: {}\n\n\
                                 Содержимое:\n{text}\n\n\
                                 Что произошло:\n\
                                 • SW перехватил запрос /api/logs/stream.\n\
                                 • SW сам сгенерировал ReadableStream \
                                 с 10 NDJSON-строками.\n\
                                 • Каждые 200 мс в поток попадала \
                                 одна строка.\n\
                                 • Клиент получил Response с потоковым \
                                 телом и прочитал его через get_stream \
                                 + read_all.\n\
                                 • Сервер в этом не участвовал.",
                                lines.len()
                            )));
                        }
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ read_all: {}",
                                e.user_message()
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Кнопка 2: SW модифицирует поток из сети ----
    {
        let s8_result = s8_result.clone();
        callbacks::listen(
            &modify_btn,
            "click",
            move |_| {
                let s8_result = s8_result.clone();
                s8_result.set_text_content(Some("Читаем модифицированный поток..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // /api/proxy-stream/stream/5 → SW проксирует на /api/stream/5,
                    // добавляя префикс [SW] к каждой строке.
                    let url = "/api/proxy-stream/stream/5";

                    let stream = match fetch::get_stream(url).await {
                        Ok(s) => s,
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ get_stream: {}\n\n\
                                 Возможно, Service Worker не активирован \
                                 или auth-server не запущен.",
                                e.user_message()
                            )));
                            return;
                        }
                    };

                    match streams::read_all(&stream).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            let lines: Vec<&str> = text.lines().collect();

                            // Проверяем, что все строки начинаются с [SW].
                            let all_prefixed = lines
                                .iter()
                                .filter(|l| !l.is_empty())
                                .all(|l| l.starts_with("[SW]"));

                            s8_result.set_text_content(Some(&format!(
                                "✅ Поток модифицирован Service Worker'ом\n\n\
                                 Строк получено: {}\n\
                                 Все строки с префиксом [SW]: {}\n\n\
                                 Содержимое:\n{text}\n\n\
                                 Что произошло:\n\
                                 • Клиент запросил /api/proxy-stream/stream/5.\n\
                                 • SW переписал путь на /api/stream/5 и \
                                 сделал fetch.\n\
                                 • Auth-server отдал поток чанков.\n\
                                 • SW читал чанки через getReader(), \
                                 декодировал через TextDecoder, добавлял \
                                 префикс [SW] к каждой строке, кодировал \
                                 обратно через TextEncoder.\n\
                                 • Клиент получил модифицированный поток, \
                                 не зная о вмешательстве.",
                                lines.len(),
                                all_prefixed
                            )));
                        }
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ read_all: {}",
                                e.user_message()
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Кнопка 3: SW стрим с задержкой, инкрементальное чтение ----
    {
        let s8_result = s8_result.clone();
        let progress = progress.clone();
        callbacks::listen(
            &progress_btn,
            "click",
            move |_| {
                let s8_result = s8_result.clone();
                let progress = progress.clone();

                progress.set_attribute("value", "0").ok();
                s8_result.set_text_content(Some("Читаем поток инкрементально..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "/api/logs/stream";

                    let stream = match fetch::get_stream(url).await {
                        Ok(s) => s,
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ get_stream: {}",
                                e.user_message()
                            )));
                            return;
                        }
                    };

                    // Прогресс и лог чанков.
                    let log_lines: Rc<RefCell<Vec<String>>> =
                        Rc::new(RefCell::new(Vec::new()));
                    let log_lines_c = log_lines.clone();
                    let s8_result_c = s8_result.clone();
                    let progress_c = progress.clone();

                    let start = js_sys::Date::now();

                    let total = match fetch::read_stream_with_callback(&stream, move |total, chunk_size| {
                        let mut lines = log_lines_c.borrow_mut();
                        lines.push(format!(
                            "  получено {} байт (чанк {})",
                            total, chunk_size
                        ));

                        let text = format!(
                            "📊 Прогресс:\n{}",
                            lines.join("\n")
                        );
                        s8_result_c.set_text_content(Some(&text));

                        // Обновляем progress: количество чанков.
                        let n = lines.len() as u64;
                        progress_c.set_attribute("value", &n.to_string()).ok();
                    })
                    .await
                    {
                        Ok(t) => t,
                        Err(e) => {
                            s8_result.set_text_content(Some(&format!(
                                "❌ read_stream_with_callback: {}",
                                e.user_message()
                            )));
                            return;
                        }
                    };

                    let elapsed = js_sys::Date::now() - start;

                    s8_result.set_text_content(Some(&format!(
                        "✅ Поток прочитан инкрементально\n\n\
                         Всего: {} байт\n\
                         Время: {:.0} мс\n\n{}\n\n\
                         Что произошло:\n\
                         • SW генерировал по одной строке каждые 200 мс.\n\
                         • Клиент читал чанки по мере поступления через \
                         read_stream_with_callback.\n\
                         • На каждый чанк обновлялся прогресс-бар и лог.\n\
                         • Данные не буферизовались целиком — это видно \
                         по инкрементальному обновлению.",
                        total,
                        elapsed,
                        log_lines.borrow().join("\n")
                    )));
                });
            },
            &closures,
        )?;
    }

    log!("Streams 17.8 demo ready");
    std::mem::forget(closures);

    Ok(())
}

// ============================================================
// Вспомогательные функции
// ============================================================

fn build_section(parent: &web_sys::Element, title: &str) -> Result<web_sys::Element, JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some(title));
    dom::append(&section, &h2)?;

    dom::append(parent, &section)?;
    Ok(section)
}

fn make_result(parent: &web_sys::Element) -> Result<web_sys::Element, JsValue> {
    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(нажмите кнопку)"));
    dom::append(parent, &result)?;
    Ok(result)
}
```

## `tests/wasm_sw_streams.rs` — 4 теста

```rust
//! Тесты Streams в Service Worker.
//!
//! **Запускать вручную:** `wasm-pack test --chrome --test wasm_sw_streams`.
//! Требуется, чтобы SW был зарегистрирован и активирован.
//!
//! В headless-Chrome SW регистрируется, но не успевает активироваться
//! до начала тестов. `wasm-bindgen-test` не умеет ждать активации SW.
//! Поэтому тесты помечены `#[ignore]`.

use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// SW генерирует поток — клиент читает его через `get_stream`.
#[wasm_bindgen_test]
#[ignore = "requires active Service Worker; run without --headless"]
async fn test_sw_stream_response() {
    let url = "/api/logs/stream";
    let stream = hello_wasm02::fetch::get_stream(url).await.unwrap();
    let bytes = hello_wasm02::streams::read_all(&stream).await.unwrap();
    let text = String::from_utf8_lossy(&bytes);

    // 10 строк NDJSON.
    let lines: Vec<&str> = text.lines().collect();
    assert_eq!(lines.len(), 10);

    // Каждая строка — валидный JSON.
    for line in lines {
        let json: serde_json::Value = serde_json::from_str(line).unwrap();
        assert!(json["seq"].is_number());
        assert!(json["level"].is_string());
    }
}

/// SW модифицирует чанки — клиент видит префикс `[SW]`.
#[wasm_bindgen_test]
#[ignore = "requires active Service Worker; run without --headless"]
async fn test_sw_stream_modification() {
    let url = "/api/proxy-stream/stream/3";
    let stream = hello_wasm02::fetch::get_stream(url).await.unwrap();
    let bytes = hello_wasm02::streams::read_all(&stream).await.unwrap();
    let text = String::from_utf8_lossy(&bytes);

    // Все непустые строки начинаются с префикса, добавленного SW.
    for line in text.lines() {
        if !line.is_empty() {
            assert!(
                line.starts_with("[SW]"),
                "строка должна иметь префикс [SW], получено: {line}"
            );
        }
    }
}

/// Заголовок `X-Generated-By` доказывает, что ответ от SW, а не от сервера.
#[wasm_bindgen_test]
#[ignore = "requires active Service Worker; run without --headless"]
async fn test_sw_marks_response() {
    let url = "/api/logs/stream";
    let resp = hello_wasm02::fetch::send_request_classified(url)
        .await
        .unwrap();
    let header = hello_wasm02::fetch::get_response_header(&resp, "x-generated-by");
    assert_eq!(header.as_deref(), Some("ServiceWorker"));
}

/// Инкрементальное чтение: колбэк вызывается на каждый чанк.
#[wasm_bindgen_test]
#[ignore = "requires active Service Worker; run without --headless"]
async fn test_sw_stream_incremental() {
    use std::cell::RefCell;
    use std::rc::Rc;

    let url = "/api/logs/stream";
    let stream = hello_wasm02::fetch::get_stream(url).await.unwrap();

    let chunks_count = Rc::new(RefCell::new(0usize));
    let counter = chunks_count.clone();

    let total = hello_wasm02::fetch::read_stream_with_callback(&stream, move |_, _| {
        *counter.borrow_mut() += 1;
    })
    .await
    .unwrap();

    assert!(total > 0);
    // SW отдаёт 10 строк — минимум 10 чанков.
    assert!(
        *chunks_count.borrow() >= 10,
        "ожидалось ≥10 чанков, получено {}",
        *chunks_count.borrow()
    );
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

**Терминал 3 — тесты (в headless, 4 теста будут ignored):**

```bash
wasm-pack test --headless --chrome --test wasm_sw_streams
```

Ожидаемый результат: `0 passed; 0 failed; 4 ignored`. Тесты помечены `#[ignore]` — они требуют активного SW.

### Ручной запуск ignored-тестов

```bash
wasm-pack test --chrome --test wasm_sw_streams -- --ignored
```

Chrome откроется в GUI. Убедитесь, что SW зарегистрирован и активирован (вкладка Application → Service Workers в DevTools).

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown --tests
```

Должно пройти без ошибок и предупреждений.

### Проверка SW вручную

1. Откройте `http://localhost:5173`.
2. Прокрутите до секции 13 (Fetch в Worker и Service Worker).
3. Нажмите «Зарегистрировать SW».
4. Перезагрузите страницу (Ctrl+R) — SW активируется.
5. Откройте DevTools → Application → Service Workers. Должен быть статус `activated`.
6. Прокрутите до секции 8 «Streams в Service Worker» и нажмите три кнопки.

## Ручная проверка демо

| Кнопка | Ожидаемый результат |
|---|---|
| «SW генерирует поток» | 10 NDJSON-строк, заголовок `X-Generated-By: ServiceWorker` |
| «SW модифицирует поток из сети» | 5 строк с префиксом `[SW]`, все начинаются с него |
| «SW: стрим с задержкой» | Прогресс-бар растёт от 0 до 10, лог обновляется по мере поступления |

Пример вывода для первой кнопки:

```
✅ Ответ от Service Worker

Заголовок X-Generated-By: ServiceWorker
Строк получено: 10

Содержимое:
{"seq":0,"timestamp":1789652274000,"level":"warn","message":"Событие номер 0"}
{"seq":1,"timestamp":1789652274200,"level":"info","message":"Событие номер 1"}
...
```

Пример вывода для второй кнопки:

```
✅ Поток модифицирован Service Worker'ом

Строк получено: 5
Все строки с префиксом [SW]: true

Содержимое:
[SW] {"message":"Ответ через 1 секунд","delay_secs":1}
[SW] {"message":"Ответ через 1 секунд","delay_secs":1}
...
```

Пример вывода для третьей кнопки:

```
✅ Поток прочитан инкрементально

Всего: 1680 байт
Время: 2000 мс

📊 Прогресс:
  получено 168 байт (чанк 168)
  получено 336 байт (чанк 168)
  ...
  получено 1680 байт (чанк 168)
```

## Что мы не меняем

- `streams.rs`, `streams_wasm.rs`, `fetch.rs`, `lib.rs` — без изменений.
- Существующие функции в `fetch.rs` уже покрывают всё, что нужно: `get_stream`, `read_all`, `read_stream_with_callback`, `send_request_classified`, `get_response_header`.
- Существующие тесты (17.1–17.7) — не затрагиваются.
- Секция 13 в `app.rs` (регистрация SW из 16.13) — не переписывается.

## Итог подглавы

Мы замкнули тему Streams API на серверной стороне браузера — Service Worker:

1. **SW перехватывает запрос** через событие `fetch` и отвечает через `event.respondWith()`.
2. **`Response` принимает `ReadableStream`** как тело. Это открывает генерацию и модификацию ответов на уровне SW.
3. **SW может сам генерировать поток** — например, NDJSON-логи, тестовые данные, offline-fallback.
4. **SW может модифицировать ответ из сети** — добавлять префиксы, логировать содержимое, распаковывать/сжимать, дополнять из кэша.
5. **Клиент не меняется.** Он получает обычный `Response` с `ReadableStream` в `body` и работает с ним через `get_stream`, `read_all`, `read_stream_with_callback`.

Ключевые уроки:

- **`Response` — универсальный контейнер.** Он принимает не только строку и `Blob`, но и `ReadableStream`.
- **`upstream.body` может быть `null`.** Проверяйте перед `getReader()` — иначе `TypeError`.
- **`Response.body` читается один раз.** Если SW прочитал его для логирования, клиенту ничего не останется. `response.clone()` до чтения — если нужно и то, и другое.
- **SW требует HTTPS или localhost.** На HTTP SW не регистрируется. В нашем проекте используется `localhost` — это разрешено.
- **Ошибки в SW не прощают.** Одна ошибка в обработчике `fetch` — и все запросы страницы падают. Оборачивайте в `try/catch`.
- **Три контекста одного потока.** Streams работают и в main thread, и в Worker, и в Service Worker. Синтаксис один и тот же, но ограничения разные: у SW нет DOM, у Worker нет `document`, у main thread нет `self`-контекста SW.

Это последняя подглава части про Streams API. Мы прошли путь от `ReadableStream` как источника данных до полноценного потокового приложения в Service Worker: конвейеры, backpressure, конвертация в обе стороны, byte streams, стриминговый JSON-парсер и, наконец, перехват HTTP с потоковыми ответами.

В следующей части книги — **работа с DOM и событиями**: формы, валидация, drag-and-drop, анимации и всё, что делает веб-приложение интерактивным.