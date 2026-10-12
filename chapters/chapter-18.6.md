# Глава 18. File API

## 18.6. Drag & Drop: файлы

В подглавах 18.1–18.5 мы получали файлы через `<input type="file">`. Но современный веб знает и второй, более естественный способ: **перетаскивание**. Пользователь «хватает» файл в проводнике и бросает его прямо на страницу. Для конвертера это особенно ценно — drag&drop ускоряет работу в разы. В этой подглаве разберём `DragEvent` и `DataTransfer`, поймём, почему без `prevent_default` ничего не работает, и соберём drop-зону с визуальной обратной связью. А в конце разберём реальный баг, с которым сталкивается каждый, кто работает с файлами: **браузер не всегда умеет определять MIME-тип**.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # pub mod drag_drop; (уже подключён)
│   ├── app.rs                     # ОБНОВЛЁН: секция 18.6 + хелперы
│   ├── drag_drop.rs               # модуль подглавы 18.6
│   ├── file_utils.rs              # ОБНОВЛЁН: расширенная таблица MIME + effective_mime_type
│   ├── data_url.rs                # из 18.5, не меняется
│   ├── base64_utils.rs            # из 18.5, не меняется
│   ├── object_url.rs              # из 18.4, не меняется
│   ├── file_api.rs                # из 18.1, не меняется
│   ├── file_reader.rs             # из 18.2, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует (экспортирует log!)
│   └── … остальные модули проекта без изменений
├── tests/
│   ├── wasm_drag_drop.rs          # ОБНОВЛЁН: расширенный набор тестов
│   └── wasm_file_utils.rs         # ОБНОВЛЁН: тесты effective_mime_type и новых расширений
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/drag_drop.rs` — модуль, вводимый этой подглавой. Автономен: зависит только от `web-sys` и `crate::file_api::FileMetadata`.
- `src/file_utils.rs` — **обновляется**: к имеющейся `mime_from_extension` добавляется расширенная таблица (Markdown, HTML, CSS, JS, YAML, TOML и др.) и функция `effective_mime_type`, которая сначала спрашивает браузер, а при пустом ответе — выводит MIME по расширению.
- `src/app.rs` — обновлён: добавлена секция 18.6 с хелперами `format_dropped_files` и `format_mime_line`.
- `tests/wasm_drag_drop.rs` и `tests/wasm_file_utils.rs` — расширены новыми тестами.

Новых фич в `Cargo.toml` не требуется: `DataTransfer`, `DragEvent`, `DragEventInit` были включены ещё при подготовке подглавы 18.1.

---

## Ссылки для детального изучения

- **MDN: HTML Drag and Drop API** — https://developer.mozilla.org/en-US/docs/Web/API/HTML_Drag_and_Drop_API
- **MDN: DragEvent** — https://developer.mozilla.org/en-US/docs/Web/API/DragEvent
- **MDN: DataTransfer** — https://developer.mozilla.org/en-US/docs/Web/API/DataTransfer
- **MDN: DataTransfer.files** — https://developer.mozilla.org/en-US/docs/Web/API/DataTransfer/files
- **MDN: MIME types — Complete list** — https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Common_types
- **HTML Living Standard: Drag and drop** — https://html.spec.whatwg.org/multipage/dnd.html
- **web-sys: DragEvent** — https://docs.rs/web-sys/latest/web_sys/struct.DragEvent.html
- **web-sys: DataTransfer** — https://docs.rs/web-sys/latest/web_sys/struct.DataTransfer.html

---

## Содержание новых / обновлённых файлов

### `src/drag_drop.rs`

Модуль автономен: зависимости только от `web-sys` и `crate::file_api::FileMetadata`. Прелюд не нужен — мы не работаем с `JsValue`.

```rust
//! Drag & Drop — получение файлов через DataTransfer.
//! Модуль автономен: не зависит от других модулей проекта,
//! кроме `file_api` (для типа `FileMetadata`).

use crate::file_api::FileMetadata;
use web_sys::{DataTransfer, DragEvent, FileList};

/// Извлекает список файлов из события `drop`.
///
/// `DataTransfer::files()` возвращает `Option<FileList>` — `None`,
/// если событие не содержит `DataTransfer` (синтетическое событие
/// или drop без данных).
pub fn files_from_drop_event(event: &DragEvent) -> Option<FileList> {
    let dt: DataTransfer = event.data_transfer()?;
    return dt.files();
}

/// Извлекает метаданные всех файлов из события `drop`.
///
/// Возвращает пустой вектор, если файлов нет — удобно для
/// отображения в UI без дополнительных проверок.
pub fn metadata_from_drop_event(event: &DragEvent) -> Vec<FileMetadata> {
    match files_from_drop_event(event) {
        Some(list) => return crate::file_api::file_list_metadata(&list),
        None => return Vec::new(),
    }
}

/// Проверяет, содержит ли событие хотя бы один файл.
///
/// Полезно для валидации на входе: некоторые перетаскивания
/// содержат только текст или HTML — их нужно игнорировать.
pub fn has_files(event: &DragEvent) -> bool {
    match files_from_drop_event(event) {
        Some(list) => return list.length() > 0,
        None => return false,
    }
}

/// Предотвращает стандартное поведение браузера при drag&drop.
///
/// **Обязательно** вызывать на событиях `dragover` и `drop`.
/// Без `prevent_default` на `dragover` браузер не разрешит drop;
/// без `prevent_default` на `drop` браузер попытается открыть
/// перетащенный файл вместо нашей обработки.
///
/// `stop_propagation` предотвращает всплытие события к родительским
/// элементам — на случай, если у них тоже есть обработчики.
pub fn prevent_default(event: &DragEvent) {
    event.prevent_default();
    event.stop_propagation();
}
```

**Проверка импортов:**

- `crate::file_api::FileMetadata` — используется в сигнатуре `metadata_from_drop_event`.
- `web_sys::{DataTransfer, DragEvent, FileList}` — все три используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- `DataTransfer::files()` возвращает `Option<FileList>` в актуальной версии `web-sys` — возвращаем его напрямую, без обёртки `Some(...)`.
- `has_files` — новая утилита: быстро проверяет, есть ли в событии файлы. Для drag&drop из другого окна браузера `DataTransfer` может содержать только текст или URL — такие события нужно отсеивать.
- `prevent_default` вызывает **обе** функции: `prevent_default()` и `stop_propagation()`.
- Все функции оформлены с явным `return` — конвенция проекта.

---

### `src/file_utils.rs` — обновление

Это ключевое изменение подглавы. В реальной работе над демо-секцией 18.6 выяснилось: при перетаскивании `chapter-18.5.md` браузер вернул пустую строку в `file.type_()`. Причина — операционная система не регистрирует расширение `.md` как известный MIME. Браузер честно возвращает `""`, и в UI появляется `(не определён)`.

**Решение:** добавить fallback по расширению. Функция `effective_mime_type` сначала спрашивает браузер, а при пустом ответе — обращается к расширенной таблице `mime_from_extension`.

```rust
//! Утилиты для работы с ArrayBuffer, Uint8Array и MIME-типами.
//! Модуль автономен: не зависит от других модулей проекта.

use js_sys::{ArrayBuffer, Uint8Array};
use wasm_bindgen::prelude::*;
use web_sys::{Blob, File};

/// Преобразует `ArrayBuffer` в `Vec<u8>`.
pub fn array_buffer_to_vec(buffer: &ArrayBuffer) -> Vec<u8> {
    let uint8 = Uint8Array::new(buffer);
    uint8.to_vec()
}

/// Преобразует срез `u8` в `Blob` с указанным MIME-типом.
pub fn vec_to_blob(data: &[u8], mime_type: &str) -> Result<Blob, JsValue> {
    let uint8 = Uint8Array::from(data);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type(mime_type);
    Blob::new_with_u8_array_sequence_and_options(&parts, &options)
}

/// Извлекает расширение файла из имени (без точки).
///
/// Возвращает `None`, если имя не содержит точки.
/// Имя с завершающей точкой (`"file."`) даёт `Some("")`.
pub fn get_extension(file_name: &str) -> Option<&str> {
    return file_name.rsplit_once('.').map(|(_, ext)| ext);
}

/// Определяет MIME-тип по расширению.
///
/// Регистр расширения не важен. Для неизвестных расширений
/// возвращает `application/octet-stream`.
///
/// Покрывает распространённые форматы: изображения, текст,
/// код, разметку, документы, архивы, аудио, видео.
pub fn mime_from_extension(ext: &str) -> &'static str {
    match ext.to_lowercase().as_str() {
        // Изображения
        "png" => "image/png",
        "jpg" | "jpeg" => "image/jpeg",
        "gif" => "image/gif",
        "webp" => "image/webp",
        "svg" => "image/svg+xml",
        "bmp" => "image/bmp",
        "ico" => "image/x-icon",
        "tiff" | "tif" => "image/tiff",
        // Текст и разметка
        "txt" => "text/plain",
        "md" | "markdown" => "text/markdown",
        "html" | "htm" => "text/html",
        "css" => "text/css",
        "csv" => "text/csv",
        "xml" => "application/xml",
        // Код и данные
        "js" | "mjs" => "text/javascript",
        "json" => "application/json",
        "yaml" | "yml" => "application/yaml",
        "toml" => "application/toml",
        "wasm" => "application/wasm",
        // Документы
        "pdf" => "application/pdf",
        "epub" => "application/epub+zip",
        "rtf" => "application/rtf",
        // Архивы
        "zip" => "application/zip",
        "gz" => "application/gzip",
        "tar" => "application/x-tar",
        // Аудио
        "mp3" => "audio/mpeg",
        "wav" => "audio/wav",
        "ogg" => "audio/ogg",
        "flac" => "audio/flac",
        // Видео
        "mp4" => "video/mp4",
        "webm" => "video/webm",
        "ogv" => "video/ogg",
        // Fallback
        _ => "application/octet-stream",
    }
}

/// Возвращает «эффективный» MIME-тип файла.
///
/// Сначала спрашивает браузер (`file.type_()`). Если браузер вернул
/// непустую строку — используем её. Иначе определяем MIME по
/// расширению через [`mime_from_extension`].
///
/// Это нужно, потому что не все ОС регистрируют MIME для
/// «нетипичных» расширений: `.md`, `.toml`, `.yaml`, `.wasm`
/// часто дают пустой `file.type_()`.
pub fn effective_mime_type(file: &File) -> String {
    let declared = file.type_();
    if !declared.is_empty() {
        return declared;
    }
    let name = file.name();
    let ext = get_extension(&name).unwrap_or("");
    return mime_from_extension(ext).to_string();
}
```

**Проверка импортов:**

- `js_sys::{ArrayBuffer, Uint8Array}` — используются в первой функции.
- `wasm_bindgen::prelude::*` — для `JsValue` в сигнатурах.
- `web_sys::{Blob, File}` — `Blob` в `vec_to_blob`, `File` в `effective_mime_type`.
- Неиспользуемых импортов нет.

**Ключевое:**

- `mime_from_extension` теперь покрывает в разы больше форматов. Существующие тесты подглавы 18.3 продолжают проходить без правок — старые пары расширение/MIME сохранены.
- `effective_mime_type` **не меняет** существующие функции — только дополняет их.
- Возвращаемый тип `String`, а не `&'static str` — потому что `file.type_()` возвращает `String`.

---

### `src/app.rs` — секция 18.6

Функция `build_drag_drop_section` самодостаточна. Отдельно — две хелпер-функции: `format_dropped_files` (принимает `&FileList`, обходит файлы) и `format_mime_line` (формирует строку MIME с fallback-пометкой).

```rust
//! Демонстрация подглавы 18.6: Drag & Drop файлов.
//!
//! Секция: drop-зона с визуальной обратной связью при dragover
//! и отображением метаданных перетащенных файлов при drop.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::callbacks::{self, EventClosures};
use crate::{dom, drag_drop, file_api, file_utils, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.6 ----
    build_drag_drop_section(&container, &closures)?;

    log!("Section 18.6 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.6: Drag & Drop
// ---------------------------------------------------------------

fn build_drag_drop_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.6. Drag & Drop — перетащите файлы"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Перетащите файлы из проводника в область ниже. \
         Обработчики dragover и drop вызывают prevent_default, \
         чтобы браузер не открывал файл вместо нашей обработки. \
         Область подсвечивается зелёным, пока над ней находится \
         курсор с файлами.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    const STYLE_IDLE: &str = "border: 2px dashed #888; padding: 30px; \
        text-align: center; margin: 10px 0; color: #555; \
        cursor: pointer; border-radius: 6px;";
    const STYLE_HOVER: &str = "border: 2px dashed #2b8; padding: 30px; \
        text-align: center; margin: 10px 0; color: #2b8; \
        cursor: pointer; border-radius: 6px; background: #f0fff5;";

    let drop_zone = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&drop_zone), "drop-zone")?;
    drop_zone.set_attribute("style", STYLE_IDLE)?;
    drop_zone.set_text_content(Some("Перетащите файлы сюда"));
    dom::append(&section, &drop_zone)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(перетащите файлы в область выше)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    // ---- dragover: prevent_default + подсветка ----
    {
        let drop_zone_c = drop_zone.clone();
        callbacks::listen(
            &drop_zone,
            "dragover",
            move |ev: web_sys::Event| {
                if let Ok(drag_ev) = ev.dyn_into::<web_sys::DragEvent>() {
                    drag_drop::prevent_default(&drag_ev);
                }
                drop_zone_c.set_attribute("style", STYLE_HOVER).ok();
            },
            closures,
        )?;
    }

    // ---- dragleave: снять подсветку ----
    {
        let drop_zone_c = drop_zone.clone();
        callbacks::listen(
            &drop_zone,
            "dragleave",
            move |_ev: web_sys::Event| {
                drop_zone_c.set_attribute("style", STYLE_IDLE).ok();
            },
            closures,
        )?;
    }

    // ---- drop: prevent_default + метаданные ----
    {
        let drop_zone_c = drop_zone.clone();
        let result_c = result.clone();
        callbacks::listen(
            &drop_zone,
            "drop",
            move |ev: web_sys::Event| {
                let drag_ev: web_sys::DragEvent = match ev.dyn_into() {
                    Ok(e) => e,
                    Err(_) => {
                        result_c.set_text_content(Some("событие не DragEvent"));
                        return;
                    }
                };
                drag_drop::prevent_default(&drag_ev);
                drop_zone_c.set_attribute("style", STYLE_IDLE).ok();

                if !drag_drop::has_files(&drag_ev) {
                    result_c.set_text_content(Some(
                        "В drop-событии нет файлов (возможно, перетащили текст или ссылку).",
                    ));
                    return;
                }

                let list = match drag_drop::files_from_drop_event(&drag_ev) {
                    Some(l) => l,
                    None => {
                        result_c.set_text_content(Some("не удалось получить FileList"));
                        return;
                    }
                };

                let message = format_dropped_files(&list);
                result_c.set_text_content(Some(&message));
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.6
// ---------------------------------------------------------------

/// Формирует человекочитаемый список перетащенных файлов.
///
/// Для каждого файла — размер и MIME-тип. Если браузер не определил
/// MIME (пустая строка `file.type_()`), используется fallback по
/// расширению через `format_mime_line`.
fn format_dropped_files(list: &web_sys::FileList) -> String {
    let count = list.length();
    if count == 0 {
        return "Список файлов пуст.".to_string();
    }

    let mut out = format!("Перетащено файлов: {count}\n\n");
    for i in 0..count {
        let file = match list.get(i) {
            Some(f) => f,
            None => continue,
        };
        let meta = file_api::FileMetadata::from_file(&file);
        let effective = file_utils::effective_mime_type(&file);

        let mime_line: String =
            format_mime_line(&meta.mime_type, &effective, &meta.name);

        out.push_str(&format!(
            "{}. {}\n   Размер: {}\n   {mime_line}\n\n",
            i + 1,
            meta.name,
            file_api::format_file_size(meta.size),
        ));
    }
    return out;
}

/// Формирует строку с MIME-типом для отчёта.
///
/// Если `declared` непустой — использует его. Иначе берёт `effective`
/// (полученный через `file_utils::effective_mime_type`) и добавляет
/// пометку об источнике — расширение файла.
///
/// Внутри используется предобъявленный `result` с присваиванием
/// в ветках; финальный `return result;` завершает функцию — этот
/// паттерн безопасен для проекта.
fn format_mime_line(declared: &str, effective: &str, file_name: &str) -> String {
    let result: String;
    if declared.is_empty() {
        let ext = file_utils::get_extension(file_name).unwrap_or("");
        result = format!("MIME: {effective} (по расширению «{ext}»)");
    } else {
        result = format!("MIME: {declared}");
    }
    return result;
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_drag_drop_section`.
- `wasm_bindgen::prelude::*` — для `JsValue` в `run()`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` при приведении к `DragEvent`.
- `callbacks`, `EventClosures`, `dom`, `drag_drop`, `file_api`, `file_utils`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- **`dragover`** — обязательно вызывает `prevent_default`. Без этого браузер не разрешит drop на этой области. Также подсвечивает drop-зону.
- **`dragleave`** — снимает подсветку, когда курсор уходит.
- **`drop`** — снова вызывает `prevent_default`, снимает подсветку, проверяет `has_files` и формирует список метаданных.
- **`has_files`** — защита от drop текста/ссылок.
- **`format_mime_line`** — предобъявленный `result` с присваиванием в ветках `if/else`. Это критично для окружения проекта: `format!` в tail-позиции блока `if/else` иногда выводится как `()` и даёт ошибку `expected String, found ()`. Паттерн с `let result: String;` надёжен.
- **`format_dropped_files`** — `mut out` нужен, потому что переменная мутируется через `push_str`.

---

### `tests/wasm_drag_drop.rs`

Расширенный набор тестов: поведение `prevent_default` при `cancelable: true`/`false`, извлечение файлов из события без `DataTransfer`, работа `has_files`.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::drag_drop::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_prevent_default_cancelable_true() {
    // Событие с cancelable = true — prevent_default выставляет флаг.
    let init = web_sys::DragEventInit::new();
    init.set_bubbles(true);
    init.set_cancelable(true);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    prevent_default(&event);
    assert!(event.default_prevented());
}

#[wasm_bindgen_test]
fn test_prevent_default_cancelable_false() {
    // Событие с cancelable = false — prevent_default не выставляет флаг,
    // потому что спецификация DOM запрещает отмену таких событий.
    let init = web_sys::DragEventInit::new();
    init.set_bubbles(true);
    init.set_cancelable(false);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    prevent_default(&event);
    assert!(!event.default_prevented());
}

#[wasm_bindgen_test]
fn test_files_from_drop_event_no_data_transfer() {
    let init = web_sys::DragEventInit::new();
    init.set_cancelable(true);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    assert!(files_from_drop_event(&event).is_none());
}

#[wasm_bindgen_test]
fn test_metadata_from_drop_event_no_data_transfer() {
    let init = web_sys::DragEventInit::new();
    init.set_cancelable(true);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    assert!(metadata_from_drop_event(&event).is_empty());
}

#[wasm_bindgen_test]
fn test_has_files_no_data_transfer() {
    let init = web_sys::DragEventInit::new();
    init.set_cancelable(true);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    assert!(!has_files(&event));
}

#[wasm_bindgen_test]
fn test_drop_event_type() {
    let init = web_sys::DragEventInit::new();
    init.set_bubbles(true);
    init.set_cancelable(true);

    let event = web_sys::DragEvent::new_with_event_init_dict("drop", &init)
        .expect("DragEvent::new_with_event_init_dict");

    assert_eq!(event.type_(), "drop");
    assert!(event.cancelable());
    assert!(event.bubbles());
}
```

**Что проверяем:**

- **`prevent_default` при `cancelable: true`** — флаг `defaultPrevented` выставляется.
- **`prevent_default` при `cancelable: false`** — флаг НЕ выставляется.
- **`files_from_drop_event` без `DataTransfer`** — `None`.
- **`metadata_from_drop_event` без `DataTransfer`** — пустой вектор.
- **`has_files` без `DataTransfer`** — `false`.
- **Базовые свойства DragEvent** — тип `"drop"`, `cancelable`, `bubbles`.

**Тест «с файлами» не пишем:** в headless-браузере нельзя программно положить файлы в `DataTransfer` без реального пользовательского действия (защита от подделки). Интеграционная проверка делается вручную через демо-секцию.

---

### `tests/wasm_file_utils.rs` — новые тесты

Добавляются проверки расширенной таблицы MIME и `effective_mime_type`:

```rust
#[wasm_bindgen_test]
fn test_mime_from_extension_markdown() {
    assert_eq!(mime_from_extension("md"), "text/markdown");
    assert_eq!(mime_from_extension("markdown"), "text/markdown");
    assert_eq!(mime_from_extension("MD"), "text/markdown");
}

#[wasm_bindgen_test]
fn test_mime_from_extension_web_formats() {
    assert_eq!(mime_from_extension("html"), "text/html");
    assert_eq!(mime_from_extension("htm"), "text/html");
    assert_eq!(mime_from_extension("css"), "text/css");
    assert_eq!(mime_from_extension("js"), "text/javascript");
    assert_eq!(mime_from_extension("mjs"), "text/javascript");
    assert_eq!(mime_from_extension("wasm"), "application/wasm");
}

#[wasm_bindgen_test]
fn test_mime_from_extension_media() {
    assert_eq!(mime_from_extension("mp3"), "audio/mpeg");
    assert_eq!(mime_from_extension("mp4"), "video/mp4");
    assert_eq!(mime_from_extension("webm"), "video/webm");
}

#[wasm_bindgen_test]
fn test_effective_mime_type_uses_declared() {
    // Если браузер определил MIME — используем его.
    let parts = js_sys::Array::of1(&"x".into());
    let bag = web_sys::FilePropertyBag::new();
    bag.set_type("image/png");
    let file = web_sys::File::new_with_str_sequence_and_options(
        &parts,
        "pic.png",
        &bag,
    )
    .expect("File::new_with_str_sequence_and_options");

    assert_eq!(effective_mime_type(&file), "image/png");
}

#[wasm_bindgen_test]
fn test_effective_mime_type_falls_back_to_extension() {
    // Если браузер вернул пустой MIME — используем расширение.
    let parts = js_sys::Array::of1(&"# Title".into());
    let bag = web_sys::FilePropertyBag::new();
    bag.set_type(""); // явно пустой
    let file = web_sys::File::new_with_str_sequence_and_options(
        &parts,
        "readme.md",
        &bag,
    )
    .expect("File::new_with_str_sequence_and_options");

    assert_eq!(effective_mime_type(&file), "text/markdown");
}

#[wasm_bindgen_test]
fn test_effective_mime_type_unknown_extension() {
    // Неизвестное расширение → octet-stream.
    let parts = js_sys::Array::of1(&"data".into());
    let bag = web_sys::FilePropertyBag::new();
    bag.set_type("");
    let file = web_sys::File::new_with_str_sequence_and_options(
        &parts,
        "file.xyz",
        &bag,
    )
    .expect("File::new_with_str_sequence_and_options");

    assert_eq!(effective_mime_type(&file), "application/octet-stream");
}
```

**Ключевое:** тест `test_effective_mime_type_falls_back_to_extension` воспроизводит именно тот баг, с которым вы столкнулись: `.md` с пустым MIME от браузера. Функция должна вернуть `text/markdown`.

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

### Тесты подглавы 18.6

```bash
# Проверка типов тестов
cargo check --target wasm32-unknown-unknown --tests

# Только тесты Drag & Drop
wasm-pack test --headless --chrome --test wasm_drag_drop

# Тесты утилит (включая новые для effective_mime_type)
wasm-pack test --headless --chrome --test wasm_file_utils

# Все тесты проекта
npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.6. Drag & Drop — перетащите файлы** с пунктирной drop-зоной.

**Сценарий 1 — перетаскивание `.md`:**
```
Перетащено файлов: 1

1. chapter-18.5.md
   Размер: 32.51 KB
   MIME: text/markdown (по расширению «md»)
```

**Сценарий 2 — перетаскивание PDF (браузер знает MIME):**
```
Перетащено файлов: 1

1. document.pdf
   Размер: 234.56 KB
   MIME: application/pdf
```

**Сценарий 3 — перетаскивание текста (не файла):**
```
В drop-событии нет файлов (возможно, перетащили текст или ссылку).
```

**Сценарий 4 — наведение без отпускания:**
- Рамка drop-зоны становится зелёной, фон — светло-зелёным.
- При уходе курсора — возврат к исходному виду.

---

## Итоги подглавы 18.6

- **`DragEvent`** — событие перетаскивания; получает `DataTransfer` через `data_transfer()`.
- **`DataTransfer`** — контейнер с данными; `files()` возвращает `Option<FileList>`.
- **`prevent_default`** — критично на `dragover` и `drop`; без него браузер откроет файл вместо нашей обработки.
- **`has_files`** — защита от drop текста/ссылок.
- **Визуальная обратная связь** — обязательный элемент UX.
- **MIME fallback** — браузер не всегда знает MIME по расширению; `effective_mime_type` решает эту проблему.

Модуль `drag_drop.rs` автономен, все tail-функции оформлены через явный `return`. `file_utils.rs` расширен — но существующее поведение сохранено. Демо-секция самодостаточна и демонстрирует четыре сценария. Тесты покрывают граничные случаи.

## Замечания на будущее

Три правила, зафиксированные в этой подглаве:

1. **`prevent_default` в тестах работает только на событиях с `cancelable: true`.** Значение по умолчанию — `false`, и тогда `defaultPrevented` останется `false`. Это требование спецификации DOM.

2. **`file.type_()` может возвращать пустую строку** для расширений, не зарегистрированных в ОС. Всегда используйте `effective_mime_type`, а не `file.type_()` напрямую.

3. **`format!` в tail-позиции блока `if/else`, присвоенного в `let`, даёт ошибку `expected String, found ()`.** Правильный паттерн — предобъявленная переменная:
   ```rust
   let result: String;
   if cond {
       result = format!(...);
   } else {
       result = format!(...);
   }
   return result;
   ```

В следующей подглаве (18.7) мы разберём перетаскивание **папок** — там потребуется `webkitGetAsEntry` и рекурсивный обход дерева через `FileSystemDirectoryReader`.