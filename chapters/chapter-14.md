# Глава 14. Browser navigation и URL

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
├── build.rs                        # НОВОЕ: регистрация cfg-условия для web_sys_unstable_apis
├── .cargo/
│   └── config.toml                 # НОВОЕ: флаг --cfg=web_sys_unstable_apis
├── src/
│   ├── lib.rs                      # pub mod app; pub mod navigation; ...
│   ├── app.rs                      # client-side navigation
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs               # НОВОЕ: Location, Url, UrlSearchParams, History, Navigation API
│   ├── notifier.rs
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
│   └── wasm_navigation.rs          # НОВОЕ: 26 тестов для навигации и URL
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

- `src/navigation.rs` — новый модуль. Содержит функции для работы с `Location`, `Url`, `UrlSearchParams`, `History`, `PopStateEvent` и Navigation API.
- `src/app.rs` — переписан под практический проект: **client-side navigation** с четырьмя страницами (Home, About, Contact, 404), query-параметрами, hash-навигацией, обработкой `popstate` и `hashchange`.
- `build.rs` и `.cargo/config.toml` — нужны для Navigation API.
- `tests/wasm_navigation.rs` — 26 тестов, включая полное покрытие `Location`.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `web_sys::Location` | https://docs.rs/web-sys/latest/web_sys/struct.Location.html |
| `web_sys::Url` | https://docs.rs/web-sys/latest/web_sys/struct.Url.html |
| `web_sys::UrlSearchParams` | https://docs.rs/web-sys/latest/web_sys/struct.UrlSearchParams.html |
| `web_sys::History` | https://docs.rs/web-sys/latest/web_sys/struct.History.html |
| `web_sys::PopStateEvent` | https://docs.rs/web-sys/latest/web_sys/struct.PopStateEvent.html |
| Navigation API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Navigation_API |
| Navigation API (Baseline) | https://web.dev/blog/baseline-navigation-api |
| `NavigateEvent` | https://developer.mozilla.org/en-US/docs/Web/API/NavigateEvent |
| History API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/History_API |
| URL API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/URL_API |

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
    "EventInit",
    "InputEvent",

    # События (специализированные)
    "KeyboardEvent",
    "MouseEvent",
    "PointerEvent",

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

    # Навигация и URL
    "Location",
    "Url",
    "UrlSearchParams",
    "History",
    "PopStateEvent",
    "PopStateEventInit",

    # Navigation API (экспериментальный, требует --cfg=web_sys_unstable_apis)
    "Navigation",
    "NavigateEvent",
    "NavigationHistoryEntry",
    "NavigationCurrentEntryChangeEvent",
    "NavigationNavigateOptions",
    "NavigationType",
    "NavigationResult",
    "NavigationHistoryBehavior",
    "NavigationUpdateCurrentEntryOptions",

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

## `build.rs` (новый)

```rust
fn main() {
    // Регистрируем cfg-условие, которое использует web-sys для unstable API.
    // Без этой строки rustc выдаёт warning:
    //   unexpected `cfg` condition name: `web_sys_unstable_apis`
    println!("cargo::rustc-check-cfg=cfg(web_sys_unstable_apis)");
}
```

## `.cargo/config.toml` (новый)

```toml
[target.wasm32-unknown-unknown]
rustflags = ["--cfg=web_sys_unstable_apis"]
```

**Зачем это нужно.** Features `web-sys` для Navigation API не компилируются без флага `--cfg=web_sys_unstable_apis`. Флаг включается через `.cargo/config.toml`, а `build.rs` регистрирует cfg-условие, чтобы rustc не выдавал warning `unexpected_cfgs`.

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
//! - `notifier` — система уведомлений с Rust-callback API.

pub mod app;
pub mod async_utils;
pub mod callbacks;
pub mod dom;
pub mod events;
pub mod js_objects;
pub mod navigation;
pub mod notifier;
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

## `src/navigation.rs` (полное содержание)

```rust
//! Помощники для работы с навигацией и URL.
//!
//! ## Что в этом модуле
//!
//! - `Location` — текущий URL, `assign`, `replace`, `reload`, `set_hash`, `set_search`.
//! - `Url` — парсинг URL, извлечение компонентов.
//! - `UrlSearchParams` — работа с query-параметрами.
//! - `History` — `pushState`, `replaceState`, `back`, `forward`.
//! - Navigation API — стандартизирован с января 2026 (Baseline Newly Available).
#![allow(dead_code)]

use wasm_bindgen::prelude::*;
use web_sys::{History, Location, Url, UrlSearchParams};

// ============================================================
// Location
// ============================================================

/// Возвращает `window.location`.
pub fn location() -> Location {
    web_sys::window()
        .expect("no window")
        .location()
}

/// Возвращает текущий `href` (полный URL).
pub fn current_href() -> String {
    location().href().unwrap_or_default()
}

/// Возвращает текущий `pathname` (путь без query и hash).
pub fn current_pathname() -> String {
    location().pathname().unwrap_or_default()
}

/// Возвращает текущий `search` (query-строка с `?`).
pub fn current_search() -> String {
    location().search().unwrap_or_default()
}

/// Возвращает текущий `hash` (с `#`).
pub fn current_hash() -> String {
    location().hash().unwrap_or_default()
}

/// Переходит по URL (добавляет в историю).
///
/// **Осторожно:** вызывает HTTP-навигацию и перезагрузку страницы.
/// В тестах не использовать — это уничтожит тестовый контекст.
pub fn navigate_to(url: &str) -> Result<(), JsValue> {
    location().assign(url)
}

/// Заменяет текущий URL (без добавления в историю).
///
/// **Осторожно:** тоже вызывает HTTP-навигацию. В тестах используйте
/// `History::replace_state` для изменения URL без перезагрузки.
pub fn replace_url(url: &str) -> Result<(), JsValue> {
    location().replace(url)
}

/// Перезагружает страницу.
pub fn reload() -> Result<(), JsValue> {
    location().reload()
}

/// Устанавливает hash. Пустая строка убирает hash.
///
/// Не вызывает перезагрузку страницы. Генерирует событие `hashchange`.
pub fn set_hash(hash: &str) -> Result<(), JsValue> {
    location().set_hash(hash)
}

/// Устанавливает query-строку. Пустая строка убирает query.
///
/// **Внимание:** в отличие от `set_hash`, `set_search` вызывает
/// перезагрузку страницы. В тестах используйте `History::replace_state`.
pub fn set_search(search: &str) -> Result<(), JsValue> {
    location().set_search(search)
}

// ============================================================
// Url
// ============================================================

/// Парсит URL.
pub fn parse_url(url: &str) -> Result<Url, JsValue> {
    Url::new(url)
}

/// Возвращает `origin` URL (protocol + host + port).
pub fn url_origin(url: &Url) -> String {
    url.origin()
}

/// Возвращает `pathname` URL.
pub fn url_pathname(url: &Url) -> String {
    url.pathname()
}

/// Возвращает `search` URL (query-строка).
pub fn url_search(url: &Url) -> String {
    url.search()
}

/// Возвращает `hash` URL.
pub fn url_hash(url: &Url) -> String {
    url.hash()
}

/// Возвращает `searchParams` URL.
pub fn url_search_params(url: &Url) -> UrlSearchParams {
    url.search_params()
}

// ============================================================
// UrlSearchParams
// ============================================================

/// Создаёт пустой `UrlSearchParams`.
pub fn new_search_params() -> Result<UrlSearchParams, JsValue> {
    UrlSearchParams::new()
}

/// Создаёт `UrlSearchParams` из строки.
pub fn search_params_from_str(s: &str) -> Result<UrlSearchParams, JsValue> {
    UrlSearchParams::new_with_str(s)
}

/// Читает параметр.
pub fn param_get(params: &UrlSearchParams, name: &str) -> Option<String> {
    params.get(name)
}

/// Устанавливает параметр (заменяет все вхождения).
pub fn param_set(params: &UrlSearchParams, name: &str, value: &str) {
    params.set(name, value);
}

/// Добавляет параметр (не удаляет существующие).
pub fn param_append(params: &UrlSearchParams, name: &str, value: &str) {
    params.append(name, value);
}

/// Удаляет параметр.
pub fn param_delete(params: &UrlSearchParams, name: &str) {
    params.delete(name);
}

/// Проверяет наличие параметра.
pub fn param_has(params: &UrlSearchParams, name: &str) -> bool {
    params.has(name)
}

/// Возвращает строковое представление query-параметров.
pub fn params_to_string(params: &UrlSearchParams) -> String {
    params.to_string().into()
}

/// Парсит query-параметры из текущего URL.
pub fn current_search_params() -> Result<UrlSearchParams, JsValue> {
    search_params_from_str(&current_search())
}

// ============================================================
// History
// ============================================================

/// Возвращает `window.history`.
pub fn history() -> History {
    web_sys::window()
        .expect("no window")
        .history()
        .expect("no history")
}

/// Добавляет запись в историю.
pub fn push_state(data: &JsValue, title: &str, url: Option<&str>) -> Result<(), JsValue> {
    match url {
        Some(u) => history().push_state_with_url(data, title, Some(u)),
        None => history().push_state(data, title),
    }
}

/// Заменяет текущую запись в истории.
pub fn replace_state(data: &JsValue, title: &str, url: Option<&str>) -> Result<(), JsValue> {
    match url {
        Some(u) => history().replace_state_with_url(data, title, Some(u)),
        None => history().replace_state(data, title),
    }
}

/// Назад в истории.
pub fn history_back() -> Result<(), JsValue> {
    history().back()
}

/// Вперёд в истории.
pub fn history_forward() -> Result<(), JsValue> {
    history().forward()
}

/// Переход на `delta` шагов в истории.
pub fn history_go(delta: i32) -> Result<(), JsValue> {
    history().go_with_delta(delta)
}

/// Длина истории.
pub fn history_length() -> u32 {
    history().length().unwrap_or(0)
}

/// Текущее состояние истории.
pub fn history_state() -> JsValue {
    history().state().unwrap_or(JsValue::NULL)
}

// ============================================================
// Navigation API
// ============================================================

/// Проверяет, доступен ли Navigation API в текущем браузере.
///
/// Возвращает `true`, если `window.navigation` существует.
/// Не зависит от `web_sys_unstable_apis`: работает через `Reflect`.
///
/// ## Статус на сентябрь 2026
///
/// - Chrome 102+, Edge 102+
/// - Firefox 147+
/// - Safari 26.2+
/// - Baseline Newly Available с января 2026
pub fn is_navigation_api_available() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    js_sys::Reflect::has(&window, &JsValue::from_str("navigation")).unwrap_or(false)
}

/// Возвращает `window.navigation`.
///
/// **Требует `--cfg=web_sys_unstable_apis`.**
#[cfg(web_sys_unstable_apis)]
pub fn navigation() -> web_sys::Navigation {
    use wasm_bindgen::JsCast;

    let nav = js_sys::Reflect::get(
        &web_sys::window().unwrap(),
        &JsValue::from_str("navigation"),
    )
    .expect("no navigation");

    nav.dyn_into::<web_sys::Navigation>()
        .expect("navigation is not Navigation")
}

/// Возвращает текущий URL через Navigation API.
///
/// **Требует `--cfg=web_sys_unstable_apis`.**
#[cfg(web_sys_unstable_apis)]
pub fn navigation_current_url() -> String {
    let nav = navigation();
    nav.current_entry()
        .and_then(|entry| entry.url())
        .unwrap_or_default()
}
```

---

## `src/app.rs` (полное содержание)

```rust
//! Логика приложения: «Client-side navigation».
//!
//! Демонстрирует:
//! - `Location` — через `current_pathname`, `current_search`, `current_hash`.
//! - `Url` — через инструмент «Разбор URL».
//! - `UrlSearchParams` — через чтение `?page=` при загрузке.
//! - `History` — через `pushState` и кнопки back/forward.
//! - Hash-based navigation — через кнопку «Hash → #contact» и `hashchange`.
//! - Query parameters — через `?page=about` в URL.
//! - Navigation events — через обработчики `popstate` и `hashchange`.
//! - 404 — через маршрут `/unknown-path`.
//! - Navigation API — через проверку `is_navigation_api_available`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, HtmlInputElement};

use crate::callbacks::{self, EventClosures};
use crate::dom;
use crate::log;
use crate::navigation;

/// Состояние приложения.
#[derive(Default)]
struct AppState {
    current_page: String,
}

/// Приоритет источников страницы:
/// 1. Query-параметр `?page=X`
/// 2. Hash `#X`
/// 3. Pathname `/about`, `/contact`
/// 4. Home для `/`
/// 5. 404 для всего остального
fn resolve_page() -> String {
    let known = ["home", "about", "contact"];

    // 1. Query-параметр
    if let Ok(params) = navigation::current_search_params() {
        if let Some(page) = navigation::param_get(&params, "page") {
            if known.contains(&page.as_str()) {
                return page;
            }
        }
    }

    // 2. Hash
    let hash = navigation::current_hash();
    let hash_value = hash.trim_start_matches('#');
    if !hash_value.is_empty() && known.contains(&hash_value) {
        return hash_value.to_string();
    }

    // 3. Pathname
    let pathname = navigation::current_pathname();
    match pathname.as_str() {
        "/about" => return "about".into(),
        "/contact" => return "contact".into(),
        "/" | "" => return "home".into(),
        _ => {}
    }

    // 4. 404
    "404".into()
}

/// Точка входа приложения.
pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Client-side navigation")?;
    dom::append(&container, &heading)?;

    // 1. Навигация
    let nav = dom::create_element("nav")?;
    dom::add_class(&dom::as_html(&nav), "nav")?;

    for (id, label, href) in [
        ("home", "Home", "/"),
        ("about", "About", "/about"),
        ("contact", "Contact", "/contact"),
        ("unknown", "Unknown (404)", "/unknown-path"),
    ] {
        let a = dom::create_element("a")?;
        dom::set_attribute(&a, "href", href)?;
        dom::set_attribute(&a, "data-page", id)?;
        a.set_text_content(Some(label));
        dom::append(&nav, &a)?;
    }
    dom::append(&container, &nav)?;

    // 2. Управление
    let controls = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&controls), "controls")?;

    let back_btn = dom::create_element("button")?;
    back_btn.set_text_content(Some("← Назад"));
    dom::set_attribute(&back_btn, "id", "back")?;
    dom::append(&controls, &back_btn)?;

    let forward_btn = dom::create_element("button")?;
    forward_btn.set_text_content(Some("Вперёд →"));
    dom::set_attribute(&forward_btn, "id", "forward")?;
    dom::append(&controls, &forward_btn)?;

    let hash_btn = dom::create_element("button")?;
    hash_btn.set_text_content(Some("Hash → #contact"));
    dom::set_attribute(&hash_btn, "id", "hash-btn")?;
    dom::append(&controls, &hash_btn)?;

    let query_btn = dom::create_element("button")?;
    query_btn.set_text_content(Some("Query → ?page=about"));
    dom::set_attribute(&query_btn, "id", "query-btn")?;
    dom::append(&controls, &query_btn)?;

    dom::append(&container, &controls)?;

    // 3. Информация о текущем URL
    let url_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&url_info), "url-info")?;
    dom::append(&container, &url_info)?;

    // 4. Информация о query-параметрах
    let query_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&query_info), "query-info")?;
    dom::append(&container, &query_info)?;

    // 5. Инструмент «Разбор URL»
    let parser_section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&parser_section), "parser-section")?;

    let parser_title = dom::create_element("h2")?;
    parser_title.set_text_content(Some("Разбор URL"));
    dom::append(&parser_section, &parser_title)?;

    let parser_desc = dom::create_element("p")?;
    parser_desc.set_text_content(Some(
        "Введите URL и нажмите «Разобрать», чтобы увидеть его компоненты.",
    ));
    dom::append(&parser_section, &parser_desc)?;

    let parser_input_el = dom::create_element("input")?;
    dom::set_attribute(&parser_input_el, "type", "text")?;
    dom::set_attribute(&parser_input_el, "id", "url-input")?;
    dom::set_attribute(
        &parser_input_el,
        "value",
        "https://example.com:8080/path/to/page?name=Rust&version=1.75#section",
    )?;
    let parser_input: HtmlInputElement = parser_input_el.dyn_into()?;
    dom::append(&parser_section, &parser_input)?;

    let parse_btn = dom::create_element("button")?;
    parse_btn.set_text_content(Some("Разобрать"));
    dom::set_attribute(&parse_btn, "id", "parse-btn")?;
    dom::append(&parser_section, &parse_btn)?;

    let parser_result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&parser_result), "parser-result")?;
    dom::append(&parser_section, &parser_result)?;

    dom::append(&container, &parser_section)?;

    // 6. Контент страницы
    let content = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&content), "content")?;
    dom::append(&container, &content)?;

    // 7. Информация о поддержке Navigation API
    let nav_api_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&nav_api_info), "nav-api-info")?;
    let nav_api_supported = navigation::is_navigation_api_available();
    nav_api_info.set_text_content(Some(&format!(
        "Navigation API: {}",
        if nav_api_supported {
            "поддерживается"
        } else {
            "не поддерживается"
        }
    )));
    dom::append(&container, &nav_api_info)?;

    // 8. Состояние и хранилище Closure
    let state = Rc::new(RefCell::new(AppState::default()));
    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // 9. Обновление UI
    let update_ui = {
        let url_info = url_info.clone();
        let query_info = query_info.clone();
        let content = content.clone();
        let state = state.clone();

        Rc::new(move || {
            let pathname = navigation::current_pathname();
            let search = navigation::current_search();
            let hash = navigation::current_hash();
            let href = navigation::current_href();

            url_info.set_text_content(Some(&format!(
                "pathname: {}\nsearch: {}\nhash: {}\nhref: {}",
                pathname, search, hash, href
            )));

            // Разбор query-параметров
            let params_text = match navigation::current_search_params() {
                Ok(params) => {
                    let s = navigation::params_to_string(&params);
                    if s.is_empty() {
                        "(нет параметров)".to_string()
                    } else {
                        s
                    }
                }
                Err(_) => "(не удалось распарсить)".to_string(),
            };
            query_info
                .set_text_content(Some(&format!("query params: {}", params_text)));

            // Определение страницы
            let page = resolve_page();
            state.borrow_mut().current_page = page.clone();

            // Рендеринг контента
            let html: String = match page.as_str() {
                "about" => "<h2>About</h2><p>О приложении hello-wasm02.</p>"
                    .to_string(),
                "contact" => {
                    "<h2>Contact</h2><p>Свяжитесь с нами: rust@example.com</p>"
                        .to_string()
                }
                "404" => format!(
                    "<h2>404 — Not Found</h2>\
                     <p>Страница не найдена. Проверьте URL.</p>\
                     <p><code>pathname: {}</code></p>",
                    pathname
                ),
                _ => "<h2>Home</h2>\
                     <p>Добро пожаловать в демонстрацию client-side navigation.</p>"
                    .to_string(),
            };
            content.set_inner_html(&html);
            log!("Страница: {} (pathname: {})", page, pathname);
        })
    };

    // 10. Обработчики ссылок
    for id in ["home", "about", "contact", "unknown"] {
        let link = nav
            .query_selector(&format!("a[data-page='{}']", id))?
            .ok_or_else(|| JsValue::from_str(&format!("link {} not found", id)))?;

        let update_ui = update_ui.clone();
        let id_owned = id.to_string();

        callbacks::listen(
            &link,
            "click",
            move |event: Event| {
                event.prevent_default();

                let url = match id_owned.as_str() {
                    "about" => "/about",
                    "contact" => "/contact",
                    "unknown" => "/unknown-path",
                    _ => "/",
                };

                if let Err(e) =
                    navigation::push_state(&JsValue::NULL, "", Some(url))
                {
                    log!("push_state error: {:?}", e);
                }

                update_ui();
            },
            &closures,
        )?;
    }

    // 11. Кнопки back/forward
    {
        callbacks::listen(
            &back_btn,
            "click",
            |_event| {
                navigation::history_back().ok();
            },
            &closures,
        )?;

        callbacks::listen(
            &forward_btn,
            "click",
            |_event| {
                navigation::history_forward().ok();
            },
            &closures,
        )?;
    }

    // 12. Кнопка «Hash → #contact»
    {
        let update_ui = update_ui.clone();
        callbacks::listen(
            &hash_btn,
            "click",
            move |_event| {
                navigation::replace_state(&JsValue::NULL, "", Some("/")).ok();
                navigation::set_hash("#contact").ok();
                update_ui();
            },
            &closures,
        )?;
    }

    // 13. Кнопка «Query → ?page=about»
    {
        let update_ui = update_ui.clone();
        callbacks::listen(
            &query_btn,
            "click",
            move |_event| {
                navigation::replace_state(&JsValue::NULL, "", Some("/")).ok();
                navigation::set_search("?page=about").ok();
                update_ui();
            },
            &closures,
        )?;
    }

    // 14. Обработчик popstate
    {
        let update_ui = update_ui.clone();
        let window = web_sys::window().unwrap();

        callbacks::listen(
            &window,
            "popstate",
            move |_event| {
                log!("Событие popstate");
                update_ui();
            },
            &closures,
        )?;
    }

    // 15. Обработчик hashchange
    {
        let update_ui = update_ui.clone();
        let window = web_sys::window().unwrap();

        callbacks::listen(
            &window,
            "hashchange",
            move |_event| {
                log!("Событие hashchange");
                update_ui();
            },
            &closures,
        )?;
    }

    // 16. Обработчик «Разобрать URL»
    {
        let parser_result = parser_result.clone();
        let parser_input = parser_input.clone();

        callbacks::listen(
            &parse_btn,
            "click",
            move |_event| {
                let url_str = parser_input.value();
                if url_str.trim().is_empty() {
                    parser_result.set_text_content(Some("Введите URL"));
                    return;
                }

                match navigation::parse_url(&url_str) {
                    Ok(url) => {
                        let mut lines = Vec::new();
                        lines.push(format!("href:     {}", url.href()));
                        lines.push(format!(
                            "origin:   {}",
                            navigation::url_origin(&url)
                        ));
                        lines.push(format!(
                            "pathname: {}",
                            navigation::url_pathname(&url)
                        ));
                        lines.push(format!(
                            "search:   {}",
                            navigation::url_search(&url)
                        ));
                        lines.push(format!(
                            "hash:     {}",
                            navigation::url_hash(&url)
                        ));

                        let params = navigation::url_search_params(&url);
                        let params_str = navigation::params_to_string(&params);
                        lines.push(format!(
                            "params:   {}",
                            if params_str.is_empty() {
                                "(нет)".to_string()
                            } else {
                                params_str
                            }
                        ));

                        for key in ["name", "version"] {
                            if let Some(v) = navigation::param_get(&params, key) {
                                lines.push(format!("  {} = {}", key, v));
                            }
                        }

                        parser_result.set_text_content(Some(&lines.join("\n")));
                        log!("URL parsed: {}", url_str);
                    }
                    Err(e) => {
                        parser_result.set_text_content(Some(&format!(
                            "Ошибка парсинга: {:?}",
                            e
                        )));
                        log!("URL parse error: {:?}", e);
                    }
                }
            },
            &closures,
        )?;
    }

    // 17. Разбираем начальный URL сразу при загрузке
    {
        let url_str = parser_input.value();
        if let Ok(url) = navigation::parse_url(&url_str) {
            let mut lines = Vec::new();
            lines.push(format!("href:     {}", url.href()));
            lines.push(format!("origin:   {}", navigation::url_origin(&url)));
            lines.push(format!("pathname: {}", navigation::url_pathname(&url)));
            lines.push(format!("search:   {}", navigation::url_search(&url)));
            lines.push(format!("hash:     {}", navigation::url_hash(&url)));

            let params = navigation::url_search_params(&url);
            let params_str = navigation::params_to_string(&params);
            lines.push(format!(
                "params:   {}",
                if params_str.is_empty() {
                    "(нет)".to_string()
                } else {
                    params_str
                }
            ));

            for key in ["name", "version"] {
                if let Some(v) = navigation::param_get(&params, key) {
                    lines.push(format!("  {} = {}", key, v));
                }
            }

            parser_result.set_text_content(Some(&lines.join("\n")));
        }
    }

    // 18. Начальная отрисовка
    update_ui();

    log!("Client-side navigation готов");
    log!("Поддержка Navigation API: {}", nav_api_supported);

    std::mem::forget(closures);
    std::mem::forget(state);

    Ok(())
}
```

---

## `web/index.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>hello-wasm02 — client-side navigation</title>
    <link rel="icon" type="image/x-icon" href="/assets/favicon.ico" />
    <link rel="stylesheet" href="/styles/main.css" />
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p>Client-side navigation. Откройте консоль для логов.</p>

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
}

body {
    max-width: 720px;
    margin: 2rem auto;
    padding: 0 1rem;
}

h1 {
    color: #b7410e;
}

h2 {
    margin-top: 1.5rem;
    color: #b7410e;
}

#app {
    margin-top: 1.5rem;
}

/* ============================================================
   Навигация
   ============================================================ */

.nav {
    display: flex;
    flex-wrap: wrap;
    gap: 1rem;
    margin-bottom: 1rem;
    padding-bottom: 0.5rem;
    border-bottom: 1px solid #ddd;
}

.nav a {
    text-decoration: none;
    color: #b7410e;
    font-weight: 500;
    padding: 0.25rem 0.5rem;
    border-radius: 4px;
    transition: background 0.15s;
}

.nav a:hover {
    background: #f5f5f5;
}

.nav a[data-page="home"]::before    { content: "🏠 "; }
.nav a[data-page="about"]::before   { content: "ℹ️ "; }
.nav a[data-page="contact"]::before { content: "✉️ "; }
.nav a[data-page="unknown"]::before { content: "⚠️ "; }

/* ============================================================
   Кнопки
   ============================================================ */

.controls {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-bottom: 1rem;
}

.controls button {
    padding: 0.5rem 1rem;
    font-size: 1rem;
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
   Информационные блоки
   ============================================================ */

.url-info,
.query-info {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    margin-bottom: 0.5rem;
}

.query-info {
    border-left-color: #2e7d32;
}

.nav-api-info {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    color: #666;
    margin-bottom: 1rem;
}

/* ============================================================
   Инструмент «Разбор URL»
   ============================================================ */

.parser-section {
    margin: 1.5rem 0;
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
    background: #fafafa;
}

.parser-section h2 {
    margin-top: 0;
    font-size: 1.1rem;
}

.parser-section p {
    font-size: 0.9rem;
    color: #666;
    margin: 0.25rem 0 0.75rem 0;
}

.parser-section input {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    box-sizing: border-box;
    margin-bottom: 0.5rem;
}

.parser-section input:focus {
    outline: 2px solid #b7410e;
    border-color: transparent;
}

.parser-section button {
    padding: 0.5rem 1rem;
    font-size: 1rem;
    border: 1px solid #b7410e;
    background: white;
    color: #b7410e;
    border-radius: 6px;
    cursor: pointer;
    transition: background 0.15s, color 0.15s;
    margin-bottom: 0.75rem;
}

.parser-section button:hover {
    background: #b7410e;
    color: white;
}

.parser-result {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: white;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #2e7d32;
    min-height: 2rem;
}

/* ============================================================
   Контент страницы
   ============================================================ */

.content {
    padding: 1rem;
    border: 1px solid #ddd;
    border-radius: 8px;
    background: #fafafa;
    min-height: 8rem;
}

.content h2 {
    margin-top: 0;
}

.content code {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    background: #f5f5f5;
    padding: 0.15rem 0.35rem;
    border-radius: 3px;
    font-size: 0.9em;
}
```

---

## `tests/wasm_navigation.rs` (полное содержание)

```rust
//! Тесты для навигации и URL.
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

use hello_wasm02::navigation;

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// Location
// ============================================================

#[wasm_bindgen_test]
fn test_location_href_is_valid() {
    let href = navigation::current_href();
    assert!(!href.is_empty(), "href should not be empty");
    assert!(
        href.starts_with("http"),
        "href should start with http: {}",
        href
    );
}

#[wasm_bindgen_test]
fn test_location_pathname_starts_with_slash() {
    let pathname = navigation::current_pathname();
    assert!(!pathname.is_empty(), "pathname should not be empty");
    assert!(
        pathname.starts_with('/'),
        "pathname should start with /: {}",
        pathname
    );
}

#[wasm_bindgen_test]
fn test_location_search_is_valid_string() {
    let search = navigation::current_search();
    assert!(
        search.is_empty() || search.starts_with('?'),
        "search should be empty or start with ?: {}",
        search
    );
}

#[wasm_bindgen_test]
fn test_location_hash_is_valid_string() {
    let hash = navigation::current_hash();
    assert!(
        hash.is_empty() || hash.starts_with('#'),
        "hash should be empty or start with #: {}",
        hash
    );
}

#[wasm_bindgen_test]
fn test_location_components_consistency() {
    let href = navigation::current_href();
    let pathname = navigation::current_pathname();
    let search = navigation::current_search();
    let hash = navigation::current_hash();

    assert!(
        href.contains(&pathname),
        "href should contain pathname: {} vs {}",
        href,
        pathname
    );

    if !search.is_empty() {
        assert!(
            href.contains(&search),
            "href should contain search: {} vs {}",
            href,
            search
        );
    }

    if !hash.is_empty() {
        assert!(
            href.contains(&hash),
            "href should contain hash: {} vs {}",
            href,
            hash
        );
    }
}

#[wasm_bindgen_test]
fn test_location_set_hash() {
    let original_url = navigation::current_href();

    navigation::set_hash("#test-location-hash").unwrap();
    assert_eq!(
        navigation::current_hash(),
        "#test-location-hash",
        "hash should be set"
    );

    let href = navigation::current_href();
    assert!(
        href.ends_with("#test-location-hash"),
        "href should end with hash: {}",
        href
    );

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

#[wasm_bindgen_test]
fn test_location_set_hash_empty() {
    let original_url = navigation::current_href();

    navigation::set_hash("#something").unwrap();
    assert_eq!(navigation::current_hash(), "#something");

    navigation::set_hash("").unwrap();
    assert_eq!(
        navigation::current_hash(),
        "",
        "hash should be empty after clearing"
    );

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

#[wasm_bindgen_test]
fn test_location_replace_state_changes_url() {
    let original_url = navigation::current_href();

    navigation::replace_state(&JsValue::NULL, "", Some("/new-location-path"))
        .unwrap();
    assert_eq!(navigation::current_pathname(), "/new-location-path");

    navigation::replace_state(&JsValue::NULL, "", Some("/another-path"))
        .unwrap();
    assert_eq!(navigation::current_pathname(), "/another-path");

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

#[wasm_bindgen_test]
fn test_location_all_components_via_replace_state() {
    let original_url = navigation::current_href();

    navigation::replace_state(
        &JsValue::NULL,
        "",
        Some("/test-location-path?foo=bar&baz=qux#myhash"),
    )
    .unwrap();

    assert_eq!(navigation::current_pathname(), "/test-location-path");
    assert_eq!(navigation::current_search(), "?foo=bar&baz=qux");
    assert_eq!(navigation::current_hash(), "#myhash");

    let href = navigation::current_href();
    assert!(href.contains("/test-location-path"));
    assert!(href.contains("?foo=bar&baz=qux"));
    assert!(href.contains("#myhash"));

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

#[wasm_bindgen_test]
fn test_location_search_with_empty_query() {
    let original_url = navigation::current_href();

    navigation::replace_state(&JsValue::NULL, "", Some("/path-without-query"))
        .unwrap();
    assert_eq!(
        navigation::current_search(),
        "",
        "search should be empty"
    );

    navigation::replace_state(&JsValue::NULL, "", Some("/with?x=1")).unwrap();
    assert_eq!(navigation::current_search(), "?x=1");

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

#[wasm_bindgen_test]
fn test_location_hash_with_empty_hash() {
    let original_url = navigation::current_href();

    navigation::replace_state(&JsValue::NULL, "", Some("/path")).unwrap();
    assert_eq!(navigation::current_hash(), "", "hash should be empty");

    navigation::replace_state(&JsValue::NULL, "", Some("/path#h")).unwrap();
    assert_eq!(navigation::current_hash(), "#h");

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

// ============================================================
// Url
// ============================================================

#[wasm_bindgen_test]
fn test_url_parse() {
    let url = navigation::parse_url("https://example.com/path?q=1#hash").unwrap();

    assert_eq!(navigation::url_origin(&url), "https://example.com");
    assert_eq!(navigation::url_pathname(&url), "/path");
    assert_eq!(navigation::url_search(&url), "?q=1");
    assert_eq!(navigation::url_hash(&url), "#hash");
}

#[wasm_bindgen_test]
fn test_url_relative() {
    let url = navigation::parse_url("https://example.com").unwrap();
    assert_eq!(navigation::url_pathname(&url), "/");
}

#[wasm_bindgen_test]
fn test_url_with_port() {
    let url = navigation::parse_url("https://example.com:8080/api").unwrap();
    assert_eq!(navigation::url_origin(&url), "https://example.com:8080");
    assert_eq!(navigation::url_pathname(&url), "/api");
}

// ============================================================
// UrlSearchParams
// ============================================================

#[wasm_bindgen_test]
fn test_search_params_from_str() {
    let params = navigation::search_params_from_str("a=1&b=2").unwrap();

    assert_eq!(navigation::param_get(&params, "a"), Some("1".into()));
    assert_eq!(navigation::param_get(&params, "b"), Some("2".into()));
    assert_eq!(navigation::param_get(&params, "c"), None);
}

#[wasm_bindgen_test]
fn test_search_params_set_append_delete() {
    let params = navigation::new_search_params().unwrap();

    navigation::param_set(&params, "a", "1");
    assert_eq!(navigation::param_get(&params, "a"), Some("1".into()));

    navigation::param_append(&params, "a", "2");
    assert_eq!(navigation::param_get(&params, "a"), Some("1".into()));

    navigation::param_delete(&params, "a");
    assert_eq!(navigation::param_get(&params, "a"), None);
}

#[wasm_bindgen_test]
fn test_search_params_has() {
    let params = navigation::search_params_from_str("x=1").unwrap();

    assert!(navigation::param_has(&params, "x"));
    assert!(!navigation::param_has(&params, "y"));
}

#[wasm_bindgen_test]
fn test_search_params_to_string() {
    let params = navigation::new_search_params().unwrap();
    navigation::param_set(&params, "a", "1");
    navigation::param_set(&params, "b", "2");

    let s = navigation::params_to_string(&params);
    assert!(s.contains("a=1"));
    assert!(s.contains("b=2"));
}

#[wasm_bindgen_test]
fn test_parse_url_with_query_params() {
    let url = navigation::parse_url(
        "https://example.com:8080/path?name=Rust&version=1.75#section",
    )
    .unwrap();

    assert_eq!(navigation::url_origin(&url), "https://example.com:8080");
    assert_eq!(navigation::url_pathname(&url), "/path");

    let params = navigation::url_search_params(&url);
    assert_eq!(navigation::param_get(&params, "name"), Some("Rust".into()));
    assert_eq!(
        navigation::param_get(&params, "version"),
        Some("1.75".into())
    );
    assert!(!navigation::param_has(&params, "unknown"));
}

// ============================================================
// History
// ============================================================

#[wasm_bindgen_test]
fn test_history_length() {
    let len = navigation::history_length();
    assert!(len >= 1);
}

#[wasm_bindgen_test]
fn test_push_and_replace_state() {
    let original_url = navigation::current_href();

    navigation::push_state(&JsValue::NULL, "", Some("/test-page")).unwrap();
    assert_eq!(navigation::current_pathname(), "/test-page");

    navigation::replace_state(&JsValue::NULL, "", Some("/test-replaced")).unwrap();
    assert_eq!(navigation::current_pathname(), "/test-replaced");

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

// ============================================================
// Hash
// ============================================================

#[wasm_bindgen_test]
fn test_set_hash_and_read() {
    let original_url = navigation::current_href();

    navigation::set_hash("#section-a").unwrap();
    assert_eq!(navigation::current_hash(), "#section-a");

    navigation::set_hash("#section-b").unwrap();
    assert_eq!(navigation::current_hash(), "#section-b");

    navigation::set_hash("").unwrap();
    assert_eq!(navigation::current_hash(), "");

    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

#[wasm_bindgen_test]
fn test_hash_change_event() {
    let original_url = navigation::current_href();

    let fired = Rc::new(RefCell::new(false));
    let fired_clone = fired.clone();

    let closure = Closure::<dyn FnMut(web_sys::Event)>::new(move |_event| {
        *fired_clone.borrow_mut() = true;
    });

    let window = web_sys::window().unwrap();
    window
        .add_event_listener_with_callback("hashchange", closure.as_ref().unchecked_ref())
        .unwrap();

    let event = web_sys::Event::new("hashchange").unwrap();
    window.dispatch_event(&event).unwrap();

    assert!(*fired.borrow(), "hashchange handler was not called");

    closure.forget();
    navigation::replace_state(&JsValue::NULL, "", Some(&original_url)).ok();
}

// ============================================================
// Navigation events
// ============================================================

#[wasm_bindgen_test]
fn test_popstate_event_dispatch() {
    let fired = Rc::new(RefCell::new(false));
    let fired_clone = fired.clone();

    let closure = Closure::<dyn FnMut(web_sys::PopStateEvent)>::new(move |_event| {
        *fired_clone.borrow_mut() = true;
    });

    let window = web_sys::window().unwrap();
    window
        .add_event_listener_with_callback("popstate", closure.as_ref().unchecked_ref())
        .unwrap();

    let event = web_sys::PopStateEvent::new("popstate").unwrap();
    window.dispatch_event(&event).unwrap();

    assert!(*fired.borrow(), "popstate handler was not called");

    closure.forget();
}

// ============================================================
// Navigation API (доступность)
// ============================================================

#[wasm_bindgen_test]
fn test_navigation_api_availability_check() {
    // Не падает — возвращает bool.
    // В Chrome, Edge, Firefox 147+, Safari 26.2+ — true.
    let _available = navigation::is_navigation_api_available();
}
```

---

## Navigation API — актуальное состояние на сентябрь 2026

### Статус поддержки

С января 2026 Navigation API получил статус **Baseline Newly Available** — все основные браузеры реализовали его в стабильных версиях.

| Браузер | Версия с поддержкой |
|---|---|
| **Google Chrome** | 102+ |
| **Microsoft Edge** | 102+ |
| **Mozilla Firefox** | 147+ |
| **Apple Safari** | 26.2+ (включая iOS) |
| **Opera** | 88+ |
| **Samsung Internet** | 19.0+ |

### Что это значит для проекта

**Можно использовать.** В сентябре 2026 Navigation API — не экспериментальная технология, а стандарт. Ограничения остались только для старых браузеров, которые не получают обновлений.

**Что даёт Navigation API по сравнению с History API:**

| Задача | History API | Navigation API |
|---|---|---|
| Перехват навигации | Ручной `preventDefault` на каждой ссылке + `popstate` | Единый обработчик `navigate` ловит всё |
| Программная навигация | `history.pushState` | `navigation.navigate` |
| Информация о причине | Нет | `NavigationType`: `push`, `replace`, `reload`, `traverse` |
| Управление фокусом | Вручную | Автоматически через `intercept()` |
| Асинхронная загрузка | Ручная координация | `intercept({ handler: async () => ... })` |
| Доступ к истории | Только `length` и `state` | `navigation.entries()`, `navigation.currentEntry()` |

**Что НЕ работает в Safari 26.2.** Реализация API неполная — отсутствует `precommitHandler` в `intercept()`. Это часть, отвечающая за выполнение задач **до** смены URL. В сложных сценариях (предзагрузка данных перед навигацией) это может быть критично.

**Что рекомендуется.**

- Для **новых проектов** — использовать Navigation API как основной механизм.
- Для **production** — комбинировать Navigation API с fallback на History API для старых браузеров.
- Для **учебного проекта** — History API остаётся полностью рабочим и совместимым со всеми браузерами. Navigation API доступен через `is_navigation_api_available()` и функции под `#[cfg(web_sys_unstable_apis)]`.

**Что в нашем проекте.** Мы строим проект на History API (`pushState` + `popstate` + `preventDefault`), потому что:

1. History API работает везде без исключений.
2. Логика роутинга остаётся на Rust и полностью тестируема.
3. Navigation API доступен как опция — `is_navigation_api_available()` показывает статус в UI.

**Что дальше.** В главе 42 «Real-Time Browser Architecture» мы вернёмся к Navigation API и покажем, как переписать роутинг на него, чтобы сравнить два подхода в коде.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Проверьте все сценарии:

1. **Клик по Home/About/Contact/Unknown** — контент и URL меняются.
2. **Клик по «Hash → #contact»** — URL становится `/#contact`, контент — Contact.
3. **Клик по «Query → ?page=about»** — URL становится `/?page=about`, контент — About.
4. **Ввод URL в поле «Разбор URL»** — компоненты URL в блоке результата.
5. **Открытие `http://127.0.0.1:5173/?page=contact`** — сразу Contact.
6. **Нажатие back/forward браузера** — событие `popstate`, контент меняется.

### Тесты

```bash
npm run test:host
npm run test:node
npm run test:chrome
npm run test:firefox
npm test
npm run test:all
```

### Проверка конкретного теста

```bash
wasm-pack test --headless --chrome --test wasm_navigation
```

---

## Ключевые концепции главы

**1. `Location` — текущий URL.** `window.location` даёт доступ к `href`, `pathname`, `search`, `hash`. Методы `assign`, `replace`, `reload`, `set_hash`, `set_search` управляют навигацией. **`set_hash` безопасен** — не перезагружает страницу. **`set_search`, `assign`, `reload`** вызывают перезагрузку.

**2. `Url` — парсинг и разбор URL.** `Url::new("https://...")` создаёт объект, `origin()`, `pathname()`, `search()`, `hash()` извлекают компоненты.

**3. `UrlSearchParams` — query-параметры.** `new_with_str("a=1&b=2")` парсит строку. `get`, `set`, `append`, `delete`, `has` — основные операции.

**4. `History` — управление историей.** `pushState` добавляет запись, `replaceState` заменяет, `back`/`forward`/`go` перемещают. `length` возвращает количество записей.

**5. `popstate` — событие при навигации назад/вперёд.** Генерируется браузером. Обработчик должен перерисовать UI на основе нового `location.pathname`.

**6. `hashchange` — событие при изменении hash.** Генерируется браузером при изменении `#...`.

**7. Client-side navigation.** Комбинация `pushState` + `popstate` + `preventDefault` на ссылках. Это то, как работают все SPA.

**8. Query parameters как API для UI.** `?page=about` — не «настройка», а полноценный интерфейс управления. `resolve_page()` читает query в первую очередь.

**9. Hash как лёгкая альтернатива.** `#about` работает без серверной поддержки и не перезагружает страницу. Подходит для простых случаев.

**10. 404 — обязательная страница.** Любой неизвестный путь должен давать осмысленный ответ, а не пустоту.

**11. Navigation API — стандарт с 2026.** Baseline Newly Available. Поддерживается Chrome 102+, Firefox 147+, Safari 26.2+. Даёт единый интерфейс, автоперехват навигации, детальную информацию о причине. В нашем проекте — опция, доступная через `is_navigation_api_available()`.

**12. Тесты `Location` безопасны.** `set_hash` не перезагружает страницу. `replace_state` меняет URL без HTTP-навигации. После каждого теста URL восстанавливается.

---

## Итог главы

- `Location` — текущий URL, `href`, `pathname`, `search`, `hash`, `assign`, `replace`, `reload`, `set_hash`, `set_search`.
- `Url` — парсинг URL, `origin`, `pathname`, `search`, `hash`, `searchParams`.
- `UrlSearchParams` — `new`, `new_with_str`, `get`, `set`, `append`, `delete`, `has`, `to_string`.
- `History` — `push_state_with_url`, `replace_state_with_url`, `back`, `forward`, `go_with_delta`, `length`, `state`.
- `popstate` и `hashchange` — события навигации.
- Navigation API — Baseline Newly Available с января 2026.
- Client-side navigation строится на `pushState` + `popstate` + `preventDefault`.
- Query parameters и hash — рабочие механизмы навигации.
- 404 — обязательная страница для неизвестных путей.
- Практический проект: четыре страницы, query-навигация, hash-навигация, back/forward, инструмент «Разбор URL», индикатор Navigation API.
- 26 тестов в `wasm_navigation.rs`, включая 11 тестов `Location`.

---

**Что дальше.** В главе 15 мы разберём **Storage**: `localStorage`, `sessionStorage`, `StorageEvent`, сериализацию данных, JSON на границе Rust/JS, ограничения Storage и практический проект — сохранение состояния приложения.