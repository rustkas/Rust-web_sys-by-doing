# Глава 28. Worker и параллельное выполнение

## 28.0. Что вы получите в этой главе

Работающий **pipeline обработки изображений**: тяжёлые фильтры (grayscale, box blur) выполняются в Web Worker, главный поток остаётся отзывчивым, прогресс отображается в UI. Демо в `app.rs` показывает загрузку изображения, выбор фильтра, прогресс-бар и результат. Тесты покрывают фильтры и протокол сообщений.

Отдельно разберём, **чего демо не показывает** — SharedArrayBuffer, передачу `OffscreenCanvas` и пул воркеров. Это требует COOP/COEP-заголовков и отдельной инфраструктуры; тема достойна следующих глав. Здесь мы работаем с `postMessage` и module worker.

## 28.1. Main thread

Главный поток браузера отвечает за DOM, события, рендеринг. Если в нём выполнять тяжёлые вычисления (обработка пикселей, сортировка, парсинг), страница «замирает»: пользователь не может нажать кнопку, скролл не работает, анимация пропадает. Это классическая проблема CPU-intensive операций.

Правило проекта: **любая операция дольше 50 мс — кандидат на вынос в Worker**. 50 мс — это порог, после которого пользователь замечает «залипание».

## 28.2. Worker

`Worker` — фоновый поток, у которого нет доступа к DOM, но есть свой event loop, свои переменные и своя WASM-инстанция. Создаётся из главного потока:

```rust
let worker = Worker::new("./workers/worker.js")?;
```

Worker не может напрямую менять DOM или читать `window`. Общение — только через `postMessage`/`onmessage`. Каждый Worker — отдельный экземпляр WASM-модуля: если вы загрузили `.wasm` в главном потоке, в Worker его нужно загрузить заново.

## 28.3. Module worker

`wasm-pack build --target web` генерирует **ES-модуль** с `export default init` и именованными экспортами. `importScripts()` работает только с классическими скриптами и с ES-модулем падает на первой же строке. Поэтому для Worker'а, собранного `--target web`, обязателен **module worker**:

```rust
let options = WorkerOptions::new();
options.set_type(WorkerType::Module);
let worker = Worker::new_with_options("./workers/worker.js", &options)?;
```

А в самом `worker.js` — обычный `import`:

```js
import init, { grayscale, box_blur } from '/pkg/hello_wasm02.js';
```

Путь `/pkg/hello_wasm02.js` — это имя crate из `Cargo.toml`, а не «app.js» и не «worker_bg.js». Если сомневаетесь — `ls web/pkg/`.

## 28.4. Message passing

Сообщения передаются через `postMessage` и принимаются через `onmessage`. Данные сериализуются алгоритмом Structured Clone: объекты, массивы, `Blob`, `ArrayBuffer`, `ImageData` — всё копируется. Функции и DOM-узлы передать нельзя.

```rust
// Отправка
worker.post_message(&msg)?;

// Приём
let onmessage = Closure::wrap(Box::new(move |e: web_sys::MessageEvent| {
    let data = e.data();
    // обработать data
}) as Box<dyn FnMut(_)>);
worker.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
onmessage.forget();
```

Для больших бинарных данных используйте **Transferable** — `ArrayBuffer` можно передать без копирования:

```rust
let transfer = js_sys::Array::new();
let uint8 = js_sys::Uint8Array::new(&buffer);
transfer.push(&uint8.buffer());
worker.post_message_with_transfer(&msg, &transfer)?;
```

После передачи исходный `ArrayBuffer` в главном потоке становится пустым (`byteLength == 0`). Это не ошибка, а признак успешной передачи владения.

## 28.5. Обязательный `onerror`

Worker может упасть на этапе `import`, `init()` или в любой строке скрипта. Без `onerror` браузер шлёт `ErrorEvent` в пустоту, а вы видите «ничего не произошло»:

```rust
let onerror = Closure::wrap(Box::new(move |e: web_sys::ErrorEvent| {
    set_status(&status, &format!("Worker не запустился: {}", e.message()));
}) as Box<dyn FnMut(_)>);
worker.set_onerror(Some(onerror.as_ref().unchecked_ref()));
onerror.forget();
```

Правило проекта: **`set_onerror` — обязательная пара к `set_onmessage`**. Без неё отладка Worker'а превращается в археологию по пустому консольному выводу.

## 28.6. Shared state

У Worker нет общей памяти с главным потоком по умолчанию. Два способа разделить состояние:

1. **Обмен сообщениями** — основной способ. Главный поток держит авторитетное состояние, Worker получает данные, обрабатывает, возвращает.
2. **`SharedArrayBuffer` + `Atomics`** — общая память. Требует COOP/COEP-заголовков (`Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Embedder-Policy: require-corp`). В демо не используется, но знать о нём важно: именно он даёт настоящую параллельность без копирования.

Правило проекта: если два потока зависят от одного состояния, **держите состояние в главном потоке** и синхронизируйте через сообщения.

## 28.7. WASM Worker

WASM-модуль загружается внутри Worker'а. `web/public/workers/worker.js` — тонкая обёртка:

```js
import init, { grayscale, box_blur } from '/pkg/hello_wasm02.js';

let ready = false;
init().then(() => {
  ready = true;
  self.postMessage({ type: 'ready' });
});

self.onmessage = (e) => {
  if (!ready) { self.postMessage({ type: 'error', message: 'not ready' }); return; }
  // ...
};
```

Важно: экспортируемые функции должны быть помечены `#[wasm_bindgen]`:

```rust
#[wasm_bindgen]
pub fn grayscale(pixels: &[u8]) -> Vec<u8> { /* ... */ }

#[wasm_bindgen]
pub fn box_blur(pixels: &[u8], width: usize, height: usize, radius: usize) -> Vec<u8> { /* ... */ }
```

Без атрибута `grayscale` в JS не появится — `import { grayscale }` вернёт `undefined`, а вызов упадёт.

## 28.8. CPU-intensive operations

Типичные кандидаты на вынос в Worker:

- Обработка изображений (фильтры, ресайз, кодирование).
- Парсинг больших JSON/CSV/XML.
- Шифрование, хеширование.
- Сортировка больших массивов.
- Физические симуляции.

Все эти операции выигрывают от Worker'а, даже без `SharedArrayBuffer`: главный поток свободен, UI отвечает на события, анимация плавная.

## 28.9. UI responsiveness

Worker не делает UI отзывчивым сам по себе. Нужны три вещи:

1. **Прогресс.** Worker отправляет промежуточные сообщения (`{type: "progress", value: 42}`).
2. **Yield в главном потоке.** Если вы обновляете UI в цикле, отдавайте управление через `TimeoutFuture::new(0).await` или `requestAnimationFrame`.
3. **Троттлинг.** Прогресс обновлять не чаще, чем раз в 16 мс (≈ 60 Гц). Проверка `now - last_yield_ms >= 16.0` занимает доли микросекунды.

## 28.10. Практический worker pipeline

**Что делает.** Загружает изображение, отправляет пиксели в Worker, применяет фильтр, получает результат, отображает прогресс.

**Почему это полезно.** Это реальный сценарий: любое приложение с обработкой изображений (фоторедактор, сканер, генератор превью) делает именно так.

### Структура файлов

```
src/
  worker_core.rs       — создание Worker, обмен, parse_response
  worker_pipeline.rs   — фильтры (grayscale, box_blur), задача
  app.rs               — демо только этой главы
web/public/workers/
  worker.js            — module worker, загружает WASM
tests/
  worker_core_test.rs       — host-тесты фильтров
  worker_pipeline_test.rs   — wasm-тесты на больших данных
```

---

## Исходные коды новых модулей

### `Cargo.toml` — добавить features

```toml
[dependencies.web-sys]
features = [
    # … существующие …
    "Worker",
    "WorkerOptions",
    "WorkerType",
    "MessageEvent",
    "MessageEventInit",
    "DedicatedWorkerGlobalScope",
    "ImageData",
    "HtmlImageElement",
    "HtmlCanvasElement",
    "CanvasRenderingContext2d",
    "ErrorEvent",
    "console",
]
```

### `src/worker_core.rs`

```rust
//! Ядро Worker: создание, двусторонний обмен сообщениями.
//! Автономный модуль — не зависит от других модулей проекта.

use wasm_bindgen::prelude::*;
use web_sys::{MessageEvent, Worker, WorkerOptions};

/// Сообщение от Worker'а к главному потоку.
#[derive(Debug, Clone)]
pub enum WorkerResponse {
    /// Worker загрузил WASM и готов принимать задачи.
    Ready,
    /// Промежуточный прогресс (0..100).
    Progress(u32),
    /// Финальный результат.
    Done(Vec<u8>),
    /// Ошибка.
    Error(String),
}

/// Создаёт классический Worker по URL.
pub fn spawn(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
        .map_err(|e| JsValue::from_str(&format!("Worker::new: {e:?}")))
}

/// Создаёт Worker с опциями (для module worker).
pub fn spawn_with_options(url: &str, options: &WorkerOptions) -> Result<Worker, JsValue> {
    Worker::new_with_options(url, options)
        .map_err(|e| JsValue::from_str(&format!("Worker::new_with_options: {e:?}")))
}

/// Отправляет сообщение Worker'у.
pub fn send(worker: &Worker, msg: &JsValue) -> Result<(), JsValue> {
    worker
        .post_message(msg)
        .map_err(|e| JsValue::from_str(&format!("post_message: {e:?}")))
}

/// Отправляет сообщение с Transferable-объектами (без копирования).
pub fn send_transfer(
    worker: &Worker,
    msg: &JsValue,
    transfer: &js_sys::Array,
) -> Result<(), JsValue> {
    worker
        .post_message_with_transfer(msg, transfer)
        .map_err(|e| JsValue::from_str(&format!("post_message_with_transfer: {e:?}")))
}

/// Парсит MessageEvent в WorkerResponse.
pub fn parse_response(e: &MessageEvent) -> Option<WorkerResponse> {
    let data = e.data();
    let obj = data.dyn_into::<js_sys::Object>().ok()?;
    let type_val = js_sys::Reflect::get(&obj, &JsValue::from_str("type"))
        .ok()?
        .as_string()?;
    match type_val.as_str() {
        "ready" => Some(WorkerResponse::Ready),
        "progress" => {
            let value = js_sys::Reflect::get(&obj, &JsValue::from_str("value"))
                .ok()?
                .as_f64()? as u32;
            Some(WorkerResponse::Progress(value))
        }
        "done" => {
            let value = js_sys::Reflect::get(&obj, &JsValue::from_str("value")).ok()?;
            let array = js_sys::Uint8Array::new(&value).to_vec();
            Some(WorkerResponse::Done(array))
        }
        "error" => {
            let msg = js_sys::Reflect::get(&obj, &JsValue::from_str("message"))
                .ok()?
                .as_string()?;
            Some(WorkerResponse::Error(msg))
        }
        _ => None,
    }
}
```

### `src/worker_pipeline.rs`

```rust
//! Практический проект Главы 28: pipeline обработки изображений.
//!
//! `grayscale` и `box_blur` экспортированы в JS через `#[wasm_bindgen]`,
//! чтобы worker.js мог их вызвать как `grayscale(bytes)`.

use wasm_bindgen::prelude::*;

/// Фильтр обработки изображения.
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Filter {
    Grayscale,
    BoxBlur,
}

/// Применяет grayscale к RGBA-пикселям.
#[wasm_bindgen]
pub fn grayscale(pixels: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(pixels.len());
    let mut i = 0;
    while i + 3 < pixels.len() {
        let r = pixels[i] as u32;
        let g = pixels[i + 1] as u32;
        let b = pixels[i + 2] as u32;
        let a = pixels[i + 3];
        let gray = ((r * 299 + g * 587 + b * 114) / 1000) as u8;
        out.push(gray);
        out.push(gray);
        out.push(gray);
        out.push(a);
        i += 4;
    }
    out
}

/// Применяет box blur радиуса `radius` к RGBA-пикселям.
#[wasm_bindgen]
pub fn box_blur(pixels: &[u8], width: usize, height: usize, radius: usize) -> Vec<u8> {
    let mut out = pixels.to_vec();
    if radius == 0 || width == 0 || height == 0 {
        return out;
    }
    let r = radius as isize;
    for y in 0..height {
        for x in 0..width {
            let mut sum_r = 0u32;
            let mut sum_g = 0u32;
            let mut sum_b = 0u32;
            let mut sum_a = 0u32;
            let mut count = 0u32;
            for dy in -r..=r {
                for dx in -r..=r {
                    let nx = x as isize + dx;
                    let ny = y as isize + dy;
                    if nx < 0 || ny < 0 || nx >= width as isize || ny >= height as isize {
                        continue;
                    }
                    let idx = (ny as usize * width + nx as usize) * 4;
                    if idx + 3 >= pixels.len() {
                        continue;
                    }
                    sum_r += pixels[idx] as u32;
                    sum_g += pixels[idx + 1] as u32;
                    sum_b += pixels[idx + 2] as u32;
                    sum_a += pixels[idx + 3] as u32;
                    count += 1;
                }
            }
            if count == 0 {
                continue;
            }
            let idx = (y * width + x) * 4;
            out[idx] = (sum_r / count) as u8;
            out[idx + 1] = (sum_g / count) as u8;
            out[idx + 2] = (sum_b / count) as u8;
            out[idx + 3] = (sum_a / count) as u8;
        }
    }
    out
}

/// Описание задачи для Worker'а.
#[derive(Debug, Clone)]
pub struct PipelineTask {
    pub filter: Filter,
    pub width: u32,
    pub height: u32,
}

/// Создаёт JS-объект задачи для Worker'а.
pub fn task_to_js(task: &PipelineTask, pixels: &[u8]) -> Result<JsValue, JsValue> {
    let obj = js_sys::Object::new();
    let filter_str = match task.filter {
        Filter::Grayscale => "grayscale",
        Filter::BoxBlur => "box_blur",
    };
    js_sys::Reflect::set(&obj, &JsValue::from_str("filter"), &JsValue::from_str(filter_str))
        .map_err(|e| JsValue::from_str(&format!("set filter: {e:?}")))?;
    js_sys::Reflect::set(&obj, &JsValue::from_str("width"), &JsValue::from_f64(task.width as f64))
        .map_err(|e| JsValue::from_str(&format!("set width: {e:?}")))?;
    js_sys::Reflect::set(&obj, &JsValue::from_str("height"), &JsValue::from_f64(task.height as f64))
        .map_err(|e| JsValue::from_str(&format!("set height: {e:?}")))?;
    let uint8 = js_sys::Uint8Array::from(pixels);
    js_sys::Reflect::set(&obj, &JsValue::from_str("pixels"), &uint8)
        .map_err(|e| JsValue::from_str(&format!("set pixels: {e:?}")))?;
    Ok(obj.into())
}
```

### `web/public/workers/worker.js`

```js
// Module worker: pipeline обработки изображений.
// Загружает WASM, ждёт сообщений от главного потока.

import init, { grayscale, box_blur } from '/pkg/hello_wasm02.js';

let ready = false;

init()
  .then(() => {
    ready = true;
    self.postMessage({ type: 'ready' });
  })
  .catch((err) => {
    self.postMessage({ type: 'error', message: String(err) });
  });

self.onmessage = (e) => {
  if (!ready) {
    self.postMessage({ type: 'error', message: 'WASM not ready' });
    return;
  }

  const { filter, width, height, pixels } = e.data;

  self.postMessage({ type: 'progress', value: 0 });

  const bytes = new Uint8Array(pixels);
  let result;
  try {
    if (filter === 'grayscale') {
      result = grayscale(bytes);
      self.postMessage({ type: 'progress', value: 100 });
    } else if (filter === 'box_blur') {
      result = box_blur(bytes, width, height, 2);
      self.postMessage({ type: 'progress', value: 100 });
    } else {
      self.postMessage({ type: 'error', message: 'unknown filter' });
      return;
    }
  } catch (err) {
    self.postMessage({ type: 'error', message: String(err) });
    return;
  }

  self.postMessage({ type: 'done', value: result });
};
```

---

## Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod worker_core;
pub mod worker_pipeline;
```

---

## Тесты

### `tests/worker_core_test.rs` (host)

```rust
//! Host-тесты модуля worker_core: чистая логика фильтров.
//! Никаких JsValue/Worker здесь быть не должно.

use hello_wasm02::worker_pipeline::{box_blur, grayscale, Filter, PipelineTask};

#[test]
fn grayscale_converts_rgba() {
    let pixels = vec![255, 0, 0, 255, 0, 255, 0, 255];
    let gray = grayscale(&pixels);
    assert_eq!(gray.len(), pixels.len());
    assert!(gray[0] > 0 && gray[0] < 255);
    assert!(gray[4] > 0 && gray[4] < 255);
}

#[test]
fn grayscale_preserves_alpha() {
    let pixels = vec![100, 100, 100, 128];
    let gray = grayscale(&pixels);
    assert_eq!(gray[3], 128);
}

#[test]
fn box_blur_radius_zero_is_identity() {
    let pixels = vec![10, 20, 30, 40, 50, 60, 70, 80];
    let blurred = box_blur(&pixels, 2, 1, 0);
    assert_eq!(blurred, pixels);
}

#[test]
fn box_blur_smooths_edges() {
    let pixels = vec![0, 0, 0, 255, 255, 255, 255, 255, 0, 0, 0, 255];
    let blurred = box_blur(&pixels, 3, 1, 1);
    assert!(blurred[4] > 0 && blurred[4] < 255);
}

#[test]
fn filter_variants_are_distinct() {
    assert_ne!(Filter::Grayscale, Filter::BoxBlur);
}

#[test]
fn pipeline_task_construction() {
    let task = PipelineTask {
        filter: Filter::Grayscale,
        width: 100,
        height: 50,
    };
    assert_eq!(task.width, 100);
    assert_eq!(task.height, 50);
}
```

### `tests/worker_pipeline_test.rs` (wasm)

```rust
//! Wasm-тесты pipeline: проверка фильтров на реальных пикселях.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::worker_pipeline::{box_blur, grayscale};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn grayscale_on_large_image() {
    let mut pixels = vec![0u8; 100 * 100 * 4];
    for i in 0..100 * 100 {
        pixels[i * 4] = 200;
        pixels[i * 4 + 1] = 100;
        pixels[i * 4 + 2] = 50;
        pixels[i * 4 + 3] = 255;
    }
    let gray = grayscale(&pixels);
    assert_eq!(gray.len(), pixels.len());
    assert_eq!(gray[0], gray[4]);
}

#[wasm_bindgen_test]
fn box_blur_on_solid_color_is_identity() {
    let pixels = vec![128u8; 10 * 10 * 4];
    let blurred = box_blur(&pixels, 10, 10, 1);
    assert_eq!(blurred, pixels);
}

#[wasm_bindgen_test]
fn grayscale_performance_under_200ms() {
    let pixels = vec![128u8; 1000 * 1000 * 4];
    let start = js_sys::Date::now();
    let _ = grayscale(&pixels);
    let elapsed = js_sys::Date::now() - start;
    assert!(elapsed < 200.0, "grayscale 1M пикселей занял {elapsed} мс");
}
```

---

## Команды запуска

```bash
# Host-тесты
cargo test --test worker_core_test

# Wasm-тесты
wasm-pack test --headless --chrome --test worker_pipeline_test

# Сборка wasm
npm run wasm:dev

# Запуск демо
npm run dev
```

---

## Демо-пример в `app.rs`

```rust
//! Демо Главы 28: Worker и параллельное выполнение.
//!
//! Что показывает демо:
//!   • Создание module Worker из Rust.
//!   • Отправка пикселей изображения в Worker.
//!   • Прогресс-бар обновляется по сообщениям от Worker'а.
//!   • Главный поток остаётся отзывчивым (кнопки, анимация).
//!   • Фильтры: grayscale, box blur.
//!
//! Фокусный подход: только демо этой главы.
//!
//! ПРАВИЛО: каждое `Rc<RefCell<T>>` (и любой Clone-объект), попадающее
//! в `move`-замыкание, получает ОТДЕЛЬНЫЙ клон с суффиксом `_cb`.
//! Никаких `let x = x.clone();` (shadowing) — только `let x_cb = x.clone();`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::spawn_local;
use web_sys::{HtmlElement, HtmlInputElement, WorkerOptions, WorkerType};

use crate::worker_core::{self, WorkerResponse};
use crate::worker_pipeline::{self, Filter, PipelineTask};

const WORKER_URL: &str = "./workers/worker.js";

const HTML: &str = r#"
<h1>Глава 28: Worker и параллельное выполнение</h1>
<p>Обработка изображения в фоновом потоке с прогрессом.</p>

<fieldset style="margin:8px 0">
  <legend>Изображение</legend>
  <input id="img-url" placeholder="URL изображения" value="/assets/1.jpg" style="width:280px">
  <button id="load-img">Загрузить</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Фильтр</legend>
  <select id="filter">
    <option value="grayscale">Grayscale</option>
    <option value="box_blur">Box Blur (r=2)</option>
  </select>
  <button id="apply">Применить</button>
</fieldset>

<div style="margin:8px 0">
  <progress id="progress" value="0" max="100" style="width:320px"></progress>
  <span id="progress-text">0%</span>
</div>

<div id="status" style="color:#555;margin:8px 0">Загрузка Worker…</div>

<div style="display:flex;gap:16px;margin:8px 0;align-items:flex-start">
  <div>
    <div><small>Исходное</small></div>
    <canvas id="canvas-in" style="border:1px solid #ccc;max-width:320px;max-height:240px"></canvas>
  </div>
  <div>
    <div><small>Результат</small></div>
    <canvas id="canvas-out" style="border:1px solid #ccc;max-width:320px;max-height:240px"></canvas>
  </div>
</div>

<div style="margin:8px 0">
  <button id="ping">Проверить отзывчивость UI</button>
  <span id="ping-count">0</span>
</div>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let status = document.get_element_by_id("status").ok_or("no status")?;
    let progress: HtmlElement = document
        .get_element_by_id("progress")
        .ok_or("no progress")?
        .dyn_into()?;
    let progress_text = document.get_element_by_id("progress-text").ok_or("no pt")?;
    let ping_count = document.get_element_by_id("ping-count").ok_or("no pc")?;

    // Общее состояние. Каждое значение — отдельный Rc.
    let source_pixels: Rc<RefCell<Vec<u8>>> = Rc::new(RefCell::new(Vec::new()));
    let source_dims: Rc<RefCell<(u32, u32)>> = Rc::new(RefCell::new((0, 0)));
    let worker_ready: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));

    // Module worker: wasm-pack --target web даёт ES-модуль.
    let options = WorkerOptions::new();
    options.set_type(WorkerType::Module);
    let worker = worker_core::spawn_with_options(WORKER_URL, &options)?;

    // ---------------- onmessage ----------------
    {
        let status_cb = status.clone();
        let progress_cb = progress.clone();
        let progress_text_cb = progress_text.clone();
        let worker_ready_cb = worker_ready.clone();
        let source_dims_cb = source_dims.clone();
        let canvas_out = document
            .get_element_by_id("canvas-out")
            .ok_or("no canvas out")?;

        let onmessage = Closure::wrap(Box::new(move |e: web_sys::MessageEvent| {
            match worker_core::parse_response(&e) {
                Some(WorkerResponse::Ready) => {
                    *worker_ready_cb.borrow_mut() = true;
                    set_status(&status_cb, "Worker готов. Загрузите изображение.");
                }
                Some(WorkerResponse::Progress(v)) => {
                    let _ = progress_cb.set_attribute("value", &v.to_string());
                    progress_text_cb.set_text_content(Some(&format!("{v}%")));
                }
                Some(WorkerResponse::Done(bytes)) => {
                    let (w, h) = *source_dims_cb.borrow();
                    match draw_to_canvas(&canvas_out, &bytes, w, h) {
                        Ok(()) => set_status(&status_cb, "Готово"),
                        Err(e) => set_status(&status_cb, &format!("Ошибка рендера: {e:?}")),
                    }
                    let _ = progress_cb.set_attribute("value", "100");
                    progress_text_cb.set_text_content(Some("100%"));
                }
                Some(WorkerResponse::Error(msg)) => {
                    set_status(&status_cb, &format!("Ошибка Worker: {msg}"));
                }
                None => {}
            }
        }) as Box<dyn FnMut(_)>);
        worker.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    }

    // ---------------- onerror ----------------
    {
        let status_cb = status.clone();
        let onerror = Closure::wrap(Box::new(move |e: web_sys::ErrorEvent| {
            set_status(&status_cb, &format!("Worker не запустился: {}", e.message()));
        }) as Box<dyn FnMut(_)>);
        worker.set_onerror(Some(onerror.as_ref().unchecked_ref()));
        onerror.forget();
    }

    // ---------------- Кнопка «Загрузить изображение» ----------------
    let load_btn: HtmlElement = document
        .get_element_by_id("load-img")
        .ok_or("no load")?
        .dyn_into()?;
    {
        let source_pixels_cb = source_pixels.clone();
        let source_dims_cb = source_dims.clone();
        let status_cb = status.clone();
        let canvas_in = document
            .get_element_by_id("canvas-in")
            .ok_or("no canvas in")?;

        set_click(&load_btn, move |_| {
            let url = get_input("img-url");
            if url.is_empty() {
                return;
            }
            let source_pixels_cb = source_pixels_cb.clone();
            let source_dims_cb = source_dims_cb.clone();
            let status_cb = status_cb.clone();
            let canvas_in = canvas_in.clone();

            spawn_local(async move {
                set_status(&status_cb, "Загрузка…");
                match load_image_pixels(&url).await {
                    Ok((pixels, w, h)) => {
                        *source_pixels_cb.borrow_mut() = pixels.clone();
                        *source_dims_cb.borrow_mut() = (w, h);
                        match draw_to_canvas(&canvas_in, &pixels, w, h) {
                            Ok(()) => set_status(
                                &status_cb,
                                &format!("Изображение загружено ({w}×{h})"),
                            ),
                            Err(e) => set_status(
                                &status_cb,
                                &format!("Ошибка рендера: {e:?}"),
                            ),
                        }
                    }
                    Err(e) => set_status(
                        &status_cb,
                        &format!("Ошибка загрузки: {e:?}"),
                    ),
                }
            });
        });
    }

    // ---------------- Кнопка «Применить фильтр» ----------------
    let apply_btn: HtmlElement = document
        .get_element_by_id("apply")
        .ok_or("no apply")?
        .dyn_into()?;
    {
        let worker_cb = worker.clone();
        let source_pixels_cb = source_pixels.clone();
        let source_dims_cb = source_dims.clone();
        let status_cb = status.clone();
        let progress_cb = progress.clone();
        let progress_text_cb = progress_text.clone();
        let worker_ready_cb = worker_ready.clone();

        set_click(&apply_btn, move |_| {
            if !*worker_ready_cb.borrow() {
                set_status(&status_cb, "Worker ещё не готов — подождите");
                return;
            }
            let pixels = source_pixels_cb.borrow().clone();
            if pixels.is_empty() {
                set_status(&status_cb, "Сначала загрузите изображение");
                return;
            }
            let (w, h) = *source_dims_cb.borrow();
            let filter = match get_input("filter").as_str() {
                "box_blur" => Filter::BoxBlur,
                _ => Filter::Grayscale,
            };
            let task = PipelineTask {
                filter,
                width: w,
                height: h,
            };
            let msg = match worker_pipeline::task_to_js(&task, &pixels) {
                Ok(m) => m,
                Err(e) => {
                    set_status(&status_cb, &format!("Ошибка подготовки: {e:?}"));
                    return;
                }
            };
            if let Err(e) = worker_core::send(&worker_cb, &msg) {
                set_status(&status_cb, &format!("Ошибка отправки: {e:?}"));
                return;
            }
            let _ = progress_cb.set_attribute("value", "0");
            progress_text_cb.set_text_content(Some("0%"));
            set_status(&status_cb, "Обработка…");
        });
    }

    // ---------------- Кнопка «Проверить отзывчивость UI» ----------------
    let ping_btn: HtmlElement = document
        .get_element_by_id("ping")
        .ok_or("no ping")?
        .dyn_into()?;
    {
        let ping_count_cb = ping_count.clone();
        let count = Rc::new(RefCell::new(0u32));
        set_click(&ping_btn, move |_| {
            let mut c = count.borrow_mut();
            *c += 1;
            ping_count_cb.set_text_content(Some(&c.to_string()));
        });
    }

    Ok(())
}

// ---------------- helpers ----------------

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn get_input(id: &str) -> String {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlInputElement>().ok())
        .map(|i| i.value())
        .unwrap_or_default()
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

/// Рисует RGBA-пиксели на canvas.
///
/// `ImageData::new_with_u8_clamped_array_and_sh` требует
/// `pixels.len() == w * h * 4`. Размеры canvas выставляются под данные.
fn draw_to_canvas(
    canvas: &web_sys::Element,
    pixels: &[u8],
    w: u32,
    h: u32,
) -> Result<(), JsValue> {
    if w == 0 || h == 0 {
        return Err(JsValue::from_str("нулевые размеры"));
    }
    let expected = (w as usize) * (h as usize) * 4;
    if pixels.len() != expected {
        return Err(JsValue::from_str(&format!(
            "несоответствие размера: {} байт при {w}×{h} (ожидалось {expected})",
            pixels.len()
        )));
    }
    let canvas: web_sys::HtmlCanvasElement = canvas.clone().dyn_into()?;
    canvas.set_width(w);
    canvas.set_height(h);
    let ctx = canvas
        .get_context("2d")?
        .ok_or("no 2d context")?
        .dyn_into::<web_sys::CanvasRenderingContext2d>()?;
    let image_data = web_sys::ImageData::new_with_u8_clamped_array_and_sh(
        wasm_bindgen::Clamped(pixels),
        w,
        h,
    )?;
    ctx.put_image_data(&image_data, 0, 0)
}

/// Загружает изображение по URL и возвращает (RGBA-пиксели, ширина, высота).
async fn load_image_pixels(url: &str) -> Result<(Vec<u8>, u32, u32), JsValue> {
    let window = web_sys::window().ok_or("no window")?;
    let document = window.document().ok_or("no doc")?;
    let img = document
        .create_element("img")?
        .dyn_into::<web_sys::HtmlImageElement>()?;
    img.set_cross_origin(Some("anonymous"));
    img.set_src(url);

    let img_clone = img.clone();
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let onload = Closure::once_into_js(move || {
            let _ = resolve.call0(&JsValue::UNDEFINED);
        });
        let onerror = Closure::once_into_js(move || {
            let _ = reject.call1(
                &JsValue::UNDEFINED,
                &JsValue::from_str("img.onerror"),
            );
        });
        img_clone.set_onload(Some(onload.unchecked_ref()));
        img_clone.set_onerror(Some(onerror.unchecked_ref()));
    });
    wasm_bindgen_futures::JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("загрузка {url}: {e:?}")))?;

    let w = img.natural_width();
    let h = img.natural_height();
    if w == 0 || h == 0 {
        return Err(JsValue::from_str("изображение не загрузилось"));
    }

    let canvas = document
        .create_element("canvas")?
        .dyn_into::<web_sys::HtmlCanvasElement>()?;
    canvas.set_width(w);
    canvas.set_height(h);
    let ctx = canvas
        .get_context("2d")?
        .ok_or("no 2d")?
        .dyn_into::<web_sys::CanvasRenderingContext2d>()?;
    ctx.draw_image_with_html_image_element(&img, 0.0, 0.0)?;
    let data = ctx.get_image_data(0, 0, w as i32, h as i32)?;
    let pixels = data.data().to_vec();
    Ok((pixels, w, h))
}
```

---

## Анализ решения и реализованные улучшения

**Что работает хорошо:**

- Module worker загружает WASM в отдельном потоке, главный — рендерит.
- Прогресс обновляется в `<progress>` по сообщениям от Worker'а.
- Кнопка «Проверить отзывчивость UI» подтверждает: счётчик растёт во время обработки.
- Фильтры `grayscale`/`box_blur` экспортированы через `#[wasm_bindgen]` — вызываются из JS.
- `onerror` показывает реальные ошибки Worker'а в статусе.

**Реализованные улучшения:**

1. **Module worker** (`WorkerOptions { type_: Module }`) — согласовано с `--target web`.
2. **`onerror`** — ошибки инициализации видны пользователю, а не в пустоте.
3. **Флаг `worker_ready`** — кнопка «Применить» блокируется до `WorkerResponse::Ready`.
4. **`draw_to_canvas` с проверкой размеров** — `ImageData` не создаётся из несовпадающих данных; canvas подстраивается под пиксели.
5. **Суффикс `_cb` для клонов в замыканиях** — исключён класс ошибок `borrow of moved value`.
6. **`img.onerror` → `reject.call1`** — загрузка, которая упала, различима от успешной.
7. **Сообщение с размером** — «Изображение загружено (800×600)» вместо безликого «загружено».

**Что осталось за рамками главы (для следующих итераций):**

1. **Transferable** — пиксели копируются при `postMessage`. Для сотен мегабайт — заметно.
2. **Отмена** — кнопка «Отмена» отсутствует; для прерывания нужен `worker.terminate()` + новый Worker или `SharedArrayBuffer`.
3. **Прогресс внутри `box_blur`** — сейчас 0% и 100%. Разбиение по строкам — тема «chunked worker».
4. **Пул Worker'ов** — один Worker = одно ядро. Для параллельной обработки — отдельная глава.

---

## Правила проекта (закрепляем)

1. **CPU-нагрузка > 50 мс → Worker.** Порог, после которого пользователь замечает «залипание».
2. **`--target web` + Worker → `WorkerOptions { type_: Module }`.** `importScripts` с ES-модулем не работает.
3. **Имя выходного JS `wasm-pack` — это имя crate из `Cargo.toml`.** `/pkg/hello_wasm02.js`, а не `/pkg/app.js`.
4. **`onerror` — обязательная пара к `onmessage`.** Без неё ошибка Worker'а уходит в пустоту.
5. **`ImageData::new_with_u8_clamped_array_and_sh` требует `len == w*h*4`.** Размеры canvas выставлять под данные.
6. **`Rc<RefCell<T>>` в `move`-замыкание → отдельный клон `_cb`.** Никогда shadowing (`let x = x.clone();`) — только `let x_cb = x.clone();`.
7. **Функции, вызываемые из JS Worker'а, должны быть `#[wasm_bindgen]`.** Без атрибута `import { grayscale }` вернёт `undefined`.
8. **Worker не делает UI отзывчивым сам по себе.** Нужны прогресс, yield (при обновлениях UI в цикле) и троттлинг (60 Гц).
9. **Transferable для больших бинарных данных.** `post_message_with_transfer` перемещает владение без копирования.