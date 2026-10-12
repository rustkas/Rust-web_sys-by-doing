# Глава 18. File API

## 18.7. Drag & Drop папок: `webkitGetAsEntry`

В подглаве 18.6 мы научились принимать отдельные файлы через перетаскивание. Но что, если пользователь хочет перетащить целую папку? Например, директорию с сотней изображений для пакетной конвертации? `DataTransfer::files()` в этом случае вернёт пустой список — папка не является файлом. Для работы с папками нужен другой механизм: `DataTransferItem.webkitGetAsEntry()` — метод, возвращающий `FileSystemEntry`, который может быть как файлом, так и директорией.

Этот API относится к устаревшему **File and Directory Entries API**, но остаётся единственным способом получить доступ к перетащенной папке — современный File System Access API (`showDirectoryPicker`) требует явного вызова диалога, а не drop-события. Поэтому для drag&drop папок `webkitGetAsEntry` по-прежнему незаменим.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # + новые фичи web-sys для FileSystem API
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod folder_drop;
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.7
│   ├── folder_drop.rs             # НОВЫЙ: модуль подглавы 18.7
│   ├── drag_drop.rs               # из 18.6, не меняется
│   ├── file_utils.rs              # из 18.6, не меняется
│   ├── data_url.rs                # из 18.5, не меняется
│   ├── base64_utils.rs            # из 18.5, не меняется
│   ├── object_url.rs              # из 18.4, не меняется
│   ├── file_api.rs                # из 18.1, не меняется
│   ├── file_reader.rs             # из 18.2, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_folder_drop.rs        # НОВЫЙ: тесты подглавы 18.7
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/folder_drop.rs` — модуль, вводимый этой подглавой. Автономен: зависит только от `web-sys`, `js-sys`, `wasm-bindgen`.
- `src/lib.rs` — **обязательно** дополняется строкой `pub mod folder_drop;`.
- `src/app.rs` — обновлён: добавлена демо-секция «18.7. Drag & Drop папок».
- `tests/wasm_folder_drop.rs` — новый файл с тестами.

**Новые фичи в `Cargo.toml`:** добавляем в секцию `[dependencies.web-sys] features`:

```toml
"DataTransferItem",
"DataTransferItemList",         # ← НОВОЕ: без неё DataTransfer::items() недоступен
"FileSystemEntry",
"FileSystemDirectoryEntry",
"FileSystemFileEntry",
"FileSystemDirectoryReader",
```

Без `DataTransferItemList` метод `DataTransfer::items()` просто не существует в web-sys, и `cargo check` падает с `no method named 'items'`.

---

## Ссылки для детального изучения

- **MDN: DataTransferItem.webkitGetAsEntry()** — https://developer.mozilla.org/en-US/docs/Web/API/DataTransferItem/webkitGetAsEntry
- **MDN: FileSystemEntry** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemEntry
- **MDN: FileSystemDirectoryEntry** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemDirectoryEntry
- **MDN: FileSystemDirectoryEntry.createReader()** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemDirectoryEntry/createReader
- **MDN: FileSystemDirectoryReader.readEntries()** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemDirectoryReader/readEntries
- **MDN: FileSystemFileEntry.file()** — https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileEntry/file
- **MDN: `<input webkitdirectory>`** — https://developer.mozilla.org/en-US/docs/Web/API/HTMLInputElement/webkitdirectory
- **MDN: Asynchronous recursion in Rust (Box::pin)** — https://rust-lang.github.io/async-book/07_workarounds/04_recursion.html

---

## Содержание новых / обновлённых файлов

### `src/lib.rs` — подключение модуля

```rust
// src/lib.rs

// … существующие модули проекта …

// ---- Глава 18. File API ----
pub mod file_api;       // 18.1
pub mod file_reader;    // 18.2
pub mod file_utils;     // 18.3
pub mod object_url;     // 18.4
pub mod data_url;       // 18.5
pub mod base64_utils;   // 18.5
pub mod drag_drop;      // 18.6
pub mod folder_drop;    // 18.7 ← НОВОЕ
```

---

### `src/folder_drop.rs`

Модуль автономен. Ключевая особенность — **рекурсивная асинхронная функция** `scan_entry` оформлена не как `async fn` (иначе компилятор потребует `Box::pin` внутри неё, что невозможно сделать для самой рекурсии), а как **обычная функция, возвращающая `Pin<Box<dyn Future<...>>>`**. Параметры взяты по владению (`FileSystemEntry`, `String`) — так future получается `'static` и не зависит от времён жизни вызывающего кода.

```rust
//! Drag & Drop папок через webkitGetAsEntry.
//!
//! Модуль автономен: не зависит от других модулей проекта.
//!
//! Работает через устаревший File and Directory Entries API:
//! `FileSystemEntry` → `FileSystemDirectoryEntry` → `createReader()`
//! → `readEntries()` (повторять, пока не вернётся пустой массив).
//!
//! Рекурсивный обход реализован через `Box::pin` — рекурсивная
//! `async fn` в Rust требует явной косвенности, иначе future
//! получается бесконечно большого размера.

use std::future::Future;
use std::pin::Pin;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{
    DataTransferItem, FileSystemDirectoryEntry, FileSystemDirectoryReader,
    FileSystemEntry, FileSystemFileEntry,
};

/// Информация о файле, найденном при рекурсивном обходе папки.
#[derive(Debug, Clone, PartialEq)]
pub struct FolderEntry {
    /// Полный путь от корня перетащенной папки, например
    /// `photos/2024/img001.png`.
    pub path: String,
    /// Размер файла в байтах.
    pub size: f64,
    /// MIME-тип (может быть пустой строкой).
    pub mime_type: String,
}

/// Рекурсивно обходит `FileSystemEntry` и собирает все файлы.
///
/// Возвращает `Pin<Box<dyn Future<...>>>` вместо `async fn` —
/// это обязательное условие для рекурсивных асинхронных функций:
/// без бокса компилятор не может вычислить размер future.
///
/// Параметры берутся по владению, чтобы future был `'static` и
/// не зависел от времени жизни вызывающего кода.
///
/// * Для файла (`FileSystemFileEntry`) — получает `File` через
///   `file()` и добавляет запись в результат.
/// * Для директории (`FileSystemDirectoryEntry`) — вызывает
///   `readEntries()` в цикле (пока не вернётся пустой массив),
///   затем рекурсивно обходит каждый элемент.
///
/// `base_path` — накопленный путь от корня; для корневого
/// вызова передаётся `String::new()`.
pub fn scan_entry(
    entry: FileSystemEntry,
    base_path: String,
) -> Pin<Box<dyn Future<Output = Vec<FolderEntry>>>> {
    return Box::pin(async move {
        let mut results = Vec::new();
        let name = entry.name();

        let full_path: String;
        if base_path.is_empty() {
            full_path = name;
        } else {
            full_path = format!("{base_path}/{name}");
        }

        if entry.is_file() {
            if let Ok(file_entry) = entry.dyn_into::<FileSystemFileEntry>() {
                if let Some(record) = read_file_entry(&file_entry, &full_path).await {
                    results.push(record);
                }
            }
        } else if entry.is_directory() {
            if let Ok(dir_entry) = entry.dyn_into::<FileSystemDirectoryEntry>() {
                let entries = read_directory_entries(&dir_entry).await;
                for child in entries {
                    let sub = scan_entry(child, full_path.clone()).await;
                    results.extend(sub);
                }
            }
        }

        return results;
    });
}

/// Читает `FileSystemFileEntry` и превращает его в `FolderEntry`.
async fn read_file_entry(entry: &FileSystemFileEntry, path: &str) -> Option<FolderEntry> {
    let file = file_from_entry(entry).await?;
    return Some(FolderEntry {
        path: path.to_string(),
        size: file.size(),
        mime_type: file.type_(),
    });
}

/// Обёртка над `FileSystemFileEntry::file()` — возвращает `File` через Promise.
async fn file_from_entry(entry: &FileSystemFileEntry) -> Option<web_sys::File> {
    let entry_clone = entry.clone();
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let on_success = Closure::once_into_js(move |f: web_sys::File| {
            let _ = resolve_c.call1(&JsValue::NULL, &f);
        });
        let on_error = Closure::once_into_js(move |err: JsValue| {
            let _ = reject.call1(&JsValue::NULL, &err);
        });
        let _ = entry_clone.file_with_callback_and_callback(
            on_success.unchecked_ref(),
            on_error.unchecked_ref(),
        );
    });
    let value = JsFuture::from(promise).await.ok()?;
    return value.dyn_into::<web_sys::File>().ok();
}

/// Читает все элементы директории, повторяя `readEntries()`,
/// пока не вернётся пустой массив.
async fn read_directory_entries(dir: &FileSystemDirectoryEntry) -> Vec<FileSystemEntry> {
    let reader = dir.create_reader();
    let mut all: Vec<FileSystemEntry> = Vec::new();

    loop {
        let batch = read_entries_once(&reader).await;
        if batch.is_empty() {
            break;
        }
        all.extend(batch);
    }
    return all;
}

/// Один вызов `readEntries()` — возвращает пакет записей.
///
/// Согласно спецификации, `readEntries` возвращает не все записи
/// сразу, а порциями; нужно вызывать повторно до пустого массива.
async fn read_entries_once(reader: &FileSystemDirectoryReader) -> Vec<FileSystemEntry> {
    let reader_clone = reader.clone();
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let on_success = Closure::once_into_js(move |entries: js_sys::Array| {
            let _ = resolve_c.call1(&JsValue::NULL, &entries);
        });
        let on_error = Closure::once_into_js(move |err: JsValue| {
            let _ = reject.call1(&JsValue::NULL, &err);
        });
        let _ = reader_clone.read_entries_with_callback_and_callback(
            on_success.unchecked_ref(),
            on_error.unchecked_ref(),
        );
    });

    let value = match JsFuture::from(promise).await {
        Ok(v) => v,
        Err(_) => return Vec::new(),
    };

    let array = match value.dyn_into::<js_sys::Array>() {
        Ok(a) => a,
        Err(_) => return Vec::new(),
    };

    let mut entries = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        if let Ok(entry) = array.get(i).dyn_into::<FileSystemEntry>() {
            entries.push(entry);
        }
    }
    return entries;
}

/// Извлекает `FileSystemEntry` из `DataTransferItem`.
///
/// Возвращает `None`, если элемент не является файлом или
/// директорией (например, перетащили текст или ссылку).
pub fn entry_from_item(item: &DataTransferItem) -> Option<FileSystemEntry> {
    return item.webkit_get_as_entry().ok().flatten();
}
```

**Проверка импортов:**

- `std::future::Future`, `std::pin::Pin` — используются в сигнатуре `scan_entry`.
- `wasm_bindgen::prelude::*` — для `JsValue`, `Closure`.
- `wasm_bindgen::JsCast` — для `.dyn_into()` и `.unchecked_ref()`.
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{DataTransferItem, FileSystemDirectoryEntry, FileSystemDirectoryReader, FileSystemEntry, FileSystemFileEntry}` — все используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- **Рекурсия через `Box::pin`.** Прямая `async fn scan_entry` не компилируется: future получается бесконечного размера из-за вложенного вызова. Оформление через `fn -> Pin<Box<dyn Future<...>>>` разрывает цикл.
- **Параметры по владению.** `entry: FileSystemEntry` и `base_path: String` — иначе future захватывает ссылки, и компилятор требует явных времён жизни. Владение даёт `'static`-future, который можно свободно возвращать и хранить.
- **`readEntries` возвращает порции.** Согласно спецификации, `readEntries()` может вернуть не все записи сразу — вызываем в цикле до пустого массива.
- **`file_with_callback_and_callback`** — асинхронный метод `FileSystemFileEntry::file()`; оборачиваем его в `Promise` так же, как `FileReader`.
- **`entry.is_file()` / `entry.is_directory()`** — методы `FileSystemEntry`; по ним определяем тип.
- **Все функции оформлены с явным `return`** — конвенция проекта.

---

### `src/app.rs` — секция 18.7

Функция `build_folder_drop_section` самодостаточна: обрабатывает и `<input type="file" webkitdirectory>`, и drop-зону для папок.

```rust
//! Демонстрация подглавы 18.7: Drag & Drop папок.
//!
//! Секция: выбор папки через `<input webkitdirectory>` и
//! drop-зона для перетаскивания папок с рекурсивным обходом.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_api, file_utils, folder_drop, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.7 ----
    build_folder_drop_section(&container, &closures)?;

    log!("Section 18.7 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.7: Drag & Drop папок
// ---------------------------------------------------------------

fn build_folder_drop_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.7. Drag & Drop папок"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите папку через поле ниже (webkitdirectory) или \
         перетащите папку в drop-зону. Будет построено дерево файлов \
         с полными путями, размерами и MIME-типами.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    // ---- <input type="file" webkitdirectory> ----
    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    // webkitdirectory — нестандартный, но широко поддержанный атрибут.
    input.set_attribute("webkitdirectory", "")?;
    input.set_attribute("directory", "")?;
    input.set_attribute("multiple", "")?;
    dom::append(&section, &input)?;

    // ---- Drop-зона ----
    const STYLE_IDLE: &str = "border: 2px dashed #888; padding: 30px; \
        text-align: center; margin: 10px 0; color: #555; \
        cursor: pointer; border-radius: 6px;";
    const STYLE_HOVER: &str = "border: 2px dashed #2b8; padding: 30px; \
        text-align: center; margin: 10px 0; color: #2b8; \
        cursor: pointer; border-radius: 6px; background: #f0fff5;";

    let drop_zone = dom::create_element("div")?;
    drop_zone.set_attribute("style", STYLE_IDLE)?;
    drop_zone.set_text_content(Some("Перетащите папку сюда"));
    dom::append(&section, &drop_zone)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    // ---- Обработчик выбора папки через input ----
    {
        let result_c = result.clone();
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
                let files = match input_el.files() {
                    Some(f) => f,
                    None => {
                        result_c.set_text_content(Some("папка не выбрана"));
                        return;
                    }
                };
                let message = format_folder_files(&files);
                result_c.set_text_content(Some(&message));
            },
            closures,
        )?;
    }

    // ---- dragover: prevent_default + подсветка ----
    {
        let drop_zone_c = drop_zone.clone();
        callbacks::listen(
            &drop_zone,
            "dragover",
            move |ev: web_sys::Event| {
                if let Ok(drag_ev) = ev.dyn_into::<web_sys::DragEvent>() {
                    drag_ev.prevent_default();
                    drag_ev.stop_propagation();
                }
                drop_zone_c.set_attribute("style", STYLE_HOVER).ok();
            },
            closures,
        )?;
    }

    // ---- dragleave ----
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

    // ---- drop: рекурсивный обход папок ----
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
                drag_ev.prevent_default();
                drag_ev.stop_propagation();
                drop_zone_c.set_attribute("style", STYLE_IDLE).ok();

                let dt = match drag_ev.data_transfer() {
                    Some(d) => d,
                    None => {
                        result_c.set_text_content(Some("нет DataTransfer"));
                        return;
                    }
                };

                // Собираем все DataTransferItem, у которых есть entry.
                let items = dt.items();
                let mut entries = Vec::new();
                for i in 0..items.length() {
                    if let Some(item) = items.get(i) {
                        if let Some(entry) = folder_drop::entry_from_item(&item) {
                            entries.push(entry);
                        }
                    }
                }

                if entries.is_empty() {
                    result_c.set_text_content(Some(
                        "Нет папок или файлов для обработки.",
                    ));
                    return;
                }

                let result_c = result_c.clone();
                result_c.set_text_content(Some("Сканируем папки..."));
                spawn_local(async move {
                    let mut all: Vec<folder_drop::FolderEntry> = Vec::new();
                    for entry in entries {
                        // scan_entry принимает владение entry и String.
                        let found = folder_drop::scan_entry(entry, String::new()).await;
                        all.extend(found);
                    }
                    let message = format_folder_entries(&all);
                    result_c.set_text_content(Some(&message));
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.7
// ---------------------------------------------------------------

/// Формирует отчёт для `<input webkitdirectory>` — на основе FileList.
fn format_folder_files(list: &web_sys::FileList) -> String {
    let count = list.length();
    if count == 0 {
        return "Папка пуста.".to_string();
    }

    let mut out = format!("Выбрано файлов в папке: {count}\n\n");
    for i in 0..count {
        let file = match list.get(i) {
            Some(f) => f,
            None => continue,
        };
        let meta = file_api::FileMetadata::from_file(&file);
        let effective = file_utils::effective_mime_type(&file);

        let mime_line: String =
            format_mime_line(&meta.mime_type, &effective, &meta.name);

        // webkitRelativePath — нестандартное свойство File.
        // Читаем его динамически через Reflect: web-sys не экспортирует
        // соответствующий метод в текущей версии.
        let path: String = {
            let reflect = js_sys::Reflect::get(
                file.as_ref(),
                &JsValue::from_str("webkitRelativePath"),
            );
            match reflect.ok().and_then(|v| v.as_string()) {
                Some(s) if !s.is_empty() => s,
                _ => meta.name.clone(),
            }
        };

        out.push_str(&format!(
            "{}. {}\n   Размер: {}\n   {mime_line}\n\n",
            i + 1,
            path,
            file_api::format_file_size(meta.size),
        ));
    }
    return out;
}

/// Формирует отчёт для drop-события — на основе рекурсивного обхода.
fn format_folder_entries(items: &[folder_drop::FolderEntry]) -> String {
    if items.is_empty() {
        return "Файлов не найдено.".to_string();
    }

    let total_size: f64 = items.iter().map(|e| e.size).sum();

    let mut out = format!(
        "Найдено файлов: {}\nОбщий размер: {}\n\n",
        items.len(),
        file_api::format_file_size(total_size),
    );

    for (i, item) in items.iter().enumerate() {
        let mime: String;
        if item.mime_type.is_empty() {
            mime = "(не определён)".to_string();
        } else {
            mime = item.mime_type.clone();
        }
        out.push_str(&format!(
            "{}. {}\n   Размер: {}\n   MIME: {}\n\n",
            i + 1,
            item.path,
            file_api::format_file_size(item.size),
            mime,
        ));
    }
    return out;
}

/// Формирует строку с MIME-типом для отчёта (та же логика,
/// что и в подглаве 18.6).
///
/// Использует предобъявленный `result` с присваиванием в ветках;
/// финальный `return result;` завершает функцию — этот паттерн
/// безопасен для окружения проекта.
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

- `RefCell`, `Rc` — используются в `run()` и `build_folder_drop_section`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`.
- `wasm_bindgen_futures::spawn_local` — для `async`-блока из обработчика.
- `callbacks`, `EventClosures`, `dom`, `file_api`, `file_utils`, `folder_drop`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- **`webkitdirectory`** — нестандартный атрибут, но поддерживается всеми современными браузерами.
- **`webkitRelativePath`** — нестандартное свойство `File`. Читаем через `js_sys::Reflect::get(file.as_ref(), "webkitRelativePath")`, потому что web-sys не экспортирует соответствующий метод.
- **Drop-зона для папок:** итерируем `DataTransferItemList` через `items()`, для каждого вызываем `webkitGetAsEntry()`.
- **Рекурсивный обход** — в `spawn_local`, потому что `scan_entry` возвращает future.
- **`scan_entry(entry, String::new())`** — передаём владение, не ссылки; `String::new()` вместо `""` — этого требует сигнатура.

---

### `tests/wasm_folder_drop.rs`

Полный рекурсивный обход требует настоящей `FileSystemEntry`, которую невозможно создать без пользовательского действия. Тесты покрывают структуру данных и компиляцию сигнатуры.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::folder_drop::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_folder_entry_construction() {
    let entry = FolderEntry {
        path: "photos/2024/img001.png".to_string(),
        size: 12345.0,
        mime_type: "image/png".to_string(),
    };
    assert_eq!(entry.path, "photos/2024/img001.png");
    assert_eq!(entry.size, 12345.0);
    assert_eq!(entry.mime_type, "image/png");
}

#[wasm_bindgen_test]
fn test_folder_entry_equality() {
    let a = FolderEntry {
        path: "a.txt".to_string(),
        size: 100.0,
        mime_type: "text/plain".to_string(),
    };
    let b = FolderEntry {
        path: "a.txt".to_string(),
        size: 100.0,
        mime_type: "text/plain".to_string(),
    };
    let c = FolderEntry {
        path: "b.txt".to_string(),
        size: 100.0,
        mime_type: "text/plain".to_string(),
    };
    assert_eq!(a, b);
    assert_ne!(a, c);
}

#[wasm_bindgen_test]
fn test_folder_entry_clone() {
    let original = FolderEntry {
        path: "dir/file.bin".to_string(),
        size: 4096.0,
        mime_type: String::new(),
    };
    let cloned = original.clone();
    assert_eq!(original, cloned);
}

#[wasm_bindgen_test]
fn test_scan_entry_signature_compiles() {
    // Фиксируем сигнатуру scan_entry: принимает владение, возвращает
    // Pin<Box<dyn Future<Output = Vec<FolderEntry>>>>. Сами данные
    // требуют настоящего FileSystemEntry и проверяются вручную.
    let _fn: fn(
        web_sys::FileSystemEntry,
        String,
    ) -> std::pin::Pin<
        Box<dyn std::future::Future<Output = Vec<FolderEntry>>>,
    > = scan_entry;
}

#[wasm_bindgen_test]
fn test_entry_from_item_signature_compiles() {
    // Сигнатура entry_from_item: &DataTransferItem → Option<FileSystemEntry>.
    let _fn: fn(&web_sys::DataTransferItem) -> Option<web_sys::FileSystemEntry> =
        entry_from_item;
}
```

**Проверка импортов:**

- `wasm_bindgen_test::*` — `wasm_bindgen_test` и `wasm_bindgen_test_configure`.
- `hello_wasm02::folder_drop::*` — `FolderEntry`, `scan_entry`, `entry_from_item`.

Полные пути `web_sys::FileSystemEntry`, `web_sys::DataTransferItem`, `std::pin::Pin`, `std::future::Future` — без импортов. Неиспользуемых нет.

**Что проверяем:**

- **Конструкция `FolderEntry`** — поля заполняются корректно.
- **Равенство и клонирование** — структура ведёт себя как обычный Rust-тип.
- **Компиляция сигнатур** — фиксируем, что `scan_entry` принимает владение и возвращает `Pin<Box<dyn Future>>`, а `entry_from_item` работает с `&DataTransferItem`. Это защищает от случайного изменения сигнатуры в будущем.

**Что не проверяем и почему:** рекурсивный обход требует настоящей `FileSystemEntry`, которую невозможно создать без пользовательского действия. Это ограничение безопасности браузеров. Интеграционная проверка делается вручную через демо-секцию.

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

### Тесты подглавы 18.7

```bash
# Проверка типов тестов
cargo check --target wasm32-unknown-unknown --tests

# Только тесты этой подглавы
wasm-pack test --headless --chrome --test wasm_folder_drop

# То же самое для Firefox
wasm-pack test --headless --firefox --test wasm_folder_drop

# Все тесты проекта
npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.7. Drag & Drop папок** с полем выбора папки и drop-зоной.

**Сценарий 1 — выбор папки через `<input webkitdirectory>`:**
1. Нажмите на поле выбора.
2. Выберите папку (например, `photos`).
3. В блоке результата появится:

```
Выбрано файлов в папке: 3

1. photos/2024/img001.png
   Размер: 234.56 KB
   MIME: image/png

2. photos/2024/img002.png
   Размер: 345.67 KB
   MIME: image/png

3. photos/readme.txt
   Размер: 1.23 KB
   MIME: text/plain
```

**Сценарий 2 — перетаскивание папки:**
1. Откройте проводник рядом с браузером.
2. Перетащите папку `photos` в drop-зону.
3. Пока курсор над зоной — рамка зелёная.
4. После отпускания — результат тот же, что в сценарии 1, но с полным рекурсивным обходом.

**Сценарий 3 — перетаскивание одного файла (не папки):**
```
Найдено файлов: 1
Общий размер: 234.56 KB

1. document.pdf
   Размер: 234.56 KB
   MIME: application/pdf
```

**Сценарий 4 — перетаскивание текста:**
```
Нет папок или файлов для обработки.
```

---

## Итоги подглавы 18.7

- **`webkitGetAsEntry()`** — единственный способ получить доступ к перетащенной папке. Возвращает `FileSystemEntry` — либо файл, либо директорию.
- **`FileSystemDirectoryEntry::createReader()`** — создаёт reader для чтения содержимого директории.
- **`readEntries()`** — читает порциями, а не всё сразу. Нужно вызывать в цикле до пустого массива.
- **`FileSystemFileEntry::file()`** — асинхронно возвращает `File` для конкретного файла.
- **`<input webkitdirectory>`** — второй способ выбрать папку; браузер сам обходит дерево и отдаёт `FileList` с `webkitRelativePath`.
- **Рекурсивный обход** — реализуется через `Box::pin` в обычной функции, возвращающей `Pin<Box<dyn Future<...>>>`.
- **`webkitRelativePath`** — нестандартное свойство; читаем через `js_sys::Reflect::get`.

Модуль `folder_drop.rs` автономен, все tail-функции оформлены через `return`, неиспользуемых импортов нет. Демо-секция самодостаточна и демонстрирует два способа выбора папки.

## Замечания на будущее

Четыре правила, зафиксированные в этой подглаве:

1. **Рекурсивная `async fn` требует `Box::pin`.** Оформляйте её как обычную функцию, возвращающую `Pin<Box<dyn Future<...>>>` с параметрами по владению. Это даёт `'static`-future и устраняет проблемы с временами жизни.

2. **`readEntries()` возвращает не все записи сразу.** Вызывайте в цикле до пустого массива — иначе потеряете часть файлов в больших папках.

3. **`DataTransfer::items()` требует фичи `DataTransferItemList`** в `Cargo.toml`. Без неё метода просто нет в web-sys.

4. **Для нестандартных JS-свойств (типа `webkitRelativePath`) используйте `js_sys::Reflect::get`.** Универсальный рецепт:
   ```rust
   let value = js_sys::Reflect::get(obj.as_ref(), &JsValue::from_str("propertyName"));
   if let Ok(v) = value {
       if let Some(s) = v.as_string() {
           // использовать s
       }
   }
   ```
   Работает для любого JS-объекта и любого свойства — не нужно ждать, пока web-sys добавит метод.

В следующей подглаве (18.8) мы займёмся **валидацией файлов** — научимся определять настоящий формат по сигнатуре, а не по расширению, и защитим конвертер от подделок.