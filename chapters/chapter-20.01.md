# 20.1. Blob как контейнер бинарных данных

## Обновлённая структура проекта

```
hello-wasm02/
├── src/
│   ├── blob_api.rs                ← из главы 20: Blob → URL → download
│   ├── blob_container.rs           ← НОВЫЙ: Blob как контейнер байтов
│   ├── app.rs                      ← ДЕМО подглавы 20.1 (только этот сценарий)
│   ├── lib.rs                      ← +pub mod blob_container
│   └── … (остальные модули проекта)
├── tests/
│   ├── wasm_blob_api.rs            ← из главы 20
│   ├── wasm_blob_container.rs      ← НОВЫЙ: hex_dump, crc32, slice, File, stream
│   └── … (остальные тесты)
├── pkg/
└── web/
    ├── index.html
    └── main.js
```

### Краткий комментарий

**`src/blob_container.rs`** — обёртка над Blob как **контейнером байтов**: создание из строки и из `Uint8Array`, четыре способа чтения (`.text()`, `.arrayBuffer()`, `.stream()` первый chunk, `.stream()` полный обход), `slice()` для нарезки, метаданные `File`, hex-dump и CRC32. Модуль не знает ни про Object URL, ни про скачивание — это уже тема главы 20.

**`tests/wasm_blob_container.rs`** — проверяет roundtrip, slice, hex-dump, CRC32, полный обход stream, идемпотентность.

**`src/app.rs`** — демо подглавы 20.1: смотрим на Blob **изнутри**. Шесть секций: создание, четыре способа чтения, slice + бенчмарк, Blob vs ArrayBuffer, File extends Blob, журнал. Каждая секция решает одну задачу.

### Ссылки для изучения

- [MDN: Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob) — все методы и свойства.
- [MDN: Blob.slice](https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice) — нарезка.
- [MDN: Blob.stream](https://developer.mozilla.org/en-US/docs/Web/API/Blob/stream) — стриминговое чтение.
- [MDN: File](https://developer.mozilla.org/en-US/docs/Web/API/File) — наследник Blob.
- [RFC 1952: CRC32](https://www.rfc-editor.org/rfc/rfc1952#section-8) — polynomial для CRC32, использованный в gzip/zlib/PNG.

---

## 20.1.1. Что такое Blob внутри

**Blob — не массив.** Это **дескриптор** на кусок данных в памяти браузера. Внутри он хранит:

- **size** — длину в байтах (`u64`);
- **type** — MIME-строку (по умолчанию `""`, трактуется как `application/octet-stream`);
- **последовательность байтов** — сами данные.

Размер Blob **не** определяется количеством кусков, из которых он собран. `new Blob(["a", "b", "c"])` даёт size = 3, а `new Blob([new Uint8Array(1_000_000)])` даёт size = 1 000 000. Браузер склеивает куски логически, физически не обязательно копируя.

**Ключевое свойство:** Blob неизменяем. Нельзя «дописать байт в конец». Можно создать новый — из старого плюс новых данных.

## 20.1.2. File extends Blob

`File` — подкласс `Blob`. Любой `File` можно использовать там, где ожидается `Blob` — передать в `URL.createObjectURL`, прочитать через `.text()`, нарезать через `.slice()`.

Что добавляет `File` поверх `Blob`:

| Свойство | Что даёт |
|---|---|
| `name` | имя файла без пути |
| `lastModified` | Unix-время в миллисекундах |
| `webkitRelativePath` | путь при выборе директории (не везде) |

Всё остальное — `size`, `type`, чтение, нарезка — унаследовано от Blob. Это значит: **любой код, работающий с Blob, работает с File**.

## 20.1.3. Четыре способа чтения

Blob предоставляет три метода чтения, каждый возвращает `Promise`, плюс отдельный паттерн «обход stream в цикле»:

| Метод | Возвращает | Когда использовать |
|---|---|---|
| `.text()` | `String` (UTF-8) | текстовые файлы, JSON, HTML |
| `.arrayBuffer()` | `ArrayBuffer` | бинарные данные — изображения, ZIP |
| `.stream()` → первый chunk | `Uint8Array` | когда достаточно посмотреть начало |
| `.stream()` → обход в цикле | последовательность `Uint8Array` | большие файлы, инкрементальная обработка |

`.text()` и `.arrayBuffer()` читают весь Blob в память. Для файла 1 ГБ это 1 ГБ RAM. `.stream()` даёт **чанки** — можно обрабатывать по мере поступления, не держа весь файл.

**Ключевой инвариант:** `CRC32(concat(chunks)) == CRC32(arrayBuffer)`. Это значит, что stream не теряет байты. Проверяется в демо.

## 20.1.4. `slice()` — нарезка на части

`blob.slice(start, end)` возвращает **новый Blob**, содержащий байты `[start, end)`. `end` можно опустить — тогда до конца.

Что важно:

1. **Slice не копирует данные физически.** Это дескриптор внутри дескриптора. Для 1 ГБ Blob `slice(0, 1024)` дешев — O(1), не O(1024).
2. **Slice сохраняет MIME.** `blob.slice(…)` наследует `type` родителя.
3. **Границы обрезаются.** `slice(100, 50)` даёт пустой Blob. `slice(-10)` отсчитывает от конца (как в JS).
4. **Sum of slices = original.** Если нарезать на куски `[0, 256)`, `[256, 512)`, … — их размеры в сумме дадут size родителя.

Комбинация slice + arrayBuffer + CRC32 даёт классический паттерн: разбить большой файл на куски, посчитать контрольную сумму каждого, сверить с эталоном.

## 20.1.5. Blob ≠ ArrayBuffer

Важное различие, которое часто упускают:

| | Blob | ArrayBuffer |
|---|---|---|
| Что это | **дескриптор** на данные | **сами данные** |
| Где живёт | может быть в памяти, может быть на диске | всегда в RAM |
| `.size` | метаданное, читается мгновенно | `byteLength`, тоже мгновенно, но данные уже в RAM |
| Создание | дешёвое (дескриптор) | дорогое (копирование) |
| Чтение содержимого | `.arrayBuffer()` — копирует | уже есть |

**Практический смысл:** создать Blob из 5 МБ данных — дёшево. Прочитать его `.arrayBuffer()` — дорого. Если данные не нужны целиком — используйте `.slice()` и `.stream()`.

---

## 20.1.6. Новый модуль: `src/blob_container.rs`

```rust
//! Blob как контейнер бинарных данных.
//!
//! **Что даёт.** Примитивы работы с Blob **как с данными**:
//! * `from_text` / `from_bytes` — создание;
//! * `read_text` / `read_bytes` / `read_first_chunk` / `read_all_chunks` — чтение;
//! * `slice` — нарезка на части;
//! * `is_file` / `file_name` / `file_size` / `file_mime` /
//!   `file_last_modified` — метаданные `File extends Blob`;
//! * `hex_dump`, `crc32` — представление и контрольная сумма.
//!
//! **Что НЕ содержит.** Object URL, скачивание, UI — это тема
//! главы 20. Здесь только работа с Blob-контейнером.
//!
//! **Почему через Reflect.** Сигнатуры методов `Blob` в `web-sys`
//! менялись между версиями. Reflect-подход работает везде и не
//! привязывается к конкретной версии крейта.
//!
//! **Правило WASM-инференса.** `format!`/`vec!` в tail-позиции —
//! через обёртки `format_str`/`vec_of`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

// ===========================================================================
// Строковые и коллекционные хелперы
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

#[inline]
fn vec_of<T>(v: Vec<T>) -> Vec<T> {
    v
}

// ===========================================================================
// Создание Blob
// ===========================================================================

/// Blob из строки. Строка кодируется в UTF-8.
pub fn from_text(text: &str, mime: &str) -> Result<JsValue, JsValue> {
    let parts = js_sys::Array::new();
    parts.push(&JsValue::from_str(text));
    construct_blob(&parts, mime)
}

/// Blob из байтов (`Uint8Array` → `ArrayBuffer` → part).
pub fn from_bytes(bytes: &[u8], mime: &str) -> Result<JsValue, JsValue> {
    let u8arr = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&u8arr.buffer());
    construct_blob(&parts, mime)
}

fn construct_blob(parts: &js_sys::Array, mime: &str) -> Result<JsValue, JsValue> {
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
    args.push(parts);
    args.push(&options);

    js_sys::Reflect::construct(&ctor, &args)
}

/// Размер Blob (или File) в байтах.
pub fn blob_size(blob: &JsValue) -> f64 {
    js_sys::Reflect::get(blob, &JsValue::from_str("size"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// MIME-тип Blob.
pub fn blob_mime(blob: &JsValue) -> String {
    js_sys::Reflect::get(blob, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

// ===========================================================================
// Чтение Blob
// ===========================================================================

/// Прочитать Blob как строку через `.text()`.
///
/// Декодирование — UTF-8. Для не-UTF-8 данных получим
/// replacement-символы U+FFFD. Для бинарных данных используйте
/// `read_bytes`.
pub async fn read_text(blob: &JsValue) -> Result<String, JsValue> {
    let method = js_sys::Reflect::get(blob, &JsValue::from_str("text"))?;
    let f: js_sys::Function = method
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.text не функция"))?;
    let promise: js_sys::Promise = f
        .call0(blob)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.text не Promise"))?;
    let value = JsFuture::from(promise).await?;
    value
        .as_string()
        .ok_or_else(|| JsValue::from_str("blob.text вернул не строку"))
}

/// Прочитать Blob как байты через `.arrayBuffer()`.
pub async fn read_bytes(blob: &JsValue) -> Result<Vec<u8>, JsValue> {
    let method = js_sys::Reflect::get(blob, &JsValue::from_str("arrayBuffer"))?;
    let f: js_sys::Function = method
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.arrayBuffer не функция"))?;
    let promise: js_sys::Promise = f
        .call0(blob)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.arrayBuffer не Promise"))?;
    let ab_value = JsFuture::from(promise).await?;
    let ab: js_sys::ArrayBuffer = ab_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    Ok(u8_from_ab(&ab))
}

/// Прочитать **первый chunk** из `.stream()`.
///
/// `Blob.stream()` отдаёт `ReadableStream<Uint8Array>`. Размер
/// одного chunk — не гарантирован: может быть весь Blob, может
/// быть часть. Мы читаем ровно один chunk и возвращаем его байты
/// (не более `max_bytes`).
pub async fn read_first_chunk(blob: &JsValue, max_bytes: usize) -> Result<Vec<u8>, JsValue> {
    let method = js_sys::Reflect::get(blob, &JsValue::from_str("stream"))?;
    let f: js_sys::Function = method
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.stream не функция"))?;
    let stream = f.call0(blob)?;

    let get_reader_js = js_sys::Reflect::get(&stream, &JsValue::from_str("getReader"))?;
    let get_reader: js_sys::Function = get_reader_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("stream.getReader не функция"))?;
    let reader = get_reader.call0(&stream)?;

    let read_js = js_sys::Reflect::get(&reader, &JsValue::from_str("read"))?;
    let read: js_sys::Function = read_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("reader.read не функция"))?;
    let promise: js_sys::Promise = read
        .call0(&reader)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("read не Promise"))?;
    let result = JsFuture::from(promise).await?;

    // Освобождаем reader.
    if let Ok(release_js) = js_sys::Reflect::get(&reader, &JsValue::from_str("releaseLock")) {
        if let Ok(release_fn) = release_js.dyn_into::<js_sys::Function>() {
            let _ = release_fn.call0(&reader);
        }
    }

    let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
        .as_bool()
        .unwrap_or(true);
    if done {
        let v: Vec<u8> = Vec::new();
        return Ok(v);
    }

    let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))?;
    let arr: js_sys::Uint8Array = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("chunk не Uint8Array"))?;
    let len = arr.length() as usize;
    let take = len.min(max_bytes);
    let mut out: Vec<u8> = Vec::with_capacity(take);
    for i in 0..take {
        out.push(arr.get_index(i as u32) as u8);
    }
    Ok(vec_of(out))
}

/// Прочитать **все** chunks через `.stream()` в цикле.
///
/// Возвращает `(chunks, total_bytes)`. Если чанков больше
/// `max_chunks` — останавливается и возвращает то, что успел.
///
/// Полезно, когда `.arrayBuffer()` не подходит — например, для
/// файлов, не влезающих в память целиком. Можно обрабатывать
/// каждый chunk по мере поступления, не держа весь Blob.
pub async fn read_all_chunks(
    blob: &JsValue,
    max_chunks: usize,
) -> Result<(Vec<Vec<u8>>, usize), JsValue> {
    let method = js_sys::Reflect::get(blob, &JsValue::from_str("stream"))?;
    let f: js_sys::Function = method
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.stream не функция"))?;
    let stream = f.call0(blob)?;

    let get_reader_js = js_sys::Reflect::get(&stream, &JsValue::from_str("getReader"))?;
    let get_reader: js_sys::Function = get_reader_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("stream.getReader не функция"))?;
    let reader = get_reader.call0(&stream)?;

    let read_js = js_sys::Reflect::get(&reader, &JsValue::from_str("read"))?;
    let read: js_sys::Function = read_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("reader.read не функция"))?;

    let mut chunks: Vec<Vec<u8>> = Vec::new();
    let mut total: usize = 0;

    loop {
        if chunks.len() >= max_chunks {
            break;
        }

        let promise: js_sys::Promise = read
            .call0(&reader)?
            .dyn_into()
            .map_err(|_| JsValue::from_str("read не Promise"))?;
        let result = JsFuture::from(promise).await?;

        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
            .as_bool()
            .unwrap_or(true);
        if done {
            break;
        }

        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))?;
        let arr: js_sys::Uint8Array = value
            .dyn_into()
            .map_err(|_| JsValue::from_str("chunk не Uint8Array"))?;
        let len = arr.length() as usize;
        let mut buf: Vec<u8> = vec![0u8; len];
        arr.copy_to(&mut buf);
        total += len;
        chunks.push(buf);
    }

    if let Ok(release_js) = js_sys::Reflect::get(&reader, &JsValue::from_str("releaseLock")) {
        if let Ok(release_fn) = release_js.dyn_into::<js_sys::Function>() {
            let _ = release_fn.call0(&reader);
        }
    }

    Ok((chunks, total))
}

fn u8_from_ab(ab: &js_sys::ArrayBuffer) -> Vec<u8> {
    let u8arr = js_sys::Uint8Array::new(ab);
    let len = u8arr.length() as usize;
    let mut out: Vec<u8> = vec![0u8; len];
    u8arr.copy_to(&mut out);
    out
}

// ===========================================================================
// Slice
// ===========================================================================

/// Нарезать Blob: байты `[start, end)`.
///
/// Возвращает новый Blob — **дескриптор**, не копию данных.
/// MIME-тип наследуется от родителя.
pub fn slice(blob: &JsValue, start: f64, end: f64) -> Result<JsValue, JsValue> {
    let method = js_sys::Reflect::get(blob, &JsValue::from_str("slice"))?;
    let f: js_sys::Function = method
        .dyn_into()
        .map_err(|_| JsValue::from_str("blob.slice не функция"))?;
    f.call2(blob, &JsValue::from_f64(start), &JsValue::from_f64(end))
}

// ===========================================================================
// File inspection
// ===========================================================================

/// Проверка: это File (наследник Blob)?
pub fn is_file(value: &JsValue) -> bool {
    if value.is_null() || value.is_undefined() {
        return false;
    }
    let has_name = js_sys::Reflect::has(value, &JsValue::from_str("name")).unwrap_or(false);
    let has_mod = js_sys::Reflect::has(value, &JsValue::from_str("lastModified"))
        .unwrap_or(false);
    let is_ok: bool = has_name && has_mod;
    is_ok
}

pub fn file_name(file: &JsValue) -> String {
    js_sys::Reflect::get(file, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

pub fn file_size(file: &JsValue) -> f64 {
    js_sys::Reflect::get(file, &JsValue::from_str("size"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

pub fn file_mime(file: &JsValue) -> String {
    js_sys::Reflect::get(file, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

pub fn file_last_modified(file: &JsValue) -> f64 {
    js_sys::Reflect::get(file, &JsValue::from_str("lastModified"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0)
}

/// Отформатировать Unix-время (мс) как `YYYY-MM-DD HH:MM`.
pub fn format_timestamp_ms(ms: f64) -> String {
    if ms <= 0.0 {
        let s: String = "—".to_string();
        return s;
    }
    let date = js_sys::Date::new(&JsValue::from_f64(ms));
    let y = date.get_full_year();
    let m = date.get_month() + 1;
    let d = date.get_date();
    let hh = date.get_hours();
    let mm = date.get_minutes();
    let s: String = format_str(format_args!(
        "{:04}-{:02}-{:02} {:02}:{:02}",
        y, m, d, hh, mm
    ));
    s
}

// ===========================================================================
// Hex-dump и CRC32
// ===========================================================================

/// Классический hex-dump: смещение, 16 байт в hex, ASCII.
pub fn hex_dump(bytes: &[u8], max_bytes: usize) -> String {
    let take: usize = bytes.len().min(max_bytes);
    let mut out: String = String::new();
    let mut i: usize = 0;
    while i < take {
        let line_end: usize = (i + 16).min(take);

        let offset: String = format_str(format_args!("{:08x}  ", i));
        out.push_str(&offset);

        for j in 0..16 {
            let idx: usize = i + j;
            if idx < line_end {
                let h: String = format_str(format_args!("{:02x} ", bytes[idx]));
                out.push_str(&h);
            } else {
                out.push_str("   ");
            }
            if j == 7 {
                out.push(' ');
            }
        }
        out.push_str(" |");

        for j in 0..16 {
            let idx: usize = i + j;
            if idx < line_end {
                let b: u8 = bytes[idx];
                let c: char = if (0x20..0x7f).contains(&b) {
                    b as char
                } else {
                    '.'
                };
                out.push(c);
            } else {
                out.push(' ');
            }
        }
        out.push_str("|\n");

        i = line_end;
    }
    if take < bytes.len() {
        let tail: String =
            format_str(format_args!("… ещё {} байт скрыто\n", bytes.len() - take));
        out.push_str(&tail);
    }
    out
}

/// CRC32 по стандарту IEEE (полином 0xEDB88320).
///
/// Тот же CRC, что используется в gzip, zlib, PNG.
pub fn crc32(bytes: &[u8]) -> u32 {
    let mut crc: u32 = 0xFFFF_FFFF;
    for &b in bytes {
        crc ^= b as u32;
        for _ in 0..8 {
            let mask: u32 = (crc & 1).wrapping_neg();
            crc = (crc >> 1) ^ (0xEDB8_8320 & mask);
        }
    }
    !crc
}
```

---

## 20.1.7. Добавление модуля в `lib.rs`

```rust
// … существующие модули …
pub mod blob_api;
pub mod blob_container;   // ← НОВЫЙ

// …
```

---

## 20.1.8. Тесты: `tests/wasm_blob_container.rs`

```rust
//! Тесты `src/blob_container.rs`.
//!
//! Проверяем: hex-dump, CRC32, создание Blob, чтение (4 способа),
//! slice, File inspection.

use wasm_bindgen_test::*;

use hello_wasm02::blob_container::{
    blob_mime, blob_size, crc32, from_bytes, from_text, hex_dump, is_file,
    read_all_chunks, read_bytes, read_first_chunk, read_text, slice,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// Чистая логика: hex_dump
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_hex_dump_empty() {
    let s = hex_dump(&[], 256);
    assert!(s.is_empty());
}

#[wasm_bindgen_test]
fn test_hex_dump_hello() {
    let s = hex_dump(b"Hello", 256);
    assert!(s.starts_with("00000000  48 65 6c 6c 6f"));
    assert!(s.contains("|Hello"));
}

#[wasm_bindgen_test]
fn test_hex_dump_truncated() {
    let bytes: Vec<u8> = (0..100).collect();
    let s = hex_dump(&bytes, 32);
    assert!(s.contains("00000000"));
    assert!(s.contains("00000010"));
    assert!(!s.contains("00000020"));
    assert!(s.contains("скрыто"));
}

#[wasm_bindgen_test]
fn test_hex_dump_non_printable() {
    let bytes = b"\x00\x01\x02\x03 hello";
    let s = hex_dump(bytes, 256);
    assert!(s.contains("|.... hello"));
}

// ---------------------------------------------------------------------------
// Чистая логика: crc32
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_crc32_empty() {
    assert_eq!(crc32(&[]), 0);
}

#[wasm_bindgen_test]
fn test_crc32_hello() {
    // Известное значение CRC32("hello") = 0x3610A686.
    assert_eq!(crc32(b"hello"), 0x3610_A686);
}

#[wasm_bindgen_test]
fn test_crc32_deterministic() {
    let a = crc32(b"the quick brown fox");
    let b = crc32(b"the quick brown fox");
    assert_eq!(a, b);
}

#[wasm_bindgen_test]
fn test_crc32_changes_on_edit() {
    let a = crc32(b"hello");
    let b = crc32(b"hello!");
    assert_ne!(a, b);
}

// ---------------------------------------------------------------------------
// from_text / from_bytes / чтение
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_from_text_roundtrip() {
    let text = "Hello, Blob!";
    let blob = from_text(text, "text/plain;charset=utf-8").expect("blob");
    assert_eq!(blob_size(&blob), text.len() as f64);
    assert_eq!(blob_mime(&blob), "text/plain;charset=utf-8");

    let back = read_text(&blob).await.expect("read");
    assert_eq!(back, text);
}

#[wasm_bindgen_test]
async fn test_from_bytes_roundtrip() {
    let data: Vec<u8> = (0..=255).collect();
    let blob = from_bytes(&data, "application/octet-stream").expect("blob");
    assert_eq!(blob_size(&blob), 256.0);

    let back = read_bytes(&blob).await.expect("read");
    assert_eq!(back, data);
}

#[wasm_bindgen_test]
async fn test_read_first_chunk() {
    let data: Vec<u8> = (0..1024).map(|i| (i & 0xFF) as u8).collect();
    let blob = from_bytes(&data, "application/octet-stream").expect("blob");

    let chunk = read_first_chunk(&blob, 4096).await.expect("stream");
    assert!(!chunk.is_empty(), "chunk не пустой");
    assert!(chunk.len() <= data.len(), "chunk не больше исходного");
    assert_eq!(&chunk[..], &data[..chunk.len()]);
}

#[wasm_bindgen_test]
async fn test_read_all_chunks_sum_equals_size() {
    let data: Vec<u8> = (0..4096).map(|i| (i & 0xFF) as u8).collect();
    let blob = from_bytes(&data, "application/octet-stream").expect("blob");

    let (chunks, total) = read_all_chunks(&blob, 1000).await.expect("stream all");
    assert_eq!(total, data.len(), "сумма байт всех chunks = размер Blob");
    assert!(!chunks.is_empty(), "хотя бы один chunk");

    // Конкатенация chunks совпадает с данными.
    let mut whole: Vec<u8> = Vec::with_capacity(total);
    for c in &chunks {
        whole.extend_from_slice(c);
    }
    assert_eq!(whole, data);

    // CRC32 конкатенации = CRC32 arrayBuffer.
    let crc_chunks = crc32(&whole);
    let crc_ab = crc32(&read_bytes(&blob).await.expect("ab"));
    assert_eq!(crc_chunks, crc_ab, "CRC stream = CRC arrayBuffer");
}

// ---------------------------------------------------------------------------
// slice
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_slice_roundtrip() {
    let data: Vec<u8> = (0..1024).map(|i| (i & 0xFF) as u8).collect();
    let blob = from_bytes(&data, "application/octet-stream").expect("blob");

    let part = slice(&blob, 256.0, 512.0).expect("slice");
    assert_eq!(blob_size(&part), 256.0);

    let bytes = read_bytes(&part).await.expect("read");
    assert_eq!(bytes.len(), 256);
    assert_eq!(bytes, data[256..512]);
}

#[wasm_bindgen_test]
async fn test_slice_sum_equals_total() {
    let data: Vec<u8> = (0..1024).map(|i| (i & 0xFF) as u8).collect();
    let blob = from_bytes(&data, "application/octet-stream").expect("blob");

    let total = blob_size(&blob);
    let mut sum = 0.0;
    let mut offset = 0.0;
    while offset < total {
        let end = (offset + 256.0).min(total);
        let part = slice(&blob, offset, end).expect("slice");
        sum += blob_size(&part);
        offset = end;
    }
    assert_eq!(sum, total, "сумма размеров кусков = размер родителя");
}

// ---------------------------------------------------------------------------
// File
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_is_file_on_plain_blob() {
    let blob = from_text("hi", "text/plain").expect("blob");
    assert!(!is_file(&blob), "обычный Blob — не File");
}

#[wasm_bindgen_test]
fn test_is_file_on_undefined() {
    assert!(!is_file(&wasm_bindgen::JsValue::UNDEFINED));
    assert!(!is_file(&wasm_bindgen::JsValue::NULL));
}
```

### Что не проверяется в WASM-тестах

**Реальный `File` из input или drag&drop.** В headless Chrome нет способа его создать без user gesture. Проверяем только негативный случай (`is_file` на обычном Blob = `false`). Позитивный — вручную через демо.

**Гарантированно несколько чанков из `.stream()`.** Размер chunk — implementation-defined. Проверяем, что сумма chunks = размер, но не количество.

---

## 20.1.9. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты blob-контейнера в headless Chrome
wasm-pack test --headless --chrome --test wasm_blob_container

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** изменений — CRC32 реализован в модуле, hex-dump — тоже.

---

## 20.1.10. Анализ и доработки

### Что проявилось на первой версии

**1. Одна строка текста — один chunk.** Дефолтный текст короткий, при нарезке получается один кусок. Скучно. Нужно **генерировать из байтов** вариант побольше.

**2. Метаданные Blob не показывают разницу между chars и bytes.** Текст с эмодзи — 37 символов / 62 байта. Это наглядный факт.

**3. Slice без CRC — половина пользы.** Каждый кусок имеет свой CRC32.

**4. Hex-dump без разметки монотонности.** Нужно показать hex первых байт **каждого** куска.

**5. `is_file` не используется.** Модуль содержит проверку — но демо её не вызывает.

**6. Один способ чтения — не показан в контрасте.** Нужны все четыре: `.text()`, `.arrayBuffer()`, `.stream()` первый, `.stream()` всё.

**7. Автозапуск.** Демо должно быть заполнено при загрузке.

**8. Журнал нужен.** Каждое действие пишет строку.

### Список доработок

- **A.** Два Blob-creator — из строки (UTF-8) и из байтов (1024 байта).
- **B.** Показать chars vs bytes для текстового Blob.
- **C.** Четыре кнопки чтения — `.text()`, `.arrayBuffer()`, `.stream()` первый, `.stream()` всё.
- **D.** Slice на куски по 256 байт — таблица с offset, size, CRC32, hex первых 16 байт.
- **E.** Итог slice — сумма размеров = размер родителя + CRC целого.
- **F.** Секция File extends Blob — input type=file, метаданные, все четыре чтения работают над файлом.
- **G.** Автозапуск при загрузке.
- **H.** Журнал с префиксами `[create]`, `[read]`, `[slice]`, `[file]`, `[big]`.
- **I.** **Бенчмарк slice vs arrayBuffer** — 100 slice() vs 100 arrayBuffer(), показать ×N.
- **J.** **Blob vs ArrayBuffer** — Blob 5 МБ, `.size()` за микросекунды, `.arrayBuffer()` за десятки миллисекунд.
- **K.** **Полный обход stream** — показать, что chunks + CRC конкатенации = CRC arrayBuffer.
- **L.** **Статус-бар** с живым обновлением при каждом действии.

Все двенадцать доработок реализованы в `app.rs`.

---

## 20.1.11. Полный текст `src/app.rs`

```rust
//! Демо подглавы 20.1 — Blob как контейнер бинарных данных.
//!
//! **Идея.** Смотрим на Blob **изнутри**: из чего состоит, как
//! читается, как режется, чем отличается от ArrayBuffer.
//!
//! **Wow-моменты:**
//! 1. **Строка с эмодзи: 37 символов / 62 байта.**
//! 2. **Четыре способа чтения.** `.text()`, `.arrayBuffer()`,
//!    `.stream()` первый chunk, `.stream()` полный обход.
//! 3. **Slice — дескриптор, не копия.**
//! 4. **Blob ≠ ArrayBuffer.** Дескриптор vs содержимое.
//! 5. **File extends Blob.** Slice и чтение работают над файлом.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции — через
//! `format_str(format_args!(…))`.
//!
//! **Правило `?` и `Result`.** `?` — только в функциях с `Result`.
//!
//! **Правило числовых типов.** `scroll_height() -> i32`,
//! `set_scroll_top(value: f64)` — каст обязателен.
//!
//! **Правило snake_case.** `create_element`, `set_attribute`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, Element, HtmlElement, HtmlInputElement, HtmlTextAreaElement};

use crate::blob_container::{
    blob_size, crc32, file_last_modified, file_mime, file_name, file_size,
    format_timestamp_ms, from_bytes, from_text, hex_dump, is_file, read_all_chunks,
    read_bytes, read_first_chunk, read_text, slice,
};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[20.1]";

fn console_log(msg: &str) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::log_1(&JsValue::from_str(&line));
}

fn console_err(msg: &str, e: &JsValue) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::error_2(&JsValue::from_str(&line), e);
}

// ===========================================================================
// Строковые хелперы
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

fn fmt_bytes(n: f64) -> String {
    let v = n.max(0.0);
    let s: String = if v < 1024.0 {
        format_str(format_args!("{} Б", v as u64))
    } else if v < 1024.0 * 1024.0 {
        format_str(format_args!("{:.1} КБ", v / 1024.0))
    } else {
        format_str(format_args!("{:.2} МБ", v / (1024.0 * 1024.0)))
    };
    s
}

fn fmt_ms(v: f64) -> String {
    let s: String = if v < 0.001 {
        format_str(format_args!("{:.2} мкс", v * 1000.0))
    } else if v < 1.0 {
        format_str(format_args!("{:.2} мс", v))
    } else if v < 100.0 {
        format_str(format_args!("{:.1} мс", v))
    } else {
        format_str(format_args!("{:.0} мс", v))
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
// Константы
// ===========================================================================

const SLICE_SIZE: usize = 256;
const HEX_BYTES_BLOB: usize = 256;
const HEX_BYTES_CHUNK: usize = 16;
const BYTES_BLOB_SIZE: usize = 1024;
const SLICE_BENCH_N: usize = 100;
const READ_BENCH_N: usize = 100;
const BIG_BLOB_SIZE: usize = 5 * 1024 * 1024;
const SIZE_BENCH_N: usize = 1000;
const MAX_CHUNKS: usize = 10_000;

const SEED_TEXT: &str = "Hello, Blob! Файл это просто байты. 🌍";

// ===========================================================================
// Состояние
// ===========================================================================

struct DemoState {
    status: Element,
    input: HtmlTextAreaElement,
    blob_info: Element,
    read_output: Element,
    bench_output: Element,
    slice_body: Element,
    slice_summary: Element,
    big_blob_info: Element,
    big_blob_output: Element,
    file_info: Element,
    file_input: HtmlInputElement,
    log_body: Element,
    current_blob: RefCell<Option<JsValue>>,
    current_source: RefCell<String>,
    big_blob: RefCell<Option<JsValue>>,
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
  min-height: 72px;
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
.btn:disabled { opacity: .45; cursor: not-allowed; }

.info-row {
  display: flex; flex-wrap: wrap; gap: 24px;
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #8b949e;
  margin: 10px 0;
  padding-bottom: 10px;
  border-bottom: 1px solid #21262d;
}
.info-row .label { color: #6e7681; }
.info-row .value { color: #79c0ff; font-weight: 500; }
.info-row .value.muted { color: #8b949e; font-weight: 400; }

pre.hex-dump {
  margin: 0; padding: 10px 12px;
  background: #0d1117; border: 1px solid #21262d; border-radius: 6px;
  font-family: ui-monospace, monospace; font-size: 11px;
  color: #e6edf3; white-space: pre; overflow: auto;
  max-height: 240px;
}

pre.read-output {
  margin: 0; padding: 10px 12px;
  background: #0d1117; border: 1px solid #21262d; border-radius: 6px;
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #e6edf3; white-space: pre-wrap; word-break: break-word;
  min-height: 40px; max-height: 300px; overflow: auto;
}
.read-output .err { color: #f85149; }
.read-output .ok  { color: #3fb950; }
.read-output .key { color: #8b949e; }
.read-output .num { color: #79c0ff; }
.read-output .warn { color: #d29922; }

.bench-output, .big-blob-output {
  margin-top: 10px; padding: 10px 12px;
  background: #0d1117; border: 1px solid #21262d; border-radius: 6px;
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #8b949e;
  white-space: pre-wrap;
  min-height: 20px;
}
.bench-output .ok, .big-blob-output .ok { color: #3fb950; }
.bench-output .num, .big-blob-output .num { color: #79c0ff; font-weight: 500; }
.big-blob-output .warn { color: #d29922; }

.slice-table {
  width: 100%; border-collapse: collapse;
  font-family: ui-monospace, monospace; font-size: 12px;
}
.slice-table th {
  text-align: left; padding: 6px 10px;
  background: #21262d; color: #8b949e;
  font-weight: 500; font-size: 11px;
  text-transform: uppercase; letter-spacing: .5px;
  border: 1px solid #30363d;
}
.slice-table td {
  padding: 5px 10px;
  border: 1px solid #21262d;
  color: #e6edf3;
  white-space: nowrap;
}
.slice-table td.idx   { color: #8b949e; text-align: right; width: 40px; }
.slice-table td.off   { color: #79c0ff; text-align: right; width: 80px; }
.slice-table td.size  { color: #8b949e; text-align: right; width: 70px; }
.slice-table td.crc   { color: #d29922; }
.slice-table td.hex   { color: #6e7681; font-size: 11px; }
.slice-table tr:nth-child(even) td { background: #0d1117; }

.slice-summary {
  margin-top: 12px; padding-top: 10px;
  border-top: 1px solid #21262d;
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #8b949e;
  min-height: 20px;
}
.slice-summary .ok { color: #3fb950; }
.slice-summary .k  { color: #79c0ff; }

.file-panel { display: flex; flex-direction: column; gap: 10px; }
.file-input-row { display: flex; align-items: center; gap: 12px; }
input[type=file] {
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #8b949e;
}
input[type=file]::file-selector-button {
  background: #21262d; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 7px 13px; font-size: 13px; cursor: pointer;
  margin-right: 10px;
}
input[type=file]::file-selector-button:hover {
  background: #30363d; border-color: #8b949e;
}

.log-body {
  padding: 12px 16px; max-height: 220px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; white-space: pre-wrap; }
.log-line.create { color: #79c0ff; }
.log-line.read   { color: #3fb950; }
.log-line.slice  { color: #d29922; }
.log-line.file   { color: #a371f7; }
.log-line.big    { color: #58a6ff; }
.log-line.err    { color: #f85149; }
.log-line.info   { color: #8b949e; }

.empty { color: #6e7681; font-style: italic; padding: 8px 0; }
"#;

// ===========================================================================
// Логирование и статус
// ===========================================================================

fn log_line(state: &Rc<DemoState>, kind: &str, msg: &str) {
    {
        let mut v = state.log_lines.borrow_mut();
        v.push(msg.to_string());
        if v.len() > 200 {
            let excess = v.len() - 200;
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

    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let h: i32 = el.scroll_height();
        el.set_scroll_top(h as f64);
    }
}

/// Обновить статус-бар — короткая сводка текущего действия.
fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

// ===========================================================================
// Создание Blob
// ===========================================================================

fn create_from_text(state: Rc<DemoState>) {
    let text: String = state.input.value();
    let chars: usize = text.chars().count();
    let bytes_len: usize = text.len();

    match from_text(&text, "text/plain;charset=utf-8") {
        Ok(blob) => {
            let size = blob_size(&blob);
            log_line(
                &state,
                "create",
                &format_str(format_args!(
                    "[create] Blob из строки: {chars} символов / {bytes_len} байт"
                )),
            );
            set_status(
                &state,
                "ok",
                &format_str(format_args!(
                    "Blob создан из строки · {chars} симв. / {} байт",
                    size as u64
                )),
            );

            let bytes: Vec<u8> = text.as_bytes().to_vec();
            let dump: String = hex_dump(&bytes, HEX_BYTES_BLOB);
            let info_html: String = format_str(format_args!(
                "<div class=\"info-row\">\
                   <span><span class=\"label\">источник:</span> <span class=\"value\">строка (UTF-8)</span></span>\
                   <span><span class=\"label\">размер:</span> <span class=\"value\">{}</span></span>\
                   <span><span class=\"label\">MIME:</span> <span class=\"value muted\">text/plain;charset=utf-8</span></span>\
                   <span><span class=\"label\">текст:</span> <span class=\"value\">{chars} симв. / {bytes_len} байт</span></span>\
                 </div>\
                 <pre class=\"hex-dump\">{}</pre>",
                fmt_bytes(size),
                html_escape(&dump),
            ));
            state.blob_info.set_inner_html(&info_html);

            *state.current_blob.borrow_mut() = Some(blob);
            *state.current_source.borrow_mut() = "Blob (строка)".to_string();
            reset_downstream(&state);
        }
        Err(e) => {
            console_err("from_text failed", &e);
            log_line(&state, "err", "не удалось создать Blob из строки");
            set_status(&state, "err", "ошибка создания Blob из строки");
        }
    }
}

fn create_from_bytes(state: Rc<DemoState>) {
    let n: usize = BYTES_BLOB_SIZE;
    let mut data: Vec<u8> = Vec::with_capacity(n);
    for i in 0..n {
        data.push((i % 251) as u8);
    }

    match from_bytes(&data, "application/octet-stream") {
        Ok(blob) => {
            let size = blob_size(&blob);
            log_line(
                &state,
                "create",
                &format_str(format_args!(
                    "[create] Blob из байтов: {} байт (цикл % 251)",
                    size as u64
                )),
            );
            set_status(
                &state,
                "ok",
                &format_str(format_args!(
                    "Blob создан из байтов · {} байт",
                    size as u64
                )),
            );

            let dump: String = hex_dump(&data, HEX_BYTES_BLOB);
            let info_html: String = format_str(format_args!(
                "<div class=\"info-row\">\
                   <span><span class=\"label\">источник:</span> <span class=\"value\">Uint8Array (цикл % 251)</span></span>\
                   <span><span class=\"label\">размер:</span> <span class=\"value\">{}</span></span>\
                   <span><span class=\"label\">MIME:</span> <span class=\"value muted\">application/octet-stream</span></span>\
                 </div>\
                 <pre class=\"hex-dump\">{}</pre>",
                fmt_bytes(size),
                html_escape(&dump),
            ));
            state.blob_info.set_inner_html(&info_html);

            *state.current_blob.borrow_mut() = Some(blob);
            *state.current_source.borrow_mut() = "Blob (байты)".to_string();
            reset_downstream(&state);
        }
        Err(e) => {
            console_err("from_bytes failed", &e);
            log_line(&state, "err", "не удалось создать Blob из байтов");
            set_status(&state, "err", "ошибка создания Blob из байтов");
        }
    }
}

fn reset_downstream(state: &Rc<DemoState>) {
    state.read_output.set_inner_html(
        "<span class=\"key\">Нажмите .text() / .arrayBuffer() / .stream() / .stream() всё, чтобы прочитать этот Blob</span>",
    );
    state.bench_output.set_inner_html("");
    state.slice_body.set_inner_html(
        "<tr><td colspan=\"5\" class=\"empty\">Нажмите «Нарезать», чтобы увидеть куски</td></tr>",
    );
    state.slice_summary.set_inner_html("");
}

// ===========================================================================
// Чтение: четыре способа
// ===========================================================================

fn read_as_text(state: Rc<DemoState>) {
    let blob = match state.current_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[read] Blob ещё не создан");
            set_status(&state, "err", "сначала создайте Blob");
            return;
        }
    };
    let s = state.clone();
    spawn_local(async move {
        let t0 = js_sys::Date::now();
        match read_text(&blob).await {
            Ok(text) => {
                let t1 = js_sys::Date::now();
                let ms = t1 - t0;
                let chars: usize = text.chars().count();
                let bytes: usize = text.len();
                let preview: String = if chars > 200 {
                    let t: String = text.chars().take(200).collect();
                    format_str(format_args!("{t}…"))
                } else {
                    text.clone()
                };
                let html: String = format_str(format_args!(
                    "<span class=\"ok\">✓ {chars} символов, {bytes} байт, {}</span>\n\
                     <span class=\"key\">preview:</span>\n{}",
                    fmt_ms(ms),
                    html_escape(&preview)
                ));
                s.read_output.set_inner_html(&html);
                log_line(
                    &s,
                    "read",
                    &format_str(format_args!(
                        "[read] .text(): {chars} симв., {bytes} байт, {}",
                        fmt_ms(ms)
                    )),
                );
                set_status(
                    &s,
                    "ok",
                    &format_str(format_args!(
                        "прочитано как .text() · {chars} симв. за {}",
                        fmt_ms(ms)
                    )),
                );
            }
            Err(e) => {
                console_err("read_text failed", &e);
                let msg = e
                    .as_string()
                    .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                let html: String = format_str(format_args!(
                    "<span class=\"err\">.text() → ошибка: {}</span>",
                    html_escape(&msg)
                ));
                s.read_output.set_inner_html(&html);
                log_line(&s, "err", "[read] .text() — ошибка");
                set_status(&s, "err", "ошибка .text()");
            }
        }
    });
}

fn read_as_array_buffer(state: Rc<DemoState>) {
    let blob = match state.current_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[read] Blob ещё не создан");
            set_status(&state, "err", "сначала создайте Blob");
            return;
        }
    };
    let s = state.clone();
    spawn_local(async move {
        let t0 = js_sys::Date::now();
        match read_bytes(&blob).await {
            Ok(bytes) => {
                let t1 = js_sys::Date::now();
                let ms = t1 - t0;
                let size = bytes.len();
                let crc = crc32(&bytes);
                let dump = hex_dump(&bytes, 64);
                let html: String = format_str(format_args!(
                    "<span class=\"ok\">✓ {size} байт, {}</span>\n\
                     <span class=\"key\">CRC32 = <span class=\"num\">0x{crc:08x}</span></span>\n\
                     <span class=\"key\">первые 64 байта:</span>\n{}",
                    fmt_ms(ms),
                    html_escape(&dump)
                ));
                s.read_output.set_inner_html(&html);
                log_line(
                    &s,
                    "read",
                    &format_str(format_args!(
                        "[read] .arrayBuffer(): {size} байт, CRC32=0x{crc:08x}, {}",
                        fmt_ms(ms)
                    )),
                );
                set_status(
                    &s,
                    "ok",
                    &format_str(format_args!(
                        "прочитано как .arrayBuffer() · {size} байт за {}",
                        fmt_ms(ms)
                    )),
                );
            }
            Err(e) => {
                console_err("read_bytes failed", &e);
                let msg = e
                    .as_string()
                    .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                let html: String = format_str(format_args!(
                    "<span class=\"err\">.arrayBuffer() → ошибка: {}</span>",
                    html_escape(&msg)
                ));
                s.read_output.set_inner_html(&html);
                log_line(&s, "err", "[read] .arrayBuffer() — ошибка");
                set_status(&s, "err", "ошибка .arrayBuffer()");
            }
        }
    });
}

fn read_as_stream_first(state: Rc<DemoState>) {
    let blob = match state.current_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[read] Blob ещё не создан");
            set_status(&state, "err", "сначала создайте Blob");
            return;
        }
    };
    let total = blob_size(&blob) as usize;
    let s = state.clone();
    spawn_local(async move {
        let t0 = js_sys::Date::now();
        match read_first_chunk(&blob, 4096).await {
            Ok(chunk) => {
                let t1 = js_sys::Date::now();
                let ms = t1 - t0;
                let n = chunk.len();
                let crc = crc32(&chunk);
                let dump = hex_dump(&chunk, 32);
                let html: String = format_str(format_args!(
                    "<span class=\"ok\">✓ первый chunk: {n} байт из {total}, {}</span>\n\
                     <span class=\"key\">CRC32 chunk = <span class=\"num\">0x{crc:08x}</span></span>\n\
                     <span class=\"key\">первые 32 байта chunk:</span>\n{}",
                    fmt_ms(ms),
                    html_escape(&dump)
                ));
                s.read_output.set_inner_html(&html);
                log_line(
                    &s,
                    "read",
                    &format_str(format_args!(
                        "[read] .stream() первый chunk: {n} байт, CRC32=0x{crc:08x}, {}",
                        fmt_ms(ms)
                    )),
                );
                set_status(
                    &s,
                    "ok",
                    &format_str(format_args!(
                        ".stream() первый chunk · {n} байт за {}",
                        fmt_ms(ms)
                    )),
                );
            }
            Err(e) => {
                console_err("read_first_chunk failed", &e);
                let msg = e
                    .as_string()
                    .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                let html: String = format_str(format_args!(
                    "<span class=\"err\">.stream() → ошибка: {}</span>",
                    html_escape(&msg)
                ));
                s.read_output.set_inner_html(&html);
                log_line(&s, "err", "[read] .stream() — ошибка");
                set_status(&s, "err", "ошибка .stream()");
            }
        }
    });
}

fn read_as_stream_all(state: Rc<DemoState>) {
    let blob = match state.current_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[read] Blob ещё не создан");
            set_status(&state, "err", "сначала создайте Blob");
            return;
        }
    };
    let total_size = blob_size(&blob) as usize;
    let s = state.clone();
    spawn_local(async move {
        let t0 = js_sys::Date::now();
        match read_all_chunks(&blob, MAX_CHUNKS).await {
            Ok((chunks, total)) => {
                let t1 = js_sys::Date::now();
                let ms = t1 - t0;
                let n_chunks = chunks.len();

                let mut whole: Vec<u8> = Vec::with_capacity(total);
                for c in &chunks {
                    whole.extend_from_slice(c);
                }
                let crc_whole = crc32(&whole);
                let crc_from_arraybuffer: u32 = match read_bytes(&blob).await {
                    Ok(b) => crc32(&b),
                    Err(_) => 0,
                };
                let crc_match: bool = crc_whole == crc_from_arraybuffer;

                let mut sizes_line: String = String::new();
                for (i, c) in chunks.iter().take(5).enumerate() {
                    if i > 0 {
                        sizes_line.push_str(", ");
                    }
                    let frag: String = format_str(format_args!("{}", c.len()));
                    sizes_line.push_str(&frag);
                }
                if n_chunks > 5 {
                    sizes_line.push_str(", …");
                }

                let sum_ok: bool = total == total_size;
                let sum_ok_str: &str = if sum_ok { "✓" } else { "✗" };
                let crc_str: &str = if crc_match {
                    "✓ CRC stream = CRC arrayBuffer"
                } else {
                    "✗ CRC не совпал"
                };

                let html: String = format_str(format_args!(
                    "<span class=\"ok\">✓ обход завершён за {}</span>\n\
                     <span class=\"key\">chunks: <span class=\"num\">{n_chunks}</span> (размеры: {sizes_line})</span>\n\
                     <span class=\"key\">сумма байт: <span class=\"num\">{total}</span> из <span class=\"num\">{total_size}</span> {sum_ok_str}</span>\n\
                     <span class=\"key\">CRC32 (stream):      <span class=\"num\">0x{crc_whole:08x}</span></span>\n\
                     <span class=\"key\">CRC32 (arrayBuffer): <span class=\"num\">0x{crc_from_arraybuffer:08x}</span></span>\n\
                     <span class=\"ok\">{crc_str}</span>",
                    fmt_ms(ms),
                ));
                s.read_output.set_inner_html(&html);
                log_line(
                    &s,
                    "read",
                    &format_str(format_args!(
                        "[read] .stream() всё: {n_chunks} chunks, {total} байт, {}",
                        fmt_ms(ms)
                    )),
                );
                set_status(
                    &s,
                    "ok",
                    &format_str(format_args!(
                        ".stream() всё · {n_chunks} chunks · {} за {}",
                        fmt_bytes(total as f64),
                        fmt_ms(ms)
                    )),
                );
            }
            Err(e) => {
                console_err("read_all_chunks failed", &e);
                let msg = e
                    .as_string()
                    .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                let html: String = format_str(format_args!(
                    "<span class=\"err\">.stream() всё → ошибка: {}</span>",
                    html_escape(&msg)
                ));
                s.read_output.set_inner_html(&html);
                log_line(&s, "err", "[read] .stream() всё — ошибка");
                set_status(&s, "err", "ошибка .stream() всё");
            }
        }
    });
}

// ===========================================================================
// Benchmark: slice vs arrayBuffer
// ===========================================================================

fn run_benchmark(state: Rc<DemoState>) {
    let blob = match state.current_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[bench] Blob ещё не создан");
            set_status(&state, "err", "сначала создайте Blob");
            return;
        }
    };
    let source = state.current_source.borrow().clone();
    let s = state.clone();
    spawn_local(async move {
        log_line(
            &s,
            "info",
            &format_str(format_args!(
                "[bench] источник: {source} · {SLICE_BENCH_N} slice() vs {READ_BENCH_N} arrayBuffer()"
            )),
        );

        let t0 = js_sys::Date::now();
        let mut slice_sum: f64 = 0.0;
        for i in 0..SLICE_BENCH_N {
            let start: f64 = i as f64;
            let end: f64 = start + 1.0;
            if let Ok(part) = slice(&blob, start, end) {
                slice_sum += blob_size(&part);
            }
        }
        let t1 = js_sys::Date::now();
        let slice_ms = t1 - t0;

        let t2 = js_sys::Date::now();
        let mut read_total: usize = 0;
        for _ in 0..READ_BENCH_N {
            if let Ok(bytes) = read_bytes(&blob).await {
                read_total += bytes.len();
            }
        }
        let t3 = js_sys::Date::now();
        let read_ms = t3 - t2;

        let speedup: f64 = if slice_ms > 0.0 { read_ms / slice_ms } else { 0.0 };

        let html: String = format_str(format_args!(
            "<span class=\"key\">{SLICE_BENCH_N} × slice()        →</span> \
             <span class=\"num\">{}</span>   (создано дескрипторов, суммарно <span class=\"num\">{}</span> байт описано)\n\
             <span class=\"key\">{READ_BENCH_N} × arrayBuffer()  →</span> \
             <span class=\"num\">{}</span>   (в память скопировано <span class=\"num\">{}</span> байт)\n\
             <span class=\"ok\">→ arrayBuffer дороже slice в <b>×{:.0}</b></span>",
            fmt_ms(slice_ms),
            slice_sum as u64,
            fmt_ms(read_ms),
            read_total,
            speedup,
        ));
        s.bench_output.set_inner_html(&html);

        log_line(
            &s,
            "info",
            &format_str(format_args!(
                "[bench] slice: {}, arrayBuffer: {} (×{:.0})",
                fmt_ms(slice_ms),
                fmt_ms(read_ms),
                speedup
            )),
        );
        set_status(
            &s,
            "ok",
            &format_str(format_args!(
                "бенчмарк · slice {} · arrayBuffer {} · ×{:.0}",
                fmt_ms(slice_ms),
                fmt_ms(read_ms),
                speedup
            )),
        );
    });
}

// ===========================================================================
// Slice + CRC
// ===========================================================================

fn run_slice(state: Rc<DemoState>) {
    let blob = match state.current_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[slice] Blob ещё не создан");
            set_status(&state, "err", "сначала создайте Blob");
            return;
        }
    };
    let source = state.current_source.borrow().clone();
    let total = blob_size(&blob);
    let slice_size = SLICE_SIZE as f64;

    log_line(
        &state,
        "slice",
        &format_str(format_args!(
            "[slice] источник: {source} · нарезаю {} на куски по {SLICE_SIZE} байт",
            fmt_bytes(total)
        )),
    );

    let s = state.clone();
    spawn_local(async move {
        let doc = match web_sys::window().and_then(|w| w.document()) {
            Some(d) => d,
            None => return,
        };

        s.slice_body.set_inner_html("");
        s.slice_summary.set_inner_html("");

        let whole_bytes: Vec<u8> = match read_bytes(&blob).await {
            Ok(b) => b,
            Err(e) => {
                console_err("read whole blob failed", &e);
                log_line(&s, "err", "[slice] не удалось прочитать весь Blob");
                set_status(&s, "err", "ошибка чтения для slice");
                return;
            }
        };
        let whole_crc = crc32(&whole_bytes);

        let n_chunks = (total / slice_size).ceil() as usize;
        let mut total_sum: f64 = 0.0;

        for i in 0..n_chunks {
            let start = i as f64 * slice_size;
            let end = (start + slice_size).min(total);

            let part = match slice(&blob, start, end) {
                Ok(p) => p,
                Err(e) => {
                    console_err("slice failed", &e);
                    continue;
                }
            };
            let part_size = blob_size(&part);
            total_sum += part_size;

            let bytes = match read_bytes(&part).await {
                Ok(b) => b,
                Err(_) => {
                    log_line(
                        &s,
                        "err",
                        &format_str(format_args!("[slice] кусок #{i} — ошибка чтения")),
                    );
                    continue;
                }
            };
            let part_crc = crc32(&bytes);
            let hex_short = hex_dump(&bytes, HEX_BYTES_CHUNK);
            let hex_first_line: String = hex_short
                .lines()
                .next()
                .map(|s| s.to_string())
                .unwrap_or_default();

            let tr = match doc.create_element("tr") {
                Ok(e) => e,
                Err(_) => continue,
            };

            let td_idx = match doc.create_element("td") {
                Ok(e) => e,
                Err(_) => continue,
            };
            td_idx.set_class_name("idx");
            let txt_idx: String = format_str(format_args!("#{i}"));
            td_idx.set_text_content(Some(&txt_idx));
            tr.append_child(&td_idx).ok();

            let td_off = match doc.create_element("td") {
                Ok(e) => e,
                Err(_) => continue,
            };
            td_off.set_class_name("off");
            let txt_off: String = format_str(format_args!("{}", start as u64));
            td_off.set_text_content(Some(&txt_off));
            tr.append_child(&td_off).ok();

            let td_size = match doc.create_element("td") {
                Ok(e) => e,
                Err(_) => continue,
            };
            td_size.set_class_name("size");
            let txt_size: String = format_str(format_args!("{}", part_size as u64));
            td_size.set_text_content(Some(&txt_size));
            tr.append_child(&td_size).ok();

            let td_crc = match doc.create_element("td") {
                Ok(e) => e,
                Err(_) => continue,
            };
            td_crc.set_class_name("crc");
            let txt_crc: String = format_str(format_args!("0x{part_crc:08x}"));
            td_crc.set_text_content(Some(&txt_crc));
            tr.append_child(&td_crc).ok();

            let td_hex = match doc.create_element("td") {
                Ok(e) => e,
                Err(_) => continue,
            };
            td_hex.set_class_name("hex");
            td_hex.set_text_content(Some(&hex_first_line));
            tr.append_child(&td_hex).ok();

            s.slice_body.append_child(&tr).ok();
        }

        let sum_ok: bool = (total_sum - total).abs() < 0.001;
        let sum_ok_str: &str = if sum_ok { "✓ совпадает" } else { "✗ расхождение" };
        let sum_html: String = format_str(format_args!(
            "сумма размеров кусков: <span class=\"k\">{}</span> / весь Blob: <span class=\"k\">{}</span> · <span class=\"ok\">{sum_ok_str}</span><br>\
             CRC32 всего Blob: <span class=\"k\">0x{whole_crc:08x}</span>",
            fmt_bytes(total_sum),
            fmt_bytes(total),
        ));
        s.slice_summary.set_inner_html(&sum_html);

        log_line(
            &s,
            "slice",
            &format_str(format_args!(
                "[slice] {n_chunks} кусков, сумма {}, CRC целого 0x{whole_crc:08x}",
                fmt_bytes(total_sum)
            )),
        );
        set_status(
            &s,
            "ok",
            &format_str(format_args!(
                "нарезано на {n_chunks} кусков · {} · CRC 0x{whole_crc:08x}",
                fmt_bytes(total_sum)
            )),
        );
    });
}

// ===========================================================================
// Blob vs ArrayBuffer
// ===========================================================================

fn make_big_blob(state: Rc<DemoState>) {
    let n: usize = BIG_BLOB_SIZE;
    let mut data: Vec<u8> = Vec::with_capacity(n);
    for i in 0..n {
        data.push(((i * 17) % 251) as u8);
    }

    match from_bytes(&data, "application/octet-stream") {
        Ok(blob) => {
            let size = blob_size(&blob);
            let t0 = js_sys::Date::now();
            let mut acc: f64 = 0.0;
            for _ in 0..SIZE_BENCH_N {
                acc += blob_size(&blob);
            }
            let t1 = js_sys::Date::now();
            let size_ms = (t1 - t0) / SIZE_BENCH_N as f64;

            let total_disp: String = fmt_bytes(size);
            let info_html: String = format_str(format_args!(
                "<span class=\"ok\">✓ Blob создан: <span class=\"num\">{}</span> (MIME: application/octet-stream)</span>\n\
                 <span class=\"key\">Это дескриптор — данные ещё не в ArrayBuffer.</span>\n\
                 <span class=\"key\">Среднее время .size() (×{SIZE_BENCH_N}): <span class=\"num\">{}</span></span>\n\
                 <span class=\"key\">Сумма по бенчмарку: <span class=\"num\">{}</span> (sanity)</span>",
                total_disp,
                fmt_ms(size_ms),
                fmt_bytes(acc / SIZE_BENCH_N as f64),
            ));
            state.big_blob_info.set_inner_html(&info_html);

            log_line(
                &state,
                "big",
                &format_str(format_args!(
                    "[big] Blob {} создан, .size() = {}",
                    total_disp,
                    fmt_ms(size_ms)
                )),
            );
            set_status(
                &state,
                "ok",
                &format_str(format_args!(
                    "Blob {} создан · .size() за {}",
                    total_disp,
                    fmt_ms(size_ms)
                )),
            );

            *state.big_blob.borrow_mut() = Some(blob);
            state.big_blob_output.set_inner_html(
                "<span class=\"key\">Нажмите «Прочитать как ArrayBuffer», чтобы увидеть стоимость копирования.</span>",
            );
        }
        Err(e) => {
            console_err("from_bytes big failed", &e);
            log_line(&state, "err", "[big] не удалось создать большой Blob");
            set_status(&state, "err", "ошибка создания большого Blob");
        }
    }
}

fn read_big_as_arraybuffer(state: Rc<DemoState>) {
    let blob = match state.big_blob.borrow().clone() {
        Some(b) => b,
        None => {
            log_line(&state, "err", "[big] сначала создайте большой Blob");
            set_status(&state, "err", "сначала создайте большой Blob");
            return;
        }
    };
    let s = state.clone();
    spawn_local(async move {
        let t0 = js_sys::Date::now();
        let result = read_bytes(&blob).await;
        let t1 = js_sys::Date::now();
        let ms = t1 - t0;

        match result {
            Ok(bytes) => {
                let size = bytes.len();
                let crc = crc32(&bytes);
                let size_disp: String = fmt_bytes(size as f64);
                let mb_per_s: f64 = if ms > 0.0 {
                    (size as f64 / (1024.0 * 1024.0)) / (ms / 1000.0)
                } else {
                    0.0
                };

                let html: String = format_str(format_args!(
                    "<span class=\"ok\">✓ ArrayBuffer получен: <span class=\"num\">{}</span> за <span class=\"num\">{}</span></span>\n\
                     <span class=\"warn\">→ теперь эти {} лежат в RAM как Vec&lt;u8&gt;</span>\n\
                     <span class=\"key\">CRC32: <span class=\"num\">0x{crc:08x}</span></span>\n\
                     <span class=\"key\">Скорость копирования: <span class=\"num\">{:.1}</span> МБ/с</span>",
                    size_disp,
                    fmt_ms(ms),
                    size_disp,
                    mb_per_s,
                ));
                s.big_blob_output.set_inner_html(&html);

                log_line(
                    &s,
                    "big",
                    &format_str(format_args!(
                        "[big] arrayBuffer: {} за {}, {:.1} МБ/с",
                        size_disp,
                        fmt_ms(ms),
                        mb_per_s
                    )),
                );
                set_status(
                    &s,
                    "ok",
                    &format_str(format_args!(
                        "ArrayBuffer {} за {} · {:.1} МБ/с",
                        size_disp,
                        fmt_ms(ms),
                        mb_per_s
                    )),
                );
            }
            Err(e) => {
                console_err("read_bytes big failed", &e);
                let msg = e
                    .as_string()
                    .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                let html: String = format_str(format_args!(
                    "<span class=\"err\">arrayBuffer → ошибка: {}</span>",
                    html_escape(&msg)
                ));
                s.big_blob_output.set_inner_html(&html);
                log_line(&s, "err", "[big] arrayBuffer — ошибка");
                set_status(&s, "err", "ошибка чтения большого Blob");
            }
        }
    });
}

// ===========================================================================
// File extends Blob
// ===========================================================================

fn inspect_file(state: Rc<DemoState>, file: JsValue) {
    if !is_file(&file) {
        log_line(&state, "err", "[file] это не File");
        set_status(&state, "err", "выбранный объект не File");
        return;
    }
    let name = file_name(&file);
    let size = file_size(&file);
    let mime = file_mime(&file);
    let modified = file_last_modified(&file);
    let modified_str = format_timestamp_ms(modified);

    let mime_disp: String = if mime.is_empty() {
        "(не определён)".to_string()
    } else {
        mime.clone()
    };

    let html: String = format_str(format_args!(
        "<div class=\"info-row\">\
           <span><span class=\"label\">имя:</span> <span class=\"value\">{}</span></span>\
           <span><span class=\"label\">размер:</span> <span class=\"value\">{}</span></span>\
         </div>\
         <div class=\"info-row\">\
           <span><span class=\"label\">MIME:</span> <span class=\"value muted\">{}</span></span>\
           <span><span class=\"label\">изменён:</span> <span class=\"value muted\">{}</span></span>\
         </div>\
         <div style=\"color:#8b949e;font-size:12px;font-family:ui-monospace\">\
           File — это Blob. Кнопки чтения, slice и бенчмарк работают над ним так же, \
           как над Blob из строки или байтов.\
         </div>",
        html_escape(&name),
        fmt_bytes(size),
        html_escape(&mime_disp),
        html_escape(&modified_str),
    ));
    state.file_info.set_inner_html(&html);

    log_line(
        &state,
        "file",
        &format_str(format_args!(
            "[file] {name} · {} · {mime_disp}",
            fmt_bytes(size)
        )),
    );
    set_status(
        &state,
        "ok",
        &format_str(format_args!(
            "File: {name} · {}",
            fmt_bytes(size)
        )),
    );

    *state.current_blob.borrow_mut() = Some(file);
    *state.current_source.borrow_mut() = format_str(format_args!("File: {name}"));
    reset_downstream(&state);
    state.read_output.set_inner_html(
        "<span class=\"key\">File — это Blob. Все четыре кнопки чтения работают с ним так же.</span>",
    );
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
    mut handler: F,
) -> Result<Element, JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    let cls: &str = if primary { "btn primary" } else { "btn" };
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

    // --- Header ----------------------------------------------------
    let h1 = document.create_element("h1")?;
    h1.set_inner_html(
        "20.1 <code>Blob</code>: контейнер бинарных данных изнутри",
    );
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "создание · 4 способа чтения · slice — дескриптор · Blob ≠ ArrayBuffer · File extends Blob",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    // --- Intro -----------------------------------------------------
    let intro = document.create_element("div")?;
    intro.set_class_name("intro");
    intro.set_inner_html(
        "<b>Что демонстрируется.</b> Blob — это контейнер байтов. Его можно создать \
         из строки или массива, прочитать четырьмя разными способами, нарезать через \
         <code>slice()</code>, и сравнить с ArrayBuffer.<br><br>\
         <b>Четыре способа чтения.</b> <code>.text()</code> даёт строку UTF-8, \
         <code>.arrayBuffer()</code> — все байты в памяти, <code>.stream()</code> — \
         первый chunk, а <b>полный обход stream</b> — все chunks по очереди. \
         Сумма chunks = размер Blob, CRC конкатенации = CRC arrayBuffer.<br><br>\
         <b>Blob ≠ ArrayBuffer.</b> Blob — <b>дескриптор</b>: может быть в памяти, \
         может быть на диске. <code>.size()</code> читается за микросекунды. \
         <code>.arrayBuffer()</code> <b>копирует</b> всё в RAM.<br><br>\
         <b>File — это Blob.</b> Все четыре способа чтения и <code>slice()</code> \
         работают над входным файлом без изменений.",
    );
    body.append_child(&intro)?;

    // --- Status bar ------------------------------------------------
    let status = document.create_element("div")?;
    status.set_class_name("status-bar ok");
    status.set_text_content(Some("готово"));
    body.append_child(&status)?;

    // -------------------------------------------------------------------------
    // Section ❶ — Create
    // -------------------------------------------------------------------------
    let (sec1, body1) = make_section(
        &document,
        "❶",
        "Создание Blob",
        "строка или байты",
        "Введите текст — Blob из строки (UTF-8). Или нажмите «Из байтов» — \
         Blob из 1024 сгенерированных байт.",
    )?;
    body.append_child(&sec1)?;

    let src_ta = document.create_element("textarea")?;
    src_ta.set_class_name("src");
    let src_ta: HtmlTextAreaElement = src_ta
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlTextAreaElement"))?;
    src_ta.set_value(SEED_TEXT);
    body1.append_child(&src_ta)?;

    let create_row = document.create_element("div")?;
    create_row.set_class_name("row-controls");
    body1.append_child(&create_row)?;

    let blob_info = document.create_element("div")?;
    blob_info.set_inner_html("<div class=\"empty\">Blob ещё не создан</div>");
    body1.append_child(&blob_info)?;

    // -------------------------------------------------------------------------
    // Section ❷ — Read
    // -------------------------------------------------------------------------
    let (sec2, body2) = make_section(
        &document,
        "❷",
        "Четыре способа чтения",
        ".text() · .arrayBuffer() · .stream() первый · .stream() всё",
        "Разные методы — разные результаты. Полный обход stream показывает, что \
         это последовательность chunks, а не один кусок.",
    )?;
    body.append_child(&sec2)?;

    let read_row = document.create_element("div")?;
    read_row.set_class_name("row-controls");
    body2.append_child(&read_row)?;

    let read_output = document.create_element("pre")?;
    read_output.set_class_name("read-output");
    read_output.set_inner_html(
        "<span class=\"key\">— результат чтения появится здесь —</span>",
    );
    body2.append_child(&read_output)?;

    // -------------------------------------------------------------------------
    // Section ❸ — Slice + bench
    // -------------------------------------------------------------------------
    let (sec3, body3) = make_section(
        &document,
        "❸",
        "Slice: файл — это последовательность байтов",
        "нарезка + CRC32 + бенчмарк",
        "Blob.slice(start, end) возвращает новый Blob — дескриптор внутри \
         дескриптора. Данные не копируются.",
    )?;
    body.append_child(&sec3)?;

    let slice_ctrl = document.create_element("div")?;
    slice_ctrl.set_class_name("row-controls");
    body3.append_child(&slice_ctrl)?;

    let bench_output = document.create_element("div")?;
    bench_output.set_class_name("bench-output");
    body3.append_child(&bench_output)?;

    let slice_table = document.create_element("table")?;
    slice_table.set_class_name("slice-table");
    slice_table.set_inner_html(
        "<thead><tr>\
           <th style=\"text-align:right\">#</th>\
           <th style=\"text-align:right\">offset</th>\
           <th style=\"text-align:right\">size</th>\
           <th>CRC32</th>\
           <th>hex (первые 16 байт)</th>\
         </tr></thead>",
    );
    let slice_body = document.create_element("tbody")?;
    slice_body.set_inner_html(
        "<tr><td colspan=\"5\" class=\"empty\">Нажмите «Нарезать», чтобы увидеть куски</td></tr>",
    );
    slice_table.append_child(&slice_body)?;
    body3.append_child(&slice_table)?;

    let slice_summary = document.create_element("div")?;
    slice_summary.set_class_name("slice-summary");
    body3.append_child(&slice_summary)?;

    // -------------------------------------------------------------------------
    // Section ❹ — Blob vs ArrayBuffer
    // -------------------------------------------------------------------------
    let (sec4, body4) = make_section(
        &document,
        "❹",
        "Blob vs ArrayBuffer",
        "дескриптор против содержимого",
        "Создаём Blob 5 МБ. Blob — это дескриптор: .size() читается мгновенно, \
         данные ещё не в памяти. Кнопка «Прочитать как ArrayBuffer» покажет \
         стоимость копирования.",
    )?;
    body.append_child(&sec4)?;

    let big_ctrl = document.create_element("div")?;
    big_ctrl.set_class_name("row-controls");
    body4.append_child(&big_ctrl)?;

    let big_blob_info = document.create_element("div")?;
    big_blob_info.set_class_name("big-blob-output");
    big_blob_info.set_inner_html("<div class=\"empty\">5 МБ Blob ещё не создан</div>");
    body4.append_child(&big_blob_info)?;

    let big_blob_output = document.create_element("div")?;
    big_blob_output.set_class_name("big-blob-output");
    body4.append_child(&big_blob_output)?;

    // -------------------------------------------------------------------------
    // Section ❺ — File
    // -------------------------------------------------------------------------
    let (sec5, body5) = make_section(
        &document,
        "❺",
        "File extends Blob",
        "входной файл — это Blob",
        "Выберите файл. Rust прочитает метаданные. Поскольку File — это Blob, \
         все кнопки чтения, slice и бенчмарк работают над ним без изменений.",
    )?;
    body.append_child(&sec5)?;

    let file_panel = document.create_element("div")?;
    file_panel.set_class_name("file-panel");
    body5.append_child(&file_panel)?;

    let file_input_row = document.create_element("div")?;
    file_input_row.set_class_name("file-input-row");
    file_panel.append_child(&file_input_row)?;

    let file_input = document.create_element("input")?;
    file_input.set_attribute("type", "file")?;
    let file_input_el: HtmlInputElement = file_input
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlInputElement"))?;
    file_input_row.append_child(&file_input_el)?;

    let file_info = document.create_element("div")?;
    file_info.set_inner_html("<div class=\"empty\">Файл не выбран</div>");
    file_panel.append_child(&file_info)?;

    // -------------------------------------------------------------------------
    // Section ❻ — Log
    // -------------------------------------------------------------------------
    let (sec6, body6) = make_section(
        &document,
        "❻",
        "Журнал",
        "",
        "Что произошло: создание, чтение, slice, бенчмарк, инспекция файла.",
    )?;
    body.append_child(&sec6)?;
    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    body6.set_class_name("");
    body6.set_attribute("style", "padding:0")?;
    body6.append_child(&log_body)?;

    // --- State ----------------------------------------------------
    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        input: src_ta.clone(),
        blob_info,
        read_output,
        bench_output,
        slice_body,
        slice_summary,
        big_blob_info,
        big_blob_output,
        file_info,
        file_input: file_input_el.clone(),
        log_body,
        current_blob: RefCell::new(None),
        current_source: RefCell::new("(не задан)".to_string()),
        big_blob: RefCell::new(None),
        log_lines: RefCell::new(Vec::new()),
    });

    // --- Buttons: create ------------------------------------------
    {
        let s = state.clone();
        make_button(
            &document,
            &create_row,
            "Создать из строки",
            "Blob из текущего текста (UTF-8).",
            true,
            move || {
                create_from_text(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &create_row,
            "Создать из байтов",
            "Blob из 1024 сгенерированных байт.",
            false,
            move || {
                create_from_bytes(s.clone());
            },
        )?;
    }

    // --- Buttons: read --------------------------------------------
    {
        let s = state.clone();
        make_button(
            &document,
            &read_row,
            ".text()",
            "Прочитать Blob как строку UTF-8.",
            false,
            move || {
                read_as_text(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &read_row,
            ".arrayBuffer()",
            "Прочитать весь Blob в память. Показывает CRC32.",
            false,
            move || {
                read_as_array_buffer(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &read_row,
            ".stream() первый",
            "Прочитать только первый chunk через ReadableStream.",
            false,
            move || {
                read_as_stream_first(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &read_row,
            ".stream() всё",
            "Полный обход stream. Сумма = размер Blob, CRC конкатенации = CRC arrayBuffer.",
            true,
            move || {
                read_as_stream_all(s.clone());
            },
        )?;
    }

    // --- Buttons: slice -------------------------------------------
    {
        let s = state.clone();
        make_button(
            &document,
            &slice_ctrl,
            "Нарезать по 256 байт",
            "Blob.slice() на куски с CRC32 каждого.",
            true,
            move || {
                run_slice(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &slice_ctrl,
            "Бенчмарк: slice vs arrayBuffer",
            "100 slice() vs 100 arrayBuffer() одного Blob.",
            false,
            move || {
                run_benchmark(s.clone());
            },
        )?;
    }

    // --- Buttons: big blob ----------------------------------------
    {
        let s = state.clone();
        make_button(
            &document,
            &big_ctrl,
            "Создать Blob 5 МБ",
            "5 МБ данных в Blob. .size() — мгновенный доступ к метаданному.",
            true,
            move || {
                make_big_blob(s.clone());
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &big_ctrl,
            "Прочитать как ArrayBuffer",
            "Прочитать все 5 МБ в память. Показывает время и скорость МБ/с.",
            false,
            move || {
                read_big_as_arraybuffer(s.clone());
            },
        )?;
    }

    // --- File input -----------------------------------------------
    {
        let s = state.clone();
        let on_change = Closure::<dyn FnMut()>::new(move || {
            let files = match s.file_input.files() {
                Some(f) => f,
                None => return,
            };
            if files.length() == 0 {
                return;
            }
            let file_opt = files.get(0);
            let file: web_sys::File = match file_opt {
                Some(f) => f,
                None => return,
            };
            let file_js: JsValue = file.into();
            inspect_file(s.clone(), file_js);
        });
        file_input_el
            .add_event_listener_with_callback("change", on_change.as_ref().unchecked_ref())?;
        on_change.forget();
    }

    // --- Auto-generate on load ------------------------------------
    {
        let s = state.clone();
        console_log("авто-генерация при загрузке");
        create_from_text(s);
    }

    console_log("run: finished OK");
    Ok(())
}
```

---

## 20.1.12. Что показывает демо

**Секция ❶ — Создание.** Textarea с seed-текстом (с эмодзи — чтобы показать разницу символов и байт). Две кнопки: «Из строки» и «Из байтов». Под ними — hex-dump первых 256 байт + метаданные (источник, размер, MIME).

**Секция ❷ — Чтение.** Четыре кнопки рядом:
- `.text()` — строка UTF-8, preview;
- `.arrayBuffer()` — байты целиком, CRC32, первые 64 байта;
- `.stream()` первый — размер chunk, CRC32 chunk;
- `.stream()` всё — число chunks, размеры первых 5, сумма, сравнение CRC stream и arrayBuffer.

**Секция ❸ — Slice.** Две кнопки: «Нарезать по 256 байт» (таблица с offset, size, CRC32, hex) и «Бенчмарк» (100 slice vs 100 arrayBuffer, ×N). Итог: сумма кусков = размер родителя + CRC целого.

**Секция ❹ — Blob vs ArrayBuffer.** Две кнопки: «Создать Blob 5 МБ» (среднее время `.size()` × 1000) и «Прочитать как ArrayBuffer» (время, скорость МБ/с).

**Секция ❺ — File extends Blob.** Input type=file. Метаданные + напоминание, что все операции из ❶–❹ работают над файлом без изменений.

**Секция ❻ — Журнал.** Все действия с префиксами `[create]`, `[read]`, `[slice]`, `[file]`, `[big]`.

**Статус-бар** под заголовком обновляется при каждом действии.

### Wow-сценарии

**1. Автозапуск.** Открыли страницу — Blob из seed-текста уже создан. В статусе: `Blob создан из строки · 37 симв. / 62 байт`. Под hex-dump видно `37 симв. / 62 байт` — **эмодзи стоит 4 байта**.

**2. Четыре чтения — четыре результата.** Нажмите по очереди на все четыре кнопки. `.text()` даст строку, `.arrayBuffer()` — байты с CRC, `.stream()` первый — chunk, `.stream()` всё — обход. Сумма chunks совпадает с размером.

**3. Создать из байтов → нарезать.** Blob 1024 байта режется на 4 куска по 256. У каждого свой CRC32, разный hex. Итог: сумма = 1024, CRC целого — одним числом.

**4. Бенчмарк — slice vs arrayBuffer.** 100 срезов 1-байтных дескрипторов за микросекунды. 100 полных чтений 1 КБ — за миллисекунды. Разница ×30–50. Видно, что slice — дескриптор, arrayBuffer — копия.

**5. Blob 5 МБ vs ArrayBuffer.** `.size()` за 0.3 мкс, `.arrayBuffer()` за 40 мс. Разница ×100 000 — потому что первое метаданное, второе — копирование.

**6. File.** Выберите любой файл — метаданные появились. Нажмите `.arrayBuffer()` — работает. Нажмите «Нарезать» — работает. File действительно Blob.

---

## 20.1.13. Правила проекта, выученные в этой подглаве

**1. Blob — это дескриптор, а не массив.** `.size` — метаданное, читается мгновенно. `.arrayBuffer()` копирует данные в RAM.

**2. `.text()` vs `.arrayBuffer()` — это разные вопросы.** Первое даёт строку, второе — байты. Для бинарного содержимого `.text()` даст replacement-символы.

**3. `.stream()` возвращает не один chunk, а последовательность.** Один chunk — implementation-defined. Для полной картины — обход в цикле.

**4. Slice — O(1), не O(n).** Дескриптор внутри дескриптора. Копии данных нет. Это видно на бенчмарке.

**5. Sum of slices = size of original.** Простая проверка, что нарезка корректна.

**6. CRC32(concat(chunks)) == CRC32(arrayBuffer).** Stream не теряет байты. Валидный инвариант.

**7. File extends Blob.** Все операции с Blob работают над File без изменений. Не нужно дублировать функции.

**8. Размер в байтах, не в символах.** `text.len()` в Rust — байты. `text.chars().count()` — символы. Эмодзи стоит 4 байта, но один char.

**9. Hex-dump полезен для отладки.** Читать байты глазами быстрее, чем выводить массив чисел.

**10. `Reflect::construct` для конструктора Blob.** Сигнатуры `Blob::new_with_…` в `web-sys` менялись. Reflect работает везде.

**11. `releaseLock()` после чтения stream.** Освобождать reader — правило работы с `ReadableStream`. Иначе ресурс держится до GC.

**12. `format_str(format_args!(…))` для `format!` в tail-позиции.** На WASM компилятор иногда выводит тип макроса как `()`.

**13. `set_scroll_top(scroll_height() as f64)`.** Сигнатуры методов `web-sys` различаются между версиями. Доверяйте компилятору.

---

## 20.1.14. Чек-лист

- [ ] `from_text` / `from_bytes` — создание Blob из строки и байтов.
- [ ] `.size` — метаданное, читается мгновенно.
- [ ] `.text()` → String, `.arrayBuffer()` → Vec<u8>.
- [ ] `.stream()` первый — один chunk; `.stream()` всё — обход в цикле.
- [ ] Sum of chunks = size of Blob.
- [ ] CRC32(concat) = CRC32(arrayBuffer).
- [ ] `slice(start, end)` — дескриптор, MIME наследуется.
- [ ] Sum of slice sizes = original size.
- [ ] Slice дешевле arrayBuffer в 30–50× (бенчмарк 100 × 100).
- [ ] `File` детектируется по `name` + `lastModified`.
- [ ] File поддерживает все операции Blob.
- [ ] `is_file` на UNDEFINED / NULL → false.
- [ ] `format_str(format_args!(…))` для строк в tail-позиции.
- [ ] `.ok()` вместо `?` в функциях, возвращающих `()`.
- [ ] `set_scroll_top(h as f64)` — с явным кастом.
- [ ] Все методы `web-sys` — snake_case.

---

## 20.1.15. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты blob-контейнера в headless Chrome
wasm-pack test --headless --chrome --test wasm_blob_container

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** изменений. CRC32 и hex-dump реализованы в модуле.

---

## 20.1.16. Что дальше

Подглава 20.1 раскрыла **внутреннее устройство Blob**: что это дескриптор, как читается, как режется, чем отличается от ArrayBuffer. Теперь можно переходить к следующим подглавам:

- **20.2 — Object URL: жизненный цикл и утечки.** Создание URL, `revokeObjectURL` в цифрах, что происходит при незакрытом URL, замеры памяти через `performance.memory`.
- **20.3 — Скачивание: три пути.** `<a download>`, `showSaveFilePicker`, `navigator.share`. Когда какой выбирать.

В главе 20 (первой итерации) мы уже затронули эти темы — теперь они разворачиваются в самостоятельные подглавы с фокусом на одном wow-эффекте каждая.