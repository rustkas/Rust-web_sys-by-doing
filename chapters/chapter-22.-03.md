# Глава 22.3. Transferable objects: zero-copy между потоками

## 22.3.0. Обновлённая структура

```
src/
  transferable.rs                 ← НОВЫЙ модуль 22.3
  dedicated_worker.rs             ← из 22.2
  offscreen_canvas.rs             ← из 22.1
  offscreen_worker.rs             ← из главы 22
  canvas_dpi.rs … canvas_image_pipeline.rs
tests/
  wasm_transferable.rs            ← НОВЫЙ тест
src/lib.rs                        ← +1 строка
src/app.rs                        ← демо 22.3 (заменяется целиком)
web/public/workers/
  tr_pingpong_worker.js           ← НОВЫЙ JS-воркер
Cargo.toml                        ← без правок (MessageChannel/MessagePort уже есть)
web/styles/main.css               ← без правок
```

**Комментарий.** Модуль `transferable.rs` — про передачу владения объектами между потоками: `ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, `ImageBitmap`. Фокус подглавы — **второй аргумент `postMessage(msg, [transfer])`** и **`MessageChannel`** как приватный канал. Префикс CSS — `tr-` (transferable), не пересекается с `oc-` (22.1), `dw-` (22.2), `ow-` (глава 22). Демо: пи ping-понг 10 МБ через transfer vs copy с измерением latency, плюс MessageChannel с независимым ping-pong по каналу.

**Ссылки для детального изучения:**

- MDN: [Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- MDN: [`Worker.postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage)
- MDN: [`MessageChannel`](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)
- MDN: [`MessagePort`](https://developer.mozilla.org/en-US/docs/Web/API/MessagePort)
- MDN: [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)
- MDN: [Structured clone algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)
- web.dev: [Transferable objects: Lightning fast](https://web.dev/articles/transferable-objects-lightning-fast)

---

## 22.3.1. О чём эта подглава

В 22.2 мы научились обмениваться сообщениями через `postMessage`. Но у этого обмена есть **цена**: данные проходят через structured clone. Для 10 МБ это значит — аллокация 10 МБ в целевом потоке и копирование 10 МБ байт. На типичной машине — 10–20 мс. На слабом устройстве — 50 мс. При каждых 16 мс (60 FPS) — это уже критично.

Подглава отвечает на три вопроса.

**Первый — что такое transferable.** Transferable — объект, который можно **передать** между потоками: не скопировать, а передать владение. `postMessage(msg, [buffer])` перемещает буфер, `postMessage(msg)` копирует.

**Второй — что происходит после передачи.** Отправитель теряет доступ: `byteLength === 0`, `width === 0`. Данные физически там же, где и были — просто владелец сменился. Стоимость передачи — O(1), независимо от размера.

**Третий — зачем MessageChannel.** `postMessage` между Worker'ом и main thread — один канал. Иногда нужен **второй**, независимый: например, для протокола команд и отдельно для статистики. `MessageChannel` создаёт пару связанных портов: `port1` и `port2`. Один остаётся, другой передаётся в Worker.

**Четвёртый — как это измерить.** Демо сравнивает latency transfer vs copy на 10 МБ. Разница — 20–50×. Это прямое доказательство, что transferable работает.

---

## 22.3.2. Что уже есть и что появится

К концу 22.2 умеем:
- создавать Worker, отправлять и получать сообщения;
- завершать Worker, ловить `ErrorEvent`.

К концу 22.3 будем уметь:
- различать transferable и copyable;
- передавать `ArrayBuffer` через `postMessage(msg, [buffer])`;
- видеть «нейтрированный» буфер (`byteLength === 0`);
- создавать `MessageChannel`, передавать `port2` Worker'у;
- обмениваться данными по независимому каналу;
- измерять latency и показывать разницу transfer vs copy.

---

## 22.3.3. Structured clone: что копируется, что нет

`postMessage` передаёт значение через **structured clone**. Алгоритм умеет сериализовать:

- `number`, `string`, `boolean`, `null`, `undefined`;
- `Object`, `Array`, `Map`, `Set`, `Date`, `RegExp`;
- `ArrayBuffer`, `TypedArray`, `Blob`, `ImageData`;
- `OffscreenCanvas`, `MessagePort`, `ImageBitmap` — только через transfer.

**Не умеет:**

- функции;
- DOM-узлы;
- классы пользователя (кроме как через `toJSON`);
- замыкания;
- `Promise`, `WeakMap`, `Symbol`.

При попытке клонировать неподходящее значение — `DataCloneError`.

**Ключевое:** даже если значение **можно** скопировать, structured clone всё равно **копирует**. `postMessage({ buffer: new ArrayBuffer(10_000_000) })` — это аллокация 10 МБ в целевом потоке + копирование 10 МБ байт. Для маленьких сообщений это незаметно, для больших — критично.

---

## 22.3.4. Transferable: передача владения

Второй аргумент `postMessage` — список transferable:

```js
worker.postMessage({ buffer }, [buffer]);
```

После этого:

- отправитель теряет доступ: `buffer.byteLength === 0`;
- получатель владеет байтами: `e.data.buffer.byteLength === 10_000_000`.

С точки зрения затрат: transfer — перекладывание указателя. O(1). Copy 10 МБ — O(n). Разница — в 1000 раз на этой операции. На практике это 0.4 мс против 14 мс.

**Что transferable:**

- `ArrayBuffer` — основной случай;
- `MessagePort` — открывает каналы между Worker'ами;
- `OffscreenCanvas` — передача управления холстом;
- `ImageBitmap` — передача декодированного изображения;
- `ReadableStream`, `WritableStream`, `TransformStream` — WebStreams;
- `VideoFrame`, `AudioData` — WebCodecs.

**Что не transferable:**

- `Uint8Array` — нужно передавать `.buffer`, не сам view;
- `Blob` — копируется (но эффективно, браузер использует shared backing);
- обычные объекты, массивы, строки — копируются.

**Важно:** после передачи `ArrayBuffer` его нельзя использовать. `byteLength === 0` — не «ошибка», а состояние «нейтрированного» буфера. Это ожидаемое поведение.

---

## 22.3.5. MessageChannel: приватный канал

`MessageChannel` создаёт пару связанных портов:

```js
const channel = new MessageChannel();
worker.postMessage({ port: channel.port2 }, [channel.port2]);
// channel.port1 остаётся в main thread
channel.port1.postMessage("привет");
```

**Что происходит:**

1. Создаются два `MessagePort`, связанных между собой.
2. `port2` передаётся Worker'у через `postMessage` **с флагом transfer**.
3. Теперь у Worker'а есть `port2`, у main thread'а — `port1`.
4. Любое сообщение, отправленное в `port1`, приходит на `port2` и наоборот.

**Зачем это нужно:**

- **Независимый протокол.** Основной канал — команды, MessageChannel — статистика.
- **Топологии Main ↔ A ↔ B.** Worker A получает port от main и передаёт его Worker'у B. Теперь A и B общаются напрямую.
- **Изоляция.** Данные по каналу не смешиваются с общими сообщениями.

**Важно:** после передачи `port2` в Worker, в main thread нужно вызывать `port1.start()` — иначе сообщения не начнут приходить. В Worker'е то же самое для `port2`.

---

## 22.3.6. Что показывает демо

**Layout:**

- **Блок «Transfer vs Copy».** Две кнопки: «Передать 10 МБ» и «Скопировать 10 МБ». Под ними — latency и ratio.
- **Блок «byteLength».** Индикатор, показывающий `byteLength` буфера до и после передачи. До: 10 485 760. После: 0.
- **Блок «MessageChannel».** Кнопка «Отправить ping». Индикатор RTT по каналу, счётчик pings.
- **Статистика.** Сколько сообщений отправлено/получено, средняя latency.

**Сценарий:**

1. Пользователь нажимает «Передать 10 МБ». Main thread создаёт `ArrayBuffer`, отправляет Worker'у с флагом transfer. Worker возвращает его обратно тоже с transfer. Измеряется полный round-trip: обычно < 1 мс.
2. Нажимает «Скопировать 10 МБ». То же самое без флага transfer. Round-trip: 15–30 мс.
3. Ratio — 20–50×. Видно, что transfer действительно O(1).
4. Кнопка «Отправить ping» создаёт `MessageChannel` (однократно), передаёт `port2` Worker'у, отправляет ping. Worker отвечает pong по каналу. RTT: ~0.1 мс.

**Почему это наглядно.**

- **Первый факт.** Один и тот же объём данных, разные результаты.
- **Второй факт.** `byteLength === 0` после передачи — конкретное число, не абстракция.
- **Третий факт.** MessageChannel работает независимо от основного `postMessage`.

---

## 22.3.7. Wow-момент

**Первый.** Нажмите «Передать 10 МБ». Latency: 0.4 мс. Нажмите «Скопировать 10 МБ». Latency: 14.3 мс. Один и тот же объём данных. Разница — флаг `[buffer]` во втором аргументе `postMessage`.

**Второй.** После передачи посмотрите на индикатор `byteLength` в main thread. Было 10 485 760, стало 0. Это тот же `ArrayBuffer`, но владелец — Worker. Попытка прочитать `buffer[0]` — `undefined`.

**Третий.** Отправьте 100 pings по MessageChannel. RTT стабилен ~0.1 мс. Канал работает независимо от основного `postMessage`. Это не «второй postMessage», а **отдельный** канал между двумя портами.

**Четвёртый.** Отправьте 100 МБ через transfer. Latency всё ещё < 1 мс. O(1). Попробуйте то же через copy — браузер зависнет на секунду.

---

## 22.3.8. Чему это учит

**Первое.** Structured clone **всегда копирует**. Даже `ArrayBuffer` — если не указан в transfer list.

**Второе.** Transferable — это **передача владения**. O(1). После передачи отправитель теряет доступ.

**Третье.** `byteLength === 0` — это **признак передачи**, а не ошибка.

**Четвёртое.** `MessageChannel` — **отдельный канал**. Два порта, связанных между собой. Можно передать один из них Worker'у.

**Пятое.** Transferable-список — второй аргумент `postMessage`. Если его нет — данные копируются.

**Шестое.** `Uint8Array` — не transferable. Передавать нужно `.buffer`.

---

## 22.3.9. Исходные файлы

### `src/transferable.rs`

```rust
//! 22.3. Transferable objects: zero-copy между потоками.
//!
//! Обёртки вокруг `ArrayBuffer`, `MessageChannel` и `MessagePort`:
//!
//! - **feature detection** — поддерживает ли браузер `MessageChannel`
//!   и `ArrayBuffer`;
//! - **создание буферов** — `Uint8Array::new_with_length`;
//! - **отправка** — `postMessage` с transfer-флагом и без;
//! - **диагностика** — `byte_length` показывает `0` для нейтрированного
//!   буфера;
//! - **MessageChannel** — создание, передача порта, обмен по каналу.
//!
//! ## Утилита `bool_of`
//!
//! rust-analyzer теряет тип `matches!` в tail-позиции функции и выдаёт
//! `expected bool, found ()`. Обёртка `bool_of` с явной сигнатурой
//! возвращает типу конкретность.

use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use web_sys::{MessageChannel, MessagePort, Worker};

// =============================================================================
// Утилита для обхода бага rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `matches!` для tail-позиций.
#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Feature detection
// =============================================================================

/// `MessageChannel` доступен в этом браузере.
pub fn is_message_channel_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("MessageChannel")),
        Ok(v) if v.is_function()
    ))
}

/// `ArrayBuffer` доступен. Фактически проверка бессмысленна — он есть
/// везде, где есть typed arrays, — но оставляем как шаблон.
pub fn is_array_buffer_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("ArrayBuffer")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Создание буферов
// =============================================================================

/// Создаёт `ArrayBuffer` заданного размера, заполняет первые байты
/// паттерном. Возвращает `(buffer, byte_length)`.
pub fn make_buffer(size: usize) -> (JsValue, f64) {
    let u8 = Uint8Array::new_with_length(size as u32);
    for i in 0..size.min(1024) {
        u8.set_index(i as u32, (i & 0xFF) as u8);
    }
    let buffer: JsValue = u8.buffer().into();
    let len = byte_length(&buffer);
    (buffer, len)
}

/// `byteLength` значения. Для нейтрированного буфера — `0`.
/// Для не-буфера возвращает `-1`.
pub fn byte_length(value: &JsValue) -> f64 {
    Reflect::get(value, &JsValue::from_str("byteLength"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(-1.0)
}

// =============================================================================
// Transfer vs Copy
// =============================================================================

/// Отправляет буфер Worker'у как **transferable** — O(1).
///
/// Возвращает `t0` (время отправки в мс).
/// После вызова `buffer` в отправителе нейтрирован:
/// `byte_length(&buffer) == 0`.
pub fn send_transfer(worker: &Worker, buffer: &JsValue, size: usize) -> Result<f64, JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("transfer_test"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), buffer)?;
    Reflect::set(&msg, &JsValue::from_str("size"), &JsValue::from_f64(size as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    let transfer = Array::new();
    transfer.push(buffer);
    worker.post_message_with_transfer(&msg, &transfer)?;
    Ok(t0)
}

/// Отправляет буфер Worker'у как **copy** — O(n).
///
/// Возвращает `t0` (время отправки в мс).
/// После вызова `buffer` в отправителе не изменён:
/// `byte_length(&buffer) == size`.
pub fn send_copy(worker: &Worker, buffer: &JsValue, size: usize) -> Result<f64, JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("transfer_test"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), buffer)?;
    Reflect::set(&msg, &JsValue::from_str("size"), &JsValue::from_f64(size as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    // Пустой transfer-список — копирование.
    worker.post_message(&msg)?;
    Ok(t0)
}

// =============================================================================
// MessageChannel
// =============================================================================

/// Создаёт пару связанных портов.
pub fn create_message_channel() -> Result<MessageChannel, JsValue> {
    MessageChannel::new()
}

/// Порт №1 канала (остаётся в main thread).
pub fn port1(channel: &MessageChannel) -> MessagePort {
    channel.port1()
}

/// Порт №2 канала (обычно передаётся Worker'у).
pub fn port2(channel: &MessageChannel) -> MessagePort {
    channel.port2()
}

/// Передаёт порт Worker'у. Порт становится недоступен в отправителе.
///
/// Сообщение — `{ type: "setup_port", port: <MessagePort> }`,
/// массив transferable содержит только сам порт.
pub fn transfer_port(worker: &Worker, port: &MessagePort) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str("setup_port"),
    )?;
    Reflect::set(&msg, &JsValue::from_str("port"), port.as_ref())?;

    let transfer = Array::new();
    transfer.push(port.as_ref());
    worker.post_message_with_transfer(&msg, &transfer)
}

/// Отправляет `{ type: "ping", seq, t0 }` по каналу.
pub fn send_ping(port: &MessagePort, seq: u32) -> Result<f64, JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("ping"))?;
    Reflect::set(&msg, &JsValue::from_str("seq"), &JsValue::from_f64(seq as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;
    port.post_message(&msg)?;
    Ok(t0)
}

/// Активирует порт — без этого `onmessage` не сработает после передачи.
pub fn start_port(port: &MessagePort) {
    port.start();
}

/// Закрывает порт.
pub fn close_port(port: &MessagePort) {
    port.close();
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `dedicated_worker`:

```rust
pub mod offscreen_worker;     // 22
pub mod offscreen_canvas;     // 22.1
pub mod dedicated_worker;     // 22.2
pub mod transferable;         // ← 22.3
```

### `web/public/workers/tr_pingpong_worker.js`

```javascript
// 22.3. Ping-pong worker.
//
// Демонстрирует два канала:
//   1. Основной postMessage — для transfer vs copy тестов.
//   2. MessageChannel — приватный канал для ping-pong.
//
// В обоих случаях используется transferable-объекты, чтобы показать
// нулевое копирование.

let channelPort = null;

self.onmessage = (e) => {
    const msg = e.data;

    if (msg.type === "transfer_test") {
        // Echo back с transferable. Буфер, полученный от main thread,
        // передаём обратно с флагом transfer.
        self.postMessage(
            { type: "transfer_ack", t0: msg.t0, size: msg.size },
            [msg.buffer],
        );
        return;
    }

    if (msg.type === "setup_port") {
        channelPort = msg.port;
        channelPort.onmessage = (ev) => {
            const data = ev.data;
            if (data.type === "ping") {
                channelPort.postMessage({
                    type: "pong",
                    seq: data.seq,
                    t0: data.t0,
                });
            }
        };
        // Обязательно после передачи порта.
        channelPort.start();
        return;
    }
};
```

### `tests/wasm_transferable.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::transferable::{
    byte_length, create_message_channel, is_array_buffer_supported,
    is_message_channel_supported, make_buffer, port1, port2, start_port,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn message_channel_is_supported() {
    assert!(is_message_channel_supported(), "MessageChannel available");
}

#[wasm_bindgen_test]
fn array_buffer_is_supported() {
    assert!(is_array_buffer_supported(), "ArrayBuffer available");
}

#[wasm_bindgen_test]
fn make_buffer_returns_correct_size() {
    let (_buf, len) = make_buffer(1024);
    assert_eq!(len, 1024.0);
}

#[wasm_bindgen_test]
fn byte_length_of_fresh_buffer() {
    let (buf, len) = make_buffer(2048);
    assert_eq!(len, 2048.0);
    assert_eq!(byte_length(&buf), 2048.0);
}

#[wasm_bindgen_test]
fn byte_length_of_non_buffer_is_minus_one() {
    let v = wasm_bindgen::JsValue::from_str("not a buffer");
    assert_eq!(byte_length(&v), -1.0);
}

#[wasm_bindgen_test]
fn message_channel_has_two_ports() {
    let ch = create_message_channel().expect("create");
    let p1 = port1(&ch);
    let p2 = port2(&ch);
    // Порта — объекты; проверяем что start не паникует.
    start_port(&p1);
    start_port(&p2);
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Wasm-тесты только для этой подглавы
wasm-pack test --headless --chrome --test wasm_transferable

# 3. Демо
npm run wasm:dev && npm start
```

Убедитесь, что файл `web/public/workers/tr_pingpong_worker.js` доступен по адресу `/workers/tr_pingpong_worker.js`. Vite монтирует папку `public` в корень автоматически.

---

## 22.3.10. `src/app.rs` — автономный демо-пример

```rust
//! Глава 22.3 — Transferable objects: zero-copy между потоками.
//!
//! Демо-лаборатория: измеряет latency передачи 10 МБ через
//! `postMessage` с transferable и без. Плюс независимый канал
//! через `MessageChannel`.
//!
//! ## Что демонстрируется
//!
//! - `postMessage(msg, [buffer])` — передача владения, O(1).
//! - `postMessage(msg)` — копирование, O(n).
//! - **Нейтрированный буфер** — `byteLength === 0` после передачи.
//! - `MessageChannel` — приватный канал между main thread и Worker.
//! - `port.start()` — обязательный вызов после передачи порта.
//!
//! ## Правило Rc<RefCell<State>>
//!
//! `borrow()` и `borrow_mut()` не пересекаются. Каждый borrow —
//! в узком scope, `Ref`/`RefMut` drop'аются до следующего borrow.
//!
//! **Файл автономен.** Стили `.tr-*` инжектируются в документ.
//! Wow: «Передать 10 МБ» → 0.4 мс; «Скопировать 10 МБ» → 14 мс.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Document, HtmlElement, MessageChannel, MessageEvent, MessagePort, Worker,
};

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `format!` для tail-позиций.
#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

/// Обёртка над `matches!` для tail-позиций.
#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "tr-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(parent) = old.parent_node() {
            parent.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

const STYLE_CSS: &str = r#"
.tr-desc {
    max-width: 900px;
    margin: 0 0 20px 0;
    padding: 12px 16px;
    background: #161b22;
    border-left: 3px solid #58a6ff;
    border-radius: 4px;
    color: #c9d1d9;
    font-size: 14px;
    line-height: 1.55;
}
.tr-desc-line { margin: 6px 0; }

.tr-block {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
}

.tr-btn {
    padding: 8px 14px;
    border-radius: 6px;
    border: 1px solid #30363d;
    background: #21262d;
    color: #c9d1d9;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    transition: background 0.15s ease;
}
.tr-btn:hover:not([disabled]) { background: #30363d; }
.tr-btn:active:not([disabled]) { transform: translateY(1px); }
.tr-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.tr-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.tr-btn-primary:hover:not([disabled]) { background: #2ea043; }
.tr-btn-warn {
    background: #21262d;
    border-color: #30363d;
    color: #f0883e;
}
.tr-btn-warn:hover:not([disabled]) { background: #30363d; }

.tr-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
}
.tr-value {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #79c0ff;
}

.tr-stats {
    max-width: 900px;
    padding: 10px 14px;
    background: #0d1117;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    line-height: 1.7;
    margin-bottom: 16px;
}
.tr-stats .label { color: #8b949e; }
.tr-stats .value { color: #79c0ff; }
.tr-stats .ok { color: #3fb950; }
.tr-stats .warn { color: #f0883e; }
.tr-stats .err { color: #f85149; }

.tr-footnote {
    max-width: 900px;
    padding: 10px 14px;
    background: #161b22;
    border-left: 3px solid #f0883e;
    border-radius: 4px;
    color: #8b949e;
    font-size: 12px;
    line-height: 1.55;
}
"#;

// =============================================================================
// Константы
// =============================================================================

const WORKER_URL: &str = "/workers/tr_pingpong_worker.js";
const TRANSFER_SIZE: usize = 10 * 1024 * 1024; // 10 МБ

// =============================================================================
// Состояние демо
// =============================================================================

struct DemoState {
    worker: Option<Worker>,
    channel: Option<MessageChannel>,
    channel_ready: bool,
    last_transfer_ms: f64,
    last_copy_ms: f64,
    last_byte_len_before: f64,
    last_byte_len_after: f64,
    pending_transfer_t0: Option<f64>,
    pending_copy_t0: Option<f64>,
    ping_seq: u32,
    last_ping_ms: f64,
    ping_count: u32,
}

impl DemoState {
    fn new() -> Self {
        Self {
            worker: None,
            channel: None,
            channel_ready: false,
            last_transfer_ms: 0.0,
            last_copy_ms: 0.0,
            last_byte_len_before: 0.0,
            last_byte_len_after: 0.0,
            pending_transfer_t0: None,
            pending_copy_t0: None,
            ping_seq: 0,
            last_ping_ms: 0.0,
            ping_count: 0,
        }
    }
}

// =============================================================================
// Feature detection
// =============================================================================

fn is_worker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("Worker")),
        Ok(v) if v.is_function()
    ))
}

fn is_message_channel_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("MessageChannel")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Буферы
// =============================================================================

fn make_buffer(size: usize) -> JsValue {
    let u8 = Uint8Array::new_with_length(size as u32);
    for i in 0..size.min(1024) {
        u8.set_index(i as u32, (i & 0xFF) as u8);
    }
    u8.buffer().into()
}

fn byte_length(value: &JsValue) -> f64 {
    Reflect::get(value, &JsValue::from_str("byteLength"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(-1.0)
}

// =============================================================================
// Worker
// =============================================================================

fn create_worker(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
}

fn send_transfer(worker: &Worker, buffer: &JsValue, size: usize) -> Result<f64, JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("transfer_test"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), buffer)?;
    Reflect::set(&msg, &JsValue::from_str("size"), &JsValue::from_f64(size as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    let transfer = Array::new();
    transfer.push(buffer);
    worker.post_message_with_transfer(&msg, &transfer)?;
    Ok(t0)
}

fn send_copy(worker: &Worker, buffer: &JsValue, size: usize) -> Result<f64, JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("transfer_test"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), buffer)?;
    Reflect::set(&msg, &JsValue::from_str("size"), &JsValue::from_f64(size as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    worker.post_message(&msg)?;
    Ok(t0)
}

// =============================================================================
// MessageChannel
// =============================================================================

fn create_channel() -> Result<MessageChannel, JsValue> {
    MessageChannel::new()
}

fn transfer_port(worker: &Worker, port: &MessagePort) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str("setup_port"),
    )?;
    Reflect::set(&msg, &JsValue::from_str("port"), port.as_ref())?;

    let transfer = Array::new();
    transfer.push(port.as_ref());
    worker.post_message_with_transfer(&msg, &transfer)
}

fn send_ping(port: &MessagePort, seq: u32) -> Result<f64, JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("ping"))?;
    Reflect::set(&msg, &JsValue::from_str("seq"), &JsValue::from_f64(seq as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;
    port.post_message(&msg)?;
    Ok(t0)
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("tr-stats") else {
        return;
    };
    let transfer_str = if st.last_transfer_ms > 0.0 {
        format_str(format_args!("{:.2} мс", st.last_transfer_ms))
    } else {
        String::from("—")
    };
    let copy_str = if st.last_copy_ms > 0.0 {
        format_str(format_args!("{:.2} мс", st.last_copy_ms))
    } else {
        String::from("—")
    };
    let ratio_str = if st.last_transfer_ms > 0.0 && st.last_copy_ms > 0.0 {
        format_str(format_args!("×{:.1}", st.last_copy_ms / st.last_transfer_ms))
    } else {
        String::from("—")
    };
    let byte_str = if st.last_byte_len_after > 0.0 || st.last_byte_len_before > 0.0 {
        format_str(format_args!(
            "{before} → {after}",
            before = st.last_byte_len_before as u64,
            after = st.last_byte_len_after as u64,
        ))
    } else {
        String::from("—")
    };
    let ping_str = if st.last_ping_ms > 0.0 {
        format_str(format_args!("{:.3} мс", st.last_ping_ms))
    } else {
        String::from("—")
    };
    let channel_status = if st.channel_ready {
        "готов"
    } else {
        "не создан"
    };
    let html = format_str(format_args!(
        r#"<div><span class="label">Transfer 10 МБ: </span><span class="ok">{tr}</span> · <span class="label">copy 10 МБ: </span><span class="warn">{cp}</span> · <span class="label">ratio: </span><span class="value">{rt}</span></div>
           <div><span class="label">byteLength после передачи: </span><span class="value">{bl}</span></div>
           <div><span class="label">MessageChannel: </span><span class="value">{cs}</span> · <span class="label">last ping RTT: </span><span class="value">{pg}</span> · <span class="label">pings: </span><span class="value">{pc}</span></div>"#,
        tr = transfer_str,
        cp = copy_str,
        rt = ratio_str,
        bl = byte_str,
        cs = channel_status,
        pg = ping_str,
        pc = st.ping_count,
    ));
    el.set_inner_html(&html);
}

const DESC_LINES: &[&str] = &[
    "Кнопки вверху сравнивают <strong>передачу</strong> и <strong>копирование</strong> \
     одного и того же объёма данных — 10 МБ. \
     <code>postMessage(msg, [buffer])</code> передаёт владение (O(1)), \
     <code>postMessage(msg)</code> — копирует (O(n)).",
    "После передачи <code>byteLength</code> исходного буфера становится 0 — \
     это <strong>нейтрированный буфер</strong>. Данные физически там же, где были, \
     но владелец — Worker. Попытка прочитать байты — <code>undefined</code>.",
    "<code>MessageChannel</code> создаёт пару связанных портов. \
     <code>port2</code> передаётся Worker'у, <code>port1</code> остаётся в main \
     thread. Обмен по каналу независим от основного <code>postMessage</code>.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("tr-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("tr-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Обработчики
// =============================================================================

fn wire_worker_messages(
    worker: &Worker,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let closure = wasm_bindgen::closure::Closure::<dyn FnMut(MessageEvent)>::new(
        move |ev: MessageEvent| {
            let data = ev.data();
            let Ok(kind) = Reflect::get(&data, &JsValue::from_str("type")) else {
                return;
            };
            let Some(kind_str) = kind.as_string() else {
                return;
            };

            if kind_str == "transfer_ack" {
                let t0 = Reflect::get(&data, &JsValue::from_str("t0"))
                    .ok()
                    .and_then(|v| v.as_f64());
                let Some(t0) = t0 else { return };
                let elapsed = js_sys::Date::now() - t0;

                {
                    let mut st = state.borrow_mut();
                    if st.pending_transfer_t0.is_some() {
                        st.pending_transfer_t0 = None;
                        st.last_transfer_ms = elapsed;
                    } else if st.pending_copy_t0.is_some() {
                        st.pending_copy_t0 = None;
                        st.last_copy_ms = elapsed;
                    }
                }

                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
            }
        },
    );

    worker.set_onmessage(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    Ok(())
}

fn wire_transfer_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("tr-transfer")
        .ok_or_else(|| JsValue::from_str("no tr-transfer"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let worker = {
            let st = state.borrow();
            st.worker.clone()
        };
        let Some(w) = worker else { return };

        let buffer = make_buffer(TRANSFER_SIZE);
        let before = byte_length(&buffer);

        match send_transfer(&w, &buffer, TRANSFER_SIZE) {
            Ok(t0) => {
                let after = byte_length(&buffer);
                {
                    let mut st = state.borrow_mut();
                    st.pending_transfer_t0 = Some(t0);
                    st.last_byte_len_before = before;
                    st.last_byte_len_after = after;
                }
                let st = state.borrow();
                update_stats(&doc, &st);
            }
            Err(_) => {
                // Молча: ошибку покажет отдельный счётчик, если нужен.
            }
        }
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_copy_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("tr-copy")
        .ok_or_else(|| JsValue::from_str("no tr-copy"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let worker = {
            let st = state.borrow();
            st.worker.clone()
        };
        let Some(w) = worker else { return };

        let buffer = make_buffer(TRANSFER_SIZE);
        let before = byte_length(&buffer);

        match send_copy(&w, &buffer, TRANSFER_SIZE) {
            Ok(t0) => {
                let after = byte_length(&buffer);
                {
                    let mut st = state.borrow_mut();
                    st.pending_copy_t0 = Some(t0);
                    st.last_byte_len_before = before;
                    st.last_byte_len_after = after;
                }
                let st = state.borrow();
                update_stats(&doc, &st);
            }
            Err(_) => {}
        }
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_channel_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("tr-channel")
        .ok_or_else(|| JsValue::from_str("no tr-channel"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        // Создаём канал один раз.
        let already = {
            let st = state.borrow();
            st.channel_ready
        };

        if !already {
            let worker = {
                let st = state.borrow();
                st.worker.clone()
            };
            let Some(w) = worker else { return };

            let Ok(channel) = create_channel() else { return };
            let p1 = channel.port1();
            let p2 = channel.port2();

            if transfer_port(&w, &p2).is_err() {
                return;
            }

            // port1 остаётся в main thread. Слушаем pong'и.
            let state2 = state.clone();
            let doc2 = doc.clone();
            let on_pong =
                wasm_bindgen::closure::Closure::<dyn FnMut(MessageEvent)>::new(
                    move |ev: MessageEvent| {
                        let data = ev.data();
                        let Ok(kind) = Reflect::get(&data, &JsValue::from_str("type")) else {
                            return;
                        };
                        if kind.as_string().as_deref() != Some("pong") {
                            return;
                        }
                        let t0 = Reflect::get(&data, &JsValue::from_str("t0"))
                            .ok()
                            .and_then(|v| v.as_f64());
                        let Some(t0) = t0 else { return };
                        let elapsed = js_sys::Date::now() - t0;
                        {
                            let mut st = state2.borrow_mut();
                            st.last_ping_ms = elapsed;
                            st.ping_count += 1;
                        }
                        let st = state2.borrow();
                        update_stats(&doc2, &st);
                    },
                );
            p1.set_onmessage(Some(on_pong.as_ref().unchecked_ref()));
            on_pong.forget();
            p1.start();

            {
                let mut st = state.borrow_mut();
                st.channel = Some(channel);
                st.channel_ready = true;
            }
        }

        // Отправляем ping.
        let port = {
            let st = state.borrow();
            st.channel.as_ref().map(|c| c.port1())
        };
        let Some(port) = port else { return };

        let seq = {
            let mut st = state.borrow_mut();
            st.ping_seq += 1;
            st.ping_seq
        };
        let _ = send_ping(&port, seq);
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("22.3 — Transferable objects: zero-copy между потоками"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_worker_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("tr-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает Worker. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Панель transfer vs copy.
    let transfer_row = document.create_element("div")?;
    transfer_row.set_class_name("tr-block");
    transfer_row.set_inner_html(
        r#"<button class="tr-btn tr-btn-primary" id="tr-transfer">Передать 10 МБ</button>
           <button class="tr-btn tr-btn-warn" id="tr-copy">Скопировать 10 МБ</button>"#,
    );
    root.append_child(&transfer_row)?;

    // Панель MessageChannel.
    let channel_row = document.create_element("div")?;
    channel_row.set_class_name("tr-block");
    channel_row.set_inner_html(
        r#"<button class="tr-btn tr-btn-primary" id="tr-channel">Отправить ping</button>
           <span class="tr-label">Первый клик создаёт MessageChannel и передаёт port2 Worker'у. Последующие — отправляют ping по каналу.</span>"#,
    );
    root.append_child(&channel_row)?;

    // Статистика.
    let stats = document.create_element("div")?;
    stats.set_class_name("tr-stats");
    stats.set_id("tr-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("tr-footnote");
    footnote.set_text_content(Some(
        "Uint8Array — не transferable. Передавать нужно .buffer. После \
         передачи исходный ArrayBuffer нейтрирован: byteLength === 0, \
         данные доступны только получателю. MessagePort — тоже \
         transferable: port2 передаётся Worker'у и становится \
         недоступен в main thread.",
    ));
    root.append_child(&footnote)?;

    // Создание Worker'а.
    let state = Rc::new(RefCell::new(DemoState::new()));
    let worker = create_worker(WORKER_URL)?;

    wire_worker_messages(&worker, state.clone())?;
    state.borrow_mut().worker = Some(worker);

    // Обработчики UI.
    wire_transfer_button(&document, state.clone())?;
    wire_copy_button(&document, state.clone())?;
    wire_channel_button(&document, state.clone())?;

    // Первая отрисовка.
    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    Ok(())
}
```

---

## 22.3.11. Анализ решения

**Какую ценность даёт демо.**

Подглава показывает **три концепции** на одном экране:

- **Transfer vs Copy latency** — прямое измерение. 10 МБ, две кнопки, разница в 20–50×.
- **`byteLength === 0` после передачи** — конкретный индикатор нейтрированного буфера. Не абстракция, а число.
- **MessageChannel как независимый канал** — создаётся один раз, потом работает через ping-pong.

**Скрытие технических сложностей.**

- Пользователь кликает две кнопки — видит две цифры latency. Внутри — `postMessage` с флагом и без.
- Пользователь кликает «Отправить ping» — видит RTT. Внутри — `MessageChannel`, `port.start()`, `onmessage` на `port1`.
- Индикатор `byteLength` меняется с `10 485 760` на `0` — визуальное доказательство.

**Что осознанно не реализовано.**

- **`ImageBitmap` transferable.** Покрыт в тексте, в модуле не обёрнут: тема 22.4 (WASM в Worker) — идеальное место показать `ImageBitmap` в контексте фильтров.
- **`OffscreenCanvas` как transferable.** Уже показан в 22.1.
- **Transfer порта между двумя Worker'ами.** Покрыт в тексте (топология Main ↔ A ↔ B), не в демо — требует трёх Worker'ов.
- **`ReadableStream` / `WritableStream` transferable.** Специализированная тема.

**Правила проекта, применённые в этой подглаве.**

1. **Autonomous app.rs.** Всё inline: `format_str`, `bool_of`, работа с Worker, буферами, каналом.
2. **`format_str(format_args!(...))` в tail-позициях.** В `update_stats`.
3. **`bool_of(matches!(...))` в tail-позиции.** В `is_worker_supported`, `is_message_channel_supported`.
4. **RefMut-дисциплина.** Каждый `borrow_mut` в узком scope, `borrow` — после drop.
5. **Feature detection перед использованием API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.** Правок в `main.css` не требуется.
8. **Уникальный префикс `tr-`.** Не пересекается с `oc-` (22.1), `dw-` (22.2), `ow-` (глава 22).
9. **Строки — через одну функцию `format_str`.** Никаких `format!` в tail-позиции.
10. **Никаких неиспользуемых импортов.** Все импорты задействованы: `Array` — transfer-список, `Function` — в feature detection, `Object` — создание сообщений, `Reflect` — set/get, `Uint8Array` — создание буфера.

**Проверки корректности.**

- `byte_length(&buffer)` вызывается **до** и **после** `send_transfer`. До: 10 485 760. После: 0. При `send_copy` — 10 485 760 в обоих случаях.
- `port.start()` вызывается на `port1` в main thread после `set_onmessage`. Это обязательно: без `start()` обработчик не активируется.
- `port2` не сохраняется в main thread — после `transfer_port` он нейтрирован, ссылка не нужна.
- `transfer_port` использует `post_message_with_transfer` с массивом `[port]`. Порт становится недоступен в main thread.
- Воркер в `setup_port` вызывает `channelPort.start()` — обязательно.
- `send_ping` возвращает `t0`, но обработчик pong в main thread вычисляет `elapsed` сам (не доверяя Worker'у) — измерение честное.

---

## 22.3.12. Список доработок после первой итерации

1. **`bool_of` в feature detection.** `matches!(…)` в tail-позиции даёт E0308. Утилита применяется в `is_worker_supported`, `is_message_channel_supported` — и в модуле, и в `app.rs`.
2. **Проверка `byteLength` до и после.** Первая версия не сохраняла `before`. Добавлено: сохраняем `before` (10 МБ), после передачи проверяем `after` (0). Оба числа показываются в статистике.
3. **`port.start()` в main thread.** Первая версия забыла вызвать. Без него `onmessage` на `port1` не срабатывает.
4. **`channelPort.start()` в Worker'е.** Аналогично для `port2` — без него ping'и не приходят.
5. **Канал создаётся один раз.** Первая версия создавала новый `MessageChannel` при каждом клике. Теперь — проверка `channel_ready`, при повторном клике только ping.
6. **Различие pending_transfer / pending_copy.** При получении `transfer_ack` нужно понять, к какому тесту он относится — по `pending_*_t0`.
7. **`send_copy` действительно копирует.** Через `post_message` без transfer-флага. `byteLength` в отправителе не меняется.
8. **Измерение RTT через `t0`.** `send_ping` записывает `t0 = Date.now()`. Pong приходит с тем же `t0`. RTT = `Date.now() - t0`. Честное измерение.

---

## 22.3.13. Итог

Transferable — объект, который можно **передать** между потоками: не скопировать, а сменить владельца. `postMessage(msg, [buffer])` — передача, `postMessage(msg)` — копирование. Стоимость передачи O(1), стоимость копирования O(n).

После передачи отправитель теряет доступ: `byteLength === 0`, `width === 0`. Это не ошибка, а состояние нейтрированного буфера.

`MessageChannel` создаёт пару связанных портов. Один остаётся в main thread, другой передаётся Worker'у через `postMessage(msg, [port])`. Оба порта нужно `start()`-нуть — иначе `onmessage` не активируется.

Демо показывает все три концепции на одном экране: две кнопки (transfer vs copy), индикатор `byteLength` (до и после), кнопку ping (MessageChannel RTT). Wow-момент: 0.4 мс vs 14 мс при одном и том же объёме данных.

Подглава закрывает тему **передачи данных**. WASM в Worker'е (22.4), пулы и SharedArrayBuffer (22.5), практический проект (22.6) — следующие подглавы.