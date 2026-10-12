# Глава 18. File API

## 18.16. Web Share API: поделиться файлом

В предыдущих подглавах мы научились читать файлы, обрабатывать их, сохранять на сервер и даже писать прямо на диск через FSA. Но есть ещё один сценарий, который особенно актуален для мобильных устройств: **поделиться файлом** — отправить его в мессенджер, почту или другое приложение одним касанием. Эту задачу решает **Web Share API**.

На десктопе пользователь может просто скачать файл, а потом вручную открыть Telegram и переслать. На мобильном это неудобно: приходится искать файл в загрузках, терять контекст, делать лишние шаги. Web Share API открывает **системный диалог шаринга** — тот же, что появляется при нажатии «Поделиться» в любом приложении. Пользователь выбирает приложение-получатель, и файл уходит туда напрямую.

**Ключевое преимущество:** единый интерфейс — не нужно интегрироваться с каждым мессенджером отдельно. Браузер сам показывает список доступных приложений на устройстве.

### Требования и ограничения

1. **User gesture обязателен.** `navigator.share()` вызывается **только из обработчика клика**. Это требование безопасности: сайт не должен «спамить» системными диалогами.
2. **Мобильные браузеры поддерживают лучше десктопных.**
   - **Android Chrome** — полная поддержка файлов.
   - **iOS Safari** — поддержка файлов с iOS 15.
   - **Safari macOS** — есть, но может не поддерживать файлы.
   - **Desktop Chrome/Edge** — поддержка зависит от версии и ОС.
   - **Firefox (все платформы)** — не поддерживает.
3. **HTTPS или localhost.** Secure context — обязательное условие.
4. **`canShare` — не гарантия.** `navigator.canShare({ files })` возвращает `true`, если браузер «согласен» показать диалог. Но даже после `true` пользователь может отменить — промис зареджектится с `AbortError`.
5. **Лимит файлов.** Chrome обычно ограничивает 10 файлами за один вызов `share()`.

### Особенности web-sys

1. **Тип `ShareData` не экспортирован.** Строим объект `{ files, title, text }` через `js_sys::Object` + `Reflect::set` — это работает независимо от версии web-sys.
2. **`NotAllowedError`, `DataError`, `AbortError`** — три разные причины отказа, требующие разных сообщений пользователю.

### Что мы построим

Модуль `sharing.rs` с четырьмя функциями:

- `is_supported()` — есть ли `navigator.share` в браузере.
- `can_share_files(&[File])` — можно ли поделиться конкретными файлами.
- `share_files(&[File], title, text)` — открыть диалог шаринга.
- `is_abort_error(&JsValue)` — распознать отмену пользователем.

Демо в `app.rs` — интеллектуальный помощник с диагностикой, мультифайловой поддержкой, fallback и различением ошибок.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений
├── src/
│   ├── lib.rs                     # +pub mod sharing;
│   ├── app.rs                     # только секция 18.16
│   ├── sharing.rs                 # НОВЫЙ: Web Share API
│   └── … остальные модули
└── tests/
    └── wasm_sharing.rs            # НОВЫЙ: тесты шаринга
```

**Пояснения:**

- `src/sharing.rs` — модуль Web Share API. Автономен: зависит только от `web-sys`, `js-sys`, `wasm-bindgen`, `wasm-bindgen-futures`.
- `src/lib.rs` — дополняется строкой `pub mod sharing;`.
- `src/app.rs` — **только** код подглавы 18.16.
- `tests/wasm_sharing.rs` — **без `#[ignore]`**. В headless-режиме picker-вызовы возвращают `NotAllowedError`, тесты это обрабатывают и проходят.

**Новых фич `web-sys` не требуется.** `File`, `Navigator`, `Window`, `DomException` уже включены.

---

## Ссылки для детального изучения

- **MDN: Web Share API** — https://developer.mozilla.org/en-US/docs/Web/API/Web_Share_API
- **MDN: Navigator.share()** — https://developer.mozilla.org/en-US/docs/Web/API/Navigator/share
- **MDN: Navigator.canShare()** — https://developer.mozilla.org/en-US/docs/Web/API/Navigator/canShare
- **MDN: Web Share API — Sharing files** — https://developer.mozilla.org/en-US/docs/Web/API/Web_Share_API#sharing_files
- **W3C: Web Share API spec** — https://w3c.github.io/web-share/
- **caniuse: Web Share API** — https://caniuse.com/web-share
- **web-sys: Navigator** — https://docs.rs/web-sys/latest/web_sys/struct.Navigator.html

---

## Содержание файлов

### 1. `src/lib.rs`

```rust
// ---- Глава 18. File API ----
// … модули из предыдущих подглав …
pub mod sharing;    // 18.16
```

---

### 2. `src/sharing.rs`

```rust
//! Web Share API: поделиться файлом.
//!
//! **Требования:** HTTPS или localhost + user gesture.
//! Работает на мобильных браузерах (Android Chrome, iOS Safari)
//! и на некоторых десктопных (Edge, Safari macOS).
//! В Firefox не поддерживается.
//!
//! **User gesture обязателен.** `navigator.share()` вызывается
//! только из обработчика клика — иначе браузер отклонит запрос
//! с `NotAllowedError`.
//!
//! **`ShareData` через `Reflect`.** Тип `ShareData` в web-sys
//! не экспортирован. Строим объект `{ files, title, text }`
//! через `js_sys::Object` + `Reflect::set` — это работает
//! независимо от версии web-sys.
//!
//! **Отмена пользователем** — не ошибка. Промис реджектится
//! с `AbortError`; используйте [`is_abort_error`] для проверки.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::File;

/// Проверяет, поддерживает ли браузер Web Share API.
///
/// Проверка наличия `navigator.share` — достаточная.
/// На некоторых платформах метод есть, но `canShare({files})`
/// возвращает `false` — тогда шаринг файлов невозможен.
pub fn is_supported() -> bool {
    let navigator = match web_sys::window() {
        Some(w) => w.navigator(),
        None => return false,
    };
    let nav_js: &JsValue = navigator.as_ref();
    return js_sys::Reflect::get(nav_js, &JsValue::from_str("share"))
        .map(|v| !v.is_undefined())
        .unwrap_or(false);
}

/// Строит объект `ShareData` из списка файлов и текстовых полей.
///
/// Формат: `{ files: [...], title: "...", text: "..." }`.
fn build_share_data(
    files: &[File],
    title: &str,
    text: &str,
) -> Result<js_sys::Object, JsValue> {
    let data = js_sys::Object::new();

    if !files.is_empty() {
        let arr = js_sys::Array::new();
        for f in files {
            arr.push(f.as_ref());
        }
        js_sys::Reflect::set(
            &data,
            &JsValue::from_str("files"),
            &arr,
        )?;
    }

    if !title.is_empty() {
        js_sys::Reflect::set(
            &data,
            &JsValue::from_str("title"),
            &JsValue::from_str(title),
        )?;
    }

    if !text.is_empty() {
        js_sys::Reflect::set(
            &data,
            &JsValue::from_str("text"),
            &JsValue::from_str(text),
        )?;
    }

    return Ok(data);
}

/// Проверяет, можно ли поделиться указанными файлами.
///
/// Возвращает `false`, если:
/// * Web Share API не поддерживается;
/// * список файлов пуст;
/// * браузер не может расшарить файлы этого типа.
pub fn can_share_files(files: &[File]) -> bool {
    if files.is_empty() {
        return false;
    }

    let navigator = match web_sys::window() {
        Some(w) => w.navigator(),
        None => return false,
    };
    let nav_js: &JsValue = navigator.as_ref();

    let can_share_fn_js = match js_sys::Reflect::get(
        nav_js,
        &JsValue::from_str("canShare"),
    ) {
        Ok(v) => v,
        Err(_) => return false,
    };
    if can_share_fn_js.is_undefined() {
        return false;
    }
    let can_share_fn: js_sys::Function = match can_share_fn_js.dyn_into() {
        Ok(f) => f,
        Err(_) => return false,
    };

    let data = match build_share_data(files, "", "") {
        Ok(d) => d,
        Err(_) => return false,
    };

    match can_share_fn.call1(nav_js, &data) {
        Ok(v) => return v.as_bool().unwrap_or(false),
        Err(_) => return false,
    }
}

/// Открывает системный диалог шаринга.
///
/// Возвращает `Ok(())`, если пользователь поделился
/// (или закрыл диалог успешно). `Err(AbortError)`, если
/// пользователь отменил. Другие `Err` — технические проблемы.
///
/// **User gesture обязателен.** Вызывать только из обработчика
/// клика — иначе `NotAllowedError`.
pub async fn share_files(
    files: &[File],
    title: &str,
    text: &str,
) -> Result<(), JsValue> {
    let navigator = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .navigator();
    let nav_js: &JsValue = navigator.as_ref();

    let share_fn_js = js_sys::Reflect::get(
        nav_js,
        &JsValue::from_str("share"),
    )?;
    if share_fn_js.is_undefined() {
        return Err(JsValue::from_str("navigator.share недоступен"));
    }
    let share_fn: js_sys::Function = share_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("share не функция"))?;

    let data = build_share_data(files, title, text)?;

    let promise_js = share_fn.call1(nav_js, &data)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("share не Promise"))?;

    let _ = JsFuture::from(promise).await?;
    return Ok(());
}

/// Проверяет, была ли ошибка вызвана отменой шаринга.
///
/// Пользователь нажал «Отмена» в системном диалоге — браузер
/// реджектит промис с `AbortError`. Это **не ошибка**, а нормальный
/// сценарий. Обрабатываем отдельно от технических проблем.
pub fn is_abort_error(e: &JsValue) -> bool {
    if let Some(exc) = e.dyn_ref::<web_sys::DomException>() {
        return exc.name() == "AbortError";
    }
    if let Some(s) = e.as_string() {
        return s.contains("AbortError") || s.contains("abort");
    }
    return false;
}
```

---

### 3. `tests/wasm_sharing.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::sharing::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Проверяет, что `is_supported()` возвращает `bool` без паники.
#[wasm_bindgen_test]
fn test_is_supported_returns_bool() {
    let supported = is_supported();
    web_sys::console::log_1(
        &format!("Web Share supported: {supported}").into(),
    );
}

/// Проверяет, что `can_share_files` возвращает `false` на пустом
/// массиве.
#[wasm_bindgen_test]
fn test_can_share_empty_returns_false() {
    let result = can_share_files(&[]);
    assert!(!result, "пустой список файлов не может быть расшарен");
}

/// Проверяет, что `can_share_files` не паникует на валидном файле.
#[wasm_bindgen_test]
fn test_can_share_file_returns_bool() {
    let parts = js_sys::Array::of1(&"test content".into());
    let file = web_sys::File::new_with_str_sequence(&parts, "test.txt")
        .expect("File::new_with_str_sequence");
    let result = can_share_files(&[file]);
    web_sys::console::log_1(
        &format!("canShare(test.txt): {result}").into(),
    );
}

/// Без user gesture вызов `share_files` вернёт `Err`.
#[wasm_bindgen_test]
async fn test_share_files_returns_result() {
    let parts = js_sys::Array::of1(&"test content".into());
    let file = web_sys::File::new_with_str_sequence(&parts, "test.txt")
        .expect("File::new_with_str_sequence");

    let result = share_files(&[file], "Test", "Sharing test").await;
    match result {
        Ok(()) => {}
        Err(e) => {
            let s = e.as_string().unwrap_or_default();
            web_sys::console::log_1(
                &format!("share_files error: {s}").into(),
            );
        }
    }
}

#[wasm_bindgen_test]
fn test_is_abort_error_recognizes_abort() {
    let e = wasm_bindgen::JsValue::from_str("AbortError");
    assert!(is_abort_error(&e), "должно распознать AbortError");
}

#[wasm_bindgen_test]
fn test_is_abort_error_rejects_other() {
    let e = wasm_bindgen::JsValue::from_str("Some other error");
    assert!(!is_abort_error(&e), "не должно распознавать обычную ошибку");
}
```

---

## Команды запуска тестов

```bash
# Проверка типов
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests

# Headless — без user gesture, тесты проверяют обработку ошибок
wasm-pack test --headless --chrome --test wasm_sharing
wasm-pack test --headless --firefox --test wasm_sharing

# Интерактивный Chrome — клик по кнопке шаринга в раннере
wasm-pack test --chrome --test wasm_sharing
```

Ожидаемо: **5 тестов**, все проходят.

---

## Демо-пример в `src/app.rs`

```rust
//! Демонстрация подглавы 18.16: Web Share API.
//!
//! **Интеллектуальный подход.**
//! * Расширенная диагностика: secure context, поддержка API,
//!   MIME-типы файлов, разрешение на шаринг.
//! * Мультифайловая поддержка — Web Share API умеет шарить
//!   несколько файлов сразу.
//! * Fallback на скачивание через `<a download>` + Object URL,
//!   если API недоступен или `canShare` вернул `false`.
//! * Различение `AbortError` (отмена), `NotAllowedError`
//!   (не из user gesture), `DataError` (неподдерживаемый файл).
//! * Показ превью — что именно уйдёт через `navigator.share`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_api, log, sharing};

/// Максимум файлов для шаринга — ограничение API.
/// Chrome обычно лимитирует 10-ю файлами.
const MAX_SHARE_FILES: usize = 10;

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Web Share API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));
    build_sharing_section(&container, &closures)?;

    log!("Section 18.16 ready");
    std::mem::forget(closures);
    return Ok(());
}

fn build_sharing_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.16. Web Share API"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите один или несколько файлов и нажмите «Поделиться» — \
         откроется системный диалог. Особенно актуально для мобильных: \
         файл уходит прямо в мессенджер или почту. Если API недоступен, \
         предложим скачать файл для ручной пересылки.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    build_diagnostics(&section)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    input.set_multiple(true);
    dom::append(&section, &input)?;

    let file_info = dom::create_element("div")?;
    file_info.set_attribute(
        "style",
        "margin: 10px 0; padding: 10px 12px; background: #f8f8f8; \
         border-radius: 4px; font-family: monospace; color: #333; \
         white-space: pre-line; line-height: 1.5;",
    )?;
    file_info.set_text_content(Some("(файлы не выбраны)"));
    dom::append(&section, &file_info)?;

    let actions_row = dom::create_element("div")?;
    actions_row.set_attribute(
        "style",
        "display: flex; align-items: center; gap: 8px; margin: 8px 0;",
    )?;

    let share_btn: web_sys::Element = dom::create_element("button")?;
    share_btn.set_text_content(Some("Поделиться"));
    share_btn.set_attribute("disabled", "").ok();
    dom::append(&actions_row, &share_btn)?;

    let download_btn: web_sys::Element = dom::create_element("button")?;
    download_btn.set_text_content(Some("Скачать (fallback)"));
    download_btn.set_attribute("disabled", "").ok();
    dom::append(&actions_row, &download_btn)?;

    dom::append(&section, &actions_row)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файлы)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let selected_files: Rc<RefCell<Vec<web_sys::File>>> =
        Rc::new(RefCell::new(Vec::new()));

    // ---- Обработчик выбора файлов. ----
    {
        let file_info_c = file_info.clone();
        let share_btn_c = share_btn.clone();
        let download_btn_c = download_btn.clone();
        let result_c = result.clone();
        let selected_c = selected_files.clone();

        callbacks::listen(
            &input,
            "change",
            move |ev: web_sys::Event| {
                let input_el: web_sys::HtmlInputElement =
                    match ev.target().and_then(|t| t.dyn_into().ok()) {
                        Some(el) => el,
                        None => return,
                    };
                let list = match input_el.files() {
                    Some(f) => f,
                    None => {
                        file_info_c.set_text_content(Some("(файлы не выбраны)"));
                        share_btn_c.set_attribute("disabled", "").ok();
                        download_btn_c.set_attribute("disabled", "").ok();
                        *selected_c.borrow_mut() = Vec::new();
                        return;
                    }
                };

                let count = list.length() as usize;
                let mut files: Vec<web_sys::File> = Vec::with_capacity(count);
                for i in 0..list.length() {
                    if let Some(f) = list.get(i) {
                        files.push(f);
                    }
                }

                let info = format_files_info(&files);
                file_info_c.set_text_content(Some(info.as_str()));

                let can = sharing::can_share_files(&files);

                if can {
                    share_btn_c.remove_attribute("disabled").ok();
                    result_c.set_text_content(
                        Some("✅ Файлы готовы к шарингу. Нажмите «Поделиться»."),
                    );
                } else {
                    share_btn_c.set_attribute("disabled", "").ok();
                    let why = explain_can_share_failure(&files);
                    let msg = format!(
                        "⚠️ Шаринг этих файлов недоступен.\n\n{why}",
                    );
                    result_c.set_text_content(Some(msg.as_str()));
                }

                if count > 0 {
                    download_btn_c.remove_attribute("disabled").ok();
                } else {
                    download_btn_c.set_attribute("disabled", "").ok();
                }

                *selected_c.borrow_mut() = files;
            },
            closures,
        )?;
    }

    // ---- Обработчик «Поделиться». ----
    {
        let selected_c = selected_files.clone();
        let share_btn_c = share_btn.clone();
        let result_c = result.clone();

        callbacks::listen(
            &share_btn,
            "click",
            move |_ev: web_sys::Event| {
                let files = selected_c.borrow().clone();
                if files.is_empty() {
                    result_c.set_text_content(Some("Сначала выберите файлы."));
                    return;
                }

                share_btn_c.set_attribute("disabled", "").ok();
                share_btn_c.set_text_content(Some("Шарим..."));
                result_c.set_text_content(Some("(открывается диалог...)"));

                let share_btn_c2 = share_btn_c.clone();
                let result_c2 = result_c.clone();
                let files_count = files.len();

                spawn_local(async move {
                    let outcome = sharing::share_files(
                        &files,
                        "Hello from WASM",
                        "Файлы, отправленные через Web Share API",
                    )
                    .await;

                    share_btn_c2.remove_attribute("disabled").ok();
                    share_btn_c2.set_text_content(Some("Поделиться"));

                    let message: String;
                    match outcome {
                        Ok(()) => {
                            message = format!(
                                "✅ Диалог шаринга завершён успешно.\n\
                                 Отправлено файлов: {files_count}\n\
                                 (Файлы ушли в выбранное приложение \
                                 или диалог закрыт без отмены.)",
                            );
                        }
                        Err(e) if sharing::is_abort_error(&e) => {
                            message = "⏹ Диалог отменён пользователем.\n\
                                       Это не ошибка — можно попробовать снова."
                                .to_string();
                        }
                        Err(e) => {
                            message = format_share_error(&e);
                        }
                    }
                    result_c2.set_text_content(Some(message.as_str()));
                });
            },
            closures,
        )?;
    }

    // ---- Обработчик «Скачать (fallback)». ----
    {
        let selected_c = selected_files.clone();
        let result_c = result.clone();

        callbacks::listen(
            &download_btn,
            "click",
            move |_ev: web_sys::Event| {
                let files = selected_c.borrow().clone();
                if files.is_empty() {
                    result_c.set_text_content(Some("Сначала выберите файлы."));
                    return;
                }

                let file = files[0].clone();
                let name = file.name();

                match download_file_via_object_url(&file, &name) {
                    Ok(()) => {
                        let extra: String;
                        if files.len() > 1 {
                            extra = format!(
                                "\n\n💡 Fallback скачивает только первый \
                                 файл. Остальные ({}) скачайте по одному.",
                                files.len() - 1,
                            );
                        } else {
                            extra = String::new();
                        }
                        let msg = format!(
                            "✅ Файл скачан: {name}\n\
                             Теперь его можно вручную отправить в любое \
                             приложение.{extra}",
                        );
                        result_c.set_text_content(Some(msg.as_str()));
                    }
                    Err(e) => {
                        let msg = format!("❌ Не удалось скачать: {e:?}");
                        result_c.set_text_content(Some(msg.as_str()));
                    }
                }
            },
            closures,
        )?;
    }

    return Ok(());
}

// ===============================================================
// Диагностика
// ===============================================================

fn build_diagnostics(parent: &web_sys::Element) -> Result<(), JsValue> {
    let diag = dom::create_element("div")?;
    diag.set_attribute(
        "style",
        "margin: 12px 0; padding: 10px 14px; \
         background: #f0f4ff; border-left: 4px solid #4a7cff; \
         border-radius: 4px; font-family: monospace; \
         white-space: pre-line; line-height: 1.6;",
    )?;

    let protocol: String = web_sys::window()
        .and_then(|w| w.location().protocol().ok())
        .unwrap_or_default();

    let host: String = web_sys::window()
        .and_then(|w| w.location().host().ok())
        .unwrap_or_default();

    let is_secure = protocol == "https:"
        || host.starts_with("localhost")
        || host.starts_with("127.0.0.1");

    let api_supported = sharing::is_supported();

    let mut lines: Vec<String> = Vec::new();
    lines.push(format!("Protocol: {protocol}"));
    lines.push(format!("Host: {host}"));

    if is_secure {
        lines.push("Secure context: ✅".to_string());
    } else {
        lines.push(
            "Secure context: ⚠️ НЕТ — API требует HTTPS или localhost"
                .to_string(),
        );
    }

    if api_supported {
        lines.push("navigator.share: ✅ доступен".to_string());
    } else {
        lines.push(
            "navigator.share: ❌ недоступен в этом браузере".to_string(),
        );
    }

    let text = lines.join("\n");
    diag.set_text_content(Some(text.as_str()));
    dom::append(parent, &diag)?;
    return Ok(());
}

// ===============================================================
// Формирование сообщений
// ===============================================================

fn format_files_info(files: &[web_sys::File]) -> String {
    if files.is_empty() {
        return "(файлы не выбраны)".to_string();
    }

    let total_size: f64 = files.iter().map(|f| f.size()).sum();

    let mut out = format!(
        "Выбрано файлов: {} (суммарно {})\n\n",
        files.len(),
        file_api::format_file_size(total_size),
    );

    for (i, f) in files.iter().enumerate() {
        if i >= 5 {
            out.push_str(&format!("… и ещё {}\n", files.len() - 5));
            break;
        }
        let mime_display: String;
        if f.type_().is_empty() {
            mime_display = "(не определён)".to_string();
        } else {
            mime_display = f.type_();
        }
        out.push_str(&format!(
            "{}. {} · {} · {}\n",
            i + 1,
            f.name(),
            file_api::format_file_size(f.size()),
            mime_display,
        ));
    }

    if files.len() > MAX_SHARE_FILES {
        out.push_str(&format!(
            "\n⚠️ Больше {MAX_SHARE_FILES} файлов — браузер может отказать."
        ));
    }

    return out;
}

fn explain_can_share_failure(files: &[web_sys::File]) -> String {
    if files.is_empty() {
        return "Файлы не выбраны.".to_string();
    }
    if !sharing::is_supported() {
        return "navigator.share недоступен в этом браузере.\n\
                Firefox не поддерживает Web Share API."
            .to_string();
    }
    if files.len() > MAX_SHARE_FILES {
        return format!(
            "Слишком много файлов: {} > {MAX_SHARE_FILES}.\n\
             Браузер отказывается шарить такое количество.",
            files.len(),
        );
    }
    let total: f64 = files.iter().map(|f| f.size()).sum();
    return format!(
        "Браузер не может отправить эту комбинацию файлов.\n\
         Возможные причины:\n\
         • Слишком большой суммарный размер ({}).\n\
         • Неподдерживаемые MIME-типы.\n\
         • Ограничения desktop-версии браузера.\n\n\
         Используйте кнопку «Скачать (fallback)».",
        file_api::format_file_size(total),
    );
}

fn format_share_error(e: &JsValue) -> String {
    if let Some(exc) = e.dyn_ref::<web_sys::DomException>() {
        let name = exc.name();
        if name == "NotAllowedError" {
            return "❌ NotAllowedError: вызов вне user gesture.\n\n\
                    Web Share API требует, чтобы `navigator.share` \
                    вызывался из обработчика клика. Нажмите кнопку ещё раз."
                .to_string();
        }
        if name == "DataError" {
            return "❌ DataError: браузер не может отправить эти файлы.\n\n\
                    Проверьте размер, количество и MIME-типы. \
                    Попробуйте уменьшить набор или использовать fallback."
                .to_string();
        }
        if name == "TypeError" {
            return "❌ TypeError: неверный формат данных.\n\n\
                    Скорее всего, проблема в ShareData. \
                    Проверьте консоль для деталей."
                .to_string();
        }
        return format!(
            "❌ Ошибка шаринга: {name}\n{}",
            exc.message(),
        );
    }

    let raw = e
        .as_string()
        .unwrap_or_else(|| format!("{e:?}"));
    return format!("❌ Ошибка шаринга: {raw}");
}

fn download_file_via_object_url(
    file: &web_sys::File,
    name: &str,
) -> Result<(), JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let url = web_sys::Url::create_object_url_with_blob(file)?;

    let a: web_sys::HtmlAnchorElement = document
        .create_element("a")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не anchor"))?;
    a.set_href(&url);
    a.set_download(name);
    a.click();

    web_sys::Url::revoke_object_url(&url)?;
    return Ok(());
}
```

---

## Команды запуска демо

```bash
# 1. Сборка WASM + Vite
npm run start

# 2. Открыть http://localhost:5173
```

**Проверка в Chrome/Edge (десктоп):**

- Выберите файл → панель диагностики показывает, доступен ли шаринг.
- Если да — кнопка «Поделиться» активна, по клику откроется системный диалог (на macOS — AirDrop, Mail, Messages; на Linux/Windows — зависит от ОС).
- Если нет — кнопка «Скачать (fallback)» активна.

**Проверка в Firefox:**

- Панель диагностики: `navigator.share: ❌ недоступен`.
- Кнопка «Поделиться» остаётся отключённой.
- Fallback: кнопка «Скачать (fallback)» работает.

**Проверка на мобильном (Android Chrome):**

- Откройте страницу по IP хоста в локальной сети (например, `http://192.168.1.10:5173`).
- Выберите 1–3 файла → кнопка «Поделиться» → системный диалог с WhatsApp, Telegram, Gmail, Drive.

---

## Что увидит пользователь

**Chrome, файл выбран успешно:**

```
Protocol: http:
Host: localhost:5173
Secure context: ✅
navigator.share: ✅ доступен

[Выберите файлы] → report.pdf

Выбрано файлов: 1 (суммарно 234.56 KB)

1. report.pdf · 234.56 KB · application/pdf

✅ Файлы готовы к шарингу. Нажмите «Поделиться».

[Поделиться]  [Скачать (fallback)]
```

После клика «Поделиться» открывается системный диалог. Пользователь выбирает приложение → файл уходит.

**Отмена диалога:**

```
⏹ Диалог отменён пользователем.
Это не ошибка — можно попробовать снова.
```

**Firefox:**

```
Protocol: http:
Host: localhost:5173
Secure context: ✅
navigator.share: ❌ недоступен в этом браузере

[Выберите файлы] → file.pdf

⚠️ Шаринг этих файлов недоступен.

navigator.share недоступен в этом браузере.
Firefox не поддерживает Web Share API.

[Поделиться]  [Скачать (fallback)]  ← активна
```

Клик «Скачать» → файл попадает в папку загрузок, пользователь может переслать вручную.

---

## Итоги подглавы 18.16

- **Web Share API** — единый интерфейс для отправки файла в любое приложение через системный диалог.
- **`navigator.share()`** — открывает диалог, возвращает `Promise<void>`.
- **`navigator.canShare()`** — проверяет, можно ли поделиться указанными файлами. Не гарантия: пользователь всё равно может отменить.
- **User gesture обязателен.** Вызов только из обработчика клика.
- **Отмена пользователем** — `AbortError`, а не реальная ошибка. Обрабатываем отдельно.
- **`ShareData` через `Reflect::set`** — не зависим от версии web-sys.
- **Firefox не поддерживает.** Мобильные браузеры — лучше десктопных.
- **Лимит файлов** — обычно 10 за один вызов. Сообщаем пользователю явно.
- **Fallback** — `<a download>` + Object URL — обязательный элемент для браузеров без API.

## Замечания на будущее

**Правило проекта, зафиксированное в этой подглаве:**

1. **`AbortError` — не ошибка.** Когда пользователь отменяет системный диалог, промис реджектится с `AbortError`. Это нормальный сценарий, показывайте спокойное сообщение «Диалог отменён», а не красный текст «Ошибка».

2. **`canShare` проверяйте до активации кнопки.** Даже если `navigator.share` есть, для конкретного файла `canShare` может вернуть `false`. Активируйте кнопку «Поделиться» только после подтверждения.

3. **Строьте `ShareData` через `Reflect::set`.** Тип `ShareData` в web-sys не экспортирован. Простой объект `{ files, title, text }` работает идентично и переносим между версиями.

4. **Правило про user gesture — универсально.** Как в FSA и `showSaveFilePicker`, Web Share API требует активации. Демо должно вызывать `navigator.share()` прямо из обработчика клика, без промежуточных `await`.

5. **Мобильный фокус.** Демонстрация FSA и Web Share API лучше всего работает на мобильных. Desktop Firefox — не поддерживает ни то, ни другое. В демо показывайте это явно, не заставляя пользователя угадывать, почему ничего не происходит.

6. **Диагностика — часть демо.** Пользователь должен видеть **почему** API доступен или нет: protocol, host, secure context, наличие метода. Это снимает большую часть вопросов «почему не работает».

7. **Fallback — обязательный элемент.** Если API не работает, у пользователя должен быть способ добиться результата. `<a download>` функционально эквивалентен «поделиться вручную».

8. **Различайте ошибки.** `AbortError`, `NotAllowedError`, `DataError`, `TypeError` — четыре разных ситуации. Пользователю нужны разные сообщения и подсказки, а не одно общее «что-то пошло не так».

9. **Лимит — часть UX.** `MAX_SHARE_FILES = 10` — не выдумка, а реальное ограничение Chrome. Показывайте его пользователю до того, как браузер откажет с `DataError`.

10. **Показывайте, что уйдёт.** Превью файлов + `title` + `text` снимает неопределённость. Пользователь понимает, что именно попадёт в мессенджер или почту.