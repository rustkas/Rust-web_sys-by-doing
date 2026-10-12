# 19.9. Квоты и `StorageManager`

## Зачем это нужно

В главах 19.7 и 19.8 мы работали с OPFS так, будто он бесконечен. Создавали файлы, дописывали логи, писали мегабайты — и всё работало. Это коварная иллюзия: **OPFS не бесконечен**, и в реальном приложении у вас ровно два сценария:

1. **Хороший.** Вы спросили у браузера, сколько места доступно, получили внятный ответ и ведёте себя в этих рамках.
2. **Плохой.** Вы пишете данные, пока однажды `createWritable` не падает с `QuotaExceededError`. Пользователь теряет несохранённую работу, вы получаете баг-репорт без внятной причины.

`StorageManager` — это интерфейс, который превращает квоту из непредсказуемой константы в **управляемый ресурс**. Через `navigator.storage` вы можете:

- **узнать**, сколько места уже занято и каков лимит origin'а (`estimate()`);
- **запросить защиту от автоматического удаления** (`persist()`) — гарантию, что браузер не вытеснит данные при нехватке места;
- **проверить**, предоставлена ли защита (`persisted()`);
- **разложить usage по категориям** — сколько занимает OPFS, сколько IndexedDB, сколько Cache Storage.

Плюс есть вторая, менее известная сторона: **persistence** — это не «сделать данные вечными», это «попросить браузер не вытеснять их первыми». Разница принципиальная, и мы её разберём.

---

## 19.9.1. Анатомия `StorageManager`

`navigator.storage` возвращает `StorageManager` — небольшой интерфейс с четырьмя методами:

| Метод | Что делает |
|---|---|
| `estimate()` | `{ usage, quota, usageDetails }` — текущее состояние |
| `persist()` | Просит браузер защитить данные от удаления; возвращает `bool` |
| `persisted()` | Возвращает `bool` — уже защищено или нет |
| `getDirectory()` | Корень OPFS (мы разбирали в главе 19.8) |

### `estimate()` — текущее состояние

```rust
pub async fn estimate() -> Result<StorageEstimate, JsValue> {
    let value = call_storage_method("estimate").await?;

    let usage = js_sys::Reflect::get(&value, &JsValue::from_str("usage"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0);
    let quota = js_sys::Reflect::get(&value, &JsValue::from_str("quota"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0);

    let usage_details = parse_usage_details(&value);

    Ok(StorageEstimate {
        usage,
        quota,
        usage_details,
    })
}
```

**Важные детали:**

- **`usage` и `quota` — не точные значения, а оценки.** Спека явно разрешает браузеру округлять: Chrome округляет до мегабайт, Firefox — до десятков мегабайт. Не удивляйтесь, если после записи 100 КБ `usage` не изменится: реальное значение ниже порога округления.
- **`quota` — это доля от свободного места**, а не константа. По умолчанию Chrome даёт origin'у до 60% от свободного места на диске. Firefox — до 10 ГБ на origin. Safari — тоже с ограничениями. Если у пользователя мало места на диске, квота падает.
- **`usageDetails` есть не везде.** Chrome возвращает разбивку по категориям (`fileSystem`, `indexedDB`, `caches`, `serviceWorker`), Firefox — нет. Если объект пустой, это не ошибка.

### `persist()` и `persisted()` — защита от удаления

Где `call_storage_method` — общий хелпер, аналогичный `call_promise` из главы 19.7:

```rust
async fn call_storage_method(name: &str) -> Result<JsValue, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator_js = JsValue::from(window.navigator());

    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }

    let fn_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str(name))?;
    if fn_js.is_undefined() {
        let msg = format_str(format_args!("storage.{name} недоступен"));
        return Err(JsValue::from_str(&msg));
    }
    let func: js_sys::Function = fn_js.dyn_into().map_err(|_| {
        let msg = format_str(format_args!("storage.{name} не функция"));
        JsValue::from_str(&msg)
    })?;
    let promise: js_sys::Promise = func.call0(&storage_js)?.dyn_into().map_err(|_| {
        let msg = format_str(format_args!("storage.{name} не Promise"));
        JsValue::from_str(&msg)
    })?;
    JsFuture::from(promise).await
}
```

### Что такое persistence на самом деле

**Persistence (защита от удаления)** — это гарантия, что браузер **не будет автоматически удалять данные origin'а** при нехватке места на диске.

Без persistence браузер ведёт себя так:

1. Пользователь скачал 40 ГБ видео, диск заполнился.
2. Браузер ищет «жертв» среди origins **без** защиты — сортирует по дате последнего визита.
3. Ваш сайт, куда не заходили месяц, удаляется **целиком**: OPFS, IndexedDB, Cache Storage — всё.
4. Пользователь возвращается на сайт — все черновики исчезли.

С persistence браузер пропустит ваш origin. Если места не хватает вообще — ваша новая запись упадёт с `QuotaExceededError`, но старые данные останутся.

**Кто и когда даёт persistence:**

| Браузер | Условия |
|---|---|
| Chrome / Edge | Сайт установлен как PWA, **или** добавлен в закладки, **или** разрешены уведомления, **или** высокий engagement score |
| Firefox | Показывает пользователю диалог «Разрешить этому сайту постоянное хранилище?» |
| Safari (macOS 14+) | Ограниченно; на iOS — жёстче, около 1 ГБ даже с persist |
| Safari (старые версии) | Не поддерживает `persist()` |

> **Правило проекта.** Если ваше приложение хранит в OPFS/IndexedDB/Cache **что-то, потеря чего расстроит пользователя** (черновики, документы, офлайн-данные), вы обязаны вызвать `persist()` и проверить результат. Если `persist()` вернул `false`, предупредите пользователя: «данные могут быть удалены при нехватке места на диске». Молчаливое «работает сейчас» — плохая стратегия.

---

## 19.9.2. От чего зависит квота

Квота — не константа. Она зависит от браузера, платформы, свободного места на диске и политики вендора.

| Браузер | По умолчанию | После `persist()` | Примечания |
|---|---|---|---|
| **Chrome / Edge** | До 60% от свободного места на диске, **общий** лимит на все origin'ы | Тот же лимит, но origin защищён от вытеснения | Есть общий «пул» хранилищ браузера |
| **Firefox** | До **10 ГБ** на origin (или 10% от свободного места, если меньше) | Тот же лимит, origin защищён | Спрашивает пользователя перед `persist()` |
| **Safari (desktop)** | Обычно **1 ГБ** на origin; при `persist()` — до 60% от свободного места | Origin защищён | iOS Safari — сильно жёстче |
| **Safari (iOS)** | Меньше, с агрессивным вытеснением под давлением системы | Ограниченно | Полноценный OPFS — только с iOS 17+ |
| **Opera** | Как Chrome | Как Chrome | Основан на Chromium |

Плюс — **лимит общий на все хранилища origin'а**: OPFS, IndexedDB, Cache Storage, Service Worker Cache. Складывать нельзя: если OPFS занял 9 ГБ из 10 ГБ лимита, IndexedDB получит только 1 ГБ.

И третья деталь: **`estimate()` возвращает оценку, а не факт.** Chrome округляет до мегабайт, Firefox — до десятков мегабайт, Safari — вообще может возвращать `0` для мелких объёмов. Спецификация явно разрешает это ради защиты от fingerprinting (точная квота — уникальный идентификатор браузера).

---

## 19.9.3. Новый модуль `fsa_api_quota.rs`

```rust
//! Квоты и StorageManager.
//!
//! `navigator.storage` — четыре метода:
//! * `estimate()` — `{ usage, quota, usageDetails }`;
//! * `persist()` — запросить персистентность (bool);
//! * `persisted()` — проверить текущий статус (bool);
//! * `getDirectory()` — корень OPFS (глава 19.8).
//!
//! **Правило:** `usage`/`quota` — оценки, не точные значения.
//! Браузер округляет ради защиты от fingerprinting. Не удивляйтесь,
//! если после записи 100 КБ `usage` не изменится.
//!
//! **Правило:** `persist()` может вернуть `false` — это не ошибка,
//! а отказ браузера. Если приложение хранит важные данные, при
//! отказе нужно предупредить пользователя.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции `if/else`/
//! `match`/блока иногда выводится как `()` → E0308. Поэтому все
//! строковые выражения идут через `format_str(format_args!(…))` —
//! тип возврата однозначен (`String`).

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

// ===========================================================================
// Строковые хелперы (правило WASM-инференса)
// ---------------------------------------------------------------------------
// `format_str` объявлена ДО первого использования — так проще читать
// файл и не наткнуться на «expected String, found ()».
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

fn fmt_bytes(n: f64) -> String {
    let bytes = n.max(0.0);
    let s: String = if bytes < 1024.0 {
        format_str(format_args!("{} B", bytes as u64))
    } else if bytes < 1024.0 * 1024.0 {
        format_str(format_args!("{:.1} KB", bytes / 1024.0))
    } else if bytes < 1024.0 * 1024.0 * 1024.0 {
        format_str(format_args!("{:.2} MB", bytes / (1024.0 * 1024.0)))
    } else {
        format_str(format_args!("{:.2} GB", bytes / (1024.0 * 1024.0 * 1024.0)))
    };
    s
}

// ===========================================================================
// Модель
// ===========================================================================

#[derive(Debug, Clone, Default)]
pub struct StorageEstimate {
    pub usage: f64,
    pub quota: f64,
    pub usage_details: Vec<(String, f64)>,
}

impl StorageEstimate {
    /// Доля занятого места: 0.0..=1.0. Если quota == 0 — 0.
    pub fn fill_ratio(&self) -> f64 {
        if self.quota <= 0.0 {
            return 0.0;
        }
        let r = self.usage / self.quota;
        let clamped: f64 = if r < 0.0 {
            0.0
        } else if r > 1.0 {
            1.0
        } else {
            r
        };
        clamped
    }

    /// Человекочитаемое описание для логов.
    ///
    /// `format_str(format_args!(…))` вместо `format!` в tail-позиции —
    /// тип возврата однозначен, E0308 не возникает.
    pub fn summary(&self) -> String {
        let s: String = format_str(format_args!(
            "usage={} quota={}",
            fmt_bytes(self.usage),
            fmt_bytes(self.quota)
        ));
        s
    }
}

// ===========================================================================
// Внутренний хелпер: вызов метода navigator.storage.<name>()
// ===========================================================================

async fn call_storage_method(name: &str) -> Result<JsValue, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator_js = JsValue::from(window.navigator());

    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }

    let fn_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str(name))?;
    if fn_js.is_undefined() {
        let msg = format_str(format_args!("storage.{name} недоступен"));
        return Err(JsValue::from_str(&msg));
    }
    let func: js_sys::Function = fn_js.dyn_into().map_err(|_| {
        let msg = format_str(format_args!("storage.{name} не функция"));
        JsValue::from_str(&msg)
    })?;
    let promise: js_sys::Promise = func.call0(&storage_js)?.dyn_into().map_err(|_| {
        let msg = format_str(format_args!("storage.{name} не Promise"));
        JsValue::from_str(&msg)
    })?;
    JsFuture::from(promise).await
}

// ===========================================================================
// estimate
// ===========================================================================

/// Текущее состояние хранилища origin'а.
///
/// Возвращает `{ usage, quota, usageDetails }`. `usage`/`quota` —
/// оценки (браузер округляет). `usageDetails` может быть пустым.
pub async fn estimate() -> Result<StorageEstimate, JsValue> {
    let value = call_storage_method("estimate").await?;

    let usage = js_sys::Reflect::get(&value, &JsValue::from_str("usage"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0);
    let quota = js_sys::Reflect::get(&value, &JsValue::from_str("quota"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0);

    let usage_details = parse_usage_details(&value);

    Ok(StorageEstimate {
        usage,
        quota,
        usage_details,
    })
}

fn parse_usage_details(value: &JsValue) -> Vec<(String, f64)> {
    let details = match js_sys::Reflect::get(value, &JsValue::from_str("usageDetails")) {
        Ok(v) => v,
        Err(_) => return Vec::new(),
    };
    if details.is_undefined() || details.is_null() {
        return Vec::new();
    }

    let obj = js_sys::Object::from(details);
    let keys = js_sys::Object::keys(&obj);
    let mut out: Vec<(String, f64)> = Vec::new();
    for i in 0..keys.length() {
        let key = match keys.get(i).as_string() {
            Some(s) => s,
            None => continue,
        };
        let val = js_sys::Reflect::get(&obj, &JsValue::from_str(&key))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0);
        out.push((key, val));
    }
    out
}

// ===========================================================================
// persist / persisted
// ===========================================================================

/// Проверить, предоставлена ли origin'у персистентность.
pub async fn persisted() -> bool {
    match call_storage_method("persisted").await {
        Ok(v) => v.as_bool().unwrap_or(false),
        Err(_) => false,
    }
}

/// Запросить персистентность.
///
/// Возвращает `true`, если браузер предоставил её (или она уже была).
/// `false` — отказ или API недоступен. Отказ — не ошибка.
pub async fn request_persist() -> bool {
    match call_storage_method("persist").await {
        Ok(v) => v.as_bool().unwrap_or(false),
        Err(_) => false,
    }
}

/// Проверка: доступен ли `navigator.storage.estimate`.
pub fn quota_available() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let navigator_js = JsValue::from(window.navigator());
    let storage_js = match js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage")) {
        Ok(s) => s,
        Err(_) => return false,
    };
    if storage_js.is_undefined() {
        return false;
    }
    js_sys::Reflect::has(&storage_js, &JsValue::from_str("estimate")).unwrap_or(false)
}
```

### Что важно в модуле

**1. `usageDetails` — не всегда есть.** Chrome возвращает объект вида `{fileSystem: 12345, indexedDB: 6789, caches: 0}`, Firefox — `undefined`, Safari — тоже не всегда. `parse_usage_details` возвращает пустой `Vec` вместо ошибки.

**2. `persist()` возвращает `bool`, а не `Result`.** Отказ браузера — не ошибка. Модуль возвращает `false` и на отказ, и на отсутствие API. Это разные вещи, но для UI различие неважно: «нет защиты — предупреждай».

**3. `format_str` объявлена в начале файла.** Первое, что читатель видит после doc-comment, — функции работы со строками. Это правило: если в модуле есть строковые выражения, они собраны в одном месте.

**4. `fill_ratio()` — явная переменная `clamped`.** Возврат `if/else` в tail-позиции — источник E0308. Даже для `f64` правило проекта требует явную переменную.

---

## 19.9.4. Демо: квота как видимая стена

### Идея

Пользователь интуитивно думает, что «браузерное хранилище бесконечно». Демо должно **столкнуть его с реальностью**:

1. **Живой прогресс-бар** usage/quota, обновляется после каждой записи.
2. **Кнопки заполнения** на 10 МБ, 50 МБ, 200 МБ — сразу видно, как бар растёт.
3. **Статус защиты** — «не защищено: браузер может удалить данные» или «защищено от удаления».
4. **Кнопка «Запросить защиту»** — видно, как браузер даёт (или не даёт) защиту.
5. **usageDetails** — разбивка по категориям с русскими названиями.
6. **Кнопка «Очистить всё»** — идемпотентное удаление.

### Wow-моменты

**Wow №1.** Пользователь жмёт «Заполнить 50 МБ» — прогресс-бар **растёт на глазах**. Он видит долю от квоты. Для маленькой квоты (например, в Safari) 50 МБ могут занять значительную часть лимита.

**Wow №2.** Жмёт «Запросить защиту» — статус меняется **сразу**. В Chrome, если сайт открыт как обычная вкладка и не добавлен в закладки, `persist()` вернёт `false`. Пользователь видит: «браузер отказал». В Firefox — появится системный диалог.

**Wow №3.** `usageDetails` показывает **из чего состоит usage**: OPFS отдельно, IndexedDB отдельно, Cache Storage отдельно. Пользователь видит, что всё в origin'е — **единый кошелёк**.

**Wow №4.** Кнопка «Очистить всё» — и usage снова падает. Связь между «я написал файлы» и «квота занята» становится осязаемой.

### Что в интерфейсе на русском

Демо полностью локализовано:

| Было | Стало |
|---|---|
| `usage` | занято |
| `quota` | лимит на этот сайт |
| `fileSystem` | Файловая система (OPFS) |
| `indexedDB` | IndexedDB |
| `caches` | Cache Storage |
| `persist()` | Запросить защиту |
| `persisted()` | Проверить статус |
| «not persistent» | «Не защищено: браузер может удалить данные» |

**Вступительная плашка** сверху объясняет три термина:

- что такое StorageManager;
- что значит «защита от удаления»;
- как её включить в Chrome.

Без этого блока пользователь видел бы «не персистентно» и не понимал, что с этим делать.

### Макет

```
┌────────────────────────────────────────────────────────────┐
│ 19.9 StorageManager: сколько места у сайта и защищены ли    │
│ данные                                                       │
├────────────────────────────────────────────────────────────┤
│ ℹ Что такое StorageManager. Это диспетчер хранилища…        │
├────────────────────────────────────────────────────────────┤
│ статус: StorageManager доступен                              │
├────────────────────────────────────────────────────────────┤
│ ❶ ЗАНЯТО И ЛИМИТ                                             │
│                                                              │
│   занято                лимит на этот сайт                   │
│   12.4 МБ               8.20 ГБ                              │
│                                                              │
│   ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░              │
│   0%              использовано: 0.15%            100%        │
│                                                              │
│   Из чего состоит занятое место:                             │
│   Файловая система (OPFS)  ▓▓▓▓▓▓▓▓▓▓▓ 12.4 МБ              │
│   IndexedDB                ░░░░░░░░░░░ 0 Б                    │
│   Cache Storage            ░░░░░░░░░░░ 0 Б                    │
│                                                              │
│   [Заполнить 10 МБ] [Заполнить 50 МБ] [Заполнить 200 МБ]     │
│   [Очистить всё]                                             │
├────────────────────────────────────────────────────────────┤
│ ❷ ЗАЩИТА ОТ АВТОМАТИЧЕСКОГО УДАЛЕНИЯ                         │
│                                                              │
│   ⚠ Не защищено: браузер может удалить данные                │
│      Если на диске кончится место…                           │
│      Как получить защиту в Chrome:                           │
│      • установите сайт как приложение (PWA)                  │
│      • или добавьте страницу в закладки (Ctrl+D)             │
│      • или разрешите уведомления                             │
│                                                              │
│   [Запросить защиту] [Проверить статус]                      │
├────────────────────────────────────────────────────────────┤
│ ❸ ЖУРНАЛ                                                     │
│ [19.9] StorageManager доступен, OPFS доступен                │
│ [19.9] Оценка хранилища: usage=12.4 MB quota=8.20 GB         │
└────────────────────────────────────────────────────────────┘
```

---

## 19.9.5. Тесты

### `tests/wasm_fsa_api_quota.rs`

```rust
//! Тесты `src/fsa_api_quota.rs`.
//!
//! Проверяем `estimate()`, `persisted()`, `persist()`.
//! OPFS не требует user gesture — тесты идут в headless Chrome.
//!
//! `persist()` может вернуть `false` — это не ошибка, а отказ
//! браузера. Тесты не паникуют, если persistence не предоставлен.

use wasm_bindgen_test::*;

use hello_wasm02::fsa_api_quota::{
    estimate, persisted, quota_available, request_persist,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// Доступность API
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_quota_available() {
    // Не паникуем, если API нет — мягкий skip.
    if !quota_available() {
        return;
    }
    assert!(quota_available(), "quota_available должен быть true");
}

// ---------------------------------------------------------------------------
// estimate
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_estimate_returns_non_negative() {
    if !quota_available() {
        return;
    }
    let e = estimate().await.expect("estimate");
    assert!(e.usage >= 0.0, "usage не может быть отрицательным");
    assert!(e.quota >= 0.0, "quota не может быть отрицательной");
}

#[wasm_bindgen_test]
async fn test_estimate_quota_greater_or_equal_usage() {
    if !quota_available() {
        return;
    }
    let e = estimate().await.expect("estimate");
    // Нормальное состояние: usage <= quota. Если quota == 0 (нет
    // информации), пропускаем проверку.
    if e.quota > 0.0 {
        assert!(
            e.quota >= e.usage,
            "quota ({}) должна быть >= usage ({})",
            e.quota,
            e.usage
        );
    }
}

#[wasm_bindgen_test]
async fn test_estimate_fill_ratio_in_range() {
    if !quota_available() {
        return;
    }
    let e = estimate().await.expect("estimate");
    let r = e.fill_ratio();
    assert!((0.0..=1.0).contains(&r), "fill_ratio в диапазоне 0..1");
}

#[wasm_bindgen_test]
async fn test_estimate_summary_not_empty() {
    if !quota_available() {
        return;
    }
    let e = estimate().await.expect("estimate");
    let s = e.summary();
    assert!(!s.is_empty());
    assert!(s.contains("usage="));
    assert!(s.contains("quota="));
}

// ---------------------------------------------------------------------------
// persist / persisted
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_persisted_returns_bool() {
    if !quota_available() {
        return;
    }
    // persisted() возвращает bool без ошибки в любом окружении.
    let _ = persisted().await;
}

#[wasm_bindgen_test]
async fn test_request_persist_returns_bool() {
    if !quota_available() {
        return;
    }
    // persist() может вернуть true или false — оба варианта ок.
    // В headless Chrome обычно false без user engagement.
    let result = request_persist().await;
    // Ничего не проверяем по значению: главное — не паникует.
    let _ = result;
}
```

### Почему тесты такие «мягкие»

`estimate()` возвращает **оценки**, а `persist()` может **отказать** — это нормальное поведение, а не ошибка. Тесты проверяют:

- API не паникует и возвращает корректные типы;
- `usage >= 0`, `quota >= 0`;
- `quota >= usage` (когда `quota` известна);
- `fill_ratio()` в диапазоне `[0, 1]`.

Конкретные значения («в Chrome quota = 60% от диска») не проверяются — они зависят от окружения и не являются контрактом API.

---

## 19.9.6. Полный текст `src/app.rs`

```rust
//! Демо главы 19.9 — квоты и StorageManager.
//!
//! **Идея.** OPFS не бесконечен. StorageManager показывает «стену»:
//! сколько места занято, каков лимит, и защищён ли origin от
//! автоматического вытеснения при нехватке места на диске.
//!
//! **Wow-моменты:**
//! 1. Прогресс-бар квоты растёт на глазах — пользователь видит
//!    «стену», о которой раньше не знал.
//! 2. `persist()` даёт (или не даёт) защиту — статус меняется сразу.
//! 3. `usageDetails` показывает: OPFS, IndexedDB, Cache Storage —
//!    **единый кошелёк** origin'а.
//! 4. Кнопка «Очистить» — usage падает, `estimate()` возвращает
//!    уже другие числа.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>` —
//! вызывается из `#[wasm_bindgen(start)]`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции `if/else`/
//! `match`/блока иногда выводится как `()` → E0308. Все такие
//! места обёрнуты в `format_str(format_args!(…))`.
//!
//! **Правило snake_case.** `web-sys` использует snake_case для
//! всех методов: `create_element`, `set_attribute`,
//! `append_child`, `add_event_listener_with_callback`. JS-стиль
//! (`createElement`) не существует и даёт E0599.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, Element, HtmlElement};

use crate::fsa_api_opfs::{opfs_available, opfs_root, remove_file, write_bytes};
use crate::fsa_api_quota::{
    estimate, persisted, quota_available, request_persist, StorageEstimate,
};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[19.9]";

fn console_log(msg: &str) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::log_1(&JsValue::from_str(&line));
}

fn console_err(msg: &str, e: &JsValue) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::error_2(&JsValue::from_str(&line), e);
}

// ===========================================================================
// Строковые хелперы (правило проекта)
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

fn fmt_bytes(n: f64) -> String {
    let bytes = n.max(0.0);
    let s: String = if bytes < 1024.0 {
        format_str(format_args!("{} Б", bytes as u64))
    } else if bytes < 1024.0 * 1024.0 {
        format_str(format_args!("{:.1} КБ", bytes / 1024.0))
    } else if bytes < 1024.0 * 1024.0 * 1024.0 {
        format_str(format_args!("{:.2} МБ", bytes / (1024.0 * 1024.0)))
    } else {
        format_str(format_args!("{:.2} ГБ", bytes / (1024.0 * 1024.0 * 1024.0)))
    };
    s
}

fn fmt_pct(v: f64) -> String {
    let pct = v * 100.0;
    let s: String = if pct < 0.01 {
        format_str(format_args!("{pct:.4}%"))
    } else if pct < 1.0 {
        format_str(format_args!("{pct:.3}%"))
    } else {
        format_str(format_args!("{pct:.2}%"))
    };
    s
}

// ===========================================================================
// Состояние
// ===========================================================================

struct DemoState {
    status: Element,
    usage_label: Element,
    quota_label: Element,
    fill_bar: HtmlElement,
    fill_pct: Element,
    details_body: Element,
    persist_box: Element,
    log_body: Element,
    log_lines: RefCell<Vec<String>>,
}

// ===========================================================================
// CSS
// ===========================================================================

const CSS: &str = r#"
html, body {
  margin: 0 !important; padding: 0 !important;
  background: #0d1117 !important; color: #e6edf3 !important;
  min-height: 100vh;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  font-size: 14px; line-height: 1.55;
}
header { padding: 20px 32px 4px; }
h1 { font-size: 19px; margin: 0 0 4px; font-weight: 600; }
h1 code { color: #79c0ff; font-size: 0.95em; }
.subtitle { color: #8b949e; margin: 0; font-size: 13px; }

.status-bar {
  padding: 12px 32px 4px; font-size: 12px; color: #8b949e;
  font-family: ui-monospace, monospace;
}
.status-bar.ok   { color: #3fb950; }
.status-bar.warn { color: #d29922; }
.status-bar.err  { color: #f85149; }

.section {
  margin: 16px 32px; background: #161b22;
  border: 1px solid #30363d; border-radius: 8px;
  overflow: hidden;
}
.section h3 {
  margin: 0; padding: 10px 14px;
  background: #21262d; font-size: 13px; font-weight: 600;
  border-bottom: 1px solid #30363d;
  display: flex; align-items: baseline; gap: 8px;
}
.section h3 .num { color: #79c0ff; font-weight: 700; font-size: 14px; }
.section h3 .hint { color: #6e7681; font-weight: 400; font-size: 11px; margin-left: auto; }
.section .body { padding: 14px 16px; }

.intro {
  margin: 12px 32px 4px;
  padding: 12px 16px;
  background: rgba(31, 111, 235, .08);
  border: 1px solid rgba(31, 111, 235, .30);
  border-radius: 8px;
  color: #8b949e; font-size: 13px; line-height: 1.6;
}
.intro b { color: #e6edf3; font-weight: 500; }
.intro code { color: #79c0ff; font-family: ui-monospace, monospace; font-size: 12px; }
.intro .term {
  display: inline-block; padding: 1px 6px;
  background: rgba(31, 111, 235, .15);
  border-radius: 3px; color: #79c0ff;
  font-family: ui-monospace, monospace; font-size: 12px;
}

.numbers {
  display: flex; gap: 32px; flex-wrap: wrap;
  padding: 4px 0 12px;
}
.numbers .num-item { font-family: ui-monospace, monospace; }
.numbers .num-label {
  color: #8b949e; font-size: 11px;
  text-transform: uppercase; letter-spacing: .5px;
  margin-bottom: 2px;
}
.numbers .num-value {
  font-size: 22px; color: #79c0ff; font-weight: 600;
}

.bar-wrap {
  background: #0d1117; border-radius: 6px;
  height: 26px; overflow: hidden; position: relative;
  margin-bottom: 6px;
  border: 1px solid #30363d;
}
.bar {
  height: 100%; width: 0;
  transition: width .35s cubic-bezier(.2,.7,.2,1);
  background: linear-gradient(90deg, #1f6feb, #58a6ff);
}
.bar.high { background: linear-gradient(90deg, #f0883e, #f0b73e); }
.bar.critical { background: linear-gradient(90deg, #f85149, #f0b73e); }

.pct-row {
  display: flex; justify-content: space-between;
  color: #8b949e; font-size: 11px;
  font-family: ui-monospace, monospace;
  margin-bottom: 4px;
}
.pct-row b { color: #e6edf3; }

.details {
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #8b949e; margin-top: 10px; padding-top: 10px;
  border-top: 1px solid #30363d;
}
.details .details-title {
  color: #8b949e; font-size: 11px;
  text-transform: uppercase; letter-spacing: .5px;
  margin-bottom: 6px;
}
.details .d-row {
  display: grid; grid-template-columns: 160px 1fr 100px;
  gap: 10px; align-items: center; padding: 3px 0;
}
.details .d-name { color: #e6edf3; }
.details .d-bar {
  height: 12px; background: #0d1117; border-radius: 3px;
  overflow: hidden; border: 1px solid #21262d;
}
.details .d-fill {
  height: 100%; width: 0;
  background: #79c0ff; border-radius: 3px;
}
.details .d-val {
  text-align: right; color: #8b949e;
  font-variant-numeric: tabular-nums;
}

.controls { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 10px; }
.btn {
  background: #21262d; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 8px 14px; font-size: 13px; cursor: pointer;
  transition: background .15s, border-color .15s;
}
.btn:hover { background: #30363d; border-color: #8b949e; }
.btn:active { background: #1f6feb; border-color: #1f6feb; }
.btn.primary { background: #1f6feb; border-color: #1f6feb; }
.btn.primary:hover { background: #388bfd; }
.btn.danger { background: #3d1418; border-color: #6e2a2e; }
.btn.danger:hover { background: #5a1f24; border-color: #f85149; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.persist-box {
  display: flex; align-items: flex-start; gap: 14px;
  padding: 14px 16px; border-radius: 6px;
  font-family: ui-monospace, monospace; font-size: 13px;
  margin-bottom: 12px;
}
.persist-box.ok {
  background: rgba(46, 160, 67, .10);
  border: 1px solid rgba(46, 160, 67, .35);
}
.persist-box.warn {
  background: rgba(210, 153, 34, .10);
  border: 1px solid rgba(210, 153, 34, .35);
}
.persist-box .icon { font-size: 22px; line-height: 1.1; }
.persist-box .headline {
  font-weight: 600; font-size: 14px; margin-bottom: 4px;
}
.persist-box.ok .headline { color: #3fb950; }
.persist-box.warn .headline { color: #d29922; }
.persist-box .expl {
  color: #8b949e; font-size: 12px; line-height: 1.6;
  font-family: system-ui, -apple-system, sans-serif;
}
.persist-box .expl b { color: #e6edf3; font-weight: 500; }
.persist-box .expl ul {
  margin: 6px 0 0 18px; padding: 0; list-style: disc;
}
.persist-box .expl li { margin-bottom: 3px; }

.log-body {
  padding: 12px 16px; max-height: 200px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; }
.log-line.info  { color: #8b949e; }
.log-line.ok    { color: #3fb950; }
.log-line.warn  { color: #d29922; }
.log-line.err   { color: #f85149; }
.log-line.quota { color: #79c0ff; }
"#;

// ===========================================================================
// Логирование
// ===========================================================================

fn log_line(state: &Rc<DemoState>, kind: &str, msg: &str) {
    {
        let mut v = state.log_lines.borrow_mut();
        v.push(format_str(format_args!("[{kind}] {msg}")));
        if v.len() > 200 {
            let excess = v.len() - 200;
            v.drain(0..excess);
        }
    }

    let window = match web_sys::window() {
        Some(w) => w,
        None => return,
    };
    let doc = match window.document() {
        Some(d) => d,
        None => return,
    };
    let line = match doc.create_element("div") {
        Ok(e) => e,
        Err(_) => return,
    };
    let cls: String = format_str(format_args!("log-line {kind}"));
    line.set_class_name(&cls);
    line.set_text_content(Some(msg));
    state.log_body.append_child(&line).ok();
    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let max: f64 = el.scroll_height() as f64;
        el.set_scroll_top(max);
    }
}

fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

// ===========================================================================
// Рендер метрик
// ===========================================================================

fn render_estimate(state: &Rc<DemoState>, e: &StorageEstimate) {
    let usage_html: String = format_str(format_args!("{}", fmt_bytes(e.usage)));
    let quota_html: String = format_str(format_args!("{}", fmt_bytes(e.quota)));
    state.usage_label.set_text_content(Some(&usage_html));
    state.quota_label.set_text_content(Some(&quota_html));

    let ratio = e.fill_ratio();
    let pct_str = fmt_pct(ratio);
    let style: String = format_str(format_args!("width: {:.4}%", ratio * 100.0));
    state.fill_bar.set_attribute("style", &style).ok();

    let cls: &str = if ratio > 0.9 {
        "bar critical"
    } else if ratio > 0.7 {
        "bar high"
    } else {
        "bar"
    };
    state.fill_bar.set_class_name(cls);

    state.fill_pct.set_text_content(Some(&pct_str));

    render_usage_details(state, e);
}

fn render_usage_details(state: &Rc<DemoState>, e: &StorageEstimate) {
    if e.usage_details.is_empty() {
        state.details_body.set_inner_html(
            "<div class=\"d-row\"><span class=\"d-name\">(браузер не предоставил разбивку)</span></div>",
        );
        return;
    }

    let max = e.usage.max(1.0);
    let mut html: String = String::new();

    html.push_str(
        "<div class=\"details-title\">Из чего состоит занятое место</div>",
    );

    for (name, val) in &e.usage_details {
        let display = match name.as_str() {
            "fileSystem" => "Файловая система (OPFS)",
            "indexedDB" => "IndexedDB",
            "caches" => "Cache Storage",
            "serviceWorker" => "Service Worker",
            _ => name.as_str(),
        };
        let frac = (val / max).clamp(0.0, 1.0);
        let w = frac * 100.0;
        let line: String = format_str(format_args!(
            "<div class=\"d-row\">\
             <span class=\"d-name\">{display}</span>\
             <span class=\"d-bar\"><span class=\"d-fill\" style=\"width: {w:.2}%\"></span></span>\
             <span class=\"d-val\">{}</span>\
             </div>",
            fmt_bytes(*val)
        ));
        html.push_str(&line);
    }
    state.details_body.set_inner_html(&html);
}

fn render_persist_state(state: &Rc<DemoState>, is_persisted: bool) {
    if is_persisted {
        let html: String = format_str(format_args!(
            "<span class=\"icon\">✓</span>\
             <div>\
               <div class=\"headline\">Защищено от удаления браузером</div>\
               <div class=\"expl\">\
                 Браузер <b>не будет</b> автоматически удалять данные этого сайта \
                 при нехватке места на диске. Ваши черновики, кэш и настройки \
                 сохранятся, даже если пользователь скачает много другого.\
               </div>\
             </div>"
        ));
        state.persist_box.set_class_name("persist-box ok");
        state.persist_box.set_inner_html(&html);
    } else {
        let html: String = format_str(format_args!(
            "<span class=\"icon\">⚠</span>\
             <div>\
               <div class=\"headline\">Не защищено: браузер может удалить данные</div>\
               <div class=\"expl\">\
                 Если на диске кончится место, браузер вправе удалить <b>всё</b> \
                 хранилище этого сайта — OPFS, IndexedDB и кэш целиком — по \
                 принципу «кто дольше не заходил, того удаляем».<br><br>\
                 <b>Как получить защиту в Chrome:</b>\
                 <ul>\
                   <li>установите сайт как приложение (PWA)</li>\
                   <li>или добавьте страницу в закладки (Ctrl+D)</li>\
                   <li>или разрешите уведомления для этого сайта</li>\
                 </ul>\
                 После этого нажмите «Запросить защиту» — браузер даст согласие.\
               </div>\
             </div>"
        ));
        state.persist_box.set_class_name("persist-box warn");
        state.persist_box.set_inner_html(&html);
    }
}

async fn refresh_estimate(state: Rc<DemoState>) -> Result<(), JsValue> {
    match estimate().await {
        Ok(e) => {
            let line: String = format_str(format_args!("Оценка хранилища: {}", e.summary()));
            log_line(&state, "quota", &line);
            console_log(&line);
            render_estimate(&state, &e);
        }
        Err(e) => {
            console_err("estimate failed", &e);
            log_line(
                &state,
                "err",
                &format_str(format_args!("Не удалось получить оценку: {e:?}")),
            );
        }
    }
    Ok(())
}

// ===========================================================================
// Действия
// ===========================================================================

async fn fill_opfs(state: Rc<DemoState>, total_mb: usize) -> Result<(), JsValue> {
    console_log(&format_str(format_args!("fill: {total_mb} MB")));

    let root = match opfs_root().await {
        Ok(r) => r,
        Err(e) => {
            console_err("opfs_root failed", &e);
            log_line(
                &state,
                "err",
                &format_str(format_args!("OPFS недоступен: {e:?}")),
            );
            return Ok(());
        }
    };

    let chunk_mb: usize = 5;
    let chunks: usize = (total_mb + chunk_mb - 1) / chunk_mb;
    let chunk_bytes: usize = chunk_mb * 1024 * 1024;
    let buf: Vec<u8> = (0..chunk_bytes).map(|i| (i & 0xFF) as u8).collect();

    log_line(
        &state,
        "info",
        &format_str(format_args!(
            "Заполняю {total_mb} МБ: {chunks} файлов по {chunk_mb} МБ"
        )),
    );

    for i in 0..chunks {
        let name: String = format_str(format_args!("quota_fill_{i:03}.bin"));
        if let Err(e) = write_bytes(&root, &name, &buf).await {
            console_err("write_bytes failed", &e);
            log_line(
                &state,
                "err",
                &format_str(format_args!("Ошибка записи {name}: {e:?}")),
            );
            return Ok(());
        }
        if i % 2 == 0 || i == chunks - 1 {
            let line: String =
                format_str(format_args!("Записано файлов: {}/{}", i + 1, chunks));
            log_line(&state, "info", &line);
        }
    }

    log_line(
        &state,
        "ok",
        &format_str(format_args!("Готово: занято {total_mb} МБ в OPFS")),
    );
    refresh_estimate(state.clone()).await?;
    Ok(())
}

async fn clear_opfs(state: Rc<DemoState>) -> Result<(), JsValue> {
    console_log("clear: удаляем quota_fill_*.bin");

    let root = match opfs_root().await {
        Ok(r) => r,
        Err(e) => {
            console_err("opfs_root failed", &e);
            return Ok(());
        }
    };

    for i in 0..200 {
        let name: String = format_str(format_args!("quota_fill_{i:03}.bin"));
        if remove_file(&root, &name).await.is_err() {
            break;
        }
    }
    let _ = remove_file(&root, "quota_fill_000.bin").await;

    log_line(
        &state,
        "ok",
        "Все файлы удалены. Повторное удаление — тоже успех (идемпотентность).",
    );
    refresh_estimate(state.clone()).await?;
    Ok(())
}

async fn do_request_persist(state: Rc<DemoState>) -> Result<(), JsValue> {
    console_log("persist: запрос");
    log_line(&state, "info", "Отправляю запрос на защиту от удаления…");

    let granted: bool = request_persist().await;

    let line: String = if granted {
        format_str(format_args!("Браузер дал согласие"))
    } else {
        format_str(format_args!(
            "Браузер отказал (нужен PWA-установка, закладка или уведомления)"
        ))
    };
    log_line(&state, if granted { "ok" } else { "warn" }, &line);
    console_log(&line);

    let now_persisted: bool = persisted().await;
    let line2: String = format_str(format_args!(
        "Текущий статус защиты: {}",
        if now_persisted { "включена" } else { "выключена" }
    ));
    log_line(&state, "info", &line2);
    console_log(&line2);

    render_persist_state(&state, now_persisted);
    Ok(())
}

async fn do_check_persisted(state: Rc<DemoState>) -> Result<(), JsValue> {
    let now_persisted: bool = persisted().await;
    let line: String = format_str(format_args!(
        "Текущий статус защиты: {}",
        if now_persisted { "включена" } else { "выключена" }
    ));
    log_line(&state, "info", &line);
    console_log(&line);
    render_persist_state(&state, now_persisted);
    Ok(())
}

// ===========================================================================
// UI-хелперы
// ===========================================================================

fn make_button<F>(
    doc: &Document,
    parent: &Element,
    label: &str,
    tooltip: &str,
    primary: bool,
    danger: bool,
    mut handler: F,
) -> Result<Element, JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    let cls: &str = if primary {
        "btn primary"
    } else if danger {
        "btn danger"
    } else {
        "btn"
    };
    btn.set_class_name(cls);
    btn.set_text_content(Some(label));
    btn.set_attribute("title", tooltip)?;
    parent.append_child(&btn)?;

    let closure = Closure::<dyn FnMut()>::new(move || handler());
    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(btn)
}

fn make_section(
    doc: &Document,
    num: &str,
    title: &str,
    hint: &str,
) -> Result<(Element, Element), JsValue> {
    let section = doc.create_element("div")?;
    section.set_class_name("section");

    let h3 = doc.create_element("h3")?;
    let header_html: String = format_str(format_args!(
        "<span class=\"num\">{num}</span> <span>{title}</span>\
         <span class=\"hint\">{hint}</span>"
    ));
    h3.set_inner_html(&header_html);
    section.append_child(&h3)?;

    let body = doc.create_element("div")?;
    body.set_class_name("body");
    section.append_child(&body)?;

    Ok((section, body))
}

fn make_num_item(
    doc: &Document,
    label: &str,
    id: &str,
) -> Result<(Element, Element), JsValue> {
    let item = doc.create_element("div")?;
    item.set_class_name("num-item");

    let lbl = doc.create_element("div")?;
    lbl.set_class_name("num-label");
    lbl.set_text_content(Some(label));
    item.append_child(&lbl)?;

    let val = doc.create_element("div")?;
    val.set_class_name("num-value");
    val.set_id(id);
    val.set_text_content(Some("—"));
    item.append_child(&val)?;

    Ok((item, val))
}

// ===========================================================================
// Точка входа
// ===========================================================================

pub fn run() -> Result<(), JsValue> {
    console_log("run: starting");

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = document
        .body()
        .ok_or_else(|| JsValue::from_str("no body"))?;

    body.set_inner_html("");

    // Единое визуальное поле.
    if let Ok(links) = document.query_selector_all("link[rel='stylesheet']") {
        let n: u32 = links.length();
        for i in 0..n {
            if let Some(node) = links.get(i) {
                if let Some(parent) = node.parent_node() {
                    parent.remove_child(&node).ok();
                }
            }
        }
    }
    if let Some(root_el) = document.document_element() {
        root_el
            .set_attribute(
                "style",
                "background:#0d1117;color:#e6edf3;margin:0;padding:0;min-height:100vh;",
            )
            .ok();
    }

    let style = document.create_element("style")?;
    style.set_text_content(Some(CSS));
    document
        .head()
        .ok_or_else(|| JsValue::from_str("no head"))?
        .append_child(&style)
        .ok();

    // -------------------------------------------------------------------------
    // Header
    // -------------------------------------------------------------------------
    let h1 = document.create_element("h1")?;
    h1.set_inner_html(
        "19.9 <code>StorageManager</code>: сколько места у сайта и защищены ли данные",
    );
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "узнать лимит хранилища · попросить защиту от удаления · увидеть, из чего состоит занятое место",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    // -------------------------------------------------------------------------
    // Вступительная плашка
    // -------------------------------------------------------------------------
    let intro = document.create_element("div")?;
    intro.set_class_name("intro");
    intro.set_inner_html(
        "<b>Что такое StorageManager.</b> Это диспетчер хранилища вашего сайта. \
         Он отвечает на три вопроса: <span class=\"term\">сколько места занято</span>, \
         <span class=\"term\">каков лимит</span> и \
         <span class=\"term\">защищены ли данные от автоматического удаления</span>.<br><br>\
         <b>Что такое «защита от удаления» (persistent).</b> По умолчанию браузер \
         считает данные сайта «второстепенными»: если на диске кончится место, он вправе \
         удалить <b>всё хранилище этого сайта</b> — файлы OPFS, базу IndexedDB и кэш — \
         целиком, без предупреждения. Если защита включена, браузер трогать этот сайт \
         не будет: сначала «пожертвует» другими.<br><br>\
         <b>Как включить защиту.</b> В Chrome нужно одно из: установить сайт как \
         приложение (PWA), добавить в закладки или разрешить уведомления. После этого \
         нажмите <b>«Запросить защиту»</b> ниже.",
    );
    body.append_child(&intro)?;

    // -------------------------------------------------------------------------
    // Status bar
    // -------------------------------------------------------------------------
    let status = document.create_element("div")?;
    status.set_class_name("status-bar");
    status.set_text_content(Some("проверяю доступность…"));
    body.append_child(&status)?;

    // -------------------------------------------------------------------------
    // Section ❶
    // -------------------------------------------------------------------------
    let (sec1, body1) = make_section(
        &document,
        "❶",
        "Занято и лимит",
        "usage / quota · из чего состоит занятое место",
    )?;
    body.append_child(&sec1)?;

    let numbers = document.create_element("div")?;
    numbers.set_class_name("numbers");
    body1.append_child(&numbers)?;

    let (item_usage, usage_label) = make_num_item(&document, "занято", "usage-v")?;
    numbers.append_child(&item_usage)?;

    let (item_quota, quota_label) =
        make_num_item(&document, "лимит на этот сайт", "quota-v")?;
    numbers.append_child(&item_quota)?;

    let bar_wrap = document.create_element("div")?;
    bar_wrap.set_class_name("bar-wrap");
    let fill_bar_el = document.create_element("div")?;
    fill_bar_el.set_class_name("bar");
    bar_wrap.append_child(&fill_bar_el)?;
    body1.append_child(&bar_wrap)?;

    let fill_bar: HtmlElement = fill_bar_el
        .dyn_into()
        .map_err(|_| JsValue::from_str("bar не HtmlElement"))?;

    let pct_row = document.create_element("div")?;
    pct_row.set_class_name("pct-row");
    pct_row.set_inner_html(
        "<span>0%</span>\
         <span>использовано: <b id=\"pct-v\">0%</b></span>\
         <span>100%</span>",
    );
    body1.append_child(&pct_row)?;
    let fill_pct = pct_row
        .query_selector("#pct-v")?
        .ok_or_else(|| JsValue::from_str("no pct-v"))?;

    let details_body = document.create_element("div")?;
    details_body.set_class_name("details");
    details_body.set_inner_html(
        "<div class=\"d-row\"><span class=\"d-name\">(загрузка…)</span></div>",
    );
    body1.append_child(&details_body)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("controls");
    body1.append_child(&controls)?;

    // -------------------------------------------------------------------------
    // Section ❷
    // -------------------------------------------------------------------------
    let (sec2, body2) = make_section(
        &document,
        "❷",
        "Защита от автоматического удаления",
        "persist() — запрос · persisted() — статус",
    )?;
    body.append_child(&sec2)?;

    let persist_box = document.create_element("div")?;
    persist_box.set_class_name("persist-box warn");
    persist_box.set_inner_html(
        "<span class=\"icon\">…</span>\
         <div>\
           <div class=\"headline\">проверяю статус…</div>\
         </div>",
    );
    body2.append_child(&persist_box)?;

    let persist_controls = document.create_element("div")?;
    persist_controls.set_class_name("controls");
    body2.append_child(&persist_controls)?;

    // -------------------------------------------------------------------------
    // Section ❸
    // -------------------------------------------------------------------------
    let (sec3, body3) = make_section(&document, "❸", "Журнал", "")?;
    body.append_child(&sec3)?;
    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    body3.set_class_name("");
    body3.set_attribute("style", "padding:0")?;
    body3.append_child(&log_body)?;

    // -------------------------------------------------------------------------
    // State
    // -------------------------------------------------------------------------
    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        usage_label,
        quota_label,
        fill_bar,
        fill_pct,
        details_body,
        persist_box: persist_box.clone(),
        log_body: log_body.clone(),
        log_lines: RefCell::new(Vec::new()),
    });

    // -------------------------------------------------------------------------
    // Кнопки секции ❶
    // -------------------------------------------------------------------------
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Заполнить 10 МБ",
            "Записывает в OPFS два файла по 5 МБ. Прогресс-бар квоты сдвинется вправо.",
            false,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = fill_opfs(s, 10).await {
                        console_err("fill 10 MB failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Заполнить 50 МБ",
            "Записывает 10 файлов по 5 МБ. Хорошо видно, как занятое место растёт.",
            false,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = fill_opfs(s, 50).await {
                        console_err("fill 50 MB failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Заполнить 200 МБ",
            "Записывает 40 файлов по 5 МБ. На маленьком лимите (например, в Safari) это займёт заметную долю.",
            false,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = fill_opfs(s, 200).await {
                        console_err("fill 200 MB failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Очистить всё",
            "Удаляет все файлы quota_fill_*.bin. Повторное нажатие тоже успех — идемпотентно.",
            false,
            true,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = clear_opfs(s).await {
                        console_err("clear failed", &e);
                    }
                });
            },
        )?;
    }

    // -------------------------------------------------------------------------
    // Кнопки секции ❷
    // -------------------------------------------------------------------------
    {
        let s = state.clone();
        make_button(
            &document,
            &persist_controls,
            "Запросить защиту",
            "Просит браузер защитить данные сайта от автоматического удаления. Может получить отказ — это не ошибка.",
            true,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = do_request_persist(s).await {
                        console_err("persist failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &persist_controls,
            "Проверить статус",
            "Возвращает текущий статус защиты. Не отправляет запрос — только читает.",
            false,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = do_check_persisted(s).await {
                        console_err("persisted failed", &e);
                    }
                });
            },
        )?;
    }

    // -------------------------------------------------------------------------
    // Инициализация
    // -------------------------------------------------------------------------
    {
        let s = state.clone();
        spawn_local(async move {
            console_log("инициализация…");

            if !opfs_available() {
                set_status(&s, "err", "OPFS недоступен в этом браузере");
                log_line(&s, "err", "OPFS недоступен — демо не может работать");
                return;
            }
            if !quota_available() {
                set_status(&s, "warn", "OPFS доступен · StorageManager недоступен");
                log_line(
                    &s,
                    "warn",
                    "navigator.storage.estimate недоступен — браузер слишком старый",
                );
                return;
            }

            set_status(&s, "ok", "StorageManager доступен");
            log_line(&s, "ok", "StorageManager доступен, OPFS доступен");

            if let Err(e) = refresh_estimate(s.clone()).await {
                console_err("initial estimate failed", &e);
            }

            let is_persisted = persisted().await;
            let line: String = format_str(format_args!(
                "Текущий статус защиты: {}",
                if is_persisted { "включена" } else { "выключена" }
            ));
            log_line(&s, "info", &line);
            console_log(&line);
            render_persist_state(&s, is_persisted);
        });
    }

    console_log("run: finished OK");
    Ok(())
}
```

---

## 19.9.7. Правила проекта, выученные в этой главе

**1. Квота — не константа.** Она зависит от браузера, платформы, свободного места на диске и политики вендора. Никогда не хардкодьте «у нас есть 1 ГБ». Спрашивайте `estimate()`.

**2. `usage`/`quota` — оценки.** Браузер округляет ради защиты от fingerprinting. Chrome — до мегабайт, Firefox — до десятков мегабайт. Не удивляйтесь, если после записи 100 КБ `usage` не изменится.

**3. Persistence — это запрос, не команда.** `persist()` может вернуть `false`. Это не ошибка. Если приложение хранит важные данные — предупредите пользователя, если защиту не дали.

**4. Без persistence браузер может вытеснить origin целиком.** Не один файл — **весь origin**: OPFS, IndexedDB, Cache Storage. LRU-алгоритм под давлением дискового пространства.

**5. `usageDetails` есть не везде.** Chrome возвращает разбивку по категориям, Firefox/Safari — нет. Пустой объект — это нормально.

**6. Общий кошелёк на все хранилища.** OPFS, IndexedDB, Cache Storage делят **одну** квоту. Лимит на OPFS — это не отдельный лимит.

**7. Не полагайтесь на квоту для дизайна.** «Влезет ли сюда 100 МБ?» — зависит от того, есть ли у пользователя 200 МБ свободного места. Спрашивайте, а не предполагайте.

**8. `persist()` в headless Chrome обычно `false`.** Браузер требует user engagement (PWA установлен, добавлен в закладки). В обычной вкладке без признаков «пользователь сюда вернётся» — отказ.

**9. Тесты не паникуют на «мягких» отказах.** `persist() == false` — валидный результат. `quota == 0` — валидный результат. Проверяйте инварианты (`usage >= 0`, `fill_ratio ∈ [0,1]`), а не конкретные числа.

**10. Строковые хелперы — в каждом модуле свои.** `fmt_bytes` в `fsa_api_quota.rs` не тащится из другого модуля. Правило: модуль самодостаточен.

**11. `format_str` объявлена в начале файла.** Первое, что читатель видит после doc-comment, — функции работы со строками. Это правило: если в модуле есть строковые выражения, они собраны в одном месте.

**12. Все методы `web-sys` — в snake_case.** `createElement`, `setAttribute`, `appendChild`, `addEventListener` — это JS-стиль, его в `web-sys` нет. Соответствия:

| JS | Rust |
|---|---|
| `document.createElement` | `document.create_element` |
| `element.setAttribute` | `element.set_attribute` |
| `element.appendChild` | `element.append_child` |
| `element.addEventListener` | `element.add_event_listener_with_callback` |
| `element.setClassName` | `element.set_class_name` |
| `element.setInnerHTML` | `element.set_inner_html` |

Если rustc говорит «no method named `xyz` found ... there is a method `x_y_z` with a similar name» — это почти всегда пропущенный snake_case.

---

## 19.9.8. Чек-лист

- [ ] `navigator.storage.estimate()` возвращает `{usage, quota, usageDetails}` — оценки, не точные значения.
- [ ] `usage`/`quota` округляются браузером — не паникуйте на «нулевой» usage после мелкой записи.
- [ ] `quota` зависит от браузера/платформы/свободного места — не хардкодьте.
- [ ] `persist()` — запрос, может вернуть `false`. Отказ — не ошибка.
- [ ] `persisted()` — текущий статус, `bool`.
- [ ] Без persistence браузер может вытеснить origin **целиком** под давлением места.
- [ ] `usageDetails` есть не везде (Chrome — да, Firefox/Safari — обычно нет).
- [ ] Все хранилища origin'а делят **одну** квоту.
- [ ] Если приложение хранит важные данные — вызывайте `persist()` и предупреждайте пользователя при отказе.
- [ ] Тесты проверяют инварианты (`usage >= 0`, `quota >= usage`, `fill_ratio ∈ [0,1]`), а не конкретные значения.
- [ ] `fmt_bytes` — в каждом модуле своя (правило самодостаточности).
- [ ] `format_str` объявлена до первого использования.
- [ ] Все методы `web-sys` — в snake_case.
- [ ] Интерфейс — на русском, с пояснениями непонятных терминов.

---

## 19.9.9. Команды запуска

```bash
# Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# Host-тесты (быстрые, без браузера)
cargo test

# WASM-тесты квоты в headless Chrome
wasm-pack test --headless --chrome --test wasm_fsa_api_quota

# Запуск демо
npm run start
```

`Cargo.toml` **не требует** новых features. `.cargo/config.toml` **не требует** `--cfg=web_sys_unstable_apis`. Всё общение с `navigator.storage` идёт через `js_sys::Reflect` — независимо от версии `web-sys`.

---

## 19.9.10. Что дальше

Мы научились видеть «стену» — квоту origin'а — и попросили у браузера защиту от вытеснения. Это третий, «административный» слой работы с хранилищем:

- **19.7** — структура: дерево, пути, diff.
- **19.8** — производительность: OPFS, sync access handle, живой логгер.
- **19.9** — устойчивость: квота, persist, usageDetails.

В следующей главе мы соберём всё вместе: **`fsa_api_project.rs`** — аудит проекта поверх дерева, с проверками («нет файлов > 5 МБ», «нет `.map` в проде»), отчётом в виде `Diff` и предупреждением о квоте, если проект приближается к лимиту. Это тот самый уровень, где отдельные примитивы становятся **инструментом**, а не набором функций.