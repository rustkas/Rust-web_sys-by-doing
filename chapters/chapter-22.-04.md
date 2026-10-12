# Глава 22.4. WASM в Worker: Rust в параллель

## 22.4.0. Обновлённая структура

```
src/
  wasm_in_worker.rs               ← НОВЫЙ модуль 22.4
  transferable.rs                 ← из 22.3
  dedicated_worker.rs             ← из 22.2
  offscreen_canvas.rs             ← из 22.1
  offscreen_worker.rs             ← из главы 22
  canvas_dpi.rs … canvas_image_pipeline.rs
tests/
  wasm_wasm_in_worker.rs          ← НОВЫЙ тест
src/lib.rs                        ← +1 строка
src/app.rs                        ← демо 22.4 (заменяется целиком)
web/public/workers/
  ww_filter_worker.js             ← НОВЫЙ JS-воркер (module worker)
Cargo.toml                        ← без правок (WorkerOptions/WorkerType уже есть из 22.2)
web/styles/main.css               ← без правок
```

**Комментарий.** Модуль `wasm_in_worker.rs` — про **вторую инстанциацию** WASM-модуля в контексте Worker'а. Ключевая идея: crate `hello-wasm02` компилируется один раз, но загружается дважды — в main thread и в Worker'е. У каждой загрузки своя линейная память, свои `#[wasm_bindgen]`-функции, свой `#[wasm_bindgen(start)]`. Общее только `.wasm`-файл на диске. Префикс CSS — `ww-` (wasm worker), не пересекается с `tr-` (22.3), `dw-` (22.2), `oc-` (22.1), `ow-` (глава 22). Демо: box-blur фильтр, применяемый к изображению — сначала в main thread (UI замирает), потом в Worker'е (UI остаётся живым). Индикатор активности main thread'а — rAF-счётчик, обновляемый каждый кадр.

**Ссылки для детального изучения:**

- RustWASM Book: [Web Workers](https://rustwasm.github.io/docs/wasm-bindgen/examples/raytrace.html)
- wasm-bindgen Guide: [Loading wasm in a Worker](https://rustwasm.github.io/wasm-bindgen/reference/deployment.html)
- MDN: [Module Workers](https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker#type)
- MDN: [`Worker()` constructor](https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker)
- Vite docs: [Web Workers](https://vitejs.dev/guide/features#web-workers)
- web.dev: [WASM Workers](https://web.dev/articles/loading-webassembly-modules)

---

## 22.4.1. О чём эта подглава

В 22.3 мы разобрали передачу данных между потоками. В 22.1 — передачу canvas'а. В 22.2 — интерфейс Worker'а. Теперь возникает естественный вопрос: **где живёт Rust-код, если основная работа должна идти в Worker'е?**

Ответ не очевиден. WASM-модуль, инстанцированный в main thread, **не существует** в Worker'е. Каждый Worker — отдельный JavaScript-контекст со своим `WebAssembly.Instance`, своей линейной памятью, своим `wasm_bindgen`-runtime. Общий только `.wasm`-файл на диске.

Подглава отвечает на четыре вопроса.

**Первый — почему WASM не разделяется между контекстами.** Потому что `WebAssembly.Memory`, `WebAssembly.Table` и `WebAssembly.Global` — объекты JavaScript-контекста, в котором созданы. Worker имеет свой контекст — значит, свои объекты.

**Второй — как загрузить WASM в Worker.** Три подхода: отдельный `.wasm`-модуль, тот же `.wasm` через `import()`, `wasm-bindgen-rayon`. В подглаве используется **второй**: тот же pkg-модуль `hello_wasm02`, загруженный в контексте Worker'а через module worker + `import()`.

**Третий — что для этого нужно на JS-стороне.** Module worker (`{ type: "module" }`) — обязательное условие: `import()` работает только в модулях. Загрузка через `import init, { filter } from "/pkg/hello_wasm02.js"; await init();`.

**Четвёртый — как доказать, что это разные инстансы.** Уникальный ID, генерируемый один раз на инстанс. Main thread и Worker возвращают разные ID — визуально видно, что это две разные загрузки одного и того же `.wasm`.

---

## 22.4.2. Что уже есть и что появится

К концу 22.3 умеем:
- передавать `ArrayBuffer` между потоками через transferable;
- создавать `MessageChannel` и обмениваться по нему;
- отличать transferable от copyable.

К концу 22.4 будем уметь:
- загружать WASM-модуль в контексте Worker'а через module worker;
- вызывать Rust-функции из Worker'а;
- видеть, что main thread и Worker имеют **разные** инстансы WASM;
- применять фильтр к изображению в фоне, не блокируя UI;
- измерять разницу «фильтр в main thread» vs «фильтр в Worker'е».

---

## 22.4.3. Почему WASM не разделяется между контекстами

WASM-модуль — это **код**. Он статичен, его можно скомпилировать один раз и загрузить во множестве контекстов. `WebAssembly.compile(bytes)` возвращает `WebAssembly.Module`, который переиспользуется. Но **инстанс** — это код + память + таблицы + глобалы. Инстанс создаётся для конкретного контекста и живёт в нём.

Что это значит на практике:

- `wasm_bindgen::memory()` в main thread возвращает **один** `WebAssembly.Memory`.
- `wasm_bindgen::memory()` в Worker'е возвращает **другой** `WebAssembly.Memory`.
- Статики в Rust (например, `static COUNTER: AtomicU32`) у каждой инстанции — **свои**.
- `thread_local!` — тоже свои.

Это не ограничение wasm-bindgen, а фундаментальное свойство WASM. Именно поэтому `SharedArrayBuffer` (тема 22.5) — отдельная большая подглава: он позволяет **разделить** память между инстансами, но требует COOP/COEP-заголовков и `Atomics` для синхронизации.

**Практическое следствие.** Если Rust-функция `filter_rgba` определена в crate'е, она существует в **обеих** инстансах — в main thread и в Worker'е. Вызов из JS-стороны `import { filter_rgba } from "/pkg/hello_wasm02.js"` в main thread попадёт в main-инстанс, тот же вызов в Worker'е — в worker-инстанс. Данные между ними **не разделяются**.

---

## 22.4.4. Module worker: `{ type: "module" }`

Чтобы Worker мог использовать `import`, нужно создать его как **module worker**:

```js
const worker = new Worker("/workers/filter.js", { type: "module" });
```

**Что это меняет:**

- Внутри Worker'а работает `import` / `export`.
- Доступны `import.meta.url`, top-level `await`.
- `importScripts` (классический способ) — **не работает**.
- URL резолвится как ESM-модуль.

**Что остаётся прежним:**

- `self.postMessage`, `self.onmessage`, `self.close`.
- Отсутствие DOM.
- Structured clone для сообщений.

**Feature detection.** Module workers поддерживаются Chrome 80+, Firefox 114+, Safari 15+. Проверка — попытка создать Worker с `{ type: "module" }` и перехват исключения. В большинстве случаев `is_worker_supported()` достаточен, потому что современные браузеры поддерживают и то, и другое.

---

## 22.4.5. Загрузка wasm-pack-пакета в Worker'е

`wasm-pack build --target web` создаёт `pkg/hello_wasm02.js` — ESM-модуль с функцией `init()` и всеми `#[wasm_bindgen]`-экспортами. В main thread его подключают обычным `import`. В Worker'е — так же:

```js
// web/public/workers/ww_filter_worker.js
import init, { box_blur_rgba, instance_id } from "/pkg/hello_wasm02.js";

let ready = false;

init().then(() => {
    ready = true;
    self.postMessage({ type: "ready", instance: instance_id() });
}).catch((err) => {
    self.postMessage({ type: "init_error", message: String(err) });
});

self.onmessage = (e) => {
    if (!ready) return;
    const { buffer, width, height, passes } = e.data;
    const input = new Uint8Array(buffer);
    const t0 = performance.now();
    const output = box_blur_rgba(input, width, height, passes);
    const t1 = performance.now();
    const copy = new Uint8Array(output);
    self.postMessage({
        type: "result",
        buffer: copy.buffer,
        width, height,
        elapsed_ms: t1 - t0,
        instance_id: instance_id(),
    }, [copy.buffer]);
};
```

**Ключевые детали:**

1. `import init, { box_blur_rgba } from "/pkg/hello_wasm02.js"` — тот же модуль, что и в main thread. Загружается **заново** в контексте Worker'а. Внутри вызывается `WebAssembly.instantiate` — создаётся второй инстанс.

2. `await init()` — загружает и инстанцирует `.wasm`. Без этого `box_blur_rgba` выбросит ошибку.

3. `box_blur_rgba(input, w, h, passes)` — вызов Rust-функции. `input` — `Uint8Array`, был скопирован в wasm-память Worker-инстанса.

4. `new Uint8Array(output)` — **копирование** результата. `output` — это view над wasm-памятью Worker-инстанса (wasm-bindgen возвращает `Vec<u8>` как view). Трансферить view нельзя — только собственный `ArrayBuffer`. Копия создаёт свежий ArrayBuffer в JS-heap Worker'а, его уже можно трансферить.

5. `performance.now()` — доступно в Worker'е. Измерение честное.

---

## 22.4.6. Как доказать, что инстансы разные

Уникальный ID, генерируемый один раз на инстанс:

```rust
use std::cell::RefCell;

thread_local! {
    static INSTANCE_ID: RefCell<Option<String>> = RefCell::new(None);
}

#[wasm_bindgen]
pub fn instance_id() -> String {
    INSTANCE_ID.with(|cell| {
        let mut slot = cell.borrow_mut();
        if slot.is_none() {
            let n = (js_sys::Math::random() * 0x1_0000_0000u64 as f64) as u32;
            *slot = Some(format_str(format_args!("wasm-{n:08x}")));
        }
        slot.as_deref().unwrap_or("").to_string()
    })
}
```

`thread_local!` в WASM — глобальная переменная конкретного инстанса. В main thread `INSTANCE_ID` содержит одно значение, в Worker'е — другое. Main thread вызывает `instance_id()` — получает `wasm-a1b2c3d4`. Worker вызывает — получает `wasm-5e6f7a8b`. Разные ID — доказательство разных инстансов.

**Почему не подходит `static AtomicU32`?** В wasm-bindgen статики — тоже per-instance. Но `AtomicU32` не отличит main-thread-инстанс от worker-инстанса: оба стартуют с одного значения. Случайный ID решает эту проблему.

---

## 22.4.7. Service Worker и module worker: конфликт

Отдельная техническая деталь, которая часто ломает демо при первой загрузке. Module worker делает `import "/pkg/hello_wasm02.js"`. Этот запрос — **обычный fetch** внутри Worker'а. И он **проходит через все активные Service Worker'ы** origin'а.

Если на `localhost:5173` уже зарегистрирован SW от предыдущего демо (кэширование, offline, работа с сетью) — он перехватит запрос к `/pkg/hello_wasm02.js`. Если fetch-обработчик SW не знает про `/pkg/`, он падает — Chrome показывает:

```
Failed to load 'http://127.0.0.1:5173/pkg/hello_wasm02.js'.
A ServiceWorker intercepted the request and encountered an unexpected error.
```

**Это не проблема 22.4.** Это проблема сосуществования демо на одном origin'е. Но решать её надо здесь, потому что именно здесь module worker впервые делает `import` внутреннего pkg-пути.

**Решение — self-healing cleanup.** Перед созданием Worker'а демо проверяет `navigator.serviceWorker.controller`. Если он есть:

1. Показывает пользователю нотис: «Очистка старых Service Worker'ов…»
2. Вызывает `navigator.serviceWorker.getRegistrations()` → `unregister()` для всех.
3. Ставит флаг в `sessionStorage` (защита от цикла).
4. Делает `location.reload()`.

На второй загрузке SW уже нет — демо работает штатно. Если по какой-то причине SW всё ещё активен — второй раз не пытаемся (флаг в sessionStorage).

**Почему не «просто закомментировать регистрацию в main.js»?** Потому что старые регистрации уже живут в браузере пользователя. `unregister()` — единственный способ их убрать. Само демо 22.4 SW не регистрирует, но обязано снять чужие.

---

## 22.4.8. Что показывает демо

**Layout:**

- **Canvas** — 480×360. Показывает сгенерированный тестовый паттерн (концентрические окружности, градиент).
- **Индикатор main thread'а** — счётчик кадров, обновляемый через `requestAnimationFrame`. Пока main thread свободен — число растёт. Если занят — замирает.
- **Кнопки:**
  - «Redraw» — перерисовать исходный паттерн.
  - «Filter in main thread» — применить box-blur в main thread.
  - «Filter in Worker» — применить box-blur в Worker'е.
- **Статистика:**
  - ID main-инстанса WASM.
  - ID worker-инстанса WASM (обновляется после первого ответа).
  - Время последней операции.
  - Кадров main thread'а за время операции.

**Сценарий:**

1. При загрузке: canvas показывает паттерн, main-id = `wasm-a1b2c3d4`. Счётчик кадров растёт.
2. Клик «Filter in main thread». Main thread занят на 300–500 мс. Счётчик кадров **замирает** — UI заблокирован. Canvas обновляется в конце.
3. Клик «Redraw» — паттерн восстанавливается.
4. Клик «Filter in Worker». Main thread свободен. Счётчик кадров растёт **во время** работы Worker'а. Canvas обновляется, когда Worker присылает результат.
5. После первого ответа Worker'а появляется его instance-id: `wasm-5e6f7a8b`. Он **отличается** от main-id.

**Почему это наглядно.**

- **Первый факт.** Один и тот же фильтр, один и тот же Rust-код. Разница только в том, **где** он выполняется.
- **Второй факт.** Разные instance-id — визуальное доказательство второй инстанциации.
- **Третий факт.** Счётчик кадров — прямой индикатор свободы main thread'а. 500 кадров vs 60 кадров за одно и то же время.

---

## 22.4.9. Wow-момент

**Первый.** Клик «Filter in main thread». Счётчик кадров мгновенно замирает на 300–500 мс. Никаких визуальных изменений — браузер полностью занят вычислениями. Это цена работы в main thread'е.

**Второй.** Клик «Filter in Worker». Счётчик кадров продолжает расти. Canvas обновляется, когда Worker закончил. Разница видна глазом.

**Третий.** Посмотрите на два instance-id. `wasm-a1b2c3d4` и `wasm-5e6f7a8b`. Это два разных `WebAssembly.Instance`, каждый со своей линейной памятью, своими `thread_local!`. Общий только `.wasm`-файл.

**Четвёртый.** Откройте DevTools → Sources → Workers. Увидите `ww_filter_worker.js` как отдельный поток. DevTools → Memory: два `WebAssembly.Memory` — по одному на инстанс.

---

## 22.4.10. Чему это учит

**Первое.** WASM-модуль компилируется один раз, но **инстанцируется** в каждом JS-контексте отдельно.

**Второе.** Module workers (`{ type: "module" }`) — обязательное условие для `import()` в Worker'е.

**Третье.** Загрузка `pkg/hello_wasm02.js` в Worker'е — стандартный паттерн: `import init, { fn } from "..."; await init();`. Никаких костылей.

**Четвёртое.** Инстансы WASM **не разделяют** память. `thread_local!` в Rust работает per-instance.

**Пятое.** `Vec<u8>` из wasm-bindgen — это view над wasm-памятью. Для transfer нужно копировать в свежий `ArrayBuffer`.

**Шестое.** Worker-версия не быстрее. Она **не блокирует** main thread. Это разные критерии.

**Седьмое.** Если на origin'е есть активный Service Worker, он перехватывает `import`-запросы module worker'а. Снимайте регистрацию чужих SW перед созданием Worker'а.

---

## 22.4.11. Исходные файлы

### `src/wasm_in_worker.rs`

```rust
//! 22.4. WASM в Worker: Rust в параллель.
//!
//! Модуль решает две задачи:
//!
//! 1. **Экспорт фильтра** — `#[wasm_bindgen] box_blur_rgba` доступен
//!    и из main thread (как обычная Rust-функция), и из Worker'а
//!    (через `import ... from "/pkg/hello_wasm02.js"`).
//! 2. **Экспорт instance_id** — уникальный идентификатор на инстанс
//!    WASM. Показывает, что main thread и Worker имеют разные
//!    инстансы одного и того же `.wasm`.
//!
//! ## Почему `#[wasm_bindgen]` на фильтре
//!
//! Worker написан на JavaScript. Чтобы вызвать Rust-функцию из
//! Worker'а, она должна быть экспортирована в JS-модуль. `#[wasm_bindgen]`
//! делает именно это — добавляет обёртку в `pkg/hello_wasm02.js`.
//!
//! ## Утилита `bool_of`
//!
//! rust-analyzer теряет тип `matches!` в tail-позиции функции.
//! Обёртка `bool_of` возвращает типу конкретность.

use std::cell::RefCell;
use std::fmt::Arguments;

use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{MessageEvent, Worker, WorkerOptions, WorkerType};

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
// Feature detection
// =============================================================================

/// `Worker` доступен в этом браузере.
pub fn is_worker_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("Worker")),
        Ok(v) if v.is_function()
    ))
}

/// Module workers (`{ type: "module" }`) поддерживаются.
///
/// В современных браузерах (Chrome 80+, Firefox 114+, Safari 15+)
/// module workers — часть стандарта. Прямой feature detection без
/// создания Worker'а невозможен, поэтому проверяем косвенно.
pub fn is_module_worker_supported() -> bool {
    is_worker_supported()
}

/// `ImageBitmap` доступен — на случай, если демо переключат на
/// ImageBitmap-транспорт.
pub fn is_image_bitmap_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("ImageBitmap")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Instance ID: уникален для каждого инстанса WASM
// =============================================================================

thread_local! {
    static INSTANCE_ID: RefCell<Option<String>> = RefCell::new(None);
}

/// Уникальный идентификатор инстанса WASM.
///
/// Генерируется один раз при первом вызове. Разные инстансы
/// (main thread, каждый Worker) получают разные ID — доказательство,
/// что WASM загружен заново в каждом контексте.
#[wasm_bindgen]
pub fn instance_id() -> String {
    INSTANCE_ID.with(|cell| {
        let mut slot = cell.borrow_mut();
        if slot.is_none() {
            let n = (js_sys::Math::random() * 0x1_0000_0000u64 as f64) as u32;
            *slot = Some(format_str(format_args!("wasm-{n:08x}")));
        }
        slot.as_deref().unwrap_or("wasm-unknown").to_string()
    })
}

// =============================================================================
// Фильтр: box blur, применяется `passes` раз
// =============================================================================

/// Box blur 3×3, применённый `passes` раз к RGBA-изображению.
///
/// Вход: `data` — RGBA-байты (`w * h * 4`), `w`, `h` — размеры.
/// Выход: новые RGBA-байты того же размера.
///
/// Экспортируется в JS через `#[wasm_bindgen]` — вызывается
/// и из main thread'а (напрямую как Rust-функция), и из Worker'а
/// (через `import { box_blur_rgba } from "/pkg/hello_wasm02.js"`).
#[wasm_bindgen]
pub fn box_blur_rgba(data: &[u8], w: u32, h: u32, passes: u32) -> Vec<u8> {
    let ww = w as i32;
    let hh = h as i32;
    let len = data.len();
    let mut src: Vec<u8> = data.to_vec();
    let mut dst: Vec<u8> = vec![0u8; len];

    for _ in 0..passes {
        for y in 0..hh {
            for x in 0..ww {
                let mut r: u32 = 0;
                let mut g: u32 = 0;
                let mut b: u32 = 0;
                let mut n: u32 = 0;
                for dy in -1i32..=1 {
                    for dx in -1i32..=1 {
                        let nx = x + dx;
                        let ny = y + dy;
                        if nx < 0 || nx >= ww || ny < 0 || ny >= hh {
                            continue;
                        }
                        let idx = ((ny * ww + nx) * 4) as usize;
                        r += src[idx] as u32;
                        g += src[idx + 1] as u32;
                        b += src[idx + 2] as u32;
                        n += 1;
                    }
                }
                let di = ((y * ww + x) * 4) as usize;
                dst[di] = (r / n) as u8;
                dst[di + 1] = (g / n) as u8;
                dst[di + 2] = (b / n) as u8;
                dst[di + 3] = src[di + 3];
            }
        }
        std::mem::swap(&mut src, &mut dst);
    }
    src
}

// =============================================================================
// Создание module worker
// =============================================================================

/// Создаёт Worker как ESM-модуль. Обязательно для `import()` внутри
/// Worker'а.
pub fn create_module_worker(url: &str) -> Result<Worker, JsValue> {
    let opts = WorkerOptions::new();
    opts.set_type(WorkerType::Module);
    Worker::new_with_options(url, &opts)
}

// =============================================================================
// Обмен с Worker'ом
// =============================================================================

/// Отправляет изображение Worker'у: `{ type, buffer, width, height, passes }`.
///
/// `buffer` создаётся в JS-heap (Uint8Array::new_with_length), поэтому
/// его ArrayBuffer transferable.
pub fn post_image_to_worker(
    worker: &Worker,
    data: &[u8],
    w: u32,
    h: u32,
    passes: u32,
) -> Result<(), JsValue> {
    let arr = Uint8Array::new_with_length(data.len() as u32);
    arr.copy_from(data);
    let buffer: JsValue = arr.buffer().into();

    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("filter"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), &buffer)?;
    Reflect::set(&msg, &JsValue::from_str("width"), &JsValue::from_f64(w as f64))?;
    Reflect::set(&msg, &JsValue::from_str("height"), &JsValue::from_f64(h as f64))?;
    Reflect::set(&msg, &JsValue::from_str("passes"), &JsValue::from_f64(passes as f64))?;

    let transfer = Array::new();
    transfer.push(&buffer);
    worker.post_message_with_transfer(&msg, &transfer)
}

/// Результат, извлечённый из `MessageEvent` от Worker'а.
pub struct FilterResult {
    pub bytes: Vec<u8>,
    pub width: u32,
    pub height: u32,
    pub elapsed_ms: f64,
    pub instance_id: String,
}

/// Извлекает результат фильтрации из `MessageEvent`.
///
/// Ожидает `{ type: "result", buffer, width, height, elapsed_ms, instance_id }`.
/// Возвращает `None`, если структура не соответствует.
pub fn extract_filter_result(event: &MessageEvent) -> Option<FilterResult> {
    let data = event.data();

    let kind = Reflect::get(&data, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())?;
    if kind != "result" {
        return None;
    }

    let buffer_val = Reflect::get(&data, &JsValue::from_str("buffer")).ok()?;
    let arr: Uint8Array = buffer_val.dyn_into().ok()?;
    let bytes: Vec<u8> = arr.to_vec();

    let width = Reflect::get(&data, &JsValue::from_str("width"))
        .ok()?
        .as_f64()? as u32;
    let height = Reflect::get(&data, &JsValue::from_str("height"))
        .ok()?
        .as_f64()? as u32;
    let elapsed_ms = Reflect::get(&data, &JsValue::from_str("elapsed_ms"))
        .ok()?
        .as_f64()?;
    let instance_id = Reflect::get(&data, &JsValue::from_str("instance_id"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| String::from("wasm-unknown"));

    Some(FilterResult {
        bytes,
        width,
        height,
        elapsed_ms,
        instance_id,
    })
}

/// Извлекает ID инстанса из сообщения `{ type: "ready", instance }`.
pub fn extract_ready_instance(event: &MessageEvent) -> Option<String> {
    let data = event.data();
    let kind = Reflect::get(&data, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())?;
    if kind != "ready" {
        return None;
    }
    Reflect::get(&data, &JsValue::from_str("instance"))
        .ok()
        .and_then(|v| v.as_string())
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `transferable`:

```rust
pub mod offscreen_worker;     // 22
pub mod offscreen_canvas;     // 22.1
pub mod dedicated_worker;     // 22.2
pub mod transferable;         // 22.3
pub mod wasm_in_worker;       // ← 22.4
```

**Важно:** без этой строки `wasm_in_worker.rs` не компилируется в составе крейта, а `tests/wasm_wasm_in_worker.rs` падает с `unresolved import hello_wasm02::wasm_in_worker`.

### `Cargo.toml` — фичи

В `[dependencies.web-sys] features` должны присутствовать:

```toml
    "Worker",
    "WorkerOptions",
    "WorkerType",
    "MessageEvent",
```

`WorkerOptions` и `WorkerType` нужны для `create_module_worker`.

### `web/public/workers/ww_filter_worker.js`

```javascript
// 22.4. Filter worker — module worker.
//
// Загружает тот же wasm-pack-пакет, что и main thread, но во втором
// JS-контексте. Это создаёт ВТОРУЮ инстанциацию WASM-модуля:
// своя линейная память, свой WebAssembly.Instance, свой thread_local!.
//
// Ключевое: { type: "module" } при создании — обязательное условие
// для import(). Без него этот файл не распарсится.

import init, { box_blur_rgba, instance_id } from "/pkg/hello_wasm02.js";

let ready = false;

init()
    .then(() => {
        ready = true;
        // Сообщаем main thread свой ID — доказательство второй инстанциации.
        self.postMessage({ type: "ready", instance: instance_id() });
    })
    .catch((err) => {
        self.postMessage({ type: "init_error", message: String(err) });
    });

self.onmessage = (e) => {
    if (!ready) return;
    const msg = e.data;
    if (msg.type !== "filter") return;

    const input = new Uint8Array(msg.buffer);
    const t0 = performance.now();
    const output = box_blur_rgba(input, msg.width, msg.height, msg.passes);
    const t1 = performance.now();

    // `output` — view над wasm-памятью Worker-инстанса. Трансферить
    // нельзя — копируем в свежий JS-heap ArrayBuffer.
    const copy = new Uint8Array(output);

    self.postMessage(
        {
            type: "result",
            buffer: copy.buffer,
            width: msg.width,
            height: msg.height,
            elapsed_ms: t1 - t0,
            instance_id: instance_id(),
        },
        [copy.buffer],
    );
};
```

### `tests/wasm_wasm_in_worker.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::wasm_in_worker::{
    box_blur_rgba, instance_id, is_image_bitmap_supported,
    is_module_worker_supported, is_worker_supported,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn worker_is_supported_in_browser() {
    assert!(is_worker_supported(), "Worker should be available");
}

#[wasm_bindgen_test]
fn module_worker_is_supported() {
    assert!(
        is_module_worker_supported(),
        "module workers should be available",
    );
}

#[wasm_bindgen_test]
fn image_bitmap_is_supported() {
    assert!(
        is_image_bitmap_supported(),
        "ImageBitmap should be available",
    );
}

#[wasm_bindgen_test]
fn instance_id_is_stable_per_instance() {
    let a = instance_id();
    let b = instance_id();
    assert_eq!(a, b, "instance_id must be stable");
    assert!(a.starts_with("wasm-"), "instance_id has prefix");
    assert_eq!(a.len(), 13, "wasm-XXXXXXXX — 13 chars");
}

#[wasm_bindgen_test]
fn box_blur_preserves_size() {
    let w = 8u32;
    let h = 8u32;
    let data = vec![100u8; (w * h * 4) as usize];
    let out = box_blur_rgba(&data, w, h, 1);
    assert_eq!(out.len(), data.len());
}

#[wasm_bindgen_test]
fn box_blur_of_uniform_is_uniform() {
    // Если все пиксели одинаковы, blur не меняет результат.
    let w = 4u32;
    let h = 4u32;
    let data = vec![128u8; (w * h * 4) as usize];
    let out = box_blur_rgba(&data, w, h, 3);
    assert!(out.iter().all(|&b| b == 128));
}

#[wasm_bindgen_test]
fn box_blur_smooths_an_edge() {
    // Создаём резкую границу: левая половина 0, правая 255.
    let w = 8u32;
    let h = 4u32;
    let mut data = vec![0u8; (w * h * 4) as usize];
    for y in 0..h {
        for x in (w / 2)..w {
            let idx = ((y * w + x) * 4) as usize;
            data[idx] = 255;
        }
    }
    let out = box_blur_rgba(&data, w, h, 1);
    // На границе (x = w/2 - 1 → x = w/2) значение должно быть промежуточным.
    let left_edge_idx = ((0 * w + (w / 2 - 1)) * 4) as usize;
    let right_edge_idx = ((0 * w + (w / 2)) * 4) as usize;
    assert!(out[left_edge_idx] > 0, "left edge receives some value");
    assert!(out[right_edge_idx] < 255, "right edge loses pure value");
}

#[wasm_bindgen_test]
fn box_blur_passes_zero_returns_input() {
    let w = 4u32;
    let h = 4u32;
    let mut data = vec![0u8; (w * h * 4) as usize];
    data[0] = 100;
    data[5] = 200;
    let out = box_blur_rgba(&data, w, h, 0);
    assert_eq!(out, data);
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Wasm-тесты только для этой подглавы
wasm-pack test --headless --chrome --test wasm_wasm_in_worker

# 3. Демо
npm run wasm:dev && npm start
```

**Важно:** файл `web/public/workers/ww_filter_worker.js` импортирует `/pkg/hello_wasm02.js`. Убедитесь, что Vite отдаёт `pkg/` по этому пути. Если `root` Vite-конфига — `web/`, добавьте `pkg` в `server.fs.allow` или скопируйте `pkg` в `web/public/pkg/`. В дев-режиме Vite по умолчанию отдаёт корень проекта, если `publicDir` настроен правильно — проверьте доступность `/pkg/hello_wasm02.js` в браузере перед запуском демо.

---

## 22.4.12. `src/app.rs` — автономный демо-пример

```rust
//! Глава 22.4 — WASM в Worker: Rust в параллель.
//!
//! Демо-лаборатория: применяет box-blur к сгенерированному паттерну
//! двумя способами — в main thread'е и в Worker'е. Сравнивает
//! «заблокированность» main thread'а через счётчик кадров.
//!
//! ## Что демонстрируется
//!
//! - **Вторая инстанциация WASM.** Worker загружает тот же
//!   `/pkg/hello_wasm02.js` в своём контексте — разные `instance_id`.
//! - **Module worker** (`{ type: "module" }`) — обязательно для
//!   `import()` внутри Worker'а.
//! - **Transferable-байты.** Буфер RGBA передаётся туда и обратно
//!   без копирования.
//! - **Свобода main thread'а.** Счётчик кадров растёт во время
//!   работы Worker'а и замирает во время main-thread-фильтра.
//!
//! ## Правило Rc<RefCell<State>>
//!
//! `borrow()` и `borrow_mut()` не пересекаются. Каждый borrow —
//! в узком scope, `Ref`/`RefMut` drop'аются до следующего borrow.
//!
//! ## Обход бага rust-analyzer с сигнатурами canvas-методов
//!
//! r-a ошибочно считает, что `fill_rect`, `move_to`, `line_to`,
//! `get_image_data`, `put_image_data` принимают `i32`. Обходим через
//! `Reflect::get` + `Function::apply` — метод вызывается динамически,
//! без проверки сигнатуры.
//!
//! ## Внутренние ссылки
//!
//! `app.rs` — модуль внутри крейта `hello_wasm02`, поэтому ссылки
//! на другие модули идут через `crate::`, а не по имени пакета.
//!
//! ## Service Worker и module worker
//!
//! Module worker делает `import "/pkg/hello_wasm02.js"`. Этот запрос
//! проходит через все активные Service Worker'ы. SW из предыдущих
//! демо (кэширование, offline) не знает `/pkg/` и ломает загрузку:
//! «A ServiceWorker intercepted the request and encountered an
//! unexpected error». Демо снимает регистрацию SW и один раз
//! перезагружает страницу.
//!
//! **Файл автономен.** Стили `.ww-*` инжектируются в документ.
//! Wow: «Filter in Worker» — счётчик кадров продолжает расти;
//! «Filter in main thread» — замирает на 300–500 мс.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement, ImageData,
    MessageEvent, Worker,
};

use crate::wasm_in_worker;

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
// Service Worker: снятие регистрации
// =============================================================================

/// Активен ли Service Worker, контролирующий текущую страницу.
///
/// Читаем через Reflect — не зависим от версии web-sys.
fn has_sw_controller() -> bool {
    let global = js_sys::global();
    let Ok(nav) = Reflect::get(&global, &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(sw) = Reflect::get(&nav, &JsValue::from_str("serviceWorker")) else {
        return false;
    };
    if sw.is_undefined() || sw.is_null() {
        return false;
    }
    let Ok(ctrl) = Reflect::get(&sw, &JsValue::from_str("controller")) else {
        return false;
    };
    !ctrl.is_null() && !ctrl.is_undefined()
}

/// Снимает регистрацию всех Service Worker'ов и перезагружает страницу.
///
/// После `unregister()` текущий документ **всё ещё управляется** SW до
/// следующей навигации. Поэтому без `location.reload()` следующий fetch
/// (за `/pkg/hello_wasm02.js`) снова пройдёт через SW.
///
/// Защита от цикла: `sessionStorage`-флаг. Если после reload'а SW
/// всё ещё активен — не пытаемся второй раз, а просто выходим.
fn unregister_sw_and_reload(document: &Document) -> Result<(), JsValue> {
    let script = document.create_element("script")?;
    script.set_text_content(Some(
        r#"(function(){
            var KEY = 'ww-sw-cleared-22-4';
            if (sessionStorage.getItem(KEY) === '1') return;
            sessionStorage.setItem(KEY, '1');
            if (!('serviceWorker' in navigator)) {
                location.reload();
                return;
            }
            navigator.serviceWorker.getRegistrations().then(function(rs){
                var ps = [];
                for (var i = 0; i < rs.length; i++) {
                    ps.push(rs[i].unregister());
                }
                return Promise.all(ps);
            }).then(function(){
                location.reload();
            }).catch(function(){
                location.reload();
            });
        })();"#,
    ));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&script)?;
    Ok(())
}

/// Показывает пользователю, что идёт очистка от SW.
fn show_sw_notice(document: &Document) -> Result<(), JsValue> {
    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html(
        r#"<div style="max-width:900px;padding:20px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#c9d1d9;font-family:system-ui,sans-serif;">
            <h2 style="margin:0 0 10px 0;color:#f0883e;">Очистка старых Service Worker'ов…</h2>
            <p style="margin:0;line-height:1.55;">
                Обнаружен активный Service Worker от предыдущего демо. Он
                перехватывает запросы к <code>/pkg/hello_wasm02.js</code> и
                ломает загрузку module worker'а. Снимаем регистрацию и
                перезагружаем страницу — это займёт меньше секунды.
            </p>
        </div>"#,
    );
    Ok(())
}

// =============================================================================
// Хелперы для canvas: вызов через Reflect
// =============================================================================

/// Вызывает метод canvas-контекста с аргументами `f64`.
fn ctx_call(ctx: &JsValue, method: &str, args: &[f64]) {
    let Ok(f) = Reflect::get(ctx, &JsValue::from_str(method)) else {
        return;
    };
    let Ok(f) = f.dyn_into::<Function>() else {
        return;
    };
    let arr = Array::new();
    for a in args {
        arr.push(&JsValue::from_f64(*a));
    }
    let _ = f.apply(ctx, &arr);
}

/// Вызывает метод canvas-контекста с произвольными JsValue-аргументами.
fn ctx_call_js(ctx: &JsValue, method: &str, args: &[JsValue]) -> Option<JsValue> {
    let f = Reflect::get(ctx, &JsValue::from_str(method)).ok()?;
    let f = f.dyn_into::<Function>().ok()?;
    let arr = Array::new();
    for a in args {
        arr.push(a);
    }
    f.apply(ctx, &arr).ok()
}

/// Устанавливает свойство canvas-контекста.
fn ctx_set(ctx: &JsValue, key: &str, value: &JsValue) {
    let _ = Reflect::set(ctx, &JsValue::from_str(key), value);
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "ww-injected-style";
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
.ww-desc {
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
.ww-desc-line { margin: 6px 0; }

.ww-canvas {
    display: block;
    width: 480px;
    height: 360px;
    max-width: 100%;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 8px;
    margin-bottom: 16px;
}

.ww-block {
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

.ww-btn {
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
.ww-btn:hover:not([disabled]) { background: #30363d; }
.ww-btn:active:not([disabled]) { transform: translateY(1px); }
.ww-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.ww-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.ww-btn-primary:hover:not([disabled]) { background: #2ea043; }
.ww-btn-warn {
    background: #21262d;
    border-color: #30363d;
    color: #f0883e;
}
.ww-btn-warn:hover:not([disabled]) { background: #30363d; }

.ww-stats {
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
.ww-stats .label { color: #8b949e; }
.ww-stats .value { color: #79c0ff; }
.ww-stats .ok { color: #3fb950; }
.ww-stats .warn { color: #f0883e; }
.ww-stats .err { color: #f85149; }

.ww-footnote {
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

const WORKER_URL: &str = "/workers/ww_filter_worker.js";
const CANVAS_W: u32 = 480;
const CANVAS_H: u32 = 360;
const BLUR_PASSES: u32 = 40;

// =============================================================================
// Состояние демо
// =============================================================================

struct DemoState {
    worker: Option<Worker>,
    worker_ready: bool,
    main_instance_id: String,
    worker_instance_id: String,
    last_main_ms: f64,
    last_worker_ms: f64,
    frames_since_start: u32,
    frames_at_op_start: u32,
    op_in_progress: bool,
}

impl DemoState {
    fn new() -> Self {
        Self {
            worker: None,
            worker_ready: false,
            main_instance_id: String::from("—"),
            worker_instance_id: String::from("—"),
            last_main_ms: 0.0,
            last_worker_ms: 0.0,
            frames_since_start: 0,
            frames_at_op_start: 0,
            op_in_progress: false,
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
// WASM-инстансы
// =============================================================================

fn local_instance_id() -> String {
    wasm_in_worker::instance_id()
}

fn local_box_blur(data: &[u8], w: u32, h: u32, passes: u32) -> Vec<u8> {
    wasm_in_worker::box_blur_rgba(data, w, h, passes)
}

// =============================================================================
// Рисование тестового паттерна — через Reflect-хелперы
// =============================================================================

fn draw_test_pattern(ctx: &CanvasRenderingContext2d, w: u32, h: u32) {
    let ctx_js: &JsValue = ctx.as_ref();
    let cx = (w as f64) / 2.0;
    let cy = (h as f64) / 2.0;

    // Радиальный градиент.
    let grad_args = [
        JsValue::from_f64(cx),
        JsValue::from_f64(cy),
        JsValue::from_f64(0.0),
        JsValue::from_f64(cx),
        JsValue::from_f64(cy),
        JsValue::from_f64(cx),
    ];
    if let Some(grad) = ctx_call_js(ctx_js, "createRadialGradient", &grad_args) {
        let _ = ctx_call_js(
            &grad,
            "addColorStop",
            &[JsValue::from_f64(0.0), JsValue::from_str("#1f6feb")],
        );
        let _ = ctx_call_js(
            &grad,
            "addColorStop",
            &[JsValue::from_f64(0.5), JsValue::from_str("#0d1117")],
        );
        let _ = ctx_call_js(
            &grad,
            "addColorStop",
            &[JsValue::from_f64(1.0), JsValue::from_str("#8b949e")],
        );
        ctx_set(ctx_js, "fillStyle", &grad);
    }

    // Фон.
    ctx_call(ctx_js, "fillRect", &[0.0, 0.0, w as f64, h as f64]);

    // Концентрические окружности.
    for i in 0..8 {
        let r = 20.0 + i as f64 * 22.0;
        let color = if i % 2 == 0 { "#3fb950" } else { "#f0883e" };
        ctx_call(ctx_js, "beginPath", &[]);
        ctx_call(
            ctx_js,
            "arc",
            &[cx, cy, r, 0.0, std::f64::consts::PI * 2.0],
        );
        ctx_set(ctx_js, "strokeStyle", &JsValue::from_str(color));
        ctx_set(ctx_js, "lineWidth", &JsValue::from_f64(6.0));
        ctx_call(ctx_js, "stroke", &[]);
    }

    // Крест поверх — для контраста.
    ctx_call(ctx_js, "beginPath", &[]);
    ctx_call(ctx_js, "moveTo", &[0.0, cy]);
    ctx_call(ctx_js, "lineTo", &[w as f64, cy]);
    ctx_call(ctx_js, "moveTo", &[cx, 0.0]);
    ctx_call(ctx_js, "lineTo", &[cx, h as f64]);
    ctx_set(ctx_js, "strokeStyle", &JsValue::from_str("#f0f6fc"));
    ctx_set(ctx_js, "lineWidth", &JsValue::from_f64(2.0));
    ctx_call(ctx_js, "stroke", &[]);
}

// =============================================================================
// ImageData: чтение / запись через Reflect
// =============================================================================

fn get_image_bytes(
    ctx: &CanvasRenderingContext2d,
    w: u32,
    h: u32,
) -> Result<Vec<u8>, JsValue> {
    let ctx_js: &JsValue = ctx.as_ref();
    let args = [
        JsValue::from_f64(0.0),
        JsValue::from_f64(0.0),
        JsValue::from_f64(w as f64),
        JsValue::from_f64(h as f64),
    ];
    let img_val = ctx_call_js(ctx_js, "getImageData", &args)
        .ok_or_else(|| JsValue::from_str("getImageData returned undefined"))?;
    let img: ImageData = img_val
        .dyn_into()
        .map_err(|_| JsValue::from_str("getImageData did not return ImageData"))?;
    Ok(img.data().0)
}

fn put_image_bytes(
    ctx: &CanvasRenderingContext2d,
    bytes: &[u8],
    w: u32,
    h: u32,
) -> Result<(), JsValue> {
    let clamped = wasm_bindgen::Clamped(bytes);
    let img = ImageData::new_with_u8_clamped_array_and_sh(clamped, w, h)?;
    let ctx_js: &JsValue = ctx.as_ref();
    let args = [
        JsValue::from(img),
        JsValue::from_f64(0.0),
        JsValue::from_f64(0.0),
    ];
    ctx_call_js(ctx_js, "putImageData", &args)
        .ok_or_else(|| JsValue::from_str("putImageData returned undefined"))?;
    Ok(())
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("ww-stats") else {
        return;
    };
    let main_str = if st.last_main_ms > 0.0 {
        format_str(format_args!("{:.1} мс", st.last_main_ms))
    } else {
        String::from("—")
    };
    let worker_str = if st.last_worker_ms > 0.0 {
        format_str(format_args!("{:.1} мс", st.last_worker_ms))
    } else {
        String::from("—")
    };
    let frames_since_op = st.frames_since_start.saturating_sub(st.frames_at_op_start);
    let frames_str = if st.op_in_progress {
        format_str(format_args!("{frames_since_op} кадров"))
    } else if st.frames_at_op_start > 0 {
        format_str(format_args!("за операцию: {frames_since_op}"))
    } else {
        String::from("—")
    };
    let worker_status = if st.worker_ready {
        st.worker_instance_id.as_str()
    } else {
        "инициализация…"
    };
    let worker_class = if st.worker_ready { "ok" } else { "warn" };

    let html = format_str(format_args!(
        r#"<div><span class="label">Main WASM instance: </span><span class="value">{mid}</span></div>
           <div><span class="label">Worker WASM instance: </span><span class="{wc}">{wid}</span></div>
           <div><span class="label">Фильтр в main: </span><span class="warn">{mms}</span> · <span class="label">в worker: </span><span class="ok">{wms}</span></div>
           <div><span class="label">Кадры main thread: </span><span class="value">{fr}</span></div>"#,
        mid = st.main_instance_id,
        wid = worker_status,
        wc = worker_class,
        mms = main_str,
        wms = worker_str,
        fr = frames_str,
    ));
    el.set_inner_html(&html);
}

const DESC_LINES: &[&str] = &[
    "Один и тот же Rust-фильтр (<code>box_blur_rgba</code>) применяется \
     к изображению двумя способами. Сначала — в main thread'е, потом — \
     в Worker'е. Код один, разница — контекст выполнения.",
    "Смотрите на <strong>счётчик кадров main thread</strong>. Во время \
     «Filter in main thread» он замирает: браузер полностью занят \
     вычислениями. Во время «Filter in Worker» — продолжает расти: \
     main thread свободен.",
    "Два <strong>instance-id</strong> в статистике — доказательство, что \
     WASM загружен дважды. У main thread'а и Worker'а разные \
     <code>WebAssembly.Instance</code>, разные <code>thread_local!</code>, \
     разная линейная память. Общий только <code>.wasm</code> на диске.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ww-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("ww-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// rAF-счётчик: индикатор свободы main thread'а
// =============================================================================

fn start_raf_counter(state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let cell: Rc<RefCell<Option<wasm_bindgen::closure::Closure<dyn FnMut(f64)>>>> =
        Rc::new(RefCell::new(None));
    let cell2 = cell.clone();

    *cell.borrow_mut() = Some(wasm_bindgen::closure::Closure::wrap(Box::new(
        move |_now: f64| {
            {
                let mut st = state.borrow_mut();
                st.frames_since_start += 1;
            }
            let update = {
                let st = state.borrow();
                st.frames_since_start % 10 == 0
            };
            if update {
                let st = state.borrow();
                update_stats(&document, &st);
            }

            if let Some(w) = web_sys::window() {
                if let Some(cb) = cell2.borrow().as_ref() {
                    let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
                }
            }
        },
    ) as Box<dyn FnMut(f64)>));

    let cb = cell.borrow();
    window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    Ok(())
}

// =============================================================================
// Обработчик Worker'а
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

            if kind_str == "ready" {
                let id = Reflect::get(&data, &JsValue::from_str("instance"))
                    .ok()
                    .and_then(|v| v.as_string())
                    .unwrap_or_else(|| String::from("wasm-unknown"));
                {
                    let mut st = state.borrow_mut();
                    st.worker_ready = true;
                    st.worker_instance_id = id;
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
                return;
            }

            if kind_str == "init_error" {
                let msg = Reflect::get(&data, &JsValue::from_str("message"))
                    .ok()
                    .and_then(|v| v.as_string())
                    .unwrap_or_else(|| String::from("unknown"));
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    if let Some(el) = doc.get_element_by_id("ww-stats") {
                        let html = format_str(format_args!(
                            r#"<div><span class="label">Main WASM instance: </span><span class="value">{mid}</span></div>
                               <div><span class="err">Worker init error: {msg}</span></div>"#,
                            mid = state.borrow().main_instance_id,
                            msg = msg,
                        ));
                        el.set_inner_html(&html);
                    }
                }
                return;
            }

            if kind_str == "result" {
                let Some(buffer_val) =
                    Reflect::get(&data, &JsValue::from_str("buffer")).ok()
                else {
                    return;
                };
                let Ok(arr) = buffer_val.dyn_into::<Uint8Array>() else {
                    return;
                };
                let bytes = arr.to_vec();
                let elapsed = Reflect::get(&data, &JsValue::from_str("elapsed_ms"))
                    .ok()
                    .and_then(|v| v.as_f64())
                    .unwrap_or(0.0);
                let instance = Reflect::get(&data, &JsValue::from_str("instance_id"))
                    .ok()
                    .and_then(|v| v.as_string())
                    .unwrap_or_else(|| String::from("wasm-unknown"));

                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    let Some(canvas_el) = doc.get_element_by_id("ww-canvas") else {
                        return;
                    };
                    let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() else {
                        return;
                    };
                    let Ok(Some(ctx_val)) = canvas.get_context("2d") else {
                        return;
                    };
                    let Ok(ctx) = ctx_val.dyn_into::<CanvasRenderingContext2d>() else {
                        return;
                    };
                    let _ = put_image_bytes(&ctx, &bytes, CANVAS_W, CANVAS_H);
                }

                {
                    let mut st = state.borrow_mut();
                    st.last_worker_ms = elapsed;
                    st.worker_instance_id = instance;
                    st.op_in_progress = false;
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

// =============================================================================
// Кнопки
// =============================================================================

fn wire_redraw_button(document: &Document) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("ww-redraw")
        .ok_or_else(|| JsValue::from_str("no ww-redraw"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(c) = doc.get_element_by_id("ww-canvas") else {
            return;
        };
        let Ok(c) = c.dyn_into::<HtmlCanvasElement>() else {
            return;
        };
        let Ok(Some(ctx)) = c.get_context("2d") else {
            return;
        };
        let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
            return;
        };
        draw_test_pattern(&ctx, CANVAS_W, CANVAS_H);
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_main_filter_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("ww-main")
        .ok_or_else(|| JsValue::from_str("no ww-main"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(c) = doc.get_element_by_id("ww-canvas") else {
            return;
        };
        let Ok(c) = c.dyn_into::<HtmlCanvasElement>() else {
            return;
        };
        let Ok(Some(ctx)) = c.get_context("2d") else {
            return;
        };
        let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
            return;
        };

        {
            let mut st = state.borrow_mut();
            st.op_in_progress = true;
            st.frames_at_op_start = st.frames_since_start;
        }

        let Ok(bytes) = get_image_bytes(&ctx, CANVAS_W, CANVAS_H) else {
            return;
        };

        let t0 = js_sys::Date::now();
        let filtered = local_box_blur(&bytes, CANVAS_W, CANVAS_H, BLUR_PASSES);
        let elapsed = js_sys::Date::now() - t0;

        let _ = put_image_bytes(&ctx, &filtered, CANVAS_W, CANVAS_H);

        {
            let mut st = state.borrow_mut();
            st.last_main_ms = elapsed;
            st.op_in_progress = false;
        }
        let st = state.borrow();
        update_stats(&doc, &st);
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_worker_filter_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("ww-worker")
        .ok_or_else(|| JsValue::from_str("no ww-worker"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let worker = {
            let st = state.borrow();
            st.worker.clone()
        };
        let Some(w) = worker else {
            return;
        };

        let Some(c) = doc.get_element_by_id("ww-canvas") else {
            return;
        };
        let Ok(c) = c.dyn_into::<HtmlCanvasElement>() else {
            return;
        };
        let Ok(Some(ctx)) = c.get_context("2d") else {
            return;
        };
        let Ok(ctx) = ctx.dyn_into::<CanvasRenderingContext2d>() else {
            return;
        };

        let Ok(bytes) = get_image_bytes(&ctx, CANVAS_W, CANVAS_H) else {
            return;
        };

        {
            let mut st = state.borrow_mut();
            st.op_in_progress = true;
            st.frames_at_op_start = st.frames_since_start;
        }

        let _ = wasm_in_worker::post_image_to_worker(
            &w,
            &bytes,
            CANVAS_W,
            CANVAS_H,
            BLUR_PASSES,
        );
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

    // --- Service Worker cleanup ---
    //
    // Module worker делает `import "/pkg/hello_wasm02.js"`. Запрос идёт
    // через активные Service Worker'ы. SW из предыдущих демо не знает
    // `/pkg/` и ломает загрузку. Снимаем регистрацию и один раз
    // перезагружаем страницу — на следующей загрузке SW уже нет.
    if has_sw_controller() {
        show_sw_notice(&document)?;
        unregister_sw_and_reload(&document)?;
        return Ok(());
    }

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("22.4 — WASM в Worker: Rust в параллель"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_worker_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("ww-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает Worker. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Canvas.
    let canvas_el = document.create_element("canvas")?;
    canvas_el.set_id("ww-canvas");
    canvas_el.set_class_name("ww-canvas");
    let canvas: HtmlCanvasElement = canvas_el.dyn_into()?;
    canvas.set_width(CANVAS_W);
    canvas.set_height(CANVAS_H);
    root.append_child(&canvas)?;

    {
        let ctx = canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d"))?
            .dyn_into::<CanvasRenderingContext2d>()?;
        draw_test_pattern(&ctx, CANVAS_W, CANVAS_H);
    }

    // Кнопки.
    let controls = document.create_element("div")?;
    controls.set_class_name("ww-block");
    controls.set_inner_html(
        r#"<button class="ww-btn" id="ww-redraw">Redraw</button>
           <button class="ww-btn ww-btn-warn" id="ww-main">Filter in main thread</button>
           <button class="ww-btn ww-btn-primary" id="ww-worker">Filter in Worker</button>"#,
    );
    root.append_child(&controls)?;

    // Статистика.
    let stats = document.create_element("div")?;
    stats.set_class_name("ww-stats");
    stats.set_id("ww-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("ww-footnote");
    footnote.set_text_content(Some(
        "Worker — module worker ({ type: \"module\" }), обязательно для \
         import() внутри него. Загружает /pkg/hello_wasm02.js — тот же \
         pkg-пакет, что и main thread. Это создаёт ВТОРУЮ инстанциацию \
         WASM: своя линейная память, свой WebAssembly.Instance. Данные \
         между инстансами не разделяются — обмен только через \
         postMessage. Для настоящей shared memory нужен SharedArrayBuffer \
         (тема 22.5).",
    ));
    root.append_child(&footnote)?;

    // Состояние.
    let state = Rc::new(RefCell::new(DemoState::new()));
    state.borrow_mut().main_instance_id = local_instance_id();

    // Создаём module worker.
    let worker = wasm_in_worker::create_module_worker(WORKER_URL)?;
    wire_worker_messages(&worker, state.clone())?;
    state.borrow_mut().worker = Some(worker);

    // Обработчики UI.
    wire_redraw_button(&document)?;
    wire_main_filter_button(&document, state.clone())?;
    wire_worker_filter_button(&document, state.clone())?;

    // rAF-счётчик.
    start_raf_counter(state.clone())?;

    // Первая отрисовка.
    {
        let st = state.borrow();
        update_stats(&document, &st);
    }

    Ok(())
}
```

---

## 22.4.13. Анализ решения

**Какую ценность даёт демо.**

Подглава показывает **вторую инстанциацию WASM** на одном экране:

- **Два instance-id** — визуальное доказательство. `wasm-a1b2c3d4` (main) и `wasm-5e6f7a8b` (worker) — не совпадают. Один и тот же `.wasm` файл, разные инстансы.
- **Счётчик кадров main thread'а** — прямой индикатор блокировки. Замирает во время main-thread-фильтра, растёт во время worker-фильтра.
- **Один и тот же фильтр** в обоих контекстах — один Rust-код, одно поведение, разные условия выполнения.

**Скрытие технических сложностей.**

- Пользователь кликает две кнопки, видит разницу в счётчике кадров. Внутри — `import()` в Worker'е, `init()`, transferable-буферы, `WebAssembly.instantiate`.
- Пользователь видит два instance-id. Внутри — `thread_local!` с генерируемым при первом вызове ID.
- Пользователь не видит `import init, { box_blur_rgba } from "/pkg/hello_wasm02.js"` — это в JS-воркере, скрыто от app.rs.
- Service Worker cleanup происходит прозрачно — пользователь видит только короткий нотис.

**Что осознанно не реализовано.**

- **`wasm-bindgen-rayon`.** Продвинутая тема. Отдельная подглава 22.5.
- **Отдельный `.wasm`-модуль для Worker'а.** Упомянут в тексте, но не реализован — тот же pkg проще и демонстрирует главное: **вторую инстанциацию**.
- **`ImageBitmap` как транспорт.** Использован `ArrayBuffer` (transferable) — проще и уже отработан в 22.3. `ImageBitmap` упомянут в тексте.
- **`SharedArrayBuffer`.** Тема 22.5.

**Правила проекта, применённые в этой подглаве.**

1. **Autonomous app.rs.** Всё inline: `format_str`, `bool_of`, feature detection, работа с Worker, буфером, canvas.
2. **`format_str(format_args!(...))` в tail-позициях.** В `update_stats`, `instance_id`.
3. **`bool_of(matches!(...))` в tail-позиции.** В `is_worker_supported`.
4. **RefMut-дисциплина.** Каждый `borrow_mut` в узком scope, `borrow` — после drop.
5. **Feature detection перед использованием API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.** Правок в `main.css` не требуется.
8. **Уникальный префикс `ww-`.** Не пересекается с `tr-` (22.3), `dw-` (22.2), `oc-` (22.1), `ow-` (глава 22).
9. **Строки — через одну функцию `format_str`.** Никаких `format!` в tail-позиции.
10. **Никаких неиспользуемых импортов.** Все импорты задействованы.
11. **Внутренние ссылки через `crate::`.** В `app.rs` нет `hello_wasm02::` — только `crate::wasm_in_worker` или `use crate::wasm_in_worker`.
12. **Service Worker cleanup.** Если активен чужой SW — снять регистрацию и один раз перезагрузить.

**Проверки корректности.**

- `instance_id` использует `thread_local!` — уникален для инстанса WASM. В main thread и Worker'е — разные значения.
- `box_blur_rgba` экспортирован `#[wasm_bindgen]` — доступен из JS-воркера. В app.rs вызывается как обычная Rust-функция (не через JS-обёртку).
- `Uint8Array::new_with_length` создаёт JS-heap ArrayBuffer — transferable. Копирование из Rust — `copy_from`.
- `new Uint8Array(output)` в JS-воркере — обязательное копирование из wasm-памяти Worker-инстанса. Без него `copy.buffer` был бы view над wasm-памятью и не подлежал transfer.
- `create_module_worker` использует `WorkerOptions::set_type(WorkerType::Module)` — обязательно для `import()`.
- `start_raf_counter` использует self-referential Rc-цикл — гарантия, что счётчик живёт всё время страницы.
- `ctx_call` / `ctx_call_js` — обход бага r-a с сигнатурами canvas-методов.
- `has_sw_controller` проверяет `navigator.serviceWorker.controller !== null` через Reflect — не зависит от версии web-sys.
- `unregister_sw_and_reload` использует `sessionStorage`-флаг — защита от бесконечного reload-цикла.

---

## 22.4.14. Список доработок после первой итерации

1. **`bool_of` в feature detection.** `matches!(…)` в tail-позиции даёт E0308 rust-analyzer'а. Утилита применяется и в модуле, и в `app.rs`.
2. **`instance_id` через `thread_local!`.** Первая версия использовала `AtomicU32` — счётчик начинается с 0 в каждом инстансе, не отличает main от worker. Заменено на `thread_local!` + случайный ID при первом вызове.
3. **Копирование `output` в JS-воркере.** Первая версия пыталась трансферить `output.buffer` напрямую — это view над wasm-памятью, transfer невозможен. Заменено на `new Uint8Array(output)` — копия в JS-heap.
4. **rAF-счётчик вместо CSS-спиннера.** CSS-анимации на `transform` часто идут через compositor и **не замирают** при блокировке main thread'а. rAF-цикл — гарантированно на main thread. Использован rAF-счётчик.
5. **Обновление статистики каждые 10 кадров.** Первая версия обновляла DOM каждый кадр — 60 обновлений в секунду избыточно. Троттлинг до 6 обновлений в секунду снижает нагрузку и мигание.
6. **Два способа вызова `box_blur_rgba`.** В main thread — прямой Rust-вызов (`local_box_blur`). В Worker'е — через JS-модуль. Один код — два пути.
7. **`Redraw` между операциями.** После «Filter in main thread» canvas содержит размытую картинку. Вторая операция размыла бы её ещё сильнее. Кнопка Redraw восстанавливает исходный паттерн.
8. **Замена `hello_wasm02::` на `crate::`.** Внутри крейта имя пакета не резолвится. `app.rs` использует `use crate::wasm_in_worker;` и короткие имена. Тесты (`tests/*.rs`) — внешние крейты, там `hello_wasm02::` корректен.
9. **Обход сигнатур canvas-методов через Reflect.** r-a ошибается на `fill_rect`, `move_to`, `line_to`, `get_image_data`, `put_image_data` — считает аргументы `i32`. Хелперы `ctx_call` / `ctx_call_js` вызывают методы динамически, без проверки сигнатуры.
10. **Service Worker cleanup.** Module worker делает `import "/pkg/hello_wasm02.js"`. Активный SW от предыдущих демо перехватывает запрос и ломает загрузку: «A ServiceWorker intercepted the request…». Демо снимает регистрацию через `navigator.serviceWorker.getRegistrations()` → `unregister()` → `location.reload()`. Защита от цикла — `sessionStorage`-флаг.
11. **Обработка `init_error` от Worker'а.** Если WASM в Worker'е не смог загрузиться, main thread получает сообщение с ошибкой и показывает её в статистике.
12. **`WorkerOptions` и `WorkerType` в `Cargo.toml`.** Добавлены фичи web-sys — нужны для `create_module_worker`.

---

## 22.4.15. Итог

WASM-модуль **компилируется** один раз, но **инстанцируется** в каждом JS-контексте отдельно. Main thread и каждый Worker получают свои `WebAssembly.Instance`, свою линейную память, свои `thread_local!`. Общий только `.wasm`-файл на диске.

Module worker (`{ type: "module" }`) — обязательное условие для `import()` внутри Worker'а. Загрузка wasm-pack-пакета — стандартный паттерн: `import init, { fn } from "/pkg/hello_wasm02.js"; await init();`.

`Vec<u8>` из wasm-bindgen — это view над wasm-памятью. Для transfer нужна копия: `new Uint8Array(output)` в JS создаёт свежий ArrayBuffer в JS-heap.

Уникальный `instance_id` через `thread_local!` — визуальное доказательство двух инстансов. Main thread и Worker возвращают разные ID.

**Внутренние ссылки в `app.rs`** — через `crate::`, а не по имени пакета. Тесты — внешние, там `hello_wasm02::`.

**Service Worker и module worker конфликтуют.** Если на origin'е активен SW от предыдущего демо, `import "/pkg/..."` проходит через его fetch-обработчик и падает. Демо снимает регистрацию и один раз перезагружает страницу — self-healing.

Демо показывает всё это на одном экране: два instance-id, счётчик кадров main thread'а, один и тот же фильтр в двух контекстах. Wow-момент: во время «Filter in Worker» счётчик кадров продолжает расти — main thread действительно свободен. Во время «Filter in main thread» — замирает.

Подглава закрывает тему **второй инстанциации WASM**. SharedArrayBuffer и пулы Worker'ов (22.5), практический проект (22.6) — следующие подглавы.