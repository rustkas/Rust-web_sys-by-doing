# 17.3. `TransformStream`: конвейер преобразования

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← без изменений (библиотечный модуль,
│   │                                  содержит всё для 17.1–17.5)
│   ├── streams_wasm.rs             ← без изменений
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.3
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_transform.rs   ← НОВЫЙ: 8 тестов
│   └── (остальные без изменений)
│
├── package.json                    ← без изменений
└── (корневые файлы без изменений)
```

**Краткий комментарий.** `TransformStream` — самый мощный инструмент Streams API. У него две стороны: `writable` (вход) и `readable` (выход). Всё, что записано во вход, проходит через колбэк `transform` и появляется на выходе. Колбэк `flush` вызывается при закрытии входа — здесь «досылают» буферизованные данные. Главная ловушка: `readable` имеет `highWaterMark = 0`, поэтому запись и чтение надо вести **параллельно** через `futures::future::join`, иначе второй `write` зависнет. В главе также разбираются `pipeThrough` и `pipeTo` с опциями `preventClose`, `preventAbort`, `preventCancel`.

**Важно про архитектуру.** Библиотечные модули (`streams.rs`, `streams_wasm.rs`, `fetch.rs`, `lib.rs`) уже содержат все необходимые функции — они написаны в предыдущих подглавах и **не меняются**. Эта глава добавляет только новый `app.rs` и новый файл тестов. Так мы не ломаем решения 17.1, 17.2 и 17.5.

## Ссылки для детального изучения

- **MDN: `TransformStream`** — основной интерфейс: [developer.mozilla.org/ru/docs/Web/API/TransformStream](https://developer.mozilla.org/ru/docs/Web/API/TransformStream)
- **MDN: Transformer** — методы `start`, `transform`, `flush`: [developer.mozilla.org/ru/docs/Web/API/TransformStream/TransformStream#parameters](https://developer.mozilla.org/ru/docs/Web/API/TransformStream/TransformStream#parameters)
- **MDN: `TransformStreamDefaultController`** — `enqueue`, `error`, `desiredSize`: [developer.mozilla.org/ru/docs/Web/API/TransformStreamDefaultController](https://developer.mozilla.org/ru/docs/Web/API/TransformStreamDefaultController)
- **MDN: `ReadableStream.pipeThrough()`** — соединение потока и преобразователя: [developer.mozilla.org/ru/docs/Web/API/ReadableStream/pipeThrough](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream/pipeThrough)
- **MDN: `ReadableStream.pipeTo()`** — соединение потока и приёмника с опциями: [developer.mozilla.org/ru/docs/Web/API/ReadableStream/pipeTo](https://developer.mozilla.org/ru/docs/Web/API/ReadableStream/pipeTo)
- **WHATWG Streams Standard: `TransformStream`** — спецификация: [streams.spec.whatwg.org/#ts-class](https://streams.spec.whatwg.org/#ts-class)
- **`futures::future::join`** — параллельный опрос двух Future: [docs.rs/futures/latest/futures/future/fn.join.html](https://docs.rs/futures/latest/futures/future/fn.join.html)
- **`web_sys::TransformStream`** — Rust-биндинг: [docs.rs/web-sys/latest/web_sys/struct.TransformStream.html](https://docs.rs/web-sys/latest/web_sys/struct.TransformStream.html)

## Теоретическая часть

### Underlying transformer
JS-объект с методами, который браузер оборачивает в `TransformStream`.

- **`start(controller)`** — вызывается один раз при создании. Инициализация.
- **`transform(chunk, controller)`** — вызывается на каждый записанный чанк. Должен сделать одно из трёх: вызвать `controller.enqueue(output)` (отдать преобразованный чанк), ничего не делать (чанк отбрасывается) или вызвать `controller.error(e)` (перевести поток в errored-состояние).
- **`flush(controller)`** — вызывается при закрытии входа. Здесь «досылают» буферизованные данные. Классический пример — остаток незавершённой строки в line splitter'е.

### Backpressure
`readable`-сторона `TransformStream` по умолчанию имеет `highWaterMark = 0`. Это значит: как только `transform` вызовет `controller.enqueue(chunk)`, поток блокируется, пока кто-нибудь не заберёт чанк из `readable`. Если писать в `writable` и не читать `readable` одновременно — второй `write` зависнет.

**Решение:** запускать запись и чтение **параллельно** через `futures::future::join`. Пока write ждёт читателя, read забирает готовый чанк, освобождает backpressure, write продолжает.

### Обработка ошибок
- Если `transform` бросит исключение — обе стороны переходят в errored-состояние.
- Читатель получает `{done: true}` с ошибкой, `read()` reject'ится.
- Явный вызов `controller.error(e)` даёт тот же результат, но под контролем.

### `flush` на примере line splitter
- Вход — чанки произвольного размера.
- Задача — разбить на строки по `\n`.
- Если чанк заканчивается серединой строки — буферизуем остаток.
- При `flush` — отдаём остаток как последнюю строку.

Без `flush` последняя строка потеряется, если файл не заканчивается `\n`.

### Два способа соединения потоков

**`readable.pipeThrough(transform)`** — соединяет источник с `TransformStream`, возвращает **новый** `ReadableStream` (выходную сторону преобразователя). Под капотом делает `this.pipeTo(transform.writable)` и возвращает `transform.readable`.

**`readable.pipeTo(writable, options)`** — соединяет источник напрямую с `WritableStream`, возвращает `Promise<void>`.

**Опции `pipeTo`:**
- **`preventClose`** (по умолчанию `false`). Если `true` — после завершения источника **не** вызывается `close()` на приёмнике.
- **`preventAbort`** (по умолчанию `false`). Если `true` — при ошибке источника **не** вызывается `abort()` на приёмнике.
- **`preventCancel`** (по умолчанию `false`). Если `true` — при ошибке приёмника **не** вызывается `cancel()` на источнике.
- **`signal`** — `AbortSignal` для отмены всей операции.

**Когда что использовать:**
- **Обычный случай:** все опции `false`.
- **`preventClose`** — когда один приёмник принимает данные из нескольких источников последовательно.
- **`preventAbort`** — когда приёмник ещё пригодится после ошибки источника.
- **`preventCancel`** — когда источник можно переиспользовать после сбоя приёмника.

## Используемые функции из `streams.rs`

Ничего нового в библиотечный модуль добавлять не нужно — все функции уже есть с предыдущих подглав:

| Функция | Назначение |
|---|---|
| `readable_from_strings` | Источник из массива строк (для pipeTo) |
| `writable_with_callbacks` | Приёмник с колбэками write/close/abort |
| `read_all_strings` | Чтение потока как строк |
| `transform_with` | Простое преобразование |
| `transform_with_error` | Преобразование с возможностью ошибки |
| `transform_with_flush` | Преобразование с flush |
| `line_splitter` | Line splitter с flush |
| `line_splitter_no_flush` | Line splitter без flush (для тестов) |
| `process_transform` | Прогон input через TransformStream |
| `call_pipe_through` | pipeThrough через Reflect |
| `pipe_to` | pipeTo с опциями |

## `src/app.rs` — только секция 17.3

```rust
//! Демонстрация главы 17.3: `TransformStream`, `pipeThrough`, `pipeTo`.
//!
//! Одна секция с шестью кнопками:
//! 1. Upper case — базовое преобразование.
//! 2. Line splitter — буферизованный преобразователь с flush.
//! 3. Счётчик слов — параллельная обработка в transform.
//! 4. Ошибка в transform — проброс ошибки на readable-сторону.
//! 5. pipeTo: обычный случай — источник → приёмник с автоматическим закрытием.
//! 6. pipeTo: preventClose — приёмник остаётся открытым после pipe.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use crate::callbacks::{self, EventClosures};
use crate::{dom, log, streams};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "TransformStream: конвейер преобразования")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "TransformStream — конвейер. У него две стороны: writable (вход) \
         и readable (выход). Всё, что записано во вход, проходит через \
         transform и появляется на выходе. Колбэк flush вызывается при \
         закрытии входа — здесь досылают буферизованные данные. Запись \
         и чтение идут параллельно через join: readable имеет \
         highWaterMark = 0, и без параллельности второй write зависнет.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 3: TransformStream — конвейер
    // ============================================================

    let s3 = build_section(&container, "3. TransformStream: конвейер")?;

    let s3_hint = dom::create_paragraph(
        "Underlying transformer получает методы transform и flush. \
         transform вызывается на каждый записанный чанк. flush — при \
         закрытии входа. controller.enqueue(output) отдаёт чанк на \
         readable-сторону. controller.error(reason) переводит обе \
         стороны в errored-состояние.",
    )?;
    dom::add_class(&dom::as_html(&s3_hint), "hint")?;
    dom::append(&s3, &s3_hint)?;

    let upper_btn = dom::create_element("button")?;
    upper_btn.set_text_content(Some("Upper case"));
    let splitter_btn = dom::create_element("button")?;
    splitter_btn.set_text_content(Some("Line splitter"));
    let wordcount_btn = dom::create_element("button")?;
    wordcount_btn.set_text_content(Some("Счётчик слов"));
    let error_btn = dom::create_element("button")?;
    error_btn.set_text_content(Some("Ошибка в transform"));
    let pipe_basic_btn = dom::create_element("button")?;
    pipe_basic_btn.set_text_content(Some("pipeTo: обычный случай"));
    let pipe_prevent_btn = dom::create_element("button")?;
    pipe_prevent_btn.set_text_content(Some("pipeTo: preventClose"));

    dom::append(&s3, &upper_btn)?;
    dom::append(&s3, &splitter_btn)?;
    dom::append(&s3, &wordcount_btn)?;
    dom::append(&s3, &error_btn)?;
    dom::append(&s3, &pipe_basic_btn)?;
    dom::append(&s3, &pipe_prevent_btn)?;

    let s3_result = make_result(&s3)?;

    // ---- Upper case ----
    {
        let s3_result = s3_result.clone();
        callbacks::listen(
            &upper_btn,
            "click",
            move |_| {
                let s3_result = s3_result.clone();
                s3_result.set_text_content(Some("Преобразуем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let transform = match streams::transform_with(|s: String| s.to_uppercase()) {
                        Ok(t) => t,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let input = vec![
                        "hello ".to_string(),
                        "world ".to_string(),
                        "rust".to_string(),
                    ];

                    match streams::process_transform(transform, input).await {
                        Ok(chunks) => {
                            s3_result.set_text_content(Some(&format!(
                                "✅ TransformStream: upper case\n\n\
                                 Вход: hello, world, rust\n\
                                 Выходные чанки: {chunks:?}\n\
                                 Объединённый результат: {}\n\n\
                                 process_transform внутри: getWriter → три \
                                 write_with_chunk с await → close с await. \
                                 Параллельно — read_all_strings, соединённые \
                                 через futures::future::join. Без join \
                                 второй write завис бы: readable-сторона \
                                 TransformStream имеет highWaterMark = 0.",
                                chunks.join("")
                            )));
                        }
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!(
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

    // ---- Line splitter ----
    {
        let s3_result = s3_result.clone();
        callbacks::listen(
            &splitter_btn,
            "click",
            move |_| {
                let s3_result = s3_result.clone();
                s3_result.set_text_content(Some("Разбиваем на строки..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let transform = match streams::line_splitter() {
                        Ok(t) => t,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    // Вход намеренно разрезан в середине строки.
                    let input = vec![
                        "line one\nli".to_string(),
                        "ne two\nline three\n".to_string(),
                        "partial line four".to_string(),
                    ];

                    match streams::process_transform(transform, input).await {
                        Ok(chunks) => {
                            s3_result.set_text_content(Some(&format!(
                                "✅ Line splitter\n\n\
                                 Входные чанки (нарезка в середине строки):\n\
                                 • \"line one\\nli\"\n\
                                 • \"ne two\\nline three\\n\"\n\
                                 • \"partial line four\"\n\n\
                                 Выходные чанки: {chunks:?}\n\n\
                                 Объединённый результат:\n{}\n\
                                 Все четыре строки на месте, включая \
                                 последнюю — её выдал flush, когда \
                                 writable-сторона закрылась.",
                                chunks.join("")
                            )));
                        }
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!(
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

    // ---- Счётчик слов ----
    {
        let s3_result = s3_result.clone();
        callbacks::listen(
            &wordcount_btn,
            "click",
            move |_| {
                let s3_result = s3_result.clone();
                s3_result.set_text_content(Some("Считаем слова..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // Счётчик, который будет обновляться в transform.
                    let word_count: Rc<RefCell<u32>> = Rc::new(RefCell::new(0));
                    let wc = word_count.clone();

                    let transform = match streams::transform_with(move |chunk: String| {
                        let count = chunk.split_whitespace().count() as u32;
                        *wc.borrow_mut() += count;
                        // Пропускаем чанк без изменений.
                        chunk
                    }) {
                        Ok(t) => t,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let input = vec![
                        "the quick brown fox\n".to_string(),
                        "jumps over the lazy dog\n".to_string(),
                    ];

                    match streams::process_transform(transform, input).await {
                        Ok(chunks) => {
                            let count = *word_count.borrow();
                            let output = chunks.join("");
                            s3_result.set_text_content(Some(&format!(
                                "✅ Счётчик слов\n\n\
                                 Вход:\n{output}\n\
                                 Подсчитано слов: {count}\n\n\
                                 transform получает каждый чанк, считает \
                                 слова через split_whitespace и передаёт \
                                 чанк дальше без изменений. Счётчик — \
                                 Rc<RefCell<u32>>, разделяемый с внешним \
                                 кодом. После process_transform читаем \
                                 итог."
                            )));
                        }
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!(
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

    // ---- Ошибка в transform ----
    {
        let s3_result = s3_result.clone();
        callbacks::listen(
            &error_btn,
            "click",
            move |_| {
                let s3_result = s3_result.clone();
                s3_result.set_text_content(Some("Провоцируем ошибку..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let transform = match streams::transform_with_error(
                        |chunk: String| {
                            if chunk.contains("fail") {
                                Err(format!("transform failed on chunk: {chunk}"))
                            } else {
                                Ok(chunk.to_uppercase())
                            }
                        },
                    ) {
                        Ok(t) => t,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let input = vec![
                        "ok chunk".to_string(),
                        "this will fail".to_string(),
                        "never reached".to_string(),
                    ];

                    match streams::process_transform(transform, input).await {
                        Ok(_) => {
                            s3_result.set_text_content(Some(
                                "⚠️ Неожиданно: ошибка не пробросилась",
                            ));
                        }
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!(
                                "✅ Ошибка в transform проброшена\n\n\
                                 Вход: ok chunk, this will fail, never reached\n\
                                 Ошибка: {}\n\n\
                                 Что произошло:\n\
                                 • Первый чанк обработан, enqueue('OK CHUNK').\n\
                                 • На втором чанке transform вернул Err.\n\
                                 • Внутри вызвано controller.error(reason).\n\
                                 • Обе стороны TransformStream перешли \
                                 в errored-состояние.\n\
                                 • reader.read() вернул reject — \
                                 process_transform вернул Err.\n\
                                 • Третий чанк не обрабатывался.",
                                e.user_message()
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- pipeTo: обычный случай ----
    {
        let s3_result = s3_result.clone();
        callbacks::listen(
            &pipe_basic_btn,
            "click",
            move |_| {
                let s3_result = s3_result.clone();
                s3_result.set_text_content(Some("pipeTo..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let source = match streams::readable_from_strings(vec![
                        "one\n".to_string(),
                        "two\n".to_string(),
                        "three\n".to_string(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
                    let buffer_c = buffer.clone();
                    let close_called: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));
                    let close_c = close_called.clone();

                    let sink = match streams::writable_with_callbacks(
                        move |chunk| buffer_c.borrow_mut().push(chunk),
                        move || *close_c.borrow_mut() = true,
                        |_| {},
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::pipe_to(&source, &sink, false, false, false).await {
                        Ok(()) => {
                            let collected = buffer.borrow().join("");
                            let was_closed = *close_called.borrow();
                            s3_result.set_text_content(Some(&format!(
                                "✅ pipeTo: обычный случай\n\n\
                                 Содержимое приёмника:\n{collected}\n\
                                 close вызван: {was_closed}\n\n\
                                 Все опции false (по умолчанию):\n\
                                 • preventClose = false → после завершения \
                                 источника приёмник закрыт.\n\
                                 • preventAbort = false → при ошибке \
                                 источника приёмник был бы отменён.\n\
                                 • preventCancel = false → при ошибке \
                                 приёмника источник был бы отменён.\n\n\
                                 pipeTo возвращает Promise, который \
                                 разрешается, когда все данные прокачаны \
                                 и обе стороны закрыты."
                            )));
                        }
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!(
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

    // ---- pipeTo: preventClose ----
    {
        let s3_result = s3_result.clone();
        callbacks::listen(
            &pipe_prevent_btn,
            "click",
            move |_| {
                let s3_result = s3_result.clone();
                s3_result.set_text_content(Some("pipeTo с preventClose..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let source = match streams::readable_from_strings(vec![
                        "A\n".to_string(),
                        "B\n".to_string(),
                    ]) {
                        Ok(s) => s,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
                    let buffer_c = buffer.clone();
                    let close_called: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));
                    let close_c = close_called.clone();

                    let sink = match streams::writable_with_callbacks(
                        move |chunk| buffer_c.borrow_mut().push(chunk),
                        move || *close_c.borrow_mut() = true,
                        |_| {},
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::pipe_to(&source, &sink, true, false, false).await {
                        Ok(()) => {
                            let after_pipe = buffer.borrow().join("");
                            let was_closed = *close_called.borrow();

                            // Приёмник не закрыт — можем дописать вручную.
                            let writer: web_sys::WritableStreamDefaultWriter =
                                match sink.get_writer() {
                                    Ok(w) => w,
                                    Err(e) => {
                                        s3_result.set_text_content(Some(&format!(
                                            "❌ get_writer после pipeTo: {e:?}"
                                        )));
                                        return;
                                    }
                                };

                            let _ = wasm_bindgen_futures::JsFuture::from(
                                writer.write_with_chunk(&JsValue::from_str("C\n")),
                            )
                            .await;
                            let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

                            let final_content = buffer.borrow().join("");
                            s3_result.set_text_content(Some(&format!(
                                "✅ pipeTo с preventClose\n\n\
                                 После pipeTo:\n\
                                 • Содержимое: {after_pipe}\n\
                                 • close вызван: {was_closed}\n\n\
                                 Приёмник остался открытым — мы дописали \"C\\n\" \
                                 вручную:\n{final_content}\n\n\
                                 Сценарий использования: один приёмник \
                                 принимает данные из нескольких источников \
                                 последовательно. preventClose останавливает \
                                 автоматическое закрытие, чтобы можно было \
                                 продолжить писать."
                            )));
                        }
                        Err(e) => {
                            s3_result.set_text_content(Some(&format!(
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

    log!("Streams 17.3 demo ready");
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

## `tests/wasm_streams_transform.rs` — 8 тестов

```rust
//! Тесты для секции 17.3: TransformStream, pipeThrough, pipeTo.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Базовое преобразование: upper case.
#[wasm_bindgen_test]
async fn test_transform_upper() {
    let transform =
        hello_wasm02::streams::transform_with(|s: String| s.to_uppercase()).unwrap();

    let result = hello_wasm02::streams::process_transform(
        transform,
        vec!["ab".to_string(), "cd".to_string()],
    )
    .await
    .unwrap();

    assert_eq!(result.join(""), "ABCD");
}

/// Ошибка в transform доходит до читателя.
#[wasm_bindgen_test]
async fn test_transform_error_propagates() {
    let transform = hello_wasm02::streams::transform_with_error(|chunk: String| {
        if chunk.contains("fail") {
            Err("test error".to_string())
        } else {
            Ok(chunk)
        }
    })
    .unwrap();

    let result = hello_wasm02::streams::process_transform(
        transform,
        vec!["ok".to_string(), "fail".to_string()],
    )
    .await;

    assert!(result.is_err(), "ошибка в transform должна проброситься");
}

/// Без flush последний чанк теряется; с flush — нет.
#[wasm_bindgen_test]
async fn test_transform_flush() {
    // Без flush: остаток после последнего \n теряется.
    let no_flush = hello_wasm02::streams::line_splitter_no_flush().unwrap();
    let result_no_flush = hello_wasm02::streams::process_transform(
        no_flush,
        vec!["a\nb\nc".to_string()],
    )
    .await
    .unwrap();

    assert_eq!(
        result_no_flush.join(""),
        "a\nb\n",
        "без flush последняя строка теряется"
    );

    // С flush: остаток отдаётся.
    let with_flush = hello_wasm02::streams::line_splitter().unwrap();
    let result_with_flush = hello_wasm02::streams::process_transform(
        with_flush,
        vec!["a\nb\nc".to_string()],
    )
    .await
    .unwrap();

    assert_eq!(
        result_with_flush.join(""),
        "a\nb\nc",
        "с flush последняя строка сохраняется"
    );
}

/// Параллельная запись и чтение не дают deadlock.
#[wasm_bindgen_test]
async fn test_transform_parallel() {
    let transform =
        hello_wasm02::streams::transform_with(|s: String| format!("[{s}]")).unwrap();

    let input = vec![
        "a".to_string(),
        "b".to_string(),
        "c".to_string(),
        "d".to_string(),
        "e".to_string(),
    ];

    let result = hello_wasm02::streams::process_transform(transform, input)
        .await
        .unwrap();

    assert_eq!(result.join(""), "[a][b][c][d][e]");
}

/// pipeThrough соединяет источник и TransformStream.
#[wasm_bindgen_test]
async fn test_transform_pipe_through() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
    ])
    .unwrap();

    let transform =
        hello_wasm02::streams::transform_with(|s: String| format!("<{s}>")).unwrap();

    let piped = hello_wasm02::streams::call_pipe_through(&source, &transform).unwrap();
    let result = hello_wasm02::streams::read_all_strings(&piped).await.unwrap();

    assert_eq!(result.join(""), "<a><b>");
}

/// Line splitter режет вход произвольной нарезки на строки.
#[wasm_bindgen_test]
async fn test_line_splitter() {
    let transform = hello_wasm02::streams::line_splitter().unwrap();

    // Нарезка в середине слов.
    let input = vec![
        "hello\nwor".to_string(),
        "ld\nfoo\n".to_string(),
        "bar".to_string(),
    ];

    let result = hello_wasm02::streams::process_transform(transform, input)
        .await
        .unwrap();

    assert_eq!(result.join(""), "hello\nworld\nfoo\nbar");
}

/// pipeTo: обычный случай переносит все данные и закрывает приёмник.
#[wasm_bindgen_test]
async fn test_pipe_to_basic() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "x".to_string(),
        "y".to_string(),
        "z".to_string(),
    ])
    .unwrap();

    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let buffer_c = buffer.clone();
    let close_called: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));
    let close_c = close_called.clone();

    let sink = hello_wasm02::streams::writable_with_callbacks(
        move |chunk| buffer_c.borrow_mut().push(chunk),
        move || *close_c.borrow_mut() = true,
        |_| {},
    )
    .unwrap();

    hello_wasm02::streams::pipe_to(&source, &sink, false, false, false)
        .await
        .unwrap();

    assert_eq!(buffer.borrow().join(""), "xyz");
    assert!(*close_called.borrow(), "close должен быть вызван");
}

/// pipeTo с preventClose оставляет приёмник открытым.
#[wasm_bindgen_test]
async fn test_pipe_to_prevent_close() {
    let source = hello_wasm02::streams::readable_from_strings(vec![
        "a".to_string(),
        "b".to_string(),
    ])
    .unwrap();

    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let buffer_c = buffer.clone();
    let close_called: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));
    let close_c = close_called.clone();

    let sink = hello_wasm02::streams::writable_with_callbacks(
        move |chunk| buffer_c.borrow_mut().push(chunk),
        move || *close_c.borrow_mut() = true,
        |_| {},
    )
    .unwrap();

    hello_wasm02::streams::pipe_to(&source, &sink, true, false, false)
        .await
        .unwrap();

    assert_eq!(buffer.borrow().join(""), "ab");
    assert!(
        !*close_called.borrow(),
        "с preventClose приёмник не должен закрываться"
    );

    // Приёмник остался открытым — можем дописать.
    let writer: web_sys::WritableStreamDefaultWriter = sink.get_writer().unwrap();
    let _ = wasm_bindgen_futures::JsFuture::from(
        writer.write_with_chunk(&JsValue::from_str("c")),
    )
    .await;
    let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

    assert_eq!(buffer.borrow().join(""), "abc");
}
```

## Команды запуска

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown --tests

# Сборка и запуск (с очисткой — см. package.json)
npm start

# Только тесты 17.3
wasm-pack test --headless --chrome --test wasm_streams_transform
```

Ожидаемый результат: `8 passed; 0 failed`.

## Ручная проверка демо

Откройте `http://localhost:5173`, найдите секцию «3. TransformStream: конвейер»:

| Кнопка | Ожидаемый результат |
|---|---|
| «Upper case» | `HELLO WORLD RUST` |
| «Line splitter» | Четыре строки: `line one`, `line two`, `line three`, `partial line four` |
| «Счётчик слов» | Текст без изменений, «Подсчитано слов: 9» |
| «Ошибка в transform» | Сообщение с reason `transform failed on chunk: this will fail` |
| «pipeTo: обычный случай» | Приёмник заполнен `one two three`, `close вызван: true` |
| «pipeTo: preventClose» | Приёмник заполнен `A B`, `close вызван: false`, потом дописан `C` |

## Что мы не меняем

Напомню правило архитектуры: **библиотечные модули не трогаем**. Эта глава:

- не добавляет новых функций в `streams.rs`;
- не трогает `streams_wasm.rs`, `fetch.rs`, `lib.rs`;
- добавляет только новый `app.rs` и новый файл тестов.

Всё нужное уже есть в библиотеке с предыдущих глав. Если какая-то функция используется только в 17.3, она всё равно живёт в общем модуле — потому что библиотека обслуживает несколько глав сразу, как `fetch.rs` обслуживает всю часть про Fetch API.

## Итог подглавы

Мы разобрали `TransformStream` — конвейер преобразования.

**Основное:**
1. **Transformer** — JS-объект с методами `transform` и `flush`. `transform` вызывается на каждый чанк, `flush` — при закрытии входа.
2. **`controller.enqueue(output)`** отдаёт преобразованный чанк на readable-сторону. Если `transform` ничего не вызовет — чанк отбрасывается. `controller.error(e)` переводит обе стороны в errored-состояние.
3. **Backpressure.** Readable имеет `highWaterMark = 0`, поэтому писать и читать нужно **параллельно** через `futures::future::join`. Иначе второй `write` зависнет.
4. **`flush` для буферизованных преобразователей.** Line splitter без `flush` теряет последнюю строку, если вход не заканчивается `\n`.

**Соединение потоков:**
5. **`pipeThrough(transform)`** соединяет источник с `TransformStream`, возвращает новый `ReadableStream`. В Rust — через `Reflect`, потому что `web-sys 0.3.105` требует `&ReadableWritablePair`.
6. **`pipeTo(writable, options)`** соединяет источник с приёмником. Возвращает Promise.
7. **Опции `preventClose` / `preventAbort` / `preventCancel`** управляют тем, что происходит с противоположной стороной при завершении или ошибке.

**Ключевые уроки:**
- **`highWaterMark = 0` у TransformStream — источник deadlock'а.** Записывать всё, потом читать — нельзя. Только параллельно.
- **`flush` не вызывается автоматически.** Его нужно реализовать, если преобразователь буферизует данные.
- **`pipeThrough` — это `pipeTo` плюс создание нового readable.** Понимание этой связки помогает выбирать подходящий метод.
- **`preventClose` полезен, когда приёмник общий.** Один приёмник — несколько источников последовательно.
- **`Reflect` выручает при несовпадении типов.** `pipeThrough` и `pipeTo` в `web-sys 0.3.105` имеют сигнатуры, отличающиеся от JavaScript-спецификации. Прямой вызов JS-метода через `Reflect` решает проблему без правки зависимостей.