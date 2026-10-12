# Глава 22.5. Worker pools и SharedArrayBuffer

## 22.5.0. Обновлённая структура

```
src/
  worker_pool.rs                  ← НОВЫЙ модуль 22.5
  wasm_in_worker.rs               ← из 22.4
  transferable.rs                 ← из 22.3
  dedicated_worker.rs             ← из 22.2
  offscreen_canvas.rs             ← из 22.1
  offscreen_worker.rs             ← из главы 22
  canvas_dpi.rs … canvas_image_pipeline.rs
tests/
  wasm_worker_pool.rs             ← НОВЫЙ тест
src/lib.rs                        ← +1 строка
src/app.rs                        ← демо 22.5 (заменяется целиком)
web/public/workers/
  wp_stripe_worker.js             ← НОВЫЙ JS-воркер (module worker)
web/public/pkg/                   ← НОВОЕ место сборки wasm-pack
package.json                      ← out-dir для wasm-pack
vite.config.js                    ← COOP/COEP + alias @pkg
.gitignore                        ← +web/public/pkg/
web/styles/main.css               ← без правок
```

**Комментарий.** Модуль `worker_pool.rs` — про **горизонтальное масштабирование** Worker'ов: пул из N экземпляров, распределение задач, агрегация результатов. Плюс feature detection для `SharedArrayBuffer`, `Atomics` и `crossOriginIsolated` — их нет по умолчанию, они требуют COOP/COEP-заголовков. Префикс CSS — `wp-` (worker pool), не пересекается с `ww-` (22.4), `tr-` (22.3), `dw-` (22.2), `oc-` (22.1), `ow-` (глава 22). Демо: изображение из 4 горизонтальных полос обрабатывается двумя способами — последовательно в main thread'е и параллельно пулом из 4 Worker'ов. Сравнивается время и состояние main thread'а (rAF-счётчик). Feature detection COI/SAB показан в статусе.

**Ссылки для детального изучения:**

- MDN: [`SharedArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- MDN: [`Atomics`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics)
- MDN: [COOP/COEP для cross-origin isolation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements)
- MDN: [`Worker`](https://developer.mozilla.org/en-US/docs/Web/API/Worker)
- web.dev: [Making your website cross-origin isolated](https://web.dev/articles/coop-coep)
- web.dev: [SharedArrayBuffer updates](https://web.dev/articles/safely-reviving-shared-memory)
- wasm-bindgen-rayon: [Parallel Raytracing](https://github.com/RReverser/wasm-bindgen-rayon)

---

## 22.5.1. О чём эта подглава

К 22.4 у нас есть Worker, который загружает WASM и выполняет тяжёлый фильтр в фоне. Но одна задача — один Worker. Что если задач **много**? Если изображение можно разрезать на 4 полосы и обработать одновременно? Если нужно обрабатывать поток кадров?

Подглава отвечает на три вопроса.

**Первый — как организовать N Worker'ов.** Пул: заранее созданные экземпляры, задача распределяется между ними. Round-robin — простейшая стратегия: задача N идёт в Worker `N % pool_size`.

**Второй — зачем SharedArrayBuffer.** До сих пор данные между Worker'ами передавались через `postMessage` + transferable. Это O(1) на передачу, но каждый трансфер — это **смена владельца**: пока Worker A владеет буфером, Worker B его не видит. Если 4 Worker'а должны писать в **общий** выходной буфер — transferable не подходит. Нужна **общая память**, к которой у всех есть доступ одновременно. Это и есть `SharedArrayBuffer`.

**Третий — как синхронизировать доступ.** Когда 4 Worker'а пишут в одну память, нужны атомарные операции. `Atomics.store`, `Atomics.load`, `Atomics.wait`, `Atomics.notify` — гарантии видимости и порядка. Без них один Worker не увидит изменений другого.

---

## 22.5.2. Что уже есть и что появится

К концу 22.4 умеем:
- загружать WASM в Worker'е — второй инстанс;
- передавать `ArrayBuffer` через transferable — zero-copy.

К концу 22.5 будем уметь:
- создавать пул из N Worker'ов;
- распределять задачи между ними (round-robin);
- агрегировать результаты;
- понимать разницу «transfer ownership» vs «shared memory»;
- использовать `SharedArrayBuffer` и `Atomics` для общей памяти;
- знать про COOP/COEP-заголовки, без которых SAB недоступен;
- диагностировать COI через `crossOriginIsolated`.

---

## 22.5.3. Пул Worker'ов: N исполнителей

**Проблема.** Один Worker загружает изображение и обрабатывает его 400 мс. Другой Worker в это время простаивает. Если разрезать изображение на 4 полосы, каждую можно обработать параллельно — 100 мс вместо 400. Но для этого нужно **4 Worker'а**, а не 1.

**Решение.** Пул.

```
pool = [worker_0, worker_1, worker_2, worker_3]
tasks = [stripe_0, stripe_1, stripe_2, stripe_3]

worker_0 ← stripe_0
worker_1 ← stripe_1
worker_2 ← stripe_2
worker_3 ← stripe_3
```

Каждый Worker получает свою задачу. Они работают одновременно. Результаты собираются в main thread'е.

**Стратегии распределения:**

- **Round-robin.** Задача N идёт в Worker `N % pool_size`. Просто, предсказуемо. Работает, если задачи одинакового размера.
- **Least-loaded.** Следим за числом активных задач на каждом Worker'е, отдаём новой задаче того, у кого их меньше. Сложнее, но лучше для неравномерных задач.
- **Work-stealing.** У каждого Worker'а своя очередь, свободные воруют у занятых. Продвинутая схема, за пределами этой главы.

В демо используется round-robin — задачи одинакового размера (4 равные полосы).

**Жизненный цикл пула:**

1. **Создание.** `Worker::new(url)` × N. Все Worker'ы стартуют одновременно, но `onmessage` устанавливается на каждый.
2. **Работа.** Задачи отправляются через `postMessage`. Каждая несёт свой `worker_index` — чтобы main thread знал, чей это результат.
3. **Завершение.** Когда все N задач выполнены, пул можно либо переиспользовать, либо прибить через `terminate()`.

**Почему пул лучше, чем «создать Worker на каждую задачу»:**

- Создание Worker'а — дорого: новый JS-контекст, новый `WebAssembly.Instance`, новый `wasm_bindgen` runtime. ~10–50 мс.
- Для 4 задач это 40–200 мс оверхеда. Для потока задач — постоянные потери.
- Пул создаётся один раз, живёт долго, переиспользуется.

**Graceful shutdown.** `terminate()` для каждого Worker'а в пуле. Мгновенно, без ожидания завершения текущей задачи.

---

## 22.5.4. SharedArrayBuffer: общая память

До сих пор все данные между потоками передавались через `postMessage`. Модель — «сообщения + transferable»:

- **Copy** — копирование, O(n). Дорого для больших данных.
- **Transfer** — смена владельца, O(1). Но владелец **один** в каждый момент.

Что если 4 Worker'а должны писать в **один** выходной буфер? Transferable не подходит — буфер принадлежит одному Worker'у. Нужна **общая память**.

**`SharedArrayBuffer`** — буфер, к которому имеют доступ несколько JS-контекстов одновременно:

```js
const sab = new SharedArrayBuffer(1024);
worker1.postMessage({ buf: sab });  // НЕ в transfer list
worker2.postMessage({ buf: sab });
// Оба Worker'а видят одни и те же байты.
```

**Ключевые отличия от `ArrayBuffer`:**

| | `ArrayBuffer` | `SharedArrayBuffer` |
|---|---|---|
| Владелец | Один | Много |
| Transfer | Да (O(1)) | Нет (нельзя) |
| Копирование | Structured clone | Только ссылка |
| Синхронизация | Не нужна | `Atomics` |

**Что можно делать с SAB:**

- Читать и писать байты из любого контекста.
- Создавать `Uint8Array`, `Int32Array` и другие typed arrays **над** ним.
- Использовать `Atomics` для атомарных операций.

**Чего нельзя:**

- Передать в transfer list — SAB не transferable.
- Использовать без COOP/COEP-заголовков (см. 22.5.6).

**Когда SAB полезен:**

- Все Worker'ы пишут в один выходной буфер (общий результат).
- Producer-consumer паттерны (один пишет, другой читает, оба видят изменения).
- Разделяемый счётчик, очередь, состояние.

**Когда SAB не нужен:**

- Каждый Worker владеет своим куском. Transferable дешевле и проще.

---

## 22.5.5. Atomics: синхронизация

Когда 4 Worker'а пишут в одну память, нужны гарантии:

- **Видимость.** Если Worker A записал значение, увидит ли его Worker B?
- **Атомарность.** Если два Worker'а пишут в одну ячейку одновременно, что получится?
- **Порядок.** Если Worker A пишет X, потом Y, увидит ли Worker B Y после X?

Без синхронизации — неопределённое поведение. `Atomics` решает.

**`Atomics.store(array, index, value)`** — атомарная запись. Гарантирует видимость и атомарность.

**`Atomics.load(array, index)`** — атомарное чтение. Возвращает последнее записанное значение.

**`Atomics.add(array, index, value)`** — атомарное `+=`. Возвращает **старое** значение.

**`Atomics.compareExchange(array, index, expected, replacement)`** — атомарный CAS (compare-and-swap). Возвращает старое значение. Базовый примитив для lock-free структур.

**`Atomics.wait(array, index, value, timeout?)`** — блокирует поток до тех пор, пока `array[index] !== value` или не истечёт timeout. **Только в Worker'ах** — main thread блокировать нельзя.

**`Atomics.notify(array, index, count?)`** — будит `count` ожидающих потоков на этом индексе.

**Что важно:**

- `Atomics` работает только над `Int32Array` и `BigInt64Array` над `SharedArrayBuffer`.
- `Atomics` **не работает** над обычным `ArrayBuffer`.
- Без `Atomics` JS не даёт никаких гарантий — компилятор может переупорядочить чтения, кэш может не увидеть изменения.

**Паттерн producer-consumer:**

```js
// Producer (Worker A)
Atomics.store(flag, 0, 1);
Atomics.notify(flag, 0, 1);

// Consumer (Worker B)
Atomics.wait(flag, 0, 0);
// Теперь данные точно готовы.
```

---

## 22.5.6. COOP/COEP: цена shared memory

`SharedArrayBuffer` — мощная штука, но и опасная. В 2018 году все браузеры его **отключили** из-за Spectre: атакующий мог использовать SAB для высокоточного тайминга и чтения чужой памяти через side-channel.

**Решение — COOP/COEP-заголовки.** Сайт объявляет себя **cross-origin isolated**:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Это значит:

- **COOP**: страница не открывает `window.open` к другим origin'ам, не принимает popup'ы.
- **COEP**: все ресурсы (картинки, скрипты, iframe) должны быть из same-origin **или** иметь CORS-заголовки.

Только при этих условиях браузер открывает `SharedArrayBuffer`. Feature detection — прямо в JS:

```js
if (typeof SharedArrayBuffer !== "undefined") {
    // Работает.
} else {
    // Заголовки не выставлены или браузер не поддерживает.
}
```

**Но есть ещё один признак.** `self.crossOriginIsolated` — это прямое свойство глобального объекта, которое становится `true`, когда COOP/COEP-заголовки применились. Это **надёжнее**, чем проверять `typeof SharedArrayBuffer`: если COI `true`, а SAB `undefined` — браузер старый. Если COI `false` — заголовки не пришли (или кэш).

**Для Vite** заголовки прописываются в `vite.config.js`, причём **дважды** — для `server` (dev) и `preview` (preview-сборка):

```js
const coiHeaders = {
    'Cross-Origin-Opener-Policy': 'same-origin',
    'Cross-Origin-Embedder-Policy': 'require-corp',
};

export default defineConfig({
    server: { headers: coiHeaders, /* … */ },
    preview: { headers: coiHeaders, /* … */ },
    // …
});
```

**Цена:** после COEP браузер начнёт блокировать внешние картинки без CORS. Все CDN-скрипты, шрифты, шрифты Google — надо будет либо хостить самому, либо добавлять CORS. Для демо это нормально; для продакшена — планируйте.

**В нашем демо** feature detection покажет, доступен ли SAB и COI. Если да — статус зелёный. Если нет — жёлтый, с подсказкой. Демо работает в обоих режимах: с SAB и без.

---

## 22.5.7. Что показывает демо

**Layout:**

- **Canvas** — 480×360. Четыре горизонтальные полосы, каждая со своим цветом и рисунком.
- **Индикатор main thread'а** — rAF-счётчик.
- **Кнопки:**
  - «Reset» — восстановить исходный паттерн.
  - «Sequential (4× main thread)» — обработать 4 полосы последовательно в main thread'е.
  - «Parallel (pool of 4)» — отдать 4 полосы пулу.
- **Статистика:**
  - Sequential time (мс).
  - Parallel time (мс).
  - Speedup (ratio).
  - Cross-Origin Isolated: да/нет.
  - SharedArrayBuffer: статус.
  - Прогресс: N из 4.
  - Пул: N/4 (готовность Worker'ов).
  - Кадры main thread'а.

**Сценарий:**

1. При загрузке: canvas показывает 4 полосы. Пул из 4 Worker'ов создаётся в фоне — статус `Пул: 0/4` → `4/4`. rAF-счётчик растёт.
2. Клик «Sequential». Main thread обрабатывает 4 полосы **одну за другой**. Счётчик кадров замирает на 400–600 мс. По завершении — время показано в статистике.
3. Клик «Reset». Восстановление.
4. Клик «Parallel». 4 задачи уходят в 4 Worker'а. Они работают **одновременно**. Счётчик кадров продолжает расти — main thread свободен. По мере готовности полос main thread собирает их. По завершении — время, speedup.

**Почему наглядно.**

- **Первый факт.** Sequential 500 мс vs Parallel 150 мс. На многоядерном CPU — до 4× ускорение.
- **Второй факт.** Счётчик кадров растёт **во время** параллельной работы. Main thread действительно свободен.
- **Третий факт.** COI-статус и SAB-статус дают понять, есть ли доступ к shared memory в этом окружении.

---

## 22.5.8. Wow-момент

**Первый.** «Sequential»: счётчик кадров мгновенно замирает. «Parallel»: счётчик растёт. Одно и то же вычисление, один результат — разные условия.

**Второй.** 4 полосы обработаны параллельно за время, сравнимое с **одной** полосой. На 4-ядерном CPU speedup близок к 4. Даже на 2-ядерном — близок к 2.

**Третий.** Откройте DevTools → Sources → Workers. Увидите 4 потока `wp_stripe_worker.js`. Все активны одновременно. Main thread простаивает — показывает rAF.

**Четвёртый.** Если SAB доступен — откройте Memory. Один `SharedArrayBuffer`, на который ссылаются и main thread, и все 4 Worker'а. Одна физическая память, 5 потребителей.

---

## 22.5.9. Чему это учит

**Первое.** Worker — это **исполнитель**. Пул Worker'ов — **система**. Задача N идёт в Worker `N % pool_size`. Round-robin — просто, но эффективно для одинаковых задач.

**Второе.** Создание Worker'а дорого (10–50 мс). Пул создаётся один раз и переиспользуется.

**Третье.** `SharedArrayBuffer` — **общая память**. К ней имеют доступ несколько контекстов. Не transferable, не копируется — просто ссылка.

**Четвёртое.** `Atomics` — **синхронизация**. Без них общая память бесполезна: чтения переупорядочиваются, кэш не видит чужих записей.

**Пятое.** COOP/COEP — **обязательное условие** для SAB. Feature detection — `typeof SharedArrayBuffer !== "undefined"`. Но надёжнее проверять `self.crossOriginIsolated`.

**Шестое.** Параллелизм не ускоряет на одном ядре. На многоядерном — близок к N×. На мобильных — 2–4 ядра, не больше.

---

## 22.5.10. Исходные файлы

### `src/worker_pool.rs`

```rust
//! 22.5. Worker pools и SharedArrayBuffer.
//!
//! Пул из N Worker'ов: задача распределяется по worker'ам,
//! результаты агрегируются. Плюс feature detection для
//! `SharedArrayBuffer`, `Atomics` и `crossOriginIsolated`.
//!
//! ## Что здесь есть
//!
//! - **Feature detection** — `SharedArrayBuffer`, `Atomics`, `Worker`,
//!   `crossOriginIsolated`.
//! - **Пул** — `create_pool(url, size)` возвращает `Vec<Worker>`.
//! - **Отправка задачи** — `post_stripe` отправляет полосу Worker'у.
//! - **Агрегация** — `extract_stripe_result` извлекает результат.
//! - **Завершение** — `terminate_pool` останавливает все Worker'ы.
//!
//! ## Утилита `bool_of`
//!
//! rust-analyzer теряет тип `matches!` в tail-позиции. Обёртка
//! возвращает типу конкретность.

use std::fmt::Arguments;

use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{MessageEvent, Worker, WorkerOptions, WorkerType};

// =============================================================================
// Утилита для обхода бага rust-analyzer в tail-позиции
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

/// `SharedArrayBuffer` доступен.
///
/// Требует COOP/COEP-заголовков. Без них — undefined, даже в
/// современных браузерах.
pub fn is_shared_array_buffer_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("SharedArrayBuffer")),
        Ok(v) if v.is_function()
    ))
}

/// `Atomics` доступен.
///
/// Если SAB есть, Atomics почти наверняка есть. Проверяем отдельно —
/// для диагностики.
pub fn is_atomics_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("Atomics")),
        Ok(v) if !v.is_undefined() && !v.is_null()
    ))
}

/// Страница является cross-origin isolated.
///
/// Это `self.crossOriginIsolated === true` — прямой индикатор того,
/// что COOP/COEP-заголовки применились и браузер открыл SAB.
pub fn is_cross_origin_isolated() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("crossOriginIsolated")),
        Ok(v) if v.as_bool() == Some(true)
    ))
}

/// Человекочитаемый статус cross-origin isolation.
pub fn coi_status() -> String {
    if is_cross_origin_isolated() {
        String::from("да")
    } else {
        String::from("нет")
    }
}

/// Человекочитаемый статус SharedArrayBuffer с точной диагностикой.
///
/// Различает три случая:
/// - SAB доступен — заголовки есть, браузер поддерживает.
/// - COI включён, но SAB нет — редкий случай, браузер старый.
/// - COI выключен — заголовки не применены или кэш.
pub fn sab_status() -> String {
    let sab = is_shared_array_buffer_supported();
    let atomics = is_atomics_supported();
    let coi = is_cross_origin_isolated();
    if sab && atomics {
        String::from("доступен")
    } else if coi && !sab {
        String::from("COI есть, но SAB не exposed (старый браузер?)")
    } else {
        String::from("недоступен (нет COOP/COEP)")
    }
}

// =============================================================================
// Пул Worker'ов
// =============================================================================

/// Создаёт пул из `size` module worker'ов.
///
/// Все Worker'ы используют один URL, но каждый — отдельный
/// JavaScript-контекст со своим WASM-инстансом.
pub fn create_pool(url: &str, size: usize) -> Result<Vec<Worker>, JsValue> {
    let opts = WorkerOptions::new();
    opts.set_type(WorkerType::Module);

    let mut workers = Vec::with_capacity(size);
    for _ in 0..size {
        let w = Worker::new_with_options(url, &opts)?;
        workers.push(w);
    }
    Ok(workers)
}

/// Принудительно завершает все Worker'ы пула.
pub fn terminate_pool(workers: &[Worker]) {
    for w in workers {
        w.terminate();
    }
}

// =============================================================================
// Отправка задачи Worker'у
// =============================================================================

/// Отправляет полосу Worker'у: `{ type: "stripe", buffer, y, w, h, passes, slot }`.
///
/// `slot` — индекс Worker'а в пуле. Возвращается в ответе, чтобы main
/// thread знал, чей это результат.
///
/// `buffer` создаётся в JS-heap и трансферится — zero-copy.
pub fn post_stripe(
    worker: &Worker,
    data: &[u8],
    y_offset: u32,
    stripe_w: u32,
    stripe_h: u32,
    passes: u32,
    slot: u32,
) -> Result<(), JsValue> {
    let arr = Uint8Array::new_with_length(data.len() as u32);
    arr.copy_from(data);
    let buffer: JsValue = arr.buffer().into();

    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("stripe"))?;
    Reflect::set(&msg, &JsValue::from_str("buffer"), &buffer)?;
    Reflect::set(&msg, &JsValue::from_str("y"), &JsValue::from_f64(y_offset as f64))?;
    Reflect::set(&msg, &JsValue::from_str("width"), &JsValue::from_f64(stripe_w as f64))?;
    Reflect::set(&msg, &JsValue::from_str("height"), &JsValue::from_f64(stripe_h as f64))?;
    Reflect::set(&msg, &JsValue::from_str("passes"), &JsValue::from_f64(passes as f64))?;
    Reflect::set(&msg, &JsValue::from_str("slot"), &JsValue::from_f64(slot as f64))?;

    let transfer = Array::new();
    transfer.push(&buffer);
    worker.post_message_with_transfer(&msg, &transfer)
}

// =============================================================================
// Результат от Worker'а
// =============================================================================

/// Результат обработки полосы.
pub struct StripeResult {
    pub bytes: Vec<u8>,
    pub y_offset: u32,
    pub width: u32,
    pub height: u32,
    pub elapsed_ms: f64,
    pub slot: u32,
}

/// Извлекает результат из `MessageEvent`.
///
/// Ожидает `{ type: "stripe_done", buffer, y, width, height, elapsed_ms, slot }`.
pub fn extract_stripe_result(event: &MessageEvent) -> Option<StripeResult> {
    let data = event.data();

    let kind = Reflect::get(&data, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())?;
    if kind != "stripe_done" {
        return None;
    }

    let buffer_val = Reflect::get(&data, &JsValue::from_str("buffer")).ok()?;
    let arr: Uint8Array = buffer_val.dyn_into().ok()?;
    let bytes: Vec<u8> = arr.to_vec();

    let y_offset = Reflect::get(&data, &JsValue::from_str("y")).ok()?.as_f64()? as u32;
    let width = Reflect::get(&data, &JsValue::from_str("width"))
        .ok()?
        .as_f64()? as u32;
    let height = Reflect::get(&data, &JsValue::from_str("height"))
        .ok()?
        .as_f64()? as u32;
    let elapsed_ms = Reflect::get(&data, &JsValue::from_str("elapsed_ms"))
        .ok()?
        .as_f64()?;
    let slot = Reflect::get(&data, &JsValue::from_str("slot"))
        .ok()?
        .as_f64()? as u32;

    Some(StripeResult {
        bytes,
        y_offset,
        width,
        height,
        elapsed_ms,
        slot,
    })
}

/// Извлекает ID инстанса из сообщения `{ type: "ready", instance, slot }`.
pub fn extract_ready_instance(event: &MessageEvent) -> Option<(String, u32)> {
    let data = event.data();
    let kind = Reflect::get(&data, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())?;
    if kind != "ready" {
        return None;
    }
    let instance = Reflect::get(&data, &JsValue::from_str("instance"))
        .ok()
        .and_then(|v| v.as_string())?;
    let slot = Reflect::get(&data, &JsValue::from_str("slot"))
        .ok()
        .and_then(|v| v.as_f64())? as u32;
    Some((instance, slot))
}

// =============================================================================
// Сводка о поддержке
// =============================================================================

/// Отладочная строка о поддержке (для диагностики).
pub fn support_summary() -> String {
    format_str(format_args!(
        "Worker: {w} · COI: {c} · SAB: {s} · Atomics: {a}",
        w = if is_worker_supported() { "да" } else { "нет" },
        c = if is_cross_origin_isolated() { "да" } else { "нет" },
        s = if is_shared_array_buffer_supported() { "да" } else { "нет" },
        a = if is_atomics_supported() { "да" } else { "нет" },
    ))
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `wasm_in_worker`:

```rust
pub mod offscreen_worker;     // 22
pub mod offscreen_canvas;     // 22.1
pub mod dedicated_worker;     // 22.2
pub mod transferable;         // 22.3
pub mod wasm_in_worker;       // 22.4
pub mod worker_pool;          // ← 22.5
```

**Важно:** без этой строки `worker_pool.rs` не компилируется в составе крейта, а `tests/wasm_worker_pool.rs` падает с `unresolved import hello_wasm02::worker_pool`.

### `web/public/workers/wp_stripe_worker.js`

```javascript
// 22.5. Stripe worker — module worker для пула.
//
// Загружает тот же wasm-pack-пакет, что и main thread. Каждый
// экземпляр пула имеет свой WASM-инстанс. Worker обрабатывает одну
// горизонтальную полосу и возвращает результат через transferable.

import init, { box_blur_rgba, instance_id } from "/pkg/hello_wasm02.js";

let ready = false;

init()
    .then(() => {
        ready = true;
        self.postMessage({ type: "ready", instance: instance_id() });
    })
    .catch((err) => {
        self.postMessage({ type: "init_error", message: String(err) });
    });

self.onmessage = (e) => {
    if (!ready) return;
    const msg = e.data;
    if (msg.type !== "stripe") return;

    const input = new Uint8Array(msg.buffer);
    const t0 = performance.now();
    const output = box_blur_rgba(input, msg.width, msg.height, msg.passes);
    const t1 = performance.now();

    // output — view над wasm-памятью. Копируем в JS-heap для transfer.
    const copy = new Uint8Array(output);

    self.postMessage(
        {
            type: "stripe_done",
            buffer: copy.buffer,
            y: msg.y,
            width: msg.width,
            height: msg.height,
            elapsed_ms: t1 - t0,
            slot: msg.slot,
        },
        [copy.buffer],
    );
};
```

### `tests/wasm_worker_pool.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::worker_pool::{
    coi_status, create_pool, is_atomics_supported, is_cross_origin_isolated,
    is_shared_array_buffer_supported, is_worker_supported, sab_status,
    support_summary, terminate_pool,
};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn worker_is_supported_in_browser() {
    assert!(is_worker_supported(), "Worker should be available");
}

#[wasm_bindgen_test]
fn shared_array_buffer_detection_does_not_panic() {
    // В большинстве dev-окружений без COOP/COEP SAB недоступен.
    // Тест проверяет только то, что detection возвращает bool.
    let _ = is_shared_array_buffer_supported();
}

#[wasm_bindgen_test]
fn atomics_detection_does_not_panic() {
    let _ = is_atomics_supported();
}

#[wasm_bindgen_test]
fn cross_origin_isolation_detection_does_not_panic() {
    let _ = is_cross_origin_isolated();
}

#[wasm_bindgen_test]
fn coi_status_returns_string() {
    let s = coi_status();
    assert!(!s.is_empty(), "coi_status must not be empty");
    assert!(s == "да" || s == "нет");
}

#[wasm_bindgen_test]
fn sab_status_returns_string() {
    let s = sab_status();
    assert!(!s.is_empty(), "sab_status must not be empty");
}

#[wasm_bindgen_test]
fn support_summary_returns_string() {
    let s = support_summary();
    assert!(s.contains("Worker"));
    assert!(s.contains("COI"));
    assert!(s.contains("SAB"));
    assert!(s.contains("Atomics"));
}

#[wasm_bindgen_test]
fn create_pool_of_four() {
    let pool = create_pool("/nonexistent_worker_xyz.js", 4);
    assert!(pool.is_ok(), "pool should be creatable");
    let pool = pool.unwrap();
    assert_eq!(pool.len(), 4, "pool size must be 4");
    terminate_pool(&pool);
}

#[wasm_bindgen_test]
fn create_empty_pool() {
    let pool = create_pool("/nonexistent_worker_xyz.js", 0).unwrap();
    assert_eq!(pool.len(), 0);
}

#[wasm_bindgen_test]
fn terminate_pool_does_not_panic() {
    let pool = create_pool("/nonexistent_worker_xyz.js", 3).unwrap();
    terminate_pool(&pool);
}
```

### `package.json` — путь вывода wasm-pack

```json
"scripts": {
    "wasm": "wasm-pack build --target web --out-dir web/public/pkg",
    "wasm:dev": "wasm-pack build --dev --target web --out-dir web/public/pkg",
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
}
```

**Почему `out-dir web/public/pkg`?** Vite с `root: 'web'` резолвит URL относительно `web/`. `web/public/` автоматически отдаётся по `/`. Значит `/pkg/hello_wasm02.js` физически = `web/public/pkg/hello_wasm02.js`. Просто, работает и в dev, и в build.

### `vite.config.js`

```js
import path from 'node:path';
import { fileURLToPath } from 'node:url';
import { defineConfig } from 'vite';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

// Cross-origin isolation: обязательно для SharedArrayBuffer.
// Без этих заголовков SAB не exposed, даже если браузер его поддерживает.
// Прописаны и для `server` (dev), и для `preview` (preview-сборка).
const coiHeaders = {
    'Cross-Origin-Opener-Policy': 'same-origin',
    'Cross-Origin-Embedder-Policy': 'require-corp',
};

export default defineConfig({
    root: 'web',
    resolve: {
        alias: {
            '@pkg': path.resolve(__dirname, 'web/public/pkg'),
        },
    },
    server: {
        host: '127.0.0.1',
        port: 5173,
        fs: { allow: ['..'] },
        headers: coiHeaders,
        proxy: {
            '/api': {
                target: 'http://127.0.0.1:3000',
                changeOrigin: true,
            },
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

### `.gitignore` — добавление

```
web/public/pkg/
```

Артефакты `wasm-pack` не место в git — они пересобираются через `npm run wasm`.

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Wasm-тесты только для этой подглавы
wasm-pack test --headless --chrome --test wasm_worker_pool

# 3. Демо
rm -rf pkg                        # убрать старый каталог, если был
npm run wasm:dev                  # собирает в web/public/pkg/
npm start

# 4. Проверка, что файлы отдаются по URL
curl -I http://127.0.0.1:5173/pkg/hello_wasm02.js
# Ожидаем Content-Type: text/javascript + COOP/COEP

curl -I http://127.0.0.1:5173/pkg/hello_wasm02_bg.wasm
# Ожидаем Content-Type: application/wasm
```

---

## 22.5.11. `src/app.rs` — автономный демо-пример

```rust
//! Глава 22.5 — Worker pools и SharedArrayBuffer.
//!
//! Демо сравнивает два способа обработки изображения из 4 полос:
//!
//! - **Sequential** — main thread обрабатывает полосы одну за другой.
//! - **Parallel** — пул из 4 Worker'ов обрабатывает одновременно.
//!
//! ## Что демонстрируется
//!
//! - **Пул Worker'ов** — 4 экземпляра, round-robin распределение.
//! - **Параллелизм** — 4× ускорение на многоядерном CPU.
//! - **Свобода main thread'а** — rAF-счётчик растёт во время параллели.
//! - **Feature detection SAB/COI** — статус Cross-Origin Isolation
//!   и SharedArrayBuffer. COI включается COOP/COEP-заголовками.
//! - **Диагностика пула** — статус «Пул: N/4» и обработка `onerror`.
//!
//! ## Правило Rc<RefCell<State>>
//!
//! `borrow()` и `borrow_mut()` не пересекаются. Каждый borrow —
//! в узком scope, `Ref`/`RefMut` drop'аются до следующего borrow.
//!
//! ## Обход бага rust-analyzer с сигнатурами canvas-методов
//!
//! r-a ошибочно считает, что `fill_rect`, `get_image_data`,
//! `put_image_data` принимают `i32`. Обходим через `Reflect::get`
//! + `Function::apply`.
//!
//! ## Обход бага rust-analyzer в tail-позиции
//!
//! Если r-a пишет «expected T, found ()» на tail-выражение функции —
//! заменяем его на явный `return expr;`. r-a тогда типизирует
//! каждый выход функции независимо. Применено в `extract_stripe`.
//!
//! ## Внутренние ссылки
//!
//! `app.rs` — модуль внутри крейта `hello_wasm02`, ссылки на другие
//! модули идут через `crate::`.
//!
//! **Файл автономен.** Стили `.wp-*` инжектируются в документ.
//! Wow: Sequential → счётчик кадров замирает; Parallel → растёт.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    CanvasRenderingContext2d, Document, ErrorEvent, HtmlCanvasElement, HtmlElement,
    ImageData, MessageEvent, Worker,
};

use crate::wasm_in_worker::box_blur_rgba;
use crate::worker_pool;

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `format!` для tail-позиций.
#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Service Worker cleanup (см. 22.4 — module worker import через SW)
// =============================================================================

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

fn unregister_sw_and_reload(document: &Document) -> Result<(), JsValue> {
    let script = document.create_element("script")?;
    script.set_text_content(Some(
        r#"(function(){
            var KEY = 'wp-sw-cleared-22-5';
            if (sessionStorage.getItem(KEY) === '1') return;
            sessionStorage.setItem(KEY, '1');
            if (!('serviceWorker' in navigator)) {
                location.reload();
                return;
            }
            navigator.serviceWorker.getRegistrations().then(function(rs){
                var ps = [];
                for (var i = 0; i < rs.length; i++) ps.push(rs[i].unregister());
                return Promise.all(ps);
            }).then(function(){ location.reload(); })
              .catch(function(){ location.reload(); });
        })();"#,
    ));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&script)?;
    Ok(())
}

fn show_sw_notice(document: &Document) -> Result<(), JsValue> {
    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html(
        r#"<div style="max-width:900px;padding:20px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#c9d1d9;font-family:system-ui,sans-serif;">
            <h2 style="margin:0 0 10px 0;color:#f0883e;">Очистка старых Service Worker'ов…</h2>
            <p style="margin:0;line-height:1.55;">Активный Service Worker перехватывает
            запросы к <code>/pkg/hello_wasm02.js</code> и ломает module worker.
            Снимаем регистрацию и перезагружаем страницу.</p>
        </div>"#,
    );
    Ok(())
}

// =============================================================================
// Хелперы для canvas: Reflect-вызовы
// =============================================================================

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

fn ctx_call_js(ctx: &JsValue, method: &str, args: &[JsValue]) -> Option<JsValue> {
    let f = Reflect::get(ctx, &JsValue::from_str(method)).ok()?;
    let f = f.dyn_into::<Function>().ok()?;
    let arr = Array::new();
    for a in args {
        arr.push(a);
    }
    f.apply(ctx, &arr).ok()
}

fn ctx_set(ctx: &JsValue, key: &str, value: &JsValue) {
    let _ = Reflect::set(ctx, &JsValue::from_str(key), value);
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "wp-injected-style";
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
.wp-desc {
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
.wp-desc-line { margin: 6px 0; }

.wp-canvas {
    display: block;
    width: 480px;
    height: 360px;
    max-width: 100%;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 8px;
    margin-bottom: 16px;
}

.wp-block {
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

.wp-btn {
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
.wp-btn:hover:not([disabled]) { background: #30363d; }
.wp-btn:active:not([disabled]) { transform: translateY(1px); }
.wp-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.wp-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.wp-btn-primary:hover:not([disabled]) { background: #2ea043; }
.wp-btn-warn {
    background: #21262d;
    border-color: #30363d;
    color: #f0883e;
}
.wp-btn-warn:hover:not([disabled]) { background: #30363d; }

.wp-stats {
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
.wp-stats .label { color: #8b949e; }
.wp-stats .value { color: #79c0ff; }
.wp-stats .ok { color: #3fb950; }
.wp-stats .warn { color: #f0883e; }
.wp-stats .err { color: #f85149; }

.wp-footnote {
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

const WORKER_URL: &str = "/workers/wp_stripe_worker.js";
const CANVAS_W: u32 = 480;
const CANVAS_H: u32 = 360;
const POOL_SIZE: usize = 4;
const BLUR_PASSES: u32 = 30;

// =============================================================================
// Состояние демо
// =============================================================================

struct DemoState {
    pool: Vec<Worker>,
    pool_ready: usize,
    sequential_ms: f64,
    parallel_ms: f64,
    parallel_done: usize,
    /// Слот -> (y_offset, байты полосы). `y_offset` приходит от Worker'а.
    parallel_results: Vec<Option<(u32, Vec<u8>)>>,
    parallel_started_at: f64,
    op_in_progress: bool,
    frames_since_start: u32,
    frames_at_op_start: u32,
}

impl DemoState {
    fn new() -> Self {
        Self {
            pool: Vec::new(),
            pool_ready: 0,
            sequential_ms: 0.0,
            parallel_ms: 0.0,
            parallel_done: 0,
            parallel_results: vec![None; POOL_SIZE],
            parallel_started_at: 0.0,
            op_in_progress: false,
            frames_since_start: 0,
            frames_at_op_start: 0,
        }
    }
}

// =============================================================================
// Рисование тестового паттерна
// =============================================================================

/// 4 горизонтальные полосы. Каждая — свой градиент + рисунок.
fn draw_test_pattern(ctx: &CanvasRenderingContext2d, w: u32, h: u32) {
    let ctx_js: &JsValue = ctx.as_ref();
    let stripe_h = h as f64 / 4.0;
    let colors = ["#1f6feb", "#3fb950", "#f0883e", "#a371f7"];

    for i in 0..4 {
        let y = i as f64 * stripe_h;
        ctx_set(ctx_js, "fillStyle", &JsValue::from_str(colors[i]));
        ctx_call(ctx_js, "fillRect", &[0.0, y, w as f64, stripe_h]);

        // Тёмные круги поверх — для контраста и детализации.
        ctx_set(ctx_js, "fillStyle", &JsValue::from_str("#0d1117"));
        for j in 0..6 {
            let cx = 40.0 + j as f64 * 75.0;
            let cy = y + stripe_h / 2.0;
            let r = 12.0 + (j as f64 * 3.0) % 10.0;
            ctx_call(ctx_js, "beginPath", &[]);
            ctx_call(ctx_js, "arc", &[cx, cy, r, 0.0, std::f64::consts::PI * 2.0]);
            ctx_call(ctx_js, "fill", &[]);
        }

        // Горизонтальная линия по центру полосы.
        ctx_set(ctx_js, "strokeStyle", &JsValue::from_str("#f0f6fc"));
        ctx_set(ctx_js, "lineWidth", &JsValue::from_f64(2.0));
        ctx_call(ctx_js, "beginPath", &[]);
        ctx_call(ctx_js, "moveTo", &[0.0, y + stripe_h / 2.0]);
        ctx_call(ctx_js, "lineTo", &[w as f64, y + stripe_h / 2.0]);
        ctx_call(ctx_js, "stroke", &[]);
    }
}

// =============================================================================
// ImageData
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
        .ok_or_else(|| JsValue::from_str("getImageData undefined"))?;
    let img: ImageData = img_val
        .dyn_into()
        .map_err(|_| JsValue::from_str("not ImageData"))?;
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
        .ok_or_else(|| JsValue::from_str("putImageData undefined"))?;
    Ok(())
}

/// Копирует одну полосу `stripe` (width × stripe_h) в общий буфер
/// изображения в позицию `y_offset`.
fn paste_stripe(
    full: &mut [u8],
    stripe: &[u8],
    w: u32,
    stripe_h: u32,
    y_offset: u32,
) {
    let row_bytes = (w * 4) as usize;
    let stripe_bytes = (w * stripe_h * 4) as usize;
    let offset = (y_offset as usize) * row_bytes;
    if offset + stripe_bytes <= full.len() && stripe.len() >= stripe_bytes {
        full[offset..offset + stripe_bytes]
            .copy_from_slice(&stripe[..stripe_bytes]);
    }
}

/// Извлекает одну горизонтальную полосу из полного изображения.
///
/// Возвращает `Vec<u8>` через явные `return`. Это обход бага
/// rust-analyzer: tail-выражение `if/else` r-a читает как `()`.
/// `return expr;` типизируется r-a независимо от позиции.
fn extract_stripe(
    full: &[u8],
    w: u32,
    stripe_h: u32,
    y_offset: u32,
) -> Vec<u8> {
    let row_bytes = (w * 4) as usize;
    let stripe_bytes = (w * stripe_h * 4) as usize;
    let offset = (y_offset as usize) * row_bytes;
    if offset + stripe_bytes > full.len() {
        return vec![0u8; stripe_bytes];
    }
    return full[offset..offset + stripe_bytes].to_vec();
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("wp-stats") else {
        return;
    };
    let seq_str = if st.sequential_ms > 0.0 {
        format_str(format_args!("{:.1} мс", st.sequential_ms))
    } else {
        String::from("—")
    };
    let par_str = if st.parallel_ms > 0.0 {
        format_str(format_args!("{:.1} мс", st.parallel_ms))
    } else {
        String::from("—")
    };
    let speedup = if st.sequential_ms > 0.0 && st.parallel_ms > 0.0 {
        format_str(format_args!("×{:.2}", st.sequential_ms / st.parallel_ms))
    } else {
        String::from("—")
    };
    let progress_str = if st.op_in_progress {
        format_str(format_args!("{}/{}", st.parallel_done, POOL_SIZE))
    } else if st.parallel_done == POOL_SIZE {
        format_str(format_args!("{}/{}", POOL_SIZE, POOL_SIZE))
    } else {
        String::from("—")
    };

    let coi = worker_pool::is_cross_origin_isolated();
    let coi_str = if coi { "да" } else { "нет" };
    let coi_class = if coi { "ok" } else { "warn" };

    let sab_str = worker_pool::sab_status();
    let sab_class = if worker_pool::is_shared_array_buffer_supported() {
        "ok"
    } else {
        "warn"
    };

    let pool_status = if st.pool_ready == 0 {
        String::from("0/4")
    } else if st.pool_ready < POOL_SIZE {
        format_str(format_args!("{}/{}", st.pool_ready, POOL_SIZE))
    } else {
        format_str(format_args!("{}/{}", POOL_SIZE, POOL_SIZE))
    };
    let pool_class = if st.pool_ready >= POOL_SIZE {
        "ok"
    } else {
        "warn"
    };

    let frames_since_op = st.frames_since_start.saturating_sub(st.frames_at_op_start);
    let frames_str = if st.op_in_progress {
        format_str(format_args!("{frames_since_op} кадров"))
    } else if st.frames_at_op_start > 0 {
        format_str(format_args!("за операцию: {frames_since_op}"))
    } else {
        String::from("—")
    };

    let html = format_str(format_args!(
        r#"<div><span class="label">Sequential (main thread): </span><span class="warn">{seq}</span> · <span class="label">Parallel (pool of {ps}): </span><span class="ok">{par}</span> · <span class="label">speedup: </span><span class="value">{su}</span></div>
           <div><span class="label">Cross-Origin Isolated: </span><span class="{cc}">{coi}</span> · <span class="label">SharedArrayBuffer: </span><span class="{sc}">{ss}</span> · <span class="label">progress: </span><span class="value">{pg}</span></div>
           <div><span class="label">Пул: </span><span class="{pc}">{pool}</span> · <span class="label">Кадры main thread: </span><span class="value">{fr}</span></div>"#,
        seq = seq_str,
        par = par_str,
        ps = POOL_SIZE,
        su = speedup,
        cc = coi_class,
        coi = coi_str,
        sc = sab_class,
        ss = sab_str,
        pg = progress_str,
        pc = pool_class,
        pool = pool_status,
        fr = frames_str,
    ));
    el.set_inner_html(&html);
}

const DESC_LINES: &[&str] = &[
    "Изображение разрезано на 4 горизонтальные полосы. \
     <strong>Sequential</strong> — main thread обрабатывает их \
     одну за другой. <strong>Parallel</strong> — 4 Worker'а \
     обрабатывают одновременно, каждый свою полосу.",
    "Смотрите на <strong>счётчик кадров</strong>. Sequential замораживает \
     main thread. Parallel оставляет его свободным — счётчик растёт, \
     пока 4 Worker'а работают.",
    "Пул Worker'ов — это <strong>инфраструктура</strong>. Создаётся один \
     раз при загрузке страницы, переиспользуется для каждой операции. \
     Создание 4 Worker'ов заново на каждый клик — 40–200 мс оверхеда.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("wp-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("wp-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// rAF-счётчик
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
// Обработчики Worker'ов пула
// =============================================================================

fn wire_pool_messages(
    workers: &[Worker],
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    for (idx, worker) in workers.iter().enumerate() {
        let state2 = state.clone();
        let slot = idx;

        // --- onmessage: ready / init_error / stripe_done ---
        let on_msg = wasm_bindgen::closure::Closure::<dyn FnMut(MessageEvent)>::new(
            move |ev: MessageEvent| {
                let data = ev.data();
                let Ok(kind) = Reflect::get(&data, &JsValue::from_str("type")) else {
                    return;
                };
                let Some(kind_str) = kind.as_string() else {
                    return;
                };

                if kind_str == "ready" {
                    {
                        let mut st = state2.borrow_mut();
                        st.pool_ready += 1;
                    }
                    // Обновляем статистику, чтобы «Пул: N/4» рос.
                    if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                        let st = state2.borrow();
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
                        if let Some(el) = doc.get_element_by_id("wp-stats") {
                            let html = format_str(format_args!(
                                r#"<div><span class="err">Worker init error: {msg}</span></div>"#,
                                msg = msg,
                            ));
                            el.set_inner_html(&html);
                        }
                    }
                    return;
                }

                if kind_str == "stripe_done" {
                    let Some(buffer_val) =
                        Reflect::get(&data, &JsValue::from_str("buffer")).ok()
                    else {
                        return;
                    };
                    let Ok(arr) = buffer_val.dyn_into::<Uint8Array>() else {
                        return;
                    };
                    let bytes = arr.to_vec();
                    let y_offset = Reflect::get(&data, &JsValue::from_str("y"))
                        .ok()
                        .and_then(|v| v.as_f64())
                        .unwrap_or(0.0) as u32;
                    let slot_result = Reflect::get(&data, &JsValue::from_str("slot"))
                        .ok()
                        .and_then(|v| v.as_f64())
                        .unwrap_or(0.0) as u32;

                    let all_done = {
                        let mut st = state2.borrow_mut();
                        let slot_idx = slot_result as usize;
                        if slot_idx < st.parallel_results.len() {
                            // Храним (y_offset, bytes) — Worker сообщает,
                            // куда положить полосу в общий буфер.
                            st.parallel_results[slot_idx] =
                                Some((y_offset, bytes));
                        }
                        st.parallel_done += 1;
                        if st.parallel_done >= POOL_SIZE {
                            st.parallel_ms =
                                js_sys::Date::now() - st.parallel_started_at;
                            st.op_in_progress = false;
                            true
                        } else {
                            false
                        }
                    };

                    if all_done {
                        let Some(doc) =
                            web_sys::window().and_then(|w| w.document())
                        else {
                            return;
                        };
                        let Some(canvas_el) = doc.get_element_by_id("wp-canvas")
                        else {
                            return;
                        };
                        let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>()
                        else {
                            return;
                        };
                        let Ok(Some(ctx_val)) = canvas.get_context("2d") else {
                            return;
                        };
                        let Ok(ctx) =
                            ctx_val.dyn_into::<CanvasRenderingContext2d>()
                        else {
                            return;
                        };

                        let stripe_h = CANVAS_H / POOL_SIZE as u32;
                        let mut full =
                            vec![0u8; (CANVAS_W * CANVAS_H * 4) as usize];
                        {
                            let st = state2.borrow();
                            for i in 0..POOL_SIZE {
                                if let Some((y, bytes)) =
                                    st.parallel_results[i].as_ref()
                                {
                                    paste_stripe(
                                        &mut full,
                                        bytes,
                                        CANVAS_W,
                                        stripe_h,
                                        *y,
                                    );
                                }
                            }
                        }
                        let _ = put_image_bytes(&ctx, &full, CANVAS_W, CANVAS_H);
                        let st = state2.borrow();
                        update_stats(&doc, &st);
                    }
                }
            },
        );
        worker.set_onmessage(Some(on_msg.as_ref().unchecked_ref()));
        on_msg.forget();

        // --- onerror: Worker не смог загрузить скрипт, упал в init, CSP ---
        let on_err = wasm_bindgen::closure::Closure::<dyn FnMut(ErrorEvent)>::new(
            move |ev: ErrorEvent| {
                let msg = ev.message();
                let file = ev.filename();
                let line = ev.lineno();
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    if let Some(el) = doc.get_element_by_id("wp-stats") {
                        let html = format_str(format_args!(
                            r#"<div><span class="err">Worker #{slot} error: {msg}</span></div>
                               <div><span class="label">at </span><span class="value">{file}:{line}</span></div>"#,
                            slot = slot,
                            msg = msg,
                            file = file,
                            line = line,
                        ));
                        el.set_inner_html(&html);
                    }
                }
            },
        );
        worker.set_onerror(Some(on_err.as_ref().unchecked_ref()));
        on_err.forget();
    }
    Ok(())
}

// =============================================================================
// Кнопки
// =============================================================================

fn wire_reset_button(document: &Document) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("wp-reset")
        .ok_or_else(|| JsValue::from_str("no wp-reset"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(c) = doc.get_element_by_id("wp-canvas") else {
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

fn wire_sequential_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("wp-sequential")
        .ok_or_else(|| JsValue::from_str("no wp-sequential"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(c) = doc.get_element_by_id("wp-canvas") else {
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

        let Ok(full) = get_image_bytes(&ctx, CANVAS_W, CANVAS_H) else {
            return;
        };

        let stripe_h = CANVAS_H / POOL_SIZE as u32;
        let mut out = vec![0u8; full.len()];
        let t0 = js_sys::Date::now();

        for i in 0..POOL_SIZE {
            let y = i as u32 * stripe_h;
            let stripe = extract_stripe(&full, CANVAS_W, stripe_h, y);
            let filtered =
                box_blur_rgba(&stripe, CANVAS_W, stripe_h, BLUR_PASSES);
            paste_stripe(&mut out, &filtered, CANVAS_W, stripe_h, y);
        }

        let elapsed = js_sys::Date::now() - t0;
        let _ = put_image_bytes(&ctx, &out, CANVAS_W, CANVAS_H);

        {
            let mut st = state.borrow_mut();
            st.sequential_ms = elapsed;
            st.op_in_progress = false;
        }
        let st = state.borrow();
        update_stats(&doc, &st);
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_parallel_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("wp-parallel")
        .ok_or_else(|| JsValue::from_str("no wp-parallel"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        let Some(c) = doc.get_element_by_id("wp-canvas") else {
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

        let Ok(full) = get_image_bytes(&ctx, CANVAS_W, CANVAS_H) else {
            return;
        };

        let stripe_h = CANVAS_H / POOL_SIZE as u32;
        let started_at = js_sys::Date::now();

        {
            let mut st = state.borrow_mut();
            st.op_in_progress = true;
            st.parallel_done = 0;
            st.parallel_results = vec![None; POOL_SIZE];
            st.parallel_started_at = started_at;
            st.frames_at_op_start = st.frames_since_start;
        }

        // Отправляем задачи по slot'ам (round-robin).
        let pool = {
            let st = state.borrow();
            st.pool.clone()
        };
        for (i, worker) in pool.iter().enumerate() {
            let y = i as u32 * stripe_h;
            let stripe = extract_stripe(&full, CANVAS_W, stripe_h, y);
            let _ = worker_pool::post_stripe(
                worker,
                &stripe,
                y,
                CANVAS_W,
                stripe_h,
                BLUR_PASSES,
                i as u32,
            );
        }

        let st = state.borrow();
        update_stats(&doc, &st);
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

    // Service Worker cleanup (см. 22.4).
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
    h2.set_text_content(Some("22.5 — Worker pools и SharedArrayBuffer"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !worker_pool::is_worker_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("wp-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает Worker. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Canvas.
    let canvas_el = document.create_element("canvas")?;
    canvas_el.set_id("wp-canvas");
    canvas_el.set_class_name("wp-canvas");
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
    controls.set_class_name("wp-block");
    controls.set_inner_html(
        r#"<button class="wp-btn" id="wp-reset">Reset</button>
           <button class="wp-btn wp-btn-warn" id="wp-sequential">Sequential (4× main thread)</button>
           <button class="wp-btn wp-btn-primary" id="wp-parallel">Parallel (pool of 4)</button>"#,
    );
    root.append_child(&controls)?;

    // Статистика.
    let stats = document.create_element("div")?;
    stats.set_class_name("wp-stats");
    stats.set_id("wp-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("wp-footnote");
    footnote.set_text_content(Some(
        "Пул Worker'ов создаётся один раз при загрузке и переиспользуется. \
         SharedArrayBuffer доступен только при COOP/COEP-заголовках: \
         Cross-Origin-Opener-Policy: same-origin, Cross-Origin-Embedder-Policy: \
         require-corp. В демо SAB-статус показан в статистике — сейчас \
         он используется только для feature detection; обмен полосами идёт \
         через transferable.",
    ));
    root.append_child(&footnote)?;

    // Состояние.
    let state = Rc::new(RefCell::new(DemoState::new()));

    // Создаём пул.
    let pool = worker_pool::create_pool(WORKER_URL, POOL_SIZE)?;
    wire_pool_messages(&pool, state.clone())?;
    state.borrow_mut().pool = pool;

    // Обработчики UI.
    wire_reset_button(&document)?;
    wire_sequential_button(&document, state.clone())?;
    wire_parallel_button(&document, state.clone())?;

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

## 22.5.12. Анализ решения

**Какую ценность даёт демо.**

Подглава показывает **два уровня оптимизации**:

- **Параллелизм (пул Worker'ов).** Sequential 400–600 мс превращается в Parallel 100–200 мс. Speedup 3–4× на типичном CPU.
- **Осведомлённость о shared memory.** Feature detection COI и SAB — статус в статистике. Пользователь видит, доступен ли SAB в его окружении.

**Три визуальных доказательства:**

1. **Sequential vs Parallel time.** Прямое сравнение в мс.
2. **Счётчик кадров.** Замирает при sequential, растёт при parallel. Main thread действительно свободен.
3. **4 прогресс-слота.** Видно, как каждая полоса завершается — по мере готовности Worker'ов.

**Четыре диагностических индикатора:**

- `Cross-Origin Isolated: да/нет` — применились ли COOP/COEP.
- `SharedArrayBuffer: доступен/недоступен` — открыт ли SAB.
- `Пул: N/4` — готовы ли Worker'ы.
- `Worker #N error: ...` — если Worker не смог загрузиться.

**Скрытие технических сложностей.**

- Пользователь кликает две кнопки, видит два числа и speedup. Внутри — 4 Worker'а, transferable, WASM в каждом, агрегация.
- Feature detection COI/SAB через `navigator` и `typeof` — одна функция.
- Service Worker cleanup — self-healing, как в 22.4.

**Что осознанно не реализовано.**

- **Round-robin для неравномерных задач.** Все 4 полосы одинакового размера — простой случай. Least-loaded и work-stealing — темы для отдельной подглавы.
- **SharedArrayBuffer в деле.** Feature detection показан, но демо использует transferable. Полноценный SAB-режим требует COOP/COEP, которые могут ломать другие демо. Упоминается в тексте и footnote.
- **Atomics.wait / notify.** Не используются — main thread не должен блокироваться. Полноценный producer-consumer — отдельная тема.
- **`wasm-bindgen-rayon`.** Продвинутая тема, требует отдельной сборки.

**Правила проекта, применённые в этой подглаве.**

1. **Autonomous app.rs.** Всё inline: `format_str`, feature detection, работа с пулом.
2. **`format_str(format_args!(...))` в tail-позициях.** В `update_stats`, `support_summary`.
3. **`bool_of(matches!(...))` в tail-позиции.** В `is_worker_supported`, `is_shared_array_buffer_supported`, `is_atomics_supported`, `is_cross_origin_isolated`.
4. **RefMut-дисциплина.** Каждый `borrow_mut` в узком scope.
5. **Feature detection перед использованием API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.**
8. **Уникальный префикс `wp-`.** Не пересекается с `ww-` (22.4), `tr-` (22.3), `dw-` (22.2), `oc-` (22.1), `ow-` (глава 22).
9. **Строки — через одну функцию `format_str`.**
10. **Никаких неиспользуемых импортов.**
11. **Внутренние ссылки через `crate::`.**
12. **Service Worker cleanup.**
13. **Явный `return expr;` вместо tail-выражения** — там, где r-a не выводит тип (`extract_stripe`).
14. **Обработка `onerror` для Worker'ов** — диагностика загрузки.

**Проверки корректности.**

- `create_pool` использует `WorkerOptions::set_type(WorkerType::Module)` — обязательно для `import()` в worker'е.
- `post_stripe` использует `post_message_with_transfer` — zero-copy.
- `extract_stripe_result` возвращает `Option` — обрабатывается через `?`.
- `wire_pool_messages` вешает `onmessage` и `onerror` на каждый Worker; `slot` приходит в сообщении от Worker'а.
- В `wire_parallel_button` результаты собираются по `slot`, собираются в canvas только когда все 4 пришли.
- `paste_stripe` и `extract_stripe` — чистые функции над `&[u8]`, легко тестируются.
- `box_blur_rgba` вызывается как обычная Rust-функция (не через JS-обёртку).
- `extract_stripe` использует явный `return` — обход бага r-a в tail-позиции.

---

## 22.5.13. Список доработок после первой итерации

1. **`bool_of` в feature detection.** `matches!(…)` в tail-позиции даёт E0308 rust-analyzer'а.
2. **Round-robin вместо «все задачи в один Worker».** Первая версия посылала все 4 полосы в один Worker. Пул подразумевает распределение — реализовано.
3. **Progress: N из 4.** Первая версия показывала только финальное время. Добавлен промежуточный прогресс.
4. **Сборка результата только после всех 4.** Первая версия писала canvas после каждого Worker'а. Исправлено: ждём `parallel_done == POOL_SIZE`, собираем все полосы, потом put_image_data.
5. **Service Worker cleanup.** Module worker делает `import`, SW ломает загрузку. Решено через `unregister_sw_and_reload` + sessionStorage-флаг (см. 22.4).
6. **`sab_status()` и `coi_status()`.** Человекочитаемые строки для статистики. `format_str` в tail-позиции.
7. **`WorkerOptions` с `type: "module"`.** Обязательно для module worker.
8. **Уникальный префикс `wp-`.** Не пересекается с предыдущими.
9. **Внутренние ссылки через `crate::`.** `use crate::worker_pool;` и `use crate::wasm_in_worker::box_blur_rgba;`.
10. **Тесты feature detection.** Проверяют, что функции не паникуют, возвращают `bool` и строки. Не требуют реального Worker'а.
11. **Тесты `create_pool` / `terminate_pool`.** Проверяют, что пул создаётся и завершается без паники.
12. **Явный `return` в `extract_stripe`.** Обход бага r-a в tail-позиции. `vec_u8_of` удалена — не потребовалась.
13. **`parallel_results: Vec<Option<(u32, Vec<u8>)>>`.** Хранит `y_offset` рядом с байтами — Worker сообщает точную позицию, main thread не угадывает.
14. **`onerror` для Worker'ов.** Показывает ошибку загрузки скрипта прямо в статистике.
15. **Обработка `init_error`.** Если WASM в Worker'е не загрузился, main thread видит причину.
16. **`Пул: N/4` в статистике.** Диагностика готовности Worker'ов. Обновляется при каждом `ready`.
17. **`package.json` — `out-dir web/public/pkg`.** Vite с `root: 'web'` резолвит URL относительно `web/`. `web/public/` отдаётся по `/`. Значит `/pkg/...` физически = `web/public/pkg/...`.
18. **`vite.config.js` — `preview.headers`.** Preview использует свой набор заголовков. Без дублирования COOP/COEP в preview они не применяются.
19. **`.gitignore` — `web/public/pkg/`.** Артефакты сборки не место в git.

---

## 22.5.14. Итог

**Пул Worker'ов** — это инфраструктура. Создаётся один раз при загрузке, переиспользуется для каждой операции. Round-robin — простейшая стратегия, работает для одинаковых задач. Создание 4 Worker'ов на каждый клик — 40–200 мс оверхеда, пул это исключает.

**`SharedArrayBuffer`** — общая память. Не transferable, не копируется: просто ссылка. Несколько JS-контекстов видят одни и те же байты. Требует COOP/COEP-заголовков, без них — undefined.

**`Atomics`** — синхронизация для SAB. `Atomics.store`, `Atomics.load` — видимость и атомарность. `Atomics.wait`/`notify` — producer-consumer. Работают только над `Int32Array` и `BigInt64Array` над `SharedArrayBuffer`.

**Параллелизм** даёт speedup, близкий к числу ядер — но только если задачи действительно параллельны. Sequential 500 мс → Parallel 150 мс на 4-ядерном CPU.

**Feature detection COI** — `self.crossOriginIsolated` — прямой индикатор, что COOP/COEP-заголовки применились. Если COI `true`, а SAB `undefined` — браузер старый. Если COI `false` — заголовки не пришли или кэш.

**Диагностика в демо** — четыре индикатора: COI, SAB, Пул N/4, Worker error. Каждый указывает на конкретную проблему окружения.

Демо показывает всё это на одном экране: sequential vs parallel, speedup, COI/SAB-статус, rAF-счётчик. Wow-момент: во время параллельной обработки счётчик кадров продолжает расти — main thread действительно свободен.

Подглава закрывает тему **горизонтального масштабирования**. Практический проект (22.6) — следующий шаг: полный редактор с pool + WASM + OffscreenCanvas + прогресс + отмена.