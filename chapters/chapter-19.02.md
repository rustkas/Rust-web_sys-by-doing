# Глава 19.2. Три picker'а: open / save / directory

## Что раскрывает глава

Подглава 18.15 показала три сценария FSA — «открыть, сохранить, обойти папку» — как отдельные обёртки. Здесь те же методы раскрываются **на уровень выше**, с акцентом на то, что не видно в одиночных вызовах:

- **Модель возможностей браузера.** Разные браузеры поддерживают разный набор: Chrome/Edge — все три, Firefox/Safari — ни одного, редкие конфигурации — частично. Демонстрация начинается с self-test, и интерфейс показывается только при полной поддержке.
- **Опции picker'ов как предметная модель.** `AcceptFilter`, `FilterPreset`, `DirectoryMode` с валидацией вместо «собери `JsValue` руками».
- **Классификация ошибок.** `PickOutcome<T>` с четырьмя осмысленными состояниями: `Picked` / `Cancelled` / `Blocked` / `Failed`. UI перестаёт различать «отмена» (не ошибка) и «нужен жест» (подсказка) от реального сбоя.
- **Три независимых workflow в одном демо.** Каждый picker — свой UI-блок со своими контролами, статусом, результатом, метриками и кнопкой «Сбросить».

## Структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── lib.rs                       # +3 pub mod fsa_choose_*
│   ├── app.rs                       # демо-секция 19.2 (три блока)
│   ├── fsa_choose_options.rs        # опции picker'ов + валидация + пресеты
│   ├── fsa_choose_exec.rs           # вызов + классификация ошибок + Capabilities
│   └── fsa_choose_fallback.rs       # <input type=file> + <a download>
└── tests/
    └── wasm_fsa_choose.rs           # тесты 19.2
```

**Краткий комментарий.** В проекте уже есть `fsa_api_pickers.rs` — прямые обёртки picker'ов с одним `types` из кортежей. Эта подглава идёт **на уровень выше**: модель возможностей, валидация фильтров, классификация ошибок, fallback. Префикс `fsa_choose_*` — уникален; с `fsa_api_pickers`, `fsa_open`, `fsa_save`, `fsa_directory` не пересекается.

## Ссылки

- [MDN: showOpenFilePicker](https://developer.mozilla.org/en-US/docs/Web/API/Window/showOpenFilePicker)
- [MDN: showSaveFilePicker](https://developer.mozilla.org/en-US/docs/Web/API/Window/showSaveFilePicker)
- [MDN: showDirectoryPicker](https://developer.mozilla.org/en-US/docs/Web/API/Window/showDirectoryPicker)
- [MDN: `<input type="file">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/file)
- [MDN: `<a download>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#download)
- [Chrome: File System Access API](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access)

---

## Исходные модули

### `src/fsa_choose_options.rs`

```rust
//! Опции picker'ов в терминах предметной области.
//!
//! Модуль не вызывает picker'ы (это делает `fsa_choose_exec`), а только
//! **описывает**, что мы хотим выбрать.
//!
//! * `AcceptFilter` — один фильтр: описание + карта MIME → расширения.
//!   Проходит валидацию: MIME содержит `/`, расширения начинаются с `.`.
//! * `FilterPreset` — готовые конфигурации (Images, Text, None).
//! * `build_*_options` — собирают JS-объекты с проверкой фильтров.

use wasm_bindgen::prelude::*;

/// Обёртка над `format!`, фиксирующая тип для rust-analyzer.
/// Хвостовое `format!` в `if/else`/`match` RA выводит как `()`.
#[inline]
pub fn format_str(args: std::fmt::Arguments<'_>) -> String {
    std::fmt::format(args)
}

// ---------------------------------------------------------------------------
// AcceptFilter
// ---------------------------------------------------------------------------

/// Один фильтр: описание + карта MIME → расширения.
#[derive(Debug, Clone, PartialEq)]
pub struct AcceptFilter {
    pub description: String,
    pub mime_to_extensions: Vec<(String, Vec<String>)>,
}

impl AcceptFilter {
    pub fn new(description: &str) -> Self {
        AcceptFilter {
            description: description.to_string(),
            mime_to_extensions: Vec::new(),
        }
    }

    pub fn add(mut self, mime: &str, extensions: &[&str]) -> Self {
        let exts: Vec<String> =
            extensions.iter().map(|s| (*s).to_string()).collect();
        self.mime_to_extensions.push((mime.to_string(), exts));
        self
    }

    /// Валидация: MIME содержит `/`, расширения начинаются с `.`,
    /// у фильтра есть хотя бы одно расширение.
    pub fn validate(&self) -> Result<(), String> {
        if self.description.trim().is_empty() {
            return Err("описание фильтра пусто".to_string());
        }
        if self.mime_to_extensions.is_empty() {
            return Err(format_str(format_args!(
                "фильтр '{}' не содержит расширений",
                self.description,
            )));
        }
        for (mime, exts) in &self.mime_to_extensions {
            if !mime.contains('/') {
                return Err(format_str(format_args!(
                    "MIME '{mime}' должен содержать '/'",
                )));
            }
            for e in exts {
                if !e.starts_with('.') {
                    return Err(format_str(format_args!(
                        "расширение '{e}' должно начинаться с точки",
                    )));
                }
            }
        }
        Ok(())
    }

    fn build_accept_object(&self) -> Result<JsValue, JsValue> {
        let accept = js_sys::Object::new();
        for (mime, exts) in &self.mime_to_extensions {
            let arr = js_sys::Array::new();
            for e in exts {
                arr.push(&JsValue::from_str(e));
            }
            js_sys::Reflect::set(&accept, &JsValue::from_str(mime), &arr)?;
        }
        Ok(accept.into())
    }
}

// ---------------------------------------------------------------------------
// FilterPreset — готовые конфигурации
// ---------------------------------------------------------------------------

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FilterPreset {
    Images,
    Text,
    None,
}

impl FilterPreset {
    pub fn as_str(self) -> &'static str {
        match self {
            FilterPreset::Images => "Изображения (PNG/JPEG)",
            FilterPreset::Text => "Текст (.txt)",
            FilterPreset::None => "Без фильтра",
        }
    }

    pub fn build(self) -> Vec<AcceptFilter> {
        match self {
            FilterPreset::Images => vec![AcceptFilter::images()],
            FilterPreset::Text => vec![AcceptFilter::plain_text()],
            FilterPreset::None => Vec::new(),
        }
    }

    /// Все пресеты в фиксированном порядке — для построения `<select>`.
    pub fn all() -> [FilterPreset; 3] {
        [FilterPreset::Images, FilterPreset::Text, FilterPreset::None]
    }
}

impl AcceptFilter {
    /// Типовой фильтр «изображения»: PNG + JPEG.
    pub fn images() -> Self {
        AcceptFilter::new("Images")
            .add("image/png", &[".png"])
            .add("image/jpeg", &[".jpg", ".jpeg"])
    }

    /// Типовой фильтр «текст»: .txt.
    pub fn plain_text() -> Self {
        AcceptFilter::new("Text").add("text/plain", &[".txt"])
    }
}

// ---------------------------------------------------------------------------
// Сборка JS-объектов
// ---------------------------------------------------------------------------

fn validate_filters(filters: &[AcceptFilter]) -> Result<(), String> {
    for f in filters {
        f.validate()?;
    }
    Ok(())
}

/// Построить `types` — массив `[{description, accept}, ...]`.
pub fn build_accept_array(filters: &[AcceptFilter]) -> Result<JsValue, JsValue> {
    validate_filters(filters).map_err(|e| JsValue::from_str(&e))?;

    let types = js_sys::Array::new();
    for f in filters {
        let entry = js_sys::Object::new();
        js_sys::Reflect::set(
            &entry,
            &JsValue::from_str("description"),
            &JsValue::from_str(&f.description),
        )?;
        js_sys::Reflect::set(
            &entry,
            &JsValue::from_str("accept"),
            &f.build_accept_object()?,
        )?;
        types.push(&entry);
    }
    Ok(types.into())
}

/// Опции для `showOpenFilePicker`.
pub fn build_open_options(
    multiple: bool,
    exclude_accept_all: bool,
    filters: &[AcceptFilter],
) -> Result<JsValue, JsValue> {
    validate_filters(filters).map_err(|e| JsValue::from_str(&e))?;

    let opts = js_sys::Object::new();
    if !filters.is_empty() {
        js_sys::Reflect::set(
            &opts,
            &JsValue::from_str("types"),
            &build_accept_array(filters)?,
        )?;
        js_sys::Reflect::set(
            &opts,
            &JsValue::from_str("excludeAcceptAllOption"),
            &JsValue::from_bool(exclude_accept_all),
        )?;
    }
    js_sys::Reflect::set(
        &opts,
        &JsValue::from_str("multiple"),
        &JsValue::from_bool(multiple),
    )?;
    Ok(opts.into())
}

/// Опции для `showSaveFilePicker`.
pub fn build_save_options(
    suggested_name: &str,
    filters: &[AcceptFilter],
) -> Result<JsValue, JsValue> {
    validate_filters(filters).map_err(|e| JsValue::from_str(&e))?;

    let opts = js_sys::Object::new();
    js_sys::Reflect::set(
        &opts,
        &JsValue::from_str("suggestedName"),
        &JsValue::from_str(suggested_name),
    )?;
    if !filters.is_empty() {
        js_sys::Reflect::set(
            &opts,
            &JsValue::from_str("types"),
            &build_accept_array(filters)?,
        )?;
    }
    Ok(opts.into())
}

/// Режим директории: только чтение или чтение-запись.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DirectoryMode {
    Read,
    ReadWrite,
}

impl DirectoryMode {
    pub fn as_str(self) -> &'static str {
        match self {
            DirectoryMode::Read => "read",
            DirectoryMode::ReadWrite => "readwrite",
        }
    }
}

/// Опции для `showDirectoryPicker`.
///
/// `id` — идентификатор, по которому браузер запоминает последнюю
/// выбранную папку для этого приложения. `start_in` — начальная
/// подсказка (`"documents"`, `"downloads"`, `"desktop"`, ...).
pub fn build_directory_options(
    mode: DirectoryMode,
    id: Option<&str>,
    start_in: Option<&str>,
) -> Result<JsValue, JsValue> {
    let opts = js_sys::Object::new();
    js_sys::Reflect::set(
        &opts,
        &JsValue::from_str("mode"),
        &JsValue::from_str(mode.as_str()),
    )?;
    if let Some(v) = id {
        js_sys::Reflect::set(&opts, &JsValue::from_str("id"), &JsValue::from_str(v))?;
    }
    if let Some(v) = start_in {
        js_sys::Reflect::set(
            &opts,
            &JsValue::from_str("startIn"),
            &JsValue::from_str(v),
        )?;
    }
    Ok(opts.into())
}

// ---------------------------------------------------------------------------
// Описания для UI
// ---------------------------------------------------------------------------

/// Строка-описание фильтров: показывает расширения и подсвечивает
/// проблемные.
pub fn describe_filters(filters: &[AcceptFilter]) -> String {
    if filters.is_empty() {
        return "фильтры не заданы (можно выбрать любой файл)".to_string();
    }
    let mut parts: Vec<String> = Vec::with_capacity(filters.len());
    for f in filters {
        let exts: Vec<String> = f
            .mime_to_extensions
            .iter()
            .flat_map(|(_, e)| e.iter().cloned())
            .collect();
        let warning: String = match f.validate() {
            Ok(()) => String::new(),
            Err(msg) => format_str(format_args!(" ⚠ {msg}")),
        };
        parts.push(format_str(format_args!(
            "{}: {} [{}]{}",
            f.description,
            f.mime_to_extensions
                .iter()
                .map(|(m, _)| m.clone())
                .collect::<Vec<_>>()
                .join(", "),
            exts.join(" "),
            warning,
        )));
    }
    parts.join("; ")
}

/// Строка «id директории» для UI.
pub fn describe_directory_id(id: &str) -> String {
    format_str(format_args!(
        "id = \"{id}\" (браузер запоминает последнюю папку по этому ключу)",
    ))
}
```

### `src/fsa_choose_exec.rs`

```rust
//! Исполнение picker'ов с осмысленной классификацией ошибок.
//!
//! Возвращает `PickOutcome<T>`:
//! * `Picked(T)` — пользователь выбрал;
//! * `Cancelled` — диалог закрыт (**не ошибка!**);
//! * `Blocked(reason)` — браузер требует user gesture или FSA недоступен;
//! * `Failed(reason)` — реальная ошибка.
//!
//! `Capabilities::detect()` — модель доступных методов. Разные браузеры
//! могут иметь разный набор: только `showOpenFilePicker`, все три, ни одного.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle};

use crate::fsa_choose_options::format_str;

// ---------------------------------------------------------------------------
// Capabilities
// ---------------------------------------------------------------------------

/// Что из FSA доступно в этом браузере.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Capabilities {
    pub open: bool,
    pub save: bool,
    pub directory: bool,
}

impl Capabilities {
    pub fn detect() -> Self {
        Capabilities {
            open: has_method("showOpenFilePicker"),
            save: has_method("showSaveFilePicker"),
            directory: has_method("showDirectoryPicker"),
        }
    }

    pub fn any(&self) -> bool {
        return self.open || self.save || self.directory;
    }

    pub fn all(&self) -> bool {
        return self.open && self.save && self.directory;
    }

    /// Короткая строка для UI.
    pub fn summary(&self) -> String {
        return format_str(format_args!(
            "open {} | save {} | directory {}",
            mark(self.open),
            mark(self.save),
            mark(self.directory),
        ));
    }
}

/// Отдельная функция с явной сигнатурой — rust-analyzer не теряет тип.
fn mark(b: bool) -> &'static str {
    if b {
        return "✓";
    }
    return "—";
}

fn has_method(name: &str) -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let w: &JsValue = window.as_ref();
    let result = js_sys::Reflect::get(w, &JsValue::from_str(name))
        .map(|v| !v.is_undefined())
        .unwrap_or(false);
    return result;
}

/// Совместимость: раньше проверяли только `showOpenFilePicker`.
pub fn is_fsa_available() -> bool {
    return has_method("showOpenFilePicker");
}

// ---------------------------------------------------------------------------
// PickOutcome
// ---------------------------------------------------------------------------

/// Результат вызова picker'а.
#[derive(Debug)]
pub enum PickOutcome<T> {
    Picked(T),
    Cancelled,
    Blocked(String),
    Failed(String),
}

impl<T> PickOutcome<T> {
    pub fn is_picked(&self) -> bool {
        return matches!(self, PickOutcome::Picked(_));
    }

    pub fn is_cancelled(&self) -> bool {
        return matches!(self, PickOutcome::Cancelled);
    }

    /// Короткая метка для UI.
    pub fn kind_label(&self) -> &'static str {
        match self {
            PickOutcome::Picked(_) => return "OK",
            PickOutcome::Cancelled => return "отменено",
            PickOutcome::Blocked(_) => return "заблокировано",
            PickOutcome::Failed(_) => return "ошибка",
        }
    }
}

// ---------------------------------------------------------------------------
// Ошибки и классификация
// ---------------------------------------------------------------------------

enum CallError {
    Cancelled,
    Blocked(String),
    Failed(String),
}

impl<T> From<CallError> for PickOutcome<T> {
    fn from(e: CallError) -> Self {
        match e {
            CallError::Cancelled => return PickOutcome::Cancelled,
            CallError::Blocked(m) => return PickOutcome::Blocked(m),
            CallError::Failed(m) => return PickOutcome::Failed(m),
        }
    }
}

/// Подсказка «что делать» к сообщению `Blocked`.
fn blocked_retry_hint() -> String {
    return format_str(format_args!(
        "Браузер требует жест пользователя.\n\
         Нажмите кнопку ещё раз — диалог должен открыться.",
    ));
}

/// Классифицирует raw ошибку JS в осмысленное состояние.
fn classify_error(e: &JsValue) -> CallError {
    let name: String = js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    if name == "AbortError" {
        return CallError::Cancelled;
    }
    if name == "SecurityError" || name == "NotAllowedError" {
        return CallError::Blocked(blocked_retry_hint());
    }
    let raw: String = e
        .as_string()
        .unwrap_or_else(|| format_str(format_args!("{e:?}")));
    return CallError::Failed(format_str(format_args!("Ошибка picker'а: {raw}")));
}

// ---------------------------------------------------------------------------
// Вызовы
// ---------------------------------------------------------------------------

async fn call_picker(method: &str, options: JsValue) -> Result<JsValue, CallError> {
    let window = web_sys::window()
        .ok_or_else(|| CallError::Failed(format_str(format_args!("no window"))))?;
    let w: &JsValue = window.as_ref();

    let fn_js = js_sys::Reflect::get(w, &JsValue::from_str(method))
        .map_err(|e| CallError::Failed(format_str(format_args!("{e:?}"))))?;
    if fn_js.is_undefined() {
        return Err(CallError::Blocked(format_str(format_args!(
            "{method} недоступен — FSA не поддерживается",
        ))));
    }
    let func: js_sys::Function = fn_js.dyn_into().map_err(|_| {
        CallError::Failed(format_str(format_args!("{method} не функция")))
    })?;

    let promise_js = func.call1(w, &options).map_err(|e| classify_error(&e))?;
    let promise: js_sys::Promise = promise_js.dyn_into().map_err(|_| {
        CallError::Failed(format_str(format_args!("{method} не Promise")))
    })?;

    return JsFuture::from(promise).await.map_err(|e| classify_error(&e));
}

pub async fn open_files(options: JsValue) -> PickOutcome<Vec<FileSystemFileHandle>> {
    let value = match call_picker("showOpenFilePicker", options).await {
        Ok(v) => v,
        Err(e) => return e.into(),
    };
    let array = match value.dyn_into::<js_sys::Array>() {
        Ok(a) => a,
        Err(_) => {
            return PickOutcome::Failed(format_str(format_args!(
                "showOpenFilePicker вернул не Array",
            )));
        }
    };
    let mut handles: Vec<FileSystemFileHandle> =
        Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let h: FileSystemFileHandle = match array.get(i).dyn_into() {
            Ok(h) => h,
            Err(_) => {
                return PickOutcome::Failed(format_str(format_args!(
                    "элемент {i} не FileSystemFileHandle",
                )));
            }
        };
        handles.push(h);
    }
    return PickOutcome::Picked(handles);
}

pub async fn save_file(options: JsValue) -> PickOutcome<FileSystemFileHandle> {
    let value = match call_picker("showSaveFilePicker", options).await {
        Ok(v) => v,
        Err(e) => return e.into(),
    };
    match value.dyn_into::<FileSystemFileHandle>() {
        Ok(h) => return PickOutcome::Picked(h),
        Err(_) => {
            return PickOutcome::Failed(format_str(format_args!(
                "showSaveFilePicker вернул не FileSystemFileHandle",
            )));
        }
    }
}

pub async fn pick_directory(
    options: JsValue,
) -> PickOutcome<FileSystemDirectoryHandle> {
    let value = match call_picker("showDirectoryPicker", options).await {
        Ok(v) => v,
        Err(e) => return e.into(),
    };
    match value.dyn_into::<FileSystemDirectoryHandle>() {
        Ok(h) => return PickOutcome::Picked(h),
        Err(_) => {
            return PickOutcome::Failed(format_str(format_args!(
                "showDirectoryPicker вернул не FileSystemDirectoryHandle",
            )));
        }
    }
}

/// Проверка «директорию можно выбирать?» — единая точка.
pub fn ensure_directory_available() -> Option<String> {
    if has_method("showDirectoryPicker") {
        return None;
    }
    return Some(crate::fsa_choose_fallback::directory_unsupported_message());
}
```

### `src/fsa_choose_fallback.rs`

```rust
//! Fallback для браузеров без FSA (Firefox, Safari).
//!
//! * открытие — `<input type="file">`;
//! * сохранение — `<a download>` + Object URL.
//!
//! Для директории fallback'а нет: сообщаем об этом явно.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, File, HtmlAnchorElement, HtmlInputElement, Url};

use crate::fsa_choose_exec::PickOutcome;
use crate::fsa_choose_options::format_str;

/// Информация о скачанном файле — что реально ушло в загрузки.
#[derive(Debug, Clone, PartialEq)]
pub struct DownloadInfo {
    pub suggested_name: String,
    pub blob_size: f64,
}

/// Выбор файла через `<input type="file">`.
pub async fn pick_via_input(accept: &str) -> PickOutcome<File> {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return PickOutcome::Failed(format_str(format_args!("no window"))),
    };
    let doc = match window.document() {
        Some(d) => d,
        None => return PickOutcome::Failed(format_str(format_args!("no document"))),
    };

    let input_el = match doc.create_element("input") {
        Ok(e) => e,
        Err(_) => {
            return PickOutcome::Failed(format_str(format_args!(
                "не удалось создать <input>",
            )));
        }
    };
    let input: HtmlInputElement = match input_el.dyn_into() {
        Ok(i) => i,
        Err(_) => {
            return PickOutcome::Failed(format_str(format_args!(
                "не HtmlInputElement",
            )));
        }
    };
    input.set_type("file");
    if !accept.is_empty() {
        input.set_accept(accept);
    }
    if let Some(body) = doc.body() {
        let _ = body.append_child(&input);
    }
    input.click();

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let input_clone = input.clone();
        let on_change = Closure::once(move || {
            let files = input_clone.files();
            match files.and_then(|f| f.get(0)) {
                Some(file) => {
                    let _ = resolve.call1(&JsValue::NULL, &file);
                }
                None => {
                    let err = js_sys::Error::new("no file selected");
                    let _ = js_sys::Reflect::set(
                        &err,
                        &JsValue::from_str("name"),
                        &JsValue::from_str("AbortError"),
                    );
                    let err_js: JsValue = err.into();
                    let _ = reject.call1(&JsValue::NULL, &err_js);
                }
            }
        });
        input.set_onchange(Some(on_change.as_ref().unchecked_ref()));
        on_change.forget();
    });

    let result = JsFuture::from(promise).await;
    if let Some(body) = doc.body() {
        let _ = body.remove_child(&input);
    }

    return match result {
        Ok(v) => match v.dyn_into::<File>() {
            Ok(f) => PickOutcome::Picked(f),
            Err(_) => PickOutcome::Failed(format_str(format_args!("не File"))),
        },
        Err(_) => PickOutcome::Cancelled,
    };
}

/// Скачать Blob через `<a download>` + Object URL.
pub fn download_blob(
    blob: &Blob,
    suggested_name: &str,
) -> Result<DownloadInfo, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let doc = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let url = Url::create_object_url_with_blob(blob)?;
    let a: HtmlAnchorElement = doc
        .create_element("a")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlAnchorElement"))?;
    a.set_href(&url);
    a.set_download(suggested_name);
    a.click();
    Url::revoke_object_url(&url)?;

    let info = DownloadInfo {
        suggested_name: suggested_name.to_string(),
        blob_size: blob.size(),
    };
    return Ok(info);
}

/// Упаковать текст в Blob `text/plain;charset=utf-8`.
pub fn make_text_blob(text: &str) -> Result<Blob, JsValue> {
    let arr = js_sys::Array::of1(&JsValue::from_str(text));
    let opts = web_sys::BlobPropertyBag::new();
    opts.set_type("text/plain;charset=utf-8");
    let blob = Blob::new_with_str_sequence_and_options(&arr, &opts)?;
    return Ok(blob);
}

/// Строка описания скачанного файла для UI.
pub fn describe_download(info: &DownloadInfo) -> String {
    return format_str(format_args!(
        "fallback · скачано как {} ({} байт)\nПроверьте папку «Загрузки».",
        info.suggested_name, info.blob_size,
    ));
}

/// Сообщение о том, что fallback для директорий не существует.
pub fn directory_unsupported_message() -> String {
    return format_str(format_args!(
        "Выбор папки в этом браузере недоступен.\n\
         File System Access API требует Chrome или Edge.\n\
         Открыть отдельный файл — можно, через <input type=file>.",
    ));
}
```

---

## Как добавить в `lib.rs`

В блок `pub mod ...` добавить три строки:

```rust
pub mod fsa_choose_options;
pub mod fsa_choose_exec;
pub mod fsa_choose_fallback;
```

---

## Тесты

### `tests/wasm_fsa_choose.rs`

```rust
//! Тесты `fsa_choose_*` — построение опций, валидация, классификация ошибок.
//!
//! Правило проекта: макрос или выражение в хвостовой позиции
//! функции/`if`/`match`-arm оборачивается в явный `return`.

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

use hello_wasm02::fsa_choose_exec::{self, Capabilities, PickOutcome};
use hello_wasm02::fsa_choose_fallback;
use hello_wasm02::fsa_choose_options::{
    build_accept_array, build_directory_options, build_open_options,
    build_save_options, describe_directory_id, describe_filters, AcceptFilter,
    DirectoryMode, FilterPreset,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// AcceptFilter — валидация
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn accept_filter_valid_passes() {
    let f = AcceptFilter::images();
    assert!(f.validate().is_ok());
}

#[wasm_bindgen_test]
fn accept_filter_missing_dot_fails() {
    let f = AcceptFilter::new("Bad").add("image/png", &["png"]);
    assert!(f.validate().is_err());
}

#[wasm_bindgen_test]
fn accept_filter_bad_mime_fails() {
    let f = AcceptFilter::new("Bad").add("not-mime", &[".png"]);
    assert!(f.validate().is_err());
}

#[wasm_bindgen_test]
fn accept_filter_empty_fails() {
    let f = AcceptFilter::new("Empty");
    assert!(f.validate().is_err());
}

#[wasm_bindgen_test]
fn accept_filter_empty_description_fails() {
    let f = AcceptFilter::new("").add("image/png", &[".png"]);
    assert!(f.validate().is_err());
}

#[wasm_bindgen_test]
fn build_open_options_with_invalid_filter_returns_err() {
    let f = AcceptFilter::new("Bad").add("image/png", &["png"]);
    let r = build_open_options(false, false, &[f]);
    assert!(r.is_err());
}

#[wasm_bindgen_test]
fn accept_filter_images_has_png_and_jpeg() {
    let f = AcceptFilter::images();
    let mimetypes: Vec<&str> = f
        .mime_to_extensions
        .iter()
        .map(|(m, _)| m.as_str())
        .collect();
    assert!(mimetypes.contains(&"image/png"));
    assert!(mimetypes.contains(&"image/jpeg"));
}

#[wasm_bindgen_test]
fn accept_filter_plain_text_has_txt() {
    let f = AcceptFilter::plain_text();
    let exts: Vec<&str> = f
        .mime_to_extensions
        .iter()
        .flat_map(|(_, e)| e.iter().map(|s| s.as_str()))
        .collect();
    assert!(exts.contains(&".txt"));
}

// ---------------------------------------------------------------------------
// FilterPreset
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn filter_preset_images_builds_one_filter() {
    let p = FilterPreset::Images;
    let f = p.build();
    assert_eq!(f.len(), 1);
    assert!(f[0].validate().is_ok());
}

#[wasm_bindgen_test]
fn filter_preset_text_builds_one_filter() {
    let p = FilterPreset::Text;
    let f = p.build();
    assert_eq!(f.len(), 1);
    assert!(f[0].validate().is_ok());
}

#[wasm_bindgen_test]
fn filter_preset_none_is_empty() {
    let p = FilterPreset::None;
    let f = p.build();
    assert!(f.is_empty());
}

#[wasm_bindgen_test]
fn filter_preset_all_has_three() {
    assert_eq!(FilterPreset::all().len(), 3);
}

#[wasm_bindgen_test]
fn filter_preset_as_str_nonempty() {
    for p in FilterPreset::all() {
        assert!(!p.as_str().is_empty());
    }
}

// ---------------------------------------------------------------------------
// Capabilities
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn capabilities_detect_returns_struct() {
    let caps = Capabilities::detect();
    let _ = caps.any();
    let _ = caps.all();
    let s = caps.summary();
    assert!(s.contains("open"));
    assert!(s.contains("save"));
    assert!(s.contains("directory"));
}

#[wasm_bindgen_test]
fn capabilities_summary_marks_available() {
    let caps = Capabilities {
        open: true,
        save: false,
        directory: true,
    };
    let s = caps.summary();
    assert!(s.contains("open ✓"));
    assert!(s.contains("save —"));
    assert!(s.contains("directory ✓"));
}

#[wasm_bindgen_test]
fn capabilities_any_all_none() {
    let none = Capabilities {
        open: false,
        save: false,
        directory: false,
    };
    assert!(!none.any());
    assert!(!none.all());
}

#[wasm_bindgen_test]
fn capabilities_any_all_full() {
    let all = Capabilities {
        open: true,
        save: true,
        directory: true,
    };
    assert!(all.any());
    assert!(all.all());
}

// ---------------------------------------------------------------------------
// Build options
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn build_accept_filter_array_length_one() {
    let f = AcceptFilter::images();
    let arr = build_accept_array(&[f]).unwrap();
    let arr = js_sys::Array::from(&arr);
    assert_eq!(arr.length(), 1);
}

#[wasm_bindgen_test]
fn build_open_options_multiple_and_exclude() {
    let f = AcceptFilter::plain_text();
    let opts = build_open_options(true, true, &[f]).unwrap();
    let multiple = js_sys::Reflect::get(&opts, &"multiple".into()).unwrap();
    assert_eq!(multiple.as_bool(), Some(true));
    let excl =
        js_sys::Reflect::get(&opts, &"excludeAcceptAllOption".into()).unwrap();
    assert_eq!(excl.as_bool(), Some(true));
}

#[wasm_bindgen_test]
fn build_open_options_without_filters_omits_types() {
    let opts = build_open_options(false, false, &[]).unwrap();
    let types = js_sys::Reflect::get(&opts, &"types".into()).unwrap();
    assert!(types.is_undefined());
}

#[wasm_bindgen_test]
fn build_save_options_has_suggested_name() {
    let opts = build_save_options("out.txt", &[]).unwrap();
    let name = js_sys::Reflect::get(&opts, &"suggestedName".into()).unwrap();
    assert_eq!(name.as_string().as_deref(), Some("out.txt"));
}

#[wasm_bindgen_test]
fn build_directory_options_readwrite_mode() {
    let opts =
        build_directory_options(DirectoryMode::ReadWrite, None, None).unwrap();
    let mode = js_sys::Reflect::get(&opts, &"mode".into()).unwrap();
    assert_eq!(mode.as_string().as_deref(), Some("readwrite"));
}

#[wasm_bindgen_test]
fn build_directory_options_read_mode_default() {
    let opts = build_directory_options(DirectoryMode::Read, None, None).unwrap();
    let mode = js_sys::Reflect::get(&opts, &"mode".into()).unwrap();
    assert_eq!(mode.as_string().as_deref(), Some("read"));
}

#[wasm_bindgen_test]
fn build_directory_options_with_id_and_start_in() {
    let opts = build_directory_options(
        DirectoryMode::Read,
        Some("myapp"),
        Some("documents"),
    )
    .unwrap();
    let id = js_sys::Reflect::get(&opts, &"id".into()).unwrap();
    assert_eq!(id.as_string().as_deref(), Some("myapp"));
    let start = js_sys::Reflect::get(&opts, &"startIn".into()).unwrap();
    assert_eq!(start.as_string().as_deref(), Some("documents"));
}

// ---------------------------------------------------------------------------
// Описания
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn describe_filters_empty_mentions_no_filters() {
    let s = describe_filters(&[]);
    assert!(s.contains("фильтры не заданы"));
}

#[wasm_bindgen_test]
fn describe_filters_images_shows_extensions() {
    let s = describe_filters(&[AcceptFilter::images()]);
    assert!(s.contains("Images"));
    assert!(s.contains(".png"));
    assert!(s.contains(".jpg"));
}

#[wasm_bindgen_test]
fn describe_filters_warns_about_invalid() {
    let bad = AcceptFilter::new("Bad").add("image/png", &["png"]);
    let s = describe_filters(&[bad]);
    assert!(s.contains("⚠"));
}

#[wasm_bindgen_test]
fn describe_directory_id_contains_id() {
    let s = describe_directory_id("fsa-choose-demo");
    assert!(s.contains("fsa-choose-demo"));
    assert!(s.contains("id"));
}

// ---------------------------------------------------------------------------
// Exec — feature detection
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn is_fsa_available_returns_bool() {
    let b: bool = fsa_choose_exec::is_fsa_available();
    let _ = b;
}

#[wasm_bindgen_test]
fn ensure_directory_available_consistent_with_caps() {
    let caps = Capabilities::detect();
    let ensured = fsa_choose_exec::ensure_directory_available();
    if caps.directory {
        assert!(ensured.is_none());
    } else {
        assert!(ensured.is_some());
    }
}

// ---------------------------------------------------------------------------
// PickOutcome — классификация без user gesture
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn open_files_without_gesture_is_not_picked() {
    let opts = build_open_options(false, false, &[]).unwrap();
    let outcome = fsa_choose_exec::open_files(opts).await;
    assert!(!outcome.is_picked(), "без gesture выбор невозможен");
    if let PickOutcome::Cancelled = outcome {
        return;
    }
}

#[wasm_bindgen_test]
async fn save_file_without_gesture_is_not_picked() {
    let opts = build_save_options("t.txt", &[]).unwrap();
    let outcome = fsa_choose_exec::save_file(opts).await;
    assert!(!outcome.is_picked(), "без gesture выбор невозможен");
    if let PickOutcome::Cancelled = outcome {
        return;
    }
}

#[wasm_bindgen_test]
async fn pick_directory_without_gesture_is_not_picked() {
    let opts =
        build_directory_options(DirectoryMode::Read, None, None).unwrap();
    let outcome = fsa_choose_exec::pick_directory(opts).await;
    assert!(!outcome.is_picked(), "без gesture выбор невозможен");
    if let PickOutcome::Cancelled = outcome {
        return;
    }
}

// ---------------------------------------------------------------------------
// PickOutcome — kind_label и предикаты
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn pick_outcome_kind_labels() {
    let ok: PickOutcome<()> = PickOutcome::Picked(());
    assert_eq!(ok.kind_label(), "OK");
    let c: PickOutcome<()> = PickOutcome::Cancelled;
    assert_eq!(c.kind_label(), "отменено");
    let b: PickOutcome<()> = PickOutcome::Blocked("x".into());
    assert_eq!(b.kind_label(), "заблокировано");
    let f: PickOutcome<()> = PickOutcome::Failed("x".into());
    assert_eq!(f.kind_label(), "ошибка");
}

#[wasm_bindgen_test]
fn pick_outcome_is_picked() {
    let ok: PickOutcome<()> = PickOutcome::Picked(());
    assert!(ok.is_picked());
    let c: PickOutcome<()> = PickOutcome::Cancelled;
    assert!(!c.is_picked());
    let b: PickOutcome<()> = PickOutcome::Blocked("x".into());
    assert!(!b.is_picked());
    let f: PickOutcome<()> = PickOutcome::Failed("x".into());
    assert!(!f.is_picked());
}

#[wasm_bindgen_test]
fn pick_outcome_is_cancelled() {
    let ok: PickOutcome<()> = PickOutcome::Picked(());
    assert!(!ok.is_cancelled());
    let c: PickOutcome<()> = PickOutcome::Cancelled;
    assert!(c.is_cancelled());
}

// ---------------------------------------------------------------------------
// Fallback — download_blob и DownloadInfo
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn download_blob_returns_info() {
    let blob = fsa_choose_fallback::make_text_blob("hi").unwrap();
    let info = fsa_choose_fallback::download_blob(&blob, "t.txt").unwrap();
    assert_eq!(info.suggested_name, "t.txt");
    assert!(info.blob_size > 0.0);
}

#[wasm_bindgen_test]
fn describe_download_includes_name_and_size() {
    let info = fsa_choose_fallback::DownloadInfo {
        suggested_name: "t.txt".into(),
        blob_size: 4.0,
    };
    let s = fsa_choose_fallback::describe_download(&info);
    assert!(s.contains("t.txt"));
    assert!(s.contains("4"));
}

#[wasm_bindgen_test]
fn make_text_blob_has_utf8_type() {
    let blob = fsa_choose_fallback::make_text_blob("тест").unwrap();
    assert!(blob.type_().starts_with("text/plain"));
}

#[wasm_bindgen_test]
fn make_text_blob_size_matches_content() {
    let text = "Hello, fallback!";
    let blob = fsa_choose_fallback::make_text_blob(text).unwrap();
    assert_eq!(blob.size() as usize, text.len());
}

#[wasm_bindgen_test]
fn directory_unsupported_message_mentions_browsers() {
    let s = fsa_choose_fallback::directory_unsupported_message();
    assert!(s.contains("Chrome"));
    assert!(s.contains("Edge"));
    assert!(s.contains("<input type=file>"));
}

// ---------------------------------------------------------------------------
// Fallback — интерактивный тест (требует user gesture)
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
#[ignore]
async fn pick_via_input_requires_gesture() {
    let out = fsa_choose_fallback::pick_via_input("text/plain").await;
    match out {
        PickOutcome::Cancelled => {}
        PickOutcome::Picked(_) => {}
        PickOutcome::Blocked(_) => {}
        PickOutcome::Failed(_) => {}
    }
}
```

---

## Команды запуска

```bash
# проверка типов
cargo check --target wasm32-unknown-unknown

# тесты подглавы
wasm-pack test --headless --chrome --test wasm_fsa_choose

# интерактивная проверка fallback через <input> (ручной прогон)
wasm-pack test --chrome --test wasm_fsa_choose -- --ignored

# демо
npm start
```

---

## Демо-пример в `src/app.rs`

**Три независимых блока** — по одному на каждый picker. У каждого свой набор контролов, свой статус, результат, метрики и кнопка «Сбросить». Плюс self-test в начале `run()`: если FSA доступен не полностью — показывается только диагностическая заглушка.

```rust
//! Демо-секция 19.2: три picker'а — open / save / directory.
//!
//! Self-test — первичная функция: если FSA доступен не полностью,
//! интерфейс не показывается.
//!
//! Три функционала вынесены в три отдельных блока:
//! * «Открытие файлов» — preset фильтра, multiple, excludeAcceptAll.
//! * «Сохранение файла» — textarea, фильтр text/plain, read-back.
//! * «Работа с папкой» — режим read / readwrite.
//! У каждого блока свой статус, результат, метрики и кнопка «Сбросить».
//!
//! Контракт с `lib.rs`: `run() -> Result<(), JsValue>`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    HtmlButtonElement, HtmlElement, HtmlInputElement, HtmlOptionElement,
    HtmlSelectElement, HtmlTextAreaElement,
};

use crate::fsa_choose_exec::{self, Capabilities, PickOutcome};
use crate::fsa_choose_options::{
    self, describe_filters, format_str, AcceptFilter, DirectoryMode,
    FilterPreset,
};

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

const DIR_ID: &str = "fsa-choose-demo";
const SAVE_NAME: &str = "output.txt";

// ---------------------------------------------------------------------------
// CSS
// ---------------------------------------------------------------------------

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);
        let css = r#"
            #fsa-choose { font-family: system-ui, sans-serif; max-width: 1000px; }
            #fsa-choose button { padding: 6px 12px; cursor: pointer; margin-right: 8px; }
            #fsa-choose button:disabled { opacity: 0.5; cursor: not-allowed; }
            #fsa-choose .unsupported {
                margin: 12px 0 0 0; padding: 16px 20px;
                background: #fdecea; border-left: 5px solid #c0392b;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px; line-height: 1.7;
                color: #5a1a13;
            }
            #fsa-choose .help {
                margin: 0 0 16px 0; padding: 10px 14px;
                background: #f9f9f9; border: 1px solid #e5e5e5;
                border-radius: 6px; font-size: 13px; line-height: 1.6;
                white-space: pre-line; color: #333;
            }
            #fsa-choose .block {
                margin: 0 0 20px 0; padding: 14px 18px;
                border: 1px solid #ddd; border-radius: 8px;
                background: #fdfdfd;
            }
            #fsa-choose .block h3 {
                margin: 0 0 10px 0; font-size: 15px; color: #222;
                border-bottom: 1px solid #eee; padding-bottom: 6px;
            }
            #fsa-choose .block .purpose {
                font-size: 12px; color: #666; margin: 0 0 12px 0;
                white-space: pre-line; line-height: 1.5;
            }
            #fsa-choose .row { margin: 8px 0; display: flex; gap: 12px; align-items: center; flex-wrap: wrap; }
            #fsa-choose .row label { font-size: 13px; color: #333; }
            #fsa-choose .row select { font-size: 13px; padding: 3px 6px; }
            #fsa-choose .row .cb { display: inline-flex; gap: 6px; align-items: center; font-size: 13px; }
            #fsa-choose .payload-label {
                font-size: 13px; color: #333; margin-top: 4px;
                display: flex; align-items: center; gap: 8px;
            }
            #fsa-choose .payload-label .hint {
                font-size: 12px; color: #888;
            }
            #fsa-choose textarea {
                width: 100%; min-height: 60px; font-family: monospace;
                font-size: 13px; padding: 6px 8px; box-sizing: border-box;
                border: 1px solid #ccc; border-radius: 4px;
            }
            #fsa-choose .status {
                margin: 10px 0; padding: 8px 12px;
                background: #f0f4ff; border-left: 4px solid #4a7cff;
                border-radius: 4px; white-space: pre-line; font-family: monospace;
            }
            #fsa-choose .status.ok      { border-left-color: #27ae60; background: #e8f7ee; }
            #fsa-choose .status.cancel  { border-left-color: #7f8c8d; background: #f2f3f4; }
            #fsa-choose .status.blocked { border-left-color: #e6a700; background: #fffbe0; }
            #fsa-choose .status.failed  { border-left-color: #c0392b; background: #fdecea; }
            #fsa-choose .result {
                background: #f6f6f6; padding: 8px 12px; border-radius: 4px;
                font-family: monospace; font-size: 13px;
                white-space: pre-wrap; margin: 8px 0; min-height: 48px;
            }
            #fsa-choose .metrics {
                font-size: 12px; color: #777;
                font-family: monospace; margin-top: 8px;
            }
        "#;
        let doc = web_sys::window()
            .ok_or_else(|| JsValue::from_str("no window"))?
            .document()
            .ok_or_else(|| JsValue::from_str("no document"))?;
        let style = doc.create_element("style")?;
        style.set_text_content(Some(css));
        if let Some(head) = doc.head() {
            let _ = head.append_child(&style);
        }
        Ok(())
    })
}

// ---------------------------------------------------------------------------
// Общие структуры
// ---------------------------------------------------------------------------

#[derive(Default)]
struct DemoStats {
    ops: u32,
    last_ms: f64,
    last_label: String,
}

struct OpenUi {
    preset: HtmlSelectElement,
    multiple: HtmlInputElement,
    exclude: HtmlInputElement,
    hint: HtmlElement,
    open_btn: HtmlButtonElement,
    reset_btn: HtmlButtonElement,
    status: HtmlElement,
    result: HtmlElement,
    metrics: HtmlElement,
    stats: Rc<RefCell<DemoStats>>,
}

struct SaveUi {
    payload: HtmlTextAreaElement,
    save_btn: HtmlButtonElement,
    reset_btn: HtmlButtonElement,
    status: HtmlElement,
    result: HtmlElement,
    metrics: HtmlElement,
    stats: Rc<RefCell<DemoStats>>,
}

struct DirUi {
    mode: HtmlSelectElement,
    dir_btn: HtmlButtonElement,
    reset_btn: HtmlButtonElement,
    status: HtmlElement,
    result: HtmlElement,
    metrics: HtmlElement,
    stats: Rc<RefCell<DemoStats>>,
}

// ---------------------------------------------------------------------------
// Точка входа
// ---------------------------------------------------------------------------

pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;

    let section = doc.create_element("section")?;
    section.set_id("fsa-choose");

    let h2 = doc.create_element("h2")?;
    h2.set_text_content(Some("19.2 — Три picker'а: open / save / directory"));
    section.append_child(&h2)?;

    // --- Self-test. ---
    let caps = Capabilities::detect();
    if !caps.all() {
        let unsupported: HtmlElement = doc
            .create_element("div")?
            .dyn_into::<HtmlElement>()
            .map_err(|_| JsValue::from_str("не HtmlElement"))?;
        unsupported.set_attribute("class", "unsupported")?;
        unsupported.set_text_content(Some(&unsupported_block_text(&caps)));
        section.append_child(&unsupported)?;

        body.append_child(&section)?;
        return Ok(());
    }

    // --- Help. ---
    let help: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    help.set_attribute("class", "help")?;
    help.set_text_content(Some(&help_text()));
    section.append_child(&help)?;

    // --- Три блока. ---
    let open_ui = build_open_block(&doc, &section)?;
    let save_ui = build_save_block(&doc, &section)?;
    let dir_ui = build_dir_block(&doc, &section)?;

    // --- Wire. ---
    wire_open(&open_ui);
    wire_save(&save_ui);
    wire_dir(&dir_ui);

    body.append_child(&section)?;
    Ok(())
}

// ---------------------------------------------------------------------------
// Блок 1: Открытие файлов
// ---------------------------------------------------------------------------

fn build_open_block(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<OpenUi, JsValue> {
    let block = make_block(doc, "Открытие файлов (showOpenFilePicker)")?;

    let purpose = make_purpose(
        doc,
        "Цель: показать диалог выбора файла с настраиваемым фильтром \
         и режимом множественного выбора. Возвращает список handle'ов \
         выбранных файлов.",
    )?;
    block.append_child(&purpose)?;

    // Контролы.
    let preset = build_preset_select(doc)?;
    let multiple = append_checkbox_row(doc, &block, "ch-open-multiple", "multiple")?;
    let exclude = append_checkbox_row(
        doc,
        &block,
        "ch-open-exclude",
        "excludeAcceptAllOption",
    )?;
    multiple.set_checked(true);
    append_row(doc, &block, "Пресет фильтра:", &preset)?;

    // Hint — пересобирается при смене опций.
    let hint: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    hint.set_attribute("class", "status")?;
    block.append_child(&hint)?;

    // Кнопки.
    let buttons = doc.create_element("div")?;
    buttons.set_attribute("style", "margin-top: 8px;")?;
    let open_btn = make_button(doc, "Открыть файлы", "ch-open-btn")?;
    let reset_btn = make_button(doc, "Сбросить", "ch-open-reset")?;
    buttons.append_child(&open_btn)?;
    buttons.append_child(&reset_btn)?;
    block.append_child(&buttons)?;

    // Статус.
    let status: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    status.set_attribute("class", "status")?;
    status.set_text_content(Some("Готово."));
    block.append_child(&status)?;

    // Результат.
    let result: HtmlElement = doc
        .create_element("pre")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    result.set_attribute("class", "result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    block.append_child(&result)?;

    // Метрики.
    let metrics: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    metrics.set_attribute("class", "metrics")?;
    metrics.set_text_content(Some(&metrics_text(0, 0.0, "")));
    block.append_child(&metrics)?;

    parent.append_child(&block)?;

    let ui = OpenUi {
        preset,
        multiple,
        exclude,
        hint,
        open_btn,
        reset_btn,
        status,
        result,
        metrics,
        stats: Rc::new(RefCell::new(DemoStats::default())),
    };

    update_open_hint(&ui);

    Ok(ui)
}

fn wire_open(ui: &OpenUi) {
    // Изменения опций → обновление hint.
    let ui_c = ui.preset.clone();
    let multiple_c = ui.multiple.clone();
    let exclude_c = ui.exclude.clone();
    let hint_c = ui.hint.clone();
    let on_change = Closure::<dyn FnMut()>::new(move || {
        let preset = preset_from_code(&ui_c.value());
        let text = open_hint_text(
            preset,
            multiple_c.checked(),
            exclude_c.checked(),
        );
        hint_c.set_text_content(Some(&text));
    });
    ui.preset
        .set_onchange(Some(on_change.as_ref().unchecked_ref()));
    ui.multiple
        .set_onchange(Some(on_change.as_ref().unchecked_ref()));
    ui.exclude
        .set_onchange(Some(on_change.as_ref().unchecked_ref()));
    on_change.forget();

    // Кнопка «Открыть файлы».
    let preset_c = ui.preset.clone();
    let multiple_c = ui.multiple.clone();
    let exclude_c = ui.exclude.clone();
    let open_btn_c = ui.open_btn.clone();
    let status_c = ui.status.clone();
    let result_c = ui.result.clone();
    let metrics_c = ui.metrics.clone();
    let stats_c = ui.stats.clone();
    let on_open = Closure::<dyn FnMut()>::new(move || {
        let preset_el = preset_c.clone();
        let multiple_el = multiple_c.clone();
        let exclude_el = exclude_c.clone();
        let status_el = status_c.clone();
        let result_el = result_c.clone();
        let metrics_el = metrics_c.clone();
        let stats = stats_c.clone();
        let _ = &open_btn_c;
        spawn_local(async move {
            set_status_on(&status_el, "", "Открытие файлов…");
            result_el.set_text_content(Some("(нет данных)"));

            let preset = preset_from_code(&preset_el.value());
            let multiple = multiple_el.checked();
            let exclude = exclude_el.checked();
            let filters = preset.build();
            let t0 = js_sys::Date::now();

            let opts = match fsa_choose_options::build_open_options(
                multiple, exclude, &filters,
            ) {
                Ok(o) => o,
                Err(e) => {
                    let msg: String = format_str(format_args!("Опции: {e:?}"));
                    set_status_on(&status_el, "failed", &msg);
                    return;
                }
            };
            let outcome = fsa_choose_exec::open_files(opts).await;
            let label = apply_open_outcome(&status_el, &result_el, outcome);
            let dt = js_sys::Date::now() - t0;
            bump_stats(&stats, dt, label);
            update_metrics_on(&metrics_el, &stats);
        });
    });
    ui.open_btn
        .set_onclick(Some(on_open.as_ref().unchecked_ref()));
    on_open.forget();

    // Кнопка «Сбросить».
    let status_c2 = ui.status.clone();
    let result_c2 = ui.result.clone();
    let metrics_c2 = ui.metrics.clone();
    let stats_c2 = ui.stats.clone();
    let on_reset = Closure::<dyn FnMut()>::new(move || {
        {
            let mut s = stats_c2.borrow_mut();
            *s = DemoStats::default();
        }
        set_status_on(&status_c2, "", "Блок сброшен.");
        result_c2.set_text_content(Some("(нет данных)"));
        let text: String = metrics_text(0, 0.0, "");
        metrics_c2.set_text_content(Some(&text));
    });
    ui.reset_btn
        .set_onclick(Some(on_reset.as_ref().unchecked_ref()));
    on_reset.forget();
}

fn update_open_hint(ui: &OpenUi) {
    let preset = preset_from_code(&ui.preset.value());
    let text = open_hint_text(preset, ui.multiple.checked(), ui.exclude.checked());
    ui.hint.set_text_content(Some(&text));
}

fn open_hint_text(
    preset: FilterPreset,
    multiple: bool,
    exclude: bool,
) -> String {
    let filter_desc: String = describe_filters(&preset.build());
    return format_str(format_args!(
        "• Фильтр: «{filter_desc}»\n\
         • multiple = {multiple}, excludeAcceptAllOption = {exclude}",
    ));
}

fn apply_open_outcome(
    status: &HtmlElement,
    result: &HtmlElement,
    outcome: PickOutcome<Vec<web_sys::FileSystemFileHandle>>,
) -> &'static str {
    let label: &'static str = outcome.kind_label();
    let class: &'static str = class_for_label(label);

    if outcome.is_picked() {
        if let PickOutcome::Picked(ref list) = outcome {
            let names: Vec<String> = list.iter().map(|h| h.name()).collect();
            let text: String = format_str(format_args!(
                "Выбрано {} файлов:\n{}",
                names.len(),
                names.join("\n"),
            ));
            result.set_text_content(Some(&text));
            set_status_on(status, "ok", label);
        }
        return label;
    }

    if outcome.is_cancelled() {
        set_status_on(status, class, label);
        return label;
    }

    match outcome {
        PickOutcome::Blocked(m) => set_status_on(status, class, &m),
        PickOutcome::Failed(m) => set_status_on(status, class, &m),
        _ => {}
    }
    return label;
}

// ---------------------------------------------------------------------------
// Блок 2: Сохранение файла
// ---------------------------------------------------------------------------

fn build_save_block(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<SaveUi, JsValue> {
    let block = make_block(doc, "Сохранение файла (showSaveFilePicker)")?;

    let purpose = make_purpose(
        doc,
        "Цель: записать введённый текст в выбранный файл через \
         FileSystemWritableFileStream, а затем перечитать его (read-back) — \
         чтобы убедиться, что данные действительно легли на диск.",
    )?;
    block.append_child(&purpose)?;

    // Метка + textarea.
    let payload_label = doc.create_element("div")?;
    payload_label.set_attribute("class", "payload-label")?;
    let label_text = doc.create_element("span")?;
    label_text.set_text_content(Some("Текст для сохранения:"));
    let label_hint = doc.create_element("span")?;
    label_hint.set_attribute("class", "hint")?;
    label_hint.set_text_content(Some(
        "(кнопка активна только если поле непустое)",
    ));
    payload_label.append_child(&label_text)?;
    payload_label.append_child(&label_hint)?;
    block.append_child(&payload_label)?;

    let payload: HtmlTextAreaElement = doc
        .create_element("textarea")?
        .dyn_into::<HtmlTextAreaElement>()
        .map_err(|_| JsValue::from_str("не HtmlTextAreaElement"))?;
    payload.set_id("ch-save-payload");
    payload.set_value("");
    payload.set_placeholder("Введите текст для сохранения");
    block.append_child(&payload)?;

    // Кнопки.
    let buttons = doc.create_element("div")?;
    buttons.set_attribute("style", "margin-top: 8px;")?;
    let save_btn = make_button(doc, "Сохранить как…", "ch-save-btn")?;
    let reset_btn = make_button(doc, "Сбросить", "ch-save-reset")?;
    save_btn.set_disabled(true);
    save_btn.set_title("Нет данных для сохранения — введите текст в поле выше");
    buttons.append_child(&save_btn)?;
    buttons.append_child(&reset_btn)?;
    block.append_child(&buttons)?;

    // Статус.
    let status: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    status.set_attribute("class", "status")?;
    status.set_text_content(Some(
        "Готово. Введите текст — и кнопка станет активной.",
    ));
    block.append_child(&status)?;

    // Результат.
    let result: HtmlElement = doc
        .create_element("pre")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    result.set_attribute("class", "result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    block.append_child(&result)?;

    // Метрики.
    let metrics: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    metrics.set_attribute("class", "metrics")?;
    metrics.set_text_content(Some(&metrics_text(0, 0.0, "")));
    block.append_child(&metrics)?;

    parent.append_child(&block)?;

    Ok(SaveUi {
        payload,
        save_btn,
        reset_btn,
        status,
        result,
        metrics,
        stats: Rc::new(RefCell::new(DemoStats::default())),
    })
}

fn wire_save(ui: &SaveUi) {
    // Состояние кнопки Save зависит от поля.
    let save_btn_c = ui.save_btn.clone();
    let payload_c = ui.payload.clone();
    let on_input = Closure::<dyn FnMut()>::new(move || {
        let has = !payload_c.value().trim().is_empty();
        if has {
            save_btn_c.set_disabled(false);
            save_btn_c.set_title(
                "Сохранить текст из поля как файл (showSaveFilePicker)",
            );
        } else {
            save_btn_c.set_disabled(true);
            save_btn_c.set_title(
                "Нет данных для сохранения — введите текст в поле выше",
            );
        }
    });
    let target: &web_sys::EventTarget = ui.payload.unchecked_ref();
    let _ = target.add_event_listener_with_callback(
        "input",
        on_input.as_ref().unchecked_ref(),
    );
    on_input.forget();

    // Клик по «Сохранить как…».
    let payload_c2 = ui.payload.clone();
    let status_c = ui.status.clone();
    let result_c = ui.result.clone();
    let metrics_c = ui.metrics.clone();
    let stats_c = ui.stats.clone();
    let on_save = Closure::<dyn FnMut()>::new(move || {
        let payload_el = payload_c2.clone();
        let status_el = status_c.clone();
        let result_el = result_c.clone();
        let metrics_el = metrics_c.clone();
        let stats = stats_c.clone();
        spawn_local(async move {
            result_el.set_text_content(Some("(нет данных)"));

            if payload_el.value().trim().is_empty() {
                set_status_on(
                    &status_el,
                    "blocked",
                    "Нет данных для сохранения — поле пустое.",
                );
                return;
            }

            set_status_on(&status_el, "", "Сохранение…");

            let filters = FilterPreset::Text.build();
            let suggested = SAVE_NAME;
            let payload = payload_el.value();
            let t0 = js_sys::Date::now();

            let label: String =
                run_save(&status_el, &result_el, suggested, &filters, &payload)
                    .await;

            let dt = js_sys::Date::now() - t0;
            bump_stats(&stats, dt, &label);
            update_metrics_on(&metrics_el, &stats);
        });
    });
    ui.save_btn
        .set_onclick(Some(on_save.as_ref().unchecked_ref()));
    on_save.forget();

    // «Сбросить».
    let payload_c3 = ui.payload.clone();
    let save_btn_c3 = ui.save_btn.clone();
    let status_c3 = ui.status.clone();
    let result_c3 = ui.result.clone();
    let metrics_c3 = ui.metrics.clone();
    let stats_c3 = ui.stats.clone();
    let on_reset = Closure::<dyn FnMut()>::new(move || {
        {
            let mut s = stats_c3.borrow_mut();
            *s = DemoStats::default();
        }
        payload_c3.set_value("");
        save_btn_c3.set_disabled(true);
        save_btn_c3.set_title("Нет данных для сохранения — введите текст в поле выше");
        set_status_on(
            &status_c3,
            "",
            "Блок сброшен. Введите текст — и кнопка станет активной.",
        );
        result_c3.set_text_content(Some("(нет данных)"));
        let text: String = metrics_text(0, 0.0, "");
        metrics_c3.set_text_content(Some(&text));
    });
    ui.reset_btn
        .set_onclick(Some(on_reset.as_ref().unchecked_ref()));
    on_reset.forget();
}

async fn run_save(
    status: &HtmlElement,
    result: &HtmlElement,
    suggested: &str,
    filters: &[AcceptFilter],
    payload: &str,
) -> String {
    let opts = match fsa_choose_options::build_save_options(suggested, filters) {
        Ok(o) => o,
        Err(e) => {
            let msg: String = format_str(format_args!("Опции: {e:?}"));
            set_status_on(status, "failed", &msg);
            return "ошибка".to_string();
        }
    };

    let outcome = fsa_choose_exec::save_file(opts).await;
    let pick_label: &'static str = outcome.kind_label();
    let pick_class: &'static str = class_for_label(pick_label);

    if outcome.is_cancelled() {
        set_status_on(status, pick_class, pick_label);
        return pick_label.to_string();
    }

    let handle = match outcome {
        PickOutcome::Picked(h) => h,
        PickOutcome::Blocked(m) => {
            set_status_on(status, pick_class, &m);
            return pick_label.to_string();
        }
        PickOutcome::Failed(m) => {
            set_status_on(status, pick_class, &m);
            return pick_label.to_string();
        }
        PickOutcome::Cancelled => {
            set_status_on(status, pick_class, pick_label);
            return pick_label.to_string();
        }
    };

    let stream = match crate::fsa_api_file::create_writable(&handle, false).await {
        Ok(s) => s,
        Err(e) => {
            let msg: String = format_str(format_args!("create_writable: {e:?}"));
            set_status_on(status, "failed", &msg);
            return "ошибка".to_string();
        }
    };
    if let Err(e) = crate::fsa_api_writable::write_text(&stream, payload).await {
        let msg: String = format_str(format_args!("write: {e:?}"));
        set_status_on(status, "failed", &msg);
        return "ошибка".to_string();
    }
    if let Err(e) = crate::fsa_api_writable::close(&stream).await {
        let msg: String = format_str(format_args!("close: {e:?}"));
        set_status_on(status, "failed", &msg);
        return "ошибка".to_string();
    }

    let file = match crate::fsa_api_file::get_file(&handle).await {
        Ok(f) => f,
        Err(e) => {
            let text: String = format_str(format_args!(
                "Сохранено: {} ({} байт)\n⚠ get_file: {e:?}",
                handle.name(),
                payload.len(),
            ));
            result.set_text_content(Some(&text));
            set_status_on(status, "ok", pick_label);
            return pick_label.to_string();
        }
    };
    let back = match crate::fsa_api_reader::read_text(&file).await {
        Ok(t) => t,
        Err(e) => {
            let text: String = format_str(format_args!(
                "Сохранено: {} ({} байт)\n⚠ read_text: {e:?}",
                handle.name(),
                payload.len(),
            ));
            result.set_text_content(Some(&text));
            set_status_on(status, "ok", pick_label);
            return pick_label.to_string();
        }
    };

    let verdict: String = if back == payload {
        "✅ read-back совпал".to_string()
    } else {
        format_str(format_args!(
            "⚠ read-back отличается (записано {} байт, прочитано {})",
            payload.len(),
            back.len(),
        ))
    };
    let text: String = format_str(format_args!(
        "Файл:      {}\nРазмер:    {} байт\n{verdict}",
        handle.name(),
        payload.len(),
    ));
    result.set_text_content(Some(&text));
    set_status_on(status, "ok", pick_label);
    return pick_label.to_string();
}

// ---------------------------------------------------------------------------
// Блок 3: Работа с папкой
// ---------------------------------------------------------------------------

fn build_dir_block(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<DirUi, JsValue> {
    let block = make_block(doc, "Работа с папкой (showDirectoryPicker)")?;

    let purpose = make_purpose(
        doc,
        "Цель: показать выбор директории через showDirectoryPicker. \
         Режим read — только чтение, readwrite — чтение и запись. \
         id позволяет браузеру запоминать последнюю папку для этого приложения.",
    )?;
    block.append_child(&purpose)?;

    let mode = build_dir_mode_select(doc)?;
    append_row(doc, &block, "Режим:", &mode)?;

    let buttons = doc.create_element("div")?;
    buttons.set_attribute("style", "margin-top: 8px;")?;
    let dir_btn = make_button(doc, "Выбрать папку", "ch-dir-btn")?;
    let reset_btn = make_button(doc, "Сбросить", "ch-dir-reset")?;
    buttons.append_child(&dir_btn)?;
    buttons.append_child(&reset_btn)?;
    block.append_child(&buttons)?;

    let status: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    status.set_attribute("class", "status")?;
    status.set_text_content(Some(&format_str(format_args!(
        "Готово. id для запоминания последней папки: \"{DIR_ID}\".",
    ))));
    block.append_child(&status)?;

    let result: HtmlElement = doc
        .create_element("pre")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    result.set_attribute("class", "result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    block.append_child(&result)?;

    let metrics: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    metrics.set_attribute("class", "metrics")?;
    metrics.set_text_content(Some(&metrics_text(0, 0.0, "")));
    block.append_child(&metrics)?;

    parent.append_child(&block)?;

    Ok(DirUi {
        mode,
        dir_btn,
        reset_btn,
        status,
        result,
        metrics,
        stats: Rc::new(RefCell::new(DemoStats::default())),
    })
}

fn wire_dir(ui: &DirUi) {
    let mode_c = ui.mode.clone();
    let status_c = ui.status.clone();
    let result_c = ui.result.clone();
    let metrics_c = ui.metrics.clone();
    let stats_c = ui.stats.clone();
    let on_dir = Closure::<dyn FnMut()>::new(move || {
        let mode_el = mode_c.clone();
        let status_el = status_c.clone();
        let result_el = result_c.clone();
        let metrics_el = metrics_c.clone();
        let stats = stats_c.clone();
        spawn_local(async move {
            result_el.set_text_content(Some("(нет данных)"));
            set_status_on(&status_el, "", "Выбор папки…");

            let mode = dir_mode_from_code(&mode_el.value());
            let t0 = js_sys::Date::now();

            let opts = match fsa_choose_options::build_directory_options(
                mode,
                Some(DIR_ID),
                None,
            ) {
                Ok(o) => o,
                Err(e) => {
                    let msg: String = format_str(format_args!("Опции: {e:?}"));
                    set_status_on(&status_el, "failed", &msg);
                    bump_stats(&stats, 0.0, "ошибка");
                    update_metrics_on(&metrics_el, &stats);
                    return;
                }
            };
            let outcome = fsa_choose_exec::pick_directory(opts).await;
            let label = apply_dir_outcome(&status_el, &result_el, outcome, mode);
            let dt = js_sys::Date::now() - t0;
            bump_stats(&stats, dt, label);
            update_metrics_on(&metrics_el, &stats);
        });
    });
    ui.dir_btn
        .set_onclick(Some(on_dir.as_ref().unchecked_ref()));
    on_dir.forget();

    let status_c2 = ui.status.clone();
    let result_c2 = ui.result.clone();
    let metrics_c2 = ui.metrics.clone();
    let stats_c2 = ui.stats.clone();
    let on_reset = Closure::<dyn FnMut()>::new(move || {
        {
            let mut s = stats_c2.borrow_mut();
            *s = DemoStats::default();
        }
        set_status_on(&status_c2, "", "Блок сброшен.");
        result_c2.set_text_content(Some("(нет данных)"));
        let text: String = metrics_text(0, 0.0, "");
        metrics_c2.set_text_content(Some(&text));
    });
    ui.reset_btn
        .set_onclick(Some(on_reset.as_ref().unchecked_ref()));
    on_reset.forget();
}

fn apply_dir_outcome(
    status: &HtmlElement,
    result: &HtmlElement,
    outcome: PickOutcome<web_sys::FileSystemDirectoryHandle>,
    mode: DirectoryMode,
) -> &'static str {
    let label: &'static str = outcome.kind_label();
    let class: &'static str = class_for_label(label);

    if outcome.is_picked() {
        if let PickOutcome::Picked(ref handle) = outcome {
            let text: String = format_str(format_args!(
                "Директория: {}\nРежим:      \"{}\"",
                handle.name(),
                mode.as_str(),
            ));
            result.set_text_content(Some(&text));
            set_status_on(status, "ok", label);
        }
        return label;
    }

    if outcome.is_cancelled() {
        set_status_on(status, class, label);
        return label;
    }

    match outcome {
        PickOutcome::Blocked(m) => set_status_on(status, class, &m),
        PickOutcome::Failed(m) => set_status_on(status, class, &m),
        _ => {}
    }
    return label;
}

// ---------------------------------------------------------------------------
// Построение общих элементов
// ---------------------------------------------------------------------------

fn make_block(
    doc: &web_sys::Document,
    title: &str,
) -> Result<HtmlElement, JsValue> {
    let block: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    block.set_attribute("class", "block")?;
    let h3 = doc.create_element("h3")?;
    h3.set_text_content(Some(title));
    block.append_child(&h3)?;
    Ok(block)
}

fn make_purpose(
    doc: &web_sys::Document,
    text: &str,
) -> Result<HtmlElement, JsValue> {
    let p: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    p.set_attribute("class", "purpose")?;
    p.set_text_content(Some(text));
    Ok(p)
}

fn build_preset_select(
    doc: &web_sys::Document,
) -> Result<HtmlSelectElement, JsValue> {
    let sel: HtmlSelectElement = doc
        .create_element("select")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlSelectElement"))?;
    sel.set_id("ch-open-preset");
    for (code, label) in [
        ("images", FilterPreset::Images.as_str()),
        ("text", FilterPreset::Text.as_str()),
        ("none", FilterPreset::None.as_str()),
    ] {
        let opt: HtmlOptionElement = doc
            .create_element("option")?
            .dyn_into()
            .map_err(|_| JsValue::from_str("не HtmlOptionElement"))?;
        opt.set_value(code);
        opt.set_text_content(Some(label));
        sel.append_child(&opt)?;
    }
    Ok(sel)
}

fn build_dir_mode_select(
    doc: &web_sys::Document,
) -> Result<HtmlSelectElement, JsValue> {
    let sel: HtmlSelectElement = doc
        .create_element("select")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlSelectElement"))?;
    sel.set_id("ch-dir-mode");
    for mode in [DirectoryMode::Read, DirectoryMode::ReadWrite] {
        let opt: HtmlOptionElement = doc
            .create_element("option")?
            .dyn_into()
            .map_err(|_| JsValue::from_str("не HtmlOptionElement"))?;
        opt.set_value(mode.as_str());
        opt.set_text_content(Some(mode.as_str()));
        sel.append_child(&opt)?;
    }
    Ok(sel)
}

fn append_checkbox_row(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
    id: &str,
    label_text: &str,
) -> Result<HtmlInputElement, JsValue> {
    let row = doc.create_element("div")?;
    row.set_attribute("class", "row")?;

    let wrapper = doc.create_element("label")?;
    wrapper.set_attribute("class", "cb")?;

    let cb: HtmlInputElement = doc
        .create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlInputElement"))?;
    cb.set_type("checkbox");
    cb.set_id(id);

    let text = doc.create_element("span")?;
    text.set_text_content(Some(label_text));

    wrapper.append_child(&cb)?;
    wrapper.append_child(&text)?;
    row.append_child(&wrapper)?;
    parent.append_child(&row)?;

    Ok(cb)
}

fn append_row(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
    label: &str,
    control: &web_sys::Element,
) -> Result<(), JsValue> {
    let row = doc.create_element("div")?;
    row.set_attribute("class", "row")?;
    if !label.is_empty() {
        let lbl = doc.create_element("label")?;
        lbl.set_text_content(Some(label));
        row.append_child(&lbl)?;
    }
    row.append_child(control)?;
    parent.append_child(&row)?;
    Ok(())
}

fn make_button(
    doc: &web_sys::Document,
    text: &str,
    id: &str,
) -> Result<HtmlButtonElement, JsValue> {
    let b: HtmlButtonElement = doc
        .create_element("button")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlButtonElement"))?;
    b.set_id(id);
    b.set_text_content(Some(text));
    Ok(b)
}

// ---------------------------------------------------------------------------
// Тексты для UI
// ---------------------------------------------------------------------------

fn mark(b: bool) -> &'static str {
    if b {
        return "✓";
    }
    return "—";
}

fn unsupported_block_text(caps: &Capabilities) -> String {
    let mut lines: Vec<String> = Vec::with_capacity(24);

    if !caps.any() {
        lines.push(
            "✗ File System Access API недоступен в этом браузере.".to_string(),
        );
    } else {
        lines.push(
            "⚠ File System Access API доступен не полностью.".to_string(),
        );
    }

    lines.push(String::new());
    lines.push("Автопроверка методов:".to_string());
    lines.push(format_str(format_args!(
        "  {}  showOpenFilePicker     — открыть один или несколько файлов",
        mark(caps.open),
    )));
    lines.push(format_str(format_args!(
        "  {}  showSaveFilePicker     — сохранить файл под новым именем",
        mark(caps.save),
    )));
    lines.push(format_str(format_args!(
        "  {}  showDirectoryPicker    — выбрать папку",
        mark(caps.directory),
    )));
    lines.push(String::new());
    lines.push(
        "Демо требует все три метода. Интерфейс скрыт до полной поддержки."
            .to_string(),
    );
    lines.push(String::new());
    lines.push("Возможные причины:".to_string());
    lines.push(
        "  •  Браузер не поддерживает FSA — Firefox и Safari пока без него."
            .to_string(),
    );
    lines.push(
        "  •  Страница открыта не в secure context: нужен HTTPS или localhost."
            .to_string(),
    );
    lines.push(
        "  •  Расширение браузера блокирует доступ к файловой системе."
            .to_string(),
    );
    lines.push(String::new());
    lines.push("Что делать:".to_string());
    lines.push(
        "  1.  Откройте демо в Chrome (86+) или Edge (86+).".to_string(),
    );
    lines.push(
        "  2.  Убедитесь, что адрес начинается с https:// или http://localhost."
            .to_string(),
    );
    lines.push(
        "  3.  Отключите расширения, блокирующие FSA, и перезагрузите страницу."
            .to_string(),
    );
    lines.push(String::new());
    lines.push("Документация:".to_string());
    lines.push(
        "  https://developer.mozilla.org/en-US/docs/Web/API/File_System_API"
            .to_string(),
    );

    return lines.join("\n");
}

fn help_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(10);
    lines.push("Как работать с демо:".to_string());
    lines.push(
        "Три независимых блока ниже демонстрируют три метода FSA. \
         Каждый блок самодостаточен: свой набор контролов, свой статус, \
         свой результат, свои метрики и своя кнопка «Сбросить»."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "1.  «Открытие файлов» — настройте фильтр (Images / Text / None), \
         multiple и excludeAcceptAllOption, затем нажмите кнопку. \
         Получите список имён выбранных файлов."
            .to_string(),
    );
    lines.push(
        "2.  «Сохранение файла» — введите текст, кнопка станет активной. \
         Сохранение всегда text/plain, suggestedName = \"output.txt\". \
         После записи файл перечитывается (read-back)."
            .to_string(),
    );
    lines.push(
        "3.  «Работа с папкой» — выберите режим read или readwrite, \
         нажмите кнопку. Получите имя директории."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "Статус каждого вызова: OK / отменено / заблокировано / ошибка. \
         Результат последней операции виден в строке «операций» внутри блока."
            .to_string(),
    );
    return lines.join("\n");
}

fn metrics_text(ops: u32, last_ms: f64, last_label: &str) -> String {
    if ops == 0 {
        return "операций: 0".to_string();
    }
    let label: &str = if last_label.is_empty() { "—" } else { last_label };
    return format_str(format_args!(
        "операций: {ops} · последняя: {last_ms:.0} ms · результат: {label}",
    ));
}

// ---------------------------------------------------------------------------
// Value-хелперы
// ---------------------------------------------------------------------------

fn preset_from_code(code: &str) -> FilterPreset {
    match code {
        "images" => return FilterPreset::Images,
        "text" => return FilterPreset::Text,
        _ => return FilterPreset::None,
    }
}

fn dir_mode_from_code(code: &str) -> DirectoryMode {
    if code == "readwrite" {
        return DirectoryMode::ReadWrite;
    }
    return DirectoryMode::Read;
}

// ---------------------------------------------------------------------------
// UI-хелперы
// ---------------------------------------------------------------------------

fn set_status_on(target: &HtmlElement, class: &str, text: &str) {
    let cls: String = if class.is_empty() {
        "status".to_string()
    } else {
        format_str(format_args!("status {class}"))
    };
    let _ = target.set_attribute("class", &cls);
    target.set_text_content(Some(text));
}

fn update_metrics_on(target: &HtmlElement, stats: &Rc<RefCell<DemoStats>>) {
    let s = stats.borrow();
    let text = metrics_text(s.ops, s.last_ms, &s.last_label);
    target.set_text_content(Some(&text));
}

fn bump_stats(stats: &Rc<RefCell<DemoStats>>, dt: f64, label: &str) {
    let mut s = stats.borrow_mut();
    s.ops += 1;
    s.last_ms = dt;
    s.last_label = label.to_string();
}

fn class_for_label(label: &str) -> &'static str {
    match label {
        "OK" => return "ok",
        "отменено" => return "cancel",
        "заблокировано" => return "blocked",
        _ => return "failed",
    }
}
```

---

## Что демонстрация показывает

**Self-test первым делом.** `Capabilities::detect()` вызывается в самом начале `run()`. Если FSA доступен не полностью — показывается **только заглушка** с диагностикой (список найденных методов, вероятные причины, что делать, ссылка на MDN). Никакого «мёртвого» интерфейса с отключёнными кнопками.

**Три независимых блока при полной поддержке:**

1. **«Открытие файлов (showOpenFilePicker)»** — пресет фильтра (`Images` / `Text` / `None`), `multiple`, `excludeAcceptAllOption`. Живая подсказка пересобирается при смене опций. Результат — список имён выбранных файлов.
2. **«Сохранение файла (showSaveFilePicker)»** — textarea, кнопка `disabled` при пустом поле (обновляется на `input` через `add_event_listener_with_callback`). После записи — read-back с вердиктом «✅ совпал / ⚠ отличается». Всегда `text/plain` + `output.txt`, никакой зависимости от пресета открытия.
3. **«Работа с папкой (showDirectoryPicker)»** — `<select>` read / readwrite, `id = "fsa-choose-demo"` для запоминания последней папки. Результат — имя директории и выбранный режим.

**У каждого блока свой workflow:** свой набор контролов, свой статус с цветовой подсветкой, свой результат, свои метрики `операций: N · последняя: M ms · результат: K`, своя кнопка «Сбросить». Сброс одного блока не трогает другие.

**Осмысленные состояния.** `PickOutcome::kind_label()` даёт четыре метки: `OK`, `отменено`, `заблокировано`, `ошибка`. Маппинг в CSS-класс — единый (`class_for_label`). Отмена picker'а — не ошибка, показывается серым; `SecurityError` — жёлтым с подсказкой «нажмите ещё раз»; реальный сбой — красным.

## Правила проекта, зафиксированные в 19.2

1. **`app.rs` содержит только демо текущей подглавы.** Предыдущие подглавы не накапливаются.
2. **Self-test — first-class.** Если функционал недоступен, интерфейс **не показывается**, вместо него — заглушка с причиной и рекомендацией.
3. **Три функционала — три блока.** Не смешивайте разные workflow в одном UI: контролы, статус, результат, метрики у каждого блока свои.
4. **Все строки через `format_str` или `Vec<String>.join`.** Нет `format!` в хвостовой позиции `if/else`/`match`-arm — RA теряет тип.
5. **`mark()` — обычная `fn` с явной сигнатурой.** Не замыкание с `-> &'static str` — RA теряет тип в замыкании.
6. **`create_element` → отдельный `dyn_into`.** Не `.and_then(|e| e.dyn_into())` — `Result<Element, JsValue>` и `Result<T, Element>` имеют разные типы ошибок.
7. **`add_event_listener_with_callback` вместо `set_oninput`** — надёжнее для `HtmlTextAreaElement`.
8. **Пользователь видит предметное.** «Выбрано 3 файлов», «Файл: output.txt, размер 42 байт, ✅ read-back совпал», «Директория: Documents, режим: readwrite» — вместо технических деталей.
9. **Идемпотентный сброс.** Каждый блок можно сбросить многократно.
10. **`cargo check` без предупреждений.** Ни `dead_code`, ни `unused_imports`.

## Проверка

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_fsa_choose
npm start
```

**Chrome** — полный интерфейс: три блока, каждый самодостаточен. **Firefox** — только красная заглушка с диагностикой. Тесты подглавы: 37 headless + 1 интерактивный (`pick_via_input_requires_gesture`, ручной прогон).