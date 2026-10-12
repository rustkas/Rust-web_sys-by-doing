# 19.12. Практический проект: локальный CSV-редактор

## Обновлённая структура проекта

```
hello-wasm02/
├── src/
│   ├── csv_editor.rs             ← НОВЫЙ: CSV + hash + I/O + модель документа
│   ├── app.rs                     ← ДЕМО главы 19.12 (только этот сценарий)
│   ├── lib.rs                     ← +pub mod csv_editor
│   └── … (остальные модули проекта)
├── tests/
│   ├── wasm_csv_editor.rs         ← НОВЫЙ: parser, hash, OPFS roundtrip
│   └── … (остальные тесты)
├── pkg/                           ← артефакты wasm-pack
└── web/
    ├── index.html
    └── main.js
```

### Краткий комментарий

**`src/csv_editor.rs`** — самодостаточный модуль локального CSV-редактора. Не использует `fsa_api_*` из предыдущих глав: всё общение с файловой системой идёт через `js_sys::Reflect`, всё общение со стандартной библиотекой — через `sha2` + `hex`. Модуль автономен: если удалить весь `fsa_api_*`, он продолжит компилироваться.

**`tests/wasm_csv_editor.rs`** — тесты парсера, сериализатора, хеша, roundtrip в OPFS.

**`src/app.rs`** — демо главы 19.12: мини-CSV-редактор с таблицей ячеек, автосохранением по дебаунсу, SHA-256-индикатором «есть ли несохранённые изменения», drag&drop и экспортом через download.

### Ссылки для изучения

- [RFC 4180 — CSV](https://www.rfc-editor.org/rfc/rfc4180) — каноничная спецификация CSV.
- [MDN: File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) — FSA в браузере.
- [MDN: DataTransfer.files](https://developer.mozilla.org/en-US/docs/Web/API/DataTransfer/files) — drag&drop файлов.
- [Chrome: Blob URL](https://developer.chrome.com/docs/web-platform/blob-urls) — экспорт через `URL.createObjectURL`.
- [Rust: sha2 crate](https://docs.rs/sha2/latest/sha2/) — SHA-256 в чистом Rust.

---

## 19.12.1. Ключевые идеи

**1. Полный workflow за один экран.** Открыть → распарсить → редактировать → автосохранить → скачать. Каждый шаг виден пользователю, каждый шаг можно отменить.

**2. Автосохранение — не на каждое нажатие.** Сохраняем через **2 секунды бездействия**. Иначе каждый `input`-event порождал бы запись в OPFS — это медленно и разрушительно для диска. Дебаунс — стандартный паттерн: «реагируем, когда пользователь перестал действовать».

**3. Хеш как детектор изменений.** SHA-256 текущего содержимого сравнивается с хешем последнего успешного сохранения. Если они равны — сохранять нечего. Если различаются — есть «грязные» изменения. Хеш — точнее любого флага `dirty`, потому что ловит даже «изменил и вернул обратно».

**4. Открыть файл = открыть handle, а не копию.** Мы храним `FileSystemFileHandle` в state и переиспользуем его при каждом сохранении. Никаких повторных `showOpenFilePicker` — иначе пользователь видел бы диалог на каждое сохранение.

**5. Два независимых входа.** Пользователь может:
- открыть CSV-файл, который уже лежит в OPFS (автозагрузка при старте);
- перетащить CSV-файл из проводника на страницу (drag&drop).

Оба пути попадают в один `CsvDoc`.

---

## 19.12.2. Новый модуль: `src/csv_editor.rs`

```rust
//! Локальный CSV-редактор с автосохранением в OPFS.
//!
//! **Что демонстрирует.** Полный workflow локального
//! файлового инструмента: открыть → распарсить → редактировать
//! → автосохранить (с дебаунсом) → проверить хеш → экспортировать.
//!
//! **Ключевые идеи.**
//! 1. **CSV parsing** — RFC 4180 в минимуме: кавычки, экранирование,
//!    переводы строк внутри ячеек. Без внешних крейтов.
//! 2. **SHA-256 хеш содержимого** — «отпечаток» состояния.
//!    По нему видно, есть ли несохранённые изменения.
//! 3. **Autosave с дебаунсом** — сохранение не на каждое нажатие,
//!    а через N миллисекунд бездействия.
//! 4. **Автономность** — модуль не зависит от других глав. Всё
//!    взаимодействие с ФС — через `js_sys::Reflect`, чтобы
//!    работать независимо от версии `web-sys`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции `if/else`/
//! `match`/блока иногда выводится как `()` → E0308. Все такие
//! места обёрнуты в `format_str(format_args!(…))`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

// ===========================================================================
// Строковые хелперы
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

// ===========================================================================
// CSV parser / serializer (чистый Rust, host-тестируемый)
// ===========================================================================

/// Разобрать CSV-строку в матрицу ячеек.
///
/// Поддерживает:
/// * поля, разделённые запятыми;
/// * строки, разделённые `\n` или `\r\n`;
/// * кавычки `"..."` вокруг поля;
/// * экранирование `""` внутри поля-в-кавычках;
/// * переносы строк внутри кавычек (не разрывают поле).
pub fn parse_csv(input: &str) -> Vec<Vec<String>> {
    let mut rows: Vec<Vec<String>> = Vec::new();
    let mut row: Vec<String> = Vec::new();
    let mut field = String::new();
    let mut chars = input.chars().peekable();
    let mut in_quotes = false;

    while let Some(c) = chars.next() {
        if in_quotes {
            if c == '"' {
                if chars.peek() == Some(&'"') {
                    field.push('"');
                    chars.next();
                } else {
                    in_quotes = false;
                }
            } else {
                field.push(c);
            }
        } else {
            match c {
                '"' => in_quotes = true,
                ',' => {
                    row.push(field.clone());
                    field.clear();
                }
                '\r' => {
                    // Игнорируем — обработаем `\n`.
                }
                '\n' => {
                    row.push(field.clone());
                    field.clear();
                    rows.push(row.clone());
                    row.clear();
                }
                _ => field.push(c),
            }
        }
    }
    // «Хвост» — последняя строка без завершающего \n.
    if !field.is_empty() || !row.is_empty() {
        row.push(field);
        rows.push(row);
    }
    rows
}

/// Сериализовать матрицу в CSV-строку.
///
/// Ячейка оборачивается в кавычки, если содержит `,`, `"`, `\n`
/// или `\r`. Внутренние кавычки удваиваются.
pub fn serialize_csv(rows: &[Vec<String>]) -> String {
    let mut out = String::new();
    for row in rows {
        let mut first = true;
        for cell in row {
            if !first {
                out.push(',');
            }
            first = false;
            let needs_quotes = cell.contains(',')
                || cell.contains('"')
                || cell.contains('\n')
                || cell.contains('\r');
            if needs_quotes {
                out.push('"');
                for c in cell.chars() {
                    if c == '"' {
                        out.push('"');
                    }
                    out.push(c);
                }
                out.push('"');
            } else {
                out.push_str(cell);
            }
        }
        out.push('\n');
    }
    out
}

// ===========================================================================
// SHA-256 (хеш содержимого)
// ===========================================================================

/// SHA-256 в hex. Использует крейт `sha2` + `hex`.
pub fn sha256_hex(bytes: &[u8]) -> String {
    use sha2::{Digest, Sha256};
    let mut h = Sha256::new();
    h.update(bytes);
    let out = h.finalize();
    hex::encode(out)
}

/// Короткая версия хеша — первые 12 символов. Для UI.
pub fn short_hash(full: &str) -> String {
    if full.len() <= 12 {
        return full.to_string();
    }
    let s: String = full.chars().take(12).collect();
    s
}

// ===========================================================================
// OPFS: чтение и запись через Reflect
// ===========================================================================

/// Получить корень OPFS.
pub async fn opfs_root() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let nav_js = js_sys::Reflect::get(&global, &JsValue::from_str("navigator"))?;

    let storage = js_sys::Reflect::get(&nav_js, &JsValue::from_str("storage"))?;
    if storage.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }

    let get_dir = js_sys::Reflect::get(&storage, &JsValue::from_str("getDirectory"))?;
    if !get_dir.is_function() {
        return Err(JsValue::from_str("getDirectory не функция"));
    }
    let f: js_sys::Function = get_dir
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Function"))?;
    let p: js_sys::Promise = f
        .call0(&storage)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    JsFuture::from(p).await
}

/// Получить handle файла в директории.
/// `create = true` — идемпотентно создаст при отсутствии.
pub async fn file_handle_in(
    dir: &JsValue,
    name: &str,
    create: bool,
) -> Result<JsValue, JsValue> {
    let method = js_sys::Reflect::get(dir, &JsValue::from_str("getFileHandle"))?;
    if !method.is_function() {
        return Err(JsValue::from_str("getFileHandle не функция"));
    }
    let f: js_sys::Function = method
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Function"))?;

    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("create"),
        &JsValue::from_bool(create),
    )?;

    let p_js = f.call2(dir, &JsValue::from_str(name), &options)?;
    let p: js_sys::Promise = p_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    JsFuture::from(p).await
}

/// Прочитать содержимое файла через `File.arrayBuffer()`.
pub async fn read_file_bytes(handle: &JsValue) -> Result<Vec<u8>, JsValue> {
    let get_file = js_sys::Reflect::get(handle, &JsValue::from_str("getFile"))?;
    let f: js_sys::Function = get_file
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFile не функция"))?;
    let p: js_sys::Promise = f
        .call0(handle)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let file = JsFuture::from(p).await?;

    let ab_method = js_sys::Reflect::get(&file, &JsValue::from_str("arrayBuffer"))?;
    let ab_fn: js_sys::Function = ab_method
        .dyn_into()
        .map_err(|_| JsValue::from_str("arrayBuffer не функция"))?;
    let ab_p: js_sys::Promise = ab_fn
        .call0(&file)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let ab_value = JsFuture::from(ab_p).await?;
    let ab: js_sys::ArrayBuffer = ab_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let u8arr = js_sys::Uint8Array::new(&ab);
    let len = u8arr.length() as usize;
    let mut out: Vec<u8> = vec![0u8; len];
    u8arr.copy_to(&mut out);
    Ok(out)
}

/// Записать байты в файл через `createWritable`.
///
/// `createWritable()` возвращает поток; пишем в него и закрываем.
/// `close()` — это фактический commit: до него данные не на диске.
pub async fn write_file_bytes(
    handle: &JsValue,
    data: &[u8],
) -> Result<(), JsValue> {
    let cw = js_sys::Reflect::get(handle, &JsValue::from_str("createWritable"))?;
    let cw_fn: js_sys::Function = cw
        .dyn_into()
        .map_err(|_| JsValue::from_str("createWritable не функция"))?;
    let p: js_sys::Promise = cw_fn
        .call0(handle)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let writable = JsFuture::from(p).await?;

    // writable.write(u8arr)
    let write = js_sys::Reflect::get(&writable, &JsValue::from_str("write"))?;
    let write_fn: js_sys::Function = write
        .dyn_into()
        .map_err(|_| JsValue::from_str("write не функция"))?;
    let u8arr = js_sys::Uint8Array::from(data);
    let w_p: js_sys::Promise = write_fn
        .call1(&writable, &u8arr)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("write не Promise"))?;
    JsFuture::from(w_p).await?;

    // writable.close()
    let close = js_sys::Reflect::get(&writable, &JsValue::from_str("close"))?;
    let close_fn: js_sys::Function = close
        .dyn_into()
        .map_err(|_| JsValue::from_str("close не функция"))?;
    let c_p: js_sys::Promise = close_fn
        .call0(&writable)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("close не Promise"))?;
    JsFuture::from(c_p).await?;
    Ok(())
}

// ===========================================================================
// Модель документа
// ===========================================================================

/// Состояние CSV-документа в памяти.
#[derive(Clone)]
pub struct CsvDoc {
    pub rows: Vec<Vec<String>>,
    /// Хеш содержимого, который **последний раз успешно записан**.
    pub saved_hash: String,
}

impl CsvDoc {
    /// Хеш текущего содержимого — по нему сравниваем с saved_hash.
    pub fn current_hash(&self) -> String {
        let text = serialize_csv(&self.rows);
        sha256_hex(text.as_bytes())
    }

    /// Есть ли несохранённые изменения.
    pub fn is_dirty(&self) -> bool {
        let cur = self.current_hash();
        cur != self.saved_hash
    }

    /// Пометить, что текущее состояние успешно записано.
    pub fn mark_saved(&mut self) {
        self.saved_hash = self.current_hash();
    }

    /// Размер в байтах (CSV-текст).
    pub fn byte_size(&self) -> usize {
        let text = serialize_csv(&self.rows);
        text.len()
    }
}

/// Сгенерировать seed-документ с примером данных.
pub fn seed_doc() -> CsvDoc {
    let rows: Vec<Vec<String>> = vec![
        vec!["name".into(), "email".into(), "age".into()],
        vec!["Alice".into(), "alice@example.com".into(), "30".into()],
        vec!["Bob".into(), "bob@example.com".into(), "25".into()],
        vec!["Carol".into(), "carol@example.com".into(), "35".into()],
        vec!["Dave".into(), "dave@example.com".into(), "28".into()],
    ];
    let text = serialize_csv(&rows);
    let h = sha256_hex(text.as_bytes());
    CsvDoc {
        rows,
        saved_hash: h,
    }
}
```

### Что важно в модуле

**1. CSV parser — RFC 4180 в минимуме.** Поддерживает кавычки, экранирование `""`, переводы строк внутри ячеек. Никаких зависимостей — это чистый Rust, тестируется host-тестами.

**2. `serialize_csv` асимметричен парсеру.** Парсер понимает больше форматов, сериализатор выдаёт один каноничный. Это правильно: чтение должно быть терпимым, запись — предсказуемой.

**3. SHA-256 через `sha2` + `hex`.** Оба крейта уже в `Cargo.toml`. `short_hash` — для UI, `sha256_hex` — для сравнения.

**4. Никаких зависимостей от `web_sys`.** OPFS-функции работают через `js_sys::Reflect` — модуль автономен, компилируется независимо от конфигурации `web-sys`.

**5. `CsvDoc::is_dirty()` через хеш, а не через флаг.** Это точнее: пользователь мог изменить ячейку и вернуть обратно — `is_dirty` вернёт `false`, потому что хеш совпадает. С флагом `dirty = true` мы бы зря записали файл.

---

## 19.12.3. Добавление модуля в `lib.rs`

```rust
// … существующие модули …
pub mod fsa_api_observer;
pub mod fsa_api_wasm;
pub mod csv_editor;   // ← НОВЫЙ

// …
```

---

## 19.12.4. Тесты: `tests/wasm_csv_editor.rs`

```rust
//! Тесты `src/csv_editor.rs`.
//!
//! **Что проверяем.**
//! * CSV parser: простые, кавычки, экранирование, перенос строки в кавычках.
//! * Roundtrip: parse → serialize → parse без потерь.
//! * SHA-256: детерминирован, меняется при изменении.
//! * `CsvDoc::is_dirty()`: true/false по хешу.
//! * OPFS roundtrip: записать → прочитать → сравнить.

use wasm_bindgen_test::*;

use hello_wasm02::csv_editor::{
    file_handle_in, opfs_root, parse_csv, read_file_bytes, seed_doc, serialize_csv,
    sha256_hex, short_hash, write_file_bytes, CsvDoc,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// CSV parser
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_parse_empty() {
    let r = parse_csv("");
    assert!(r.is_empty());
}

#[wasm_bindgen_test]
fn test_parse_simple() {
    let r = parse_csv("a,b,c\n1,2,3\n");
    assert_eq!(r.len(), 2);
    assert_eq!(r[0], vec!["a", "b", "c"]);
    assert_eq!(r[1], vec!["1", "2", "3"]);
}

#[wasm_bindgen_test]
fn test_parse_no_trailing_newline() {
    let r = parse_csv("a,b\n1,2");
    assert_eq!(r.len(), 2);
    assert_eq!(r[1], vec!["1", "2"]);
}

#[wasm_bindgen_test]
fn test_parse_quoted() {
    let r = parse_csv(r#""hello, world",x"#);
    assert_eq!(r.len(), 1);
    assert_eq!(r[0], vec!["hello, world", "x"]);
}

#[wasm_bindgen_test]
fn test_parse_escaped_quotes() {
    let r = parse_csv(r#""he said ""hi""",x"#);
    assert_eq!(r[0], vec![r#"he said "hi""#, "x"]);
}

#[wasm_bindgen_test]
fn test_parse_newline_in_quotes() {
    let r = parse_csv("\"line1\nline2\",x");
    assert_eq!(r.len(), 1);
    assert_eq!(r[0][0], "line1\nline2");
}

#[wasm_bindgen_test]
fn test_parse_crlf() {
    let r = parse_csv("a,b\r\n1,2\r\n");
    assert_eq!(r.len(), 2);
    assert_eq!(r[0], vec!["a", "b"]);
    assert_eq!(r[1], vec!["1", "2"]);
}

// ---------------------------------------------------------------------------
// Serialize + roundtrip
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_serialize_simple() {
    let rows = vec![
        vec!["a".to_string(), "b".to_string()],
        vec!["1".to_string(), "2".to_string()],
    ];
    assert_eq!(serialize_csv(&rows), "a,b\n1,2\n");
}

#[wasm_bindgen_test]
fn test_serialize_quotes() {
    let rows = vec![vec![
        "with, comma".to_string(),
        "with \"quote\"".to_string(),
    ]];
    let out = serialize_csv(&rows);
    assert_eq!(out, "\"with, comma\",\"with \"\"quote\"\"\"\n");
}

#[wasm_bindgen_test]
fn test_roundtrip() {
    let original = vec![
        vec!["a,b".to_string(), "c\"d".to_string(), "e\nf".to_string()],
        vec!["1".to_string(), "2".to_string(), "3".to_string()],
    ];
    let text = serialize_csv(&original);
    let parsed = parse_csv(&text);
    assert_eq!(parsed, original);
}

// ---------------------------------------------------------------------------
// SHA-256
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_sha256_deterministic() {
    let a = sha256_hex(b"hello");
    let b = sha256_hex(b"hello");
    assert_eq!(a, b);
    assert_eq!(a.len(), 64);
}

#[wasm_bindgen_test]
fn test_sha256_changes_on_edit() {
    let a = sha256_hex(b"hello");
    let b = sha256_hex(b"hello!");
    assert_ne!(a, b);
}

#[wasm_bindgen_test]
fn test_short_hash() {
    let full = "abcdef0123456789";
    assert_eq!(short_hash(full), "abcdef012345");
    // Короткий хеш остаётся как есть.
    assert_eq!(short_hash("abc"), "abc");
}

// ---------------------------------------------------------------------------
// CsvDoc
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn test_doc_dirty_tracking() {
    let mut doc = seed_doc();
    assert!(!doc.is_dirty(), "новый seed-документ не грязный");

    doc.rows[1][0] = "Alice2".to_string();
    assert!(doc.is_dirty(), "после правки — грязный");

    // Вернули как было — снова чистый.
    doc.rows[1][0] = "Alice".to_string();
    assert!(!doc.is_dirty(), "вернули — снова чистый");
}

#[wasm_bindgen_test]
fn test_doc_mark_saved() {
    let mut doc = seed_doc();
    doc.rows[1][0] = "Alice2".to_string();
    assert!(doc.is_dirty());

    doc.mark_saved();
    assert!(!doc.is_dirty(), "после mark_saved — чистый");
}

// ---------------------------------------------------------------------------
// OPFS roundtrip
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_opfs_csv_roundtrip() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = match file_handle_in(&root, "csv_test_roundtrip.csv", true).await {
        Ok(h) => h,
        Err(_) => return,
    };

    let doc = seed_doc();
    let text = serialize_csv(&doc.rows);

    write_file_bytes(&handle, text.as_bytes())
        .await
        .expect("write");

    let back = read_file_bytes(&handle).await.expect("read");
    let back_text = String::from_utf8_lossy(&back).to_string();
    assert_eq!(back_text, text, "содержимое совпадает");

    // Повторная запись — идемпотентна (тот же хеш → тот же размер).
    write_file_bytes(&handle, text.as_bytes())
        .await
        .expect("write 2");
    let back2 = read_file_bytes(&handle).await.expect("read 2");
    assert_eq!(back, back2);
}

#[wasm_bindgen_test]
async fn test_opfs_persists_across_reads() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = match file_handle_in(&root, "csv_test_persist.csv", true).await {
        Ok(h) => h,
        Err(_) => return,
    };

    // Записываем одну версию.
    write_file_bytes(&handle, b"v1\n").await.expect("write v1");

    // Отдельно читаем — тот же handle, тот же файл.
    let back = read_file_bytes(&handle).await.expect("read");
    assert_eq!(&back, b"v1\n");
}
```

### Почему эти тесты

**Парсер** — единственная часть, которую действительно можно и нужно тестировать программно. Строки, кавычки, переносы — всё это проверяется без браузера (host-тесты), но мы держим их в WASM-файле для единообразия.

**Roundtrip** — гарантия, что `parse ∘ serialize` даёт идентичность. Это самая частая ошибка в CSV-библиотеках.

**Хеш** — два свойства: детерминизм (одинаковый вход → одинаковый выход) и чувствительность (разный вход → разный выход).

**`CsvDoc`** — самое важное: `is_dirty()` возвращает `false` и при «не трогал», и при «изменил и вернул обратно». Это то, что отличает хеш от флага.

**OPFS roundtrip** — минимальная проверка, что модуль работает с реальной файловой системой. Без user gesture, через OPFS.

---

## 19.12.5. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты CSV-редактора в headless Chrome
wasm-pack test --headless --chrome --test wasm_csv_editor

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** изменений. `sha2`, `hex`, `wasm-bindgen-futures`, `js-sys` — уже подключены.

---

## 19.12.6. Анализ и список доработок

### Что проявилось на первой версии

**1. Сохранение на каждое нажатие клавиши.** Первая версия вешала `input`-listener, который сразу писал в OPFS. Это создавало лаг: каждое нажатие → `createWritable` → `write` → `close` → несколько миллисекунд ожидания. При быстром наборе UI подвисал.

**2. Нет индикатора «сохранено/не сохранено».** Пользователь не видит, ушли ли его правки на диск.

**3. Одно поле ввода на весь CSV.** Textarea технически проще, но пользователь ожидает редактор ячеек.

**4. Нет drag&drop.** Файл нельзя перетащить с рабочего стола.

**5. Нет экспорта.** Результат нельзя сохранить на диск — только в OPFS.

**6. `createWritable` требует user gesture?** Нет. В отличие от `showSaveFilePicker`, `createWritable` работает без клика — это OPFS. Но `showSaveFilePicker` — да, требует. Для экспорта нужна отдельная кнопка с user gesture.

### Список доработок

- **A.** **Дебаунс автосохранения — 2 секунды.** Не пишем на каждое нажатие. Отдельный polling-loop: раз в 500 мс проверяет, «прошло ли 2 секунды с последней правки, и есть ли несохранённые изменения».
- **B.** **SHA-256 индикатор.** Показываем `текущий хеш` vs `сохранённый хеш`. Если совпадают — зелёный «✓ сохранено». Если разные — жёлтый «● есть несохранённые изменения».
- **C.** **Таблица ячеек.** Каждая ячейка — `<input type="text">`. Один listener на таблице (event delegation) обновляет `doc.rows[r][c]`.
- **D.** **Drag & drop CSV-файла.** Body-level `dragover` + `drop` → читаем файл, парсим, заменяем документ, сохраняем в OPFS.
- **E.** **Кнопка «Скачать».** Через `showSaveFilePicker` (с user gesture) или через Blob URL + `<a download>`. Второе проще и работает везде.
- **F.** **Кнопка «Сбросить к seed».** Восстанавливает исходный пример — удобно для повторных демонстраций.
- **G.** **Кнопка «Сохранить сейчас».** Ручное сохранение в обход дебаунса.

Все эти доработки реализованы ниже в `app.rs`.

---

## 19.12.7. Финальный `src/app.rs`

```rust
//! Демо главы 19.12 — локальный CSV-редактор с автосохранением.
//!
//! **Идея.** Полный workflow локального файлового инструмента:
//! открыть CSV → редактировать ячейки → автосохранить (через 2 сек
//! бездействия) → проверить хеш → скачать.
//!
//! **Wow-моменты:**
//! 1. **Автосохранение с дебаунсом.** Правите ячейку — статус
//!    становится «есть изменения». Перестали печатать — через
//!    2 секунды файл в OPFS обновлён, статус «✓ сохранено».
//! 2. **Хеш как индикатор.** Видно два хеша: текущий и сохранённый.
//!    Совпадают — зелёный, различаются — жёлтый.
//! 3. **Drag & drop.** Перетащите CSV-файл на страницу — он
//!    заменит текущий документ.
//! 4. **Экспорт.** Кнопка «Скачать» отдаёт текущее содержимое
//!    через Blob URL.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции — через
//! `format_str(format_args!(…))`.
//!
//! **Правило snake_case.** `create_element`, `set_attribute`,
//! `append_child`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, Element, HtmlElement, HtmlInputElement};

use crate::csv_editor::{
    file_handle_in, opfs_root, parse_csv, read_file_bytes, seed_doc, serialize_csv,
    sha256_hex, short_hash, write_file_bytes, CsvDoc,
};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[19.12]";

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

const FILE_NAME: &str = "contacts.csv";
const AUTOSAVE_DELAY_MS: f64 = 2000.0;

// ===========================================================================
// Состояние
// ===========================================================================

struct DemoState {
    status: Element,
    pill: Element,
    table: Element,
    hash_current: Element,
    hash_saved: Element,
    size_info: Element,
    log_body: Element,

    root: RefCell<Option<JsValue>>,
    handle: RefCell<Option<JsValue>>,
    doc: RefCell<CsvDoc>,

    /// Время последней правки в миллисекундах (Date::now).
    last_edit_ms: Cell<f64>,
    /// Идёт ли прямо сейчас запись — не запускаем параллельных.
    saving: Cell<bool>,
    /// Флаг, что фоновый цикл автосохранения уже запущен.
    autosave_running: Cell<bool>,
    /// Отображать ли подсказку после успешного сохранения.
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
.status-bar.ok   { color: #3fb950; }
.status-bar.warn { color: #d29922; }
.status-bar.err  { color: #f85149; }

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

.controls { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 12px; align-items: center; }
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

.pill {
  display: inline-flex; align-items: center; gap: 6px;
  padding: 5px 12px; border-radius: 999px;
  font-size: 12px; font-family: ui-monospace, monospace;
  border: 1px solid transparent;
}
.pill.saved {
  color: #3fb950;
  background: rgba(46, 160, 67, .10);
  border-color: rgba(46, 160, 67, .35);
}
.pill.dirty {
  color: #d29922;
  background: rgba(210, 153, 34, .10);
  border-color: rgba(210, 153, 34, .35);
}
.pill.saving {
  color: #79c0ff;
  background: rgba(31, 111, 235, .10);
  border-color: rgba(31, 111, 235, .35);
}

.csv-table {
  width: 100%;
  border-collapse: collapse;
  font-family: ui-monospace, monospace; font-size: 13px;
}
.csv-table th {
  text-align: left; padding: 8px 10px;
  background: #21262d; color: #8b949e;
  font-weight: 500; font-size: 11px;
  text-transform: uppercase; letter-spacing: .5px;
  border: 1px solid #30363d;
}
.csv-table td {
  padding: 0;
  border: 1px solid #30363d;
}
.csv-table input {
  width: 100%; box-sizing: border-box;
  background: #0d1117; color: #e6edf3;
  border: none; outline: none;
  padding: 8px 10px;
  font-family: ui-monospace, monospace; font-size: 13px;
}
.csv-table input:focus {
  background: #161b22;
  box-shadow: inset 0 0 0 1px #1f6feb;
}

.hash-row {
  display: flex; gap: 24px; flex-wrap: wrap;
  font-family: ui-monospace, monospace; font-size: 12px;
  color: #8b949e; margin-top: 12px; padding-top: 12px;
  border-top: 1px solid #30363d;
}
.hash-row .item { display: flex; gap: 8px; }
.hash-row .label { color: #6e7681; }
.hash-row code {
  color: #79c0ff;
  background: rgba(31, 111, 235, .08);
  padding: 1px 6px; border-radius: 3px;
}
.hash-row .eq { color: #3fb950; }
.hash-row .neq { color: #d29922; }

.drop-hint {
  margin: 0 32px 24px;
  padding: 12px 16px;
  background: rgba(31, 111, 235, .05);
  border: 1px dashed rgba(31, 111, 235, .35);
  border-radius: 8px;
  color: #8b949e; font-size: 12px;
  font-family: ui-monospace, monospace;
}
.drop-hint b { color: #79c0ff; font-weight: 500; }

.drop-active {
  outline: 2px dashed #1f6feb;
  outline-offset: -8px;
}

.log-body {
  padding: 12px 16px; max-height: 160px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; white-space: pre-wrap; }
.log-line.info    { color: #8b949e; }
.log-line.ok      { color: #3fb950; }
.log-line.err     { color: #f85149; }
.log-line.bench   { color: #79c0ff; }
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
    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let max: f64 = el.scroll_height() as f64;
        el.set_scroll_top(max);
    }
}

fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

// ===========================================================================
// Рендер таблицы
// ===========================================================================

fn render_table(state: &Rc<DemoState>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let doc = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    state.table.set_inner_html("");

    let rows = state.doc.borrow().rows.clone();
    if rows.is_empty() {
        return Ok(());
    }

    // Заголовок — первая строка CSV.
    let thead = doc.create_element("thead")?;
    let tr_head = doc.create_element("tr")?;
    for (c, cell) in rows[0].iter().enumerate() {
        let th = doc.create_element("th")?;
        th.set_text_content(Some(cell));
        th.set_attribute("data-col", &c.to_string())?;
        tr_head.append_child(&th)?;
    }
    thead.append_child(&tr_head)?;
    state.table.append_child(&thead)?;

    // Тело — все строки со второй.
    let tbody = doc.create_element("tbody")?;
    for (r, row) in rows.iter().enumerate().skip(1) {
        let tr = doc.create_element("tr")?;
        for (c, cell) in row.iter().enumerate() {
            let td = doc.create_element("td")?;
            let input = doc.create_element("input")?;
            input.set_attribute("type", "text")?;
            input.set_attribute("data-row", &r.to_string())?;
            input.set_attribute("data-col", &c.to_string())?;
            let input_el: HtmlInputElement = input
                .dyn_into()
                .map_err(|_| JsValue::from_str("не HtmlInputElement"))?;
            input_el.set_value(cell);
            td.append_child(&input_el)?;
            tr.append_child(&td)?;
        }
        tbody.append_child(&tr)?;
    }
    state.table.append_child(&tbody)?;
    Ok(())
}

// ===========================================================================
// Рендер статуса
// ===========================================================================

fn render_status(state: &Rc<DemoState>) {
    let doc_borrow = state.doc.borrow();
    let dirty = doc_borrow.is_dirty();
    let cur = doc_borrow.current_hash();
    let saved = doc_borrow.saved_hash.clone();
    let size = doc_borrow.byte_size();
    drop(doc_borrow);

    let saving = state.saving.get();

    // Pill.
    if saving {
        state.pill.set_class_name("pill saving");
        state.pill.set_text_content(Some("◐ сохраняю…"));
    } else if dirty {
        state.pill.set_class_name("pill dirty");
        state.pill.set_text_content(Some("● есть несохранённые изменения"));
    } else {
        state.pill.set_class_name("pill saved");
        state.pill.set_text_content(Some("✓ сохранено"));
    }

    // Хеши.
    state.hash_current.set_text_content(Some(&short_hash(&cur)));
    state.hash_saved.set_text_content(Some(&short_hash(&saved)));

    let eq_class: &str = if cur == saved { "eq" } else { "neq" };
    state.hash_saved.set_class_name(eq_class);

    // Размер.
    let size_text: String = format_str(format_args!("размер: {}", fmt_bytes(size)));
    state.size_info.set_text_content(Some(&size_text));
}

// ===========================================================================
// Действия
// ===========================================================================

/// Записать текущий документ в OPFS через handle.
async fn do_save(state: Rc<DemoState>) -> Result<(), JsValue> {
    if state.saving.get() {
        return Ok(());
    }
    state.saving.set(true);
    render_status(&state);

    let handle_opt = state.handle.borrow().clone();
    let handle = match handle_opt {
        Some(h) => h,
        None => {
            state.saving.set(false);
            render_status(&state);
            return Ok(());
        }
    };

    let text = {
        let doc = state.doc.borrow();
        serialize_csv(&doc.rows)
    };

    let res = write_file_bytes(&handle, text.as_bytes()).await;
    match res {
        Ok(()) => {
            let new_hash = sha256_hex(text.as_bytes());
            {
                let mut doc = state.doc.borrow_mut();
                doc.saved_hash = new_hash;
            }
            let line: String = format_str(format_args!(
                "записано {} байт, хеш {}",
                text.len(),
                short_hash(&sha256_hex(text.as_bytes()))
            ));
            log_line(&state, "ok", &line);
        }
        Err(e) => {
            console_err("write_file_bytes failed", &e);
            log_line(&state, "err", "ошибка записи в OPFS");
        }
    }

    state.saving.set(false);
    render_status(&state);
    Ok(())
}

/// Фоновый цикл автосохранения. Раз в 500 мс проверяет:
/// «есть ли несохранённые изменения» и «прошло ли 2 секунды
/// с последней правки». Если да — сохраняет.
fn start_autosave_loop(state: Rc<DemoState>) {
    if state.autosave_running.get() {
        return;
    }
    state.autosave_running.set(true);

    spawn_local(async move {
        while state.autosave_running.get() {
            TimeoutFuture::new(500).await;

            let dirty = state.doc.borrow().is_dirty();
            if !dirty {
                continue;
            }
            if state.saving.get() {
                continue;
            }

            let now = js_sys::Date::now();
            let last = state.last_edit_ms.get();
            if now - last < AUTOSAVE_DELAY_MS {
                continue;
            }

            let _ = do_save(state.clone()).await;
        }
    });
}

/// Обработать изменение ячейки: обновить doc, обновить статус.
fn on_cell_input(state: &Rc<DemoState>, r: usize, c: usize, value: String) {
    {
        let mut doc = state.doc.borrow_mut();
        if r < doc.rows.len() && c < doc.rows[r].len() {
            doc.rows[r][c] = value;
        }
    }
    state.last_edit_ms.set(js_sys::Date::now());
    render_status(state);
}

/// Загрузить документ в OPFS из seed.
async fn load_or_seed(state: Rc<DemoState>) -> Result<(), JsValue> {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(e) => {
            console_err("opfs_root failed", &e);
            set_status(&state, "err", "OPFS недоступен — сохранение не работает");
            log_line(&state, "err", "OPFS недоступен");
            return Ok(());
        }
    };
    *state.root.borrow_mut() = Some(root.clone());

    let handle = match file_handle_in(&root, FILE_NAME, true).await {
        Ok(h) => h,
        Err(e) => {
            console_err("getFileHandle failed", &e);
            log_line(&state, "err", "не удалось получить handle");
            return Ok(());
        }
    };
    *state.handle.borrow_mut() = Some(handle.clone());

    let bytes = read_file_bytes(&handle).await.unwrap_or_default();
    let text = String::from_utf8_lossy(&bytes).to_string();

    if text.trim().is_empty() {
        // Файл пуст — записываем seed.
        let seed = seed_doc();
        let seed_text = serialize_csv(&seed.rows);
        write_file_bytes(&handle, seed_text.as_bytes()).await?;
        *state.doc.borrow_mut() = seed;
        log_line(&state, "ok", "seed-документ записан в OPFS");
    } else {
        let rows = parse_csv(&text);
        let h = sha256_hex(text.as_bytes());
        *state.doc.borrow_mut() = CsvDoc {
            rows,
            saved_hash: h,
        };
        let line: String = format_str(format_args!(
            "загружено из OPFS: {} строк, {} байт",
            state.doc.borrow().rows.len(),
            text.len()
        ));
        log_line(&state, "ok", &line);
    }

    render_table(&state)?;
    render_status(&state);
    Ok(())
}

/// Прочитать входящий File (из drag&drop) и заменить документ.
async fn import_file(state: Rc<DemoState>, file: web_sys::File) -> Result<(), JsValue> {
    let name = file.name();
    let p = file.array_buffer();
    let ab_value = wasm_bindgen_futures::JsFuture::from(p).await?;
    let ab: js_sys::ArrayBuffer = ab_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let u8arr = js_sys::Uint8Array::new(&ab);
    let len = u8arr.length() as usize;
    let mut bytes: Vec<u8> = vec![0u8; len];
    u8arr.copy_to(&mut bytes);

    let text = String::from_utf8_lossy(&bytes).to_string();
    let rows = parse_csv(&text);

    {
        let mut doc = state.doc.borrow_mut();
        doc.rows = rows;
        // В OPFS записан старый файл — хеш должен отличаться,
        // чтобы автопрув сработал.
        doc.saved_hash = String::new();
    }
    state.last_edit_ms.set(js_sys::Date::now());

    let line: String = format_str(format_args!(
        "загружен из drag&drop: {name}, {} строк, {} байт",
        state.doc.borrow().rows.len(),
        text.len()
    ));
    log_line(&state, "ok", &line);

    render_table(&state)?;
    render_status(&state);
    // Автосохранение сработает через 2 секунды через фоновый цикл.
    Ok(())
}

/// Экспорт текущего документа через Blob URL + <a download>.
fn download_current(state: &Rc<DemoState>) -> Result<(), JsValue> {
    let doc = state.doc.borrow();
    let text = serialize_csv(&doc.rows);
    drop(doc);

    let u8arr = js_sys::Uint8Array::from(text.as_bytes());
    let parts = js_sys::Array::new();
    parts.push(&u8arr.buffer());
    let blob = web_sys::Blob::new_with_buffer_source_sequence(&parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let doc_el = window.document().ok_or_else(|| JsValue::from_str("no document"))?;
    let a = doc_el.create_element("a")?;
    a.set_attribute("href", &url)?;
    a.set_attribute("download", "contacts.csv")?;
    // Chrome требует, чтобы <a> был в DOM.
    doc_el.body().ok_or_else(|| JsValue::from_str("no body"))?.append_child(&a)?;
    let a_html: web_sys::HtmlElement = a
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    a_html.click();
    a_html.remove();

    // Отзываем URL через мгновение — по спеке можно сразу, но
    // некоторые браузеры не успевают скачать.
    let url_clone = url.clone();
    let closure = Closure::once(move || {
        web_sys::Url::revoke_object_url(&url_clone).ok();
    });
    window.set_timeout_with_callback_and_timeout_and_arguments_0(
        closure.as_ref().unchecked_ref(),
        1000,
    ).ok();
    closure.forget();

    Ok(())
}

/// Сброс к seed-документу.
async fn reset_to_seed(state: Rc<DemoState>) -> Result<(), JsValue> {
    let seed = seed_doc();
    {
        let mut doc = state.doc.borrow_mut();
        doc.rows = seed.rows;
        // Форсим «грязный» статус — пусть автопрув запишет новый seed.
        doc.saved_hash = String::new();
    }
    state.last_edit_ms.set(js_sys::Date::now());
    log_line(&state, "info", "сброс к seed — файл перезапишется через 2 сек");
    render_table(&state)?;
    render_status(&state);
    Ok(())
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

    // Единое визуальное поле.
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

    // Header.
    let h1 = document.create_element("h1")?;
    h1.set_inner_html(
        "19.12 <code>Локальный CSV-редактор</code>: полный workflow за один экран",
    );
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "открыть · редактировать · автосохранить через 2 сек · хеш как индикатор · drag&drop · export",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    // Intro.
    let intro = document.create_element("div")?;
    intro.set_class_name("intro");
    intro.set_inner_html(
        "<b>Что демонстрируется.</b> Локальный CSV-редактор, работающий поверх OPFS: \
         <code>contacts.csv</code> создаётся при первом заходе, читается при последующих, \
         автосохраняется при правках.<br><br>\
         <b>Автосохранение — с дебаунсом 2 сек.</b> Правьте ячейку — статус «● есть изменения». \
         Перестаньте печатать — через 2 секунды статус «✓ сохранено», хеши совпадут.<br><br>\
         <b>Хеш как детектор.</b> Справа видно два хеша: <code>текущий</code> и \
         <code>сохранённый</code>. Равны — зелёный. Разные — жёлтый. Хеш ловит даже \
         «изменил и вернул обратно» — тогда сохранения не будет, потому что нечего сохранять.<br><br>\
         <b>Два входа.</b> Файл можно перетащить мышью из проводника прямо на страницу — \
         он заменит текущий документ и запишется в OPFS.",
    );
    body.append_child(&intro)?;

    // Status bar.
    let status = document.create_element("div")?;
    status.set_class_name("status-bar");
    status.set_text_content(Some("инициализация…"));
    body.append_child(&status)?;

    // Section ❶ — редактор.
    let (sec1, body1) = make_section(
        &document,
        "❶",
        "CSV-редактор",
        "правьте ячейки — сохранение через 2 сек",
        "Таблица CSV. Первая строка — заголовки. Правьте любую ячейку — \
         документ станет «грязным», через 2 секунды бездействия \
         произойдёт автосохранение в OPFS.",
    )?;
    body.append_child(&sec1)?;

    // Строка с кнопками и pill.
    let controls = document.create_element("div")?;
    controls.set_class_name("controls");
    body1.append_child(&controls)?;

    let pill = document.create_element("div")?;
    pill.set_class_name("pill saved");
    pill.set_text_content(Some("✓ сохранено"));
    pill.set_attribute(
        "title",
        "Индикатор состояния. Зелёный — всё сохранено. Жёлтый — есть \
         несохранённые изменения (ждут автосохранения). Синий — \
         идёт запись прямо сейчас.",
    )?;
    controls.append_child(&pill)?;

    // Таблица.
    let table = document.create_element("table")?;
    table.set_class_name("csv-table");
    body1.append_child(&table)?;

    // Хеши.
    let hash_row = document.create_element("div")?;
    hash_row.set_class_name("hash-row");
    hash_row.set_inner_html(
        "<div class=\"item\">\
           <span class=\"label\">текущий:</span>\
           <code id=\"hash-current\">—</code>\
         </div>\
         <div class=\"item\">\
           <span class=\"label\">сохранённый:</span>\
           <code id=\"hash-saved\">—</code>\
         </div>\
         <div class=\"item\" id=\"size-info\">размер: —</div>",
    );
    body1.append_child(&hash_row)?;

    let hash_current = hash_row
        .query_selector("#hash-current")?
        .ok_or_else(|| JsValue::from_str("no hash-current"))?;
    let hash_saved = hash_row
        .query_selector("#hash-saved")?
        .ok_or_else(|| JsValue::from_str("no hash-saved"))?;
    let size_info = hash_row
        .query_selector("#size-info")?
        .ok_or_else(|| JsValue::from_str("no size-info"))?;

    // Section ❷ — журнал.
    let (sec2, body2) = make_section(
        &document,
        "❷",
        "Журнал операций",
        "",
        "Что произошло: загрузка seed, загрузка из OPFS, автосохранение, \
         импорт через drag&drop.",
    )?;
    body.append_child(&sec2)?;
    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    body2.set_class_name("");
    body2.set_attribute("style", "padding:0")?;
    body2.append_child(&log_body)?;

    // Drop hint.
    let drop_hint = document.create_element("div")?;
    drop_hint.set_class_name("drop-hint");
    drop_hint.set_inner_html(
        "📂 <b>Перетащите CSV-файл</b> в любое место страницы — он заменит \
         текущий документ и запишется в OPFS.",
    );
    body.append_child(&drop_hint)?;

    // State.
    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        pill: pill.clone(),
        table: table.clone(),
        hash_current,
        hash_saved,
        size_info,
        log_body: log_body.clone(),
        root: RefCell::new(None),
        handle: RefCell::new(None),
        doc: RefCell::new(CsvDoc {
            rows: Vec::new(),
            saved_hash: String::new(),
        }),
        last_edit_ms: Cell::new(0.0),
        saving: Cell::new(false),
        autosave_running: Cell::new(false),
        log_lines: RefCell::new(Vec::new()),
    });

    // Кнопки.
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Сохранить сейчас",
            "Немедленно записывает документ в OPFS, не дожидаясь автосохранения.",
            true,
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = do_save(s).await {
                        console_err("do_save failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Скачать CSV",
            "Экспортировать текущий документ на диск через Blob URL. \
             Работает без user gesture только с Blob — showSaveFilePicker требует клика.",
            false,
            false,
            move || {
                if let Err(e) = download_current(&s) {
                    console_err("download failed", &e);
                }
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &controls,
            "Сбросить к примеру",
            "Заменить содержимое seed-документом. Файл в OPFS будет \
             перезаписан через 2 секунды (автосохранение).",
            false,
            true,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = reset_to_seed(s).await {
                        console_err("reset_to_seed failed", &e);
                    }
                });
            },
        )?;
    }

    // Event delegation на таблице: один listener на все ячейки.
    {
        let s = state.clone();
        let on_input = Closure::<dyn FnMut(web_sys::Event)>::new(
            move |e: web_sys::Event| {
                let target = e.target();
                let input = match target.and_then(|t| t.dyn_into::<HtmlInputElement>().ok()) {
                    Some(i) => i,
                    None => return,
                };
                let r = input
                    .get_attribute("data-row")
                    .and_then(|v| v.parse::<usize>().ok());
                let c = input
                    .get_attribute("data-col")
                    .and_then(|v| v.parse::<usize>().ok());
                if let (Some(r), Some(c)) = (r, c) {
                    on_cell_input(&s, r, c, input.value());
                }
            },
        );
        table.add_event_listener_with_callback(
            "input",
            on_input.as_ref().unchecked_ref(),
        )?;
        on_input.forget();
    }

    // Drag & drop: запрещаем дефолтную обработку, чтобы drop сработал.
    {
        let body_clone = body.clone();
        let on_dragover = Closure::<dyn FnMut(web_sys::DragEvent)>::new(
            move |e: web_sys::DragEvent| {
                e.prevent_default();
                body_clone.class_list().add_1("drop-active").ok();
            },
        );
        body.add_event_listener_with_callback(
            "dragover",
            on_dragover.as_ref().unchecked_ref(),
        )?;
        on_dragover.forget();

        let body_clone2 = body.clone();
        let on_dragleave = Closure::<dyn FnMut(web_sys::DragEvent)>::new(
            move |_e: web_sys::DragEvent| {
                body_clone2.class_list().remove_1("drop-active").ok();
            },
        );
        body.add_event_listener_with_callback(
            "dragleave",
            on_dragleave.as_ref().unchecked_ref(),
        )?;
        on_dragleave.forget();
    }
    {
        let s = state.clone();
        let body_clone = body.clone();
        let on_drop = Closure::<dyn FnMut(web_sys::DragEvent)>::new(
            move |e: web_sys::DragEvent| {
                e.prevent_default();
                body_clone.class_list().remove_1("drop-active").ok();

                let dt = match e.data_transfer() {
                    Some(dt) => dt,
                    None => return,
                };
                let files = dt.files();
                if files.is_none() {
                    return;
                }
                let files = files.unwrap();
                if files.length() == 0 {
                    return;
                }
                let file = match files.get(0) {
                    Some(f) => f,
                    None => return,
                };
                let s2 = s.clone();
                spawn_local(async move {
                    if let Err(err) = import_file(s2, file).await {
                        console_err("import_file failed", &err);
                    }
                });
            },
        );
        body.add_event_listener_with_callback(
            "drop",
            on_drop.as_ref().unchecked_ref(),
        )?;
        on_drop.forget();
    }

    // Инициализация.
    {
        let s = state.clone();
        spawn_local(async move {
            console_log("инициализация…");
            set_status(&s, "ok", "готово");

            if let Err(e) = load_or_seed(s.clone()).await {
                console_err("load_or_seed failed", &e);
            }

            // Запускаем фоновый цикл автосохранения.
            start_autosave_loop(s.clone());

            log_line(
                &s,
                "info",
                "готово. Правьте ячейки — автосохранение через 2 секунды бездействия.",
            );
        });
    }

    console_log("run: finished OK");
    Ok(())
}
```

---

## 19.12.8. Что показывает демо

**Секция ❶** — таблица CSV с заголовком и 4 строками данных. Каждая ячейка — `<input>`. Рядом с таблицей: pill-индикатор (зелёный / жёлтый / синий), три кнопки и хеши.

**Секция ❷** — журнал операций: загрузка seed, загрузка из OPFS, автосохранение, импорт.

**Drop-hint** — пунктирная плашка снизу: «перетащите CSV-файл сюда».

### Wow-моменты

**1. Автосохранение с дебаунсом.** Правите ячейку → pill становится жёлтым «● есть несохранённые изменения». Перестаёте печатать → через 2 секунды pill становится зелёным «✓ сохранено». В журнале: `записано 87 байт, хеш a3f4b2c8d1e5`.

**2. Хеш ловит «изменил и вернул обратно».** Поменяйте значение ячейки, потом верните как было. Pill снова зелёный — сохранения не будет, потому что хеш совпал. Это отличается от флага `dirty = true`, который остался бы красным.

**3. Drag & drop.** Перетащите любой CSV-файл из проводника на страницу. Он заменит документ и через 2 секунды запишется в OPFS.

**4. Экспорт.** Кнопка «Скачать CSV» отдаёт файл на диск браузера через Blob URL.

**5. Persistence.** Перезагрузите страницу — ваши правки на месте. `contacts.csv` в OPFS.

---

## 19.12.9. Правила проекта, выученные в этой главе

**1. Дебаунс автосохранения — 2 секунды.** Не на каждое нажатие. Иначе `createWritable` + `write` + `close` запускается десятки раз в секунду, что разрушительно для OPFS и UI.

**2. Хеш содержимого — точнее флага `dirty`.** SHA-256 ловит «изменил и вернул обратно». Флаг — нет.

**3. Handle переиспользуется.** Не открывайте `showOpenFilePicker` на каждое сохранение — держите `FileSystemFileHandle` в state.

**4. `createWritable` — это транзакция.** До `close()` данные не на диске. Если упало между `write` и `close` — на диск ничего не попало.

**5. Event delegation.** Один listener на `<table>` вместо 20 на каждый `<input>`. Меньше closure, меньше памяти, проще код.

**6. `Blob` URL + `<a download>` работает без user gesture.** В отличие от `showSaveFilePicker`, это не требует клика — может запускаться из async-контекста.

**7. Отзыв Blob URL через timeout.** По спеке можно сразу после `click()`, но некоторые браузеры не успевают скачать — пауза 1 секунда надёжнее.

**8. `body.dragover` → `preventDefault`.** Без этого `drop` не сработает — браузер по умолчанию открывает файл в новой вкладке.

**9. Полный workflow — от seed к экспорту.** Не демонстрируйте отдельные API. Покажите, как они складываются в реальный инструмент.

**10. Модуль автономен.** `csv_editor.rs` не импортирует `fsa_api_*`. Всё общение с ФС — через `js_sys::Reflect`. Так он компилируется и работает независимо от конфигурации других глав.

---

## 19.12.10. Чек-лист

- [ ] CSV parser поддерживает кавычки, экранирование, `\r\n`, перенос строк внутри кавычек.
- [ ] `serialize_csv` оборачивает в кавычки ячейки с `,`, `"`, `\n`, `\r`.
- [ ] `parse ∘ serialize` идемпотентен — roundtrip-тест.
- [ ] SHA-256 — детерминирован и меняется при изменении.
- [ ] `is_dirty()` — через хеш, а не через флаг.
- [ ] OPFS-функции через `js_sys::Reflect` — модуль автономен.
- [ ] Дебаунс автосохранения — 2 секунды бездействия.
- [ ] Event delegation на `<table>`, а не listener на каждой ячейке.
- [ ] `body.dragover` с `preventDefault` — иначе drop не сработает.
- [ ] Blob URL отзывается с задержкой 1 секунда.
- [ ] Handle файла хранится в state, не пересоздаётся.
- [ ] Pill-индикатор: зелёный / жёлтый / синий.
- [ ] При старте — загрузка из OPFS или seed.
- [ ] `format_str(format_args!(…))` для всех строковых выражений в tail-позиции.
- [ ] Все методы `web-sys` — snake_case.

---

## 19.12.11. Команды запуска

```bash
# 1. Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# 2. Host-тесты (быстрые, без браузера)
cargo test

# 3. WASM-тесты CSV-редактора в headless Chrome
wasm-pack test --headless --chrome --test wasm_csv_editor

# 4. Запуск демо
npm run start
```

`Cargo.toml` **не требует** изменений. `sha2`, `hex`, `wasm-bindgen-futures`, `js-sys`, `gloo-timers` — уже подключены.

---

## 19.12.12. Что дальше

Мы прошли полный путь работы с файловой системой браузера:

- **19.7** — структура: дерево, пути, diff.
- **19.8** — производительность: OPFS, sync access handle.
- **19.9** — устойчивость: квота, persist.
- **19.10** — реактивность: FileSystemObserver.
- **19.11** — интеграция: Rust-крейты в браузере.
- **19.12** — практика: локальный файловый инструмент.

Дальше — либо **SQLite WASM** (OPFS как файловая система для полноценной SQL-базы), либо **интеграция с сервером** (аутентификация, синхронизация, chunked upload — это материал других глав). Все принципы остаются те же:

- feature detection — первым делом;
- идемпотентность деструктивных операций;
- дебаунс для UI-bound операций;
- хеш вместо флага `dirty`;
- явные ошибки в консоль;
- единое визуальное поле;
- автономные модули без скрытых зависимостей.