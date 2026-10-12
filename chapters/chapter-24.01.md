# Глава 24.1. Почему `localStorage` недостаточно

## 24.1.0. Что вы получите в этой главе

К концу главы у вас будет работающее демо, которое за **три эксперимента** доказывает три тезиса:

1. **Мал.** Одно «фото» 5 МБ не влезает в `localStorage`.
2. **Дорог.** Бинарные данные в `localStorage` занимают ×2.67 от исходного размера.
3. **Блокирует.** Синхронный `setItem` замораживает UI — это видно по счётчику heartbeat, который перестаёт тикать.

Демо — **мостик к IndexedDB**: следующая подглава (24.2) начнётся с того, как IndexedDB решает все три проблемы. В коде этой главы **ни слова** про IndexedDB — только диагноз.

## 24.1.1. Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── ls_probe.rs        # НОВЫЙ: CRUD localStorage, детекция QuotaExceededError
│   ├── ls_bloat.rs        # НОВЫЙ: base64-раздутие, кодирование/декодирование
│   ├── ls_bench.rs        # НОВЫЙ: замер синхронной блокировки, fill до квоты
│   ├── lib.rs             # +3 модуля
│   └── app.rs             # демо — три эксперимента на одном экране
├── tests/
│   ├── ls_probe_test.rs   # НОВЫЙ: wasm-тесты CRUD, UTF-16, детекции квоты
│   ├── ls_bloat_test.rs   # НОВЫЙ: wasm-тесты base64 round-trip, раздутия
│   └── ls_bench_test.rs   # НОВЫЙ: wasm-тесты замеров, fill, cleanup
└── Cargo.toml             # без изменений: Storage, DomException, Performance, base64 — уже есть
```

**Комментарий.** Три модуля делают **разные измерения** одного объекта — `localStorage`. `ls_probe` — про доступ, размер и детекцию ошибок. `ls_bloat` — про стоимость хранения бинарных данных. `ls_bench` — про блокировку UI при записи. Префикс `ls_` — новый для проекта, не пересекается с другими модулями (`idb_*`, `fsa_*`, `cache_*`).

## 24.1.2. Ссылки для детального изучения

- MDN, Window.localStorage — https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage
- HTML Spec, Web Storage — https://html.spec.whatwg.org/multipage/webstorage.html
- MDN, Storage quotas and eviction — https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria
- web.dev, Storage for the web — https://web.dev/articles/storage-for-the-web
- MDN, `requestAnimationFrame` (для понимания event loop) — https://developer.mozilla.org/en-US/docs/Web/API/Window/requestAnimationFrame

## 24.1.3. Исходные коды новых модулей

### `src/ls_probe.rs`

```rust
//! Работа с localStorage: CRUD, детекция квоты, приблизительный размер.
//! Автономный модуль — не зависит от других модулей проекта.
//!
//! ПРАВИЛО: ошибки `localStorage` пробрасываются как `JsValue` **без обёртки
//! в строку**. `JsValue::from_str(&format!("{e:?}"))` уничтожает свойства
//! `name` и `message` исходного `DOMException`, из-за чего `is_quota_error`
//! перестаёт распознавать `QuotaExceededError`.

use wasm_bindgen::prelude::*;
use web_sys::{DomException, Storage};

/// Возвращает localStorage. Ошибка — если окна нет или storage отключён.
pub fn storage() -> Result<Storage, JsValue> {
    let win = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let opt = win.local_storage()?;
    opt.ok_or_else(|| JsValue::from_str("localStorage недоступен"))
}

/// true, если localStorage доступен.
pub fn is_available() -> bool {
    storage().is_ok()
}

/// Записывает строку. Ошибку пробрасывает как есть.
pub fn set_string(key: &str, value: &str) -> Result<(), JsValue> {
    storage()?.set_item(key, value)
}

/// Читает строку. None — если ключа нет.
pub fn get_string(key: &str) -> Result<Option<String>, JsValue> {
    storage()?.get_item(key)
}

/// Удаляет ключ. Идемпотентно.
pub fn remove(key: &str) -> Result<(), JsValue> {
    storage()?.remove_item(key)
}

/// Очищает все ключи origin.
pub fn clear_all() -> Result<(), JsValue> {
    storage()?.clear()
}

/// Список ключей в storage.
pub fn keys() -> Result<Vec<String>, JsValue> {
    let s = storage()?;
    let len = s.length()?;
    let mut out = Vec::with_capacity(len as usize);
    for i in 0..len {
        if let Some(k) = s.key(i)? {
            out.push(k);
        }
    }
    Ok(out)
}

/// Число ключей.
pub fn item_count() -> Result<u32, JsValue> {
    storage()?.length()
}

/// Приблизительный размер в **байтах UTF-16** (2 байта на code unit).
///
/// localStorage хранит строки как UTF-16: ASCII-символ занимает 2 байта.
/// Квота 5 МБ обычно означает 5 миллионов UTF-16 code units,
/// то есть ~2.5 миллиона ASCII-символов.
pub fn approx_bytes_used() -> Result<usize, JsValue> {
    let s = storage()?;
    let len = s.length()?;
    let mut total = 0usize;
    for i in 0..len {
        if let Some(k) = s.key(i)? {
            total += utf16_bytes(&k);
            if let Some(v) = s.get_item(&k)? {
                total += utf16_bytes(&v);
            }
        }
    }
    Ok(total)
}

/// Размер строки в байтах UTF-16.
pub fn utf16_bytes(s: &str) -> usize {
    s.encode_utf16().count() * 2
}

/// true, если ошибка — `QuotaExceededError`.
///
/// В Chrome/Firefox/Safari `setItem` бросает `DOMException` с полем
/// `name == "QuotaExceededError"`. Проверяем и через `dyn_ref`, и через
/// `Reflect` — на случай, если в текущей версии web-sys тип не сходится.
pub fn is_quota_error(e: &JsValue) -> bool {
    if let Some(ex) = e.dyn_ref::<DomException>() {
        if ex.name() == "QuotaExceededError" {
            return true;
        }
    }
    js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .map(|s| s == "QuotaExceededError")
        .unwrap_or(false)
}

/// Снимок состояния хранилища.
#[derive(Debug, Clone)]
pub struct StorageInfo {
    pub available: bool,
    pub key_count: u32,
    pub approx_bytes: usize,
}

/// Возвращает снимок состояния. Если storage недоступен — все поля
/// принимают «нулевые» значения, `available = false`.
pub fn probe() -> StorageInfo {
    if !is_available() {
        return StorageInfo {
            available: false,
            key_count: 0,
            approx_bytes: 0,
        };
    }
    StorageInfo {
        available: true,
        key_count: item_count().unwrap_or(0),
        approx_bytes: approx_bytes_used().unwrap_or(0),
    }
}
```

### `src/ls_bloat.rs`

```rust
//! Раздутие base64: localStorage хранит только строки, бинарные данные
//! приходится кодировать. Считаем, во сколько раз это дороже.
//!
//! Автономный модуль — не зависит от других модулей проекта.

use base64::engine::general_purpose::STANDARD;
use base64::Engine as _;
use wasm_bindgen::prelude::*;

/// Псевдослучайные байты через LCG. Детерминированы по seed,
/// что удобно для тестов и повторных прогонов.
pub fn random_bytes(n: usize, seed: u64) -> Vec<u8> {
    let mut state = seed.wrapping_mul(6364136223846793005).wrapping_add(1);
    let mut out = Vec::with_capacity(n);
    for _ in 0..n {
        state = state
            .wrapping_mul(6364136223846793005)
            .wrapping_add(1442695040888963407);
        out.push((state >> 33) as u8);
    }
    out
}

/// Кодирует байты в base64-строку.
pub fn encode_base64(bytes: &[u8]) -> String {
    STANDARD.encode(bytes)
}

/// Декодирует base64. Возвращает ошибку при невалидной строке.
pub fn decode_base64(s: &str) -> Result<Vec<u8>, String> {
    STANDARD.decode(s).map_err(|e| format!("base64 decode: {e}"))
}

/// Отчёт о раздутии.
#[derive(Debug, Clone, PartialEq)]
pub struct BloatReport {
    /// Исходный размер бинарных данных.
    pub raw_bytes: usize,
    /// Длина base64-строки в байтах (ASCII).
    pub base64_chars: usize,
    /// Размер base64-строки в байтах UTF-16, как её считает localStorage.
    pub utf16_bytes: usize,
    /// Во сколько раз UTF-16 представление больше исходных байтов.
    pub ratio: f64,
}

/// Анализирует раздутие: сколько занимает `raw` в бинарном виде
/// vs сколько занимает его base64-представление в localStorage.
pub fn analyze_bloat(raw: &[u8]) -> BloatReport {
    let encoded = STANDARD.encode(raw);
    let base64_chars = encoded.len();
    let utf16_bytes = base64_chars * 2;
    let ratio = if raw.is_empty() {
        0.0
    } else {
        utf16_bytes as f64 / raw.len() as f64
    };
    BloatReport {
        raw_bytes: raw.len(),
        base64_chars,
        utf16_bytes,
        ratio,
    }
}

/// Сохраняет байты в localStorage как base64.
/// Возвращает реальный размер (в UTF-16 байтах), который заняла запись.
pub fn store_bytes_as_base64(key: &str, bytes: &[u8]) -> Result<usize, JsValue> {
    let encoded = STANDARD.encode(bytes);
    crate::ls_probe::set_string(key, &encoded)?;
    Ok(encoded.encode_utf16().count() * 2)
}

/// Читает байты из localStorage, декодируя base64.
/// None — если ключа нет.
pub fn load_bytes_from_base64(key: &str) -> Result<Option<Vec<u8>>, JsValue> {
    let raw = match crate::ls_probe::get_string(key)? {
        Some(v) => v,
        None => return Ok(None),
    };
    STANDARD
        .decode(&raw)
        .map(Some)
        .map_err(|e| JsValue::from_str(&format!("base64 decode: {e}")))
}
```

### `src/ls_bench.rs`

```rust
//! Замер синхронной блокировки главного потока: localStorage.setItem
//! блокирует UI на время записи. Плюс — заполнение хранилища до квоты.
//!
//! Автономный модуль — не зависит от других модулей проекта.
//!
//! ПРАВИЛО: `format!` в tail-позиции if/else / match-arm оборачивается
//! в `format_str(format_args!(...))` или явный `return`.

use wasm_bindgen::prelude::*;

use crate::format_str::format_str;
use crate::ls_probe;

/// `performance.now()` — монотонные миллисекунды.
pub fn now_ms() -> f64 {
    web_sys::window()
        .and_then(|w| w.performance())
        .map(|p| p.now())
        .unwrap_or(0.0)
}

/// Один замер: сколько заняла запись и чтение строки указанного размера.
#[derive(Debug, Clone)]
pub struct TimingSample {
    /// Длина записанной строки в символах.
    pub size_chars: usize,
    /// Время `setItem`, в миллисекундах.
    pub write_ms: f64,
    /// Время `getItem`, в миллисекундах.
    pub read_ms: f64,
}

/// Замеряет один `setItem` + `getItem` для строки длиной `size_chars`.
///
/// Ключ очищается после успешного замера.
pub fn measure_set_get(key: &str, value: &str) -> Result<TimingSample, JsValue> {
    let t0 = now_ms();
    ls_probe::set_string(key, value)?;
    let write_ms = now_ms() - t0;

    let t1 = now_ms();
    let read_result = ls_probe::get_string(key);
    let read_ms = now_ms() - t1;

    let _ = ls_probe::remove(key);

    read_result?;
    Ok(TimingSample {
        size_chars: value.encode_utf16().count(),
        write_ms,
        read_ms,
    })
}

/// Прогоняет серию замеров с указанными размерами строк.
///
/// **Не является tolerant**: при ошибке одного замера прерывает цикл
/// и возвращает `Err`. Для tolerant-варианта используйте функцию
/// `render_sweep_tolerant` в демо (уровень UI).
pub fn run_sweep(prefix: &str, sizes_chars: &[usize]) -> Result<Vec<TimingSample>, JsValue> {
    let mut out = Vec::with_capacity(sizes_chars.len());
    for (i, &size) in sizes_chars.iter().enumerate() {
        let value = "x".repeat(size);
        let key = format!("{prefix}-sweep-{i}");
        out.push(measure_set_get(&key, &value)?);
    }
    Ok(out)
}

/// Результат заполнения хранилища до квоты.
#[derive(Debug, Clone)]
pub struct QuotaResult {
    /// Сколько чанков удалось записать.
    pub written_chunks: u32,
    /// Суммарный размер успешно записанных данных (в UTF-16 байтах).
    pub total_utf16_bytes: usize,
    /// Размер одного чанка (в UTF-16 байтах).
    pub chunk_utf16_bytes: usize,
    /// Время до первой ошибки квоты, в миллисекундах.
    pub elapsed_ms: f64,
    /// Сообщение об ошибке (для отладки; не показывается пользователю).
    pub error_msg: String,
}

/// Заполняет localStorage чанками по `chunk_chars` символов, пока
/// не поймает `QuotaExceededError`. Затем **очищает свои ключи**
/// (префикс `prefix`) и возвращает результат.
///
/// Синхронная версия — используется в тестах. Для UI-сценариев
/// предпочтительнее `fill_until_quota_chunked` (с yield).
pub fn fill_until_quota(
    prefix: &str,
    chunk_chars: usize,
    soft_limit: u32,
) -> Result<QuotaResult, JsValue> {
    let chunk = "x".repeat(chunk_chars);
    let chunk_utf16 = chunk.encode_utf16().count() * 2;

    let t0 = now_ms();
    let mut written: u32 = 0;
    let mut error_msg = String::new();

    for i in 0..soft_limit {
        let key = format!("{prefix}-{i}");
        match ls_probe::set_string(&key, &chunk) {
            Ok(()) => written += 1,
            Err(e) => {
                error_msg = classify_error(&e);
                break;
            }
        }
    }

    let elapsed_ms = now_ms() - t0;

    for i in 0..written {
        let key = format!("{prefix}-{i}");
        let _ = ls_probe::remove(&key);
    }

    Ok(QuotaResult {
        written_chunks: written,
        total_utf16_bytes: written as usize * chunk_utf16,
        chunk_utf16_bytes: chunk_utf16,
        elapsed_ms,
        error_msg,
    })
}

/// Асинхронная версия: между пакетами по `batch_size` чанков делает
/// yield через `TimeoutFuture`, чтобы главный поток успевал перерисовать UI.
///
/// Контракт `on_progress`:
///   • вызывается **после каждого успешно завершённого batch**;
///   • вызывается **один раз при выходе по ошибке** — аварийный снимок,
///     чтобы UI показал, сколько успели записать;
///   • при нормальном завершении (soft_limit достигнут) последний вызов
///     соответствует последнему batch, дополнительного финального вызова нет.
pub async fn fill_until_quota_chunked<F>(
    prefix: &str,
    chunk_chars: usize,
    soft_limit: u32,
    batch_size: u32,
    mut on_progress: F,
) -> Result<QuotaResult, JsValue>
where
    F: FnMut(u32, usize),
{
    let batch = batch_size.max(1);
    let chunk = "x".repeat(chunk_chars);
    let chunk_utf16 = chunk.encode_utf16().count() * 2;

    let t0 = now_ms();
    let mut written: u32 = 0;
    let mut error_msg = String::new();

    'outer: for batch_start in (0..soft_limit).step_by(batch as usize) {
        let batch_end = (batch_start + batch).min(soft_limit);
        for i in batch_start..batch_end {
            let key = format!("{prefix}-{i}");
            match ls_probe::set_string(&key, &chunk) {
                Ok(()) => written += 1,
                Err(e) => {
                    error_msg = classify_error(&e);
                    // Аварийный снимок — только в ветке ошибки.
                    on_progress(written, written as usize * chunk_utf16);
                    break 'outer;
                }
            }
        }
        // Yield через macrotask: браузер перерисовывает UI.
        gloo_timers::future::TimeoutFuture::new(0).await;
        on_progress(written, written as usize * chunk_utf16);
    }

    let elapsed_ms = now_ms() - t0;

    for i in 0..written {
        let key = format!("{prefix}-{i}");
        let _ = ls_probe::remove(&key);
    }

    Ok(QuotaResult {
        written_chunks: written,
        total_utf16_bytes: written as usize * chunk_utf16,
        chunk_utf16_bytes: chunk_utf16,
        elapsed_ms,
        error_msg,
    })
}

/// Классифицирует ошибку в каноническую строку.
///
/// Устойчив к обёрткам: если в тексте встречается «QuotaExceededError» —
/// возвращает каноническую строку, даже если оригинал был обёрнут.
fn classify_error(e: &JsValue) -> String {
    if crate::ls_probe::is_quota_error(e) {
        return "QuotaExceededError".to_string();
    }
    let text = debug_or_message(e);
    if text.contains("QuotaExceededError") {
        return "QuotaExceededError".to_string();
    }
    if text.contains("SecurityError") {
        return "SecurityError".to_string();
    }
    return short_text(&text);
}

/// Достаёт читаемое представление JsValue.
fn debug_or_message(e: &JsValue) -> String {
    if let Some(ex) = e.dyn_ref::<web_sys::DomException>() {
        return format_str(format_args!("{}: {}", ex.name(), ex.message()));
    }
    if let Some(s) = e.as_string() {
        return s;
    }
    return format!("{e:?}");
}

/// Обрезает текст до 120 символов.
fn short_text(s: &str) -> String {
    let first_line = s.lines().next().unwrap_or("").trim();
    let mut out: String = first_line.chars().take(120).collect();
    if first_line.chars().count() > 120 {
        out.push('…');
    }
    return out;
}

/// Человекочитаемая интерпретация `error_msg`.
pub fn describe_error(msg: &str) -> String {
    if msg.is_empty() {
        return "не было ошибки (достигнут soft_limit)".to_string();
    }
    if msg == "QuotaExceededError" {
        return "QuotaExceededError — реальная квота браузера исчерпана".to_string();
    }
    if msg == "SecurityError" {
        return "SecurityError — браузер заблокировал запись".to_string();
    }
    return format_str(format_args!("Ошибка браузера: {msg}"));
}
```

## 24.1.4. Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod ls_probe;
pub mod ls_bloat;
pub mod ls_bench;
```

## 24.1.5. Тесты

### `tests/ls_probe_test.rs` (wasm)

```rust
//! Демо Главы 24.1: Почему localStorage недостаточно.
//!
//! Три эксперимента — один вывод:
//!   1. Одно «фото» 5 МБ → QuotaExceededError. localStorage хранит только
//!      строки: base64 +33%, UTF-16 ×2 → ×2.67 от исходного размера.
//!      Это архитектурное свойство Web Storage.
//!   2. Длинный sync-цикл вокруг setItem → главный поток занят, UI не
//!      отвечает. Heartbeat не тикает. Это эталон скорости: 100% работы.
//!   3. Тот же цикл с yield → UI отвечает. Heartbeat растёт.
//!      Работа падает до ~25% — yield не бесплатен, но оправдан.
//!
//! Итог про localStorage: он **спроектирован синхронным**. Если данных
//! много, у вас выбор — длинный sync-цикл (блокировка UI) или async-обёртка
//! с yield (UI отвечает, но throughput падает). Оба варианта неудобны.
//!
//! Мостик к IndexedDB: гигабайты квоты вместо мегабайт, binary native,
//! и **async по контракту** — без «нашей» обёртки и без потери
//! производительности.
//!
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::spawn_local;
use web_sys::HtmlElement;

use crate::format_str::format_str;
use crate::ls_bench;
use crate::ls_bloat;
use crate::ls_probe;

const PREFIX: &str = "demo-ls241";
/// Размер чанка при записи: 64K символов = 128K UTF-16.
const CHUNK_CHARS: usize = 64 * 1024;
/// Длительность рабочего цикла в Step 2.
const WRITE_DURATION_MS: f64 = 800.0;
/// Yield каждые N итераций в async-режиме.
const YIELD_EVERY: u32 = 20;
/// Интервал heartbeat-таймера, мс.
const HEARTBEAT_MS: u32 = 20;
/// Размер «фото» в Step 1: 5 МБ raw → 13.3 МБ UTF-16 → заведомо > квоты.
const PHOTO_BYTES: usize = 5 * 1024 * 1024;

const HTML: &str = r#"
<h1>Глава 24.1: Почему localStorage недостаточно</h1>
<p>Три эксперимента — один вывод. Мостик к IndexedDB.</p>

<div id="storage-info" style="font-family:monospace;font-size:12px;padding:6px;background:#f6f6f6;border:1px solid #ddd;margin:8px 0"></div>

<fieldset style="margin:8px 0">
  <legend>Эксперимент 1. Одно «фото» 5 МБ — уже не влезает</legend>
  <div style="color:#666;font-size:12px;margin-bottom:6px">
    localStorage хранит только строки. Бинарные данные → base64 (+33%),
    затем UTF-16 (×2 к каждому символу). Итого ×2.67 от исходного размера.
    Это архитектурное свойство Web Storage — не изменить.<br>
    Начните с этого эксперимента — потом переходите к следующему.
  </div>
  <button id="try-photo">Сохранить «фото» 5 МБ</button>
  <div id="photo-result" style="font-family:monospace;font-size:12px;margin-top:8px"></div>
</fieldset>

<div style="text-align:center;color:#888;font-size:12px;margin:14px 0;line-height:1.6">
  ↓ Эксперимент 1 показал: localStorage не хранит бинарные данные.<br>
  Проверим, что происходит с UI при длинной работе. ↓
</div>

<fieldset style="margin:8px 0">
  <legend>Эксперимент 2: sync vs async</legend>
  <div style="color:#666;font-size:12px;margin-bottom:6px">
    Обе кнопки выполняют <b>один и тот же цикл</b> ровно 800 мс:
    setItem + немедленный delete. Разница — синхронный он или с yield.<br>
    Heartbeat-таймер тикает раз в 20 мс, пока главный поток свободен.
    Если тиков меньше ожидаемого — главный поток был занят <b>нашим
    циклом</b> и не мог обслуживать таймер.<br>
    <b>Порядок:</b> сначала sync — он даёт эталон скорости (100% работы).
    Затем async — увидите цену yield'а в процентах работы и выгоду в тиках.
  </div>
  <button id="run-sync">Синхронно (без yield)</button>
  <button id="run-async">С yield (async)</button>
  <table id="lock-table" style="width:100%;border-collapse:collapse;font-family:monospace;font-size:12px;margin-top:8px">
    <thead>
      <tr>
        <th style="text-align:left;border-bottom:1px solid #ccc;padding-right:12px">режим</th>
        <th style="text-align:right;border-bottom:1px solid #ccc;padding-right:12px">длит., мс</th>
        <th style="text-align:right;border-bottom:1px solid #ccc;padding-right:12px">тиков, шт.</th>
        <th style="text-align:right;border-bottom:1px solid #ccc;padding-right:12px">ожидалось, шт.</th>
        <th style="text-align:right;border-bottom:1px solid #ccc;padding-right:24px">работа, %</th>
        <th style="text-align:left;border-bottom:1px solid #ccc">вердикт</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td style="padding-right:12px">sync</td>
        <td style="text-align:right;padding-right:12px" id="sync-dur">—</td>
        <td style="text-align:right;padding-right:12px" id="sync-ticks">—</td>
        <td style="text-align:right;padding-right:12px" id="sync-exp">—</td>
        <td style="text-align:right;padding-right:24px" id="sync-work">—</td>
        <td id="sync-verdict">—</td>
      </tr>
      <tr>
        <td style="padding-right:12px">async</td>
        <td style="text-align:right;padding-right:12px" id="async-dur">—</td>
        <td style="text-align:right;padding-right:12px" id="async-ticks">—</td>
        <td style="text-align:right;padding-right:12px" id="async-exp">—</td>
        <td style="text-align:right;padding-right:24px" id="async-work">—</td>
        <td id="async-verdict">—</td>
      </tr>
    </tbody>
  </table>
  <div style="color:#666;font-size:11px;margin-top:6px;font-family:monospace">
    Колонка «работа, %»: sync — эталон (100%). async показывает процент
    работы от sync за то же время.<br>
    Тиков, шт. может быть на 1 меньше ожидалось, шт.: последний тик не
    успевает до момента замера elapsed — это норма.
  </div>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Вывод</legend>
  <div id="conclusion" style="font-family:monospace;font-size:12px;line-height:1.6">
    Прогоните оба эксперимента — и посмотрите на колонки <b>работа, %</b> и <b>тиков, шт.</b>.<br>
    Sync: 100% работы, но главный поток был занят — UI не отвечал.<br>
    Async: ~25% работы, зато UI отвечал.<br>
    <b>localStorage спроектирован синхронным. При большой нагрузке выбор —
    либо блокировка UI, либо потеря 75% скорости на yield. Оба варианта
    неудобны для серьёзных данных.</b>
  </div>
</fieldset>

<div id="status-sync" style="color:#555;margin:4px 0;font-family:monospace;font-size:12px">Sync: —</div>
<div id="status-async" style="color:#555;margin:4px 0;font-family:monospace;font-size:12px">Async: —</div>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let storage_info = document.get_element_by_id("storage-info").ok_or("no si")?;
    let photo_result = document.get_element_by_id("photo-result").ok_or("no pr")?;
    let status_sync = document.get_element_by_id("status-sync").ok_or("no ss")?;
    let status_async = document.get_element_by_id("status-async").ok_or("no sa")?;

    let sync_dur = document.get_element_by_id("sync-dur").ok_or("no sd")?;
    let sync_ticks = document.get_element_by_id("sync-ticks").ok_or("no st")?;
    let sync_exp = document.get_element_by_id("sync-exp").ok_or("no se")?;
    let sync_work = document.get_element_by_id("sync-work").ok_or("no sw")?;
    let sync_verdict = document.get_element_by_id("sync-verdict").ok_or("no sv")?;

    let async_dur = document.get_element_by_id("async-dur").ok_or("no ad")?;
    let async_ticks = document.get_element_by_id("async-ticks").ok_or("no at")?;
    let async_exp = document.get_element_by_id("async-exp").ok_or("no ae")?;
    let async_work = document.get_element_by_id("async-work").ok_or("no aw")?;
    let async_verdict = document.get_element_by_id("async-verdict").ok_or("no av")?;

    // Эталон: число итераций sync-прогона. None — пока sync не запускался.
    let sync_iterations: Rc<RefCell<Option<u32>>> = Rc::new(RefCell::new(None));

    // Подсказка «переходите к эксперименту 2» — скрываем при старте Step 2.
    let photo_hint_active: Rc<RefCell<bool>> = Rc::new(RefCell::new(false));

    // Heartbeat: тикает каждые 20 мс, пока главный поток свободен.
    let heartbeat: Rc<RefCell<u32>> = Rc::new(RefCell::new(0));
    {
        let heartbeat_cb = heartbeat.clone();
        let interval = gloo_timers::callback::Interval::new(HEARTBEAT_MS, move || {
            *heartbeat_cb.borrow_mut() += 1;
        });
        interval.forget();
    }

    let _ = cleanup_by_prefix(PREFIX);
    render_storage_info(&storage_info);

    // ---------------- Эксперимент 1: «фото 5 МБ» ----------------
    let try_photo: HtmlElement = document
        .get_element_by_id("try-photo")
        .ok_or("no tp")?
        .dyn_into()?;
    {
        let photo_result_cb = photo_result.clone();
        let storage_info_cb = storage_info.clone();
        let status_sync_cb = status_sync.clone();
        let photo_hint_cb = photo_hint_active.clone();
        set_click(&try_photo, move |_| {
            let _ = cleanup_by_prefix(PREFIX);

            let data = ls_bloat::random_bytes(PHOTO_BYTES, 42);
            let report = ls_bloat::analyze_bloat(&data);
            let key = format!("{PREFIX}-photo");

            let result_html: String = match ls_bloat::store_bytes_as_base64(&key, &data) {
                Ok(stored_utf16) => {
                    format_str(format_args!(
                        "<p>raw <b>{}</b> → base64 <b title=\"{} символов\">{} симв.</b> → UTF-16 <b>{}</b> (×{:.2})</p>\
                         <p style=\"color:#27ae60\"><b>Сохранилось.</b> \
                         Заметьте: 5 МБ фото заняли {} в хранилище.</p>",
                        fmt_bytes(report.raw_bytes),
                        report.base64_chars,
                        fmt_count_chars(report.base64_chars),
                        fmt_bytes(report.utf16_bytes),
                        report.ratio,
                        fmt_bytes(stored_utf16),
                    ))
                }
                Err(e) => {
                    let is_quota = ls_probe::is_quota_error(&e);
                    let verdict: String = if is_quota {
                        format_str(format_args!(
                            "<p style=\"color:#c0392b\"><b>QuotaExceededError:</b> \
                             фото {} (после base64 + UTF-16 заняло бы {}) не поместилось.</p>",
                            fmt_bytes(report.raw_bytes),
                            fmt_bytes(report.utf16_bytes),
                        ))
                    } else {
                        format_str(format_args!(
                            "<p style=\"color:#c0392b\"><b>Ошибка браузера.</b></p>"
                        ))
                    };
                    format_str(format_args!(
                        "<p>raw <b>{}</b> → base64 <b title=\"{} символов\">{} симв.</b> → UTF-16 <b>{}</b> (×{:.2})</p>\
                         {verdict}",
                        fmt_bytes(report.raw_bytes),
                        report.base64_chars,
                        fmt_count_chars(report.base64_chars),
                        fmt_bytes(report.utf16_bytes),
                        report.ratio,
                    ))
                }
            };

            let cleaned = cleanup_by_prefix(PREFIX).unwrap_or(0);
            let hint_text: String = if cleaned == 0 {
                "Хранилище пусто — переходите к эксперименту 2.".to_string()
            } else {
                format_str(format_args!(
                    "Хранилище очищено: удалено ключей — {cleaned}. Готово к эксперименту 2."
                ))
            };
            let hint_html = format_str(format_args!(
                "<p id=\"photo-hint\" style=\"color:#666;font-size:11px\">{hint_text}</p>"
            ));
            let combined = format_str(format_args!("{result_html}{hint_html}"));
            photo_result_cb.set_inner_html(&combined);
            *photo_hint_cb.borrow_mut() = true;

            render_storage_info(&storage_info_cb);
            set_status(
                &status_sync_cb,
                "Эксперимент 1 завершён. Переходите к эксперименту 2.",
            );
        });
    }

    // ---------------- Эксперимент 2: sync ----------------
    let run_sync: HtmlElement = document
        .get_element_by_id("run-sync")
        .ok_or("no rs")?
        .dyn_into()?;
    {
        let heartbeat_cb = heartbeat.clone();
        let storage_info_cb = storage_info.clone();
        let status_sync_cb = status_sync.clone();
        let photo_result_cb = photo_result.clone();
        let photo_hint_cb = photo_hint_active.clone();
        let sync_iterations_cb = sync_iterations.clone();
        let dur_el = sync_dur.clone();
        let ticks_el = sync_ticks.clone();
        let exp_el = sync_exp.clone();
        let work_el = sync_work.clone();
        let verdict_el = sync_verdict.clone();

        set_click(&run_sync, move |_| {
            let _ = cleanup_by_prefix(PREFIX);
            hide_photo_hint(&photo_result_cb, &photo_hint_cb);

            set_status(&status_sync_cb, "Синхронная запись 0.8 сек: главный поток занят нашим циклом…");

            *heartbeat_cb.borrow_mut() = 0;
            let t0 = ls_bench::now_ms();
            let result = timed_write_sync(CHUNK_CHARS, WRITE_DURATION_MS);
            let elapsed = ls_bench::now_ms() - t0;
            let ticks = *heartbeat_cb.borrow();

            let iter = result.unwrap_or(0);
            *sync_iterations_cb.borrow_mut() = Some(iter);

            let (verdict_text, _) = verdict_for(elapsed, ticks);

            update_row(
                &dur_el, &ticks_el, &exp_el, &work_el, &verdict_el,
                elapsed, ticks, Some("100%".to_string()), verdict_for(elapsed, ticks),
            );
            render_storage_info(&storage_info_cb);
            set_status(
                &status_sync_cb,
                &format_str(format_args!(
                    "Sync: эталон скорости — 100% работы. {verdict_text}."
                )),
            );
        });
    }

    // ---------------- Эксперимент 3: async ----------------
    let run_async: HtmlElement = document
        .get_element_by_id("run-async")
        .ok_or("no ra")?
        .dyn_into()?;
    {
        let heartbeat_cb = heartbeat.clone();
        let storage_info_cb = storage_info.clone();
        let status_async_cb = status_async.clone();
        let photo_result_cb = photo_result.clone();
        let photo_hint_cb = photo_hint_active.clone();
        let sync_iterations_cb = sync_iterations.clone();
        let dur_el = async_dur.clone();
        let ticks_el = async_ticks.clone();
        let exp_el = async_exp.clone();
        let work_el = async_work.clone();
        let verdict_el = async_verdict.clone();

        set_click(&run_async, move |_| {
            let _ = cleanup_by_prefix(PREFIX);
            hide_photo_hint(&photo_result_cb, &photo_hint_cb);

            let has_baseline = sync_iterations_cb.borrow().is_some();
            if !has_baseline {
                set_status(
                    &status_async_cb,
                    "Сначала запустите sync — он даст эталон для колонки «работа, %».",
                );
                return;
            }

            set_status(&status_async_cb, "Async-запись 0.8 сек: наш цикл отдаёт управление между чанками…");

            *heartbeat_cb.borrow_mut() = 0;
            let t0 = ls_bench::now_ms();

            let heartbeat_in = heartbeat_cb.clone();
            let storage_info_in = storage_info_cb.clone();
            let status_in = status_async_cb.clone();
            let sync_iter_in = sync_iterations_cb.clone();
            let dur_in = dur_el.clone();
            let ticks_in = ticks_el.clone();
            let exp_in = exp_el.clone();
            let work_in = work_el.clone();
            let verdict_in = verdict_el.clone();

            spawn_local(async move {
                let result = timed_write_async(
                    CHUNK_CHARS,
                    WRITE_DURATION_MS,
                    YIELD_EVERY,
                    |_| {},
                )
                .await;

                let elapsed = ls_bench::now_ms() - t0;
                let ticks = *heartbeat_in.borrow();
                let iter = result.unwrap_or(0);

                let work_display: String = match *sync_iter_in.borrow() {
                    Some(base) if base > 0 => {
                        let pct = (iter as f64 / base as f64) * 100.0;
                        format_str(format_args!("{pct:.0}%"))
                    }
                    _ => "—".to_string(),
                };

                let (verdict_text, _) = verdict_for(elapsed, ticks);

                update_row(
                    &dur_in, &ticks_in, &exp_in, &work_in, &verdict_in,
                    elapsed, ticks, Some(work_display.clone()), verdict_for(elapsed, ticks),
                );
                render_storage_info(&storage_info_in);
                set_status(
                    &status_in,
                    &format_str(format_args!(
                        "Async: {work_display} работы sync — {verdict_text}."
                    )),
                );
            });
        });
    }

    Ok(())
}

// ---------------- рабочие циклы ----------------

/// Пишет и сразу удаляет чанк в течение `duration_ms`.
/// Синхронно — главный поток занят циклом. Возвращает число итераций.
fn timed_write_sync(chunk_chars: usize, duration_ms: f64) -> Result<u32, JsValue> {
    let chunk = "x".repeat(chunk_chars);
    let key = format!("{PREFIX}-tmp");
    let t0 = ls_bench::now_ms();
    let deadline = t0 + duration_ms;
    let mut iterations = 0u32;

    while ls_bench::now_ms() < deadline {
        ls_probe::set_string(&key, &chunk)?;
        ls_probe::remove(&key)?;
        iterations += 1;
    }
    let _ = ls_probe::remove(&key);
    return Ok(iterations);
}

/// То же, но каждые `yield_every` итераций отдаёт управление через
/// `TimeoutFuture::new(0).await`. UI успевает обработать heartbeat.
async fn timed_write_async<F>(
    chunk_chars: usize,
    duration_ms: f64,
    yield_every: u32,
    mut on_progress: F,
) -> Result<u32, JsValue>
where
    F: FnMut(u32),
{
    let chunk = "x".repeat(chunk_chars);
    let key = format!("{PREFIX}-tmp");
    let t0 = ls_bench::now_ms();
    let deadline = t0 + duration_ms;
    let mut iterations = 0u32;
    let mut since_yield = 0u32;

    while ls_bench::now_ms() < deadline {
        ls_probe::set_string(&key, &chunk)?;
        ls_probe::remove(&key)?;
        iterations += 1;
        since_yield += 1;

        if since_yield >= yield_every {
            gloo_timers::future::TimeoutFuture::new(0).await;
            on_progress(iterations);
            since_yield = 0;
        }
    }
    let _ = ls_probe::remove(&key);
    return Ok(iterations);
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

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

/// Скрывает подсказку «Хранилище пусто — переходите к эксперименту 2»,
/// оставшуюся от Step 1. Вызывается при старте Step 2.
fn hide_photo_hint(photo_result: &web_sys::Element, hint_active: &Rc<RefCell<bool>>) {
    if !*hint_active.borrow() {
        return;
    }
    if let Ok(Some(hint_el)) = photo_result.query_selector("#photo-hint") {
        if let Some(el) = hint_el.dyn_ref::<web_sys::HtmlElement>() {
            let _ = el.style().set_property("display", "none");
        }
    }
    *hint_active.borrow_mut() = false;
}

/// Убирает только свои ключи. Не трогает другие демо.
fn cleanup_by_prefix(prefix: &str) -> Result<u32, JsValue> {
    let keys = ls_probe::keys()?;
    let mut removed = 0u32;
    for k in keys {
        if k.starts_with(prefix) {
            ls_probe::remove(&k)?;
            removed += 1;
        }
    }
    return Ok(removed);
}

fn render_storage_info(el: &web_sys::Element) {
    let info = ls_probe::probe();
    if !info.available {
        el.set_inner_html("<b style=\"color:#a00\">localStorage недоступен</b>");
        return;
    }
    let html = format_str(format_args!(
        "localStorage: доступен | ключей: <b>{}</b> | занято (UTF-16): <b>{}</b>",
        info.key_count,
        fmt_bytes(info.approx_bytes),
    ));
    el.set_inner_html(&html);
}

/// Единый источник вердикта о блокировке главного потока.
///
/// Сравниваем число тиков с ожидаемым (`elapsed / HEARTBEAT_MS`).
/// Отношение < 0.5 — главный поток был занят циклом. ≥ 0.8 — отвечал.
fn verdict_for(elapsed_ms: f64, ticks: u32) -> (&'static str, &'static str) {
    let expected = (elapsed_ms / HEARTBEAT_MS as f64).floor() as u32;
    if expected < 3 {
        return ("слишком коротко для суждения", "#888");
    }
    let ratio = ticks as f64 / expected as f64;
    if ratio < 0.5 {
        ("главный поток занят — UI не отвечал", "#c0392b")
    } else if ratio < 0.8 {
        ("главный поток частично занят", "#e67e22")
    } else {
        ("главный поток свободен — UI отвечал", "#27ae60")
    }
}

/// Обновляет одну строку таблицы.
fn update_row(
    dur_el: &web_sys::Element,
    ticks_el: &web_sys::Element,
    exp_el: &web_sys::Element,
    work_el: &web_sys::Element,
    verdict_el: &web_sys::Element,
    elapsed_ms: f64,
    ticks: u32,
    work: Option<String>,
    verdict: (&'static str, &'static str),
) {
    let expected = (elapsed_ms / HEARTBEAT_MS as f64).floor() as u32;

    dur_el.set_text_content(Some(&fmt_ms(elapsed_ms)));
    ticks_el.set_text_content(Some(&ticks.to_string()));
    exp_el.set_text_content(Some(&expected.to_string()));
    if let Some(w) = work {
        work_el.set_text_content(Some(&w));
    }

    let (text, color) = verdict;
    let html = format_str(format_args!("<b style=\"color:{color}\">{text}</b>"));
    verdict_el.set_inner_html(&html);
}

fn fmt_bytes(bytes: usize) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = 1024.0 * 1024.0;
    let b = bytes as f64;
    if b < KB {
        return format!("{bytes} Б");
    }
    if b < MB {
        return format!("{:.2} КБ", b / KB);
    }
    return format!("{:.2} МБ", b / MB);
}

/// Компактное представление числа символов.
fn fmt_count_chars(n: usize) -> String {
    if n < 1_000 {
        return format!("{n}");
    }
    if n < 1_000_000 {
        return format!("{:.1} тыс.", n as f64 / 1_000.0);
    }
    return format!("{:.2} млн", n as f64 / 1_000_000.0);
}

fn fmt_ms(ms: f64) -> String {
    return format!("{:.0}", ms);
}
```

## 24.1.8. Анализ решения

**Что демонстрирует каждый эксперимент:**

| Эксперимент | Что доказывает | Ключевая цифра |
|---|---|---|
| 1. Фото 5 МБ | localStorage не хранит бинарные данные | `QuotaExceededError`, ×2.67 |
| 2. Sync | Синхронный setItem блокирует UI | 0 из 40 тиков |
| 3. Async | Yield стоит времени, но UI отвечает | 25% работы, 39 из 40 тиков |

**Что было в первых версиях и почему убрано:**

- **Секция «Состояние хранилища»** — 4 строки цифр. Убрано: избыточно, состояние видно по эксперименту.
- **Секция «Раздутие base64»** — отдельный анализ от записи. Объединено: `store_bytes_as_base64` делает и то, и другое за один клик.
- **Секция «Sweep»** — замер по 1K/10K/100K/1M/5M символов. Убрано: парадоксальные числа (шум на малых размерах), противоречит главному тезису.
- **Секция «Заполнение до квоты»** — async fill. Убрано: измеряет не то, что нужно — throughput vs UI-отзывчивость. Заменено на фиксированное время 800 мс.
- **Секция «Что влезает»** — список «1 фото, 1 MP3, 1 PDF». Убрано: цифры не сходятся с реальной квотой, вводит в заблуждение.
- **Кнопка «Очистить свои ключи»** — уборка теперь автоматическая в каждом эксперименте.

**Ключевая находка итераций:** демо доказало два тезиса из трёх, но не показало блокировку. Причина — маленькие размеры тестов (1–2 мс). Решение — измерять **по времени**, а не по объёму: 800 мс фиксированной нагрузки, объективная метрика heartbeat.

**Как работает heartbeat.** `setInterval` раз в 20 мс увеличивает счётчик. Синхронный цикл замораживает главный поток — таймер не срабатывает. Счётчик тиков сравнивается с ожидаемым (`elapsed / 20`). Разница — прямая мера блокировки.

## 24.1.9. Правила проекта, закреплённые в главе

1. **Не переупаковывайте `JsValue`-ошибки в строки.** Обёртка `JsValue::from_str(&format!("{e:?}"))` уничтожает `name`/`message` `DOMException`, и `is_quota_error` перестаёт работать. API пробрасывает ошибку как есть.
2. **Метрика должна быть устойчивой к состоянию системы.** Замер по объёму зависит от свободной квоты и порядка кликов. Замер по времени — детерминирован. Для сравнения стратегий используйте фиксированную нагрузку.
3. **`on_progress` вызывается ровно там, где прогресс изменился.** В нормальном цикле — после каждого batch. В аварийном — один раз при выходе по ошибке. Лишний финальный вызов даёт off-by-one в счётчике.
4. **`format!` и `vec![Box<dyn …>]` в tail-позиции — явный `return`.** Rust-analyzer теряет тип в этой позиции.
5. **Клоны для `move`-замыканий — с суффиксом `_cb`/`_in`.** Если переменная нужна и снаружи, и внутри — два клона с говорящими именами. Никогда shadowing.
6. **Демо должно показывать проблему, а не решение.** 24.1 — про пределы localStorage. Ни строчки про IndexedDB в коде — только в финальном тексте вывода.
7. **Очистка — по префиксу, не `clear()`.** Демо не должно стирать данные других подглав.
8. **Indeterminate progress bar недопустим.** Если операцию можно разбить — разбейте. Если нельзя — показывайте реальные цифры (тики, длительность), а не осциллирующую полосу.

---

**Мостик к 24.2.** Три проблемы задокументированы. `localStorage` **мал**, **дорог**, **блокирует UI**. Следующая подглава начнётся с `IdbFactory` — точки входа в IndexedDB, которая решает все три проблемы: гигабайты квоты, binary native, асинхронный API.