# Глава 20. Blob, Object URL и скачивание

## Обновлённая структура проекта

```
hello-wasm02/
├── src/
│   ├── blob_api.rs               ← НОВЫЙ: Blob, Object URL, скачивание
│   ├── app.rs                     ← ДЕМО главы 20 (только этот сценарий)
│   ├── lib.rs                     ← +pub mod blob_api
│   └── … (остальные модули проекта)
├── tests/
│   ├── wasm_blob_api.rs           ← НОВЫЙ: create_blob, URL, revoke, download
│   └── … (остальные тесты)
├── pkg/
└── web/
    ├── index.html
    └── main.js
```

### Краткий комментарий

**`src/blob_api.rs`** — обёртка над четырьмя примитивами: `Blob`, `URL.createObjectURL`, `URL.revokeObjectURL`, скачивание через `<a download>`. Модуль не знает ни про какие форматы, генерацию данных или UI — только байты, MIME и URL.

**`tests/wasm_blob_api.rs`** — проверяет создание Blob из байтов, формат URL, идемпотентность `revoke`, отсутствие паники при download.

**`src/app.rs`** — демо главы 20: один входной текст → четыре формата вывода (TXT, JSON, HTML, PNG). Rust генерирует байты, JS упаковывает в Blob, получает Object URL, показывает превью и предлагает скачать. Счётчик живых URL — наглядная память.

### Ссылки для изучения

- [MDN: Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob) — контейнер бинарных данных.
- [MDN: URL.createObjectURL](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL_static) — временный URL на Blob.
- [MDN: URL.revokeObjectURL](https://developer.mozilla.org/en-US/docs/Web/API/URL/revokeObjectURL_static) — освобождение.
- [MDN: \<a download\>](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#download) — атрибут скачивания.
- [web.dev: Blob URLs](https://web.dev/articles/blob-urls) — практический обзор.

---

## 20.1. Три ключевых примитива

**1. `Blob`** — контейнер для бинарных данных плюс MIME-тип. Неотличим для потребителя от файла: те же `size`, `type`, `arrayBuffer()`, `text()`. Создаётся из массива кусков — строк или буферов.

**2. `URL.createObjectURL(blob)`** — возвращает строку вида `blob:http://localhost:5173/a1b2c3d4-…`. Эта строка — **полноценный URL**: её можно подставить в `<img src>`, `<a href>`, `<iframe src>`, `fetch()`, `new Worker()`. Никакой сериализации данных в URL не происходит — она остаётся в памяти, а URL — просто ссылка.

**3. `URL.revokeObjectURL(url)`** — освобождает память. Пока URL активна, Blob держится в памяти даже если JS-переменная потеряна. Каждая `createObjectURL` — это утечка, пока не вызовут `revoke`. Правило: **revoke ровно один раз на каждый create**.

**4. Скачивание через `<a download>`** — атрибут `download` превращает навигацию по ссылке в скачивание файла. С blob URL это работает без user gesture (в отличие от `showSaveFilePicker`).

---

## 20.2. Новый модуль: `src/blob_api.rs`

```rust
//! Обёртка над Blob, Object URL и скачиванием файлов.
//!
//! **Что даёт.** Четыре универсальных примитива:
//! * `create_blob(bytes, mime)` — Blob из байтов;
//! * `create_object_url(blob)` — временный URL вида `blob:…`;
//! * `revoke_object_url(url)` — освободить URL (идемпотентно);
//! * `download_url(url, filename)` — скачать по URL через `<a download>`.
//!
//! **Что НЕ содержит.** Никаких форматов (JSON, PNG, HTML),
//! никакой генерации данных, никакого UI. Только байты → Blob → URL.
//!
//! **Почему через Reflect.** Сигнатуры конструкторов `Blob`
//! в `web-sys` менялись между версиями. Reflect-подход работает
//! везде и не привязывается к крейту.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции — через
//! `format_str(format_args!(…))`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

// ===========================================================================
// Строковые хелперы
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

// ===========================================================================
// Создание Blob
// ===========================================================================

/// Создать Blob из байтов с MIME-типом.
///
/// `mime` должен быть валидным типом: `"text/plain"`,
/// `"application/json"`, `"image/png"`, `"text/html;charset=utf-8"`.
/// Пустая строка тоже допустима — браузер подставит
/// `"application/octet-stream"` при использовании.
pub fn create_blob(bytes: &[u8], mime: &str) -> Result<web_sys::Blob, JsValue> {
    let u8arr = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&u8arr.buffer());

    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("type"),
        &JsValue::from_str(mime),
    )?;

    let global = js_sys::global();
    let ctor_js = js_sys::Reflect::get(&global, &JsValue::from_str("Blob"))?;
    let ctor: js_sys::Function = ctor_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("Blob не конструктор"))?;

    let args = js_sys::Array::new();
    args.push(&parts);
    args.push(&options);

    let blob_js = js_sys::Reflect::construct(&ctor, &args)?;
    blob_js
        .dyn_into::<web_sys::Blob>()
        .map_err(|_| JsValue::from_str("не Blob"))
}

/// Размер Blob в байтах.
pub fn blob_size(blob: &web_sys::Blob) -> f64 {
    blob.size()
}

/// MIME-тип Blob.
pub fn blob_mime(blob: &web_sys::Blob) -> String {
    blob.type_()
}

// ===========================================================================
// Object URL
// ===========================================================================

/// Создать Object URL из Blob.
///
/// Возвращает строку вида `blob:http://localhost:5173/uuid`.
/// **Каждый вызов `create_object_url` нужно парно завершить
/// `revoke_object_url`.** Иначе Blob держится в памяти до
/// закрытия вкладки.
pub fn create_object_url(blob: &web_sys::Blob) -> Result<String, JsValue> {
    web_sys::Url::create_object_url_with_blob(blob)
}

/// Освободить Object URL.
///
/// **Идемпотентно.** Повторный revoke того же URL — no-op,
/// не паника. Браузер молча игнорирует.
pub fn revoke_object_url(url: &str) {
    web_sys::Url::revoke_object_url(url).ok();
}

// ===========================================================================
// Скачивание
// ===========================================================================

/// Скачать ресурс по URL под указанным именем.
///
/// Создаёт `<a href="..." download="...">`, кладёт его в `<body>`,
/// программно кликает, удаляет. `<a>` **обязан быть в DOM** —
/// некоторые браузеры игнорируют программный click на отвязанном
/// элементе.
///
/// **User gesture не требуется.** Blob URL — безопасный ресурс
/// того же origin, браузер разрешает скачивание программным
/// кликом. В отличие от `showSaveFilePicker`, который требует
/// клика пользователя.
pub fn download_url(url: &str, filename: &str) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let doc = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;

    let a = doc.create_element("a")?;
    a.set_attribute("href", url)?;
    a.set_attribute("download", filename)?;
    a.set_attribute("style", "display:none")?;
    body.append_child(&a)?;

    let a_el: web_sys::HtmlElement = a
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    a_el.click();

    a_el.remove();
    Ok(())
}

/// Утилита: отрезать префикс blob URL для отображения.
///
/// `blob:http://localhost:5173/uuid` → `…/uuid`.
/// Полная строка остаётся внутри — это только для UI.
pub fn shorten_url(url: &str) -> String {
    let max = 48;
    if url.len() <= max {
        return url.to_string();
    }
    let tail: String = url.chars().rev().take(24).collect::<String>();
    let tail: String = tail.chars().rev().collect();
    let s: String = format_str(format_args!("…{tail}"));
    s
}
```

### Что важно в модуле

**1. `create_blob` — через `Reflect::construct`.** Сигнатуры `Blob::new_with_…` в `web-sys` менялись. Reflect работает везде.

**2. `revoke_object_url` — игнорирует ошибки.** Revoke идемпотентен: повторный вызов — либо no-op, либо `InvalidStateError`, но нам это безразлично, ресурс уже освобождён.

**3. `download_url` — `<a>` в DOM.** Chrome/Safari игнорируют программный `click()` на отвязанном элементе. Вставляем в `body`, кликаем, удаляем.

**4. `shorten_url`** — UI-хелпер, но общий: формат вывода не зависит от демо.

---

## 20.3. Добавление модуля в `lib.rs`

```rust
// … существующие модули …
pub mod blob_api;   // ← НОВЫЙ

// …
```

---

## 20.4. Тесты: `tests/wasm_blob_api.rs`

```rust
//! Тесты `src/blob_api.rs`.
//!
//! Проверяем: создание Blob из байтов, корректный MIME, формат
//! object URL, идемпотентность revoke, отсутствие паники при
//! скачивании.

use wasm_bindgen_test::*;

use hello_wasm02::blob_api::{
    blob_mime, blob_size, create_blob, create_object_url, download_url,
    revoke_object_url, shorten_url,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// create_blob
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_create_blob_empty() {
    let blob = create_blob(&[], "application/octet-stream").expect("blob");
    assert_eq!(blob_size(&blob), 0.0);
    assert_eq!(blob_mime(&blob), "application/octet-stream");
}

#[wasm_bindgen_test]
fn test_create_blob_with_bytes() {
    let data = b"hello, blob!";
    let blob = create_blob(data, "text/plain").expect("blob");
    assert_eq!(blob_size(&blob), data.len() as f64);
    assert_eq!(blob_mime(&blob), "text/plain");
}

#[wasm_bindgen_test]
fn test_create_blob_with_utf8_mime() {
    let data = "привет мир".as_bytes();
    let blob = create_blob(data, "text/plain;charset=utf-8").expect("blob");
    assert_eq!(blob_size(&blob), data.len() as f64);
    assert_eq!(blob_mime(&blob), "text/plain;charset=utf-8");
}

// ---------------------------------------------------------------------------
// object URL
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_create_object_url_format() {
    let blob = create_blob(b"test", "text/plain").expect("blob");
    let url = create_object_url(&blob).expect("url");
    assert!(url.starts_with("blob:"), "URL должен начинаться с blob: — {url}");
    revoke_object_url(&url);
}

#[wasm_bindgen_test]
fn test_revoke_is_idempotent() {
    let blob = create_blob(b"x", "text/plain").expect("blob");
    let url = create_object_url(&blob).expect("url");
    // Три вызова подряд — все Ok, без паники.
    revoke_object_url(&url);
    revoke_object_url(&url);
    revoke_object_url(&url);
}

// ---------------------------------------------------------------------------
// download
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_download_url_does_not_panic() {
    let blob = create_blob(b"test", "text/plain").expect("blob");
    let url = create_object_url(&blob).expect("url");
    // В headless Chrome фактического скачивания не будет, но
    // ошибок быть не должно. Это проверка «не паникует».
    download_url(&url, "test.txt").expect("download");
    revoke_object_url(&url);
}

// ---------------------------------------------------------------------------
// shorten_url
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_shorten_url_short_unchanged() {
    let short = "blob:abc";
    assert_eq!(shorten_url(short), short);
}

#[wasm_bindgen_test]
fn test_shorten_url_long_truncated() {
    let long = "blob:http://localhost:5173/a1b2c3d4-e5f6-7890-abcd-ef1234567890";
    let out = shorten_url(long);
    assert!(out.len() < long.len());
    assert!(out.starts_with('…'));
}
```

### Что не проверяется в тестах

**Фактическое скачивание файла.** В headless Chrome нельзя проверить, что файл приземлился на диск — этой интеграции нет в WASM-тестах. Проверяем только «не паникует».

**Реальный revoke + попытка использовать URL после.** В headless Chrome контент уже загружен — не отличить. В браузере это проверяется визуально: превью ломается.

---

## 20.5. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты blob API в headless Chrome
wasm-pack test --headless --chrome --test wasm_blob_api

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** изменений. `image` уже подключён (для PNG-генерации в демо).

---

## 20.6. Анализ и доработки

### Что проявилось на первой версии

**1. Одно превью — мало.** Первая версия показывала один артефакт. Скучно. Пользователь не видит, что «Rust file processor» — это про **множество форматов**.

**2. Нет счётчика URL.** Ключевое в Object URL — их нужно освобождать. Без визуализации количества живых URL главная идея теряется.

**3. Ревок не виден.** Нажал «release» — и ничего не изменилось. Нужен визуальный статус «URL активна / URL освобождена».

**4. Preview для HTML опасен.** Рендерить HTML как HTML — прямой XSS, даже если контент наш. Правильно — показывать как код.

**5. PNG-превью без визуализации.** Просто `<img src>` — но что именно нарисовано, непонятно. Нужно сделать визуализацию осмысленной (например, столбики по байтам текста).

**6. Автозапуск.** Как в предыдущих главах — при загрузке всё должно быть заполнено.

**7. Скачивание без диалога.** Blob URL скачивается программно, но пользователь может не заметить, куда ушёл файл. Нужна подсказка: «файл в папке Загрузки».

### Список доработок

- **A.** **4 артефакта** — TXT, JSON, HTML, PNG. Один вход, четыре выхода.
- **B.** **Счётчик живых URL** на видном месте.
- **C.** **Статус URL у каждого артефакта** — «активна» / «освобождена».
- **D.** **HTML показать как код** (без рендера) + отдельный `<iframe sandbox>` с blob URL — демонстрация «URL работает как src».
- **E.** **PNG-визуализация** — столбики высотой по байтам текста. Разный текст → разная картинка.
- **F.** **Автозапуск** — сразу после загрузки генерируем из seed-текста.
- **G.** **Кнопки «Освободить все» и «Сгенерировать заново»** — наглядное управление URL.

Все семь доработок реализованы в финальном `app.rs` ниже.

---

## 20.7. Правила проекта, выученные в этой главе

**1. Object URL — это ресурс.** Каждая `createObjectURL` удерживает Blob в памяти. Пока URL не отозван, Blob не собирается GC.

**2. Парность create/revoke.** Один `createObjectURL` → ровно один `revokeObjectURL`. При перегенерации — отзываем старые, создаём новые.

**3. `revoke` идемпотентен.** Повторный вызов — no-op. Обёртка должна игнорировать ошибки.

**4. URL — не строка, а указатель.** Один и тот же URL работает как `<img src>`, `<iframe src>`, `<a href download>`, `fetch()`, `Worker()`. Это не «ссылка на данные» в смысле URL-encode — данные остаются в памяти.

**5. `<a download>` работает без user gesture.** В отличие от `showSaveFilePicker`. `<a>` должен быть в DOM на момент `click()`.

**6. MIME-тип влияет на обработку.** Браузер решает по `blob.type` — показывать картинку или скачать. `text/html` в `<iframe sandbox>` рендерится без скриптов.

**7. Sandbox для HTML.** Рендерить чужой HTML напрямую — XSS. `<iframe sandbox="">` блокирует скрипты. Если нужен интерактив — `sandbox="allow-scripts"`, но это опасно.

**8. Blob из Rust без сериализации.** `Uint8Array::from(bytes)` → `.buffer()` → parts для Blob. Одна копия, без base64.

**9. Reflect для конструктора Blob.** Сигнатуры `Blob::new_with_…` в `web-sys` менялись. `Reflect::construct` работает везде.

**10. Счётчик URL — визуальная память.** Как в 19.9 показывали квоту, здесь показываем живые URL. Это не «украшение», а прямая связь с концепцией.

**11. `set_scroll_top` принимает `f64`, `scroll_height()` возвращает `i32`.** В текущей сборке `web-sys 0.3.105` — каст `i32 as f64` обязателен. Проверяйте сигнатуры методов через компилятор, не по памяти.

**12. `?` работает только в функциях с `Result` или `Option`.** В функции, возвращающей `()` — используем `.ok()` для необязательных вызовов (`set_attribute`).

**13. Ни `format!`, ни `vec!` в tail-позиции.** Обёртки `format_str(format_args!(…))` и `vec_of(vec![…])` фиксируют тип на каждом шаге.

---

## 20.8. Чек-лист

- [ ] `create_blob(bytes, mime)` — Blob с правильным размером и MIME.
- [ ] `create_object_url(blob)` — строка начинается с `blob:`.
- [ ] `revoke_object_url(url)` — идемпотентно, не паникует.
- [ ] `download_url(url, name)` — `<a>` в DOM, click, remove.
- [ ] Object URL — временный, требует парного revoke.
- [ ] Один URL → `<img>`, `<iframe>`, `<a download>`, `fetch`.
- [ ] HTML в `<iframe sandbox>` — без скриптов.
- [ ] При перегенерации — старые URL отозваны до создания новых.
- [ ] Счётчик живых URL — визуальный, обновляется при create/revoke.
- [ ] Автозапуск — карточки заполнены при загрузке.
- [ ] `format_str(format_args!(…))` для всех строковых выражений в tail-позиции.
- [ ] `vec_of(vec![…])` для `vec!` в tail-позиции.
- [ ] `.ok()` вместо `?` в функциях, возвращающих `()`.
- [ ] Все методы `web-sys` — snake_case.
- [ ] `set_scroll_top(h as f64)` — с явным кастом.

---

## 20.9. Финальный `src/app.rs`

```rust
//! Демо главы 20 — Blob, Object URL, скачивание.
//!
//! **Идея.** Один входной текст → четыре формата вывода (TXT,
//! JSON, HTML, PNG). Rust генерирует байты, `blob_api` упаковывает
//! их в Blob, получает Object URL, показывает превью и предлагает
//! скачать.
//!
//! **Wow-моменты:**
//! 1. **Один вход — четыре файла.** Rust-«процессор» реально
//!    конвертирует текст в четыре разных MIME-типа.
//! 2. **URL — это ресурс.** Счётчик живых URL растёт при
//!    генерации, падает при освобождении.
//! 3. **URL — это URL.** HTML-артефакт вставляется в
//!    `<iframe sandbox src="blob:...">` — тот же URL, что и
//!    для скачивания, но работает как `src`.
//! 4. **Скачивание без диалогов.** Blob URL скачивается
//!    программным кликом `<a download>`.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>`.
//!
//! **Правило WASM-инференса (расширенное).**
//! Ни `format!`, ни `vec!` не должны стоять в tail-позиции функции,
//! ветки `if/else`, `match`-arm или блока. Обёртки:
//! * `format!(…)` → `format_str(format_args!(…))`;
//! * `vec![…]`    → `vec_of(vec![…])`.
//!
//! **Правило `?` и `Result`.** `?` применим только в функции,
//! возвращающей `Result` или `Option`. Если функция возвращает
//! `()` — использовать `.ok()`.
//!
//! **Правило числовых типов.** Сигнатуры методов `web-sys`
//! различаются между версиями. Доверяйте **текущему** выводу
//! компилятора. В этой сборке:
//! * `HtmlElement::scroll_height() -> i32`;
//! * `HtmlElement::set_scroll_top(value: f64)`.
//! — нужен явный каст `i32 as f64`.
//!
//! **Правило snake_case.** `create_element`, `set_attribute`,
//! `append_child`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Document, Element, HtmlElement, HtmlTextAreaElement};

use crate::blob_api::{
    blob_mime, blob_size, create_blob, create_object_url, download_url,
    revoke_object_url, shorten_url,
};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[20]";

fn console_log(msg: &str) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::log_1(&JsValue::from_str(&line));
}

fn console_err(msg: &str, e: &JsValue) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::error_2(&JsValue::from_str(&line), e);
}

// ===========================================================================
// Строковые и коллекционные хелперы (правило WASM-инференса)
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

/// Обёртка для `vec![…]` в tail-позиции функции.
#[inline]
fn vec_of<T>(v: Vec<T>) -> Vec<T> {
    v
}

fn fmt_bytes(n: usize) -> String {
    let s: String = if n < 1024 {
        format_str(format_args!("{n} Б"))
    } else if n < 1024 * 1024 {
        format_str(format_args!("{:.1} КБ", n as f64 / 1024.0))
    } else {
        format_str(format_args!("{:.2} МБ", n as f64 / (1024.0 * 1024.0)))
    };
    s
}

fn html_escape(s: &str) -> String {
    let mut out: String = String::with_capacity(s.len());
    for c in s.chars() {
        match c {
            '&' => out.push_str("&amp;"),
            '<' => out.push_str("&lt;"),
            '>' => out.push_str("&gt;"),
            '"' => out.push_str("&quot;"),
            '\'' => out.push_str("&#39;"),
            _ => out.push(c),
        }
    }
    out
}

// ===========================================================================
// Модель артефакта
// ===========================================================================

struct Artifact {
    name: String,
    mime: String,
    bytes: Vec<u8>,
    url: Option<String>,
}

impl Artifact {
    fn new(name: &str, mime: &str, bytes: Vec<u8>) -> Self {
        Artifact {
            name: name.to_string(),
            mime: mime.to_string(),
            bytes,
            url: None,
        }
    }
}

// ===========================================================================
// Генерация артефактов (Rust file processor)
// ===========================================================================

fn make_txt(input: &str) -> Vec<u8> {
    input.as_bytes().to_vec()
}

fn make_json(input: &str) -> Vec<u8> {
    let escaped: String = input
        .replace('\\', "\\\\")
        .replace('"', "\\\"")
        .replace('\n', "\\n");
    let len: usize = input.chars().count();
    let text: String = format_str(format_args!(
        "{{\n  \"text\": \"{escaped}\",\n  \"chars\": {len},\n  \"bytes\": {},\n  \"generator\": \"rust-wasm\"\n}}\n",
        input.len()
    ));
    text.into_bytes()
}

fn make_html(input: &str) -> Vec<u8> {
    let escaped: String = html_escape(input);
    let chars: usize = input.chars().count();
    let text: String = format_str(format_args!(
        "<!doctype html>\n\
         <html lang=\"ru\">\n\
         <head>\n\
         <meta charset=\"utf-8\">\n\
         <title>Rust report</title>\n\
         <style>\n\
         body {{ font-family: system-ui; padding: 24px; background: #f6f8fa; color: #24292f; }}\n\
         h1 {{ font-size: 18px; }}\n\
         .box {{ background: #fff; border: 1px solid #d0d7de; border-radius: 6px; padding: 16px; }}\n\
         .meta {{ color: #57606a; font-size: 12px; }}\n\
         </style>\n\
         </head>\n\
         <body>\n\
         <h1>Rust report</h1>\n\
         <div class=\"box\">\n\
         <p>{escaped}</p>\n\
         </div>\n\
         <p class=\"meta\">символов: {chars}</p>\n\
         </body>\n\
         </html>\n"
    ));
    text.into_bytes()
}

fn make_png(input: &str) -> Result<Vec<u8>, JsValue> {
    use image::{ImageBuffer, Rgb, RgbImage};

    let bytes: &[u8] = input.as_bytes();
    let w: u32 = 480;
    let h: u32 = 200;
    let mut img: RgbImage = ImageBuffer::new(w, h);

    for (_, _, px) in img.enumerate_pixels_mut() {
        *px = Rgb([13, 17, 23]);
    }

    if !bytes.is_empty() {
        for x in 0..w {
            let b: u8 = bytes[(x as usize) % bytes.len()];
            let bar_h: u32 = (h as f32 * b as f32 / 255.0).max(2.0) as u32;
            let y_start: u32 = h.saturating_sub(bar_h);
            let color = Rgb([b, (b / 2).saturating_add(96), 200 - b / 3]);
            for y in y_start..h {
                img.put_pixel(x, y, color);
            }
        }
    }

    let mut out: Vec<u8> = Vec::new();
    img.write_to(&mut std::io::Cursor::new(&mut out), image::ImageFormat::Png)
        .map_err(|e| {
            let msg: String = format_str(format_args!("png encode: {e}"));
            JsValue::from_str(&msg)
        })?;
    Ok(out)
}

fn build_artifacts(input: &str) -> Vec<Artifact> {
    let png_bytes: Vec<u8> = make_png(input).unwrap_or_default();
    vec_of(vec![
        Artifact::new("report.txt", "text/plain;charset=utf-8", make_txt(input)),
        Artifact::new(
            "report.json",
            "application/json;charset=utf-8",
            make_json(input),
        ),
        Artifact::new(
            "report.html",
            "text/html;charset=utf-8",
            make_html(input),
        ),
        Artifact::new("chart.png", "image/png", png_bytes),
    ])
}

// ===========================================================================
// Состояние
// ===========================================================================

struct DemoState {
    status: Element,
    input: HtmlTextAreaElement,
    cards_body: Element,
    url_count_el: Element,
    log_body: Element,
    artifacts: RefCell<Vec<Artifact>>,
    live_urls: Cell<u32>,
    log_lines: RefCell<Vec<String>>,
}

// ===========================================================================
// CSS
// ===========================================================================

const CSS: &str = r#"
html, body {
  margin: 0 !important; padding: 0 !important;
  background: #0d1117 !important; color: #e6edf3 !important;
  min-height: 100vh;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  font-size: 14px; line-height: 1.55;
}
header { padding: 20px 32px 4px; }
h1 { font-size: 19px; margin: 0 0 4px; font-weight: 600; }
h1 code { color: #79c0ff; font-size: 0.95em; }
.subtitle { color: #8b949e; margin: 0; font-size: 13px; }

.status-bar {
  padding: 12px 32px 4px; font-size: 12px; color: #8b949e;
  font-family: ui-monospace, monospace;
}
.status-bar.ok { color: #3fb950; }
.status-bar.err { color: #f85149; }

.section {
  margin: 16px 32px; background: #161b22;
  border: 1px solid #30363d; border-radius: 8px;
  overflow: hidden;
}
.section h3 {
  margin: 0; padding: 10px 14px;
  background: #21262d; font-size: 13px; font-weight: 600;
  border-bottom: 1px solid #30363d;
  display: flex; align-items: baseline; gap: 8px;
  cursor: help;
}
.section h3 .num { color: #79c0ff; font-weight: 700; font-size: 14px; }
.section h3 .right {
  color: #8b949e; font-weight: 400; font-size: 11px;
  margin-left: auto;
}
.section .body { padding: 14px 16px; }

.intro {
  margin: 12px 32px 4px;
  padding: 12px 16px;
  background: rgba(31, 111, 235, .08);
  border: 1px solid rgba(31, 111, 235, .30);
  border-radius: 8px;
  color: #8b949e; font-size: 13px; line-height: 1.6;
}
.intro b { color: #e6edf3; font-weight: 500; }
.intro code { color: #79c0ff; font-family: ui-monospace, monospace; font-size: 12px; }

textarea.src {
  width: 100%; box-sizing: border-box;
  min-height: 80px;
  background: #0d1117; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 10px 12px;
  font-family: ui-monospace, monospace; font-size: 13px;
  line-height: 1.55; resize: vertical; outline: none;
}
textarea.src:focus { border-color: #1f6feb; }

.row-controls {
  display: flex; flex-wrap: wrap; gap: 8px;
  align-items: center; margin-top: 12px;
}
.btn {
  background: #21262d; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 8px 14px; font-size: 13px; cursor: pointer;
  transition: background .15s, border-color .15s;
}
.btn:hover { background: #30363d; border-color: #8b949e; }
.btn:active { background: #1f6feb; border-color: #1f6feb; }
.btn.primary { background: #1f6feb; border-color: #1f6feb; }
.btn.primary:hover { background: #388bfd; }
.btn.danger { background: #3d1418; border-color: #6e2a2e; }
.btn.danger:hover { background: #5a1f24; border-color: #f85149; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.cards {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 14px;
}
@media (max-width: 900px) { .cards { grid-template-columns: 1fr; } }

.card {
  background: #0d1117;
  border: 1px solid #30363d; border-radius: 8px;
  overflow: hidden;
  display: flex; flex-direction: column;
}
.card .head {
  padding: 10px 14px;
  background: #21262d; border-bottom: 1px solid #30363d;
  display: flex; align-items: baseline; gap: 10px;
}
.card .head .name {
  font-family: ui-monospace, monospace; font-size: 13px;
  color: #79c0ff; font-weight: 500;
}
.card .head .mime {
  color: #6e7681; font-family: ui-monospace, monospace;
  font-size: 11px;
}
.card .head .size {
  color: #8b949e; font-family: ui-monospace, monospace;
  font-size: 11px; margin-left: auto;
}
.card .preview {
  padding: 10px 14px; flex: 1;
  background: #0d1117;
  min-height: 180px;
  max-height: 260px;
  overflow: auto;
}
.card .preview pre {
  margin: 0; font-family: ui-monospace, monospace; font-size: 12px;
  color: #e6edf3; white-space: pre-wrap; word-break: break-word;
}
.card .preview img {
  display: block; max-width: 100%; image-rendering: pixelated;
  border: 1px solid #30363d; border-radius: 4px;
}
.card .preview iframe {
  display: block; width: 100%; height: 180px;
  border: 1px solid #30363d; border-radius: 4px;
  background: #fff;
  margin-top: 8px;
}
.card .foot {
  padding: 10px 14px;
  border-top: 1px solid #21262d;
  display: flex; align-items: center; gap: 10px;
}
.card .url-line {
  font-family: ui-monospace, monospace; font-size: 11px;
  color: #6e7681;
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
}
.card .url-line.released { color: #f85149; }
.card .url-line.active   { color: #3fb950; }

.mem-panel {
  display: flex; align-items: baseline; gap: 20px;
  font-family: ui-monospace, monospace; font-size: 13px;
  margin-bottom: 12px;
}
.mem-panel .label { color: #8b949e; }
.mem-panel .value { color: #79c0ff; font-weight: 600; font-size: 18px; }
.mem-panel .spacer { flex: 1; }

.log-body {
  padding: 12px 16px; max-height: 180px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; white-space: pre-wrap; }
.log-line.info  { color: #8b949e; }
.log-line.ok    { color: #3fb950; }
.log-line.err   { color: #f85149; }
.log-line.blob  { color: #79c0ff; }
"#;

// ===========================================================================
// Логирование
// ===========================================================================

fn log_line(state: &Rc<DemoState>, kind: &str, msg: &str) {
    {
        let mut v = state.log_lines.borrow_mut();
        v.push(msg.to_string());
        if v.len() > 100 {
            let excess = v.len() - 100;
            v.drain(0..excess);
        }
    }

    let window = match web_sys::window() {
        Some(w) => w,
        None => return,
    };
    let doc = match window.document() {
        Some(d) => d,
        None => return,
    };
    let line = match doc.create_element("div") {
        Ok(e) => e,
        Err(_) => return,
    };
    let cls: String = format_str(format_args!("log-line {kind}"));
    line.set_class_name(&cls);
    line.set_text_content(Some(msg));
    state.log_body.append_child(&line).ok();

    // `scroll_height() -> i32`, `set_scroll_top(value: f64)`.
    // Явный каст — см. правило числовых типов в шапке файла.
    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let h: i32 = el.scroll_height();
        el.set_scroll_top(h as f64);
    }
}

fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

// ===========================================================================
// Управление URL
// ===========================================================================

fn release_all_urls(state: &Rc<DemoState>) {
    let mut arts = state.artifacts.borrow_mut();
    let mut count: u32 = 0;
    for a in arts.iter_mut() {
        if let Some(url) = a.url.take() {
            revoke_object_url(&url);
            count += 1;
        }
    }
    state.live_urls.set(0);
    drop(arts);
    let line: String = format_str(format_args!("освобождено URL: {count}"));
    log_line(state, "blob", &line);
}

fn generate(state: Rc<DemoState>) {
    let input: String = state.input.value();

    release_all_urls(&state);

    let mut new_arts: Vec<Artifact> = build_artifacts(&input);
    log_line(
        &state,
        "info",
        &format_str(format_args!(
            "сгенерировано {} артефактов",
            new_arts.len()
        )),
    );

    let mut created: u32 = 0;
    for a in new_arts.iter_mut() {
        match create_blob(&a.bytes, &a.mime) {
            Ok(blob) => {
                let sz = blob_size(&blob);
                let mime = blob_mime(&blob);
                if sz as usize != a.bytes.len() {
                    let line: String = format_str(format_args!(
                        "blob size mismatch for {}: {} vs {}",
                        a.name,
                        sz as usize,
                        a.bytes.len()
                    ));
                    log_line(&state, "err", &line);
                }
                match create_object_url(&blob) {
                    Ok(url) => {
                        let line: String = format_str(format_args!(
                            "{} → {} (blob size={}, mime={})",
                            a.name,
                            shorten_url(&url),
                            sz as usize,
                            mime
                        ));
                        log_line(&state, "blob", &line);
                        a.url = Some(url);
                        created += 1;
                    }
                    Err(e) => {
                        console_err("create_object_url failed", &e);
                        log_line(&state, "err", "не удалось создать URL");
                    }
                }
            }
            Err(e) => {
                console_err("create_blob failed", &e);
                let line: String = format_str(format_args!(
                    "не удалось создать Blob для {}",
                    a.name
                ));
                log_line(&state, "err", &line);
            }
        }
    }
    state.live_urls.set(created);

    *state.artifacts.borrow_mut() = new_arts;
    render_cards(&state);
    update_mem_panel(&state);

    let status_text: String = format_str(format_args!(
        "готово · {} артефактов · {} живых URL",
        created, created
    ));
    set_status(&state, "ok", &status_text);
}

// ===========================================================================
// Рендер
// ===========================================================================

fn update_mem_panel(state: &Rc<DemoState>) {
    let n: u32 = state.live_urls.get();
    let s: String = format_str(format_args!("{n}"));
    state.url_count_el.set_text_content(Some(&s));
}

fn render_cards(state: &Rc<DemoState>) {
    let doc = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d,
        None => return,
    };

    state.cards_body.set_inner_html("");

    let arts = state.artifacts.borrow();
    for (idx, a) in arts.iter().enumerate() {
        let card = match doc.create_element("div") {
            Ok(e) => e,
            Err(_) => continue,
        };
        card.set_class_name("card");

        let head = match doc.create_element("div") {
            Ok(e) => e,
            Err(_) => continue,
        };
        head.set_class_name("head");

        let name = match doc.create_element("span") {
            Ok(e) => e,
            Err(_) => continue,
        };
        name.set_class_name("name");
        name.set_text_content(Some(&a.name));
        head.append_child(&name).ok();

        let mime = match doc.create_element("span") {
            Ok(e) => e,
            Err(_) => continue,
        };
        mime.set_class_name("mime");
        mime.set_text_content(Some(&a.mime));
        head.append_child(&mime).ok();

        let size = match doc.create_element("span") {
            Ok(e) => e,
            Err(_) => continue,
        };
        size.set_class_name("size");
        let sz: String = fmt_bytes(a.bytes.len());
        size.set_text_content(Some(&sz));
        head.append_child(&size).ok();

        card.append_child(&head).ok();

        let preview = match doc.create_element("div") {
            Ok(e) => e,
            Err(_) => continue,
        };
        preview.set_class_name("preview");

        let is_image = a.mime.starts_with("image/");
        let is_html = a.mime.starts_with("text/html");

        if is_image {
            let img = match doc.create_element("img") {
                Ok(e) => e,
                Err(_) => continue,
            };
            match &a.url {
                Some(url) => {
                    img.set_attribute("src", url).ok();
                }
                None => {
                    img.set_attribute("alt", "URL освобождена").ok();
                }
            }
            img.set_attribute("width", "100%").ok();
            preview.append_child(&img).ok();
        } else if is_html {
            let pre = match doc.create_element("pre") {
                Ok(e) => e,
                Err(_) => continue,
            };
            let text = String::from_utf8_lossy(&a.bytes).to_string();
            pre.set_text_content(Some(&text));
            preview.append_child(&pre).ok();

            if let Some(url) = &a.url {
                let iframe = match doc.create_element("iframe") {
                    Ok(e) => e,
                    Err(_) => continue,
                };
                iframe.set_attribute("src", url).ok();
                iframe.set_attribute("sandbox", "").ok();
                iframe.set_attribute("title", "HTML preview").ok();
                preview.append_child(&iframe).ok();
            }
        } else {
            let pre = match doc.create_element("pre") {
                Ok(e) => e,
                Err(_) => continue,
            };
            let text = String::from_utf8_lossy(&a.bytes).to_string();
            pre.set_text_content(Some(&text));
            preview.append_child(&pre).ok();
        }
        card.append_child(&preview).ok();

        let foot = match doc.create_element("div") {
            Ok(e) => e,
            Err(_) => continue,
        };
        foot.set_class_name("foot");

        let url_line = match doc.create_element("div") {
            Ok(e) => e,
            Err(_) => continue,
        };
        match &a.url {
            Some(url) => {
                url_line.set_class_name("url-line active");
                let s: String =
                    format_str(format_args!("URL: {} (активна)", shorten_url(url)));
                url_line.set_text_content(Some(&s));
            }
            None => {
                url_line.set_class_name("url-line released");
                url_line.set_text_content(Some("URL: освобождена"));
            }
        }
        foot.append_child(&url_line).ok();

        let spacer = match doc.create_element("span") {
            Ok(e) => e,
            Err(_) => continue,
        };
        spacer.set_attribute("style", "flex:1").ok();
        foot.append_child(&spacer).ok();

        let can_download = a.url.is_some();
        let btn = match make_download_button(&doc, state, idx, can_download) {
            Ok(b) => b,
            Err(_) => continue,
        };
        foot.append_child(&btn).ok();

        card.append_child(&foot).ok();

        state.cards_body.append_child(&card).ok();
    }
}

fn make_download_button(
    doc: &Document,
    state: &Rc<DemoState>,
    idx: usize,
    enabled: bool,
) -> Result<Element, JsValue> {
    let btn = doc.create_element("button")?;
    btn.set_class_name("btn");
    btn.set_text_content(Some("Скачать"));
    btn.set_attribute(
        "title",
        "Скачать файл через <a download> по Object URL.",
    )?;
    if !enabled {
        btn.set_attribute("disabled", "").ok();
    }

    let s = state.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        let arts = s.artifacts.borrow();
        if let Some(a) = arts.get(idx) {
            if let Some(url) = &a.url {
                match download_url(url, &a.name) {
                    Ok(()) => {
                        let line: String = format_str(format_args!(
                            "скачано: {} ({})",
                            a.name,
                            fmt_bytes(a.bytes.len())
                        ));
                        log_line(&s, "ok", &line);
                        let st: String =
                            format_str(format_args!("скачано · {}", a.name));
                        set_status(&s, "ok", &st);
                    }
                    Err(e) => {
                        console_err("download_url failed", &e);
                        log_line(&s, "err", "ошибка скачивания");
                    }
                }
            }
        }
    });

    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(btn)
}

// ===========================================================================
// UI-хелперы
// ===========================================================================

fn make_button<F>(
    doc: &Document,
    parent: &Element,
    label: &str,
    tooltip: &str,
    primary: bool,
    danger: bool,
    mut handler: F,
) -> Result<Element, JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    let cls: &str = if primary {
        "btn primary"
    } else if danger {
        "btn danger"
    } else {
        "btn"
    };
    btn.set_class_name(cls);
    btn.set_text_content(Some(label));
    btn.set_attribute("title", tooltip)?;
    parent.append_child(&btn)?;

    let closure = Closure::<dyn FnMut()>::new(move || handler());
    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(btn)
}

fn make_section(
    doc: &Document,
    num: &str,
    title: &str,
    hint: &str,
    tooltip: &str,
) -> Result<(Element, Element), JsValue> {
    let section = doc.create_element("div")?;
    section.set_class_name("section");

    let h3 = doc.create_element("h3")?;
    let header_html: String = format_str(format_args!(
        "<span class=\"num\">{num}</span> <span>{title}</span>\
         <span class=\"right\">{hint}</span>"
    ));
    h3.set_inner_html(&header_html);
    h3.set_attribute("title", tooltip)?;
    section.append_child(&h3)?;

    let body = doc.create_element("div")?;
    body.set_class_name("body");
    section.append_child(&body)?;

    Ok((section, body))
}

// ===========================================================================
// Точка входа
// ===========================================================================

pub fn run() -> Result<(), JsValue> {
    console_log("run: starting");

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = document
        .body()
        .ok_or_else(|| JsValue::from_str("no body"))?;

    body.set_inner_html("");

    if let Ok(links) = document.query_selector_all("link[rel='stylesheet']") {
        let n: u32 = links.length();
        for i in 0..n {
            if let Some(node) = links.get(i) {
                if let Some(parent) = node.parent_node() {
                    parent.remove_child(&node).ok();
                }
            }
        }
    }
    if let Some(root_el) = document.document_element() {
        root_el
            .set_attribute(
                "style",
                "background:#0d1117;color:#e6edf3;margin:0;padding:0;min-height:100vh;",
            )
            .ok();
    }

    let style = document.create_element("style")?;
    style.set_text_content(Some(CSS));
    document
        .head()
        .ok_or_else(|| JsValue::from_str("no head"))?
        .append_child(&style)
        .ok();

    let h1 = document.create_element("h1")?;
    h1.set_inner_html(
        "20 <code>Blob, Object URL, скачивание</code>: один вход — четыре файла",
    );
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "Rust генерирует байты · JS упаковывает в Blob · Object URL · download без диалогов",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    let intro = document.create_element("div")?;
    intro.set_class_name("intro");
    intro.set_inner_html(
        "<b>Что демонстрируется.</b> Rust-«процессор» превращает входной текст \
         в четыре формата: <code>TXT</code>, <code>JSON</code>, <code>HTML</code>, \
         <code>PNG</code>. Каждый формат — это <code>Vec&lt;u8&gt;</code>, который \
         упаковывается в <code>Blob</code>, получает свой <code>Object URL</code> \
         и используется для превью и скачивания.<br><br>\
         <b>Что важно понять.</b> Object URL — это <b>временный указатель</b> на \
         Blob в памяти. Он живёт до <code>revokeObjectURL</code>. Пока URL активна, \
         Blob держится в памяти, даже если переменная потеряна. Кнопка \
         «Освободить все» показывает счётчик — сколько сейчас живых URL.<br><br>\
         <b>Wow.</b> Один и тот же blob URL используется как <code>&lt;img src&gt;</code>, \
         как <code>&lt;iframe src&gt;</code> и как <code>&lt;a download&gt;</code>. \
         URL — универсальный интерфейс к байтам в памяти.",
    );
    body.append_child(&intro)?;

    let status = document.create_element("div")?;
    status.set_class_name("status-bar ok");
    status.set_text_content(Some("инициализация…"));
    body.append_child(&status)?;

    let (sec1, body1) = make_section(
        &document,
        "❶",
        "Исходный текст",
        "Rust превратит его в четыре файла",
        "Введите любой текст. При нажатии «Сгенерировать» Rust построит \
         четыре разных представления: сырой TXT, обёртку JSON, HTML-страницу \
         и PNG-график, где высота столбиков кодирует байты текста.",
    )?;
    body.append_child(&sec1)?;

    let src_ta = document.create_element("textarea")?;
    src_ta.set_class_name("src");
    let src_ta: HtmlTextAreaElement = src_ta
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlTextAreaElement"))?;
    src_ta.set_value("Hello, WASM! Этот текст превратится в четыре файла.");
    body1.append_child(&src_ta)?;

    let row = document.create_element("div")?;
    row.set_class_name("row-controls");
    body1.append_child(&row)?;

    let (sec2, body2) = make_section(
        &document,
        "❷",
        "Сгенерированные файлы",
        "превью · URL · скачивание",
        "Каждая карточка — один артефакт. Превью рендерится по MIME-типу: \
         текст как <pre>, PNG как <img>, HTML как <pre> + iframe sandbox \
         с тем же blob URL в src. Кнопка «Скачать» использует тот же URL.",
    )?;
    body.append_child(&sec2)?;

    let cards_body = document.create_element("div")?;
    cards_body.set_class_name("cards");
    body2.append_child(&cards_body)?;

    let (sec3, body3) = make_section(
        &document,
        "❸",
        "Память и жизненный цикл URL",
        "revoke — обязательно",
        "Каждая createObjectURL удерживает Blob в памяти. revokeObjectURL \
         освобождает. Кнопка «Освободить все» отзывает все живые URL — \
         превью после этого перестанут подгружаться. Это и есть наглядная \
         цена не освобождать ресурсы.",
    )?;
    body.append_child(&sec3)?;

    let mem_panel = document.create_element("div")?;
    mem_panel.set_class_name("mem-panel");
    mem_panel.set_inner_html(
        "<span class=\"label\">живых URL:</span>\
         <span class=\"value\" id=\"url-count\">0</span>\
         <span class=\"spacer\"></span>",
    );
    body3.append_child(&mem_panel)?;

    let url_count_el = mem_panel
        .query_selector("#url-count")?
        .ok_or_else(|| JsValue::from_str("no url-count"))?;

    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    body3.append_child(&log_body)?;

    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        input: src_ta.clone(),
        cards_body,
        url_count_el,
        log_body,
        artifacts: RefCell::new(Vec::new()),
        live_urls: Cell::new(0),
        log_lines: RefCell::new(Vec::new()),
    });

    {
        let s = state.clone();
        make_button(
            &document,
            &row,
            "⚙ Сгенерировать",
            "Превратить текущий текст в четыре файла: TXT, JSON, HTML, PNG.",
            true,
            false,
            move || {
                generate(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &row,
            "↺ Сбросить текст",
            "Вернуть seed-текст и сгенерировать заново.",
            false,
            false,
            move || {
                s.input
                    .set_value("Hello, WASM! Этот текст превратится в четыре файла.");
                generate(s.clone());
            },
        )?;
    }

    {
        let s = state.clone();
        make_button(
            &document,
            &mem_panel,
            "Освободить все URL",
            "Отозвать все активные Object URL. Счётчик упадёт до 0, \
             кнопки «Скачать» станут неактивны, превью PNG перестанет грузиться.",
            false,
            true,
            move || {
                release_all_urls(&s);
                render_cards(&s);
                update_mem_panel(&s);
                set_status(&s, "ok", "все URL освобождены");
            },
        )?;
    }

    {
        let s = state.clone();
        console_log("авто-генерация при загрузке");
        generate(s);
    }

    console_log("run: finished OK");
    Ok(())
}
```

---

## 20.10. Что показывает демо

**Секция ❶** — textarea с исходным текстом и две кнопки: «Сгенерировать», «Сбросить текст».

**Секция ❷** — четыре карточки:

| Артефакт | MIME | Превью |
|---|---|---|
| `report.txt` | `text/plain;charset=utf-8` | `<pre>` с текстом |
| `report.json` | `application/json;charset=utf-8` | `<pre>` с pretty JSON |
| `report.html` | `text/html;charset=utf-8` | `<pre>` с кодом + `<iframe sandbox src="blob:...">` |
| `chart.png` | `image/png` | `<img src="blob:...">` — столбики по байтам текста |

У каждой карточки: имя файла, MIME, размер, превью, статус URL («активна» / «освобождена»), кнопка «Скачать».

**Секция ❸** — счётчик живых URL, кнопка «Освободить все URL», журнал операций.

### Wow-сценарии

**1. Автозапуск.** Открыли страницу — сразу видны четыре карточки с превью. Счётчик URL = 4.

**2. Разные форматы — разное превью.** TXT и JSON — текст. PNG — картинка со столбиками. HTML — код + живой рендер в iframe.

**3. Один URL — три роли.** В HTML-карточке видно: тот же `blob:...` URL служит `src` для `<iframe>` и `href` для `<a download>`. Наглядное доказательство, что URL — универсальный указатель.

**4. Скачивание без диалогов.** Клик «Скачать» — файл уходит в папку Загрузки. Без `showSaveFilePicker`, без user gesture.

**5. Освобождение URL.** Клик «Освободить все URL» — счётчик падает до 0, статус у каждой карточки меняется на «освобождена», кнопки «Скачать» блокируются. **PNG-превью** перестаёт грузиться при следующей перерисовке — прямое следствие revoke.

**6. Изменение текста → изменение PNG.** Смените текст, нажмите «Сгенерировать» — PNG-график изменится (столбики по байтам).

---

## 20.11. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты blob API в headless Chrome
wasm-pack test --headless --chrome --test wasm_blob_api

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** изменений. `image` уже подключён для PNG-генерации в демо.

---

## 20.12. Что дальше

Глава 20 закрывает «нижний» уровень работы с файлами: байты → Blob → URL → скачивание. Вместе с предыдущими главами это полный стек работы с данными в браузере:

- **19.7** — структура: дерево, пути, diff.
- **19.8** — производительность: OPFS, sync access handle.
- **19.9** — устойчивость: квота, persist.
- **19.10** — реактивность: FileSystemObserver.
- **19.11** — интеграция: Rust-крейты в браузере.
- **19.12** — практика: локальный CSV-редактор.
- **20** — выход: Blob, Object URL, скачивание.

Дальше логично перейти к **blob в worker'ах** (OffscreenCanvas, transferable objects), **File System Access API для экспорта** (`showSaveFilePicker` + `createWritable`), или к **стримингу** (`ReadableStream` → `Response` → `download` через `fetch`). Все принципы остаются те же: feature detection, идемпотентность, явная работа с ресурсами, единое визуальное поле.