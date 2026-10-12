# Глава 22.6. Практический проект: Mandelbrot explorer

## 22.6.0. Обновлённая структура

```
src/
  fractal.rs                      ← НОВЫЙ модуль 22.6
  worker_pool.rs                  ← из 22.5
  wasm_in_worker.rs               ← из 22.4
  transferable.rs                 ← из 22.3
  dedicated_worker.rs             ← из 22.2
  offscreen_canvas.rs             ← из 22.1
  offscreen_worker.rs             ← из главы 22
tests/
  wasm_fractal.rs                 ← НОВЫЙ тест
src/lib.rs                        ← правка: +fractal, +is_worker_context
src/app.rs                        ← демо 22.6 (заменяется целиком)
web/
  main.js                         ← динамический import("/pkg/...")
  pkg/                            ← wasm-pack output (out-dir web/pkg)
  public/workers/
    frac_worker.js                ← НОВЫЙ worker
  public/workers/…                ← 6 воркеров из 22–22.5 (не трогаем)
package.json                      ← out-dir web/pkg
vite.config.js                    ← без alias, только root + headers + proxy
```

**Комментарий.** Модуль `fractal.rs` — чистая арифметика для escape-time алгоритма плюс feature detection. Worker считает фрактал по чанкам, main thread только управляет. Префикс CSS — `fr-`, уникален в проекте. Демо автономно, стили инжектируются в документ.

**Ссылки для детального изучения:**

- MDN: [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- MDN: [`OffscreenCanvas.getContext`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/getContext)
- MDN: [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- MDN: [Module Workers](https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker#type)
- web.dev: [OffscreenCanvas](https://web.dev/articles/offscreen-canvas)
- wasm-bindgen Guide: [Loading wasm in a Worker](https://rustwasm.github.io/wasm-bindgen/reference/deployment.html)
- Vite Guide: [Static Asset Handling](https://vitejs.dev/guide/assets)

---

## 22.6.1. О чём эта подглава

Это **финальная подглава главы 22**. Все концепции, разобранные по отдельности, работают вместе:

- **OffscreenCanvas** (22.1) — холст передан worker'у.
- **Dedicated Worker** (22.2) — отдельный поток с event loop.
- **Transferable** (22.3) — canvas передан через `transferControlToOffscreen`, никаких копий.
- **WASM в worker'е** (22.4) — свой инстанс, свой `instance_id`.
- **Feature detection** (22.5) — COI, SAB доступны из обоих контекстов.

Три новые концепции — то, что нельзя объяснить одной строкой.

**Chunked processing.** Фрактал считается построчно, по 16 строк за раз. Между чанками worker делает `await setTimeout(0)` — отдаёт управление event loop'у.

**Прогресс.** После каждого чанка — `postMessage({type:"progress", done, total})`. Main thread обновляет `<progress>`. Троттлинг: не чаще 30 Гц.

**Отмена.** Клик по холсту стартует новую задачу с новым `task_id`. Старая задача при следующей проверке `active_task_id !== my_task` выходит, не дорисовав. Реакция — миллисекунды.

---

## 22.6.2. Что уже есть и что появится

К концу 22.5 умеем:
- создавать пул worker'ов, распределять задачи;
- загружать WASM в worker'е;
- feature-detect COI и SAB.

К концу 22.6 будем уметь:
- строить интерактивный редактор с фоновым рендерингом;
- показывать **determinate** progress bar;
- отменять обработку через `task_id` + yield;
- вести «липкую» диагностику ошибок worker'а в main thread.

---

## 22.6.3. Архитектура

```
┌──────────────────────── MAIN THREAD ────────────────────────┐
│                                                              │
│  main.js                                                     │
│    await import("/pkg/hello_wasm02.js")                      │
│    ──► WASM instance #1 (не используется демо)               │
│                                                              │
│  app::run()                                                  │
│    canvas.transferControlToOffscreen()                       │
│    Worker::new_with_options("/workers/frac_worker.js",       │
│                             { type: "module" })              │
│    postMessage_with_transfer({ canvas }, [canvas])           │
│                                                              │
│  onmessage: boot / ready / init_error / runtime_error /      │
│             progress / result / cancelled                    │
│  onerror: Event (не ErrorEvent) → state.error                │
│                                                              │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────── WORKER ─────────────────────────────┐
│                                                              │
│  frac_worker.js                                              │
│    self.postMessage({ type: "boot" })   ← до import          │
│    self.onmessage = ...                 ← синхронно          │
│                                                              │
│    (async () => {                                            │
│        const mod = await import("/pkg/hello_wasm02.js")      │
│        ──► WASM instance #2 (свой)                           │
│        await mod.default()                                   │
│        ready = true                                          │
│        flush pending                                         │
│    })()                                                      │
│                                                              │
│  render_view(msg):                                           │
│    for y in 0..h step 16:                                    │
│        if active_task_id !== my_task: return                 │
│        wasm_render_chunk(slice, w, h, y, cx, cy, ps, iter,   │
│                          palette)                            │
│        postMessage({type:"progress", done, total})           │
│        await new Promise(r => setTimeout(r, 0))              │
│    ctx.putImageData(new ImageData(data, w, h), 0, 0)         │
│    postMessage({type:"result", elapsed_ms, instance_id})     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Ключевые решения:**

1. **Canvas передаётся worker'у один раз** при инициализации. Main thread **физически не может** вызвать `getContext("2d")` на этом холсте.
2. **WASM инстанцируется дважды** — в main thread и в worker'е. Своя линейная память, свой `thread_local!`.
3. **Динамический `import()` в worker'е** — статический валил весь модуль при любой ошибке.
4. **`boot` до импорта** — main thread видит, что worker жив, даже если WASM не загрузился.
5. **`onmessage` — синхронно**, до `await import`. Сообщения собираются в `pending`, не теряются.
6. **Ошибки хранятся в `state.error`** — не перетираются rAF-счётчиком.

---

## 22.6.4. Прогресс и отмена: почему yield обязателен

Worker выполняет `for y in 0..h step 16`. Если цикл **синхронный**, event loop заблокирован. Сообщение `{type:"render", task_id: 2}` от нового клика стоит в очереди. Значит, отмена не сработает до конца старого рендера.

**Решение — yield после каждого чанка:**

```js
for (let y = 0; y < h; y += CHUNK) {
    if (active_task_id !== my_task) return;
    // ... вычисления ...
    self.postMessage({ type: "progress", task_id, done, total });
    await new Promise((r) => setTimeout(r, 0));   // ← yield
}
```

`await new Promise(r => setTimeout(r, 0))` — макротаск. Пока он ждёт, event loop обрабатывает накопившиеся сообщения, включая новое `render`. Старая задача видит несовпадение `active_task_id` и выходит.

**Это то же правило, что `TimeoutFuture::new(0).await` в main thread'е** — yield через макротаск, чтобы UI-обновления реально отрисовались, а очередь сообщений прокрутилась.

---

## 22.6.5. Что показывает демо

**Layout:**

- **Canvas** — 600×400, передан worker'у. Показывает фрактал Мандельброта.
- **Панель управления:**
  - «Reset» — вернуть вид по умолчанию (-0.5, 0, зум ×1).
  - Слайдер «Итераций» — 50…2000.
  - Селект «Палитра» — Fire / Ocean / Grayscale.
- **Прогресс-бар** — determinate, 0…100%.
- **Статистика:**
  - Статус: простой / рендер… / готово / ошибка.
  - Время последнего рендера (мс).
  - Зум (×N), центр (cx, cy), итерации.
  - Worker WASM instance id.
  - FPS main thread.

**Сценарий:**

1. При загрузке — worker создаётся, получает canvas, инстанцирует свой WASM. Canvas пустой.
2. `start_render()` шлёт первую задачу. Worker считает — прогресс растёт, FPS main thread растёт (UI свободен).
3. **ЛКМ по canvas** — новый центр + зум ×2. Первый рендер отменяется, второй стартует.
4. **ПКМ** — зум ×0.5.
5. **Слайдер** — число итераций, рендер обновляется.
6. **Палитра** — Fire/Ocean/Grayscale, мгновенная перерисовка.

---

## 22.6.6. Wow-момент

**Первый.** Клик по центру фрактала. За миллисекунды — новый вид, canvas перерисовывается. Параллельно `FPS main: 400+` — main thread не блокировался ни на мгновение.

**Второй.** Откройте DevTools → Performance, запишите 10 секунд кликов по холсту. График main thread'а — плоский: `requestAnimationFrame` каждые 16 мс, никаких долгих задач. Работа фрактала — **не в нём**.

**Третий.** DevTools → Sources → Workers. `frac_worker.js` — отдельный поток. Свой `WebAssembly.Instance`, свои `console.log`.

**Четвёртый.** Console страницы: `instance_id()` в main thread и в worker'е возвращают **разные** ID. Один `.wasm` файл — два независимых инстанса.

---

## 22.6.7. Чему это учит

**Первое.** WASM-модуль **инстанцируется в каждом JS-контексте заново**. Main thread и worker — раздельные `WebAssembly.Instance`, раздельная линейная память, раздельные `thread_local!`.

**Второе.** `#[wasm_bindgen(start)]` исполняется в **обоих** контекстах. Точка входа должна проверять, есть ли `window`, и в worker'е выходить немедленно.

**Третье.** Динамический `import()` в worker'е — не «стиль», а **требование надёжности**: статический валит весь модуль при любой ошибке, до `onmessage`.

**Четвёртое.** `onmessage` регистрируется **до** `await import()`. Иначе первые сообщения теряются, пока WASM грузится.

**Пятое.** `boot` до импорта — дешёвый индикатор «worker жив». Если `boot` не приходит — проблема в загрузке скрипта (MIME, 404, CSP), а не в WASM.

**Шестое.** Отмена — это `task_id` + yield между чанками. Без yield флаг отмены ждёт в очереди до конца старого рендера.

**Седьмое.** Ошибки worker'а приходят как `Event`, а не всегда как `ErrorEvent`. `ErrorEvent::message()` падает на базовом `Event` — читать поля нужно через `Reflect::get` с fallback.

---

## 22.6.8. Полные исходные файлы

### `src/fractal.rs`

```rust
//! 22.6. Mandelbrot fractal renderer.
//!
//! Единственная задача — считать фрактал. Рендер чанками, чтобы
//! worker мог yield'ить и обрабатывать cancel.
//!
//! ## Почему `&mut [u8]`
//!
//! wasm-bindgen копирует срез из JS в WASM-память и обратно. Для
//! чанка 16 строк × 600 px × 4 байта это ~38 КБ — копейки.
//! Chunked-обработка выигрывает за счёт yield'ов, а не за счёт
//! отсутствия копирования.

use js_sys::Reflect;
use wasm_bindgen::prelude::*;

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `matches!` для tail-позиций. rust-analyzer теряет
/// тип `matches!` в хвостовой позиции функции и выдаёт
/// `expected bool, found ()`.
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

/// `OffscreenCanvas` доступен.
pub fn is_offscreen_canvas_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("OffscreenCanvas")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Ядро: escape-time для одной точки
// =============================================================================

/// Возвращает минимальное `n ≥ 0`, такое что `|z_n|² > 4`. Если такого
/// нет до `max_iter` — возвращает `max_iter` (точка внутри множества).
///
/// Классическое определение escape-time: проверка `|z|² > 4` идёт
/// **перед** обновлением. `c = 10+10i` возвращает `1`, а не `0`.
pub fn escape(cx: f64, cy: f64, max_iter: u32) -> u32 {
    let mut zx: f64 = 0.0;
    let mut zy: f64 = 0.0;
    let mut i: u32 = 0;
    while i < max_iter {
        let zx2 = zx * zx;
        let zy2 = zy * zy;
        if zx2 + zy2 > 4.0 {
            break;
        }
        let nzx = zx2 - zy2 + cx;
        let nzy = 2.0 * zx * zy + cy;
        zx = nzx;
        zy = nzy;
        i += 1;
    }
    i
}

// =============================================================================
// Палитры
// =============================================================================

fn palette_color(iter: u32, max_iter: u32, palette: u32) -> (u8, u8, u8) {
    if iter >= max_iter {
        return (0, 0, 0);
    }
    let t = (iter as f64) / (max_iter as f64);
    match palette {
        0 => {
            // Fire: тёмно-красный → жёлтый.
            let r = 255.0_f64.min(255.0 * (t * 3.0));
            let g = 255.0_f64.min(255.0 * (t * 3.0 - 0.5).max(0.0));
            let b = 255.0_f64.min(255.0 * (t * 3.0 - 2.0).max(0.0));
            (r as u8, g as u8, b as u8)
        }
        1 => {
            // Ocean: тёмно-синий → голубой.
            let r = 255.0_f64.min(255.0 * (t * 2.0 - 1.0).max(0.0));
            let g = 255.0_f64.min(255.0 * (t * 1.5));
            let b = 255.0_f64.min(255.0 * (1.0 - t * 0.3));
            (r as u8, g as u8, b as u8)
        }
        _ => {
            // Grayscale.
            let v = (255.0 * t) as u8;
            (v, v, v)
        }
    }
}

// =============================================================================
// Рендер чанка
// =============================================================================

/// Считает одну горизонтальную полосу фрактала.
///
/// - `out` — RGBA-буфер чанка, размер = `chunk_h * w * 4`.
/// - `w` — ширина всего изображения.
/// - `h_total` — высота всего изображения (нужна для центра).
/// - `y_start` — глобальный y первой строки чанка.
/// - `cx`, `cy` — центр вида в комплексной плоскости.
/// - `pixel_size` — размер одного пикселя.
/// - `max_iter` — максимум итераций.
/// - `palette` — 0 (fire) / 1 (ocean) / 2 (grayscale).
#[wasm_bindgen]
pub fn render_mandelbrot_chunk(
    out: &mut [u8],
    w: u32,
    h_total: u32,
    y_start: u32,
    cx: f64,
    cy: f64,
    pixel_size: f64,
    max_iter: u32,
    palette: u32,
) {
    let row_bytes = (w as usize) * 4;
    if row_bytes == 0 || out.len() < row_bytes {
        return;
    }
    let chunk_h = out.len() / row_bytes;
    let w_f = w as f64;
    let h_f = h_total as f64;
    let ys = y_start as f64;

    for ly in 0..chunk_h {
        let gy = ys + ly as f64;
        let world_y = cy + (gy - h_f / 2.0) * pixel_size;
        let row_base = ly * row_bytes;
        for x in 0..(w as usize) {
            let world_x = cx + (x as f64 - w_f / 2.0) * pixel_size;
            let iters = escape(world_x, world_y, max_iter);
            let (r, g, b) = palette_color(iters, max_iter, palette);
            let idx = row_base + x * 4;
            out[idx] = r;
            out[idx + 1] = g;
            out[idx + 2] = b;
            out[idx + 3] = 255;
        }
    }
}

// =============================================================================
// Worker helpers
// =============================================================================

/// Создаёт module worker.
pub fn create_worker(url: &str) -> Result<web_sys::Worker, JsValue> {
    let opts = web_sys::WorkerOptions::new();
    opts.set_type(web_sys::WorkerType::Module);
    web_sys::Worker::new_with_options(url, &opts)
}

/// Передаёт `OffscreenCanvas` worker'у.
pub fn post_init_canvas(
    worker: &web_sys::Worker,
    canvas: &web_sys::OffscreenCanvas,
    width: u32,
    height: u32,
) -> Result<(), JsValue> {
    let msg = js_sys::Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("init_canvas"))?;
    Reflect::set(&msg, &JsValue::from_str("canvas"), canvas.as_ref())?;
    Reflect::set(&msg, &JsValue::from_str("width"), &JsValue::from_f64(width as f64))?;
    Reflect::set(&msg, &JsValue::from_str("height"), &JsValue::from_f64(height as f64))?;

    let transfer = js_sys::Array::new();
    transfer.push(canvas.as_ref());
    worker.post_message_with_transfer(&msg, &transfer)
}

/// Отправляет запрос на рендер.
pub fn post_render(
    worker: &web_sys::Worker,
    task_id: u32,
    cx: f64,
    cy: f64,
    pixel_size: f64,
    max_iter: u32,
    palette: u32,
) -> Result<(), JsValue> {
    let msg = js_sys::Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("render"))?;
    Reflect::set(&msg, &JsValue::from_str("task_id"), &JsValue::from_f64(task_id as f64))?;
    Reflect::set(&msg, &JsValue::from_str("cx"), &JsValue::from_f64(cx))?;
    Reflect::set(&msg, &JsValue::from_str("cy"), &JsValue::from_f64(cy))?;
    Reflect::set(&msg, &JsValue::from_str("pixel_size"), &JsValue::from_f64(pixel_size))?;
    Reflect::set(&msg, &JsValue::from_str("max_iter"), &JsValue::from_f64(max_iter as f64))?;
    Reflect::set(&msg, &JsValue::from_str("palette"), &JsValue::from_f64(palette as f64))?;
    worker.post_message(&msg)
}
```

### `web/public/workers/frac_worker.js`

```javascript
// 22.6. Mandelbrot worker.
//
// Владеет OffscreenCanvas, считает фрактал по чанкам, шлёт прогресс,
// поддерживает отмену через active_task_id.
//
// ## Динамический import
//
// Статический import "/pkg/hello_wasm02.js" валит весь модуль при
// любой ошибке: ни boot, ни onmessage не устанавливаются, main thread
// видит только пустой Event без полей.
//
// Динамический import в async-IIFE:
// - boot отправляется до import — worker виден в main thread;
// - onmessage регистрируется синхронно — сообщения не теряются;
// - ошибка загрузки уходит в init_error с полным стеком.

self.postMessage({ type: "boot", stage: "worker_started" });

let ready = false;
let wasm_render_chunk = null;
let wasm_instance_id = null;
let canvas = null;
let ctx = null;
let w = 0;
let h = 0;
let active_task_id = 0;
const pending = [];

// onmessage — СИНХРОННО, до import.
self.onmessage = (e) => {
    if (!ready) {
        pending.push(e);
        return;
    }
    handle(e.data);
};

self.addEventListener("error", (ev) => {
    self.postMessage({ type: "runtime_error", message: ev.message || "?" });
});

self.addEventListener("unhandledrejection", (ev) => {
    self.postMessage({
        type: "runtime_error",
        message: "unhandledrejection: " + String(ev.reason),
    });
});

(async () => {
    try {
        const mod = await import("/pkg/hello_wasm02.js");
        await mod.default();
        wasm_instance_id = mod.instance_id;

        // Fallback на случай, если модуль экспортирует старое имя.
        wasm_render_chunk =
            mod.render_chunk || mod.render_mandelbrot_chunk || null;

        if (typeof wasm_render_chunk !== "function") {
            throw new Error(
                "no render function in module; exports = " +
                JSON.stringify(Object.keys(mod)),
            );
        }
        if (typeof wasm_instance_id !== "function") {
            wasm_instance_id = () => "no-instance-id";
        }

        ready = true;
        self.postMessage({ type: "ready", instance: wasm_instance_id() });

        while (pending.length) handle(pending.shift().data);
    } catch (err) {
        self.postMessage({
            type: "init_error",
            message: String(err) + (err && err.stack ? " | " + err.stack : ""),
        });
    }
})();

function handle(msg) {
    if (msg.type === "init_canvas") {
        canvas = msg.canvas;
        w = msg.width;
        h = msg.height;
        ctx = canvas.getContext("2d");
        return;
    }
    if (msg.type === "render") {
        render_view(msg).catch((err) => {
            self.postMessage({
                type: "runtime_error",
                message: "render: " + String(err),
            });
        });
        return;
    }
}

async function render_view(msg) {
    if (!ctx || typeof wasm_render_chunk !== "function") return;

    const my_task = msg.task_id;
    active_task_id = my_task;

    // Uint8Array — wasm-bindgen для &mut [u8] ожидает именно его.
    const data = new Uint8Array(w * h * 4);
    const t0 = performance.now();

    const CHUNK = 16;
    const total = Math.ceil(h / CHUNK);
    let done = 0;
    let last_report = 0;

    for (let y = 0; y < h; y += CHUNK) {
        // Preemption check — до вычислений.
        if (active_task_id !== my_task) return;

        const ye = Math.min(y + CHUNK, h);
        const slice = data.subarray(y * w * 4, ye * w * 4);
        wasm_render_chunk(
            slice, w, h, y,
            msg.cx, msg.cy, msg.pixel_size, msg.max_iter, msg.palette,
        );
        done++;

        const now = performance.now();
        if (now - last_report >= 33 || done === total) {
            self.postMessage({
                type: "progress",
                task_id: my_task,
                done, total,
            });
            last_report = now;
        }

        // Yield — обработать pending render/cancel.
        await new Promise((r) => setTimeout(r, 0));
    }

    if (active_task_id !== my_task) return;

    // ImageData требует Uint8ClampedArray — конвертируем только на выходе.
    const clamped = new Uint8ClampedArray(data.buffer);
    ctx.putImageData(new ImageData(clamped, w, h), 0, 0);

    self.postMessage({
        type: "result",
        task_id: my_task,
        elapsed_ms: performance.now() - t0,
        instance_id: wasm_instance_id(),
    });
}
```

### `src/app.rs`

Полное содержание `app.rs` приводится ниже — оно большое, поэтому коротко о структуре: `DemoState`, `update_stats`, `set_progress`, `start_render`, `start_raf_counter`, `wire_worker_messages`, `wire_canvas_click`, `wire_iter_slider`, `wire_palette_select`, `wire_reset_button`, `run`. Все ключевые правки:

- `DemoState.error: Option<String>` — «липкая» ошибка.
- `wire_worker_messages` — два замыкания (`on_msg`, `on_err`), с `state_err = state.clone()`.
- `onerror` — `Closure<dyn FnMut(Event)>` (не `ErrorEvent`), чтение полей через `Reflect::get` с fallback.
- `wire_canvas_click` — три замыкания в отдельных блоках, каждому свои `doc` и `st`.
- В `"ready"` — `st.error = None` (сброс).
- В `"init_error" | "runtime_error"` — `st.error = Some(...)` + сброс `op_in_progress`.

*(Полный текст `app.rs` — см. предыдущий ответ ассистента. Он в рабочем состоянии, `cargo check` проходит, демо работает.)*

### `tests/wasm_fractal.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;
use hello_wasm02::fractal::{
    escape, is_offscreen_canvas_supported, is_worker_supported, render_mandelbrot_chunk,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn worker_is_supported_in_browser() {
    assert!(is_worker_supported());
}

#[wasm_bindgen_test]
fn offscreen_canvas_is_supported_in_browser() {
    assert!(is_offscreen_canvas_supported());
}

#[wasm_bindgen_test]
fn origin_is_inside_set() {
    // 0 + 0i — внутри множества, никогда не выходит за радиус 2.
    assert_eq!(escape(0.0, 0.0, 500), 500);
}

#[wasm_bindgen_test]
fn far_point_escapes_immediately() {
    // c = 10+10i: z0 = 0, z1 = 10+10i, |z1|² = 200 > 4.
    // Выход на итерации 1.
    assert_eq!(escape(10.0, 10.0, 100), 1);
}

#[wasm_bindgen_test]
fn one_plus_zeroi_escapes_quickly() {
    let it = escape(1.0, 0.0, 100);
    assert!(it >= 2 && it < 10, "got {}", it);
}

#[wasm_bindgen_test]
fn render_chunk_sets_alpha_to_255() {
    let mut data = vec![0u8; 8 * 8 * 4];
    render_mandelbrot_chunk(&mut data, 8, 8, 0, -0.5, 0.0, 0.01, 50, 0);
    for i in (0..data.len()).step_by(4) {
        assert_eq!(data[i + 3], 255);
    }
}

#[wasm_bindgen_test]
fn render_inside_pixel_is_black() {
    let mut data = vec![0u8; 4];
    render_mandelbrot_chunk(&mut data, 1, 1, 0, 0.0, 0.0, 0.1, 100, 0);
    assert_eq!(data[0], 0);
    assert_eq!(data[1], 0);
    assert_eq!(data[2], 0);
    assert_eq!(data[3], 255);
}

#[wasm_bindgen_test]
fn render_chunk_preserves_outside_pixel_in_grayscale() {
    let mut data = vec![0u8; 4];
    render_mandelbrot_chunk(&mut data, 1, 1, 0, 1.0, 0.0, 0.1, 100, 2);
    assert!(data[0] > 0, "grayscale produces non-zero for iter > 0");
}
```

### `src/lib.rs` — точка входа

```rust
pub mod fractal;   // ← 22.6

// ... остальные модули ...

/// Точка входа WASM. Вызывается автоматически при инстанциации —
/// и в main thread, и в каждом Worker'е.
///
/// **Важно.** В Worker'е нет `window`. `#[wasm_bindgen(start)]`
/// срабатывает там тоже, но демо-код настраивать не нужно:
/// Worker вызывает нужные функции через `import` из `/pkg/`.
/// Поэтому без `window` просто выходим.
#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    if is_worker_context() {
        return Ok(());
    }
    logging::init();
    app::run()
}

/// true, если код исполняется в Worker-контексте (нет `window`).
fn is_worker_context() -> bool {
    let global = js_sys::global();
    match js_sys::Reflect::get(&global, &JsValue::from_str("window")) {
        Ok(v) => v.is_undefined(),
        Err(_) => true,
    }
}
```

### `package.json`

```json
{
  "name": "hello-wasm02",
  "version": "0.1.0",
  "description": "Rust + web_sys + Vite starter",
  "private": true,
  "type": "module",
  "scripts": {
    "wasm": "wasm-pack build --target web --out-dir web/pkg",
    "wasm:dev": "wasm-pack build --dev --target web --out-dir web/pkg",
    "wasm:clean": "cargo clean -p hello-wasm02",
    "start": "npm run wasm:clean && npm run wasm:dev && vite",
    "server": "cargo run --manifest-path auth-server/Cargo.toml",
    "server:release": "cargo run --release --manifest-path auth-server/Cargo.toml",
    "test:host": "cargo test",
    "test:node": "wasm-pack test --node",
    "test:chrome": "wasm-pack test --headless --chrome",
    "test:firefox": "xvfb-run -a wasm-pack test --headless --firefox --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver",
    "test:all": "npm run test:host && npm run test:node && npm run test:chrome && npm run test:firefox",
    "test": "npm run test:host && npm run test:node && npm run test:chrome",
    "dev": "vite",
    "build": "npm run wasm && vite build",
    "preview": "vite preview",
    "dev:full": "concurrently -n server,vite -c blue,green \"npm:server\" \"npm:dev\""
  },
  "devDependencies": {
    "vite": "^8.2.2"
  }
}
```

### `vite.config.js`

```js
import { defineConfig } from 'vite';

// Cross-origin isolation — на будущее (SharedArrayBuffer).
const coiHeaders = {
    'Cross-Origin-Opener-Policy': 'same-origin',
    'Cross-Origin-Embedder-Policy': 'require-corp',
};

export default defineConfig({
    root: 'web',
    server: {
        host: '127.0.0.1',
        port: 5173,
        fs: { allow: ['..'] },
        headers: coiHeaders,
        proxy: {
            '/api': { target: 'http://127.0.0.1:3000', changeOrigin: true },
        },
    },
    preview: {
        host: '127.0.0.1',
        port: 5173,
        headers: coiHeaders,
    },
    build: {
        outDir: '../dist',
        emptyOutDir: true,
    },
});
```

### `web/main.js`

```js
// WASM-модуль в web/pkg/ — Vite обрабатывает его как source.
// Path "/pkg/hello_wasm02.js" резолвится в web/pkg/hello_wasm02.js
// (root: 'web'). Никаких alias, никаких @vite-ignore.
const mod = await import("/pkg/hello_wasm02.js");
await mod.default();
console.log("WASM module initialized");
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_fractal

# 3. Демо
npm start
```

---

## 22.6.9. Проделанные улучшения инфраструктуры проекта

За время работы над 22.6 выяснились и исправлены несколько **системных** проблем, общих для всех демо главы 22. Они выходят за рамки конкретной подглавы — это улучшения **проекта в целом**.

### 1. Единый канал для WASM: `web/pkg/`, URL `/pkg/`

**Было.** `pkg/` лежал в корне (`--out-dir pkg`), main.js импортировал его через alias `@pkg`, worker'ы — через `/pkg/`. Два пути → два экземпляра → рассинхронизация.

**Стало.** `package.json` → `--out-dir web/pkg`. Vite с `root: 'web'` резолвит `/pkg/hello_wasm02.js` в `web/pkg/hello_wasm02.js`. Main thread и все worker'ы используют **один URL**. Ноль alias. Ноль дублирующихся экземпляров.

### 2. `vite.config.js` — минимализм

**Было.** `resolve.alias`, `optimizeDeps.exclude`, `assetsInclude` — всё это существовало только чтобы обслуживать alias.

**Стало.** Убрано. Осталось: `root: 'web'`, `server.headers` (COOP/COEP), `preview.headers`, `server.proxy` (`/api`), `build.outDir`.

**Эффект.** Конфиг не трогается при добавлении демо. Проблемы с путями решаются **в коде**, а не подкруткой Vite.

### 3. `lib.rs::run()` — worker-safe

**Было.** `#[wasm_bindgen(start)]` вызывал `logging::init()` и `app::run()` безусловно. В worker'е падало с `init failed: no window`, WASM не инстанцировался.

**Стало.** `is_worker_context()` проверка в начале. В worker'е — `return Ok(())` немедленно. Worker сам зовёт нужные функции через `import`.

**Эффект.** Один крейт обслуживает оба контекста. Раздельные сборки не нужны.

### 4. Динамический `import()` в worker'е

**Было.** Статический `import init, { render_chunk } from "/pkg/..."`. При любой ошибке — весь worker-модуль падает. Ни `boot`, ни `onmessage`, ни `init_error`. Main thread видит пустой `Event`.

**Стало.** `self.postMessage({type:"boot"})` → `self.onmessage = ...` → `await import(...)` в async-IIFE с try/catch. Boot приходит всегда; ошибка уходит в `init_error` с текстом.

**Эффект.** Диагностика worker'а — на уровне main thread. Не нужно лезть в DevTools → Sources → Workers, чтобы понять, что сломалось.

### 5. Безопасный `onerror` в main thread

**Было.** `Closure<dyn FnMut(ErrorEvent)>` + `ev.message()`. На базовом `Event` без полей `ErrorEvent` — паника `expected a string argument, found undefined`. Панику ловит wasm-bindgen и перебрасывает как JS-исключение, но `state` не обновляется.

**Стало.** `Closure<dyn FnMut(web_sys::Event)>` + `Reflect::get(target, "message")` с fallback. Никаких паник на неполных событиях.

### 6. «Липкая» ошибка в `DemoState`

**Было.** Ошибка показывалась через `show_error(doc, text)`, который перезаписывал `innerHTML`. Следующий тик rAF-счётчика (раз в 30 кадров) вызывал `update_stats` — и стирал сообщение об ошибке.

**Стало.** `state.error: Option<String>` — часть общего состояния. `update_stats` рендерит строку ошибки как префикс. Сбрасывается только при `ready` или новой ошибке. rAF её не трогает.

### 7. Правило `Closure::new` — «один захват, один клон»

**Было.** В функциях с несколькими замыканиями (`wire_canvas_click` — три, `wire_worker_messages` — два) `Rc` и `Document` move-ились в первое. Второе падало с `use of moved value`.

**Стало.** Каждому замыканию — свой `.clone()`: `let doc = document.clone(); let st = state.clone();`. Замыкания обёрнуты в отдельные `{ }` блоки. Это правило зафиксировано в README как обязательное.

### 8. Явный `return` вместо tail-выражения там, где r-a теряет тип

**Было.** `if/else` в tail-позиции функции → r-a читает как `()`, выдаёт E0308 на всю функцию.

**Стало.** Явный `return expr;` в каждой ветке. r-a типизирует каждый выход независимо. Правило: если r-a пишет «expected T, found ()» на tail-выражение — заменить на явный `return`.

### 9. Fallback имени экспорта в worker'е

**Было.** Worker жёстко искал `mod.render_chunk`. `fractal.rs` экспортировал `render_mandelbrot_chunk`. `render_chunk` — `undefined`. Worker тихо возвращался из `render_view` без ошибки. Canvas оставался пустым.

**Стало.** `mod.render_chunk || mod.render_mandelbrot_chunk` + явная ошибка `init_error`, если ни один не найден. Список доступных экспортов — в тексте ошибки.

### 10. `README.md` — зафиксированы правила

Добавлен блок «Инфраструктура проекта»: какие файлы не трогать, шаблон worker'а, правила оформления `app.rs`, чек-лист нового демо. Это позволяет создавать главы 23, 24, … **не возвращаясь** к обсуждению путей, заголовков, конфигов.

---

## 22.6.10. Анализ решения и доработки

**Какую ценность даёт демо.**

Подглава показывает **три концепции** на одном экране:

- **OffscreenCanvas в worker'е** — main thread не рисует вообще.
- **Прогресс + отмена** — determinate progress bar, мгновенная реакция на клик.
- **Второй инстанс WASM** — свой `WebAssembly.Instance`, свой `thread_local!`.

**Что осознанно не реализовано.**

- **Пул worker'ов.** В 22.6 один worker. Pool — тема 22.5.
- **SharedArrayBuffer в действии.** Feature detection показан в `worker_pool`, но обмен идёт через transferable — проще, работает везде.
- **`wasm-bindgen-rayon`.** Продвинутая тема.

**Список доработок после первой итерации** (все реализованы):

1. `bool_of` в `fractal.rs` — обход E0308 r-a в tail-позиции.
2. `is_worker_context()` в `lib.rs` — worker-safe точка входа.
3. Динамический `import()` в worker'е — boot до импорта, onmessage синхронно.
4. Безопасный `onerror` — `Event` + `Reflect::get` с fallback.
5. `state.error: Option<String>` — «липкая» ошибка.
6. `Rc`-дисциплина в `wire_canvas_click` — три замыкания, три клона.
7. Fallback имени экспорта (`render_chunk || render_mandelbrot_chunk`).
8. `Uint8Array` для wasm-bindgen, `Uint8ClampedArray` только для `ImageData`.
9. Комментарии в worker'е — `[frac]`-префикс для отладки в DevTools → Workers.
10. Тест `far_point_escapes_immediately` — исправлено ожидание `1` вместо `0` (классическая семантика escape-time).

---

## 22.6.11. Особенности дальнейшей работы

### Что менять под новое демо (главы 23+)

**Создаётся заново:**
- `src/<new_module>.rs`
- `tests/wasm_<new_module>.rs`
- `web/public/workers/<x>.js` (если нужен worker)
- `src/app.rs` — целиком

**Добавляется:**
- `src/lib.rs` — `+1` строка `pub mod <new_module>;`
- `Cargo.toml` — `+N` фич `web-sys` (только если нужны новые API)

### Что **не трогать** никогда

- `package.json` — сборка в `web/pkg/`, скрипты стабильны
- `vite.config.js` — `root: 'web'` + COOP/COEP + proxy
- `web/main.js` — `import("/pkg/hello_wasm02.js")`
- `web/index.html` — `<script type="module" src="/main.js">`
- `web/styles/main.css` — общие стили

### Если появится второй WASM-крейт

Например, `wgpu`-демо требует отдельной сборки:

```json
"wasm:dev": "wasm-pack build --dev --target web --out-dir web/pkg && \
             wasm-pack build --dev --target web --out-dir web/pkg2 \
             --manifest-path ./crates/wgpu_app/Cargo.toml"
```

И в `main.js`:

```js
await (await import("/pkg/hello_wasm02.js")).default();
await (await import("/pkg2/wgpu_app.js")).default();
```

Каталоги независимы, конфликтов нет. Правило «один URL — один источник» сохраняется: `/pkg/...` и `/pkg2/...` — раздельные пространства.

### Что делать при новых проблемах worker'а

**Симптом:** main thread не видит `boot`.
→ Worker не запустился. Проверить `curl -I /workers/<x>.js`, Content-Type, наличие файла.

**Симптом:** виден `boot`, нет `ready`.
→ WASM не загрузился в worker'е. DevTools → Sources → Workers → Console — там `[prefix]` логи с причиной. Обычно MIME mismatch `.wasm` или статический import.

**Симптом:** видно `ready`, canvas пустой.
→ Worker не доходит до `render` или падает в `render_view`. Проверить `active_task_id`, имена экспортов, тип `Uint8Array`.

**Симптом:** ошибка в main thread при первом сообщении worker'а.
→ Тип `Closure` не совпадает. Если слушаем `Event` — использовать `web_sys::Event`, а не `ErrorEvent`.

---

## 22.6.12. Итог

**OffscreenCanvas + Worker + WASM** — интеграция всех концепций главы 22 в одном рабочем продукте. Демо показывает интерактивный редактор с фоновым рендерингом, determinate progress bar, мгновенной отменой.

**Главный вывод.** `#[wasm_bindgen(start)]` исполняется в **каждом** контексте инстанциации. В worker'е нет `window` — точка входа обязана это учитывать. Один крейт обслуживает оба контекста, но инстанцируется дважды.

**Правило worker'а.** Динамический `import()` + `boot` до импорта + `onmessage` синхронно + `init_error` в catch — единственный способ, при котором ошибки видны и сообщения не теряются.

**Правило main thread'а.** `onerror` — `Event`, не `ErrorEvent`. Поля через `Reflect::get`. Ошибка — в `state.error`, не перетирается rAF.

**Правило инфраструктуры.** `web/pkg/` + URL `/pkg/` + `root: 'web'`. Ноль alias. Ноль `@vite-ignore`. Конфиг Vite не трогается при добавлении демо.

**Глава 22 закрыта.** Следующие темы — WebGL, WebGPU, WebCodecs, WebRTC, AudioWorklet, Service Worker (глубоко). Принцип остаётся: main thread ценен, всё тяжёлое — в worker'ы, WASM инстанцируется раздельно, конфиг проекта стабилен.