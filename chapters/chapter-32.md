# Глава 32. Clipboard API

## 32.0. Что вы получите в этой главе

К концу главы у вас будет работающее демо **«Clipboard Inspector»** с четырьмя экспериментами:

1. **Текст.** `writeText` / `readText` с roundtrip-проверкой — показывает, что буфер не мгновенный и общий для всех программ.
2. **Бинарные данные.** Копирование PNG-изображений через `ClipboardItem` + `Blob`.
3. **Разрешения.** `Permissions API` для `clipboard-read` / `clipboard-write` с честной обработкой отказа.
4. **Multi-format item.** «Копировать как Markdown» — один `ClipboardItem` с `text/plain` + `text/html` + `text/markdown`.

Плюс модуль `clip_core`, который можно использовать как высокоуровневую обёртку над Async Clipboard API.

## 32.1. Структура файлов

```
hello-wasm02/
├── src/
│   ├── clip_core.rs        # НОВЫЙ: текст, бинарные данные, ClipboardItem
│   ├── clip_perms.rs       # НОВЫЙ: Permissions API, статус разрешений
│   ├── lib.rs              # +2 модуля
│   └── app.rs              # демо — Clipboard Inspector
├── tests/
│   ├── wasm_clip_core.rs   # НОВЫЙ: wasm-тесты ядра
│   └── wasm_clip_perms.rs  # НОВЫЙ: wasm-тесты разрешений
└── Cargo.toml              # без изменений
```

**Комментарий.** Два модуля разделяют тему по ответственности: `clip_core` — операции с содержимым (текст, бинарные данные, multi-format items, классификация ошибок), `clip_perms` — работа с разрешениями. Префикс `clip_` — новый для проекта, не пересекается с другими модулями.

Все нужные `web-sys`-фичи (`HtmlCanvasElement`, `CanvasRenderingContext2d`, `HtmlTextAreaElement`, `BlobPropertyBag`, `Url`, `Blob`, `DomException`) уже включены в `Cargo.toml`. Ничего добавлять не нужно.

## 32.2. Ссылки для детального изучения

- MDN, Clipboard API — https://developer.mozilla.org/en-US/docs/Web/API/Clipboard_API
- MDN, `Clipboard.writeText()` — https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/writeText
- MDN, `Clipboard.read()` — https://developer.mozilla.org/en-US/docs/Web/API/Clipboard/read
- MDN, `ClipboardItem` — https://developer.mozilla.org/en-US/docs/Web/API/ClipboardItem
- MDN, Permissions API — https://developer.mozilla.org/en-US/docs/Web/API/Permissions_API
- Spec, «Clipboard API and events» — https://w3c.github.io/clipboard-apis/

## 32.3. Почему Clipboard API — не «просто текст в буфере»

Первое, что приходит в голову при слове «буфер обмена» — это строка. Скопировал текст, вставил текст. Но Async Clipboard API даёт больше:

**Два уровня API.** `writeText` / `readText` — простой уровень для текста. `write` / `read` — общий уровень, работающий с `ClipboardItem[]`. `ClipboardItem` — это контейнер: один фрагмент может содержать несколько MIME-форматов одновременно.

**Один ClipboardItem — несколько представлений.** Когда вы копируете Markdown-фрагмент в Obsidian, приложение кладёт в буфер три версии: `text/plain` (без разметки), `text/html` (с тегами), `text/markdown` (с синтаксисом). Получатель сам выбирает подходящую. Текстовый редактор возьмёт plain, браузер — html, markdown-редактор — markdown.

**Разрешения.** Запись требует user gesture — клика. Чтение может потребовать явного разрешения `clipboard-read`. Без разрешения `readText` отклоняется с `NotAllowedError`.

**Бинарные данные.** `image/png` можно положить в буфер как `Blob`. Пользователь вставит его в графический редактор или чат — как будто скопировал из нативного приложения.

**Ограничения браузера.** Не все MIME-типы разрешены. Chrome принимает `text/plain`, `text/html`, `image/png`. `text/markdown` — нет. Один неподдерживаемый MIME отклоняет **всю** пачку.

Всё это делает Clipboard API интереснее, чем «прочитать строку из буфера».

## 32.4. Исходный код `src/clip_core.rs`

```rust
//! Операции с содержимым системного буфера обмена.
//!
//! Async Clipboard API даёт два уровня:
//!
//!   • `writeText` / `readText` — только текст. Простой API, работает
//!     без явных разрешений (для записи достаточно user gesture).
//!   • `write` / `read` — `ClipboardItem[]`. Каждый `ClipboardItem`
//!     содержит один или несколько MIME-форматов одновременно.
//!
//! ПРАВИЛО: `navigator.clipboard` требует secure context (HTTPS или
//! localhost). При отсутствии — явная ошибка, не паника.
//!
//! ПРАВИЛО: ошибки пробрасываем как `JsValue` без обёртки в строку.
//!
//! ПРАВИЛО: `ClipboardItem` — класс, вызывается через
//! `js_sys::Reflect::construct`, а не через `Function::call1`.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

// ============================================================
//                    ОШИБКИ
// ============================================================

/// Классифицированная ошибка Clipboard API.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ClipError {
    /// Нет user gesture, отклонено разрешение или неподдерживаемый MIME.
    NotAllowed,
    /// Нет разрешения `clipboard-read`.
    PermissionDenied,
    /// Secure context отсутствует.
    Security,
    /// Буфер пуст или формат не найден.
    NotFound,
    /// Данные слишком большие или неподдерживаемый формат.
    Data,
    Other(String),
}

impl ClipError {
    pub fn short(&self) -> &'static str {
        match self {
            ClipError::NotAllowed => "отказано: нет жеста или формат отвергнут",
            ClipError::PermissionDenied => "разрешение отклонено",
            ClipError::Security => "нужен HTTPS или localhost",
            ClipError::NotFound => "в буфере нет данных такого формата",
            ClipError::Data => "неподдерживаемый формат данных",
            ClipError::Other(_) => "ошибка буфера обмена",
        }
    }
    pub fn name(&self) -> &str {
        match self {
            ClipError::NotAllowed => "NotAllowedError",
            ClipError::PermissionDenied => "NotAllowedError",
            ClipError::Security => "SecurityError",
            ClipError::NotFound => "NotFoundError",
            ClipError::Data => "DataError",
            ClipError::Other(s) => s.as_str(),
        }
    }
}

pub fn clip_error_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    if let Some(s) = e.as_string() {
        return s;
    }
    js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| "unknown".to_string())
}

pub fn classify_clip_error(e: &JsValue) -> ClipError {
    match clip_error_name(e).as_str() {
        "NotAllowedError" => ClipError::NotAllowed,
        "SecurityError" => ClipError::Security,
        "NotFoundError" => ClipError::NotFound,
        "DataError" => ClipError::Data,
        other => ClipError::Other(other.to_string()),
    }
}

// ============================================================
//                    ДОСТУП К CLIPBOARD
// ============================================================

fn clipboard() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))
        .map_err(|e| JsValue::from_str(&format!("Reflect navigator: {e:?}")))?;
    let clip = js_sys::Reflect::get(&nav, &JsValue::from_str("clipboard"))
        .map_err(|e| JsValue::from_str(&format!("Reflect clipboard: {e:?}")))?;
    if clip.is_undefined() || clip.is_null() {
        return Err(JsValue::from_str(
            "navigator.clipboard недоступен (нужен HTTPS или localhost)",
        ));
    }
    Ok(clip)
}

/// true, если `navigator.clipboard` доступен.
pub fn is_available() -> bool {
    return clipboard().is_ok();
}

/// true, если браузер разрешает записывать этот MIME в `ClipboardItem`.
///
/// Использует статический `ClipboardItem.supports(mime)` (Chrome 121+).
/// Если метод недоступен — `true` (оптимистично: пусть попытка
/// провалится с осмысленной ошибкой).
pub fn supports_mime(mime: &str) -> bool {
    let global = js_sys::global();
    let ctor = match js_sys::Reflect::get(&global, &JsValue::from_str("ClipboardItem")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return true,
    };
    let supports = match js_sys::Reflect::get(&ctor, &JsValue::from_str("supports")) {
        Ok(v) if !v.is_undefined() => v,
        _ => return true,
    };
    let f: js_sys::Function = match supports.dyn_into() {
        Ok(f) => f,
        Err(_) => return true,
    };
    f.call1(&ctor, &JsValue::from_str(mime))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(true)
}

// ============================================================
//                    ТЕКСТ
// ============================================================

/// `clipboard.writeText(text)`.
pub async fn write_text(text: &str) -> Result<(), JsValue> {
    let clip = clipboard()?;
    let f = method(&clip, "writeText")?;
    let promise = f
        .call1(&clip, &JsValue::from_str(text))
        .map_err(|e| JsValue::from_str(&format!("writeText: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("writeText не вернул Promise"))?;
    JsFuture::from(promise).await?;
    Ok(())
}

/// `clipboard.readText()`. Пустой буфер → пустая строка.
pub async fn read_text() -> Result<String, JsValue> {
    let clip = clipboard()?;
    let f = method(&clip, "readText")?;
    let promise = f
        .call0(&clip)
        .map_err(|e| JsValue::from_str(&format!("readText: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("readText не вернул Promise"))?;
    let result = JsFuture::from(promise).await?;
    Ok(result.as_string().unwrap_or_default())
}

// ============================================================
//                    CLIPBOARD ITEM
// ============================================================

/// Создаёт `ClipboardItem` из одного MIME-формата и `Blob`.
fn make_item(mime: &str, blob: &web_sys::Blob) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("ClipboardItem"))
        .map_err(|e| JsValue::from_str(&format!("Reflect ClipboardItem: {e:?}")))?;
    if ctor.is_undefined() || ctor.is_null() {
        return Err(JsValue::from_str("ClipboardItem недоступен"));
    }
    let ctor: js_sys::Function = ctor
        .dyn_into()
        .map_err(|_| JsValue::from_str("ClipboardItem не конструктор"))?;

    let data = js_sys::Object::new();
    js_sys::Reflect::set(&data, &JsValue::from_str(mime), blob)
        .map_err(|e| JsValue::from_str(&format!("set mime: {e:?}")))?;

    // ПРАВИЛО: `ClipboardItem` — класс, вызывается только через `new`.
    // `Function::call1` даёт TypeError: "constructor: 'new' is required".
    let args = js_sys::Array::new();
    args.push(&data);
    js_sys::Reflect::construct(&ctor, &args)
        .map_err(|e| JsValue::from_str(&format!("new ClipboardItem: {e:?}")))
}

/// Создаёт `Blob` из строки с указанным MIME.
pub fn text_blob(text: &str, mime: &str) -> Result<web_sys::Blob, JsValue> {
    let parts = js_sys::Array::new();
    parts.push(&JsValue::from_str(text));
    let opts = web_sys::BlobPropertyBag::new();
    opts.set_type(mime);
    web_sys::Blob::new_with_str_sequence_and_options(&parts, &opts)
        .map_err(|e| JsValue::from_str(&format!("Blob::new: {e:?}")))
}

/// Создаёт `Blob` из байтов с указанным MIME.
pub fn bytes_blob(bytes: &[u8], mime: &str) -> Result<web_sys::Blob, JsValue> {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let opts = web_sys::BlobPropertyBag::new();
    opts.set_type(mime);
    web_sys::Blob::new_with_u8_array_sequence_and_options(&parts, &opts)
        .map_err(|e| JsValue::from_str(&format!("Blob::new bytes: {e:?}")))
}

/// Записывает один `ClipboardItem` с одним MIME-форматом.
pub async fn write_item(mime: &str, blob: &web_sys::Blob) -> Result<(), JsValue> {
    let item = make_item(mime, blob)?;
    let items = js_sys::Array::new();
    items.push(&item);
    write_items(&items).await
}

/// Записывает несколько MIME-форматов в одном `ClipboardItem`.
///
/// ПРАВИЛО: браузер пропускает только поддерживаемые MIME. Один
/// неподдерживаемый отклоняет **всю** пачку с `NotAllowedError`.
/// Фильтруйте через `supports_mime` до вызова.
pub async fn write_multi_formats(formats: &[(&str, &str)]) -> Result<(), JsValue> {
    let data = js_sys::Object::new();
    for (mime, text) in formats {
        let blob = text_blob(text, mime)?;
        js_sys::Reflect::set(&data, &JsValue::from_str(mime), &blob)
            .map_err(|e| JsValue::from_str(&format!("set {mime}: {e:?}")))?;
    }
    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("ClipboardItem"))
        .map_err(|e| JsValue::from_str(&format!("Reflect ClipboardItem: {e:?}")))?;
    let ctor: js_sys::Function = ctor
        .dyn_into()
        .map_err(|_| JsValue::from_str("ClipboardItem не конструктор"))?;

    let args = js_sys::Array::new();
    args.push(&data);
    let item = js_sys::Reflect::construct(&ctor, &args)
        .map_err(|e| JsValue::from_str(&format!("new ClipboardItem: {e:?}")))?;

    let items = js_sys::Array::new();
    items.push(&item);
    write_items(&items).await
}

/// Низкоуровневый `clipboard.write(items: ClipboardItem[])`.
async fn write_items(items: &js_sys::Array) -> Result<(), JsValue> {
    let clip = clipboard()?;
    let f = method(&clip, "write")?;
    let promise = f
        .call1(&clip, items)
        .map_err(|e| JsValue::from_str(&format!("write: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("write не вернул Promise"))?;
    JsFuture::from(promise).await?;
    Ok(())
}

/// Один элемент из буфера: MIME-тип и `Blob` с данными.
#[derive(Debug, Clone)]
pub struct ClipItem {
    pub mime: String,
    pub blob: web_sys::Blob,
}

/// Читает все элементы буфера.
pub async fn read_items() -> Result<Vec<ClipItem>, JsValue> {
    let clip = clipboard()?;
    let f = method(&clip, "read")?;
    let promise = f
        .call0(&clip)
        .map_err(|e| JsValue::from_str(&format!("read: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("read не вернул Promise"))?;
    let result = JsFuture::from(promise).await?;
    let arr: js_sys::Array = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("read вернул не Array"))?;

    let mut out = Vec::with_capacity(arr.length() as usize);
    for i in 0..arr.length() {
        let item = arr.get(i);
        let types_val = js_sys::Reflect::get(&item, &JsValue::from_str("types"))
            .map_err(|e| JsValue::from_str(&format!("item.types: {e:?}")))?;
        let types: js_sys::Array = types_val
            .dyn_into()
            .map_err(|_| JsValue::from_str("types не Array"))?;
        for j in 0..types.length() {
            let mime = types.get(j).as_string().unwrap_or_default();
            if let Ok(blob) = get_item_type(&item, &mime).await {
                out.push(ClipItem { mime, blob });
            }
        }
    }
    Ok(out)
}

async fn get_item_type(item: &JsValue, mime: &str) -> Result<web_sys::Blob, JsValue> {
    let f = method(item, "getType")?;
    let promise = f
        .call1(item, &JsValue::from_str(mime))
        .map_err(|e| JsValue::from_str(&format!("getType: {e:?}")))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| JsValue::from_str("getType не вернул Promise"))?;
    let blob = JsFuture::from(promise).await?;
    blob.dyn_into::<web_sys::Blob>()
        .map_err(|_| JsValue::from_str("getType вернул не Blob"))
}

/// Читает `Blob` как текст.
pub async fn blob_to_text(blob: &web_sys::Blob) -> Result<String, JsValue> {
    let promise = blob.text();
    let result = JsFuture::from(promise).await?;
    Ok(result.as_string().unwrap_or_default())
}

/// Читает `Blob` как байты.
pub async fn blob_to_bytes(blob: &web_sys::Blob) -> Result<Vec<u8>, JsValue> {
    let promise = blob.array_buffer();
    let result = JsFuture::from(promise).await?;
    let buf: js_sys::ArrayBuffer = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("array_buffer вернул не ArrayBuffer"))?;
    Ok(js_sys::Uint8Array::new(&buf).to_vec())
}

// ============================================================
//                    CF_HTML
// ============================================================

/// Извлекает содержимое из CF_HTML-обёртки, если она есть.
///
/// Chrome оборачивает `text/html` в Clipboard Format (стандарт MS Office):
///   `<html><body><!--StartFragment--><реальный HTML><!--EndFragment--></body></html>`
///
/// Если маркеры есть — берём фрагмент между ними. Иначе — как есть.
pub fn strip_cf_html_wrapper(s: &str) -> &str {
    const START: &str = "<!--StartFragment-->";
    const END: &str = "<!--EndFragment-->";
    if let Some(i) = s.find(START) {
        let after = &s[i + START.len()..];
        if let Some(j) = after.find(END) {
            return after[..j].trim();
        }
    }
    s
}

// ============================================================
//                    ОШИБКИ JS
// ============================================================

/// Обрезает произвольную строку до первой строки и 120 символов.
///
/// Для JS-ошибок это спасает от попадания stack trace в UI.
/// `TypeError.toString()` включает весь стек — 1000+ символов,
/// которые бесполезны пользователю.
pub fn clamp_first_line(s: &str) -> String {
    let first = s.lines().next().unwrap_or("").trim();
    let mut out: String = first.chars().take(120).collect();
    if first.chars().count() > 120 {
        out.push('…');
    }
    return out;
}

/// Извлекает читаемое сообщение из `JsValue`-ошибки.
///
/// Порядок попыток:
///   1. `DomException` — через `name()` + `message()`.
///   2. Примитив — через `as_string()`.
///   3. JS `Error`-объект — через `Reflect::get("name"/"message")`.
///   4. Fallback — `toString()` через `Reflect`.
///   5. Отладочный `Debug` — в крайнем случае.
///
/// Результат всегда прогнан через `clamp_first_line`.
pub fn js_error_message(e: &JsValue) -> String {
    // 1. DomException — самый частый случай.
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        let msg = dx.message();
        return format!("{}: {}", dx.name(), clamp_first_line(&msg));
    }

    // 2. Примитив (строка).
    if let Some(s) = e.as_string() {
        return clamp_first_line(&s);
    }

    // 3. JS Error-объект: name + message.
    if e.is_object() {
        let name = js_sys::Reflect::get(e, &JsValue::from_str("name"))
            .ok()
            .and_then(|v| v.as_string());
        let message = js_sys::Reflect::get(e, &JsValue::from_str("message"))
            .ok()
            .and_then(|v| v.as_string());

        match (name, message) {
            (Some(n), Some(m)) => {
                return format!("{n}: {}", clamp_first_line(&m));
            }
            (Some(n), None) => {
                return n;
            }
            _ => {}
        }
    }

    // 4. Fallback через .toString() — обрезаем сразу.
    if let Ok(f) = js_sys::Reflect::get(e, &JsValue::from_str("toString"))
        .and_then(|v| v.dyn_into::<js_sys::Function>())
    {
        if let Ok(s) = f.call0(e) {
            if let Some(s) = s.as_string() {
                if !s.is_empty() {
                    return clamp_first_line(&s);
                }
            }
        }
    }

    // 5. Крайний случай.
    return format!("{e:?}")
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn method(obj: &JsValue, name: &str) -> Result<js_sys::Function, JsValue> {
    js_sys::Reflect::get(obj, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("Reflect {name}: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{name} не функция")))
}
```

## 32.5. Исходный код `src/clip_perms.rs`

```rust
//! Разрешения Clipboard API через Permissions API.
//!
//! Clipboard — одно из немногих API, где разрешения запрашиваются
//! **автоматически** при первом вызове `read`/`write`, а не через
//! явный `request()`. Однако `navigator.permissions.query()` позволяет
//! узнать текущее состояние до вызова:
//!
//!   • `granted` — разрешение выдано, операция пройдёт без диалога.
//!   • `prompt` — браузер спросит при первом вызове.
//!   • `denied` — операция отклонена (пользователь или политика).
//!
//! ПРАВИЛО: `Permissions.query()` возвращает `Promise<PermissionStatus>`.
//!
//! ПРАВИЛО: не маскируйте ошибку `query()` под `Unknown`. Если браузер
//! не поддерживает имя разрешения — покажите причину, иначе UI
//! превращается в «неизвестно, и непонятно почему».
//!
//! ПРАВИЛО: Firefox не поддерживает `clipboard-read` / `clipboard-write`
//! в `Permissions API` — `query()` бросает `TypeError`. Это не ошибка
//! пользователя, а ограничение браузера. `readText` / `writeText` при
//! этом работают через native prompt.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

/// Состояние разрешения.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum PermState {
    Granted,
    Prompt,
    Denied,
    Unknown,
}

impl PermState {
    pub fn from_str(s: &str) -> Self {
        match s {
            "granted" => PermState::Granted,
            "prompt" => PermState::Prompt,
            "denied" => PermState::Denied,
            _ => PermState::Unknown,
        }
    }
    pub fn as_str(&self) -> &'static str {
        match self {
            PermState::Granted => "granted",
            PermState::Prompt => "prompt",
            PermState::Denied => "denied",
            PermState::Unknown => "unknown",
        }
    }
    pub fn short(&self) -> &'static str {
        match self {
            PermState::Granted => "разрешено",
            PermState::Prompt => "будет запрошено",
            PermState::Denied => "запрещено",
            PermState::Unknown => "неизвестно",
        }
    }
}

/// Результат запроса разрешения: состояние или причина отказа.
#[derive(Debug, Clone)]
pub enum QueryOutcome {
    State(PermState),
    /// `Permissions.query()` бросил исключение.
    Failed(String),
}

impl QueryOutcome {
    pub fn state_or_unknown(&self) -> PermState {
        match self {
            QueryOutcome::State(s) => *s,
            QueryOutcome::Failed(_) => PermState::Unknown,
        }
    }
    pub fn reason(&self) -> Option<&str> {
        match self {
            QueryOutcome::State(_) => None,
            QueryOutcome::Failed(s) => Some(s.as_str()),
        }
    }
}

/// true, если Permissions API доступен.
pub fn is_available() -> bool {
    return permissions().is_ok();
}

fn permissions() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))
        .map_err(|e| JsValue::from_str(&format!("Reflect navigator: {e:?}")))?;
    let perms = js_sys::Reflect::get(&nav, &JsValue::from_str("permissions"))
        .map_err(|e| JsValue::from_str(&format!("Reflect permissions: {e:?}")))?;
    if perms.is_undefined() || perms.is_null() {
        return Err(JsValue::from_str("Permissions API недоступен"));
    }
    Ok(perms)
}

/// Запрашивает текущее состояние разрешения. Маскирует ошибку под `Unknown`.
pub async fn query(perm_name: &str) -> Result<PermState, JsValue> {
    match query_inner(perm_name).await {
        Ok(s) => Ok(s),
        Err(msg) => Err(JsValue::from_str(&msg)),
    }
}

/// Строгий вариант: не маскирует ошибку под `Unknown`.
pub async fn query_strict(perm_name: &str) -> QueryOutcome {
    match query_inner(perm_name).await {
        Ok(s) => QueryOutcome::State(s),
        Err(msg) => QueryOutcome::Failed(msg),
    }
}

async fn query_inner(perm_name: &str) -> Result<PermState, String> {
    let perms = permissions()
        .map_err(|e| crate::clip_core::js_error_message(&e))?;
    let query_fn = method(&perms, "query")
        .map_err(|e| crate::clip_core::js_error_message(&e))?;

    let desc = js_sys::Object::new();
    js_sys::Reflect::set(&desc, &JsValue::from_str("name"), &JsValue::from_str(perm_name))
        .map_err(|e| format!("set name: {e:?}"))?;

    let promise = query_fn
        .call1(&perms, &desc)
        .map_err(|e| format!("query: {e:?}"))?;
    let promise: js_sys::Promise = promise
        .dyn_into()
        .map_err(|_| "query не вернул Promise".to_string())?;

    let status = JsFuture::from(promise).await.map_err(|e| {
        let msg = crate::clip_core::js_error_message(&e);
        // Firefox бросает TypeError: "'clipboard-read' (value of 'name'
        // member of PermissionDescriptor) is not a valid value for
        // enumeration PermissionName." Это не ошибка пользователя —
        // просто браузер не знает такого разрешения. Даём короткую
        // формулировку вместо пятисотсимвольного стектрейса.
        if msg.contains("not a valid value")
            || msg.contains("not a valid permission")
        {
            return format!("браузер не поддерживает '{perm_name}' в Permissions API");
        }
        return msg;
    })?;

    let state = js_sys::Reflect::get(&status, &JsValue::from_str("state"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| "unknown".to_string());
    Ok(PermState::from_str(&state))
}

/// Запрашивает оба разрешения одним вызовом. Маскирует ошибки.
pub async fn query_both() -> (PermState, PermState) {
    let read = query("clipboard-read").await.unwrap_or(PermState::Unknown);
    let write = query("clipboard-write").await.unwrap_or(PermState::Unknown);
    (read, write)
}

/// Снимок состояния Clipboard-разрешений.
#[derive(Debug, Clone)]
pub struct ClipPermsInfo {
    pub api_available: bool,
    pub perms_api_available: bool,
    pub read: PermState,
    pub write: PermState,
}

/// Возвращает снимок разрешений. Никогда не паникует.
pub async fn probe() -> ClipPermsInfo {
    let api_available = crate::clip_core::is_available();
    let perms_available = is_available();

    if !perms_available {
        return ClipPermsInfo {
            api_available,
            perms_api_available: false,
            read: PermState::Unknown,
            write: PermState::Unknown,
        };
    }
    let (read, write) = query_both().await;
    ClipPermsInfo {
        api_available,
        perms_api_available: true,
        read,
        write,
    }
}

/// Расширенный снимок: содержит причины, если запрос провалился.
#[derive(Debug, Clone)]
pub struct ClipPermsProbe {
    pub api_available: bool,
    pub perms_api_available: bool,
    pub read: QueryOutcome,
    pub write: QueryOutcome,
}

/// Строгий probe: не маскирует ошибки под `Unknown`.
pub async fn probe_strict() -> ClipPermsProbe {
    let api_available = crate::clip_core::is_available();
    if !is_available() {
        return ClipPermsProbe {
            api_available,
            perms_api_available: false,
            read: QueryOutcome::Failed("Permissions API недоступен".into()),
            write: QueryOutcome::Failed("Permissions API недоступен".into()),
        };
    }
    let read = query_strict("clipboard-read").await;
    let write = query_strict("clipboard-write").await;
    ClipPermsProbe {
        api_available,
        perms_api_available: true,
        read,
        write,
    }
}

fn method(obj: &JsValue, name: &str) -> Result<js_sys::Function, JsValue> {
    js_sys::Reflect::get(obj, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("Reflect {name}: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{name} не функция")))
}
```

## 32.6. Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod clip_core;
pub mod clip_perms;
```

## 32.7. Тесты

### `tests/wasm_clip_core.rs`

```rust
//! Wasm-тесты `clip_core`: создание Blob, классификация ошибок,
//! доступность API.
//!
//! ПРАВИЛО: headless-тесты не могут эмулировать user gesture.
//! Тесты, требующие интерактивного разрешения, помечены `#[ignore]`.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::clip_core::{self, ClipError};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn classify_not_allowed() {
    let e = web_sys::DomException::new_with_message_and_name(
        "denied",
        "NotAllowedError",
    )
    .expect("make DomException");
    let js: JsValue = e.into();
    assert_eq!(clip_core::classify_clip_error(&js), ClipError::NotAllowed);
}

#[wasm_bindgen_test]
fn classify_security() {
    let e = web_sys::DomException::new_with_message_and_name(
        "insecure",
        "SecurityError",
    )
    .unwrap();
    let js: JsValue = e.into();
    assert_eq!(clip_core::classify_clip_error(&js), ClipError::Security);
}

#[wasm_bindgen_test]
fn classify_unknown() {
    let e = JsValue::from_str("strange");
    let kind = clip_core::classify_clip_error(&e);
    assert_eq!(kind, ClipError::Other("strange".to_string()));
}

#[wasm_bindgen_test]
fn text_blob_has_correct_mime() {
    let blob = clip_core::text_blob("hello", "text/plain").expect("text_blob");
    assert_eq!(blob.type_(), "text/plain");
    assert_eq!(blob.size() as u32, 5);
}

#[wasm_bindgen_test]
fn bytes_blob_has_correct_mime() {
    let bytes: [u8; 4] = [0x89, 0x50, 0x4E, 0x47];
    let blob = clip_core::bytes_blob(&bytes, "image/png").expect("bytes_blob");
    assert_eq!(blob.type_(), "image/png");
    assert_eq!(blob.size() as u32, 4);
}

#[wasm_bindgen_test]
async fn blob_to_text_roundtrip() {
    let blob = clip_core::text_blob("привет мир", "text/plain").unwrap();
    let text = clip_core::blob_to_text(&blob).await.unwrap();
    assert_eq!(text, "привет мир");
}

#[wasm_bindgen_test]
async fn blob_to_bytes_roundtrip() {
    let bytes: Vec<u8> = (0..32u8).collect();
    let blob = clip_core::bytes_blob(&bytes, "application/octet-stream").unwrap();
    let back = clip_core::blob_to_bytes(&blob).await.unwrap();
    assert_eq!(back, bytes);
}

#[wasm_bindgen_test]
fn clipboard_is_available_in_headless() {
    assert!(clip_core::is_available(), "clipboard недоступен в headless");
}

#[wasm_bindgen_test]
fn clamp_first_line_short() {
    assert_eq!(clip_core::clamp_first_line("hello"), "hello");
}

#[wasm_bindgen_test]
fn clamp_first_line_multiline() {
    assert_eq!(clip_core::clamp_first_line("first\nsecond"), "first");
}

#[wasm_bindgen_test]
fn strip_cf_html_wrapper_removes_markers() {
    let wrapped = "<html><body><!--StartFragment--><b>hi</b><!--EndFragment--></body></html>";
    assert_eq!(clip_core::strip_cf_html_wrapper(wrapped), "<b>hi</b>");
}

#[wasm_bindgen_test]
fn strip_cf_html_wrapper_leaves_plain() {
    let plain = "<b>hi</b>";
    assert_eq!(clip_core::strip_cf_html_wrapper(plain), "<b>hi</b>");
}

#[wasm_bindgen_test]
fn js_error_message_dom_exception() {
    let e = web_sys::DomException::new_with_message_and_name(
        "denied",
        "NotAllowedError",
    )
    .unwrap();
    let js: JsValue = e.into();
    let msg = clip_core::js_error_message(&js);
    assert!(msg.contains("NotAllowedError"));
    assert!(msg.contains("denied"));
}

#[wasm_bindgen_test]
fn js_error_message_string() {
    let js = JsValue::from_str("plain string");
    assert_eq!(clip_core::js_error_message(&js), "plain string");
}

#[wasm_bindgen_test]
#[ignore = "требует clipboard-write permission в headless"]
async fn write_read_text_roundtrip() {
    clip_core::write_text("test-clipboard-123").await.expect("writeText");
    let back = clip_core::read_text().await.expect("readText");
    assert_eq!(back, "test-clipboard-123");
}
```

### `tests/wasm_clip_perms.rs`

```rust
//! Wasm-тесты `clip_perms`: PermState, доступность, query.
//!
//! ПРАВИЛО: Permissions API может быть недоступен для конкретного
//! имени разрешения (Firefox не поддерживает clipboard-*). Тесты
//! не предполагают конкретного ответа — только что функция не паникует.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::clip_perms::{self, PermState, QueryOutcome};

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn perm_state_from_str() {
    assert_eq!(PermState::from_str("granted"), PermState::Granted);
    assert_eq!(PermState::from_str("prompt"), PermState::Prompt);
    assert_eq!(PermState::from_str("denied"), PermState::Denied);
    assert_eq!(PermState::from_str("garbage"), PermState::Unknown);
}

#[wasm_bindgen_test]
fn perm_state_as_str_roundtrip() {
    for s in [PermState::Granted, PermState::Prompt, PermState::Denied, PermState::Unknown] {
        assert_eq!(PermState::from_str(s.as_str()), s);
    }
}

#[wasm_bindgen_test]
fn perm_state_shorts() {
    assert_eq!(PermState::Granted.short(), "разрешено");
    assert_eq!(PermState::Prompt.short(), "будет запрошено");
    assert_eq!(PermState::Denied.short(), "запрещено");
}

#[wasm_bindgen_test]
fn perms_api_is_available() {
    assert!(clip_perms::is_available());
}

#[wasm_bindgen_test]
async fn query_clipboard_write_returns_valid() {
    let state = clip_perms::query_strict("clipboard-write").await;
    match state {
        QueryOutcome::State(s) => {
            assert!(matches!(
                s,
                PermState::Granted | PermState::Prompt | PermState::Denied | PermState::Unknown
            ));
        }
        QueryOutcome::Failed(msg) => {
            // Firefox: "браузер не поддерживает ..."
            assert!(!msg.is_empty(), "причина не должна быть пустой");
            // Убеждаемся, что это короткая строка, а не стек.
            assert!(msg.chars().count() < 200, "сообщение должно быть коротким");
        }
    }
}

#[wasm_bindgen_test]
async fn probe_returns_info() {
    let info = clip_perms::probe_strict().await;
    assert!(info.api_available, "clipboard API должен быть доступен");
    let _ = info.read.reason();
    let _ = info.write.reason();
}
```

## 32.8. Команды запуска

```bash
# Проверка типов
cargo check
cargo check --target wasm32-unknown-unknown

# Тесты ядра
wasm-pack test --headless --chrome --test wasm_clip_core
wasm-pack test --headless --firefox --test wasm_clip_core

# Тесты разрешений
wasm-pack test --headless --chrome --test wasm_clip_perms
wasm-pack test --headless --firefox --test wasm_clip_perms

# Группа по подстроке
wasm-pack test --headless --chrome --test wasm_clip_core -- classify_

# С игнорируемыми (требует разрешений)
wasm-pack test --headless --chrome --test wasm_clip_core -- --ignored
```

## 32.9. Демо — `src/app.rs`

Демо — **три колонки на одном экране**:

```
┌──────────────────────┬────────────────────────┬──────────────────┐
│ 1. Текст             │ 2. Изображение (PNG)   │ Журнал операций  │
│    writeText/readText│    ClipboardItem       │ (последние 8)    │
│    roundtrip-проверка│    canvas → PNG        │                  │
│                      │                        │ Когда что       │
│ 4. Multi-format      │ 3. Чтение из буфера    │ использовать     │
│    plain/html/md     │    clipboard.read()    │                  │
│    support-бейджи    │    CF_HTML обработка   │                  │
└──────────────────────┴────────────────────────┴──────────────────┘
```

**Четыре сценария для проверки:**

1. **Roundtrip-проверка текста.** Нажать «Копировать и проверить» — увидеть, что содержимое совпадает. Нажать ещё раз — увидеть, что совпадает (буфер перезаписан). Если открыть два таба и копировать в одном, проверять в другом — увидеть несоответствие.

2. **PNG.** Выбрать размер, нажать «Копировать PNG» — в превью появится изображение с актуальной цифрой в центре. Вставить в чат — картинка вставится.

3. **Multi-format.** Живое превью трёх форматов. Кнопка «Копировать как Markdown» — Chrome отбросит `text/markdown`, но сохранит plain и html. Кнопка «Что в буфере?» подтвердит это.

4. **Разрешения.** В Chrome — бейджи `prompt` или `granted`. В Firefox — `н/д` с tooltip «браузер не поддерживает 'clipboard-read' в Permissions API». В логе — короткая строка вместо стектрейса.

**Ключевые UI-решения:**

- **Радио-кнопки и прямые кнопки.** Никаких выпадающих меню — всё видно сразу.
- **Live-превью форматов** в панели Markdown — три блока показывают, что уйдёт в буфер до нажатия.
- **Бейджи поддержки MIME** (`✓ в буфер` / `× не поддерживается`) — `ClipboardItem.supports` вызывается один раз при старте.
- **CF_HTML примечание** под html-блоком в превью чтения — если браузер обернул фрагмент, показываем примечание.
- **ObjectURL tracking** в `App.urls` — отзываются перед каждой новой отрисовкой превью.

**Проверки корректности:**

- Все `format!` в tail-позиции — через `format_str(format_args!(...))` или явный `return`.
- HTML с ≥2 `"` — через `push_str`.
- Строки для `write_multi_formats` строятся **внутри** `spawn_local` — иначе ссылки протухают до старта future.
- `ClipboardItem` — через `Reflect::construct`, не `call1`.
- `scroll_height() as f64` — явный каст.

## 32.10. Практические находки

Этот раздел — самое ценное в главе. Каждая находка стоила итерации отладки.

### 1. `ClipboardItem` — это класс, а не функция

Первая попытка:
```rust
ctor.call1(&JsValue::NULL, &data)   // TypeError: constructor: 'new' is required
```

Конструктор в JS вызывается только через `new`. В Rust это `js_sys::Reflect::construct(&ctor, &args_array)`.

**Правило.** Для любого JS-класса (`ClipboardItem`, `Blob`, `File`, `ImageData`, `OffscreenCanvas`) — `Reflect::construct`. Симптом ошибки в консоли: `TypeError: X constructor: 'new' is required`.

### 2. `text/markdown` не поддерживается в Chrome

Первая версия отправляла три формата: plain, html, markdown. Chrome отвечал:
```
NotAllowedError: ...user denied permission
```

Но user gesture был — это была кнопка. Причина — не жест, а `text/markdown`: Chrome не знает такого MIME для `ClipboardItem`. **Один неподдерживаемый MIME отклоняет всю пачку.**

**Решение.** `ClipboardItem.supports(mime)` — статический метод (Chrome 121+). Вызывать до `write`. Фильтровать. В логе писать, что пропущено.

### 3. Firefox не поддерживает `clipboard-read`/`clipboard-write` в Permissions API

Firefox бросает:
```
TypeError: 'clipboard-read' (value of 'name' member of PermissionDescriptor)
is not a valid value for enumeration PermissionName.
[~1500 символов stack trace]
```

**Решение.** Детектировать `not a valid value` в сообщении и возвращать короткое `браузер не поддерживает 'clipboard-read' в Permissions API`. `readText`/`writeText` при этом работают — Firefox покажет native prompt.

### 4. `JsValue` в UI — stack trace

`format!("{e:?}")` для `JsValue`, содержащего JS `TypeError`, даёт полный стек. Это бесполезно пользователю.

**Решение.** `js_error_message` — универсальный экстрактор: `DomException` → `name + message`, примитивы → строка, `Error`-объект → `Reflect::get("name"/"message")`, fallback → `toString()` с обрезкой. Плюс `clamp_first_line` — 120 символов максимум.

### 5. Chrome оборачивает `text/html` в CF_HTML

Пишем `<h2>Title</h2>`, читаем:
```
<html><body><!--StartFragment--><h2>Title</h2><!--EndFragment--></body></html>
```

**Решение.** `strip_cf_html_wrapper` — извлекает фрагмент между маркерами. Под превью — примечание, чтобы пользователь знал о нормализации.

### 6. ObjectURL нужно отзывать

Каждый `URL.createObjectURL` держит `Blob` в памяти до `revokeObjectURL`. В превью, которые перерисовываются часто, это утечка.

**Решение.** `App.urls: Rc<RefCell<Vec<String>>>` — трекер. Перед каждой новой отрисовкой — `revoke_urls`.

### 7. Строки для multi-format нельзя строить до `spawn_local`

Первая версия:
```rust
let (plain, html, markdown) = build_three_formats(&title, &body);
spawn_local(async move {
    let formats = vec![("text/plain", plain.as_str()), ...];  // ← borrow against 'static
    write_multi_formats(&formats).await
});
```

Ошибка: `borrowed value does not live long enough`. `spawn_local` требует `Future + 'static`, а ссылки указывают на стек внешнего замыкания.

**Решение.** Строить строки **внутри** `async`-блока. Владение принадлежит future.

### 8. `Function::call` для конструктора не работает

См. пункт 1. Но стоит запомнить симптом: `TypeError: ... constructor: 'new' is required`. Это не ошибка типа аргументов — это ошибка способа вызова.

### 9. `scroll_height()` возвращает `i32`

`web_sys::Element::scroll_height() -> i32`, `set_scroll_top(f64)`. Нужен явный каст `. as f64`.

### 10. `format!` в tail-позиции функции

```rust
// Неправильно — rust-analyzer теряет тип:
fn short_msg(e: &JsValue) -> String {
    format!("{e:?}")   // expected String, found ()
}

// Правильно — явный return:
fn short_msg(e: &JsValue) -> String {
    return format!("{e:?}")
}
```

Правило распространяется на `if let ... else`, `match`-arms, тела циклов.

### 11. Мёртвый код после рефакторинга

После замены `short_msg` на `js_error_message` функция `short_msg` осталась в файле — предупреждение `dead_code`. **Удалять сразу.** Правило проекта: `cargo check` после каждого рефакторинга.

## 32.11. Правила проекта, закреплённые в главе

1. **JS-классы вызываются через `Reflect::construct`, не через `Function::call`.** `ClipboardItem`, `Blob`, `File` — все они. Симптом: `TypeError: X constructor: 'new' is required`.
2. **`ClipboardItem.supports(mime)` до записи.** Один неподдерживаемый MIME отклоняет всю пачку. Фильтруйте через `supports_mime`.
3. **`JsValue`-ошибки не показываем через `{:?}`.** Полный stack trace бесполезен. `js_error_message` + `clamp_first_line` — короткая понятная строка.
4. **`Permissions.query()` для clipboard-* не везде работает.** Firefox бросает `TypeError`. Не маскируйте под `Unknown` — покажите причину.
5. **Chrome оборачивает `text/html` в CF_HTML.** `strip_cf_html_wrapper` — обязательно перед показом пользователю.
6. **ObjectURL трекаются и отзываются.** Утечка `Blob` в памяти вкладки — реальна.
7. **Строки для `write_multi_formats` строятся внутри `spawn_local`.** Иначе lifetime-ошибка.
8. **`format!` в tail-позиции — через `format_str(format_args!(...))` или явный `return`.**
9. **HTML с ≥2 `"` — через `push_str`.** Парсер `format_args!` путается.
10. **`dead_code` убирается сразу.** Мёртвая функция в модуле вводит читателя в заблуждение.
11. **User gesture обязателен для записи.** `writeText` без клика → `NotAllowedError`.
12. **`navigator.clipboard` требует secure context.** HTTPS или localhost.

**app.rs**
```
//! Демо Главы 32: Clipboard API.
//!
//! Четыре эксперимента:
//!   1. Текст — writeText / readText, включая roundtrip-проверку.
//!   2. PNG — write ClipboardItem с image/png.
//!   3. Чтение — clipboard.read(), MIME-чипы, снятие CF_HTML-обёртки.
//!   4. Multi-format — ClipboardItem с text/plain + text/html + text/markdown,
//!      с live-превью и фильтрацией через ClipboardItem.supports.
//!
//! ПРАВИЛО: HTML внутри `format_args!` с ≥2 `"` — через push_str.
//! ПРАВИЛО: ObjectURL трекаются в App.urls и отзываются перед новой
//! отрисовкой.
//! ПРАВИЛО: строки для `write_multi_formats` строятся ВНУТРИ async-блока,
//! иначе ссылки протухают до старта future.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;

use crate::clip_core;
use crate::clip_perms::{self, PermState};
use crate::format_str::format_str;

const HTML: &str = r#"
<style>
  :root {
    --cp-info: #3498db; --cp-ok: #27ae60; --cp-err: #c0392b; --cp-warn: #e67e22;
    --cp-text: #222; --cp-muted: #666;
    --cp-bg: #fff; --cp-soft: #fafafa; --cp-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --cp-text: #e6e6e6; --cp-muted: #a0a0a0;
      --cp-bg: #1e1e1e; --cp-soft: #262626; --cp-border: #3a3a3a;
    }
    body { background: #121212; }
    .cp-anno { background: #1a2233; color: #9fc4ff; }
    .cp-anno code { background: rgba(52,152,219,0.2); }
    textarea, input { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .cp-log-entry { background: #1e1e1e; }
    .cp-perm-badge.unknown { background: #2a2a2a; color: #888; }
    .cp-md-fmt pre { background: #151515; }
    .cp-preview { background: #1a1a1a; }
  }
  html, body { width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important; overflow-x: hidden; }
  body {
    display: block !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--cp-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app {
    display: block !important; width: 100% !important; max-width: 100% !important;
    margin: 0 !important; padding: 12px 16px 24px !important;
    box-sizing: border-box !important;
  }
  #app, #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .cp-subtitle { color: var(--cp-muted); font-size: 12px; margin: 0 0 12px; }
  .cp-subtitle code { background: rgba(128,128,128,0.15);
    padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .cp-subtitle a { color: var(--cp-info); }

  .cp-perms-bar {
    display: flex; align-items: center; gap: 12px; flex-wrap: wrap;
    padding: 8px 14px; margin: 0 0 12px;
    background: linear-gradient(135deg, #eaf4ff 0%, #f5f0ff 100%);
    border: 2px solid var(--cp-info); border-radius: 8px;
  }
  @media (prefers-color-scheme: dark) {
    .cp-perms-bar { background: linear-gradient(135deg, #1a2a3f 0%, #2a1e3f 100%); }
  }
  .cp-perm-item { display: flex; align-items: center; gap: 6px; }
  .cp-perm-label { font-size: 10px; text-transform: uppercase;
    letter-spacing: 0.7px; color: var(--cp-muted); }
  .cp-perm-badge {
    display: inline-block; padding: 2px 10px; border-radius: 10px;
    font-family: ui-monospace, monospace; font-size: 11px; font-weight: 700;
  }
  .cp-perm-badge.granted { background: #e8f8ed; color: #1e8449; }
  .cp-perm-badge.prompt { background: #fff3cd; color: #7d5a00; }
  .cp-perm-badge.denied { background: #fdedec; color: #922b21; }
  .cp-perm-badge.unknown { background: #f0f0f0; color: #888; }
  .cp-perms-actions { margin-left: auto; display: flex; gap: 8px; }

  .cp-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(0, 1.1fr) minmax(0, 1.1fr) minmax(0, 0.9fr);
  }
  @media (max-width: 1100px) { .cp-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px) { .cp-grid { grid-template-columns: 1fr; } }
  .cp-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .cp-panel {
    border: 1px solid var(--cp-border); border-radius: 8px;
    padding: 10px 12px; background: var(--cp-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0;
  }
  .cp-panel-title { font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .cp-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--cp-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }

  .cp-anno { background: #eef4ff; border-left: 3px solid var(--cp-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere; word-break: break-word; }
  .cp-anno code { background: rgba(52,152,219,0.12);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10px; }
  .cp-anno b { color: #0d3b66; }
  @media (prefers-color-scheme: dark) { .cp-anno b { color: #c0d8ff; } }

  .cp-form-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; min-width: 0; }
  textarea {
    width: 100%; min-height: 80px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--cp-bg); color: var(--cp-text);
    resize: vertical;
  }
  textarea:focus {
    outline: none; border-color: var(--cp-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }
  input[type="text"] {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--cp-bg); color: var(--cp-text);
    font-family: ui-monospace, monospace;
    flex: 1 1 100px; min-width: 0;
  }
  input[type="text"]:focus {
    outline: none; border-color: var(--cp-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--cp-soft); color: var(--cp-text);
    cursor: pointer; transition: background 0.15s;
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--cp-info); outline-offset: 2px; }
  button.cp-primary { background: #eaf4ff; border-color: var(--cp-info); color: #1f4e79; }
  button.cp-danger { background: #fdedec; border-color: var(--cp-err); color: #922b21; }
  button.cp-success { background: #e8f8ed; border-color: var(--cp-ok); color: #1e8449; }

  .cp-status {
    display: flex; align-items: center; gap: 6px;
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--cp-soft); border-left: 3px solid #ccc;
    color: var(--cp-muted);
    min-height: 22px; overflow-wrap: anywhere; word-break: break-word;
  }
  .cp-status.ok { background: #e8f8ed; border-color: var(--cp-ok); color: #1e8449; }
  .cp-status.err { background: #fdedec; border-color: var(--cp-err); color: #922b21; }
  .cp-status.busy { background: #fff9e6; border-color: var(--cp-warn); color: #7d5a00; }

  .cp-log {
    border: 1px solid var(--cp-border); border-radius: 5px;
    background: var(--cp-soft); max-height: 240px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .cp-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .cp-log-entry { display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent; }
  .cp-log-entry.ok { border-left-color: var(--cp-ok); }
  .cp-log-entry.err { border-left-color: var(--cp-err); }
  .cp-log-entry.info { border-left-color: var(--cp-info); }
  .cp-log-entry .t { color: var(--cp-muted); flex-shrink: 0; }
  .cp-log-entry .m { color: var(--cp-text); flex: 1; overflow-wrap: anywhere; }
  .cp-log-entry.err .m { color: #922b21; }
  .cp-log-counter {
    margin-left: auto; padding: 0 10px; border-radius: 10px;
    background: rgba(52,152,219,0.15); color: #1f4e79;
    font-family: ui-monospace, monospace; font-size: 11px; font-weight: 700;
  }
  @media (prefers-color-scheme: dark) {
    .cp-log-counter { background: rgba(52,152,219,0.25); color: #c0d8ff; }
  }

  .cp-preview {
    border: 1px dashed var(--cp-border); border-radius: 4px;
    padding: 8px; margin-top: 6px;
    background: var(--cp-soft);
    max-height: 240px; overflow-y: auto;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--cp-muted); white-space: pre-wrap;
  }
  .cp-preview img { max-width: 100%; max-height: 180px; border-radius: 4px; }

  .cp-md-formats { display: flex; flex-direction: column; gap: 6px; margin: 8px 0; }
  .cp-md-fmt {
    border: 1px solid var(--cp-border); border-radius: 5px;
    background: var(--cp-soft); overflow: hidden;
  }
  .cp-md-fmt-label {
    padding: 3px 10px; font-family: ui-monospace, monospace;
    font-size: 10px; font-weight: 700;
    background: rgba(52,152,219,0.1); color: #1f4e79;
    border-bottom: 1px solid var(--cp-border);
    display: flex; align-items: center; gap: 6px;
  }
  @media (prefers-color-scheme: dark) {
    .cp-md-fmt-label { background: rgba(52,152,219,0.15); color: #c0d8ff; }
  }
  .cp-md-fmt pre {
    margin: 0; padding: 6px 10px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--cp-text);
    background: #fbfbfb;
    white-space: pre-wrap; word-break: break-word;
    max-height: 100px; overflow-y: auto;
  }

  .cp-md-support {
    margin-left: auto; padding: 0 8px; border-radius: 8px;
    font-size: 10px; font-weight: 700;
  }
  .cp-md-support.ok { background: rgba(39,174,96,0.15); color: #1e8449; }
  .cp-md-support.no { background: rgba(192,57,43,0.15); color: #922b21; }
  .cp-md-support.unknown { background: rgba(128,128,128,0.15); color: #666; }
  @media (prefers-color-scheme: dark) {
    .cp-md-support.ok { background: rgba(39,174,96,0.25); color: #b8e0b8; }
    .cp-md-support.no { background: rgba(192,57,43,0.25); color: #e0b8b8; }
    .cp-md-support.unknown { background: rgba(128,128,128,0.25); color: #aaa; }
  }

  .cp-mime-chip {
    display: inline-block; padding: 1px 8px; border-radius: 10px;
    font-family: ui-monospace, monospace; font-size: 10px; font-weight: 700;
    margin-right: 6px;
  }
  .cp-mime-plain    { background: #eef4ff; color: #1f4e79; }
  .cp-mime-html     { background: #fff3cd; color: #7d5a00; }
  .cp-mime-markdown { background: #e8f8ed; color: #1e8449; }
  .cp-mime-png      { background: #e6f3ff; color: #1a5b8a; }
  .cp-mime-jpeg     { background: #f0e6ff; color: #5b2a8a; }
  .cp-mime-webp     { background: #e0f7fa; color: #006064; }
  .cp-mime-other    { background: #f0f0f0; color: #555; }
  @media (prefers-color-scheme: dark) {
    .cp-mime-plain    { background: #1a3a5a; color: #cfe0ff; }
    .cp-mime-html     { background: #4a3a1a; color: #ffe0a0; }
    .cp-mime-markdown { background: #1a3a1a; color: #b8e0b8; }
    .cp-mime-png      { background: #1a2a4a; color: #a0c8e8; }
    .cp-mime-jpeg     { background: #2a1a4a; color: #c8a0e8; }
    .cp-mime-webp     { background: #1a3a3a; color: #a0e8e0; }
    .cp-mime-other    { background: #2a2a2a; color: #aaa; }
  }
  .cp-read-item {
    padding: 6px 8px; border-bottom: 1px solid var(--cp-border);
    margin-bottom: 4px;
  }
  .cp-read-item:last-child { border-bottom: none; margin-bottom: 0; }
  .cp-read-item .cp-read-size { color: var(--cp-muted); font-size: 10px; }
  .cp-read-item pre {
    margin: 4px 0 0; white-space: pre-wrap; word-break: break-word;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .cp-read-note {
    font-size: 10px; color: var(--cp-muted); margin-top: 4px;
    font-style: italic;
  }
</style>

<h1>Глава 32: Clipboard API</h1>
<p class="cp-subtitle">
  Текст, бинарные данные, разрешения. Multi-format ClipboardItem.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/Clipboard_API" target="_blank" rel="noopener">MDN →</a>
</p>

<div class="cp-perms-bar">
  <div class="cp-perm-item">
    <span class="cp-perm-label">clipboard-read</span>
    <span class="cp-perm-badge unknown" id="cp-perm-read">—</span>
  </div>
  <div class="cp-perm-item">
    <span class="cp-perm-label">clipboard-write</span>
    <span class="cp-perm-badge unknown" id="cp-perm-write">—</span>
  </div>
  <div class="cp-perms-actions">
    <button id="cp-refresh-perms" title="Перечитать состояние разрешений">↻ Обновить разрешения</button>
  </div>
</div>

<div class="cp-grid">

  <!-- ============ Колонка 1 ============ -->
  <div class="cp-col">

    <div class="cp-panel">
      <h2 class="cp-panel-title">
        <span class="cp-num">1</span>Текст
      </h2>
      <div class="cp-anno">
        <b>writeText</b> — простой API для текста, требует user gesture.
        <b>readText</b> — чтение, может запросить разрешение <code>clipboard-read</code>.
        <b>Копировать и проверить</b> — write + read + сравнение: показывает, что буфер
        не мгновенный и общий для всех программ.
      </div>

      <div class="cp-form-row">
        <textarea id="cp-text-input" placeholder="Введите текст для копирования">Привет из Clipboard API!</textarea>
      </div>

      <div class="cp-form-row">
        <button id="cp-copy-text" class="cp-primary" title="clipboard.writeText">📋 Копировать</button>
        <button id="cp-copy-verify" class="cp-success" title="writeText → readText → сравнение">📋 Копировать и проверить</button>
        <button id="cp-paste-text" title="clipboard.readText">📥 Вставить</button>
        <button id="cp-clear-text" class="cp-danger" title="Очистить поле">× Очистить</button>
      </div>

      <div id="cp-text-status" class="cp-status">готово</div>
    </div>

    <div class="cp-panel">
      <h2 class="cp-panel-title">
        <span class="cp-num">4</span>Копировать как Markdown
      </h2>
      <div class="cp-anno">
        <b>Multi-format ClipboardItem.</b> Один фрагмент — три MIME-формата
        одновременно. Получатель сам выбирает подходящий. Но браузер
        пропустит только те MIME, которые поддерживает — <b>один неподдерживаемый
        отклоняет всю пачку</b>. Зелёный бейдж = формат уйдёт в буфер, красный = нет.
      </div>

      <div class="cp-form-row">
        <input type="text" id="cp-md-title" placeholder="Заголовок" value="Clipboard API">
      </div>
      <div class="cp-form-row">
        <textarea id="cp-md-body" placeholder="Текст в Markdown" style="min-height:56px">**Clipboard API** позволяет читать и записывать данные в системный буфер обмена.</textarea>
      </div>

      <div class="cp-md-formats">
        <div class="cp-md-fmt">
          <div class="cp-md-fmt-label">
            <span>text/plain</span>
            <span class="cp-md-support unknown" id="cp-md-plain-badge" title="проверяется">…</span>
          </div>
          <pre id="cp-md-plain">—</pre>
        </div>
        <div class="cp-md-fmt">
          <div class="cp-md-fmt-label">
            <span>text/html</span>
            <span class="cp-md-support unknown" id="cp-md-html-badge" title="проверяется">…</span>
          </div>
          <pre id="cp-md-html">—</pre>
        </div>
        <div class="cp-md-fmt">
          <div class="cp-md-fmt-label">
            <span>text/markdown</span>
            <span class="cp-md-support unknown" id="cp-md-markdown-badge" title="проверяется">…</span>
          </div>
          <pre id="cp-md-markdown">—</pre>
        </div>
      </div>

      <div class="cp-form-row">
        <button id="cp-copy-markdown" class="cp-success" title="Копировать один ClipboardItem с поддерживаемыми MIME-форматами">📋 Копировать как Markdown</button>
        <button id="cp-read-markdown" title="Прочитать буфер и посмотреть, что в нём">🔍 Что в буфере?</button>
      </div>

      <div id="cp-md-status" class="cp-status">готово</div>
    </div>

  </div>

  <!-- ============ Колонка 2 ============ -->
  <div class="cp-col">

    <div class="cp-panel">
      <h2 class="cp-panel-title">
        <span class="cp-num">2</span>Изображение (PNG)
      </h2>
      <div class="cp-anno">
        <b>ClipboardItem</b> с <code>image/png</code>. Браузер запишет
        изображение в системный буфер — его можно вставить в графический
        редактор или чат. PNG генерируется через <code>&lt;canvas&gt;</code>.
      </div>

      <div class="cp-form-row">
        <label style="font-size:11px;color:var(--cp-muted)">
          Размер:
          <select id="cp-img-size" style="font-size:12px;margin-left:4px">
            <option value="32">32×32</option>
            <option value="64" selected>64×64</option>
            <option value="128">128×128</option>
          </select>
        </label>
      </div>

      <div class="cp-form-row">
        <button id="cp-copy-image" class="cp-primary" title="Создать PNG и скопировать">📋 Копировать PNG</button>
      </div>

      <div id="cp-img-status" class="cp-status">готово</div>
      <div id="cp-img-preview" class="cp-preview" style="display:none"></div>
    </div>

    <div class="cp-panel">
      <h2 class="cp-panel-title">
        <span class="cp-num">3</span>Чтение из буфера
      </h2>
      <div class="cp-anno">
        <b>clipboard.read()</b> возвращает <code>ClipboardItem[]</code>.
        Каждый item содержит один или несколько MIME-типов. Chrome оборачивает
        <code>text/html</code> в CF_HTML-обёртку — её мы снимаем перед показом.
      </div>

      <div class="cp-form-row">
        <button id="cp-read-all" class="cp-primary" title="Прочитать все форматы">📥 Прочитать буфер</button>
      </div>

      <div id="cp-read-status" class="cp-status">готово</div>
      <div id="cp-read-preview" class="cp-preview" style="display:none"></div>
    </div>

  </div>

  <!-- ============ Колонка 3 ============ -->
  <div class="cp-col">

    <div class="cp-panel">
      <h2 class="cp-panel-title">
        Журнал операций
        <span class="cp-log-counter" id="cp-log-counter" title="счётчик всех событий за сессию">0</span>
      </h2>
      <div id="cp-log" class="cp-log">
        <div class="cp-log-empty">здесь появятся операции с буфером</div>
      </div>
    </div>

    <div class="cp-panel">
      <h2 class="cp-panel-title">Когда что использовать</h2>
      <table style="width:100%;border-collapse:collapse;font-size:11px">
        <thead>
          <tr>
            <th style="text-align:left;padding:4px 8px;border-bottom:1px solid var(--cp-border);color:var(--cp-muted)">Ситуация</th>
            <th style="text-align:left;padding:4px 8px;border-bottom:1px solid var(--cp-border);color:var(--cp-muted)">Метод</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--cp-border)">Простой текст</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--cp-border)"><code>writeText</code> / <code>readText</code></td>
          </tr>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--cp-border)">Изображение, HTML, Markdown</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--cp-border)"><code>write</code> + <code>ClipboardItem</code></td>
          </tr>
          <tr>
            <td style="padding:4px 8px;border-bottom:1px solid var(--cp-border)">Несколько форматов сразу</td>
            <td style="padding:4px 8px;border-bottom:1px solid var(--cp-border)">Один item, но проверьте <code>supports</code></td>
          </tr>
          <tr>
            <td style="padding:4px 8px">Проверка перед операцией</td>
            <td style="padding:4px 8px"><code>permissions.query</code></td>
          </tr>
        </tbody>
      </table>
    </div>

  </div>

</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info }

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

struct Ui {
    perm_read: web_sys::Element,
    perm_write: web_sys::Element,
    text_input: web_sys::HtmlTextAreaElement,
    text_status: web_sys::Element,
    md_title: web_sys::HtmlInputElement,
    md_body: web_sys::HtmlTextAreaElement,
    md_plain: web_sys::Element,
    md_html: web_sys::Element,
    md_markdown: web_sys::Element,
    md_plain_badge: web_sys::Element,
    md_html_badge: web_sys::Element,
    md_markdown_badge: web_sys::Element,
    md_status: web_sys::Element,
    img_size: web_sys::HtmlSelectElement,
    img_status: web_sys::Element,
    img_preview: web_sys::Element,
    read_status: web_sys::Element,
    read_preview: web_sys::Element,
    log: web_sys::Element,
    log_counter: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    total_events: Rc<RefCell<u32>>,
    /// Список созданных ObjectURL. Отзываются перед каждой новой
    /// отрисовкой превью — иначе утечка blob'ов в памяти вкладки.
    urls: Rc<RefCell<Vec<String>>>,
    ui: Ui,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        perm_read: get_el("cp-perm-read")?,
        perm_write: get_el("cp-perm-write")?,
        text_input: get_textarea("cp-text-input")?,
        text_status: get_el("cp-text-status")?,
        md_title: get_input("cp-md-title")?,
        md_body: get_textarea("cp-md-body")?,
        md_plain: get_el("cp-md-plain")?,
        md_html: get_el("cp-md-html")?,
        md_markdown: get_el("cp-md-markdown")?,
        md_plain_badge: get_el("cp-md-plain-badge")?,
        md_html_badge: get_el("cp-md-html-badge")?,
        md_markdown_badge: get_el("cp-md-markdown-badge")?,
        md_status: get_el("cp-md-status")?,
        img_size: get_select("cp-img-size")?,
        img_status: get_el("cp-img-status")?,
        img_preview: get_el("cp-img-preview")?,
        read_status: get_el("cp-read-status")?,
        read_preview: get_el("cp-read-preview")?,
        log: get_el("cp-log")?,
        log_counter: get_el("cp-log-counter")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        total_events: Rc::new(RefCell::new(0)),
        urls: Rc::new(RefCell::new(Vec::new())),
        ui,
    });

    app.render_log();
    refresh_md_formats(&app);
    update_support_badges(&app);

    // Стартовая проверка разрешений.
    {
        let app_c = app.clone();
        spawn_local(async move {
            refresh_perms(&app_c).await;
        });
    }

    // Обновить разрешения.
    {
        let app_c = app.clone();
        let btn = get_button("cp-refresh-perms")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            spawn_local(async move {
                refresh_perms(&app_in).await;
            });
        });
    }

    // Live-обновление превью форматов.
    for id in ["cp-md-title", "cp-md-body"] {
        let app_c = app.clone();
        let el = get_el(id)?;
        let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
            refresh_md_formats(&app_c);
        });
        el.add_event_listener_with_callback("input", cb.as_ref().unchecked_ref())?;
        cb.forget();
    }

    // Копировать текст.
    {
        let app_c = app.clone();
        let btn = get_button("cp-copy-text")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let text = app_in.ui.text_input.value();
            if text.is_empty() {
                set_status(&app_in.ui.text_status, "err", "нечего копировать");
                return;
            }
            set_status(&app_in.ui.text_status, "busy", "writeText…");
            spawn_local(async move {
                match clip_core::write_text(&text).await {
                    Ok(()) => {
                        let msg = format_str(format_args!(
                            "скопировано {} символов", text.chars().count()
                        ));
                        app_in.push_log(LogKind::Ok, msg.clone());
                        set_status(&app_in.ui.text_status, "ok", &msg);
                    }
                    Err(e) => {
                        let kind = clip_core::classify_clip_error(&e);
                        let msg = format_str(format_args!(
                            "writeText: {} — {}", kind.name(), kind.short()
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.text_status, "err", &msg);
                    }
                }
            });
        });
    }

    // Копировать и проверить.
    {
        let app_c = app.clone();
        let btn = get_button("cp-copy-verify")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let text = app_in.ui.text_input.value();
            if text.is_empty() {
                set_status(&app_in.ui.text_status, "err", "нечего копировать");
                return;
            }
            set_status(&app_in.ui.text_status, "busy", "write + read + verify…");
            spawn_local(async move {
                copy_and_verify(app_in, text).await;
            });
        });
    }

    // Вставить текст.
    {
        let app_c = app.clone();
        let btn = get_button("cp-paste-text")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            set_status(&app_in.ui.text_status, "busy", "readText…");
            spawn_local(async move {
                match clip_core::read_text().await {
                    Ok(text) => {
                        app_in.ui.text_input.set_value(&text);
                        let msg = format_str(format_args!(
                            "прочитано {} символов", text.chars().count()
                        ));
                        app_in.push_log(LogKind::Ok, msg.clone());
                        set_status(&app_in.ui.text_status, "ok", &msg);
                    }
                    Err(e) => {
                        let kind = clip_core::classify_clip_error(&e);
                        let msg = format_str(format_args!(
                            "readText: {} — {}", kind.name(), kind.short()
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.text_status, "err", &msg);
                    }
                }
            });
        });
    }

    // Очистить поле.
    {
        let app_c = app.clone();
        let btn = get_button("cp-clear-text")?;
        set_click(&btn, move |_| {
            app_c.ui.text_input.set_value("");
            set_status(&app_c.ui.text_status, "", "готово");
        });
    }

    // Копировать как Markdown.
    {
        let app_c = app.clone();
        let btn = get_button("cp-copy-markdown")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let title = app_in.ui.md_title.value();
            let body = app_in.ui.md_body.value();
            if title.is_empty() && body.is_empty() {
                set_status(&app_in.ui.md_status, "err", "нечего копировать");
                return;
            }

            set_status(&app_in.ui.md_status, "busy", "write форматов…");
            spawn_local(async move {
                // Строки строим ВНУТРИ async-блока: владение принадлежит
                // future, ссылки живут всё время await.
                let (plain, html, markdown) = build_three_formats(&title, &body);
                let candidates: [(&str, &str); 3] = [
                    ("text/plain", plain.as_str()),
                    ("text/html", html.as_str()),
                    ("text/markdown", markdown.as_str()),
                ];

                // Фильтр: один неподдерживаемый MIME отклоняет всю пачку.
                let mut formats: Vec<(&str, &str)> = Vec::new();
                let mut skipped: Vec<&str> = Vec::new();
                for (mime, text) in candidates {
                    if clip_core::supports_mime(mime) {
                        formats.push((mime, text));
                    } else {
                        skipped.push(mime);
                    }
                }

                if formats.is_empty() {
                    let msg = "ни один формат не поддерживается браузером".to_string();
                    app_in.push_log(LogKind::Err, msg.clone());
                    set_status(&app_in.ui.md_status, "err", &msg);
                    return;
                }

                match clip_core::write_multi_formats(&formats).await {
                    Ok(()) => {
                        let written: Vec<&str> = formats.iter().map(|(m, _)| *m).collect();
                        let msg = if skipped.is_empty() {
                            format_str(format_args!(
                                "скопировано: {}", written.join(", "),
                            ))
                        } else {
                            format_str(format_args!(
                                "скопировано: {}. Пропущено (не поддерживается браузером): {}",
                                written.join(", "), skipped.join(", "),
                            ))
                        };
                        app_in.push_log(LogKind::Ok, msg.clone());
                        set_status(&app_in.ui.md_status, "ok", &msg);
                    }
                    Err(e) => {
                        let kind = clip_core::classify_clip_error(&e);
                        let msg = format_str(format_args!(
                            "write multi: {} — {}", kind.name(), kind.short()
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.md_status, "err", &msg);
                    }
                }
            });
        });
    }

    // Прочитать буфер после multi-format записи.
    {
        let app_c = app.clone();
        let btn = get_button("cp-read-markdown")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            set_status(&app_in.ui.md_status, "busy", "clipboard.read()…");
            spawn_local(async move {
                match clip_core::read_items().await {
                    Ok(items) => {
                        let count = items.len();
                        if count == 0 {
                            set_status(&app_in.ui.md_status, "ok", "буфер пуст");
                            app_in.push_log(LogKind::Info, "буфер пуст".into());
                            return;
                        }
                        let mimes: Vec<String> = items.iter().map(|i| i.mime.clone()).collect();
                        let summary = format_str(format_args!(
                            "в буфере {} форматов: {}",
                            count, mimes.join(", "),
                        ));
                        app_in.push_log(LogKind::Ok, summary.clone());
                        set_status(&app_in.ui.md_status, "ok", &summary);
                        render_read_preview(&app_in, &items).await;
                    }
                    Err(e) => {
                        let kind = clip_core::classify_clip_error(&e);
                        let msg = format_str(format_args!(
                            "read: {} — {}", kind.name(), kind.short()
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.md_status, "err", &msg);
                    }
                }
            });
        });
    }

    // Копировать PNG.
    {
        let app_c = app.clone();
        let btn = get_button("cp-copy-image")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let size: u32 = app_in.ui.img_size.value().parse().unwrap_or(64);
            set_status(&app_in.ui.img_status, "busy", "генерация PNG…");

            spawn_local(async move {
                match make_png_blob(size).await {
                    Ok(blob) => {
                        show_image_preview(&app_in, &app_in.ui.img_preview.clone(), &blob);

                        match clip_core::write_item("image/png", &blob).await {
                            Ok(()) => {
                                let msg = format_str(format_args!(
                                    "PNG {size}×{size} скопирован ({})",
                                    fmt_bytes(blob.size() as u64),
                                ));
                                app_in.push_log(LogKind::Ok, msg.clone());
                                set_status(&app_in.ui.img_status, "ok", &msg);
                            }
                            Err(e) => {
                                let kind = clip_core::classify_clip_error(&e);
                                let msg = format_str(format_args!(
                                    "write PNG: {} — {}", kind.name(), kind.short()
                                ));
                                app_in.push_log(LogKind::Err, msg.clone());
                                set_status(&app_in.ui.img_status, "err", &msg);
                            }
                        }
                    }
                    Err(e) => {
                        let msg = format_str(format_args!(
                            "генерация PNG: {}", short_error(&e)
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.img_status, "err", &msg);
                    }
                }
            });
        });
    }

    // Читать всё из буфера.
    {
        let app_c = app.clone();
        let btn = get_button("cp-read-all")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            set_status(&app_in.ui.read_status, "busy", "clipboard.read()…");
            spawn_local(async move {
                match clip_core::read_items().await {
                    Ok(items) => {
                        let count = items.len();
                        if count == 0 {
                            set_status(&app_in.ui.read_status, "ok", "буфер пуст");
                            app_in.push_log(LogKind::Info, "буфер пуст".into());
                            return;
                        }
                        let mimes: Vec<String> = items.iter().map(|i| i.mime.clone()).collect();
                        let summary = format_str(format_args!(
                            "прочитано {count} форматов: {}",
                            mimes.join(", "),
                        ));
                        // Обновляем и лог, и status-bar — пользователь
                        // видит результат в основной панели.
                        app_in.push_log(LogKind::Ok, summary.clone());
                        set_status(&app_in.ui.read_status, "ok", &summary);
                        render_read_preview(&app_in, &items).await;
                    }
                    Err(e) => {
                        let kind = clip_core::classify_clip_error(&e);
                        let msg = format_str(format_args!(
                            "read: {} — {}", kind.name(), kind.short()
                        ));
                        app_in.push_log(LogKind::Err, msg.clone());
                        set_status(&app_in.ui.read_status, "err", &msg);
                    }
                }
            });
        });
    }

    Ok(())
}

// ============================================================
//                    ОПЕРАЦИИ
// ============================================================

async fn refresh_perms(app: &Rc<App>) {
    let info = clip_perms::probe_strict().await;
    set_perm_badge_with_reason(&app.ui.perm_read, &info.read);
    set_perm_badge_with_reason(&app.ui.perm_write, &info.write);

    let read_label = match &info.read {
        clip_perms::QueryOutcome::State(s) => s.short().to_string(),
        clip_perms::QueryOutcome::Failed(_) => "н/д".to_string(),
    };
    let write_label = match &info.write {
        clip_perms::QueryOutcome::State(s) => s.short().to_string(),
        clip_perms::QueryOutcome::Failed(_) => "н/д".to_string(),
    };
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "разрешения: read={read_label}, write={write_label}",
        )),
    );

    // Причины отказа логируем по одной строке. `reason` теперь
    // короткая — благодаря `js_error_message` в clip_core/clip_perms.
    let mut both_failed = true;
    if let Some(reason) = info.read.reason() {
        app.push_log(
            LogKind::Info,
            format_str(format_args!("clipboard-read: {reason}")),
        );
    } else {
        both_failed = false;
    }
    if let Some(reason) = info.write.reason() {
        app.push_log(
            LogKind::Info,
            format_str(format_args!("clipboard-write: {reason}")),
        );
    } else {
        both_failed = false;
    }

    // Если оба разрешения недоступны через Permissions API — объясняем,
    // что это не блокер. В Firefox `readText`/`writeText` работают
    // через native prompt при первом вызове.
    if both_failed {
        app.push_log(
            LogKind::Info,
            "read/write всё равно работают — браузер спросит разрешение при первом вызове"
                .to_string(),
        );
    }
}

fn set_perm_badge_with_reason(el: &web_sys::Element, outcome: &clip_perms::QueryOutcome) {
    match outcome {
        clip_perms::QueryOutcome::State(s) => {
            set_perm_badge(el, *s);
            let _ = el.set_attribute("title", "разрешение известно Permissions API");
        }
        clip_perms::QueryOutcome::Failed(reason) => {
            el.set_class_name("cp-perm-badge unknown");
            el.set_text_content(Some("н/д"));
            let _ = el.set_attribute("title", reason);
        }
    }
}

/// Проверяет и подсвечивает поддержку каждого MIME-формата браузером.
fn update_support_badges(app: &Rc<App>) {
    let items: [(&str, &web_sys::Element); 3] = [
        ("text/plain", &app.ui.md_plain_badge),
        ("text/html", &app.ui.md_html_badge),
        ("text/markdown", &app.ui.md_markdown_badge),
    ];
    for (mime, el) in items {
        if clip_core::supports_mime(mime) {
            el.set_class_name("cp-md-support ok");
            el.set_text_content(Some("✓ в буфер"));
            let _ = el.set_attribute("title", "браузер разрешает запись этого формата");
        } else {
            el.set_class_name("cp-md-support no");
            el.set_text_content(Some("× не поддерживается"));
            let _ = el.set_attribute(
                "title",
                "браузер отклонит всю пачку, если этот формат в ней будет",
            );
        }
    }
}

/// `writeText` → `readText` → сравнение.
async fn copy_and_verify(app: Rc<App>, text: String) {
    if let Err(e) = clip_core::write_text(&text).await {
        let kind = clip_core::classify_clip_error(&e);
        let msg = format_str(format_args!(
            "writeText: {} — {}", kind.name(), kind.short()
        ));
        app.push_log(LogKind::Err, msg.clone());
        set_status(&app.ui.text_status, "err", &msg);
        return;
    }

    let back = match clip_core::read_text().await {
        Ok(t) => t,
        Err(e) => {
            let kind = clip_core::classify_clip_error(&e);
            let msg = format_str(format_args!(
                "записано, но readText упал: {}", kind.short()
            ));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.text_status, "err", &msg);
            return;
        }
    };

    if back == text {
        let msg = format_str(format_args!(
            "проверено: {} символов, содержимое совпадает",
            text.chars().count(),
        ));
        app.push_log(LogKind::Ok, msg.clone());
        set_status(&app.ui.text_status, "ok", &msg);
    } else {
        let msg = format_str(format_args!(
            "несоответствие: записали {} символов, прочитали {}",
            text.chars().count(),
            back.chars().count(),
        ));
        app.push_log(LogKind::Err, msg.clone());
        set_status(&app.ui.text_status, "err", &msg);
    }
}

/// Строит три представления фрагмента.
fn build_three_formats(title: &str, body: &str) -> (String, String, String) {
    let plain = format_str(format_args!("{title}\n\n{body}"));
    let html = format_str(format_args!(
        "<h2>{}</h2>\n<p>{}</p>",
        esc_html(title),
        md_inline_to_html(body),
    ));
    let markdown = format_str(format_args!("# {title}\n\n{body}"));
    (plain, html, markdown)
}

/// Live-превью трёх форматов в панели multi-format.
fn refresh_md_formats(app: &Rc<App>) {
    let title = app.ui.md_title.value();
    let body = app.ui.md_body.value();
    let (plain, html, markdown) = build_three_formats(&title, &body);
    app.ui.md_plain.set_text_content(Some(&plain));
    app.ui.md_html.set_text_content(Some(&html));
    app.ui.md_markdown.set_text_content(Some(&markdown));
}

/// Минимальный конвертер inline-Markdown → HTML.
/// Поддерживает `**bold**` — этого достаточно для демонстрации.
fn md_inline_to_html(md: &str) -> String {
    let mut out = String::with_capacity(md.len());
    let mut rest = md;
    while let Some(start) = rest.find("**") {
        out.push_str(&esc_html(&rest[..start]));
        let after = &rest[start + 2..];
        if let Some(end) = after.find("**") {
            out.push_str("<strong>");
            out.push_str(&esc_html(&after[..end]));
            out.push_str("</strong>");
            rest = &after[end + 2..];
        } else {
            out.push_str("**");
            rest = after;
        }
    }
    out.push_str(&esc_html(rest));
    out
}

/// Генерирует PNG-изображение с помощью `<canvas>`.
async fn make_png_blob(size: u32) -> Result<web_sys::Blob, JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let canvas = document.create_element("canvas")?
        .dyn_into::<web_sys::HtmlCanvasElement>()
        .map_err(|_| JsValue::from_str("canvas dyn_into failed"))?;
    canvas.set_width(size);
    canvas.set_height(size);

    let ctx = canvas.get_context("2d")?
        .ok_or("no 2d context")?
        .dyn_into::<web_sys::CanvasRenderingContext2d>()
        .map_err(|_| JsValue::from_str("ctx dyn_into failed"))?;

    let gradient = ctx.create_linear_gradient(0.0, 0.0, size as f64, size as f64);
    gradient.add_color_stop(0.0, "#3498db").ok();
    gradient.add_color_stop(1.0, "#9b59b6").ok();
    ctx.set_fill_style_canvas_gradient(&gradient);
    ctx.fill_rect(0.0, 0.0, size as f64, size as f64);

    let r = size as f64 / 2.0;
    ctx.set_fill_style_str("rgba(255,255,255,0.8)");
    ctx.begin_path();
    ctx.arc(r, r, r * 0.5, 0.0, std::f64::consts::PI * 2.0).ok();
    ctx.fill();

    let label = size.to_string();
    ctx.set_fill_style_str("#fff");
    ctx.set_font("bold 14px system-ui");
    ctx.set_text_align("center");
    ctx.set_text_baseline("middle");
    ctx.fill_text(&label, r, r).ok();

    let promise = canvas_to_blob_promise(&canvas, "image/png")?;
    let result = wasm_bindgen_futures::JsFuture::from(promise).await?;
    result.dyn_into::<web_sys::Blob>()
        .map_err(|_| JsValue::from_str("toBlob вернул не Blob"))
}

/// `canvas.toBlob(cb)` → `Promise<Blob>`.
fn canvas_to_blob_promise(
    canvas: &web_sys::HtmlCanvasElement,
    mime: &str,
) -> Result<js_sys::Promise, JsValue> {
    let canvas_c = canvas.clone();
    let mime_s = mime.to_string();
    let promise = js_sys::Promise::new(&mut move |resolve, _reject| {
        let resolve_c = resolve.clone();
        let cb = Closure::once_into_js(move |blob: JsValue| {
            let _ = resolve_c.call1(&JsValue::UNDEFINED, &blob);
        });
        let to_blob: js_sys::Function =
            js_sys::Reflect::get(&canvas_c, &JsValue::from_str("toBlob"))
                .unwrap()
                .dyn_into()
                .unwrap();
        let _ = to_blob.call2(
            &canvas_c,
            cb.unchecked_ref(),
            &JsValue::from_str(&mime_s),
        );
    });
    Ok(promise)
}

fn set_perm_badge(el: &web_sys::Element, state: PermState) {
    let cls = match state {
        PermState::Granted => "cp-perm-badge granted",
        PermState::Prompt => "cp-perm-badge prompt",
        PermState::Denied => "cp-perm-badge denied",
        PermState::Unknown => "cp-perm-badge unknown",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(state.short()));
}

/// Отзывает все ранее созданные ObjectURL.
fn revoke_urls(app: &Rc<App>) {
    let mut urls = app.urls.borrow_mut();
    for u in urls.drain(..) {
        let _ = web_sys::Url::revoke_object_url(&u);
    }
}

/// Создаёт ObjectURL и регистрирует его для будущего отзыва.
fn track_url(app: &Rc<App>, url: String) {
    app.urls.borrow_mut().push(url);
}

/// Показывает превью изображения. Отзывает предыдущие URL.
fn show_image_preview(app: &Rc<App>, el: &web_sys::Element, blob: &web_sys::Blob) {
    revoke_urls(app);

    let url = match web_sys::Url::create_object_url_with_blob(blob) {
        Ok(u) => u,
        Err(_) => return,
    };
    track_url(app, url.clone());

    let mut html = String::with_capacity(128);
    html.push_str("<img src=\"");
    html.push_str(&esc_html(&url));
    html.push_str("\" alt=\"preview\">");
    el.set_inner_html(&html);
    let _ = el.set_attribute("style", "display:block");
}

/// Формирует HTML-чип для MIME-типа.
fn mime_chip(mime: &str) -> String {
    let cls = match mime {
        "text/plain" => "cp-mime-plain",
        "text/html" => "cp-mime-html",
        "text/markdown" => "cp-mime-markdown",
        "image/png" => "cp-mime-png",
        "image/jpeg" => "cp-mime-jpeg",
        "image/webp" => "cp-mime-webp",
        _ => "cp-mime-other",
    };
    let mut html = String::with_capacity(80);
    html.push_str("<span class=\"cp-mime-chip ");
    html.push_str(cls);
    html.push_str("\" title=\"");
    html.push_str(&esc_html(mime));
    html.push_str("\">");
    html.push_str(&esc_html(mime));
    html.push_str("</span>");
    html
}

/// Отрисовывает превью прочитанных форматов.
async fn render_read_preview(app: &Rc<App>, items: &[clip_core::ClipItem]) {
    revoke_urls(app);

    let mut html = String::with_capacity(512);
    for item in items {
        html.push_str("<div class=\"cp-read-item\">");
        html.push_str(&mime_chip(&item.mime));
        html.push_str(&format_str(format_args!(
            "<span class=\"cp-read-size\">{}</span>",
            fmt_bytes(item.blob.size() as u64),
        )));

        if item.mime == "text/html" {
            // Специальная обработка: снимаем CF_HTML-обёртку, иначе
            // пользователь видит мусор от браузера, а не свой фрагмент.
            match clip_core::blob_to_text(&item.blob).await {
                Ok(raw) => {
                    let clean = clip_core::strip_cf_html_wrapper(&raw);
                    let stripped = clean.len() != raw.trim().len();
                    let total_chars = clean.chars().count();
                    let preview: String = clean.chars().take(200).collect();
                    let truncated = total_chars > 200;
                    html.push_str("<pre>");
                    html.push_str(&esc_html(&preview));
                    if truncated {
                        html.push_str(&format_str(format_args!(
                            "\n… ещё {} символов", total_chars - 200,
                        )));
                    }
                    html.push_str("</pre>");
                    if stripped {
                        html.push_str(
                            "<div class=\"cp-read-note\">\
                             ↑ показан фрагмент без CF_HTML-обёртки браузера\
                             </div>",
                        );
                    }
                }
                Err(_) => html.push_str("<pre>не удалось прочитать</pre>"),
            }
        } else if item.mime.starts_with("text/") {
            match clip_core::blob_to_text(&item.blob).await {
                Ok(text) => {
                    let total_chars = text.chars().count();
                    let preview: String = text.chars().take(200).collect();
                    let truncated = total_chars > 200;
                    html.push_str("<pre>");
                    html.push_str(&esc_html(&preview));
                    if truncated {
                        html.push_str(&format_str(format_args!(
                            "\n… ещё {} символов", total_chars - 200,
                        )));
                    }
                    html.push_str("</pre>");
                }
                Err(_) => html.push_str("<pre>не удалось прочитать</pre>"),
            }
        } else if item.mime.starts_with("image/") {
            if let Ok(url) = web_sys::Url::create_object_url_with_blob(&item.blob) {
                track_url(app, url.clone());
                html.push_str("<div style=\"margin-top:4px\"><img src=\"");
                html.push_str(&esc_html(&url));
                html.push_str("\" style=\"max-height:120px;border-radius:4px\"></div>");
            }
        } else {
            html.push_str("<pre>бинарные данные</pre>");
        }
        html.push_str("</div>");
    }
    app.ui.read_preview.set_inner_html(&html);
    let _ = app.ui.read_preview.set_attribute("style", "display:block");
}

// ============================================================
//                    LOG
// ============================================================

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        let time = fmt_now_hms();
        *self.total_events.borrow_mut() += 1;
        let total = *self.total_events.borrow();

        let mut v = self.log.borrow_mut();
        v.push(LogEntry { time, text, kind });
        if v.len() > 8 {
            let drop_n = v.len() - 8;
            v.drain(0..drop_n);
        }
        drop(v);

        // Счётчик всех событий за сессию — если записей больше 8,
        // пользователь видит, что часть истории вытеснена.
        self.ui.log_counter.set_text_content(Some(&total.to_string()));
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"cp-log-empty\">здесь появятся операции с буфером</div>");
            return;
        }
        let mut html = String::with_capacity(512);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
            };
            html.push_str("<div class=\"cp-log-entry ");
            html.push_str(cls);
            html.push_str("\"><span class=\"t\">");
            html.push_str(&esc_html(&e.time));
            html.push_str("</span><span class=\"m\">");
            html.push_str(&esc_html(&e.text));
            html.push_str("</span></div>");
        }
        self.ui.log.set_inner_html(&html);
        self.ui.log.set_scroll_top(self.ui.log.scroll_height() as f64);
    }
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "cp-status ok",
        "err" => "cp-status err",
        "busy" => "cp-status busy",
        _ => "cp-status",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn fmt_bytes(n: u64) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = 1024.0 * 1024.0;
    let f = n as f64;
    if f < KB {
        return format_str(format_args!("{n} Б"));
    }
    if f < MB {
        return format_str(format_args!("{:.1} КБ", f / KB));
    }
    return format_str(format_args!("{:.2} МБ", f / MB));
}

fn short_error(e: &JsValue) -> String {
     return crate::clip_core::js_error_message(e);
}

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(), d.get_minutes(), d.get_seconds(),
    ));
}

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window().ok_or("no window")?
        .document().ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}

fn get_input(id: &str) -> Result<web_sys::HtmlInputElement, JsValue> {
    get_el(id)?.dyn_into().map_err(|_| JsValue::from_str(&format!("#{id} не input")))
}

fn get_textarea(id: &str) -> Result<web_sys::HtmlTextAreaElement, JsValue> {
    get_el(id)?.dyn_into().map_err(|_| JsValue::from_str(&format!("#{id} не textarea")))
}

fn get_select(id: &str) -> Result<web_sys::HtmlSelectElement, JsValue> {
    get_el(id)?.dyn_into().map_err(|_| JsValue::from_str(&format!("#{id} не select")))
}

fn get_button(id: &str) -> Result<web_sys::HtmlButtonElement, JsValue> {
    get_el(id)?.dyn_into().map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}

fn set_click<F>(el: &web_sys::HtmlButtonElement, f: F)
where F: FnMut(web_sys::MouseEvent) + 'static {
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```

## 32.12. Итоги главы

- **Clipboard API — два уровня.** `writeText`/`readText` — простой. `write`/`read` + `ClipboardItem` — общий.
- **Один `ClipboardItem` — несколько MIME.** Это то, что делает «копировать как Markdown» возможным.
- **`supports_mime`** — обязательная проверка перед multi-format записью.
- **`ClipboardItem` — класс.** Через `Reflect::construct`.
- **`Permissions API` не везде поддерживает clipboard-*.** Firefox — нет. Показывайте причину.
- **Ошибки JS — объекты.** `name`/`message` через `Reflect`, обрезка до первой строки.
- **CF_HTML-обёртка** Chrome для `text/html` — снимается `strip_cf_html_wrapper`.
- **ObjectURL** — трекаются и отзываются.
- **Правило проекта:** короткие строки вместо стектрейсов, честные ошибки вместо fallback `Unknown`, отсутствие мёртвого кода.

**Мостик к 32.13.** Буфер обмена — это пример API, который выглядит простым, но требует аккуратности: user gesture, разрешения, ограничения браузера, форматы. Дальше — **Web Share API**: ещё один способ поделиться данными между приложениями, с той же дисциплиной обработки отказа и разрешений.