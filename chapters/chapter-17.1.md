# 17.1. `ReadableStream`: источник данных

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← ОБНОВЛЁН: +readable_from_generator, readable_async,
│   │                                  stream_is_locked, tee_stream
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.1
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_readable.rs    ← НОВЫЙ: 7 тестов
│   └── (остальные без изменений)
│
├── package.json                    ← ОБНОВЛЁН: wasm:clean, start с очисткой
└── (корневые файлы без изменений)
```

**Краткий комментарий.** `ReadableStream` — источник данных. Он строится поверх JS-объекта с методами `start`, `pull`, `cancel`. Браузер сам решает, когда вызывать `pull`: когда очередь пуста и потребитель готов принять чанк. Поток читается один раз и одним reader'ом. Отдельный большой раздел этой подглавы — `tee()` и его реальные применения: логирование, кэширование, отладка, параллельная обработка.

## Ссылки для детального изучения

- **MDN: `ReadableStream`** — основной интерфейс: [developer.mozilla.org/ru/docs/Web/API/ReadableStream](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream)
- **MDN: Underlying source** — методы `start`, `pull`, `cancel`: [developer.mozilla.org/ru/docs/Web/API/ReadableStream/ReadableStream#parameters](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream/ReadableStream#parameters)
- **MDN: `ReadableStreamDefaultReader`** — читатель по умолчанию: [developer.mozilla.org/ru/docs/Web/API/ReadableStreamDefaultReader](https://developer.mozilla.org/ru/docs/Web/API/ReadableStreamDefaultReader)
- **MDN: `ReadableStream.tee()`** — разветвление: [developer.mozilla.org/ru/docs/Web/API/ReadableStream/tee](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream/tee)
- **MDN: `TextDecoderStream`** — встроенный преобразователь для текста: [developer.mozilla.org/ru/docs/Web/API/TextDecoderStream](https://developer.mozilla.org/ru/docs/Web/API/TextDecoderStream)
- **WHATWG Streams Standard: `tee`** — спецификация: [streams.spec.whatwg.org/#readable-stream-tee](https://streams.spec.whatwg.org/#readable-stream-tee)
- **`wasm_bindgen_futures::future_to_promise`** — Rust Future → JS Promise: [docs.rs/wasm-bindgen-futures/latest/wasm_bindgen_futures/fn.future_to_promise.html](https://docs.rs/wasm-bindgen-futures/latest/wasm_bindgen_futures/fn.future_to_promise.html)

## Теоретическая часть

### Underlying source
Источник — обычный JS-объект с методами, который браузер оборачивает в `ReadableStream`. Методы не обязательны все: можно реализовать только `pull`.

- **`start(controller)`** — вызывается один раз при создании потока. Здесь делают инициализацию: открывают файл, сбрасывают счётчики, читают первый чанк.
- **`pull(controller)`** — вызывается, когда очередь пуста и потребитель готов принять данные. Может возвращать `Promise` — тогда следующий `pull` не вызывается до разрешения промиса.
- **`cancel(reason)`** — вызывается при `reader.cancel(reason)`. Освобождает ресурсы.

### Три способа создать поток

1. **Из массива** — простейший случай, поток отдаёт элементы по одному.
2. **Из Rust-генератора** — замыкание `FnMut() -> Option<String>`, которое вызывается на каждый `pull`.
3. **Из асинхронного pull** — замыкание, возвращающее `Future<Option<String>>`. Браузер ждёт завершения Future перед следующим `pull` — это естественный backpressure.

### Чтение
- **`stream.getReader()`** — получить `ReadableStreamDefaultReader`.
- **`reader.read()`** — `Promise<{value, done}>`.
- **Цикл:** пока `done == false`, обрабатываем `value`. Когда `done == true` — поток закрыт.
- **`reader.releaseLock()`** — освободить блокировку.

### Locked
Поток locked, пока у него есть reader. Второй `getReader()` бросит `TypeError`. `stream.locked()` возвращает `true`/`false`. `releaseLock()` разблокирует.

### `tee()` — разветвление потока

- `stream.tee()` возвращает массив из двух новых `ReadableStream`.
- Каждый рукав получает **все** чанки исходного.
- Исходный поток после `tee()` считается locked — читать его напрямую нельзя.
- Внутри `tee()` **буферизует** чанки, если один рукав читается быстрее другого. Память растёт пропорционально разнице скоростей.

## `tee()` в реальных сценариях

`tee()` — не экзотика, а рабочий инструмент. Ниже четыре сценария, где он экономит код и время.

### Сценарий 1. Логирование без задержки потребителя

**Задача.** Показать данные в UI и одновременно записать их в аналитику. Наивное решение — прочитать поток в память, показать, потом отправить в аналитику. Но если поток большой (десятки мегабайт), это неприемлемо.

**Решение.** Один рукав идёт в UI, второй — в фоновую отправку. Оба читаются параллельно, память не растёт.

```rust
let (ui_stream, analytics_stream) = streams::tee_stream(&source)?;

// UI и аналитика работают одновременно.
futures::future::join(
    render_to_ui(ui_stream),
    send_to_analytics(analytics_stream),
).await;
```

UI показывает данные сразу, как только приходит первый чанк. Аналитика отправляет батчами по мере накопления. Если аналитика отстаёт — буферизуются только её чанки, UI не страдает.

### Сценарий 2. Кэширование с одновременной обработкой

**Задача.** Ответ от сервера нужно показать пользователю **и** сохранить в `CacheStorage` для офлайн-режима. Классический подход — загрузить в память, показать, потом сохранить. Для большого ответа это плохо.

**Решение.**

```rust
let (display_stream, cache_stream) = streams::tee_stream(&response_body)?;

let display_task = async move {
    let bytes = streams::read_all(&display_stream).await?;
    render(&bytes);
    Ok::<_, FetchError>(())
};

let cache_task = async move {
    let bytes = streams::read_all(&cache_stream).await?;
    save_to_cache(&bytes).await?;
    Ok::<_, FetchError>(())
};

futures::future::join(display_task, cache_task).await;
```

Пользователь видит данные мгновенно. Кэш заполняется параллельно. Если пользователь закроет вкладку до окончания — ничего страшного, при следующем открытии кэш дозаполнится.

### Сценарий 3. Отладка потока

**Задача.** Поток приходит от сервера и обрабатывается сложной цепочкой. Что-то не так — но непонятно, на каком этапе данные искажаются.

**Решение.** Разветвить поток **перед** обработкой, один рукав отправить в обработку, другой — в логирование:

```rust
let (process_stream, debug_stream) = streams::tee_stream(&source)?;

let debug_task = async move {
    let chunks = streams::read_stream_chunks(&debug_stream).await?;
    for (i, chunk) in chunks.iter().enumerate() {
        log!("chunk {}: {} bytes, first bytes: {:02X?}", i, chunk.len(), &chunk[..4.min(chunk.len())]);
    }
    Ok::<_, FetchError>(())
};

let process_task = process_stream_logic(process_stream);

futures::future::join(process_task, debug_task).await;
```

В DevTools видно каждый чанк, который пришёл от сервера. Обработка идёт как обычно. Когда баг найден — `tee()` убирается, в продакшене лишней буферизации нет.

### Сценарий 4. Два потребителя с разной скоростью

**Задача.** Один и тот же поток должен идти в UI (быстро, каждые 100 мс) и в фоновый Worker для тяжёлых вычислений (медленно, по 500 мс на чанк).

**Решение.**

```rust
let (fast_stream, slow_stream) = streams::tee_stream(&source)?;

// UI читает быстро.
let ui_task = async move {
    let mut reader = get_reader(fast_stream);
    while let Some(chunk) = read_chunk(&mut reader).await {
        render_chunk(&chunk);
    }
};

// Worker читает медленно.
let worker_task = async move {
    let mut reader = get_reader(slow_stream);
    while let Some(chunk) = read_chunk(&mut reader).await {
        process_heavy(&chunk).await;   // 500 мс
    }
};

futures::future::join(ui_task, worker_task).await;
```

`tee()` буферизует чанки для медленного рукава. UI не ждёт Worker'а. Память растёт, но только на разницу в скорости — если она невелика, всё укладывается в разумные пределы.

### Когда `tee()` **не** использовать

- **Когда можно прочитать поток один раз и передать дальше.** Если один потребитель, `tee()` только добавит буферизацию.
- **Когда потоки очень большие и скорости сильно разные.** Медленный рукав накопит гигабайты. Лучше разбить данные на части и обрабатывать их независимо.
- **Когда нужна разная обработка.** `tee()` даёт **одинаковые** данные в оба рукава. Если одному нужно распарсить JSON, а другому — распаковать gzip, порядок операций всё равно один; `tee()` не поможет.

### Альтернативы

- **`TransformStream`** — если нужно преобразовать данные перед вторым потребителем.
- **Промежуточный буфер в `IndexedDB`** — если скорости совсем разные.
- **Два независимых запроса** — если данные можно получить с сервера дважды, дешевле, чем буферизовать в памяти.

## `src/streams.rs` — новые функции

```rust
// ============================================================
// 17.1: ReadableStream — источник данных
// ============================================================

use std::cell::RefCell;
use std::collections::VecDeque;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::fetch::{classify_error, FetchError};

/// Создаёт `ReadableStream` из Rust-генератора.
///
/// `next` вызывается каждый раз, когда браузер готов принять
/// следующий чанк. Возвращает `Some(chunk)` для данных и `None`
/// для сигнала «поток закончился».
pub fn readable_from_generator<F>(next: F) -> Result<web_sys::ReadableStream, JsValue>
where
    F: FnMut() -> Option<String> + 'static,
{
    let next = Rc::new(RefCell::new(next));

    let pull_callback = {
        let next = next.clone();
        Closure::<dyn FnMut(web_sys::ReadableStreamDefaultController)>::wrap(Box::new(
            move |controller: web_sys::ReadableStreamDefaultController| {
                let value = (next.borrow_mut())();

                match value {
                    Some(chunk) => {
                        controller
                            .enqueue_with_chunk(&JsValue::from_str(&chunk))
                            .ok();
                    }
                    None => {
                        controller.close().ok();
                    }
                }
            },
        ))
    };

    let source = js_sys::Object::new();
    js_sys::Reflect::set(&source, &"pull".into(), pull_callback.as_ref().unchecked_ref()).ok();
    pull_callback.forget();

    web_sys::ReadableStream::new_with_underlying_source(&source)
}

/// Создаёт `ReadableStream` с асинхронным `pull`.
///
/// `next` — замыкание без аргументов, возвращающее `Future<Option<String>>`.
/// Каждый вызов `pull` запускает новый экземпляр Future; пока он не
/// разрешится, следующий `pull` не вызывается. Это даёт естественный
/// backpressure.
pub fn readable_async<F, Fut>(next: F) -> Result<web_sys::ReadableStream, JsValue>
where
    F: FnMut() -> Fut + 'static,
    Fut: std::future::Future<Output = Option<String>> + 'static,
{
    let next = Rc::new(RefCell::new(next));

    let pull_callback = Closure::<dyn FnMut(JsValue) -> JsValue>::wrap(Box::new(
        move |controller: JsValue| {
            // Синхронно получаем Future, сразу освобождая RefCell.
            let fut = (next.borrow_mut())();
            let controller_for_async = controller.clone();

            let async_block = async move {
                let value = fut.await;

                match value {
                    Some(chunk) => {
                        if let Ok(enqueue) = js_sys::Reflect::get(&controller_for_async, &"enqueue".into())
                            .and_then(|f| f.dyn_into::<js_sys::Function>())
                        {
                            enqueue
                                .call1(&controller_for_async, &JsValue::from_str(&chunk))
                                .ok();
                        }
                    }
                    None => {
                        if let Ok(close) = js_sys::Reflect::get(&controller_for_async, &"close".into())
                            .and_then(|f| f.dyn_into::<js_sys::Function>())
                        {
                            close.call0(&controller_for_async).ok();
                        }
                    }
                }

                Ok(JsValue::UNDEFINED)
            };

            wasm_bindgen_futures::future_to_promise(async_block).into()
        },
    ));

    let source = js_sys::Object::new();
    js_sys::Reflect::set(&source, &"pull".into(), pull_callback.as_ref().unchecked_ref()).ok();
    pull_callback.forget();

    web_sys::ReadableStream::new_with_underlying_source(&source)
}

/// Возвращает `true`, если у потока есть активный reader.
pub fn stream_is_locked(stream: &web_sys::ReadableStream) -> bool {
    stream.locked()
}

/// Разветвляет поток на два независимых.
///
/// Возвращает два `ReadableStream`, каждый из которых получит
/// все чанки исходного. Исходный поток после `tee()` становится
/// locked и больше не читается.
///
/// **Почему через Reflect.** В `web-sys 0.3.105` сигнатура `tee()`
/// может отличаться от ожидаемой. Прямой вызов JS-метода через
/// `Reflect::get` + `Function::call0` работает во всех версиях.
pub fn tee_stream(
    stream: &web_sys::ReadableStream,
) -> Result<(web_sys::ReadableStream, web_sys::ReadableStream), FetchError> {
    let fn_val = js_sys::Reflect::get(stream, &JsValue::from_str("tee"))
        .map_err(classify_error)?;

    let func: js_sys::Function = fn_val
        .dyn_into()
        .map_err(|_| FetchError::Parse("tee is not a function".into()))?;

    let result = func.call0(stream).map_err(classify_error)?;

    let array: js_sys::Array = result
        .dyn_into()
        .map_err(|_| FetchError::Parse("tee did not return an Array".into()))?;

    if array.length() != 2 {
        return Err(FetchError::Parse(format!(
            "tee returned array of length {}, expected 2",
            array.length()
        )));
    }

    let first: web_sys::ReadableStream = array
        .get(0)
        .dyn_into()
        .map_err(|_| FetchError::Parse("first branch is not a ReadableStream".into()))?;

    let second: web_sys::ReadableStream = array
        .get(1)
        .dyn_into()
        .map_err(|_| FetchError::Parse("second branch is not a ReadableStream".into()))?;

    Ok((first, second))
}
```

## `src/app.rs` — только секция 17.1

```rust
//! Демонстрация главы 17.1: `ReadableStream` как источник данных.
//!
//! Одна секция с четырьмя кнопками:
//! 1. Поток из массива.
//! 2. Поток из генератора.
//! 3. Асинхронный pull.
//! 4. Tee: два читателя.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::{dom, log, streams};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "ReadableStream: источник данных")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "ReadableStream — источник данных. Он строится поверх обычного \
         JS-объекта с методами start, pull, cancel. Браузер сам вызывает \
         pull, когда готов принять следующий чанк. Поток можно прочитать \
         только один раз и только одним reader'ом. tee() даёт два \
         независимых рукава.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 1: ReadableStream — источник данных
    // ============================================================

    let s1 = build_section(&container, "1. ReadableStream: источник")?;

    let s1_hint = dom::create_paragraph(
        "Underlying source получает только pull-колбэк. Браузер вызывает \
         его, когда внутренняя очередь пуста и потребитель готов принять \
         чанк. Когда чанки кончились, вызывается controller.close().",
    )?;
    dom::add_class(&dom::as_html(&s1_hint), "hint")?;
    dom::append(&s1, &s1_hint)?;

    let read_array_btn = dom::create_element("button")?;
    read_array_btn.set_text_content(Some("Поток из массива"));
    let read_gen_btn = dom::create_element("button")?;
    read_gen_btn.set_text_content(Some("Поток из генератора"));
    let read_async_btn = dom::create_element("button")?;
    read_async_btn.set_text_content(Some("Асинхронный pull"));
    let read_tee_btn = dom::create_element("button")?;
    read_tee_btn.set_text_content(Some("Tee: два читателя"));

    dom::append(&s1, &read_array_btn)?;
    dom::append(&s1, &read_gen_btn)?;
    dom::append(&s1, &read_async_btn)?;
    dom::append(&s1, &read_tee_btn)?;

    let s1_result = make_result(&s1)?;

    // ---- Поток из массива ----
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &read_array_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Читаем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let stream = match streams::readable_from_strings(vec![
                        "Первый чанк\n".to_string(),
                        "Второй чанк\n".to_string(),
                        "Третий чанк\n".to_string(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let locked_before = streams::stream_is_locked(&stream);

                    match streams::read_all(&stream).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            let locked_after = streams::stream_is_locked(&stream);
                            s1_result.set_text_content(Some(&format!(
                                "✅ ReadableStream из массива\n\n\
                                 Прочитано:\n{text}\n\
                                 locked до чтения: {locked_before}\n\
                                 locked после read_all: {locked_after}\n\n\
                                 read_all внутри создаёт reader через \
                                 getReader(), читает до done, затем вызывает \
                                 releaseLock(). Поэтому после чтения \
                                 поток снова разблокирован."
                            )));
                        }
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!(
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

    // ---- Поток из генератора ----
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &read_gen_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Читаем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Генератор: счётчик от 1 до 5.
                    let counter: Rc<RefCell<u32>> = Rc::new(RefCell::new(1));
                    let counter_clone = counter.clone();

                    let stream = match streams::readable_from_generator(move || {
                        let mut c = counter_clone.borrow_mut();
                        if *c > 5 {
                            None
                        } else {
                            let v = *c;
                            *c += 1;
                            Some(format!("{v}\n"))
                        }
                    }) {
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
                                "✅ ReadableStream из генератора\n\n\
                                 Прочитано:\n{text}\n\n\
                                 Генератор — замыкание FnMut() -> Option<String>. \
                                 Каждый вызов pull берёт следующее значение. \
                                 Когда генератор возвращает None — поток \
                                 закрывается через controller.close()."
                            )));
                        }
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!(
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

    // ---- Асинхронный pull ----
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &read_async_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Читаем с задержкой 300 мс..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let counter: Rc<RefCell<u32>> = Rc::new(RefCell::new(1));
                    let counter_clone = counter.clone();

                    let stream = match streams::readable_async(move || {
                        let counter = counter_clone.clone();
                        async move {
                            // Задержка 300 мс через gloo-timers.
                            gloo_timers::future::TimeoutFuture::new(300).await;

                            let mut c = counter.borrow_mut();
                            if *c > 4 {
                                None
                            } else {
                                let v = *c;
                                *c += 1;
                                Some(format!("Чанк {v} (после задержки)\n"))
                            }
                        }
                    }) {
                        Ok(s) => s,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let start = js_sys::Date::now();
                    match streams::read_all(&stream).await {
                        Ok(bytes) => {
                            let elapsed = js_sys::Date::now() - start;
                            let text = String::from_utf8_lossy(&bytes);
                            s1_result.set_text_content(Some(&format!(
                                "✅ Асинхронный pull\n\n\
                                 Прочитано:\n{text}\n\
                                 Общее время: {:.0} мс\n\n\
                                 Каждый pull запускает Future, который \
                                 ждёт 300 мс через gloo-timers. Браузер \
                                 не вызывает следующий pull, пока промис \
                                 не разрешится. Это естественный \
                                 backpressure: источник не может обогнать \
                                 потребителя.",
                                elapsed
                            )));
                        }
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!(
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

    // ---- Tee: два читателя ----
    {
        let s1_result = s1_result.clone();
        callbacks::listen(
            &read_tee_btn,
            "click",
            move |_| {
                let s1_result = s1_result.clone();
                s1_result.set_text_content(Some("Разветвляем и читаем оба..."));
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

                    let (branch_a, branch_b) = match streams::tee_stream(&source) {
                        Ok(pair) => pair,
                        Err(e) => {
                            s1_result.set_text_content(Some(&format!(
                                "❌ tee: {}",
                                e.user_message()
                            )));
                            return;
                        }
                    };

                    // Исходный поток после tee() заблокирован.
                    let source_locked = streams::stream_is_locked(&source);

                    // Читаем оба рукава параллельно.
                    let (res_a, res_b) = futures::future::join(
                        streams::read_all(&branch_a),
                        streams::read_all(&branch_b),
                    )
                    .await;

                    let text_a = match res_a {
                        Ok(bytes) => String::from_utf8_lossy(&bytes).to_string(),
                        Err(e) => format!("(ошибка: {})", e.user_message()),
                    };
                    let text_b = match res_b {
                        Ok(bytes) => String::from_utf8_lossy(&bytes).to_string(),
                        Err(e) => format!("(ошибка: {})", e.user_message()),
                    };

                    s1_result.set_text_content(Some(&format!(
                        "✅ Tee: два независимых рукава\n\n\
                         Исходный поток locked после tee(): {source_locked}\n\n\
                         Рукав A:\n{text_a}\n\
                         Рукав B:\n{text_b}\n\n\
                         Оба рукава получили все пять чанков. tee() \
                         буферизует данные внутри себя, если один \
                         рукав читается быстрее другого. Для больших \
                         потоков это может увеличить потребление \
                         памяти — компромисс ради независимости."
                    )));
                });
            },
            &closures,
        )?;
    }

    log!("Streams 17.1 demo ready");
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

## `tests/wasm_streams_readable.rs` — 7 тестов

```rust
//! Тесты для секции 17.1: ReadableStream как источник данных.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Поток из массива читается целиком.
#[wasm_bindgen_test]
async fn test_read_from_array() {
    let stream = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
        "c".to_string(),
    ])
    .unwrap();

    let bytes = hello_wasm02::streams::read_all(&stream).await.unwrap();
    assert_eq!(String::from_utf8_lossy(&bytes), "abc");
}

/// Генератор отдаёт ожидаемую последовательность.
#[wasm_bindgen_test]
async fn test_read_from_generator() {
    let counter: Rc<RefCell<u32>> = Rc::new(RefCell::new(1));
    let counter_clone = counter.clone();

    let stream = hello_wasm02::streams::readable_from_generator(move || {
        let mut c = counter_clone.borrow_mut();
        if *c > 5 {
            None
        } else {
            let v = *c;
            *c += 1;
            Some(format!("{v}"))
        }
    })
    .unwrap();

    let bytes = hello_wasm02::streams::read_all(&stream).await.unwrap();
    assert_eq!(String::from_utf8_lossy(&bytes), "12345");
}

/// Асинхронный pull отдаёт данные с задержкой, но все.
#[wasm_bindgen_test]
async fn test_read_from_async_pull() {
    let counter: Rc<RefCell<u32>> = Rc::new(RefCell::new(1));
    let counter_clone = counter.clone();

    let stream = hello_wasm02::streams::readable_async(move || {
        let counter = counter_clone.clone();
        async move {
            gloo_timers::future::TimeoutFuture::new(50).await;
            let mut c = counter.borrow_mut();
            if *c > 3 {
                None
            } else {
                let v = *c;
                *c += 1;
                Some(format!("{v}"))
            }
        }
    })
    .unwrap();

    let start = js_sys::Date::now();
    let bytes = hello_wasm02::streams::read_all(&stream).await.unwrap();
    let elapsed = js_sys::Date::now() - start;

    assert_eq!(String::from_utf8_lossy(&bytes), "123");
    // Три чанка по 50 мс каждый — минимум 150 мс.
    assert!(elapsed >= 150.0, "прошло {elapsed} мс, ожидалось ≥150");
}

/// Поток locked, пока у него есть reader.
#[wasm_bindgen_test]
async fn test_stream_locked() {
    let stream = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
    ])
    .unwrap();

    assert!(!hello_wasm02::streams::stream_is_locked(&stream));

    let reader: web_sys::ReadableStreamDefaultReader =
        stream.get_reader().dyn_into().unwrap();

    assert!(hello_wasm02::streams::stream_is_locked(&stream));

    reader.release_lock();
    assert!(!hello_wasm02::streams::stream_is_locked(&stream));
}

/// reader.cancel() прекращает чтение, следующий read возвращает done.
#[wasm_bindgen_test]
async fn test_reader_cancel() {
    let stream = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
        "c".to_string(),
    ])
    .unwrap();

    let reader: web_sys::ReadableStreamDefaultReader =
        stream.get_reader().dyn_into().unwrap();

    // Отменяем.
    let _ = wasm_bindgen_futures::JsFuture::from(reader.cancel()).await;

    // После cancel следующий read возвращает done.
    let result = wasm_bindgen_futures::JsFuture::from(reader.read()).await.unwrap();
    let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))
        .unwrap()
        .as_bool()
        .unwrap();
    assert!(done, "после cancel поток должен быть закрыт");

    reader.release_lock();
}

/// tee() — оба рукава получают все чанки.
#[wasm_bindgen_test]
async fn test_tee_both_branches_receive_data() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "x".to_string(),
        "y".to_string(),
        "z".to_string(),
    ])
    .unwrap();

    let (branch_a, branch_b) = hello_wasm02::streams::tee_stream(&source).unwrap();

    let (res_a, res_b) = futures::future::join(
        hello_wasm02::streams::read_all(&branch_a),
        hello_wasm02::streams::read_all(&branch_b),
    )
    .await;

    let text_a = String::from_utf8_lossy(&res_a.unwrap()).to_string();
    let text_b = String::from_utf8_lossy(&res_b.unwrap()).to_string();

    assert_eq!(text_a, "xyz");
    assert_eq!(text_b, "xyz");
}

/// tee() — исходный поток становится locked.
#[wasm_bindgen_test]
async fn test_tee_source_locked() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
    ])
    .unwrap();

    assert!(!hello_wasm02::streams::stream_is_locked(&source));

    let _ = hello_wasm02::streams::tee_stream(&source).unwrap();

    assert!(
        hello_wasm02::streams::stream_is_locked(&source),
        "после tee() исходный поток должен быть locked"
    );

    // Попытка создать второго reader'а на locked-потоке — TypeError.
    let get_reader = js_sys::Reflect::get(&source, &JsValue::from_str("getReader"))
        .unwrap()
        .dyn_into::<js_sys::Function>()
        .unwrap();
    let result = get_reader.call0(&source);
    assert!(result.is_err(), "второй getReader() должен упасть");
}
```

## `package.json` — обновлённые скрипты

```json
"wasm": "wasm-pack build --target web --out-dir pkg",
"wasm:dev": "wasm-pack build --dev --target web --out-dir pkg",
"wasm:clean": "cargo clean -p hello-wasm02",

"start": "npm run wasm:clean && npm run wasm:dev && vite",
```

`npm start` теперь делает три шага: сбрасывает артефакты клиента, пересобирает `.wasm`, запускает Vite. Это гарантирует, что после правок в `.rs` браузер получит свежую сборку, а не кэшированную старую.

## Команды запуска

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown --tests

# Сборка и запуск (с очисткой)
npm start

# Только тесты 17.1
wasm-pack test --headless --chrome --test wasm_streams_readable
```

Ожидаемый результат тестов: `7 passed; 0 failed`.

## Ручная проверка демо

Откройте `http://localhost:5173`:

| Кнопка | Ожидаемый результат |
|---|---|
| «Поток из массива» | Три чанка, `locked до: false`, `locked после: false` |
| «Поток из генератора» | Числа 1–5 |
| «Асинхронный pull» | Четыре чанка, общее время ≥ 1200 мс |
| «Tee: два читателя» | Оба рукава с `1 2 3 4 5`, исходный поток locked |

## Итог подглавы

Мы разобрали `ReadableStream` как источник данных и его разветвление через `tee()`.

**Основное:**
1. **Underlying source** — JS-объект с методами `start`, `pull`, `cancel`. Браузер сам вызывает `pull`, когда потребитель готов.
2. **Три способа создать поток:** из массива, из генератора, из асинхронного pull. Асинхронный pull даёт естественный backpressure — браузер ждёт завершения Future перед следующим вызовом.
3. **Чтение** через `getReader()` и цикл `{done, value}`. `releaseLock()` освобождает.
4. **Locked** — поток занят reader'ом. Второй `getReader()` бросает `TypeError`.
5. **`tee()`** — разветвляет поток на два рукава. Оба получают все чанки. Исходный поток становится locked. Буферизация — цена независимости.

**Четыре реальных сценария для `tee()`:**
- **Логирование** — UI и аналитика без задержки потребителя.
- **Кэширование** — показ пользователю и запись в `CacheStorage` параллельно.
- **Отладка** — один рукав идёт в обработку, другой в лог для анализа.
- **Разные скорости** — UI читает быстро, Worker медленно, буфер сглаживает разницу.

**Ключевые уроки:**
- **`pull` может быть асинхронным.** `future_to_promise` превращает Rust Future в JS Promise. Браузер ждёт его разрешения — это естественный backpressure.
- **`RefCell` и `await` несовместимы.** Заимствование должно завершиться до первого `.await`. В `readable_async` мы синхронно получаем Future, освобождаем `RefCell`, потом ждём.
- **`tee()` — не бесплатная операция.** Буферизация между рукавами может привести к росту памяти, если скорости сильно разные.
- **`tee()` даёт одинаковые данные.** Если одному потребителю нужно распарсить JSON, а другому — распаковать gzip, `tee()` не поможет: преобразование всё равно одно.