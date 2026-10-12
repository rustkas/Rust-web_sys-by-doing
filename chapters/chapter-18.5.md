# Глава 18. File API

## 18.5. Data URL, base64 и crate `base64`

В подглаве 18.4 мы превращали `Blob` в Object URL — временную ссылку `blob:...`. Сегодня разберём альтернативу: **Data URL** — способ упаковать бинарные данные прямо в строку вида `data:image/png;base64,iVBOR…`. Такая строка самодостаточна: её можно сохранить в JSON, вставить в CSS, отправить в письме или записать в базу данных — она везде сработает без сервера. Внутри Data URL почти всегда лежит **base64** — способ кодировать произвольные байты в ASCII-символы. Мы разберём обе стороны: чтение `Blob` как Data URL (браузерный API `FileReader.readAsDataURL`) и кодирование/декодирование base64 на стороне Rust через crate `base64`.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # base64 = "0.23" уже есть
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod data_url; + pub mod base64_utils;
│   ├── app.rs                     # ОБНОВЛЁН: добавлена секция 18.5
│   ├── data_url.rs                # НОВЫЙ: модуль подглавы 18.5
│   ├── base64_utils.rs            # НОВЫЙ: модуль подглавы 18.5
│   ├── object_url.rs              # из 18.4, не меняется
│   ├── file_utils.rs              # из 18.3, не меняется
│   ├── file_api.rs                # из 18.1, не меняется
│   ├── file_reader.rs             # из 18.2, не меняется
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует (экспортирует log!)
│   └── … остальные модули проекта без изменений
├── tests/
│   └── wasm_data_url.rs           # НОВЫЙ: тесты подглавы 18.5
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/data_url.rs` — модуль для чтения `Blob` как Data URL через браузерный `FileReader.readAsDataURL` плюс парсер Data URL.
- `src/base64_utils.rs` — модуль для кодирования/декодирования base64 на стороне Rust через crate `base64`. Автономен от остальных модулей проекта.
- `src/lib.rs` — **обязательно** дополняется двумя строками `pub mod data_url;` и `pub mod base64_utils;`.
- `src/app.rs` — обновлён: добавлена демо-секция «18.5. Data URL, base64».
- `tests/wasm_data_url.rs` — новый файл с тестами.

`base64 = "0.23"` уже присутствует в `Cargo.toml` — теперь он наконец используется.

---

## Ссылки для детального изучения

- **MDN: Data URLs** — https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs
- **MDN: FileReader.readAsDataURL()** — https://developer.mozilla.org/en-US/docs/Web/API/FileReader/readAsDataURL
- **MDN: btoa()** — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/btoa
- **MDN: atob()** — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/atob
- **RFC 4648 — Base16, Base32, Base64** — https://datatracker.ietf.org/doc/html/rfc4648
- **crate `base64` — документация** — https://docs.rs/base64/latest/base64/
- **web-sys: FileReader** — https://docs.rs/web-sys/latest/web_sys/struct.FileReader.html

---

## Содержание новых / обновлённых файлов

### `src/lib.rs` — подключение двух модулей

```rust
// src/lib.rs

// … существующие модули проекта …

// ---- Глава 18. File API ----
pub mod file_api;       // 18.1
pub mod file_reader;    // 18.2
pub mod file_utils;     // 18.3
pub mod object_url;     // 18.4
pub mod data_url;       // 18.5 ← НОВОЕ
pub mod base64_utils;   // 18.5 ← НОВОЕ
```

Без этих двух строк `crate::data_url` и `crate::base64_utils` не существуют для компилятора, и `app.rs` не соберётся с ошибкой `unresolved import`.

---

### `src/data_url.rs`

Модуль автономен: зависимости только от `wasm-bindgen`, `wasm-bindgen-futures`, `js-sys`, `web-sys`. Он умеет читать `Blob` как Data URL и разбирать готовую строку на составные части.

```rust
//! Data URL — представление бинарных данных в виде `data:` URL.
//! Модуль автономен: не зависит от других модулей проекта.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, FileReader};

/// Разобранные части Data URL.
#[derive(Debug, Clone, PartialEq)]
pub struct DataUrlParts {
    /// MIME-тип из префикса, например `image/png`.
    /// Для `data:,hello` возвращается `text/plain;charset=US-ASCII`
    /// (значение по умолчанию согласно спецификации).
    pub mime_type: String,
    /// `true`, если payload закодирован в base64 (`;base64,`).
    pub is_base64: bool,
    /// Полезная нагрузка после запятой — base64-строка
    /// или URL-encoded содержимое.
    pub payload: String,
}

/// Асинхронно читает `Blob` как Data URL.
///
/// Возвращает строку вида `data:image/png;base64,iVBORw0KGgo=`.
/// Для больших файлов длина строки примерно в 1.37 раза больше
/// размера исходных данных.
pub async fn read_as_data_url(blob: &Blob) -> Result<String, JsValue> {
    let reader = FileReader::new()?;
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let reader_clone = reader.clone();
        let onload = Closure::once_into_js(move || {
            let _ = resolve.call0(&JsValue::NULL);
        });
        let onerror = Closure::once_into_js(move |err: JsValue| {
            let _ = reject.call1(&JsValue::NULL, &err);
        });
        reader_clone.set_onload(Some(onload.unchecked_ref()));
        reader_clone.set_onerror(Some(onerror.unchecked_ref()));
    });

    reader.read_as_data_url(blob)?;
    JsFuture::from(promise).await?;

    return reader
        .result()?
        .as_string()
        .ok_or_else(|| JsValue::from_str("Data URL не является строкой"));
}

/// Разбирает Data URL на части.
///
/// Формат: `data:[<mediatype>][;base64],<data>`
///
/// * `mediatype` может отсутствовать — тогда возвращается
///   `text/plain;charset=US-ASCII` (значение по умолчанию).
/// * `;base64` может отсутствовать — тогда payload в URL-encoded виде,
///   и `is_base64` будет `false`.
///
/// Возвращает `None`, если строка не начинается с `data:` или
/// не содержит запятой.
pub fn parse_data_url(url: &str) -> Option<DataUrlParts> {
    let after_data = url.strip_prefix("data:")?;
    let comma = after_data.find(',')?;
    let (meta, rest) = after_data.split_at(comma);
    // rest начинается с запятой — пропускаем её.
    let payload = &rest[1..];

    let is_base64 = meta.ends_with(";base64");
    let mime_raw = if is_base64 {
        &meta[..meta.len() - 7] // ";base64" — ровно 7 символов
    } else {
        meta
    };

    let mime_type = if mime_raw.is_empty() {
        "text/plain;charset=US-ASCII".to_string()
    } else {
        mime_raw.to_string()
    };

    return Some(DataUrlParts {
        mime_type,
        is_base64,
        payload: payload.to_string(),
    });
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — для `JsValue`, `Closure`, `JsCast` (`unchecked_ref`).
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{Blob, FileReader}` — оба используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- `read_as_data_url` полностью симметрична `read_as_text` из подглавы 18.2: та же обёртка `Promise` вокруг событий `onload` / `onerror`.
- `parse_data_url` — чистая функция без зависимостей от DOM: работает со строкой, легко тестируется.
- `meta.len() - 7` — корректно, потому что `;base64` это ASCII-строка фиксированной длины.
- Для `data:,hello` возвращается MIME `text/plain;charset=US-ASCII` — значение по умолчанию из спецификации.

---

### `src/base64_utils.rs`

Модуль автономен: зависит только от crate `base64`. Никаких `web-sys`, `js-sys`, `wasm-bindgen` — это чистый Rust-код, который работает и в браузере, и на сервере.

```rust
//! Base64 — кодирование и декодирование через crate `base64`.
//! Модуль автономен: работает на любой платформе, не только в браузере.

use base64::engine::general_purpose::STANDARD;
use base64::Engine;

/// Кодирует байты в base64 (стандартный алфавит с padding `=`).
///
/// Соответствует тому, что выдаёт браузерная `btoa` для ASCII-строк.
pub fn encode(bytes: &[u8]) -> String {
    return STANDARD.encode(bytes);
}

/// Декодирует base64-строку в байты.
///
/// Возвращает `Err(String)` с описанием проблемы, если строка
/// не является валидным base64.
pub fn decode(s: &str) -> Result<Vec<u8>, String> {
    return STANDARD.decode(s).map_err(|e| e.to_string());
}
```

**Проверка импортов:**

- `base64::engine::general_purpose::STANDARD` — используется.
- `base64::Engine` — trait, дающий методы `.encode()` и `.decode()`.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- `STANDARD` — движок с классическим алфавитом (`A–Z a–z 0–9 + /`) и padding `=`.
- Обе функции оформлены с явным `return`.
- `decode` возвращает `Err(String)`, а не `Err(JsValue)` — модуль не зависит от JS.

---

### `src/app.rs` — новая секция 18.5

Ниже — обновлённый блок для `app.rs`. Функция `build_data_url_section` самодостаточна: её можно вставить в `run()` рядом с другими секциями без правок последних.

```rust
//! Демонстрация подглавы 18.5: Data URL, base64.
//!
//! Секция: выбрать файл → прочитать как Data URL → превью
//! изображения, длина строки, overhead, сравнение с Rust crate.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{base64_utils, data_url, dom, file_reader, file_utils, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // ---- Секция 18.5 ----
    build_data_url_section(&container, &closures)?;

    log!("Section 18.5 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.5: Data URL, base64
// ---------------------------------------------------------------

fn build_data_url_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.5. Data URL, base64"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите файл — он будет прочитан как Data URL через \
         FileReader.readAsDataURL. Мы покажем длину строки, overhead \
         относительно исходного размера и сравним браузерный base64 \
         с результатом Rust crate `base64`.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Прочитать как Data URL"));
    dom::append(&section, &btn)?;

    let img: web_sys::HtmlImageElement = dom::create_element("img")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не img"))?;
    img.set_attribute(
        "style",
        "max-width: 300px; max-height: 300px; display: block; margin: 10px 0;",
    )?;
    dom::append(&section, &img)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(результат появится здесь)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let input_c = input.clone();
    let img_c = img.clone();
    let result_c = result.clone();

    callbacks::listen(
        &btn,
        "click",
        move |_ev: web_sys::Event| {
            let file = match input_c.files().and_then(|f| f.get(0)) {
                Some(f) => f,
                None => {
                    result_c.set_text_content(Some("Сначала выберите файл."));
                    return;
                }
            };
            let result_c = result_c.clone();
            let img_c = img_c.clone();
            result_c.set_text_content(Some("Читаем..."));
            spawn_local(async move {
                // --- 1. Читаем как Data URL через FileReader. ---
                let url = match data_url::read_as_data_url(&file).await {
                    Ok(s) => s,
                    Err(e) => {
                        let message = format!("❌ read_as_data_url: {e:?}");
                        result_c.set_text_content(Some(&message));
                        return;
                    }
                };

                // --- 2. Превью: если это изображение, ставим src. ---
                if file.type_().starts_with("image/") {
                    img_c.set_src(&url);
                } else {
                    img_c.remove_attribute("src").ok();
                }

                // --- 3. Читаем тот же файл как ArrayBuffer для Rust base64. ---
                let ab_js = match file_reader::read_as_array_buffer(&file).await {
                    Ok(v) => v,
                    Err(e) => {
                        let message = format!("❌ read_as_array_buffer: {e:?}");
                        result_c.set_text_content(Some(&message));
                        return;
                    }
                };
                let array_buffer: js_sys::ArrayBuffer = match ab_js.dyn_into() {
                    Ok(a) => a,
                    Err(_) => {
                        result_c.set_text_content(Some("не ArrayBuffer"));
                        return;
                    }
                };
                let bytes = file_utils::array_buffer_to_vec(&array_buffer);
                let rust_b64 = base64_utils::encode(&bytes);

                // --- 4. Разбираем Data URL и сравниваем payload. ---
                let parts = match data_url::parse_data_url(&url) {
                    Some(p) => p,
                    None => {
                        result_c.set_text_content(Some("не удалось разобрать Data URL"));
                        return;
                    }
                };

                let message = format_data_url_report(
                    &file.name(),
                    bytes.len(),
                    &url,
                    &parts.payload,
                    &rust_b64,
                    &parts.mime_type,
                    parts.is_base64,
                );
                result_c.set_text_content(Some(&message));
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции секции 18.5
// ---------------------------------------------------------------

/// Формирует отчёт о Data URL: длины, overhead, сравнение с crate.
///
/// Ветка `if` использует обычный `format!` без `return`: `return`
/// внутри ветки завершил бы функцию целиком и вернул бы только
/// усечённый preview. Явный `return format!(...)` стоит в tail-позиции.
fn format_data_url_report(
    file_name: &str,
    file_size: usize,
    data_url: &str,
    browser_b64: &str,
    rust_b64: &str,
    mime: &str,
    is_base64: bool,
) -> String {
    let url_len = data_url.len();
    let b64_len = browser_b64.len();

    let overhead_pct: f64 = if file_size > 0 {
        (b64_len as f64 - file_size as f64) / file_size as f64 * 100.0
    } else {
        0.0
    };

    let match_status: &str = if browser_b64 == rust_b64 {
        "✅ совпадают — Rust crate `base64` и браузерный readAsDataURL \
         дают идентичный результат"
    } else {
        "❌ НЕ совпадают — проверьте алфавит / padding"
    };

    let preview: String = if data_url.len() > 120 {
        format!("{}...", &data_url[..120])
    } else {
        data_url.to_string()
    };

    return format!(
        "✅ Data URL готов\n\
         Файл: {file_name}\n\
         MIME: {mime}\n\
         is_base64: {is_base64}\n\n\
         Размер файла: {file_size} байт\n\
         Длина payload (base64): {b64_len} символов\n\
         Длина всей Data URL: {url_len} символов\n\
         Overhead base64 относительно файла: {overhead_pct:.1}%\n\n\
         Сравнение браузерного base64 с Rust crate:\n\
         Rust crate вернул: {} символов\n\
         {match_status}\n\n\
         Первые 120 символов Data URL:\n{preview}",
        rust_b64.len()
    );
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — используются в `run()` и `build_data_url_section`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`.
- `wasm_bindgen_futures::spawn_local` — для `async`-блока из обработчика.
- `callbacks`, `EventClosures`, `base64_utils`, `data_url`, `dom`, `file_reader`, `file_utils`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- Один и тот же файл читается дважды: один раз как Data URL, второй раз как `ArrayBuffer` — чтобы закодировать в Rust и сравнить.
- `data_url::parse_data_url` извлекает base64-часть из готовой строки.
- `format_data_url_report` вычисляет overhead: для base64 он всегда ≈ 33 % (4 символа на каждые 3 байта), плюс overhead от префикса `data:...;base64,`.
- `&data_url[..120]` безопасно, потому что Data URL состоит только из ASCII-символов — границы char и byte совпадают.

**Важное исправление в `format_data_url_report`:**

- В ветке `if data_url.len() > 120 { ... }` — **обычный `format!` без `return`**. Раньше здесь стоял `return`, и это приводило к досрочному выходу из функции: вместо полного отчёта возвращался только усечённый preview.
- В tail-позиции функции — **`return format!(...)`** согласно конвенции проекта.

---

### `tests/wasm_data_url.rs`

Тесты подглавы 18.5. Проверяют round-trip base64, сравнение с браузерной `btoa` и парсинг Data URL.

```rust
use wasm_bindgen_test::*;
use hello_wasm02::base64_utils;
use hello_wasm02::data_url;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_base64_roundtrip_ascii() {
    let original = "Hello, WASM!";
    let encoded = base64_utils::encode(original.as_bytes());
    assert_eq!(encoded, "SGVsbG8sIFdBU00h");

    let decoded = base64_utils::decode(&encoded).expect("decode");
    assert_eq!(decoded, original.as_bytes());
}

#[wasm_bindgen_test]
fn test_base64_roundtrip_binary() {
    let original: Vec<u8> = vec![0x00, 0x01, 0x02, 0x7F, 0x80, 0xFE, 0xFF];
    let encoded = base64_utils::encode(&original);
    let decoded = base64_utils::decode(&encoded).expect("decode");
    assert_eq!(original, decoded);
}

#[wasm_bindgen_test]
fn test_base64_decode_invalid() {
    // Символ '*' не входит в стандартный алфавит base64.
    let result = base64_utils::decode("SGVsbG8*");
    assert!(result.is_err(), "ожидалась ошибка декодирования");
}

#[wasm_bindgen_test]
fn test_base64_matches_browser_btoa() {
    // btoa работает только для ASCII (latin1).
    let s = "Hello, WASM!";
    let rust_encoded = base64_utils::encode(s.as_bytes());

    let btoa = js_sys::Function::new_with_args("s", "return btoa(s);");
    let browser_result = btoa
        .call1(
            &wasm_bindgen::JsValue::NULL,
            &wasm_bindgen::JsValue::from_str(s),
        )
        .expect("btoa")
        .as_string()
        .expect("btoa вернул строку");

    assert_eq!(
        rust_encoded, browser_result,
        "Rust crate `base64` и браузерная btoa должны давать одинаковый результат"
    );
}

#[wasm_bindgen_test]
fn test_parse_data_url_base64() {
    let url = "data:image/png;base64,iVBORw0KGgo=";
    let parts = data_url::parse_data_url(url).expect("parse");
    assert_eq!(parts.mime_type, "image/png");
    assert!(parts.is_base64);
    assert_eq!(parts.payload, "iVBORw0KGgo=");
}

#[wasm_bindgen_test]
fn test_parse_data_url_plain() {
    let url = "data:text/plain,Hello%20World";
    let parts = data_url::parse_data_url(url).expect("parse");
    assert_eq!(parts.mime_type, "text/plain");
    assert!(!parts.is_base64);
    assert_eq!(parts.payload, "Hello%20World");
}

#[wasm_bindgen_test]
fn test_parse_data_url_default_mime() {
    // data:,hello — без MIME; по спецификации text/plain;charset=US-ASCII.
    let url = "data:,hello";
    let parts = data_url::parse_data_url(url).expect("parse");
    assert_eq!(parts.mime_type, "text/plain;charset=US-ASCII");
    assert!(!parts.is_base64);
    assert_eq!(parts.payload, "hello");
}

#[wasm_bindgen_test]
fn test_parse_data_url_invalid() {
    assert!(data_url::parse_data_url("http://example.com").is_none());
    assert!(data_url::parse_data_url("data:image/png;base64").is_none());
}

#[wasm_bindgen_test]
async fn test_read_as_data_url_from_blob() {
    let blob = web_sys::Blob::new_with_str_sequence(
        &js_sys::Array::of1(&"hello".into()),
    )
    .expect("Blob::new_with_str_sequence");

    let url = data_url::read_as_data_url(&blob)
        .await
        .expect("read_as_data_url");
    assert!(url.starts_with("data:"), "Data URL должен начинаться с 'data:'");
    assert!(url.contains(";base64,"), "Data URL должен содержать ';base64,'");

    let parts = data_url::parse_data_url(&url).expect("parse");
    let decoded = base64_utils::decode(&parts.payload).expect("decode");
    assert_eq!(decoded, b"hello");
}
```

**Что проверяем:**

- **Round-trip ASCII** — `encode("Hello, WASM!")` даёт `SGVsbG8sIFdBU00h`.
- **Round-trip binary** — байты с `0x00`, `0xFF`, `0x80` переживают encode/decode.
- **Невалидный base64** — символ `*` вызывает ошибку.
- **Совпадение с `btoa`** — Rust crate и браузер используют один алфавит.
- **Парсинг Data URL** — с MIME и `;base64`, без `;base64`, без MIME.
- **Интеграционный тест** — `read_as_data_url` + `parse_data_url` + `decode` дают исходные байты.

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

### Тесты подглавы 18.5

```bash
# Проверка типов тестов
cargo check --target wasm32-unknown-unknown --tests

# Только тесты этой подглавы
wasm-pack test --headless --chrome --test wasm_data_url

# То же самое для Firefox
wasm-pack test --headless --firefox --test wasm_data_url

# Все тесты проекта
npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.5. Data URL, base64** с полем выбора файла и кнопкой «Прочитать как Data URL». После выбора PNG-файла размером 5000 байт и нажатия кнопки отобразится:

```
✅ Data URL готов
Файл: logo.png
MIME: image/png
is_base64: true

Размер файла: 5000 байт
Длина payload (base64): 6668 символов
Длина всей Data URL: 6690 символов
Overhead base64 относительно файла: 33.4%

Сравнение браузерного base64 с Rust crate:
Rust crate вернул: 6668 символов
✅ совпадают — Rust crate `base64` и браузерный readAsDataURL дают идентичный результат

Первые 120 символов Data URL:
data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA…
```

Хорошо видно, что base64-представление примерно **на 33 % длиннее** исходных байтов. Это цена за то, что данные теперь можно хранить в виде текста. Для сравнения: Object URL из подглавы 18.4 не имел такого overhead — там байты оставались бинарными, а мы получали лишь короткую ссылку. Зато Data URL самодостаточен: его можно сохранить в JSON-поле базы данных и потом использовать где угодно, даже вне браузера.

---

## Сравнение Object URL и Data URL

| Критерий | Object URL (`blob:...`) | Data URL (`data:...;base64,...`) |
|---|---|---|
| Размер строки | ~50 символов | ~1.37 × размер файла |
| Память | Ссылка на существующий Blob | Копия данных в виде строки |
| Время жизни | До `revokeObjectURL` или выгрузки страницы | Бессрочно |
| Переживает перезагрузку | ❌ | ✅ |
| Работает вне браузера | ❌ | ✅ |
| Годится для `<img src>`, `<a href>` | ✅ | ✅ |
| Годится для JSON, писем, БД | ❌ | ✅ |
| Может быть в CSS inline | ❌ | ✅ |
| Подходит для больших файлов | ✅ | ⚠️ Только для мелких |
| Утечка памяти при неаккуратности | Да, без `revoke` | Нет |

**Правило выбора:**

- Файл в интерфейсе → **Object URL**.
- Данные для хранения, JSON, письма, CSS → **Data URL** (если файл маленький).
- Файл больше 1 МБ → **Object URL** или потоковая загрузка.

---

## Итоги подглавы 18.5

- **Data URL** — самодостаточная строка `data:<mime>[;base64],<payload>`, переносящая бинарные данные в текстовом виде.
- **`FileReader.readAsDataURL`** — браузерный способ превратить `Blob` в Data URL.
- **base64** — способ кодировать байты в ASCII-алфавит, увеличивая размер на ~33 %.
- **Crate `base64`** — кодирует/декодирует base64 на стороне Rust через движок `STANDARD` и trait `Engine`.
- **Результаты совпадают** — Rust crate и браузерный `readAsDataURL` используют один и тот же алфавит и padding.
- **Object URL vs Data URL** — не конкуренты, а инструменты для разных сценариев.

Модуль `data_url.rs` автономен, `base64_utils.rs` — чистый Rust без зависимостей от WASM-окружения. Демо-секция самодостаточна, тесты покрывают как кодирование, так и парсинг, и интеграцию. **9 тестов прошли успешно** — это подтверждает корректность round-trip и совпадение с браузерным base64.

## Замечание на будущее

Два правила, зафиксированные в этой подглаве:

1. **`return format!(...)` в tail-позиции функции, но не внутри ветки `if`.** Внутри `let`-биндингов и веток `if/else` — обычный `format!` без `return`. Иначе `return` завершит функцию досрочно, и результат окажется неполным.

2. **Каждый новый модуль, использующий внешний crate, обязан проверить, что crate уже в `Cargo.toml`.** Для этой подглавы `base64 = "0.23"` был добавлен ещё в главе 17 и до сих пор висел мёртвым грузом. Теперь он используется.

В следующей подглаве (18.6) мы вернёмся к Drag & Drop — второй по важности после `<input type="file">` способ получения файлов от пользователя.