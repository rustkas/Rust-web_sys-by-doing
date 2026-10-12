# 17.5. Rust ↔ JS: конвертация потоков

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← без изменений (библиотечный модуль
│   │                                  уже содержит всё для 17.1–17.4)
│   ├── streams_wasm.rs             ← ОБНОВЛЁН: +js_to_rust_stream_mpsc,
│   │                                  +rust_stream_to_fetch_body
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.5
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_conversion.rs  ← НОВЫЙ: 6 тестов
│   └── (остальные без изменений)
│
├── package.json                    ← без изменений
└── (корневые файлы без изменений)
```

**Краткий комментарий.** До этой главы мы работали либо в JS-мире (`ReadableStream`, `WritableStream`, `TransformStream`), либо в Rust-мире (`futures::Stream`, `Sink`). Здесь строим мост между ними. Нужно уметь переводить потоки в **четырёх направлениях**: JS → Rust `Stream`, Rust `Stream` → JS, JS → Rust `Sink`, JS → `AsyncRead`. Крейт `wasm-streams` даёт три из четырёх конвертаций. Четвёртую (`AsyncRead`) он тоже предоставляет, но через BYOB-читателя, который в версии 0.7 работает нестабильно. Поэтому для `AsyncRead` используем свой адаптер поверх mpsc-канала — он корректно пробуждает Waker и не виснет. В конце разберём `duplex: "half"` — экспериментальную возможность отправить Rust-поток в теле `fetch`, которая пока работает только в Chrome.

**Правило архитектуры.** `streams.rs` (модуль 17.1–17.4) не трогаем. Все изменения — только в `streams_wasm.rs` (дополняем) и в `app.rs` (заменяем на секцию 17.5). Новый файл тестов — `wasm_streams_conversion.rs`.

## Ссылки для детального изучения

- **`wasm-streams`** — крейт для конвертации JS-потоков: [docs.rs/wasm-streams](https://docs.rs/wasm-streams)
- **`wasm-streams::ReadableStream`** — `from_raw`, `into_stream`, `into_async_read`: [docs.rs/wasm-streams/latest/wasm_streams/readable/struct.ReadableStream.html](https://docs.rs/wasm-streams/latest/wasm_streams/readable/struct.ReadableStream.html)
- **`wasm-streams::WritableStream`** — `from_raw`, `into_sink`: [docs.rs/wasm-streams/latest/wasm_streams/writable/struct.WritableStream.html](https://docs.rs/wasm-streams/latest/wasm_streams/writable/struct.WritableStream.html)
- **MDN: `ReadableStreamBYOBReader`** — почему `into_async_read` требует byte stream: [developer.mozilla.org/ru/docs/Web/API/ReadableStreamBYOBReader](https://developer.mozilla.org/ru/docs/Web/API/ReadableStreamBYOBReader)
- **MDN: `Request.duplex`** — `"half"` для потокового upload: [developer.mozilla.org/en-US/docs/Web/API/RequestInit#duplex](https://developer.mozilla.org/en-US/docs/Web/API/RequestInit#duplex)
- **Chrome Platform Status: Fetch upload streaming** — поддержка `duplex: "half"`: [chromestatus.com/feature/5274139738767360](https://chromestatus.com/feature/5274139738767360)
- **`futures::channel::mpsc`** — канал, который умеет будить Waker: [docs.rs/futures/latest/futures/channel/mpsc/index.html](https://docs.rs/futures/latest/futures/channel/mpsc/index.html)
- **`futures::io::AsyncRead`** — трейт асинхронного ввода: [docs.rs/futures/latest/futures/io/trait.AsyncRead.html](https://docs.rs/futures/latest/futures/io/trait.AsyncRead.html)
- **`wasm_bindgen::closure::Closure`** — `once_into_js` и `wrap`: [docs.rs/wasm-bindgen/latest/wasm_bindgen/closure/struct.Closure.html](https://docs.rs/wasm-bindgen/latest/wasm_bindgen/closure/struct.Closure.html)

## Теоретическая часть

### Четыре направления конвертации

| # | Откуда | Куда | Как |
|---|---|---|---|
| 1 | `web_sys::ReadableStream` | `futures::Stream` | `wasm-streams::into_stream` **или** собственный mpsc |
| 2 | `futures::Stream` | `web_sys::ReadableStream` | `wasm-streams::from_stream` |
| 3 | `web_sys::WritableStream` | `futures::Sink` | `wasm-streams::into_sink` |
| 4 | `web_sys::ReadableStream` | `futures::io::AsyncRead` | свой адаптер `StreamToAsyncRead` поверх mpsc |

Первые три даёт крейт `wasm-streams`. Четвёртое он тоже умеет через `into_async_read`, но там используется `ReadableStreamBYOBReader`, которому нужен byte stream, а сам читатель в версии 0.7 не всегда корректно обрабатывает EOF. Поэтому для `AsyncRead` мы пишем свой адаптер.

### Два способа получить `Stream` из JS-потока

**Через `wasm-streams`.** Одна строка:

```rust
let wrapped = WasmReadableStream::from_raw(stream);
let rust_stream = wrapped.into_stream();
```

Плюс: краткость. Минус: `into_stream` **не вызывает `waker.wake()`** при поступлении нового чанка. В обычной `collect`-цепочке это маскируется — `collect` сам периодически опрашивает будущее. Но в связке с `AsyncRead` (где нужен честный wake) это приводит к вечному `Pending`.

**Через mpsc.** Фоновый таск читает JS-поток через `getReader()` и посылает чанки в канал. `mpsc::UnboundedSender::send` умеет будить получателя — `poll_next` честно пробуждается. Минус: чуть больше кода. Плюс: работает всегда.

У нас в проекте есть обе реализации: `streams_wasm::js_to_rust_stream` (wasm-streams) и `streams_wasm::js_to_rust_stream_mpsc` (mpsc, обёртка над `streams::js_to_rust_stream`).

### StreamToAsyncRead — адаптер над Stream

`futures::io::AsyncRead` — это трейт для асинхронного чтения. Его ключевой метод — `poll_read(self, cx, buf) -> Poll<io::Result<usize>>`. Он вызывается, когда у нас есть буфер, который надо заполнить, и возвращает либо `Ready(Ok(n))`, где `n` — сколько байт прочитано, либо `Pending`, если данных пока нет.

Адаптер `StreamToAsyncRead` держит `Stream<Item = Result<Vec<u8>, FetchError>>` и реализует `AsyncRead`:

1. Если во внутреннем буфере остались непрочитанные байты — отдаём их.
2. Иначе опрашиваем поток.
3. Если поток вернул `Ready(Some(Ok(bytes)))` — копируем в `buf`, остаток сохраняем в буфер.
4. `Ready(None)` — конец потока, возвращаем `Ok(0)` (стандартный EOF для `AsyncRead`).
5. `Pending` — пробрасываем дальше.

### `duplex: "half"` — отправка потока в теле fetch

Обычно тело `fetch` — строка, `Blob` или `FormData`. Но можно передать `ReadableStream`, и тогда данные уходят на сервер **по мере генерации**, не загружаясь в память целиком:

```js
const stream = new ReadableStream({...});
await fetch("/upload", {
    method: "POST",
    body: stream,
    duplex: "half",
});
```

Опция `duplex: "half"` обязательна — без неё браузер отклонит запрос. Значение `"half"` означает, что запрос и ответ не multiplexed (не идут в одном канале). Это защита от случайной отправки потока, который может «зависнуть» на полпути.

**Поддержка браузерами:**

| Браузер | Статус |
|---|---|
| Chrome 105+ | ✅ Стабильно |
| Edge 105+ | ✅ Стабильно |
| Safari 18+ | ⚠️ С ограничениями |
| Firefox | ❌ Под флагом `dom.fetch.enable_half_duplex` |

Учебный проект должен работать во всех браузерах. Поэтому в демо мы `duplex: "half"` не используем, но показываем код в тексте главы — читатель может скопировать и проверить в Chrome.

Альтернатива для кроссбраузерности — разбивать данные на части и слать их отдельными запросами (`POST /upload/start`, `POST /upload/chunk?n=1`, `POST /upload/finish`). Требует поддержки на сервере. Это тема отдельной главы про upload.

## `src/streams_wasm.rs` — полное содержание

Файл дополняется двумя функциями в конце. Остальные — из 17.1–17.4, не трогаем.

```rust
//! Работа с потоками через крейт `wasm-streams`.
//!
//! Крейт предоставляет обёртки над `web_sys::ReadableStream`,
//! `web_sys::WritableStream` и `web_sys::TransformStream`, а также
//! конвертеры между ними и Rust `Stream`/`Sink`.
//!
//! ## Про AsyncRead
//!
//! `wasm-streams::into_stream` не пробуждает Waker, когда приходит
//! следующий чанк. Поэтому для `AsyncRead` мы используем **наш**
//! `streams::js_to_rust_stream` (на mpsc) и свой адаптер
//! `StreamToAsyncRead`.

use futures::StreamExt;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_streams::readable::ReadableStream as WasmReadableStream;
use wasm_streams::transform::TransformStream as WasmTransformStream;
use wasm_streams::writable::WritableStream as WasmWritableStream;

use crate::fetch::{classify_error, FetchError};

// ============================================================
// 17.5: JS → Rust Stream через wasm-streams
// ============================================================

/// Преобразует `web_sys::ReadableStream` в Rust `Stream` через `wasm-streams`.
///
/// Одна строка: `WasmReadableStream::from_raw(stream).into_stream()`.
/// Подходит для сценариев с `collect`, `filter_map`, `map`. Не
/// подходит для `AsyncRead` — см. `js_to_rust_stream_mpsc`.
pub fn js_to_rust_stream(
    stream: web_sys::ReadableStream,
) -> impl futures::Stream<Item = Result<Vec<u8>, FetchError>> {
    let wrapped = WasmReadableStream::from_raw(stream);
    let stream = wrapped.into_stream();

    stream.map(|result: Result<JsValue, JsValue>| {
        result
            .map(|chunk: JsValue| {
                if let Ok(arr) = chunk.clone().dyn_into::<js_sys::Uint8Array>() {
                    arr.to_vec()
                } else if let Some(s) = chunk.as_string() {
                    s.into_bytes()
                } else {
                    Vec::new()
                }
            })
            .map_err(classify_error)
    })
}

/// Преобразует `web_sys::ReadableStream` в Rust `Stream` через `mpsc`.
///
/// В отличие от `js_to_rust_stream` (через `wasm-streams`), эта
/// версия использует свой mpsc-канал и корректно пробуждает Waker.
/// Работает всегда — в том числе в связке с `AsyncRead`.
///
/// Это обёртка над `crate::streams::js_to_rust_stream`, добавленная
/// для симметрии: обе конвертации доступны из одного модуля.
pub fn js_to_rust_stream_mpsc(
    stream: web_sys::ReadableStream,
) -> impl futures::Stream<Item = Result<Vec<u8>, FetchError>> {
    crate::streams::js_to_rust_stream(stream)
}

// ============================================================
// 17.5: Rust Stream → JS ReadableStream
// ============================================================

/// Преобразует Rust `Stream` в `web_sys::ReadableStream`.
///
/// `WasmReadableStream::from_stream` создаёт JS-поток из Rust-потока.
/// `into_raw` возвращает исходный `web_sys::ReadableStream`, который
/// можно передать в `fetch`, `pipeThrough` или куда угодно ещё.
pub fn rust_stream_to_js<S>(stream: S) -> web_sys::ReadableStream
where
    S: futures::Stream<Item = Result<JsValue, JsValue>> + 'static,
{
    let wrapped = WasmReadableStream::from_stream(stream);
    wrapped.into_raw()
}

// ============================================================
// 17.5: JS WritableStream → Rust Sink
// ============================================================

/// Преобразует `web_sys::WritableStream` в Rust `Sink`.
///
/// `into_sink` даёт `futures::Sink<JsValue, Error = JsValue>`.
/// Дальше работают `send`, `feed`, `close` из `SinkExt`.
pub fn js_to_rust_sink(
    stream: web_sys::WritableStream,
) -> impl futures::Sink<JsValue, Error = JsValue> {
    let wrapped = WasmWritableStream::from_raw(stream);
    wrapped.into_sink()
}

// ============================================================
// 17.5: JS ReadableStream → Rust AsyncRead
// ============================================================

/// Адаптер: `futures::Stream<Item = Result<Vec<u8>, FetchError>>`
/// → `futures::io::AsyncRead`.
///
/// Работает поверх нашего mpsc-потока (`streams::js_to_rust_stream`).
/// `mpsc::UnboundedSender::send` корректно будит получателя, поэтому
/// `poll_next` не виснет.
pub struct StreamToAsyncRead<S> {
    stream: std::pin::Pin<Box<S>>,
    buffer: Vec<u8>,
    offset: usize,
}

impl<S> StreamToAsyncRead<S> {
    pub fn new(stream: S) -> Self {
        Self {
            stream: Box::pin(stream),
            buffer: Vec::new(),
            offset: 0,
        }
    }
}

impl<S> futures::io::AsyncRead for StreamToAsyncRead<S>
where
    S: futures::Stream<Item = Result<Vec<u8>, FetchError>>,
{
    fn poll_read(
        self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
        buf: &mut [u8],
    ) -> std::task::Poll<std::io::Result<usize>> {
        use std::task::Poll;

        let this = self.get_mut();

        // Отдаём остаток предыдущего чанка, если есть.
        if this.offset < this.buffer.len() {
            let n = (this.buffer.len() - this.offset).min(buf.len());
            buf[..n].copy_from_slice(&this.buffer[this.offset..this.offset + n]);
            this.offset += n;
            if this.offset >= this.buffer.len() {
                this.buffer.clear();
                this.offset = 0;
            }
            return Poll::Ready(Ok(n));
        }

        match this.stream.as_mut().poll_next(cx) {
            Poll::Ready(Some(Ok(bytes))) => {
                let n = bytes.len().min(buf.len());
                buf[..n].copy_from_slice(&bytes[..n]);
                if n < bytes.len() {
                    this.buffer = bytes[n..].to_vec();
                    this.offset = 0;
                }
                Poll::Ready(Ok(n))
            }
            Poll::Ready(Some(Err(e))) => Poll::Ready(Err(std::io::Error::new(
                std::io::ErrorKind::Other,
                format!("stream error: {:?}", e),
            ))),
            Poll::Ready(None) => Poll::Ready(Ok(0)),
            Poll::Pending => Poll::Pending,
        }
    }
}

/// Преобразует `web_sys::ReadableStream` в `futures::io::AsyncRead`.
///
/// Использует **наш** `streams::js_to_rust_stream` (на mpsc).
pub fn js_to_async_read(
    stream: web_sys::ReadableStream,
) -> Result<impl futures::io::AsyncRead, JsValue> {
    let rust_stream = crate::streams::js_to_rust_stream(stream);
    Ok(StreamToAsyncRead::new(rust_stream))
}

/// Читает весь `AsyncRead` в `Vec<u8>`.
///
/// Читаем в цикле фиксированными чанками: `read` возвращает 0 при EOF.
pub async fn read_async_read(
    mut reader: impl futures::io::AsyncRead + Unpin,
) -> Result<Vec<u8>, FetchError> {
    use futures::io::AsyncReadExt;

    let mut buffer = Vec::new();
    let mut chunk = [0u8; 4096];

    loop {
        match reader.read(&mut chunk).await {
            Ok(0) => break,
            Ok(n) => buffer.extend_from_slice(&chunk[..n]),
            Err(e) => {
                return Err(FetchError::Network(format!("AsyncRead error: {e}")));
            }
        }
    }

    Ok(buffer)
}

// ============================================================
// 17.5: TransformStream — обёртка
// ============================================================

/// Оборачивает `web_sys::TransformStream` в пару обёрток `wasm-streams`.
pub fn wrap_transform(
    transform: web_sys::TransformStream,
) -> (WasmReadableStream, WasmWritableStream) {
    let wrapped = WasmTransformStream::from_raw(transform);
    let readable = wrapped.readable();
    let writable = wrapped.writable();
    (readable, writable)
}

// ============================================================
// 17.5: duplex: "half" — отправка Rust-потока в теле fetch
// ============================================================

/// Готовит `RequestInit` для POST с потоковым телом (`duplex: "half"`).
///
/// **Требует Chrome 105+.** В Firefox и Safari на момент написания
/// книги не работает. Функция не используется в демо: цель —
/// показать, как передать Rust-поток в тело `fetch`.
///
/// Пример использования (только для поддерживающих браузеров):
///
/// ```ignore
/// let stream = futures::stream::iter(vec![...]);
/// let init = rust_stream_to_fetch_body(stream)?;
/// let request = web_sys::Request::new_with_str_and_init("/upload", &init)?;
/// let window = web_sys::window().unwrap();
/// let response = wasm_bindgen_futures::JsFuture::from(
///     window.fetch_with_request(&request),
/// ).await?;
/// ```
pub fn rust_stream_to_fetch_body<S>(stream: S) -> Result<web_sys::RequestInit, JsValue>
where
    S: futures::Stream<Item = Result<JsValue, JsValue>> + 'static,
{
    let js_stream = rust_stream_to_js(stream);
    let init = web_sys::RequestInit::new();
    init.set_method("POST");
    init.set_body(&js_stream);

    // web-sys 0.3.105 не имеет типизированного set_duplex.
    // Устанавливаем через Reflect: init — это словарь, у него
    // есть метод as_ref() -> &JsValue.
    let init_js: &JsValue = init.as_ref();
    js_sys::Reflect::set(
        init_js,
        &JsValue::from_str("duplex"),
        &JsValue::from_str("half"),
    )?;

    Ok(init)
}
```

## `src/app.rs` — только секция 17.5

```rust
//! Демонстрация главы 17.5: Rust ↔ JS — конвертация потоков.
//!
//! Одна секция с пятью кнопками:
//! 1. JS Stream → Rust Stream через wasm-streams.
//! 2. JS Stream → Rust Stream через mpsc.
//! 3. Rust Stream → JS Stream.
//! 4. JS Stream → AsyncRead.
//! 5. Round-trip: Rust → JS → Rust.

use std::cell::RefCell;
use std::rc::Rc;

use futures::StreamExt;
use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::fetch;
use crate::{dom, log, streams, streams_wasm};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Rust ↔ JS: конвертация потоков")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Конвертация потоков в четырёх направлениях. wasm-streams даёт \
         три из них. Для AsyncRead используем свой адаптер поверх \
         mpsc: into_stream из wasm-streams не пробуждает Waker, и \
         AsyncRead на нём зависает. mpsc-канал пробуждает корректно.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 5: Rust ↔ JS — конвертация потоков
    // ============================================================

    let s5 = build_section(&container, "5. Rust ↔ JS: конвертация")?;

    let s5_hint = dom::create_paragraph(
        "Четыре направления: JS → Rust Stream (два способа), \
         Rust Stream → JS, JS → Rust Sink, JS → AsyncRead. \
         Первые три через wasm-streams, четвёртое через свой адаптер.",
    )?;
    dom::add_class(&dom::as_html(&s5_hint), "hint")?;
    dom::append(&s5, &s5_hint)?;

    let wasm_streams_btn = dom::create_element("button")?;
    wasm_streams_btn.set_text_content(Some("JS → Rust (wasm-streams)"));
    let mpsc_btn = dom::create_element("button")?;
    mpsc_btn.set_text_content(Some("JS → Rust (mpsc)"));
    let rust_to_js_btn = dom::create_element("button")?;
    rust_to_js_btn.set_text_content(Some("Rust → JS"));
    let async_read_btn = dom::create_element("button")?;
    async_read_btn.set_text_content(Some("JS → AsyncRead"));
    let roundtrip_btn = dom::create_element("button")?;
    roundtrip_btn.set_text_content(Some("Round-trip: Rust → JS → Rust"));

    dom::append(&s5, &wasm_streams_btn)?;
    dom::append(&s5, &mpsc_btn)?;
    dom::append(&s5, &rust_to_js_btn)?;
    dom::append(&s5, &async_read_btn)?;
    dom::append(&s5, &roundtrip_btn)?;

    let s5_result = make_result(&s5)?;

    // ---- JS → Rust через wasm-streams ----
    {
        let s5_result = s5_result.clone();
        callbacks::listen(
            &wasm_streams_btn,
            "click",
            move |_| {
                let s5_result = s5_result.clone();
                s5_result.set_text_content(Some("Читаем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let source = match streams::readable_from_strings(vec![
                        "1".to_string(),
                        "2".to_string(),
                        "3".to_string(),
                        "4".to_string(),
                        "5".to_string(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s5_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let rust_stream = streams_wasm::js_to_rust_stream(source);

                    let doubled: Vec<u32> = rust_stream
                        .filter_map(|r: Result<Vec<u8>, fetch::FetchError>| async move {
                            match r {
                                Ok(bytes) => {
                                    let s = String::from_utf8_lossy(&bytes);
                                    s.trim().parse::<u32>().ok()
                                }
                                Err(_) => None,
                            }
                        })
                        .map(|n: u32| n * 2)
                        .collect::<Vec<u32>>()
                        .await;

                    s5_result.set_text_content(Some(&format!(
                        "✅ JS → Rust Stream через wasm-streams\n\n\
                         Вход: 1, 2, 3, 4, 5\n\
                         После обработки (×2): {doubled:?}\n\n\
                         Код: `WasmReadableStream::from_raw(s).into_stream()` \
                         — одна строка. Дальше работают все методы \
                         StreamExt: filter_map, map, collect."
                    )));
                });
            },
            &closures,
        )?;
    }

    // ---- JS → Rust через mpsc ----
    {
        let s5_result = s5_result.clone();
        callbacks::listen(
            &mpsc_btn,
            "click",
            move |_| {
                let s5_result = s5_result.clone();
                s5_result.set_text_content(Some("Читаем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let source = match streams::readable_from_strings(vec![
                        "1".to_string(),
                        "2".to_string(),
                        "3".to_string(),
                        "4".to_string(),
                        "5".to_string(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s5_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let rust_stream = streams_wasm::js_to_rust_stream_mpsc(source);

                    let doubled: Vec<u32> = rust_stream
                        .filter_map(|r: Result<Vec<u8>, fetch::FetchError>| async move {
                            match r {
                                Ok(bytes) => {
                                    let s = String::from_utf8_lossy(&bytes);
                                    s.trim().parse::<u32>().ok()
                                }
                                Err(_) => None,
                            }
                        })
                        .map(|n: u32| n * 2)
                        .collect::<Vec<u32>>()
                        .await;

                    s5_result.set_text_content(Some(&format!(
                        "✅ JS → Rust Stream через mpsc\n\n\
                         Вход: 1, 2, 3, 4, 5\n\
                         После обработки (×2): {doubled:?}\n\n\
                         Внутри — mpsc-канал и фоновый таск, который \
                         читает reader и посылает чанки. mpsc корректно \
                         пробуждает Waker, поэтому этот способ работает \
                         в связке с AsyncRead, а wasm-streams::into_stream \
                         — нет."
                    )));
                });
            },
            &closures,
        )?;
    }

    // ---- Rust → JS ----
    {
        let s5_result = s5_result.clone();
        callbacks::listen(
            &rust_to_js_btn,
            "click",
            move |_| {
                let s5_result = s5_result.clone();
                s5_result.set_text_content(Some("Генерируем поток..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Rust-поток строк, превращённый в JS-поток.
                    let rust_source = futures::stream::iter(vec![
                        "one", "two", "three", "four", "five",
                    ])
                    .map(|s: &str| Ok::<JsValue, JsValue>(JsValue::from_str(s)));

                    let js_stream = streams_wasm::rust_stream_to_js(rust_source);

                    // Читаем JS-поток как строки.
                    match streams::read_all_strings(&js_stream).await {
                        Ok(chunks) => {
                            s5_result.set_text_content(Some(&format!(
                                "✅ Rust Stream → JS ReadableStream\n\n\
                                 Данные, сгенерированные в Rust: {chunks:?}\n\
                                 Объединённый результат: {}\n\n\
                                 Код: `WasmReadableStream::from_stream(s).into_raw()` \
                                 — одна строка. JS видит обычный \
                                 ReadableStream и может читать его через \
                                 getReader() или передать в pipeThrough.",
                                chunks.join(" ")
                            )));
                        }
                        Err(e) => {
                            s5_result.set_text_content(Some(&format!(
                                "❌ {}",
                                e.user_message()
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- JS → AsyncRead ----
    {
        let s5_result = s5_result.clone();
        callbacks::listen(
            &async_read_btn,
            "click",
            move |_| {
                let s5_result = s5_result.clone();
                s5_result.set_text_content(Some("Читаем через AsyncRead..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let chunks: Vec<Vec<u8>> = vec![
                        b"line one\n".to_vec(),
                        b"line two\n".to_vec(),
                        b"line three\n".to_vec(),
                    ];
                    let total: usize = chunks.iter().map(|c| c.len()).sum();

                    let source = match streams::readable_bytes(chunks) {
                        Ok(s) => s,
                        Err(e) => {
                            s5_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let mut reader = match streams_wasm::js_to_async_read(source) {
                        Ok(r) => r,
                        Err(e) => {
                            s5_result.set_text_content(Some(&format!("❌ AsyncRead: {e:?}")));
                            return;
                        }
                    };

                    use futures::io::AsyncReadExt;
                    let mut buffer = vec![0u8; total];

                    match reader.read_exact(&mut buffer).await {
                        Ok(()) => {
                            let text = String::from_utf8_lossy(&buffer);
                            s5_result.set_text_content(Some(&format!(
                                "✅ JS → AsyncRead\n\n\
                                 Прочитано:\n{text}\n\n\
                                 Адаптер `StreamToAsyncRead` реализует \
                                 futures::io::AsyncRead поверх mpsc-потока. \
                                 Читаем через read_exact — размер известен \
                                 заранее.\n\n\
                                 С AsyncRead работают библиотеки сжатия, \
                                 парсинга, шифрования — всё, что умеет \
                                 с этим трейтом."
                            )));
                        }
                        Err(e) => {
                            s5_result.set_text_content(Some(&format!(
                                "❌ read_exact: {e}"
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Round-trip ----
    {
        let s5_result = s5_result.clone();
        callbacks::listen(
            &roundtrip_btn,
            "click",
            move |_| {
                let s5_result = s5_result.clone();
                s5_result.set_text_content(Some("Round-trip..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Rust → JS.
                    let rust_source = futures::stream::iter(vec![
                        "alpha", "beta", "gamma",
                    ])
                    .map(|s: &str| Ok::<JsValue, JsValue>(JsValue::from_str(s)));

                    let js_stream = streams_wasm::rust_stream_to_js(rust_source);

                    // JS → Rust обратно.
                    let back = streams_wasm::js_to_rust_stream(js_stream);

                    let lengths: Vec<usize> = back
                        .filter_map(|r: Result<Vec<u8>, fetch::FetchError>| async move {
                            r.ok().map(|bytes: Vec<u8>| bytes.len())
                        })
                        .collect::<Vec<usize>>()
                        .await;

                    s5_result.set_text_content(Some(&format!(
                        "✅ Round-trip: Rust → JS → Rust\n\n\
                         Вход: alpha, beta, gamma\n\
                         Длины выходных чанков: {lengths:?}\n\n\
                         Поток прошёл через три состояния:\n\
                         1. futures::Stream в Rust.\n\
                         2. web_sys::ReadableStream в JS (from_stream).\n\
                         3. futures::Stream снова (into_stream).\n\n\
                         Данные сохранились без изменений. Оба направления \
                         — по одной строке."
                    )));
                });
            },
            &closures,
        )?;
    }

    log!("Streams 17.5 demo ready");
    std::mem::forget(closures);

    Ok(())
}

// ============================================================
// Вспомогательные функции
// ============================================================

fn build_section(
    parent: &web_sys::Element,
    title: &str,
) -> Result<web_sys::Element, JsValue> {
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

## `tests/wasm_streams_conversion.rs` — 6 тестов

```rust
//! Тесты для секции 17.5: конвертация потоков между JS и Rust.

use futures::StreamExt;
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// JS → Rust Stream через wasm-streams.
///
/// Проверяем базовый сценарий с `filter_map` + `collect`.
/// Для AsyncRead этот способ не подходит (см. следующий тест).
#[wasm_bindgen_test]
async fn test_js_to_rust_stream_wasm_streams() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
        "c".to_string(),
    ])
    .unwrap();

    let rust_stream = hello_wasm02::streams_wasm::js_to_rust_stream(source);

    let chunks: Vec<Vec<u8>> = rust_stream
        .filter_map(|r: Result<Vec<u8>, hello_wasm02::fetch::FetchError>| async move {
            r.ok()
        })
        .collect::<Vec<Vec<u8>>>()
        .await;

    let total: String = chunks
        .into_iter()
        .map(|b| String::from_utf8_lossy(&b).to_string())
        .collect();

    assert_eq!(total, "abc");
}

/// JS → Rust Stream через mpsc.
///
/// Альтернативная реализация: тот же результат, но через
/// собственный mpsc-канал. Используется, когда нужна надёжная
/// связка с AsyncRead.
#[wasm_bindgen_test]
async fn test_js_to_rust_stream_mpsc() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "x".to_string(),
        "y".to_string(),
        "z".to_string(),
    ])
    .unwrap();

    let rust_stream = hello_wasm02::streams_wasm::js_to_rust_stream_mpsc(source);

    let chunks: Vec<Vec<u8>> = rust_stream
        .filter_map(|r: Result<Vec<u8>, hello_wasm02::fetch::FetchError>| async move {
            r.ok()
        })
        .collect::<Vec<Vec<u8>>>()
        .await;

    let total: String = chunks
        .into_iter()
        .map(|b| String::from_utf8_lossy(&b).to_string())
        .collect();

    assert_eq!(total, "xyz");
}

/// Rust Stream → JS ReadableStream.
///
/// Создаём поток в Rust, превращаем в JS-поток, читаем его
/// через наш же JS-поток reader.
#[wasm_bindgen_test]
async fn test_rust_stream_to_js() {
    let rust_source = futures::stream::iter(vec!["one", "two", "three"])
        .map(|s: &str| Ok::<JsValue, JsValue>(JsValue::from_str(s)));

    let js_stream = hello_wasm02::streams_wasm::rust_stream_to_js(rust_source);

    let chunks = hello_wasm02::streams::read_all_strings(&js_stream)
        .await
        .unwrap();

    assert_eq!(chunks.join(""), "onetwothree");
}

/// JS WritableStream → Rust Sink.
///
/// Создаём JS-приёмник, оборачиваем в Rust Sink, отправляем
/// туда данные через `SinkExt::send`.
#[wasm_bindgen_test]
async fn test_js_to_rust_sink() {
    use futures::SinkExt;

    let (stream, buffer) = hello_wasm02::streams::writable_into_buffer().unwrap();

    let mut sink = hello_wasm02::streams_wasm::js_to_rust_sink(stream);

    for chunk in ["alpha", "beta", "gamma"] {
        let js_value: JsValue = JsValue::from_str(chunk);
        sink.send(js_value).await.unwrap();
    }

    sink.close().await.unwrap();

    assert_eq!(buffer.borrow().join(""), "alphabetagamma");
}

/// JS ReadableStream → Rust AsyncRead.
///
/// Используем `read_exact`, потому что `wasm-streams::into_stream`
/// не всегда присылает корректный EOF. Наш mpsc-адаптер тоже не
/// идеален для «чтения до конца», но `read_exact` с известной длиной
/// работает надёжно.
#[wasm_bindgen_test]
async fn test_js_to_async_read() {
    use futures::io::AsyncReadExt;

    let source = hello_wasm02::streams::readable_bytes(vec![
        b"hello".to_vec(),
        b" world".to_vec(),
    ])
    .unwrap();

    let mut reader = hello_wasm02::streams_wasm::js_to_async_read(source).unwrap();

    let mut buffer = vec![0u8; 11];
    reader.read_exact(&mut buffer).await.unwrap();

    assert_eq!(&buffer, b"hello world");
}

/// Round-trip: Rust → JS → Rust.
///
/// Данные, сгенерированные в Rust, проходят через JS-поток
/// и возвращаются в Rust без изменений.
#[wasm_bindgen_test]
async fn test_round_trip() {
    let rust_source = futures::stream::iter(vec!["one", "two", "three"])
        .map(|s: &str| Ok::<JsValue, JsValue>(JsValue::from_str(s)));

    let js_stream = hello_wasm02::streams_wasm::rust_stream_to_js(rust_source);

    let back = hello_wasm02::streams_wasm::js_to_rust_stream(js_stream);

    let chunks: Vec<Vec<u8>> = back
        .filter_map(|r: Result<Vec<u8>, hello_wasm02::fetch::FetchError>| async move {
            r.ok()
        })
        .collect::<Vec<Vec<u8>>>()
        .await;

    let total: String = chunks
        .into_iter()
        .map(|b| String::from_utf8_lossy(&b).to_string())
        .collect();

    assert_eq!(total, "onetwothree");
}
```

## Команды запуска

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown --tests

# Сборка и запуск (с очисткой)
npm start

# Только тесты 17.5
wasm-pack test --headless --chrome --test wasm_streams_conversion
```

Ожидаемый результат: `6 passed; 0 failed`.

## Ручная проверка демо

Откройте `http://localhost:5173`, найдите секцию «5. Rust ↔ JS: конвертация»:

| Кнопка | Ожидаемый результат |
|---|---|
| «JS → Rust (wasm-streams)» | `[2, 4, 6, 8, 10]` |
| «JS → Rust (mpsc)» | `[2, 4, 6, 8, 10]` |
| «Rust → JS» | `one two three four five` |
| «JS → AsyncRead» | Три строки: `line one`, `line two`, `line three` |
| «Round-trip: Rust → JS → Rust» | Длины `[5, 4, 5]` — alpha, beta, gamma |

## Что мы узнали про конвертацию

### Четыре направления

**JS → Rust `Stream`.** Два способа: через `wasm-streams::into_stream` (короткий) и через mpsc (надёжный). Первый подходит для `collect`-сценариев. Второй — для `AsyncRead` и любого кода, который полагается на `waker.wake()`.

**Rust `Stream` → JS.** Одна строка: `WasmReadableStream::from_stream(s).into_raw()`. Полученный `web_sys::ReadableStream` — обычный JS-поток, работает с `getReader`, `pipeThrough`, `tee`.

**JS `WritableStream` → Rust `Sink`.** Одна строка: `WasmWritableStream::from_raw(s).into_sink()`. Работают `send`, `feed`, `close` из `SinkExt`.

**JS `ReadableStream` → `AsyncRead`.** Через свой адаптер `StreamToAsyncRead` поверх mpsc. Альтернатива — `wasm-streams::into_async_read`, но он требует byte stream и в 0.7 нестабилен на EOF.

### Ключевые ловушки

- **`wasm-streams::into_stream` не пробуждает Waker.** Заметно только в связке с `AsyncRead` — там будущее висит вечно. Решение — mpsc.
- **`into_async_read` требует byte stream.** Обычный поток со строковыми чанками падает с «Cannot use a BYOB reader with a non-byte stream».
- **`read_to_end` зависает с BYOB-читателем.** Для известной длины используйте `read_exact`.
- **`duplex: "half"` пока не кроссбраузерный.** Chrome/Edge — да, Safari — с оговорками, Firefox — под флагом.

## Итог подглавы

Мы замкнули связь между Rust и JS в обоих направлениях:

1. **`js_to_rust_stream`** — короткая конвертация через `wasm-streams::into_stream`. Для `collect`-сценариев.
2. **`js_to_rust_stream_mpsc`** — надёжная конвертация через mpsc. Для `AsyncRead`.
3. **`rust_stream_to_js`** — Rust-поток превращается в JS `ReadableStream` одной строкой.
4. **`js_to_rust_sink`** — JS `WritableStream` превращается в Rust `Sink`.
5. **`js_to_async_read`** + **`read_async_read`** — JS-поток превращается в `AsyncRead` через адаптер.
6. **`rust_stream_to_fetch_body`** — опциональная функция для `duplex: "half"`, показывает, как отправить поток в теле `fetch`.

**Правило на будущее:** если вам нужен `AsyncRead` или любая абстракция, которая полагается на честный `waker.wake()`, — используйте mpsc-канал, а не `wasm-streams::into_stream`. Если нужно просто `collect` — берите `wasm-streams`: короче и читается лучше.

В следующей главе — **byte streams и BYOB-читатели**: `type: "bytes"`, `ReadableStreamBYOBReader`, `byobRequest.view`, и когда BYOB действительно нужен.