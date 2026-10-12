# Глава 9. События

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
│   ├── lib.rs                      # pub mod dom; mod app; pub mod events; mod form; mod logging;
│   ├── app.rs                      # интерактивный счётчик
│   ├── dom.rs                      # помощники для работы с DOM
│   ├── events.rs                   # помощники для работы с событиями
│   ├── form.rs                     # работа с элементами формы
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs              # дополнен тестами keyboard/mouse/pointer
│   └── wasm_async.rs
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # только стили счётчика (глава 9)
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/events.rs` — новый модуль с помощниками для работы с событиями.
- `src/app.rs` — переписан под практический проект: интерактивный счётчик с кнопками «+»/«−», сбросом и клавиатурным управлением.
- `src/lib.rs` — добавляется **`pub mod events;`**. Модуль должен быть публичным, потому что интеграционные тесты в `tests/wasm_events.rs` обращаются к нему через `use hello_wasm02::events;`.
- `Cargo.toml` — добавляются features `KeyboardEvent`, `MouseEvent`, `PointerEvent`, `EventInit`.
- `web/styles/main.css` — **заменяется целиком**: стили формы из главы 8 удалены, добавлены стили счётчика.
- `tests/wasm_events.rs` — дополняется тестами для клавиатурных, мышиных и pointer-событий, а также тестами управления распространением.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `EventTarget` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.EventTarget.html |
| `add_event_listener_with_callback` | https://docs.rs/web-sys/latest/web_sys/struct.EventTarget.html#method.add_event_listener_with_callback |
| `Event` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.Event.html |
| `EventInit` | https://docs.rs/web-sys/latest/web_sys/struct.EventInit.html |
| `MouseEvent` | https://docs.rs/web-sys/latest/web_sys/struct.MouseEvent.html |
| `KeyboardEvent` | https://docs.rs/web-sys/latest/web_sys/struct.KeyboardEvent.html |
| `PointerEvent` | https://docs.rs/web-sys/latest/web_sys/struct.PointerEvent.html |
| `Closure` (wasm-bindgen) | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html |
| Управление временем жизни `Closure` | https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/closure/struct.Closure.html#method.forget |
| MDN: Event propagation | https://developer.mozilla.org/en-US/docs/Web/API/DOM_Event_Event_flow |
| MDN: addEventListener | https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener |
| wasm-bindgen: Event listeners | https://rustwasm.github.io/wasm-bindgen/examples/event-listener.html |

---

## `Cargo.toml` (обновлённый)

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

    # События (базовые)
    "Event",
    "EventTarget",
    "EventInit",         # нужно для Event::new_with_event_init_dict
    "InputEvent",

    # События (специализированные)
    "KeyboardEvent",     # клавиатурные события
    "MouseEvent",        # мышиные события
    "PointerEvent",      # pointer-события

    # Элементы формы
    "HtmlInputElement",
    "HtmlButtonElement",
    "HtmlTextAreaElement",
    "HtmlSelectElement",
    "HtmlOptionElement",
    "HtmlOptionsCollection",
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

**Новые features:**

- **`KeyboardEvent`** — события `keydown`, `keyup`, `keypress`. Даёт доступ к `key()`, `code()`, `shift_key()`, `ctrl_key()` и т.д.
- **`MouseEvent`** — события `click`, `mousedown`, `mouseup`, `mousemove`. Даёт доступ к `client_x()`, `client_y()`, `button()`, `buttons()`.
- **`PointerEvent`** — события `pointerdown`, `pointerup`, `pointermove`. Объединяет мышь, касания и стилус. Наследуется от `MouseEvent`.
- **`EventInit`** — словарь параметров инициализации события. Нужен для создания **bubbling** и **cancelable** событий через `Event::new_with_event_init_dict`.

**Правило `web_sys`:** если метод возвращает тип `X`, нужна feature `X`. Для `Event::new_with_event_init_dict` тип `EventInit` — обязателен.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Модуль `dom` публичен: он содержит переиспользуемые помощники
//! для работы с DOM.
//!
//! Модуль `events` публичен: интеграционные тесты в `tests/`
//! обращаются к нему через `use hello_wasm02::events;`.
//!
//! Модуль `form` — внутренний слой для работы с элементами формы.

pub mod dom;
pub mod events;
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

**Ключевое:** `pub mod events;` — модуль публичный. Без этого строки `use hello_wasm02::events;` в `tests/wasm_events.rs` не скомпилируются.

---

## `src/events.rs`

```rust
//! Помощники для работы с событиями.
//!
//! Этот модуль — переиспользуемая библиотека. В текущем `app.rs`
//! используется только часть функций; остальные понадобятся
//! в следующих главах. Подавляем `dead_code`, чтобы предупреждения
//! не засоряли вывод сборки.
//!
//! ## Замечание о `return`
//!
//! В функциях, где возвращаемое значение — результат макроса
//! (`matches!`) или цепочки методов, используется явный `return`.
//! Это защищает от редких случаев, когда `rust-analyzer` неверно
//! интерпретирует tail-выражение, и делает намерение однозначным.
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, EventTarget, KeyboardEvent, MouseEvent, PointerEvent};

// ============================================================
// Добавление обработчиков
// ============================================================

/// Добавляет обработчик события к `EventTarget`.
///
/// Closure перемещается внутрь и сохраняется через `forget()`.
/// Это простой, но одноразовый способ: обработчик живёт
/// до конца работы приложения.
///
/// Для production-кода предпочтительнее хранить `Closure`
/// явно (см. `add_event_listener_keep` ниже).
pub fn add_event_listener<F>(
    target: &EventTarget,
    event_type: &str,
    mut f: F,
) -> Result<(), JsValue>
where
    F: FnMut(Event) + 'static,
{
    let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        f(event);
    });
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

/// Добавляет обработчик и возвращает `Closure` для явного управления временем жизни.
///
/// Вызывающий код должен сохранить `Closure` в структуре или переменной.
/// Когда `Closure` будет удалён, обработчик перестанет работать.
pub fn add_event_listener_keep<F>(
    target: &EventTarget,
    event_type: &str,
    f: F,
) -> Result<Closure<dyn FnMut(Event)>, JsValue>
where
    F: FnMut(Event) + 'static,
{
    let closure = Closure::<dyn FnMut(Event)>::new(f);
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    Ok(closure)
}

/// Добавляет обработчик с конкретным типом события.
///
/// `T` должен реализовать `JsCast`. Для большинства типов событий
/// (`MouseEvent`, `KeyboardEvent`, `InputEvent`) приведение через
/// `dyn_into` работает автоматически.
pub fn add_typed_listener<T, F>(
    target: &EventTarget,
    event_type: &str,
    mut f: F,
) -> Result<(), JsValue>
where
    T: JsCast + 'static,
    F: FnMut(T) + 'static,
{
    let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        if let Ok(typed) = event.dyn_into::<T>() {
            f(typed);
        }
    });
    target.add_event_listener_with_callback(event_type, closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ============================================================
// Приведение событий к конкретным типам
// ============================================================

/// Приводит `Event` к `MouseEvent`.
pub fn as_mouse_event(event: &Event) -> Result<MouseEvent, JsValue> {
    event
        .clone()
        .dyn_into::<MouseEvent>()
        .map_err(|_| JsValue::from_str("event is not a MouseEvent"))
}

/// Приводит `Event` к `KeyboardEvent`.
pub fn as_keyboard_event(event: &Event) -> Result<KeyboardEvent, JsValue> {
    event
        .clone()
        .dyn_into::<KeyboardEvent>()
        .map_err(|_| JsValue::from_str("event is not a KeyboardEvent"))
}

/// Приводит `Event` к `PointerEvent`.
pub fn as_pointer_event(event: &Event) -> Result<PointerEvent, JsValue> {
    event
        .clone()
        .dyn_into::<PointerEvent>()
        .map_err(|_| JsValue::from_str("event is not a PointerEvent"))
}

// ============================================================
// Управление распространением
// ============================================================

/// Останавливает дальнейшее распространение события (bubbling).
///
/// Событие всё ещё будет обработано текущим обработчиком,
/// но не поднимется к родительским элементам.
pub fn stop_propagation(event: &Event) {
    event.stop_propagation();
}

/// Останавливает распространение и предотвращает вызов
/// других обработчиков на том же элементе.
pub fn stop_immediate_propagation(event: &Event) {
    event.stop_immediate_propagation();
}

/// Предотвращает действие браузера по умолчанию.
///
/// Работает только если событие создано с `cancelable: true`.
pub fn prevent_default(event: &Event) {
    event.prevent_default();
}

// ============================================================
// Вспомогательные проверки
// ============================================================

/// Проверяет, нажата ли клавиша Enter.
pub fn is_enter(event: &KeyboardEvent) -> bool {
    return event.key() == "Enter";
}

/// Проверяет, нажата ли клавиша Escape.
pub fn is_escape(event: &KeyboardEvent) -> bool {
    return event.key() == "Escape";
}

/// Проверяет, нажата ли клавиша-стрелка.
///
/// Возвращает `true` для `ArrowUp`, `ArrowDown`, `ArrowLeft`, `ArrowRight`.
pub fn is_arrow(event: &KeyboardEvent) -> bool {
    return matches!(
        event.key().as_str(),
        "ArrowUp" | "ArrowDown" | "ArrowLeft" | "ArrowRight"
    );
}

/// Проверяет, нажата ли клавиша-стрелка вверх.
pub fn is_arrow_up(event: &KeyboardEvent) -> bool {
    return event.key() == "ArrowUp";
}

/// Проверяет, нажата ли клавиша-стрелка вниз.
pub fn is_arrow_down(event: &KeyboardEvent) -> bool {
    return event.key() == "ArrowDown";
}

/// Проверяет, нажата ли клавиша-стрелка влево.
pub fn is_arrow_left(event: &KeyboardEvent) -> bool {
    return event.key() == "ArrowLeft";
}

/// Проверяет, нажата ли клавиша-стрелка вправо.
pub fn is_arrow_right(event: &KeyboardEvent) -> bool {
    return event.key() == "ArrowRight";
}

/// Проверяет, что событие мыши вызвано левой кнопкой.
pub fn is_left_button(event: &MouseEvent) -> bool {
    return event.button() == 0;
}

/// Проверяет, что событие мыши вызвано правой кнопкой.
pub fn is_right_button(event: &MouseEvent) -> bool {
    return event.button() == 2;
}

/// Проверяет, что событие мыши вызвано средней кнопкой.
pub fn is_middle_button(event: &MouseEvent) -> bool {
    return event.button() == 1;
}

// ============================================================
// Модификаторы клавиатуры
// ============================================================

/// Проверяет, зажат ли Shift.
pub fn has_shift(event: &KeyboardEvent) -> bool {
    return event.shift_key();
}

/// Проверяет, зажат ли Ctrl.
pub fn has_ctrl(event: &KeyboardEvent) -> bool {
    return event.ctrl_key();
}

/// Проверяет, зажат ли Alt.
pub fn has_alt(event: &KeyboardEvent) -> bool {
    return event.alt_key();
}

/// Проверяет, зажата ли Meta (Cmd на macOS, Win на Windows).
pub fn has_meta(event: &KeyboardEvent) -> bool {
    return event.meta_key();
}
```

**Ключевые моменты:**

- **`add_event_listener`** — простая функция для одноразовой подписки. `Closure` перемещается внутрь и забывается через `forget()`.
- **`add_event_listener_keep`** — возвращает `Closure` для явного управления временем жизни.
- **`add_typed_listener<T>`** — дженерик для обработки конкретного типа события.
- **Явный `return` во всех булевых функциях.** Это защищает от редких случаев, когда `rust-analyzer` неверно интерпретирует `matches!` в tail-позиции.
- **`is_arrow_up/down/left/right`**, **`is_left/right/middle_button`**, **`has_shift/ctrl/alt/meta`** — дополнительные проверки, добавленные для удобства.

---

## `src/app.rs` (интерактивный счётчик)

```rust
//! Логика приложения: «Интерактивный счётчик».
//!
//! Компонент состоит из:
//!   <div id="app">
//!     <h1>Счётчик</h1>
//!     <div class="counter-display">0</div>
//!     <div class="controls">
//!       <button id="dec">−</button>
//!       <button id="reset">Сброс</button>
//!       <button id="inc">+</button>
//!     </div>
//!     <p class="hint">Стрелки вверх/вниз, Enter — сброс, Escape — выход.</p>
//!   </div>

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, KeyboardEvent};

use crate::dom;
use crate::events;
use crate::log;

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    // 1. Заголовок
    let heading = dom::create_heading(1, "Счётчик")?;
    dom::append(&container, &heading)?;

    // 2. Отображение значения
    let display = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&display), "counter-display")?;
    display.set_text_content(Some("0"));
    dom::append(&container, &display)?;

    // 3. Контейнер для кнопок
    let controls = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&controls), "controls")?;

    let dec_btn = dom::create_element("button")?;
    dec_btn.set_text_content(Some("−"));
    dom::set_attribute(&dec_btn, "id", "dec")?;
    dom::append(&controls, &dec_btn)?;

    let reset_btn = dom::create_element("button")?;
    reset_btn.set_text_content(Some("Сброс"));
    dom::set_attribute(&reset_btn, "id", "reset")?;
    dom::append(&controls, &reset_btn)?;

    let inc_btn = dom::create_element("button")?;
    inc_btn.set_text_content(Some("+"));
    dom::set_attribute(&inc_btn, "id", "inc")?;
    dom::append(&controls, &inc_btn)?;

    dom::append(&container, &controls)?;

    // 4. Подсказка
    let hint = dom::create_paragraph("Стрелки вверх/вниз, Enter — сброс, Escape — выход.")?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&container, &hint)?;

    // 5. Состояние счётчика: Rc<RefCell<i32>>
    let counter = Rc::new(RefCell::new(0i32));

    // 6. Функция обновления отображения
    let update_display = {
        let counter = counter.clone();
        let display = display.clone();
        Rc::new(move || {
            let value = *counter.borrow();
            display.set_text_content(Some(&value.to_string()));
        })
    };

    // 7. Обработчик «+»
    {
        let counter = counter.clone();
        let update_display = update_display.clone();
        let closure = Closure::<dyn FnMut(Event)>::new(move |_event: Event| {
            *counter.borrow_mut() += 1;
            update_display();
            log!("Счётчик: {}", *counter.borrow());
        });
        inc_btn
            .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // 8. Обработчик «−»
    {
        let counter = counter.clone();
        let update_display = update_display.clone();
        let closure = Closure::<dyn FnMut(Event)>::new(move |_event: Event| {
            *counter.borrow_mut() -= 1;
            update_display();
            log!("Счётчик: {}", *counter.borrow());
        });
        dec_btn
            .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // 9. Обработчик «Сброс»
    {
        let counter = counter.clone();
        let update_display = update_display.clone();
        let closure = Closure::<dyn FnMut(Event)>::new(move |_event: Event| {
            *counter.borrow_mut() = 0;
            update_display();
            log!("Счётчик сброшен");
        });
        reset_btn
            .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // 10. Клавиатурное управление
    {
        let counter = counter.clone();
        let update_display = update_display.clone();
        let document = dom::document();

        let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
            let key_event: KeyboardEvent = match event.dyn_into() {
                Ok(k) => k,
                Err(_) => return,
            };

            if !events::is_arrow(&key_event)
                && !events::is_enter(&key_event)
                && !events::is_escape(&key_event)
            {
                return;
            }

            // Предотвращаем прокрутку страницы стрелками
            key_event.prevent_default();

            match key_event.key().as_str() {
                "ArrowUp" => {
                    *counter.borrow_mut() += 1;
                    update_display();
                    log!("↑ Счётчик: {}", *counter.borrow());
                }
                "ArrowDown" => {
                    *counter.borrow_mut() -= 1;
                    update_display();
                    log!("↓ Счётчик: {}", *counter.borrow());
                }
                "Enter" => {
                    *counter.borrow_mut() = 0;
                    update_display();
                    log!("Enter: счётчик сброшен");
                }
                "Escape" => {
                    log!("Escape: выход (в приложении — просто лог)");
                }
                _ => {}
            }
        });

        document
            .add_event_listener_with_callback("keydown", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    log!("Счётчик готов. Используйте кнопки или клавиши.");
    Ok(())
}
```

**Что демонстрирует `app.rs`:**

1. **`Rc<RefCell<i32>>`** — состояние счётчика. `Rc` разделяет владение между замыканиями, `RefCell` позволяет мутировать значение.
2. **`update_display`** — `Rc<dyn Fn()>`, вызывается после каждого изменения. Все обработчики имеют свою копию `Rc`.
3. **Обработчики `click`** на кнопках. Каждый `Closure` захватывает `counter` и `update_display`, мутирует состояние и обновляет DOM.
4. **Клавиатурный обработчик** на `document`. Слушает `keydown`, приводит `Event` к `KeyboardEvent` через `dyn_into`, проверяет `key()` и выполняет действие.
5. **`prevent_default()`** — блокирует прокрутку страницы при нажатии стрелок.

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — счётчик на Rust</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Счётчик построен Rust-кодом. Откройте консоль для логов.</p>

    <!-- Контейнер, в который Rust будет строить счётчик -->
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
}

body {
    max-width: 720px;
    margin: 2rem auto;
    padding: 0 1rem;
}

h1 {
    color: #b7410e;
}

/* ============================================================
   Контейнер #app
   ============================================================ */

#app {
    margin-top: 1.5rem;
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
}

/* ============================================================
   Отображение счётчика
   ============================================================ */

.counter-display {
    font-size: 3rem;
    font-weight: bold;
    text-align: center;
    margin: 1.5rem 0;
    color: #b7410e;
    font-variant-numeric: tabular-nums;
}

/* ============================================================
   Кнопки управления
   ============================================================ */

.controls {
    display: flex;
    justify-content: center;
    gap: 1rem;
    margin-bottom: 1rem;
}

.controls button {
    min-width: 80px;
    padding: 0.75rem 1.25rem;
    font-size: 1.1rem;
    border: 1px solid #b7410e;
    background: white;
    color: #b7410e;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
}

.controls button:hover {
    background: #b7410e;
    color: white;
}

.controls button:active {
    transform: scale(0.96);
}

/* ============================================================
   Подсказка
   ============================================================ */

.hint {
    text-align: center;
    font-size: 0.9rem;
    color: #666;
}
```

**Что изменилось относительно главы 8.** Стили формы (`form`, `label`, `input`, `textarea`, `select`, `input.valid`, `input.invalid`, `button[type="submit"]`, `.output`) удалены. Добавлены стили счётчика. `#app` вернул рамку — теперь она оформляет карточку счётчика.

---

## `tests/wasm_events.rs`

```rust
//! Тесты событий: click, input, keyboard, mouse, pointer.
use wasm_bindgen_test::*;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, EventInit, HtmlElement, KeyboardEvent, MouseEvent, PointerEvent};
use hello_wasm02::events;

wasm_bindgen_test_configure!(run_in_browser);

mod common;

// ============================================================
// Базовые события
// ============================================================

#[wasm_bindgen_test]
fn test_click_event() {
    use std::cell::RefCell;
    use std::rc::Rc;

    let fixture = common::create_fixture();

    let button = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("button")
        .unwrap();
    button.set_text_content(Some("click me"));
    fixture.append_child(&button).unwrap();

    let button: HtmlElement = button.dyn_into().unwrap();

    let clicked = Rc::new(RefCell::new(false));
    let clicked_clone = clicked.clone();

    let closure = Closure::<dyn FnMut(Event)>::new(move |_event: Event| {
        *clicked_clone.borrow_mut() = true;
    });

    button
        .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())
        .unwrap();

    let event = Event::new("click").unwrap();
    button.dispatch_event(&event).unwrap();

    assert!(*clicked.borrow(), "click handler was not called");

    closure.forget();
    common::cleanup_fixture(&fixture);
}

#[wasm_bindgen_test]
fn test_event_prevent_default() {
    use std::cell::RefCell;
    use std::rc::Rc;

    let fixture = common::create_fixture();

    let link = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("a")
        .unwrap();
    link.set_attribute("href", "https://example.com").unwrap();
    fixture.append_child(&link).unwrap();

    let prevented = Rc::new(RefCell::new(false));
    let prevented_clone = prevented.clone();

    let closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        event.prevent_default();
        *prevented_clone.borrow_mut() = event.default_prevented();
    });

    link.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())
        .unwrap();

    // ВАЖНО: создаём отменяемое событие (cancelable: true).
    // По умолчанию Event::new создаёт non-cancelable событие,
    // и prevent_default() не имеет эффекта.
    let init = EventInit::new();
    init.set_cancelable(true);
    let event = Event::new_with_event_init_dict("click", &init).unwrap();

    link.dispatch_event(&event).unwrap();

    assert!(*prevented.borrow(), "default should be prevented");

    closure.forget();
    common::cleanup_fixture(&fixture);
}

// ============================================================
// Клавиатурные события
// ============================================================

#[wasm_bindgen_test]
fn test_keyboard_event_dyn_into() {
    let event = KeyboardEvent::new("keydown").unwrap();
    let generic: Event = event.clone().dyn_into().unwrap();

    let keyboard: KeyboardEvent = generic.dyn_into().unwrap();
    assert_eq!(keyboard.type_(), "keydown");
}

#[wasm_bindgen_test]
fn test_key_helpers_on_empty_event() {
    let event = KeyboardEvent::new("keydown").unwrap();

    assert!(!events::is_arrow(&event));
    assert!(!events::is_enter(&event));
    assert!(!events::is_escape(&event));
    assert!(!events::is_arrow_up(&event));
    assert!(!events::is_arrow_down(&event));
}

// ============================================================
// Мышиные события
// ============================================================

#[wasm_bindgen_test]
fn test_mouse_event_dyn_into() {
    let event = MouseEvent::new("click").unwrap();
    let generic: Event = event.clone().dyn_into().unwrap();

    let mouse: MouseEvent = generic.dyn_into().unwrap();
    assert_eq!(mouse.type_(), "click");
}

#[wasm_bindgen_test]
fn test_left_button_check() {
    let event = MouseEvent::new("click").unwrap();
    // button() по умолчанию 0 (левая кнопка)
    assert_eq!(event.button(), 0);
}

// ============================================================
// Pointer-события
// ============================================================

#[wasm_bindgen_test]
fn test_pointer_event_dyn_into() {
    let event = PointerEvent::new("pointerdown").unwrap();
    let generic: Event = event.clone().dyn_into().unwrap();

    let pointer: PointerEvent = generic.dyn_into().unwrap();
    assert_eq!(pointer.type_(), "pointerdown");
}

// ============================================================
// Управление распространением
// ============================================================

#[wasm_bindgen_test]
fn test_stop_propagation() {
    use std::cell::RefCell;
    use std::rc::Rc;

    let fixture = common::create_fixture();

    let parent = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("div")
        .unwrap();
    let child = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("button")
        .unwrap();

    fixture.append_child(&parent).unwrap();
    parent.append_child(&child).unwrap();

    let parent_called = Rc::new(RefCell::new(false));
    let parent_called_clone = parent_called.clone();

    // Обработчик на родителе
    let parent_closure = Closure::<dyn FnMut(Event)>::new(move |_event: Event| {
        *parent_called_clone.borrow_mut() = true;
    });
    parent
        .add_event_listener_with_callback("click", parent_closure.as_ref().unchecked_ref())
        .unwrap();
    parent_closure.forget();

    // Обработчик на ребёнке останавливает распространение
    let child_closure = Closure::<dyn FnMut(Event)>::new(move |event: Event| {
        event.stop_propagation();
    });
    child
        .add_event_listener_with_callback("click", child_closure.as_ref().unchecked_ref())
        .unwrap();
    child_closure.forget();

    // ВАЖНО: событие должно быть bubbling (bubbles: true),
    // иначе оно не поднимется к родителю и тест ничего не проверит.
    let init = EventInit::new();
    init.set_bubbles(true);
    let event = Event::new_with_event_init_dict("click", &init).unwrap();

    child.dispatch_event(&event).unwrap();

    assert!(
        !*parent_called.borrow(),
        "parent should not receive event after stop_propagation"
    );

    common::cleanup_fixture(&fixture);
}
```

**Ключевые моменты:**

- **`use hello_wasm02::events;`** — интеграционные тесты обращаются к библиотеке через имя crate'а, а не через `crate::`. `crate::` в тестовом файле указывает на сам тестовый crate.
- **`EventInit::set_cancelable(true)` и `EventInit::set_bubbles(true)`** — актуальные методы. Устаревшие `cancelable()` / `bubbles()` заменены.
- **Без `mut`** — в текущей версии `web-sys` сеттеры не требуют `mut`.
- **`Event::new_with_event_init_dict`** — создаёт событие с явными флагами.
- **`test_stop_propagation`** — событие создаётся с `bubbles: true`, иначе тест прошёл бы по ложной причине.

---

## `tests/common/mod.rs`

```rust
#![allow(dead_code)]

use wasm_bindgen::JsCast;
use web_sys::Element;

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

pub fn cleanup_fixture(fixture: &Element) {
    fixture.remove();
}

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
```

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. В контейнере `#app` появится счётчик с кнопками «+», «−», «Сброс» и клавиатурным управлением: стрелки вверх/вниз, Enter, Escape.

### Тесты

```bash
# Хостовые тесты (чистый Rust)
npm run test:host

# Doc-тесты (compile-only для Wasm)
npm run test:doc

# Wasm-тесты логики в Node.js
npm run test:node

# DOM-тесты и события в headless Chrome
npm run test:chrome

# DOM-тесты и события в headless Firefox (WSL2)
npm run test:firefox

# Основной набор: host + node + chrome
npm test

# Полный прогон: host + node + chrome + firefox
npm run test:all
```

### Проверка конкретного теста

```bash
wasm-pack test --headless --chrome --test wasm_events
```

---

## Ключевые концепции главы

**1. `EventTarget` — базовый интерфейс для событий.** `Window`, `Document`, `Element`, `Node` — все наследуются от `EventTarget`. Метод `add_event_listener_with_callback(event_type, callback)` регистрирует обработчик.

**2. `Closure` — мост между Rust-замыканием и JS-функцией.** `Closure::new(f)` создаёт JS-функцию из Rust-замыкания. `closure.as_ref().unchecked_ref()` даёт `&Function` для передачи в `add_event_listener_with_callback`.

**3. Время жизни `Closure` — критический момент.** `Closure` должен жить, пока JS вызывает обработчик. Способы управления:

- `closure.forget()` — утечка, но просто. Подходит для обработчиков, живущих всё время работы приложения.
- Хранение `Closure` в структуре — правильный способ для production.
- `Closure::once_into_js` — для одноразовых обработчиков.

**4. Типы событий и их features.** Каждый тип события требует feature: `MouseEvent`, `KeyboardEvent`, `PointerEvent`, `InputEvent`, `EventInit`.

**5. `KeyboardEvent::key()` vs `key_code()`.** `key()` возвращает строку: `"Enter"`, `"Escape"`, `"ArrowUp"`. Используйте `key()`.

**6. `MouseEvent` и `PointerEvent`.** `PointerEvent` наследуется от `MouseEvent`. Pointer-события объединяют мышь, касания и стилус.

**7. Распространение событий.** Событие проходит три фазы: capture, target, bubble. `stop_propagation()` останавливает bubbling. `prevent_default()` блокирует действие браузера, но **только для cancelable-событий**.

**8. `EventInit` — обязательный инструмент для синтетических событий.** `Event::new("click")` создаёт событие с `bubbles: false`, `cancelable: false`. Чтобы `prevent_default()` работал, нужно `EventInit::set_cancelable(true)`. Чтобы событие всплывало, нужно `EventInit::set_bubbles(true)`.

**9. `Rc<RefCell<T>>` для разделяемого состояния.** Несколько обработчиков могут разделять одно состояние. `Rc` даёт общее владение, `RefCell` — мутабельность во время выполнения.

**10. `pub mod events;` для интеграционных тестов.** Интеграционные тесты в `tests/` — это отдельные crate'ы. Чтобы обратиться к модулю библиотеки, он должен быть публичным, а путь — начинаться с имени crate'а: `use hello_wasm02::events;`.

**11. Явный `return` в булевых функциях.** В отдельных случаях `rust-analyzer` неверно интерпретирует `matches!` в tail-позиции. Явный `return` решает проблему и делает намерение однозначным.

---

## Итог главы

- `EventTarget` — базовый тип для всех объектов, которые могут генерировать события.
- `Closure` создаёт JS-функцию из Rust-замыкания. Управление временем жизни — через `forget`, хранение или `Rc`.
- `add_event_listener_with_callback` — основной метод регистрации обработчика.
- Типы событий (`MouseEvent`, `KeyboardEvent`, `PointerEvent`, `InputEvent`) требуют соответствующих features.
- `EventInit` нужен для создания bubbling и cancelable событий в тестах.
- `prevent_default()`, `stop_propagation()`, `stop_immediate_propagation()` управляют поведением события.
- `Rc<RefCell<T>>` — стандартный паттерн для разделяемого состояния между обработчиками.
- `pub mod events;` в `lib.rs` — обязательно для интеграционных тестов.
- `main.css` заменяется целиком: стили формы удалены, добавлены стили счётчика.
- Практический проект: интерактивный счётчик с кнопками и клавиатурным управлением.

---

**Что дальше.** В главе 10 мы перейдём к части IV и разберём `JsValue` как универсальный тип границы Rust ↔ JavaScript: примитивы, строки, числа, массивы, объекты, функции, проверки типов и преобразования.