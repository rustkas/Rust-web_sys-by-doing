# 15.2. Полный API Storage

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
│   ├── app.rs                      # демо: знакомство + инспектор хранилища
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # 12 функций
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
│   └── wasm_storage.rs             # 15 тестов
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

**Комментарии к структуре:**

- `src/storage.rs` — расширен с 5 до 12 функций. Добавлены: `length`, `key`, `keys`, `entries`, `has_key`, `clear`.
- `src/app.rs` — дополнен секцией «Инспектор». Появляется таблица всех ключей, размер, кнопки «Обновить» и «Очистить всё». Вспомогательные функции `make_render_inspector` и `preview_value` вынесены в конец файла.
- `tests/wasm_storage.rs` — дополнен 8 тестами на итерацию и коллекции. Всего 15 тестов.
- `web/styles/main.css` — дополнен стилями для секций и таблицы ключей.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `Storage.length` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage/length |
| `Storage.key` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage/key |
| `Storage.clear` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage/clear |
| `web_sys::Storage` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.Storage.html |

---

## Зачем нужны остальные методы

В 15.1 мы покрыли 90% задач: сохранить, прочитать, удалить. Но есть класс сценариев, где нужен **полный API**:

- **Инспектор хранилища** — показать пользователю все ключи и значения. Требует `keys()`, `entries()`, `length()`.
- **Массовая очистка** — удалить всё, что начинается с определённого префикса. Требует `keys()` и цикл с `remove`.
- **Проверка существования** — узнать, есть ли ключ, не читая его значение. Требует `has_key()` (обёртка над `get().is_some()`).
- **Полная очистка** — стереть всё хранилище. Требует `clear()`.
- **Итерация в детерминированном порядке** — обойти все ключи в том порядке, в каком их возвращает браузер. Требует `length()` + `key(i)`.

**Как Storage устроен внутри.** Это **плоский список** пар «ключ → значение». Не массив, не объект, не Map. Нет вложенности, нет индексов (кроме позиции в списке), нет запросов. Ключи хранятся в порядке вставки, но спецификация это не гарантирует. Обход выполняется через `length()` (сколько всего) и `key(i)` (какой ключ на позиции `i`).

**`has_key`.** Прямого метода `has` у `Storage` нет. Проверка реализуется через `get(key).is_some()`. Мы оформляем это как отдельную функцию для читаемости.

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
//!
//! ## Что будет дальше
//!
//! В следующих подглавах модуль расширится:
//! - 15.4 — `is_storage_writable`.
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
/// доступ к `localStorage` (например, в приватном режиме).
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

// ============================================================
// Базовые операции
// ============================================================

/// Читает значение по ключу.
///
/// Возвращает `None`, если ключа нет.
///
/// Внутри: `get_item` возвращает `Result<Option<String>, JsValue>`.
/// `.ok()` превращает `Result` в `Option`, `.flatten()` убирает
/// вложенный `Option`. Итог — плоский `Option<String>`.
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
///
/// Удаление несуществующего ключа не считается ошибкой.
pub fn remove(storage: &Storage, key: &str) -> Result<(), JsValue> {
    return storage.remove_item(key);
}

// ============================================================
// Итерация по хранилищу
// ============================================================

/// Возвращает количество ключей в хранилище.
///
/// `length` в `web_sys` возвращает `Result<u32, JsValue>`. Ошибка
/// маловероятна, поэтому используем `unwrap_or(0)` — при сбое
/// вернём 0, что безопасно для итерации.
pub fn length(storage: &Storage) -> u32 {
    return storage.length().unwrap_or(0);
}

/// Возвращает ключ по индексу.
///
/// Индекс — позиция в хранилище, от 0 до `length() - 1`.
/// Если индекс вне диапазона, возвращает `None`.
///
/// Порядок ключей зависит от браузера. Спецификация не гарантирует
/// порядок вставки, но большинство браузеров возвращают ключи
/// в порядке добавления.
pub fn key(storage: &Storage, index: u32) -> Option<String> {
    return storage.key(index).ok().flatten();
}

/// Возвращает все ключи хранилища.
///
/// Обходит хранилище через `length()` + `key(i)`. Возвращает
/// `Vec<String>` в порядке, который вернул браузер.
pub fn keys(storage: &Storage) -> Vec<String> {
    let len = length(storage);
    return (0..len).filter_map(|i| key(storage, i)).collect();
}

/// Возвращает все пары ключ-значение.
///
/// Использует `keys()` для получения списка ключей, затем читает
/// значение каждого через `get()`. Пары, у которых значение не
/// удалось прочитать, пропускаются.
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
///
/// У `Storage` нет метода `has`. Проверка реализуется через
/// `get(key).is_some()`. Обёртка добавлена для читаемости:
/// `has_key(&ls, "theme")` понятнее, чем `get(&ls, "theme").is_some()`.
pub fn has_key(storage: &Storage, key: &str) -> bool {
    return get(storage, key).is_some();
}

// ============================================================
// Полная очистка
// ============================================================

/// Очищает хранилище.
///
/// Удаляет **все** ключи. Не путать с удалением одного ключа
/// через `remove`.
pub fn clear(storage: &Storage) -> Result<(), JsValue> {
    return storage.clear();
}
```

---

## `src/app.rs`

```rust
//! Логика приложения: «Storage: знакомство + инспектор».
//!
//! Две секции:
//! - «Знакомство» — сохранение имени, чтение, удаление.
//! - «Инспектор» — таблица всех ключей, размер, очистка.
//!
//! ## Про хранение DOM-элементов между обработчиками
//!
//! `web_sys::Element` не реализует `Copy`. Если один элемент
//! используется в нескольких обработчиках, каждый блок должен
//! получить свою копию через `.clone()`. Оригинал остаётся жив
//! и доступен для следующих блоков.
//!
//! ## Про вынесение замыканий в отдельные функции
//!
//! Замыкание `render_inspector` содержит ранний `return` внутри
//! `if len == 0`. Когда такое замыкание вложено во внешний блок
//! `{ ... Rc::new(move || { ... }) }`, `rust-analyzer` неверно
//! выводит тип. Решение — вынести замыкание в отдельную функцию
//! с явным возвращаемым типом `Rc<dyn Fn()>`.

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

    // Замыкание для отрисовки инспектора.
    // Вынесено в отдельную функцию — см. `make_render_inspector`.
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

    // Начальная отрисовка инспектора
    render_inspector();

    // Проверка доступности localStorage при старте
    match storage::local_storage() {
        Ok(_) => log!("localStorage доступен"),
        Err(_) => log!("localStorage НЕ доступен"),
    }

    // `closures` должен жить, пока работают обработчики.
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
/// `rust-analyzer` с type inference во вложенных блоках:
/// когда замыкание с ранним `return` обёрнуто во внешний
/// блок `{ ... Rc::new(...) }`, анализатор неверно выводит тип.
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

        // Заголовок с длиной
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

        // Таблица ключей
        let table = dom::create_element("table").unwrap();

        let header = dom::create_element("tr").unwrap();
        for name in ["#", "key", "value", "has_key"] {
            let th = dom::create_element("th").unwrap();
            th.set_text_content(Some(name));
            let _ = dom::append(&header, &th);
        }
        let _ = dom::append(&table, &header);

        for (i, k) in storage::keys(&ls).iter().enumerate() {
            let value = storage::get(&ls, k)
                .unwrap_or_else(|| "(none)".into());
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

        // Размер хранилища
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
/// срез по байтам паникует на границе UTF-8 символа. Например,
/// `&"Привет"[..5]` вызовет panic, потому что 5-й байт попадает
/// в середину многобайтового символа 'и'.
fn preview_value(value: &str, max_chars: usize) -> String {
    let char_count = value.chars().count();
    if char_count <= max_chars {
        return value.to_string();
    }
    let truncated: String = value.chars().take(max_chars).collect();
    return format!("{}...", truncated);
}
```

**Что демонстрирует `app.rs`:**

- **Секция 1 «Знакомство»** — то же, что в 15.1.
- **Секция 2 «Инспектор»** — таблица всех ключей: позиция, ключ, значение, `has_key`. Плюс количество ключей и суммарный размер.
- **`make_render_inspector`** — отдельная функция, возвращающая `Rc<dyn Fn()>`. Внутри — замыкание с ранним `return` в ветке «хранилище пусто». Вынесение в функцию обходит проблему `rust-analyzer`.
- **`preview_value`** — Unicode-безопасная обрезка. Использует `chars()`, а не `[..n]`.

**Что важно понять:**

- **`Rc<dyn Fn()>` для переиспользуемых замыканий.** `render_inspector` клонируется в два обработчика. Клонирование `Rc` дешёвое — счётчик ссылок, само замыкание в памяти одно.
- **`make_render_inspector(inspector_result.clone())`** — без внешнего блока `{ ... }`. `inspector_result.clone()` создаёт копию ссылки на DOM-элемент, оригинал остаётся доступен.
- **Обрезка по символам, а не по байтам.** `preview_value` безопасна для Unicode: `chars().take(40)` берёт 40 символов, а не 40 байт.
- **Пустое хранилище — отдельная ветка.** Показываем «(хранилище пусто)» вместо пустой таблицы.

---

## `web/index.html`

Без изменений относительно 15.1.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
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

/* ============================================================
   Инспектор
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
```

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.1):

```rust
// ============================================================
// Итерация: length, key(i), keys, entries
// ============================================================

#[wasm_bindgen_test]
fn test_length_empty_storage() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    assert_eq!(storage::length(&ls), 0);

    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_length_after_inserts() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    storage::set(&ls, "test:len1", "a").unwrap();
    storage::set(&ls, "test:len2", "b").unwrap();
    storage::set(&ls, "test:len3", "c").unwrap();

    assert_eq!(storage::length(&ls), 3);

    storage::clear(&ls).ok();
    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_key_by_index() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    storage::set(&ls, "test:first", "1").unwrap();

    assert_eq!(storage::key(&ls, 0), Some("test:first".into()));
    assert_eq!(storage::key(&ls, 99), None);

    storage::clear(&ls).ok();
    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_keys_iteration() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    storage::set(&ls, "test:key1", "v1").unwrap();
    storage::set(&ls, "test:key2", "v2").unwrap();

    let keys = storage::keys(&ls);
    assert_eq!(keys.len(), 2);
    assert!(keys.contains(&"test:key1".to_string()));
    assert!(keys.contains(&"test:key2".to_string()));

    storage::clear(&ls).ok();
    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_keys_empty_storage() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    let keys = storage::keys(&ls);
    assert!(keys.is_empty());

    storage::clear(&ls).ok();
    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_entries_consistency() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    storage::set(&ls, "test:a", "1").unwrap();
    storage::set(&ls, "test:b", "2").unwrap();
    storage::set(&ls, "test:c", "3").unwrap();

    let entries = storage::entries(&ls);
    assert_eq!(entries.len(), 3);
    assert_eq!(entries.len() as u32, storage::length(&ls));

    for (key, value) in &entries {
        assert_eq!(storage::get(&ls, key), Some(value.clone()));
    }

    storage::clear(&ls).ok();
    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

// ============================================================
// has_key
// ============================================================

#[wasm_bindgen_test]
fn test_has_key() {
    let ls = storage::local_storage().unwrap();

    storage::set(&ls, "test:has", "value").unwrap();

    assert!(storage::has_key(&ls, "test:has"));
    assert!(!storage::has_key(&ls, "test:nonexistent"));

    storage::remove(&ls, "test:has").ok();
}

// ============================================================
// clear
// ============================================================

#[wasm_bindgen_test]
fn test_clear_removes_all() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);

    storage::set(&ls, "test:clear1", "a").unwrap();
    storage::set(&ls, "test:clear2", "b").unwrap();
    assert!(storage::length(&ls) >= 2);

    storage::clear(&ls).unwrap();
    assert_eq!(storage::length(&ls), 0);

    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

#[wasm_bindgen_test]
fn test_clear_empty_storage() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    let result = storage::clear(&ls);
    assert!(result.is_ok());
    assert_eq!(storage::length(&ls), 0);

    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_length_empty_storage` | `length` возвращает 0 для пустого хранилища |
| `test_length_after_inserts` | `length` считает вставленные ключи |
| `test_key_by_index` | `key(i)` возвращает ключ по позиции, `None` вне диапазона |
| `test_keys_iteration` | `keys()` возвращает все ключи |
| `test_keys_empty_storage` | `keys()` возвращает пустой вектор |
| `test_entries_consistency` | `entries()` согласован с `get()` и `length()` |
| `test_has_key` | `has_key` для существующих и отсутствующих ключей |
| `test_clear_removes_all` | `clear()` удаляет все ключи |
| `test_clear_empty_storage` | `clear()` на пустом хранилище не вызывает ошибок |

**Восстановление состояния.** Каждый тест сохраняет существующие ключи, очищает хранилище, работает, затем восстанавливает ключи. Это изоляция: один тест не влияет на другой.

**Всего тестов:** 7 из 15.1 + 8 новых = **15 тестов**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Проверьте сценарий:

1. **Секция 1** — введите имя, нажмите «Сохранить».
2. **Секция 2** — нажмите «Обновить». Появится таблица:

   ```
   Всего ключей: 1
   # | key        | value       | has_key
   0 | demo:name  | Иван        | ✓
   Суммарный размер: ~19 символов
   ```

3. Нажмите «Очистить всё» — таблица обновится: «Всего ключей: 0», «(хранилище пусто)».
4. Сохраните имя с эмодзи (например, `🦀 Rust`) — таблица покажет его без паники благодаря `preview_value`.

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

**Ожидаемый результат:** 15 тестов `ok` в обоих браузерах.

---

## Что важно запомнить

**1. `length` — количество ключей.** Возвращает `u32`. Для пустого хранилища — `0`.

**2. `key(i)` — ключ по индексу.** Индекс от 0 до `length - 1`. Вне диапазона — `None`. Порядок ключей зависит от браузера.

**3. `keys()` — все ключи.** Построен через `length` + `key(i)`. Возвращает `Vec<String>`.

**4. `entries()` — все пары.** Построен через `keys` + `get`. Возвращает `Vec<(String, String)>`.

**5. `has_key` — обёртка над `get().is_some()`.** Читаемость важнее микрооптимизации.

**6. `clear()` — полная очистка.** Не путать с `remove(key)`.

**7. Storage — плоский список.** Нет вложенности, нет индексов (кроме позиции), нет запросов.

**8. `Rc<dyn Fn()>` для переиспользуемых замыканий.** Дешёвое клонирование, одно замыкание в памяти.

**9. `rust-analyzer` и вложенные блоки с `return`.** Когда замыкание обёрнуто во внешний блок `{ ... Rc::new(move || { ... return; ... }) }`, анализатор может неверно вывести тип. **Решение:** вынести замыкание в отдельную функцию с явным возвращаемым типом `Rc<dyn Fn()>`.

**10. Байтовый срез vs символьный.** `&s[..n]` паникует на границе UTF-8. `s.chars().take(n)` безопасен для любой строки. Всегда используйте символьный, если работаете с произвольным текстом.

---

## Что дальше

**15.3. `localStorage` vs `sessionStorage`.** Мы разберём ключевое различие между двумя хранилищами: время жизни и область видимости. Демо получит **два счётчика рядом** — LS и SS. Перезагрузка, закрытие вкладки, открытие второй вкладки — все сценарии, где проявляется разница. Плюс расширенная подсказка про изоляцию `sessionStorage` между вкладками.