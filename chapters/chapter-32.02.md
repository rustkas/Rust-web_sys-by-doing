# Глава 32.2. Асинхронный `ClipboardItem` — что реально работает

## 32.2.1. Что вы получите в этой главе

В главе 32 мы писали `ClipboardItem` с уже готовыми `Blob`. Казалось, что это единственная форма. Оказалось — есть ещё две:

1. **`Promise<Blob>` в конструкторе.** Запись стартует **до** того, как данные готовы. Пока браузер держит буфер открытым, генерация идёт в фоне.
2. **Массив `ClipboardItem[]` в `write()`.** Spec допускает несколько независимых представлений в одном вызове.

За эту главу вы:

- Разберёте, чем `Promise<Blob>` отличается от готового `Blob` — и почему это **работающая** техника, применимая в любом браузере.
- Честно увидите, что **массив items не реализован** ни в Chrome, ни в Firefox. Не как курьёз, а как факт: spec написан, реализация отстаёт.
- Освоите диагностику ошибок, специфичных для браузера, через **классификацию по паттернам**, а не по точной строке.
- Обнаружите **расхождение размеров PNG** между движками в 3.4 раза.
- Соберёте демо с живым прогрессом, фазовыми индикаторами и карточкой вердикта.

**Главный тезис.** `Promise<Blob>` — это то, что имеет ценность. Массив items — то, что нужно знать, но не нужно использовать. Отрицательный результат исследования — тоже результат.

## 32.2.2. Теоретическая справка — три уровня `ClipboardItem`

`ClipboardItem` эволюционировал через три уровня:

**Уровень 1 — синхронный item.** `new ClipboardItem({ "text/plain": blob })`. Все данные готовы до вызова. Разобран в главе 32.

**Уровень 2 — асинхронный item.** `new ClipboardItem({ "image/png": promise })`, где `promise: Promise<Blob>`. Запись стартует немедленно, но завершается, когда промис резолвится. UI остаётся отзывчивым. **Работает в Chrome, Edge, Brave, Opera, Firefox, Safari.**

**Уровень 3 — массив items.** `clipboard.write([item1, item2])`. Spec описывает как массив независимых представлений. **Не реализовано ни в одном браузере на 2024-й год.** Chrome и Firefox падают с `NotAllowedError`.

| Что даёт | Синхронный `Blob` | `Promise<Blob>` | Массив items |
|---|---|---|---|
| Данные до `write` | готовы | обещаны | — |
| UI блокируется | да, если генерация длинная | нет | — |
| Прогресс возможен | после генерации | да | — |
| Реализовано | везде | везде | нигде |

**Что даёт `Promise<Blob>` в реальности.**

- Пользователь копирует скриншот — генерация PNG занимает 500 мс. UI свободен, прогресс виден.
- Приложение обрабатывает картинку — ресайз, водяной знак. 2000 мс в фоне. Запись стартует сразу.
- Сервер вернул данные — ждать нельзя, буфер обмена должен быть уже занят. Promise решает.

**Что даёт массив items — в теории.**

- Приложение «поделиться фрагментом» кладёт `[markdown_item, html_item, plain_item]` — получатель сам выберет подходящее.
- Заметки: `[text_item, image_item]` — в мессенджер уйдёт картинка, в блокнот — текст.

**Что даёт массив items — на практике.** Ничего. Читатель, увидевший пример в MDN и попробовавший его в консоли, потратит час на отладку и подумает, что делает что-то не так. Это и есть причина написать эту главу.

## 32.2.3. Структура файлов

```
hello-wasm02/
├── src/
│   ├── clip_async.rs       # НОВЫЙ: Promise<Blob>, ClipboardItem, write
│   ├── lib.rs              # +1 модуль
│   └── app.rs              # демо — два эксперимента
├── tests/
│   └── wasm_clip_async.rs  # НОВЫЙ: wasm-тесты
└── Cargo.toml              # без изменений
```

**Комментарий.** Модуль `clip_async` не пересекается с `clip_core` (глава 32) и `paste_evt` (глава 32.1). `clip_core` работает с готовыми данными, `paste_evt` — с событиями, `clip_async` — с асинхронной генерацией. Все нужные `web-sys`-фичи уже в `Cargo.toml`.

**Ссылки для изучения:**
- MDN, `ClipboardItem` — https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem
- MDN, `ClipboardItem.supports()` — https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem/supports_static
- MDN, `Clipboard.write()` — https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/write
- Spec, «ClipboardItem» — https://w3c.github.io/clipboard-apis/#clipboard-item-interface

## 32.2.4. Исходный код `src/clip_async.rs`

```rust
//! Async ClipboardItem: `Promise<Blob>` и множественные items.
//!
//! `ClipboardItem` (глава 32) принимает готовый `Blob` — данные должны
//! быть собраны до `clipboard.write()`. Для генерации изображения это
//! проблема: 500-2000 мс блокировки между «хочу скопировать» и
//! «write()». UI зависает.
//!
//! Async ClipboardItem решает две задачи:
//!
//!   1. `Promise<Blob>` в конструкторе. `clipboard.write()` стартует
//!      сразу, но завершается, когда промис резолвится. UI остаётся
//!      отзывчивым. Мы можем показывать прогресс.
//!
//!   2. Массив `ClipboardItem[]` в `write()`. Несколько альтернативных
//!      представлений. Получатель (приложение при вставке) сам
//!      выбирает подходящее.
//!
//! ПРАВИЛО: `ClipboardItem` — класс, вызывается через
//! `js_sys::Reflect::construct`.
//!
//! ПРАВИЛО: значения в объекте-источнике могут быть как `Blob`, так
//! и `Promise<Blob>`. Оба варианта валидны.
//!
//! ПРАВИЛО: `canvas.toBlob` — callback API. Оборачиваем в `Promise`
//! через `Closure::once_into_js`.

use std::cell::RefCell;
use std::rc::Rc;

use js_sys::{Array, Function, Promise};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

// ============================================================
//                    ДОСТУП
// ============================================================

fn clipboard() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))
        .map_err(|e| JsValue::from_str(&format!("Reflect navigator: {e:?}")))?;
    let clip = js_sys::Reflect::get(&nav, &JsValue::from_str("clipboard"))
        .map_err(|e| JsValue::from_str(&format!("Reflect clipboard: {e:?}")))?;
    if clip.is_undefined() || clip.is_null() {
        return Err(JsValue::from_str(
            "navigator.clipboard недоступен (нужен HTTPS или localhost)",
        ));
    }
    Ok(clip)
}

/// true, если `navigator.clipboard` доступен.
pub fn is_available() -> bool {
    return clipboard().is_ok();
}

/// true, если браузер разрешает записывать этот MIME в `ClipboardItem`.
///
/// Использует статический `ClipboardItem.supports(mime)`. Если метод
/// недоступен — `true` (оптимистично).
pub fn supports_mime(mime: &str) -> bool {
    let global = js_sys::global();
    let ctor = match js_sys::Reflect::get(&global, &JsValue::from_str("ClipboardItem")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return true,
    };
    let supports = match js_sys::Reflect::get(&ctor, &JsValue::from_str("supports")) {
        Ok(v) if !v.is_undefined() => v,
        _ => return true,
    };
    let f: Function = match supports.dyn_into() {
        Ok(f) => f,
        Err(_) => return true,
    };
    f.call1(&ctor, &JsValue::from_str(mime))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(true)
}

// ============================================================
//                    PROMISE PNG С ЗАДЕРЖКОЙ
// ============================================================

/// Создаёт `Promise<Blob>` с PNG размером `size × size`, который
/// резолвится через `delay_ms` миллисекунд.
///
/// Пока промис не резолвлен, его можно передать в `ClipboardItem` —
/// запись стартует сразу, а данные придут когда будут готовы.
///
/// `on_ready` вызывается с размером готового `Blob` в байтах — для
/// логирования и UI. Вызывается ровно один раз.
///
/// ПРАВИЛО: `Promise::new` создаёт `FnMut`-замыкание. `on_ready` —
/// `FnOnce`, поэтому его нельзя переместить в callback напрямую.
/// Оборачиваем в `Rc<RefCell<Option<F>>>` и вызываем через `take()`.
pub fn make_png_promise<F>(size: u32, delay_ms: u32, on_ready: F) -> Result<Promise, JsValue>
where
    F: FnOnce(u64) + 'static,
{
    let on_ready_cell: Rc<RefCell<Option<F>>> = Rc::new(RefCell::new(Some(on_ready)));

    let promise = Promise::new(&mut move |resolve, reject| {
        let resolve_outer = resolve.clone();
        let reject_outer = reject.clone();
        let on_ready_for_cb = on_ready_cell.clone();

        let cb = Closure::once_into_js(move || {
            let canvas = match create_painted_canvas(size) {
                Ok(c) => c,
                Err(e) => {
                    let _ = reject_outer.call1(&JsValue::UNDEFINED, &e);
                    return;
                }
            };
            let resolve_final = resolve_outer.clone();
            let reject_final = reject_outer.clone();
            let on_ready_inner = on_ready_for_cb.clone();

            let on_blob = Closure::once_into_js(move |blob: JsValue| {
                if blob.is_null() || blob.is_undefined() {
                    let _ = reject_final.call1(
                        &JsValue::UNDEFINED,
                        &JsValue::from_str("canvas.toBlob вернул null"),
                    );
                    return;
                }
                if let Some(b) = blob.dyn_ref::<web_sys::Blob>() {
                    // take() — вызываем FnOnce ровно один раз.
                    // Повторный вызов не сработает: Option уже None.
                    if let Some(f) = on_ready_inner.borrow_mut().take() {
                        f(b.size() as u64);
                    }
                }
                let _ = resolve_final.call1(&JsValue::UNDEFINED, &blob);
            });

            let to_blob: Function =
                match js_sys::Reflect::get(&canvas, &JsValue::from_str("toBlob"))
                    .and_then(|v| v.dyn_into::<Function>())
                {
                    Ok(f) => f,
                    Err(_) => {
                        let _ = reject_outer.call1(
                            &JsValue::UNDEFINED,
                            &JsValue::from_str("toBlob не функция"),
                        );
                        return;
                    }
                };
            let _ = to_blob.call2(
                &canvas,
                on_blob.unchecked_ref(),
                &JsValue::from_str("image/png"),
            );
        });

        let win = match web_sys::window() {
            Some(w) => w,
            None => return,
        };
        let _ = win.set_timeout_with_callback_and_timeout_and_arguments_0(
            cb.unchecked_ref(),
            delay_ms as i32,
        );
    });
    Ok(promise)
}

/// Рисует canvas — градиент, круг, цифра.
fn create_painted_canvas(size: u32) -> Result<web_sys::HtmlCanvasElement, JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let canvas = document
        .create_element("canvas")?
        .dyn_into::<web_sys::HtmlCanvasElement>()
        .map_err(|_| JsValue::from_str("createElement('canvas') не canvas"))?;
    canvas.set_width(size);
    canvas.set_height(size);
    let ctx = canvas
        .get_context("2d")?
        .ok_or("no 2d context")?
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .map_err(|_| JsValue::from_str("ctx не CanvasRenderingContext2d"))?;

    let gradient = ctx.create_linear_gradient(0.0, 0.0, size as f64, size as f64);
    let _ = gradient.add_color_stop(0.0, "#3498db");
    let _ = gradient.add_color_stop(1.0, "#9b59b6");
    ctx.set_fill_style_canvas_gradient(&gradient);
    ctx.fill_rect(0.0, 0.0, size as f64, size as f64);

    let r = size as f64 / 2.0;
    ctx.set_fill_style_str("rgba(255,255,255,0.8)");
    ctx.begin_path();
    let _ = ctx.arc(r, r, r * 0.5, 0.0, std::f64::consts::PI * 2.0);
    ctx.fill();

    let label = size.to_string();
    ctx.set_fill_style_str("#fff");
    ctx.set_font("bold 14px system-ui");
    ctx.set_text_align("center");
    ctx.set_text_baseline("middle");
    let _ = ctx.fill_text(&label, r, r);

    Ok(canvas)
}

// ============================================================
//                    BLOB ДЛЯ ТЕКСТА
// ============================================================

/// Создаёт `Blob` из строки с указанным MIME.
pub fn make_text_blob(text: &str, mime: &str) -> Result<web_sys::Blob, JsValue> {
    let parts = Array::new();
    parts.push(&JsValue::from_str(text));
    let opts = web_sys::BlobPropertyBag::new();
    opts.set_type(mime);
    web_sys::Blob::new_with_str_sequence_and_options(&parts, &opts)
        .map_err(|e| JsValue::from_str(&format!("Blob::new: {e:?}")))
}

// ============================================================
//                    CLIPBOARD ITEM
// ============================================================

/// Создаёт `ClipboardItem` из набора источников.
///
/// Каждое значение может быть как `Blob`, так и `Promise<Blob>`.
/// `mime` — ключ объекта-источника; например, `"image/png"` или
/// `"text/plain"`.
pub fn make_item(sources: &[(String, JsValue)]) -> Result<JsValue, JsValue> {
    let data = js_sys::Object::new();
    for (mime, value) in sources {
        js_sys::Reflect::set(&data, &JsValue::from_str(mime), value)
            .map_err(|e| JsValue::from_str(&format!("set {mime}: {e:?}")))?;
    }
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("ClipboardItem"))
        .map_err(|e| JsValue::from_str(&format!("Reflect ClipboardItem: {e:?}")))?;
    if ctor.is_undefined() || ctor.is_null() {
        return Err(JsValue::from_str("ClipboardItem недоступен"));
    }
    let ctor: Function = ctor
        .dyn_into()
        .map_err(|_| JsValue::from_str("ClipboardItem не конструктор"))?;
    let args = Array::new();
    args.push(&data);
    js_sys::Reflect::construct(&ctor, &args)
        .map_err(|e| JsValue::from_str(&format!("new ClipboardItem: {e:?}")))
}

// ============================================================
//                    WRITE
// ============================================================

/// Вызывает `clipboard.write(items)` и возвращает **сырой** `Promise`.
///
/// Вызывающий сам решает, как ждать: через `JsFuture::from` или
/// параллельно с другими задачами — например, обновляя прогресс-бар.
pub fn write_items_promise(items: &[JsValue]) -> Result<Promise, JsValue> {
    let clip = clipboard()?;
    let arr = Array::new();
    for item in items {
        arr.push(item);
    }
    let write_fn: Function = js_sys::Reflect::get(&clip, &JsValue::from_str("write"))
        .map_err(|e| JsValue::from_str(&format!("Reflect write: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str("write не функция"))?;
    let promise_val = write_fn
        .call1(&clip, &arr)
        .map_err(|e| JsValue::from_str(&format!("write: {e:?}")))?;
    promise_val
        .dyn_into::<Promise>()
        .map_err(|_| JsValue::from_str("write не вернул Promise"))
}

/// Удобная обёртка: вызывает `write` и ждёт резолва.
///
/// Для сценариев, где не нужен прогресс.
pub async fn write_items(items: &[JsValue]) -> Result<(), JsValue> {
    let promise = write_items_promise(items)?;
    JsFuture::from(promise).await?;
    Ok(())
}
```

## 32.2.5. Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod clip_async;
```

## 32.2.6. Тесты — `tests/wasm_clip_async.rs`

```rust
//! Wasm-тесты `clip_async`: Blob для текста, ClipboardItem из
//! Promise<Blob>, поддержка MIME, поведение с задержкой.
//!
//! ПРАВИЛО: wasm-тесты называются с префиксом `wasm_` в имени файла.
//! ПРАВИЛО: `format!` в tail-позиции — через явный `return`.
//! ПРАВИЛО: headless-тесты не могут эмулировать user gesture —
//! операции записи в буфер помечены `#[ignore]`.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::clip_async;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- доступность ----------------

#[wasm_bindgen_test]
fn api_available_in_headless() {
    assert!(
        clip_async::is_available(),
        "navigator.clipboard должен быть доступен в headless Chrome на localhost",
    );
}

#[wasm_bindgen_test]
fn supports_text_plain() {
    assert!(
        clip_async::supports_mime("text/plain"),
        "text/plain должен поддерживаться",
    );
}

#[wasm_bindgen_test]
fn supports_text_html() {
    assert!(
        clip_async::supports_mime("text/html"),
        "text/html должен поддерживаться",
    );
}

// ---------------- text blob ----------------

#[wasm_bindgen_test]
fn text_blob_has_correct_mime() {
    let blob = clip_async::make_text_blob("hello", "text/plain").expect("blob");
    assert_eq!(blob.type_(), "text/plain");
    assert_eq!(blob.size() as u32, 5);
}

#[wasm_bindgen_test]
fn text_blob_empty_string() {
    let blob = clip_async::make_text_blob("", "text/plain").expect("blob");
    assert_eq!(blob.size() as u32, 0);
}

// ---------------- make_item ----------------

#[wasm_bindgen_test]
fn make_item_from_blob_returns_object() {
    let blob = clip_async::make_text_blob("hi", "text/plain").expect("blob");
    let item = clip_async::make_item(&[("text/plain".to_string(), blob.into())])
        .expect("item");
    assert!(item.is_object(), "ClipboardItem должен быть объектом");

    // У ClipboardItem есть поле `types` — массив MIME.
    let types = js_sys::Reflect::get(&item, &JsValue::from_str("types"))
        .expect("types");
    let arr = types.dyn_into::<js_sys::Array>().expect("types — Array");
    assert_eq!(arr.length(), 1);
    assert_eq!(arr.get(0).as_string().as_deref(), Some("text/plain"));
}

#[wasm_bindgen_test]
fn make_item_multi_format_single_item() {
    let text_blob = clip_async::make_text_blob("hi", "text/plain").expect("text blob");
    let html_blob = clip_async::make_text_blob("<b>hi</b>", "text/html").expect("html blob");
    let item = clip_async::make_item(&[
        ("text/plain".to_string(), text_blob.into()),
        ("text/html".to_string(), html_blob.into()),
    ])
    .expect("item");

    let types = js_sys::Reflect::get(&item, &JsValue::from_str("types"))
        .expect("types");
    let arr = types.dyn_into::<js_sys::Array>().expect("types — Array");
    assert_eq!(arr.length(), 2, "item должен содержать 2 MIME-типа");
}

// ---------------- png promise ----------------

#[wasm_bindgen_test]
async fn png_promise_resolves_with_blob() {
    // Быстрая задержка — тест не должен ждать секунды.
    let promise = clip_async::make_png_promise(64, 50, |_sz| {})
        .expect("png promise");

    let blob_val = wasm_bindgen_futures::JsFuture::from(promise)
        .await
        .expect("await png promise");
    let blob = blob_val.dyn_into::<web_sys::Blob>().expect("Blob");
    assert_eq!(blob.type_(), "image/png");
    assert!(blob.size() > 0.0, "PNG должен быть непустым");
}

#[wasm_bindgen_test]
async fn png_promise_callback_reports_size() {
    // Проверяем, что on_ready получает размер.
    let observed: std::rc::Rc<std::cell::RefCell<Option<u64>>> =
        std::rc::Rc::new(std::cell::RefCell::new(None));
    let obs = observed.clone();

    let promise = clip_async::make_png_promise(64, 50, move |sz| {
        *obs.borrow_mut() = Some(sz);
    })
    .expect("png promise");

    wasm_bindgen_futures::JsFuture::from(promise)
        .await
        .expect("await");

    assert!(
        observed.borrow().is_some(),
        "on_ready должен был получить размер",
    );
    assert!(
        observed.borrow().unwrap() > 0,
        "размер должен быть больше нуля",
    );
}

// ---------------- запись в буфер (требует разрешения) ----------------

#[wasm_bindgen_test]
#[ignore = "требует clipboard-write permission в headless"]
async fn write_single_text_item_works() {
    let blob = clip_async::make_text_blob("test write", "text/plain").expect("blob");
    let item = clip_async::make_item(&[("text/plain".to_string(), blob.into())])
        .expect("item");
    clip_async::write_items(&[item])
        .await
        .expect("write_items");
}

#[wasm_bindgen_test]
#[ignore = "требует clipboard-write permission в headless"]
async fn write_two_items_works() {
    let text_blob = clip_async::make_text_blob("test", "text/plain").expect("text");
    let text_item = clip_async::make_item(&[("text/plain".to_string(), text_blob.into())])
        .expect("text item");
    let png_promise = clip_async::make_png_promise(64, 50, |_| {}).expect("png promise");
    let png_item = clip_async::make_item(&[("image/png".to_string(), png_promise.into())])
        .expect("png item");
    clip_async::write_items(&[text_item, png_item])
        .await
        .expect("write_items");
}
```

**Команды запуска:**

```bash
# Проверка типов
cargo check
cargo check --target wasm32-unknown-unknown --tests

# Тесты
wasm-pack test --headless --chrome  --test wasm_clip_async
wasm-pack test --headless --firefox --test wasm_clip_async

# Группа по подстроке
wasm-pack test --headless --chrome --test wasm_clip_async -- text_blob
wasm-pack test --headless --chrome --test wasm_clip_async -- png_promise
wasm-pack test --headless --chrome --test wasm_clip_async -- make_item

# С игнорируемыми
wasm-pack test --headless --chrome --test wasm_clip_async -- --ignored
```

## 32.2.7. Демо — `src/app.rs`

Демо — **три панели**:

1. **Promise\<Blob\>.** Прогресс-бар, три фазы (создать → write → данные в буфере), превью PNG после резолва.
2. **Массив items — spec vs reality.** Две кнопки с явными бейджами до клика. После клика — **крупная цветная карточка вердикта**.
3. **Журнал** без сырых `NotAllowedError` — только человеческие формулировки.

**Ключевые архитектурные решения:**

- **Живой счётчик мс** обновляется каждые 50 мс через `TimeoutFuture::new(COUNTDOWN_TICK_MS).await`. Без yield UI не перерисовывается — правило проекта.
- **Фазы с анимацией** — активная пульсирует, пройденная зелёная. Пользователь видит, где сейчас находится операция.
- **Бейджи `❌ не поддерживается` / `✅ работает` — до клика.** Пользователь не гадает, что работает.
- **Карточка вердикта** — крупный HTML-блок после клика. Зелёная рамка для успеха, красная для ошибки. Не нужно читать журнал, чтобы понять результат.
- **`is_multiple_items_unsupported` — классификация по паттернам.** Chrome: `only one clipboarditem`, Firefox: `support for multiple clipboarditem`. Точное сравнение сломается при следующем обновлении движка.
- **ObjectURL трекается и освобождается** при следующей генерации PNG.
- **`detect_browser()` через UA-sniffing** — для UI-подсказок этого достаточно, но в рантайме поведение определяется по факту ошибки, а не по имени браузера.

**`src/app.rs`:**

```
//! Демо Главы 32.2: Async ClipboardItem — Promise<Blob> и реальность.
//!
//! Два блока:
//!   1. Promise<Blob> — write() стартует сразу, данные приходят через
//!      N миллисекунд. Live-счётчик показывает ожидание. После резолва —
//!      превью PNG.
//!   2. Массив items — бинарный вердикт: spec не поддерживается
//!      ни Chrome, ни Firefox. Workaround (один item с двумя MIME) —
//!      работает. Явная зелёная/красная карточка после клика.
//!
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`.
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через `push_str`.
//! ПРАВИЛО: ObjectURL трекается и освобождается при следующей генерации.
//! ПРАВИЛО: имя браузера — из `detect_browser()`, не из хардкода.
//! ПРАВИЛО: классификация ошибок — по паттернам, не по точной строке.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;

use crate::clip_async;
use crate::format_str::format_str;

const LOG_LIMIT: usize = 12;
const COUNTDOWN_TICK_MS: u32 = 50;

const HTML: &str = r#"
<style>
  :root {
    --ca-info: #3498db; --ca-ok: #27ae60; --ca-err: #c0392b; --ca-warn: #e67e22;
    --ca-text: #222; --ca-muted: #666;
    --ca-bg: #fff; --ca-soft: #fafafa; --ca-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --ca-text: #e6e6e6; --ca-muted: #a0a0a0;
      --ca-bg: #1e1e1e; --ca-soft: #262626; --ca-border: #3a3a3a;
    }
    body { background: #121212; }
    .ca-anno { background: #1a2233; color: #9fc4ff; }
    .ca-anno code { background: rgba(52,152,219,0.2); }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .ca-log-entry { background: #1e1e1e; }
    progress { background: #2a2a2a; }
    .ca-phase { background: #1e1e1e; color: var(--ca-muted); }
    .ca-phase.active { background: #1a3a5a; color: #cfe0ff; border-color: var(--ca-info); }
    .ca-phase.done { background: #1a3a1a; color: #b8e0b8; border-color: var(--ca-ok); }
    .ca-browser-badge.chromium { background: rgba(39,174,96,0.18); color: #b8e0b8; }
    .ca-browser-badge.firefox  { background: rgba(230,126,34,0.2); color: #f0d8a0; }
    .ca-browser-badge.safari   { background: rgba(230,126,34,0.2); color: #f0d8a0; }
    .ca-browser-badge.other    { background: rgba(128,128,128,0.2); color: #aaa; }
  }
  html, body { width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important; overflow-x: hidden; }
  body {
    display: block !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--ca-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app {
    display: block !important; width: 100% !important; max-width: 100% !important;
    margin: 0 !important; padding: 12px 16px 24px !important;
    box-sizing: border-box !important;
  }
  #app, #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .ca-subtitle { color: var(--ca-muted); font-size: 12px; margin: 0 0 12px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .ca-subtitle code { background: rgba(128,128,128,0.15);
    padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .ca-subtitle a { color: var(--ca-info); }
  .ca-browser-badge {
    display: inline-block; padding: 2px 10px; border-radius: 10px;
    font-family: ui-monospace, monospace; font-size: 11px; font-weight: 700;
  }
  .ca-browser-badge.chromium { background: #e8f8ed; color: #1e8449; }
  .ca-browser-badge.firefox  { background: #fff3cd; color: #7d5a00; }
  .ca-browser-badge.safari   { background: #fff3cd; color: #7d5a00; }
  .ca-browser-badge.other    { background: #f0f0f0; color: #666; }

  .ca-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(0, 1.1fr) minmax(0, 1.1fr) minmax(0, 0.9fr);
  }
  @media (max-width: 1100px) { .ca-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px) { .ca-grid { grid-template-columns: 1fr; } }
  .ca-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .ca-panel {
    border: 1px solid var(--ca-border); border-radius: 8px;
    padding: 10px 12px; background: var(--ca-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0;
  }
  .ca-panel-title { font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .ca-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--ca-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .ca-anno { background: #eef4ff; border-left: 3px solid var(--ca-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere; word-break: break-word; }
  .ca-anno code { background: rgba(52,152,219,0.12);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10px; }
  .ca-anno b { color: #0d3b66; }
  @media (prefers-color-scheme: dark) { .ca-anno b { color: #c0d8ff; } }

  .ca-form-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; min-width: 0; }
  input[type="text"], select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--ca-bg); color: var(--ca-text);
    font-family: ui-monospace, monospace;
    flex: 1 1 120px; min-width: 0;
  }
  input[type="text"]:focus, select:focus {
    outline: none; border-color: var(--ca-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }
  label.ca-field {
    display: flex; align-items: center; gap: 6px;
    font-size: 11px; color: var(--ca-muted); flex-shrink: 0;
  }
  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--ca-soft); color: var(--ca-text);
    cursor: pointer; transition: background 0.15s;
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--ca-info); outline-offset: 2px; }
  button.ca-primary { background: #eaf4ff; border-color: var(--ca-info); color: #1f4e79; }
  button.ca-success { background: #e8f8ed; border-color: var(--ca-ok); color: #1e8449; }

  .ca-phases {
    display: flex; gap: 6px; flex-wrap: wrap; align-items: center;
    margin: 8px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .ca-phase {
    padding: 3px 8px; border-radius: 4px;
    border: 1px solid var(--ca-border);
    background: var(--ca-soft); color: var(--ca-muted);
    transition: all 0.25s;
    white-space: nowrap;
  }
  .ca-phase.active {
    border-color: var(--ca-info); background: #eaf4ff; color: #1f4e79;
    font-weight: 700;
    animation: ca-pulse 1.2s infinite;
  }
  .ca-phase.done {
    border-color: var(--ca-ok); background: #e8f8ed; color: #1e8449;
  }
  @keyframes ca-pulse {
    0%, 100% { box-shadow: 0 0 0 0 rgba(52,152,219,0.5); }
    60% { box-shadow: 0 0 0 6px rgba(52,152,219,0); }
  }
  .ca-arrow { color: var(--ca-muted); font-size: 12px; }

  .ca-progress-wrap {
    display: flex; align-items: center; gap: 8px; margin: 8px 0 4px;
  }
  progress {
    flex: 1 1 auto; height: 14px;
    appearance: none; border: none; border-radius: 4px; overflow: hidden;
    background: #eee;
  }
  progress::-webkit-progress-bar { background: #eee; border-radius: 4px; }
  progress::-webkit-progress-value {
    background: var(--ca-info); border-radius: 4px;
    transition: width 0.05s linear;
  }
  progress::-moz-progress-bar { background: var(--ca-info); border-radius: 4px; }
  .ca-progress-label {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--ca-muted); flex-shrink: 0; min-width: 80px; text-align: right;
  }

  .ca-status {
    display: flex; align-items: center; gap: 6px;
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--ca-soft); border-left: 3px solid #ccc;
    color: var(--ca-muted);
    min-height: 22px; overflow-wrap: anywhere; word-break: break-word;
  }
  .ca-status.ok   { background: #e8f8ed; border-color: var(--ca-ok); color: #1e8449; }
  .ca-status.err  { background: #fdedec; border-color: var(--ca-err); color: #922b21; }
  .ca-status.busy { background: #fff9e6; border-color: var(--ca-warn); color: #7d5a00; }
  .ca-status.warn { background: #fff9e6; border-color: var(--ca-warn); color: #7d5a00; }

  .ca-preview-empty {
    padding: 14px; text-align: center; color: #999;
    font-style: italic; font-size: 11px;
    border: 1px dashed var(--ca-border); border-radius: 5px;
    margin-top: 8px;
  }
  .ca-preview-png {
    margin-top: 8px; padding: 8px;
    border: 1px solid var(--ca-border); border-radius: 5px;
    background: var(--ca-soft);
    display: flex; align-items: center; gap: 10px;
  }
  .ca-preview-png img {
    display: block;
    max-width: 96px; max-height: 96px;
    border-radius: 4px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.15);
  }
  .ca-preview-png-meta {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--ca-muted); line-height: 1.6;
  }
  .ca-preview-png-meta b { color: var(--ca-text); }

  /* ============ Бейдж статуса кнопки ============ */
  .ca-btn-with-badge {
    display: flex; align-items: center; gap: 8px;
    padding: 6px 10px; margin: 6px 0;
    border: 1px solid var(--ca-border); border-radius: 5px;
    background: var(--ca-soft);
  }
  .ca-btn-with-badge button { flex: 0 0 auto; margin: 0; }
  .ca-btn-badge {
    padding: 2px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 10px; font-weight: 700;
  }
  .ca-btn-badge.no {
    background: #fdedec; color: #922b21;
    border: 1px solid #f5c6cb;
  }
  .ca-btn-badge.yes {
    background: #e8f8ed; color: #1e8449;
    border: 1px solid #c3e6cb;
  }
  @media (prefers-color-scheme: dark) {
    .ca-btn-badge.no { background: #2a1a1a; color: #e0b8b8; border-color: #4a2a2a; }
    .ca-btn-badge.yes { background: #1a2a1a; color: #b8e0b8; border-color: #2a4a2a; }
  }
  .ca-btn-hint {
    margin-left: auto; font-size: 10.5px; color: var(--ca-muted);
    font-style: italic;
  }

  /* ============ Результат (карточка вердикта) ============ */
  .ca-verdict {
    margin: 10px 0;
    padding: 12px 14px; border-radius: 6px;
    font-family: ui-monospace, monospace; font-size: 12px;
    line-height: 1.6;
  }
  .ca-verdict.pending {
    background: var(--ca-soft); color: var(--ca-muted);
    border: 1px dashed var(--ca-border);
  }
  .ca-verdict.fail {
    background: #fdedec; color: #922b21;
    border: 2px solid var(--ca-err);
  }
  .ca-verdict.success {
    background: #e8f8ed; color: #1e8449;
    border: 2px solid var(--ca-ok);
  }
  @media (prefers-color-scheme: dark) {
    .ca-verdict.fail { background: #2a1a1a; color: #e0b8b8; }
    .ca-verdict.success { background: #1a2a1a; color: #b8e0b8; }
  }
  .ca-verdict-icon { font-size: 20px; margin-right: 8px; vertical-align: middle; }
  .ca-verdict-title { font-size: 14px; font-weight: 700; }
  .ca-verdict-body { margin-top: 6px; }
  .ca-verdict-body code {
    background: rgba(0,0,0,0.08); padding: 0 4px; border-radius: 3px;
    font-size: 11px;
  }
  @media (prefers-color-scheme: dark) {
    .ca-verdict-body code { background: rgba(255,255,255,0.1); }
  }

  .ca-log {
    border: 1px solid var(--ca-border); border-radius: 5px;
    background: var(--ca-soft); max-height: 500px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .ca-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .ca-log-entry { display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent; }
  .ca-log-entry.ok    { border-left-color: var(--ca-ok); }
  .ca-log-entry.err   { border-left-color: var(--ca-err); }
  .ca-log-entry.info  { border-left-color: var(--ca-info); }
  .ca-log-entry.warn  { border-left-color: var(--ca-warn); }
  .ca-log-entry .t { color: var(--ca-muted); flex-shrink: 0; }
  .ca-log-entry .m { color: var(--ca-text); flex: 1; overflow-wrap: anywhere; }
  .ca-log-entry.err .m { color: #922b21; }
  .ca-log-entry.warn .m { color: #7d5a00; }
  @media (prefers-color-scheme: dark) {
    .ca-log-entry.warn .m { color: #e6c98c; }
  }
</style>

<h1>Глава 32.2: Async ClipboardItem</h1>
<p class="ca-subtitle">
  <code>Promise&lt;Blob&gt;</code> — запись стартует до готовности данных.
  Массив <code>ClipboardItem[]</code> — spec vs reality.
  <span id="ca-browser-badge" class="ca-browser-badge other">…</span>
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem" target="_blank" rel="noopener">MDN →</a>
</p>

<div class="ca-grid">

  <!-- ============ Колонка 1: Promise<Blob> ============ -->
  <div class="ca-col">
    <div class="ca-panel">
      <h2 class="ca-panel-title">
        <span class="ca-num">1</span>Promise&lt;Blob&gt; — запись без ожидания
      </h2>
      <div class="ca-anno">
        <code>new ClipboardItem({ "image/png": Promise&lt;Blob&gt; })</code>.
        Запись стартует <b>сразу</b>, но завершается, когда промис резолвится.
        Live-счётчик показывает ожидание. UI остаётся отзывчивым —
        генерируй в фоне сколько нужно.
      </div>

      <div class="ca-form-row">
        <label class="ca-field">
          Размер PNG:
          <select id="ca-size">
            <option value="64">64×64</option>
            <option value="128" selected>128×128</option>
            <option value="256">256×256</option>
          </select>
        </label>
        <label class="ca-field">
          Задержка:
          <select id="ca-delay">
            <option value="500">500 мс</option>
            <option value="1000">1 сек</option>
            <option value="2000" selected>2 сек</option>
            <option value="4000">4 сек</option>
          </select>
        </label>
      </div>

      <div class="ca-form-row">
        <button id="ca-write-png" class="ca-primary">
          📋 Записать PNG через Promise
        </button>
      </div>

      <div class="ca-phases">
        <span class="ca-phase" id="ca-phase-1">1. создать Promise</span>
        <span class="ca-arrow">→</span>
        <span class="ca-phase" id="ca-phase-2">2. write()</span>
        <span class="ca-arrow">→</span>
        <span class="ca-phase" id="ca-phase-3">3. данные в буфере</span>
      </div>

      <div class="ca-progress-wrap">
        <progress id="ca-progress" value="0" max="100"></progress>
        <span class="ca-progress-label" id="ca-progress-label">—</span>
      </div>

      <div id="ca-status-1" class="ca-status">готово</div>

      <div id="ca-preview-1" class="ca-preview-empty">
        После записи здесь появится превью PNG из буфера
      </div>
    </div>
  </div>

  <!-- ============ Колонка 2: Массив items ============ -->
  <div class="ca-col">
    <div class="ca-panel">
      <h2 class="ca-panel-title">
        <span class="ca-num">2</span>Массив items — spec vs reality
      </h2>
      <div class="ca-anno">
        Spec допускает <code>clipboard.write([item1, item2])</code> —
        массив независимых items. Реальность: <b>Chrome и Firefox его не
        реализовали</b>. Две кнопки ниже показывают границу. Жмите
        по очереди — вердикт появится крупной карточкой.
      </div>

      <div class="ca-form-row">
        <input type="text" id="ca-text" value="Привет из Clipboard API" placeholder="текст">
      </div>

      <div class="ca-btn-with-badge">
        <button id="ca-write-spec" class="ca-primary">
          Попробовать spec
        </button>
        <span class="ca-btn-badge no">❌ не поддерживается</span>
        <span class="ca-btn-hint">Chrome · Firefox</span>
      </div>

      <div class="ca-btn-with-badge">
        <button id="ca-write-workaround" class="ca-success">
          Workaround
        </button>
        <span class="ca-btn-badge yes">✅ работает</span>
        <span class="ca-btn-hint">1 item · 2 MIME</span>
      </div>

      <div id="ca-verdict" class="ca-verdict pending">
        Нажмите любую кнопку — здесь появится вердикт
      </div>
    </div>
  </div>

  <!-- ============ Колонка 3: журнал ============ -->
  <div class="ca-col">
    <div class="ca-panel">
      <h2 class="ca-panel-title">Журнал</h2>
      <div id="ca-log" class="ca-log">
        <div class="ca-log-empty">здесь появятся операции с буфером</div>
      </div>
    </div>
  </div>

</div>
"#;

// ============================================================
//                    БРАУЗЕР
// ============================================================

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
enum Browser { Chromium, Firefox, Safari, Other }

impl Browser {
    fn label(&self) -> &'static str {
        match self {
            Browser::Chromium => "Chromium (Chrome/Edge/Brave/Opera)",
            Browser::Firefox => "Firefox",
            Browser::Safari => "Safari",
            Browser::Other => "неизвестный",
        }
    }
    fn short(&self) -> &'static str {
        match self {
            Browser::Chromium => "Chromium",
            Browser::Firefox => "Firefox",
            Browser::Safari => "Safari",
            Browser::Other => "браузер",
        }
    }
    fn badge_class(&self) -> &'static str {
        match self {
            Browser::Chromium => "chromium",
            Browser::Firefox => "firefox",
            Browser::Safari => "safari",
            Browser::Other => "other",
        }
    }
}

fn detect_browser() -> Browser {
    let ua = user_agent().to_lowercase();
    if ua.contains("firefox") || ua.contains("fxios") {
        return Browser::Firefox;
    }
    if ua.contains("edg/") || ua.contains("chrome/") || ua.contains("chromium") || ua.contains("crios") {
        return Browser::Chromium;
    }
    if ua.contains("safari") {
        return Browser::Safari;
    }
    return Browser::Other;
}

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info, Warn }

struct LogEntry { time: String, text: String, kind: LogKind }

struct Ui {
    size: web_sys::HtmlSelectElement,
    delay: web_sys::HtmlSelectElement,
    btn_write_png: web_sys::HtmlButtonElement,
    phase1: web_sys::Element,
    phase2: web_sys::Element,
    phase3: web_sys::Element,
    progress: web_sys::Element,
    progress_label: web_sys::Element,
    status1: web_sys::Element,
    preview1: web_sys::Element,

    text_input: web_sys::HtmlInputElement,
    btn_write_spec: web_sys::HtmlButtonElement,
    btn_write_workaround: web_sys::HtmlButtonElement,
    verdict: web_sys::Element,

    browser_badge: web_sys::Element,
    log: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    busy: Rc<Cell<bool>>,
    last_png_url: Rc<RefCell<Option<String>>>,
    ui: Ui,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        size: get_select("ca-size")?,
        delay: get_select("ca-delay")?,
        btn_write_png: get_button("ca-write-png")?,
        phase1: get_el("ca-phase-1")?,
        phase2: get_el("ca-phase-2")?,
        phase3: get_el("ca-phase-3")?,
        progress: get_el("ca-progress")?,
        progress_label: get_el("ca-progress-label")?,
        status1: get_el("ca-status-1")?,
        preview1: get_el("ca-preview-1")?,
        text_input: get_input("ca-text")?,
        btn_write_spec: get_button("ca-write-spec")?,
        btn_write_workaround: get_button("ca-write-workaround")?,
        verdict: get_el("ca-verdict")?,
        browser_badge: get_el("ca-browser-badge")?,
        log: get_el("ca-log")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        busy: Rc::new(Cell::new(false)),
        last_png_url: Rc::new(RefCell::new(None)),
        ui,
    });

    render_browser_badge(&app);
    app.render_log();
    app.push_log(LogKind::Info, "начните с эксперимента 1 или 2".to_string());
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "браузер определён как: {}",
            detect_browser().label(),
        )),
    );

    if !clip_async::is_available() {
        app.push_log(LogKind::Err,
            "navigator.clipboard недоступен — нужен HTTPS или localhost".to_string());
    } else {
        app.push_log(LogKind::Info, "navigator.clipboard доступен".to_string());
    }

    // Кнопка Promise<Blob>.
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_write_png, move |_| {
            let app_in = app_c.clone();
            if app_in.busy.get() { return; }
            let size: u32 = app_in.ui.size.value().parse().unwrap_or(128);
            let delay: u32 = app_in.ui.delay.value().parse().unwrap_or(2000);
            spawn_local(async move { run_promise_experiment(app_in, size, delay).await; });
        });
    }

    // Кнопка spec.
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_write_spec, move |_| {
            let app_in = app_c.clone();
            if app_in.busy.get() { return; }
            let text = app_in.ui.text_input.value();
            spawn_local(async move { run_spec_attempt(app_in, text).await; });
        });
    }

    // Кнопка workaround.
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_write_workaround, move |_| {
            let app_in = app_c.clone();
            if app_in.busy.get() { return; }
            let text = app_in.ui.text_input.value();
            spawn_local(async move { run_workaround_attempt(app_in, text).await; });
        });
    }

    Ok(())
}

fn render_browser_badge(app: &Rc<App>) {
    let b = detect_browser();
    let cls = format_str(format_args!("ca-browser-badge {}", b.badge_class()));
    app.ui.browser_badge.set_class_name(&cls);
    app.ui.browser_badge.set_text_content(Some(b.short()));
    let _ = app.ui.browser_badge.set_attribute("title", b.label());
}

// ============================================================
//                    ЭКСПЕРИМЕНТ 1: Promise<Blob>
// ============================================================

async fn run_promise_experiment(app: Rc<App>, size: u32, delay: u32) {
    app.busy.set(true);
    app.ui.btn_write_png.set_disabled(true);

    set_phase(&app.ui.phase1, "active");
    set_phase(&app.ui.phase2, "");
    set_phase(&app.ui.phase3, "");
    set_progress(&app, 0.0, "0 мс");
    set_status(&app.ui.status1, "busy", "создание Promise…");
    app.push_log(LogKind::Info, format_str(format_args!(
        "PNG {size}×{size}, задержка {delay} мс — старт",
    )));

    let active: Rc<Cell<bool>> = Rc::new(Cell::new(true));
    {
        let app_c = app.clone();
        let active_c = active.clone();
        spawn_local(async move { countdown_loop(app_c, active_c, delay).await; });
    }

    let png_size_cell: Rc<Cell<u64>> = Rc::new(Cell::new(0));
    let png_size_cb = png_size_cell.clone();
    let promise = match clip_async::make_png_promise(size, delay, move |sz| {
        png_size_cb.set(sz);
    }) {
        Ok(p) => p,
        Err(e) => {
            active.set(false);
            set_phase(&app.ui.phase1, "");
            let msg = format_str(format_args!("Promise: {}", short_error(&e)));
            set_status(&app.ui.status1, "err", &msg);
            app.push_log(LogKind::Err, msg);
            app.busy.set(false);
            app.ui.btn_write_png.set_disabled(false);
            return;
        }
    };

    set_phase(&app.ui.phase1, "done");
    set_phase(&app.ui.phase2, "active");
    set_status(&app.ui.status1, "busy", "clipboard.write()…");

    let sources = vec![("image/png".to_string(), promise.into())];
    let item = match clip_async::make_item(&sources) {
        Ok(i) => i,
        Err(e) => {
            active.set(false);
            set_phase(&app.ui.phase2, "");
            let msg = format_str(format_args!("make_item: {}", short_error(&e)));
            set_status(&app.ui.status1, "err", &msg);
            app.push_log(LogKind::Err, msg);
            app.busy.set(false);
            app.ui.btn_write_png.set_disabled(false);
            return;
        }
    };

    let write_promise = match clip_async::write_items_promise(&[item]) {
        Ok(p) => p,
        Err(e) => {
            active.set(false);
            set_phase(&app.ui.phase2, "");
            let msg = format_str(format_args!("write(): {}", short_error(&e)));
            set_status(&app.ui.status1, "err", &msg);
            app.push_log(LogKind::Err, msg);
            app.busy.set(false);
            app.ui.btn_write_png.set_disabled(false);
            return;
        }
    };

    set_phase(&app.ui.phase2, "done");
    set_phase(&app.ui.phase3, "active");
    set_status(&app.ui.status1, "busy", "write() в полёте — ждём генерации данных");

    let t0 = now_ms();
    let result = wasm_bindgen_futures::JsFuture::from(write_promise).await;
    let elapsed = now_ms() - t0;
    active.set(false);

    match result {
        Ok(_) => {
            let sz = png_size_cell.get();
            set_phase(&app.ui.phase3, "done");
            set_progress(&app, 100.0, "готово");
            let msg = format_str(format_args!(
                "PNG {size}×{size} записан за {:.0} мс ({} байт)",
                elapsed, fmt_bytes(sz),
            ));
            app.push_log(LogKind::Ok, msg.clone());
            set_status(&app.ui.status1, "ok", &msg);
            render_png_preview(&app, size).await;
        }
        Err(e) => {
            set_phase(&app.ui.phase3, "");
            set_progress(&app, 0.0, "—");
            let msg = format_str(format_args!("write: {}", short_error(&e)));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status1, "err", &msg);
        }
    }

    app.busy.set(false);
    app.ui.btn_write_png.set_disabled(false);
}

async fn countdown_loop(app: Rc<App>, active: Rc<Cell<bool>>, delay: u32) {
    let start = now_ms();
    while active.get() {
        let elapsed = now_ms() - start;
        let remaining = (delay as f64 - elapsed).max(0.0);
        let pct = (elapsed / delay as f64 * 100.0).min(100.0);
        let label = if remaining > 0.0 {
            format_str(format_args!("{:.0} мс", remaining))
        } else {
            "ожидание данных…".to_string()
        };
        set_progress(&app, pct, &label);
        TimeoutFuture::new(COUNTDOWN_TICK_MS).await;
    }
}

async fn render_png_preview(app: &Rc<App>, size: u32) {
    revoke_png_url(app);
    let promise = match clip_async::make_png_promise(size, 0, |_| {}) {
        Ok(p) => p, Err(_) => return,
    };
    let blob_val = match wasm_bindgen_futures::JsFuture::from(promise).await {
        Ok(v) => v, Err(_) => return,
    };
    let blob = match blob_val.dyn_into::<web_sys::Blob>() {
        Ok(b) => b, Err(_) => return,
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u, Err(_) => return,
    };
    *app.last_png_url.borrow_mut() = Some(url.clone());

    let mut html = String::with_capacity(320);
    html.push_str("<img src=\"");
    html.push_str(&esc_html(&url));
    html.push_str("\" alt=\"PNG preview\" width=\"");
    html.push_str(&size.to_string());
    html.push_str("\" height=\"");
    html.push_str(&size.to_string());
    html.push_str("\">");
    html.push_str("<div class=\"ca-preview-png-meta\"><b>");
    html.push_str(&format_str(format_args!("PNG {size}×{size}")));
    html.push_str("</b><br>размер: ");
    html.push_str(&fmt_bytes(blob.size() as u64));
    html.push_str("<br>MIME: image/png</div>");

    app.ui.preview1.set_class_name("ca-preview-png");
    app.ui.preview1.set_inner_html(&html);
}

fn revoke_png_url(app: &Rc<App>) {
    if let Some(url) = app.last_png_url.borrow_mut().take() {
        let _ = web_sys::Url::revoke_object_url(&url);
    }
}

// ============================================================
//                    ЭКСПЕРИМЕНТ 2: Массив items
// ============================================================

async fn run_spec_attempt(app: Rc<App>, text: String) {
    if text.is_empty() {
        set_verdict_fail(&app, "Пустой текст", "Введите текст в поле выше.");
        return;
    }
    app.busy.set(true);
    app.ui.btn_write_spec.set_disabled(true);
    app.ui.btn_write_workaround.set_disabled(true);

    set_verdict_pending(&app, "Идёт попытка spec-варианта…");
    app.push_log(LogKind::Info, "spec: write([item1, item2])".to_string());

    let items = match build_two_items(&text) {
        Ok(v) => v,
        Err(msg) => {
            set_verdict_fail(&app, "Ошибка подготовки", &msg);
            app.push_log(LogKind::Err, msg);
            app.busy.set(false);
            app.ui.btn_write_spec.set_disabled(false);
            app.ui.btn_write_workaround.set_disabled(false);
            return;
        }
    };

    let t0 = now_ms();
    let result = clip_async::write_items(&items).await;
    let elapsed = now_ms() - t0;

    match result {
        Ok(()) => {
            let msg = format_str(format_args!("spec сработал за {:.0} мс", elapsed));
            app.push_log(LogKind::Ok, msg.clone());
            set_verdict_success(&app, "Spec сработал!", &msg);
        }
        Err(e) => {
            let raw = short_error(&e);
            if is_multiple_items_unsupported(&raw) {
                let b = detect_browser();
                let title = format_str(format_args!(
                    "{} не поддерживает массив items",
                    b.short(),
                ));
                let body = "Это spec-feature, ожидающий реализации. \
                            Не баг вашего кода. Для совместимости — кнопка Workaround ниже.";
                app.push_log(LogKind::Warn, format_str(format_args!("spec: {body}")));
                set_verdict_fail(&app, &title, body);
            } else {
                let title = "Неизвестная ошибка";
                app.push_log(LogKind::Err, format_str(format_args!("spec: {raw}")));
                set_verdict_fail(&app, title, &raw);
            }
        }
    }

    app.busy.set(false);
    app.ui.btn_write_spec.set_disabled(false);
    app.ui.btn_write_workaround.set_disabled(false);
}

async fn run_workaround_attempt(app: Rc<App>, text: String) {
    if text.is_empty() {
        set_verdict_fail(&app, "Пустой текст", "Введите текст в поле выше.");
        return;
    }
    app.busy.set(true);
    app.ui.btn_write_spec.set_disabled(true);
    app.ui.btn_write_workaround.set_disabled(true);

    set_verdict_pending(&app, "Идёт попытка workaround-варианта…");
    app.push_log(LogKind::Info, "workaround: write([item{image/png, text/plain}])".to_string());

    let item = match build_single_item_multi_mime(&text) {
        Ok(i) => i,
        Err(msg) => {
            set_verdict_fail(&app, "Ошибка подготовки", &msg);
            app.push_log(LogKind::Err, msg);
            app.busy.set(false);
            app.ui.btn_write_spec.set_disabled(false);
            app.ui.btn_write_workaround.set_disabled(false);
            return;
        }
    };

    let t0 = now_ms();
    let result = clip_async::write_items(&[item]).await;
    let elapsed = now_ms() - t0;

    match result {
        Ok(()) => {
            let title = "Записано в буфер";
            let body = format_str(format_args!(
                "Один ClipboardItem с двумя MIME-типами. Время: {:.0} мс.\n\
                 В буфере лежат: image/png и text/plain.\n\
                 Блокнот возьмёт текст, графический редактор — картинку.",
                elapsed,
            ));
            app.push_log(LogKind::Ok, format_str(format_args!(
                "workaround сработал за {:.0} мс",
                elapsed,
            )));
            set_verdict_success(&app, title, &body);
        }
        Err(e) => {
            let raw = short_error(&e);
            app.push_log(LogKind::Err, format_str(format_args!("workaround: {raw}")));
            set_verdict_fail(&app, "Workaround не сработал", &raw);
        }
    }

    app.busy.set(false);
    app.ui.btn_write_spec.set_disabled(false);
    app.ui.btn_write_workaround.set_disabled(false);
}

fn build_two_items(text: &str) -> Result<Vec<JsValue>, String> {
    let text_blob = clip_async::make_text_blob(text, "text/plain")
        .map_err(|e| format_str(format_args!("text blob: {}", short_error(&e))))?;
    let text_item = clip_async::make_item(&[("text/plain".to_string(), text_blob.into())])
        .map_err(|e| format_str(format_args!("text item: {}", short_error(&e))))?;

    let png_promise = clip_async::make_png_promise(128, 200, |_| {})
        .map_err(|e| format_str(format_args!("png promise: {}", short_error(&e))))?;
    let png_item = clip_async::make_item(&[("image/png".to_string(), png_promise.into())])
        .map_err(|e| format_str(format_args!("png item: {}", short_error(&e))))?;

    return Ok(vec![png_item, text_item]);
}

fn build_single_item_multi_mime(text: &str) -> Result<JsValue, String> {
    let text_blob = clip_async::make_text_blob(text, "text/plain")
        .map_err(|e| format_str(format_args!("text blob: {}", short_error(&e))))?;
    let png_promise = clip_async::make_png_promise(128, 200, |_| {})
        .map_err(|e| format_str(format_args!("png promise: {}", short_error(&e))))?;
    let sources = vec![
        ("image/png".to_string(), png_promise.into()),
        ("text/plain".to_string(), text_blob.into()),
    ];
    return clip_async::make_item(&sources)
        .map_err(|e| format_str(format_args!("single item: {}", short_error(&e))));
}

fn is_multiple_items_unsupported(msg: &str) -> bool {
    let lower = msg.to_lowercase();
    if !lower.contains("clipboarditem") { return false; }
    if lower.contains("only one clipboarditem") { return true; }
    if lower.contains("only supported with one clipboarditem") { return true; }
    if lower.contains("support for multiple clipboarditem") { return true; }
    if lower.contains("multiple clipboarditem") && lower.contains("not implemented") { return true; }
    return false;
}

// ============================================================
//                    КАРТОЧКА ВЕРДИКТА
// ============================================================

fn set_verdict_pending(app: &Rc<App>, msg: &str) {
    app.ui.verdict.set_class_name("ca-verdict pending");
    app.ui.verdict.set_text_content(Some(msg));
}

fn set_verdict_fail(app: &Rc<App>, title: &str, body: &str) {
    let mut html = String::with_capacity(256);
    html.push_str("<span class=\"ca-verdict-icon\">❌</span>");
    html.push_str("<span class=\"ca-verdict-title\">");
    html.push_str(&esc_html(title));
    html.push_str("</span>");
    if !body.is_empty() {
        html.push_str("<div class=\"ca-verdict-body\">");
        html.push_str(&esc_html(body));
        html.push_str("</div>");
    }
    app.ui.verdict.set_class_name("ca-verdict fail");
    app.ui.verdict.set_inner_html(&html);
}

fn set_verdict_success(app: &Rc<App>, title: &str, body: &str) {
    let mut html = String::with_capacity(256);
    html.push_str("<span class=\"ca-verdict-icon\">✅</span>");
    html.push_str("<span class=\"ca-verdict-title\">");
    html.push_str(&esc_html(title));
    html.push_str("</span>");
    if !body.is_empty() {
        html.push_str("<div class=\"ca-verdict-body\">");
        html.push_str(&esc_html(body));
        html.push_str("</div>");
    }
    app.ui.verdict.set_class_name("ca-verdict success");
    app.ui.verdict.set_inner_html(&html);
}

// ============================================================
//                    LOG
// ============================================================

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        let time = fmt_now_hms();
        let mut v = self.log.borrow_mut();
        v.push(LogEntry { time, text, kind });
        if v.len() > LOG_LIMIT {
            let drop_n = v.len() - LOG_LIMIT;
            v.drain(0..drop_n);
        }
        drop(v);
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"ca-log-empty\">здесь появятся операции с буфером</div>");
            return;
        }
        let mut html = String::with_capacity(512);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
                LogKind::Warn => "warn",
            };
            html.push_str("<div class=\"ca-log-entry ");
            html.push_str(cls);
            html.push_str("\"><span class=\"t\">");
            html.push_str(&esc_html(&e.time));
            html.push_str("</span><span class=\"m\">");
            html.push_str(&esc_html(&e.text));
            html.push_str("</span></div>");
        }
        self.ui.log.set_inner_html(&html);
        self.ui.log.set_scroll_top(self.ui.log.scroll_height() as f64);
    }
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn set_phase(el: &web_sys::Element, state: &str) {
    let cls = if state.is_empty() {
        "ca-phase".to_string()
    } else {
        format_str(format_args!("ca-phase {state}"))
    };
    el.set_class_name(&cls);
}

fn set_progress(app: &Rc<App>, pct: f64, label: &str) {
    if let Some(p) = app.ui.progress.dyn_ref::<web_sys::HtmlProgressElement>() {
        p.set_value(pct);
    }
    app.ui.progress_label.set_text_content(Some(label));
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "ca-status ok",
        "err" => "ca-status err",
        "busy" => "ca-status busy",
        "warn" => "ca-status warn",
        _ => "ca-status",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn fmt_bytes(n: u64) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = 1024.0 * 1024.0;
    let f = n as f64;
    if f < KB { return format_str(format_args!("{n} Б")); }
    if f < MB { return format_str(format_args!("{:.1} КБ", f / KB)); }
    return format_str(format_args!("{:.2} МБ", f / MB));
}

fn short_error(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        let msg = clamp_first_line(&dx.message());
        return format_str(format_args!("{}: {}", dx.name(), msg));
    }
    if let Some(s) = e.as_string() {
        return clamp_first_line(&s);
    }
    if e.is_object() {
        let name = js_sys::Reflect::get(e, &JsValue::from_str("name"))
            .ok().and_then(|v| v.as_string());
        let message = js_sys::Reflect::get(e, &JsValue::from_str("message"))
            .ok().and_then(|v| v.as_string());
        match (name, message) {
            (Some(n), Some(m)) => {
                return format_str(format_args!("{n}: {}", clamp_first_line(&m)));
            }
            (Some(n), None) => return n,
            _ => {}
        }
    }
    return format_str(format_args!("{e:?}"));
}

fn clamp_first_line(s: &str) -> String {
    let first = s.lines().next().unwrap_or("").trim();
    let mut out: String = first.chars().take(200).collect();
    if first.chars().count() > 200 { out.push('…'); }
    return out;
}

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(), d.get_minutes(), d.get_seconds(),
    ));
}

fn now_ms() -> f64 {
    web_sys::window().and_then(|w| w.performance())
        .map(|p| p.now()).unwrap_or(0.0)
}

fn user_agent() -> String {
    web_sys::window().map(|w| w.navigator())
        .and_then(|nav| nav.user_agent().ok())
        .unwrap_or_else(|| "—".to_string())
}

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;").replace('<', "&lt;").replace('>', "&gt;")
        .replace('"', "&quot;").replace('\'', "&#39;");
}

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window().ok_or("no window")?
        .document().ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}

fn get_input(id: &str) -> Result<web_sys::HtmlInputElement, JsValue> {
    get_el(id)?.dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не input")))
}

fn get_select(id: &str) -> Result<web_sys::HtmlSelectElement, JsValue> {
    get_el(id)?.dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не select")))
}

fn get_button(id: &str) -> Result<web_sys::HtmlButtonElement, JsValue> {
    get_el(id)?.dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}

fn set_click<F>(el: &web_sys::HtmlButtonElement, f: F)
where F: FnMut(web_sys::MouseEvent) + 'static {
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```


## 32.2.8. Практические находки — путь исследования

### Находка 1. `Promise<Blob>` работает везде, массив items — нигде

Первая версия демо показывала три уровня как «равнозначные»: синхронный blob, async blob, массив items. Скриншоты быстро показали: уровень 2 работает, уровень 3 падает. В Chrome и Firefox — по-разному падает.

**Вывод.** Материал главы разбился на две части: работающая (Promise\<Blob\>) и отрицательный результат (массив items). Первое показываем подробно, второе — коротко и с объяснением.

### Находка 2. Chrome и Firefox падают с разными сообщениями

**Chrome 154:**
```
NotAllowedError: Clipboard write is only supported with one ClipboardItem at the moment
```

**Firefox 157:**
```
NotAllowedError: Failed to execute 'write' on 'Clipboard':
Support for multiple ClipboardItems is not implemented.
```

Одно ограничение, две формулировки. Классификатор `is_multiple_items_unsupported` не может полагаться на точную строку — иначе сломается при следующем релизе браузера.

**Решение.**
```rust
fn is_multiple_items_unsupported(msg: &str) -> bool {
    let lower = msg.to_lowercase();
    if !lower.contains("clipboarditem") { return false; }
    if lower.contains("only one clipboarditem") { return true; }
    if lower.contains("only supported with one clipboarditem") { return true; }
    if lower.contains("support for multiple clipboarditem") { return true; }
    if lower.contains("multiple clipboarditem") && lower.contains("not implemented") { return true; }
    return false;
}
```

**Правило.** Классификация ошибок — по устойчивым фрагментам, не по полной строке. Строки меняются между версиями браузеров, фрагменты — нет.

### Находка 3. Имя браузера — не хардкод

В первой версии сообщение было: «Chrome не поддерживает массив items». Скриншот из Firefox показал это же сообщение — ложная информация о браузере, в котором работает пользователь.

**Решение.** `detect_browser() → Browser`. `Browser::short()` даёт имя. `format_str(format_args!("{} не поддерживает...", b.short()))` подставляет правильное.

**Правило.** Если демо работает в нескольких браузерах — имя браузера в UI берётся из `detect_browser()`, а не пишется в шаблоне.

### Находка 4. Размер PNG различается в 3.4 раза

Одна и та же генерация (128×128, градиент, круг, цифра) даёт разный размер в зависимости от движка:

| Браузер | Размер PNG |
|---|---|
| Firefox 157 | **4.8 КБ** |
| Chromium 154 | **16.1 КБ** |

**Причина.** `canvas.toBlob("image/png")` использует PNG-энкодер движка: Firefox — собственный, Chromium — Skia. Firefox применяет более агрессивные PNG-фильтры (Paeth/Up), дающие лучшее сжатие градиента. Chromium — консервативнее.

**Что это значит для кода.** Нельзя хардкодить ожидаемый размер. Только динамическое измерение через `blob.size()`.

**Что это значит для читателя.** Одна операция — два размера. Это часть реальности Web-платформы, и её надо знать.

**Правило.** Если измеряешь что-то через API браузера — сравни в двух движках. Расхождение может оказаться в 3 раза.

### Находка 5. Live-прогресс требует yield

Первая версия обновляла прогресс-бар в цикле `while` без `TimeoutFuture`. UI не перерисовывался — пользователь видел только финальный результат.

**Решение.** `countdown_loop` параллельно с `write()`:
```rust
while active.get() {
    let pct = (elapsed / delay as f64 * 100.0).min(100.0);
    set_progress(&app, pct, &label);
    TimeoutFuture::new(COUNTDOWN_TICK_MS).await;  // ← yield
}
```

`TimeoutFuture::new(50)` — это `setTimeout(50)`, macrotask. Браузер между macrotask'ами перерисовывает страницу. Без этого `await` на промисах — микротаски, рендер не происходит.

**Правило.** Любая длительная async-операция, обновляющая UI, должна явно yield'ить через `TimeoutFuture::new(0)` или больший интервал.

### Находка 6. FnOnce в FnMut-замыкании

`Promise::new` создаёт `FnMut`-замыкание. `on_ready: FnOnce` внутри него — ошибка компиляции: `cannot move out of on_ready, a captured variable in an FnMut closure`.

**Решение.** `Rc<RefCell<Option<F>>>` + `take()`:
```rust
let on_ready_cell = Rc::new(RefCell::new(Some(on_ready)));
// в замыкании:
if let Some(f) = on_ready_inner.borrow_mut().take() {
    f(b.size() as u64);
}
```

`take()` извлекает значение, оставляя `None`. Повторный вызов не сработает — но по контракту `on_ready` вызывается один раз.

**Правило.** `FnOnce` в `FnMut`-замыкании — через `Rc<RefCell<Option<F>>>` + `take()`. Не пытайтесь перемещать значение в callback напрямую.

### Находка 7. UI эксперимента 2 был непонятен

Первые версии показывали ошибку в журнале мелким шрифтом: `write: NotAllowedError: ...`. Пользователь не понимал: это баг демо, баг его кода, или ограничение браузера.

**Решение.**
1. **Бейдж на кнопке** — `❌ не поддерживается` для spec, `✅ работает` для workaround. До клика.
2. **Крупная карточка вердикта** после клика — заголовок + объяснение. Зелёная или красная.
3. **Убрать сырые сообщения** из журнала — оставить человеческие формулировки.

**Правило.** Если операция может провалиться, пользователь должен это видеть **до** клика и понять результат **после** — без чтения журнала.

### Находка 8. `cargo check --tests` ловит рассинхронизацию

`wasm-pack test --test wasm_clip_async` собирает только целевой файл. Если в другом тесте (`wasm_paste_evt.rs`) сломанный код — он всё равно компилируется, потому что `cargo` собирает весь workspace.

**Решение.** Перед коммитом главы:
```bash
cargo check --target wasm32-unknown-unknown --tests
```
2 секунды — ловит рассинхронизацию между модулем и старыми тестами.

**Правило.** Модуль и его тесты должны быть обновлены в одном коммите. Если модуль эволюционирует (переименование поля), тесты обновляются сразу.

## 32.2.9. Что не сработало и почему

### Не работает: массив items

Spec (W3C Clipboard API) описывает `clipboard.write(items: ClipboardItem[])` как способ передать несколько представлений. Chrome и Firefox не реализовали. Причина — политическая: реализация требует расширения системного буфера обмена, а договорённости между движками нет.

**Что делать вместо.** Один `ClipboardItem` с несколькими MIME — работает везде. Это то, что делает «копировать как Markdown» (глава 32, секция 4).

### Не пригодилось: `presentationStyle`

`ClipboardItem` в некоторых реализациях принимает третий параметр `presentationStyle: "unspecified" | "inline" | "attachment"`. Служит для подсказки приложению-получателю, как обращаться с данными. В Chrome не реализован. В Firefox частично.

**Что делать.** Не использовать. Не упоминать в UI.

### Не проверено: Safari

Демо тестировалось в Chrome и Firefox. Safari на Windows/Linux недоступен. Ожидаемое поведение: `Promise<Blob>` работает, `image/png` работает, массив items не работает.

**Что делать.** Не заявлять «работает везде», а «работает в проверенных движках». Safari — на совести читателя.

## 32.2.10. Правила проекта, закреплённые в главе

1. **`Promise<Blob>` в `ClipboardItem` — это рабочий паттерн.** Используйте его, когда генерация данных занимает время. Запись стартует до готовности данных.
2. **Массив `ClipboardItem[]` не работает** ни в Chrome, ни в Firefox. Не используйте. Один item с несколькими MIME — единственный совместимый способ.
3. **Классификация ошибок по паттернам, не по точной строке.** Chrome и Firefox пишут одно и то же ограничение разными словами.
4. **Имя браузера — из `detect_browser()`, не из хардкода.** Если сообщение говорит «Chrome не поддерживает», а демо работает в Firefox — это баг.
5. **`FnOnce` в `FnMut`-замыкании — через `Rc<RefCell<Option<F>>>` + `take()`.** Прямое перемещение запрещено.
6. **Live-прогресс требует yield через `TimeoutFuture`.** Иначе UI видит только финальный результат.
7. **Размер выходного файла зависит от движка.** Не хардкодьте ожидаемые байты. Измеряйте через `blob.size()`.
8. **`cargo check --target wasm32-unknown-unknown --tests`** — обязательная проверка перед коммитом.
9. **Бейдж на кнопке до клика + карточка вердикта после.** Пользователь не гадает, что работает и что произошло.
10. **Отрицательный результат исследования — тоже результат.** Не боритесь с ограничением API — документируйте его.

## 32.2.11. Итоги главы

- **`ClipboardItem` принимает `Promise<Blob>`.** Запись стартует сразу, данные приходят позже. Работает везде.
- **Массив `ClipboardItem[]` — spec без реализации.** Chrome и Firefox его не поддерживают. Заменитель — один item с несколькими MIME.
- **Классификация ошибок — по паттернам.** Chrome и Firefox формулируют одинаковое ограничение по-разному.
- **Размер PNG различается в 3.4×** между Firefox и Chromium. Не хардкодьте.
- **Live-прогресс — через yield.** `TimeoutFuture::new(N)` между обновлениями.
- **`FnOnce` + `FnMut` = `Rc<RefCell<Option<F>>>`** + `take()`.
- **Демо должно быть честным.** Не показывать неработающее как работающее. Не показывать хардкод «Chrome» в Firefox.
- **`cargo check --tests`** ловит рассинхронизацию между модулем и его тестами.

**Мостик к 32.3.** Мы разобрали `ClipboardItem` в трёх формах: синхронный blob, `Promise<Blob>`, массив items. Дальше — **запись через ClipboardEvent**: события `copy` и `cut`, `clipboardData.setData()`, перехват копирования из приложения и подмена содержимого. Та же дисциплина обработки ошибок, но обратное направление — не чтение буфера, а запись в него через событие.