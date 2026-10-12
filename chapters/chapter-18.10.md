# Глава 18. File API

## 18.10. `Blob.stream()` и потоковое чтение

Подглава 18.9 научила нас читать файл по частям через `Blob.slice`. Но у этого подхода есть недостаток: мы сами решаем, где чанки, и сами управляем чтением. Браузер умеет делать это эффективнее — через **`Blob.stream()`**, который возвращает `ReadableStream`. Поток сам отдаёт данные чанками (обычно по 64 КБ), сам применяет backpressure и сам сигнализирует о конце. Это тот же интерфейс, что используют fetch-ответы, `TransformStream` и вообще весь современный streaming API в браузере.

Главная идея этой подглавы — **файл не буферизуется в памяти целиком**. Мы обходим его чанками и накапливаем только метаданные. Именно поэтому поток обязателен для больших файлов: `readAsArrayBuffer` на 4-гигабайтном файле упадёт с `capacity overflow` в wasm32, где `usize` — 32 бита и `Vec<u8>` не может превысить ~2 ГБ.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod blob_stream;
│   ├── app.rs                     # ОБНОВЛЁН: секция 18.10
│   ├── blob_stream.rs             # НОВЫЙ: модуль подглавы 18.10
│   ├── blob_slice.rs              # из 18.9, не меняется
│   ├── validation.rs              # из 18.8, не меняется
│   ├── folder_drop.rs             # из 18.7, не меняется
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
│   └── wasm_blob_stream.rs        # НОВЫЙ: тесты подглавы 18.10
└── web/
    └── index.html                 # без изменений
```

**Пояснения:**

- `src/blob_stream.rs` — модуль, вводимый этой подглавой. Автономен: зависит только от `web-sys`, `js-sys`, `wasm-bindgen`, `wasm-bindgen-futures`.
- `src/lib.rs` — **обязательно** дополняется строкой `pub mod blob_stream;`. Без неё компилятор не видит файл, и `crate::blob_stream` не существует.
- `src/app.rs` — обновлён: секция «18.10. Blob.stream() — потоковое чтение».
- `tests/wasm_blob_stream.rs` — тесты подглавы.

Новых фич в `Cargo.toml` не требуется: `ReadableStream`, `ReadableStreamDefaultReader` включены ещё в главе 17.

---

## Ссылки для детального изучения

- **MDN: Blob.stream()** — https://developer.mozilla.org/en-US/docs/Web/API/Blob/stream
- **MDN: ReadableStream** — https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream
- **MDN: ReadableStreamDefaultReader** — https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader
- **MDN: Streams API concepts** — https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Concepts
- **web-sys: Blob** — https://docs.rs/web-sys/latest/web_sys/struct.Blob.html
- **web-sys: ReadableStreamDefaultReader** — https://docs.rs/web-sys/latest/web_sys/struct.ReadableStreamDefaultReader.html

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
pub mod folder_drop;    // 18.7
pub mod validation;     // 18.8
pub mod blob_slice;     // 18.9
pub mod blob_stream;    // 18.10 ← НОВОЕ
```

---

### `src/blob_stream.rs`

Модуль автономен. Ключевые типы и функции:

- `Chunk` — метаданные одного чанка. Размер одного чанка всегда укладывается в `u32`, но накопленная сумма — уже `f64`, потому что `u32` переполняется на файлах > 4 ГБ.
- `StreamStats` — итоговая статистика по всем чанкам.
- `stream_chunks` — основной API с callback на каждый чанк. Возвращает `f64` — суммарное число прочитанных байт.
- `stream_stats` — обёртка над `stream_chunks`, собирающая статистику.

```rust
//! Blob.stream() — потоковое чтение больших файлов.
//!
//! Модуль автономен: не зависит от других модулей проекта.
//!
//! Размеры байтов хранятся как `f64` — `u32` переполняется на
//! файлах > 4 ГБ, а `Blob::size()` возвращает именно `f64`.
//! `f64` точно представляет целые до 2^53 ≈ 9 ПБ.
//!
//! **Важно:** результат `reader.read()` — это plain JS-объект
//! `{ done: bool, value: any }`, а не инстанс типа
//! `ReadableStreamReadResult` из web-sys. Поля извлекаются через
//! `js_sys::Reflect::get` — у web-sys-типа есть только сеттеры,
//! геттеров нет.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, ReadableStreamDefaultReader};

/// Один прочитанный чанк из потока.
#[derive(Debug, Clone, PartialEq)]
pub struct Chunk {
    /// Порядковый номер чанка (с нуля).
    pub index: u32,
    /// Размер этого чанка в байтах. Один чанк всегда << 4 ГБ,
    /// поэтому `u32` достаточно.
    pub size: u32,
    /// Суммарный объём прочитанного, включая этот чанк.
    /// `f64` — чтобы не переполняться на файлах > 4 ГБ.
    pub total_so_far: f64,
}

/// Статистика потокового чтения.
#[derive(Debug, Clone, PartialEq)]
pub struct StreamStats {
    /// Суммарный объём всех чанков (f64 — не переполняется).
    pub total_bytes: f64,
    /// Количество чанков.
    pub chunk_count: u32,
    /// Размер самого маленького чанка.
    pub min_chunk: u32,
    /// Размер самого большого чанка.
    pub max_chunk: u32,
    /// Средний размер чанка.
    pub avg_chunk: f64,
}

/// Извлекает поля `done` и `value` из результата `reader.read()`.
///
/// web-sys в актуальной версии экспортирует `ReadableStreamReadResult`
/// только как словарь с сеттерами; геттеров нет. Поэтому читаем
/// поля через `Reflect::get` — это работает с plain JS-объектом,
/// который браузер возвращает из `reader.read()`.
fn extract_read_result(value: &JsValue) -> Result<Option<js_sys::Uint8Array>, JsValue> {
    let obj: &js_sys::Object = value
        .dyn_ref()
        .ok_or_else(|| JsValue::from_str("read() вернул не Object"))?;

    let done_js = js_sys::Reflect::get(obj, &JsValue::from_str("done"))?;
    let done: bool = done_js.as_bool().unwrap_or(false);
    if done {
        return Ok(None);
    }

    let chunk_js = js_sys::Reflect::get(obj, &JsValue::from_str("value"))?;
    let uint8: js_sys::Uint8Array = chunk_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("чанк не Uint8Array"))?;
    return Ok(Some(uint8));
}

/// Читает `Blob` потоково, вызывая `on_chunk` на каждый чанк.
///
/// Возвращает общее количество прочитанных байт как `f64`.
///
/// **Не буферизует байты.** Каждый чанк живёт ровно один шаг
/// цикла; после вызова `on_chunk` он освобождается GC.
/// Поэтому функция безопасна для файлов любого размера.
pub async fn stream_chunks<F>(blob: &Blob, mut on_chunk: F) -> Result<f64, JsValue>
where
    F: FnMut(&Chunk),
{
    let stream = blob.stream();
    let reader_js = stream.get_reader();
    let reader: ReadableStreamDefaultReader = reader_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ReadableStreamDefaultReader"))?;

    let mut index: u32 = 0;
    let mut total: f64 = 0.0;

    loop {
        // reader.read() возвращает Promise напрямую, без Result —
        // `?` применяется только к `.await`.
        let read_promise = reader.read();
        let value = JsFuture::from(read_promise).await?;

        let chunk_opt = extract_read_result(&value)?;
        let uint8 = match chunk_opt {
            Some(u) => u,
            None => break,
        };

        let size: u32 = uint8.length();
        total += size as f64;

        let chunk = Chunk {
            index,
            size,
            total_so_far: total,
        };
        on_chunk(&chunk);

        index = index.wrapping_add(1);
    }

    return Ok(total);
}

/// Читает `Blob` потоково и возвращает статистику по чанкам.
///
/// Полезно для демонстрации: сколько чанков отдал поток,
/// какие у них размеры, какова средняя длина.
pub async fn stream_stats(blob: &Blob) -> Result<StreamStats, JsValue> {
    let mut count: u32 = 0;
    let mut total: f64 = 0.0;
    let mut min_chunk: u32 = u32::MAX;
    let mut max_chunk: u32 = 0;

    let on_chunk = |chunk: &Chunk| {
        count = count.wrapping_add(1);
        total = chunk.total_so_far;
        if chunk.size < min_chunk {
            min_chunk = chunk.size;
        }
        if chunk.size > max_chunk {
            max_chunk = chunk.size;
        }
    };

    let _ = stream_chunks(blob, on_chunk).await?;

    let avg: f64;
    if count > 0 {
        avg = total / count as f64;
    } else {
        avg = 0.0;
    }

    let min_final: u32;
    if count > 0 {
        min_final = min_chunk;
    } else {
        min_final = 0;
    }

    return Ok(StreamStats {
        total_bytes: total,
        chunk_count: count,
        min_chunk: min_final,
        max_chunk,
        avg_chunk: avg,
    });
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — `JsValue`, `JsCast`.
- `wasm_bindgen::JsCast` — `.dyn_ref()`, `.dyn_into()`.
- `wasm_bindgen_futures::JsFuture` — `.await` на `Promise`.
- `web_sys::{Blob, ReadableStreamDefaultReader}` — оба используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- **`reader.read()` возвращает `Promise` напрямую** — не `Result<Promise, JsValue>`. Поэтому `?` ставится только на `.await`.
- **Результат `reader.read()` — plain JS-объект**, а не инстанс `ReadableStreamReadResult`. Геттеров у web-sys-типа нет, поэтому читаем поля через `Reflect::get`.
- **`f64` для накопленных размеров.** Это критично: файл 4.12 ГБ в `u32` дал бы `127 196 401` вместо `4 422 163 697` — байты не потеряются, но статистика соврёт.
- **Никакого накопления байтов.** Чанк живёт один шаг цикла. Память O(1) независимо от размера файла.
- **Все tail-функции через явный `return`** — конвенция проекта.

---

### `src/app.rs` — секция 18.10

Функция `build_blob_stream_section` самодостаточна: input, кнопка, прогресс-бар, результат.

```rust
//! Демонстрация подглавы 18.10: Blob.stream() — потоковое чтение.
//!
//! Файл НЕ буферизуется в памяти целиком: обходим его чанками,
//! обновляем прогресс, накапливаем только метаданные.
//! Память O(1) независимо от размера — работает и на файлах > 4 ГБ.
//!
//! Размеры в статистике хранятся как `f64`: `u32` переполняется
//! на файлах > 4 ГБ (именно так можно получить 127 МБ вместо 4.4 ГБ).

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{blob_stream, dom, file_api, log};

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "File API")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    build_blob_stream_section(&container, &closures)?;

    log!("Section 18.10 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.10: Blob.stream()
// ---------------------------------------------------------------

fn build_blob_stream_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("18.10. Blob.stream() — потоковое чтение"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Выберите файл — прочитаем его через Blob.stream(). Файл НЕ \
         буферизуется в памяти: обходим чанками (обычно по 64 КБ), \
         обновляем прогресс, накапливаем только метаданные. \
         Память O(1) независимо от размера — работает и на файлах \
         больше 4 ГБ, где readAsArrayBuffer падает с capacity overflow.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Прочитать потоком"));
    dom::append(&section, &btn)?;

    // Прогресс-бар создаём как Element: свойства `value` и `max`
    // обновляем через Reflect::set, поэтому фича HtmlProgressElement
    // в Cargo.toml не нужна.
    let progress: web_sys::Element = dom::create_element("progress")?;
    progress.set_attribute(
        "style",
        "width: 100%; height: 20px; display: block; margin: 8px 0;",
    )?;
    set_progress_max(&progress, 1.0);
    set_progress_value(&progress, 0.0);
    dom::append(&section, &progress)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файл и нажмите «Прочитать потоком»)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let input_c = input.clone();
    let result_c = result.clone();
    let progress_c = progress.clone();

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
            let progress_c = progress_c.clone();
            result_c.set_text_content(Some("Читаем потоком..."));
            set_progress_max(&progress_c, file.size().max(1.0));
            set_progress_value(&progress_c, 0.0);

            spawn_local(async move {
                let message = analyze_stream(file, progress_c.clone()).await;
                result_c.set_text_content(Some(&message));
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Потоковый обход с прогрессом
// ---------------------------------------------------------------

/// Аккумулятор статистики. Хранит только метаданные — байтов нет.
struct StatsAccum {
    count: u32,
    min: u32,
    max: u32,
    total: f64, // f64 — переживает файлы > 4 ГБ
}

impl StatsAccum {
    fn new() -> Self {
        return Self {
            count: 0,
            min: u32::MAX,
            max: 0,
            total: 0.0,
        };
    }
}

/// Читает файл потоково, обновляя прогресс-бар.
async fn analyze_stream(file: web_sys::File, progress: web_sys::Element) -> String {
    let file_name = file.name();
    let declared_size = file.size();

    let stats_ref: Rc<RefCell<StatsAccum>> = Rc::new(RefCell::new(StatsAccum::new()));
    let stats_ref_c = stats_ref.clone();
    let progress_c = progress.clone();

    let start = js_sys::Date::now();

    let total: f64 = match blob_stream::stream_chunks(&file, move |chunk| {
        set_progress_value(&progress_c, chunk.total_so_far);
        let mut s = stats_ref_c.borrow_mut();
        s.count = s.count.wrapping_add(1);
        s.total = chunk.total_so_far;
        if chunk.size < s.min {
            s.min = chunk.size;
        }
        if chunk.size > s.max {
            s.max = chunk.size;
        }
    })
    .await
    {
        Ok(t) => t,
        Err(e) => {
            let message = format!("❌ stream_chunks: {e:?}");
            return message;
        }
    };

    let elapsed = js_sys::Date::now() - start;
    let stats = stats_ref.borrow();

    return format_stream_report(
        &file_name,
        declared_size,
        stats.count,
        stats.min,
        stats.max,
        total,
        elapsed,
    );
}

/// Формирует отчёт о потоковом чтении.
fn format_stream_report(
    file_name: &str,
    declared_size: f64,
    chunk_count: u32,
    min_chunk: u32,
    max_chunk: u32,
    total_bytes: f64,
    elapsed_ms: f64,
) -> String {
    let avg_chunk: f64;
    if chunk_count > 0 {
        avg_chunk = total_bytes / chunk_count as f64;
    } else {
        avg_chunk = 0.0;
    }

    let min_display: u32;
    if chunk_count > 0 {
        min_display = min_chunk;
    } else {
        min_display = 0;
    }

    let speed_mbps: f64;
    if elapsed_ms > 0.0 {
        speed_mbps = declared_size / (elapsed_ms / 1000.0) / (1024.0 * 1024.0);
    } else {
        speed_mbps = 0.0;
    }

    let size_check: String;
    if total_bytes == declared_size {
        size_check = "✅ прочитано ровно столько, сколько заявлено".to_string();
    } else {
        let diff = declared_size - total_bytes;
        size_check = format!(
            "❌ размер не совпадает: прочитано {:.0}, ожидалось {:.0}, разница {:.0} байт",
            total_bytes, declared_size, diff,
        );
    }

    return format!(
        "Файл: {file_name}\n\
         Размер: {} ({:.0} байт)\n\n\
         === Поток: статистика (Blob.stream()) ===\n\
         Всего байт: {:.0}\n\
         Чанков: {}\n\
           минимум: {min_display} байт\n\
           максимум: {max_chunk} байт\n\
           средний: {avg_chunk:.1} байт\n\
         Время: {elapsed_ms:.1} мс\n\
         Скорость: {speed_mbps:.2} МБ/с\n\n\
         === Проверка ===\n\
         {size_check}\n\n\
         === Примечание ===\n\
         Файл НЕ буферизован в Vec<u8> — обходили чанками. \
         Размеры хранятся как f64: u32 переполняется на файлах > 4 ГБ.",
        file_api::format_file_size(declared_size),
        declared_size,
        total_bytes,
        chunk_count,
    );
}

// ---------------------------------------------------------------
// Обновление progress через Reflect::set
// ---------------------------------------------------------------

fn set_progress_max(progress: &web_sys::Element, max: f64) {
    let js: &JsValue = progress.unchecked_ref();
    let _ = js_sys::Reflect::set(
        js,
        &JsValue::from_str("max"),
        &JsValue::from_f64(max),
    );
}

fn set_progress_value(progress: &web_sys::Element, value: f64) {
    let js: &JsValue = progress.unchecked_ref();
    let _ = js_sys::Reflect::set(
        js,
        &JsValue::from_str("value"),
        &JsValue::from_f64(value),
    );
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — для `StatsAccum` между `stream_chunks` и внешним кодом.
- `wasm_bindgen::prelude::*` — `JsValue`.
- `wasm_bindgen::JsCast` — `.dyn_into()`, `.unchecked_ref()`.
- `wasm_bindgen_futures::spawn_local` — `async`-блок из обработчика.
- `callbacks`, `EventClosures`, `blob_stream`, `dom`, `file_api`, `log` — все используются.
- Неиспользуемых импортов нет.

**Особенности:**

- **`<progress>` как `Element`** — свойства `max` и `value` устанавливаем через `Reflect::set`. Это избавляет от необходимости подключать фичу `HtmlProgressElement` в web-sys.
- **`Rc<RefCell<StatsAccum>>`** — замыкание `FnMut` в `stream_chunks` обновляет аккумулятор; после завершения мы читаем его снаружи.
- **`declared_size` — `f64`** — совпадает с типом `Blob::size()` и `Chunk.total_so_far`. Никаких приведений типов.
- **Форматирование чисел `{:.0}`** — печатает `f64` без десятичной точки. Для точных значений < 2⁵³ это корректно.

---

### `tests/wasm_blob_stream.rs`

12 тестов. Проверяют типы (`f64`), монотонность, callback-поведение и недетерминированность разбиения.

```rust
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen_test::*;
use hello_wasm02::blob_stream::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Создаёт Blob из последовательности байт.
fn make_blob(bytes: &[u8]) -> web_sys::Blob {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::of1(&uint8.buffer());
    return web_sys::Blob::new_with_u8_array_sequence(&parts)
        .expect("Blob::new_with_u8_array_sequence");
}

#[wasm_bindgen_test]
async fn test_stream_chunks_small() {
    let original: Vec<u8> = (0..100u8).collect();
    let blob = make_blob(&original);

    let total: f64 = stream_chunks(&blob, |_c| {})
        .await
        .expect("stream_chunks");
    assert_eq!(total, original.len() as f64);
}

#[wasm_bindgen_test]
async fn test_stream_chunks_large() {
    let original: Vec<u8> = (0..200_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&original);

    let total: f64 = stream_chunks(&blob, |_c| {})
        .await
        .expect("stream_chunks");
    assert_eq!(total, original.len() as f64);
}

#[wasm_bindgen_test]
async fn test_stream_stats_matches_size() {
    let original: Vec<u8> = (0..150_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&original);

    let stats = stream_stats(&blob).await.expect("stream_stats");
    assert_eq!(stats.total_bytes, original.len() as f64);
    assert!(stats.chunk_count >= 1);
    assert!(stats.min_chunk > 0);
    assert!(stats.max_chunk >= stats.min_chunk);
    assert!(stats.avg_chunk > 0.0);
    assert!(stats.avg_chunk <= stats.max_chunk as f64);
}

#[wasm_bindgen_test]
async fn test_stream_empty_blob() {
    let blob = make_blob(&[]);
    let stats = stream_stats(&blob).await.expect("stream_stats");
    assert_eq!(stats.total_bytes, 0.0);
    assert_eq!(stats.chunk_count, 0);
    assert_eq!(stats.min_chunk, 0);
    assert_eq!(stats.max_chunk, 0);
    assert_eq!(stats.avg_chunk, 0.0);
}

#[wasm_bindgen_test]
async fn test_stream_chunks_callback() {
    let original: Vec<u8> = (0..100_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&original);

    let chunks: Rc<RefCell<Vec<Chunk>>> = Rc::new(RefCell::new(Vec::new()));
    let chunks_c = chunks.clone();

    let total: f64 = stream_chunks(&blob, move |c| {
        chunks_c.borrow_mut().push(c.clone());
    })
    .await
    .expect("stream_chunks");

    assert_eq!(total, original.len() as f64);

    let collected = chunks.borrow();
    assert!(!collected.is_empty());

    assert_eq!(collected[0].index, 0);
    for (i, c) in collected.iter().enumerate() {
        assert_eq!(c.index, i as u32);
    }

    let sum: u64 = collected.iter().map(|c| c.size as u64).sum();
    assert_eq!(sum, original.len() as u64);

    let last = collected.last().expect("last chunk");
    assert_eq!(last.total_so_far, original.len() as f64);
}

#[wasm_bindgen_test]
async fn test_total_so_far_monotonic() {
    let original: Vec<u8> = (0..100_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&original);

    let totals: Rc<RefCell<Vec<f64>>> = Rc::new(RefCell::new(Vec::new()));
    let totals_c = totals.clone();

    let _: f64 = stream_chunks(&blob, move |c| {
        totals_c.borrow_mut().push(c.total_so_far);
    })
    .await
    .expect("stream_chunks");

    let list = totals.borrow();
    assert!(!list.is_empty());

    for i in 1..list.len() {
        assert!(
            list[i] >= list[i - 1],
            "total_so_far должен монотонно расти: list[{}]={} < list[{}]={}",
            i,
            list[i],
            i - 1,
            list[i - 1],
        );
    }

    let last = *list.last().expect("last total");
    assert_eq!(last, original.len() as f64);
}

#[wasm_bindgen_test]
async fn test_total_so_far_is_f64() {
    let original: Vec<u8> = vec![0u8; 100];
    let blob = make_blob(&original);

    let captured: Rc<RefCell<Option<f64>>> = Rc::new(RefCell::new(None));
    let captured_c = captured.clone();

    let _: f64 = stream_chunks(&blob, move |c| {
        *captured_c.borrow_mut() = Some(c.total_so_far);
    })
    .await
    .expect("stream_chunks");

    let value: f64 = captured.borrow().expect("captured value");
    assert_eq!(value, 100.0);
}

#[wasm_bindgen_test]
async fn test_stream_chunks_returns_f64() {
    let original: Vec<u8> = vec![1u8; 50];
    let blob = make_blob(&original);

    let total: f64 = stream_chunks(&blob, |_c| {})
        .await
        .expect("stream_chunks");
    assert_eq!(total, 50.0);
}

#[wasm_bindgen_test]
async fn test_total_bytes_is_f64() {
    let original: Vec<u8> = vec![2u8; 50];
    let blob = make_blob(&original);

    let stats = stream_stats(&blob).await.expect("stream_stats");
    let total: f64 = stats.total_bytes;
    assert_eq!(total, 50.0);
}

#[wasm_bindgen_test]
async fn test_f64_represents_large_sizes() {
    let big: f64 = 4_422_163_697.0;
    assert!(big > u32::MAX as f64, "значение должно быть > u32::MAX");

    let recovered: u64 = big as u64;
    assert_eq!(recovered, 4_422_163_697u64);

    let wrapped_mod: u64 = (big as u64) % (u32::MAX as u64 + 1);
    assert_eq!(wrapped_mod, 127_196_401u64);
}

#[wasm_bindgen_test]
async fn test_stream_stats_avg_chunk_bounds() {
    let original: Vec<u8> = (0..200_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&original);

    let stats = stream_stats(&blob).await.expect("stream_stats");
    assert!(stats.avg_chunk >= stats.min_chunk as f64);
    assert!(stats.avg_chunk <= stats.max_chunk as f64);
}

#[wasm_bindgen_test]
async fn test_stream_total_bytes_is_deterministic() {
    // Единственный стабильный инвариант двух проходов — `total_bytes`.
    //
    // `Blob.stream()` НЕ гарантирует одинаковое разбиение на чанки
    // между вызовами: браузер может отдать 1 чанк из 100 000 байт
    // в одном проходе и 2 чанка (65 536 + 34 464) в другом.
    // Поэтому `chunk_count`, `min_chunk`, `max_chunk` и `avg_chunk`
    // сравнивать между проходами нельзя.
    let original: Vec<u8> = (0..100_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&original);

    let stats1 = stream_stats(&blob).await.expect("stats1");
    let stats2 = stream_stats(&blob).await.expect("stats2");

    assert_eq!(stats1.total_bytes, stats2.total_bytes);
    assert_eq!(stats1.total_bytes, original.len() as f64);

    assert!(stats1.chunk_count >= 1);
    assert!(stats2.chunk_count >= 1);
}
```

**Проверка импортов:**

- `std::cell::RefCell`, `std::rc::Rc` — используются в тестах с callback.
- `wasm_bindgen_test::*` — `wasm_bindgen_test`, `wasm_bindgen_test_configure`.
- `hello_wasm02::blob_stream::*` — `stream_chunks`, `stream_stats`, `Chunk`, `StreamStats`.

`JsCast` не импортируется — в этом файле нет ни `dyn_into`, ни `unchecked_ref`. Полные пути `js_sys::*`, `web_sys::*` — без импортов, как внешние crate. Неиспользуемых импортов нет.

**Что проверяют тесты:**

- **Типы** — три compile-time проверки: `Chunk.total_so_far: f64`, `stream_chunks` возвращает `f64`, `StreamStats.total_bytes: f64`. При смене типа на `u32` эти тесты перестанут собираться.
- **Корректность** — на маленьком и большом Blob суммарный размер совпадает с исходным.
- **Монотонность** — `total_so_far` в callback растёт: 0 → 65536 → 100000, никогда не уменьшается.
- **Пустой Blob** — нулевая статистика без паники.
- **Порядок чанков** — `index` идёт 0, 1, 2, ...
- **Связка min/max/avg** — среднее лежит между min и max.
- **Недетерминированность разбиения** — тест на двух проходах проверяет только `total_bytes`, потому что `chunk_count` может отличаться от прохода к проходу.
- **Границы `f64`** — значение 4 422 163 697 представляет собой пример с реального 4-гигабайтного файла и корректно восстанавливается через `as u64`; а `% (u32::MAX + 1)` даёт то самое ошибочное значение 127 196 401, которое показала бы статистика в `u32`.

---

## Команды запуска

### Сборка и запуск демо

```bash
cargo check --target wasm32-unknown-unknown
npm run wasm:dev
npm run dev
# или одной командой:
npm run start
```

### Тесты подглавы 18.10

```bash
cargo check --target wasm32-unknown-unknown --tests

wasm-pack test --headless --chrome --test wasm_blob_stream
wasm-pack test --headless --firefox --test wasm_blob_stream

npm run test
```

### Что ожидать в браузере

После `npm run start` появится секция **18.10. Blob.stream() — потоковое чтение** с полем выбора файла, кнопкой «Прочитать потоком» и прогресс-баром.

**Сценарий 1 — маленький файл (10 КБ):**
```
Файл: small.txt
Размер: 10.00 KB (10240 байт)

=== Поток: статистика (Blob.stream()) ===
Всего байт: 10240
Чанков: 1
  минимум: 10240 байт
  максимум: 10240 байт
  средний: 10240.0 байт
Время: 0.5 мс
Скорость: 19.53 МБ/с

=== Проверка ===
✅ прочитано ровно столько, сколько заявлено

=== Примечание ===
Файл НЕ буферизован в Vec<u8> — обходили чанками.
```

**Сценарий 2 — большой файл (4.12 ГБ):**
```
Файл: 5417632533003.mp4
Размер: 4.12 GB (4422163697 байт)

=== Поток: статистика (Blob.stream()) ===
Всего байт: 4422163697
Чанков: 85097
  минимум: 4096 байт
  максимум: 65536 байт
  средний: 51959.6 байт
Время: 10607.0 мс
Скорость: 397.60 МБ/с

=== Проверка ===
✅ прочитано ровно столько, сколько заявлено
```

Прогресс-бар плавно заполняется по мере чтения. Память не растёт независимо от размера файла.

---

## Итоги подглавы 18.10

- **`Blob::stream()`** возвращает `ReadableStream` — тот же интерфейс, что у fetch-ответов и `TransformStream`.
- **`reader.read()`** возвращает `Promise` **напрямую** — не `Result<Promise, JsValue>`. `?` применяется только к `.await`.
- **Результат `reader.read()`** — plain JS-объект `{ done, value }`. Читаем поля через `Reflect::get`; у web-sys-типа есть только сеттеры.
- **`f64` для всех накопленных размеров.** `u32` молча переполняется на файлах > 4 ГБ; `Blob::size()` и `Chunk.total_so_far` — `f64`.
- **Никакого `Vec<u8>` для больших файлов.** В wasm32 `usize` — 32 бита, `Vec<u8>` не может превысить ~2 ГБ. На файле 4 ГБ попытка собрать всё в вектор падает с `capacity overflow`.
- **Разбиение на чанки недетерминировано.** Только `total_bytes` — стабильный инвариант. `chunk_count`, `min`, `max`, `avg` могут отличаться между вызовами для одного Blob.
- **Память O(1).** Каждый чанк живёт один шаг цикла. Аккумулируем только метаданные.

Модуль `blob_stream.rs` автономен, все tail-функции через `return`, неиспользуемых импортов нет. Демо-секция самодостаточна и демонстрирует потоковое чтение с прогресс-баром. Тесты покрывают корректность, монотонность, типы и недетерминированность разбиения.

## Замечания на будущее

Пять правил, зафиксированных в этой подглаве:

1. **`u32` переполняется на файлах > 4 ГБ молча.** `wrapping_add` не паникует — просто даёт неправильный результат. Признак проблемы: значение `total mod 2³²`. Используйте `f64` для накопленных сумм байтов.

2. **`reader.read()` возвращает `Promise`, а не `Result`.** Если компилятор пишет `the '?' operator can only be applied to values that implement 'Try' — the trait 'Try' is not implemented for 'Promise'` — уберите `?` у вызова и оставьте его у `.await`.

3. **Для dictionary-типов web-sys (например, `ReadableStreamReadResult`) используйте `js_sys::Reflect::get`.** Геттеров у них нет — только конструктор и сеттеры.

4. **`Blob.stream()` недетерминирован.** Разбиение на чанки может отличаться между вызовами одного Blob. В тестах и в UI опирайтесь только на `total_so_far` и `total_bytes`.

5. **`Vec<u8>` не выдержит большой файл.** В wasm32 максимальный размер вектора — около 2 ГБ. Для чтения больших файлов — только поток с callback-обработкой, без накопления байтов.

В следующей подглаве (18.11) мы перейдём к **Web Workers** — вынесем тяжёлую файловую обработку в отдельный поток, чтобы UI оставался отзывчивым.