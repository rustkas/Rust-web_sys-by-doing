# 17.4. Backpressure и очереди

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← ОБНОВЛЁН: +5 функций для стратегий
│   ├── streams_wasm.rs             ← без изменений
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.4
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_backpressure.rs ← НОВЫЙ: 4 теста
│   └── (остальные без изменений)
│
├── package.json                    ← без изменений
└── (корневые файлы без изменений)
```

**Краткий комментарий.** Backpressure — самая абстрактная часть Streams API. Это механизм, который не даёт быстрому источнику захлестнуть медленного потребителя. Работает через две величины: `highWaterMark` (порог очереди) и `desiredSize` (сколько ещё можно принять). Когда `desiredSize` становится отрицательным, источник обязан притормозить. В `ReadableStream` это выражается в том, что браузер перестаёт вызывать `pull`. В `WritableStream` — в том, что `writer.ready` не разрешается. В `TransformStream` — в том, что `enqueue` блокируется. Разберём, как этим управлять через `QueuingStrategy`, и покажем, как `desiredSize` меняется в реальном времени.

По ходу главы мы упрёмся в три специфичные для `web-sys 0.3.105` и Streams API ловушки: `desired_size()` у контроллера и writer'а возвращают разные типы, `QueuingStrategy` — интерфейс, а не класс, и byte stream не принимает `size()` в стратегии. Плюс четвёртая — rust-analyzer с `if/else` в tail-позиции. Каждую разберём.

**Правило архитектуры.** Библиотечные модули (`streams.rs`, `streams_wasm.rs`, `fetch.rs`, `lib.rs`) не переписываются — только дополняются. Существующие функции для 17.1–17.3 и 17.5 остаются нетронутыми. Эта глава добавляет в `streams.rs` пять новых функций, полностью заменяет `app.rs` и создаёт новый файл тестов.

## Ссылки для детального изучения

- **MDN: Backpressure** — концепция и зачем она нужна: [developer.mozilla.org/ru/docs/Web/API/Streams_API/Concepts#backpressure](https://developer.mozilla.org/ru/docs/Web/API/Streams_API/Concepts#backpressure)
- **MDN: `QueuingStrategy`** — интерфейс стратегии очереди: [developer.mozilla.org/ru/docs/Web/API/QueuingStrategy](https://developer.mozilla.org/ru/docs/Web/API/QueuingStrategy)
- **MDN: `CountQueuingStrategy`** — каждый чанк за 1: [developer.mozilla.org/ru/docs/Web/API/CountQueuingStrategy](https://developer.mozilla.org/ru/docs/Web/API/CountQueuingStrategy)
- **MDN: `ByteLengthQueuingStrategy`** — размер в байтах: [developer.mozilla.org/ru/docs/Web/API/ByteLengthQueuingStrategy](https://developer.mozilla.org/ru/docs/Web/API/ByteLengthQueuingStrategy)
- **MDN: `ReadableStreamDefaultController.desiredSize`** — сколько ещё можно принять: [developer.mozilla.org/ru/docs/Web/API/ReadableStreamDefaultController/desiredSize](https://developer.mozilla.org/ru/docs/Web/API/ReadableStreamDefaultController/desiredSize)
- **MDN: `WritableStreamDefaultWriter.ready`** — Promise, разрешающийся, когда можно писать: [developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/ready](https://developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/ready)
- **MDN: `WritableStreamDefaultWriter.desiredSize`** — размер очереди приёмника: [developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/desiredSize](https://developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/desiredSize)
- **WHATWG Streams Standard: Queuing** — модель очередей: [streams.spec.whatwg.org/#queuing-strategies](https://streams.spec.whatwg.org/#queuing-strategies)
- **MDN: Readable byte streams** — почему byte stream не принимает `size()`: [developer.mozilla.org/ru/docs/Web/API/Streams_API/Using_readable_byte_streams](https://developer.mozilla.org/ru/docs/Web/API/Streams_API/Using_readable_byte_streams)

## Теоретическая часть

### `desiredSize` — сколько ещё поместится
Число, которое показывает, сколько данных потребитель готов принять прямо сейчас.

- **Положительное** — есть место, можно отправлять.
- **Ноль** — очередь заполнена ровно до `highWaterMark`, но поток ещё не блокируется.
- **Отрицательное** — очередь переполнена, источник должен подождать.

`desiredSize` уменьшается на «размер» каждого чанка при попадании в очередь и увеличивается, когда чанк уходит потребителю. Как считается «размер» — определяется стратегией.

### `highWaterMark` — порог очереди
Число, при котором `desiredSize` становится нулём. По умолчанию:

| Поток | HWM по умолчанию | Что это значит |
|---|---|---|
| `ReadableStream` | 1 | Пока очередь пуста — `pull` вызывается. |
| `WritableStream` | 1 | Пока очередь пуста — `writer.ready` разрешён. |
| `TransformStream` (readable) | 0 | `enqueue` блокирует следующий `write`. |

Отсюда и deadlock в 17.3: писать всё сразу, потом читать — нельзя, потому что у readable HWM=0.

### `QueuingStrategy` — правило подсчёта
Две стандартные стратегии:

**`CountQueuingStrategy({ highWaterMark: N })`** — каждый чанк считается за 1. `desiredSize` уменьшается на 1 при `enqueue`. Простая и универсальная.

**`ByteLengthQueuingStrategy({ highWaterMark: N })`** — размер чанка измеряется в байтах. Требует, чтобы у чанка было свойство `byteLength` (как у `Uint8Array`). Используется для бинарных данных.

Пример: HWM=8, чанк 4 байта. После двух `enqueue` `desiredSize` станет 0. После третьего — отрицательным.

### Поведение при переполнении

| Поток | Что происходит |
|---|---|
| `ReadableStream` | Браузер перестаёт вызывать `pull`, пока потребитель не заберёт чанк. |
| `WritableStream` | `writer.ready` не разрешается, пока не появится место. |
| `TransformStream` | `enqueue` блокирует следующий `write`. |

### Как потребитель узнаёт, что данные готовы
- **`writer.ready`** — Promise. `await writer.ready` перед `write`, если нужен точный контроль.
- **`reader.read()`** — сам ждёт, если данных нет. Не нужно проверять `desiredSize`.
- **`desiredSize`** — читается синхронно. Используется для собственных решений: «писать сейчас или подождать».

### Четыре ловушки, в которые мы попадём

**1. `QueuingStrategy` — это интерфейс, а не класс.** `dyn_into::<web_sys::QueuingStrategy>()` всегда падает, потому что в JS нет класса `QueuingStrategy`. Приводить надо через `unchecked_into`, потому что мы сами создали объект и знаем его форму.

**2. `Function::construct` отсутствует в `js-sys 0.3.105`.** Вызов конструктора идёт через `js_sys::Reflect::construct(ctor.as_ref(), &args)`.

**3. Byte stream не принимает `size()` в стратегии.** Спецификация Streams запрещает `ByteLengthQueuingStrategy` вместе с `type: "bytes"` — размер каждого чанка вычисляется автоматически как `byteLength`. Для byte stream нужна простая стратегия `{ highWaterMark: N }` без `size`. А `ByteLengthQueuingStrategy` работает только с **обычными** потоками, но там она всё равно бесполезна: у строк нет `byteLength`.

Плюс расхождение в типах: `ReadableStreamDefaultController::desired_size()` возвращает `Option<f64>`, а `WritableStreamDefaultWriter::desired_size()` — `Result<Option<f64>, JsValue>`. У первого `.ok()` лишний, у второго — нужен.

**4. rust-analyzer спотыкается на `if/else` в tail-позиции.** Конструкция вида `let x: String = if cond { "a".to_string() } else { format!("{}", y) };` вызывает ложную ошибку «expected String, found ()». Обходим через `let x: String;` + присваивание внутри каждой ветки.

### Зачем это всё
Без backpressure быстрый источник может выдать миллион чанков в память, пока медленный потребитель обрабатывает первый. Backpressure — автоматический тормоз: очередь не растёт бесконечно.

## Новые функции в `src/streams.rs`

Добавьте в конец файла (существующие функции не трогайте):

```rust
// ============================================================
// 17.4: Backpressure и очереди
// ============================================================

/// Создаёт `CountQueuingStrategy` — каждый чанк считается за 1.
///
/// Используем Reflect + JS-конструктор, а не `js_sys::eval` с
/// `dyn_into`: `QueuingStrategy` — это интерфейс, а не класс,
/// у него нет `instanceof`. Приведение делаем через `unchecked_into`.
pub fn count_strategy(high_water_mark: u32) -> Result<web_sys::QueuingStrategy, JsValue> {
    make_queuing_strategy("CountQueuingStrategy", high_water_mark)
}

/// Создаёт `ByteLengthQueuingStrategy` — размер в байтах.
///
/// **Важно:** эту стратегию **нельзя** использовать с byte streams
/// (`type: "bytes"`). Спецификация Streams запрещает `size()`
/// в byte stream. Для byte stream используйте `plain_strategy`.
pub fn byte_length_strategy(high_water_mark: u32) -> Result<web_sys::QueuingStrategy, JsValue> {
    make_queuing_strategy("ByteLengthQueuingStrategy", high_water_mark)
}

/// Создаёт простую стратегию `{ highWaterMark: N }` без `size()`.
///
/// Нужна для byte streams: спецификация Streams запрещает
/// передавать `size()` вместе с `type: "bytes"`. Размер чанка
/// в byte stream вычисляется автоматически как `byteLength`.
pub fn plain_strategy(high_water_mark: u32) -> Result<web_sys::QueuingStrategy, JsValue> {
    let obj = js_sys::Object::new();
    js_sys::Reflect::set(
        &obj,
        &"highWaterMark".into(),
        &JsValue::from_f64(high_water_mark as f64),
    )?;
    Ok(obj.unchecked_into::<web_sys::QueuingStrategy>())
}

/// Общая логика: получить JS-конструктор по имени, создать init,
/// вызвать `new Ctor(init)`, привести к `QueuingStrategy`.
fn make_queuing_strategy(
    name: &str,
    high_water_mark: u32,
) -> Result<web_sys::QueuingStrategy, JsValue> {
    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str(name))?;

    let ctor: js_sys::Function = ctor_value
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{name} is not a constructor")))?;

    let init = js_sys::Object::new();
    js_sys::Reflect::set(
        &init,
        &"highWaterMark".into(),
        &JsValue::from_f64(high_water_mark as f64),
    )?;

    // Function::construct отсутствует в js-sys 0.3.105.
    // Вызов конструктора через Reflect::construct работает во всех версиях.
    let instance = js_sys::Reflect::construct(ctor.as_ref(), &js_sys::Array::of1(&init))?;
    Ok(instance.unchecked_into::<web_sys::QueuingStrategy>())
}

/// Создаёт `ReadableStream` из массива строк с заданной стратегией.
pub fn readable_with_strategy(
    chunks: Vec<String>,
    strategy: &web_sys::QueuingStrategy,
) -> Result<web_sys::ReadableStream, JsValue> {
    readable_with_pull_callback(chunks, strategy, |_| {})
}

/// Создаёт `ReadableStream` с колбэком на каждый `pull`.
///
/// `on_pull` получает `desiredSize` перед `enqueue`. Удобно для
/// демонстрации: видно, как очередь меняется по ходу чтения.
pub fn readable_with_pull_callback<F>(
    chunks: Vec<String>,
    strategy: &web_sys::QueuingStrategy,
    on_pull: F,
) -> Result<web_sys::ReadableStream, JsValue>
where
    F: FnMut(Option<f64>) + 'static,
{
    let queue: Rc<RefCell<VecDeque<String>>> = Rc::new(RefCell::new(chunks.into()));
    let on_pull = Rc::new(RefCell::new(on_pull));

    let pull_callback = {
        let queue = queue.clone();
        let on_pull = on_pull.clone();
        Closure::<dyn FnMut(web_sys::ReadableStreamDefaultController)>::wrap(Box::new(
            move |controller: web_sys::ReadableStreamDefaultController| {
                // controller.desired_size() возвращает Option<f64>
                // напрямую, без Result. У writer'а — иначе (Result).
                let desired: Option<f64> = controller.desired_size();
                (on_pull.borrow_mut())(desired);

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
    js_sys::Reflect::set(
        &source,
        &"pull".into(),
        pull_callback.as_ref().unchecked_ref(),
    )?;
    pull_callback.forget();

    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("ReadableStream"))?;

    let ctor: js_sys::Function = ctor_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("ReadableStream is not a constructor"))?;

    let args = js_sys::Array::new();
    args.push(&source);
    args.push(strategy);

    let stream = js_sys::Reflect::construct(ctor.as_ref(), &args)?;
    Ok(stream.unchecked_into::<web_sys::ReadableStream>())
}

/// Создаёт **byte stream** (`type: "bytes"`) с заданным HWM.
///
/// Byte stream не принимает `size()` в стратегии: размер каждого
/// чанка вычисляется автоматически как `byteLength`. Поэтому
/// мы формируем простую стратегию `{ highWaterMark: N }` внутри.
pub fn readable_bytes_with_strategy(
    chunks: Vec<Vec<u8>>,
    high_water_mark: u32,
) -> Result<web_sys::ReadableStream, JsValue> {
    let queue: Rc<RefCell<VecDeque<Vec<u8>>>> = Rc::new(RefCell::new(chunks.into()));

    let pull_callback = {
        let queue = queue.clone();
        Closure::<dyn FnMut(web_sys::ReadableStreamDefaultController)>::wrap(Box::new(
            move |controller: web_sys::ReadableStreamDefaultController| {
                let mut q = queue.borrow_mut();
                if let Some(chunk) = q.pop_front() {
                    let arr = js_sys::Uint8Array::new_with_length(chunk.len() as u32);
                    arr.copy_from(&chunk);
                    controller.enqueue_with_chunk(&arr).ok();
                } else {
                    controller.close().ok();
                }
            },
        ))
    };

    let source = js_sys::Object::new();
    // Ключевое отличие: type: "bytes".
    js_sys::Reflect::set(&source, &"type".into(), &JsValue::from_str("bytes"))?;
    js_sys::Reflect::set(
        &source,
        &"pull".into(),
        pull_callback.as_ref().unchecked_ref(),
    )?;
    pull_callback.forget();

    // Plain strategy: только highWaterMark, без size().
    let strategy = plain_strategy(high_water_mark)?;

    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("ReadableStream"))?;

    let ctor: js_sys::Function = ctor_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("ReadableStream is not a constructor"))?;

    let args = js_sys::Array::new();
    args.push(&source);
    args.push(&strategy);

    let stream = js_sys::Reflect::construct(ctor.as_ref(), &args)?;
    Ok(stream.unchecked_into::<web_sys::ReadableStream>())
}

/// Создаёт `WritableStream` в буфер с заданным `highWaterMark`.
pub fn writable_with_strategy(
    high_water_mark: u32,
) -> Result<(web_sys::WritableStream, Rc<RefCell<Vec<String>>>), JsValue> {
    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let buffer_c = buffer.clone();

    let write_callback = Closure::<dyn FnMut(JsValue)>::wrap(Box::new(move |chunk: JsValue| {
        let text = chunk.as_string().unwrap_or_default();
        buffer_c.borrow_mut().push(text);
    }));

    let close_callback = Closure::<dyn FnMut()>::wrap(Box::new(|| {}));

    let sink = js_sys::Object::new();
    js_sys::Reflect::set(
        &sink,
        &"write".into(),
        write_callback.as_ref().unchecked_ref(),
    )?;
    js_sys::Reflect::set(
        &sink,
        &"close".into(),
        close_callback.as_ref().unchecked_ref(),
    )?;
    write_callback.forget();
    close_callback.forget();

    let strategy = count_strategy(high_water_mark)?;

    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("WritableStream"))?;

    let ctor: js_sys::Function = ctor_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("WritableStream is not a constructor"))?;

    let args = js_sys::Array::new();
    args.push(&sink);
    args.push(&strategy);

    let stream_value = js_sys::Reflect::construct(ctor.as_ref(), &args)?;
    let stream: web_sys::WritableStream = stream_value.unchecked_into();

    Ok((stream, buffer))
}
```

## `src/app.rs` — только секция 17.4

```rust
//! Демонстрация главы 17.4: Backpressure и очереди.
//!
//! Одна секция с тремя кнопками:
//! 1. Быстрый источник, медленный приёмник.
//! 2. CountQueuingStrategy vs byte stream.
//! 3. desiredSize в действии.

use std::cell::RefCell;
use std::collections::VecDeque;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::callbacks::{self, EventClosures};
use crate::{dom, log, streams};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Backpressure и очереди")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Backpressure — механизм, который не даёт быстрому источнику \
         захлестнуть медленного потребителя. Работает через desiredSize \
         (сколько ещё можно принять) и highWaterMark (порог очереди). \
         Стратегия определяет, как считается размер чанка: Count — за 1, \
         byte stream — по byteLength автоматически.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 4: Backpressure и очереди
    // ============================================================

    let s4 = build_section(&container, "4. Backpressure")?;

    let s4_hint = dom::create_paragraph(
        "desiredSize уменьшается на размер чанка при enqueue и \
         увеличивается, когда чанк уходит потребителю. Когда \
         desiredSize <= 0, источник должен подождать. В ReadableStream \
         браузер перестаёт вызывать pull. В WritableStream не \
         разрешается writer.ready.",
    )?;
    dom::add_class(&dom::as_html(&s4_hint), "hint")?;
    dom::append(&s4, &s4_hint)?;

    let fast_slow_btn = dom::create_element("button")?;
    fast_slow_btn.set_text_content(Some("Быстрый источник, медленный приёмник"));
    let strategies_btn = dom::create_element("button")?;
    strategies_btn.set_text_content(Some("Count vs ByteLength"));
    let desired_btn = dom::create_element("button")?;
    desired_btn.set_text_content(Some("desiredSize в действии"));

    dom::append(&s4, &fast_slow_btn)?;
    dom::append(&s4, &strategies_btn)?;
    dom::append(&s4, &desired_btn)?;

    let s4_result = make_result(&s4)?;

    // ---- Быстрый источник, медленный приёмник ----
    {
        let s4_result = s4_result.clone();
        callbacks::listen(
            &fast_slow_btn,
            "click",
            move |_| {
                let s4_result = s4_result.clone();
                s4_result.set_text_content(Some("Пишем 20 чанков в медленный sink..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Sink с задержкой 50 мс на write.
                    let sink = js_sys::Object::new();

                    let write_fn = Closure::<dyn FnMut(JsValue) -> js_sys::Promise>::wrap(
                        Box::new(|_chunk: JsValue| {
                            // Возвращаем Promise, который разрешается через 50 мс.
                            js_sys::Promise::new(&mut |resolve, _reject| {
                                let resolve_cb = Closure::once_into_js(move || {
                                    resolve.call0(&JsValue::UNDEFINED).ok();
                                });
                                if let Some(window) = web_sys::window() {
                                    window
                                        .set_timeout_with_callback_and_timeout_and_arguments_0(
                                            resolve_cb.unchecked_ref(),
                                            50,
                                        )
                                        .ok();
                                }
                            })
                        }),
                    );

                    let close_fn = Closure::<dyn FnMut()>::wrap(Box::new(|| {}));
                    let abort_fn = Closure::<dyn FnMut(JsValue)>::wrap(Box::new(|_| {}));

                    js_sys::Reflect::set(&sink, &"write".into(), write_fn.as_ref().unchecked_ref())
                        .ok();
                    js_sys::Reflect::set(&sink, &"close".into(), close_fn.as_ref().unchecked_ref())
                        .ok();
                    js_sys::Reflect::set(&sink, &"abort".into(), abort_fn.as_ref().unchecked_ref())
                        .ok();
                    write_fn.forget();
                    close_fn.forget();
                    abort_fn.forget();

                    // WritableStream с HWM = 1.
                    let strategy = match streams::count_strategy(1) {
                        Ok(s) => s,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let ctor_value = match js_sys::Reflect::get(
                        &js_sys::global(),
                        &JsValue::from_str("WritableStream"),
                    ) {
                        Ok(v) => v,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ Reflect: {e:?}")));
                            return;
                        }
                    };
                    let ctor: js_sys::Function = match ctor_value.dyn_into() {
                        Ok(f) => f,
                        Err(_) => {
                            s4_result
                                .set_text_content(Some("❌ WritableStream is not a constructor"));
                            return;
                        }
                    };

                    let args = js_sys::Array::new();
                    args.push(&sink);
                    args.push(&strategy);

                    // Function::construct отсутствует в js-sys 0.3.105.
                    // Используем Reflect::construct.
                    let stream_value = match js_sys::Reflect::construct(ctor.as_ref(), &args) {
                        Ok(v) => v,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ construct: {e:?}")));
                            return;
                        }
                    };
                    let stream: web_sys::WritableStream = stream_value.unchecked_into();

                    let writer: web_sys::WritableStreamDefaultWriter = match stream.get_writer() {
                        Ok(w) => w,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ get_writer: {e:?}")));
                            return;
                        }
                    };

                    let mut log_lines: Vec<String> = Vec::new();
                    log_lines.push("Пишем 20 чанков без await — sink медленный.".to_string());
                    log_lines.push(String::new());

                    for i in 0..20 {
                        let _ = writer.write_with_chunk(&JsValue::from_str(&format!("chunk {i}")));

                        // writer.desired_size() возвращает Result<Option<f64>, JsValue>.
                        let desired: Option<f64> = writer.desired_size().ok().flatten();
                        log_lines.push(format!("write #{}: desiredSize = {:?}", i + 1, desired));

                        s4_result.set_text_content(Some(&log_lines.join("\n")));

                        gloo_timers::future::TimeoutFuture::new(5).await;
                    }

                    log_lines.push(String::new());
                    log_lines.push("Все 20 чанков отправлены. Ждём завершения sink...".to_string());
                    s4_result.set_text_content(Some(&log_lines.join("\n")));

                    let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

                    log_lines.push(String::new());
                    log_lines.push("✅ sink обработал все чанки. Очередь разгружена.".to_string());

                    s4_result.set_text_content(Some(&log_lines.join("\n")));
                });
            },
            &closures,
        )?;
    }

    // ---- CountQueuingStrategy vs ByteLengthQueuingStrategy ----
    {
        let s4_result = s4_result.clone();
        callbacks::listen(
            &strategies_btn,
            "click",
            move |_| {
                let s4_result = s4_result.clone();
                s4_result.set_text_content(Some("Сравниваем стратегии..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Count: HWM = 3, четыре чанка.
                    let count_events: Rc<RefCell<Vec<(Option<f64>, usize, Option<f64>)>>> =
                        Rc::new(RefCell::new(Vec::new()));
                    let count_events_c = count_events.clone();

                    let count_strategy = match streams::count_strategy(3) {
                        Ok(s) => s,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ count_strategy: {e:?}")));
                            return;
                        }
                    };

                    let count_stream = match readable_strings_with_full_log(
                        vec![
                            "a".to_string(),
                            "b".to_string(),
                            "c".to_string(),
                            "d".to_string(),
                        ],
                        &count_strategy,
                        move |before, size, after| {
                            count_events_c.borrow_mut().push((before, size, after));
                        },
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ count stream: {e:?}")));
                            return;
                        }
                    };

                    // Byte: HWM = 8, четыре чанка по 4 байта.
                    let byte_events: Rc<RefCell<Vec<(Option<f64>, usize, Option<f64>)>>> =
                        Rc::new(RefCell::new(Vec::new()));
                    let byte_events_c = byte_events.clone();

                    let byte_stream = match readable_bytes_with_full_log(
                        vec![
                            vec![0u8; 4],
                            vec![0u8; 4],
                            vec![0u8; 4],
                            vec![0u8; 4],
                        ],
                        8,
                        move |before, size, after| {
                            byte_events_c.borrow_mut().push((before, size, after));
                        },
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ byte stream: {e:?}")));
                            return;
                        }
                    };

                    let _ = streams::read_all(&count_stream).await;
                    let _ = streams::read_all(&byte_stream).await;

                    let count_readings = count_events.borrow().clone();
                    let byte_readings = byte_events.borrow().clone();

                    let mut lines = vec![
                        "✅ CountQueuingStrategy(3) vs byte stream (HWM=8)".to_string(),
                        String::new(),
                        "Логируется desiredSize в момент pull: до enqueue, \
                         размер чанка, после enqueue."
                            .to_string(),
                        String::new(),
                        "CountQueuingStrategy(3) — каждый чанк = 1 условной единице:"
                            .to_string(),
                        format!(
                            "  {:<5}{:<18}{:<15}{:<18}{}",
                            "pull", "desired до", "размер чанка", "desired после", "примечание"
                        ),
                    ];

                    for (i, (before, size, after)) in count_readings.iter().enumerate() {
                        let note: String;
                        if *size == 0 {
                            note = "close".to_string();
                        } else {
                            note = format!("было {}, стало {}", fmt_opt(before), fmt_opt(after));
                        }
                        lines.push(format!(
                            "  {:<5}{:<18}{:<15}{:<18}{}",
                            i + 1,
                            fmt_opt(before),
                            size,
                            fmt_opt(after),
                            note
                        ));
                    }

                    lines.push(String::new());
                    lines.push(
                        "Byte stream (HWM=8, size = byteLength) — чанк = 4 байта:"
                            .to_string(),
                    );
                    lines.push(format!(
                        "  {:<5}{:<18}{:<15}{:<18}{}",
                        "pull", "desired до", "размер чанка", "desired после", "примечание"
                    ));

                    for (i, (before, size, after)) in byte_readings.iter().enumerate() {
                        let note: String;
                        if *size == 0 {
                            note = "close".to_string();
                        } else {
                            note = format!("было {}, стало {}", fmt_opt(before), fmt_opt(after));
                        }
                        lines.push(format!(
                            "  {:<5}{:<18}{:<15}{:<18}{}",
                            i + 1,
                            fmt_opt(before),
                            size,
                            fmt_opt(after),
                            note
                        ));
                    }

                    lines.push(String::new());
                    lines.push("Что видно из таблиц:".to_string());
                    lines.push(
                        "• В Count размер каждого чанка равен 1 — содержимое \
                         не важно."
                            .to_string(),
                    );
                    lines.push(
                        "• В byte stream размер = byteLength. Чанк из 4 байт \
                         уменьшает desiredSize сразу на 4."
                            .to_string(),
                    );
                    lines.push(
                        [
                            "• HWM задаёт потолок: 3 для Count, 8 для byte.",
                            "  После enqueue desiredSize становится max(0, before − size).",
                            "  Если места не хватает, оставшееся переносится",
                            "  на следующий pull.",
                        ]
                        .join("\n"),
                    );

                    s4_result.set_text_content(Some(&lines.join("\n")));
                });
            },
            &closures,
        )?;
    }

    // ---- desiredSize в действии ----
    {
        let s4_result = s4_result.clone();
        callbacks::listen(
            &desired_btn,
            "click",
            move |_| {
                let s4_result = s4_result.clone();
                s4_result.set_text_content(Some("Логируем desiredSize..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let events: Rc<RefCell<Vec<(Option<f64>, usize, Option<f64>)>>> =
                        Rc::new(RefCell::new(Vec::new()));
                    let events_c = events.clone();

                    let strategy = match streams::count_strategy(2) {
                        Ok(s) => s,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let stream = match readable_strings_with_full_log(
                        vec![
                            "first".to_string(),
                            "second".to_string(),
                            "third".to_string(),
                        ],
                        &strategy,
                        move |before, size, after| {
                            events_c.borrow_mut().push((before, size, after));
                        },
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s4_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let _ = streams::read_all_strings(&stream).await;

                    let readings = events.borrow().clone();

                    let mut lines = vec![
                        "✅ desiredSize в действии".to_string(),
                        String::new(),
                        "CountQueuingStrategy(2), три чанка.".to_string(),
                        String::new(),
                        format!(
                            "  {:<5}{:<18}{:<15}{:<18}{}",
                            "pull", "desired до", "размер чанка", "desired после", "что произошло"
                        ),
                    ];

                    for (i, (before, size, after)) in readings.iter().enumerate() {
                        let note: String;
                        if *size == 0 {
                            note = "чанков больше нет — close".to_string();
                        } else {
                            note = format!("enqueue чанка длиной {size}");
                        }
                        lines.push(format!(
                            "  {:<5}{:<18}{:<15}{:<18}{}",
                            i + 1,
                            fmt_opt(before),
                            size,
                            fmt_opt(after),
                            note
                        ));
                    }

                    lines.push(String::new());
                    lines.push("Как это читать:".to_string());
                    lines.push(
                        "• «desired до» — сколько места в очереди в момент, \
                         когда браузер вызвал pull."
                            .to_string(),
                    );
                    lines.push(
                        "• «desired после» — сколько осталось после enqueue. \
                         Разница ровно на размер чанка (для Count это 1)."
                            .to_string(),
                    );
                    lines.push(
                        "• Перед следующим pull очередь успевает разгрузиться, \
                         поэтому «desired до» снова равен HWM = 2."
                            .to_string(),
                    );
                    lines.push(
                        "• Backpressure станет заметен, если потребитель \
                         медленный: очередь не будет разгружаться, и \
                         браузер не вызовет pull, пока desiredSize > 0."
                            .to_string(),
                    );

                    s4_result.set_text_content(Some(&lines.join("\n")));
                });
            },
            &closures,
        )?;
    }

    log!("Streams 17.4 demo ready");
    std::mem::forget(closures);

    Ok(())
}

// ============================================================
// Вспомогательные функции
// ============================================================

/// Создаёт `ReadableStream` строк с логированием до/после enqueue.
///
/// `on_event` получает: desiredSize до, размер чанка, desiredSize после.
fn readable_strings_with_full_log<F>(
    chunks: Vec<String>,
    strategy: &web_sys::QueuingStrategy,
    on_event: F,
) -> Result<web_sys::ReadableStream, JsValue>
where
    F: FnMut(Option<f64>, usize, Option<f64>) + 'static,
{
    let queue: Rc<RefCell<VecDeque<String>>> = Rc::new(RefCell::new(chunks.into()));
    let on_event = Rc::new(RefCell::new(on_event));

    let pull_callback = {
        let queue = queue.clone();
        let on_event = on_event.clone();
        Closure::<dyn FnMut(web_sys::ReadableStreamDefaultController)>::wrap(Box::new(
            move |controller: web_sys::ReadableStreamDefaultController| {
                let before: Option<f64> = controller.desired_size();
                let mut q = queue.borrow_mut();

                if let Some(chunk) = q.pop_front() {
                    let size = chunk.len();
                    controller
                        .enqueue_with_chunk(&JsValue::from_str(&chunk))
                        .ok();
                    let after: Option<f64> = controller.desired_size();
                    (on_event.borrow_mut())(before, size, after);
                } else {
                    (on_event.borrow_mut())(before, 0, None);
                    controller.close().ok();
                }
            },
        ))
    };

    let source = js_sys::Object::new();
    js_sys::Reflect::set(
        &source,
        &"pull".into(),
        pull_callback.as_ref().unchecked_ref(),
    )?;
    pull_callback.forget();

    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("ReadableStream"))?;
    let ctor: js_sys::Function = ctor_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("ReadableStream is not a constructor"))?;

    let args = js_sys::Array::new();
    args.push(&source);
    args.push(strategy);

    let stream = js_sys::Reflect::construct(ctor.as_ref(), &args)?;
    Ok(stream.unchecked_into::<web_sys::ReadableStream>())
}

/// То же для byte stream. Размер чанка — длина в байтах.
fn readable_bytes_with_full_log<F>(
    chunks: Vec<Vec<u8>>,
    high_water_mark: u32,
    on_event: F,
) -> Result<web_sys::ReadableStream, JsValue>
where
    F: FnMut(Option<f64>, usize, Option<f64>) + 'static,
{
    let queue: Rc<RefCell<VecDeque<Vec<u8>>>> = Rc::new(RefCell::new(chunks.into()));
    let on_event = Rc::new(RefCell::new(on_event));

    let pull_callback = {
        let queue = queue.clone();
        let on_event = on_event.clone();
        Closure::<dyn FnMut(web_sys::ReadableStreamDefaultController)>::wrap(Box::new(
            move |controller: web_sys::ReadableStreamDefaultController| {
                let before: Option<f64> = controller.desired_size();
                let mut q = queue.borrow_mut();

                if let Some(chunk) = q.pop_front() {
                    let size = chunk.len();
                    let arr = js_sys::Uint8Array::new_with_length(chunk.len() as u32);
                    arr.copy_from(&chunk);
                    controller.enqueue_with_chunk(&arr).ok();
                    let after: Option<f64> = controller.desired_size();
                    (on_event.borrow_mut())(before, size, after);
                } else {
                    (on_event.borrow_mut())(before, 0, None);
                    controller.close().ok();
                }
            },
        ))
    };

    let source = js_sys::Object::new();
    js_sys::Reflect::set(&source, &"type".into(), &JsValue::from_str("bytes"))?;
    js_sys::Reflect::set(
        &source,
        &"pull".into(),
        pull_callback.as_ref().unchecked_ref(),
    )?;
    pull_callback.forget();

    // Plain strategy — byte stream не принимает size() в стратегии.
    let strategy = streams::plain_strategy(high_water_mark)?;

    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("ReadableStream"))?;
    let ctor: js_sys::Function = ctor_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("ReadableStream is not a constructor"))?;

    let args = js_sys::Array::new();
    args.push(&source);
    args.push(&strategy);

    let stream = js_sys::Reflect::construct(ctor.as_ref(), &args)?;
    Ok(stream.unchecked_into::<web_sys::ReadableStream>())
}

/// Форматирует `Option<f64>` для табличного вывода.
/// `None` печатается как «n/a», число — с одним знаком после точки.
fn fmt_opt(v: &Option<f64>) -> String {
    match v {
        Some(x) => format!("{x:.1}"),
        None => "n/a".to_string(),
    }
}

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

## `tests/wasm_streams_backpressure.rs` — 4 теста

```rust
//! Тесты для секции 17.4: Backpressure и очереди.

use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Начальный `desiredSize` равен `highWaterMark`.
#[wasm_bindgen_test]
async fn test_desired_size_signals() {
    let (stream, _buffer) = hello_wasm02::streams::writable_with_strategy(3).unwrap();

    let writer: web_sys::WritableStreamDefaultWriter = stream.get_writer().unwrap();

    // writer.desired_size() возвращает Result<Option<f64>, JsValue>.
    let desired: Option<f64> = writer.desired_size().ok().flatten();
    assert_eq!(
        desired,
        Some(3.0),
        "начальный desiredSize должен быть равен highWaterMark"
    );
}

/// `CountQueuingStrategy` применяется к обычному `ReadableStream`.
#[wasm_bindgen_test]
async fn test_count_strategy() {
    let strategy = hello_wasm02::streams::count_strategy(2).unwrap();

    let stream = hello_wasm02::streams::readable_with_strategy(
        vec!["a".to_string(), "b".to_string(), "c".to_string()],
        &strategy,
    )
    .unwrap();

    let result = hello_wasm02::streams::read_all_strings(&stream)
        .await
        .unwrap();

    assert_eq!(result.join(""), "abc");
}

/// Byte stream с `highWaterMark` работает.
///
/// Для byte stream спецификация запрещает передавать `size()`
/// в стратегии — размер каждого чанка вычисляется автоматически
/// как `byteLength`. Мы передаём только `{ highWaterMark: N }`.
#[wasm_bindgen_test]
async fn test_byte_length_strategy() {
    let stream = hello_wasm02::streams::readable_bytes_with_strategy(
        vec![b"abcd".to_vec(), b"efgh".to_vec()],
        16,
    )
    .unwrap();

    let result = hello_wasm02::streams::read_all(&stream).await.unwrap();

    assert_eq!(String::from_utf8_lossy(&result), "abcdefgh");
}

/// При медленном sink очередь заполняется — `desiredSize` становится <= 0.
#[wasm_bindgen_test]
async fn test_writer_ready_waits() {
    // Sink с задержкой 100 мс на write.
    let sink = js_sys::Object::new();

    let write_fn =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue) -> js_sys::Promise>::wrap(
            Box::new(|_chunk: JsValue| {
                js_sys::Promise::new(&mut |resolve, _reject| {
                    let resolve_cb = wasm_bindgen::closure::Closure::once_into_js(move || {
                        resolve.call0(&JsValue::UNDEFINED).ok();
                    });
                    if let Some(window) = web_sys::window() {
                        window
                            .set_timeout_with_callback_and_timeout_and_arguments_0(
                                resolve_cb.unchecked_ref(),
                                100,
                            )
                            .ok();
                    }
                })
            }),
        );

    let close_fn = wasm_bindgen::closure::Closure::<dyn FnMut()>::wrap(Box::new(|| {}));
    let abort_fn = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::wrap(Box::new(|_| {}));

    js_sys::Reflect::set(&sink, &"write".into(), write_fn.as_ref().unchecked_ref()).unwrap();
    js_sys::Reflect::set(&sink, &"close".into(), close_fn.as_ref().unchecked_ref()).unwrap();
    js_sys::Reflect::set(&sink, &"abort".into(), abort_fn.as_ref().unchecked_ref()).unwrap();
    write_fn.forget();
    close_fn.forget();
    abort_fn.forget();

    let strategy = hello_wasm02::streams::count_strategy(1).unwrap();

    let ctor_value =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("WritableStream")).unwrap();
    let ctor: js_sys::Function = ctor_value.dyn_into().unwrap();

    let args = js_sys::Array::new();
    args.push(&sink);
    args.push(&strategy);

    // Function::construct отсутствует в js-sys 0.3.105.
    // Используем Reflect::construct.
    let stream_value = js_sys::Reflect::construct(ctor.as_ref(), &args).unwrap();
    let stream: web_sys::WritableStream = stream_value.unchecked_into();

    let writer: web_sys::WritableStreamDefaultWriter = stream.get_writer().unwrap();

    // Пишем три чанка без await — очередь накапливается.
    let _p1 = writer.write_with_chunk(&JsValue::from_str("a"));
    let _p2 = writer.write_with_chunk(&JsValue::from_str("b"));
    let _p3 = writer.write_with_chunk(&JsValue::from_str("c"));

    gloo_timers::future::TimeoutFuture::new(10).await;

    let desired: Option<f64> = writer.desired_size().ok().flatten();
    assert!(
        desired.unwrap_or(1.0) <= 0.0,
        "после трёх write без await desiredSize должен быть <= 0, получено {desired:?}"
    );
}
```

## Команды запуска

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown --tests

# Сборка и запуск (с очисткой)
npm start

# Только тесты 17.4
wasm-pack test --headless --chrome --test wasm_streams_backpressure
```

Ожидаемый результат: `4 passed; 0 failed`.

## Ручная проверка демо

Откройте `http://localhost:5173`, найдите секцию «4. Backpressure»:

| Кнопка | Ожидаемый результат |
|---|---|
| «Быстрый источник, медленный приёмник» | Лог с 20 записями, `desiredSize` уходит в минус, потом очередь разгружается |
| «Count vs ByteLength» | Две таблицы с колонками pull / desired до / размер / desired после |
| «desiredSize в действии» | Таблица из четырёх pull, пояснение как читать |

## Четыре ловушки, которые мы поймали по ходу главы

### Ловушка 1. `QueuingStrategy` — интерфейс, а не класс
`dyn_into::<web_sys::QueuingStrategy>()` всегда падает с «not a QueuingStrategy». В JavaScript нет класса `QueuingStrategy` — есть только `CountQueuingStrategy` и `ByteLengthQueuingStrategy`. Приводим через `unchecked_into`, потому что сами создали объект и знаем его форму.

### Ловушка 2. `Function::construct` отсутствует в `js-sys 0.3.105`
Вызов конструктора идёт через `js_sys::Reflect::construct(ctor.as_ref(), &args)`. Тот же приём, что и для `pipeThrough` в 17.3.

### Ловушка 3. Byte stream не принимает `size()` в стратегии
Спецификация Streams запрещает `ByteLengthQueuingStrategy` вместе с `type: "bytes"` — размер каждого чанка вычисляется автоматически как `byteLength`. Для byte stream нужна простая стратегия `{ highWaterMark: N }`. А `ByteLengthQueuingStrategy` работает только с **обычными** потоками, где есть метод `size`.

Плюс расхождение в типах:
- `ReadableStreamDefaultController::desired_size()` → `Option<f64>` (`.ok()` лишний).
- `WritableStreamDefaultWriter::desired_size()` → `Result<Option<f64>, JsValue>` (`.ok().flatten()` нужен).

### Ловушка 4. rust-analyzer спотыкается на `if/else` в tail-позиции
Конструкция:

```rust
let note: String = if *size == 0 {
    "close".to_string()
} else {
    format!("было {}, стало {}", fmt_opt(before), fmt_opt(after))
};
```

вызывает ложную ошибку `expected String, found ()`. rust-analyzer теряет вывод типа в tail-позиции и подставляет `()` в одну из веток. Обход:

```rust
let note: String;
if *size == 0 {
    note = "close".to_string();
} else {
    note = format!("было {}, стало {}", fmt_opt(before), fmt_opt(after));
}
```

То же касается многострочных `format!` с `\`-продолжениями и `\n` внутри. Разбиваем на массив строк + `join("\n")`:

```rust
lines.push(
    [
        "• HWM задаёт потолок: 3 для Count, 8 для byte.",
        "  После enqueue desiredSize становится max(0, before − size).",
        "  Если места не хватает, оставшееся переносится",
        "  на следующий pull.",
    ]
    .join("\n"),
);
```

Это правило действует во всём проекте: если rust-analyzer жалуется на `if/else` в tail-позиции с `format!` — переписывайте на `let x: T; if ... { x = ...; } else { x = ...; }`.

## Итог подглавы

Мы разобрали Backpressure и очереди:

1. **`desiredSize`** — сколько ещё можно принять. Уменьшается на размер каждого чанка при `enqueue`, растёт при уходе к потребителю.
2. **`highWaterMark`** — порог очереди. При нём `desiredSize` становится нулём. По умолчанию: 1 для читаемых и записываемых, 0 для readable-стороны `TransformStream`.
3. **`QueuingStrategy`** определяет, как считается размер чанка. `CountQueuingStrategy` — каждый за 1. `ByteLengthQueuingStrategy` — по `byteLength`, но только для обычных потоков.
4. **Поведение при переполнении:**
   - `ReadableStream` — браузер перестаёт вызывать `pull`.
   - `WritableStream` — `writer.ready` не разрешается.
   - `TransformStream` — `enqueue` блокирует следующий `write`.
5. **`writer.ready`** — Promise, который разрешается, когда в очереди есть место. Полезен для явного управления темпом.

Ключевые уроки:

- **Backpressure — автоматический тормоз.** Без него быстрый источник накопил бы неограниченную очередь в памяти.
- **`desiredSize` — синхронный сигнал.** Его можно читать в любой момент, чтобы решить, писать или подождать.
- **`writer.ready` — асинхронный сигнал.** Позволяет дождаться появления места, не полагаясь на таймеры.
- **`highWaterMark = 0` у readable-стороны `TransformStream`** — источник deadlock'а из 17.3. Именно из-за него нужен `join`.
- **Стратегия важна для бинарных данных.** `ByteLengthQueuingStrategy` считает байты, а не чанки. Но её нельзя комбинировать с byte stream — там `byteLength` считается автоматически.
- **`js_sys` требует внимания к деталям.** `Reflect::construct` вместо `Function::construct`, `unchecked_into` для интерфейсов, разные типы у одноимённых методов `desired_size`.
- **rust-analyzer не любит `if/else` в tail-позиции.** Обходим через `let x: T;` + присваивание внутри веток.

## Что мы не меняем

- `streams_wasm.rs`, `fetch.rs`, `lib.rs` — без изменений.
- Существующие функции в `streams.rs` (17.1–17.3, 17.5) — не трогаются. Добавлены только новые: `count_strategy`, `byte_length_strategy`, `plain_strategy`, `readable_with_strategy`, `readable_with_pull_callback`, `readable_bytes_with_strategy`, `writable_with_strategy` и вспомогательная `make_queuing_strategy`.
- Существующие тесты (17.1, 17.2, 17.3, 17.5) — не затрагиваются.