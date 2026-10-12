# Глава 8. Типизированный DOM

## Обновлённая структура проекта

```
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
├── src/
│   ├── lib.rs                      # pub mod dom; mod app; mod form; mod logging;
│   ├── app.rs                      # логика приложения (форма на Rust)
│   ├── dom.rs                      # помощники для работы с DOM
│   ├── form.rs                     # работа с элементами формы
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs                  # общие помощники (директория с mod.rs)
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs                 # дополнен тестами формы
│   ├── wasm_events.rs
│   └── wasm_async.rs
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css
│   └── assets/
│       └── favicon.ico
├── pkg/                            # генерируется wasm-pack
├── target/                         # генерируется cargo
└── node_modules/                   # генерируется npm
```

**Комментарии к структуре:**

- `src/form.rs` — модуль для работы с типизированными элементами формы: `HtmlInputElement`, `HtmlTextAreaElement`, `HtmlSelectElement`, `HtmlOptionElement`, `HtmlButtonElement`, `HtmlFormElement`.
- `src/lib.rs` объявляет `mod form;` — без этой строки модуль не будет виден, и `crate::form` в `app.rs` не скомпилируется.
- `tests/common/mod.rs` — **директория с `mod.rs`**, а не файл `tests/common.rs`. Это важно: Cargo трактует каждый `.rs`-файл в `tests/` как отдельный тестовый крейт. Помещая помощники в `tests/common/mod.rs`, мы исключаем их из тестовых крейтов — они подключаются через `mod common;` в каждом тестовом файле, но не компилируются как самостоятельные тесты.
- `pkg/LICENSE` и `pkg/README.md` — `wasm-pack` автоматически копирует эти файлы из корня проекта при сборке.

---

## Ссылки для детального изучения

| Тема                    | Ссылка                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| `HtmlInputElement`      | https://docs.rs/web-sys/latest/web_sys/struct.HtmlInputElement.html                              |
| `HtmlButtonElement`     | https://docs.rs/web-sys/latest/web_sys/struct.HtmlButtonElement.html                             |
| `HtmlTextAreaElement`   | https://docs.rs/web-sys/latest/web_sys/struct.HtmlTextAreaElement.html                           |
| `HtmlSelectElement`     | https://docs.rs/web-sys/latest/web_sys/struct.HtmlSelectElement.html                             |
| `HtmlOptionElement`     | https://docs.rs/web-sys/latest/web_sys/struct.HtmlOptionElement.html                             |
| `HtmlOptionsCollection` | https://docs.rs/web-sys/latest/web_sys/struct.HtmlOptionsCollection.html                         |
| `HtmlFormElement`       | https://docs.rs/web-sys/latest/web_sys/struct.HtmlFormElement.html                               |
| `JsCast`                | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/trait.JsCast.html                       |
| `dyn_into`              | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/trait.JsCast.html#method.dyn_into       |
| `unchecked_into`        | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/trait.JsCast.html#method.unchecked_into |
| Модули в Rust           | https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html        |
| `must_use` в Rust       | https://doc.rust-lang.org/reference/attributes/diagnostics.html#the-must_use-attribute           |

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

[dependencies.web-sys]
version = "0.3"
features = [
    # Основные типы DOM
    "Window",
    "Document",
    "Node",
    "Element",
    "HtmlElement",

    # Поиск и создание
    "NodeList",
    "HtmlCollection",

    # Атрибуты и классы
    "DomTokenList",
    "NamedNodeMap",

    # События
    "Event",
    "InputEvent",
    "EventTarget",

    # Элементы формы
    "HtmlInputElement",
    "HtmlButtonElement",
    "HtmlTextAreaElement",
    "HtmlSelectElement",
    "HtmlOptionElement",
    "HtmlOptionsCollection",   # нужно для select.options()
    "HtmlFormElement",

    # Медиа-элементы
    "HtmlImageElement",
    "HtmlCanvasElement",
    "HtmlVideoElement",
    "HtmlAudioElement",

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

**Ключевые моменты:**

- **`repository`** указывает на реальный репозиторий. Это убирает предупреждение `wasm-pack`: «Optional field missing from Cargo.toml: 'repository'».
- **`LICENSE`** — файл в корне проекта с текстом MIT. Без него `wasm-pack` предупреждает: «License key is set in Cargo.toml but no LICENSE file(s) were found».
- **`HtmlOptionsCollection`** — feature, которая нужна для метода `HtmlSelectElement::options()`. Без неё метод не компилируется, даже если сам тип `HtmlSelectElement` подключён.

---

## Особенности модульной системы Rust

**Каждый новый файл-модуль должен быть объявлен в родителе.** Это принципиальное отличие Rust от многих других языков: модули не обнаруживаются автоматически по файловой системе.

Если вы создали `src/form.rs` и хотите использовать `crate::form` в `app.rs` — недостаточно просто создать файл. Нужно **явно** объявить модуль в `src/lib.rs`:

```rust
pub mod dom;
mod app;
mod form;      // ← обязательно
mod logging;
```

Без `mod form;` компилятор выдаст ошибку:

```
error[E0432]: unresolved import `crate::form`
 --> src/app.rs:15:5
  |
15| use crate::form;
  |     ^^^^^^^^^^^ no `form` in the root
```

**Публичность модуля и его содержимого — разные вещи.**

- `mod form;` — модуль приватный. Из внешних crate'ов обратиться к `hello_wasm02::form` нельзя.
- `pub mod form;` — модуль публичный. Но содержимое всё ещё требует `pub` для каждой функции.
- `pub mod dom;` — публичный модуль, потому что его функции используются в doc-тестах и внешних интеграционных тестах.
- `mod form;` — приватный, потому что используется только внутри приложения.

Такое разделение отражает реальные границы API: `dom` — библиотека, `form` — внутренняя деталь.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Точка входа — функция `run`, которая вызывается автоматически
//! при инициализации Wasm-модуля.
//!
//! Модуль `dom` публичен: он содержит переиспользуемые помощники
//! для работы с DOM.
//!
//! Модуль `form` — внутренний слой для работы с элементами формы.

pub mod dom;
mod app;
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

## `src/form.rs`

```rust
//! Помощники для работы с элементами формы.
//!
//! Этот модуль — переиспользуемая библиотека. В текущем `app.rs`
//! используется только часть функций; остальные понадобятся
//! в следующих главах. Подавляем `dead_code`, чтобы предупреждения
//! не засоряли вывод сборки.
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    HtmlButtonElement, HtmlFormElement, HtmlInputElement,
    HtmlOptionElement, HtmlSelectElement, HtmlTextAreaElement,
};

use crate::dom;

// ============================================================
// Создание типизированных элементов
// ============================================================

/// Создаёт `<input>` с заданным типом и `id`.
pub fn create_input(input_type: &str, id: &str) -> Result<HtmlInputElement, JsValue> {
    let el = dom::create_element("input")?;
    let input = el.dyn_into::<HtmlInputElement>().map_err(|_| {
        JsValue::from_str("failed to cast Element to HtmlInputElement")
    })?;
    input.set_type(input_type);
    input.set_id(id);
    Ok(input)
}

/// Создаёт `<textarea>` с заданным `id`.
pub fn create_textarea(id: &str) -> Result<HtmlTextAreaElement, JsValue> {
    let el = dom::create_element("textarea")?;
    let ta = el.dyn_into::<HtmlTextAreaElement>().map_err(|_| {
        JsValue::from_str("failed to cast Element to HtmlTextAreaElement")
    })?;
    ta.set_id(id);
    Ok(ta)
}

/// Создаёт `<select>` с заданным `id`.
pub fn create_select(id: &str) -> Result<HtmlSelectElement, JsValue> {
    let el = dom::create_element("select")?;
    let select = el.dyn_into::<HtmlSelectElement>().map_err(|_| {
        JsValue::from_str("failed to cast Element to HtmlSelectElement")
    })?;
    select.set_id(id);
    Ok(select)
}

/// Создаёт `<option>` с заданным значением и текстом.
pub fn create_option(value: &str, text: &str) -> Result<HtmlOptionElement, JsValue> {
    let el = dom::create_element("option")?;
    let option = el.dyn_into::<HtmlOptionElement>().map_err(|_| {
        JsValue::from_str("failed to cast Element to HtmlOptionElement")
    })?;
    option.set_value(value);
    option.set_text(text);
    Ok(option)
}

/// Создаёт `<button>` с заданным типом и текстом.
pub fn create_button(button_type: &str, text: &str) -> Result<HtmlButtonElement, JsValue> {
    let el = dom::create_element("button")?;
    let button = el.dyn_into::<HtmlButtonElement>().map_err(|_| {
        JsValue::from_str("failed to cast Element to HtmlButtonElement")
    })?;
    button.set_type(button_type);
    button.set_text_content(Some(text));
    Ok(button)
}

/// Создаёт `<form>` с заданным `id`.
pub fn create_form(id: &str) -> Result<HtmlFormElement, JsValue> {
    let el = dom::create_element("form")?;
    let form = el.dyn_into::<HtmlFormElement>().map_err(|_| {
        JsValue::from_str("failed to cast Element to HtmlFormElement")
    })?;
    form.set_id(id);
    Ok(form)
}

// ============================================================
// Чтение значений
// ============================================================

/// Читает значение `<input>`.
pub fn value_of(input: &HtmlInputElement) -> String {
    input.value()
}

/// Читает значение `<textarea>`.
pub fn textarea_value(ta: &HtmlTextAreaElement) -> String {
    ta.value()
}

/// Читает выбранное значение `<select>`.
pub fn select_value(select: &HtmlSelectElement) -> String {
    select.value()
}

// ============================================================
// Приведение из EventTarget
// ============================================================

/// Приводит `EventTarget` к `HtmlInputElement`.
///
/// Возвращает `Result`, потому что `EventTarget` может быть
/// любым элементом, а не только input.
pub fn target_as_input(target: &web_sys::EventTarget) -> Result<HtmlInputElement, JsValue> {
    target
        .dyn_ref::<HtmlInputElement>()
        .cloned()
        .ok_or_else(|| JsValue::from_str("event target is not an HtmlInputElement"))
}

/// Приводит `EventTarget` к `HtmlTextAreaElement`.
pub fn target_as_textarea(
    target: &web_sys::EventTarget,
) -> Result<HtmlTextAreaElement, JsValue> {
    target
        .dyn_ref::<HtmlTextAreaElement>()
        .cloned()
        .ok_or_else(|| JsValue::from_str("event target is not an HtmlTextAreaElement"))
}

/// Приводит `EventTarget` к `HtmlSelectElement`.
pub fn target_as_select(
    target: &web_sys::EventTarget,
) -> Result<HtmlSelectElement, JsValue> {
    target
        .dyn_ref::<HtmlSelectElement>()
        .cloned()
        .ok_or_else(|| JsValue::from_str("event target is not an HtmlSelectElement"))
}

// ============================================================
// Валидация
// ============================================================

/// Проверяет валидность `<input>` по встроенным правилам HTML.
pub fn is_valid(input: &HtmlInputElement) -> bool {
    input.check_validity()
}

/// Возвращает сообщение об ошибке валидации.
pub fn validation_message(input: &HtmlInputElement) -> String {
    input.validation_message().unwrap_or_default()
}

/// Проверяет, пустое ли значение.
pub fn is_empty(value: &str) -> bool {
    value.trim().is_empty()
}
```

**Ключевые моменты:**

- **`#![allow(dead_code)]`** подавляет warnings о неиспользуемых функциях (`target_as_textarea`, `target_as_select`, `is_valid`, `validation_message`). Модуль — библиотека «на вырост».
- **`HtmlElement` убран из импорта** — он не используется: все функции работают с конкретными типами.
- **`dyn_into` для `Element` → конкретный тип.** Если приведение не удалось, возвращаем ошибку через `.map_err()`.
- **`dyn_ref` для `EventTarget` → конкретный тип.** `dyn_ref` принимает `&self` и возвращает `Option<&T>`. Мы клонируем результат через `.cloned()`.

---

## `src/app.rs`

```rust
//! Логика приложения: «форма на Rust».
//!
//! Rust строит форму внутри контейнера `#app`, навешивает
//! обработчики на кнопку и поля, читает значения и выводит
//! результат в отдельный `<div>`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, InputEvent};

use crate::dom;
use crate::form;
use crate::log;

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    // 1. Заголовок
    let heading = dom::create_heading(1, "Форма на Rust")?;
    dom::append(&container, &heading)?;

    // 2. Форма
    let form = form::create_form("demo-form")?;

    // 3. Поле «Имя»
    let name_label = dom::create_element("label")?;
    name_label.set_text_content(Some("Имя: "));
    let name_input = form::create_input("text", "name")?;
    name_input.set_placeholder("Введите имя");
    name_input.set_required(true);
    dom::append(&name_label, &name_input)?;
    dom::append(&form, &name_label)?;

    // 4. Поле «Email»
    let email_label = dom::create_element("label")?;
    email_label.set_text_content(Some("Email: "));
    let email_input = form::create_input("email", "email")?;
    email_input.set_placeholder("user@example.com");
    dom::append(&email_label, &email_input)?;
    dom::append(&form, &email_label)?;

    // 5. Textarea «Сообщение»
    let msg_label = dom::create_element("label")?;
    msg_label.set_text_content(Some("Сообщение: "));
    let textarea = form::create_textarea("message")?;
    textarea.set_placeholder("Ваше сообщение...");
    textarea.set_rows(4);
    dom::append(&msg_label, &textarea)?;
    dom::append(&form, &msg_label)?;

    // 6. Select «Тема»
    let topic_label = dom::create_element("label")?;
    topic_label.set_text_content(Some("Тема: "));
    let select = form::create_select("topic")?;
    for (value, text) in [
        ("general", "Общий вопрос"),
        ("bug", "Баг"),
        ("feature", "Предложение"),
    ] {
        let option = form::create_option(value, text)?;
        select.add_with_html_option_element(&option)?;
    }
    dom::append(&topic_label, &select)?;
    dom::append(&form, &topic_label)?;

    // 7. Кнопка «Отправить»
    let submit_btn = form::create_button("submit", "Отправить")?;
    dom::append(&form, &submit_btn)?;

    // 8. Область для вывода результата
    let output = dom::create_element("div")?;
    dom::set_attribute(&output, "id", "output")?;
    dom::add_class(&dom::as_html(&output), "output")?;
    dom::append(&form, &output)?;

    // 9. Обработчик submit: читаем значения, валидируем, выводим
    let name_input_clone = name_input.clone();
    let email_input_clone = email_input.clone();
    let textarea_clone = textarea.clone();
    let select_clone = select.clone();
    let output_clone = output.clone();

    let submit_closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        event.prevent_default();

        let name = form::value_of(&name_input_clone);
        let email = form::value_of(&email_input_clone);
        let message = form::textarea_value(&textarea_clone);
        let topic = form::select_value(&select_clone);

        let errors = validate_form(&name, &email, &message);

        if errors.is_empty() {
            let result = format!(
                "✅ Отправлено!\nИмя: {}\nEmail: {}\nТема: {}\nСообщение: {}",
                name, email, topic, message
            );
            output_clone.set_text_content(Some(&result));
            log!("Rust: форма отправлена");
        } else {
            let result = format!("❌ Ошибки:\n{}", errors.join("\n"));
            output_clone.set_text_content(Some(&result));
            log!("Rust: форма содержит ошибки: {}", errors.len());
        }
    });

    form.add_event_listener_with_callback("submit", submit_closure.as_ref().unchecked_ref())?;
    submit_closure.forget();

    // 10. Обработчик input: живая валидация поля «Имя»
    let name_input_live = name_input.clone();
    let input_closure = Closure::<dyn FnMut(InputEvent)>::new(move |event: InputEvent| {
        let target = event.target().unwrap();
        if let Ok(input) = form::target_as_input(&target) {
            let value = form::value_of(&input);
            let valid = !form::is_empty(&value);
            let html = dom::as_html(&input);
            if valid {
                dom::remove_class(&html, "invalid").ok();
                dom::add_class(&html, "valid").ok();
            } else {
                dom::remove_class(&html, "valid").ok();
                dom::add_class(&html, "invalid").ok();
            }
        }
    });

    name_input_live
        .add_event_listener_with_callback("input", input_closure.as_ref().unchecked_ref())?;
    input_closure.forget();

    // 11. Добавляем форму в контейнер
    dom::append(&container, &form)?;

    log!("Rust: форма построена");

    Ok(())
}

/// Простая валидация формы.
fn validate_form(name: &str, email: &str, message: &str) -> Vec<String> {
    let mut errors = Vec::new();

    if form::is_empty(name) {
        errors.push("• Имя обязательно".into());
    }
    if form::is_empty(email) {
        errors.push("• Email обязателен".into());
    } else if !email.contains('@') {
        errors.push("• Email должен содержать @".into());
    }
    if form::is_empty(message) {
        errors.push("• Сообщение обязательно".into());
    }

    errors
}
```

**Ключевые моменты:**

- **Импорты `Rc` и `RefCell` отсутствуют.** В первой версии я предполагал использовать `Rc<RefCell<...>>` для хранения состояния, но в финальном варианте состояние читается прямо из полей формы при каждом событии. Лишние импорты удалены — Rust выдаст warning `unused import`, если их оставить.
- **`.ok()` для `remove_class`** подавляет warning `unused_must_use`. Все четыре вызова `class_list`-операций обрабатываются одинаково.
- **Обработчик `submit`** — `event.prevent_default()` блокирует стандартную отправку формы (перезагрузку страницы).
- **Обработчик `input`** — живая валидация: класс `valid`/`invalid` переключается на элементе.

---

## Предупреждения Rust и как с ними работать

При разработке главы 8 мы столкнулись с четырьмя типами предупреждений. Разберём каждый.

### 1. `unresolved import crate::form`

**Причина:** файл `src/form.rs` создан, но не объявлен в `src/lib.rs`.

**Решение:** добавить `mod form;` в `lib.rs`.

**Правило:** каждый новый файл-модуль должен быть явно объявлен в родителе. Rust не сканирует директории в поисках модулей.

### 2. `unused import: std::cell::RefCell` / `std::rc::Rc`

**Причина:** импорты были добавлены на этапе планирования, но в финальном коде не используются.

**Решение:** удалить неиспользуемые импорты.

**Правило:** Rust предупреждает о неиспользуемых импортах, и это полезно — они указывают на код, который можно упростить. Если импорт нужен для типажа, который используется только через метод, попробуйте `use Trait as _;`.

### 3. `unused Result that must be used`

**Причина:** функция `dom::remove_class` возвращает `Result<(), JsValue>` с атрибутом `#[must_use]`. Мы вызываем её, но игнорируем результат.

**Решение:** добавить `.ok()` или `let _ = ...`:

```rust
dom::remove_class(&html, "invalid").ok();
```

**Правило:** `#[must_use]` — механизм Rust, который заставляет обрабатывать результат функции. Для `Result` это особенно важно: `Err` может содержать критичную информацию. Если вы сознательно игнорируете результат, `.ok()` или `let _ = ...` документируют это.

### 4. `function ... is never used` (dead_code)

**Причина:** модуль `form.rs` содержит функции (`target_as_textarea`, `target_as_select`, `is_valid`, `validation_message`), которые не вызываются в текущем `app.rs`.

**Решение:** добавить `#![allow(dead_code)]` в начало модуля.

**Правило:** `#![allow(dead_code)]` оправдан для модулей-библиотек, написанных «на вырост». Когда функции начнут использоваться — директиву можно убрать. В production-коде, наоборот, неиспользуемые функции удаляют: если код не нужен — зачем его компилировать?

---

## `tests/common/mod.rs`

```
tests/
├── common/
│   └── mod.rs
├── pure_logic.rs
├── wasm_async.rs
├── wasm_dom.rs
├── wasm_events.rs
└── wasm_logic.rs
```

**Почему `common/mod.rs`, а не `common.rs`.** Cargo трактует **каждый `.rs`-файл в `tests/`** как отдельный тестовый крейт. Если положить помощники в `tests/common.rs`, Cargo попытается скомпилировать их как отдельный тест и выведет в логе:

```
Running tests/common.rs ... no tests to run!
```

Это не ошибка, но шум. Помещая помощники в директорию `tests/common/` с файлом `mod.rs`, мы исключаем их из автообнаружения: директория без `.rs` в имени не считается тестовым крейтом. Cargo компилирует `common/mod.rs` только тогда, когда его подключают через `mod common;`.

```rust
//! Общие помощники для интеграционных тестов.
//!
//! Каждый тестовый файл подключает этот модуль через `mod common;`.
//! Разные файлы используют разные функции, поэтому часть из них
//! остаётся неиспользуемой в конкретном файле. `dead_code` здесь —
//! ожидаемое поведение, а не ошибка.
#![allow(dead_code)]

use wasm_bindgen::JsCast;
use web_sys::Element;

/// Создаёт изолированный контейнер `<div>` в `document.body`
/// и возвращает его. Используется как fixture для DOM-тестов.
pub fn create_fixture() -> Element {
    let document = web_sys::window()
        .expect("no window")
        .document()
        .expect("no document");

    let fixture = document
        .create_element("div")
        .expect("failed to create fixture");

    fixture.set_attribute("data-test", "true").unwrap();
    document
        .body()
        .expect("no body")
        .append_child(&fixture)
        .expect("failed to append fixture");

    fixture
}

/// Удаляет fixture из DOM после теста.
pub fn cleanup_fixture(fixture: &Element) {
    fixture.remove();
}

/// Утверждение: элемент содержит дочерний узел с заданным тегом.
pub fn assert_has_child(parent: &Element, tag: &str) {
    let children = parent.children();
    let found = (0..children.length()).any(|i: u32| {
        children
            .item(i)
            .and_then(|node| node.dyn_into::<Element>().ok())
            .map(|el| el.tag_name().eq_ignore_ascii_case(tag))
            .unwrap_or(false)
    });
    assert!(found, "expected child <{}> in fixture", tag);
}

/// Приводит `Element` к `T` через `dyn_into`.
pub fn cast<T: JsCast>(el: &Element) -> T {
    el.clone()
        .dyn_into::<T>()
        .expect("failed to cast element to expected type")
}
```

---

## `tests/wasm_dom.rs` (дополненный)

```rust
//! Тесты, требующие реального браузера (DOM, форма).
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

mod common;

use wasm_bindgen::JsCast;
use web_sys::{HtmlElement, HtmlInputElement, HtmlSelectElement};

#[wasm_bindgen_test]
fn test_document_exists() {
    let document = web_sys::window()
        .expect("no window")
        .document()
        .expect("no document");
    assert!(document.body().is_some());
}

#[wasm_bindgen_test]
fn test_create_paragraph() {
    let fixture = common::create_fixture();

    let p = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("p")
        .unwrap();
    p.set_text_content(Some("test paragraph"));
    fixture.append_child(&p).unwrap();

    common::assert_has_child(&fixture, "p");

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_set_attribute() {
    let fixture = common::create_fixture();

    fixture.set_attribute("data-role", "test").unwrap();
    assert_eq!(fixture.get_attribute("data-role").unwrap(), "test");

    fixture.remove_attribute("data-role").unwrap();
    assert!(fixture.get_attribute("data-role").is_none());

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_class_list() {
    let fixture = common::create_fixture();
    let html: HtmlElement = fixture.clone().unchecked_into();

    html.class_list().add_1("foo").unwrap();
    html.class_list().add_1("bar").unwrap();

    assert!(html.class_list().contains("foo"));
    assert!(html.class_list().contains("bar"));
    assert!(!html.class_list().contains("baz"));

    html.class_list().remove_1("foo").unwrap();
    assert!(!html.class_list().contains("foo"));
    assert!(html.class_list().contains("bar"));

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_query_selector() {
    let fixture = common::create_fixture();

    let child = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("span")
        .unwrap();
    child.set_attribute("data-test-child", "yes").unwrap();
    fixture.append_child(&child).unwrap();

    let found = fixture
        .query_selector("[data-test-child='yes']")
        .unwrap()
        .unwrap();
    assert_eq!(found.tag_name(), "SPAN");

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_input_element_dyn_into() {
    let fixture = common::create_fixture();

    let input = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("input")
        .unwrap()
        .dyn_into::<HtmlInputElement>()
        .unwrap();

    input.set_type("text");
    input.set_value("hello");

    assert_eq!(input.value(), "hello");
    assert_eq!(input.type_(), "text");

    fixture.append_child(&input).unwrap();

    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_select_element_options() {
    let fixture = common::create_fixture();

    let document = web_sys::window().unwrap().document().unwrap();

    let select = document
        .create_element("select")
        .unwrap()
        .dyn_into::<HtmlSelectElement>()
        .unwrap();

    for (value, text) in [("a", "Option A"), ("b", "Option B")] {
        let option = document.create_element("option").unwrap();
        option.set_attribute("value", value).unwrap();
        option.set_text_content(Some(text));
        select.add_with_html_option_element(
            &option.dyn_into::<web_sys::HtmlOptionElement>().unwrap(),
        ).unwrap();
    }

    fixture.append_child(&select).unwrap();

    assert_eq!(select.options().length(), 2);
    assert_eq!(select.value(), "a");

    common::cleanup_fixture(&fixture);
}
```

**Обратите внимание на `test_select_element_options`.** Без feature `HtmlOptionsCollection` в `Cargo.toml` метод `select.options()` не скомпилируется, и тест упадёт с ошибкой:

```
error[E0599]: no method named `options` found for struct `HtmlSelectElement`
```

Это тот самый случай, когда feature нужна для **возвращаемого** типа, а не для типа, на котором вызывается метод.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. В контейнере `#app` появится форма: поля «Имя», «Email», textarea «Сообщение», select «Тема» и кнопка «Отправить». При нажатии кнопки Rust валидирует форму и выводит результат в блок `output`. При вводе в поле «Имя» — живая валидация через классы `valid`/`invalid`.

### Тесты

```bash
# Хостовые тесты (чистый Rust)
npm run test:host

# Doc-тесты (compile-only для Wasm)
npm run test:doc

# Wasm-тесты логики в Node.js
npm run test:node

# DOM-тесты в headless Chrome
npm run test:chrome

# DOM-тесты в headless Firefox (WSL2)
npm run test:firefox

# Основной набор: host + node + chrome
npm test

# Полный прогон: host + node + chrome + firefox
npm run test:all
```

### Проверка конкретного теста

```bash
wasm-pack test --headless --chrome --test wasm_dom
```

---

## Ключевые концепции главы

**1. Типизированные HTML-элементы.** `web_sys` предоставляет отдельный тип для каждого HTML-элемента: `HtmlInputElement`, `HtmlButtonElement`, `HtmlTextAreaElement`, `HtmlSelectElement`, `HtmlOptionElement`, `HtmlImageElement`, `HtmlCanvasElement`, `HtmlVideoElement`, `HtmlAudioElement`. Каждый добавляет методы, специфичные для элемента.

**2. `dyn_into` и `JsCast`.** `create_element()` возвращает `Element` — базовый тип без специфичных методов. Чтобы получить `HtmlInputElement`, используйте `dyn_into::<HtmlInputElement>()`. Это безопасное приведение с проверкой типа во время выполнения.

**3. `dyn_ref` vs `dyn_into`.** `dyn_into` принимает `self` по значению и возвращает `Result<T, JsValue>`. `dyn_ref` принимает `&self` и возвращает `Option<&T>`. Для `EventTarget` удобнее `dyn_ref`.

**4. `unchecked_into`.** Быстрое статическое приведение без проверки типа. Паникует при неверном типе. Используйте только тогда, когда точно знаете тип.

**5. Ошибки неправильного cast.** `dyn_into::<HtmlInputElement>()` на элементе `<div>` вернёт `Err`. Это ожидаемое поведение.

**6. Валидация формы.** `check_validity()` запускает встроенную HTML-валидацию (required, type="email", pattern). `validation_message()` возвращает сообщение об ошибке.

**7. Обработчики формы.** `submit` — основное событие формы. `event.prevent_default()` блокирует стандартную отправку. `input` на полях — для живой валидации.

**8. `mod form;` в `lib.rs`.** Каждый новый файл-модуль должен быть объявлен в родителе. Rust не обнаруживает модули автоматически.

**9. `#![allow(dead_code)]` для библиотечных модулей.** Модуль `form.rs` — библиотека «на вырост». Часть функций не используется в `app.rs`, но понадобится позже.

**10. `.ok()` для `unused_must_use`.** `Result` с атрибутом `#[must_use]` требует обработки. `.ok()` явно говорит: «мы сознательно игнорируем ошибку».

**11. `HtmlOptionsCollection` для `select.options()`.** Feature нужна для **возвращаемого** типа, а не для типа, на котором вызывается метод.

**12. `tests/common/mod.rs` вместо `tests/common.rs`.** Директория с `mod.rs` исключает помощники из автообнаружения тестовых крейтов.

---

## Итог главы

- Каждый HTML-элемент имеет типизированный Rust-тип в `web_sys`: `HtmlInputElement`, `HtmlSelectElement` и т.д.
- `create_element()` возвращает `Element`; для доступа к специфичным методам используйте `dyn_into::<T>()`.
- `dyn_into` — безопасное приведение с проверкой типа во время выполнения. `unchecked_into` — быстрое, но паникует при неверном типе.
- `dyn_ref` — ссылочный вариант для `EventTarget`, возвращает `Option<&T>`.
- Features `HtmlInputElement`, `HtmlSelectElement`, `HtmlOptionsCollection` и другие нужно включать в `Cargo.toml`. Feature нужна для **возвращаемого** типа метода.
- `InputEvent` — feature для обработки события `input` на полях формы.
- `mod form;` в `lib.rs` обязателен — без него модуль не будет виден.
- `#![allow(dead_code)]` в `form.rs` оправдан: модуль задуман как расширяемая библиотека.
- `.ok()` для `remove_class` подавляет warning `unused_must_use`.
- `tests/common/mod.rs` — правильная организация помощников, исключающая их из автообнаружения тестов.
- `wasm-pack` копирует `LICENSE` и `README.md` в `pkg/` — файлы должны быть в корне проекта.
- Практический проект: форма на Rust с полями, select, textarea, кнопкой, валидацией и живой проверкой.

---

**Что дальше.** В главе 9 мы разберём **события** глубже: `EventTarget`, `Event`, `EventListener`, `Closure`, `add_event_listener_with_callback`, типы событий (click, input, change, submit, keyboard, mouse, pointer), распространение событий и практический проект — интерактивный DOM-компонент.
