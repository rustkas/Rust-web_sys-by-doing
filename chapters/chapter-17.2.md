# 17.2. `WritableStream`: приёмник данных

## Обновлённая структура проекта

```
.
├── src/
│   ├── streams.rs                  ← ОБНОВЛЁН: +writable_with_callbacks, write_all
│   ├── lib.rs                      ← без изменений
│   ├── app.rs                      ← ОБНОВЛЁН: только секция 17.2
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_streams_writable.rs    ← НОВЫЙ: 5 тестов
│   └── (остальные без изменений)
│
├── package.json                    ← без изменений (start уже с очисткой)
└── (корневые файлы без изменений)
```

**Краткий комментарий.** `WritableStream` — приёмник данных. Он строится поверх JS-объекта с методами `write`, `close`, `abort`. `writer.write()` возвращает Promise, который разрешается, когда sink действительно принял чанк. `close()` тоже возвращает Promise — его **обязательно** нужно дожидаться. После `close()` любые попытки записи отклоняются с `TypeError`. В этой подглаве мы построим универсальную обёртку `writable_with_callbacks`, разберём все состояния writer'а и покажем, что происходит при abort с причиной. По ходу отладки мы поймаем две классические ловушки — с заимствованием `RefCell` в `match` и с приведением `JsValue` к `js_sys::Object` — и разберём их.

## Ссылки для детального изучения

- **MDN: `WritableStream`** — основной интерфейс: [developer.mozilla.org/ru/docs/Web/API/WritableStream](https://developer.mozilla.org/ru/docs/Web/API/WritableStream)
- **MDN: Underlying sink** — методы `start`, `write`, `close`, `abort`: [developer.mozilla.org/ru/docs/Web/API/WritableStream/WritableStream#parameters](https://developer.mozilla.org/ru/docs/Web/API/WritableStream/WritableStream#parameters)
- **MDN: `WritableStreamDefaultWriter`** — писатель по умолчанию: [developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter](https://developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter)
- **MDN: `writer.ready`** — Promise, разрешающийся, когда можно писать: [developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/ready](https://developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/ready)
- **MDN: `writer.closed`** — Promise, отражающий состояние потока: [developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/closed](https://developer.mozilla.org/ru/docs/Web/API/WritableStreamDefaultWriter/closed)
- **WHATWG Streams Standard: WritableStream** — спецификация: [streams.spec.whatwg.org/#ws-class](https://streams.spec.whatwg.org/#ws-class)
- **`web_sys::WritableStream`** — Rust-биндинг: [docs.rs/web-sys/latest/web_sys/struct.WritableStream.html](https://docs.rs/web-sys/latest/web_sys/struct.WritableStream.html)
- **`web_sys::WritableStreamDefaultWriter`** — Rust-биндинг для writer'а: [docs.rs/web-sys/latest/web_sys/struct.WritableStreamDefaultWriter.html](https://docs.rs/web-sys/latest/web_sys/struct.WritableStreamDefaultWriter.html)
- **`wasm_bindgen::JsCast`** — трейт `dyn_into`, без которого не обойтись при работе с `JsValue`: [docs.rs/wasm-bindgen/latest/wasm_bindgen/trait.JsCast.html](https://docs.rs/wasm-bindgen/latest/wasm_bindgen/trait.JsCast.html)

## Теоретическая часть

### Underlying sink
Обычный JS-объект с методами, который браузер оборачивает в `WritableStream`.

- **`start(controller)`** — вызывается один раз при создании. Инициализация: открытие файла, соединения.
- **`write(chunk, controller)`** — вызывается на каждый `writer.write(chunk)`. **Может возвращать Promise** — тогда следующий `write` не начнётся до его разрешения. Это и есть backpressure на стороне приёмника.
- **`close()`** — вызывается при `writer.close()`. Здесь дописывают буферизованные данные, закрывают файл.
- **`abort(reason)`** — вызывается при `writer.abort(reason)`. Освобождает ресурсы, отменяет незавершённые операции.

### `getWriter()`
`stream.getWriter()` возвращает `WritableStreamDefaultWriter`. Пока writer активен, поток **locked** — второй `getWriter()` бросит `TypeError`.

### `writer.ready` и `writer.closed`
- **`writer.ready`** — Promise. Разрешается, когда в очереди есть место и можно писать. Если очередь полна — Promise висит. Это нужно для случаев, когда приложение само решает, когда писать, а не просто спамит `write()`.
- **`writer.closed`** — Promise. Разрешается после успешного `close()`. Reject'ится, если поток перешёл в errored-состояние (например, sink бросил исключение в `write`). Через этот Promise удобно ловить асинхронные ошибки.

### Почему `close()` возвращает Promise
`close()` может потребовать flush'а: дописать буферизованные данные, дождаться завершения сетевых операций, вызвать `close` в sink. Если во время flush'а ошибка — Promise reject'ится. **Без `await` вы не узнаете о завершении и не поймаете ошибку.** В связке с `TransformStream` это критично: пока `close()` не разрешён, readable-сторона не получит событие `close`, и читатель зависнет.

### Что происходит при записи после close
- `writer.write()` после `close()` — Promise отклоняется с `TypeError`.
- `writer.abort()` после `close()` — тоже отклоняется.
- Разница между «закрыт» и «ошибочен» — через `writer.closed`: если resolve, поток закрыт нормально; если reject, произошла ошибка.

### `write_all` — удобная обёртка
Мы будем часто использовать последовательность «записать N чанков, затем закрыть». Свернём это в одну функцию:

```rust
pub async fn write_all(
    stream: &web_sys::WritableStream,
    chunks: Vec<String>,
) -> Result<(), FetchError>
```

Внутри — цикл `write_with_chunk` с `await`, потом `close` с `await`. Ошибка на любом шаге возвращается как `FetchError`.

## `src/streams.rs` — новые функции

```rust
// ============================================================
// 17.2: WritableStream — приёмник данных
// ============================================================

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::fetch::{classify_error, FetchError};

/// Создаёт `WritableStream` с Rust-колбэками.
///
/// `on_write` вызывается на каждый записанный чанк.
/// `on_close` — когда поток закрывается.
/// `on_abort` — когда поток отменяется; получает reason, если он был.
pub fn writable_with_callbacks<W, C, A>(
    on_write: W,
    on_close: C,
    on_abort: A,
) -> Result<web_sys::WritableStream, JsValue>
where
    W: FnMut(String) + 'static,
    C: FnMut() + 'static,
    A: FnMut(Option<String>) + 'static,
{
    let on_write = Rc::new(RefCell::new(on_write));
    let on_close = Rc::new(RefCell::new(on_close));
    let on_abort = Rc::new(RefCell::new(on_abort));

    let write_callback = {
        let on_write = on_write.clone();
        Closure::<dyn FnMut(JsValue)>::wrap(Box::new(move |chunk: JsValue| {
            let text = chunk.as_string().unwrap_or_default();
            (on_write.borrow_mut())(text);
        }))
    };

    let close_callback = {
        let on_close = on_close.clone();
        Closure::<dyn FnMut()>::wrap(Box::new(move || {
            (on_close.borrow_mut())();
        }))
    };

    let abort_callback = {
        let on_abort = on_abort.clone();
        Closure::<dyn FnMut(JsValue)>::wrap(Box::new(move |reason: JsValue| {
            let reason_str = if reason.is_undefined() || reason.is_null() {
                None
            } else {
                reason.as_string()
            };
            (on_abort.borrow_mut())(reason_str);
        }))
    };

    let sink = js_sys::Object::new();
    js_sys::Reflect::set(
        &sink,
        &"write".into(),
        write_callback.as_ref().unchecked_ref(),
    )
    .ok();
    js_sys::Reflect::set(
        &sink,
        &"close".into(),
        close_callback.as_ref().unchecked_ref(),
    )
    .ok();
    js_sys::Reflect::set(
        &sink,
        &"abort".into(),
        abort_callback.as_ref().unchecked_ref(),
    )
    .ok();

    // Замыкания должны жить, пока жив поток. forget() перемещает их
    // во внутренний статический слот wasm-bindgen.
    write_callback.forget();
    close_callback.forget();
    abort_callback.forget();

    web_sys::WritableStream::new_with_underlying_sink(&sink)
}

/// Записывает все чанки в поток и закрывает его.
///
/// Возвращает `Ok(())` только после того, как:
/// 1. Все `write_with_chunk` разрешились.
/// 2. `close()` разрешился — то есть sink действительно закрыт.
pub async fn write_all(
    stream: &web_sys::WritableStream,
    chunks: Vec<String>,
) -> Result<(), FetchError> {
    let writer: web_sys::WritableStreamDefaultWriter =
        stream.get_writer().map_err(classify_error)?;

    for chunk in chunks {
        let promise = writer.write_with_chunk(&JsValue::from_str(&chunk));
        wasm_bindgen_futures::JsFuture::from(promise)
            .await
            .map_err(classify_error)?;
    }

    let close_promise = writer.close();
    wasm_bindgen_futures::JsFuture::from(close_promise)
        .await
        .map_err(classify_error)?;

    Ok(())
}
```

## `src/app.rs` — только секция 17.2

```rust
//! Демонстрация главы 17.2: `WritableStream` как приёмник данных.
//!
//! Одна секция с тремя кнопками:
//! 1. Записать три чанка.
//! 2. Записать после close — показать TypeError.
//! 3. Abort с причиной.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::callbacks::{self, EventClosures};
use crate::{dom, log, streams};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "WritableStream: приёмник данных")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "WritableStream — приёмник данных. Он строится поверх JS-объекта \
         с методами write, close, abort. writer.write() возвращает \
         Promise, который разрешается, когда sink принял чанк. \
         close() тоже возвращает Promise. После close() запись \
         отклоняется с TypeError.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 2: WritableStream — приёмник
    // ============================================================

    let s2 = build_section(&container, "2. WritableStream: приёмник")?;

    let s2_hint = dom::create_paragraph(
        "Underlying sink получает методы write, close и abort. \
         write вызывается на каждый writer.write(chunk). Если sink \
         вернёт Promise — следующий write не начнётся, пока Promise \
         не разрешится. Это backpressure на стороне приёмника.",
    )?;
    dom::add_class(&dom::as_html(&s2_hint), "hint")?;
    dom::append(&s2, &s2_hint)?;

    let write_chunks_btn = dom::create_element("button")?;
    write_chunks_btn.set_text_content(Some("Записать три чанка"));
    let write_after_close_btn = dom::create_element("button")?;
    write_after_close_btn.set_text_content(Some("Записать после close"));
    let abort_btn = dom::create_element("button")?;
    abort_btn.set_text_content(Some("Abort с причиной"));

    dom::append(&s2, &write_chunks_btn)?;
    dom::append(&s2, &write_after_close_btn)?;
    dom::append(&s2, &abort_btn)?;

    let s2_result = make_result(&s2)?;

    // ---- Записать три чанка ----
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &write_chunks_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Пишем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
                    let closed: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));

                    let buffer_c = buffer.clone();
                    let closed_c = closed.clone();

                    let stream = match streams::writable_with_callbacks(
                        move |chunk| buffer_c.borrow_mut().push(chunk),
                        move || *closed_c.borrow_mut() = true,
                        |_reason| {},
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    match streams::write_all(
                        &stream,
                        vec![
                            "Alpha\n".to_string(),
                            "Beta\n".to_string(),
                            "Gamma\n".to_string(),
                        ],
                    )
                    .await
                    {
                        Ok(()) => {
                            let collected = buffer.borrow().join("");
                            let was_closed = *closed.borrow();
                            s2_result.set_text_content(Some(&format!(
                                "✅ WritableStream: три чанка\n\n\
                                 Записано: Alpha, Beta, Gamma\n\
                                 Содержимое буфера:\n{collected}\n\
                                 close вызван: {was_closed}\n\n\
                                 write_all внутри: getWriter, три \
                                 write_with_chunk с await, close с await. \
                                 Каждый write возвращает Promise, который \
                                 разрешается только после того, как sink \
                                 принял чанк."
                            )));
                        }
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!(
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

    // ---- Записать после close ----
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &write_after_close_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Пишем и закрываем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
                    let buffer_c = buffer.clone();

                    let stream = match streams::writable_with_callbacks(
                        move |chunk| buffer_c.borrow_mut().push(chunk),
                        || {},
                        |_| {},
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let writer: web_sys::WritableStreamDefaultWriter =
                        match stream.get_writer() {
                            Ok(w) => w,
                            Err(e) => {
                                s2_result.set_text_content(Some(&format!(
                                    "❌ get_writer: {e:?}"
                                )));
                                return;
                            }
                        };

                    // Первый write — успех.
                    let _ = wasm_bindgen_futures::JsFuture::from(
                        writer.write_with_chunk(&JsValue::from_str("X\n")),
                    )
                    .await;

                    // Close — обязательно await.
                    let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

                    // Второй write — должен упасть.
                    let result = wasm_bindgen_futures::JsFuture::from(
                        writer.write_with_chunk(&JsValue::from_str("Y\n")),
                    )
                    .await;

                    match result {
                        Ok(_) => {
                            s2_result.set_text_content(Some(
                                "⚠️ Неожиданно: write после close не упал",
                            ));
                        }
                        Err(e) => {
                            let msg = e
                                .dyn_ref::<js_sys::Error>()
                                .map(|err| String::from(err.message()))
                                .unwrap_or_else(|| format!("{e:?}"));

                            s2_result.set_text_content(Some(&format!(
                                "✅ Ожидаемая ошибка после close\n\n\
                                 Попытка write после close отклонена.\n\
                                 Сообщение браузера: {msg}\n\n\
                                 Что произошло:\n\
                                 • writer.write() → getWriter().write_with_chunk()\n\
                                 • После close() состояние writer'а — 'closed'.\n\
                                 • Любая попытка write или abort отклоняется \
                                 с TypeError.\n\
                                 • Это защита от случайной записи после \
                                 завершения потока.\n\n\
                                 В реальном приложении здесь вы бы \
                                 сбросили UI или создали новый поток."
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Abort с причиной ----
    {
        let s2_result = s2_result.clone();
        callbacks::listen(
            &abort_btn,
            "click",
            move |_| {
                let s2_result = s2_result.clone();
                s2_result.set_text_content(Some("Отменяем..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let abort_reason: Rc<RefCell<Option<String>>> =
                        Rc::new(RefCell::new(None));
                    let abort_reason_c = abort_reason.clone();

                    let stream = match streams::writable_with_callbacks(
                        |_chunk| {},
                        || {},
                        move |reason| *abort_reason_c.borrow_mut() = reason,
                    ) {
                        Ok(s) => s,
                        Err(e) => {
                            s2_result.set_text_content(Some(&format!("❌ {e:?}")));
                            return;
                        }
                    };

                    let writer: web_sys::WritableStreamDefaultWriter =
                        match stream.get_writer() {
                            Ok(w) => w,
                            Err(e) => {
                                s2_result.set_text_content(Some(&format!(
                                    "❌ get_writer: {e:?}"
                                )));
                                return;
                            }
                        };

                    // Пишем один чанк до abort.
                    let _ = wasm_bindgen_futures::JsFuture::from(
                        writer.write_with_chunk(&JsValue::from_str("X\n")),
                    )
                    .await;

                    // Abort с причиной.
                    let _ = wasm_bindgen_futures::JsFuture::from(
                        writer.abort_with_reason(&JsValue::from_str("user cancelled")),
                    )
                    .await;

                    // Копируем значение из RefCell и сразу отпускаем borrow.
                    // Иначе временный Ref живёт до конца match и конфликтует
                    // с set_text_content, который заимствует другие данные.
                    let reason_opt: Option<String> = abort_reason.borrow().clone();

                    match reason_opt {
                        Some(reason) => {
                            s2_result.set_text_content(Some(&format!(
                                "✅ Abort сработал\n\n\
                                 Переданная причина: {reason}\n\n\
                                 Что произошло:\n\
                                 • writer.abort_with_reason('user cancelled')\n\
                                 • Sink получил вызов abort(reason) \
                                 с этой причиной.\n\
                                 • Поток перешёл в состояние 'errored'.\n\
                                 • writer.closed теперь reject'ится.\n\n\
                                 Abort отличается от close тем, что \
                                 сигналит об ошибке, а не о нормальном \
                                 завершении. Все незавершённые операции \
                                 отменяются."
                            )));
                        }
                        None => {
                            s2_result.set_text_content(Some(
                                "❌ on_abort не был вызван",
                            ));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    log!("Streams 17.2 demo ready");
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

## `tests/wasm_streams_writable.rs` — 5 тестов

```rust
//! Тесты для секции 17.2: WritableStream как приёмник данных.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Все чанки дошли до sink в правильном порядке.
#[wasm_bindgen_test]
async fn test_write_multiple_chunks() {
    let buffer: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let buffer_c = buffer.clone();

    let stream = hello_wasm02::streams::writable_with_callbacks(
        move |chunk| buffer_c.borrow_mut().push(chunk),
        || {},
        |_| {},
    )
    .unwrap();

    hello_wasm02::streams::write_all(
        &stream,
        vec!["a".to_string(), "b".to_string(), "c".to_string()],
    )
    .await
    .unwrap();

    let collected = buffer.borrow().join("");
    assert_eq!(collected, "abc");
}

/// close-колбэк вызывается при writer.close().
#[wasm_bindgen_test]
async fn test_close_calls_close_callback() {
    let close_called: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));
    let close_called_c = close_called.clone();

    let stream = hello_wasm02::streams::writable_with_callbacks(
        |_chunk| {},
        move || *close_called_c.borrow_mut() = true,
        |_| {},
    )
    .unwrap();

    hello_wasm02::streams::write_all(&stream, vec!["x".to_string()])
        .await
        .unwrap();

    assert!(
        *close_called.borrow(),
        "close callback должен быть вызван после write_all"
    );
}

/// Запись после close отклоняется с ошибкой.
#[wasm_bindgen_test]
async fn test_write_after_close_fails() {
    let stream = hello_wasm02::streams::writable_with_callbacks(
        |_chunk| {},
        || {},
        |_| {},
    )
    .unwrap();

    let writer: web_sys::WritableStreamDefaultWriter =
        stream.get_writer().unwrap();

    // Первый write успешен.
    let _ = wasm_bindgen_futures::JsFuture::from(
        writer.write_with_chunk(&JsValue::from_str("first")),
    )
    .await;

    // Close.
    let _ = wasm_bindgen_futures::JsFuture::from(writer.close()).await;

    // Второй write должен упасть.
    let result = wasm_bindgen_futures::JsFuture::from(
        writer.write_with_chunk(&JsValue::from_str("second")),
    )
    .await;

    assert!(result.is_err(), "write после close должен вернуть ошибку");
}

/// abort-колбэк вызывается с переданной причиной.
#[wasm_bindgen_test]
async fn test_abort_calls_abort_callback() {
    let abort_reason: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
    let abort_reason_c = abort_reason.clone();

    let stream = hello_wasm02::streams::writable_with_callbacks(
        |_chunk| {},
        || {},
        move |reason| *abort_reason_c.borrow_mut() = reason,
    )
    .unwrap();

    let writer: web_sys::WritableStreamDefaultWriter =
        stream.get_writer().unwrap();

    let _ = wasm_bindgen_futures::JsFuture::from(
        writer.abort_with_reason(&JsValue::from_str("test abort")),
    )
    .await;

    // Копируем значение из RefCell и сразу отпускаем borrow.
    let reason: Option<String> = abort_reason.borrow().clone();
    assert_eq!(reason.as_deref(), Some("test abort"));
}

/// Ошибка в sink.write прокидывается наружу.
///
/// Используем `js_sys::eval`, чтобы создать sink, чей write бросает
/// исключение. Через Rust-замыкание бросить JS-исключение из `write`
/// нельзя — Closure имеет фиксированную сигнатуру без Result.
#[wasm_bindgen_test]
async fn test_error_in_write_propagates() {
    // js_sys::eval возвращает JsValue, а new_with_underlying_sink
    // ожидает &js_sys::Object. Приводим тип через dyn_into.
    let sink: js_sys::Object = js_sys::eval(
        r#"({
            write(chunk) {
                if (String(chunk).includes('fail')) {
                    throw new Error('sink write failed');
                }
            },
            close() {},
            abort() {},
        })"#,
    )
    .unwrap()
    .dyn_into()
    .expect("eval result is not an Object");

    let stream =
        web_sys::WritableStream::new_with_underlying_sink(&sink).unwrap();

    let writer: web_sys::WritableStreamDefaultWriter =
        stream.get_writer().unwrap();

    // Первый write — успех.
    let _ = wasm_bindgen_futures::JsFuture::from(
        writer.write_with_chunk(&JsValue::from_str("ok")),
    )
    .await;

    // Второй write содержит "fail" — должен упасть.
    let result = wasm_bindgen_futures::JsFuture::from(
        writer.write_with_chunk(&JsValue::from_str("this will fail")),
    )
    .await;

    assert!(result.is_err(), "write с 'fail' должен вернуть ошибку");

    // Извлекаем сообщение.
    if let Err(e) = result {
        if let Some(err) = e.dyn_ref::<js_sys::Error>() {
            let msg = String::from(err.message());
            assert!(
                msg.contains("sink write failed"),
                "ожидалось сообщение 'sink write failed', получено: {msg}"
            );
        }
    }
}
```

## Команды запуска

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown --tests

# Сборка и запуск (с очисткой)
npm start

# Только тесты 17.2
wasm-pack test --headless --chrome --test wasm_streams_writable
```

Ожидаемый результат тестов: `5 passed; 0 failed`.

## Ручная проверка демо

Откройте `http://localhost:5173`, найдите секцию «2. WritableStream: приёмник»:

| Кнопка | Ожидаемый результат |
|---|---|
| «Записать три чанка» | Буфер с `Alpha Beta Gamma`, `close вызван: true` |
| «Записать после close» | Ошибка TypeError: «The writer is closed» |
| «Abort с причиной» | «Переданная причина: user cancelled» |

## Две ловушки, которые мы поймали при отладке

### Ловушка 1. `RefCell` и `match` — заимствование живёт до конца блока

Первый вариант обработчика «Abort с причиной» использовал:

```rust
match abort_reason.borrow().as_ref() {
    Some(reason) => { /* set_text_content с format! */ }
    None => { /* set_text_content */ }
}
```

Компилятор выдал:

```
error: `abort_reason` does not live long enough
borrowed value does not live long enough
```

Причина: `.borrow()` создаёт временный `Ref`, который живёт до конца **всего** выражения `match`, включая ветку `None`. В обеих ветках мы вызываем `set_text_content`, которая заимствует другие данные — компилятор не может доказать, что конфликта нет.

**Решение:** скопировать значение из `RefCell` в локальную переменную, потом работать с ней:

```rust
let reason_opt: Option<String> = abort_reason.borrow().clone();
match reason_opt { ... }
```

Временный `Ref` уничтожается сразу после `.clone()`, дальше никаких заимствований из `RefCell` нет.

Это третья встреча с одной и той же ловушкой в книге (16.7 — `active_controller`, 17.1 — `RefCell` и `await`). Общее правило: **если в блоке используется что-то ещё, что заимствует данные, — сначала скопируйте, потом работайте**.

### Ловушка 2. `js_sys::eval` возвращает `JsValue`, а не `Object`

В тесте `test_error_in_write_propagates` мы хотели создать sink с методом `write`, который бросает исключение. Через `Closure<dyn FnMut(JsValue)>` это невозможно — сигнатура без `Result`. Пришлось использовать `js_sys::eval` и вернуть JS-объект.

Но `WritableStream::new_with_underlying_sink` ожидает `&js_sys::Object`, а `eval` возвращает `JsValue`. Компилятор выдал:

```
error: expected &Object<JsValue>, found &JsValue
```

**Решение:** явное приведение через `dyn_into`:

```rust
let sink: js_sys::Object = js_sys::eval(...)
    .unwrap()
    .dyn_into()
    .expect("eval result is not an Object");
```

Для этого нужен трейт `JsCast`, который подключается через `use wasm_bindgen::JsCast;`. Без него `dyn_into` не виден.

Это общее правило при работе с `web-sys` и `js-sys`: **везде, где API ожидает конкретный тип, а у вас `JsValue`, — приводите через `dyn_into`**.

## Итог подглавы

Мы разобрали `WritableStream` как приёмник данных:

1. **Underlying sink** — JS-объект с методами `write`, `close`, `abort`. `write` может быть асинхронным — это даёт backpressure на приёмнике.
2. **`getWriter()`** возвращает `WritableStreamDefaultWriter`. Поток становится locked.
3. **`write_with_chunk`** возвращает Promise, который разрешается, когда sink принял чанк.
4. **`close()`** возвращает Promise. **Обязательно `await`** — иначе читаемая сторона (в `TransformStream`) не получит сигнал завершения.
5. **`abort_with_reason(reason)`** отменяет поток с указанием причины. Sink получает `abort(reason)`.
6. **После `close()`** любые `write` или `abort` отклоняются с `TypeError`.
7. **`writer.ready`** и **`writer.closed`** — Promise'ы для контроля состояния. Они понадобятся в 17.4 (Backpressure).

Ключевые уроки:

- **`close()` без `await` — источник багов.** В главе 17.3 мы увидим, как это приводит к зависанию `TransformStream`.
- **Sink может быть асинхронным.** Возврат Promise из `write` — стандартный способ задать backpressure на приёмнике.
- **Ошибка в `write` переводит поток в errored-состояние.** Все последующие операции отклоняются.
- **`RefCell` и `match` несовместимы без клонирования.** Временный `Ref` живёт до конца выражения — копируйте значение.
- **`JsValue` и `js_sys::Object` — разные типы.** API `web-sys` ожидает конкретные типы, а `eval` возвращает общий `JsValue`. Всегда приводите через `dyn_into`.
- **Обёртка `write_all` экономит код.** Вместо пяти строк на каждый вызов — одна. Мы будем переиспользовать её в 17.3, 17.5 и 17.7.