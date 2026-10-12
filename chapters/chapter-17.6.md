# 17.6. Byte streams и BYOB

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← ОБНОВЛЁН: +readable_bytes_with_byob_support, read_byob
│   ├── streams_wasm.rs             ← без изменений
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.6
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_byte.rs        ← НОВЫЙ: 4 теста
│   └── (остальные без изменений)
│
├── package.json                    ← без изменений
└── (корневые файлы без изменений)
```

**Краткий комментарий.** Byte stream — это особый вид `ReadableStream`, у которого в underlying source указано `type: "bytes"`, а чанки — `Uint8Array`. Такой поток умеет работать с BYOB-читателем (`ReadableStreamBYOBReader`): читатель предоставляет собственный буфер, а данные пишутся прямо в него, без промежуточного копирования. В `pull`-колбэке появляется `controller.byobRequest` — свойство, через которое поток общается с BYOB-читателем. В этой главе добавим в `streams.rs` поддержку BYOB: функция `readable_bytes_with_byob_support` создаёт byte stream с полной обработкой `byobRequest`, а `read_byob` читает из него, используя буфер фиксированного размера. По ходу отладки упрёмся в тонкость с EOF: `close()` на byte stream не разрешает pending `byobRequest`, для этого нужен явный `respond(0)` **после** `close()`.

**Правило архитектуры.** `streams_wasm.rs`, `fetch.rs`, `lib.rs` не трогаются. Существующие функции `streams.rs` (17.1–17.5) не переписываются. Эта глава добавляет только две новые функции и полностью заменяет `app.rs` на секцию 17.6.

## Ссылки для детального изучения

- **MDN: Using readable byte streams** — практическое руководство: [developer.mozilla.org/ru/docs/Web/API/Streams_API/Using_readable_byte_streams](https://developer.mozilla.org/ru/docs/Web/API/Streams_API/Using_readable_byte_streams)
- **MDN: `ReadableStreamBYOBReader`** — читатель с собственным буфером: [developer.mozilla.org/ru/docs/Web/API/ReadableStreamBYOBReader](https://developer.mozilla.org/ru/docs/Web/API/ReadableStreamBYOBReader)
- **MDN: `ReadableStreamBYOBRequest`** — `view`, `respond`: [developer.mozilla.org/ru/docs/Web/API/ReadableStreamBYOBRequest](https://developer.mozilla.org/ru/docs/Web/API/ReadableStreamBYOBRequest)
- **MDN: `ReadableByteStreamController`** — контроллер byte stream: [developer.mozilla.org/ru/docs/Web/API/ReadableByteStreamController](https://developer.mozilla.org/ru/docs/Web/API/ReadableByteStreamController)
- **MDN: `ReadableStream.getReader()`** — параметр `{ mode: "byob" }`: [developer.mozilla.org/ru/docs/Web/API/ReadableStream/getReader](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream/getReader)
- **WHATWG Streams Standard: BYOB readers** — спецификация: [streams.spec.whatwg.org/#byob-readers](https://streams.spec.whatwg.org/#byob-readers)
- **WHATWG: `ReadableByteStreamControllerClose`** — почему `close()` не разрешает pending-буфер: [streams.spec.whatwg.org/#readable-byte-stream-controller-close](https://streams.spec.whatwg.org/#readable-byte-stream-controller-close)
- **WHATWG: `ReadableStreamBYOBRequest.respond()`** — правила `respond(0)`: [streams.spec.whatwg.org/#rs-byob-request-respond](https://streams.spec.whatwg.org/#rs-byob-request-respond)
- **`js_sys::Uint8Array`** — Rust-биндинг: [docs.rs/js-sys/latest/js_sys/struct.Uint8Array.html](https://docs.rs/js-sys/latest/js_sys/struct.Uint8Array.html)
- **`js_sys::Reflect::construct`** — вызов JS-конструктора: [docs.rs/js-sys/latest/js_sys/Reflect/fn.construct.html](https://docs.rs/js-sys/latest/js_sys/Reflect/fn.construct.html)

## Теоретическая часть

### Обычный поток vs byte stream

| | Обычный поток | Byte stream |
|---|---|---|
| `type` в source | не указан | `"bytes"` |
| Тип чанка | любое JS-значение | `ArrayBufferView` (`Uint8Array`) |
| Default reader | ✅ работает | ✅ работает |
| BYOB reader | ❌ бросает `TypeError` | ✅ работает |
| `size()` в стратегии | разрешён | **запрещён** (размер = `byteLength`) |

Обычный поток можно читать только `ReadableStreamDefaultReader` — он забирает чанки из очереди. Byte stream — оба читателя:

- **Default reader** — как обычный поток, `enqueue(chunk)`.
- **BYOB reader** — предоставляет буфер, поток пишет **прямо в него** через `byobRequest.respond(n)`.

### `byobRequest` — мост между потоком и BYOB-читателем

Когда BYOB-читатель вызывает `reader.read(buffer)`, у потока появляется запрос на заполнение этого буфера. Он виден в `pull`-колбэке через `controller.byobRequest`:

- **`byobRequest.view`** — `Uint8Array`, в который надо писать данные. Это подвид переданного буфера.
- **`byobRequest.respond(n)`** — сигнал: «я записал `n` байт в view». Промис читателя разрешается значением — свежим `Uint8Array` с `byteLength = n`.
- **`byobRequest.respond(0)`** — сигнал EOF. Требует, чтобы `closeRequested = true` (то есть `controller.close()` уже был вызван).

Если `byobRequest` отсутствует (default reader), работаем как обычно: `controller.enqueue(chunk)`.

### Полный цикл BYOB-чтения

Разберём на примере: чанк = 10 байт, размер буфера читателя = 4 байта.

1. Читатель вызывает `reader.read(buffer4)`. В `pull` приходит `byobRequest` с view длиной 4.
2. Поток берёт первый чанк (10 байт), копирует первые 4 в view, вызывает `respond(4)`.
3. Промис читателя разрешается значением `Uint8Array` длины 4.
4. Остаток чанка (6 байт) поток кладёт обратно в очередь.
5. Читатель снова вызывает `read(buffer4)`. `pull` снова вызывается с view длиной 4.
6. Поток отдаёт 4 байта остатка, `respond(4)`. Остаётся 2 байта.
7. Читатель снова `read`. Поток отдаёт 2 байта, `respond(2)`.
8. Читатель снова `read`. Очередь пуста — `close()`, затем `respond(0)`. Промис разрешается `{done: true}`.

Итого: 10 = 4 + 4 + 2.

### EOF и почему `close()` в одиночку не работает

Ключевой момент, который неочевиден из спецификации и на котором мы споткнулись в отладке.

**Что делает `ReadableByteStreamControllerClose()`:**
- Устанавливает `[[closeRequested]] = true`.
- Если очередь непуста — откладывает фактическое закрытие до момента, когда очередь разгрузится.
- Если `[[pendingPullIntos]]` непуст и в первом из них `bytesFilled = 0` — просто закрывает флаг.
- **Не разрешает** pending `byobRequest`.

**Что делает `ReadableStreamBYOBRequest.respond(0)`:**
- Если `[[closeRequested]] = true` и очередь пуста — закрывает поток и разрешает pending-read с `{done: true}`.
- Если `[[closeRequested]] = false` — бросает `TypeError`.

Из этих двух фактов следует **строгий порядок**:

1. Сначала `controller.close()` — устанавливает `closeRequested`.
2. Потом `byobRequest.respond(0)` — разрешает pending-буфер.

В обратном порядке — ошибка. Только `close()` без `respond(0)` — зависание читателя. Именно это мы наблюдали в тесте `test_byte_stream_byob_reader`: Chrome убивался по таймауту (`signal: 9 (SIGKILL)`), потому что BYOB-читатель ждал ответа на свой `read`, а pull больше не вызывался.

### Когда BYOB полезен

- **Большие файлы.** Избегаем промежуточной копии: данные идут из сети прямо в буфер приложения.
- **Сетевой стриминг.** Меньше аллокаций — меньше пауз GC.
- **Обработка бинарных форматов.** Например, `Uint8Array` парсится напрямую, без создания промежуточных `Vec<u8>`.

### Когда BYOB не нужен

- **Текстовые потоки с чанками переменного размера.** У строк нет `byteLength`, byte stream для них не подходит.
- **Маленькие потоки.** Выигрыш от отсутствия копии незаметен.
- **Простой pipeline.** Default reader проще и не требует обработки `byobRequest`.

## Новые функции в `src/streams.rs`

Добавьте в конец файла. Существующие функции не трогайте.

```rust
// ============================================================
// 17.6: Byte streams и BYOB
// ============================================================

/// Создаёт **byte stream** с полной поддержкой BYOB-читателя.
///
/// В `pull`-колбэке проверяет наличие `controller.byobRequest`:
/// - если есть — пишет данные прямо в `byobRequest.view` и вызывает
///   `respond(n)`;
/// - если нет — работает как обычный `enqueue` для default reader.
///
/// **EOF.** При пустой очереди вызывается `controller.close()`,
/// затем `byobRequest.respond(0)`. Порядок строгий: `respond(0)`
/// требует `closeRequested = true`, а `close()` сам pending-буфер
/// не разрешает. Без `respond(0)` BYOB-читатель зависает.
pub fn readable_bytes_with_byob_support(
    chunks: Vec<Vec<u8>>,
) -> Result<web_sys::ReadableStream, JsValue> {
    let queue: Rc<RefCell<VecDeque<Vec<u8>>>> = Rc::new(RefCell::new(chunks.into()));

    let pull_callback = {
        let queue = queue.clone();
        Closure::<dyn FnMut(JsValue)>::wrap(Box::new(move |controller: JsValue| {
            let mut q = queue.borrow_mut();

            // Проверяем, ждёт ли BYOB-читатель со своим буфером.
            let byob = js_sys::Reflect::get(&controller, &JsValue::from_str("byobRequest"))
                .ok()
                .filter(|v| !v.is_null() && !v.is_undefined());

            if let Some(req) = byob {
                let view_opt = js_sys::Reflect::get(&req, &JsValue::from_str("view"))
                    .ok()
                    .and_then(|v| v.dyn_into::<js_sys::Uint8Array>().ok());

                if let Some(view) = view_opt {
                    if let Some(chunk) = q.pop_front() {
                        let n = chunk.len().min(view.length() as usize);
                        view.subarray(0, n as u32).copy_from(&chunk[..n]);

                        if let Ok(respond) =
                            js_sys::Reflect::get(&req, &JsValue::from_str("respond"))
                                .and_then(|f| f.dyn_into::<js_sys::Function>())
                        {
                            respond.call1(&req, &JsValue::from_f64(n as f64)).ok();
                        }

                        // Если данных больше, чем поместилось, — вернуть
                        // остаток в очередь. Следующий pull отдаст его.
                        if n < chunk.len() {
                            q.push_front(chunk[n..].to_vec());
                        }
                        return;
                    } else {
                        // Очередь пуста. Порядок важен:
                        // 1. close() — устанавливает closeRequested = true.
                        //    Сам по себе он не разрешает pending byobRequest.
                        // 2. respond(0) — разрешает pending-буфер читателя
                        //    с {done: true}. Требует closeRequested = true,
                        //    поэтому вызывается после close().
                        if let Ok(close) =
                            js_sys::Reflect::get(&controller, &JsValue::from_str("close"))
                                .and_then(|f| f.dyn_into::<js_sys::Function>())
                        {
                            close.call0(&controller).ok();
                        }

                        if let Ok(respond) =
                            js_sys::Reflect::get(&req, &JsValue::from_str("respond"))
                                .and_then(|f| f.dyn_into::<js_sys::Function>())
                        {
                            respond.call1(&req, &JsValue::from_f64(0.0)).ok();
                        }
                        return;
                    }
                }
            }

            // Default reader: работаем через enqueue.
            if let Some(chunk) = q.pop_front() {
                if let Ok(enqueue) =
                    js_sys::Reflect::get(&controller, &JsValue::from_str("enqueue"))
                        .and_then(|f| f.dyn_into::<js_sys::Function>())
                {
                    let arr = js_sys::Uint8Array::new_with_length(chunk.len() as u32);
                    arr.copy_from(&chunk);
                    enqueue.call1(&controller, &arr).ok();
                }
            } else if let Ok(close) =
                js_sys::Reflect::get(&controller, &JsValue::from_str("close"))
                    .and_then(|f| f.dyn_into::<js_sys::Function>())
            {
                close.call0(&controller).ok();
            }
        }))
    };

    let source = js_sys::Object::new();
    js_sys::Reflect::set(
        &source,
        &JsValue::from_str("type"),
        &JsValue::from_str("bytes"),
    )?;
    js_sys::Reflect::set(
        &source,
        &JsValue::from_str("pull"),
        pull_callback.as_ref().unchecked_ref(),
    )?;
    pull_callback.forget();

    let ctor_val =
        js_sys::Reflect::get(&js_sys::global(), &JsValue::from_str("ReadableStream"))?;
    let ctor: js_sys::Function = ctor_val
        .dyn_into()
        .map_err(|_| JsValue::from_str("ReadableStream is not a constructor"))?;

    let stream = js_sys::Reflect::construct(ctor.as_ref(), &js_sys::Array::of1(&source))?;
    Ok(stream.unchecked_into::<web_sys::ReadableStream>())
}

/// Читает byte stream через BYOB-читатель с буфером заданного размера.
///
/// `reader.read(buffer)` возвращает `Uint8Array` с `byteLength`,
/// равным фактически прочитанным байтам. Мы складываем их в `Vec<u8>`.
/// Когда поток закрыт, `read` возвращает `{done: true}` — цикл
/// завершается.
///
/// `buffer_size` задаёт размер буфера на одну операцию чтения. Если
/// чанк в потоке больше, он будет отдан несколькими порциями.
pub async fn read_byob(
    stream: &web_sys::ReadableStream,
    buffer_size: usize,
) -> Result<Vec<u8>, FetchError> {
    // stream.getReader({ mode: "byob" }) — через Reflect, потому что
    // web-sys не предоставляет типизированного метода с опциями.
    let get_reader_val = js_sys::Reflect::get(stream, &JsValue::from_str("getReader"))
        .map_err(classify_error)?;
    let get_reader: js_sys::Function = get_reader_val
        .dyn_into()
        .map_err(|_| FetchError::Parse("getReader is not a function".into()))?;

    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("mode"),
        &JsValue::from_str("byob"),
    )
    .map_err(classify_error)?;

    let reader: JsValue = get_reader
        .call1(stream.as_ref(), &options)
        .map_err(classify_error)?;

    let mut result = Vec::new();

    loop {
        // Свежий буфер на каждый read — после respond(n) переданный
        // буфер может быть отсоединён, повторное использование
        // небезопасно.
        let buffer = js_sys::Uint8Array::new_with_length(buffer_size as u32);

        let read_fn_val =
            js_sys::Reflect::get(&reader, &JsValue::from_str("read")).map_err(classify_error)?;
        let read_fn: js_sys::Function = read_fn_val
            .dyn_into()
            .map_err(|_| FetchError::Parse("read is not a function".into()))?;

        let promise_val = read_fn
            .call1(&reader, buffer.as_ref())
            .map_err(classify_error)?;
        let promise: js_sys::Promise = promise_val
            .dyn_into()
            .map_err(|_| FetchError::Parse("read did not return a Promise".into()))?;

        let result_value = JsFuture::from(promise).await.map_err(classify_error)?;

        let done = js_sys::Reflect::get(&result_value, &JsValue::from_str("done"))
            .ok()
            .and_then(|v| v.as_bool())
            .unwrap_or(true);
        if done {
            break;
        }

        let value = js_sys::Reflect::get(&result_value, &JsValue::from_str("value"))
            .map_err(classify_error)?;
        let arr: js_sys::Uint8Array = value
            .dyn_into()
            .map_err(|_| FetchError::Parse("value is not a Uint8Array".into()))?;

        result.extend_from_slice(&arr.to_vec());
    }

    // Освобождаем читателя, чтобы поток можно было переиспользовать.
    if let Ok(release) =
        js_sys::Reflect::get(&reader, &JsValue::from_str("releaseLock"))
            .and_then(|f| f.dyn_into::<js_sys::Function>())
    {
        release.call0(&reader).ok();
    }

    Ok(result)
}
```

## `src/app.rs` — только секция 17.6

```rust
//! Демонстрация главы 17.6: Byte streams и BYOB.
//!
//! Одна секция с тремя кнопками:
//! 1. Byte stream + default reader.
//! 2. Byte stream + BYOB reader.
//! 3. BYOB: маленький буфер, много данных.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::{dom, log, streams};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Byte streams и BYOB")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Byte stream — ReadableStream с type: \"bytes\" и чанками \
         Uint8Array. Работает и с default reader, и с BYOB-читателем. \
         BYOB (bring your own buffer) передаёт собственный буфер, \
         в который поток пишет напрямую — без промежуточного копирования.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 6: Byte streams и BYOB
    // ============================================================

    let s6 = build_section(&container, "6. Byte streams и BYOB")?;

    let s6_hint = dom::create_paragraph(
        "В pull-колбэке byte stream доступен controller.byobRequest. \
         Если он есть — пишем данные в byobRequest.view и вызываем \
         respond(n). Если нет — работаем через enqueue. При EOF нужны \
         close() и respond(0) в этом порядке.",
    )?;
    dom::add_class(&dom::as_html(&s6_hint), "hint")?;
    dom::append(&s6, &s6_hint)?;

    let default_btn = dom::create_element("button")?;
    default_btn.set_text_content(Some("Byte stream + default reader"));
    let byob_btn = dom::create_element("button")?;
    byob_btn.set_text_content(Some("Byte stream + BYOB reader"));
    let small_btn = dom::create_element("button")?;
    small_btn.set_text_content(Some("BYOB: маленький буфер, много данных"));

    dom::append(&s6, &default_btn)?;
    dom::append(&s6, &byob_btn)?;
    dom::append(&s6, &small_btn)?;

    let s6_result = make_result(&s6)?;

    // ---- Byte stream + default reader ----
    {
        let s6_result = s6_result.clone();
        callbacks::listen(
            &default_btn,
            "click",
            move |_| {
                let s6_result = s6_result.clone();
                s6_result.set_text_content(Some("Читаем default reader'ом..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let stream = match streams::readable_bytes_with_byob_support(vec![
                        b"hello ".to_vec(),
                        b"byte ".to_vec(),
                        b"stream".to_vec(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s6_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::read_all(&stream).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            s6_result.set_text_content(Some(&format!(
                                "✅ Byte stream прочитан default reader'ом\n\n\
                                 Прочитано: \"{text}\"\n\
                                 Размер: {} байт\n\n\
                                 Default reader вызывает pull через \
                                 controller.enqueue(chunk). BYOB-ветка не \
                                 активируется, потому что byobRequest \
                                 отсутствует.",
                                bytes.len()
                            )));
                        }
                        Err(e) => {
                            s6_result.set_text_content(Some(&format!(
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

    // ---- Byte stream + BYOB reader ----
    {
        let s6_result = s6_result.clone();
        callbacks::listen(
            &byob_btn,
            "click",
            move |_| {
                let s6_result = s6_result.clone();
                s6_result.set_text_content(Some("Читаем BYOB-читателем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let stream = match streams::readable_bytes_with_byob_support(vec![
                        b"hello ".to_vec(),
                        b"byte ".to_vec(),
                        b"stream".to_vec(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s6_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::read_byob(&stream, 1024).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            s6_result.set_text_content(Some(&format!(
                                "✅ Byte stream прочитан BYOB-читателем\n\n\
                                 Прочитано: \"{text}\"\n\
                                 Размер: {} байт\n\
                                 Размер буфера: 1024 байта\n\n\
                                 BYOB-читатель предоставил буфер 1024 байта. \
                                 Поток писал данные прямо в byobRequest.view \
                                 и сигналил через respond(n). Копирования \
                                 между буферами не было. При завершении — \
                                 close() и respond(0) в этом порядке.",
                                bytes.len()
                            )));
                        }
                        Err(e) => {
                            s6_result.set_text_content(Some(&format!(
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

    // ---- BYOB: маленький буфер, много данных ----
    {
        let s6_result = s6_result.clone();
        callbacks::listen(
            &small_btn,
            "click",
            move |_| {
                let s6_result = s6_result.clone();
                s6_result.set_text_content(Some("Читаем по 4 байта..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Один чанк на 20 байт. Читаем буфером 4 байта —
                    // потребуется пять операций read.
                    let stream = match streams::readable_bytes_with_byob_support(vec![
                        b"0123456789ABCDEFGHIJ".to_vec(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s6_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::read_byob(&stream, 4).await {
                        Ok(bytes) => {
                            let text = String::from_utf8_lossy(&bytes);
                            s6_result.set_text_content(Some(&format!(
                                "✅ BYOB с буфером 4 байта\n\n\
                                 Исходные данные: 0123456789ABCDEFGHIJ (20 байт)\n\
                                 Прочитано: \"{text}\"\n\
                                 Всего: {} байт\n\n\
                                 Что произошло:\n\
                                 • Чанк 20 байт. Буфер читателя — 4 байта.\n\
                                 • Первый read: 4 байта, respond(4).\n\
                                 • Второй read: 4 байта, respond(4).\n\
                                 • Третий read: 4 байта, respond(4).\n\
                                 • Четвёртый read: 4 байта, respond(4).\n\
                                 • Пятый read: 4 байта, respond(4).\n\
                                 • Шестой read: очередь пуста, close() \
                                 и respond(0).\n\
                                 • Пять операций read, суммарно 20 байт.",
                                bytes.len()
                            )));
                        }
                        Err(e) => {
                            s6_result.set_text_content(Some(&format!(
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

    log!("Streams 17.6 demo ready");
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

## `tests/wasm_streams_byte.rs` — 4 теста

```rust
//! Тесты для секции 17.6: Byte streams и BYOB.

use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Byte stream можно читать default reader'ом.
#[wasm_bindgen_test]
async fn test_byte_stream_default_reader() {
    let stream = hello_wasm02::streams::readable_bytes_with_byob_support(vec![
        b"hello".to_vec(),
        b" world".to_vec(),
    ])
    .unwrap();

    let result = hello_wasm02::streams::read_all(&stream).await.unwrap();

    assert_eq!(String::from_utf8_lossy(&result), "hello world");
}

/// Byte stream корректно читается BYOB-читателем.
#[wasm_bindgen_test]
async fn test_byte_stream_byob_reader() {
    let stream = hello_wasm02::streams::readable_bytes_with_byob_support(vec![
        b"hello".to_vec(),
        b" world".to_vec(),
    ])
    .unwrap();

    let result = hello_wasm02::streams::read_byob(&stream, 1024)
        .await
        .unwrap();

    assert_eq!(String::from_utf8_lossy(&result), "hello world");
}

/// BYOB с маленьким буфером: чанк больше буфера — читается частями.
#[wasm_bindgen_test]
async fn test_byob_partial_fill() {
    // Один чанк 10 байт, буфер 4 байта — читается как 4+4+2.
    let stream = hello_wasm02::streams::readable_bytes_with_byob_support(vec![
        b"0123456789".to_vec(),
    ])
    .unwrap();

    let result = hello_wasm02::streams::read_byob(&stream, 4)
        .await
        .unwrap();

    assert_eq!(String::from_utf8_lossy(&result), "0123456789");
}

/// BYOB корректно сигналит EOF на пустом потоке.
#[wasm_bindgen_test]
async fn test_byob_eof() {
    let stream = hello_wasm02::streams::readable_bytes_with_byob_support(vec![]).unwrap();

    let result = hello_wasm02::streams::read_byob(&stream, 1024)
        .await
        .unwrap();

    assert!(result.is_empty(), "пустой поток должен вернуть пустой Vec");
}
```

## Команды запуска

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown --tests

# Сборка и запуск (с очисткой)
npm start

# Только тесты 17.6
wasm-pack test --headless --chrome --test wasm_streams_byte
```

Ожидаемый результат: `4 passed; 0 failed`.

## Ручная проверка демо

Откройте `http://localhost:5173`, найдите секцию «6. Byte streams и BYOB»:

| Кнопка | Ожидаемый результат |
|---|---|
| «Byte stream + default reader» | `hello byte stream` через `enqueue` |
| «Byte stream + BYOB reader» | Тот же текст через `respond(n)`, буфер 1024 |
| «BYOB: маленький буфер, много данных» | 20 байт, прочитанных пятью операциями по 4 байта |

## Ловушка, которую мы поймали

Единственная сложность этой главы — EOF на byte stream. Разберём подробнее, потому что это самая неинтуитивная часть спецификации.

### Что не работает

Если в `pull`-колбэке при пустой очереди вызвать только `controller.close()` — BYOB-читатель зависнет. В тесте это проявляется как `signal: 9 (SIGKILL)`: Chrome убивается по таймауту, потому что `reader.read(buffer)` возвращает Promise, который никогда не разрешается.

### Что тоже не работает

Если вызвать `byobRequest.respond(0)` **до** `close()` — будет `TypeError`. Спецификация требует, чтобы `closeRequested = true`.

### Что работает

Строгий порядок:

```rust
// 1. close() — устанавливает closeRequested = true.
controller.close();

// 2. respond(0) — разрешает pending-буфер с {done: true}.
byob_request.respond(0);
```

Первый вызов только помечает поток как «закрывающийся». Второй — извещает читателя о завершении.

### Почему так

`ReadableByteStreamControllerClose()` намеренно **не** разрешает pending-буфер сам: по спецификации читатель может быть в середине передачи большого блока данных, и `close()` не должен «разрывать» буфер на полпути. Разрешение pending-буфера передаётся явно — через `respond(0)`. Это даёт приложению контроль над тем, когда именно читатель увидит EOF.

Правило для будущих похожих случаев: **если поток читается BYOB-читателем, при пустой очереди нужно вызвать `close()`, затем `respond(0)`**. Только `close()` — зависание. Только `respond(0)` — `TypeError`.

## Итог подглавы

Мы разобрали byte streams и BYOB-читатели:

1. **Byte stream** — `ReadableStream` с `type: "bytes"` и `Uint8Array`-чанками.
2. **`byobRequest`** — объект в `pull`-колбэке, через который поток общается с BYOB-читателем.
3. **`byobRequest.view`** — буфер, в который надо писать.
4. **`byobRequest.respond(n)`** — сигнал «записано `n` байт».
5. **EOF требует двух вызовов.** `controller.close()` устанавливает `closeRequested`, `byobRequest.respond(0)` разрешает pending-буфер. Порядок строгий.
6. **`reader.getReader({ mode: "byob" })`** — BYOB-читатель. В Rust — через `Reflect`, потому что web-sys не даёт типизированного метода с опциями.
7. **Частичное заполнение** — если чанк больше буфера, поток возвращает остаток в очередь, и следующие `read` его доберут.

Ключевые уроки:

- **BYOB экономит копирование.** Данные идут из источника прямо в буфер читателя.
- **Один поток — два режима.** Проверка `byobRequest` в `pull` позволяет поддерживать и default, и BYOB читателя.
- **EOF — это два вызова, а не один.** `close()` без `respond(0)` оставляет читателя висеть. `respond(0)` без `close()` бросает `TypeError`.
- **Свежий буфер на каждый `read`.** После `respond(n)` переданный буфер может быть отсоединён — переиспользование небезопасно.
- **Byte stream не принимает `size()` в стратегии.** Размер чанка считается автоматически как `byteLength`.

## Что мы не меняем

- `streams_wasm.rs`, `fetch.rs`, `lib.rs` — без изменений.
- Существующие функции в `streams.rs` (17.1–17.5) — не трогаются. Добавлены только две новые: `readable_bytes_with_byob_support` и `read_byob`.
- Существующие тесты (17.1–17.5) — не затрагиваются.
- Функция `readable_bytes` из 17.5 остаётся как есть: она работает с default reader, там EOF через простой `close()` достаточен.