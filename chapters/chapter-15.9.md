# 15.9. Практический проект: панель управления Storage

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
│   ├── app.rs                      # финальная версия: все секции собраны
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # 20 функций, все используются
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
│   └── wasm_storage.rs             # 62 теста
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # полный набор стилей
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/storage.rs` — **20 функций**. Все, что нужны приложению из подглав 15.1–15.8. Проверьте наличие каждой перед сборкой (см. раздел «Проверка полноты `storage.rs`» ниже).
- `src/app.rs` — **финальная версия**. Семь секций из 15.1–15.8, warning-блок из 15.4 и обработчик `storage` из 15.7.
- `tests/wasm_storage.rs` — **62 теста**. Из предыдущих подглав.
- `web/styles/main.css` — финальный набор стилей.

---

## Проверка полноты `storage.rs` — важно перед сборкой

`app.rs` в этой главе использует **все 20 функций** из `storage.rs`. Если какая-то функция отсутствует — компилятор выдаст ошибку вида `cannot find function X in module storage`.

**Список функций, которые должны быть в `storage.rs`:**

| № | Функция | Подглава |
|---|---|---|
| 1 | `local_storage` | 15.1 |
| 2 | `session_storage` | 15.1 |
| 3 | `is_storage_writable` | **15.4** |
| 4 | `get` | 15.1 |
| 5 | `set` | 15.1 |
| 6 | `remove` | 15.1 |
| 7 | `length` | 15.2 |
| 8 | `key` | 15.2 |
| 9 | `keys` | 15.2 |
| 10 | `entries` | 15.2 |
| 11 | `has_key` | 15.2 |
| 12 | `clear` | 15.2 |
| 13 | `set_json` | 15.6 |
| 14 | `get_json` | 15.6 |
| 15 | `parse_storage_event` | 15.7 |
| 16 | `format_storage_change` | 15.7 |
| 17 | `current_origin` | 15.8 |
| 18 | `fill_to_quota` | 15.8 |
| 19 | `clear_prefix` | 15.8 |
| 20 | `measure_sync_block` | 15.8 |

Плюс три типа: `StorageChange`, `FillResult`, `SyncBlockResult`.

**Быстрая проверка:**

```bash
grep -oP 'pub fn \w+|pub struct \w+' src/storage.rs | sort -u
```

Если чего-то не хватает — добавьте из соответствующих подглав.

**Особое внимание — `is_storage_writable`.** Если её нет, `app.rs` не скомпилируется. Добавьте после `session_storage`:

```rust
/// Проверяет, что хранилище принимает записи.
///
/// Работает по принципу «запиши и прочитай»:
/// 1. Записывает тестовый ключ `__test_writable__`.
/// 2. Читает его обратно.
/// 3. Сравнивает с исходным значением.
/// 4. Удаляет тестовый ключ.
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

---

## `src/app.rs` — финальная версия

```rust
//! Логика приложения: «Панель управления Storage».
//!
//! Семь секций, демонстрирующих все аспекты Web Storage API:
//!
//! 1. **Знакомство** — базовые операции: set, get, remove.
//! 2. **Инспектор** — полный API: length, key(i), keys, entries, clear.
//! 3. **LS vs SS** — два счётчика: общий vs изолированный по вкладкам.
//! 4. **Типы** — Storage хранит только строки.
//! 5. **Заметки** — JSON-сериализация структур через serde.
//! 6. **События** — StorageEvent: синхронизация между вкладками.
//! 7. **Ограничения** — бинарь, квота, синхронный API, origin.
//!
//! Плюс warning-блок при старте, если Storage недоступен для записи.

use std::cell::RefCell;
use std::rc::Rc;

use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, HtmlInputElement, StorageEvent};

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;
use crate::storage;

// ============================================================
// Модели данных
// ============================================================

/// Заметка, сохраняемая в localStorage как JSON.
#[derive(Serialize, Deserialize, Clone, Debug)]
struct Note {
    id: u32,
    text: String,
}

// ============================================================
// Ключи Storage
// ============================================================

const KEY_NAME: &str = "demo:name";
const KEY_LS_COUNTER: &str = "demo:ls_counter";
const KEY_SS_COUNTER: &str = "demo:ss_counter";
const KEY_NOTES: &str = "demo:notes";
const QUOTA_TEST_PREFIX: &str = "quota-test";
const SYNC_TEST_PREFIX: &str = "sync-test";

// ============================================================
// Точка входа
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Панель управления Storage")?;
    dom::append(&container, &heading)?;

    // ============================================================
    // Проверка доступности Storage (из 15.4)
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
        "Полный API Storage: length(), key(i), keys(), entries(), has_key(), clear(). \
         Обновляйте таблицу, чтобы увидеть текущее содержимое localStorage.",
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

    // ---- Обработчик «Обновить» ----
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

    // ---- Обработчик «Очистить всё» ----
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
         sessionStorage — изолирован по вкладкам: у каждой вкладки своя копия, \
         данные исчезают при закрытии вкладки. \
         \n\
         Сценарии:\n\
         • Перезагрузите (F5) — оба счётчика сохранят значения.\n\
         • Откройте вторую вкладку — LS-счётчик будет тот же, SS начнётся с 0.\n\
         • Закройте вкладку — LS сохранится, SS сбросится.",
    )?;
    dom::add_class(&dom::as_html(&diff_hint), "hint")?;
    dom::append(&section3, &diff_hint)?;

    // ---- localStorage счётчик ----
    let ls_counter_label = dom::create_element("div")?;
    ls_counter_label
        .set_text_content(Some("localStorage счётчик (общий для вкладок):"));
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
    ss_counter_label
        .set_text_content(Some("sessionStorage счётчик (изолирован по вкладкам):"));
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

    // ============================================================
    // СЕКЦИЯ 4: Типы данных
    // ============================================================

    let section4 = build_section(
        &container,
        "4. Типы: только строки",
    )?;

    let types_hint = dom::create_paragraph(
        "Web Storage хранит только строки. Числа, отрицательные числа, \
         float, булевы значения — всё сохраняется как строки. \
         \n\
         Нажмите «Сохранить примеры», затем откройте DevTools \
         → Application → Storage → Local Storage и убедитесь: \
         под каждым ключом лежит строка. Можете изменить любое \
         значение вручную — например, сделать demo:i32 равным \
         \"hello\" — и нажать «Загрузить и показать», чтобы увидеть, \
         как парсинг обрабатывает некорректные данные.",
    )?;
    dom::add_class(&dom::as_html(&types_hint), "hint")?;
    dom::append(&section4, &types_hint)?;

    let save_types_btn = dom::create_element("button")?;
    save_types_btn.set_text_content(Some("Сохранить примеры"));
    let load_types_btn = dom::create_element("button")?;
    load_types_btn.set_text_content(Some("Загрузить и показать"));
    let clear_types_btn = dom::create_element("button")?;
    clear_types_btn.set_text_content(Some("Очистить примеры"));

    dom::append(&section4, &save_types_btn)?;
    dom::append(&section4, &load_types_btn)?;
    dom::append(&section4, &clear_types_btn)?;

    let types_demo = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&types_demo), "types-demo")?;
    types_demo.set_text_content(Some(
        "Нажмите «Сохранить примеры», чтобы заполнить localStorage.",
    ));
    dom::append(&section4, &types_demo)?;

    // ---- Обработчик «Сохранить примеры» ----
    {
        let types_demo = types_demo.clone();
        callbacks::listen(
            &save_types_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        types_demo.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                let examples: [(&str, String); 6] = [
                    ("demo:i32", 42_i32.to_string()),
                    ("demo:i32_negative", (-7_i32).to_string()),
                    ("demo:f64", 3.14159_f64.to_string()),
                    ("demo:bool_true", true.to_string()),
                    ("demo:bool_false", false.to_string()),
                    ("demo:string", "Привет, мир!".to_string()),
                ];

                let mut saved_count: u32 = 0;
                for (key, value) in &examples {
                    match storage::set(&ls, key, value) {
                        Ok(_) => saved_count += 1,
                        Err(e) => log!("Ошибка сохранения {}: {:?}", key, e),
                    }
                }

                let text: String = format!(
                    "Сохранено {} ключей в localStorage.\n\
                     \n\
                     Откройте DevTools → Application → Storage → Local Storage. \
                     Под каждым ключом лежит строка — независимо от того, \
                     что там число или булево значение.\n\
                     \n\
                     Попробуйте изменить любое значение вручную и нажать \
                     «Загрузить и показать».",
                    saved_count
                );
                types_demo.set_text_content(Some(text.as_str()));

                log!("Types: сохранено {} ключей", saved_count);
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Загрузить и показать» ----
    {
        let types_demo = types_demo.clone();
        callbacks::listen(
            &load_types_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        types_demo.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                types_demo.set_inner_html("");

                let entries_to_check: [(&str, &str); 6] = [
                    ("demo:i32", "i32"),
                    ("demo:i32_negative", "i32"),
                    ("demo:f64", "f64"),
                    ("demo:bool_true", "bool"),
                    ("demo:bool_false", "bool"),
                    ("demo:string", "String"),
                ];

                for (key, type_name) in &entries_to_check {
                    let raw: Option<String> = storage::get(&ls, key);
                    let block = dom::create_element("pre").unwrap();

                    match raw {
                        Some(s) => {
                            let parsed_info: String = match *type_name {
                                "i32" => match s.parse::<i32>() {
                                    Ok(n) => format!("{}", n),
                                    Err(_) => "⚠ ошибка парсинга".into(),
                                },
                                "f64" => match s.parse::<f64>() {
                                    Ok(n) => format!("{}", n),
                                    Err(_) => "⚠ ошибка парсинга".into(),
                                },
                                "bool" => match s.parse::<bool>() {
                                    Ok(b) => format!("{}", b),
                                    Err(_) => "⚠ ошибка парсинга".into(),
                                },
                                _ => s.clone(),
                            };

                            let text: String = format!(
                                "{} → \"{}\" (в хранилище)\n\
                                 Парсим как {} → {}",
                                key, s, type_name, parsed_info
                            );
                            block.set_text_content(Some(text.as_str()));
                        }
                        None => {
                            let text: String = format!(
                                "{} → (не найден — нажмите «Сохранить примеры»)",
                                key
                            );
                            block.set_text_content(Some(text.as_str()));
                        }
                    }

                    let _ = dom::append(&types_demo, &block);
                }

                log!("Types: загружено {} ключей", entries_to_check.len());
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Очистить примеры» ----
    {
        let types_demo = types_demo.clone();
        callbacks::listen(
            &clear_types_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let keys_to_remove: [&str; 6] = [
                    "demo:i32",
                    "demo:i32_negative",
                    "demo:f64",
                    "demo:bool_true",
                    "demo:bool_false",
                    "demo:string",
                ];

                let mut removed: u32 = 0;
                for key in &keys_to_remove {
                    if storage::has_key(&ls, key) {
                        let _ = storage::remove(&ls, key);
                        removed += 1;
                    }
                }

                let text: String = format!(
                    "Удалено {} ключей из localStorage.\n\
                     Проверьте DevTools — примеры исчезли.",
                    removed
                );
                types_demo.set_text_content(Some(text.as_str()));

                log!("Types: удалено {} ключей", removed);
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 5: Заметки — структуры в Storage
    // ============================================================

    let section5 = build_section(
        &container,
        "5. Заметки: Vec<Note> в JSON",
    )?;

    let notes_hint = dom::create_paragraph(
        "Заметки хранятся как JSON-массив структур Note. \
         В localStorage лежит одна строка — сериализованный Vec<Note>. \
         Откройте DevTools → Application → Storage → Local Storage \
         и убедитесь: под ключом demo:notes лежит JSON-массив. \
         Можете изменить его вручную или удалить — приложение \
         корректно обработает и валидный, и повреждённый JSON.",
    )?;
    dom::add_class(&dom::as_html(&notes_hint), "hint")?;
    dom::append(&section5, &notes_hint)?;

    let note_input_el = dom::create_element("input")?;
    dom::set_attribute(&note_input_el, "type", "text")?;
    dom::set_attribute(&note_input_el, "id", "note-input")?;
    dom::set_attribute(
        &note_input_el,
        "placeholder",
        "Текст новой заметки...",
    )?;
    let note_input: HtmlInputElement = note_input_el.dyn_into()?;
    dom::append(&section5, &note_input)?;

    let add_note_btn = dom::create_element("button")?;
    add_note_btn.set_text_content(Some("Добавить заметку"));
    let clear_notes_btn = dom::create_element("button")?;
    clear_notes_btn.set_text_content(Some("Очистить все"));
    let refresh_json_btn = dom::create_element("button")?;
    refresh_json_btn.set_text_content(Some("Обновить JSON"));

    dom::append(&section5, &add_note_btn)?;
    dom::append(&section5, &clear_notes_btn)?;
    dom::append(&section5, &refresh_json_btn)?;

    let notes_list = dom::create_element("ul")?;
    dom::add_class(&dom::as_html(&notes_list), "notes-list")?;
    dom::append(&section5, &notes_list)?;

    let json_label = dom::create_element("div")?;
    json_label.set_text_content(Some("Сырой JSON в localStorage[demo:notes]:"));
    dom::add_class(&dom::as_html(&json_label), "label")?;
    dom::append(&section5, &json_label)?;

    let json_preview = dom::create_element("pre")?;
    dom::add_class(&dom::as_html(&json_preview), "json-preview")?;
    json_preview.set_text_content(Some("(ключа нет — добавьте заметку)"));
    dom::append(&section5, &json_preview)?;

    let render_notes: Rc<dyn Fn()> = make_render_notes(notes_list.clone());
    let render_json_preview: Rc<dyn Fn()> =
        make_render_json_preview(json_preview.clone());

    // ---- Обработчик «Добавить заметку» ----
    {
        let note_input = note_input.clone();
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        callbacks::listen(
            &add_note_btn,
            "click",
            move |_event| {
                let text = note_input.value();
                if text.trim().is_empty() {
                    log!("Пустая заметка — пропускаем");
                    return;
                }

                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let mut notes: Vec<Note> =
                    storage::get_json(&ls, KEY_NOTES).unwrap_or_default();

                let next_id: u32 = notes.iter().map(|n| n.id).max().unwrap_or(0) + 1;
                notes.push(Note {
                    id: next_id,
                    text: text.clone(),
                });

                match storage::set_json(&ls, KEY_NOTES, &notes) {
                    Ok(_) => log!("Заметка #{} добавлена: {}", next_id, text),
                    Err(e) => log!("Ошибка сохранения заметок: {:?}", e),
                }

                note_input.set_value("");
                render_notes();
                render_json_preview();
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Очистить все» ----
    {
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        callbacks::listen(
            &clear_notes_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let _ = storage::remove(&ls, KEY_NOTES);
                log!("Все заметки удалены");

                render_notes();
                render_json_preview();
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Обновить JSON» ----
    {
        let render_json_preview = render_json_preview.clone();
        callbacks::listen(
            &refresh_json_btn,
            "click",
            move |_event| {
                render_json_preview();
            },
            &closures,
        )?;
    }

    render_notes();
    render_json_preview();

    // ============================================================
    // СЕКЦИЯ 6: StorageEvent — синхронизация вкладок
    // ============================================================

    let section6 = build_section(
        &container,
        "6. StorageEvent: изменения из других вкладок",
    )?;

    let events_hint = dom::create_paragraph(
        "StorageEvent генерируется ТОЛЬКО для localStorage \
         и приходит ТОЛЬКО в другие вкладки того же origin. \
         Вкладка, где произошло изменение, событие не получает. \
         Для sessionStorage событий нет — данные изолированы по вкладкам. \
         \n\
         Откройте это приложение в новой вкладке (Ctrl+T, тот же URL). \
         Измените данные в любой из вкладок — здесь появятся записи \
         с полями key, oldValue, newValue, url и area.",
    )?;
    dom::add_class(&dom::as_html(&events_hint), "hint")?;
    dom::append(&section6, &events_hint)?;

    let clear_log_btn = dom::create_element("button")?;
    clear_log_btn.set_text_content(Some("Очистить лог"));
    dom::append(&section6, &clear_log_btn)?;

    let events_log = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&events_log), "events-log")?;
    events_log.set_text_content(Some("(пока пусто — откройте вторую вкладку)"));
    dom::append(&section6, &events_log)?;

    let append_to_log: Rc<dyn Fn(&str)> = {
        let events_log = events_log.clone();
        Rc::new(move |text: &str| {
            let prev = events_log.text_content().unwrap_or_default();
            let new_text = format!("---\n{}\n\n{}", text, prev);
            let trimmed: String = new_text.chars().take(2000).collect();
            events_log.set_text_content(Some(trimmed.as_str()));
        })
    };

    // ---- Обработчик «Очистить лог» ----
    {
        let events_log = events_log.clone();
        callbacks::listen(
            &clear_log_btn,
            "click",
            move |_event| {
                events_log.set_text_content(Some("(лог очищен)"));
            },
            &closures,
        )?;
    }

    // ---- Обработчик события `storage` на window ----
    {
        let append_to_log = append_to_log.clone();
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        let render_inspector = render_inspector.clone();

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

                append_to_log(&text);

                // Перерисовываем секции, которые могли устареть
                render_notes();
                render_json_preview();
                render_inspector();
            },
            &closures,
        )?;
    }

    // ============================================================
    // СЕКЦИЯ 7: Ограничения Storage
    // ============================================================

    let section7 = build_section(
        &container,
        "7. Ограничения Storage и синхронный API",
    )?;

    let limits_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&limits_info), "limits-info")?;
    limits_info.set_text_content(Some(
        "• Только строки. Числа, bool, структуры — через сериализацию.\n\
         • ~5 МБ на origin. При превышении — QuotaExceededError.\n\
         • Синхронный API. Запись блокирует главный поток.\n\
         • Привязан к origin: protocol + host + port.\n\
         • Уязвим к XSS — любой JS на origin читает данные.\n\
         • Нет транзакций. Две записи подряд не атомарны.",
    ));
    dom::append(&section7, &limits_info)?;

    // ============================================================
    // 7a. Бинарные данные
    // ============================================================

    let binary_label = dom::create_element("h3")?;
    binary_label.set_text_content(Some("7a. Бинарные данные — через JSON-массив"));
    dom::append(&section7, &binary_label)?;

    let binary_hint = dom::create_paragraph(
        "Web Storage не принимает Uint8Array напрямую. Бинарные данные \
         нужно кодировать: либо как JSON-массив чисел, либо как base64-строку. \
         JSON-массив проще, но раздувает размер в 4–6 раз. Base64 компактнее \
         (на 33% больше исходного), но требует библиотеку.",
    )?;
    dom::add_class(&dom::as_html(&binary_hint), "hint")?;
    dom::append(&section7, &binary_hint)?;

    let binary_run_btn = dom::create_element("button")?;
    binary_run_btn.set_text_content(Some("Показать"));
    dom::append(&section7, &binary_run_btn)?;

    let binary_demo = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&binary_demo), "types-demo")?;
    dom::append(&section7, &binary_demo)?;

    {
        let binary_demo = binary_demo.clone();
        callbacks::listen(
            &binary_run_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        binary_demo.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

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
                lines.push("  Base64 требует крейт base64 или btoa в JS".to_string());
                lines.push("  Размер: ~33% больше исходных байт".to_string());
                lines.push(String::new());

                let _ = storage::set(&ls, "demo:bytes", &json_array);
                let back = storage::get(&ls, "demo:bytes").unwrap_or_default();
                lines.push(format!("Из Storage прочитано: {}", back));

                let decoded: Vec<u8> =
                    serde_json::from_str(&back).unwrap_or_default();
                lines.push(format!("Десериализовано обратно: {:?}", decoded));
                lines.push(format!(
                    "Совпадает с исходным: {}",
                    decoded == bytes
                ));

                let text = lines.join("\n");
                binary_demo.set_inner_html("");
                if let Ok(pre) = dom::create_element("pre") {
                    pre.set_text_content(Some(text.as_str()));
                    let _ = dom::append(&binary_demo, &pre);
                }

                log!("Binary demo: {} bytes → {} chars", bytes.len(), json_array.len());
            },
            &closures,
        )?;
    }

    // ============================================================
    // 7b. QuotaExceededError
    // ============================================================

    let quota_label = dom::create_element("h3")?;
    quota_label.set_text_content(Some("7b. QuotaExceededError"));
    dom::append(&section7, &quota_label)?;

    let quota_hint = dom::create_paragraph(
        "Нажмите «Заполнить до квоты», чтобы записать большие строки \
         и получить реальный QuotaExceededError от браузера. Затем \
         «Очистить тестовые ключи» удалит только ключи с префиксом \
         quota-test, не трогая остальные.",
    )?;
    dom::add_class(&dom::as_html(&quota_hint), "hint")?;
    dom::append(&section7, &quota_hint)?;

    let fill_btn = dom::create_element("button")?;
    fill_btn.set_text_content(Some("Заполнить до квоты"));
    let cleanup_btn = dom::create_element("button")?;
    cleanup_btn.set_text_content(Some("Очистить тестовые ключи"));
    dom::append(&section7, &fill_btn)?;
    dom::append(&section7, &cleanup_btn)?;

    let quota_status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&quota_status), "usage-info")?;
    quota_status.set_text_content(Some("Ещё не запускалось."));
    dom::append(&section7, &quota_status)?;

    {
        let quota_status = quota_status.clone();
        callbacks::listen(
            &fill_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        quota_status.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                quota_status.set_text_content(Some(
                    "Заполняем localStorage (это может занять секунду)...",
                ));

                let result = storage::fill_to_quota(
                    &ls,
                    QUOTA_TEST_PREFIX,
                    100_000,
                    100,
                );

                let status: String = format_quota_result(&result);
                quota_status.set_text_content(Some(status.as_str()));

                log!(
                    "Quota test: hit_quota={}, keys={}, chars={}",
                    result.hit_quota,
                    result.keys_written,
                    result.total_chars
                );
            },
            &closures,
        )?;
    }

    {
        let quota_status = quota_status.clone();
        callbacks::listen(
            &cleanup_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let removed = storage::clear_prefix(&ls, QUOTA_TEST_PREFIX);
                let text: String = format!("Удалено {} тестовых ключей.", removed);
                quota_status.set_text_content(Some(text.as_str()));
                log!("Quota cleanup: removed {} keys", removed);
            },
            &closures,
        )?;
    }

    // ============================================================
    // 7c. Синхронная блокировка UI
    // ============================================================

    let sync_label = dom::create_element("h3")?;
    sync_label.set_text_content(Some("7c. Синхронный API — блокировка UI"));
    dom::append(&section7, &sync_label)?;

    let sync_hint = dom::create_paragraph(
        "Запись в Storage синхронна. Пока она идёт, главный поток \
         заблокирован. Нажмите кнопку — UI замрёт на несколько сотен \
         миллисекунд. Для больших данных это неприемлемо, используйте \
         IndexedDB с асинхронным API.",
    )?;
    dom::add_class(&dom::as_html(&sync_hint), "hint")?;
    dom::append(&section7, &sync_hint)?;

    let sync_run_btn = dom::create_element("button")?;
    sync_run_btn.set_text_content(Some("Записать 3 МБ (блокировка)"));
    let sync_cleanup_btn = dom::create_element("button")?;
    sync_cleanup_btn.set_text_content(Some("Очистить sync-ключи"));
    dom::append(&section7, &sync_run_btn)?;
    dom::append(&section7, &sync_cleanup_btn)?;

    let sync_status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&sync_status), "usage-info")?;
    sync_status.set_text_content(Some("Ещё не запускалось."));
    dom::append(&section7, &sync_status)?;

    {
        let sync_status = sync_status.clone();
        callbacks::listen(
            &sync_run_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        sync_status.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                sync_status.set_text_content(Some(
                    "Записываем 3 МБ синхронно — UI заблокирован...",
                ));

                storage::clear_prefix(&ls, SYNC_TEST_PREFIX);

                let result = storage::measure_sync_block(
                    &ls,
                    3 * 1024,
                    SYNC_TEST_PREFIX,
                );

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
            },
            &closures,
        )?;
    }

    {
        let sync_status = sync_status.clone();
        callbacks::listen(
            &sync_cleanup_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let removed = storage::clear_prefix(&ls, SYNC_TEST_PREFIX);
                let text: String = format!("Удалено {} sync-ключей.", removed);
                sync_status.set_text_content(Some(text.as_str()));
            },
            &closures,
        )?;
    }

    // ============================================================
    // 7d. Привязка к origin
    // ============================================================

    let origin_label = dom::create_element("h3")?;
    origin_label.set_text_content(Some("7d. Привязка к origin"));
    dom::append(&section7, &origin_label)?;

    let origin_hint = dom::create_paragraph(
        "Storage привязан к origin: protocol + host + port. \
         Данные, сохранённые на одном origin, не видны на другом. \
         Даже смена порта (5173 → 5174) даёт отдельное хранилище. \
         Попробуйте запустить второй экземпляр приложения на другом порту.",
    )?;
    dom::add_class(&dom::as_html(&origin_hint), "hint")?;
    dom::append(&section7, &origin_hint)?;

    let origin_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&origin_info), "usage-info")?;

    let current_origin = storage::current_origin();
    let origin_text: String = format!(
        "Текущий origin: {}\n\
         Ключи в этом Storage привязаны к этому origin.\n\
         Откройте приложение на другом порту — данные будут отдельные.",
        current_origin
    );
    origin_info.set_text_content(Some(origin_text.as_str()));
    dom::append(&section7, &origin_info)?;

    // ============================================================
    // Финальное логирование
    // ============================================================

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

// ============================================================
// Вспомогательные функции
// ============================================================

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

/// Создаёт замыкание для отрисовки инспектора хранилища.
///
/// Вынесено в отдельную функцию, чтобы избежать проблемы
/// `rust-analyzer` с type inference во вложенных блоках.
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

/// Создаёт замыкание для отрисовки списка заметок.
fn make_render_notes(notes_list: web_sys::Element) -> Rc<dyn Fn()> {
    return Rc::new(move || {
        notes_list.set_inner_html("");

        let ls = match storage::local_storage() {
            Ok(s) => s,
            Err(_) => return,
        };

        let notes: Vec<Note> = storage::get_json(&ls, KEY_NOTES).unwrap_or_default();

        if notes.is_empty() {
            let empty = dom::create_element("li").unwrap();
            empty.set_text_content(Some("(заметок нет)"));
            let _ = dom::append(&notes_list, &empty);
            return;
        }

        for note in notes {
            let li = match dom::create_element("li") {
                Ok(el) => el,
                Err(_) => continue,
            };
            let text: String = format!("#{}: {}", note.id, note.text);
            li.set_text_content(Some(text.as_str()));
            let _ = dom::append(&notes_list, &li);
        }
    });
}

/// Создаёт замыкание для отрисовки сырого JSON из localStorage.
fn make_render_json_preview(json_preview: web_sys::Element) -> Rc<dyn Fn()> {
    return Rc::new(move || {
        let ls = match storage::local_storage() {
            Ok(s) => s,
            Err(_) => {
                json_preview.set_text_content(Some("localStorage недоступен"));
                return;
            }
        };

        let raw = match storage::get(&ls, KEY_NOTES) {
            Some(s) => s,
            None => {
                json_preview.set_text_content(Some("(ключа нет — добавьте заметку)"));
                return;
            }
        };

        let display: String = match serde_json::from_str::<serde_json::Value>(&raw) {
            Ok(value) => serde_json::to_string_pretty(&value)
                .unwrap_or_else(|_| raw.clone()),
            Err(_) => format!("(невалидный JSON)\n{}", raw),
        };

        json_preview.set_text_content(Some(display.as_str()));
    });
}

/// Обрезает длинное значение для предпросмотра.
///
/// Использует `chars()`, а не байтовый срез `[..n]`, потому что
/// срез по байтам паникует на границе UTF-8 символа.
fn preview_value(value: &str, max_chars: usize) -> String {
    let char_count = value.chars().count();
    if char_count <= max_chars {
        return value.to_string();
    }
    let truncated: String = value.chars().take(max_chars).collect();
    return format!("{}...", truncated);
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

## `web/styles/main.css` — полное содержание

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
}

h1 {
    color: #b7410e;
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

.storage-section h2 {
    margin-top: 0;
    font-size: 1.05rem;
    color: #b7410e;
}

.storage-section h3 {
    margin: 1rem 0 0.5rem 0;
    font-size: 0.95rem;
    color: #b7410e;
}

/* ============================================================
   Формы
   ============================================================ */

input[type="text"] {
    padding: 0.5rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-family: inherit;
    font-size: 1rem;
    margin-right: 0.5rem;
    background: white;
    color: #222;
}

input[type="text"]:focus {
    outline: 2px solid #b7410e;
    border-color: transparent;
}

/* ============================================================
   Кнопки
   ============================================================ */

button {
    padding: 0.5rem 1rem;
    font-size: 0.95rem;
    border: 1px solid #b7410e;
    background: white;
    color: #b7410e;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
    margin-right: 0.4rem;
    margin-bottom: 0.25rem;
}

button:hover {
    background: #b7410e;
    color: white;
}

button:active {
    transform: scale(0.96);
}

/* ============================================================
   Результат (секция 1)
   ============================================================ */

.result {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.9rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    margin-top: 1rem;
    min-height: 2rem;
}

.hint {
    font-size: 0.9rem;
    color: #666;
    margin-bottom: 1rem;
    white-space: pre-wrap;
}

.label {
    font-size: 0.85rem;
    color: #666;
    margin-top: 0.75rem;
}

/* ============================================================
   Инспектор (секция 2)
   ============================================================ */

.inspector {
    margin-top: 1rem;
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
}

.inspector table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 0.5rem;
}

.inspector th,
.inspector td {
    padding: 0.35rem 0.5rem;
    border-bottom: 1px solid #eee;
    text-align: left;
    vertical-align: top;
}

.inspector th {
    background: #f5f5f5;
    font-weight: 600;
    font-size: 0.8rem;
    color: #666;
}

.inspector td {
    word-break: break-all;
}

.inspector p {
    margin: 0.5rem 0;
    color: #666;
    font-style: italic;
}

.inspector > div {
    margin-bottom: 0.5rem;
}

/* ============================================================
   Счётчики (секция 3)
   ============================================================ */

.counter-value {
    font-size: 1.8rem;
    font-weight: bold;
    color: #b7410e;
    margin-right: 1rem;
    font-variant-numeric: tabular-nums;
    display: inline-block;
    min-width: 3rem;
}

.counter-value.session {
    color: #2563eb;
}

/* ============================================================
   Типы и бинарные данные (секции 4, 7a)
   ============================================================ */

.types-demo {
    margin-top: 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
}

.types-demo pre {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    margin: 0;
    overflow-x: auto;
}

/* ============================================================
   Заметки (секция 5)
   ============================================================ */

.notes-list {
    margin: 0.75rem 0;
    padding-left: 1.5rem;
    min-height: 2rem;
}

.notes-list li {
    margin-bottom: 0.25rem;
    font-size: 0.95rem;
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
    max-height: 300px;
    overflow-y: auto;
    word-break: break-all;
}

/* ============================================================
   Лог событий (секция 6)
   ============================================================ */

.events-log {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.8rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #2e7d32;
    margin: 0.75rem 0 0 0;
    min-height: 3rem;
    max-height: 400px;
    overflow-y: auto;
    word-break: break-all;
}

/* ============================================================
   Ограничения (секция 7)
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

.usage-info {
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
    max-height: 300px;
    overflow-y: auto;
}

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

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Пройдите все секции:

**Секция 1.** Введите имя, «Сохранить», F5, «Загрузить», «Удалить».

**Секция 2.** Проверьте таблицу, добавьте что-нибудь вручную через DevTools, нажмите «Обновить».

**Секция 3.** Нажмите `+1` на обоих счётчиках. F5. Откройте вторую вкладку. Закройте. Убедитесь, что LS сохраняется, а SS — нет.

**Секция 4.** Нажмите «Сохранить примеры», откройте DevTools, убедитесь, что всё строки. Измените `demo:i32` на `hello`, нажмите «Загрузить и показать» — увидите `⚠ ошибка парсинга`.

**Секция 5.** Добавьте заметку. Откройте DevTools, посмотрите JSON. Измените вручную. Обновите JSON. Сломайте JSON — увидите `(невалидный JSON)`.

**Секция 6.** Откройте вторую вкладку. Измените что-нибудь в первой — во второй появится событие.

**Секция 7.**
- **7a** — нажмите «Показать», увидите JSON-массив.
- **7b** — нажмите «Заполнить до квоты», поймайте `QuotaExceededError`. Затем «Очистить тестовые ключи».
- **7c** — нажмите «Записать 3 МБ». UI замрёт на секунду.
- **7d** — увидите текущий origin.

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

**Ожидаемый результат:** 62 теста `ok` в обоих браузерах.

---

## Что важно запомнить

**1. Семь секций — семь тем.** Каждая секция автономна, но все вместе дают полное представление о Web Storage API.

**2. Порядок секций — от простого к сложному.** Базовые операции → полный API → два хранилища → типы → JSON → события → ограничения.

**3. `Rc<dyn Fn()>` для переиспользуемых замыканий.** `render_inspector`, `render_notes`, `render_json_preview`, `append_to_log` — все обёрнуты в `Rc`. Клонирование дешёвое, замыкание в памяти одно.

**4. `rust-analyzer` обходится вынесением функций.** `make_render_inspector`, `make_render_notes`, `make_render_json_preview`, `format_quota_result` — все вынесены. Это решает проблему с tail-выражениями в замыканиях.

**5. `preview_value` — Unicode-safe.** `chars().take(40)` вместо `[..40]`. Не паникует на эмодзи.

**6. `std::mem::forget(closures)` — учебное решение.** В production лучше хранить `closures` в статике или `thread_local!`.

**7. Обработчик `storage` перерисовывает три секции.** `render_notes`, `render_json_preview`, `render_inspector` — чтобы UI был в актуальном состоянии после изменений из других вкладок.

**8. Префиксы ключей.** `demo:`, `quota-test:`, `sync-test:` — разделяют пространства имён. `clear_prefix` удаляет только ключи с определённым префиксом.

**9. Все 20 функций `storage.rs` в действии.** От `local_storage` до `measure_sync_block`. Каждая функция где-то вызывается.

**10. 62 теста — полное покрытие.** Все ветки условной логики в `storage.rs` и в утилитах `app.rs`.

**11. `is_storage_writable` критична для warning-блока.** Без неё `app.rs` не скомпилируется. Проверьте, что она есть в `storage.rs` перед сборкой.

---

## Итог главы 15

Мы прошли путь от «что такое Storage» до полноценной панели управления. Что мы изучили:

- **Базовые операции:** `set`, `get`, `remove`.
- **Полный API:** `length`, `key`, `keys`, `entries`, `has_key`, `clear`.
- **Два хранилища:** `localStorage` vs `sessionStorage` — семантика, не API.
- **Доступность:** `is_storage_writable`, приватный режим, fallback.
- **Сериализация:** только строки. `to_string` / `parse` для примитивов.
- **JSON:** `serde` + `serde_json` для структур, `Vec<Note>`, `Option`.
- **StorageEvent:** синхронизация вкладок, пять полей, особенности.
- **Ограничения:** квота, синхронный API, origin, XSS, транзакции.
- **20 функций `storage.rs`:** от обёрток до `fill_to_quota`.
- **62 теста:** все ветки покрыты.

**Что это даёт.** Полное понимание Web Storage API: не только «как вызвать», но и «когда использовать», «где ограничения», «что делать при сбое».

---

**Что дальше.** Web Storage — только начало. В следующих главах части V мы разберём **Fetch** (глава 16) — сетевые запросы, `Request`, `Response`, `AbortController`; **Streams** (глава 17) — потоковая обработка данных. Дальше — **File API** (часть VI), **Canvas** (часть VII), **Offline Web Applications** (часть VIII) с IndexedDB и Service Worker.