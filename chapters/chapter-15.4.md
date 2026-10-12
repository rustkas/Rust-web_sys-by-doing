# 15.4. Доступность Storage: private mode, исключения

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
│   ├── app.rs                      # +1 warning-блок при старте
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # +1 функция: is_storage_writable
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
│   └── wasm_storage.rs             # +3 теста → 25 тестов
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +1 правило: .warning
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/storage.rs` — **+1 функция**: `is_storage_writable`. Проверяет, что хранилище не только открывается, но и реально принимает записи.
- `src/app.rs` — **+1 warning-блок** в начале `run()`. Если storage недоступен для записи, показывается предупреждение.
- `tests/wasm_storage.rs` — **+3 теста** на проверку доступности и очистку тестового ключа. Всего 25 тестов.
- `web/styles/main.css` — **+1 правило** `.warning` для стилизации предупреждающего блока.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Storage quotas and eviction (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria |
| `Window.localStorage` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage |
| `QuotaExceededError` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria#when_is_there_a_quota_exceeded_error |
| Private Browsing (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage#exceptions |

---

## Когда Storage может быть недоступен

В обычной ситуации `localStorage` и `sessionStorage` работают всегда. Но есть три сценария, где доступ ограничен:

**1. Приватный режим (private browsing).** В некоторых браузерах (Safari, старые версии Firefox, режим инкогнито в Chrome с определёнными политиками) `localStorage` **открывается**, но `set_item` бросает `QuotaExceededError`. Хранилище есть, но оно «только для чтения» с нулевой квотой.

**2. Политики безопасности.** Корпоративные браузеры могут блокировать Storage на определённых сайтах. Расширения вроде uBlock Origin могут отключать Storage на нежелательных доменах.

**3. `file://` — локальное открытие.** При открытии HTML-файла напрямую из файловой системы (`file:///path/to/index.html`) некоторые браузеры ограничивают Storage. Это касается нашего проекта при попытке открыть `index.html` без dev-сервера.

**Важно понять.** Проблема не в том, что `local_storage()` вернёт `Err` — в большинстве случаев она вернёт `Ok` с объектом `Storage`. Проблема в том, что **запись** в этот объект провалится с `QuotaExceededError`. Наш код должен это проверить.

---

## `is_storage_writable`

Проверка `is_storage_writable` работает по принципу **«запиши и прочитай»**. Мы записываем тестовый ключ, читаем его обратно, сравниваем, удаляем. Если всё получилось — хранилище рабочее.

```rust
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
```

**Что здесь важно:**

- **`set_item` возвращает `Result`.** Если `Err` — хранилище не пишется. Возвращаем `false`.
- **Проверяем не только запись, но и чтение.** В некоторых режимах запись проходит «в никуда», а чтение возвращает `None`. Сравнение значений ловит этот случай.
- **Тестовый ключ удаляется.** Функция не оставляет мусора. Имя ключа начинается с `__test_` — стандартное соглашение для временных ключей.

**Почему не проверять через `length` или `keys`.** Они возвращают «сколько уже есть». Если хранилище пустое, `length == 0` — но это не значит, что оно недоступно. Запись в него могла бы провалиться. Только реальная запись даёт ответ.

**Почему не использовать `try/catch` в JS.** В Rust `set_item` уже возвращает `Result`. Никаких `try/catch` — только pattern matching на `Ok`/`Err`.

---

## `src/storage.rs`

```rust
//! Помощники для работы с Web Storage API.
//!
//! ## Что в этом модуле
//!
//! - `local_storage` / `session_storage` — доступ к хранилищам.
//! - `get` / `set` / `remove` — базовые операции.
//! - `length` / `key` / `keys` / `entries` — итерация по хранилищу.
//! - `has_key` — проверка наличия ключа.
//! - `clear` — полная очистка.
//! - `is_storage_writable` — проверка, что хранилище принимает записи.
//!
//! ## Что будет дальше
//!
//! В следующих подглавах модуль расширится:
//! - 15.6 — `set_json`, `get_json`.
//! - 15.7 — `parse_storage_event`, `format_storage_change`.
//! - 15.8 — `fill_to_quota`, `measure_sync_block`, `current_origin`.
#![allow(dead_code)]

use wasm_bindgen::JsValue;
use web_sys::Storage;

// ============================================================
// Получение доступа к хранилищам
// ============================================================

/// Возвращает `window.localStorage`.
///
/// # Ошибки
///
/// Возвращает `Err`, если окно недоступно или браузер запретил
/// доступ к `localStorage` полностью.
///
/// **Важно.** Успешный `Ok` не гарантирует, что в хранилище
/// **можно писать**. В приватном режиме некоторых браузеров
/// `localStorage` открывается, но `set_item` бросает
/// `QuotaExceededError`. Для проверки записи используйте
/// `is_storage_writable`.
pub fn local_storage() -> Result<Storage, JsValue> {
    return web_sys::window()
        .expect("no window")
        .local_storage()?
        .ok_or_else(|| JsValue::from_str("localStorage is not available"));
}

/// Возвращает `window.sessionStorage`.
///
/// # Ошибки
///
/// Возвращает `Err`, если окно недоступно или браузер запретил
/// доступ к `sessionStorage` полностью.
pub fn session_storage() -> Result<Storage, JsValue> {
    return web_sys::window()
        .expect("no window")
        .session_storage()?
        .ok_or_else(|| JsValue::from_str("sessionStorage is not available"));
}

// ============================================================
// Проверка доступности записи
// ============================================================

/// Проверяет, что хранилище принимает записи.
///
/// Работает по принципу «запиши и прочитай»:
/// 1. Записывает тестовый ключ `__test_writable__`.
/// 2. Читает его обратно.
/// 3. Сравнивает с исходным значением.
/// 4. Удаляет тестовый ключ.
///
/// Возвращает `true`, если все четыре шага прошли успешно.
///
/// # Когда это нужно
///
/// В приватном режиме некоторых браузеров `localStorage`
/// открывается, но **запись** в него проваливается с
/// `QuotaExceededError`. Простая проверка `local_storage().is_ok()`
/// этого не поймает. `is_storage_writable` — ловит.
///
/// # Пример
///
/// ```no_run
/// use hello_wasm02::storage;
/// if let Ok(ls) = storage::local_storage() {
///     if storage::is_storage_writable(&ls) {
///         // можно безопасно писать
///     }
/// }
/// ```
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

// ============================================================
// Базовые операции
// ============================================================

/// Читает значение по ключу.
pub fn get(storage: &Storage, key: &str) -> Option<String> {
    return storage.get_item(key).ok().flatten();
}

/// Устанавливает значение.
///
/// # Ошибки
///
/// Возвращает `Err`, если хранилище переполнено (`QuotaExceededError`)
/// или доступ запрещён.
pub fn set(storage: &Storage, key: &str, value: &str) -> Result<(), JsValue> {
    return storage.set_item(key, value);
}

/// Удаляет ключ.
pub fn remove(storage: &Storage, key: &str) -> Result<(), JsValue> {
    return storage.remove_item(key);
}

// ============================================================
// Итерация по хранилищу
// ============================================================

/// Возвращает количество ключей в хранилище.
pub fn length(storage: &Storage) -> u32 {
    return storage.length().unwrap_or(0);
}

/// Возвращает ключ по индексу.
pub fn key(storage: &Storage, index: u32) -> Option<String> {
    return storage.key(index).ok().flatten();
}

/// Возвращает все ключи хранилища.
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
// Проверка наличия
// ============================================================

/// Проверяет наличие ключа.
pub fn has_key(storage: &Storage, key: &str) -> bool {
    return get(storage, key).is_some();
}

// ============================================================
// Полная очистка
// ============================================================

/// Очищает хранилище.
pub fn clear(storage: &Storage) -> Result<(), JsValue> {
    return storage.clear();
}
```

**Ключевые моменты:**

- **`is_storage_writable` — единственная новая функция.** Остальные 12 — из предыдущих подглав.
- **Возвращает `bool`.** Нет `Result`, нет `Option`. Вопрос простой: пишется или нет.
- **Не оставляет мусора.** Тестовый ключ удаляется в любом случае (даже если чтение провалилось).
- **Проверка чтения после записи.** В некоторых браузерах запись «проходит», но данные не сохраняются. Чтение это ловит.

---

## `src/app.rs`

Показаны только фрагменты, которые меняются. Остальной код из 15.3 без изменений.

### Новый warning-блок в начале `run()`

```rust
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Storage: базовый API")?;
    dom::append(&container, &heading)?;

    // ============================================================
    // Проверка доступности Storage
    // ============================================================
    //
    // В приватном режиме некоторых браузеров localStorage
    // открывается, но запись в него проваливается с
    // QuotaExceededError. Простая проверка `is_ok()` этого
    // не ловит. Проверяем реальную запись через
    // `is_storage_writable`.

    let ls_available = storage::local_storage()
        .map(|s| storage::is_storage_writable(&s))
        .unwrap_or(false);
    let ss_available = storage::session_storage()
        .map(|s| storage::is_storage_writable(&s))
        .unwrap_or(false);

    if !ls_available || !ss_available {
        let warning = dom::create_element("div")?;
        dom::add_class(&dom::as_html(&warning), "warning")?;

        let warning_text: String = format!(
            "⚠ Хранилище ограничено.\n\
             localStorage: {}\n\
             sessionStorage: {}\n\
             \n\
             Возможные причины: приватный режим браузера, \
             политики безопасности, ограничения расширений. \
             Сохранение данных будет недоступно.",
            if ls_available { "доступен для записи" } else { "НЕ доступен для записи" },
            if ss_available { "доступен для записи" } else { "НЕ доступен для записи" },
        );
        warning.set_text_content(Some(warning_text.as_str()));
        dom::append(&container, &warning)?;

        log!(
            "Storage ограничен: localStorage writable = {}, sessionStorage writable = {}",
            ls_available,
            ss_available
        );
    } else {
        log!("Storage полностью доступен: оба хранилища принимают записи");
    }

    // ... остальной код без изменений ...
}
```

**Что демонстрирует warning-блок:**

- **Проверка перед всем остальным.** Warning появляется до секций. Если storage недоступен, пользователь видит предупреждение сразу.
- **`map + unwrap_or(false)`.** Если `local_storage()` вернул `Err`, сразу `false`. Если `Ok` — проверяем `is_storage_writable`.
- **Обе проверки независимы.** `localStorage` может быть writable, а `sessionStorage` — нет (и наоборот). Показываем статус обоих.
- **Warning не блокирует приложение.** Кнопки работают, но записи будут падать. Warning сообщает пользователю, почему сохранение не работает.

**Что важно понять:**

- **Доступность ≠ работоспособность.** `local_storage()` возвращает `Ok`, но `set_item` может провалиться. `is_storage_writable` делает реальную проверку.
- **Warning не ошибка.** Приложение продолжает работать. Просто данные не сохраняются.
- **Логирование для отладки.** `log!` показывает статус в консоли — полезно при проверке в приватном режиме.

### Полное содержание `src/app.rs`

```rust
//! Логика приложения: «Storage: знакомство + инспектор + два счётчика».
//!
//! Три секции + warning-блок при старте:
//! - Warning: если Storage недоступен для записи.
//! - Секция 1: сохранение имени.
//! - Секция 2: инспектор хранилища.
//! - Секция 3: два счётчика LS vs SS.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::HtmlInputElement;

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;
use crate::storage;

const KEY_NAME: &str = "demo:name";
const KEY_LS_COUNTER: &str = "demo:ls_counter";
const KEY_SS_COUNTER: &str = "demo:ss_counter";

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Storage: базовый API")?;
    dom::append(&container, &heading)?;

    // ============================================================
    // Проверка доступности Storage
    // ============================================================

    let ls_available = storage::local_storage()
        .map(|s| storage::is_storage_writable(&s))
        .unwrap_or(false);
    let ss_available = storage::session_storage()
        .map(|s| storage::is_storage_writable(&s))
        .unwrap_or(false);

    if !ls_available || !ss_available {
        let warning = dom::create_element("div")?;
        dom::add_class(&dom::as_html(&warning), "warning")?;

        let warning_text: String = format!(
            "⚠ Хранилище ограничено.\n\
             localStorage: {}\n\
             sessionStorage: {}\n\
             \n\
             Возможные причины: приватный режим браузера, \
             политики безопасности, ограничения расширений. \
             Сохранение данных будет недоступно.",
            if ls_available { "доступен для записи" } else { "НЕ доступен для записи" },
            if ss_available { "доступен для записи" } else { "НЕ доступен для записи" },
        );
        warning.set_text_content(Some(warning_text.as_str()));
        dom::append(&container, &warning)?;

        log!(
            "Storage ограничен: ls writable = {}, ss writable = {}",
            ls_available,
            ss_available
        );
    } else {
        log!("Storage полностью доступен");
    }

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ============================================================
    // СЕКЦИЯ 1: Знакомство
    // ============================================================

    let section1 = build_section(
        &container,
        "1. Знакомство: сохранение имени",
    )?;

    let description = dom::create_paragraph(
        "Введите имя, сохраните его в localStorage, перезагрузите страницу \
         и загрузите обратно. Данные переживут перезагрузку.",
    )?;
    dom::add_class(&dom::as_html(&description), "hint")?;
    dom::append(&section1, &description)?;

    let input_el = dom::create_element("input")?;
    dom::set_attribute(&input_el, "type", "text")?;
    dom::set_attribute(&input_el, "id", "name-input")?;
    dom::set_attribute(&input_el, "placeholder", "Введите имя...")?;
    let input: HtmlInputElement = input_el.dyn_into()?;
    dom::append(&section1, &input)?;

    let save_btn = dom::create_element("button")?;
    save_btn.set_text_content(Some("Сохранить"));
    let load_btn = dom::create_element("button")?;
    load_btn.set_text_content(Some("Загрузить"));
    let remove_btn = dom::create_element("button")?;
    remove_btn.set_text_content(Some("Удалить"));

    dom::append(&section1, &save_btn)?;
    dom::append(&section1, &load_btn)?;
    dom::append(&section1, &remove_btn)?;

    let result1 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result1), "result")?;
    result1.set_text_content(Some("(ещё ничего не сделано)"));
    dom::append(&section1, &result1)?;

    // ---- Обработчик «Сохранить» ----
    {
        let input = input.clone();
        let result1 = result1.clone();
        callbacks::listen(
            &save_btn,
            "click",
            move |_event| {
                let value = input.value();

                if value.trim().is_empty() {
                    result1.set_text_content(Some("Введите имя перед сохранением"));
                    log!("Пустое значение — сохранение пропущено");
                    return;
                }

                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        result1.set_text_content(Some("localStorage недоступен"));
                        log!("localStorage недоступен");
                        return;
                    }
                };

                match storage::set(&ls, KEY_NAME, &value) {
                    Ok(_) => {
                        let text = format!("Сохранено: \"{}\"", value);
                        result1.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                    Err(e) => {
                        let text = format!(
                            "Ошибка сохранения: {:?}. \
                             Возможно, Storage ограничен (приватный режим?).",
                            e
                        );
                        result1.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                }
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Загрузить» ----
    {
        let result1 = result1.clone();
        callbacks::listen(
            &load_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        result1.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                match storage::get(&ls, KEY_NAME) {
                    Some(value) => {
                        let text = format!("Загружено: \"{}\"", value);
                        result1.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                    None => {
                        result1.set_text_content(Some(
                            "Ключ не найден. Сначала сохраните имя.",
                        ));
                        log!("Ключ {} не найден", KEY_NAME);
                    }
                }
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Удалить» ----
    {
        let result1 = result1.clone();
        callbacks::listen(
            &remove_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        result1.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                match storage::remove(&ls, KEY_NAME) {
                    Ok(_) => {
                        result1.set_text_content(Some("Ключ удалён из localStorage"));
                        log!("Ключ {} удалён", KEY_NAME);
                    }
                    Err(e) => {
                        let text = format!("Ошибка удаления: {:?}", e);
                        result1.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                }
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 2: Инспектор хранилища
    // ============================================================

    let section2 = build_section(
        &container,
        "2. Инспектор: все ключи, длина, очистка",
    )?;

    let inspector_hint = dom::create_paragraph(
        "Полный API Storage: length(), key(i), keys(), entries(), has_key(), clear().",
    )?;
    dom::add_class(&dom::as_html(&inspector_hint), "hint")?;
    dom::append(&section2, &inspector_hint)?;

    let refresh_btn = dom::create_element("button")?;
    refresh_btn.set_text_content(Some("Обновить"));
    let clear_all_btn = dom::create_element("button")?;
    clear_all_btn.set_text_content(Some("Очистить всё"));
    dom::append(&section2, &refresh_btn)?;
    dom::append(&section2, &clear_all_btn)?;

    let inspector_result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&inspector_result), "inspector")?;
    dom::append(&section2, &inspector_result)?;

    let render_inspector: Rc<dyn Fn()> =
        make_render_inspector(inspector_result.clone());

    {
        let render_inspector = render_inspector.clone();
        callbacks::listen(
            &refresh_btn,
            "click",
            move |_event| {
                render_inspector();
            },
            &closures,
        )?;
    }

    {
        let render_inspector = render_inspector.clone();
        callbacks::listen(
            &clear_all_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                match storage::clear(&ls) {
                    Ok(_) => {
                        log!("localStorage очищен");
                        render_inspector();
                    }
                    Err(e) => log!("Ошибка очистки: {:?}", e),
                }
            },
            &closures,
        )?;
    }

    render_inspector();

    // ============================================================
    // СЕКЦИЯ 3: localStorage vs sessionStorage
    // ============================================================

    let section3 = build_section(
        &container,
        "3. localStorage vs sessionStorage: два счётчика",
    )?;

    let diff_hint = dom::create_paragraph(
        "localStorage — общий для всех вкладок, переживает закрытие браузера. \
         sessionStorage — изолирован по вкладкам, данные исчезают при закрытии. \
         Перезагрузите страницу (F5), откройте вторую вкладку, закройте вкладку — \
         проверьте разницу.",
    )?;
    dom::add_class(&dom::as_html(&diff_hint), "hint")?;
    dom::append(&section3, &diff_hint)?;

    // ---- localStorage счётчик ----
    let ls_counter_label = dom::create_element("div")?;
    ls_counter_label.set_text_content(Some("localStorage счётчик:"));
    dom::append(&section3, &ls_counter_label)?;

    let ls_counter_display = dom::create_element("span")?;
    dom::add_class(&dom::as_html(&ls_counter_display), "counter-value")?;

    let ls_counter_initial: i32 = match storage::local_storage() {
        Ok(ls) => storage::get(&ls, KEY_LS_COUNTER)
            .and_then(|s| s.parse::<i32>().ok())
            .unwrap_or(0),
        Err(_) => 0,
    };
    let ls_counter = Rc::new(RefCell::new(ls_counter_initial));
    {
        let initial_text: String = ls_counter_initial.to_string();
        ls_counter_display.set_text_content(Some(initial_text.as_str()));
    }

    let ls_inc_btn = dom::create_element("button")?;
    ls_inc_btn.set_text_content(Some("+1"));
    let ls_dec_btn = dom::create_element("button")?;
    ls_dec_btn.set_text_content(Some("−1"));
    let ls_reset_btn = dom::create_element("button")?;
    ls_reset_btn.set_text_content(Some("Сброс"));

    dom::append(&section3, &ls_counter_display)?;
    dom::append(&section3, &ls_inc_btn)?;
    dom::append(&section3, &ls_dec_btn)?;
    dom::append(&section3, &ls_reset_btn)?;

    // ---- sessionStorage счётчик ----
    let ss_counter_label = dom::create_element("div")?;
    ss_counter_label.set_text_content(Some("sessionStorage счётчик:"));
    dom::append(&section3, &ss_counter_label)?;

    let ss_counter_display = dom::create_element("span")?;
    dom::add_class(&dom::as_html(&ss_counter_display), "counter-value session")?;

    let ss_counter_initial: i32 = match storage::session_storage() {
        Ok(ss) => storage::get(&ss, KEY_SS_COUNTER)
            .and_then(|s| s.parse::<i32>().ok())
            .unwrap_or(0),
        Err(_) => 0,
    };
    let ss_counter = Rc::new(RefCell::new(ss_counter_initial));
    {
        let initial_text: String = ss_counter_initial.to_string();
        ss_counter_display.set_text_content(Some(initial_text.as_str()));
    }

    let ss_inc_btn = dom::create_element("button")?;
    ss_inc_btn.set_text_content(Some("+1"));
    let ss_dec_btn = dom::create_element("button")?;
    ss_dec_btn.set_text_content(Some("−1"));
    let ss_reset_btn = dom::create_element("button")?;
    ss_reset_btn.set_text_content(Some("Сброс"));

    dom::append(&section3, &ss_counter_display)?;
    dom::append(&section3, &ss_inc_btn)?;
    dom::append(&section3, &ss_dec_btn)?;
    dom::append(&section3, &ss_reset_btn)?;

    // ---- Обработчики LS ----
    {
        let counter = ls_counter.clone();
        let display = ls_counter_display.clone();
        callbacks::listen(
            &ls_inc_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() += 1;
                if let Ok(ls) = storage::local_storage() {
                    let value = counter.borrow().to_string();
                    let _ = storage::set(&ls, KEY_LS_COUNTER, &value);
                }
                let text = counter.borrow().to_string();
                display.set_text_content(Some(text.as_str()));
                log!("LS counter: {}", text);
            },
            &closures,
        )?;
    }
    {
        let counter = ls_counter.clone();
        let display = ls_counter_display.clone();
        callbacks::listen(
            &ls_dec_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() -= 1;
                if let Ok(ls) = storage::local_storage() {
                    let value = counter.borrow().to_string();
                    let _ = storage::set(&ls, KEY_LS_COUNTER, &value);
                }
                let text = counter.borrow().to_string();
                display.set_text_content(Some(text.as_str()));
                log!("LS counter: {}", text);
            },
            &closures,
        )?;
    }
    {
        let counter = ls_counter.clone();
        let display = ls_counter_display.clone();
        callbacks::listen(
            &ls_reset_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() = 0;
                if let Ok(ls) = storage::local_storage() {
                    let _ = storage::remove(&ls, KEY_LS_COUNTER);
                }
                display.set_text_content(Some("0"));
                log!("LS counter сброшен");
            },
            &closures,
        )?;
    }

    // ---- Обработчики SS ----
    {
        let counter = ss_counter.clone();
        let display = ss_counter_display.clone();
        callbacks::listen(
            &ss_inc_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() += 1;
                if let Ok(ss) = storage::session_storage() {
                    let value = counter.borrow().to_string();
                    let _ = storage::set(&ss, KEY_SS_COUNTER, &value);
                }
                let text = counter.borrow().to_string();
                display.set_text_content(Some(text.as_str()));
                log!("SS counter: {}", text);
            },
            &closures,
        )?;
    }
    {
        let counter = ss_counter.clone();
        let display = ss_counter_display.clone();
        callbacks::listen(
            &ss_dec_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() -= 1;
                if let Ok(ss) = storage::session_storage() {
                    let value = counter.borrow().to_string();
                    let _ = storage::set(&ss, KEY_SS_COUNTER, &value);
                }
                let text = counter.borrow().to_string();
                display.set_text_content(Some(text.as_str()));
                log!("SS counter: {}", text);
            },
            &closures,
        )?;
    }
    {
        let counter = ss_counter.clone();
        let display = ss_counter_display.clone();
        callbacks::listen(
            &ss_reset_btn,
            "click",
            move |_event| {
                *counter.borrow_mut() = 0;
                if let Ok(ss) = storage::session_storage() {
                    let _ = storage::remove(&ss, KEY_SS_COUNTER);
                }
                display.set_text_content(Some("0"));
                log!("SS counter сброшен");
            },
            &closures,
        )?;
    }

    std::mem::forget(closures);

    Ok(())
}

// ============================================================
// Вспомогательные функции
// ============================================================

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

fn make_render_inspector(inspector_result: web_sys::Element) -> Rc<dyn Fn()> {
    return Rc::new(move || {
        inspector_result.set_inner_html("");

        let ls = match storage::local_storage() {
            Ok(s) => s,
            Err(_) => {
                inspector_result.set_text_content(Some("localStorage недоступен"));
                return;
            }
        };

        let len = storage::length(&ls);

        let summary = dom::create_element("div").unwrap();
        let summary_text: String = format!("Всего ключей: {}", len);
        summary.set_text_content(Some(summary_text.as_str()));
        let _ = dom::append(&inspector_result, &summary);

        if len == 0 {
            let empty = dom::create_element("p").unwrap();
            empty.set_text_content(Some("(хранилище пусто)"));
            let _ = dom::append(&inspector_result, &empty);
            return;
        }

        let table = dom::create_element("table").unwrap();

        let header = dom::create_element("tr").unwrap();
        for name in ["#", "key", "value", "has_key"] {
            let th = dom::create_element("th").unwrap();
            th.set_text_content(Some(name));
            let _ = dom::append(&header, &th);
        }
        let _ = dom::append(&table, &header);

        for (i, k) in storage::keys(&ls).iter().enumerate() {
            let value = storage::get(&ls, k).unwrap_or_else(|| "(none)".into());
            let has = storage::has_key(&ls, k);

            let row = dom::create_element("tr").unwrap();

            let idx_cell = dom::create_element("td").unwrap();
            idx_cell.set_text_content(Some(&i.to_string()));
            let _ = dom::append(&row, &idx_cell);

            let key_cell = dom::create_element("td").unwrap();
            key_cell.set_text_content(Some(k));
            let _ = dom::append(&row, &key_cell);

            let value_cell = dom::create_element("td").unwrap();
            let value_preview: String = preview_value(&value, 40);
            value_cell.set_text_content(Some(value_preview.as_str()));
            let _ = dom::append(&row, &value_cell);

            let has_cell = dom::create_element("td").unwrap();
            has_cell.set_text_content(Some(if has { "✓" } else { "✗" }));
            let _ = dom::append(&row, &has_cell);

            let _ = dom::append(&table, &row);
        }

        let _ = dom::append(&inspector_result, &table);

        let size = storage::entries(&ls)
            .iter()
            .map(|(k, v)| k.len() + v.len())
            .sum::<usize>();
        let size_line = dom::create_element("div").unwrap();
        let size_text: String = format!("Суммарный размер: ~{} символов", size);
        size_line.set_text_content(Some(size_text.as_str()));
        let _ = dom::append(&inspector_result, &size_line);

        log!("Инспектор: {} ключей, ~{} символов", len, size);
    });
}

fn preview_value(value: &str, max_chars: usize) -> String {
    let char_count = value.chars().count();
    if char_count <= max_chars {
        return value.to_string();
    }
    let truncated: String = value.chars().take(max_chars).collect();
    return format!("{}...", truncated);
}
```

**Что изменилось относительно 15.3:**

- **Добавлен warning-блок** в начале `run()` — проверка доступности обоих хранилищ.
- **В обработчике «Сохранить»** сообщение об ошибке теперь содержит подсказку про приватный режим.
- **Остальное без изменений.**

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   Warning-блок
   ============================================================ */

.warning {
    padding: 0.75rem 1rem;
    border-left: 4px solid #d97706;
    background: #fff8f0;
    border-radius: 4px;
    margin-bottom: 1rem;
    font-size: 0.9rem;
    color: #7c2d12;
    white-space: pre-wrap;
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
}
```

**Что делает:**

- **Оранжевая полоса слева** — стандартный визуальный сигнал «внимание, но не ошибка».
- **Светло-оранжевый фон** — отличает от обычных блоков.
- **`white-space: pre-wrap`** — сохраняет переводы строк в тексте предупреждения.
- **Моноширинный шрифт** — согласуется с другими информационными блоками.

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.3):

```rust
// ============================================================
// Доступность Storage
// ============================================================

#[wasm_bindgen_test]
fn test_local_storage_is_writable() {
    let ls = storage::local_storage().unwrap();
    assert!(
        storage::is_storage_writable(&ls),
        "localStorage should be writable in test environment"
    );
}

#[wasm_bindgen_test]
fn test_session_storage_is_writable() {
    let ss = storage::session_storage().unwrap();
    assert!(
        storage::is_storage_writable(&ss),
        "sessionStorage should be writable in test environment"
    );
}

#[wasm_bindgen_test]
fn test_is_storage_writable_does_not_leave_residue() {
    let ls = storage::local_storage().unwrap();

    // Вызываем проверку
    let _ = storage::is_storage_writable(&ls);

    // Тестовый ключ должен быть удалён после проверки
    assert!(
        !storage::has_key(&ls, "__test_writable__"),
        "is_storage_writable should clean up after itself"
    );
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_local_storage_is_writable` | `localStorage` принимает записи в тестовом окружении |
| `test_session_storage_is_writable` | `sessionStorage` принимает записи в тестовом окружении |
| `test_is_storage_writable_does_not_leave_residue` | Функция не оставляет после себя тестовый ключ |

**Почему только три теста.**

- **Нельзя симулировать «недоступный storage» в `wasm-bindgen-test`.** Тесты запускаются в нормальном браузере, где Storage всегда доступен. Мы не можем переключить браузер в приватный режим во время теста.
- **Можно проверить только положительный случай.** `is_storage_writable` возвращает `true` — значит, функция работает и не паникует.
- **Проверка очистки — ключевой тест.** Убеждаемся, что функция не оставляет мусора. Это единственное поведение, которое можно проверить без симуляции.

**Проверка отрицательного случая — вручную.** Откройте приложение в приватном режиме Safari или Firefox — увидите warning. В Chrome инкогнито Storage обычно работает, так что warning не появится.

**Всего тестов:** 22 из 15.1–15.3 + 3 новых = **25 тестов**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. В нормальном режиме:

- Warning не появляется.
- Все три секции работают.
- В консоли: `Storage полностью доступен`.

**Проверка warning в приватном режиме.**

- **Safari**: File → New Private Window, откройте приложение. Warning появится.
- **Firefox**: File → New Private Window, откройте приложение. В свежих версиях Storage работает — warning не появится.
- **Chrome Incognito**: Storage работает, warning не появится.

**Проверка через DevTools.** Можно искусственно сломать Storage, переопределив `setItem` до загрузки приложения. В DevTools → Sources → Snippets создайте сниппет:

```javascript
Storage.prototype.setItem = function() {
    throw new DOMException("QuotaExceededError", "QuotaExceededError");
};
```

Запустите его **до** загрузки страницы (через `Page.addScriptToEvaluateOnNewDocument` в Chrome или через консоль с последующей перезагрузкой). После этого `is_storage_writable` вернёт `false`, warning появится.

Это продвинутая проверка — в учебном примере достаточно понимать, как warning устроен.

### Тесты

```bash
# Только тесты Storage в Chrome
wasm-pack test --headless --chrome --test wasm_storage
```

```bash
# Только тесты Storage в Firefox
xvfb-run -a wasm-pack test --headless --firefox \
  --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver \
  --test wasm_storage
```

```bash
# Все тесты проекта
npm run test:all
```

**Ожидаемый результат:** 25 тестов `ok` в обоих браузерах.

---

## Что важно запомнить

**1. `local_storage()` может вернуть `Ok`, но запись всё равно провалится.** В приватном режиме некоторых браузеров Storage открывается, но `set_item` бросает `QuotaExceededError`. Проверка `is_ok()` это не ловит.

**2. `is_storage_writable` — реальная проверка.** Записать, прочитать, сравнить, удалить. Единственный надёжный способ узнать, работает ли Storage.

**3. Тестовый ключ не остаётся.** Функция сама удаляет `__test_writable__` после проверки.

**4. Warning не блокирует приложение.** Секции продолжают работать. Просто записи будут падать. Warning сообщает пользователю причину.

**5. Приватный режим — не единственная причина.** Политики безопасности, расширения, `file://` — всё может ограничить Storage.

**6. Проверить warning можно только вручную.** `wasm-bindgen-test` запускается в обычном браузере. Симулировать приватный режим внутри теста нельзя.

**7. Проверка доступности — часть инициализации.** Мы делаем её **до** построения секций. Пользователь видит warning раньше, чем начнёт взаимодействовать с UI.

---

## Что дальше

**15.5. Сериализация: строки, числа, булевы.** Мы разберём ограничение №1 Web Storage: хранилище принимает **только строки**. Числа, булевы значения, объекты, массивы — всё нужно кодировать. Начнём с простого: как сохранить число `42` и прочитать его обратно как `i32`. Добавим в `app.rs` секцию «Типы» с демонстрацией. `storage.rs` не меняется — это работа на уровне вызывающего кода. В тестах — 6 новых проверок на числа, булевы, пустые значения, округление float.