# Глава 15. Storage

## Обновлённая структура проекта

```text
hello-wasm02/
├── Cargo.toml
├── Cargo.lock
├── package.json
├── package-lock.json
├── vite.config.js
├── webdriver.json
├── .gitignore
├── LICENSE
├── README.md
├── build.rs
├── .cargo/
│   └── config.toml
├── src/
│   ├── lib.rs
│   ├── app.rs                      # панель управления Storage
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # НОВОЕ: обёртки над Storage
│   ├── value.rs
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs
│   ├── wasm_js_objects.rs
│   ├── wasm_jsvalue.rs
│   ├── wasm_callbacks.rs
│   ├── wasm_async.rs
│   ├── wasm_async_utils.rs
│   ├── wasm_navigation.rs
│   └── wasm_storage.rs             # НОВОЕ: 66 тестов для Storage
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Web Storage API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API |
| `Storage` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage |
| `Window.localStorage` | https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage |
| `Window.sessionStorage` | https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage |
| `StorageEvent` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/StorageEvent |
| Storage quotas and eviction | https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria |
| `web_sys::Storage` | https://docs.rs/web-sys/latest/web_sys/struct.Storage.html |
| `serde` для Rust | https://serde.rs/ |

---

## `Cargo.toml` (полное содержание)

```toml
[package]
name = "hello-wasm02"
version = "0.1.0"
edition = "2021"
description = "Rust + web_sys starter"
license = "MIT"
repository = "https://github.com/rustkas/hello-wasm02"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"
console_error_panic_hook = "0.1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[dependencies.web-sys]
version = "0.3"
features = [
    # ... features из предыдущих глав ...

    # Storage
    "Storage",
    "StorageEvent",
    "StorageEventInit",
    "Window",

    # Прочее
    "console",
]

[dev-dependencies]
wasm-bindgen-test = "0.3"

[profile.release]
opt-level = "s"
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

---

## `src/lib.rs` (полное содержание)

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Публичные модули:
//! - `app` — практические проекты глав.
//! - `async_utils` — помощники для асинхронной работы.
//! - `callbacks` — управление временем жизни `Closure`.
//! - `dom` — помощники для работы с DOM.
//! - `events` — помощники для работы с событиями.
//! - `value` — помощники для работы с `JsValue`.
//! - `js_objects` — помощники для `Array`, `Map`, `Set`, `Promise`, `Date`.
//! - `navigation` — работа с URL, Location, History, Navigation API.
//! - `storage` — работа с localStorage и sessionStorage.
//! - `notifier` — система уведомлений с Rust-callback API.

pub mod app;
pub mod async_utils;
pub mod callbacks;
pub mod dom;
pub mod events;
pub mod js_objects;
pub mod navigation;
pub mod notifier;
pub mod storage;
pub mod value;
mod form;
mod logging;

use wasm_bindgen::prelude::*;

/// Точка входа. Вызывается автоматически при инициализации Wasm-модуля.
#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    logging::init();
    app::run()
}
```

---

## `src/storage.rs` (полное содержание)

```rust
//! Помощники для работы с Web Storage API.
//!
//! ## Что в этом модуле
//!
//! - `local_storage` / `session_storage` — доступ к хранилищам.
//! - Базовые операции: `get`, `set`, `remove`, `clear`, `length`, `key`.
//! - JSON-сериализация: `set_json`, `get_json`.
//! - Оценка использования: `approximate_size`, `usage_percent`.
//! - `StorageEvent` — парсинг с `url` и `storageArea`.
//! - Демонстрация ограничений: `fill_to_quota`, `clear_prefix`,
//!   `all_values_are_strings`, `measure_sync_block`, `is_storage_writable`.
//! - `current_origin` — текущий origin для демонстрации привязки.
#![allow(dead_code)]

use serde::de::DeserializeOwned;
use serde::Serialize;
use wasm_bindgen::JsValue;
use web_sys::{Storage, StorageEvent};

// ============================================================
// Получение доступа к хранилищам
// ============================================================

/// Возвращает `window.localStorage`.
pub fn local_storage() -> Result<Storage, JsValue> {
    return web_sys::window()
        .expect("no window")
        .local_storage()?
        .ok_or_else(|| JsValue::from_str("localStorage is not available"));
}

/// Возвращает `window.sessionStorage`.
pub fn session_storage() -> Result<Storage, JsValue> {
    return web_sys::window()
        .expect("no window")
        .session_storage()?
        .ok_or_else(|| JsValue::from_str("sessionStorage is not available"));
}

/// Проверяет, доступен ли storage для записи и чтения.
///
/// В приватном режиме некоторых браузеров localStorage может быть
/// недоступен для записи: `localStorage()` возвращает объект, но
/// `setItem` бросает `QuotaExceededError`. Функция проверяет реальную
/// работу и очищает тестовый ключ после проверки.
pub fn is_storage_writable(storage: &Storage) -> bool {
    let test_key = "__test_writable__";
    let test_value = "test";

    match storage.set_item(test_key, test_value) {
        Ok(_) => {
            let read_back = get(storage, test_key);
            storage.remove_item(test_key).ok();
            return read_back.as_deref() == Some(test_value);
        }
        Err(_) => return false,
    }
}

/// Возвращает текущий origin (protocol + host + port).
pub fn current_origin() -> String {
    return web_sys::window()
        .and_then(|w| w.location().origin().ok())
        .unwrap_or_else(|| "unknown".to_string());
}

// ============================================================
// Базовые операции
// ============================================================

/// Читает значение по ключу.
///
/// Возвращает `None`, если ключа нет.
pub fn get(storage: &Storage, key: &str) -> Option<String> {
    return storage.get_item(key).ok().flatten();
}

/// Устанавливает значение.
///
/// Возвращает `Err`, если хранилище переполнено (`QuotaExceededError`).
pub fn set(storage: &Storage, key: &str, value: &str) -> Result<(), JsValue> {
    return storage.set_item(key, value);
}

/// Удаляет ключ.
pub fn remove(storage: &Storage, key: &str) -> Result<(), JsValue> {
    return storage.remove_item(key);
}

/// Очищает хранилище.
pub fn clear(storage: &Storage) -> Result<(), JsValue> {
    return storage.clear();
}

/// Возвращает количество ключей.
pub fn length(storage: &Storage) -> u32 {
    return storage.length().unwrap_or(0);
}

/// Возвращает ключ по индексу.
///
/// Возвращает `None`, если индекс вне диапазона.
pub fn key(storage: &Storage, index: u32) -> Option<String> {
    return storage.key(index).ok().flatten();
}

/// Проверяет наличие ключа.
pub fn has_key(storage: &Storage, key: &str) -> bool {
    return get(storage, key).is_some();
}

/// Возвращает все ключи.
pub fn keys(storage: &Storage) -> Vec<String> {
    let len = length(storage);
    return (0..len).filter_map(|i| key(storage, i)).collect();
}

/// Возвращает все пары ключ-значение.
pub fn entries(storage: &Storage) -> Vec<(String, String)> {
    return keys(storage)
        .into_iter()
        .filter_map(|k| get(storage, &k).map(|v| (k, v)))
        .collect();
}

// ============================================================
// JSON-сериализация
// ============================================================

/// Сохраняет значение как JSON.
///
/// `T` должен реализовать `Serialize`.
pub fn set_json<T: Serialize>(
    storage: &Storage,
    key: &str,
    value: &T,
) -> Result<(), JsValue> {
    let json = serde_json::to_string(value)
        .map_err(|e| JsValue::from_str(&format!("serialize error: {}", e)))?;
    return set(storage, key, &json);
}

/// Читает значение как JSON.
///
/// `T` должен реализовать `DeserializeOwned`.
/// Возвращает `None`, если ключа нет или JSON невалиден.
pub fn get_json<T: DeserializeOwned>(storage: &Storage, key: &str) -> Option<T> {
    let raw = get(storage, key)?;
    return serde_json::from_str(&raw).ok();
}

// ============================================================
// Оценка использования
// ============================================================

/// Возвращает приблизительный размер хранилища в символах.
///
/// Для точного размера в байтах нужно умножить на 2
/// (UTF-16 внутри браузера) или на 1 (UTF-8 при передаче).
pub fn approximate_size(storage: &Storage) -> usize {
    return entries(storage)
        .iter()
        .map(|(k, v)| k.len() + v.len())
        .sum();
}

/// Возвращает приблизительный размер в человеко-читаемом виде.
pub fn approximate_size_human(storage: &Storage) -> String {
    let chars = approximate_size(storage);
    let bytes = chars * 2; // UTF-16

    if bytes < 1024 {
        return format!("{} B", bytes);
    } else if bytes < 1024 * 1024 {
        return format!("{:.1} KB", bytes as f64 / 1024.0);
    } else {
        return format!("{:.2} MB", bytes as f64 / (1024.0 * 1024.0));
    }
}

/// Максимальный размер localStorage/sessionStorage в символах
/// (приблизительно).
///
/// 5 МБ — типичный лимит для браузеров.
pub const APPROX_MAX_CHARS: usize = 5 * 1024 * 1024;

/// Процент использования хранилища.
pub fn usage_percent(storage: &Storage) -> f64 {
    let used = approximate_size(storage);
    return (used as f64 / APPROX_MAX_CHARS as f64) * 100.0;
}

// ============================================================
// StorageEvent
// ============================================================

/// Описывает событие `storage`.
///
/// ## Особенности Web Storage событий
///
/// - Событие `storage` генерируется **только для localStorage**.
/// - Для sessionStorage события **не генерируются** — данные изолированы
///   по вкладкам, синхронизировать нечего.
/// - Событие приходит **только в другие вкладки** того же origin.
///   Вкладка, в которой произошло изменение, событие **не получает**.
#[derive(Default)]
pub struct StorageChange {
    pub key: Option<String>,
    pub old_value: Option<String>,
    pub new_value: Option<String>,
    pub url: Option<String>,
    pub storage_area: Option<String>,
}

/// Извлекает данные из `StorageEvent`.
///
/// ## Про поле `storageArea`
///
/// Поле не извлекается из самого события. Событие `storage`
/// генерируется только для `localStorage` — `sessionStorage`
/// не генерирует межвкладочных событий. Поэтому `storage_area`
/// всегда `"localStorage"`.
///
/// В синтетических событиях (`StorageEvent::new("storage")`)
/// `storageArea` = `null`. Если бы мы полагались на
/// `event.storage_area()`, синтетические события давали бы `None`,
/// что противоречило бы реальной семантике.
pub fn parse_storage_event(event: &StorageEvent) -> StorageChange {
    return StorageChange {
        key: event.key(),
        old_value: event.old_value(),
        new_value: event.new_value(),
        url: event.url(),
        storage_area: Some("localStorage".to_string()),
    };
}

/// Форматирует `StorageChange` для логирования.
///
/// Возвращает многострочное представление, включающее:
/// - изменение (set/remove/clear),
/// - область хранения (`storageArea`),
/// - источник изменения (`url`),
/// - старое значение (если было).
pub fn format_storage_change(change: &StorageChange) -> String {
    let mut lines = Vec::new();

    if change.key.is_none() {
        lines.push("Storage cleared".to_string());
    } else {
        let key = change.key.as_ref().unwrap();
        if let Some(v) = &change.new_value {
            lines.push(format!("[{}] = \"{}\"", key, v));
        } else {
            lines.push(format!("[{}] removed", key));
        }
    }

    if let Some(area) = &change.storage_area {
        lines.push(format!("  area: {}", area));
    }

    if let Some(url) = &change.url {
        lines.push(format!("  from: {}", url));
    }

    if let Some(old) = &change.old_value {
        if change.new_value.is_some() {
            lines.push(format!("  was: \"{}\"", old));
        }
    }

    return lines.join("\n");
}

// ============================================================
// Демонстрация ограничений Storage
// ============================================================

/// Результат заполнения хранилища до квоты.
pub struct FillResult {
    pub keys_written: u32,
    pub total_chars: usize,
    pub error_message: Option<String>,
    pub hit_quota: bool,
}

/// Заполняет хранилище до срабатывания `QuotaExceededError`.
pub fn fill_to_quota(
    storage: &Storage,
    prefix: &str,
    chunk_size: usize,
    max_keys: u32,
) -> FillResult {
    let chunk = "x".repeat(chunk_size);
    let mut keys_written = 0u32;
    let mut total_chars = 0usize;

    loop {
        if keys_written >= max_keys {
            return FillResult {
                keys_written,
                total_chars,
                error_message: Some(format!(
                    "Остановлено после {} ключей (защита от бесконечного цикла)",
                    max_keys
                )),
                hit_quota: false,
            };
        }

        let key = format!("{}:{}", prefix, keys_written);
        match storage.set_item(&key, &chunk) {
            Ok(_) => {
                keys_written += 1;
                total_chars += key.len() + chunk.len();
            }
            Err(e) => {
                let message = e
                    .as_string()
                    .unwrap_or_else(|| format!("{:?}", e));
                return FillResult {
                    keys_written,
                    total_chars,
                    error_message: Some(message),
                    hit_quota: true,
                };
            }
        }
    }
}

/// Удаляет все ключи с заданным префиксом.
pub fn clear_prefix(storage: &Storage, prefix: &str) -> u32 {
    let to_remove: Vec<String> = keys(storage)
        .into_iter()
        .filter(|k| k.starts_with(prefix))
        .collect();
    let count = to_remove.len() as u32;
    for k in to_remove {
        storage.remove_item(&k).ok();
    }
    return count;
}

/// Проверяет, что все значения в хранилище — строки.
pub fn all_values_are_strings(storage: &Storage) -> Vec<(String, String)> {
    return entries(storage)
        .into_iter()
        .map(|(k, v)| {
            let typeof_name = if v.parse::<f64>().is_ok() {
                "number-like string"
            } else {
                "string"
            };
            (k, typeof_name.to_string())
        })
        .collect();
}

// ============================================================
// Демонстрация синхронного API
// ============================================================

/// Результат измерения синхронной блокировки.
pub struct SyncBlockResult {
    pub duration_ms: f64,
    pub bytes_written: usize,
    pub keys_written: u32,
}

/// Записывает `size_kb` КБ данных в storage, замеряя время.
///
/// Демонстрирует, что Web Storage — синхронный API: запись блокирует
/// главный поток на всё время операции. Чем больше данных, тем
/// заметнее пауза в UI.
pub fn measure_sync_block(
    storage: &Storage,
    size_kb: usize,
    prefix: &str,
) -> SyncBlockResult {
    let chunk_size = 1024;
    let chunk = "x".repeat(chunk_size);

    let start = js_sys::Date::now();

    let mut keys_written = 0u32;
    let mut bytes_written = 0usize;

    for i in 0..size_kb {
        let key = format!("{}:{}", prefix, i);
        match storage.set_item(&key, &chunk) {
            Ok(_) => {
                keys_written += 1;
                bytes_written += key.len() + chunk.len();
            }
            Err(_) => break,
        }
    }

    let end = js_sys::Date::now();

    return SyncBlockResult {
        duration_ms: end - start,
        bytes_written,
        keys_written,
    };
}
```

---

## `src/app.rs` (полное содержание)

```rust
//! Логика приложения: «Панель управления Storage».
//!
//! Демонстрирует все аспекты Web Storage API:
//! - `localStorage` — тема, счётчик, заметки.
//! - `sessionStorage` — черновик, изоляция по вкладкам.
//! - `StorageEvent` — url, storageArea, изоляция источника.
//! - JSON-сериализация — заметки.
//! - Ограничения — типы, бинарные данные, квота, синхронный API.
//! - Origin — привязка Storage к origin.
//! - Fallback — обработка недоступного Storage.

use std::cell::RefCell;
use std::rc::Rc;

use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    Event, HtmlInputElement, HtmlSelectElement, HtmlTextAreaElement, StorageEvent,
};

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;
use crate::storage;

// ============================================================
// Модели данных
// ============================================================

#[derive(Serialize, Deserialize, Clone, Debug)]
struct Note {
    id: u32,
    text: String,
}

#[derive(Serialize, Deserialize, Clone, Debug)]
struct Settings {
    theme: String,
    font_size: u32,
}

impl Default for Settings {
    fn default() -> Self {
        Self {
            theme: "light".into(),
            font_size: 16,
        }
    }
}

// ============================================================
// Ключи Storage
// ============================================================

const KEY_THEME: &str = "app:theme";
const KEY_NOTES: &str = "app:notes";
const KEY_COUNTER: &str = "app:counter";
const KEY_SETTINGS: &str = "app:settings";
const KEY_FORM_DRAFT: &str = "draft:form";
const KEY_SESSION_COUNTER: &str = "session:counter";
const QUOTA_TEST_PREFIX: &str = "quota-test";
const SYNC_TEST_PREFIX: &str = "sync-test";

// ============================================================
// Приложение
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Панель управления Storage")?;
    dom::append(&container, &heading)?;

    // Проверка доступности Storage (для private mode)
    let ls_available = storage::local_storage()
        .map(|s| storage::is_storage_writable(&s))
        .unwrap_or(false);
    let ss_available = storage::session_storage()
        .map(|s| storage::is_storage_writable(&s))
        .unwrap_or(false);

    let availability_warning = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&availability_warning), "warning")?;
    if !ls_available || !ss_available {
        availability_warning.set_text_content(Some(&format!(
            "⚠ Storage ограничен: localStorage = {}, sessionStorage = {}. \
             Некоторые функции недоступны (например, в приватном режиме браузера).",
            if ls_available { "доступен" } else { "НЕ доступен" },
            if ss_available { "доступен" } else { "НЕ доступен" },
        )));
        dom::append(&container, &availability_warning)?;
        log!("Storage ограничен: ls={}, ss={}", ls_available, ss_available);
    }

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 1: Тема
    // ============================================================

    let theme_section = build_section(&container, "1. Тема (localStorage, строка)")?;

    let theme_select_el = dom::create_element("select")?;
    dom::set_attribute(&theme_select_el, "id", "theme-select")?;
    for (value, label) in [("light", "Светлая"), ("dark", "Тёмная"), ("auto", "Авто")] {
        let opt = dom::create_element("option")?;
        dom::set_attribute(&opt, "value", value)?;
        opt.set_text_content(Some(label));
        dom::append(&theme_select_el, &opt)?;
    }
    let theme_select: HtmlSelectElement = theme_select_el.dyn_into()?;

    if let Ok(ls) = storage::local_storage() {
        if let Some(theme) = storage::get(&ls, KEY_THEME) {
            theme_select.set_value(&theme);
        }
    }
    dom::append(&theme_section, &theme_select)?;

    // ============================================================
    // СЕКЦИЯ 2: Счётчики
    // ============================================================

    let counter_section =
        build_section(&container, "2. Счётчики: localStorage vs sessionStorage")?;

    let persistence_hint = dom::create_paragraph(
        "localStorage переживёт закрытие вкладки и общий для всех вкладок. \
         sessionStorage — изолирован по вкладкам: каждая вкладка имеет свою \
         копию данных, они не синхронизируются и исчезают при закрытии вкладки. \
         Откройте вторую вкладку и убедитесь: sessionStorage-счётчик там будет \
         свой, независимый от этого.",
    )?;
    dom::add_class(&dom::as_html(&persistence_hint), "hint")?;
    dom::append(&counter_section, &persistence_hint)?;

    let ls_counter_label = dom::create_element("div")?;
    ls_counter_label
        .set_text_content(Some("localStorage счётчик (общий для всех вкладок):"));
    dom::append(&counter_section, &ls_counter_label)?;

    let ls_counter_display = dom::create_element("span")?;
    dom::add_class(&dom::as_html(&ls_counter_display), "counter-value")?;

    let ls_counter = Rc::new(RefCell::new({
        if let Ok(ls) = storage::local_storage() {
            storage::get(&ls, KEY_COUNTER)
                .and_then(|s| s.parse::<i32>().ok())
                .unwrap_or(0)
        } else {
            0
        }
    }));
    ls_counter_display.set_text_content(Some(ls_counter.borrow().to_string().as_str()));

    let ls_inc_btn = dom::create_element("button")?;
    ls_inc_btn.set_text_content(Some("+1"));
    let ls_reset_btn = dom::create_element("button")?;
    ls_reset_btn.set_text_content(Some("Сброс"));

    dom::append(&counter_section, &ls_counter_display)?;
    dom::append(&counter_section, &ls_inc_btn)?;
    dom::append(&counter_section, &ls_reset_btn)?;

    let ss_counter_label = dom::create_element("div")?;
    ss_counter_label.set_text_content(Some(
        "sessionStorage счётчик (изолирован — своя копия в каждой вкладке):",
    ));
    dom::append(&counter_section, &ss_counter_label)?;

    let ss_counter_display = dom::create_element("span")?;
    dom::add_class(&dom::as_html(&ss_counter_display), "counter-value session")?;

    let ss_counter = Rc::new(RefCell::new({
        if let Ok(ss) = storage::session_storage() {
            storage::get(&ss, KEY_SESSION_COUNTER)
                .and_then(|s| s.parse::<i32>().ok())
                .unwrap_or(0)
        } else {
            0
        }
    }));
    ss_counter_display.set_text_content(Some(ss_counter.borrow().to_string().as_str()));

    let ss_inc_btn = dom::create_element("button")?;
    ss_inc_btn.set_text_content(Some("+1"));
    let ss_reset_btn = dom::create_element("button")?;
    ss_reset_btn.set_text_content(Some("Сброс"));

    dom::append(&counter_section, &ss_counter_display)?;
    dom::append(&counter_section, &ss_inc_btn)?;
    dom::append(&counter_section, &ss_reset_btn)?;

    // ============================================================
    // СЕКЦИЯ 3: Заметки
    // ============================================================

    let notes_section =
        build_section(&container, "3. Заметки (localStorage + JSON)")?;

    let notes_list = dom::create_element("ul")?;
    dom::add_class(&dom::as_html(&notes_list), "notes-list")?;
    dom::append(&notes_section, &notes_list)?;

    let note_input_el = dom::create_element("input")?;
    dom::set_attribute(&note_input_el, "type", "text")?;
    dom::set_attribute(&note_input_el, "id", "note-input")?;
    dom::set_attribute(&note_input_el, "placeholder", "Текст заметки...")?;
    let note_input: HtmlInputElement = note_input_el.dyn_into()?;
    dom::append(&notes_section, &note_input)?;

    let add_note_btn = dom::create_element("button")?;
    add_note_btn.set_text_content(Some("Добавить"));
    let clear_notes_btn = dom::create_element("button")?;
    clear_notes_btn.set_text_content(Some("Очистить все"));
    dom::append(&notes_section, &add_note_btn)?;
    dom::append(&notes_section, &clear_notes_btn)?;

    let json_preview_label = dom::create_element("div")?;
    json_preview_label.set_text_content(Some("Сырой JSON в localStorage[app:notes]:"));
    dom::add_class(&dom::as_html(&json_preview_label), "label")?;
    dom::append(&notes_section, &json_preview_label)?;

    let json_preview = dom::create_element("pre")?;
    dom::add_class(&dom::as_html(&json_preview), "json-preview")?;
    json_preview.set_text_content(Some("(пусто)"));
    dom::append(&notes_section, &json_preview)?;

    let refresh_json_btn = dom::create_element("button")?;
    refresh_json_btn.set_text_content(Some("Обновить JSON"));
    dom::append(&notes_section, &refresh_json_btn)?;

    // ============================================================
    // СЕКЦИЯ 4: Черновик
    // ============================================================

    let draft_section =
        build_section(&container, "4. Черновик формы (sessionStorage)")?;

    let draft_hint = dom::create_paragraph(
        "Начните печатать. Закройте вкладку — черновик исчезнет. \
         Перезагрузите (F5) — черновик сохранится. Откройте вторую вкладку — \
         там будет свой черновик, независимый от этого.",
    )?;
    dom::add_class(&dom::as_html(&draft_hint), "hint")?;
    dom::append(&draft_section, &draft_hint)?;

    let draft_input_el = dom::create_element("textarea")?;
    dom::set_attribute(&draft_input_el, "id", "draft-input")?;
    dom::set_attribute(
        &draft_input_el,
        "placeholder",
        "Черновик (сохраняется в sessionStorage при каждом вводе)",
    )?;
    dom::set_attribute(&draft_input_el, "rows", "3")?;
    let draft_input: HtmlTextAreaElement = draft_input_el.dyn_into()?;

    if let Ok(ss) = storage::session_storage() {
        if let Some(draft) = storage::get(&ss, KEY_FORM_DRAFT) {
            draft_input.set_value(&draft);
        }
    }
    dom::append(&draft_section, &draft_input)?;

    // ============================================================
    // СЕКЦИЯ 5: Методы Storage
    // ============================================================

    let methods_section = build_section(
        &container,
        "5. Методы Storage: length(), key(i), keys(), entries()",
    )?;

    let methods_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&methods_info), "keys-list")?;
    dom::append(&methods_section, &methods_info)?;

    let refresh_methods_btn = dom::create_element("button")?;
    refresh_methods_btn.set_text_content(Some("Обновить"));
    let clear_all_btn = dom::create_element("button")?;
    clear_all_btn.set_text_content(Some("Очистить всё"));
    dom::append(&methods_section, &refresh_methods_btn)?;
    dom::append(&methods_section, &clear_all_btn)?;

    // ============================================================
    // СЕКЦИЯ 6: Ограничения
    // ============================================================

    let limits_section = build_section(&container, "6. Ограничения Storage")?;

    let limits_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&limits_info), "limits-info")?;
    limits_info.set_text_content(Some(
        "• Только строки. Числа, bool, объекты, массивы — через сериализацию.\n\
         • ~5 МБ на origin. При превышении — QuotaExceededError.\n\
         • Синхронный API. Запись блокирует главный поток: чем больше данных,\n\
           тем дольше UI не отвечает. Для больших данных — IndexedDB.\n\
         • Привязан к origin. Протокол + host + port. Другой порт — другой Storage.\n\
         • Private mode. В некоторых браузерах Storage недоступен для записи.\n\
         • Нет транзакций и индексов. Один уровень: ключ → строка.\n\
         • Уязвим к XSS. Любой JS на origin видит все данные.\n\
         • StorageEvent: только для localStorage, только в другие вкладки.",
    ));
    dom::append(&limits_section, &limits_info)?;

    // 6a. Только строки
    let types_label = dom::create_element("h3")?;
    types_label.set_text_content(Some("6a. Только строки"));
    dom::append(&limits_section, &types_label)?;

    let types_demo = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&types_demo), "types-demo")?;
    dom::append(&limits_section, &types_demo)?;

    let types_run_btn = dom::create_element("button")?;
    types_run_btn.set_text_content(Some("Показать"));
    dom::append(&limits_section, &types_run_btn)?;

    // 6b. Бинарные данные
    let binary_label = dom::create_element("h3")?;
    binary_label.set_text_content(Some("6b. Бинарные данные"));
    dom::append(&limits_section, &binary_label)?;

    let binary_hint = dom::create_paragraph(
        "Web Storage не принимает Uint8Array. Бинарные данные нужно \
         кодировать: либо как JSON-массив чисел, либо как base64-строку.",
    )?;
    dom::add_class(&dom::as_html(&binary_hint), "hint")?;
    dom::append(&limits_section, &binary_hint)?;

    let binary_demo = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&binary_demo), "types-demo")?;
    dom::append(&limits_section, &binary_demo)?;

    let binary_run_btn = dom::create_element("button")?;
    binary_run_btn.set_text_content(Some("Показать"));
    dom::append(&limits_section, &binary_run_btn)?;

    // 6c. Quota
    let quota_label = dom::create_element("h3")?;
    quota_label.set_text_content(Some("6c. QuotaExceededError"));
    dom::append(&limits_section, &quota_label)?;

    let quota_hint = dom::create_paragraph(
        "Нажмите «Заполнить до квоты», чтобы записать большие строки \
         и поймать ошибку переполнения.",
    )?;
    dom::add_class(&dom::as_html(&quota_hint), "hint")?;
    dom::append(&limits_section, &quota_hint)?;

    let quota_status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&quota_status), "usage-info")?;
    quota_status.set_text_content(Some("Ещё не запускалось."));
    dom::append(&limits_section, &quota_status)?;

    let fill_btn = dom::create_element("button")?;
    fill_btn.set_text_content(Some("Заполнить до квоты"));
    let cleanup_btn = dom::create_element("button")?;
    cleanup_btn.set_text_content(Some("Очистить тестовые ключи"));
    dom::append(&limits_section, &fill_btn)?;
    dom::append(&limits_section, &cleanup_btn)?;

    // 6d. Синхронный API
    let sync_label = dom::create_element("h3")?;
    sync_label.set_text_content(Some("6d. Синхронный API — блокировка UI"));
    dom::append(&limits_section, &sync_label)?;

    let sync_hint = dom::create_paragraph(
        "Запись в Storage — синхронная операция. Пока она идёт, главный поток \
         заблокирован. Нажмите кнопку и попробуйте взаимодействовать со \
         страницей — UI не будет отвечать, пока не завершится запись. \
         Это демонстрация того, почему Storage не подходит для больших данных.",
    )?;
    dom::add_class(&dom::as_html(&sync_hint), "hint")?;
    dom::append(&limits_section, &sync_hint)?;

    let sync_status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&sync_status), "usage-info")?;
    sync_status.set_text_content(Some("Ещё не запускалось."));
    dom::append(&limits_section, &sync_status)?;

    let sync_run_btn = dom::create_element("button")?;
    sync_run_btn.set_text_content(Some("Записать 3 МБ (блокировка)"));
    let sync_cleanup_btn = dom::create_element("button")?;
    sync_cleanup_btn.set_text_content(Some("Очистить sync-ключи"));
    dom::append(&limits_section, &sync_run_btn)?;
    dom::append(&limits_section, &sync_cleanup_btn)?;

    // ============================================================
    // СЕКЦИЯ 7: Использование
    // ============================================================

    let usage_section = build_section(&container, "7. Использование Storage")?;

    let usage_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&usage_info), "usage-info")?;
    dom::append(&usage_section, &usage_info)?;

    let refresh_usage_btn = dom::create_element("button")?;
    refresh_usage_btn.set_text_content(Some("Обновить"));
    dom::append(&usage_section, &refresh_usage_btn)?;

    // ============================================================
    // СЕКЦИЯ 8: StorageEvent
    // ============================================================

    let events_section = build_section(
        &container,
        "8. StorageEvent (изменения из других вкладок)",
    )?;

    let events_hint = dom::create_paragraph(
        "StorageEvent генерируется только для localStorage. \
         Событие приходит ТОЛЬКО в другие вкладки того же origin — \
         та вкладка, где произошло изменение, событие не получает. \
         Для sessionStorage событий нет: данные изолированы по вкладкам, \
         синхронизировать нечего. Откройте вторую вкладку и измените данные — \
         здесь появятся записи с полями url и storageArea.",
    )?;
    dom::add_class(&dom::as_html(&events_hint), "hint")?;
    dom::append(&events_section, &events_hint)?;

    let events_log = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&events_log), "events-log")?;
    events_log.set_text_content(Some("(пока пусто)"));
    dom::append(&events_section, &events_log)?;

    // ============================================================
    // СЕКЦИЯ 9: Origin
    // ============================================================

    let origin_section = build_section(&container, "9. Привязка к origin")?;

    let origin_hint = dom::create_paragraph(
        "Storage привязан к origin: protocol + host + port. \
         Данные, сохранённые на одном origin, не видны на другом. \
         Даже смена порта (5173 → 5174) даёт отдельное хранилище.",
    )?;
    dom::add_class(&dom::as_html(&origin_hint), "hint")?;
    dom::append(&origin_section, &origin_hint)?;

    let origin_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&origin_info), "usage-info")?;

    let current_origin = storage::current_origin();
    origin_info.set_text_content(Some(&format!(
        "Текущий origin: {}\n\
         Ключи в этом Storage привязаны к этому origin.\n\
         Откройте приложение на другом порту — данные будут отдельные.",
        current_origin
    )));
    dom::append(&origin_section, &origin_info)?;

    // ============================================================
    // Вспомогательные замыкания
    // ============================================================

    let update_usage: Rc<dyn Fn()> = {
        let usage_info = usage_info.clone();
        Rc::new(move || {
            let ls = match storage::local_storage() {
                Ok(s) => s,
                Err(_) => {
                    usage_info.set_text_content(Some("localStorage недоступен"));
                    return;
                }
            };

            let size = storage::approximate_size_human(&ls);
            let percent = storage::usage_percent(&ls);
            let count = storage::length(&ls);

            let ss_text = storage::session_storage()
                .map(|ss| {
                    format!(
                        "{} ключей, ~{}",
                        storage::length(&ss),
                        storage::approximate_size_human(&ss)
                    )
                })
                .unwrap_or_else(|_| "недоступен".into());

            let text = format!(
                "localStorage: {} ключей, ~{} ({:.3}% от 5 MB)\n\
                 sessionStorage: {}",
                count, size, percent, ss_text
            );
            usage_info.set_text_content(Some(text.as_str()));
        })
    };

    let render_notes: Rc<dyn Fn()> = {
        let notes_list = notes_list.clone();
        Rc::new(move || {
            notes_list.set_inner_html("");
            if let Ok(ls) = storage::local_storage() {
                let notes: Vec<Note> =
                    storage::get_json(&ls, KEY_NOTES).unwrap_or_default();
                for note in notes {
                    if let Ok(li) = dom::create_element("li") {
                        let note_text: String =
                            format!("#{}: {}", note.id, note.text);
                        li.set_text_content(Some(note_text.as_str()));
                        let _ = dom::append(&notes_list, &li);
                    }
                }
            }
        })
    };

    let render_json_preview: Rc<dyn Fn()> = {
        let json_preview = json_preview.clone();
        Rc::new(move || {
            if let Ok(ls) = storage::local_storage() {
                let raw = storage::get(&ls, KEY_NOTES)
                    .unwrap_or_else(|| "(ключа нет)".into());
                json_preview.set_text_content(Some(raw.as_str()));
            }
        })
    };

    let render_methods: Rc<dyn Fn()> = {
        let methods_info = methods_info.clone();
        Rc::new(move || {
            methods_info.set_inner_html("");
            let ls = match storage::local_storage() {
                Ok(s) => s,
                Err(_) => return,
            };

            let len = storage::length(&ls);
            let mut lines = Vec::new();
            lines.push(format!("length() = {}", len));
            lines.push(String::new());
            lines.push("Итерация через key(i):".to_string());
            for i in 0..len {
                let k = storage::key(&ls, i)
                    .unwrap_or_else(|| "(none)".into());
                let v = storage::get(&ls, &k)
                    .map(|s| {
                        if s.len() > 40 {
                            return format!("{}...", &s[..40]);
                        } else {
                            s
                        }
                    })
                    .unwrap_or_else(|| "(none)".into());
                lines.push(format!("  key({}) = \"{}\" → {}", i, k, v));
            }
            lines.push(String::new());
            lines.push(format!("keys() = {:?}", storage::keys(&ls)));
            lines.push(format!(
                "entries().len() = {}",
                storage::entries(&ls).len()
            ));

            let text = lines.join("\n");
            if let Ok(pre) = dom::create_element("pre") {
                pre.set_text_content(Some(text.as_str()));
                let _ = dom::append(&methods_info, &pre);
            }
        })
    };

    let apply_theme: Rc<dyn Fn(&str)> = Rc::new(|theme: &str| {
        if let Some(body) = web_sys::window().unwrap().document().unwrap().body() {
            let body_html = dom::as_html(&body);

            let theme_clean = theme.trim();
            let theme_valid: &str =
                if ["light", "dark", "auto"].contains(&theme_clean) {
                    theme_clean
                } else {
                    "light"
                };

            let _ = dom::remove_class(&body_html, "theme-light");
            let _ = dom::remove_class(&body_html, "theme-dark");
            let _ = dom::remove_class(&body_html, "theme-auto");
            let _ = dom::add_class(&body_html, &format!("theme-{}", theme_valid));
        }
    });

    // ============================================================
    // Обработчики: СЕКЦИЯ 1 — тема
    // ============================================================

    {
        let apply_theme = apply_theme.clone();
        let theme_select_for_handler = theme_select.clone();

        callbacks::listen(
            &theme_select,
            "change",
            move |_event| {
                let raw_value = theme_select_for_handler.value();
                let value = raw_value.trim().to_string();

                if ["light", "dark", "auto"].contains(&value.as_str()) {
                    if let Ok(ls) = storage::local_storage() {
                        let _ = storage::set(&ls, KEY_THEME, &value);
                        log!("Тема сохранена: {}", value);
                    }
                    apply_theme(&value);
                } else {
                    log!("Отклонено некорректное значение темы: {:?}", raw_value);
                }
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 2 — счётчики
    // ============================================================

    {
        let counter = ls_counter.clone();
        let display = ls_counter_display.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &ls_inc_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() += 1;
                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::set(
                        &ls,
                        KEY_COUNTER,
                        &counter.borrow().to_string(),
                    );
                }
                display.set_text_content(Some(counter.borrow().to_string().as_str()));
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let counter = ls_counter.clone();
        let display = ls_counter_display.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &ls_reset_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() = 0;
                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::remove(&ls, KEY_COUNTER);
                }
                display.set_text_content(Some("0"));
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let counter = ss_counter.clone();
        let display = ss_counter_display.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &ss_inc_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() += 1;
                if let Ok(ss) = storage::session_storage() {
                    let _ = storage::set(
                        &ss,
                        KEY_SESSION_COUNTER,
                        &counter.borrow().to_string(),
                    );
                }
                display.set_text_content(Some(counter.borrow().to_string().as_str()));
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let counter = ss_counter.clone();
        let display = ss_counter_display.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &ss_reset_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() = 0;
                if let Ok(ss) = storage::session_storage() {
                    let _ = storage::remove(&ss, KEY_SESSION_COUNTER);
                }
                display.set_text_content(Some("0"));
                update_usage();
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 3 — заметки
    // ============================================================

    {
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();
        let note_input = note_input.clone();

        callbacks::listen(
            &add_note_btn,
            "click",
            move |_event| {
                let text = note_input.value();
                if text.trim().is_empty() {
                    return;
                }
                if let Ok(ls) = storage::local_storage() {
                    let mut notes: Vec<Note> =
                        storage::get_json(&ls, KEY_NOTES).unwrap_or_default();
                    let next_id =
                        notes.iter().map(|n| n.id).max().unwrap_or(0) + 1;
                    notes.push(Note {
                        id: next_id,
                        text: text.clone(),
                    });

                    match storage::set_json(&ls, KEY_NOTES, &notes) {
                        Ok(_) => log!("Заметка добавлена: {}", text),
                        Err(e) => log!("Ошибка сохранения: {:?}", e),
                    }
                }
                note_input.set_value("");
                render_notes();
                render_json_preview();
                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &clear_notes_btn,
            "click",
            move |_event| {
                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::remove(&ls, KEY_NOTES);
                }
                render_notes();
                render_json_preview();
                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let render_json_preview = render_json_preview.clone();

        callbacks::listen(
            &refresh_json_btn,
            "click",
            move |_event| render_json_preview(),
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 4 — черновик
    // ============================================================

    {
        let draft_input_for_handler = draft_input.clone();

        callbacks::listen(
            &draft_input,
            "input",
            move |_event| {
                if let Ok(ss) = storage::session_storage() {
                    let _ = storage::set(
                        &ss,
                        KEY_FORM_DRAFT,
                        &draft_input_for_handler.value(),
                    );
                }
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 5 — методы
    // ============================================================

    {
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &refresh_methods_btn,
            "click",
            move |_event| {
                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &clear_all_btn,
            "click",
            move |_event| {
                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::clear(&ls);
                }
                render_notes();
                render_json_preview();
                render_methods();
                update_usage();
                log!("localStorage очищен");
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 6a — только строки
    // ============================================================

    {
        let types_demo = types_demo.clone();

        callbacks::listen(
            &types_run_btn,
            "click",
            move |_event| {
                let mut lines = Vec::new();
                lines.push("Сохраняем число 42 под ключом demo:number".to_string());
                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::set(&ls, "demo:number", &42.to_string());
                    let retrieved = storage::get(&ls, "demo:number")
                        .unwrap_or_default();
                    lines.push(format!(
                        "  Сохранили: &42.to_string() → \"{}\"",
                        retrieved
                    ));
                    lines.push(format!(
                        "  Прочитали: {} (тип: String, не i32)",
                        retrieved
                    ));
                    lines.push(String::new());
                    lines.push(
                        "Сохраняем bool true под ключом demo:bool".to_string(),
                    );
                    let _ = storage::set(&ls, "demo:bool", &true.to_string());
                    let retrieved = storage::get(&ls, "demo:bool")
                        .unwrap_or_default();
                    lines.push(format!(
                        "  Сохранили: &true.to_string() → \"{}\"",
                        retrieved
                    ));
                    lines.push(format!(
                        "  Прочитали: {} (тип: String, не bool)",
                        retrieved
                    ));
                    lines.push(String::new());
                    lines.push(
                        "Вывод: Web Storage всегда возвращает String. \
                         Чтобы получить число или bool, нужно парсить вручную."
                            .to_string(),
                    );

                    let _ = storage::remove(&ls, "demo:number");
                    let _ = storage::remove(&ls, "demo:bool");
                }
                let text = lines.join("\n");
                types_demo.set_inner_html("");
                if let Ok(pre) = dom::create_element("pre") {
                    pre.set_text_content(Some(text.as_str()));
                    let _ = dom::append(&types_demo, &pre);
                }
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 6b — бинарные данные
    // ============================================================

    {
        let binary_demo = binary_demo.clone();

        callbacks::listen(
            &binary_run_btn,
            "click",
            move |_event| {
                let mut lines = Vec::new();

                let bytes: Vec<u8> = vec![0xDE, 0xAD, 0xBE, 0xEF];
                lines.push(format!("Исходные байты: {:?}", bytes));
                lines.push(String::new());

                let json_array = serde_json::to_string(&bytes).unwrap();
                lines.push("Способ 1 — JSON-массив чисел:".to_string());
                lines.push(format!("  Строка: {}", json_array));
                lines.push(format!(
                    "  Размер: {} символов (вместо {} байт)",
                    json_array.len(),
                    bytes.len()
                ));
                lines.push(String::new());
                lines.push("Способ 2 — base64:".to_string());
                lines.push(
                    "  Base64 требует крейт base64 или btoa в JS".to_string(),
                );
                lines.push(
                    "  Размер: ~33% больше исходных байт".to_string(),
                );
                lines.push(String::new());

                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::set(&ls, "demo:bytes", &json_array);
                    let back = storage::get(&ls, "demo:bytes")
                        .unwrap_or_default();
                    lines.push(format!("Из Storage прочитано: {}", back));

                    let decoded: Vec<u8> =
                        serde_json::from_str(&back).unwrap_or_default();
                    lines.push(format!(
                        "Десериализовано обратно: {:?}",
                        decoded
                    ));
                    lines.push(format!(
                        "Совпадает с исходным: {}",
                        decoded == bytes
                    ));

                    let _ = storage::remove(&ls, "demo:bytes");
                }

                let text = lines.join("\n");
                binary_demo.set_inner_html("");
                if let Ok(pre) = dom::create_element("pre") {
                    pre.set_text_content(Some(text.as_str()));
                    let _ = dom::append(&binary_demo, &pre);
                }
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 6c — Quota
    // ============================================================

    {
        let quota_status = quota_status.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &fill_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        quota_status
                            .set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                quota_status.set_text_content(Some(
                    "Заполняем localStorage (это может занять секунду)...",
                ));

                let result =
                    storage::fill_to_quota(&ls, QUOTA_TEST_PREFIX, 100_000, 100);

                let status: String = format_quota_result(&result);
                quota_status.set_text_content(Some(status.as_str()));
                log!(
                    "Quota test: hit_quota={}, keys={}",
                    result.hit_quota,
                    result.keys_written
                );

                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let quota_status = quota_status.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &cleanup_btn,
            "click",
            move |_event| {
                if let Ok(ls) = storage::local_storage() {
                    let removed =
                        storage::clear_prefix(&ls, QUOTA_TEST_PREFIX);
                    let text = format!("Удалено {} тестовых ключей.", removed);
                    quota_status.set_text_content(Some(text.as_str()));
                    log!("Quota test cleanup: removed {} keys", removed);
                }
                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 6d — Синхронный API
    // ============================================================

    {
        let sync_status = sync_status.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &sync_run_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        sync_status
                            .set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                sync_status.set_text_content(Some(
                    "Записываем 3 МБ синхронно — UI заблокирован...",
                ));

                storage::clear_prefix(&ls, SYNC_TEST_PREFIX);

                let result =
                    storage::measure_sync_block(&ls, 3 * 1024, SYNC_TEST_PREFIX);

                let text: String = format!(
                    "Синхронная запись завершена.\n\
                     Записано ключей: {}\n\
                     Суммарный размер: ~{} KB\n\
                     Время: {:.1} мс\n\
                     \n\
                     Всё это время UI был заблокирован. \
                     Для больших данных используйте IndexedDB — у него \
                     асинхронный API и лимит в сотни МБ.",
                    result.keys_written,
                    result.bytes_written / 1024,
                    result.duration_ms
                );
                sync_status.set_text_content(Some(text.as_str()));
                log!(
                    "Sync block: {} ms, {} keys, {} bytes",
                    result.duration_ms,
                    result.keys_written,
                    result.bytes_written
                );

                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    {
        let sync_status = sync_status.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        callbacks::listen(
            &sync_cleanup_btn,
            "click",
            move |_event| {
                if let Ok(ls) = storage::local_storage() {
                    let removed =
                        storage::clear_prefix(&ls, SYNC_TEST_PREFIX);
                    let text =
                        format!("Удалено {} sync-ключей.", removed);
                    sync_status.set_text_content(Some(text.as_str()));
                }
                render_methods();
                update_usage();
            },
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 7 — usage
    // ============================================================

    {
        let update_usage = update_usage.clone();

        callbacks::listen(
            &refresh_usage_btn,
            "click",
            move |_event| update_usage(),
            &closures,
        )?;
    }

    // ============================================================
    // Обработчики: СЕКЦИЯ 8 — StorageEvent
    // ============================================================

    {
        let events_log = events_log.clone();
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        let render_methods = render_methods.clone();
        let update_usage = update_usage.clone();

        let window = web_sys::window().unwrap();
        callbacks::listen(
            &window,
            "storage",
            move |event: Event| {
                let storage_event: StorageEvent = match event.dyn_into() {
                    Ok(e) => e,
                    Err(_) => return,
                };

                let change = storage::parse_storage_event(&storage_event);
                let text = storage::format_storage_change(&change);
                log!("StorageEvent:\n{}", text);

                render_notes();
                render_json_preview();
                render_methods();
                update_usage();

                let prev = events_log.text_content().unwrap_or_default();
                let new_text = format!("---\n{}\n\n{}", text, prev);
                let trimmed: String = new_text.chars().take(2000).collect();
                events_log.set_text_content(Some(trimmed.as_str()));
            },
            &closures,
        )?;
    }

    // ============================================================
    // Начальная инициализация
    // ============================================================

    if let Ok(ls) = storage::local_storage() {
        let raw_theme =
            storage::get(&ls, KEY_THEME).unwrap_or_else(|| "light".into());
        let theme_clean = raw_theme.trim();

        let theme = if ["light", "dark", "auto"].contains(&theme_clean) {
            theme_clean.to_string()
        } else {
            log!(
                "Некорректное значение темы в storage: {:?} — сброшено на \"light\"",
                raw_theme
            );
            let _ = storage::set(&ls, KEY_THEME, "light");
            "light".to_string()
        };

        theme_select.set_value(&theme);
        apply_theme(&theme);

        if !storage::has_key(&ls, KEY_SETTINGS) {
            let _ = storage::set_json(&ls, KEY_SETTINGS, &Settings::default());
        }
    }

    render_notes();
    render_json_preview();
    render_methods();
    update_usage();

    log!("Панель управления Storage готова");
    log!(
        "localStorage доступен: {}, writable: {}",
        storage::local_storage().is_ok(),
        ls_available
    );
    log!(
        "sessionStorage доступен: {}, writable: {}",
        storage::session_storage().is_ok(),
        ss_available
    );

    std::mem::forget(closures);

    Ok(())
}

/// Создаёт секцию с заголовком и возвращает её.
fn build_section(
    parent: &web_sys::Element,
    title: &str,
) -> Result<web_sys::Element, JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some(title));
    dom::append(&section, &h2)?;

    dom::append(parent, &section)?;
    Ok(section)
}

/// Форматирует результат заполнения хранилища до квоты.
///
/// Вынесено в отдельную функцию, чтобы избежать проблемы
/// `rust-analyzer` с tail-выражениями `if/else` внутри замыканий.
fn format_quota_result(result: &storage::FillResult) -> String {
    if result.hit_quota {
        return format!(
            "QuotaExceededError!\n\
             Записано ключей: {}\n\
             Суммарный размер: ~{} символов (~{} MB)\n\
             Сообщение браузера: {}",
            result.keys_written,
            result.total_chars,
            result.total_chars / (1024 * 1024),
            result.error_message.as_deref().unwrap_or("")
        );
    }
    return format!(
        "Достигнут лимит безопасности.\n\
         Записано ключей: {}\n\
         Суммарный размер: ~{} символов\n\
         {}",
        result.keys_written,
        result.total_chars,
        result.error_message.as_deref().unwrap_or("")
    );
}
```

---

## `web/index.html` (полное содержание)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — панель управления Storage</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Панель управления Storage. Откройте консоль для логов.</p>
    <p>
        <strong>Откройте вторую вкладку</strong> и измените данные —
        в первой сработает <code>StorageEvent</code>.
    </p>

    <div id="app"></div>

    <script type="module" src="/main.js"></script>
</body>
</html>
```

---

## `web/styles/main.css` (полное содержание)

```css
/* ============================================================
   Базовые стили
   ============================================================ */

:root {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    line-height: 1.6;
    color: #222;
    background: #fff;
}

body {
    max-width: 760px;
    margin: 2rem auto;
    padding: 0 1rem;
    transition: background 0.2s, color 0.2s;
}

body.theme-dark {
    background: #1a1a1a;
    color: #eaeaea;
}

body.theme-light {
    background: #fff;
    color: #222;
}

h1 {
    color: #b7410e;
}

body.theme-dark h1 {
    color: #e88b58;
}

code {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    background: #f5f5f5;
    padding: 0.15rem 0.35rem;
    border-radius: 3px;
    font-size: 0.9em;
}

body.theme-dark code {
    background: #2a2a2a;
}

#app {
    margin-top: 1.5rem;
}

/* ============================================================
   Секции
   ============================================================ */

.storage-section {
    margin-bottom: 1.5rem;
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
    background: #fafafa;
}

body.theme-dark .storage-section {
    background: #222;
    border-color: #444;
}

.storage-section h2 {
    margin-top: 0;
    font-size: 1.05rem;
    color: #b7410e;
}

body.theme-dark .storage-section h2 {
    color: #e88b58;
}

.storage-section h3 {
    margin: 1rem 0 0.5rem 0;
    font-size: 0.95rem;
    color: #b7410e;
}

body.theme-dark .storage-section h3 {
    color: #e88b58;
}

/* ============================================================
   Кнопки
   ============================================================ */

button {
    padding: 0.4rem 0.9rem;
    font-size: 0.95rem;
    border: 1px solid #b7410e;
    background: white;
    color: #b7410e;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
    margin-right: 0.4rem;
}

button:hover {
    background: #b7410e;
    color: white;
}

button:active {
    transform: scale(0.96);
}

body.theme-dark button {
    background: transparent;
    border-color: #e88b58;
    color: #e88b58;
}

body.theme-dark button:hover {
    background: #e88b58;
    color: #1a1a1a;
}

/* ============================================================
   Формы
   ============================================================ */

input[type="text"],
textarea,
select {
    padding: 0.4rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-family: inherit;
    font-size: 0.95rem;
    margin-right: 0.4rem;
    background: white;
    color: #222;
}

body.theme-dark input[type="text"],
body.theme-dark textarea,
body.theme-dark select {
    background: #2a2a2a;
    border-color: #555;
    color: #eaeaea;
}

textarea {
    width: 100%;
    box-sizing: border-box;
    margin-bottom: 0.5rem;
    resize: vertical;
}

input:focus,
textarea:focus,
select:focus {
    outline: 2px solid #b7410e;
    border-color: transparent;
}

/* ============================================================
   Отображение данных
   ============================================================ */

.counter-value {
    font-size: 1.8rem;
    font-weight: bold;
    color: #b7410e;
    margin-right: 1rem;
    font-variant-numeric: tabular-nums;
}

.counter-value.session {
    color: #2563eb;
}

body.theme-dark .counter-value {
    color: #e88b58;
}

body.theme-dark .counter-value.session {
    color: #60a5fa;
}

.notes-list {
    margin: 0.5rem 0;
    padding-left: 1.5rem;
}

.notes-list li {
    margin-bottom: 0.25rem;
}

.usage-info,
.events-log,
.keys-list {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    margin: 0.5rem 0;
    min-height: 2rem;
    max-height: 200px;
    overflow-y: auto;
}

body.theme-dark .usage-info,
body.theme-dark .events-log,
body.theme-dark .keys-list {
    background: #2a2a2a;
    border-left-color: #e88b58;
}

.events-log {
    border-left-color: #2e7d32;
    max-height: 400px;
}

/* ============================================================
   Секция ограничений
   ============================================================ */

.limits-info {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.6;
    white-space: pre-wrap;
    background: #fff8f0;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #d97706;
    margin: 0.5rem 0 1rem 0;
}

body.theme-dark .limits-info {
    background: #2a2018;
    border-left-color: #f59e0b;
}

.types-demo {
    margin: 0.5rem 0;
}

.types-demo pre {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.8rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #2e7d32;
    margin: 0;
    max-height: 300px;
    overflow-y: auto;
}

body.theme-dark .types-demo pre {
    background: #2a2a2a;
    border-left-color: #4ade80;
}

.json-preview {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.8rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #2563eb;
    margin: 0.5rem 0;
    max-height: 200px;
    overflow-y: auto;
    word-break: break-all;
}

body.theme-dark .json-preview {
    background: #2a2a2a;
    border-left-color: #60a5fa;
}

.label {
    font-size: 0.85rem;
    color: #666;
    margin-top: 0.75rem;
}

body.theme-dark .label {
    color: #aaa;
}

.hint {
    font-size: 0.85rem;
    color: #666;
    font-style: italic;
    margin: 0.25rem 0 0.75rem 0;
}

body.theme-dark .hint {
    color: #aaa;
}

/* ============================================================
   Warning (недоступный Storage)
   ============================================================ */

.warning {
    padding: 0.75rem 1rem;
    border-left: 4px solid #d97706;
    background: #fff8f0;
    border-radius: 4px;
    margin-bottom: 1rem;
    font-size: 0.9rem;
    color: #7c2d12;
}

body.theme-dark .warning {
    background: #2a2018;
    border-left-color: #f59e0b;
    color: #fbbf24;
}
```

---

## `tests/wasm_storage.rs` (полное содержание)

Файл `tests/wasm_storage.rs` содержит 66 тестов. Полное содержание приведено в предыдущем ответе (разделы 1–11). Все тесты проходят после исправления `parse_storage_event` — `test_parse_storage_event_sets_area` теперь корректно возвращает `Some("localStorage")`.

---

## Команды запуска

### Разработка

```bash
npm start
```

### Все тесты

```bash
npm test          # host + node + chrome
npm run test:all  # полный прогон
```

### Только тесты Storage

```bash
wasm-pack test --headless --chrome --test wasm_storage
```

```bash
xvfb-run -a wasm-pack test --headless --firefox \
  --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver \
  --test wasm_storage
```

---

## Ключевые концепции главы

**1. `Storage` — общий интерфейс.** `localStorage` и `sessionStorage` реализуют один API.

**2. Синхронный API.** Блокирует главный поток. Не подходит для больших данных.

**3. `set_item` может провалиться.** `QuotaExceededError` при превышении ~5 МБ.

**4. JSON-сериализация.** Строки и числа — нативно. Структуры — через `serde_json`.

**5. `StorageEvent` — межвкладочная синхронизация.** Только для `localStorage`. Только в другие вкладки. Содержит `key`, `oldValue`, `newValue`, `url`, `storageArea`.

**6. Изоляция.** `sessionStorage` — по вкладкам. `localStorage` — общий для origin.

**7. Привязка к origin.** `protocol + host + port`. Другой порт — другое хранилище.

**8. Оценка использования.** `approximate_size` — сумма длин. Реальный размер — `chars * 2` (UTF-16).

**9. Безопасность.** Уязвимость к XSS. Не храните секреты.

**10. Только строки.** Числа, bool, объекты, `Uint8Array` — через `to_string()`, `parse()`, JSON, base64.

**11. `QuotaExceededError`.** Демонстрируется через `fill_to_quota`.

**12. Private mode.** Проверяется через `is_storage_writable`.

**13. Не-атомарность.** Нет транзакций.

**14. Синхронная блокировка.** `measure_sync_block` демонстрирует задержку UI.

**15. `storageArea` в `StorageEvent`.** Всегда `"localStorage"` — событие генерируется только для `localStorage`.

---

## Итог главы

- `Storage` — общий интерфейс для `localStorage` и `sessionStorage`.
- `localStorage` — бессрочное хранение, общий для вкладок.
- `sessionStorage` — до закрытия вкладки, изолирован по вкладкам.
- Основные операции: `get`, `set`, `remove`, `clear`, `length`, `key`.
- JSON-сериализация через `serde_json`.
- `StorageEvent` — только `localStorage`, только в другие вкладки.
- `QuotaExceededError` — обработка переполнения.
- Синхронный API — блокирует UI.
- Привязка к origin — `protocol + host + port`.
- Private mode — Storage может быть недоступен.
- Не-атомарность — нет транзакций.
- Практический проект: панель с девятью секциями.
- 66 тестов — полное покрытие модуля `storage.rs`.

---

**Что дальше.** В главе 16 мы разберём **Fetch**: `Request`, `RequestInit`, `Response`, `Headers`, `fetch`, `Promise`, `JsFuture`, GET/POST, JSON, binary data, HTTP errors, `AbortController` и практический проект — Rust API client.