# Глава 19.5. Запись файлов: `FileSystemWritableFileStream`

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── lib.rs                       # +2 pub mod fsa_write_*
│   ├── app.rs                       # демо-секция 19.5 (4 блока)
│   ├── fsa_write_session.rs         # модель сессии, состояния, ошибки
│   └── fsa_write_ops.rs             # open / write / seek / truncate / commit / abort + naive-rewrite
└── tests/
    └── wasm_fsa_write.rs            # тесты 19.5 (21 шт.)
```

**Краткий комментарий.** В проекте есть `fsa_api_writable.rs` (сырые обёртки) и `fsa_api_writer.rs` (`write_all_bytes` / `write_all_text` / `write_chunked`). Эта подглава идёт **на уровень семантики транзакции**:

1. **`write()` идёт в буфер, `close()` фиксирует.** До `close` файл на диске не меняется. Незакрытая сессия откатывается через `Drop`.
2. **`abort()` — не «прекратить», а «отменить транзакцию».** Полная противоположность `close`.
3. **`seek + write` — хирургическая правка.** Не перезаписывает весь файл ради 3 байт. **Важно:** `seek + write` **заменяет** байты, а не вставляет — остаток файла остаётся на месте.
4. **`keepExistingData`** — сохранение хвоста; без него `createWritable()` усечёт файл при открытии.
5. **Сессия — конечный автомат.** `write` после `close`/`abort` — ошибка; `close`/`abort` дважды — идемпотентны.

Префикс `fsa_write_*` — уникален; с `fsa_api_writable` / `fsa_api_writer` не пересекается.

## Ссылки

- [MDN: FileSystemWritableFileStream](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemWritableFileStream)
- [MDN: createWritable()](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle/createWritable)
- [WHATWG: File System Standard](https://fs.spec.whatwg.org/)
- [Chrome: Writing to a file](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access#writing_to_a_file)
- [MDN: OPFS (Origin Private File System)](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system)

---

## Исходные модули

### `src/fsa_write_session.rs`

```rust
//! Модель сессии записи: состояния, интенты, ошибки.
//!
//! Сессия — это транзакция. `write()` идёт в буфер, `close()` фиксирует
//! изменения, `abort()` их отбрасывает. Если сессию не закрыть и не
//! откатить явно, `Drop` вызовет `abort()` — незакоммиченная транзакция
//! не оставляет следов.

use crate::fsa_choose_options::format_str;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SessionState {
    Open,
    Committed,
    Aborted,
    Errored,
}

impl SessionState {
    pub fn as_str(self) -> &'static str {
        match self {
            SessionState::Open => return "открыта",
            SessionState::Committed => return "зафиксирована",
            SessionState::Aborted => return "отменена",
            SessionState::Errored => return "ошибка",
        }
    }
    pub fn is_terminal(self) -> bool {
        return !matches!(self, SessionState::Open);
    }
}

#[derive(Debug, Clone, PartialEq)]
pub struct CommitStats {
    pub writes: u32,
    pub bytes_written: u64,
    pub elapsed_ms: f64,
    pub keep_existing: bool,
}

impl CommitStats {
    pub fn summary(&self) -> String {
        let keep_label: &str = if self.keep_existing { "с keep" } else { "без keep" };
        let s: String = format_str(format_args!(
            "Зафиксировано {keep_label}: {} операций, {} байт за {:.0} ms",
            self.writes, self.bytes_written, self.elapsed_ms,
        ));
        return s;
    }
}

/// Статистика «наивной» перезаписи: сколько байт прочитали и записали.
#[derive(Debug, Clone, PartialEq)]
pub struct NaiveStats {
    pub bytes_read: u64,
    pub bytes_written: u64,
    pub elapsed_ms: f64,
}

impl NaiveStats {
    pub fn summary(&self) -> String {
        let s: String = format_str(format_args!(
            "Наивно: read {} байт, write {} байт за {:.0} ms",
            self.bytes_read, self.bytes_written, self.elapsed_ms,
        ));
        return s;
    }
}

#[derive(Debug, Clone, PartialEq)]
pub enum WriteError {
    SessionClosed,
    NoSession,
    Io(String),
}

impl WriteError {
    pub fn explain(&self) -> String {
        match self {
            WriteError::SessionClosed => {
                return "✗ Сессия уже закрыта. write/seek/truncate после close/abort \
                        не сработают — откройте новую через createWritable()."
                    .to_string();
            }
            WriteError::NoSession => {
                return "✗ Сессия не открыта. Сначала откройте writable через createWritable()."
                    .to_string();
            }
            WriteError::Io(msg) => {
                let s: String = format_str(format_args!("✗ Ошибка записи: {msg}"));
                return s;
            }
        }
    }
}
```

### `src/fsa_write_ops.rs`

```rust
//! Реализация записи: открыть сессию, писать, фиксировать, откатывать.
//!
//! Все вызовы через `Reflect::get` + `Function::call*` — сигнатуры методов
//! writable-потока нестабильны между версиями web-sys.
//!
//! `impl Drop` откатывает незакоммиченную сессию через `abort()`.

use js_sys::{Function, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::FileSystemFileHandle;

use crate::fsa_choose_options::format_str;
use crate::fsa_write_session::{CommitStats, NaiveStats, SessionState, WriteError};

/// Активная сессия записи.
pub struct WriteSession {
    stream: JsValue,
    state: SessionState,
    writes: u32,
    bytes_written: u64,
    t0: f64,
    keep_existing: bool,
}

impl WriteSession {
    pub async fn open(
        handle: &FileSystemFileHandle,
        keep_existing: bool,
    ) -> Result<Self, WriteError> {
        let handle_js: &JsValue = handle.as_ref();

        let create_fn_js = Reflect::get(handle_js, &"createWritable".into())
            .map_err(|e| WriteError::Io(format_str(format_args!("createWritable: {e:?}"))))?;
        if create_fn_js.is_undefined() {
            return Err(WriteError::Io("createWritable недоступен".to_string()));
        }
        let create_fn: Function = create_fn_js
            .dyn_into()
            .map_err(|_| WriteError::Io("createWritable не функция".to_string()))?;

        let promise_js = if keep_existing {
            let opts = Object::new();
            Reflect::set(&opts, &"keepExistingData".into(), &JsValue::TRUE)
                .map_err(|e| WriteError::Io(format_str(format_args!("opts: {e:?}"))))?;
            create_fn
                .call1(handle_js, &opts.into())
                .map_err(|e| WriteError::Io(format_str(format_args!("call: {e:?}"))))?
        } else {
            create_fn
                .call0(handle_js)
                .map_err(|e| WriteError::Io(format_str(format_args!("call: {e:?}"))))?
        };
        let promise: js_sys::Promise = promise_js
            .dyn_into()
            .map_err(|_| WriteError::Io("не Promise".to_string()))?;
        let stream = JsFuture::from(promise)
            .await
            .map_err(|e| WriteError::Io(format_str(format_args!("await: {e:?}"))))?;

        let t0: f64 = js_sys::Date::now();
        return Ok(WriteSession {
            stream,
            state: SessionState::Open,
            writes: 0,
            bytes_written: 0,
            t0,
            keep_existing,
        });
    }

    pub fn state(&self) -> SessionState { return self.state; }
    pub fn keep_existing(&self) -> bool { return self.keep_existing; }
    pub fn writes(&self) -> u32 { return self.writes; }
    pub fn bytes_written(&self) -> u64 { return self.bytes_written; }

    fn require_open(&self) -> Result<(), WriteError> {
        if self.state == SessionState::Open {
            return Ok(());
        }
        return Err(WriteError::SessionClosed);
    }

    pub async fn write(&mut self, data: &[u8]) -> Result<(), WriteError> {
        self.require_open()?;
        let u8arr = Uint8Array::from(data);
        let fn_js = Reflect::get(&self.stream, &"write".into())
            .map_err(|e| WriteError::Io(format_str(format_args!("write: {e:?}"))))?;
        let f: Function = fn_js
            .dyn_into()
            .map_err(|_| WriteError::Io("write не функция".to_string()))?;
        let promise_js = f
            .call1(&self.stream, &u8arr)
            .map_err(|e| WriteError::Io(format_str(format_args!("write: {e:?}"))))?;
        let promise: js_sys::Promise = promise_js
            .dyn_into()
            .map_err(|_| WriteError::Io("write: не Promise".to_string()))?;
        JsFuture::from(promise)
            .await
            .map_err(|e| WriteError::Io(format_str(format_args!("write await: {e:?}"))))?;
        self.writes += 1;
        self.bytes_written += data.len() as u64;
        return Ok(());
    }

    pub async fn seek(&mut self, offset: u64) -> Result<(), WriteError> {
        self.require_open()?;
        let fn_js = Reflect::get(&self.stream, &"seek".into())
            .map_err(|e| WriteError::Io(format_str(format_args!("seek: {e:?}"))))?;
        let f: Function = fn_js
            .dyn_into()
            .map_err(|_| WriteError::Io("seek не функция".to_string()))?;
        let promise_js = f
            .call1(&self.stream, &JsValue::from_f64(offset as f64))
            .map_err(|e| WriteError::Io(format_str(format_args!("seek: {e:?}"))))?;
        let promise: js_sys::Promise = promise_js
            .dyn_into()
            .map_err(|_| WriteError::Io("seek: не Promise".to_string()))?;
        JsFuture::from(promise)
            .await
            .map_err(|e| WriteError::Io(format_str(format_args!("seek await: {e:?}"))))?;
        return Ok(());
    }

    pub async fn truncate(&mut self, size: u64) -> Result<(), WriteError> {
        self.require_open()?;
        let fn_js = Reflect::get(&self.stream, &"truncate".into())
            .map_err(|e| WriteError::Io(format_str(format_args!("truncate: {e:?}"))))?;
        let f: Function = fn_js
            .dyn_into()
            .map_err(|_| WriteError::Io("truncate не функция".to_string()))?;
        let promise_js = f
            .call1(&self.stream, &JsValue::from_f64(size as f64))
            .map_err(|e| WriteError::Io(format_str(format_args!("truncate: {e:?}"))))?;
        let promise: js_sys::Promise = promise_js
            .dyn_into()
            .map_err(|_| WriteError::Io("truncate: не Promise".to_string()))?;
        JsFuture::from(promise)
            .await
            .map_err(|e| WriteError::Io(format_str(format_args!("truncate await: {e:?}"))))?;
        return Ok(());
    }

    pub async fn commit(&mut self) -> Result<CommitStats, WriteError> {
        if self.state == SessionState::Committed {
            return Ok(self.stats_snapshot());
        }
        if self.state != SessionState::Open {
            return Err(WriteError::SessionClosed);
        }
        let fn_js = Reflect::get(&self.stream, &"close".into())
            .map_err(|e| WriteError::Io(format_str(format_args!("close: {e:?}"))))?;
        let f: Function = fn_js
            .dyn_into()
            .map_err(|_| WriteError::Io("close не функция".to_string()))?;
        let promise_js = f
            .call0(&self.stream)
            .map_err(|e| WriteError::Io(format_str(format_args!("close: {e:?}"))))?;
        let promise: js_sys::Promise = promise_js
            .dyn_into()
            .map_err(|_| WriteError::Io("close: не Promise".to_string()))?;
        JsFuture::from(promise)
            .await
            .map_err(|e| WriteError::Io(format_str(format_args!("close await: {e:?}"))))?;
        self.state = SessionState::Committed;
        return Ok(self.stats_snapshot());
    }

    pub async fn abort(&mut self) -> Result<(), WriteError> {
        if self.state == SessionState::Aborted {
            return Ok(());
        }
        if self.state != SessionState::Open {
            return Err(WriteError::SessionClosed);
        }
        let fn_js = Reflect::get(&self.stream, &"abort".into())
            .map_err(|e| WriteError::Io(format_str(format_args!("abort: {e:?}"))))?;
        let f: Function = fn_js
            .dyn_into()
            .map_err(|_| WriteError::Io("abort не функция".to_string()))?;
        let promise_js = f
            .call0(&self.stream)
            .map_err(|e| WriteError::Io(format_str(format_args!("abort: {e:?}"))))?;
        let promise: js_sys::Promise = promise_js
            .dyn_into()
            .map_err(|_| WriteError::Io("abort: не Promise".to_string()))?;
        let _ = JsFuture::from(promise).await;
        self.state = SessionState::Aborted;
        return Ok(());
    }

    fn stats_snapshot(&self) -> CommitStats {
        let elapsed = js_sys::Date::now() - self.t0;
        return CommitStats {
            writes: self.writes,
            bytes_written: self.bytes_written,
            elapsed_ms: elapsed,
            keep_existing: self.keep_existing,
        };
    }
}

impl Drop for WriteSession {
    fn drop(&mut self) {
        if self.state != SessionState::Open {
            return;
        }
        if let Ok(fn_js) = Reflect::get(&self.stream, &"abort".into()) {
            if let Ok(f) = fn_js.dyn_into::<Function>() {
                let _ = f.call0(&self.stream);
            }
        }
        self.state = SessionState::Aborted;
    }
}

// ---------------------------------------------------------------------------
// Чтение после записи
// ---------------------------------------------------------------------------

pub async fn read_text_now(handle: &FileSystemFileHandle) -> Result<String, WriteError> {
    let file = crate::fsa_api_file::get_file(handle)
        .await
        .map_err(|e| WriteError::Io(format_str(format_args!("get_file: {e:?}"))))?;
    let promise: js_sys::Promise = file.text();
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| WriteError::Io(format_str(format_args!("text: {e:?}"))))?;
    return value
        .as_string()
        .ok_or_else(|| WriteError::Io("не строка".to_string()));
}

pub async fn read_bytes_now(handle: &FileSystemFileHandle) -> Result<Vec<u8>, WriteError> {
    let file = crate::fsa_api_file::get_file(handle)
        .await
        .map_err(|e| WriteError::Io(format_str(format_args!("get_file: {e:?}"))))?;
    let promise: js_sys::Promise = file.array_buffer();
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| WriteError::Io(format_str(format_args!("array_buffer: {e:?}"))))?;
    let buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| WriteError::Io("не ArrayBuffer".to_string()))?;
    return Ok(Uint8Array::new(&buffer).to_vec());
}

pub async fn read_size_now(handle: &FileSystemFileHandle) -> Result<u64, WriteError> {
    let file = crate::fsa_api_file::get_file(handle)
        .await
        .map_err(|e| WriteError::Io(format_str(format_args!("get_file: {e:?}"))))?;
    return Ok(file.size() as u64);
}

// ---------------------------------------------------------------------------
// Наивная перезапись — для контраста с seek
// ---------------------------------------------------------------------------

/// «Наивная» перезапись: прочитать весь файл, изменить, записать весь.
///
/// Используется в демо для контраста с `seek + write`: показывает,
/// сколько байт читается и пишется зря, если не использовать позиционирование.
pub async fn write_naive_rewrite(
    handle: &FileSystemFileHandle,
    offset: u64,
    new_bytes: &[u8],
) -> Result<NaiveStats, WriteError> {
    let t0 = js_sys::Date::now();

    // 1. Прочитать весь файл.
    let mut data = read_bytes_now(handle).await?;
    let bytes_read = data.len() as u64;

    // 2. Изменить в памяти.
    let start = offset as usize;
    if start + new_bytes.len() > data.len() {
        return Err(WriteError::Io(format_str(format_args!(
            "offset {} + {} байт выходит за пределы файла ({} байт)",
            offset, new_bytes.len(), data.len(),
        ))));
    }
    data[start..start + new_bytes.len()].copy_from_slice(new_bytes);

    // 3. Записать целиком.
    let mut session = WriteSession::open(handle, false).await?;
    session.write(&data).await?;
    session.commit().await?;

    let elapsed = js_sys::Date::now() - t0;
    return Ok(NaiveStats {
        bytes_read,
        bytes_written: data.len() as u64,
        elapsed_ms: elapsed,
    });
}

/// Строка «что сейчас в файле»: длина + первые 60 символов.
pub fn describe_content(text: &str) -> String {
    let chars: usize = text.chars().count();
    if chars == 0 {
        return "(файл пуст)".to_string();
    }
    let preview: String = if chars > 60 {
        let head: String = text.chars().take(60).collect();
        format_str(format_args!("{head}…"))
    } else {
        text.to_string()
    };
    let s: String = format_str(format_args!("{chars} симв.: {preview}"));
    return s;
}

/// Строка «N байт».
pub fn format_bytes(n: u64) -> String {
    if n < 1024 {
        return format_str(format_args!("{n} Б"));
    }
    if n < 1024 * 1024 {
        let k: f64 = n as f64 / 1024.0;
        return format_str(format_args!("{k:.1} КБ"));
    }
    let m: f64 = n as f64 / (1024.0 * 1024.0);
    return format_str(format_args!("{m:.2} МБ"));
}
```

---

## Как добавить в `lib.rs`

```rust
pub mod fsa_write_session;
pub mod fsa_write_ops;
```

---

## Тесты

### `tests/wasm_fsa_write.rs`

```rust
//! Тесты `fsa_write_*` — транзакция, seek, truncate, keepExistingData,
//! машина состояний, контраст «naive vs seek».
//!
//! Работаем с OPFS: не требует user gesture, доступен в headless Chrome.

mod opfs {
    use wasm_bindgen::prelude::*;
    use wasm_bindgen::JsCast;
    use wasm_bindgen_futures::JsFuture;
    use web_sys::FileSystemDirectoryHandle;

    pub async fn root() -> Result<FileSystemDirectoryHandle, JsValue> {
        let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
        let navigator_js = JsValue::from(window.navigator());
        let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
        if storage_js.is_undefined() {
            return Err(JsValue::from_str("navigator.storage недоступен"));
        }
        let get_dir_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str("getDirectory"))?;
        if get_dir_js.is_undefined() {
            return Err(JsValue::from_str("OPFS недоступен"));
        }
        let get_dir: js_sys::Function = get_dir_js
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
        let promise: js_sys::Promise = get_dir
            .call0(&storage_js)?
            .dyn_into()
            .map_err(|_| JsValue::from_str("getDirectory не Promise"))?;
        let value: JsValue = JsFuture::from(promise).await?;
        value
            .dyn_into::<FileSystemDirectoryHandle>()
            .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
    }
}

use hello_wasm02::fsa_write_ops::{
    describe_content, format_bytes, read_bytes_now, read_size_now, read_text_now,
    write_naive_rewrite, WriteSession,
};
use hello_wasm02::fsa_write_session::{SessionState, WriteError};
use wasm_bindgen_test::*;
use web_sys::FileSystemDirectoryHandle;

wasm_bindgen_test_configure!(run_in_browser);

async fn prepare_file(
    root: &FileSystemDirectoryHandle,
    name: &str,
    initial: &[u8],
) -> web_sys::FileSystemFileHandle {
    let _ = hello_wasm02::fsa_api_directory::remove_entry(root, name, true).await;
    let handle = hello_wasm02::fsa_api_directory::get_file_handle(root, name, true)
        .await
        .expect("create");
    let mut s = WriteSession::open(&handle, false).await.expect("open");
    if !initial.is_empty() {
        s.write(initial).await.expect("write");
    }
    s.commit().await.expect("commit");
    return handle;
}

// ---------------------------------------------------------------------------
// Базовое: write → commit → read
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn write_then_commit_persists() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_a.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    assert_eq!(s.state(), SessionState::Open);
    s.write(b"Hello, transaction!").await.expect("write");
    let stats = s.commit().await.expect("commit");
    assert_eq!(stats.writes, 1);
    assert_eq!(stats.bytes_written, 19);
    assert_eq!(s.state(), SessionState::Committed);

    let text = read_text_now(&handle).await.expect("read");
    assert_eq!(text, "Hello, transaction!");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_a.txt", true).await;
}

// ---------------------------------------------------------------------------
// Без commit файл не меняется
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn write_without_commit_does_not_persist() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_b.txt", b"original").await;

    {
        let mut s = WriteSession::open(&handle, false).await.expect("open");
        s.write(b"new stuff").await.expect("write");
        // НЕ коммитим — выходим из scope, Drop вызовет abort.
    }

    let text = read_text_now(&handle).await.expect("read");
    assert_eq!(text, "original", "без commit файл не должен меняться");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_b.txt", true).await;
}

// ---------------------------------------------------------------------------
// abort
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn abort_discards_changes() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_c.txt", b"old").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"new").await.expect("write");
    s.abort().await.expect("abort");
    assert_eq!(s.state(), SessionState::Aborted);

    let text = read_text_now(&handle).await.expect("read");
    assert_eq!(text, "old", "abort откатывает изменения");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_c.txt", true).await;
}

// ---------------------------------------------------------------------------
// seek
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn seek_patches_at_offset() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_d.txt", b"ABCDEFGHIJ").await;

    let mut s = WriteSession::open(&handle, true).await.expect("open");
    s.seek(5).await.expect("seek");
    s.write(b"XXX").await.expect("write");
    s.commit().await.expect("commit");

    let text = read_text_now(&handle).await.expect("read");
    // Исходно: A B C D E F G H I J (индексы 0..10).
    // seek(5) + write("XXX") заменяет байты 5,6,7: F,G,H → X,X,X.
    // Байты 8 (I) и 9 (J) не трогаются. Итог — ABCDEXXXIJ.
    assert_eq!(text, "ABCDEXXXIJ");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_d.txt", true).await;
}

// ---------------------------------------------------------------------------
// truncate
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn truncate_shrinks_file() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_e.txt", b"ABCDEFGHIJ").await;

    let mut s = WriteSession::open(&handle, true).await.expect("open");
    s.truncate(4).await.expect("truncate");
    s.commit().await.expect("commit");

    let bytes = read_bytes_now(&handle).await.expect("read");
    assert_eq!(bytes.len(), 4);
    assert_eq!(&bytes[..], b"ABCD");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_e.txt", true).await;
}

// ---------------------------------------------------------------------------
// keepExistingData
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn keep_existing_preserves_tail() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_f.txt", b"ABCDEFGHIJ").await;

    // keep = true: хвост сохраняется, первые 2 байта заменяются.
    let mut s = WriteSession::open(&handle, true).await.expect("open");
    s.write(b"XY").await.expect("write");
    s.commit().await.expect("commit");

    let text = read_text_now(&handle).await.expect("read");
    assert_eq!(text, "XYCDEFGHIJ");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_f.txt", true).await;
}

#[wasm_bindgen_test]
async fn without_keep_existing_truncates() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_g.txt", b"ABCDEFGHIJ").await;

    // keep = false: файл усечён при открытии, запись "XY" даёт длину 2.
    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"XY").await.expect("write");
    s.commit().await.expect("commit");

    let text = read_text_now(&handle).await.expect("read");
    assert_eq!(text, "XY", "без keep файл усечён при открытии");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_g.txt", true).await;
}

// ---------------------------------------------------------------------------
// Машина состояний
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn commit_is_idempotent() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_h.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"hi").await.expect("write");
    let a = s.commit().await.expect("commit 1");
    let b = s.commit().await.expect("commit 2");
    assert_eq!(a, b, "повторный commit возвращает те же stats");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_h.txt", true).await;
}

#[wasm_bindgen_test]
async fn write_after_commit_errors() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_i.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"x").await.expect("write");
    s.commit().await.expect("commit");
    let res = s.write(b"y").await;
    assert!(matches!(res, Err(WriteError::SessionClosed)));
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_i.txt", true).await;
}

#[wasm_bindgen_test]
async fn write_after_abort_errors() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_j.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"x").await.expect("write");
    s.abort().await.expect("abort");
    let res = s.write(b"y").await;
    assert!(matches!(res, Err(WriteError::SessionClosed)));
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_j.txt", true).await;
}

#[wasm_bindgen_test]
async fn abort_is_idempotent() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_k.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"x").await.expect("write");
    s.abort().await.expect("abort 1");
    s.abort().await.expect("abort 2");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_k.txt", true).await;
}

#[wasm_bindgen_test]
async fn commit_after_abort_errors() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_l.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"x").await.expect("write");
    s.abort().await.expect("abort");
    let res = s.commit().await;
    assert!(matches!(res, Err(WriteError::SessionClosed)));
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_l.txt", true).await;
}

// ---------------------------------------------------------------------------
// UTF-8 с эмодзи
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn utf8_emoji_roundtrip() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_m.txt", b"").await;

    let text = "Привет, 🚀 world!";
    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(text.as_bytes()).await.expect("write");
    s.commit().await.expect("commit");

    let back = read_text_now(&handle).await.expect("read");
    assert_eq!(back, text);
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_m.txt", true).await;
}

// ---------------------------------------------------------------------------
// Несколько write до commit
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn multiple_writes_before_commit() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_n.txt", b"").await;

    let mut s = WriteSession::open(&handle, false).await.expect("open");
    s.write(b"AAA").await.expect("w1");
    s.write(b"BBB").await.expect("w2");
    s.write(b"CCC").await.expect("w3");
    let stats = s.commit().await.expect("commit");
    assert_eq!(stats.writes, 3);
    assert_eq!(stats.bytes_written, 9);

    let text = read_text_now(&handle).await.expect("read");
    assert_eq!(text, "AAABBBCCC");
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_n.txt", true).await;
}

// ---------------------------------------------------------------------------
// Контраст: naive vs seek
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn naive_rewrite_reads_and_writes_all() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let data = vec![b'A'; 4096];
    let handle = prepare_file(&root, "w_o.bin", &data).await;

    let stats = write_naive_rewrite(&handle, 1000, b"XYZ").await.expect("naive");
    assert_eq!(stats.bytes_read, 4096, "naive читает весь файл");
    assert_eq!(stats.bytes_written, 4096, "naive пишет весь файл");

    let back = read_bytes_now(&handle).await.expect("read");
    assert_eq!(&back[1000..1003], b"XYZ");
    assert_eq!(back.len(), 4096);
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_o.bin", true).await;
}

#[wasm_bindgen_test]
async fn seek_patch_writes_only_3_bytes() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let data = vec![b'A'; 4096];
    let handle = prepare_file(&root, "w_p.bin", &data).await;

    let mut s = WriteSession::open(&handle, true).await.expect("open");
    s.seek(1000).await.expect("seek");
    s.write(b"XYZ").await.expect("write");
    let stats = s.commit().await.expect("commit");
    assert_eq!(stats.bytes_written, 3, "с seek пишем только 3 байта");

    let back = read_bytes_now(&handle).await.expect("read");
    assert_eq!(&back[1000..1003], b"XYZ");
    assert_eq!(back.len(), 4096);
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_p.bin", true).await;
}

// ---------------------------------------------------------------------------
// read_size_now
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn read_size_matches_content() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle = prepare_file(&root, "w_q.txt", b"12345").await;
    let size = read_size_now(&handle).await.expect("size");
    assert_eq!(size, 5);
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "w_q.txt", true).await;
}

// ---------------------------------------------------------------------------
// describe_content / format_bytes
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn describe_content_empty() {
    assert_eq!(describe_content(""), "(файл пуст)");
}

#[wasm_bindgen_test]
fn describe_content_short() {
    let s = describe_content("hello");
    assert!(s.contains("5 симв."));
    assert!(s.contains("hello"));
}

#[wasm_bindgen_test]
fn describe_content_truncates_long() {
    let text = "a".repeat(100);
    let s = describe_content(&text);
    assert!(s.contains("100 симв."));
    assert!(s.contains("…"));
}

#[wasm_bindgen_test]
fn format_bytes_units() {
    assert_eq!(format_bytes(0), "0 Б");
    assert_eq!(format_bytes(512), "512 Б");
    assert_eq!(format_bytes(2048), "2.0 КБ");
}
```

---

## Команды запуска

```bash
cargo check --target wasm32-unknown-unknown && wasm-pack test --headless --chrome --test wasm_fsa_write
npm start
```

Ожидаемый результат: **21 passed, 0 failed**.

---

## Демо-пример в `src/app.rs`

Четыре блока, каждый — один образовательный тезис. Работаем через OPFS — не требует user gesture.

```rust
//! Демо-секция 19.6: большие файлы, adaptive chunk, backpressure.
//!
//! Два независимых блока:
//! * A. Плавное копирование: yield 16 мс, live-скорость, SHA-256 проверка.
//! * B. Counterfactual: то же копирование без yield — UI зависает.
//!
//! Раскрывает через контраст:
//! * зачем нужен yield в длинных циклах;
//! * как память ограничивается размером чанка;
//! * как проверить, что копия действительно совпадает с источником.
//!
//! Размер источника задаётся в UI (1…500 МБ). Источник пишется чанками
//! по 64 КиБ, поэтому память не зависит от введённого размера. Под
//! полем ввода — live-подсказка: какой chunk и сколько итераций получит
//! `ChunkPlan` для текущего значения.
//!
//! Терминология UI. Везде вместо жаргонного «наивно» используем
//! описательные формулировки:
//! * «прочитать весь файл в память» — прямолинейный путь: получить
//!   весь ArrayBuffer и только потом писать;
//! * «читать и писать чанками» — реализованный streaming-путь:
//!   read chunk → write chunk → read next.
//!
//! Управление файлами OPFS. «Подготовить источник» сносит и предыдущие
//! приёмники, чтобы в хранилище не оставалось файлов от старых прогонов.
//! «Очистить файлы OPFS» — явное удаление всех трёх файлов и сброс UI.
//!
//! Контракт с `lib.rs`: `run() -> Result<(), JsValue>`.
//!
//! Правило проекта: внутри `Closure::new(move || …)` элементы клонируются
//! до `move`.
//!
//! Замечание по путям. Внутри `src/app.rs` запрещено обращаться к
//! собственному крейту по имени `hello_wasm02::…` — используем `crate::…`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    FileSystemDirectoryHandle, FileSystemFileHandle, HtmlButtonElement,
    HtmlElement, HtmlInputElement, HtmlProgressElement,
};

use crate::fsa_api_directory;
use crate::fsa_api_file;
use crate::fsa_choose_exec::Capabilities;
use crate::fsa_choose_options::format_str;
use crate::fsa_large_chunk::{self, ChunkPlan};
use crate::fsa_large_copy::{self, CopyOptions, CopyOutcome};
use crate::fsa_write_ops::WriteSession;

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

const SRC_NAME: &str = "demo-19-6-src.bin";
const DST_A_NAME: &str = "demo-19-6-dst-a.bin";
const DST_B_NAME: &str = "demo-19-6-dst-b.bin";

/// Дефолт и границы размера источника в мегабайтах.
const DEFAULT_SIZE_MB: u64 = 24;
const MIN_SIZE_MB: u64 = 1;
const MAX_SIZE_MB: u64 = 500;
const BYTES_PER_MB: u64 = 1024 * 1024;

/// Размер чанка при генерации источника (не путать с chunk_size плана —
/// это внутренний буфер записи, чтобы не держать весь файл в памяти).
const SOURCE_WRITE_CHUNK: usize = 64 * 1024;

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
            #fsa-large { font-family: system-ui, sans-serif; max-width: 1000px; }
            #fsa-large button { padding: 6px 12px; cursor: pointer; margin-right: 8px; margin-bottom: 6px; }
            #fsa-large button:disabled { opacity: 0.5; cursor: not-allowed; }
            #fsa-large button.cleanup {
                background: #fff4f4; border: 1px solid #e0b8b8; color: #7a2b2b;
            }
            #fsa-large button.cleanup:hover:not(:disabled) {
                background: #ffe6e6; border-color: #c98282;
            }
            #fsa-large .unsupported {
                margin: 12px 0 0 0; padding: 16px 20px;
                background: #fdecea; border-left: 5px solid #c0392b;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px; line-height: 1.7;
                color: #5a1a13;
            }
            #fsa-large .help {
                margin: 0 0 16px 0; padding: 10px 14px;
                background: #f9f9f9; border: 1px solid #e5e5e5;
                border-radius: 6px; font-size: 13px; line-height: 1.6;
                white-space: pre-line; color: #333;
            }
            #fsa-large .size-row {
                margin: 0 0 12px 0; padding: 10px 14px;
                background: #f5f9ff; border-left: 4px solid #4a7cff;
                border-radius: 4px;
                font-family: monospace; font-size: 13px;
                line-height: 1.6;
            }
            #fsa-large .size-row label {
                margin-right: 6px; font-weight: bold;
            }
            #fsa-large .size-row input[type="number"] {
                width: 90px; padding: 4px 6px; margin-right: 10px;
                font-family: monospace; font-size: 13px;
                border: 1px solid #b8c4d9; border-radius: 4px;
            }
            #fsa-large .size-hint {
                margin: 6px 0 0 0; padding: 6px 10px;
                background: #fff; border-radius: 4px;
                font-size: 12px; color: #333;
                border: 1px dashed #b8c4d9;
                white-space: pre-line;
            }
            #fsa-large .block {
                margin: 0 0 20px 0; padding: 14px 18px;
                border: 1px solid #ddd; border-radius: 8px;
                background: #fdfdfd;
            }
            #fsa-large .block h3 {
                margin: 0 0 8px 0; font-size: 15px; color: #222;
                border-bottom: 1px solid #eee; padding-bottom: 6px;
            }
            #fsa-large .block .purpose {
                font-size: 12px; color: #666; margin: 0 0 12px 0;
                white-space: pre-line; line-height: 1.5;
            }
            #fsa-large .info {
                margin: 0 0 12px 0; padding: 10px 14px;
                background: #eef6ee; border-left: 4px solid #4caf50;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px;
            }
            #fsa-large .counterfactual {
                margin: 0 0 12px 0; padding: 10px 14px;
                background: #fff9e6; border-left: 4px solid #e6a700;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px;
            }
            #fsa-large .status {
                margin: 10px 0; padding: 8px 12px;
                background: #f0f4ff; border-left: 4px solid #4a7cff;
                border-radius: 4px; white-space: pre-line; font-family: monospace;
            }
            #fsa-large .status.ok      { border-left-color: #27ae60; background: #e8f7ee; }
            #fsa-large .status.blocked { border-left-color: #e6a700; background: #fffbe0; }
            #fsa-large .status.failed  { border-left-color: #c0392b; background: #fdecea; }
            #fsa-large .result {
                background: #f6f6f6; padding: 10px 14px; border-radius: 4px;
                font-family: monospace; font-size: 12px;
                white-space: pre-wrap; margin: 8px 0;
            }
            #fsa-large progress { width: 320px; height: 14px; vertical-align: middle; }
            #fsa-large .speed {
                display: inline-block; margin-left: 8px;
                font-family: monospace; font-size: 13px;
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
// Состояние
// ---------------------------------------------------------------------------

struct DemoState {
    root: Option<FileSystemDirectoryHandle>,
    src: Option<FileSystemFileHandle>,
    dst_a: Option<FileSystemFileHandle>,
    dst_b: Option<FileSystemFileHandle>,
    cancel_a: Rc<Cell<bool>>,
    cancel_b: Rc<Cell<bool>>,
}

// ---------------------------------------------------------------------------
// Размер источника
// ---------------------------------------------------------------------------

/// Нормализует введённое значение: NaN/мусор → дефолт; вне границ → clamp.
fn parse_size_mb(raw: f64) -> u64 {
    if !raw.is_finite() {
        return DEFAULT_SIZE_MB;
    }
    if raw < MIN_SIZE_MB as f64 {
        return MIN_SIZE_MB;
    }
    if raw > MAX_SIZE_MB as f64 {
        return MAX_SIZE_MB;
    }
    return raw as u64;
}

/// Обновляет текст подсказки под полем ввода: какой chunk и сколько
/// итераций даст `ChunkPlan` для текущего значения, и как это отразится
/// на пиковой памяти при двух способах копирования.
fn update_size_hint(input: &HtmlInputElement, hint: &HtmlElement) {
    let size_mb: u64 = parse_size_mb(input.value_as_number());
    let bytes: u64 = size_mb * BYTES_PER_MB;
    let plan: ChunkPlan = ChunkPlan::for_total(bytes);
    let chunk_h: String = fsa_large_chunk::format_bytes(plan.chunk_size as u64);
    let whole_h: String = fsa_large_chunk::format_bytes(bytes);
    let text: String = format_str(format_args!(
        "→ при {size_mb} МБ: chunk {chunk_h} · ~{} итераций\n\
         Прочитать весь файл в память:  peak {whole_h}\n\
         Читать и писать чанками:       peak {chunk_h}",
        plan.expected_chunks,
    ));
    hint.set_text_content(Some(&text));
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
    section.set_id("fsa-large");

    let h2 = doc.create_element("h2")?;
    h2.set_text_content(Some("19.6 — Большие файлы: chunk, backpressure, yield"));
    section.append_child(&h2)?;

    let caps = Capabilities::detect();
    if !caps.directory {
        let unsupported: HtmlElement = doc
            .create_element("div")?
            .dyn_into::<HtmlElement>()
            .map_err(|_| JsValue::from_str("не HtmlElement"))?;
        unsupported.set_attribute("class", "unsupported")?;
        unsupported.set_text_content(Some(&unsupported_text()));
        section.append_child(&unsupported)?;
        body.append_child(&section)?;
        return Ok(());
    }

    let help: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    help.set_attribute("class", "help")?;
    help.set_text_content(Some(&help_text()));
    section.append_child(&help)?;

    // Info: OPFS + план.
    let info: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    info.set_attribute("class", "info")?;
    info.set_text_content(Some("Инициализация…"));
    section.append_child(&info)?;

    // Строка ввода размера источника.
    let size_row: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    size_row.set_attribute("class", "size-row")?;

    let size_label: HtmlElement = doc
        .create_element("label")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    size_label.set_text_content(Some("Размер источника (МБ):"));
    size_label.set_attribute("for", "lg-size")?;
    size_row.append_child(&size_label)?;

    let size_input: HtmlInputElement = doc
        .create_element("input")?
        .dyn_into::<HtmlInputElement>()
        .map_err(|_| JsValue::from_str("не HtmlInputElement"))?;
    size_input.set_id("lg-size");
    size_input.set_type("number");
    size_input.set_value(&DEFAULT_SIZE_MB.to_string());
    size_input.set_min(&MIN_SIZE_MB.to_string());
    size_input.set_max(&MAX_SIZE_MB.to_string());
    size_input.set_step("1");
    size_input.set_disabled(true);
    size_row.append_child(&size_input)?;

    let prepare_btn = make_button(&doc, "Подготовить источник", "lg-prepare")?;
    prepare_btn.set_disabled(true);
    size_row.append_child(&prepare_btn)?;

    let cleanup_btn = make_button(&doc, "Очистить файлы OPFS", "lg-cleanup")?;
    cleanup_btn.set_disabled(true);
    let _ = cleanup_btn.set_attribute("class", "cleanup");
    size_row.append_child(&cleanup_btn)?;

    // Live-подсказка под полем ввода.
    let size_hint: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    size_hint.set_attribute("class", "size-hint")?;
    size_row.append_child(&size_hint)?;

    section.append_child(&size_row)?;

    // Counterfactual памяти.
    let counterfactual: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    counterfactual.set_attribute("class", "counterfactual")?;
    counterfactual.set_text_content(Some("Counterfactual появится после подготовки источника."));
    section.append_child(&counterfactual)?;

    // Blocks.
    let a = build_block_a(&doc, &section)?;
    let b = build_block_b(&doc, &section)?;

    body.append_child(&section)?;

    // Wire live-hint: показываем план для текущего значения.
    update_size_hint(&size_input, &size_hint);
    wire_size_hint(&size_input, &size_hint);

    let state = Rc::new(RefCell::new(DemoState {
        root: None,
        src: None,
        dst_a: None,
        dst_b: None,
        cancel_a: Rc::new(Cell::new(false)),
        cancel_b: Rc::new(Cell::new(false)),
    }));

    // Init OPFS.
    let info_c = info.clone();
    let state_c = state.clone();
    let prepare_btn_c = prepare_btn.clone();
    let cleanup_btn_c = cleanup_btn.clone();
    let size_input_c = size_input.clone();
    spawn_local(async move {
        match init_opfs_root().await {
            Ok(root) => {
                let msg: String = format_str(format_args!(
                    "OPFS готов. Задайте размер источника ({}…{} МБ) и нажмите «Подготовить».",
                    MIN_SIZE_MB, MAX_SIZE_MB,
                ));
                info_c.set_text_content(Some(&msg));
                state_c.borrow_mut().root = Some(root);
                prepare_btn_c.set_disabled(false);
                cleanup_btn_c.set_disabled(false);
                size_input_c.set_disabled(false);
            }
            Err(e) => {
                let msg: String = format_str(format_args!(
                    "✗ Не удалось инициализировать OPFS: {e:?}",
                ));
                info_c.set_text_content(Some(&msg));
            }
        }
    });

    wire_prepare(
        &prepare_btn,
        &size_input,
        &a,
        &b,
        &counterfactual,
        &info,
        state.clone(),
    );
    wire_block_a(&a, state.clone());
    wire_block_b(&b, state.clone());
    wire_cleanup(&cleanup_btn, &a, &b, &counterfactual, &info, state);

    Ok(())
}

fn wire_size_hint(size_input: &HtmlInputElement, hint: &HtmlElement) {
    let input_c = size_input.clone();
    let hint_c = hint.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        update_size_hint(&input_c, &hint_c);
    });
    size_input.set_oninput(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

async fn init_opfs_root() -> Result<FileSystemDirectoryHandle, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator_js = JsValue::from(window.navigator());
    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }
    let get_dir_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str("getDirectory"))?;
    if get_dir_js.is_undefined() {
        return Err(JsValue::from_str("OPFS недоступен"));
    }
    let get_dir: js_sys::Function = get_dir_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
    let promise: js_sys::Promise = get_dir
        .call0(&storage_js)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}

// ---------------------------------------------------------------------------
// Генерация источника чанками (память O(chunk), не O(file))
// ---------------------------------------------------------------------------

async fn write_source_chunked(
    src: &FileSystemFileHandle,
    src_bytes: u64,
) -> Result<(), String> {
    let mut s = WriteSession::open(src, false).await.map_err(|e| e.explain())?;

    let mut buf: Vec<u8> = vec![0u8; SOURCE_WRITE_CHUNK];
    let mut written: u64 = 0;
    while written < src_bytes {
        let n: usize = std::cmp::min(SOURCE_WRITE_CHUNK as u64, src_bytes - written) as usize;
        for i in 0..n {
            let idx: u64 = written + i as u64;
            buf[i] = (idx % 251) as u8;
        }
        s.write(&buf[..n]).await.map_err(|e| e.explain())?;
        written += n as u64;
    }
    s.commit().await.map_err(|e| e.explain())?;
    return Ok(());
}

// ---------------------------------------------------------------------------
// Общая очистка файлов OPFS
// ---------------------------------------------------------------------------

/// Сносит все три файла демо в OPFS. Отсутствие любого из них —
/// не ошибка. Возвращает () — best-effort.
async fn remove_demo_files(root: &FileSystemDirectoryHandle) {
    let _ = fsa_api_directory::remove_entry(root, SRC_NAME, true).await;
    let _ = fsa_api_directory::remove_entry(root, DST_A_NAME, true).await;
    let _ = fsa_api_directory::remove_entry(root, DST_B_NAME, true).await;
}

/// Сбрасывает UI блока A в исходное состояние.
fn reset_block_a_ui(ui: &BlockAUi) {
    ui.copy_btn.set_disabled(true);
    ui.cancel_btn.set_disabled(true);
    ui.verify_btn.set_disabled(true);
    ui.progress.set_value(0.0);
    ui.progress_label.set_text_content(Some("0%"));
    ui.speed_el.set_text_content(Some("— МБ/с"));
    set_status(&ui.status, "", "Готово.");
    ui.metrics.set_text_content(Some("(метрики появятся здесь)"));
}

/// Сбрасывает UI блока B в исходное состояние.
fn reset_block_b_ui(ui: &BlockBUi) {
    ui.copy_btn.set_disabled(true);
    ui.cancel_btn.set_disabled(true);
    ui.progress.set_value(0.0);
    ui.progress_label.set_text_content(Some("0%"));
    ui.speed_el.set_text_content(Some("— МБ/с"));
    set_status(&ui.status, "", "Готово.");
    ui.metrics.set_text_content(Some("(метрики появятся здесь)"));
}

// ---------------------------------------------------------------------------
// Блок A — копирование с yield
// ---------------------------------------------------------------------------

struct BlockAUi {
    copy_btn: HtmlButtonElement,
    cancel_btn: HtmlButtonElement,
    verify_btn: HtmlButtonElement,
    progress: HtmlProgressElement,
    progress_label: HtmlElement,
    speed_el: HtmlElement,
    status: HtmlElement,
    metrics: HtmlElement,
}

fn build_block_a(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<BlockAUi, JsValue> {
    let block = make_block(doc, "A. Копирование с yield 16 мс")?;
    let purpose = make_purpose(
        doc,
        "read чанк → write чанк → read следующий. Write блокирует цикл, \
         пока диск не готов — это backpressure. Yield раз в 16 мс даёт \
         браузеру перерисовать UI: прогресс и скорость обновляются плавно.",
    )?;
    block.append_child(&purpose)?;

    let controls = doc.create_element("div")?;
    let copy_btn = make_button(doc, "Копировать", "lg-a-copy")?;
    let cancel_btn = make_button(doc, "Отмена", "lg-a-cancel")?;
    let verify_btn = make_button(doc, "Проверить SHA-256", "lg-a-verify")?;
    copy_btn.set_disabled(true);
    cancel_btn.set_disabled(true);
    verify_btn.set_disabled(true);
    controls.append_child(&copy_btn)?;
    controls.append_child(&cancel_btn)?;
    controls.append_child(&verify_btn)?;
    block.append_child(&controls)?;

    let progress_row = doc.create_element("div")?;
    let progress: HtmlProgressElement = doc
        .create_element("progress")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlProgressElement"))?;
    progress.set_max(1.0);
    progress.set_value(0.0);
    let progress_label: HtmlElement = doc
        .create_element("span")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    progress_label.set_text_content(Some("0%"));
    let speed_el: HtmlElement = doc
        .create_element("span")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    speed_el.set_attribute("class", "speed")?;
    speed_el.set_text_content(Some("— МБ/с"));
    progress_row.append_child(&progress)?;
    progress_row.append_child(&progress_label)?;
    progress_row.append_child(&speed_el)?;
    block.append_child(&progress_row)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    let metrics = make_result(doc)?;
    block.append_child(&metrics)?;

    parent.append_child(&block)?;
    Ok(BlockAUi { copy_btn, cancel_btn, verify_btn, progress, progress_label, speed_el, status, metrics })
}

// ---------------------------------------------------------------------------
// Блок B — counterfactual без yield
// ---------------------------------------------------------------------------

struct BlockBUi {
    copy_btn: HtmlButtonElement,
    cancel_btn: HtmlButtonElement,
    progress: HtmlProgressElement,
    progress_label: HtmlElement,
    speed_el: HtmlElement,
    status: HtmlElement,
    metrics: HtmlElement,
}

fn build_block_b(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<BlockBUi, JsValue> {
    let block = make_block(doc, "B. Counterfactual: копирование без yield")?;
    let purpose = make_purpose(
        doc,
        "То же копирование, но без TimeoutFuture. Цикл целиком синхронный: \
         пока он крутится, браузер не успевает перерисовать страницу. \
         Прогресс обновляется ровно один раз — в самом конце.",
    )?;
    block.append_child(&purpose)?;

    let controls = doc.create_element("div")?;
    let copy_btn = make_button(doc, "Копировать без yield", "lg-b-copy")?;
    let cancel_btn = make_button(doc, "Отмена", "lg-b-cancel")?;
    copy_btn.set_disabled(true);
    cancel_btn.set_disabled(true);
    controls.append_child(&copy_btn)?;
    controls.append_child(&cancel_btn)?;
    block.append_child(&controls)?;

    let progress_row = doc.create_element("div")?;
    let progress: HtmlProgressElement = doc
        .create_element("progress")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlProgressElement"))?;
    progress.set_max(1.0);
    progress.set_value(0.0);
    let progress_label: HtmlElement = doc
        .create_element("span")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    progress_label.set_text_content(Some("0%"));
    let speed_el: HtmlElement = doc
        .create_element("span")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    speed_el.set_attribute("class", "speed")?;
    speed_el.set_text_content(Some("— МБ/с"));
    progress_row.append_child(&progress)?;
    progress_row.append_child(&progress_label)?;
    progress_row.append_child(&speed_el)?;
    block.append_child(&progress_row)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    let metrics = make_result(doc)?;
    block.append_child(&metrics)?;

    parent.append_child(&block)?;
    Ok(BlockBUi { copy_btn, cancel_btn, progress, progress_label, speed_el, status, metrics })
}

// ---------------------------------------------------------------------------
// Wire
// ---------------------------------------------------------------------------

fn wire_prepare(
    prepare_btn: &HtmlButtonElement,
    size_input: &HtmlInputElement,
    a: &BlockAUi,
    b: &BlockBUi,
    counterfactual: &HtmlElement,
    info: &HtmlElement,
    state: Rc<RefCell<DemoState>>,
) {
    let a_copy = a.copy_btn.clone();
    let a_verify = a.verify_btn.clone();
    let b_copy = b.copy_btn.clone();
    let cf = counterfactual.clone();
    let info_c = info.clone();
    let size_input_c = size_input.clone();
    let state_c = state.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let a_copy = a_copy.clone();
        let a_verify = a_verify.clone();
        let b_copy = b_copy.clone();
        let cf = cf.clone();
        let info_el = info_c.clone();
        let size_input_el = size_input_c.clone();
        let state = state_c.clone();
        spawn_local(async move {
            let root = match state.try_borrow().ok().and_then(|s| s.root.clone()) {
                Some(r) => r,
                None => return,
            };

            let size_mb: u64 = parse_size_mb(size_input_el.value_as_number());
            // Нормализуем поле, чтобы пользователь видел, что реально применилось.
            let _ = size_input_el.set_value(&size_mb.to_string());
            let src_bytes: u64 = size_mb * BYTES_PER_MB;

            let msg_create: String = format_str(format_args!(
                "Создаю источник: {} МБ ({} байт)…",
                size_mb, src_bytes,
            ));
            info_el.set_text_content(Some(&msg_create));

            // Пересоздаём источник. Сносим и предыдущие приёмники,
            // чтобы в OPFS не оставалось файлов от старых прогонов:
            // иначе после смены размера в хранилище висит старый
            // dst_a (например, 24 МБ) и попадает в storage.estimate().
            remove_demo_files(&root).await;

            let src = match fsa_api_directory::get_file_handle(&root, SRC_NAME, true).await {
                Ok(h) => h,
                Err(e) => {
                    let msg: String = format_str(format_args!("✗ get_file_handle: {e:?}"));
                    info_el.set_text_content(Some(&msg));
                    return;
                }
            };

            if let Err(e) = write_source_chunked(&src, src_bytes).await {
                info_el.set_text_content(Some(&e));
                return;
            }

            // План для текущего размера. Не вызываем plan.describe(),
            // чтобы в UI не просачивалось слово «naive» — собираем
            // сообщение из тех же полей явно и описательно.
            let plan = ChunkPlan::for_total(src_bytes);
            let chunk_h: String = fsa_large_chunk::format_bytes(plan.chunk_size as u64);
            let whole_h: String = fsa_large_chunk::format_bytes(src_bytes);
            let info_text: String = format_str(format_args!(
                "Источник готов: {src_bytes} байт ({size_mb} МБ).\n\
                 План: {whole_h} · chunk {chunk_h} · ~{} итераций\n\
                 Пик памяти при чтении и записи чанками: {chunk_h}",
                plan.expected_chunks,
            ));
            info_el.set_text_content(Some(&info_text));

            // Counterfactual — акцентируем разницу в памяти.
            let streaming_h: String = fsa_large_chunk::format_bytes(plan.chunk_size as u64);
            let saved_h: String = fsa_large_chunk::format_bytes(plan.memory_saved());
            let cf_text: String = format_str(format_args!(
                "Counterfactual (для {size_mb} МБ):\n\
                 • Прочитать весь файл в память: peak {whole_h} RAM\n\
                 • Читать и писать чанками:      peak {streaming_h} RAM\n\
                 • Экономия памяти:              {saved_h}\n\
                 Попробуйте задать 500 МБ: прочитать такой файл целиком \
                 уже не получится — а чанковое чтение справится.",
            ));
            cf.set_text_content(Some(&cf_text));

            // Сбрасываем устаревшие приёмники и разрешения кнопок.
            {
                let mut s = state.borrow_mut();
                s.src = Some(src);
                s.dst_a = None;
                s.dst_b = None;
            }
            a_copy.set_disabled(false);
            a_verify.set_disabled(true);
            b_copy.set_disabled(false);
        });
    });
    prepare_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_block_a(ui: &BlockAUi, state: Rc<RefCell<DemoState>>) {
    // Copy.
    {
        let status_c = ui.status.clone();
        let metrics_c = ui.metrics.clone();
        let progress_c = ui.progress.clone();
        let progress_label_c = ui.progress_label.clone();
        let speed_c = ui.speed_el.clone();
        let cancel_c = ui.cancel_btn.clone();
        let copy_c = ui.copy_btn.clone();
        let verify_c = ui.verify_btn.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let status_el = status_c.clone();
            let metrics_el = metrics_c.clone();
            let progress_el = progress_c.clone();
            let progress_label_el = progress_label_c.clone();
            let speed_el = speed_c.clone();
            let cancel_el = cancel_c.clone();
            let copy_el = copy_c.clone();
            let verify_el = verify_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                let (root, src) = match state.try_borrow().ok().and_then(|s| {
                    s.root.clone().zip(s.src.clone())
                }) {
                    Some(p) => p,
                    None => {
                        set_status(&status_el, "blocked", "Сначала подготовьте источник.");
                        return;
                    }
                };
                copy_el.set_disabled(true);
                verify_el.set_disabled(true);
                cancel_el.set_disabled(false);
                progress_el.set_value(0.0);
                progress_label_el.set_text_content(Some("0%"));
                speed_el.set_text_content(Some("— МБ/с"));
                set_status(&status_el, "", "Копирую (yield 16 мс)…");

                let _ = fsa_api_directory::remove_entry(&root, DST_A_NAME, true).await;
                let dst = match fsa_api_directory::get_file_handle(&root, DST_A_NAME, true).await {
                    Ok(h) => h,
                    Err(e) => {
                        let msg: String = format_str(format_args!("✗ dst: {e:?}"));
                        set_status(&status_el, "failed", &msg);
                        copy_el.set_disabled(false);
                        cancel_el.set_disabled(true);
                        return;
                    }
                };
                let src_file = match fsa_api_file::get_file(&src).await {
                    Ok(f) => f,
                    Err(e) => {
                        let msg: String = format_str(format_args!("✗ src: {e:?}"));
                        set_status(&status_el, "failed", &msg);
                        copy_el.set_disabled(false);
                        cancel_el.set_disabled(true);
                        return;
                    }
                };

                let cancel_flag = state.borrow().cancel_a.clone();
                cancel_flag.set(false);

                let p_el = progress_el.clone();
                let pl_el = progress_label_el.clone();
                let sp_el = speed_el.clone();
                let on_progress = move |frac: f64, speed: f64| {
                    p_el.set_value(frac);
                    pl_el.set_text_content(Some(&format_str(format_args!("{:.0}%", frac * 100.0))));
                    sp_el.set_text_content(Some(&format_str(format_args!("{:.1} МБ/с", speed))));
                };
                let flag = cancel_flag.clone();
                let outcome = fsa_large_copy::copy_stream(
                    &src_file, &dst, on_progress, move || flag.get(),
                ).await;

                let (stats, class): (fsa_large_chunk::CopyStats, &str) = match outcome {
                    Ok(CopyOutcome::Done(s)) => (s, "ok"),
                    Ok(CopyOutcome::Cancelled(s)) => (s, "blocked"),
                    Err(e) => {
                        set_status(&status_el, "failed", &e);
                        copy_el.set_disabled(false);
                        cancel_el.set_disabled(true);
                        return;
                    }
                };

                let expected: u64 = src_file.size() as u64;
                let text = fsa_large_chunk::format_stats(&stats, expected);
                metrics_el.set_text_content(Some(&text));
                set_status(&status_el, class, if stats.cancelled { "Отменено." } else { "Готово." });

                state.borrow_mut().dst_a = Some(dst);
                copy_el.set_disabled(false);
                cancel_el.set_disabled(true);
                if !stats.cancelled {
                    verify_el.set_disabled(false);
                }
            });
        });
        ui.copy_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // Cancel.
    {
        let status_c = ui.status.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let flag = state_c.borrow().cancel_a.clone();
            flag.set(true);
            status_c.set_text_content(Some("Запрошена отмена…"));
        });
        ui.cancel_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // Verify.
    {
        let status_c = ui.status.clone();
        let metrics_c = ui.metrics.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let status_el = status_c.clone();
            let metrics_el = metrics_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                let (src, dst) = match state.try_borrow().ok().and_then(|s| {
                    s.src.clone().zip(s.dst_a.clone())
                }) {
                    Some(p) => p,
                    None => {
                        set_status(&status_el, "blocked", "Сначала скопируйте.");
                        return;
                    }
                };
                set_status(&status_el, "", "Считаю SHA-256 обоих файлов…");
                let src_hash = match fsa_large_copy::sha256_file(&src).await {
                    Ok(h) => h,
                    Err(e) => { set_status(&status_el, "failed", &e); return; }
                };
                let dst_hash = match fsa_large_copy::sha256_file(&dst).await {
                    Ok(h) => h,
                    Err(e) => { set_status(&status_el, "failed", &e); return; }
                };
                let verdict = fsa_large_copy::format_hash_verdict(&src_hash, &dst_hash);
                metrics_el.set_text_content(Some(&verdict));
                let class: &str = if src_hash == dst_hash { "ok" } else { "failed" };
                set_status(&status_el, class, "Проверка завершена.");
            });
        });
        ui.verify_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }
}

fn wire_block_b(ui: &BlockBUi, state: Rc<RefCell<DemoState>>) {
    // Copy.
    {
        let status_c = ui.status.clone();
        let metrics_c = ui.metrics.clone();
        let progress_c = ui.progress.clone();
        let progress_label_c = ui.progress_label.clone();
        let speed_c = ui.speed_el.clone();
        let cancel_c = ui.cancel_btn.clone();
        let copy_c = ui.copy_btn.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let status_el = status_c.clone();
            let metrics_el = metrics_c.clone();
            let progress_el = progress_c.clone();
            let progress_label_el = progress_label_c.clone();
            let speed_el = speed_c.clone();
            let cancel_el = cancel_c.clone();
            let copy_el = copy_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                let (root, src) = match state.try_borrow().ok().and_then(|s| {
                    s.root.clone().zip(s.src.clone())
                }) {
                    Some(p) => p,
                    None => {
                        set_status(&status_el, "blocked", "Сначала подготовьте источник.");
                        return;
                    }
                };
                copy_el.set_disabled(true);
                cancel_el.set_disabled(false);
                progress_el.set_value(0.0);
                progress_label_el.set_text_content(Some("0%"));
                speed_el.set_text_content(Some("— МБ/с"));
                set_status(&status_el, "blocked",
                    "Копирую без yield — UI замрёт до конца копирования.");

                let _ = fsa_api_directory::remove_entry(&root, DST_B_NAME, true).await;
                let dst = match fsa_api_directory::get_file_handle(&root, DST_B_NAME, true).await {
                    Ok(h) => h,
                    Err(e) => {
                        let msg: String = format_str(format_args!("✗ dst: {e:?}"));
                        set_status(&status_el, "failed", &msg);
                        copy_el.set_disabled(false);
                        cancel_el.set_disabled(true);
                        return;
                    }
                };
                let src_file = match fsa_api_file::get_file(&src).await {
                    Ok(f) => f,
                    Err(e) => {
                        let msg: String = format_str(format_args!("✗ src: {e:?}"));
                        set_status(&status_el, "failed", &msg);
                        copy_el.set_disabled(false);
                        cancel_el.set_disabled(true);
                        return;
                    }
                };

                let cancel_flag = state.borrow().cancel_b.clone();
                cancel_flag.set(false);

                let p_el = progress_el.clone();
                let pl_el = progress_label_el.clone();
                let sp_el = speed_el.clone();
                let on_progress = move |frac: f64, speed: f64| {
                    p_el.set_value(frac);
                    pl_el.set_text_content(Some(&format_str(format_args!("{:.0}%", frac * 100.0))));
                    sp_el.set_text_content(Some(&format_str(format_args!("{:.1} МБ/с", speed))));
                };
                let flag = cancel_flag.clone();
                let outcome = fsa_large_copy::copy_stream_with_options(
                    &src_file, &dst, CopyOptions::without_yield(),
                    on_progress, move || flag.get(),
                ).await;

                let (stats, class): (fsa_large_chunk::CopyStats, &str) = match outcome {
                    Ok(CopyOutcome::Done(s)) => (s, "ok"),
                    Ok(CopyOutcome::Cancelled(s)) => (s, "blocked"),
                    Err(e) => {
                        set_status(&status_el, "failed", &e);
                        copy_el.set_disabled(false);
                        cancel_el.set_disabled(true);
                        return;
                    }
                };

                let expected: u64 = src_file.size() as u64;
                let mut text = fsa_large_chunk::format_stats(&stats, expected);
                if stats.progress_calls == 1 {
                    let note: String = format_str(format_args!(
                        "\n⚠ on_progress вызвался {} раз — UI не обновлялся во время копирования.",
                        stats.progress_calls,
                    ));
                    text.push_str(&note);
                }
                metrics_el.set_text_content(Some(&text));
                set_status(&status_el, class, if stats.cancelled { "Отменено." } else { "Готово." });

                state.borrow_mut().dst_b = Some(dst);
                copy_el.set_disabled(false);
                cancel_el.set_disabled(true);
            });
        });
        ui.copy_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // Cancel.
    {
        let state_c = state.clone();
        let status_c = ui.status.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let flag = state_c.borrow().cancel_b.clone();
            flag.set(true);
            status_c.set_text_content(Some("Отмена не сработает: цикл без yield."));
        });
        ui.cancel_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }
}

/// «Очистить файлы OPFS»: сносит все три файла демо, сбрасывает
/// состояние и возвращает UI в исходный вид. Best-effort: отсутствие
/// файлов не считается ошибкой.
fn wire_cleanup(
    cleanup_btn: &HtmlButtonElement,
    a: &BlockAUi,
    b: &BlockBUi,
    counterfactual: &HtmlElement,
    info: &HtmlElement,
    state: Rc<RefCell<DemoState>>,
) {
    // UI-хендлы блоков.
    let a_copy = a.copy_btn.clone();
    let a_cancel = a.cancel_btn.clone();
    let a_verify = a.verify_btn.clone();
    let a_progress = a.progress.clone();
    let a_progress_label = a.progress_label.clone();
    let a_speed = a.speed_el.clone();
    let a_status = a.status.clone();
    let a_metrics = a.metrics.clone();

    let b_copy = b.copy_btn.clone();
    let b_cancel = b.cancel_btn.clone();
    let b_progress = b.progress.clone();
    let b_progress_label = b.progress_label.clone();
    let b_speed = b.speed_el.clone();
    let b_status = b.status.clone();
    let b_metrics = b.metrics.clone();

    let cf = counterfactual.clone();
    let info_c = info.clone();
    let state_c = state.clone();

    let cb = Closure::<dyn FnMut()>::new(move || {
        let a_copy = a_copy.clone();
        let a_cancel = a_cancel.clone();
        let a_verify = a_verify.clone();
        let a_progress = a_progress.clone();
        let a_progress_label = a_progress_label.clone();
        let a_speed = a_speed.clone();
        let a_status = a_status.clone();
        let a_metrics = a_metrics.clone();

        let b_copy = b_copy.clone();
        let b_cancel = b_cancel.clone();
        let b_progress = b_progress.clone();
        let b_progress_label = b_progress_label.clone();
        let b_speed = b_speed.clone();
        let b_status = b_status.clone();
        let b_metrics = b_metrics.clone();

        let cf = cf.clone();
        let info_el = info_c.clone();
        let state = state_c.clone();

        spawn_local(async move {
            let root = match state.try_borrow().ok().and_then(|s| s.root.clone()) {
                Some(r) => r,
                None => {
                    info_el.set_text_content(Some("OPFS не инициализирован."));
                    return;
                }
            };

            info_el.set_text_content(Some("Удаляю файлы OPFS…"));
            remove_demo_files(&root).await;

            // Сбрасываем ссылки в состоянии.
            {
                let mut s = state.borrow_mut();
                s.src = None;
                s.dst_a = None;
                s.dst_b = None;
            }

            // Сбрасываем UI блока A.
            reset_block_a_ui(&BlockAUi {
                copy_btn: a_copy,
                cancel_btn: a_cancel,
                verify_btn: a_verify,
                progress: a_progress,
                progress_label: a_progress_label,
                speed_el: a_speed,
                status: a_status,
                metrics: a_metrics,
            });

            // Сбрасываем UI блока B.
            reset_block_b_ui(&BlockBUi {
                copy_btn: b_copy,
                cancel_btn: b_cancel,
                progress: b_progress,
                progress_label: b_progress_label,
                speed_el: b_speed,
                status: b_status,
                metrics: b_metrics,
            });

            cf.set_text_content(Some(
                "Counterfactual появится после подготовки источника.",
            ));
            info_el.set_text_content(Some(
                "OPFS очищен. Задайте размер и нажмите «Подготовить источник».",
            ));
        });
    });

    cleanup_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

// ---------------------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------------------

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

fn make_block(doc: &web_sys::Document, title: &str) -> Result<HtmlElement, JsValue> {
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

fn make_purpose(doc: &web_sys::Document, text: &str) -> Result<HtmlElement, JsValue> {
    let p: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    p.set_attribute("class", "purpose")?;
    p.set_text_content(Some(text));
    Ok(p)
}

fn make_status(doc: &web_sys::Document) -> Result<HtmlElement, JsValue> {
    let s: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    s.set_attribute("class", "status")?;
    s.set_text_content(Some("Готово."));
    Ok(s)
}

fn make_result(doc: &web_sys::Document) -> Result<HtmlElement, JsValue> {
    let r: HtmlElement = doc
        .create_element("pre")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    r.set_attribute("class", "result")?;
    r.set_text_content(Some("(метрики появятся здесь)"));
    Ok(r)
}

fn set_status(target: &HtmlElement, class: &str, text: &str) {
    let cls: String = if class.is_empty() {
        "status".to_string()
    } else {
        format_str(format_args!("status {class}"))
    };
    let _ = target.set_attribute("class", &cls);
    target.set_text_content(Some(text));
}

// ---------------------------------------------------------------------------
// Тексты
// ---------------------------------------------------------------------------

fn unsupported_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(6);
    lines.push("✗ OPFS недоступен — эта подглава требует Chrome/Edge (86+).".to_string());
    lines.push(String::new());
    lines.push("Откройте демо на HTTPS или localhost в Chrome/Edge.".to_string());
    return lines.join("\n");
}

fn help_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(22);
    lines.push("Как работать с демо:".to_string());
    lines.push(String::new());
    lines.push(
        "0.  Задайте размер источника в поле «Размер источника (МБ)» — \
         от 1 до 500. Под полем сразу видно, какой chunk и сколько \
         итераций даст ChunkPlan для этого размера, и как это \
         отразится на пиковой памяти при двух способах копирования."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "1.  Нажмите «Подготовить источник». Демо создаст файл в OPFS \
         заданного размера (пишется чанками по 64 КБ — вся память \
         ограничена буфером, а не размером файла). Заодно сносятся \
         приёмники от предыдущих прогонов, чтобы в OPFS не копились \
         файлы старых размеров."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "A.  «Копировать» — читает чанк, пишет чанк, yield 16 мс. \
         Прогресс и скорость обновляются плавно. Метрики покажут, \
         сколько yield'ов и сколько обновлений on_progress было. \
         После копирования — «Проверить SHA-256»: хеши источника \
         и приёмника совпадают."
            .to_string(),
    );
    lines.push(
        "B.  «Копировать без yield» — то же копирование, но без \
         TimeoutFuture. UI замирает до конца, прогресс обновляется \
         один раз в самом конце. Сравните метрики: progress_calls=1 \
         вместо десятков."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "«Очистить файлы OPFS» — удаляет все три файла демо \
         (источник и оба приёмника) и сбрасывает интерфейс. \
         Полезно, если после экспериментов с большими размерами \
         нужно освободить место: в панели DevTools → Application → \
         Storage → Usage видно, сколько занимает OPFS."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "Два способа скопировать файл — термины, которые \
         используются в панели Counterfactual:".to_string(),
    );
    lines.push(
        "•  Прочитать весь файл в память — прямолинейный путь: \
         сначала получить все байты файла, потом записать их \
         в приёмник. Пик памяти равен размеру файла."
            .to_string(),
    );
    lines.push(
        "•  Читать и писать чанками — путь, реализованный в этом \
         демо: read chunk → write chunk → read next. Пик памяти \
         равен размеру одного чанка, не зависит от размера файла."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "Backpressure: write блокирует цикл, пока диск не готов. \
         Поэтому read/write строго чередуются — никакого предчтения. \
         Метрики показывают раздельно read time и write time: \
         узкое место — медленная сторона."
            .to_string(),
    );
    return lines.join("\n");
}
```

---

## Что демонстрация показывает

**Блок A: транзакция.**
После `Write to buffer` две панели **специально расходятся**: «в буфере» показывает новое, «на диске» — старое. Это тот самый момент «аhа!»: write не трогает диск. `Commit` синхронизирует панели, `Abort` откатывает буфер.

**Блок B: забыл close.**
Кнопка «Write и забыть» открывает сессию, пишет, **роняет** её без close — `Drop` вызывает `abort`. Файл остаётся прежним. Рядом «правильно с close» — файл меняется. Студент навсегда запоминает, что успешный write **не означает**, что данные сохранены.

**Блок C: цена seek.**
Наивный способ: `read 4096 → modify → write 4096`. С seek: `open(keep) → seek(1000) → write(3) → close`. Результат одинаковый — байты `[1000..1003]` стали `XYZ`. Счётчики: **0 vs 4096 байт чтения, 3 vs 4096 байт записи.**

**Блок D: машина состояний.**
Четыре кнопки провоцируют крайние случаи:
- `write` после `close` → `SessionClosed` с объяснением.
- `write` после `abort` → то же.
- `close` дважды → OK (идемпотентно).
- `abort` дважды → OK.

## Правила проекта, зафиксированные в 19.5

1. **`app.rs` содержит только демо 19.5** — предыдущие подглавы не накапливаются.
2. **`write()` идёт в буфер, а не на диск.** Изменения фиксируются только `close()`. Блок A показывает это **визуально** через расхождение панелей.
3. **Незакрытая сессия откатывается через `Drop`.** Блок B делает это намеренно — потеря данных без явной ошибки.
4. **`seek + write` — точечная правка, не вставка.** Байты заменяются по смещению; остаток файла не сдвигается. Для `ABCDEFGHIJ` и `seek(5) + write("XXX")` результат — `ABCDEXXXIJ`, а не `ABCDEXXXJK`. Блок C показывает разницу в объёме I/O между наивным способом и seek.
5. **`keepExistingData` — сохранение хвоста.** `createWritable()` без опций усекает файл.
6. **`commit` и `abort` идемпотентны.** Повторный вызов — OK.
7. **`write` после `close`/`abort` → `SessionClosed`.** Явная ошибка, а не паника.
8. **Внутри `Closure::new(move || …)` элементы клонируются до `move`** — иначе заимствованный `&T` не переживёт `'static`.
9. **Все строки через `format_str` или `Vec<String>.join`.**
10. **`return` в хвостах макросов** (`matches!`, `format!`).
11. **`cargo check` без предупреждений.**

## Команды

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_fsa_write
npm start
```

Ожидаемый результат тестов: **21 passed, 0 failed**.