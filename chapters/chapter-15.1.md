# 15.1. Знакомство со Storage

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
│   ├── app.rs                      # демо: сохранение имени
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # 5 функций: local_storage, session_storage, get, set, remove
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
│   └── wasm_storage.rs             # 7 тестов базового API
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

- `src/storage.rs` — минимальная версия: пять функций. Работа только с базовыми операциями — `set`, `get`, `remove`. JSON-сериализация, `StorageEvent`, оценка использования появятся в следующих подглавах.
- `src/app.rs` — переписан с нуля под минимальное демо. Один input, три кнопки, один div для результата. Никаких секций, никаких `Rc<RefCell>` — только базовые элементы и обработчики.
- `src/lib.rs` — добавляется `pub mod storage;`.
- `tests/wasm_storage.rs` — 7 тестов, покрывающих базовые операции и edge cases.
- `web/index.html` и `web/styles/main.css` — упрощены: только каркас `<div id="app">` и минимальные стили.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Web Storage API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API |
| `Storage` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage |
| `Window.localStorage` | https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage |
| `Window.sessionStorage` | https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage |
| `web_sys::Storage` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.Storage.html |

---

## Что такое `localStorage` и `sessionStorage`

Web Storage — это **встроенное в браузер хранилище пар «ключ → строка»**. Данные не отправляются на сервер. Они живут внутри браузера, доступны через JavaScript и — благодаря `web_sys` — через Rust/Wasm.

Два хранилища реализуют **один и тот же интерфейс `Storage`**:

- **`localStorage`** — данные живут до явного удаления. Переживают закрытие вкладки, обновление страницы, перезапуск браузера. Доступны всем вкладкам одного origin.
- **`sessionStorage`** — данные живут до закрытия вкладки. Переживают перезагрузку (F5), но исчезают при закрытии. Изолированы между вкладками: у каждой вкладки своя копия.

**Разница — в одном предложении.** `localStorage` — общий долгоживущий склад. `sessionStorage` — личная сессионная папка.

В этой подглаве мы работаем **только с `localStorage`**. Разницу между двумя хранилищами разберём в 15.3.

---

## Три базовые операции

Интерфейс `Storage` содержит много методов, но **три из них — самые важные**:

| Операция | Метод | Что делает |
|---|---|---|
| Сохранить | `set_item(key, value)` | Кладёт строку `value` под ключом `key` |
| Прочитать | `get_item(key)` | Возвращает строку или `null`, если ключа нет |
| Удалить | `remove_item(key)` | Удаляет ключ |

В `web_sys` эти методы возвращают `Result` или `Option`:

- `set_item(key, value)` → `Result<(), JsValue>`. Может провалиться с `QuotaExceededError`.
- `get_item(key)` → `Result<Option<String>, JsValue>`. `Ok(None)` — ключа нет.
- `remove_item(key)` → `Result<(), JsValue>`. Ошибка маловероятна.

Мы оборачиваем их в свои функции с более простыми сигнатурами, чтобы вызывающий код не думал о двух уровнях `Result<Option<...>>`.

---

## `src/lib.rs`

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

## `src/storage.rs`

```rust
//! Помощники для работы с Web Storage API.
//!
//! ## Что в этом модуле
//!
//! - `local_storage` / `session_storage` — доступ к хранилищам.
//! - `get` / `set` / `remove` — базовые операции.
//!
//! ## Что будет дальше
//!
//! В следующих подглавах модуль расширится:
//! - 15.2 — `length`, `key(i)`, `keys`, `entries`, `has_key`, `clear`.
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
///
/// # Ошибки
///
/// Возвращает `Err`, если окно недоступно или браузер запретил
/// доступ к `sessionStorage`.
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
```

**Ключевые моменты:**

- **`local_storage` и `session_storage`** возвращают `Result<Storage, JsValue>`. Если браузер запретил доступ — `Err`.
- **`get` возвращает `Option<String>`**, а не `Result<Option<String>, JsValue>`. Ошибки чтения маловероятны и скрываются через `.ok()`. Для базового API это удобно; в подглаве 15.4 мы покажем, как проверять доступность явно.
- **`set` возвращает `Result`**, потому что `QuotaExceededError` — реальный сценарий.
- **`remove` возвращает `Result`**, хотя ошибка при удалении почти невозможна. Единообразие важнее.

---

## `src/app.rs`

```rust
//! Логика приложения: «Знакомство со Storage».
//!
//! Минимальное демо: сохранить имя в localStorage,
//! прочитать его обратно, удалить.
//!
//! Компонент:
//!   <div id="app">
//!     <h1>Storage: знакомство</h1>
//!     <input type="text" placeholder="Введите имя...">
//!     <button>Сохранить</button>
//!     <button>Загрузить</button>
//!     <button>Удалить</button>
//!     <div class="result">...</div>
//!   </div>

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

    let heading = dom::create_heading(1, "Storage: знакомство")?;
    dom::append(&container, &heading)?;

    let description = dom::create_paragraph(
        "Введите имя, сохраните его в localStorage, перезагрузите страницу \
         и загрузите обратно. Данные переживут перезагрузку.",
    )?;
    dom::add_class(&dom::as_html(&description), "hint")?;
    dom::append(&container, &description)?;

    // Поле ввода
    let input_el = dom::create_element("input")?;
    dom::set_attribute(&input_el, "type", "text")?;
    dom::set_attribute(&input_el, "id", "name-input")?;
    dom::set_attribute(&input_el, "placeholder", "Введите имя...")?;
    let input: HtmlInputElement = input_el.dyn_into()?;
    dom::append(&container, &input)?;

    // Кнопки
    let save_btn = dom::create_element("button")?;
    save_btn.set_text_content(Some("Сохранить"));
    let load_btn = dom::create_element("button")?;
    load_btn.set_text_content(Some("Загрузить"));
    let remove_btn = dom::create_element("button")?;
    remove_btn.set_text_content(Some("Удалить"));

    dom::append(&container, &save_btn)?;
    dom::append(&container, &load_btn)?;
    dom::append(&container, &remove_btn)?;

    // Область результата
    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "result")?;
    result.set_text_content(Some("(ещё ничего не сделано)"));
    dom::append(&container, &result)?;

    // Хранилище Closure — живёт до конца `run`
    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Обработчик «Сохранить» ----
    {
        let input = input.clone();
        let result = result.clone();
        callbacks::listen(
            &save_btn,
            "click",
            move |_event| {
                let value = input.value();

                if value.trim().is_empty() {
                    result.set_text_content(Some("Введите имя перед сохранением"));
                    log!("Пустое значение — сохранение пропущено");
                    return;
                }

                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        result.set_text_content(Some("localStorage недоступен"));
                        log!("localStorage недоступен");
                        return;
                    }
                };

                match storage::set(&ls, KEY_NAME, &value) {
                    Ok(_) => {
                        let text = format!("Сохранено: \"{}\"", value);
                        result.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                    Err(e) => {
                        let text = format!("Ошибка сохранения: {:?}", e);
                        result.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                }
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Загрузить» ----
    {
        let result = result.clone();
        callbacks::listen(
            &load_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        result.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                match storage::get(&ls, KEY_NAME) {
                    Some(value) => {
                        let text = format!("Загружено: \"{}\"", value);
                        result.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                    None => {
                        result.set_text_content(Some(
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
        let result = result.clone();
        callbacks::listen(
            &remove_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        result.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                match storage::remove(&ls, KEY_NAME) {
                    Ok(_) => {
                        result.set_text_content(Some(
                            "Ключ удалён из localStorage",
                        ));
                        log!("Ключ {} удалён", KEY_NAME);
                    }
                    Err(e) => {
                        let text = format!("Ошибка удаления: {:?}", e);
                        result.set_text_content(Some(text.as_str()));
                        log!("{}", text);
                    }
                }
            },
            &closures,
        )?;
    }

    // Проверяем доступность localStorage при старте
    match storage::local_storage() {
        Ok(_) => log!("localStorage доступен"),
        Err(_) => log!("localStorage НЕ доступен"),
    }

    // `closures` должен жить, пока работают обработчики.
    // В учебном примере — до конца работы страницы.
    std::mem::forget(closures);

    Ok(())
}
```

**Что демонстрирует `app.rs`:**

- **Один input**, в который пользователь вводит имя.
- **Три кнопки**: «Сохранить» кладёт значение в `localStorage` под ключом `demo:name`; «Загрузить» читает; «Удалить» стирает.
- **Один div результата**, в котором отображается, что произошло.
- **Проверка доступности `localStorage`** при старте.

**Что важно понять:**

- **Каждый обработчик — в своём блоке `{ }`.** Внутри блока создаются копии `input` и `result` через `.clone()`. Эти копии уходят в `move`-замыкание, а **оригиналы** `input` и `result` остаются живы и доступны для следующих блоков.
- **`web_sys::Element` не реализует `Copy`.** Клонирование явное, через `.clone()`. Клонирование дешёвое: создаётся новая JS-ссылка на тот же DOM-узел.
- **Обработчики используют `callbacks::listen`**, а не `add_event_listener_with_callback` напрямую. `Closure` сохраняется в контейнере `EventClosures`, живёт до конца работы страницы.
- **Никакого `Rc<RefCell<...>>`** для общего состояния — не нужно. Каждый обработчик независим.

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — знакомство со Storage</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Знакомство со Storage. Откройте консоль для логов.</p>

    <div id="app"></div>

    <script type="module" src="/main.js"></script>
</body>
</html>
```

---

## `web/styles/main.css`

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
    max-width: 720px;
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
}

button:hover {
    background: #b7410e;
    color: white;
}

button:active {
    transform: scale(0.96);
}

/* ============================================================
   Результат
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
}
```

---

## `tests/wasm_storage.rs`

```rust
//! Тесты для базового Web Storage API.
//!
//! Покрытие:
//! - Доступность localStorage и sessionStorage.
//! - Три базовые операции: set, get, remove.
//! - Edge cases: отсутствующий ключ, пустая строка, Unicode.
use wasm_bindgen_test::*;

use hello_wasm02::storage;

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// Доступность хранилищ
// ============================================================

#[wasm_bindgen_test]
fn test_local_storage_available() {
    let ls = storage::local_storage();
    assert!(ls.is_ok(), "localStorage should be available");
}

#[wasm_bindgen_test]
fn test_session_storage_available() {
    let ss = storage::session_storage();
    assert!(ss.is_ok(), "sessionStorage should be available");
}

// ============================================================
// Базовые операции
// ============================================================

#[wasm_bindgen_test]
fn test_set_and_get() {
    let ls = storage::local_storage().unwrap();

    storage::set(&ls, "test:set_get", "hello").unwrap();
    assert_eq!(storage::get(&ls, "test:set_get"), Some("hello".into()));

    storage::remove(&ls, "test:set_get").unwrap();
    assert_eq!(storage::get(&ls, "test:set_get"), None);
}

#[wasm_bindgen_test]
fn test_get_missing_key() {
    let ls = storage::local_storage().unwrap();

    // Убеждаемся, что ключа нет
    storage::remove(&ls, "test:missing").ok();

    assert_eq!(storage::get(&ls, "test:missing"), None);
}

#[wasm_bindgen_test]
fn test_remove() {
    let ls = storage::local_storage().unwrap();

    storage::set(&ls, "test:remove", "value").unwrap();
    assert_eq!(storage::get(&ls, "test:remove"), Some("value".into()));

    storage::remove(&ls, "test:remove").unwrap();
    assert_eq!(storage::get(&ls, "test:remove"), None);
}

// ============================================================
// Edge cases
// ============================================================

#[wasm_bindgen_test]
fn test_set_empty_string() {
    let ls = storage::local_storage().unwrap();

    storage::set(&ls, "test:empty", "").unwrap();
    assert_eq!(storage::get(&ls, "test:empty"), Some("".into()));

    storage::remove(&ls, "test:empty").ok();
}

#[wasm_bindgen_test]
fn test_set_unicode_string() {
    let ls = storage::local_storage().unwrap();

    let unicode = "Привет, мир! 🦀 Rust + Wasm = ❤";
    storage::set(&ls, "test:unicode", unicode).unwrap();

    assert_eq!(
        storage::get(&ls, "test:unicode"),
        Some(unicode.to_string())
    );

    storage::remove(&ls, "test:unicode").ok();
}
```

**Что проверяют тесты:**

| Тест | Что проверяет |
|---|---|
| `test_local_storage_available` | `localStorage` доступен в браузере |
| `test_session_storage_available` | `sessionStorage` доступен в браузере |
| `test_set_and_get` | `set` кладёт значение, `get` читает |
| `test_get_missing_key` | `get` возвращает `None` для отсутствующего ключа |
| `test_remove` | `remove` удаляет ключ |
| `test_set_empty_string` | пустая строка — валидное значение |
| `test_set_unicode_string` | Unicode и эмодзи сохраняются корректно |

**7 тестов, минимум для базового API.**

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Введите имя, нажмите «Сохранить». Перезагрузите страницу (F5). Нажмите «Загрузить» — имя вернётся. Нажмите «Удалить» — ключ исчезнет.

**Что проверить в DevTools** (Application → Storage → Local Storage → `http://127.0.0.1:5173`):

- При сохранении появляется ключ `demo:name` со значением.
- При удалении ключ исчезает.
- После перезагрузки страницы ключ остаётся на месте.

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

**Ожидаемый результат:** 7 тестов `ok` в обоих браузерах.

---

## Что важно запомнить

**1. `localStorage` и `sessionStorage` реализуют один интерфейс.** Разница — только во времени жизни и области видимости. В этой подглаве мы работаем с `localStorage`.

**2. Три операции — основа.** `set_item`, `get_item`, `remove_item` покрывают 90% задач.

**3. `get` возвращает `Option<String>`.** `None` означает, что ключа нет — это не ошибка.

**4. `set` возвращает `Result`.** Может провалиться с `QuotaExceededError` при переполнении. Обрабатывайте ошибку явно.

**5. Данные переживают перезагрузку.** Это ключевое отличие `localStorage` от обычных переменных в памяти Rust или JavaScript.

**6. Только строки.** `Storage` хранит строки. Числа, булевы, объекты — через сериализацию. Это тема подглавы 15.5.

**7. Проверяйте доступность.** В приватном режиме некоторых браузеров `localStorage` может быть недоступен. Мы это делаем неявно через `Result`. В 15.4 — сделаем явно.

**8. `Element` не реализует `Copy`.** Если элемент используется в нескольких обработчиках, каждый блок должен получить свою копию через `.clone()`. Оригинал остаётся доступен для следующих блоков. Клонирование `Element` дешёвое — создаётся JS-ссылка, DOM-узел не дублируется.

---

## Что дальше

**15.2. Полный API Storage.** Мы расширим `storage.rs` с 5 до 12 функций: добавим `length`, `key(i)`, `keys`, `entries`, `has_key`, `clear`. Демо превратится из простой формы в **инспектор хранилища**, который показывает все ключи в виде таблицы. Это откроет, как Storage устроен внутри: плоский список пар «ключ → значение», без вложенности и индексов.