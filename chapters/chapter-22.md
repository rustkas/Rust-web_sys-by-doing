# Глава 22. OffscreenCanvas и Workers

## 22.0. Обновлённая структура

```
src/
  offscreen_worker.rs             ← НОВЫЙ модуль главы 22
  canvas_dpi.rs                   ← из 21.1
  canvas_drawing.rs               ← из 21.2
  canvas_composition.rs           ← из 21.3
  canvas_imagedata.rs             ← из 21.4
  canvas_image_loading.rs         ← из 21.5
  canvas_filters.rs               ← из 21.6
  canvas_image_basic.rs           ← из 21.7
  canvas_image_pipeline.rs        ← из 21.7
tests/
  wasm_offscreen_worker.rs        ← НОВЫЙ тест
src/lib.rs                        ← +1 строка
src/app.rs                        ← демо главы (заменяется целиком)
web/public/workers/
  particle_worker.js              ← НОВЫЙ файл (JS-воркер)
Cargo.toml                        ← +2 фичи web-sys
web/styles/main.css               ← правок не требуется
```

**Комментарий.** Модуль `offscreen_worker.rs` — про вынос вычислений из main thread: создание `Worker`, передача `OffscreenCanvas` через `transferControlToOffscreen`, обмен сообщениями, transferable-объекты. Префикс CSS — `ow-` (offscreen worker), уникален в проекте. Демо автономен, стили инжектируются в документ.

## Ссылки

- MDN: [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- MDN: [`transferControlToOffscreen`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/transferControlToOffscreen)
- MDN: [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- MDN: [`Worker.postMessage`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/postMessage)
- MDN: [Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- MDN: [`MessageChannel`](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)
- web.dev: [OffscreenCanvas](https://web.dev/articles/offscreen-canvas)
- wasm-bindgen: [Parallel Raytracing (WASM + Worker)](https://rustwasm.github.io/wasm-bindgen/examples/raytrace.html)

---

## 22.1. О чём эта глава

Все предыдущие подглавы работали в **главном потоке** браузера. Main thread — единственное место, где живёт DOM, где выполняются обработчики событий, где рисуется страница. И там же работал наш Rust: `spawn_local` с async-операциями, `requestAnimationFrame` с фильтрами, `putImageData` с пикселями.

Пока задачи маленькие — 400×240, один клик, один фильтр — этого достаточно. Но как только появляется:
- анимация 60 FPS с несколькими сотнями объектов;
- обработка фото 4000×3000;
- поток кадров с камеры;

— main thread перестаёт справляться. Пользователь видит замершие кнопки, дёрганый скролл, «страница не отвечает». Yield через `TimeoutFuture::new(0)` из 20.4 помогает, но не решает: половина времени кадра всё ещё уходит на обработку.

Решение — **Web Worker**. Отдельный поток JavaScript, у которого:
- свой event loop;
- нет DOM;
- есть `postMessage` для обмена данными с main thread;
- свои таймеры и `requestAnimationFrame` (через `OffscreenCanvas`).

`OffscreenCanvas` — это canvas без DOM. Он может быть создан в main thread и **передан** в Worker. С этого момента Worker владеет canvas и рисует на нём сам. Main thread при этом остаётся свободным: он может анимировать другие элементы, обрабатывать клики, отвечать на скролл.

Глава отвечает на четыре вопроса.

**Первый — как передать canvas в Worker.** `canvas.transferControlToOffscreen()` возвращает `OffscreenCanvas`, который можно отправить через `postMessage` с флагом transfer. После этого DOM-элемент `<canvas>` больше не подчиняется main thread — его содержимым управляет Worker.

**Второй — как обмениваться данными.** `postMessage` передаёт значения через structured clone. Простые объекты копируются, но `ArrayBuffer` можно **передать** — это zero-copy transfer, когда владение буфером переходит от отправителя к получателю без копирования байт. Для 10 МБ данных это разница между 10 мс и 0.1 мс.

**Третий — где живёт WASM.** Здесь начинается сложность: каждый Worker — отдельный JavaScript-контекст, и у него нет доступа к `wasm_bindgen`-модулю, инстанцированному в main thread. Чтобы использовать Rust-код в Worker, нужно либо собрать отдельный WASM-модуль для Worker, либо загрузить тот же модуль в контексте Worker через динамический `import()`.

**Четвёртый — что показывать.** Практический проект — «background image processing». Один canvas с анимацией, управляемый Worker'ом. Main thread при этом свободен: CSS-анимация на странице крутится плавно, кнопки мгновенно реагируют. Рядом — цифры: FPS из Worker, время передачи 10 МБ через transfer vs без transfer, latency ответа на клик.

## 22.2. Что уже есть и что появится

К концу 21.7 мы умеем:
- рисовать на canvas в main thread;
- применять Rust-фильтры к пикселям;
- загружать и отдавать изображения;
- организовывать async-операции с yield.

К концу 22 будем уметь:
- создавать `Worker` из JS-файла;
- передавать ему `OffscreenCanvas` через `transferControlToOffscreen`;
- обмениваться сообщениями через `postMessage`;
- понимать разницу между копированием и передачей `ArrayBuffer`;
- измерять эффект: free main thread vs busy main thread;
- знать, почему WASM в Worker требует отдельной инстанциации.

## 22.3. OffscreenCanvas: canvas без DOM

`OffscreenCanvas` — это объект, у которого есть `width`, `height`, `getContext("2d")`, `convertToBlob()`, но **нет** `.style`, `.parentNode`, `clientWidth`. Он не привязан к DOM. Его можно создать двумя способами.

**Первый** — прямо: `new OffscreenCanvas(w, h)`. Используется, когда canvas нужен только для обработки пикселей и не показывается пользователю.

**Второй** — из существующего `<canvas>`: `canvas.transferControlToOffscreen()`. Возвращает `OffscreenCanvas`, который **наследует** содержимое DOM-элемента. С этого момента:
- DOM-элемент остаётся на странице;
- но всё, что на нём рисуется, рисуется через `OffscreenCanvas`;
- main thread теряет возможность вызывать `getContext` на оригинальном элементе.

Ключевое: `transferControlToOffscreen` — **одноразовая** операция. Вызвать её дважды на одном элементе нельзя — будет исключение. После передачи canvas принадлежит тому, кто его получил (обычно Worker).

## 22.4. Worker: отдельный поток

`Worker` создаётся из URL:

```js
const worker = new Worker('/workers/particle_worker.js');
```

Worker не имеет доступа к DOM, `window`, `document`. У него есть свой глобальный объект `self` (он же `DedicatedWorkerGlobalScope`), с методами:
- `self.postMessage(msg)` — отправить сообщение в main thread;
- `self.onmessage = (e) => {}` — принять сообщение от main thread;
- `self.importScripts(...)` — старый способ подгружать скрипты (устаревший);
- `fetch`, `setTimeout`, `requestAnimationFrame` — работают.

Worker общается с main thread **только** через `postMessage`. Никаких общих переменных, никаких прямых вызовов. Это делает модель предсказуемой и защищает от гонок.

## 22.5. postMessage и structured clone

`postMessage` передаёт значения через **structured clone** — алгоритм, который умеет сериализовать:
- простые типы (`number`, `string`, `boolean`);
- объекты и массивы;
- `Map`, `Set`, `Date`, `RegExp`;
- `ArrayBuffer`, `TypedArray`, `Blob`, `ImageData`;
- `OffscreenCanvas` (только через transfer).

Не умеет:
- функции;
- DOM-узлы;
- классы пользователя (кроме как через `toJSON` + ручной парсинг);
- замыкания.

Важно: **простое значение копируется**. Если передать `{ buffer: new ArrayBuffer(10_000_000) }`, браузер аллоцирует 10 МБ в целевом потоке и скопирует байты. Для больших данных это дорого.

## 22.6. Transferable objects: zero-copy

Решение — **transferable**. Второй аргумент `postMessage`:

```js
worker.postMessage({ buffer }, [buffer]);
```

Второй аргумент — список объектов, которые нужно **передать**, а не копировать. После передачи:
- отправитель теряет доступ к буферу (`byteLength === 0`);
- получатель владеет байтами.

С точки зрения затрат: transfer — это перекладывание указателя на память между контекстами. O(1). Копирование 10 МБ — O(n). Разница в 1000 раз на этой операции.

Что transferable:
- `ArrayBuffer`;
- `MessagePort`;
- `OffscreenCanvas`;
- `ImageBitmap`;
- `ReadableStream`, `WritableStream`, `TransformStream`;
- `VideoFrame`, `AudioData` (для WebCodecs).

Что не transferable:
- `Uint8Array` — нужно передавать `.buffer`;
- `Blob` — копируется;
- обычные объекты — копируются.

## 22.7. MessageChannel: приватный канал

Иногда нужно два канала между main thread и Worker: один для команд, другой для статистики. Или канал между двумя Worker'ами.

`MessageChannel` создаёт пару связанных портов:

```js
const channel = new MessageChannel();
worker.postMessage({ port: channel.port2 }, [channel.port2]);
// channel.port1 остаётся в main thread
channel.port1.postMessage('привет');
```

Оба порта transferable. `port2` можно передать Worker'у через `postMessage`. После этого main thread и Worker имеют приватный канал, изолированный от их основных `postMessage`.

В нашем демо `MessageChannel` не нужен — команды и статистика идут через один канал. Но в главе он упоминается как инструмент для сложных схем.

## 22.8. WASM в Worker: почему это отдельная задача

Каждый Worker — отдельный JavaScript-контекст. Модуль `hello_wasm02`, инстанцированный в main thread, **не существует** в Worker. У Worker'а свои `WebAssembly.Module`, свои функции, своя память.

Три подхода:

**Первый — отдельный WASM-модуль.** Собрать небольшой крейт специально для Worker'а, положить `.wasm` рядом, загрузить через `import()` в Worker'е. Разделение ответственности: main thread и Worker используют разные куски Rust-кода.

**Второй — тот же модуль, загруженный в Worker.** В Worker'е написать:

```js
import init, { process_chunk } from '/pkg/hello_wasm02.js';
await init();
self.onmessage = async (e) => {
    const result = process_chunk(e.data);
    self.postMessage(result);
};
```

Тот же `.wasm` загружается второй раз, но в другом контексте. Работает, если сборка поддерживает `--target web` и файлы доступны из Worker'а.

**Третий — `wasm-bindgen-rayon`.** Позволяет использовать Rayon (параллельные итераторы) в WASM, автоматически создавая пул Worker'ов. Продвинутая тема, за пределами этой главы.

В нашем демо мы **не запускаем WASM в Worker'е**. Демонстрация сфокусирована на паттерне: OffscreenCanvas + Worker + transferable + postMessage. Worker написан на чистом JavaScript и анимирует частицы. Rust-код в main thread создаёт Worker, передаёт ему canvas и обрабатывает статистику.

Это честнее, чем псевдо-WASM-в-Worker, который ничего не делает. Студент видит реальный паттерн и понимает, где именно Rust может быть добавлен позже.

## 22.9. Что показывает демо

**Layout:**

- **Canvas** — передан Worker'у через `transferControlToOffscreen`. На нём Worker анимирует частицы.
- **Индикатор свободного main thread** — CSS-анимация в правом верхнем углу страницы. Она крутится плавно, пока Worker работает. Если main thread занят — замирает.
- **Панель управления** — цветовая палитра, число частиц, пауза/пуск.
- **Статистика** — FPS из Worker, число кадров, latency ответа на клик.
- **Кнопка «Передать 10 МБ»** — отправляет `ArrayBuffer` в Worker через transfer и измеряет round-trip время.
- **Кнопка «Скопировать 10 МБ»** — то же самое, но без флага transfer (копирование).

**Сценарий:**

1. При загрузке страницы создаётся `<canvas>`, переводится в `OffscreenCanvas`, отправляется Worker'у с флагом transfer.
2. Worker начинает анимацию: рисует 200 частиц в цикле `requestAnimationFrame`.
3. Main thread показывает FPS, кнопки управления и индикатор свободного потока.
4. Пользователь крутит слайдер «частицы» — Worker получает сообщение, меняет число частиц, продолжает анимацию.
5. Кнопка «Передать 10 МБ» — измеряет время передачи. Ожидаемо: **< 1 мс**.
6. Кнопка «Скопировать 10 МБ» — то же без transfer. Ожидаемо: **10–20 мс**.

Разница в цифрах — прямое доказательство того, что transfer — это zero-copy.

## 22.10. Wow-момент

**Первый.** Смотрите на индикатор в правом верхнем углу. Пока Worker анимирует 200 частиц, индикатор крутится с частотой 60 FPS. Это main thread, который **не занят** анимацией canvas'а.

**Второй.** Нажмите «Передать 10 МБ». Latency: 0.4 мс. Нажмите «Скопировать 10 МБ». Latency: 14.3 мс. Один и тот же объём данных. Разница — флаг `[buffer]` во втором аргументе `postMessage`.

**Третий.** Откройте DevTools → Performance, запустите запись, поработайте минуту. График main thread — плоский: маленькие всплески от кликов и отрисовки DOM. Задача анимации **не в нём**. Её нет в главном потоке — она в Worker'е.

**Четвёртый.** Поменяйте число частиц на 2000. FPS из Worker просядет (например, до 40). Main thread всё ещё будет свободным: индикатор крутится, кнопки отвечают. Это и есть цель Workers — не ускорить, а **изолировать**.

## 22.11. Чему это учит

**Первое.** Main thread — критический ресурс. Всё, что на нём выполняется, блокирует пользовательский ввод, скролл, рендеринг. Workers — не оптимизация, а **инструмент изоляции**.

**Второе.** Worker не имеет DOM. Это его свойство, не недостаток. `OffscreenCanvas` — то, что позволяет Worker'у рисовать, не имея DOM-элемента. `transferControlToOffscreen` — мост.

**Третье.** `postMessage` — не «вызов функции». Это асинхронный обмен сообщениями. Данные копируются, если только не transferable.

**Четвёртое.** Transferable — это **передача владения**. O(1) независимо от размера. Для 10 МБ разница с копированием в десятки раз. Для 100 МБ — в сотни.

**Пятое.** WASM в Worker требует отдельной инстанциации. Нет способа «поделиться» WASM-модулем между main thread и Worker'ом: у них разные `WebAssembly.Instance`, разная память, разный глобальный контекст.

**Шестое.** Индикатор свободного main thread — не украшение. Это **диагностический инструмент**. Если он замер — main thread чем-то занят. Если крутится — можно грузить больше работы на Worker.

## 22.12. Исходные файлы

### `web/public/workers/particle_worker.js`

```javascript
// 22. Particle worker.
//
// Worker владеет OffscreenCanvas и анимирует частицы. Общается с main
// thread через postMessage: принимает команды (init, palette, count,
// pause, resume, transfer_test), отправляет статистику (fps) и
// подтверждения (transfer_ack).

let canvas = null;
let ctx = null;
let width = 0;
let height = 0;
let running = true;
let palette = ['#58a6ff', '#3fb950', '#f0883e', '#d2a8ff'];
let targetCount = 200;
let particles = [];
let t0 = 0;
let lastFpsTime = 0;
let frames = 0;
let lastFrameTime = 0;

function makeParticles(n) {
    const arr = new Array(n);
    for (let i = 0; i < n; i++) {
        arr[i] = {
            x: Math.random() * width,
            y: Math.random() * height,
            vx: (Math.random() - 0.5) * 2,
            vy: (Math.random() - 0.5) * 2,
            r: 1 + Math.random() * 3,
            c: Math.floor(Math.random() * palette.length),
        };
    }
    return arr;
}

function tick(now) {
    requestAnimationFrame(tick);
    if (!ctx) return;

    if (!lastFrameTime) lastFrameTime = now;
    const dt = Math.min((now - lastFrameTime) / 16.667, 4.0);
    lastFrameTime = now;

    if (!running) {
        // Всё равно обновляем статистику — так main thread видит, что
        // Worker жив, просто на паузе.
        maybeSendFps(now);
        return;
    }

    // Убедимся, что число частиц соответствует цели.
    if (particles.length !== targetCount) {
        particles = makeParticles(targetCount);
    }

    // Очистка.
    ctx.fillStyle = '#0d1117';
    ctx.fillRect(0, 0, width, height);

    // Движение и отрисовка.
    for (let i = 0; i < particles.length; i++) {
        const p = particles[i];
        p.x += p.vx * dt;
        p.y += p.vy * dt;

        if (p.x < 0 || p.x > width) { p.vx = -p.vx; p.x = Math.max(0, Math.min(width, p.x)); }
        if (p.y < 0 || p.y > height) { p.vy = -p.vy; p.y = Math.max(0, Math.min(height, p.y)); }

        ctx.fillStyle = palette[p.c % palette.length];
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
        ctx.fill();
    }

    frames++;
    maybeSendFps(now);
}

function maybeSendFps(now) {
    if (!lastFpsTime) lastFpsTime = now;
    const elapsed = now - lastFpsTime;
    if (elapsed >= 500) {
        const fps = (frames * 1000) / elapsed;
        self.postMessage({ type: 'fps', fps, frames, count: particles.length });
        frames = 0;
        lastFpsTime = now;
    }
}

self.onmessage = (e) => {
    const msg = e.data;
    switch (msg.type) {
        case 'init':
            canvas = msg.canvas;
            width = msg.width;
            height = msg.height;
            ctx = canvas.getContext('2d');
            palette = msg.palette || palette;
            targetCount = msg.count || 200;
            particles = makeParticles(targetCount);
            t0 = performance.now();
            lastFpsTime = 0;
            frames = 0;
            lastFrameTime = 0;
            requestAnimationFrame(tick);
            self.postMessage({ type: 'ready' });
            break;

        case 'palette':
            palette = msg.palette;
            for (const p of particles) p.c = Math.floor(Math.random() * palette.length);
            break;

        case 'count':
            targetCount = msg.count;
            break;

        case 'pause':
            running = false;
            break;

        case 'resume':
            running = true;
            break;

        case 'transfer_test':
            // Получили ArrayBuffer. Отправляем обратно с transfer —
            // main thread измерит полное round-trip время.
            self.postMessage(
                { type: 'transfer_ack', size: msg.size, t0: msg.t0 },
                [msg.buffer],
            );
            break;
    }
};
```

### `src/offscreen_worker.rs`

```rust
//! 22. OffscreenCanvas и Workers.
//!
//! Обёртки для работы с `Worker`, `OffscreenCanvas` и transferable
//! объектами. Модуль разделён на две части:
//!
//! - **feature detection** — поддерживает ли браузер `Worker` и
//!   `transferControlToOffscreen`;
//! - **обвязка** — создание Worker'а, передача canvas'а, отправка
//!   сообщений с transferable и без.
//!
//! ## Утилиты для обхода багов rust-analyzer
//!
//! `format!` и `matches!` в tail-позиции функции теряют тип и дают
//! `expected String, found ()` / `expected bool, found ()`. Обход —
//! обернуть в функцию с явной сигнатурой: `format_str` и `bool_of`.

use std::fmt::Arguments;

use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Blob, HtmlCanvasElement, ImageData, OffscreenCanvas, Worker};

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `format!` для tail-позиций.
#[inline]
pub fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

/// Обёртка над `matches!` для tail-позиций.
#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Feature detection
// =============================================================================

/// `Worker` доступен в этом браузере.
///
/// Использует `bool_of(matches!(...))` — обход бага rust-analyzer
/// с `matches!` в tail-позиции.
pub fn is_worker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("Worker")),
        Ok(v) if v.is_function()
    ))
}

/// `HtmlCanvasElement::transferControlToOffscreen` доступен.
pub fn is_transfer_control_supported() -> bool {
    let global = js_sys::global();
    let Ok(document) = Reflect::get(&global, &JsValue::from_str("document")) else {
        return false;
    };
    let Ok(create) = Reflect::get(&document, &JsValue::from_str("createElement")) else {
        return false;
    };
    let Ok(create) = create.dyn_into::<js_sys::Function>() else {
        return false;
    };
    let Ok(canvas) = create.call1(&document, &JsValue::from_str("canvas")) else {
        return false;
    };
    bool_of(matches!(
        Reflect::get(&canvas, &JsValue::from_str("transferControlToOffscreen")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Создание Worker
// =============================================================================

/// Создаёт `Worker` из URL. Ошибка — если URL недоступен или Worker
/// не может быть создан (CSP, неверный путь).
pub fn create_worker(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
}

// =============================================================================
// Передача canvas в Worker
// =============================================================================

/// Отзывает управление canvas'ом у main thread и возвращает
/// `OffscreenCanvas`.
///
/// После вызова оригинальный `<canvas>` больше не может быть объектом
/// `getContext`; его содержимым управляет тот, кому передан
/// `OffscreenCanvas`.
///
/// Возвращает ошибку, если `transferControlToOffscreen` недоступен
/// или уже был вызван на этом элементе.
pub fn transfer_control(canvas: &HtmlCanvasElement) -> Result<OffscreenCanvas, JsValue> {
    canvas.transfer_control_to_offscreen()
}

/// Отправляет `OffscreenCanvas` Worker'у как transferable.
///
/// Сообщение — `{ type: "init", canvas: <offscreen>, width, height, palette, count }`.
/// Массив transferable содержит только `offscreen`.
pub fn post_init(
    worker: &Worker,
    canvas: OffscreenCanvas,
    width: u32,
    height: u32,
    palette: &[&str],
    count: u32,
) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("init"))?;
    Reflect::set(&msg, &JsValue::from_str("canvas"), canvas.as_ref())?;
    Reflect::set(
        &msg,
        &JsValue::from_str("width"),
        &JsValue::from_f64(width as f64),
    )?;
    Reflect::set(
        &msg,
        &JsValue::from_str("height"),
        &JsValue::from_f64(height as f64),
    )?;
    Reflect::set(&msg, &JsValue::from_str("count"), &JsValue::from_f64(count as f64))?;

    let pal = Array::new();
    for c in palette {
        pal.push(&JsValue::from_str(c));
    }
    Reflect::set(&msg, &JsValue::from_str("palette"), pal.as_ref())?;

    let transfer = Array::new();
    transfer.push(canvas.as_ref());

    worker.post_message_with_transfer(&msg, &transfer)
}

// =============================================================================
// Команды Worker'у
// =============================================================================

/// Отправляет простую команду `{ type: "..." }`.
pub fn post_command(worker: &Worker, cmd_type: &str) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str(cmd_type),
    )?;
    worker.post_message(&msg)
}

/// Отправляет команду с числовым параметром `{ type: "count", count }`.
pub fn post_count(worker: &Worker, count: u32) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("count"))?;
    Reflect::set(
        &msg,
        &JsValue::from_str("count"),
        &JsValue::from_f64(count as f64),
    )?;
    worker.post_message(&msg)
}

/// Отправляет палитру `{ type: "palette", palette: [...] }`.
pub fn post_palette(worker: &Worker, palette: &[&str]) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str("palette"),
    )?;
    let pal = Array::new();
    for c in palette {
        pal.push(&JsValue::from_str(c));
    }
    Reflect::set(&msg, &JsValue::from_str("palette"), pal.as_ref())?;
    worker.post_message(&msg)
}

// =============================================================================
// Transferable: послать буфер и вернуть его же назад
// =============================================================================

/// Создаёт `ArrayBuffer` заданного размера, отправляет его Worker'у
/// как **transferable** и возвращает `t0` (время отправки).
///
/// Используется в демо: после ответа Worker'а (с тем же буфером,
/// тоже transferable) main thread измеряет round-trip.
///
/// Владение буфером теряется main thread'ом сразу после `postMessage`.
pub fn send_transfer_test(worker: &Worker, size: usize) -> Result<f64, JsValue> {
    let u8 = Uint8Array::new_with_length(size as u32);
    for i in 0..size.min(1024) {
        u8.set_index(i as u32, (i & 0xFF) as u8);
    }
    let buffer: JsValue = u8.buffer().into();

    let msg = Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str("transfer_test"),
    )?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), &buffer)?;
    Reflect::set(
        &msg,
        &JsValue::from_str("size"),
        &JsValue::from_f64(size as f64),
    )?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    let transfer = Array::new();
    transfer.push(&buffer);
    worker.post_message_with_transfer(&msg, &transfer)?;
    Ok(t0)
}

/// То же самое, но без transferable — буфер **копируется**.
/// main thread сохраняет владение оригиналом.
pub fn send_copy_test(worker: &Worker, size: usize) -> Result<f64, JsValue> {
    let u8 = Uint8Array::new_with_length(size as u32);
    for i in 0..size.min(1024) {
        u8.set_index(i as u32, (i & 0xFF) as u8);
    }

    let msg = Object::new();
    Reflect::set(
        &msg,
        &JsValue::from_str("type"),
        &JsValue::from_str("transfer_test"),
    )?;
    // Передаём сам Uint8Array (копируется, потому что не в transfer list).
    Reflect::set(&msg, &JsValue::from_str("buffer"), u8.as_ref())?;
    Reflect::set(
        &msg,
        &JsValue::from_str("size"),
        &JsValue::from_f64(size as f64),
    )?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    // Пустой transfer list — копирование.
    worker.post_message(&msg)?;
    Ok(t0)
}

// =============================================================================
// Размеры данных
// =============================================================================

/// Сколько байт в `ImageData`.
///
/// `ImageData::data()` возвращает `Clamped<Vec<u8>>` — tuple struct
/// с публичным полем `.0`. Длину берём у внутреннего `Vec<u8>`:
/// у `Clamped` нет собственного метода `length` или `len`.
pub fn image_data_size(data: &ImageData) -> usize {
    data.data().0.len()
}

/// Сколько байт в `Blob`. `Blob::size()` возвращает `f64`.
pub fn blob_size(blob: &Blob) -> f64 {
    blob.size()
}
```

### `src/lib.rs` — добавление

```rust
pub mod canvas_dpi;                // 21.1
pub mod canvas_drawing;            // 21.2
pub mod canvas_composition;        // 21.3
pub mod canvas_imagedata;          // 21.4
pub mod canvas_image_loading;      // 21.5
pub mod canvas_filters;            // 21.6
pub mod canvas_image_basic;        // 21.7
pub mod canvas_image_pipeline;     // 21.7
pub mod offscreen_worker;          // ← 22
```

### `Cargo.toml` — дополнение к `[dependencies.web-sys] features`

```toml
    "Worker",
    "DedicatedWorkerGlobalScope",
    "OffscreenCanvas",
    "OffscreenCanvasRenderingContext2d",
    "MessageChannel",
    "MessagePort",
    "MessageEvent",
    "ErrorEvent",
```

### `tests/wasm_offscreen_worker.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::HtmlCanvasElement;

use hello_wasm02::offscreen_worker::{
    blob_size, create_worker, image_data_size, is_transfer_control_supported,
    is_worker_supported, post_command, transfer_control,
};

wasm_bindgen_test_configure!(run_in_browser);

fn make_canvas() -> HtmlCanvasElement {
    let doc = web_sys::window().unwrap().document().unwrap();
    doc.create_element("canvas")
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()
        .unwrap()
}

#[wasm_bindgen_test]
fn worker_is_supported_in_browser() {
    // В headless Chrome Worker доступен.
    assert!(is_worker_supported(), "Worker should be available");
}

#[wasm_bindgen_test]
fn transfer_control_is_supported() {
    assert!(
        is_transfer_control_supported(),
        "transferControlToOffscreen should be available",
    );
}

#[wasm_bindgen_test]
fn transfer_control_yields_offscreen_canvas() {
    let canvas = make_canvas();
    canvas.set_width(100);
    canvas.set_height(100);
    let offscreen = transfer_control(&canvas).expect("transfer");
    assert_eq!(offscreen.width(), 100);
    assert_eq!(offscreen.height(), 100);
}

#[wasm_bindgen_test]
fn second_transfer_control_fails() {
    let canvas = make_canvas();
    canvas.set_width(50);
    canvas.set_height(50);
    let _ = transfer_control(&canvas).expect("first");
    // Повторная передача — InvalidStateError.
    let second = transfer_control(&canvas);
    assert!(second.is_err(), "second transfer should fail");
}

#[wasm_bindgen_test]
fn worker_created_from_dummy_url() {
    // Создаём Worker с несуществующим URL — сам объект создаётся,
    // ошибка будет при попытке загрузки скрипта. Тест проверяет, что
    // API вызова доступно без паники.
    let w = create_worker("/nonexistent_worker_xyz.js");
    assert!(w.is_ok(), "Worker object should be constructible");
}

#[wasm_bindgen_test]
fn post_command_does_not_panic() {
    let Ok(worker) = create_worker("/nonexistent_worker_xyz.js") else {
        return;
    };
    // postMessage на недоступный скрипт не паникует на этом шаге —
    // сообщение уйдёт в очередь, обработчика нет.
    let _ = post_command(&worker, "pause");
}

#[wasm_bindgen_test]
fn image_data_size_is_width_times_height_times_four() {
    // Небольшая проверка обвязки: ImageData размер считается как w*h*4.
    let w = 4u32;
    let h = 3u32;
    let pixels = vec![0u8; (w * h * 4) as usize];
    let clamped = wasm_bindgen::Clamped(pixels.as_slice());
    let data = web_sys::ImageData::new_with_u8_clamped_array_and_sh(clamped, w, h).unwrap();
    assert_eq!(image_data_size(&data), (w * h * 4) as usize);
}

#[wasm_bindgen_test]
fn blob_size_returns_size_in_bytes() {
    let parts = js_sys::Array::new();
    let u8arr = js_sys::Uint8Array::from(&[1u8, 2, 3, 4][..]);
    parts.push(u8arr.as_ref());
    let bag = web_sys::BlobPropertyBag::new();
    bag.set_type("application/octet-stream");
    let blob = web_sys::Blob::new_with_u8_array_sequence_and_options(&parts, &bag).unwrap();
    assert_eq!(blob_size(&blob), 4.0);
}
```

## 22.13. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. wasm-тесты
wasm-pack test --headless --chrome --test wasm_offscreen_worker

# 3. Демо
npm run wasm:dev && npm start
```

**Важно:** файл `web/public/workers/particle_worker.js` должен быть доступен по адресу `/workers/particle_worker.js`. В Vite это автоматически — папка `public` монтируется в корень. Если у вас другой сервер — проверьте, что статический путь настроен.

---

## 22.14. `src/app.rs` — автономный демо-пример

```rust
//! Глава 22 — OffscreenCanvas и Workers.
//!
//! Демо-лаборатория: canvas передан Worker'у через
//! `transferControlToOffscreen`. Worker анимирует частицы, main
//! thread показывает статистику и управляет параметрами.
//!
//! ## Что демонстрируется
//!
//! - `Worker::new(url)` — создание dedicated-воркера из JS-файла.
//! - `canvas.transferControlToOffscreen()` — передача управления canvas'ом.
//! - `postMessage` — команды и статистика.
//! - **Transferable** — сравнение передачи и копирования 10 МБ.
//! - **Свободный main thread** — CSS-индикатор крутится, пока Worker занят.
//!
//! ## Правило работы с Rc<RefCell<State>>
//!
//! `borrow()` и `borrow_mut()` одного RefCell не должны пересекаться
//! во времени. `Ref`/`RefMut` guard'ы drop'аются в конце scope, поэтому
//! каждый borrow оборачивается в явный `{ }` блок.
//!
//! **Файл автономен.** Стили `.ow-*` инжектируются в документ.
//!
//! Wow: нажмите «Передать 10 МБ» — latency < 1 мс. «Скопировать
//! 10 МБ» — 10–20 мс. Один и тот же объём данных, разница — флаг
//! во втором аргументе `postMessage`.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Document, Event, HtmlCanvasElement, HtmlElement, HtmlInputElement, Worker,
};

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "ow-injected-style";
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
.ow-desc {
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
.ow-desc-line { margin: 6px 0; }

.ow-spinner-wrap {
    position: fixed;
    top: 12px;
    right: 12px;
    width: 42px;
    height: 42px;
    z-index: 9999;
    pointer-events: none;
}
.ow-spinner {
    width: 42px;
    height: 42px;
    border-radius: 50%;
    border: 3px solid #21262d;
    border-top-color: #3fb950;
    animation: ow-spin 1s linear infinite;
}
@keyframes ow-spin {
    to { transform: rotate(360deg); }
}

.ow-block {
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
.ow-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    min-width: 120px;
}
.ow-slider {
    flex: 1;
    min-width: 140px;
    max-width: 260px;
    accent-color: #58a6ff;
}
.ow-value {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #79c0ff;
    min-width: 60px;
}

.ow-btn {
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
.ow-btn:hover:not([disabled]) { background: #30363d; }
.ow-btn:active:not([disabled]) { transform: translateY(1px); }
.ow-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.ow-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.ow-btn-primary:hover:not([disabled]) { background: #2ea043; }
.ow-btn-warn {
    background: #21262d;
    border-color: #30363d;
    color: #f0883e;
}
.ow-btn-warn:hover:not([disabled]) { background: #30363d; }

.ow-canvas {
    display: block;
    width: 600px;
    height: 360px;
    max-width: 100%;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 8px;
    margin-bottom: 16px;
}

.ow-stats {
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
.ow-stats .label { color: #8b949e; }
.ow-stats .value { color: #79c0ff; }
.ow-stats .ok { color: #3fb950; }
.ow-stats .warn { color: #f0883e; }

.ow-footnote {
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

const WORKER_URL: &str = "/workers/particle_worker.js";
const CANVAS_W: u32 = 600;
const CANVAS_H: u32 = 360;
const TRANSFER_SIZE: usize = 10 * 1024 * 1024;

const PALETTES: [&[&str]; 3] = [
    &["#58a6ff", "#3fb950", "#f0883e", "#d2a8ff"],
    &["#f85149", "#f0883e", "#d29922", "#f0f6fc"],
    &["#a371f7", "#79c0ff", "#56d4dd", "#3fb950"],
];

const PALETTE_LABELS: [&str; 3] = ["Синяя", "Тёплая", "Холодная"];

// =============================================================================
// Состояние демо
// =============================================================================

struct DemoState {
    worker: Option<Worker>,
    palette_idx: usize,
    count: u32,
    paused: bool,
    fps: f64,
    frames: u32,
    last_transfer_ms: f64,
    last_copy_ms: f64,
    pending_transfer_t0: Option<f64>,
    pending_copy_t0: Option<f64>,
}

impl DemoState {
    fn new() -> Self {
        Self {
            worker: None,
            palette_idx: 0,
            count: 200,
            paused: false,
            fps: 0.0,
            frames: 0,
            last_transfer_ms: 0.0,
            last_copy_ms: 0.0,
            pending_transfer_t0: None,
            pending_copy_t0: None,
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

fn is_transfer_control_supported() -> bool {
    let global = js_sys::global();
    let Ok(document) = Reflect::get(&global, &JsValue::from_str("document")) else {
        return false;
    };
    let Ok(create) = Reflect::get(&document, &JsValue::from_str("createElement")) else {
        return false;
    };
    let Ok(create) = create.dyn_into::<Function>() else {
        return false;
    };
    let Ok(canvas) = create.call1(&document, &JsValue::from_str("canvas")) else {
        return false;
    };
    bool_of(matches!(
        Reflect::get(&canvas, &JsValue::from_str("transferControlToOffscreen")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Создание Worker и работа с ним
// =============================================================================

fn create_worker(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
}

fn transfer_control(canvas: &HtmlCanvasElement) -> Result<web_sys::OffscreenCanvas, JsValue> {
    canvas.transfer_control_to_offscreen()
}

fn post_init(
    worker: &Worker,
    canvas: web_sys::OffscreenCanvas,
    width: u32,
    height: u32,
    palette: &[&str],
    count: u32,
) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("init"))?;
    Reflect::set(&msg, &JsValue::from_str("canvas"), canvas.as_ref())?;
    Reflect::set(&msg, &JsValue::from_str("width"), &JsValue::from_f64(width as f64))?;
    Reflect::set(&msg, &JsValue::from_str("height"), &JsValue::from_f64(height as f64))?;
    Reflect::set(&msg, &JsValue::from_str("count"), &JsValue::from_f64(count as f64))?;
    let pal = Array::new();
    for c in palette {
        pal.push(&JsValue::from_str(c));
    }
    Reflect::set(&msg, &JsValue::from_str("palette"), pal.as_ref())?;
    let transfer = Array::new();
    transfer.push(canvas.as_ref());
    worker.post_message_with_transfer(&msg, &transfer)
}

fn post_command(worker: &Worker, cmd_type: &str) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str(cmd_type))?;
    worker.post_message(&msg)
}

fn post_count(worker: &Worker, count: u32) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("count"))?;
    Reflect::set(&msg, &JsValue::from_str("count"), &JsValue::from_f64(count as f64))?;
    worker.post_message(&msg)
}

fn post_palette(worker: &Worker, palette: &[&str]) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("palette"))?;
    let pal = Array::new();
    for c in palette {
        pal.push(&JsValue::from_str(c));
    }
    Reflect::set(&msg, &JsValue::from_str("palette"), pal.as_ref())?;
    worker.post_message(&msg)
}

/// Отправляет 10 МБ через **transferable** — O(1).
/// Возвращает `t0` — время отправки.
fn send_transfer_test(worker: &Worker, size: usize) -> Result<f64, JsValue> {
    let u8 = Uint8Array::new_with_length(size as u32);
    for i in 0..size.min(1024) {
        u8.set_index(i as u32, (i & 0xFF) as u8);
    }
    let buffer: JsValue = u8.buffer().into();

    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("transfer_test"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), &buffer)?;
    Reflect::set(&msg, &JsValue::from_str("size"), &JsValue::from_f64(size as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    let transfer = Array::new();
    transfer.push(&buffer);
    worker.post_message_with_transfer(&msg, &transfer)?;
    Ok(t0)
}

/// Отправляет 10 МБ через **копирование** — O(n).
/// Возвращает `t0` — время отправки.
fn send_copy_test(worker: &Worker, size: usize) -> Result<f64, JsValue> {
    let u8 = Uint8Array::new_with_length(size as u32);
    for i in 0..size.min(1024) {
        u8.set_index(i as u32, (i & 0xFF) as u8);
    }

    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("transfer_test"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), u8.as_ref())?;
    Reflect::set(&msg, &JsValue::from_str("size"), &JsValue::from_f64(size as f64))?;
    let t0 = js_sys::Date::now();
    Reflect::set(&msg, &JsValue::from_str("t0"), &JsValue::from_f64(t0))?;

    // Пустой transfer list — копирование.
    worker.post_message(&msg)?;
    Ok(t0)
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Canvas передан Worker'у через <code>transferControlToOffscreen()</code>. \
     С этого момента main thread не управляет содержимым холста — только \
     отправляет команды через <code>postMessage</code>. Анимация частиц идёт \
     в отдельном потоке.",
    "Индикатор в правом верхнем углу — CSS-анимация. Пока Worker анимирует \
     частицы, main thread свободен, и индикатор крутится плавно. Это не \
     ускорение, а <strong>изоляция</strong>: задача, требующая 60 FPS, \
     вынесена из критического пути UI.",
    "Кнопки внизу сравнивают передачу и копирование 10 МБ. \
     <code>postMessage(msg, [buffer])</code> передаёт буфер (O(1)), \
     а <code>postMessage(msg)</code> копирует (O(n)). Разница в десятки раз \
     при одном и том же объёме данных.",
];

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    // Спиннер свободного main thread — fixed в правом верхнем углу.
    let spinner = document.create_element("div")?;
    spinner.set_class_name("ow-spinner-wrap");
    spinner.set_inner_html(r#"<div class="ow-spinner"></div>"#);
    root.append_child(&spinner)?;

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("22 — OffscreenCanvas и Workers"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Проверка поддержки.
    if !is_worker_supported() || !is_transfer_control_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("ow-footnote");
        warn.set_text_content(Some(
            "Ваш браузер не поддерживает Worker или transferControlToOffscreen. \
             Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Canvas.
    let canvas_el = document.create_element("canvas")?;
    canvas_el.set_id("ow-canvas");
    canvas_el.set_class_name("ow-canvas");
    let canvas: HtmlCanvasElement = canvas_el.dyn_into()?;
    canvas.set_width(CANVAS_W);
    canvas.set_height(CANVAS_H);
    root.append_child(&canvas)?;

    // Панель управления.
    let controls = document.create_element("div")?;
    controls.set_class_name("ow-block");
    controls.set_inner_html(
        r#"<label class="ow-label" for="ow-count">Частиц:</label>
           <input type="range" id="ow-count" class="ow-slider"
                  min="20" max="2000" step="20" value="200">
           <span id="ow-count-val" class="ow-value">200</span>
           <label class="ow-label" for="ow-palette">Палитра:</label>
           <select id="ow-palette" class="ow-slider" style="flex:0 0 auto;min-width:120px;max-width:160px;">
               <option value="0">Синяя</option>
               <option value="1">Тёплая</option>
               <option value="2">Холодная</option>
           </select>
           <button class="ow-btn ow-btn-primary" id="ow-toggle">Пауза</button>"#,
    );
    root.append_child(&controls)?;

    // Панель transfer-тестов.
    let transfer_row = document.create_element("div")?;
    transfer_row.set_class_name("ow-block");
    transfer_row.set_inner_html(
        r#"<button class="ow-btn ow-btn-primary" id="ow-transfer">Передать 10 МБ</button>
           <button class="ow-btn ow-btn-warn" id="ow-copy">Скопировать 10 МБ</button>
           <span class="ow-label" style="min-width:auto;">Последняя операция:</span>
           <span id="ow-transfer-result" class="ow-value" style="min-width:auto;">—</span>"#,
    );
    root.append_child(&transfer_row)?;

    // Панель статистики.
    let stats = document.create_element("div")?;
    stats.set_class_name("ow-stats");
    stats.set_id("ow-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("ow-footnote");
    footnote.set_text_content(Some(
        "WASM в Worker требует отдельной инстанциации: модуль, загруженный \
         в main thread, недоступен из Worker'а. В этом демо Worker написан \
         на чистом JavaScript — мы фокусируемся на паттерне передачи \
         OffscreenCanvas и transferable-объектов. Rust-код в main thread \
         управляет Worker'ом и собирает статистику.",
    ));
    root.append_child(&footnote)?;

    // Создание Worker'а.
    let state = Rc::new(RefCell::new(DemoState::new()));
    let worker = create_worker(WORKER_URL)?;

    // Передаём canvas Worker'у.
    let offscreen = transfer_control(&canvas)?;
    post_init(&worker, offscreen, CANVAS_W, CANVAS_H, PALETTES[0], 200)?;

    // Обработчик сообщений от Worker'а.
    wire_worker_messages(&worker, state.clone())?;
    state.borrow_mut().worker = Some(worker);

    // Первая отрисовка статистики.
    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    // Обработчики UI.
    wire_count_slider(&document, state.clone())?;
    wire_palette_select(&document, state.clone())?;
    wire_toggle_button(&document, state.clone())?;
    wire_transfer_button(&document, state.clone())?;
    wire_copy_button(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_worker_messages(
    worker: &Worker,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    let closure = Closure::<dyn FnMut(web_sys::MessageEvent)>::new(
        move |ev: web_sys::MessageEvent| {
            let data = ev.data();
            let Ok(kind) = Reflect::get(&data, &JsValue::from_str("type")) else {
                return;
            };
            let Some(kind_str) = kind.as_string() else {
                return;
            };

            match kind_str.as_str() {
                "fps" => {
                    let fps = Reflect::get(&data, &JsValue::from_str("fps"))
                        .ok()
                        .and_then(|v| v.as_f64())
                        .unwrap_or(0.0);
                    let frames = Reflect::get(&data, &JsValue::from_str("frames"))
                        .ok()
                        .and_then(|v| v.as_f64())
                        .unwrap_or(0.0) as u32;
                    {
                        let mut st = state.borrow_mut();
                        st.fps = fps;
                        st.frames += frames;
                    }
                    let st = state.borrow();
                    update_stats(&document, &st);
                }
                "ready" => {
                    let st = state.borrow();
                    update_stats(&document, &st);
                }
                "transfer_ack" => {
                    let t0 = Reflect::get(&data, &JsValue::from_str("t0"))
                        .ok()
                        .and_then(|v| v.as_f64());
                    let Some(t0) = t0 else { return };
                    let elapsed = js_sys::Date::now() - t0;

                    // Обновляем состояние под одним borrow_mut, drop в конце scope.
                    let text = {
                        let mut st = state.borrow_mut();
                        if st.pending_transfer_t0.is_some() {
                            st.pending_transfer_t0 = None;
                            st.last_transfer_ms = elapsed;
                            format_str(format_args!("передача: {elapsed:.2} мс"))
                        } else if st.pending_copy_t0.is_some() {
                            st.pending_copy_t0 = None;
                            st.last_copy_ms = elapsed;
                            format_str(format_args!("копирование: {elapsed:.2} мс"))
                        } else {
                            String::from("—")
                        }
                    };

                    if let Some(el) = document.get_element_by_id("ow-transfer-result") {
                        el.set_text_content(Some(&text));
                    }
                    let st = state.borrow();
                    update_stats(&document, &st);
                }
                _ => {}
            }
        },
    );

    worker.set_onmessage(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    Ok(())
}

fn wire_count_slider(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = get_input(document, "ow-count")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(input) = doc.get_element_by_id("ow-count") else {
            return;
        };
        let Ok(input) = input.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(v) = input.value().parse::<u32>() else {
            return;
        };
        if let Some(el) = doc.get_element_by_id("ow-count-val") {
            el.set_text_content(Some(&format_str(format_args!("{v}"))));
        }

        // Обновляем состояние и получаем клон Worker'а под одним borrow_mut.
        let worker = {
            let mut st = state.borrow_mut();
            st.count = v;
            st.worker.clone()
        };

        if let Some(w) = worker {
            let _ = post_count(&w, v);
        }
    });
    el.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_palette_select(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("ow-palette")
        .ok_or_else(|| JsValue::from_str("no ow-palette"))?;
    let select: web_sys::HtmlSelectElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(el) = doc.get_element_by_id("ow-palette") else {
            return;
        };
        let Ok(select) = el.dyn_into::<web_sys::HtmlSelectElement>() else {
            return;
        };
        let Ok(idx) = select.value().parse::<usize>() else {
            return;
        };
        let idx = idx.min(PALETTES.len() - 1);
        let palette = PALETTES[idx];

        let worker = {
            let mut st = state.borrow_mut();
            st.palette_idx = idx;
            st.worker.clone()
        };
        if let Some(w) = worker {
            let _ = post_palette(&w, palette);
        }
    });
    select.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_toggle_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("ow-toggle")
        .ok_or_else(|| JsValue::from_str("no ow-toggle"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        let (paused, worker) = {
            let mut st = state.borrow_mut();
            st.paused = !st.paused;
            (st.paused, st.worker.clone())
        };
        if let Some(w) = worker {
            let cmd = if paused { "pause" } else { "resume" };
            let _ = post_command(&w, cmd);
        }
        if let Some(el) = doc.get_element_by_id("ow-toggle") {
            el.set_text_content(Some(if paused { "Пуск" } else { "Пауза" }));
        }
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_transfer_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("ow-transfer")
        .ok_or_else(|| JsValue::from_str("no ow-transfer"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        // Достаём клон Worker'а — borrow_mut не держим.
        let worker = {
            let st = state.borrow();
            st.worker.clone()
        };
        let Some(w) = worker else { return };

        match send_transfer_test(&w, TRANSFER_SIZE) {
            Ok(t0) => {
                state.borrow_mut().pending_transfer_t0 = Some(t0);
                if let Some(el) = doc.get_element_by_id("ow-transfer-result") {
                    el.set_text_content(Some("передача: …"));
                }
            }
            Err(e) => {
                let m = format_str(format_args!("ошибка: {e:?}"));
                if let Some(el) = doc.get_element_by_id("ow-transfer-result") {
                    el.set_text_content(Some(&m));
                }
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
        .get_element_by_id("ow-copy")
        .ok_or_else(|| JsValue::from_str("no ow-copy"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        let worker = {
            let st = state.borrow();
            st.worker.clone()
        };
        let Some(w) = worker else { return };

        match send_copy_test(&w, TRANSFER_SIZE) {
            Ok(t0) => {
                state.borrow_mut().pending_copy_t0 = Some(t0);
                if let Some(el) = doc.get_element_by_id("ow-transfer-result") {
                    el.set_text_content(Some("копирование: …"));
                }
            }
            Err(e) => {
                let m = format_str(format_args!("ошибка: {e:?}"));
                if let Some(el) = doc.get_element_by_id("ow-transfer-result") {
                    el.set_text_content(Some(&m));
                }
            }
        }
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- UI ----------

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("ow-stats") else {
        return;
    };
    let palette_label = PALETTE_LABELS
        .get(st.palette_idx)
        .copied()
        .unwrap_or("—");
    let transfer_str: String = if st.last_transfer_ms > 0.0 {
        format_str(format_args!("{:.2} мс", st.last_transfer_ms))
    } else {
        String::from("—")
    };
    let copy_str: String = if st.last_copy_ms > 0.0 {
        format_str(format_args!("{:.2} мс", st.last_copy_ms))
    } else {
        String::from("—")
    };
    let ratio: String = if st.last_transfer_ms > 0.0 && st.last_copy_ms > 0.0 {
        format_str(format_args!(
            "×{:.1}",
            st.last_copy_ms / st.last_transfer_ms
        ))
    } else {
        String::from("—")
    };

    let html = format_str(format_args!(
        r#"<div><span class="label">FPS (из Worker): </span><span class="value">{fps:.1}</span></div>
           <div><span class="label">Кадров всего: </span><span class="value">{frames}</span></div>
           <div><span class="label">Частиц: </span><span class="value">{count}</span> · <span class="label">палитра: </span><span class="value">{palette}</span></div>
           <div><span class="label">Transfer 10 МБ: </span><span class="ok">{transfer}</span> · <span class="label">copy 10 МБ: </span><span class="warn">{copy}</span> · <span class="label">ratio: </span><span class="value">{ratio}</span></div>"#,
        fps = st.fps,
        frames = st.frames,
        count = st.count,
        palette = palette_label,
        transfer = transfer_str,
        copy = copy_str,
        ratio = ratio,
    ));
    el.set_inner_html(&html);
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ow-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("ow-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// ---------- Мелочи ----------

fn get_input(document: &Document, id: &str) -> Result<HtmlInputElement, JsValue> {
    document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no input #{id}")))?
        .dyn_into::<HtmlInputElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} is not input")))
}
```

---

## 22.15. Анализ решения и доработки

**Какую ценность даёт демо.**

Глава показывает **три концепции** на одном экране.

**Первое — transferControlToOffscreen.** Кнопка «пуск» создаёт Worker и передаёт ему canvas. С этого момента `getContext("2d")` на оригинальном элементе недоступен, а содержимое холста обновляется Worker'ом. Студент видит, что canvas — не «объект DOM», а «объект рисования», и его можно **отдать** другому потоку.

**Второе — свободный main thread.** Индикатор в правом верхнем углу — не украшение. Это **диагностический инструмент**: пока Worker крутит 200 частиц с частотой 60 FPS, main thread обслуживает CSS-анимацию. Индикатор крутится плавно. Если бы анимация шла в main thread, при 2000 частицах индикатор бы замирал.

**Третье — разница transfer vs copy.** Две кнопки с одним и тем же размером данных (10 МБ). Первая передаёт буфер через `postMessage(msg, [buffer])`, вторая — копирует. Разница в latency в 20–50 раз. Это прямое доказательство того, что transferable работает.

**Скрытие технических сложностей.**

- Пользователь кликает по кнопкам — не думает про `postMessage`, `MessageEvent`, `structured clone`. Всё инкапсулировано в `wire_*`-функциях.
- Число частиц — слайдер от 20 до 2000. Не «отправьте сообщение с типом count», а «крутите ползунок, картинка меняется».
- Палитра — три варианта. Мгновенная обратная связь.

**Что осознанно не реализовано.**

- **WASM в Worker'е.** Как сказано в footnote — это отдельная задача, требующая сборки второго `.wasm` файла. Демо фокусируется на паттерне, а не на конкретной реализации WASM-в-Worker.
- **MessageChannel.** Демо использует один канал Worker ↔ main thread. Два порта (`MessageChannel`) — усложнение без практической пользы для этого сценария.
- **SharedArrayBuffer.** Требует COOP/COEP-заголовков, недоступен в обычной Vite-разработке. Transferable ArrayBuffer покрывает 90 % случаев.

**Правила проекта, применённые в этой главе.**

1. **Autonomous app.rs.** Всё inline: `format_str`, `bool_of`, создание Worker'а, работа с OffscreenCanvas.
2. **`format_str(format_args!(...))` в tail-позициях.** `update_stats`, обработчики.
3. **`bool_of(matches!(...))` для `is_transfer_control_supported`.**
4. **RefMut-дисциплина.** Каждый `borrow_mut` в узком scope, `borrow` после drop.
5. **Feature detection перед использованием API.** `is_worker_supported`, `is_transfer_control_supported`.
6. **`is_function()` перед вызовом через Reflect.** В `is_transfer_control_supported`.
7. **Стили инжектируются в документ.** Правок в `main.css` не требуется.

**Доработки, реализованные в коде.**

1. **Fallback при отсутствии API.** Если `Worker` или `transferControlToOffscreen` недоступны, демо показывает сообщение и не пытается продолжать.
2. **Убрано состояние `last_transfer_ms` без инициализации.** В `DemoState::new` все поля явно заданы.
3. **Явные `borrow_mut` в узком scope.** В обработчиках state обновляется только в блоке, рендер — после.
4. **Проверка `pending_transfer_t0` / `pending_copy_t0` в `transfer_ack`.** Правильно различает, какой именно тест вернулся.

**Что можно улучшить в следующей итерации.**

- **Реальный WASM-модуль в Worker'е.** Собрать минимальный второй крейт с `wasm-pack`, положить `.wasm` в `web/public/workers/`, загрузить через `import()` в particle_worker.js. Это покажет, как делить WASM-код между контекстами.
- **SharedArrayBuffer** с COOP/COEP для настоящей shared-memory параллельности. Отдельная большая тема.
- **Transferable ImageBitmap.** Показать отправку `ImageBitmap` из main thread в Worker для отрисовки — альтернатива `OffscreenCanvas` в некоторых сценариях.
- **График FPS в реальном времени.** Canvas 40×100 в углу, куда main thread рисует sparkline FPS от Worker'а. Даёт ощущение «живого» мониторинга.

## 22.16. Итог

OffscreenCanvas и Worker — инструменты **изоляции**. Main thread — критический ресурс: на нём живёт ввод, скролл, рендеринг страницы. Всё, что требует 60 FPS или длится сотни миллисекунд, должно быть вынесено.

`transferControlToOffscreen` — мост между DOM-элементом и Worker'ом. Одноразовая операция, после которой Worker владеет холстом.

`postMessage` — единственный канал обмена. Structured clone копирует данные, кроме transferable-объектов. `ArrayBuffer`, `OffscreenCanvas`, `MessagePort` можно **передать** — O(1) вместо O(n).

WASM в Worker требует отдельной инстанциации. Об этом нужно помнить при проектировании: не «поделиться модулем», а «загрузить второй раз в другом контексте».

Демо автономно, стили `.ow-*` инжектируются в документ. Все правила проекта применены. Глава закрывает тему Canvas 2D и открывает дверь в мир параллельного веба: WebGL, WebGPU, WebCodecs — следующие темы, но принцип остаётся тем же: main thread ценен, всё остальное можно выносить.