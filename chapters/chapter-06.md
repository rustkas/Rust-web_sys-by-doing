# Глава 6. Тестирование Rust/WASM

## Обновлённая структура проекта

```
hello-wasm02/
├── Cargo.toml
├── package.json
├── vite.config.js
├── webdriver.json                  # конфигурация WebDriver
├── .gitignore
├── LICENSE                         # текст лицензии
├── src/
│   ├── lib.rs                      # pub mod dom; mod app; mod logging;
│   ├── app.rs
│   ├── dom.rs                      # с doc-тестом (no_run)
│   └── logging.rs
├── tests/
│   ├── common.rs                   # общие помощники
│   ├── pure_logic.rs               # хостовые тесты (обычный #[test])
│   ├── wasm_logic.rs               # Wasm-тесты логики (Node.js)
│   ├── wasm_dom.rs                 # DOM-тесты (headless-браузер)
│   ├── wasm_events.rs              # тесты событий и callbacks
│   └── wasm_async.rs               # асинхронные тесты
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/main.css
│   └── assets/favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/lib.rs` объявляет `pub mod dom;` — модуль `dom` становится частью публичного API, что позволяет doc-тестам его импортировать.
- `src/dom.rs` содержит doc-тест с атрибутом `no_run` — компилируется, но не выполняется на хосте.
- `tests/pure_logic.rs` — тесты чистой логики с обычным `#[test]`.
- `tests/wasm_logic.rs` — Wasm-тесты логики через `#[wasm_bindgen_test]` **без** `run_in_browser`. Запускаются в Node.js.
- `tests/wasm_dom.rs`, `wasm_events.rs`, `wasm_async.rs` — DOM-тесты с `wasm_bindgen_test_configure!(run_in_browser)`.
- `tests/common.rs` — общие помощники, подключается через `mod common;`.
- `webdriver.json` — конфигурация для `wasm-pack test`.
- `LICENSE` — файл лицензии, упомянутый в `Cargo.toml` (`license = "MIT"`).

Три уровня тестов отражают три уровня зависимости от Web Platform:

| Уровень | Атрибут | Директива | Runner |
|---|---|---|---|
| Чистая логика | `#[test]` | — | `cargo test` |
| Wasm без DOM | `#[wasm_bindgen_test]` | — | `wasm-pack test --node` |
| DOM/Web API | `#[wasm_bindgen_test]` | `run_in_browser` | `wasm-pack test --headless --<browser>` |

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `wasm-bindgen-test` (гайд) | https://wasm-bindgen.github.io/wasm-bindgen/wasm-bindgen-test/index.html |
| Использование `wasm-bindgen-test` | https://wasm-bindgen.github.io/wasm-bindgen/wasm-bindgen-test/usage.html |
| Тестирование в headless-браузерах | https://wasm-bindgen.github.io/wasm-bindgen/wasm-bindgen-test/browsers.html |
| Асинхронные тесты | https://wasm-bindgen.github.io/wasm-bindgen/wasm-bindgen-test/asynchronous-tests.html |
| CI-интеграция | https://wasm-bindgen.github.io/wasm-bindgen/wasm-bindgen-test/continuous-integration.html |
| `wasm-pack test` (docs.rs) | https://docs.rs/wasm-pack/latest/wasm_pack/command/test/index.html |
| Chrome for Testing | https://googlechromelabs.github.io/chrome-for-testing/ |
| Firefox ESR PPA | https://launchpad.net/~mozillateam/+archive/ubuntu/ppa |
| Rustdoc: doc-тесты | https://doc.rust-lang.org/rustdoc/documentation-tests.html |
| Пример полного тестирования | https://github.com/ChrisWhealy/svg-dom/blob/main/docs/testing.md |

---

## `Cargo.toml`

```toml
[package]
name = "hello-wasm02"
version = "0.1.0"
edition = "2021"
description = "Rust + web_sys starter"
license = "MIT"

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

    # Поиск и создание элементов
    "NodeList",           # query_selector_all
    "HtmlCollection",     # Element::children()

    # Атрибуты и классы
    "DomTokenList",       # classList
    "NamedNodeMap",       # attributes

    # События
    "Event",
    "EventTarget",

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

- **`wasm-bindgen-futures`** — отдельный crate для `JsFuture` и `spawn_local`. Не входит в `wasm-bindgen`.
- **`js-sys`** — привязки к ECMAScript-объектам (`Promise`, `Array`, `Object`).
- **`crate-type = ["cdylib", "rlib"]`** — `cdylib` для Wasm, `rlib` для хостовых тестов.
- **Features `NodeList`, `DomTokenList`, `NamedNodeMap`, `HtmlCollection`** — нужны для методов, возвращающих соответствующие типы.

**Правило `web_sys`:** если метод возвращает тип `X`, нужна feature `X`, даже если вы не используете `X` напрямую. Документация метода всегда указывает полный список required features.

---

## `webdriver.json`

Файл в корне проекта. `wasm-pack` автоматически находит его и передаёт содержимое как capabilities WebDriver.

```json
{
  "goog:chromeOptions": {
    "args": [
      "--no-sandbox",
      "--disable-gpu",
      "--disable-dev-shm-usage",
      "--headless=new"
    ]
  },
  "moz:firefoxOptions": {
    "binary": "/usr/bin/firefox-esr",
    "args": [
      "-headless",
      "-no-remote"
    ]
  }
}
```

**Пояснения:**

- `--no-sandbox` — обязателен в WSL2, Docker и CI: sandbox ядра недоступен.
- `--disable-dev-shm-usage` — обходит проблему ограниченного `/dev/shm` в WSL2.
- `--headless=new` — новый headless-режим Chrome.
- `binary` — явный путь к `firefox-esr`. `geckodriver` ищет бинарник с именем `firefox`, и без этой строки может найти Snap-обёртку.
- `-no-remote` — предотвращает попытку Firefox подключиться к существующему экземпляру.

---

## `src/lib.rs`

```rust
//! hello-wasm02 — демонстрация Rust + web_sys.
//!
//! Точка входа — функция `run`, которая вызывается автоматически
//! при инициализации Wasm-модуля.
//!
//! Модуль `dom` публичен: он содержит переиспользуемые помощники
//! для работы с DOM. Публичность нужна, в частности, для doc-тестов
//! и интеграционных тестов через `hello_wasm02::dom::...`.

pub mod dom;
mod app;
mod logging;

use wasm_bindgen::prelude::*;

/// Точка входа. Вызывается автоматически при инициализации Wasm-модуля.
#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    logging::init();
    app::run()
}
```

**Ключевой момент:** `pub mod dom;` — модуль публичен, потому что его функции используются в doc-тестах и могут использоваться из внешних тестов.

---

## `src/dom.rs` (фрагмент с doc-тестом)

```rust
//! Помощники для работы с DOM.

#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Document, Element, HtmlElement, Node, Window};

/// Возвращает глобальный объект `window`.
///
/// # Panics
///
/// Паникует, если код выполняется вне браузерного контекста.
///
/// # Пример
///
/// ```no_run
/// use hello_wasm02::dom;
/// let w = dom::window();
/// assert!(w.document().is_some());
/// ```
///
/// Атрибут `no_run` нужен потому, что doc-тесты выполняются
/// на хосте, а `web_sys::window()` доступен только в браузере.
pub fn window() -> Window {
    web_sys::window().expect("no global `window` exists")
}

// ... остальные функции без doc-тестов
```

**Ключевой момент:** `no_run` — doc-тест компилируется, но не запускается на хосте. Без него падает с `cannot access imported statics on non-wasm targets`, потому что `web_sys::window()` обращается к импортированным статикам `js-sys`, которых на нативном таргете нет.

---

## `tests/common.rs`

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
```

**Ключевые моменты:**

- `#![allow(dead_code)]` подавляет warning «`assert_has_child` is never used» в тех тестовых файлах, которые подключают `common.rs`, но не используют все его функции.
- `Element::children()` возвращает `HtmlCollection` — причина, по которой нужна feature `HtmlCollection`.
- `children.item(i)` возвращает `Option<Node>`. Чтобы вызвать `tag_name()`, нужен `Element`, поэтому используем `dyn_into::<Element>()`.

---

## `tests/pure_logic.rs`

```rust
//! Тесты чистой логики, которые не зависят от браузера.
//!
//! Запускаются через `cargo test` на хосте.
//! Не используют `web_sys`, `wasm_bindgen` или другие
//! Wasm-специфичные API.

#[test]
fn test_arithmetic() {
    assert_eq!(2 + 2, 4);
}

#[test]
fn test_string_processing() {
    let input = "hello";
    let output = input.to_uppercase();
    assert_eq!(output, "HELLO");
}
```

**Зачем отдельный файл.** `#[wasm_bindgen_test]` **не генерирует** `#[test]` на хосте. Поэтому `cargo test` без этого файла выводит «0 tests to run». Хостовые тесты должны быть обычными `#[test]`-функциями без Wasm-зависимостей.

---

## `tests/wasm_logic.rs`

```rust
//! Wasm-тесты чистой логики — без браузера.
//!
//! Запускаются через `wasm-pack test --node`.
//!
//! ВАЖНО: не добавляйте сюда `wasm_bindgen_test_configure!(run_in_browser)`.
//! Иначе тесты будут пропускаться в Node.js с сообщением
//! "this test suite is only configured to run in a browser".
//!
//! При запуске `wasm-pack test --headless --chrome` этот файл
//! также будет пропущен — это ожидаемое поведение.
use wasm_bindgen_test::*;

#[wasm_bindgen_test]
fn test_wasm_logic() {
    // Здесь тесты логики, которые используют Wasm,
    // но не требуют DOM и других Web API.
    assert_eq!(2 + 2, 4);
}
```

**Ключевой момент.** Без `run_in_browser` harness по умолчанию запускает тесты в Node.js. При запуске браузерных тестов этот файл **пропускается** — это корректное поведение.

---

## `tests/wasm_dom.rs`

```rust
//! Тесты, требующие реального браузера (DOM, Promise).
//!
//! Запускаются ТОЛЬКО через:
//!   wasm-pack test --headless --chrome
//!   wasm-pack test --headless --firefox
use wasm_bindgen_test::*;

// Обязательная директива: сообщает harness'у, что тесты
// должны выполняться в браузере, а не в Node.js.
wasm_bindgen_test_configure!(run_in_browser);

mod common;

use wasm_bindgen::JsCast;
use web_sys::HtmlElement;

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

    let text = fixture
        .first_child()
        .unwrap()
        .dyn_into::<HtmlElement>()
        .unwrap()
        .text_content()
        .unwrap();
    assert_eq!(text, "test paragraph");

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
async fn test_promise_resolve() {
    use wasm_bindgen_futures::JsFuture;

    // Простейший async-тест: resolve промиса.
    let promise = js_sys::Promise::resolve(&wasm_bindgen::JsValue::from(42));
    let result = JsFuture::from(promise).await.unwrap();
    assert_eq!(result.as_f64().unwrap(), 42.0);
}
```

**Ключевые моменты:**

- `wasm_bindgen_test_configure!(run_in_browser)` — обязательна для DOM-тестов.
- `mod common;` — подключает `tests/common.rs`.
- `class_list().contains()` возвращает `bool`, не `Result` — никаких `unwrap` или `?`.
- `class_list().add_1()` / `remove_1()` возвращают `Result<(), JsValue>` — `.unwrap()` уместен.

---

## `tests/wasm_events.rs`

```rust
use wasm_bindgen_test::*;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, HtmlElement};

wasm_bindgen_test_configure!(run_in_browser);

mod common;

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

    // Флаг, который будет установлен обработчиком.
    let clicked = Rc::new(RefCell::new(false));
    let clicked_clone = clicked.clone();

    // Создаём Closure с правильным lifetime.
    let closure = Closure::<dyn FnMut(Event)>::new(move |_event: Event| {
        *clicked_clone.borrow_mut() = true;
    });

    // Навешиваем обработчик.
    button
        .add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())
        .unwrap();

    // Эмулируем клик.
    let event = Event::new("click").unwrap();
    button.dispatch_event(&event).unwrap();

    assert!(*clicked.borrow(), "click handler was not called");

    // ВАЖНО: сохраняем Closure, чтобы он не был удалён до конца теста.
    // В тестах можно использовать forget(), но в production — храните явно.
    closure.forget();

    common::cleanup_fixture(&fixture);
}
```

**Ключевые моменты:**

- Feature `Event` обязателен — без него `Event::new` и `Closure::<dyn FnMut(Event)>` не скомпилируются.
- `closure.as_ref().unchecked_ref()` — приведение `Closure` к `&js_sys::Function`.
- `closure.forget()` — в тестах допустимо. В production всегда храните `Closure` явно.

---

## `tests/wasm_async.rs`

```rust
use wasm_bindgen_test::*;
use wasm_bindgen_futures::JsFuture;
use wasm_bindgen::JsValue;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_async_timeout() {
    // Создаём Promise, который разрешается через setTimeout.
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        web_sys::window()
            .unwrap()
            .set_timeout_with_callback_and_timeout_and_arguments_0(
                &resolve,
                10,
            )
            .unwrap();
    });

    let result = JsFuture::from(promise).await;
    assert!(result.is_ok());
}

#[wasm_bindgen_test]
async fn test_rejected_promise() {
    let promise = js_sys::Promise::reject(&JsValue::from("expected error"));
    let result = JsFuture::from(promise).await;
    assert!(result.is_err());
    assert_eq!(result.unwrap_err().as_string().unwrap(), "expected error");
}
```

**Ключевые моменты:**

- `js_sys::Promise::new` принимает `&mut FnMut(Function, Function)` — замыкание, получающее `resolve` и `reject`.
- Async-тесты корректно обрабатывают reject: `JsFuture` возвращает `Err(JsValue)`.

---

## Установка браузеров в WSL2

**Этот раздел нужен только для WSL2 и контейнеров.** Если вы работаете в нативном Linux или macOS с уже установленными браузерами — пропустите.

### Chrome for Testing

Chrome, установленный в Windows, **не виден** из WSL2 — это разные файловые системы. Установите Chrome for Testing внутри WSL2.

```bash
# Создаём директорию
mkdir -p ~/.chrome-for-testing
cd ~/.chrome-for-testing

# Скачиваем Chrome for Testing и chromedriver
# Актуальные версии: https://googlechromelabs.github.io/chrome-for-testing/
wget https://storage.googleapis.com/chrome-for-testing-public/131.0.6778.204/linux64/chrome-linux64.zip
wget https://storage.googleapis.com/chrome-for-testing-public/131.0.6778.204/linux64/chromedriver-linux64.zip

# Распаковываем
unzip chrome-linux64.zip
unzip chromedriver-linux64.zip

# Добавляем в PATH
echo 'export PATH=~/.chrome-for-testing/chrome-linux64:~/.chrome-for-testing/chromedriver-linux64:$PATH' >> ~/.bashrc
source ~/.bashrc

# Проверяем
chrome --headless --version
# Google Chrome for Testing 131.0.6778.204
```

После установки Chrome-тесты работают без дополнительных флагов:

```bash
wasm-pack test --headless --chrome
```

### Firefox ESR

Firefox в Ubuntu по умолчанию ставится как **Snap-пакет**, который **не работает** в WSL2 в headless-режиме: `snapd` не запущен (systemd в WSL2 отключён), и Firefox не может стартовать.

**Шаг 1. Установить Firefox ESR из PPA Mozilla.**

```bash
# Добавляем PPA и устанавливаем Firefox ESR как .deb
sudo add-apt-repository ppa:mozillateam/ppa
sudo apt update
sudo apt install -y firefox-esr

# Проверяем
which firefox-esr
firefox-esr --headless --version
```

**Шаг 2. Создать симлинк `firefox` → `firefox-esr`.**

`geckodriver` ищет бинарник с именем `firefox` в `PATH`. Без симлинка он находит Snap-обёртку `/usr/bin/firefox`, которая падает с SIGKILL.

```bash
sudo ln -s /usr/bin/firefox-esr /usr/local/bin/firefox

# Проверяем
which firefox
# /usr/local/bin/firefox
firefox --headless --version
```

**Шаг 3. Установить `xvfb`.**

Firefox, даже в headless-режиме, требует наличия X-сервера в WSL2. Без него процесс убивается сигналом 9.

```bash
sudo apt update
sudo apt install -y xvfb
```

**Шаг 4. Запускать Firefox-тесты через `xvfb-run`.**

```bash
xvfb-run -a wasm-pack test --headless --firefox --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver
```

Флаг `-a` автоматически выбирает свободный дисплей. Флаг `--geckodriver` явно указывает путь к скачанному `wasm-pack` драйверу — иначе он может использовать другую (несовместимую) версию.

**Почему все четыре шага обязательны:**

- Без PPA — Snap-версия не работает в WSL2.
- Без симлинка — `geckodriver` находит неработающий `/usr/bin/firefox`.
- Без `xvfb` — Firefox убивается сигналом 9.
- Без `--geckodriver` — может использоваться несовместимая версия драйвера.

Все четыре проблемы проявляются одинаково: `driver status: signal: 9 (SIGKILL)` и `http status: 500`. Диагностировать их по отдельности сложно, поэтому применяйте все сразу.

---

## `package.json`

```json
{
  "name": "hello-wasm02",
  "version": "0.1.0",
  "description": "Rust + web_sys + Vite starter",
  "private": true,
  "type": "module",
  "scripts": {
    "wasm": "wasm-pack build --target web --out-dir pkg",
    "wasm:dev": "wasm-pack build --dev --target web --out-dir pkg",
    "test:host": "cargo test",
    "test:doc": "cargo test --doc",
    "test:node": "wasm-pack test --node",
    "test:chrome": "wasm-pack test --headless --chrome",
    "test:firefox": "xvfb-run -a wasm-pack test --headless --firefox --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver",
    "test": "npm run test:host && npm run test:node && npm run test:chrome",
    "test:all": "npm run test:host && npm run test:node && npm run test:chrome && npm run test:firefox",
    "dev": "vite",
    "build": "npm run wasm && vite build",
    "preview": "vite preview",
    "start": "npm run wasm:dev && vite"
  },
  "devDependencies": {
    "vite": "^8.2.2"
  }
}
```

**Почему `test` не включает Firefox.** Chrome headless работает быстрее и надёжнее в WSL2 и CI. Firefox-тесты запускаются отдельно (`test:firefox`) — перед важными коммитами или в CI. Полный прогон — `test:all`.

---

## `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: wasm32-unknown-unknown

      - name: Install wasm-pack
        run: curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

      - name: Cache cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/bin
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('Cargo.lock') }}

      - name: Install Chrome
        uses: browser-actions/setup-chrome@latest

      - name: Host tests (pure Rust)
        run: cargo test

      - name: Doc tests (compile-only for Wasm APIs)
        run: cargo test --doc

      - name: WASM tests in Node.js
        run: wasm-pack test --node

      - name: WASM tests in headless Chrome
        run: wasm-pack test --headless --chrome

      - name: Build WASM (release)
        run: wasm-pack build --release --target web --out-dir pkg

      - name: Install Node dependencies
        run: npm ci

      - name: Build frontend
        run: npm run build

  test-firefox:
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: wasm32-unknown-unknown

      - name: Install wasm-pack
        run: curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

      - name: Install Firefox
        uses: browser-actions/setup-firefox@latest

      - name: WASM tests in headless Firefox
        run: wasm-pack test --headless --firefox
```

**Ключевые моменты:**

- Основной job использует Chrome — стабильно в CI.
- Firefox-job отдельный и `continue-on-error: true`.
- `webdriver.json` в корне обеспечивает `--no-sandbox` для обоих браузеров.
- В CI `xvfb-run` не нужен: GitHub Actions предоставляет X-сервер виртуально через `xvfb`, который уже активирован для `browser-actions/setup-firefox`.

---

## Команды запуска

### Разработка

```bash
npm start
```

### Тесты

```bash
# Хостовые тесты (чистый Rust)
npm run test:host

# Только doc-тесты
npm run test:doc

# Wasm-тесты логики в Node.js
npm run test:node

# DOM-тесты в headless Chrome (основной browser-таргет)
npm run test:chrome

# DOM-тесты в headless Firefox (WSL2, требует xvfb + симлинк)
npm run test:firefox

# Основной набор: host + node + chrome
npm test

# Полный прогон: host + node + chrome + firefox
npm run test:all
```

### Диагностика

```bash
# Проверить компиляцию без запуска
cargo check --target wasm32-unknown-unknown

# Собрать тесты без запуска
cargo test --no-run

# Проверить, что Chrome for Testing установлен
chrome --headless --version

# Проверить, что firefox-esr на месте и в PATH
which firefox
firefox --headless --version

# Активные features
cargo tree -e features
```

---

## Как выглядит успешный прогон

После установки браузеров `npm test` выводит:

```
cargo test:
  pure_logic.rs ......... 2 passed
  wasm_*.rs ............. 0 tests (skip)
  Doc-tests:
    dom::window ......... ok (compile-only)

wasm-pack test --node:
  wasm_logic.rs ......... 1 passed

wasm-pack test --headless --chrome:
  wasm_async.rs ......... 2 passed
  wasm_dom.rs ........... 5 passed
  wasm_events.rs ........ 1 passed
  wasm_logic.rs ......... skip (configured for Node.js)
```

**Итого: 11 выполненных тестов + 1 compile-only doc-тест.** Файлы, сконфигурированные для другого окружения, пропускаются — это норма.

---

## Типичные проблемы и их решения

**1. `cannot find Chrome binary` (WSL2).** Chrome из Windows не виден из WSL2. Установите Chrome for Testing внутри WSL2.

**2. `http status: 500` и SIGKILL для geckodriver.** Firefox не стартует в headless-режиме. Причины: Snap-версия, отсутствие `xvfb`, отсутствие симлинка `firefox` → `firefox-esr`, несовместимый `geckodriver`. Решения: установить `firefox-esr` из PPA, создать симлинк, установить `xvfb`, запускать через `xvfb-run -a`, явно указать `--geckodriver`.

**3. `this test suite is only configured to run in a browser, but we're only testing node.js tests so skipping`.** Ожидаемое поведение, если в файле есть `run_in_browser`, а вы запускаете `--node`. Аналогично для обратного случая. Это не ошибка.

**4. `0 tests to run` в `cargo test`.** Ожидаемо для `#[wasm_bindgen_test]`. Хостовые тесты должны быть в `pure_logic.rs` с обычным `#[test]`.

**5. `warning: function ... is never used` в `common.rs`.** Решение: `#![allow(dead_code)]` в начале файла.

**6. `no method named 'children' found`.** Не включена feature `HtmlCollection`. Feature нужна для возвращаемого типа.

**7. `unresolved import 'wasm_bindgen_futures'` или `'js_sys'`.** Добавьте их в `Cargo.toml` явно.

**8. `systemctl is-system-running: offline` в WSL2.** systemd отключён. Snap-пакеты не работают. Используйте `.deb`-пакеты (Firefox ESR из PPA) вместо Snap.

**9. `cannot communicate with server: ... snapd.socket`.** Snap-демон не запущен. Не пытайтесь установить Firefox через Snap в WSL2 — используйте PPA.

**10. `module 'dom' is private` в doc-тесте.** Модуль `dom` объявлен как `mod dom;` в `src/lib.rs`. Doc-тест компилируется как отдельный crate и не видит приватные модули. Решение: `pub mod dom;`.

**11. `cannot access imported statics on non-wasm targets` в doc-тесте.** Doc-тесты запускаются на хосте, а `web_sys::window()` использует статики `js-sys`, которых на хосте нет. Решение: `no_run` в блоке кода doc-теста.

---

## Практические рекомендации

1. **Три уровня тестов — три команды.** Хостовые (`cargo test`), Wasm-логика (`wasm-pack test --node`), DOM (`wasm-pack test --headless --chrome`). Не смешивайте их в одном файле.

2. **Не используйте `#[test]` для Wasm-тестов.** Всегда `#[wasm_bindgen_test]`.

3. **`run_in_browser` — только для DOM-тестов.** Наличие директивы в логическом тесте приведёт к пропуску в Node.js.

4. **Doc-тесты для браузерозависимого кода — с `no_run`.** Компилируется, но не выполняется. Реальные проверки — в `tests/wasm_dom.rs` с `#[wasm_bindgen_test]`.

5. **`pub mod dom;` в `lib.rs`.** Нужно, чтобы doc-тесты и внешние интеграционные тесты видели модуль.

6. **Управляйте временем жизни `Closure`.** В тестах `forget()` допустим, в production — нет.

7. **Chrome — основной browser-таргет.** Firefox в headless-режиме требует больше усилий и чаще ломается в WSL2.

8. **Все четыре шага для Firefox обязательны.** PPA + симлинк + `xvfb` + явный `--geckodriver`. Пропуск любого из них приводит к одной и той же ошибке.

9. **`webdriver.json` в корне проекта.** Добавляет `--no-sandbox` и другие флаги, которые `wasm-pack` не передаёт.

10. **Правило feature.** Если метод возвращает тип `X`, нужна feature `X`. Документация `web_sys` для каждого метода указывает полный список.

---

**Итог главы:**

- `wasm-bindgen-test` — тестовый harness для Wasm; `#[wasm_bindgen_test]` заменяет `#[test]`.
- Тесты делятся на три уровня: хостовые (`cargo test`), Wasm-логика (`--node`), DOM (`--headless --chrome`).
- DOM-тесты требуют `wasm_bindgen_test_configure!(run_in_browser)`.
- Async-тесты — обычные `async fn` с `#[wasm_bindgen_test]`.
- Doc-тесты для браузерозависимого кода — только с `no_run`; для этого модуль должен быть публичным (`pub mod dom;`).
- В WSL2 Chrome for Testing устанавливается вручную; Firefox — через PPA + симлинк + `xvfb` + явный `--geckodriver`.
- `webdriver.json` решает проблемы с sandbox.
- `#![allow(dead_code)]` в `common.rs` убирает ложные warnings.
- Каждый тип тестов запускается в своём окружении; пропуск «чужих» тестов — норма.
- Chrome — основной таргет; Firefox — для кросс-браузерной проверки.

---

**Что дальше.** В главе 7 мы работаем с `Window`, `Document` и DOM: поиск элементов, создание, атрибуты, классы и практический проект — «Rust управляет HTML».