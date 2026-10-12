# Глава 18. File API

## 18.1. Базовые типы: `File`, `Blob`, `FileList`, метаданные

В этой подглаве мы разберём фундамент File API: что такое `File` как частный случай `Blob`, как браузер отдаёт список выбранных файлов и как прочитать метаданные каждого файла — имя, размер, MIME-тип и дату последнего изменения. Эти данные — отправная точка для любого сценария работы с файлами: прежде чем читать содержимое, мы должны понимать, что именно нам дал пользователь.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # pub mod file_api; (уже подключён)
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.1
│   ├── file_api.rs                # модуль подглавы 18.1
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует (экспортирует log!)
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_file_api.rs           # тесты подглавы 18.1
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/file_api.rs` — новый модуль, вводимый этой подглавой. Он автономен и не зависит от других модулей проекта.
- `src/app.rs` — обновлён: в `run()` добавлена демо-секция «18.1. File, Blob, FileList, метаданные». Существующие секции не тронуты.
- `tests/wasm_file_api.rs` — тесты, соответствующие модулю.

---

## Ссылки для детального изучения

- **MDN: File** — интерфейс `File`: https://developer.mozilla.org/en-US/docs/Web/API/File
- **MDN: Blob** — интерфейс `Blob` (базовый для `File`): https://developer.mozilla.org/en-US/docs/Web/API/Blob
- **MDN: FileList** — список выбранных файлов: https://developer.mozilla.org/en-US/docs/Web/API/FileList
- **MDN: `<input type="file">`** — элемент выбора файла: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/file
- **web-sys: File** — Rust-обёртка: https://docs.rs/web-sys/latest/web_sys/struct.File.html
- **web-sys: FileList** — Rust-обёртка: https://docs.rs/web-sys/latest/web_sys/struct.FileList.html
- **Спецификация W3C File API**: https://w3c.github.io/FileAPI/

---

## Содержание новых / обновлённых файлов

### `src/file_api.rs`

Модуль автономен: зависимость только от `web-sys` (типы `File` и `FileList`). `wasm_bindgen::prelude` не нужен — мы не объявляем `#[wasm_bindgen]`-функций и не работаем с `JsValue`.

```rust
//! Работа с File, Blob, FileList и метаданными файлов.
//! Модуль автономен: не зависит от других модулей проекта.

use web_sys::{File, FileList};

/// Метаданные файла: имя, размер, MIME-тип, дата изменения.
#[derive(Debug, Clone, PartialEq)]
pub struct FileMetadata {
    pub name: String,
    pub size: f64,
    pub mime_type: String,
    pub last_modified: f64,
}

impl FileMetadata {
    /// Извлекает метаданные из `web_sys::File`.
    pub fn from_file(file: &File) -> Self {
        Self {
            name: file.name(),
            size: file.size(),
            mime_type: file.type_(),
            last_modified: file.last_modified(),
        }
    }
}

/// Возвращает количество файлов в списке.
pub fn file_list_length(list: &FileList) -> u32 {
    list.length()
}

/// Возвращает файл по индексу, если он существует.
pub fn file_list_get(list: &FileList, index: u32) -> Option<File> {
    list.get(index)
}

/// Преобразует `FileList` в вектор метаданных.
pub fn file_list_metadata(list: &FileList) -> Vec<FileMetadata> {
    (0..list.length())
        .filter_map(|i| list.get(i))
        .map(|f| FileMetadata::from_file(&f))
        .collect()
}

/// Проверяет, является ли файл изображением (по MIME-типу).
pub fn is_image(file: &File) -> bool {
    file.type_().starts_with("image/")
}

/// Форматирует размер файла в человекочитаемый вид.
pub fn format_file_size(bytes: f64) -> String {
    const UNITS: &[&str] = &["B", "KB", "MB", "GB", "TB"];
    let mut size = bytes;
    let mut unit_idx: usize = 0;
    while size >= 1024.0 && unit_idx < UNITS.len() - 1 {
        size /= 1024.0;
        unit_idx += 1;
    }
    return format!("{:.2} {}", size, UNITS[unit_idx]);
}
```

**Ключевые моменты:**

- `File` в `web-sys` — это отдельный тип, но по спецификации он наследуется от `Blob` и добавляет поля `name` и `lastModified`. Всё, что работает с `Blob`, работает и с `File`.
- `FileList` — не `Vec`: это live-коллекция DOM, к которой обращаются через `length()` и `get(index)`. `get` возвращает `Option<File>`, что безопаснее прямого индексирования.
- `file.type_()` — не `type`, потому что `type` — зарезервированное слово Rust. Возвращает MIME-тип, определённый браузером по расширению и сигнатуре; для неизвестных форматов — пустая строка.
- `format_file_size` завершается явным `return format!(...);` — это конвенция проекта (см. `format_read_text_result`, `format_read_binary_result` и другие из последующих подглав).

---

### `src/app.rs` — новая секция 18.1

Ниже — новый блок для `app.rs`. Функция `build_input_section` самодостаточна: она не опирается на другие секции, не требует чужих хелперов и может быть вставлена в `app.rs` как есть.

```rust
//! Демонстрация подглавы 18.1: File, Blob, FileList, метаданные.
//!
//! Секция: `<input type="file" multiple>` → таблица метаданных
//! с человекочитаемым размером и форматированной датой.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

use crate::callbacks::{self, EventClosures};
use crate::file_api;
use crate::{dom, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.1 ----
    build_input_section(&container, &closures)?;

    log!("Section 18.1 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.1: File, Blob, FileList, метаданные
// ---------------------------------------------------------------

fn build_input_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.1. File, Blob, FileList, метаданные"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите один или несколько файлов — отобразятся их \
         метаданные: имя, размер (в человекочитаемом виде), MIME-тип \
         и дата последнего изменения.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    input.set_multiple(true);
    dom::append(&section, &input)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(файлы не выбраны)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let result_c = result.clone();
    callbacks::listen(
        &input,
        "change",
        move |ev: web_sys::Event| {
            let target = match ev.target() {
                Some(t) => t,
                None => {
                    result_c.set_text_content(Some("нет target"));
                    return;
                }
            };
            let input_el: web_sys::HtmlInputElement = match target.dyn_into() {
                Ok(el) => el,
                Err(_) => {
                    result_c.set_text_content(Some("target не input"));
                    return;
                }
            };
            let files = match input_el.files() {
                Some(f) => f,
                None => {
                    result_c.set_text_content(Some("файлы не выбраны"));
                    return;
                }
            };
            let meta = file_api::file_list_metadata(&files);
            let message = format_metadata_table(&meta);
            result_c.set_text_content(Some(&message));
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.1
// ---------------------------------------------------------------

/// Формирует текстовую таблицу метаданных для отображения в браузере.
fn format_metadata_table(items: &[file_api::FileMetadata]) -> String {
    if items.is_empty() {
        return "Файлы не выбраны.".to_string();
    }

    let mut out = format!("Выбрано файлов: {}\n\n", items.len());
    for (i, m) in items.iter().enumerate() {
        let mime = if m.mime_type.is_empty() {
            "(не определён)".to_string()
        } else {
            m.mime_type.clone()
        };
        let modified = format_timestamp(m.last_modified);
        out.push_str(&format!(
            "{}. {}\n   Размер: {} ({} байт)\n   MIME: {}\n   Изменён: {}\n\n",
            i + 1,
            m.name,
            file_api::format_file_size(m.size),
            m.size as u64,
            mime,
            modified,
        ));
    }
    return out;
}

/// Преобразует миллисекунды Unix-эпохи в строку `YYYY-MM-DD HH:MM:SS`.
fn format_timestamp(ms: f64) -> String {
    let date = js_sys::Date::new(&JsValue::from_f64(ms));
    return format!(
        "{:04}-{:02}-{:02} {:02}:{:02}:{:02}",
        date.get_full_year(),
        date.get_month() + 1, // в JS месяцы 0-индексированы
        date.get_date(),
        date.get_hours(),
        date.get_minutes(),
        date.get_seconds(),
    );
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_input_section`.
- `wasm_bindgen::prelude::*` — нужен для `JsValue` (в `run()`, `format_timestamp`).
- `wasm_bindgen::JsCast` — нужен для `.dyn_into()`.
- `callbacks`, `EventClosures`, `dom`, `file_api`, `log` — все используются.
- Никаких неиспользуемых импортов нет.

**Особенности:**

- `set_multiple(true)` — разрешает выбор нескольких файлов; `FileList` тогда содержит все.
- Обработчик `change` читает `ev.target()`, приводит к `HtmlInputElement`, получает `files()` (это `Option<FileList>` — `None`, если пользователь закрыл диалог без выбора).
- `format_metadata_table` — самодостаточная функция, использующая только `file_api` и `js_sys::Date`. Она не тянет за собой код других подглав.
- `format_timestamp` — тоже локальная утилита; она превращает миллисекунды в читаемую дату через `js_sys::Date`. Это стандартный способ работы с `lastModified` в web-sys.

---

### `tests/wasm_file_api.rs`

Тесты подглавы 18.1. Используют `FilePropertyBag`, чтобы задать MIME-тип явно: браузер сам по себе тип для `File::new_with_str_sequence` не устанавливает.

```rust
use wasm_bindgen::JsValue;
use wasm_bindgen_test::*;
use hello_wasm02::file_api::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_format_file_size() {
    assert_eq!(format_file_size(512.0), "512.00 B");
    assert_eq!(format_file_size(2048.0), "2.00 KB");
    assert_eq!(format_file_size(1024.0 * 1024.0), "1.00 MB");
    assert_eq!(format_file_size(1024.0 * 1024.0 * 1024.0), "1.00 GB");
}

#[wasm_bindgen_test]
fn test_file_metadata_from_file() {
    // FilePropertyBag позволяет задать MIME-тип явно —
    // иначе File::type_() вернёт пустую строку.
    let parts = js_sys::Array::of1(&JsValue::from_str("hello"));
    let bag = web_sys::FilePropertyBag::new();
    bag.set_type("text/plain");

    let file = web_sys::File::new_with_str_sequence_and_options(
        &parts,
        "test.txt",
        &bag,
    )
    .expect("File::new_with_str_sequence_and_options");

    let meta = FileMetadata::from_file(&file);
    assert_eq!(meta.name, "test.txt");
    assert_eq!(meta.mime_type, "text/plain");
    assert!(meta.size > 0.0);
    assert!(meta.last_modified > 0.0);
}

#[wasm_bindgen_test]
fn test_is_image_by_mime() {
    let parts = js_sys::Array::of1(&JsValue::from_str("png-bytes"));
    let bag = web_sys::FilePropertyBag::new();
    bag.set_type("image/png");

    let file = web_sys::File::new_with_str_sequence_and_options(
        &parts,
        "pic.png",
        &bag,
    )
    .expect("File::new_with_str_sequence_and_options");

    assert!(is_image(&file));
}
```

**Что проверяем:**

- `format_file_size` корректно переключает единицы измерения на границах 1 KB / 1 MB / 1 GB.
- `FileMetadata::from_file` извлекает все четыре поля, включая MIME и `last_modified`.
- `is_image` распознаёт MIME-префикс `image/`.

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

### Тесты подглавы 18.1

```bash
# Только тесты этой подглавы
wasm-pack test --headless --chrome --test wasm_file_api

# То же самое для Firefox
wasm-pack test --headless --firefox --test wasm_file_api

# Все тесты проекта (включая подглаву 18.1)
npm run test
```

### Что ожидать в браузере

После `npm run start` и открытия страницы появится секция **18.1. File, Blob, FileList, метаданные** с полем выбора файлов. После выбора одного или нескольких файлов в блоке результата отобразится:

```
Выбрано файлов: 2

1. document.pdf
   Размер: 234.56 KB (240192 байт)
   MIME: application/pdf
   Изменён: 2025-03-14 09:27:11

2. photo.jpg
   Размер: 1.24 MB (1304576 байт)
   MIME: image/jpeg
   Изменён: 2025-03-10 22:05:48
```

Если MIME-тип не определён (неизвестное расширение), в поле MIME отобразится `(не определён)`.

---

## Итоги подглавы 18.1

Мы разобрали фундамент File API:

- **`File`** — частный случай `Blob`, добавляющий имя и дату изменения.
- **`Blob`** — неизменяемый бинарный объект; базовые методы `size()` и `type_()`.
- **`FileList`** — живая DOM-коллекция с доступом через `length()` и `get(index) → Option<File>`.
- **Метаданные** — `name()`, `size()`, `type_()`, `last_modified()`.
- **`<input type="file" multiple>`** — единственный способ получить `FileList` без Drag & Drop (о котором — в 18.6).
- **`FilePropertyBag`** — словарь опций, позволяющий задать MIME-тип при создании `File` (используется в тестах).

Модуль `file_api.rs` автономен, все функции имеют явный `return` в tail-позиции, ни одного неиспользуемого импорта. Демо-секция в `app.rs` самодостаточна: её можно вставить рядом с другими секциями без правок последних.