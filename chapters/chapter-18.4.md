# Глава 18. File API

## 18.4. Object URL: превью и скачивание

В подглаве 18.3 мы получили байты в Rust. Теперь разберёмся, как показать их пользователю: превратить `Blob` в адрес, который можно подставить в `<img>`, `<video>` или `<a download>`. Речь о **Object URL** — временных ссылках вида `blob:http://localhost:3000/uuid`. Браузер создаёт их по запросу и держит в памяти, пока мы не освободим их явно. Это ключевой механизм для превью изображений, аудио, видео и для скачивания результатов конвертации.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod object_url;
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.4
│   ├── object_url.rs              # НОВЫЙ: модуль подглавы 18.4
│   ├── file_utils.rs              # из 18.3, не меняется
│   ├── file_api.rs                # из 18.1, не меняется
│   ├── file_reader.rs             # из 18.2, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует (экспортирует log!)
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_object_url.rs         # НОВЫЙ: тесты подглавы 18.4
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/object_url.rs` — модуль, вводимый этой подглавой. Автономен: зависит только от `web-sys` и `wasm-bindgen`.
- `src/lib.rs` — **обязательно** дополняется строкой `pub mod object_url;`. Без неё `crate::object_url` не существует для компилятора, и `app.rs` не соберётся (ошибка `unresolved import 'crate::object_url'`). Это правило действует для **каждого** нового модуля любой подглавы.
- `src/app.rs` — обновлён: в `run()` добавлена демо-секция «18.4. Object URL: превью и скачивание».
- `tests/wasm_object_url.rs` — новый файл с тестами.

---

## Ссылки для детального изучения

- **MDN: URL.createObjectURL()** — https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL
- **MDN: URL.revokeObjectURL()** — https://developer.mozilla.org/en-US/docs/Web/API/URL/revokeObjectURL
- **MDN: Memory management (Blob URLs)** — https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL#memory_management
- **MDN: `<a download>`** — https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#download
- **Спецификация W3C: File API — Object URLs** — https://w3c.github.io/FileAPI/#dfn-createObjectURL
- **web-sys: Url** — https://docs.rs/web-sys/latest/web_sys/struct.Url.html
- **web-sys: HtmlAnchorElement** — https://docs.rs/web-sys/latest/web_sys/struct.HtmlAnchorElement.html
- **web-sys: HtmlImageElement** — https://docs.rs/web-sys/latest/web_sys/struct.HtmlImageElement.html
- **web-sys: HtmlVideoElement** — https://docs.rs/web-sys/latest/web_sys/struct.HtmlVideoElement.html

---

## Содержание новых / обновлённых файлов

### `src/lib.rs` — подключение модуля

Добавьте объявление модуля в секцию главы 18:

```rust
// src/lib.rs

// … существующие модули проекта …
pub mod dom;
pub mod callbacks;
pub mod logging;
// … и другие …

// ---- Глава 18. File API ----
pub mod file_api;       // 18.1
pub mod file_reader;    // 18.2
pub mod file_utils;     // 18.3
pub mod object_url;     // 18.4 ← НОВОЕ
```

**Ключевое:**

- Без `pub mod object_url;` компилятор не видит файл `src/object_url.rs`, даже если он физически существует. Rust не сканирует файловую систему автоматически.
- `pub` обязателен, потому что модуль используется и из `app.rs` (через `crate::object_url`), и из тестов (через `hello_wasm02::object_url`).

---

### `src/object_url.rs`

Модуль автономен: зависимости только от `web-sys` и `wasm-bindgen`. Он предоставляет безопасные обёртки над `URL.createObjectURL` / `URL.revokeObjectURL` и утилиты для скачивания `Blob` через `<a download>`.

```rust
//! Object URL — временные ссылки на Blob для превью и скачивания.
//! Модуль автономен: не зависит от других модулей проекта.

use wasm_bindgen::prelude::*;
use web_sys::{Blob, HtmlAnchorElement, Url};

/// Создаёт Object URL для `Blob`.
///
/// Возвращает строку вида `blob:http://localhost:3000/uuid`.
/// URL остаётся валидным, пока не будет освобождён вызовом
/// [`revoke_object_url`].
pub fn create_object_url(blob: &Blob) -> Result<String, JsValue> {
    return Url::create_object_url_with_blob(blob);
}

/// Освобождает Object URL, позволяя браузеру выгрузить связанный Blob.
///
/// После вызова URL становится недействительным: `<img>`, `<video>`
/// и другие потребители не смогут загрузить данные.
pub fn revoke_object_url(url: &str) -> Result<(), JsValue> {
    return Url::revoke_object_url(url);
}

/// Подставляет Object URL в `src` элемента `<img>`.
///
/// Полезно для превью изображений.
pub fn set_image_src(img: &web_sys::HtmlImageElement, url: &str) {
    img.set_src(url);
}

/// Подставляет Object URL в `src` элемента `<video>`.
pub fn set_video_src(video: &web_sys::HtmlVideoElement, url: &str) {
    video.set_src(url);
}

/// Инициирует скачивание `Blob` через `<a download>`.
///
/// Создаёт временный `<a>`, подставляет Object URL и имя файла,
/// программно кликает, а затем освобождает URL.
///
/// Возвращает созданный Object URL — на случай, если вызывающая
/// сторона хочет управлять его жизненным циклом самостоятельно.
pub fn download_blob(blob: &Blob, file_name: &str) -> Result<String, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let url = Url::create_object_url_with_blob(blob)?;

    let anchor: HtmlAnchorElement = document
        .create_element("a")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не anchor"))?;

    anchor.set_href(&url);
    anchor.set_download(file_name);
    anchor.click();

    // Браузер уже начал скачивание — освобождаем URL.
    Url::revoke_object_url(&url)?;

    return Ok(url);
}

/// Инициирует скачивание `Blob` с автоматическим определением
/// имени файла из MIME-типа (fallback: `download.bin`).
pub fn download_blob_with_mime(blob: &Blob, mime_type: &str) -> Result<String, JsValue> {
    let file_name = file_name_from_mime(mime_type);
    return download_blob(blob, &file_name);
}

/// Определяет имя файла по MIME-типу (базовый набор).
fn file_name_from_mime(mime_type: &str) -> String {
    match mime_type {
        "image/png" => "download.png".to_string(),
        "image/jpeg" => "download.jpg".to_string(),
        "image/gif" => "download.gif".to_string(),
        "image/webp" => "download.webp".to_string(),
        "text/plain" => "download.txt".to_string(),
        "application/json" => "download.json".to_string(),
        "application/pdf" => "download.pdf".to_string(),
        _ => "download.bin".to_string(),
    }
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — для `JsValue` и `JsCast` (через prelude, для `.dyn_into()`).
- `web_sys::{Blob, HtmlAnchorElement, Url}` — все три используются.
- `web_sys::HtmlImageElement` и `web_sys::HtmlVideoElement` — используются в полных путях, без импорта.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- `Url::create_object_url_with_blob` возвращает `Result<String, JsValue>` — мы пробрасываем `?` или возвращаем результат напрямую.
- `Url::revoke_object_url` принимает `&str`. После вызова URL недействителен — `<img>` больше не загрузит данные.
- `download_blob` создаёт `<a>` программно, устанавливает `href` и `download`, кликает и **сразу освобождает URL**. Это безопасно: браузер уже начал скачивание к моменту `click()`. Согласно спецификации, каждый Object URL должен быть освобождён; иначе `Blob` остаётся в памяти до выгрузки страницы.
- В Safari есть известная проблема с полным освобождением памяти даже после `revokeObjectURL`, но это не повод не вызывать `revoke` — в Chrome и Firefox освобождение работает корректно.
- `file_name_from_mime` — маленькая локальная утилита для случая, когда имя файла неизвестно.
- Все tail-функции оформлены с явным `return` — конвенция проекта.

---

### `src/app.rs` — новая секция 18.4

Ниже — новый блок для `app.rs`. Функция `build_object_url_section` самодостаточна: её можно вставить в `run()` рядом с другими секциями без правок последних.

```rust
//! Демонстрация подглавы 18.4: Object URL.
//!
//! Секция: превью изображения через Object URL, скачивание
//! любого файла через `<a download>`, освобождение URL.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::callbacks::{self, EventClosures};
use crate::object_url;
use crate::{dom, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.4 ----
    build_object_url_section(&container, &closures)?;

    log!("Section 18.4 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.4: Object URL
// ---------------------------------------------------------------

fn build_object_url_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.4. Object URL: превью и скачивание"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите изображение — оно отобразится через Object URL. \
         Любой файл можно скачать через <a download>. Кнопка \
         «Освободить URL» делает ссылку недействительной.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    // ---- Input для выбора файла ----
    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    // ---- Кнопки ----
    let btn_download = dom::create_element("button")?;
    btn_download.set_text_content(Some("Скачать"));
    dom::append(&section, &btn_download)?;

    let btn_revoke = dom::create_element("button")?;
    btn_revoke.set_text_content(Some("Освободить URL"));
    dom::append(&section, &btn_revoke)?;

    // ---- Превью изображения ----
    let img: web_sys::HtmlImageElement = dom::create_element("img")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не img"))?;
    img.set_attribute(
        "style",
        "max-width: 300px; max-height: 300px; display: block; margin: 10px 0;",
    )?;
    dom::append(&section, &img)?;

    // ---- Область результата ----
    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файл)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    // Общее состояние: текущий Object URL и File.
    // Rc<RefCell<...>> — чтобы разделять между обработчиками.
    let current_url: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
    let current_file: Rc<RefCell<Option<web_sys::File>>> = Rc::new(RefCell::new(None));

    // ---- Обработчик выбора файла ----
    {
        let img_c = img.clone();
        let result_c = result.clone();
        let current_url_c = current_url.clone();
        let current_file_c = current_file.clone();

        callbacks::listen(
            &input,
            "change",
            move |ev: web_sys::Event| {
                let input_el: web_sys::HtmlInputElement =
                    match ev.target().and_then(|t| t.dyn_into().ok()) {
                        Some(el) => el,
                        None => {
                            result_c.set_text_content(Some("target не input"));
                            return;
                        }
                    };
                let file = match input_el.files().and_then(|f| f.get(0)) {
                    Some(f) => f,
                    None => {
                        result_c.set_text_content(Some("файл не выбран"));
                        return;
                    }
                };

                // Освобождаем предыдущий URL, если он есть.
                if let Some(old) = current_url_c.borrow().as_ref() {
                    object_url::revoke_object_url(old).ok();
                }

                let url = match object_url::create_object_url(&file) {
                    Ok(u) => u,
                    Err(e) => {
                        let message = format!("❌ create_object_url: {e:?}");
                        result_c.set_text_content(Some(&message));
                        return;
                    }
                };

                // Если это изображение — ставим в <img>.
                if file.type_().starts_with("image/") {
                    object_url::set_image_src(&img_c, &url);
                } else {
                    img_c.remove_attribute("src").ok();
                }

                let mime_display = if file.type_().is_empty() {
                    "(не определён)".to_string()
                } else {
                    file.type_()
                };
                let preview_note = if file.type_().starts_with("image/") {
                    "Изображение показано выше через <img src>."
                } else {
                    "Файл не является изображением — превью не показано."
                };

                let message = format!(
                    "✅ Object URL создан\n\
                     Файл: {}\n\
                     MIME: {mime_display}\n\
                     Размер: {} байт\n\
                     URL: {url}\n\n\
                     {preview_note}\n\n\
                     Нажмите «Освободить URL», чтобы сделать ссылку \
                     недействительной.",
                    file.name(),
                    file.size() as u64,
                );
                result_c.set_text_content(Some(&message));

                *current_url_c.borrow_mut() = Some(url);
                *current_file_c.borrow_mut() = Some(file);
            },
            closures,
        )?;
    }

    // ---- Обработчик скачивания ----
    {
        let result_c = result.clone();
        let current_file_c = current_file.clone();

        callbacks::listen(
            &btn_download,
            "click",
            move |_ev: web_sys::Event| {
                let file = match current_file_c.borrow().as_ref() {
                    Some(f) => f.clone(),
                    None => {
                        result_c.set_text_content(Some("Сначала выберите файл."));
                        return;
                    }
                };
                let name = file.name();
                match object_url::download_blob(&file, &name) {
                    Ok(url) => {
                        let message = format!(
                            "✅ Скачивание запущено\n\
                             Файл: {name}\n\
                             URL: {url}\n\n\
                             URL освобождён сразу после клика — браузер \
                             уже начал скачивание."
                        );
                        result_c.set_text_content(Some(&message));
                    }
                    Err(e) => {
                        let message = format!("❌ download_blob: {e:?}");
                        result_c.set_text_content(Some(&message));
                    }
                }
            },
            closures,
        )?;
    }

    // ---- Обработчик освобождения URL ----
    {
        let img_c = img.clone();
        let result_c = result.clone();
        let current_url_c = current_url.clone();

        callbacks::listen(
            &btn_revoke,
            "click",
            move |_ev: web_sys::Event| {
                let url = match current_url_c.borrow_mut().take() {
                    Some(u) => u,
                    None => {
                        result_c.set_text_content(Some("Нет активного URL."));
                        return;
                    }
                };
                match object_url::revoke_object_url(&url) {
                    Ok(()) => {
                        img_c.remove_attribute("src").ok();
                        let message = format!(
                            "✅ URL освобождён: {url}\n\n\
                             Теперь <img> не сможет загрузить данные — \
                             ссылка недействительна. Если попробовать \
                             снова скачать, Object URL будет создан заново."
                        );
                        result_c.set_text_content(Some(&message));
                    }
                    Err(e) => {
                        let message = format!("❌ revoke_object_url: {e:?}");
                        result_c.set_text_content(Some(&message));
                    }
                }
            },
            closures,
        )?;
    }

    return Ok(());
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_object_url_section` (для разделяемого состояния между обработчиками).
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` при получении `HtmlInputElement` и `HtmlImageElement`.
- `callbacks`, `EventClosures`, `dom`, `object_url`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- Состояние (текущий URL и файл) хранится в `Rc<RefCell<Option<...>>>` — это позволяет разделить его между тремя обработчиками (выбор, скачивание, освобождение).
- При выборе нового файла старый Object URL освобождается — иначе накопилась бы утечка.
- Для не-изображений атрибут `src` у `<img>` удаляется, чтобы не показывать «сломанную» картинку.
- `download_blob` создаёт `<a>`, кликает и сразу освобождает URL — это соответствует рекомендации MDN по управлению памятью.
- При освобождении URL удаляется `src` у `<img>` — визуально это демонстрирует, что ссылка больше не работает.

---

### `tests/wasm_object_url.rs`

Тесты подглавы 18.4. Проверяют создание URL, его формат, освобождение и то, что повторный вызов даёт новый URL.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::object_url::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_create_object_url_prefix() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"hello".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let url = create_object_url(&blob).expect("create_object_url");
    assert!(
        url.starts_with("blob:"),
        "URL должен начинаться с 'blob:', получено: {url}"
    );

    // Освобождаем, чтобы не текла память в тестовом окружении.
    revoke_object_url(&url).ok();
}

#[wasm_bindgen_test]
fn test_create_and_revoke() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"data".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let url = create_object_url(&blob).expect("create_object_url");
    assert!(!url.is_empty());

    // Освобождение не должно возвращать ошибку.
    revoke_object_url(&url).expect("revoke_object_url");
}

#[wasm_bindgen_test]
fn test_two_urls_differ() {
    // Один и тот же Blob, два вызова create_object_url —
    // должны получиться разные URL (каждый вызов создаёт новый).
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"same".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let url1 = create_object_url(&blob).expect("create #1");
    let url2 = create_object_url(&blob).expect("create #2");

    assert_ne!(
        url1, url2,
        "Два вызова create_object_url должны давать разные URL"
    );

    revoke_object_url(&url1).ok();
    revoke_object_url(&url2).ok();
}
```

**Проверка импортов:**

- `wasm_bindgen_test::*` — `wasm_bindgen_test` и `wasm_bindgen_test_configure`.
- `hello_wasm02::object_url::*` — `create_object_url`, `revoke_object_url`.

Все импорты используются, неиспользуемых нет.

**Что проверяем:**

- **Префикс `blob:`** — Object URL всегда начинается с `blob:`.
- **Освобождение не паникует** — `revoke_object_url` на валидном URL возвращает `Ok`.
- **Два вызова дают разные URL** — согласно спецификации, каждый вызов `createObjectURL` создаёт новый URL, даже для одного и того же `Blob`.

---

## Команды запуска

### Сборка и запуск демо

```bash
# Проверка типов без полной сборки
cargo check --target wasm32-unknown-unknown

# Dev-сборка WASM
npm run wasm:dev

# Запуск Vite (в отдельном терминале)
npm run dev

# Или одной командой — сборка WASM + запуск Vite
npm run start
```

### Тесты подглавы 18.4

```bash
# Проверка типов тестов
cargo check --target wasm32-unknown-unknown --tests

# Только тесты этой подглавы
wasm-pack test --headless --chrome --test wasm_object_url

# То же самое для Firefox
wasm-pack test --headless --firefox --test wasm_object_url

# Все тесты проекта
npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.4. Object URL: превью и скачивание** с полем выбора файла, кнопками «Скачать» и «Освободить URL», а также областью для превью изображения.

**Сценарий 1 — изображение:**
1. Выберите PNG или JPEG.
2. Изображение отобразится через `<img src="blob:...">`.
3. В блоке результата — имя, MIME, размер и сам URL.

**Сценарий 2 — скачивание:**
1. Выберите любой файл.
2. Нажмите «Скачать» — браузер предложит сохранить файл под исходным именем.
3. В блоке результата — URL, который был создан и сразу освобождён.

**Сценарий 3 — освобождение:**
1. Выберите изображение.
2. Нажмите «Освободить URL» — изображение исчезнет, `src` удалится.
3. В блоке результата — сообщение, что ссылка больше недействительна.

---

## Итоги подглавы 18.4

- **`Url::create_object_url_with_blob`** — создаёт временный адрес `blob:...` для `Blob`.
- **`Url::revoke_object_url`** — освобождает адрес, позволяя браузеру выгрузить связанный `Blob`. Без этого вызова накапливается утечка памяти.
- **`<img src="blob:...">`** — превью изображения без загрузки на сервер и без base64-оверхеда.
- **`<video src="blob:...">`** — аналогично для видео (в модуле есть `set_video_src`).
- **`<a download>` + Object URL + `click()`** — программное скачивание `Blob` с заданным именем файла.
- **Жизненный цикл URL** — при выборе нового файла старый URL должен быть освобождён; при скачивании URL освобождается сразу после клика.

Модуль `object_url.rs` автономен, все tail-выражения оформлены через `return`, неиспользуемых импортов нет. Демо-секция самодостаточна и демонстрирует три ключевых сценария: превью, скачивание и освобождение.

## Замечание на будущее

Правило, зафиксированное в этой подглаве: **каждый новый модуль обязан быть объявлен в `src/lib.rs` строкой `pub mod <имя>;`**. Без этого объявления файл `.rs` существует на диске, но компилятор его не видит, и появляется ошибка `unresolved import 'crate::<имя>'`. Добавляйте строку в `lib.rs` сразу после создания файла модуля — тогда `cargo check` подскажет оставшиеся проблемы, не отвлекаясь на «no such module».

В следующей подглаве (18.5) мы сравним Object URL с Data URL — вторым способом представить бинарные данные в виде адреса.