# Глава 31. Performance API

## 31.0. Что вы получите в этой главе

Работающий **инструмент измерения производительности**: реестр задач, замер каждой через `Performance::now()`, статистика по нескольким итерациям (min, max, mean, median, stddev), ранжирование задач по времени, подсветка узкого места. Плюс живой `PerformanceObserver`, который ловит `mark`/`measure` в реальном времени, и вывод `Navigation Timing` / `Resource Timing` из браузера.

Отдельно разберём, **чего демо не показывает** — профилирование в DevTools Performance panel. Это отдельный инструмент, он вне Web API и не может быть запущен из Rust.

## 31.1. `Performance`

`Performance` — корневой объект API. Доступен как `window.performance` и из Worker-контекста (`self.performance`).

```rust
let perf = web_sys::window()
    .ok_or("no window")?
    .performance()
    .ok_or("no performance")?;
```

Ключевые методы:

| Метод | Что делает |
|---|---|
| `now()` | монотонные миллисекунды с высокой точностью |
| `time_origin()` | время старта origin в Unix-миллисекундах |
| `mark(name)` | создать отметку с текущим временем |
| `measure(name, start, end)` | измерить промежуток между отметками |
| `get_entries()` | все записи производительности |
| `get_entries_by_name(name)` | фильтр по имени |
| `get_entries_by_type(type)` | фильтр по типу (`mark`, `measure`, `resource`, `navigation`) |
| `clear_marks()` / `clear_measures()` | очистка |

`performance.now()` — **монотонные** миллисекунды: они не зависят от системных часов и идут только вперёд. В отличие от `Date.now()`, `performance.now()` не «прыгает» при синхронизации часов с NTP и имеет микросекундную точность в некоторых браузерах.

**Особенность `web-sys 0.3.105`:** сигнатуры методов отличаются от WebIDL из MDN.

| Метод | Возвращает |
|---|---|
| `mark(name)` | `Result<PerformanceMark, JsValue>` — не `Result<(), _>` |
| `clear_marks()` / `clear_measures()` | `()` без `Result` |
| `get_entries_by_type(type)` | `js_sys::Array` напрямую |
| `measure_with_start_mark_and_end_mark` | **отсутствует** — вызываем `measure` через `Reflect` |

## 31.2. Performance marks

`mark(name)` ставит именованную отметку в таймлайне с текущим `now()`. Отметки нужны, чтобы потом `measure` мог рассчитать промежуток. Имя произвольное, но с префиксом — хорошая практика: `app:init`, `fetch:start`, `render:end`. Без префикса легко получить конфликт с другими библиотеками.

```rust
perf.mark("app:init")?;
// … инициализация …
perf.mark("app:ready")?;
```

Отметки остаются в буфере `Performance` и участвуют в `getEntriesByType("mark")`. Их нужно очищать (`clear_marks()`), иначе буфер растёт: у Chrome лимит — 5000 записей на страницу.

**Важно:** у `mark` нет `duration`. Это точка во времени. При отображении не показывайте «0 мс» — покажите только `start_time`.

## 31.3. Performance measures

`measure(name, start, end)` вычисляет длительность между двумя отметками и создаёт запись типа `measure`. Если `start`/`end` не указать — берётся `0` и текущее время.

В `web-sys 0.3.105` типизированного метода с тремя строками нет, поэтому вызываем `performance.measure(name, start, end)` через `js_sys::Reflect`:

```rust
let f = js_sys::Reflect::get(&perf, &JsValue::from_str("measure"))?;
let func: js_sys::Function = f.dyn_into()?;
func.call3(&perf, &name.into(), &start.into(), &end.into())?;
```

Результат доступен через `getEntriesByType("measure")`, у каждой записи — `name()`, `start_time()`, `duration()`. Также `Performance` отдаёт в DevTools секцию «Timings» — `measure` можно увидеть прямо в Performance panel.

Правило проекта: **каждая фаза инициализации приложения — `mark` + `measure`**. При старте это дешёвая операция (одна запись в буфер), а при отладке — сразу видно, где потеряны 200 мс.

## 31.4. `PerformanceObserver`

`mark`/`measure` пишутся в буфер, но их можно слушать **асинхронно** через `PerformanceObserver`. Это способ получать события производительности без polling `getEntries()`.

```rust
let cb = Closure::wrap(Box::new(move |list: PerformanceObserverEntryList, _obs: PerformanceObserver| {
    let entries = list.get_entries();
    for i in 0..entries.length() {
        let entry: PerformanceEntry = entries.get(i).unchecked_into();
        // entry.name(), entry.entry_type(), entry.duration(), entry.start_time()
    }
}) as Box<dyn FnMut(_, _)>);

let observer = PerformanceObserver::new(cb.as_ref().unchecked_ref())?;
```

`PerformanceObserverEntryList::get_entries()` возвращает **все** записи с момента последнего колбэка, не только новые. `get_entries_by_type(type)` — тот же список, отфильтрованный по типу.

Observer живёт до конца страницы. `observer.disconnect()` — отписка. Колбэк должен быть «забыт» через `Closure::forget`, иначе его соберёт GC.

**Особенность `web-sys 0.3.105`:** `PerformanceObserverInit::new()` требует обязательный аргумент `&JsValue`, а `observe_with_options` отсутствует. Собираем объект опций вручную и вызываем `observer.observe(...)` через `Reflect`:

```rust
let opts = js_sys::Object::new();
let arr = js_sys::Array::new();
for t in types { arr.push(&JsValue::from_str(t)); }
js_sys::Reflect::set(&opts, &JsValue::from_str("entryTypes"), &arr)?;

let observe_fn = js_sys::Reflect::get(&observer, &JsValue::from_str("observe"))?;
let observe: js_sys::Function = observe_fn.dyn_into()?;
observe.call1(&observer, &opts)?;
```

## 31.5. Resource timing

`PerformanceResourceTiming` — запись о загрузке ресурса: скрипт, стиль, изображение, `fetch`, `XMLHttpRequest`. Поля:

| Поле | Что означает |
|---|---|
| `name` | URL ресурса |
| `start_time` | когда началась загрузка |
| `duration` | сколько заняла всего |
| `transfer_size` | байт передано по сети (0 из кэша) |
| `encoded_body_size` | размер тела в кодировке |
| `decoded_body_size` | размер после декодирования |
| `initiator_type` | `script`, `img`, `fetch`, … |

Пример: получить всё, что грузилось дольше 100 мс:

```rust
let entries: js_sys::Array = perf.get_entries_by_type("resource");
for i in 0..entries.length() {
    let e: PerformanceResourceTiming = entries.get(i).unchecked_into();
    if e.duration() > 100.0 {
        // медленный ресурс
    }
}
```

**Cross-origin ресурсы** не отдают `transfer_size` и детали, если сервер не поставил `Timing-Allow-Origin`. Это часть security-модели — сайт не должен знать, что грузит чужой ресурс.

## 31.6. Navigation timing

`PerformanceNavigationTiming` — одна запись на страницу, описывающая загрузку документа. Ключевые фазы:

| Поле | Что означает |
|---|---|
| `redirect_start` → `redirect_end` | редиректы |
| `fetch_start` | начало запроса за HTML |
| `domain_lookup_start/end` | DNS |
| `connect_start/end` | TCP/TLS |
| `request_start` → `response_start` | ожидание ответа |
| `response_start` → `response_end` | получение тела |
| `dom_content_loaded_event_start` | DCL |
| `load_event_start` | onload |
| `dom_complete` | DOM готов |
| `dom_interactive` | интерактивный DOM |

Разница между фазами — это и есть «диагноз»: DNS-запрос занимает 200 мс → медленный DNS; `response_start` через 800 мс → медленный сервер; `dom_content_loaded` долго → тяжёлый HTML.

**Ограничение:** запись одна и читается через `getEntriesByType("navigation")[0]`. Она не обновляется после `load`. Фазы, которые не выполнялись (например, `dns`, если соединение уже установлено), дают `0.000` — это не ошибка.

## 31.7. Измерение Rust/WASM

Rust-функция не может «отдать» время наружу — не потому что недоступен `performance.now()`, а потому что замер нужно структурировать. Три уровня:

1. **Макроуровень — `performance.now()` вокруг блока.**
   ```rust
   let t0 = perf.now();
   do_heavy_work();
   let elapsed = perf.now() - t0;
   ```
   Просто, но без деталей.

2. **Фазовый уровень — `mark` + `measure`.**
   Отметки видны в DevTools, можно наложить на таймлайн.

3. **Статистический уровень — N итераций + статистика.**
   Одна итерация даёт шум: GC, JIT, компиляция WASM. Минимум, максимум, среднее, медиана — уже надёжная картина.

Правило проекта: **прогрев перед измерением.** Первый вызов любой Rust/WASM-функции — самый медленный: страница ещё не прогрета, JIT не оптимизировал JS-glue, кэш пуст. Первые `WARMUP_ITERATIONS` (по умолчанию 3) вызовов не попадают в статистику. Контракт `run_benchmark(name, iterations)`: `iterations` — именно число **измеренных** сэмплов, прогрев идёт **до** них и в результат не входит.

## 31.8. Поиск bottleneck

Bottleneck — самая медленная часть системы. Определяется по **среднему** времени (`mean`), а не по `min` или `max`:

- `min` — идеальный случай, недостижимый в продакшене.
- `max` — единичный выброс (GC, peer-throttle).
- `mean` и `median` — реальная картина.

`stddev` показывает стабильность: 5 мс ± 0.2 — стабильно; 5 мс ± 8 — нестабильно, значит замер шумный и надо переделать.

Лаборатория (модуль `perf_lab`) считает все пять величин и сортирует задачи по `mean`. Bottleneck — верхняя.

## 31.9. Практический performance laboratory

**Что делает.** Реестр задач, каждая — Rust-функция. Для каждой: N итераций с замером через `performance.now()`, статистика, ранжирование, подсветка bottleneck.

**Почему это полезно.** Каждое серьёзное приложение должно уметь отвечать на вопрос «что тормозит?». В браузере нет «профилировщика Rust», но есть `Performance API` — если правильно структурировать замеры. Этот модуль даёт базу.

### Структура файлов

```
src/
  perf_core.rs       — обёртка Performance, mark, measure, Span
  perf_observer.rs   — PerformanceObserver → события
  perf_timing.rs     — Resource Timing + Navigation Timing
  perf_lab.rs        — лаборатория: бенчмарки, статистика, bottleneck
  app.rs             — демо только этой главы
tests/
  perf_core_test.rs  — host-тесты (статистика, форматирование)
  perf_lab_test.rs   — wasm-тесты (реальные замеры)
```

---

## Исходные коды новых модулей

### Cargo.toml — добавить features

В `[dependencies.web-sys]`, в массив `features`, добавьте:

```toml
    "Performance",
    "PerformanceEntry",
    "PerformanceMark",
    "PerformanceMeasure",
    "PerformanceObserver",
    "PerformanceObserverEntryList",
    "PerformanceObserverInit",
    "PerformanceResourceTiming",
    "PerformanceNavigationTiming",
```

Больше ничего не требуется. `performance.now()` доступен через `Window` (уже включён).

### `src/perf_core.rs`

```rust
//! Обёртка над Performance API: mark, measure, Span.
//! Автономный модуль — не зависит от других модулей проекта.
//!
//! ВАЖНО (web-sys 0.3.105):
//!   • Performance::mark возвращает Result<PerformanceMark, JsValue>.
//!   • Performance::clear_marks / clear_measures — возвращают () без Result.
//!   • Performance::get_entries_by_type — js_sys::Array напрямую.
//!   • measure(name, start, end) вызывается через Reflect:
//!     типизированного метода с тремя строками в web-sys нет.

use wasm_bindgen::prelude::*;
use web_sys::Performance;

/// Возвращает `window.performance`.
pub fn performance() -> Result<Performance, JsValue> {
    web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .performance()
        .ok_or_else(|| JsValue::from_str("no performance"))
}

/// `performance.now()` — монотонные миллисекунды.
pub fn now_ms() -> f64 {
    performance().map(|p| p.now()).unwrap_or(0.0)
}

/// Ставит именованную отметку. Возвращает `Ok(())`.
pub fn mark(name: &str) -> Result<(), JsValue> {
    // Performance::mark возвращает PerformanceMark — отбрасываем его.
    performance()?
        .mark(name)
        .map(|_| ())
        .map_err(|e| JsValue::from_str(&format!("mark {name}: {e:?}")))
}

/// Измеряет промежуток между двумя отметками.
///
/// В web-sys 0.3.105 типизированного метода с тремя строками нет,
/// поэтому вызываем `performance.measure(name, start, end)` через Reflect.
/// Возвращает `duration` созданной записи.
pub fn measure(name: &str, start: &str, end: &str) -> Result<f64, JsValue> {
    let perf = performance()?;

    let f = js_sys::Reflect::get(&perf, &JsValue::from_str("measure"))
        .map_err(|e| JsValue::from_str(&format!("Reflect measure: {e:?}")))?;
    let func: js_sys::Function = f
        .dyn_into()
        .map_err(|_| JsValue::from_str("measure не функция"))?;

    func.call3(
        &perf,
        &JsValue::from_str(name),
        &JsValue::from_str(start),
        &JsValue::from_str(end),
    )
    .map_err(|e| JsValue::from_str(&format!("measure {name}: {e:?}")))?;

    // Читаем duration только что созданной записи.
    let entries: js_sys::Array = perf.get_entries_by_name(name);
    if entries.length() == 0 {
        return Ok(0.0);
    }
    let entry: web_sys::PerformanceEntry =
        entries.get(entries.length() - 1).unchecked_into();
    Ok(entry.duration())
}

/// Очищает все mark'и. Возвращает `Ok(())` для единообразия API.
pub fn clear_marks() -> Result<(), JsValue> {
    performance()?.clear_marks();
    Ok(())
}

/// Очищает все measure'ы.
pub fn clear_measures() -> Result<(), JsValue> {
    performance()?.clear_measures();
    Ok(())
}

/// Возвращает список имён всех записей указанного типа.
///
/// `get_entries_by_type` возвращает `js_sys::Array` — без Result.
pub fn entry_names_of_type(entry_type: &str) -> Result<Vec<String>, JsValue> {
    let perf = performance()?;
    let entries: js_sys::Array = perf.get_entries_by_type(entry_type);
    let mut out = Vec::with_capacity(entries.length() as usize);
    for i in 0..entries.length() {
        let entry: web_sys::PerformanceEntry = entries.get(i).unchecked_into();
        out.push(entry.name());
    }
    Ok(out)
}

/// RAII-обёртка: засекает старт при создании, возвращает длительность
/// при `finish()`.
pub struct Span {
    start: f64,
    name: String,
}

impl Span {
    pub fn start(name: &str) -> Self {
        Self {
            start: now_ms(),
            name: name.to_string(),
        }
    }

    pub fn elapsed(&self) -> f64 {
        now_ms() - self.start
    }

    /// Возвращает длительность. Отметки не создаёт.
    pub fn finish(self) -> f64 {
        let end = now_ms();
        let elapsed = end - self.start;
        let _ = &self.name;
        elapsed
    }
}
```

### `src/perf_observer.rs`

```rust
//! PerformanceObserver: подписка на mark, measure, resource, navigation.
//! Автономный модуль — не зависит от других модулей проекта.
//!
//! ВАЖНО: `PerformanceObserverInit` в web-sys 0.3.105 —
//! dictionary с обязательным аргументом в `new`. Проще собрать
//! объект опций вручную через `js_sys::Object` и вызвать
//! `observer.observe({entryTypes: [...]})` через Reflect.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use web_sys::{PerformanceEntry, PerformanceObserver, PerformanceObserverEntryList};

/// Одна запись производительности в удобном виде.
#[derive(Debug, Clone)]
pub struct PerfEvent {
    pub name: String,
    pub entry_type: String,
    pub start_time: f64,
    pub duration: f64,
}

/// Хранилище событий + сам observer. Держим observer живым, чтобы
/// колбэк не был собран GC.
pub struct PerfSubscription {
    _observer: PerformanceObserver,
    events: Rc<RefCell<Vec<PerfEvent>>>,
}

impl PerfSubscription {
    pub fn subscribe(types: &[&str]) -> Result<Self, JsValue> {
        let events: Rc<RefCell<Vec<PerfEvent>>> = Rc::new(RefCell::new(Vec::new()));
        let events_in = events.clone();

        let cb = Closure::wrap(Box::new(
            move |list: PerformanceObserverEntryList, _obs: PerformanceObserver| {
                let entries = list.get_entries();
                let mut store = events_in.borrow_mut();
                for i in 0..entries.length() {
                    let entry: PerformanceEntry = entries.get(i).unchecked_into();
                    store.push(PerfEvent {
                        name: entry.name(),
                        entry_type: entry.entry_type(),
                        start_time: entry.start_time(),
                        duration: entry.duration(),
                    });
                }
            },
        ) as Box<dyn FnMut(_, _)>);

        let observer = PerformanceObserver::new(cb.as_ref().unchecked_ref())
            .map_err(|e| JsValue::from_str(&format!("PerformanceObserver::new: {e:?}")))?;
        cb.forget();

        // Собираем опции: { entryTypes: ["mark", "measure", ...] }
        let opts = js_sys::Object::new();
        let arr = js_sys::Array::new();
        for t in types {
            arr.push(&JsValue::from_str(t));
        }
        js_sys::Reflect::set(&opts, &JsValue::from_str("entryTypes"), &arr)
            .map_err(|e| JsValue::from_str(&format!("set entryTypes: {e:?}")))?;

        // observer.observe(opts) — через Reflect.
        let observe_fn = js_sys::Reflect::get(&observer, &JsValue::from_str("observe"))
            .map_err(|e| JsValue::from_str(&format!("Reflect observe: {e:?}")))?;
        let observe: js_sys::Function = observe_fn
            .dyn_into()
            .map_err(|_| JsValue::from_str("observe не функция"))?;
        observe
            .call1(&observer, &opts)
            .map_err(|e| JsValue::from_str(&format!("observe: {e:?}")))?;

        Ok(Self {
            _observer: observer,
            events,
        })
    }

    pub fn events(&self) -> Vec<PerfEvent> {
        self.events.borrow().clone()
    }

    pub fn clear(&self) {
        self.events.borrow_mut().clear();
    }

    pub fn disconnect(self) {
        let _ = self._observer.disconnect();
    }
}
```

### `src/perf_timing.rs`

```rust
//! Resource Timing и Navigation Timing.
//!
//! ВАЖНО: `get_entries_by_type` возвращает `js_sys::Array` напрямую,
//! без Result. Ошибок на этом шаге быть не может.

use wasm_bindgen::prelude::*;

use crate::perf_core;

/// Одна запись Resource Timing в удобном виде.
#[derive(Debug, Clone)]
pub struct ResourceInfo {
    pub name: String,
    pub duration: f64,
    pub transfer_size: f64,
    pub initiator_type: String,
}

/// Возвращает все ресурсы, загруженные страницей.
pub fn resources() -> Result<Vec<ResourceInfo>, JsValue> {
    let perf = perf_core::performance()?;
    let entries: js_sys::Array = perf.get_entries_by_type("resource");

    let mut out = Vec::with_capacity(entries.length() as usize);
    for i in 0..entries.length() {
        let e: web_sys::PerformanceResourceTiming = entries.get(i).unchecked_into();
        out.push(ResourceInfo {
            name: e.name(),
            duration: e.duration(),
            transfer_size: e.transfer_size(),
            initiator_type: e.initiator_type(),
        });
    }
    Ok(out)
}

/// Только медленные ресурсы (duration > threshold_ms).
pub fn slow_resources(threshold_ms: f64) -> Result<Vec<ResourceInfo>, JsValue> {
    Ok(resources()?
        .into_iter()
        .filter(|r| r.duration > threshold_ms)
        .collect())
}

/// Фазы Navigation Timing для текущей страницы.
#[derive(Debug, Clone)]
pub struct NavPhases {
    pub redirect: f64,
    pub dns: f64,
    pub tcp_tls: f64,
    pub request: f64,
    pub response: f64,
    pub dom_interactive: f64,
    pub dom_complete: f64,
    pub load_event: f64,
}

impl NavPhases {
    pub fn total(&self) -> f64 {
        self.redirect
            + self.dns
            + self.tcp_tls
            + self.request
            + self.response
            + self.dom_interactive
            + self.dom_complete
            + self.load_event
    }
}

fn phase(end: f64, start: f64) -> f64 {
    (end - start).max(0.0)
}

/// Читает Navigation Timing. Возвращает None, если запись отсутствует.
pub fn navigation() -> Result<Option<NavPhases>, JsValue> {
    let perf = perf_core::performance()?;
    let entries: js_sys::Array = perf.get_entries_by_type("navigation");
    if entries.length() == 0 {
        return Ok(None);
    }
    let n: web_sys::PerformanceNavigationTiming = entries.get(0).unchecked_into();
    Ok(Some(NavPhases {
        redirect: phase(n.redirect_end(), n.redirect_start()),
        dns: phase(n.domain_lookup_end(), n.domain_lookup_start()),
        tcp_tls: phase(n.connect_end(), n.connect_start()),
        request: phase(n.response_start(), n.request_start()),
        response: phase(n.response_end(), n.response_start()),
        dom_interactive: phase(n.dom_interactive(), n.response_end()),
        dom_complete: phase(n.dom_complete(), n.dom_interactive()),
        load_event: phase(n.load_event_start(), n.dom_complete()),
    }))
}
```

### `src/perf_lab.rs`

```rust
//! Лаборатория производительности: реестр замеров, статистика, bottleneck.

use crate::perf_core;

/// Сколько первых итераций отбрасывать как «прогрев».
pub const WARMUP_ITERATIONS: u32 = 3;

#[derive(Debug, Clone)]
pub struct BenchResult {
    pub name: String,
    pub samples_ms: Vec<f64>,
}

impl BenchResult {
    pub fn new(name: &str, samples_ms: Vec<f64>) -> Self {
        Self {
            name: name.into(),
            samples_ms,
        }
    }

    pub fn count(&self) -> usize {
        self.samples_ms.len()
    }

    pub fn min_ms(&self) -> f64 {
        self.samples_ms.iter().cloned().fold(f64::INFINITY, f64::min)
    }

    pub fn max_ms(&self) -> f64 {
        self.samples_ms
            .iter()
            .cloned()
            .fold(f64::NEG_INFINITY, f64::max)
    }

    pub fn mean_ms(&self) -> f64 {
        if self.samples_ms.is_empty() {
            return 0.0;
        }
        self.samples_ms.iter().sum::<f64>() / self.samples_ms.len() as f64
    }

    pub fn median_ms(&self) -> f64 {
        if self.samples_ms.is_empty() {
            return 0.0;
        }
        let mut v = self.samples_ms.clone();
        v.sort_by(|a, b| a.partial_cmp(b).unwrap_or(std::cmp::Ordering::Equal));
        let mid = v.len() / 2;
        if v.len() % 2 == 0 {
            (v[mid - 1] + v[mid]) / 2.0
        } else {
            v[mid]
        }
    }

    pub fn std_dev_ms(&self) -> f64 {
        if self.samples_ms.len() < 2 {
            return 0.0;
        }
        let m = self.mean_ms();
        let var = self
            .samples_ms
            .iter()
            .map(|&x| (x - m).powi(2))
            .sum::<f64>()
            / (self.samples_ms.len() as f64 - 1.0);
        var.sqrt()
    }
}

/// Прогоняет задачу: сначала `WARMUP_ITERATIONS` прогревных вызовов
/// (не замеряются), затем `iterations` измеренных.
///
/// Контракт: `iterations` — именно число сэмплов в результате.
/// `BenchResult::count() == iterations` (при `iterations >= 1`).
pub fn run_benchmark<F: FnMut()>(name: &str, iterations: u32, mut task: F) -> BenchResult {
    // Прогрев: JIT, первый GC, пустой кэш — не должны попасть в статистику.
    for _ in 0..WARMUP_ITERATIONS {
        task();
    }

    let mut samples = Vec::with_capacity(iterations as usize);
    for _ in 0..iterations {
        let t0 = perf_core::now_ms();
        task();
        let dt = perf_core::now_ms() - t0;
        samples.push(dt);
    }
    BenchResult::new(name, samples)
}

/// Индекс узкого места — задача с наибольшим средним временем.
pub fn find_bottleneck(results: &[BenchResult]) -> Option<usize> {
    results
        .iter()
        .enumerate()
        .max_by(|(_, a), (_, b)| {
            a.mean_ms()
                .partial_cmp(&b.mean_ms())
                .unwrap_or(std::cmp::Ordering::Equal)
        })
        .map(|(i, _)| i)
}

/// Индекс самой стабильной задачи (наименьший stddev).
pub fn find_most_stable(results: &[BenchResult]) -> Option<usize> {
    results
        .iter()
        .enumerate()
        .min_by(|(_, a), (_, b)| {
            a.std_dev_ms()
                .partial_cmp(&b.std_dev_ms())
                .unwrap_or(std::cmp::Ordering::Equal)
        })
        .map(|(i, _)| i)
}

/// Форматирование в читаемую строку. Явный `return` — правило проекта:
/// rust-analyzer теряет тип `format!` в tail-позиции.
pub fn format_result(r: &BenchResult) -> String {
    return format!(
        "{}: mean {:.3} ms, median {:.3}, min {:.3}, max {:.3}, stddev {:.3} (n={})",
        r.name,
        r.mean_ms(),
        r.median_ms(),
        r.min_ms(),
        r.max_ms(),
        r.std_dev_ms(),
        r.count()
    );
}
```

---

## Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod perf_core;
pub mod perf_observer;
pub mod perf_timing;
pub mod perf_lab;
```

---

## Тесты

### `tests/perf_core_test.rs` (host)

```rust
//! Host-тесты perf_lab: чистая математика (статистика).
//! Performance API недоступно на host — здесь только BenchResult.

use hello_wasm02::perf_lab::BenchResult;

#[test]
fn min_max_mean_median() {
    let r = BenchResult::new("test", vec![1.0, 2.0, 3.0, 4.0, 5.0]);
    assert_eq!(r.min_ms(), 1.0);
    assert_eq!(r.max_ms(), 5.0);
    assert_eq!(r.mean_ms(), 3.0);
    assert_eq!(r.median_ms(), 3.0);
    assert_eq!(r.count(), 5);
}

#[test]
fn median_even_length_is_average_of_middle() {
    let r = BenchResult::new("even", vec![1.0, 2.0, 3.0, 4.0]);
    assert_eq!(r.median_ms(), 2.5);
}

#[test]
fn std_dev_zero_for_single_sample() {
    let r = BenchResult::new("single", vec![42.0]);
    assert_eq!(r.std_dev_ms(), 0.0);
}

#[test]
fn std_dev_matches_formula() {
    // [2, 4, 4, 4, 5, 5, 7, 9] → σ_sample ≈ 2.138
    let r = BenchResult::new("sample", vec![2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]);
    let sd = r.std_dev_ms();
    assert!((sd - 2.138).abs() < 0.01, "получили {sd}");
}

#[test]
fn empty_result_handles_gracefully() {
    let r = BenchResult::new("empty", vec![]);
    assert_eq!(r.count(), 0);
    assert_eq!(r.mean_ms(), 0.0);
    assert_eq!(r.median_ms(), 0.0);
    assert_eq!(r.std_dev_ms(), 0.0);
}
```

### `tests/perf_lab_test.rs` (wasm)

```rust
//! Wasm-тесты: реальные замеры через Performance API.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::perf_core;
use hello_wasm02::perf_lab::{self, BenchResult};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn now_is_monotonic() {
    let t0 = perf_core::now_ms();
    let t1 = perf_core::now_ms();
    assert!(t1 >= t0);
}

#[wasm_bindgen_test]
fn span_measures_positive_duration() {
    let span = perf_core::Span::start("test");
    let mut acc = 0u64;
    for i in 0..1000u64 {
        acc = acc.wrapping_add(i);
    }
    let _ = acc;
    let elapsed = span.finish();
    assert!(elapsed >= 0.0);
}

#[wasm_bindgen_test]
fn mark_and_measure() {
    perf_core::clear_marks().ok();
    perf_core::clear_measures().ok();
    perf_core::mark("start").expect("mark start");
    perf_core::mark("end").expect("mark end");
    let dur = perf_core::measure("span", "start", "end").expect("measure");
    assert!(dur >= 0.0);
}

#[wasm_bindgen_test]
fn entry_names_of_marks() {
    perf_core::clear_marks().ok();
    perf_core::mark("alpha").expect("mark alpha");
    perf_core::mark("beta").expect("mark beta");
    let names = perf_core::entry_names_of_type("mark").expect("names");
    assert!(names.iter().any(|n| n == "alpha"));
    assert!(names.iter().any(|n| n == "beta"));
}

#[wasm_bindgen_test]
fn benchmark_returns_samples() {
    let iterations = 5u32;
    let result = perf_lab::run_benchmark("work", iterations, || {
        let mut acc = 0u64;
        for i in 0..1000u64 {
            acc = acc.wrapping_add(i.wrapping_mul(3));
        }
        let _ = acc;
    });
    // Контракт: count() == iterations. Прогрев не входит в результат.
    assert_eq!(result.count(), iterations as usize);
    assert!(result.mean_ms() >= 0.0);
    assert!(result.median_ms() >= 0.0);
}

#[wasm_bindgen_test]
fn bottleneck_is_slowest() {
    let a = BenchResult::new("fast", vec![1.0, 1.2, 1.1]);
    let b = BenchResult::new("slow", vec![10.0, 11.0, 10.5]);
    let idx = perf_lab::find_bottleneck(&[a, b]).unwrap();
    assert_eq!(idx, 1, "должна быть медленная задача");
}
```

---

## Команды запуска

```bash
# Host-тесты
cargo test --test perf_core_test

# Wasm-тесты
wasm-pack test --headless --chrome --test perf_lab_test

# Сборка wasm
npm run wasm:dev

# Запуск демо
npm run dev
```

---

## Демо-пример в `app.rs`

```rust
//! Демо Главы 31: Performance API — лаборатория производительности.
//!
//! Что показывает демо:
//!   • performance.now() — замер Rust-функций.
//!   • mark / measure — фазовая разметка прогона.
//!   • PerformanceObserver — события mark/measure/resource/navigation.
//!   • Resource Timing — медленные ресурсы страницы.
//!   • Navigation Timing — фазы загрузки документа.
//!   • Ранжирование задач: bottleneck сверху, стабильная — снизу.
//!
//! Фокусный подход: только демо этой главы.
//!
//! ПРАВИЛО: каждое `Rc<RefCell<T>>`, попадающее в `move`-замыкание,
//! получает отдельный клон с суффиксом `_cb` или `_in`.

use std::rc::Rc;

use wasm_bindgen::prelude::*;
use web_sys::HtmlElement;

use crate::perf_core;
use crate::perf_lab::{self, BenchResult, WARMUP_ITERATIONS};
use crate::perf_observer::PerfSubscription;
use crate::perf_timing;

const HTML: &str = r#"
<h1>Глава 31: Performance API</h1>
<p>Лаборатория: замер Rust-функций, поиск bottleneck, снимок тайминга.</p>

<fieldset style="margin:8px 0">
  <legend>Лаборатория</legend>
  <label>Замеров:
    <input id="iters" type="number" value="10" min="2" max="100" style="width:70px">
  </label>
  <button id="run">Прогнать</button>
  <button id="clear-perf">Очистить marks/measures</button>
  <div style="color:#666;font-size:12px;margin-top:6px">
    Прогрев — <b>3 вызова</b> задачи перед измерением, в статистику не входят.
    Замер синхронный — UI кратко недоступен во время прогона.
  </div>
</fieldset>

<div id="status" style="color:#555;margin:8px 0">Готово</div>

<fieldset style="margin:8px 0">
  <legend>Результаты (отсортированы по среднему, убывание)</legend>
  <div id="results-count" style="color:#555"></div>
  <div style="color:#666;font-size:12px">
    🔴 bottleneck — самое большое среднее время ·
    🟢 самая стабильная — минимальный stddev
  </div>
  <table id="results" style="width:100%;border-collapse:collapse;font-family:monospace;font-size:12px;margin-top:6px">
    <thead>
      <tr>
        <th style="text-align:left;border-bottom:1px solid #ccc">задача</th>
        <th style="text-align:right;border-bottom:1px solid #ccc">mean</th>
        <th style="text-align:right;border-bottom:1px solid #ccc">median</th>
        <th style="text-align:right;border-bottom:1px solid #ccc">min</th>
        <th style="text-align:right;border-bottom:1px solid #ccc">max</th>
        <th style="text-align:right;border-bottom:1px solid #ccc">stddev</th>
        <th style="text-align:right;border-bottom:1px solid #ccc">n</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>PerformanceObserver — события</legend>
  <div id="obs-count" style="color:#555">записей: 0</div>
  <button id="obs-refresh">↻ Обновить</button>
  <button id="obs-clear">Очистить буфер</button>
  <ul id="obs-list" style="font-family:monospace;font-size:12px;max-height:200px;overflow:auto"></ul>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Resource Timing — медленные ресурсы (&gt; 20 мс)</legend>
  <button id="res-refresh">↻ Обновить</button>
  <div id="res-hint" style="color:#666;font-size:12px;margin-top:4px">
    Если список пуст — все ресурсы отдаются быстро (или из кэша).
    Порог можно снизить, отредактировав вызов <code>slow_resources(20.0)</code>.
  </div>
  <ul id="res-list" style="font-family:monospace;font-size:12px;max-height:200px;overflow:auto"></ul>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Navigation Timing — фазы загрузки документа</legend>
  <button id="nav-refresh">↻ Обновить</button>
  <div style="color:#666;font-size:12px;margin-top:4px">
    0.000 ms означает, что фаза не выполнялась: соединение уже установлено
    или ресурс получен из кэша.
  </div>
  <div id="nav-phases" style="font-family:monospace;font-size:12px;margin-top:6px"></div>
</fieldset>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let status = document.get_element_by_id("status").ok_or("no status")?;
    let results_count = document.get_element_by_id("results-count").ok_or("no rc")?;
    let results_tbody = document
        .get_element_by_id("results")
        .and_then(|t| t.query_selector("tbody").ok().flatten())
        .ok_or("no tbody")?;
    let obs_count = document.get_element_by_id("obs-count").ok_or("no oc")?;
    let obs_list = document.get_element_by_id("obs-list").ok_or("no ol")?;
    let res_list = document.get_element_by_id("res-list").ok_or("no rl")?;
    let nav_phases = document.get_element_by_id("nav-phases").ok_or("no np")?;

    // Единый observer: mark, measure, resource, navigation.
    let subscription: Rc<PerfSubscription> = Rc::new(
        PerfSubscription::subscribe(&["mark", "measure", "resource", "navigation"])
            .map_err(|e| JsValue::from_str(&format!("subscribe: {e:?}")))?,
    );

    // ---------------- Кнопка «Прогнать» ----------------
    let run_btn: HtmlElement = document
        .get_element_by_id("run")
        .ok_or("no run")?
        .dyn_into()?;
    {
        let status_cb = status.clone();
        let results_count_cb = results_count.clone();
        let results_tbody_cb = results_tbody.clone();
        let subscription_cb = subscription.clone();
        let obs_count_cb = obs_count.clone();
        let obs_list_cb = obs_list.clone();
        set_click(&run_btn, move |_| {
            let iterations = get_input_u32("iters", 10).clamp(2, 100);

            // mark:start — виден в observer и в DevTools → Performance.
            let _ = perf_core::mark("bench:start");

            set_status(
                &status_cb,
                &format!("Прогон {iterations} замеров + {WARMUP_ITERATIONS} прогревов…"),
            );

            // Реестр задач: каждая — Rust-функция без аргументов.
            let tasks = build_tasks();
            let mut results: Vec<BenchResult> = Vec::with_capacity(tasks.len());
            for (name, mut task) in tasks {
                results.push(perf_lab::run_benchmark(name, iterations, &mut task));
            }

            // mark:end + measure. `bench:total` — единственный источник
            // истины для общего времени: его видит и observer, и статус.
            let _ = perf_core::mark("bench:end");
            let bench_total = perf_core::measure("bench:total", "bench:start", "bench:end")
                .unwrap_or(0.0);

            render_results_table(
                &mut results,
                &results_tbody_cb,
                &results_count_cb,
                bench_total,
            );

            // Автообновление observer — mark/measure уже в буфере.
            render_observer(&subscription_cb, &obs_count_cb, &obs_list_cb);

            let bottleneck = results
                .first()
                .map(|r| r.name.clone())
                .unwrap_or_default();
            set_status(
                &status_cb,
                &format!(
                    "Готово за {} мс. Bottleneck: {}.",
                    fmt_ms(bench_total),
                    bottleneck,
                ),
            );
        });
    }

    // ---------------- Кнопка «Очистить marks/measures» ----------------
    let clear_btn: HtmlElement = document
        .get_element_by_id("clear-perf")
        .ok_or("no clear")?
        .dyn_into()?;
    {
        let status_cb = status.clone();
        set_click(&clear_btn, move |_| {
            let _ = perf_core::clear_marks();
            let _ = perf_core::clear_measures();
            set_status(&status_cb, "marks/measures очищены");
        });
    }

    // ---------------- Observer: обновить список ----------------
    let obs_refresh: HtmlElement = document
        .get_element_by_id("obs-refresh")
        .ok_or("no obs-refresh")?
        .dyn_into()?;
    {
        let subscription_cb = subscription.clone();
        let obs_count_cb = obs_count.clone();
        let obs_list_cb = obs_list.clone();
        set_click(&obs_refresh, move |_| {
            render_observer(&subscription_cb, &obs_count_cb, &obs_list_cb);
        });
    }

    // ---------------- Observer: очистить буфер ----------------
    let obs_clear: HtmlElement = document
        .get_element_by_id("obs-clear")
        .ok_or("no obs-clear")?
        .dyn_into()?;
    {
        let subscription_cb = subscription.clone();
        let obs_count_cb = obs_count.clone();
        let obs_list_cb = obs_list.clone();
        set_click(&obs_clear, move |_| {
            subscription_cb.clear();
            render_observer(&subscription_cb, &obs_count_cb, &obs_list_cb);
        });
    }

    // ---------------- Resource Timing ----------------
    let res_refresh: HtmlElement = document
        .get_element_by_id("res-refresh")
        .ok_or("no res-refresh")?
        .dyn_into()?;
    {
        let res_list_cb = res_list.clone();
        let status_cb = status.clone();
        set_click(&res_refresh, move |_| {
            render_resources(&res_list_cb, &status_cb);
        });
    }

    // ---------------- Navigation Timing ----------------
    let nav_refresh: HtmlElement = document
        .get_element_by_id("nav-refresh")
        .ok_or("no nav-refresh")?
        .dyn_into()?;
    {
        let nav_phases_cb = nav_phases.clone();
        let status_cb = status.clone();
        set_click(&nav_refresh, move |_| {
            render_navigation(&nav_phases_cb, &status_cb);
        });
    }

    // Первичная отрисовка observer и navigation timing.
    render_observer(&subscription, &obs_count, &obs_list);
    render_navigation(&nav_phases, &status);

    Ok(())
}

// ---------------- задачи ----------------

/// Реестр задач. Каждая — замыкание без аргументов.
///
/// Явный `return` — правило проекта: `vec![]` с элементами
/// `Box<dyn FnMut()>` в хвостовой позиции теряет тип в rust-analyzer.
fn build_tasks() -> Vec<(&'static str, Box<dyn FnMut()>)> {
    return vec![
        ("fib(25)", Box::new(|| { let _ = fib(25); })),
        ("sum 100k u64", Box::new(|| { let _ = sum_u64(100_000); })),
        ("sort 10k f64", Box::new(|| { let _ = sort_random_f64(10_000); })),
        ("string concat 1k", Box::new(|| { let _ = concat_ints(1000); })),
        ("matrix 30×30", Box::new(|| { let _ = matmul_30x30(); })),
    ];
}

/// Наивная рекурсивная Fibonacci — типичная CPU-задача для бенчмарка.
fn fib(n: u32) -> u64 {
    if n < 2 {
        n as u64
    } else {
        fib(n - 1) + fib(n - 2)
    }
}

/// Сумма первых `n` чисел — плотная арифметика без аллокаций.
fn sum_u64(n: u64) -> u64 {
    let mut acc: u64 = 0;
    for i in 0..n {
        acc = acc.wrapping_add(i);
    }
    acc
}

/// Генерация псевдослучайных f64 и сортировка.
///
/// Литерал `2654435761` выходит за диапазон i32 — используем `u64`.
fn sort_random_f64(n: u64) -> f64 {
    let mut v: Vec<f64> = (0..n)
        .map(|i| (i.wrapping_mul(2_654_435_761u64) % 1_000_000) as f64)
        .collect();
    v.sort_by(|a, b| a.partial_cmp(b).unwrap_or(std::cmp::Ordering::Equal));
    v.first().copied().unwrap_or(0.0)
}

/// Конкатенация чисел в строку — много мелких аллокаций.
fn concat_ints(n: u32) -> usize {
    let mut s = String::new();
    for i in 0..n {
        s.push_str(&i.to_string());
    }
    s.len()
}

/// Умножение матриц 30×30 — плотная арифметика на f64.
fn matmul_30x30() -> f64 {
    let n = 30usize;
    let mut a = vec![0.0f64; n * n];
    let mut b = vec![0.0f64; n * n];
    for i in 0..n * n {
        a[i] = (i as f64) * 0.1;
        b[i] = (i as f64) * 0.2;
    }
    let mut c = vec![0.0f64; n * n];
    for i in 0..n {
        for j in 0..n {
            let mut acc = 0.0;
            for k in 0..n {
                acc += a[i * n + k] * b[k * n + j];
            }
            c[i * n + j] = acc;
        }
    }
    c[0]
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

fn get_input_u32(id: &str, default: u32) -> u32 {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<web_sys::HtmlInputElement>().ok())
        .and_then(|i| i.value().parse::<u32>().ok())
        .unwrap_or(default)
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

/// Единое форматирование миллисекунд — иначе в таблице разнобой.
fn fmt_ms(v: f64) -> String {
    return format!("{:.3}", v);
}

/// Строит таблицу результатов, сортирует по убыванию среднего,
/// подсвечивает bottleneck и стабильную задачу.
fn render_results_table(
    results: &mut Vec<BenchResult>,
    tbody: &web_sys::Element,
    count_el: &web_sys::Element,
    total_ms: f64,
) {
    // Сортировка по убыванию: bottleneck сверху.
    results.sort_by(|a, b| {
        b.mean_ms()
            .partial_cmp(&a.mean_ms())
            .unwrap_or(std::cmp::Ordering::Equal)
    });

    let bottleneck_idx = perf_lab::find_bottleneck(results);
    let stable_idx = perf_lab::find_most_stable(results);

    let mut html = String::with_capacity(1024);
    for (i, r) in results.iter().enumerate() {
        let is_bottleneck = Some(i) == bottleneck_idx;
        let is_stable = Some(i) == stable_idx && results.len() > 1;
        let row_style = if is_bottleneck {
            "background:#ffe6e6"
        } else if is_stable {
            "background:#e6ffe6"
        } else {
            ""
        };
        let marker = if is_bottleneck {
            " 🔴"
        } else if is_stable {
            " 🟢"
        } else {
            ""
        };
        html.push_str(&format!(
            "<tr style=\"{row_style}\">\
             <td>{}{marker}</td>\
             <td style=\"text-align:right\">{}</td>\
             <td style=\"text-align:right\">{}</td>\
             <td style=\"text-align:right\">{}</td>\
             <td style=\"text-align:right\">{}</td>\
             <td style=\"text-align:right\">{}</td>\
             <td style=\"text-align:right\">{}</td>\
             </tr>",
            esc(&r.name),
            fmt_ms(r.mean_ms()),
            fmt_ms(r.median_ms()),
            fmt_ms(r.min_ms()),
            fmt_ms(r.max_ms()),
            fmt_ms(r.std_dev_ms()),
            r.count(),
        ));
    }
    tbody.set_inner_html(&html);

    let bn = bottleneck_idx
        .map(|i| results[i].name.clone())
        .unwrap_or_default();
    let st = stable_idx
        .map(|i| results[i].name.clone())
        .unwrap_or_default();
    count_el.set_text_content(Some(&format!(
        "задач: {} | всего: {} мс | bottleneck: {} | стабильная: {}",
        results.len(),
        fmt_ms(total_ms),
        bn,
        st,
    )));
}

/// Рендерит события observer.
///
/// Различие mark и measure: у `mark` нет длительности — это точка
/// во времени. Показывать её как «0.000 ms» некорректно; вместо
/// этого выводим только `start_time`. У `measure` есть `duration`.
fn render_observer(
    sub: &Rc<PerfSubscription>,
    count_el: &web_sys::Element,
    list_el: &web_sys::Element,
) {
    let events = sub.events();
    count_el.set_text_content(Some(&format!("записей: {}", events.len())));
    if events.is_empty() {
        list_el.set_inner_html("<li><em>событий нет</em></li>");
        return;
    }
    let mut html = String::with_capacity(256);
    for e in events.iter().rev().take(50) {
        let line = if e.entry_type == "mark" {
            format!(
                "<li>[{}] {} — точка @ {} ms</li>",
                esc(&e.entry_type),
                esc(&e.name),
                fmt_ms(e.start_time),
            )
        } else {
            format!(
                "<li>[{}] {} — {} ms @ {} ms</li>",
                esc(&e.entry_type),
                esc(&e.name),
                fmt_ms(e.duration),
                fmt_ms(e.start_time),
            )
        };
        html.push_str(&line);
    }
    list_el.set_inner_html(&html);
}

fn render_resources(list_el: &web_sys::Element, status_el: &web_sys::Element) {
    match perf_timing::slow_resources(20.0) {
        Ok(items) if items.is_empty() => {
            list_el.set_inner_html("<li><em>нет ресурсов медленнее 20 мс</em></li>");
        }
        Ok(items) => {
            let mut html = String::with_capacity(512);
            for r in &items {
                html.push_str(&format!(
                    "<li>{} ms | {} байт | {} | {}</li>",
                    fmt_ms(r.duration),
                    r.transfer_size as u64,
                    esc(&r.initiator_type),
                    esc(&r.name),
                ));
            }
            list_el.set_inner_html(&html);
            set_status(status_el, &format!("найдено {} ресурсов", items.len()));
        }
        Err(e) => {
            list_el.set_inner_html(&format!("<li>Ошибка: {e:?}</li>"));
        }
    }
}

fn render_navigation(el: &web_sys::Element, status_el: &web_sys::Element) {
    match perf_timing::navigation() {
        Ok(Some(p)) => {
            let html = format!(
                "redirect: {} ms<br>\
                 dns: {} ms<br>\
                 tcp+tls: {} ms<br>\
                 request: {} ms<br>\
                 response: {} ms<br>\
                 dom_interactive: {} ms<br>\
                 dom_complete: {} ms<br>\
                 load: {} ms<br>\
                 <b>total: {} ms</b>",
                fmt_ms(p.redirect),
                fmt_ms(p.dns),
                fmt_ms(p.tcp_tls),
                fmt_ms(p.request),
                fmt_ms(p.response),
                fmt_ms(p.dom_interactive),
                fmt_ms(p.dom_complete),
                fmt_ms(p.load_event),
                fmt_ms(p.total()),
            );
            el.set_inner_html(&html);
        }
        Ok(None) => el.set_inner_html("<em>navigation timing недоступен</em>"),
        Err(e) => set_status(status_el, &format!("nav error: {e:?}")),
    }
}

/// HTML-escape. Явный `return` — правило проекта: цепочки `.replace()`
/// в хвостовой позиции функции теряют тип в rust-analyzer.
fn esc(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}
```

---

## Анализ решения и реализованные улучшения

**Что работает хорошо:**

- `perf_core::Span` — RAII-обёртка для замера.
- `perf_observer` — единая подписка на 4 типа записей, события в `Rc<RefCell>`.
- `perf_timing` — resource + navigation timing в типизированных структурах, фазы нормализованы через `.max(0.0)`.
- `perf_lab` — реестр замеров, статистика, ранжирование.
- Демо — таблица с подсветкой bottleneck и стабильной задачи.
- Прогрев (`WARMUP_ITERATIONS = 3`) — не портит статистику.

**Реализованные улучшения:**

1. **Контракт `run_benchmark` изменён:** `iterations` — число **измеренных** сэмплов, прогрев отдельно. `BenchResult::count() == iterations`.
2. **Observer различает mark/measure:** для `mark` выводится «точка @ X ms», для остальных — «duration @ start».
3. **Единый источник total:** `bench_total` из `measure` используется и в статусе, и в `results-count` — рассинхрон устранён.
4. **`get_entries_by_type` — `js_sys::Array` напрямую:** убран `.map_err`.
5. **`mark` возвращает `PerformanceMark`:** `.map(|_| ())`.
6. **`clear_marks`/`clear_measures` — без `Result`:** вызов + `Ok(())`.
7. **`measure` через `Reflect`:** типизированного метода с тремя строками в `web-sys 0.3.105` нет.
8. **`PerformanceObserverInit` через `js_sys::Object`:** dictionary требует обязательный аргумент в `new`.
9. **`observer.observe(...)` через `Reflect`:** в `web-sys 0.3.105` нет `observe_with_options`.
10. **`format!` в tail-позиции:** `format_result` и `esc` используют явный `return`.
11. **`vec![Box<dyn FnMut()>]` в tail-позиции:** `build_tasks` использует `return vec![...]`.
12. **Задачи вынесены в отдельные функции:** `fib`, `sum_u64`, `sort_random_f64`, `concat_ints`, `matmul_30x30` — вместо длинных инлайн-замыканий.
13. **`render_results_table` вынесен в отдельную функцию:** обработчик кнопки стал коротким.
14. **Литерал `2654435761` → `2_654_435_761u64`:** не влезает в `i32` без суффикса.
15. **Автообновление observer** после прогона: не нужно кликать «↻ Обновить».
16. **Сноски для пустых/нулевых результатов:** в Resource Timing и Navigation Timing — пояснения.
17. **Колонка `n`** в таблице — видно число сэмплов.

## Правила проекта (закрепляем)

1. **`performance.now()` для замеров и троттлинга.** Монотонные миллисекунды не «прыгают».
2. **`mark`/`measure` — фазовая разметка.** Каждая фаза инициализации — пара mark'ов и measure.
3. **Прогрев обязателен.** Первые 3 вызова задачи не попадают в статистику.
4. **Контракт `iterations` — число сэмплов, не «итераций всего».** `count() == iterations`.
5. **Bottleneck — по `mean`, не по `min`/`max`.** `min` — идеальный случай, `max` — выброс.
6. **`stddev` показывает стабильность.** Большой stddev → шумные замеры.
7. **`mark` не имеет `duration`.** Показывать «0 мс» для mark — семантическая ошибка.
8. **Cross-origin resources ограничены.** `transfer_size` без `Timing-Allow-Origin` не отдаётся.
9. **`format!` и `vec![Box<dyn ...>]` в tail-позиции — всегда `return`.**
10. **`PerformanceObserverInit` и `observe` — через `Reflect` в `web-sys 0.3.105`.**
11. **`vec![Box<dyn FnMut()>]` в tail-позиции — `return vec![...]`.**
12. **Длинная логика — в отдельных функциях.** Обработчик кнопки — короткий, реестр задач и рендер — отдельные функции.
13. **Числовые литералы, превышающие `i32`, — с явным суффиксом.** `2_654_435_761u64`.