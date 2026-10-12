# 15.6. JSON: структуры в Storage

## Обновлённая структура проекта

```text
hello-wasm02/
├── Cargo.toml                      # +serde, +serde_json
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
│   ├── app.rs                      # +1 секция: «Заметки»
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # +2 функции: set_json, get_json
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
│   └── wasm_storage.rs             # +10 тестов → 41 тест
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +несколько правил для заметок
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `Cargo.toml` — добавляются `serde` (с feature `derive`) и `serde_json`. `serde` даёт макросы `Serialize`/`Deserialize`, `serde_json` — функции `to_string`/`from_str`.
- `src/storage.rs` — **+2 функции**: `set_json` и `get_json`. Обёртки над `serde_json` с `Storage::set_item` и `get_item`.
- `src/app.rs` — **+1 секция «Заметки»**. Список заметок как `Vec<Note>`, поле ввода, кнопки «Добавить», «Очистить все», «Обновить JSON». Preview сырого JSON в `<pre>`.
- `tests/wasm_storage.rs` — **+10 тестов** на структуры, `Option`, `Vec`, невалидный JSON, отсутствующий ключ. Всего 41 тест.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Serde (официальный сайт) | https://serde.rs/ |
| `serde::Serialize` | https://docs.rs/serde/latest/serde/trait.Serialize.html |
| `serde::Deserialize` | https://docs.rs/serde/latest/serde/trait.Deserialize.html |
| `serde_json::to_string` | https://docs.rs/serde_json/latest/serde_json/fn.to_string.html |
| `serde_json::from_str` | https://docs.rs/serde_json/latest/serde_json/fn.from_str.html |
| `#[derive(Serialize, Deserialize)]` | https://serde.rs/derive.html |

---

## Зачем нужна JSON-сериализация

Web Storage хранит **только строки**. Числа и булевы мы разобрали в 15.5 — они тривиально преобразуются через `to_string()` и `parse()`. Но что делать со **сложными структурами**?

**`Vec<Note>`** — список заметок. **`Settings`** — объект с несколькими полями. **Вложенные структуры** — заметка с автором, датой, тегами. Ничего из этого нельзя преобразовать в строку тривиально.

**Решение — JSON.** Это текстовый формат, который:
- **Представляет любую структуру**: объекты, массивы, вложенность.
- **Читаем глазами**: пользователь может открыть DevTools и увидеть реальные данные.
- **Стандартизован**: одинаково работает в Rust, JavaScript, Python и любом другом языке.
- **Поддерживается serde**: derive-макросы автоматически генерируют код сериализации.

**Как это работает:**

```
Rust-структура             Строка в Storage                Rust-структура
┌──────────────┐          ┌──────────────────────────┐    ┌──────────────┐
│ Note {       │  to_string│ [{"id":1,"text":"Купить"}│ from_str│ Note {       │
│   id: 1,     │ ────────► │  ,{"id":2,"text":"Позвон"│ ────────► │   id: 1,     │
│   text: "...",│          │  }]                       │          │   text: "..."│
│ }            │          │                          │          │ }            │
└──────────────┘          └──────────────────────────┘    └──────────────┘
       ▲                                                            ▲
       │ serde_json::to_string                                      │
       │                                                            │
       └──────────── serde_json::from_str ─────────────────────────┘
```

**Что важно понять:**

- **JSON-строка — это тоже строка.** Storage не отличает её от `"42"` или `"Привет"`. Но внутри неё — структурированные данные.
- **`serde` генерирует код.** Макросы `#[derive(Serialize, Deserialize)]` автоматически создают функции преобразования. Вам не нужно писать это вручную.
- **Обработка ошибок обязательна.** JSON может быть повреждён: пользователь изменит в DevTools, программа обновится с новой схемой. `from_str` возвращает `Result`.

---

## `Cargo.toml`

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

**Что добавилось:**

- **`serde = { version = "1", features = ["derive"] }`** — сам crate + feature `derive` для макросов.
- **`serde_json = "1"`** — функции `to_string` и `from_str`.

**Почему две зависимости, а не одна.** `serde` предоставляет трейты и макросы, но не знает о конкретных форматах. `serde_json` — это реализация для JSON. Есть также `serde_yaml`, `serde_cbor`, `bincode` — на случай, если понадобится другой формат.

---

## `src/storage.rs`

Показаны **только новые функции**. Остальное содержание — из 15.4 (базовые операции, итерация, `is_storage_writable`).

Добавьте в начало файла импорты и в конец — две функции:

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
//! - `set_json` / `get_json` — сериализация структур через serde_json.
//!
//! ## Что будет дальше
//!
//! В следующих подглавах модуль расширится:
//! - 15.7 — `parse_storage_event`, `format_storage_change`.
//! - 15.8 — `fill_to_quota`, `measure_sync_block`, `current_origin`.
#![allow(dead_code)]

use serde::de::DeserializeOwned;
use serde::Serialize;
use wasm_bindgen::JsValue;
use web_sys::Storage;

// ... код из 15.1–15.4 ...

// ============================================================
// JSON-сериализация
// ============================================================

/// Сохраняет значение как JSON.
///
/// `T` должен реализовать `Serialize`. Обычно это достигается
/// через `#[derive(Serialize)]` на структуре.
///
/// # Ошибки
///
/// Возвращает `Err`, если:
/// - Сериализация в JSON провалилась (редкий случай — структура
///   содержит что-то, что serde не умеет кодировать).
/// - Запись в Storage провалилась (`QuotaExceededError`).
///
/// # Пример
///
/// ```no_run
/// use hello_wasm02::storage;
/// use serde::Serialize;
///
/// #[derive(Serialize)]
/// struct Config { theme: String }
///
/// let ls = storage::local_storage().unwrap();
/// let config = Config { theme: "dark".into() };
/// storage::set_json(&ls, "config", &config).unwrap();
/// ```
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
/// `T` должен реализовать `DeserializeOwned`. Обычно это
/// достигается через `#[derive(Deserialize)]` на структуре.
///
/// Возвращает `None`, если:
/// - Ключа нет в хранилище.
/// - Значение не является валидным JSON.
/// - JSON валиден, но не соответствует схеме `T`.
///
/// # Пример
///
/// ```no_run
/// use hello_wasm02::storage;
/// use serde::Deserialize;
///
/// #[derive(Deserialize)]
/// struct Config { theme: String }
///
/// let ls = storage::local_storage().unwrap();
/// let config: Option<Config> = storage::get_json(&ls, "config");
/// ```
pub fn get_json<T: DeserializeOwned>(
    storage: &Storage,
    key: &str,
) -> Option<T> {
    let raw = get(storage, key)?;
    return serde_json::from_str(&raw).ok();
}
```

**Ключевые моменты:**

- **`set_json` — обёртка.** Сериализует в строку через `serde_json::to_string`, затем вызывает `set`. Ошибка сериализации конвертируется в `JsValue` через `.map_err`.
- **`get_json` — обёртка.** Читает строку через `get`, десериализует через `serde_json::from_str`. Все ошибки (`None` от `get`, `Err` от `from_str`) превращаются в `None` через `.ok()`.
- **`T: Serialize` vs `T: DeserializeOwned`.** `DeserializeOwned` — это сокращение для `for<'de> Deserialize<'de>`. Означает, что тип может быть десериализован из любых данных, а не только из заимствованных. Для `get_json` это правильно — мы не привязываем `T` к времени жизни исходной строки.

**Почему `get_json` не возвращает `Result`.**

Разница между двумя подходами:

| Возвращает | Плюсы | Минусы |
|---|---|---|
| `Result<T, JsValue>` | Точная информация об ошибке | Вызывающий код должен обрабатывать три случая: ключ не найден, JSON невалиден, JSON не той схемы |
| `Option<T>` | Просто: есть или нет | Теряется информация о причине |

Для большинства сценариев **«ключа нет»** и **«JSON повреждён»** обрабатываются одинаково: используем значение по умолчанию. `Option` это отражает. Если нужна детальная диагностика — используйте `serde_json::from_str` напрямую.

**Сводка `storage.rs` после 15.6:**

| Функция | Появилась в | Что делает |
|---|---|---|
| `local_storage` | 15.1 | Доступ к localStorage |
| `session_storage` | 15.1 | Доступ к sessionStorage |
| `get` | 15.1 | Чтение строки |
| `set` | 15.1 | Запись строки |
| `remove` | 15.1 | Удаление ключа |
| `length` | 15.2 | Количество ключей |
| `key` | 15.2 | Ключ по индексу |
| `keys` | 15.2 | Все ключи |
| `entries` | 15.2 | Все пары |
| `has_key` | 15.2 | Проверка наличия |
| `clear` | 15.2 | Полная очистка |
| `is_storage_writable` | 15.4 | Проверка записи |
| **`set_json`** | **15.6** | **Сериализация в JSON** |
| **`get_json`** | **15.6** | **Десериализация из JSON** |

**14 функций.**

---

## `src/app.rs`

Показаны новые части. Остальное — из 15.5.

### Новый импорт

Добавьте в раздел импортов:

```rust
use serde::{Deserialize, Serialize};
```

### Модель данных

Добавьте **до** `pub fn run()`:

```rust
/// Заметка, сохраняемая в localStorage как JSON.
///
/// `Serialize` и `Deserialize` генерируются макросом `derive`.
/// Без них `set_json` и `get_json` не скомпилируются.
#[derive(Serialize, Deserialize, Clone, Debug)]
struct Note {
    id: u32,
    text: String,
}
```

### Новая секция 5: «Заметки»

Добавьте **после секции 4** (Типы) и **перед** `std::mem::forget(closures)`:

```rust
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

    // ---- Поле ввода новой заметки ----
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

    // ---- Кнопки ----
    let add_note_btn = dom::create_element("button")?;
    add_note_btn.set_text_content(Some("Добавить заметку"));
    let clear_notes_btn = dom::create_element("button")?;
    clear_notes_btn.set_text_content(Some("Очистить все"));
    let refresh_json_btn = dom::create_element("button")?;
    refresh_json_btn.set_text_content(Some("Обновить JSON"));

    dom::append(&section5, &add_note_btn)?;
    dom::append(&section5, &clear_notes_btn)?;
    dom::append(&section5, &refresh_json_btn)?;

    // ---- Список заметок ----
    let notes_list = dom::create_element("ul")?;
    dom::add_class(&dom::as_html(&notes_list), "notes-list")?;
    dom::append(&section5, &notes_list)?;

    // ---- Preview сырого JSON ----
    let json_label = dom::create_element("div")?;
    json_label.set_text_content(Some("Сырой JSON в localStorage[demo:notes]:"));
    dom::add_class(&dom::as_html(&json_label), "label")?;
    dom::append(&section5, &json_label)?;

    let json_preview = dom::create_element("pre")?;
    dom::add_class(&dom::as_html(&json_preview), "json-preview")?;
    json_preview.set_text_content(Some("(ключа нет — добавьте заметку)"));
    dom::append(&section5, &json_preview)?;

    // ---- Замыкание для отрисовки списка заметок ----
    let render_notes: Rc<dyn Fn()> = make_render_notes(notes_list.clone());

    // ---- Замыкание для отрисовки JSON-preview ----
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
                    Err(_) => {
                        log!("localStorage недоступен");
                        return;
                    }
                };

                // Читаем существующие заметки или начинаем с пустого Vec
                let mut notes: Vec<Note> =
                    storage::get_json(&ls, KEY_NOTES).unwrap_or_default();

                // Генерируем id: max + 1
                let next_id: u32 = notes.iter().map(|n| n.id).max().unwrap_or(0) + 1;
                notes.push(Note {
                    id: next_id,
                    text: text.clone(),
                });

                // Сохраняем обратно
                match storage::set_json(&ls, KEY_NOTES, &notes) {
                    Ok(_) => log!("Заметка #{} добавлена: {}", next_id, text),
                    Err(e) => log!("Ошибка сохранения заметок: {:?}", e),
                }

                // Очищаем поле ввода и обновляем UI
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

    // ---- Начальная отрисовка ----
    render_notes();
    render_json_preview();
```

### Константа для ключа

Добавьте **рядом с другими константами** в начале `app.rs`:

```rust
const KEY_NOTES: &str = "demo:notes";
```

### Вспомогательные функции

Добавьте **в конец файла**, рядом с `build_section`, `make_render_inspector`, `preview_value`:

```rust
/// Создаёт замыкание для отрисовки списка заметок.
///
/// Читает `Vec<Note>` из localStorage, очищает `<ul>` и наполняет
/// его `<li>` — по одному на заметку.
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

        log!("Заметки: список отрисован");
    });
}

/// Создаёт замыкание для отрисовки сырого JSON из localStorage.
///
/// Показывает ровно то, что лежит под ключом `demo:notes`,
/// без парсинга. Это позволяет пользователю увидеть сырую строку.
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

        // Форматируем JSON для читаемости: parse + pretty-print.
        // Если JSON повреждён — показываем как есть.
        let display: String = match serde_json::from_str::<serde_json::Value>(&raw) {
            Ok(value) => serde_json::to_string_pretty(&value)
                .unwrap_or_else(|_| raw.clone()),
            Err(_) => format!("(невалидный JSON)\n{}", raw),
        };

        json_preview.set_text_content(Some(display.as_str()));
    });
}
```

**Ключевые моменты:**

- **`Note` — простая структура.** Два поля: `id: u32`, `text: String`. Достаточно для демонстрации.
- **`#[derive(Serialize, Deserialize, Clone, Debug)]`** — генерирует код сериализации. Без этих derive `set_json` и `get_json` не скомпилируются.
- **`Vec<Note>` — массив структур.** Читается одной операцией `get_json::<Vec<Note>>`. Сохраняется одной операцией `set_json`.
- **`unwrap_or_default()`** — если ключа нет или JSON повреждён, получаем пустой `Vec`. Это безопасный fallback.
- **`next_id` — max + 1.** Если заметки есть, `id` продолжает последовательность. Если пусто — `1`.
- **`render_notes` и `render_json_preview` — вынесены в отдельные функции.** Это обходит проблему `rust-analyzer` с ранним `return` внутри вложенных блоков.
- **JSON-preview форматируется через `to_string_pretty`.** Пользователь видит структурированный JSON, а не однострочную строку. Если JSON повреждён — показываем как есть с пометкой.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   Секция заметок
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

.notes-list li:only-child:contains("заметок нет") {
    color: #666;
    font-style: italic;
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

.label {
    font-size: 0.85rem;
    color: #666;
    margin-top: 0.75rem;
}
```

**Что делает:**

- **`.notes-list`** — стандартный список. `min-height` для случая «(заметок нет)».
- **`.json-preview`** — моноширинный блок с синей полосой слева, чтобы отличать от других `pre`.
- **`.label`** — серая подпись над JSON-preview.

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.5):

```rust
// ============================================================
// JSON: сериализация структур
// ============================================================

#[wasm_bindgen_test]
fn test_set_get_json_simple_struct() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct SimpleData {
        name: String,
        count: u32,
    }

    let ls = storage::local_storage().unwrap();

    let original = SimpleData {
        name: "Rust".into(),
        count: 42,
    };

    storage::set_json(&ls, "test:json_simple", &original).unwrap();

    let retrieved: SimpleData = storage::get_json(&ls, "test:json_simple").unwrap();
    assert_eq!(retrieved, original);

    storage::remove(&ls, "test:json_simple").ok();
}

#[wasm_bindgen_test]
fn test_set_get_json_vec() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct Item {
        id: u32,
        label: String,
    }

    let ls = storage::local_storage().unwrap();

    let original = vec![
        Item { id: 1, label: "один".into() },
        Item { id: 2, label: "два".into() },
        Item { id: 3, label: "три".into() },
    ];

    storage::set_json(&ls, "test:json_vec", &original).unwrap();

    let retrieved: Vec<Item> = storage::get_json(&ls, "test:json_vec").unwrap();
    assert_eq!(retrieved, original);
    assert_eq!(retrieved.len(), 3);

    storage::remove(&ls, "test:json_vec").ok();
}

#[wasm_bindgen_test]
fn test_set_get_json_empty_vec() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct Item {
        id: u32,
    }

    let ls = storage::local_storage().unwrap();

    let original: Vec<Item> = Vec::new();
    storage::set_json(&ls, "test:json_empty_vec", &original).unwrap();

    let retrieved: Vec<Item> = storage::get_json(&ls, "test:json_empty_vec").unwrap();
    assert!(retrieved.is_empty());

    storage::remove(&ls, "test:json_empty_vec").ok();
}

#[wasm_bindgen_test]
fn test_set_get_json_option_none() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct WithOption {
        name: String,
        description: Option<String>,
    }

    let ls = storage::local_storage().unwrap();

    let original = WithOption {
        name: "item".into(),
        description: None,
    };

    storage::set_json(&ls, "test:json_opt_none", &original).unwrap();

    let retrieved: WithOption = storage::get_json(&ls, "test:json_opt_none").unwrap();
    assert_eq!(retrieved, original);
    assert!(retrieved.description.is_none());

    storage::remove(&ls, "test:json_opt_none").ok();
}

#[wasm_bindgen_test]
fn test_set_get_json_option_some() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct WithOption {
        name: String,
        description: Option<String>,
    }

    let ls = storage::local_storage().unwrap();

    let original = WithOption {
        name: "item".into(),
        description: Some("описание".into()),
    };

    storage::set_json(&ls, "test:json_opt_some", &original).unwrap();

    let retrieved: WithOption = storage::get_json(&ls, "test:json_opt_some").unwrap();
    assert_eq!(retrieved, original);
    assert_eq!(retrieved.description, Some("описание".into()));

    storage::remove(&ls, "test:json_opt_some").ok();
}

#[wasm_bindgen_test]
fn test_set_get_json_nested_struct() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct Inner {
        value: i32,
    }

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct Outer {
        name: String,
        inner: Inner,
    }

    let ls = storage::local_storage().unwrap();

    let original = Outer {
        name: "outer".into(),
        inner: Inner { value: 42 },
    };

    storage::set_json(&ls, "test:json_nested", &original).unwrap();

    let retrieved: Outer = storage::get_json(&ls, "test:json_nested").unwrap();
    assert_eq!(retrieved, original);
    assert_eq!(retrieved.inner.value, 42);

    storage::remove(&ls, "test:json_nested").ok();
}

#[wasm_bindgen_test]
fn test_get_json_missing_key() {
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct Unused {
        #[allow(dead_code)]
        field: String,
    }

    let ls = storage::local_storage().unwrap();
    storage::remove(&ls, "test:json_missing").ok();

    let result: Option<Unused> = storage::get_json(&ls, "test:json_missing");
    assert!(result.is_none());
}

#[wasm_bindgen_test]
fn test_get_json_invalid_syntax() {
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct Unused {
        #[allow(dead_code)]
        field: String,
    }

    let ls = storage::local_storage().unwrap();

    // Не валидный JSON вообще
    storage::set(&ls, "test:json_invalid", "not json at all").unwrap();

    let result: Option<Unused> = storage::get_json(&ls, "test:json_invalid");
    assert!(result.is_none(), "invalid JSON syntax should return None");

    storage::remove(&ls, "test:json_invalid").ok();
}

#[wasm_bindgen_test]
fn test_get_json_wrong_schema() {
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct Expected {
        #[allow(dead_code)]
        required_field: String,
    }

    let ls = storage::local_storage().unwrap();

    // Валидный JSON, но не подходит под схему
    storage::set(&ls, "test:json_schema", r#"{"other_field": 42}"#).unwrap();

    let result: Option<Expected> = storage::get_json(&ls, "test:json_schema");
    assert!(result.is_none(), "wrong schema should return None");

    storage::remove(&ls, "test:json_schema").ok();
}

#[wasm_bindgen_test]
fn test_json_roundtrip_preserves_unicode() {
    use serde::{Deserialize, Serialize};

    #[derive(Serialize, Deserialize, PartialEq, Debug)]
    struct TextHolder {
        text: String,
    }

    let ls = storage::local_storage().unwrap();

    let original = TextHolder {
        text: "Привет, мир! 🦀 Rust + JSON = ❤".into(),
    };

    storage::set_json(&ls, "test:json_unicode", &original).unwrap();

    let retrieved: TextHolder = storage::get_json(&ls, "test:json_unicode").unwrap();
    assert_eq!(retrieved, original);
    assert!(retrieved.text.contains("🦀"));

    storage::remove(&ls, "test:json_unicode").ok();
}

#[wasm_bindgen_test]
fn test_json_stored_as_string_in_storage() {
    use serde::Serialize;

    #[derive(Serialize)]
    struct Data {
        value: i32,
    }

    let ls = storage::local_storage().unwrap();

    let original = Data { value: 42 };
    storage::set_json(&ls, "test:json_raw", &original).unwrap();

    // Читаем сырую строку — она должна быть валидным JSON,
    // а не Rust-представлением структуры
    let raw = storage::get(&ls, "test:json_raw").unwrap();
    assert!(raw.starts_with('{'));
    assert!(raw.contains("\"value\""));
    assert!(raw.contains("42"));

    storage::remove(&ls, "test:json_raw").ok();
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_set_get_json_simple_struct` | Простая структура с двумя полями |
| `test_set_get_json_vec` | `Vec<Item>` из трёх элементов |
| `test_set_get_json_empty_vec` | Пустой `Vec` |
| `test_set_get_json_option_none` | Поле `Option::None` |
| `test_set_get_json_option_some` | Поле `Option::Some` |
| `test_set_get_json_nested_struct` | Вложенная структура `Outer { inner: Inner }` |
| `test_get_json_missing_key` | Отсутствующий ключ → `None` |
| `test_get_json_invalid_syntax` | `"not json at all"` → `None` |
| `test_get_json_wrong_schema` | Валидный JSON, но не та схема → `None` |
| `test_json_roundtrip_preserves_unicode` | Unicode и эмодзи в JSON |
| `test_json_stored_as_string_in_storage` | В хранилище лежит именно JSON-строка |

**Всего тестов:** 31 из 15.1–15.5 + 10 новых = **41 тест**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Прокрутите до секции 5 «Заметки».

**Сценарий 1: добавление заметок.**

1. Введите текст — «Купить молоко» — нажмите «Добавить заметку».
2. Список покажет `#1: Купить молоко`.
3. Введите вторую — «Позвонить маме» — нажмите «Добавить».
4. Список покажет две заметки.
5. Откройте DevTools → Application → Local Storage → `demo:notes`.

   Увидите **одну строку** JSON:

   ```json
   [{"id":1,"text":"Купить молоко"},{"id":2,"text":"Позвонить маме"}]
   ```

6. В UI блок JSON-preview покажет этот же JSON, но отформатированный:

   ```json
   [
     {
       "id": 1,
       "text": "Купить молоко"
     },
     {
       "id": 2,
       "text": "Позвонить маме"
     }
   ]
   ```

**Сценарий 2: ручное изменение JSON в DevTools.**

1. В DevTools измените `demo:notes` на:

   ```json
   [{"id":10,"text":"Изменённая вручную"},{"id":11,"text":"Ещё одна"}]
   ```

2. В UI нажмите **«Обновить JSON»** — увидите новый JSON.
3. Нажмите **«Добавить заметку»** и введите «Третья».
4. Список покажет три заметки: `#10`, `#11`, `#12`. Приложение прочитало ручные изменения и продолжило с `max(id)+1`.

**Сценарий 3: повреждение JSON.**

1. В DevTools измените `demo:notes` на `not json at all`.
2. Нажмите **«Обновить JSON»** — UI покажет:

   ```
   (невалидный JSON)
   not json at all
   ```

3. Список заметок покажет `(заметок нет)` — потому что `get_json` вернул `None`, а `unwrap_or_default` дал пустой `Vec`.
4. Добавьте новую заметку — она начнёт список заново с `#1`.

**Сценарий 4: пустое значение.**

1. Нажмите **«Очистить все»**.
2. В DevTools ключ `demo:notes` исчезнет.
3. JSON-preview покажет `(ключа нет — добавьте заметку)`.

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

**Ожидаемый результат:** 41 тест `ok` в обоих браузерах.

---

## Что важно запомнить

**1. `serde` — стандарт для сериализации в Rust.** Макросы `#[derive(Serialize, Deserialize)]` генерируют весь необходимый код. Вам не нужно писать это вручную.

**2. `serde_json` — реализация для JSON.** Отдельный crate. Можно было бы использовать `serde_yaml`, `serde_cbor` или `bincode` — но JSON универсальнее.

**3. `set_json` и `get_json` — обёртки.** Минимальный код, который скрывает `serde_json` и `Storage` за простым интерфейсом.

**4. `Vec<Note>` — это одна строка в Storage.** JSON-массив. В хранилище лежит `[{"id":1,...},{"id":2,...}]`. Одно чтение — весь массив.

**5. `get_json` возвращает `Option<T>`.** `None` означает «ключа нет **или** JSON невалиден **или** схема не совпадает». Три разные причины, одинаковое поведение.

**6. `unwrap_or_default()` — безопасный fallback.** Если данных нет — пустой `Vec`. Не паникуем, не показываем ошибку — просто начинаем с чистого листа.

**7. Unicode сохраняется.** JSON — это UTF-8. Эмодзи и кириллица работают без проблем.

**8. JSON-preview нагляден.** Пользователь видит структурированный JSON, а не однострочную строку. Это помогает понять, что именно лежит в Storage.

**9. Пользователь может ломать данные.** Это **фича**, а не баг. Ручное изменение JSON в DevTools показывает, как приложение обрабатывает некорректные данные.

**10. JSON не подходит для бинарных данных.** Если нужно сохранить `Vec<u8>` или изображение — JSON раздувает размер в 4–6 раз. Для бинарных данных — base64 или IndexedDB.

**11. JSON не подходит для очень больших структур.** Если `Vec<Note>` содержит миллион элементов — JSON-строка будет гигабайты. Storage не примет. Для больших данных — IndexedDB (глава 24).

---

## Что дальше

**15.7. `StorageEvent` — синхронизация вкладок.** Мы разберём, как приложения синхронизируют состояние между вкладками через `StorageEvent`. Это событие генерируется **только для `localStorage`**, **только в других вкладках** того же origin, и содержит поля `key`, `oldValue`, `newValue`, `url`, `storageArea`. В `storage.rs` появятся две функции: `parse_storage_event` и `format_storage_change`. В `app.rs` — секция «События»: лог всех изменений из других вкладок с полями `url` и `area`. Плюс обработчик `storage` на `window`, который перерисовывает все секции при изменениях. В тестах — 9 новых проверок на парсинг, форматирование и edge cases.