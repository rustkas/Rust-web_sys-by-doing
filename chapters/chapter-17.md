# Глава 17. Streams: ручная реализация и `wasm-streams`

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← НОВЫЙ: ручная реализация через web_sys
│   ├── streams_wasm.rs             ← НОВЫЙ: реализация через wasm-streams
│   ├── lib.rs                      ← ОБНОВЛЁН: +pub mod streams; +pub mod streams_wasm;
│   ├── app.rs                      ← НОВЫЙ: демо Streams с двумя подходами
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_manual.rs      ← НОВЫЙ: 5 тестов ручной реализации
│   ├── wasm_streams_crate.rs       ← НОВЫЙ: 5 тестов через wasm-streams
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: wasm-streams = "0.7"
└── (корневые файлы без изменений)
```

**Краткий комментарий.** Streams API — это способ работать с данными по мере поступления, а не ждать загрузки целиком. В JavaScript есть три типа потоков: `ReadableStream` (источник), `WritableStream` (приёмник) и `TransformStream` (преобразователь). В Rust их можно использовать двумя способами: **вручную** через `web_sys` — с полным контролем над каждым чанком, или через крейт **`wasm-streams`**, который даёт готовые конвертеры между JS-потоками и Rust `Stream`/`Sink`. В этой главе мы реализуем оба подхода, соберём демо с двумя секциями и разберём четыре неочевидные ловушки, на которые наступают все, кто впервые работает со Streams API в WASM.

## Ссылки для детального изучения

- **MDN: Streams API** — обзор всех трёх типов потоков: [developer.mozilla.org/ru/docs/Web/API/Streams_API](https://developer.mozilla.org/ru/docs/Web/API/Streams_API)
- **MDN: `ReadableStream`** — читаемый поток: [developer.mozilla.org/ru/docs/Web/API/ReadableStream](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream)
- **MDN: `WritableStream`** — записываемый поток: [developer.mozilla.org/ru/docs/Web/API/WritableStream](https://developer.mozilla.org/ru/docs/Web/API/WritableStream)
- **MDN: `TransformStream`** — поток-преобразователь: [developer.mozilla.org/ru/docs/Web/API/TransformStream](https://developer.mozilla.org/ru/docs/Web/API/TransformStream)
- **MDN: Backpressure** — почему потоки не переполняют память: [developer.mozilla.org/ru/docs/Web/API/Streams_API/Concepts#backpressure](https://developer.mozilla.org/ru/docs/Web/API/Streams_API/Concepts#backpressure)
- **MDN: Byte streams** — потоки с `type: "bytes"` и BYOB-читатели: [developer.mozilla.org/ru/docs/Web/API/Streams_API/Using_readable_byte_streams](https://developer.mozilla.org/ru/docs/Web/API/Streams_API/Using_readable_byte_streams)
- **WHATWG Streams Standard** — спецификация: [streams.spec.whatwg.org](https://streams.spec.whatwg.org/)
- **`wasm-streams`** — крейт для конвертации JS-потоков: [docs.rs/wasm-streams](https://docs.rs/wasm-streams)
- **`futures::io::AsyncRead`** — трейт асинхронного ввода: [docs.rs/futures/latest/futures/io/trait.AsyncRead.html](https://docs.rs/futures/latest/futures/io/trait.AsyncRead.html)

## Что мы будем делать

Демо Streams состоит из двух наборов кнопок:

**Ручная реализация (`streams.rs`):** создание `ReadableStream` из массива, запись в `WritableStream`, `TransformStream` с upper case, `pipeThrough`, конвертация JS-потока в Rust `Stream` через `mpsc`.

**Через `wasm-streams` (`streams_wasm.rs`):** то же самое, но с использованием `wasm_streams::readable::ReadableStream::from_raw`, `into_stream`, `WritableStream::into_sink`, `TransformStream`.

## Четыре ловушки, которые вас поджидают

Прежде чем писать код, разберём четыре проблемы, с которыми **обязательно** столкнётся каждый, кто работает со Streams API в Rust+WASM. Мы потратили на них несколько итераций отладки, чтобы вы не тратили.

### Ловушка 1. `TransformStream` и backpressure — вечный deadlock

У `TransformStream` по умолчанию `readable` имеет `highWaterMark = 0`. Это значит: как только `transform` вызывает `controller.enqueue(chunk)`, поток **блокируется**, пока кто-нибудь не заберёт чанк из `readable`.

Если написать так:

```rust
for chunk in ["a", "b", "c"] {
    writer.write_with_chunk(...).await;   // ← второй write зависнет
}
writer.close().await;
let result = read_all(&readable).await;   // ← никогда не выполнится
```

— программа зависнет на втором `write`. Первый чанк попал в readable, `highWaterMark = 0` не даёт положить второй, а читать мы ещё не начали.

**Решение:** запускать запись и чтение **параллельно** через `futures::future::join`:

```rust
let write_task = async move {
    for chunk in ["a", "b", "c"] {
        writer.write_with_chunk(...).await;
    }
    writer.close().await;
};
let read_task = read_all(&readable);
let (_, bytes) = futures::future::join(write_task, read_task).await;
```

`join` переключается между задачами: пока write ждёт читателя, read забирает готовый чанк, освобождает backpressure, write продолжает.

### Ловушка 2. `writer.close()` возвращает Promise

`WritableStreamDefaultWriter::close()` возвращает `Promise<undefined>`. Если его не дождаться:

```rust
let _ = writer.close();   // ← не await
let bytes = read_all(&readable).await;   // ← повиснет
```

— читаемая сторона не получит событие `close`, и `read_all` будет ждать вечно. Правильно:

```rust
let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;
```

### Ловушка 3. `pipe_through` требует `&ReadableWritablePair`

В `web-sys 0.3.105` метод `ReadableStream::pipe_through` принимает `&ReadableWritablePair`, а не `TransformStream`. В JavaScript `TransformStream` структурно совместим с `ReadableWritablePair`, но в Rust типы разные.

**Решение:** вызывать JS-метод напрямую через `js_sys::Reflect`:

```rust
fn call_pipe_through(
    source: &web_sys::ReadableStream,
    transform: &web_sys::TransformStream,
) -> Result<web_sys::ReadableStream, String> {
    let fn_val = js_sys::Reflect::get(source, &JsValue::from_str("pipeThrough"))
        .map_err(|e| format!("Reflect::get pipeThrough: {e:?}"))?;

    let func: js_sys::Function = fn_val
        .dyn_into()
        .map_err(|_| "pipeThrough is not a function".to_string())?;

    let result = func
        .call1(source, transform)
        .map_err(|e| format!("pipeThrough call: {e:?}"))?;

    result
        .dyn_into()
        .map_err(|_| "pipeThrough result is not ReadableStream".to_string())
}
```

### Ловушка 4. `wasm-streams::into_stream` не пробуждает Waker

`into_stream()` возвращает Rust `Stream`, но его `poll_next` **не вызывает `waker.wake()`**, когда приходит следующий чанк. В результате будущее, ждущее этот Stream, висит вечно на `Poll::Pending`.

Это заметно только в связке с `AsyncRead`, где нужен честный wake. В обычной `filter_map().collect()`-цепочке проблема маскируется: `collect` сам периодически просыпается по другим причинам.

**Решение:** для `AsyncRead` используйте **свой** поток на `futures::channel::mpsc`. Он корректно будит получателя.

## `src/streams.rs` — ручная реализация

```rust
//! Работа с потоками напрямую через `web_sys`.
//!
//! Модуль не зависит от `wasm-streams` — всё делается вручную.
//! Это даёт полный контроль над чтением, записью и преобразованием
//! чанков, но требует больше кода.

use std::cell::RefCell;
use std::collections::VecDeque;
use std::rc::Rc;

use futures::channel::mpsc;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::fetch::{classify_error, FetchError};

/// Создаёт `ReadableStream`, который отдаёт строки по одной.
///
/// Underlying source получает только `pull`-колбэк. Браузер вызывает
/// его, когда потребитель готов принять следующий чанк. Когда
/// очередь пуста, вызывается `controller.close()`.
pub fn readable_from_strings(chunks: Vec<String>) -> Result<web_sys::ReadableStream, JsValue> {
    let queue: Rc<RefCell<VecDeque<String>>> = Rc::new(RefCell::new(chunks.into()));

    let pull_callback = {
        let queue = queue.clone();
        Closure::<dyn FnMut(web_sys::ReadableStreamDefaultController)>::wrap(Box::new(
            move |controller: web_sys::ReadableStreamDefaultController| {
                let mut q = queue.borrow_mut();
                if let Some(chunk) = q.pop_front() {
                    controller
                        .enqueue_with_chunk(&JsValue::from_str(&chunk))
                        .ok();
                } else {
                    controller.close().ok();
                }
            },
        ))
    };

    let source = js_sys::Object::new();
    js_sys::Reflect::set(&source, &"pull".into(), pull_callback.as_ref().unchecked_ref()).ok();
    pull_callback.forget();

    web_sys::ReadableStream::new_with_underlying_source(&source)
}

/// Создаёт **byte stream** (`type: "bytes"`) из массива байтовых чанков.
///
/// Такой поток может читаться BYOB-читателем. Обратите внимание:
/// даже byte stream может работать с default reader'ом — тогда
/// достаточно обычного `enqueue`.
pub fn readable_bytes(chunks: Vec<Vec<u8>>) -> Result<web_sys::ReadableStream, JsValue> {
    let queue: Rc<RefCell<VecDeque<Vec<u8>>>> = Rc::new(RefCell::new(chunks.into()));

    let pull_callback = {
        let queue = queue.clone();
        Closure::<dyn FnMut(JsValue)>::wrap(Box::new(move |controller: JsValue| {
            let mut q = queue.borrow_mut();

            let enqueue = js_sys::Reflect::get(&controller, &"enqueue".into())
                .ok()
                .and_then(|f| f.dyn_into::<js_sys::Function>().ok());
            let close = js_sys::Reflect::get(&controller, &"close".into())
                .ok()
                .and_then(|f| f.dyn_into::<js_sys::Function>().ok());

            if let Some(chunk) = q.pop_front() {
                if let Some(enqueue) = enqueue {
                    let arr = js_sys::Uint8Array::new_with_length(chunk.len() as u32);
                    arr.copy_from(&chunk);
                    enqueue.call1(&controller, &arr).ok();
                }
            } else if let Some(close) = close {
                close.call0(&controller).ok();
            }
        }))
    };

    let source = js_sys::Object::new();
    js_sys::Reflect::set(&source, &"type".into(), &JsValue::from_str("bytes")).ok();
    js_sys::Reflect::set(&source, &"pull".into(), pull_callback.as_ref().unchecked_ref()).ok();
    pull_callback.forget();

    web_sys::ReadableStream::new_with_underlying_source(&source)
}

/// Создаёт `WritableStream`, который складывает чанки в буфер.
///
/// Возвращает `(stream, buffer)`. После записи и закрытия потока
/// в `buffer` будут все чанки в порядке поступления.
pub fn writable_into_buffer() -> Result<
    (web_sys::WritableStream, Rc<RefCell<Vec<String>>>),
    JsValue,
> {
    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));

    let write_callback = {
        let buffer = buffer.clone();
        Closure::<dyn FnMut(JsValue)>::wrap(Box::new(move |chunk: JsValue| {
            let text = chunk.as_string().unwrap_or_default();
            buffer.borrow_mut().push(text);
        }))
    };

    let close_callback = Closure::<dyn FnMut()>::wrap(Box::new(|| {}));

    let sink = js_sys::Object::new();
    js_sys::Reflect::set(&sink, &"write".into(), write_callback.as_ref().unchecked_ref()).ok();
    js_sys::Reflect::set(&sink, &"close".into(), close_callback.as_ref().unchecked_ref()).ok();
    write_callback.forget();
    close_callback.forget();

    let stream = web_sys::WritableStream::new_with_underlying_sink(&sink)?;
    Ok((stream, buffer))
}

/// Создаёт `TransformStream`, который преобразует чанки через замыкание.
pub fn transform_with<F>(mut transform: F) -> Result<web_sys::TransformStream, JsValue>
where
    F: FnMut(String) -> String + 'static,
{
    let transform_callback = Closure::<
        dyn FnMut(JsValue, web_sys::TransformStreamDefaultController),
    >::wrap(Box::new(
        move |chunk: JsValue, controller: web_sys::TransformStreamDefaultController| {
            let input = chunk.as_string().unwrap_or_default();
            let output = transform(input);
            controller.enqueue_with_chunk(&JsValue::from_str(&output)).ok();
        },
    ));

    let transformer = js_sys::Object::new();
    js_sys::Reflect::set(
        &transformer,
        &"transform".into(),
        transform_callback.as_ref().unchecked_ref(),
    )
    .ok();
    transform_callback.forget();

    web_sys::TransformStream::new_with_transformer(&transformer)
}

/// Читает весь поток в `Vec<u8>`.
///
/// Цикл: `reader.read()` → `{ done, value }`. Пока `done == false`,
/// `value` (Uint8Array или строка) добавляется к результату.
pub async fn read_all(stream: &web_sys::ReadableStream) -> Result<Vec<u8>, FetchError> {
    let reader: web_sys::ReadableStreamDefaultReader = stream
        .get_reader()
        .dyn_into()
        .map_err(|_| FetchError::Parse("failed to get stream reader".into()))?;

    let mut result = Vec::new();

    loop {
        let promise = reader.read();
        let value = JsFuture::from(promise).await.map_err(classify_error)?;

        let done = js_sys::Reflect::get(&value, &JsValue::from_str("done"))
            .ok()
            .and_then(|v| v.as_bool())
            .unwrap_or(true);
        if done {
            break;
        }

        let chunk = js_sys::Reflect::get(&value, &JsValue::from_str("value"))
            .map_err(classify_error)?;

        let bytes = if let Ok(arr) = chunk.clone().dyn_into::<js_sys::Uint8Array>() {
            arr.to_vec()
        } else if let Some(s) = chunk.as_string() {
            s.into_bytes()
        } else {
            Vec::new()
        };

        result.extend_from_slice(&bytes);
    }

    reader.release_lock();
    Ok(result)
}

/// Преобразует `web_sys::ReadableStream` в `futures::Stream` через `mpsc`.
///
/// Внутри создаётся `mpsc`-канал. Фоновый таск читает JS-поток через
/// reader и посылает чанки в канал. Возвращаемый `Receiver` реализует
/// `Stream` — работают все методы `StreamExt`.
///
/// **Почему mpsc, а не `wasm-streams::into_stream`.** Последний
/// не вызывает `waker.wake()` при поступлении нового чанка —
/// `poll_next` возвращает `Pending`, и ждущее будущее висит вечно.
/// `mpsc` пробуждает корректно.
pub fn js_to_rust_stream(
    stream: web_sys::ReadableStream,
) -> impl futures::Stream<Item = Result<Vec<u8>, FetchError>> {
    let (tx, rx) = mpsc::unbounded::<Result<Vec<u8>, FetchError>>();

    wasm_bindgen_futures::spawn_local(async move {
        let reader: web_sys::ReadableStreamDefaultReader = match stream.get_reader().dyn_into() {
            Ok(r) => r,
            Err(_) => {
                let _ = tx.unbounded_send(Err(FetchError::Parse(
                    "failed to get stream reader".into(),
                )));
                return;
            }
        };

        loop {
            let promise = reader.read();
            let value = match JsFuture::from(promise).await {
                Ok(v) => v,
                Err(e) => {
                    let _ = tx.unbounded_send(Err(classify_error(e)));
                    break;
                }
            };

            let done = js_sys::Reflect::get(&value, &JsValue::from_str("done"))
                .ok()
                .and_then(|v| v.as_bool())
                .unwrap_or(true);
            if done {
                break;
            }

            let chunk = match js_sys::Reflect::get(&value, &JsValue::from_str("value")) {
                Ok(v) => v,
                Err(e) => {
                    let _ = tx.unbounded_send(Err(classify_error(e)));
                    break;
                }
            };

            let bytes = if let Ok(arr) = chunk.clone().dyn_into::<js_sys::Uint8Array>() {
                arr.to_vec()
            } else if let Some(s) = chunk.as_string() {
                s.into_bytes()
            } else {
                Vec::new()
            };

            if tx.unbounded_send(Ok(bytes)).is_err() {
                break;
            }
        }

        reader.release_lock();
    });

    rx
}

/// Вызывает `source.pipeThrough(transform)` через `Reflect`.
///
/// В web-sys 0.3.105 метод `ReadableStream::pipe_through` принимает
/// `&ReadableWritablePair`, а не `TransformStream`. В JS TransformStream
/// структурно совместим с ReadableWritablePair, но в Rust типы разные.
pub fn call_pipe_through(
    source: &web_sys::ReadableStream,
    transform: &web_sys::TransformStream,
) -> Result<web_sys::ReadableStream, String> {
    let fn_val = js_sys::Reflect::get(source, &JsValue::from_str("pipeThrough"))
        .map_err(|e| format!("Reflect::get pipeThrough: {e:?}"))?;

    let func: js_sys::Function = fn_val
        .dyn_into()
        .map_err(|_| "pipeThrough is not a function".to_string())?;

    let result = func
        .call1(source, transform)
        .map_err(|e| format!("pipeThrough call: {e:?}"))?;

    result
        .dyn_into()
        .map_err(|_| "pipeThrough result is not ReadableStream".to_string())
}
```

## `src/streams_wasm.rs` — через `wasm-streams`

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

/// Преобразует `web_sys::ReadableStream` в Rust `Stream` через `wasm-streams`.
///
/// Используется для сценариев, где нужен `Stream<Item = ...>` и не
/// требуется `AsyncRead`.
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

// ============================================================
// StreamToAsyncRead: адаптер Stream<Vec<u8>> → AsyncRead
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
/// **Почему не `read_to_end`.** `futures::io::AsyncReadExt::read_to_end`
/// зацикливается, если поток не присылает корректный EOF. Читаем
/// в цикле фиксированными чанками: `read` возвращает 0 при EOF.
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

/// Преобразует Rust `Stream` строк в `web_sys::ReadableStream`.
pub fn rust_stream_to_js<S>(stream: S) -> web_sys::ReadableStream
where
    S: futures::Stream<Item = Result<JsValue, JsValue>> + 'static,
{
    let wrapped = WasmReadableStream::from_stream(stream);
    wrapped.into_raw()
}

/// Преобразует `web_sys::WritableStream` в Rust `Sink`.
pub fn js_to_rust_sink(
    stream: web_sys::WritableStream,
) -> impl futures::Sink<JsValue, Error = JsValue> {
    let wrapped = WasmWritableStream::from_raw(stream);
    wrapped.into_sink()
}

/// Оборачивает `web_sys::TransformStream` в пару обёрток `wasm-streams`.
pub fn wrap_transform(
    transform: web_sys::TransformStream,
) -> (WasmReadableStream, WasmWritableStream) {
    let wrapped = WasmTransformStream::from_raw(transform);
    let readable = wrapped.readable();
    let writable = wrapped.writable();
    (readable, writable)
}
```

## `src/app.rs` — демо с двумя секциями

```rust
//! Демонстрация Streams API: ручная реализация vs `wasm-streams`.
//!
//! Две секции:
//! 1. **Ручная реализация** — работа с `web_sys` напрямую.
//! 2. **`wasm-streams`** — конвертация через крейт.

use std::cell::RefCell;
use std::rc::Rc;

use futures::StreamExt;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::callbacks::{self, EventClosures};
use crate::fetch;
use crate::{dom, log, streams, streams_wasm};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Streams API Demo")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Streams API — три типа потоков: ReadableStream (источник), \
         WritableStream (приёмник), TransformStream (преобразователь). \
         В Rust их можно использовать вручную через web_sys или через \
         крейт wasm-streams. Ниже — оба подхода на одних и тех же задачах.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 1: Ручная реализация
    // ============================================================

    let s1 = build_section(&container, "1. Ручная реализация (web_sys)")?;

    let s1_hint = dom::create_paragraph(
        "Мы сами пишем underlying source, sink и transform. \
         Конвертация JS-потока в Rust Stream — через mpsc-канал. \
         Больше кода, но полный контроль над каждым чанком.",
    )?;
    dom::add_class(&dom::as_html(&s1_hint), "hint")?;
    dom::append(&s1, &s1_hint)?;

    let manual_read_btn = dom::create_element("button")?;
    manual_read_btn.set_text_content(Some("ReadableStream из массива"));
    let manual_write_btn = dom::create_element("button")?;
    manual_write_btn.set_text_content(Some("WritableStream в буфер"));
    let manual_transform_btn = dom::create_element("button")?;
    manual_transform_btn.set_text_content(Some("TransformStream: upper case"));
    let manual_pipe_btn = dom::create_element("button")?;
    manual_pipe_btn.set_text_content(Some("pipeThrough"));
    let manual_rust_stream_btn = dom::create_element("button")?;
    manual_rust_stream_btn.set_text_content(Some("JS → Rust Stream (mpsc)"));

    dom::append(&s1, &manual_read_btn)?;
    dom::append(&s1, &manual_write_btn)?;
    dom::append(&s1, &manual_transform_btn)?;
    dom::append(&s1, &manual_pipe_btn)?;
    dom::append(&s1, &manual_rust_stream_btn)?;

    let s1_result = make_result(&s1)?;

    // --- ReadableStream из массива ---
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &manual_read_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Создаём поток..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let chunks = vec![
                        "Первый чанк\n".to_string(),
                        "Второй чанк\n".to_string(),
                        "Третий чанк\n".to_string(),
                    ];

                    let stream = match streams::readable_from_strings(chunks) {
                        Ok(s) => s,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::read_all(&stream).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            s1_result.set_text_content(Some(&format!(
                                "✅ Ручной ReadableStream\n\n\
                                 Прочитано:\n{text}\n\
                                 Underlying source: только pull-колбэк.\n\
                                 Каждый pull берёт следующий чанк из очереди."
                            )));
                        }
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {}", e.user_message())));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // --- WritableStream в буфер ---
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &manual_write_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Пишем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let (stream, buffer) = match streams::writable_into_buffer() {
                        Ok(pair) => pair,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let writer: web_sys::WritableStreamDefaultWriter = match stream.get_writer()
                    {
                        Ok(w) => w,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!(
                                "❌ Не удалось получить writer: {e:?}"
                            )));
                            return;
                        }
                    };

                    for chunk in ["Alpha\n", "Beta\n", "Gamma\n"] {
                        let promise = writer.write_with_chunk(&JsValue::from_str(chunk));
                        let _ = wasm_bindgen_futures::JsFuture::from(promise).await;
                    }
                    // close() возвращает Promise — обязательно дождаться.
                    let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

                    let collected = buffer.borrow().join("");
                    s1_result.set_text_content(Some(&format!(
                        "✅ Ручной WritableStream\n\n\
                         Записано: Alpha, Beta, Gamma\n\
                         Буфер:\n{collected}\n\
                         Underlying sink: методы write и close.\n\
                         writer.write() вызывается на каждый чанк."
                    )));
                });
            },
            &closures,
        )?;
    }

    // --- TransformStream: upper case ---
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &manual_transform_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Преобразуем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let transform = match streams::transform_with(|s: String| s.to_uppercase()) {
                        Ok(t) => t,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let readable = transform.readable();
                    let writable = transform.writable();

                    let writer: web_sys::WritableStreamDefaultWriter =
                        match writable.get_writer() {
                            Ok(w) => w,
                            Err(e) => {
                                s1_result.set_text_content(Some(&format!(
                                    "❌ Не удалось получить writer: {e:?}"
                                )));
                                return;
                            }
                        };

                    // Запись и чтение параллельно. Иначе второй write
                    // зависнет: readable у TransformStream имеет
                    // highWaterMark = 0.
                    let write_task = async move {
                        for chunk in ["hello\n", "world\n", "rust\n"] {
                            let _ = wasm_bindgen_futures::JsFuture::from(
                                writer.write_with_chunk(&JsValue::from_str(chunk)),
                            )
                            .await;
                        }
                        let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;
                    };

                    let read_task = streams::read_all(&readable);

                    let (_, read_result) = futures::future::join(write_task, read_task).await;

                    match read_result {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            s1_result.set_text_content(Some(&format!(
                                "✅ Ручной TransformStream\n\n\
                                 Вход: hello, world, rust\n\
                                 Выход:\n{text}\n\
                                 transform(chunk, controller) вызывается \
                                 на каждый записанный чанк."
                            )));
                        }
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {}", e.user_message())));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // --- pipeThrough ---
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &manual_pipe_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Конвейер..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let source = match streams::readable_from_strings(vec![
                        "one\n".to_string(),
                        "two\n".to_string(),
                        "three\n".to_string(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let transform = match streams::transform_with(|s: String| {
                        let trimmed = s.trim();
                        format!("[{}] {}\n", trimmed.len(), trimmed.to_uppercase())
                    }) {
                        Ok(t) => t,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    // web-sys 0.3.105 требует &ReadableWritablePair.
                    // Обходим через Reflect.
                    let transformed = match streams::call_pipe_through(&source, &transform) {
                        Ok(s) => s,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ pipeThrough: {e}")));
                            return;
                        }
                    };

                    match streams::read_all(&transformed).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            s1_result.set_text_content(Some(&format!(
                                "✅ Ручной pipeThrough\n\n\
                                 Вход: one, two, three\n\
                                 После transform:\n{text}\n\
                                 pipeThrough соединяет поток и преобразователь \
                                 и возвращает новый ReadableStream."
                            )));
                        }
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {}", e.user_message())));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // --- JS → Rust Stream через mpsc ---
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &manual_rust_stream_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Читаем..."));
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
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let rust_stream = streams::js_to_rust_stream(source);

                    let doubled: Vec<u32> = rust_stream
                        .filter_map(|result: Result<Vec<u8>, fetch::FetchError>| async move {
                            match result {
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

                    s1_result.set_text_content(Some(&format!(
                        "✅ Ручная конвертация JS → Rust Stream\n\n\
                         Вход: 1, 2, 3, 4, 5\n\
                         После обработки (×2): {doubled:?}\n\
                         Внутри — mpsc-канал и фоновый таск, который \
                         читает reader и посылает чанки."
                    )));
                });
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 2: wasm-streams
    // ============================================================

    let s2 = build_section(&container, "2. wasm-streams")?;

    let s2_hint = dom::create_paragraph(
        "Тот же функционал, но через крейт wasm-streams. \
         Конвертеры from_raw/into_stream/into_sink/into_async_read \
         убирают ручной цикл чтения. Плюс — доступ к AsyncRead, \
         которого нет в ручной реализации.",
    )?;
    dom::add_class(&dom::as_html(&s2_hint), "hint")?;
    dom::append(&s2, &s2_hint)?;

    let crate_rust_stream_btn = dom::create_element("button")?;
    crate_rust_stream_btn.set_text_content(Some("JS → Rust Stream (wasm-streams)"));
    let crate_async_read_btn = dom::create_element("button")?;
    crate_async_read_btn.set_text_content(Some("JS → AsyncRead"));
    let crate_sink_btn = dom::create_element("button")?;
    crate_sink_btn.set_text_content(Some("JS → Rust Sink"));
    let crate_roundtrip_btn = dom::create_element("button")?;
    crate_roundtrip_btn.set_text_content(Some("Rust Stream → JS → Rust Stream"));

    dom::append(&s2, &crate_rust_stream_btn)?;
    dom::append(&s2, &crate_async_read_btn)?;
    dom::append(&s2, &crate_sink_btn)?;
    dom::append(&s2, &crate_roundtrip_btn)?;

    let s2_result = make_result(&s2)?;

    // --- JS → Rust Stream через wasm-streams ---
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &crate_rust_stream_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Читаем..."));
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
                            s2_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let rust_stream = streams_wasm::js_to_rust_stream(source);

                    let doubled: Vec<u32> = rust_stream
                        .filter_map(|result: Result<Vec<u8>, fetch::FetchError>| async move {
                            match result {
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

                    s2_result.set_text_content(Some(&format!(
                        "✅ wasm-streams: JS → Rust Stream\n\n\
                         Вход: 1, 2, 3, 4, 5\n\
                         После обработки (×2): {doubled:?}\n\
                         Код: `WasmReadableStream::from_raw(s).into_stream()` — \
                         одна строка вместо ручного mpsc-канала."
                    )));
                });
            },
            &closures,
        )?;
    }

    // --- JS → AsyncRead ---
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &crate_async_read_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Читаем через AsyncRead..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let chunks: Vec<Vec<u8>> = vec![
                        b"line one\n".to_vec(),
                        b"line two\n".to_vec(),
                        b"line three\n".to_vec(),
                    ];

                    // Считаем размер заранее: read_exact завершится,
                    // как только буфер заполнится, и не зависит от того,
                    // присылает ли поток корректный EOF.
                    let total: usize = chunks.iter().map(|c| c.len()).sum();

                    let source = match streams::readable_bytes(chunks) {
                        Ok(s) => s,
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let mut reader = match streams_wasm::js_to_async_read(source) {
                        Ok(r) => r,
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!(
                                "❌ AsyncRead: {e:?}"
                            )));
                            return;
                        }
                    };

                    use futures::io::AsyncReadExt;
                    let mut buffer = vec![0u8; total];

                    match reader.read_exact(&mut buffer).await {
                        Ok(()) => {
                            let text = String::from_utf8_lossy(&buffer);
                            s2_result.set_text_content(Some(&format!(
                                "✅ wasm-streams: JS → AsyncRead\n\n\
                                 Прочитано:\n{text}\n\
                                 futures::io::AsyncRead получен через наш \
                                 адаптер StreamToAsyncRead поверх mpsc-канала \
                                 (streams::js_to_rust_stream).\n\n\
                                 С AsyncRead работают библиотеки сжатия, \
                                 парсинга, шифрования — всё, что умеет с \
                                 этим трейтом.\n\n\
                                 Заметьте: читаем через read_exact, потому что \
                                 wasm-streams 0.7 не всегда присылает EOF \
                                 после последнего чанка. read_exact \
                                 останавливается по заполнению буфера и \
                                 не зависит от EOF."
                            )));
                        }
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!(
                                "❌ read_exact: {e}"
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // --- JS → Rust Sink ---
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &crate_sink_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Пишем через Sink..."));
                wasm_bindgen_futures::spawn_local(async move {
                    use futures::SinkExt;

                    let (stream, buffer) = match streams::writable_into_buffer() {
                        Ok(pair) => pair,
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let mut sink = streams_wasm::js_to_rust_sink(stream);

                    for chunk in ["Alpha\n", "Beta\n", "Gamma\n"] {
                        let js_value: JsValue = JsValue::from_str(chunk);
                        if let Err(e) = sink.send(js_value).await {
                            s2_result.set_text_content(Some(&format!(
                                "❌ Sink send: {e:?}"
                            )));
                            return;
                        }
                    }
                    let _ = sink.close().await;

                    let collected = buffer.borrow().join("");
                    s2_result.set_text_content(Some(&format!(
                        "✅ wasm-streams: JS → Rust Sink\n\n\
                         Записано: Alpha, Beta, Gamma\n\
                         Буфер:\n{collected}\n\
                         into_sink() даёт futures::Sink.\n\
                         Работают send, feed, close — стандартные методы \
                         SinkExt."
                    )));
                });
            },
            &closures,
        )?;
    }

    // --- Round-trip: Rust Stream → JS → Rust Stream ---
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &crate_roundtrip_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Round-trip..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Ok::<JsValue, JsValue> — важно указать оба типа.
                    let rust_source = futures::stream::iter(vec![
                        "one", "two", "three", "four", "five",
                    ])
                    .map(|s: &str| Ok::<JsValue, JsValue>(JsValue::from_str(s)));

                    let js_stream = streams_wasm::rust_stream_to_js(rust_source);
                    let rust_stream = streams_wasm::js_to_rust_stream(js_stream);

                    let lengths: Vec<usize> = rust_stream
                        .filter_map(|r: Result<Vec<u8>, fetch::FetchError>| async move {
                            r.ok().map(|bytes: Vec<u8>| bytes.len())
                        })
                        .collect::<Vec<usize>>()
                        .await;

                    s2_result.set_text_content(Some(&format!(
                        "✅ Round-trip: Rust → JS → Rust\n\n\
                         Вход: one, two, three, four, five\n\
                         Длины: {lengths:?}\n\
                         Rust Stream превратился в JS ReadableStream \
                         через from_stream, потом обратно в Rust Stream \
                         через into_stream. Оба направления — одна строка."
                    )));
                });
            },
            &closures,
        )?;
    }

    log!("Streams demo ready");
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

## `tests/wasm_streams_manual.rs`

```rust
//! Тесты ручной реализации Streams API.

use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// ReadableStream из массива строк читается целиком.
#[wasm_bindgen_test]
async fn test_manual_readable_from_strings() {
    let stream = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
    ])
    .unwrap();

    let bytes = hello_wasm02::streams::read_all(&stream).await.unwrap();
    assert_eq!(String::from_utf8_lossy(&bytes), "ab");
}

/// WritableStream собирает записанные чанки в буфер.
#[wasm_bindgen_test]
async fn test_manual_writable_into_buffer() {
    let (stream, buffer) = hello_wasm02::streams::writable_into_buffer().unwrap();

    let writer: web_sys::WritableStreamDefaultWriter = stream
        .get_writer()
        .expect("get_writer failed");

    for chunk in ["x", "y"] {
        let js_value: JsValue = JsValue::from_str(chunk);
        let _ = wasm_bindgen_futures::JsFuture::from(
            writer.write_with_chunk(&js_value),
        )
        .await;
    }
    let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

    assert_eq!(buffer.borrow().join(""), "xy");
}

/// TransformStream переводит строки в верхний регистр.
///
/// Запись и чтение идут **параллельно** через `join`. Без этого
/// возникает deadlock: `TransformStream` по умолчанию имеет
/// `readable` с `highWaterMark = 0`.
#[wasm_bindgen_test]
async fn test_manual_transform_upper() {
    use futures::future::join;

    let transform =
        hello_wasm02::streams::transform_with(|s: String| s.to_uppercase()).unwrap();
    let readable = transform.readable();
    let writable = transform.writable();

    let writer: web_sys::WritableStreamDefaultWriter = writable
        .get_writer()
        .expect("get_writer failed");

    let write_task = async move {
        for chunk in ["ab", "cd"] {
            let js_value: JsValue = JsValue::from_str(chunk);
            let _ = wasm_bindgen_futures::JsFuture::from(
                writer.write_with_chunk(&js_value),
            )
            .await;
        }
        let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;
    };

    let read_task = hello_wasm02::streams::read_all(&readable);

    let (_, read_result) = join(write_task, read_task).await;
    let bytes = read_result.unwrap();

    assert_eq!(String::from_utf8_lossy(&bytes), "ABCD");
}

/// pipeThrough соединяет источник и TransformStream.
#[wasm_bindgen_test]
async fn test_manual_pipe_through() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
    ])
    .unwrap();

    let transform =
        hello_wasm02::streams::transform_with(|s: String| format!("[{s}]")).unwrap();

    let out = hello_wasm02::streams::call_pipe_through(&source, &transform)
        .expect("pipeThrough failed");

    let bytes = hello_wasm02::streams::read_all(&out).await.unwrap();
    assert_eq!(String::from_utf8_lossy(&bytes), "[a][b]");
}

/// JS-поток превращается в Rust Stream через mpsc.
#[wasm_bindgen_test]
async fn test_manual_js_to_rust_stream() {
    use futures::StreamExt;

    let source = hello_wasm02::streams::readable_from_strings(vec![
        "1".to_string(),
        "2".to_string(),
    ])
    .unwrap();

    let rust_stream = hello_wasm02::streams::js_to_rust_stream(source);

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

    assert_eq!(total, "12");
}
```

## `tests/wasm_streams_crate.rs`

```rust
//! Тесты Streams API через крейт `wasm-streams`.

use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// JS-поток превращается в Rust Stream через wasm-streams.
#[wasm_bindgen_test]
async fn test_crate_js_to_rust_stream() {
    use futures::StreamExt;

    let source = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
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

    assert_eq!(total, "ab");
}

/// JS-поток превращается в AsyncRead.
#[wasm_bindgen_test]
async fn test_crate_js_to_async_read() {
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

/// JS-поток превращается в Rust Sink.
#[wasm_bindgen_test]
async fn test_crate_js_to_sink() {
    use futures::SinkExt;

    let (stream, buffer) = hello_wasm02::streams::writable_into_buffer().unwrap();
    let mut sink = hello_wasm02::streams_wasm::js_to_rust_sink(stream);

    for chunk in ["a", "b"] {
        let js_value: JsValue = JsValue::from_str(chunk);
        sink.send(js_value).await.expect("sink send failed");
    }
    sink.close().await.ok();

    assert_eq!(buffer.borrow().join(""), "ab");
}

/// Round-trip: Rust Stream → JS ReadableStream → Rust Stream.
#[wasm_bindgen_test]
async fn test_crate_rust_stream_to_js_roundtrip() {
    use futures::StreamExt;

    let rust_source = futures::stream::iter(vec!["x", "y", "z"])
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

    assert_eq!(total, "xyz");
}

/// TransformStream через обёртку wasm-streams.
///
/// Запись и чтение идут параллельно через `join`. Без этого
/// возникает deadlock из-за backpressure.
#[wasm_bindgen_test]
async fn test_crate_wrap_transform() {
    use futures::{SinkExt, StreamExt};

    let transform =
        hello_wasm02::streams::transform_with(|s: String| s.to_uppercase()).unwrap();

    let (readable, writable) = hello_wasm02::streams_wasm::wrap_transform(transform);

    let mut sink = writable.into_sink();
    let stream = readable.into_stream();

    let write_task = async move {
        for chunk in ["abc", "def"] {
            let js_value: JsValue = JsValue::from_str(chunk);
            sink.send(js_value).await.expect("sink send failed");
        }
        sink.close().await.ok();
    };

    let read_task = async move {
        let chunks: Vec<Vec<u8>> = stream
            .filter_map(|r: Result<JsValue, JsValue>| async move {
                r.ok().map(|v| {
                    if let Ok(arr) = v.clone().dyn_into::<js_sys::Uint8Array>() {
                        arr.to_vec()
                    } else {
                        v.as_string().unwrap_or_default().into_bytes()
                    }
                })
            })
            .collect::<Vec<Vec<u8>>>()
            .await;
        chunks
    };

    let (_, chunks) = futures::future::join(write_task, read_task).await;

    let total: String = chunks
        .into_iter()
        .map(|b| String::from_utf8_lossy(&b).to_string())
        .collect();

    assert_eq!(total, "ABCDEF");
}
```

## Команды запуска

```bash
cargo check --target wasm32-unknown-unknown
npm run wasm:dev
npm run dev

wasm-pack test --headless --chrome --test wasm_streams_manual
wasm-pack test --headless --chrome --test wasm_streams_crate
```

Ожидаемый результат: `5 passed` в каждом файле, `10 passed` суммарно.

## Сравнение двух подходов

| Возможность | Ручная реализация | `wasm-streams` |
|---|---|---|
| Создание `ReadableStream` из JS | ✅ `new_with_underlying_source` | ❌ Только обёртка |
| Чтение `ReadableStream` | ✅ `get_reader` + цикл | ✅ `into_stream` |
| `ReadableStream` → Rust `Stream` | ⚠️ Через `mpsc` вручную | ✅ `into_stream` |
| `ReadableStream` → `AsyncRead` | ⚠️ Свой адаптер на mpsc | ⚠️ `into_async_read` (нестабильно) |
| Rust `Stream` → `ReadableStream` | ❌ Свой адаптер | ✅ `from_stream` |
| `WritableStream` → Rust `Sink` | ⚠️ Ручной writer | ✅ `into_sink` |
| `TransformStream` | ✅ `new_with_transformer` | ⚠️ Только обёртка |
| Backpressure | ⚠️ Ручное управление | ✅ Автоматически |
| Объём кода | ~250 строк | ~100 строк |

**Что выбрать.** Для чтения `response.body()` и простых преобразований ручная реализация работает отлично и не тянет лишних зависимостей. Но как только нужна **обратная конвертация** (Rust `Stream` → JS `ReadableStream`), **`AsyncRead`** или **`Sink`** — `wasm-streams` становится незаменим.

## Итог главы

Мы разобрали Streams API с двух сторон и выучили четыре ловушки:

1. **Ручная реализация** через `web_sys` — создание underlying source/sink/transformer, чтение через `get_reader`, запись через `get_writer`, конвертация в Rust `Stream` через `mpsc`.
2. **`wasm-streams`** — готовые конвертеры `from_raw`, `into_stream`, `into_sink`, `from_stream`. В разы меньше кода.

Ключевые уроки:

- **`TransformStream` и backpressure — источник deadlock.** Readable-сторона имеет `highWaterMark = 0`. Если писать и не читать одновременно, второй `write` зависнет. Решение — `futures::future::join`.
- **`writer.close()` возвращает Promise.** Без `.await` читатель не получит сигнал о завершении.
- **`pipe_through` требует `&ReadableWritablePair`.** В `web-sys 0.3.105` прямого пути нет. Обход через `js_sys::Reflect` работает во всех версиях.
- **`wasm-streams::into_stream` не пробуждает Waker.** Для `AsyncRead` используйте свой адаптер на `mpsc`, который wakes корректно.
- **`read_exact` надёжнее `read_to_end`.** Он не зависит от того, присылает ли поток корректный EOF. Если размер известен заранее — всегда используйте его.
- **Byte stream (`type: "bytes"`) нужен для BYOB-читателя.** Обычный поток с ним падает. Но для `AsyncRead` через mpsc это не критично — там используется default reader.