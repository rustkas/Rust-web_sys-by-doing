# Глава 22.2. Dedicated Worker: базовые примитивы

## 22.2.0. Обновлённая структура

```
src/
  dedicated_worker.rs             ← НОВЫЙ модуль 22.2
  offscreen_canvas.rs             ← из 22.1
  offscreen_worker.rs             ← из главы 22
  canvas_dpi.rs … canvas_image_pipeline.rs
tests/
  wasm_dedicated_worker.rs        ← НОВЫЙ тест
src/lib.rs                        ← +1 строка
src/app.rs                        ← демо 22.2 (заменяется целиком)
web/public/workers/
  dw_echo_worker.js               ← НОВЫЙ JS-воркер
Cargo.toml                        ← без правок (фичи Worker, MessageEvent уже есть)
web/styles/main.css               ← без правок
```

**Комментарий.** Модуль `dedicated_worker.rs` — про базовые примитивы `DedicatedWorkerGlobalScope`: создание воркера, отправка сообщений, приём ответов, обработка ошибок, завершение. Префикс CSS — `dw-` (dedicated worker), не пересекается с `oc-` (22.1) и `ow-` (глава 22). Фокус подглавы — **интерфейс Worker'а с обеих сторон**, без OffscreenCanvas и transferable-данных: это отдельные темы. Демо — «echo-воркер»: строка уходит в Worker, возвращается в верхнем регистре; кнопка terminate завершает Worker; кнопка «Кинуть ошибку» демонстрирует `ErrorEvent`.

**Ссылки для детального изучения:**

- MDN: [Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- MDN: [`DedicatedWorkerGlobalScope`](https://developer.mozilla.org/en-US/docs/Web/API/DedicatedWorkerGlobalScope)
- MDN: [`Worker`](https://developer.mozilla.org/en-US/docs/Web/API/Worker)
- MDN: [`Worker.terminate()`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/terminate)
- MDN: [`Worker.onerror`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/error_event)
- MDN: [`ErrorEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ErrorEvent)
- MDN: [`Worker.postMessage()`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage)
- web.dev: [Web Workers](https://web.dev/articles/workers-basics)

---

## 22.2.1. О чём эта подглава

В 22.1 мы научились создавать `OffscreenCanvas` и передавать его Worker'у. Но между main thread и Worker'ом должен быть **канал обмена**. Этот канал — `postMessage`, а сам Worker как объект имеет жизненный цикл: создание, работа, завершение, ошибки.

Подглава отвечает на пять вопросов.

**Первый — как создать Worker.** `new Worker(url)` — конструктор, принимающий путь к JS-файлу. Опционально — `{ type: "module" }` для ESM-синтаксиса и `{ name: "..." }` для отладки.

**Второй — где живёт код Worker'а.** Внутри Worker'а нет `window` и `document`. Есть глобальный объект `self` (он же `DedicatedWorkerGlobalScope`) с методами `self.postMessage`, `self.onmessage`, `self.close`.

**Третий — как обмениваться данными.** `worker.postMessage(msg)` в main thread, `self.onmessage` в Worker'е. Ответ — через `self.postMessage` и `worker.onmessage`. Данные проходят через structured clone.

**Четвёртый — как завершить Worker.** `worker.terminate()` — принудительно останавливает выполнение. Немедленно. Без возможности сохранить состояние.

**Пятый — что происходит при ошибке.** Необработанное исключение в Worker'е вызывает `worker.onerror` в main thread и **терминирует Worker**. Это важно: Worker не продолжает работу после необработанного исключения.

---

## 22.2.2. Что уже есть и что появится

К концу 22.1 умеем:
- создавать `OffscreenCanvas` двумя способами;
- передавать DOM-canvas Worker'у через `transferControlToOffscreen`;
- видеть разницу «main thread рисует» vs «Worker рисует».

К концу 22.2 будем уметь:
- создавать Worker из URL;
- понимать, что такое `DedicatedWorkerGlobalScope`;
- отправлять и принимать сообщения через `postMessage` / `onmessage`;
- завершать Worker через `terminate()`;
- обрабатывать `ErrorEvent` из Worker'а;
- использовать module workers (`{ type: "module" }`).

---

## 22.2.3. DedicatedWorkerGlobalScope: глобальный объект Worker'а

Внутри Worker'а нет `window`. Вместо него — `self`, который в случае dedicated worker'а имеет тип `DedicatedWorkerGlobalScope`.

**Что доступно:**

- `self.postMessage(msg)` — отправить сообщение в main thread;
- `self.onmessage = (e) => {}` — принять сообщение от main thread;
- `self.close()` — завершить Worker изнутри;
- `self.name` — имя Worker'а, заданное при создании;
- `fetch`, `setTimeout`, `setInterval`, `requestAnimationFrame` — работают;
- `IndexedDB`, `WebSockets`, `CacheStorage` — доступны.

**Что недоступно:**

- `window`, `document`, `alert`, `localStorage` (в большинстве браузеров);
- прямой доступ к DOM;
- `parent`, `top`, `frameElement`.

Это не ограничение, а **изоляция**. Worker не может случайно изменить DOM или прочитать `localStorage` — он работает в своей песочнице.

---

## 22.2.4. Создание Worker: URL, module, name

**Базовый вариант:**

```js
const worker = new Worker("/workers/echo.js");
```

Worker загружает скрипт по URL и выполняет его в отдельном потоке.

**Module worker:**

```js
const worker = new Worker("/workers/echo.js", { type: "module" });
```

С `type: "module"` Worker может использовать `import` и `export`. Без этого — только классический скрипт.

**С именем:**

```js
const worker = new Worker("/workers/echo.js", { name: "echo-worker" });
```

`name` доступно как `self.name` внутри Worker'а. Помогает в отладке: в DevTools видно, какой Worker чем занят.

**Path и ESM:**

В ESM-контексте (с `import.meta.url`) путь к Worker'у строится относительно текущего модуля:

```js
const worker = new Worker(new URL("./echo.js", import.meta.url), { type: "module" });
```

Это стандартный паттерн для Vite, webpack 5 и Deno.

---

## 22.2.5. postMessage и onmessage: обмен данными

**Main thread → Worker:**

```js
worker.postMessage({ type: "echo", text: "hello" });
```

**Worker принимает:**

```js
self.onmessage = (e) => {
    const msg = e.data;
    // msg = { type: "echo", text: "hello" }
};
```

**Worker → Main thread:**

```js
self.postMessage({ type: "result", text: "HELLO" });
```

**Main thread принимает:**

```js
worker.onmessage = (e) => {
    const msg = e.data;
    // msg = { type: "result", text: "HELLO" }
};
```

Данные проходят через **structured clone**: простые объекты копируются. Функции, DOM-узлы и замыкания — не клонируются и вызывают `DataCloneError`.

**Важно:** `postMessage` — асинхронный. Отправитель не ждёт, пока получатель обработает сообщение. Это не «вызов функции», а «бросить письмо в почтовый ящик».

---

## 22.2.6. terminate: принудительное завершение

```js
worker.terminate();
```

`terminate()` — **немедленно** останавливает Worker. Текущая задача прерывается, очередь сообщений очищается. Никаких `onmessage` больше не будет.

**Когда использовать:**

- Пользователь отменил длительную операцию;
- Worker завис (бесконечный цикл);
- Страница разгружается, нужно освободить ресурсы.

**Чего не делать:**

- Не вызывать `terminate()` из самого Worker'а — для этого есть `self.close()`.
- Не ждать, что `terminate()` вернёт результат — он всегда `undefined`.

**Разница между `terminate()` и `close()`:**

- `worker.terminate()` — из main thread, принудительно.
- `self.close()` — из Worker'а, добровольно. Worker завершается после выполнения текущего обработчика.

---

## 22.2.7. onerror: обработка исключений

Если в Worker'е возникает необработанное исключение, main thread получает событие `error`:

```js
worker.onerror = (event) => {
    console.error("Worker error:", event.message);
    console.error("File:", event.filename, "Line:", event.lineno);
};
```

**Что содержит `ErrorEvent`:**

- `message` — текст ошибки;
- `filename` — файл, где произошла ошибка;
- `lineno`, `colno` — позиция;
- `error` — объект `Error` (если доступен).

**Ключевое:** после `onerror` Worker **терминируется автоматически**. Он не продолжает работу. Это важно понимать: необработанная ошибка в Worker'е — это конец Worker'а.

Чтобы Worker не умирал, нужно **перехватывать** исключения внутри Worker'а:

```js
self.onmessage = (e) => {
    try {
        // опасная операция
    } catch (err) {
        self.postMessage({ type: "error", message: err.message });
    }
};
```

---

## 22.2.8. Что показывает демо

**Layout:**

- **Echo-воркер**: поле ввода, кнопка «Отправить», поле вывода.
- **Индикатор состояния**: «Worker жив / мёртв».
- **Кнопка «terminate»** — завершает Worker.
- **Кнопка «Кинуть ошибку»** — Worker бросает исключение.
- **Кнопка «Пересоздать»** — создаёт новый Worker после terminate.
- **Статистика**: сообщений отправлено, получено, ошибок.

**Сценарий:**

1. При загрузке страницы создаётся Worker. Индикатор — «жив».
2. Пользователь вводит строку, нажимает «Отправить». Worker возвращает её в верхнем регистре. Поле вывода обновляется.
3. Пользователь нажимает «terminate». Worker завершается. Индикатор — «мёртв». Кнопка «Отправить» блокируется.
4. Пользователь нажимает «Пересоздать». Новый Worker создаётся, индикатор — «жив».
5. Пользователь нажимает «Кинуть ошибку». Worker бросает исключение. Main thread ловит `ErrorEvent`, показывает сообщение. Индикатор — «мёртв» (Worker терминирован ошибкой).

**Почему это наглядно.**

- **Первый факт.** Worker общается только через `postMessage`. Никаких общих переменных.
- **Второй факт.** `terminate()` мгновенно убивает Worker. Ответ на следующее сообщение не приходит.
- **Третий факт.** Необработанная ошибка убивает Worker. Это не «предупреждение», а «смерть».

---

## 22.2.9. Wow-момент

**Первый.** Введите строку на русском, нажмите «Отправить». Worker вернёт её в верхнем регистре. Работает и с Unicode — structured clone корректно копирует строки.

**Второй.** Нажмите «terminate», затем «Отправить». Сообщение уходит в никуда. Индикатор — «мёртв». Worker'а больше нет.

**Третий.** Нажмите «Кинуть ошибку». В консоли DevTools появится сообщение от Worker'а. Main thread поймает `ErrorEvent`. Worker терминирован автоматически. Попробуйте отправить сообщение — не работает.

**Четвёртый.** Откройте DevTools → Sources → Workers. Увидите Worker как отдельный поток с именем (если задано `name`). Он живёт независимо от main thread.

---

## 22.2.10. Чему это учит

**Первое.** Worker — это **отдельный поток** с собственным event loop и глобальным объектом. `window` там нет, есть `self`.

**Второе.** Обмен данными — **только через `postMessage`**. Общих переменных нет. Structured clone копирует данные.

**Третье.** `terminate()` — **принудительное** завершение. Мгновенное. Без сохранения состояния.

**Четвёртое.** Необработанная ошибка в Worker'е **убивает Worker**. Нужно ловить исключения внутри.

**Пятое.** `{ type: "module" }` включает ESM-синтаксис в Worker'е. `{ name: "..." }` помогает в отладке.

---

## 22.2.11. Исходные файлы

### `src/dedicated_worker.rs`

```rust
//! 22.2. Dedicated Worker: базовые примитивы.
//!
//! Обёртки вокруг `Worker` и `DedicatedWorkerGlobalScope`:
//!
//! - **feature detection** — поддерживает ли браузер `Worker`;
//! - **создание** — `new Worker(url)` с опциями `type` и `name`;
//! - **отправка сообщений** — `postMessage` со строкой или объектом;
//! - **завершение** — `terminate()`;
//! - **обработка ошибок** — `onerror` и `ErrorEvent`.
//!
//! ## Утилита `bool_of`
//!
//! rust-analyzer теряет тип `matches!` в tail-позиции функции и выдаёт
//! `expected bool, found ()`. Обёртка `bool_of` с явной сигнатурой
//! возвращает типу конкретность.

use js_sys::Reflect;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{ErrorEvent, MessageEvent, Worker, WorkerOptions, WorkerType};

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

/// Глобальный конструктор `Worker` доступен.
pub fn is_worker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("Worker")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Создание Worker
// =============================================================================

/// Создаёт `Worker` из URL (классический скрипт).
pub fn create_worker(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
}

/// Создаёт `Worker` с опциями: `type` (module) и `name`.
pub fn create_worker_with_options(
    url: &str,
    module: bool,
    name: Option<&str>,
) -> Result<Worker, JsValue> {
    let opts = WorkerOptions::new();
    if module {
        opts.set_type(WorkerType::Module);
    }
    if let Some(n) = name {
        opts.set_name(n);
    }
    Worker::new_with_options(url, &opts)
}

// =============================================================================
// Отправка сообщений
// =============================================================================

/// Отправляет строку Worker'у.
pub fn post_string(worker: &Worker, text: &str) -> Result<(), JsValue> {
    worker.post_message(&JsValue::from_str(text))
}

/// Отправляет объект `{ type, text }`.
pub fn post_echo(worker: &Worker, text: &str) -> Result<(), JsValue> {
    let obj = js_sys::Object::new();
    Reflect::set(&obj, &JsValue::from_str("type"), &JsValue::from_str("echo"))?;
    Reflect::set(&obj, &JsValue::from_str("text"), &JsValue::from_str(text))?;
    worker.post_message(&obj)
}

/// Отправляет команду `{ type: "..." }`.
pub fn post_command(worker: &Worker, cmd: &str) -> Result<(), JsValue> {
    let obj = js_sys::Object::new();
    Reflect::set(&obj, &JsValue::from_str("type"), &JsValue::from_str(cmd))?;
    worker.post_message(&obj)
}

// =============================================================================
// Завершение
// =============================================================================

/// Принудительно завершает Worker.
pub fn terminate(worker: &Worker) {
    worker.terminate();
}

// =============================================================================
// Извлечение полей из MessageEvent
// =============================================================================

/// Извлекает поле `type` из `MessageEvent.data` как `String`.
pub fn message_type(event: &MessageEvent) -> Option<String> {
    Reflect::get(&event.data(), &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
}

/// Извлекает поле `text` из `MessageEvent.data` как `String`.
pub fn message_text(event: &MessageEvent) -> Option<String> {
    Reflect::get(&event.data(), &JsValue::from_str("text"))
        .ok()
        .and_then(|v| v.as_string())
}

// =============================================================================
// Извлечение полей из ErrorEvent
// =============================================================================

/// Текст ошибки из `ErrorEvent`.
pub fn error_message(event: &ErrorEvent) -> String {
    event.message()
}

/// Имя файла, где произошла ошибка.
pub fn error_filename(event: &ErrorEvent) -> String {
    event.filename()
}

/// Строка ошибки.
pub fn error_lineno(event: &ErrorEvent) -> u32 {
    event.lineno()
}

/// Столбец ошибки.
pub fn error_colno(event: &ErrorEvent) -> u32 {
    event.colno()
}

/// Внутренний `Error` объект, если доступен.
pub fn error_object(event: &ErrorEvent) -> Option<JsValue> {
    let err = event.error();
    if err.is_undefined() || err.is_null() {
        None
    } else {
        Some(err)
    }
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `offscreen_canvas`:

```rust
pub mod offscreen_worker;     // 22
pub mod offscreen_canvas;     // 22.1
pub mod dedicated_worker;     // ← 22.2
```

### `web/public/workers/dw_echo_worker.js`

```javascript
// 22.2. Echo worker.
//
// Демонстрирует базовые примитивы DedicatedWorkerGlobalScope:
// self.onmessage, self.postMessage, self.close.
// Не использует OffscreenCanvas — фокус на интерфейсе Worker'а.

self.onmessage = (e) => {
    const msg = e.data;

    if (msg.type === "echo") {
        self.postMessage({
            type: "result",
            text: String(msg.text).toUpperCase(),
        });
        return;
    }

    if (msg.type === "throw") {
        // Необработанное исключение — Worker будет терминирован,
        // main thread получит ErrorEvent.
        throw new Error("Намеренная ошибка из Worker'а");
    }

    if (msg.type === "close") {
        self.close();
        return;
    }
};
```

### `tests/wasm_dedicated_worker.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::dedicated_worker::{
    create_worker, create_worker_with_options, is_worker_supported, post_command,
    post_echo, post_string, terminate,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn worker_is_supported_in_browser() {
    assert!(is_worker_supported(), "Worker should be available");
}

#[wasm_bindgen_test]
fn create_worker_returns_object() {
    let w = create_worker("/nonexistent_worker_xyz.js");
    assert!(w.is_ok(), "Worker object should be constructible");
}

#[wasm_bindgen_test]
fn create_worker_with_module_option() {
    let w = create_worker_with_options(
        "/nonexistent_worker_xyz.js",
        true,
        Some("test-worker"),
    );
    assert!(w.is_ok(), "module Worker should be constructible");
}

#[wasm_bindgen_test]
fn post_string_does_not_panic() {
    let Ok(worker) = create_worker("/nonexistent_worker_xyz.js") else {
        return;
    };
    let _ = post_string(&worker, "hello");
}

#[wasm_bindgen_test]
fn post_echo_does_not_panic() {
    let Ok(worker) = create_worker("/nonexistent_worker_xyz.js") else {
        return;
    };
    let _ = post_echo(&worker, "hello");
}

#[wasm_bindgen_test]
fn post_command_does_not_panic() {
    let Ok(worker) = create_worker("/nonexistent_worker_xyz.js") else {
        return;
    };
    let _ = post_command(&worker, "throw");
}

#[wasm_bindgen_test]
fn terminate_does_not_panic() {
    let Ok(worker) = create_worker("/nonexistent_worker_xyz.js") else {
        return;
    };
    terminate(&worker);
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Wasm-тесты только для этой подглавы
wasm-pack test --headless --chrome --test wasm_dedicated_worker

# 3. Демо
npm run wasm:dev && npm start
```

Убедитесь, что файл `web/public/workers/dw_echo_worker.js` доступен по адресу `/workers/dw_echo_worker.js`. Vite монтирует папку `public` в корень автоматически.

---

## 22.2.12. `src/app.rs` — автономный демо-пример

```rust
//! Глава 22.2 — Dedicated Worker: базовые примитивы.
//!
//! Демо-лаборатория: echo-воркер, который возвращает строку в верхнем
//! регистре. Main thread управляет Worker'ом: отправляет сообщения,
//! завершает, пересоздаёт, ловит ошибки.
//!
//! ## Что демонстрируется
//!
//! - `new Worker(url)` — создание dedicated-воркера.
//! - `postMessage` / `onmessage` — обмен данными.
//! - `worker.terminate()` — принудительное завершение.
//! - `worker.onerror` — ловля исключений из Worker'а.
//! - `ErrorEvent` — поля `message`, `filename`, `lineno`, `colno`.
//! - **Необработанная ошибка убивает Worker** — важное свойство модели.
//!
//! ## Правило Rc<RefCell<State>>
//!
//! `borrow()` и `borrow_mut()` не пересекаются. Каждый borrow —
//! в узком scope, `Ref`/`RefMut` drop'аются до следующего borrow.
//!
//! **Файл автономен.** Стили `.dw-*` инжектируются в документ.
//! Wow: нажмите «Кинуть ошибку» — Worker умирает, main thread
//! получает ErrorEvent и показывает подробности.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Document, ErrorEvent, HtmlElement, HtmlInputElement, MessageEvent, Worker,
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
    let id = "dw-injected-style";
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
.dw-desc {
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
.dw-desc-line { margin: 6px 0; }

.dw-block {
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

.dw-input {
    flex: 1;
    min-width: 200px;
    padding: 8px 12px;
    border-radius: 6px;
    border: 1px solid #30363d;
    background: #0d1117;
    color: #c9d1d9;
    font-size: 13px;
    font-family: ui-monospace, monospace;
}
.dw-input:focus {
    outline: none;
    border-color: #58a6ff;
}

.dw-output {
    max-width: 900px;
    padding: 10px 14px;
    background: #0d1117;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
    font-size: 13px;
    line-height: 1.6;
    margin-bottom: 16px;
    min-height: 24px;
    word-break: break-all;
}
.dw-output.dw-error {
    border-left-color: #f85149;
    color: #f85149;
}

.dw-btn {
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
.dw-btn:hover:not([disabled]) { background: #30363d; }
.dw-btn:active:not([disabled]) { transform: translateY(1px); }
.dw-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.dw-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.dw-btn-primary:hover:not([disabled]) { background: #2ea043; }
.dw-btn-warn {
    background: #21262d;
    border-color: #30363d;
    color: #f0883e;
}
.dw-btn-warn:hover:not([disabled]) { background: #30363d; }
.dw-btn-danger {
    background: #21262d;
    border-color: #30363d;
    color: #f85149;
}
.dw-btn-danger:hover:not([disabled]) { background: #30363d; }

.dw-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
}

.dw-stats {
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
.dw-stats .label { color: #8b949e; }
.dw-stats .value { color: #79c0ff; }
.dw-stats .ok { color: #3fb950; }
.dw-stats .warn { color: #f0883e; }
.dw-stats .err { color: #f85149; }

.dw-footnote {
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

const WORKER_URL: &str = "/workers/dw_echo_worker.js";

// =============================================================================
// Состояние демо
// =============================================================================

struct DemoState {
    worker: Option<Worker>,
    alive: bool,
    sent: u32,
    received: u32,
    errors: u32,
}

impl DemoState {
    fn new() -> Self {
        Self {
            worker: None,
            alive: false,
            sent: 0,
            received: 0,
            errors: 0,
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

// =============================================================================
// Работа с Worker
// =============================================================================

fn create_worker(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
}

fn post_echo(worker: &Worker, text: &str) -> Result<(), JsValue> {
    let obj = Object::new();
    Reflect::set(&obj, &JsValue::from_str("type"), &JsValue::from_str("echo"))?;
    Reflect::set(&obj, &JsValue::from_str("text"), &JsValue::from_str(text))?;
    worker.post_message(&obj)
}

fn post_throw(worker: &Worker) -> Result<(), JsValue> {
    let obj = Object::new();
    Reflect::set(&obj, &JsValue::from_str("type"), &JsValue::from_str("throw"))?;
    worker.post_message(&obj)
}

fn terminate_worker(worker: &Worker) {
    worker.terminate();
}

fn message_type(event: &MessageEvent) -> Option<String> {
    Reflect::get(&event.data(), &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
}

fn message_text(event: &MessageEvent) -> Option<String> {
    Reflect::get(&event.data(), &JsValue::from_str("text"))
        .ok()
        .and_then(|v| v.as_string())
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("dw-stats") else {
        return;
    };
    let status = if st.alive { "жив" } else { "мёртв" };
    let status_class = if st.alive { "ok" } else { "err" };
    let html = format_str(format_args!(
        r#"<div><span class="label">Состояние: </span><span class="{sc}">{status}</span></div>
           <div><span class="label">Отправлено: </span><span class="value">{sent}</span> · <span class="label">получено: </span><span class="value">{recv}</span> · <span class="label">ошибок: </span><span class="err">{errs}</span></div>"#,
        sc = status_class,
        status = status,
        sent = st.sent,
        recv = st.received,
        errs = st.errors,
    ));
    el.set_inner_html(&html);
}

fn set_output(document: &Document, text: &str, is_error: bool) {
    let Some(el) = document.get_element_by_id("dw-output") else {
        return;
    };
    el.set_text_content(Some(text));
    if is_error {
        el.set_class_name("dw-output dw-error");
    } else {
        el.set_class_name("dw-output");
    }
}

fn set_enabled(document: &Document, id: &str, enabled: bool) {
    if let Some(el) = document.get_element_by_id(id) {
        if let Ok(btn) = el.dyn_into::<HtmlElement>() {
            if enabled {
                let _ = btn.remove_attribute("disabled");
            } else {
                let _ = btn.set_attribute("disabled", "disabled");
            }
        }
    }
}

fn refresh_ui(document: &Document, st: &DemoState) {
    update_stats(document, st);
    set_enabled(document, "dw-send", st.alive);
    set_enabled(document, "dw-throw", st.alive);
    set_enabled(document, "dw-terminate", st.alive);
    set_enabled(document, "dw-recreate", !st.alive);
}

// =============================================================================
// Точка входа
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Echo-воркер: строка уходит в Worker, возвращается в верхнем регистре. \
     Обмен — только через <code>postMessage</code> и <code>onmessage</code>. \
     Никаких общих переменных, никаких прямых вызовов.",
    "Кнопка <strong>«terminate»</strong> мгновенно завершает Worker. \
     После этого «Отправить» и «Кинуть ошибку» блокируются. Worker'а \
     больше нет — сообщение уходит в никуда.",
    "Кнопка <strong>«Кинуть ошибку»</strong> заставляет Worker выбросить \
     необработанное исключение. Main thread получает <code>ErrorEvent</code>, \
     а Worker <strong>терминируется автоматически</strong>. Это свойство \
     модели: после <code>onerror</code> Worker'а уже нет.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("dw-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("dw-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

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
    h2.set_text_content(Some("22.2 — Dedicated Worker: базовые примитивы"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_worker_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("dw-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает Worker. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Панель ввода.
    let input_row = document.create_element("div")?;
    input_row.set_class_name("dw-block");
    input_row.set_inner_html(
        r#"<input type="text" id="dw-input" class="dw-input"
                  placeholder="Введите строку…" value="hello worker">
           <button class="dw-btn dw-btn-primary" id="dw-send">Отправить</button>"#,
    );
    root.append_child(&input_row)?;

    // Панель управления.
    let ctrl_row = document.create_element("div")?;
    ctrl_row.set_class_name("dw-block");
    ctrl_row.set_inner_html(
        r#"<button class="dw-btn dw-btn-danger" id="dw-terminate">terminate</button>
           <button class="dw-btn dw-btn-warn" id="dw-throw">Кинуть ошибку</button>
           <button class="dw-btn dw-btn-primary" id="dw-recreate" disabled>Пересоздать</button>"#,
    );
    root.append_child(&ctrl_row)?;

    // Вывод.
    let output = document.create_element("div")?;
    output.set_class_name("dw-output");
    output.set_id("dw-output");
    output.set_text_content(Some("—"));
    root.append_child(&output)?;

    // Статистика.
    let stats = document.create_element("div")?;
    stats.set_class_name("dw-stats");
    stats.set_id("dw-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("dw-footnote");
    footnote.set_text_content(Some(
        "DedicatedWorkerGlobalScope не имеет window и document. Внутри \
         Worker'а доступен только self с методами postMessage, onmessage, \
         close. Обмен данными — через structured clone: простые объекты \
         копируются, функции и DOM-узлы вызывают DataCloneError.",
    ));
    root.append_child(&footnote)?;

    // Состояние.
    let state = Rc::new(RefCell::new(DemoState::new()));

    // Создаём Worker.
    let worker = create_worker(WORKER_URL)?;
    {
        let mut st = state.borrow_mut();
        st.worker = Some(worker.clone());
        st.alive = true;
    }

    // --- Обработчик onmessage ---
    {
        let state2 = state.clone();
        let doc = document.clone();
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut(MessageEvent)>::new(
            move |ev: MessageEvent| {
                let kind = message_type(&ev).unwrap_or_default();
                if kind == "result" {
                    let text = message_text(&ev).unwrap_or_default();
                    {
                        let mut st = state2.borrow_mut();
                        st.received += 1;
                    }
                    set_output(&doc, &text, false);
                    let st = state2.borrow();
                    update_stats(&doc, &st);
                }
            },
        );
        worker.set_onmessage(Some(closure.as_ref().unchecked_ref()));
        closure.forget();
    }

    // --- Обработчик onerror ---
    {
        let state2 = state.clone();
        let doc = document.clone();
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut(ErrorEvent)>::new(
            move |ev: ErrorEvent| {
                let msg = ev.message();
                let file = ev.filename();
                let line = ev.lineno();
                let col = ev.colno();
                {
                    let mut st = state2.borrow_mut();
                    st.errors += 1;
                    st.alive = false;
                }
                let text = format_str(format_args!(
                    "WorkerError: {msg}\n  at {file}:{line}:{col}"
                ));
                set_output(&doc, &text, true);
                let st = state2.borrow();
                refresh_ui(&doc, &st);
            },
        );
        worker.set_onerror(Some(closure.as_ref().unchecked_ref()));
        closure.forget();
    }

    // --- Кнопка «Отправить» ---
    {
        let state2 = state.clone();
        let doc = document.clone();
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let Some(input) = doc.get_element_by_id("dw-input") else {
                return;
            };
            let Ok(input) = input.dyn_into::<HtmlInputElement>() else {
                return;
            };
            let text = input.value();

            let worker = {
                let st = state2.borrow();
                st.worker.clone()
            };
            let Some(w) = worker else { return };

            if post_echo(&w, &text).is_ok() {
                {
                    let mut st = state2.borrow_mut();
                    st.sent += 1;
                }
                set_output(&doc, "…", false);
                let st = state2.borrow();
                update_stats(&doc, &st);
            }
        });
        let btn = document
            .get_element_by_id("dw-send")
            .ok_or_else(|| JsValue::from_str("no dw-send"))?
            .dyn_into::<HtmlElement>()?;
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // --- Кнопка «terminate» ---
    {
        let state2 = state.clone();
        let doc = document.clone();
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let worker = {
                let st = state2.borrow();
                st.worker.clone()
            };
            let Some(w) = worker else { return };
            terminate_worker(&w);
            {
                let mut st = state2.borrow_mut();
                st.alive = false;
            }
            set_output(&doc, "Worker завершён через terminate().", false);
            let st = state2.borrow();
            refresh_ui(&doc, &st);
        });
        let btn = document
            .get_element_by_id("dw-terminate")
            .ok_or_else(|| JsValue::from_str("no dw-terminate"))?
            .dyn_into::<HtmlElement>()?;
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // --- Кнопка «Кинуть ошибку» ---
    {
        let state2 = state.clone();
        let doc = document.clone();
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let worker = {
                let st = state2.borrow();
                st.worker.clone()
            };
            let Some(w) = worker else { return };
            let _ = post_throw(&w);
            set_output(&doc, "… (ожидаем ErrorEvent)", false);
        });
        let btn = document
            .get_element_by_id("dw-throw")
            .ok_or_else(|| JsValue::from_str("no dw-throw"))?
            .dyn_into::<HtmlElement>()?;
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // --- Кнопка «Пересоздать» ---
    {
        let state2 = state.clone();
        let doc = document.clone();
        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let Ok(w) = create_worker(WORKER_URL) else {
                set_output(&doc, "Не удалось создать Worker.", true);
                return;
            };

            // onmessage.
            {
                let state3 = state2.clone();
                let doc3 = doc.clone();
                let c = wasm_bindgen::closure::Closure::<dyn FnMut(MessageEvent)>::new(
                    move |ev: MessageEvent| {
                        if message_type(&ev).as_deref() == Some("result") {
                            let text = message_text(&ev).unwrap_or_default();
                            {
                                let mut st = state3.borrow_mut();
                                st.received += 1;
                            }
                            set_output(&doc3, &text, false);
                            let st = state3.borrow();
                            update_stats(&doc3, &st);
                        }
                    },
                );
                w.set_onmessage(Some(c.as_ref().unchecked_ref()));
                c.forget();
            }

            // onerror.
            {
                let state3 = state2.clone();
                let doc3 = doc.clone();
                let c = wasm_bindgen::closure::Closure::<dyn FnMut(ErrorEvent)>::new(
                    move |ev: ErrorEvent| {
                        let msg = ev.message();
                        {
                            let mut st = state3.borrow_mut();
                            st.errors += 1;
                            st.alive = false;
                        }
                        set_output(&doc3, &format_str(format_args!("WorkerError: {msg}")), true);
                        let st = state3.borrow();
                        refresh_ui(&doc3, &st);
                    },
                );
                w.set_onerror(Some(c.as_ref().unchecked_ref()));
                c.forget();
            }

            {
                let mut st = state2.borrow_mut();
                st.worker = Some(w);
                st.alive = true;
            }
            set_output(&doc, "Worker пересоздан.", false);
            let st = state2.borrow();
            refresh_ui(&doc, &st);
        });
        let btn = document
            .get_element_by_id("dw-recreate")
            .ok_or_else(|| JsValue::from_str("no dw-recreate"))?
            .dyn_into::<HtmlElement>()?;
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Первичный рендер.
    {
        let st = state.borrow();
        refresh_ui(&document, &st);
    }

    Ok(())
}
```

---

## 22.2.13. Анализ решения

**Какую ценность даёт демо.**

Подглава показывает **жизненный цикл Worker'а** на одном экране:

- **Создание** — Worker появляется при загрузке страницы.
- **Работа** — echo: строка уходит, возвращается в верхнем регистре.
- **Завершение** — `terminate()` мгновенно убивает Worker.
- **Пересоздание** — новый Worker создаётся по кнопке.
- **Ошибка** — необработанное исключение убивает Worker, main thread ловит `ErrorEvent`.

Все пять состояний видны в одной панели. Индикатор «жив / мёртв» меняется в реальном времени.

**Скрытие технических сложностей.**

- Пользователь вводит строку, нажимает «Отправить», видит результат. Внутри — `postMessage`, `onmessage`, `structured clone`.
- Пользователь нажимает «terminate» — Worker мгновенно завершается. Кнопки блокируются. Не нужно объяснять, что такое `Worker.terminate()` — поведение говорит само за себя.
- Пользователь нажимает «Кинуть ошибку» — main thread показывает `ErrorEvent` с координатами. Worker умирает. Видно, что после ошибки Worker'а нет.

**Что осознанно не реализовано.**

- **Module workers** (`{ type: "module" }`). Покрыты feature detection в модуле и тестами. В демо — классический скрипт, потому что echo-воркеру не нужен `import`.
- **`self.close()`** — завершение изнутри. Покрыто в модуле, но не показано: визуально не отличается от `terminate()`.
- **`worker.name`** — имя для отладки. Покрыто в модуле.
- **MessageChannel.** Не тема 22.2.
- **Transferable.** Не тема 22.2.

**Правила проекта, применённые в этой подглаве.**

1. **Autonomous app.rs.** Всё inline: `format_str`, `bool_of`, feature detection, работа с Worker.
2. **`format_str(format_args!(...))` в tail-позициях.** В `update_stats`, `format_str` для `ErrorEvent`.
3. **`bool_of(matches!(...))` в tail-позиции.** В `is_worker_supported`.
4. **RefMut-дисциплина.** Каждый `borrow_mut` в узком scope, `borrow` — после drop.
5. **Feature detection перед использованием API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.** Правок в `main.css` не требуется.
8. **Уникальный префикс `dw-`.** Не пересекается с `oc-` (22.1), `ow-` (глава 22), `canvas-` (глава 21).
9. **Строки — через одну функцию `format_str`.** Никаких `format!` в tail-позиции.
10. **Никаких неиспользуемых импортов.** Все импорты задействованы.

**Проверки корректности.**

- `Closure` для `onmessage`, `onerror`, кнопок сохраняются через `closure.forget()` — это правильный способ для долгоживущих обработчиков.
- При пересоздании Worker'а создаются **новые** `Closure` для `onmessage` и `onerror` — они привязываются к новому Worker'у.
- `set_enabled` использует `remove_attribute` / `set_attribute` для управления `disabled` — работает надёжно.
- `post_throw` отправляет `{ type: "throw" }`, Worker бросает `Error` — main thread ловит `ErrorEvent`.
- После `terminate()` кнопка «Пересоздать» разблокируется, «Отправить», «terminate», «Кинуть ошибку» — блокируются.

---

## 22.2.14. Список доработок после первой итерации

1. **`bool_of` в feature detection.** `matches!(…)` в tail-позиции даёт E0308 от rust-analyzer. Утилита `bool_of` применяется и в модуле, и в `app.rs`.
2. **Пересоздание Worker'а с новыми Closure.** При повторном создании Worker'а старые `Closure` привязаны к мёртвому Worker'у. Нужно создавать новые — реализовано в кнопке «Пересоздать».
3. **Кнопки блокируются после terminate.** `set_enabled` вызывается из `refresh_ui` — после terminate кнопка «Отправить» становится `disabled`.
4. **ErrorEvent показывает file:line:col.** `ev.message()`, `ev.filename()`, `ev.lineno()`, `ev.colno()` — все четыре поля используются.
5. **Статистика: sent / received / errors.** Счётчики обновляются при каждом событии. Видно, что после ошибки `received` не растёт, а `errors` — растёт.
6. **Убран лишний импорт `Event`.** В `app.rs` не используется `Event` — обработчики принимают `MessageEvent`, `ErrorEvent` или ничего.
7. **`WorkerOptions` в модуле.** `create_worker_with_options` использует `WorkerOptions::new()`, `set_type`, `set_name` — тесты покрывают.

---

## 22.2.15. Итог

`DedicatedWorkerGlobalScope` — глобальный объект Worker'а. Внутри нет `window` и `document`, есть `self` с методами `postMessage`, `onmessage`, `close`.

Создание Worker'а: `new Worker(url)`, опционально `{ type: "module" }` для ESM и `{ name: "..." }` для отладки.

Обмен данными — только через `postMessage` / `onmessage`. Structured clone копирует данные. Функции и DOM-узлы не клонируются.

`worker.terminate()` — принудительное завершение. Мгновенное. Без сохранения состояния.

Необработанное исключение в Worker'е **убивает Worker** и вызывает `ErrorEvent` в main thread. Это не «предупреждение», а «смерть». Нужно ловить исключения внутри Worker'а, если он должен продолжать работу.

Демо показывает все пять состояний Worker'а на одном экране: создание, работа, завершение, пересоздание, ошибка. Индикатор «жив / мёртв» и счётчики дают мгновенную обратную связь.

Подглава закрывает тему **интерфейса Worker'а**. Transferable-объекты, MessageChannel, WASM в Worker'е, пулы — следующие подглавы (22.3–22.6).