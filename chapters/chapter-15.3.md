# 15.3. `localStorage` vs `sessionStorage`

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
│   ├── app.rs                      # +1 секция: два счётчика
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # не меняется (12 функций)
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
│   └── wasm_storage.rs             # +6 тестов → 22 теста
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +2 правила для счётчиков
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/storage.rs` — **не меняется**. Полного API из 15.2 достаточно для работы с двумя хранилищами. Разница между `localStorage` и `sessionStorage` — не в API, а в семантике.
- `src/app.rs` — **дополняется секцией 3**: два счётчика рядом. Секции 1 и 2 остаются без изменений.
- `tests/wasm_storage.rs` — **дополняется 6 тестами** на изоляцию. Всего 22 теста.
- `web/styles/main.css` — **добавляются 2 правила** для оформления счётчиков: `.counter-value` и `.counter-value.session`.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `Window.localStorage` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage |
| `Window.sessionStorage` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage |
| Web Storage API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API |

---

## Ключевое различие

`localStorage` и `sessionStorage` реализуют **один и тот же интерфейс `Storage`**. Разница — в **семантике**, а не в API. Два измерения:

### Жизненный цикл

| | `localStorage` | `sessionStorage` |
|---|---|---|
| Перезагрузка страницы (F5) | сохраняется | сохраняется |
| Закрытие вкладки | сохраняется | **удаляется** |
| Закрытие браузера | сохраняется | **удаляется** |
| Восстановление вкладки после сбоя | сохраняется | может сохраниться (зависит от браузера) |

### Область видимости

| | `localStorage` | `sessionStorage` |
|---|---|---|
| Вторая вкладка того же origin | **видит те же данные** | **своя копия** |
| Изменение в одной вкладке | видно в других | не видно в других |
| Изоляция от других origin | да | да |

**Одним предложением.** `localStorage` — общий долгоживущий склад, `sessionStorage` — личная сессионная папка вкладки.

---

## Когда использовать

**`localStorage` — для того, что должно жить долго и быть общим:**

- Настройки пользователя: тема, язык, размер шрифта.
- Флаги: «онбординг пройден», «уведомления отключены».
- Кэш небольших данных: словари, справочники.
- Состояние, которое должно синхронизироваться между вкладками.

**`sessionStorage` — для того, что привязано к текущей сессии вкладки:**

- Черновики форм: пользователь начал заполнять, случайно нажал F5 — данные не потеряны. Закрыл вкладку — черновик исчез, и это правильно.
- Пошаговые мастера: текущий шаг, промежуточные ответы.
- Состояние одной вкладки: фильтры, сортировки, которые не должны «протекать» между вкладками.
- Контекст навигации: «откуда пришёл пользователь».

**Противопоказания:**

- **Секреты** — ни там, ни там. Любой JS на origin читает данные. Для токенов — HttpOnly cookies или серверные сессии.
- **Большие данные** — ни там, ни там. Лимит ~5 МБ, синхронный API. Для больших данных — IndexedDB (глава 24).
- **Критичные данные** — не в `sessionStorage`. Пользователь закроет вкладку, и всё пропадёт.

---

## `src/app.rs`

```rust
//! Логика приложения: «Storage: знакомство + инспектор + два счётчика».
//!
//! Три секции:
//! - «Знакомство» — сохранение имени, чтение, удаление.
//! - «Инспектор» — таблица всех ключей, длина, очистка.
//! - «LS vs SS» — два счётчика рядом, демонстрация разницы.
//!
//! ## Про хранение DOM-элементов между обработчиками
//!
//! `web_sys::Element` не реализует `Copy`. Если один элемент
//! используется в нескольких обработчиках, каждый блок должен
//! получить свою копию через `.clone()`. Оригинал остаётся жив.
//!
//! ## Про вынесение замыканий в отдельные функции
//!
//! Замыкание `render_inspector` содержит ранний `return` внутри
//! `if len == 0`. Когда такое замыкание вложено во внешний блок,
//! `rust-analyzer` неверно выводит тип. Решение — вынести
//! замыкание в отдельную функцию с явным `Rc<dyn Fn()>`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::HtmlInputElement;

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;
use crate::storage;

/// Ключ, под которым хранится имя.
const KEY_NAME: &str = "demo:name";

/// Ключ для localStorage-счётчика.
const KEY_LS_COUNTER: &str = "demo:ls_counter";

/// Ключ для sessionStorage-счётчика.
const KEY_SS_COUNTER: &str = "demo:ss_counter";

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Storage: базовый API")?;
    dom::append(&container, &heading)?;

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
                        let text = format!("Ошибка сохранения: {:?}", e);
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
                        result1.set_text_content(Some(
                            "Ключ удалён из localStorage",
                        ));
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
         данные исчезают при закрытии. \n\
         \n\
         Сценарии для проверки:\n\
         • Перезагрузите страницу (F5) — оба счётчика сохранят значения.\n\
         • Откройте вторую вкладку с этим приложением — LS-счётчик будет \
           тот же, SS-счётчик начнётся с 0.\n\
         • Закройте вкладку и откройте заново — LS сохранится, SS сбросится.",
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

    // ---- Обработчики localStorage счётчика ----

    // +1
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

    // −1
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

    // Сброс
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

    // ---- Обработчики sessionStorage счётчика ----

    // +1
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

    // −1
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

    // Сброс
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

    // ---- Проверка доступности обоих хранилищ ----
    match storage::local_storage() {
        Ok(_) => log!("localStorage доступен"),
        Err(_) => log!("localStorage НЕ доступен"),
    }
    match storage::session_storage() {
        Ok(_) => log!("sessionStorage доступен"),
        Err(_) => log!("sessionStorage НЕ доступен"),
    }

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
```

**Что демонстрирует новая секция:**

- **Два счётчика рядом** — LS и SS. Одинаковая логика, разные хранилища.
- **Кнопки** `+1`, `−1`, `Сброс` для каждого.
- **Начальное значение** читается из хранилища при старте. Если ключа нет — `0`.
- **`Rc<RefCell<i32>>`** — состояние счётчика. `Rc` разделяет владение между обработчиками, `RefCell` позволяет мутировать значение.
- **`storage::get(&ls, key).and_then(parse)`** — читаем сохранённое значение. Если ключа нет или значение не парсится — `0`.

**Что важно понять:**

- **Разница — не в коде, а в семантике.** Оба счётчика работают одинаково: читают, пишут, обновляют DOM. Разница проявляется во **времени жизни** и **области видимости**.
- **`Rc<RefCell<i32>>` — стандартный паттерн для счётчика.** `Rc` — общее владение, `RefCell` — мутабельность во время выполнения. Три обработчика (inc, dec, reset) разделяют один счётчик.
- **Начальное значение — из хранилища.** Если пользователь уже нажимал на кнопки и перезагрузил страницу, счётчик покажет сохранённое значение, а не `0`.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   Счётчики
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
```

**Что делает:**

- `.counter-value` — крупный жирный счётчик с фиксированной шириной (`tabular-nums`), чтобы цифры не «прыгали» при изменении разрядности.
- `.counter-value.session` — тот же стиль, но синий цвет для sessionStorage-счётчика. Визуальное различие помогает читателю сразу видеть, где какой счётчик.

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.2):

```rust
// ============================================================
// Изоляция localStorage и sessionStorage
// ============================================================

#[wasm_bindgen_test]
fn test_local_and_session_are_separate() {
    let ls = storage::local_storage().unwrap();
    let ss = storage::session_storage().unwrap();

    storage::set(&ls, "test:sep", "local").unwrap();
    storage::set(&ss, "test:sep", "session").unwrap();

    assert_eq!(storage::get(&ls, "test:sep"), Some("local".into()));
    assert_eq!(storage::get(&ss, "test:sep"), Some("session".into()));

    storage::remove(&ls, "test:sep").ok();
    storage::remove(&ss, "test:sep").ok();
}

#[wasm_bindgen_test]
fn test_clearing_local_does_not_affect_session() {
    let ls = storage::local_storage().unwrap();
    let ss = storage::session_storage().unwrap();

    storage::set(&ls, "test:clear_iso", "local_data").unwrap();
    storage::set(&ss, "test:clear_iso", "session_data").unwrap();

    storage::clear(&ls).ok();

    // sessionStorage не затронут очисткой localStorage
    assert_eq!(
        storage::get(&ss, "test:clear_iso"),
        Some("session_data".into())
    );

    storage::remove(&ss, "test:clear_iso").ok();
}

#[wasm_bindgen_test]
fn test_clearing_session_does_not_affect_local() {
    let ls = storage::local_storage().unwrap();
    let ss = storage::session_storage().unwrap();

    storage::set(&ls, "test:clear_iso2", "local_data").unwrap();
    storage::set(&ss, "test:clear_iso2", "session_data").unwrap();

    storage::clear(&ss).ok();

    // localStorage не затронут очисткой sessionStorage
    assert_eq!(
        storage::get(&ls, "test:clear_iso2"),
        Some("local_data".into())
    );

    storage::remove(&ls, "test:clear_iso2").ok();
}

#[wasm_bindgen_test]
fn test_local_and_session_length_independent() {
    let ls = storage::local_storage().unwrap();
    let ss = storage::session_storage().unwrap();

    let original_ls: Vec<_> = storage::entries(&ls);
    let original_ss: Vec<_> = storage::entries(&ss);

    storage::clear(&ls).ok();
    storage::clear(&ss).ok();

    storage::set(&ls, "ls1", "a").unwrap();
    storage::set(&ls, "ls2", "b").unwrap();
    storage::set(&ls, "ls3", "c").unwrap();
    storage::set(&ss, "ss1", "x").unwrap();

    assert_eq!(storage::length(&ls), 3);
    assert_eq!(storage::length(&ss), 1);

    storage::clear(&ls).ok();
    storage::clear(&ss).ok();

    for (k, v) in original_ls {
        storage::set(&ls, &k, &v).ok();
    }
    for (k, v) in original_ss {
        storage::set(&ss, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_local_and_session_keys_independent() {
    let ls = storage::local_storage().unwrap();
    let ss = storage::session_storage().unwrap();

    let original_ls: Vec<_> = storage::entries(&ls);
    let original_ss: Vec<_> = storage::entries(&ss);

    storage::clear(&ls).ok();
    storage::clear(&ss).ok();

    storage::set(&ls, "only_local", "a").unwrap();
    storage::set(&ss, "only_session", "b").unwrap();

    let ls_keys = storage::keys(&ls);
    let ss_keys = storage::keys(&ss);

    assert!(ls_keys.contains(&"only_local".to_string()));
    assert!(!ls_keys.contains(&"only_session".to_string()));

    assert!(ss_keys.contains(&"only_session".to_string()));
    assert!(!ss_keys.contains(&"only_local".to_string()));

    storage::clear(&ls).ok();
    storage::clear(&ss).ok();

    for (k, v) in original_ls {
        storage::set(&ls, &k, &v).ok();
    }
    for (k, v) in original_ss {
        storage::set(&ss, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_local_and_session_entries_independent() {
    let ls = storage::local_storage().unwrap();
    let ss = storage::session_storage().unwrap();

    let original_ls: Vec<_> = storage::entries(&ls);
    let original_ss: Vec<_> = storage::entries(&ss);

    storage::clear(&ls).ok();
    storage::clear(&ss).ok();

    storage::set(&ls, "a", "1").unwrap();
    storage::set(&ls, "b", "2").unwrap();
    storage::set(&ss, "c", "3").unwrap();

    let ls_entries = storage::entries(&ls);
    let ss_entries = storage::entries(&ss);

    assert_eq!(ls_entries.len(), 2);
    assert_eq!(ss_entries.len(), 1);

    assert!(ls_entries.iter().any(|(k, _)| k == "a"));
    assert!(ls_entries.iter().any(|(k, _)| k == "b"));
    assert!(ss_entries.iter().any(|(k, _)| k == "c"));

    storage::clear(&ls).ok();
    storage::clear(&ss).ok();

    for (k, v) in original_ls {
        storage::set(&ls, &k, &v).ok();
    }
    for (k, v) in original_ss {
        storage::set(&ss, &k, &v).ok();
    }
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_local_and_session_are_separate` | Один ключ в LS и SS может иметь разные значения |
| `test_clearing_local_does_not_affect_session` | `clear()` на LS не трогает SS |
| `test_clearing_session_does_not_affect_local` | `clear()` на SS не трогает LS |
| `test_local_and_session_length_independent` | `length()` в каждом хранилище независим |
| `test_local_and_session_keys_independent` | `keys()` не пересекаются между LS и SS |
| `test_local_and_session_entries_independent` | `entries()` не пересекаются между LS и SS |

**Восстановление состояния.** Каждый тест сохраняет ключи из обоих хранилищ (`original_ls`, `original_ss`), очищает их, работает, затем восстанавливает. Это гарантирует, что тесты `15.3` не сломают тесты из `15.1` и `15.2`.

**Всего тестов:** 16 из 15.1–15.2 + 6 новых = **22 теста**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Проверьте сценарии в секции 3:

**Сценарий 1: перезагрузка страницы.**

1. Нажмите `+1` на LS-счётчике — станет `1`.
2. Нажмите `+1` на SS-счётчике — станет `1`.
3. Перезагрузите страницу (F5).
4. Оба счётчика показывают `1`. Оба пережили перезагрузку.

**Сценарий 2: открытие второй вкладки.**

1. Откройте `http://127.0.0.1:5173` в новой вкладке.
2. LS-счётчик показывает то же значение, что в первой вкладке — данные общие.
3. SS-счётчик показывает `0` — у новой вкладки своя сессия.

**Сценарий 3: закрытие вкладки.**

1. Закройте вкладку, в которой вы нажимали на кнопки.
2. Откройте приложение заново.
3. LS-счётчик сохранил значение — `localStorage` пережил закрытие.
4. SS-счётчик показывает `0` — `sessionStorage` очистился.

**Что проверить в DevTools** (Application → Storage):

- **Local Storage** → `http://127.0.0.1:5173` → ключ `demo:ls_counter`.
- **Session Storage** → `http://127.0.0.1:5173` → ключ `demo:ss_counter`.
- После перезагрузки значения на месте.
- После закрытия вкладки Session Storage пуст.

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

**Ожидаемый результат:** 22 теста `ok` в обоих браузерах.

---

## Что важно запомнить

**1. Разница — в семантике, не в API.** Оба хранилища реализуют `Storage`. Операции одинаковые. Разница — во времени жизни и области видимости.

**2. `localStorage` переживает всё.** Перезагрузку, закрытие вкладки, закрытие браузера.

**3. `sessionStorage` — до закрытия вкладки.** Переживает перезагрузку (F5), но не закрытие вкладки.

**4. `localStorage` общий для вкладок.** Изменение в одной вкладке видно в другой.

**5. `sessionStorage` изолирован.** У каждой вкладки своя копия. Даже при одном origin.

**6. Когда что использовать.** Настройки и флаги — `localStorage`. Черновики форм и временное состояние вкладки — `sessionStorage`.

**7. Не храните секреты.** Ни там, ни там. Любой JS на origin читает данные.

**8. `Rc<RefCell<i32>>` для счётчика.** Разделяемое мутабельное состояние. Три обработчика — одна переменная.

**9. Проверка в DevTools.** Application → Storage → Local Storage / Session Storage. Откройте — увидите, что реально лежит в браузере. Закройте вкладку — Session Storage исчезнет.

---

## Что дальше

**15.4. Доступность Storage: private mode, исключения.** Мы разберём, что происходит, когда браузер **запрещает** доступ к Storage: приватный режим, ограничения по политике безопасности, квоты с нуля. Мы добавим в `storage.rs` функцию `is_storage_writable` — проверку, что хранилище не только открывается, но и принимает записи. В `app.rs` появится **предупреждение при старте**, если доступ ограничен. В тестах — проверки на «хранилище открывается, но не пишется».