# Глава 18. File API

## 18.17. Хранение: Cache API, IndexedDB, OPFS

В предыдущих подглавах мы работали с файлами в рамках одной сессии: прочитали, обработали, отправили, сохранили на диск. Но что, если пользователь закроет вкладку и вернётся через день? Все данные исчезнут. Чтобы этого избежать, нужна **персистентность** — сохранение данных между сессиями браузера.

В браузере есть три независимых механизма хранения:

| Механизм | Для чего | Особенности |
|---|---|---|
| **Cache API** | HTTP-ответы, Response, офлайн-ресурсы | Умеет хранить `Response` с `Blob`-телом. Подходит для файлов и запросов. |
| **IndexedDB** | Структурированные данные, большие объекты | Хранит JS-объекты, включая `Blob` нативно. Транзакционный. |
| **OPFS** | Большие бинарные файлы | Настоящая файловая система в браузере: файлы, директории, потоки. |

**Cache API** — самый простой для «файлов-как-ответов»: превращаем `Blob` в `Response` и кладём в именованный кэш. Работает с любым MIME-типом, поддерживает диапазоны байтов через `Range`-заголовки.

**IndexedDB** — универсальное хранилище структурированных данных. Может содержать миллионы записей, индексировать поля, делать транзакции. Каждый `Blob` хранится **нативно** — не сериализуется, не копируется в массив байтов.

**OPFS (Origin Private File System)** — самая «файловая» из трёх. Даёт реальный доступ к виртуальной файловой системе: директории, файлы, потоки чтения и записи. Идеальна для больших файлов, которые не хочется грузить в память целиком.

### Когда что использовать

- **Кэшируем результат `fetch`** → Cache API.
- **Храним метаданные + небольшой Blob** (аватар, превью, черновик) → IndexedDB.
- **Работаем с большим бинарным файлом** (видео, датасет, архив) → OPFS.

Механизмы **сосуществуют**: можно параллельно вести кэш HTTP-ресурсов в Cache API, пользовательские настройки в IndexedDB и большие файлы в OPFS. Они не конфликтуют и имеют раздельные квоты.

### Про квоты

Каждый origin получает **общую квоту** на всё хранилище: Cache API + IndexedDB + OPFS + Service Worker. Размер зависит от устройства и свободного места. `navigator.storage.estimate()` возвращает `{ usage, quota }` в байтах — покажем их в демо.

### Про доступность API в разных браузерах

Диагностика — важная часть демонстрации. Браузер может:

- Не поддерживать OPFS вообще (старые версии).
- Запретить хранение данных для сайта (`SecurityError`) — особенно в приватном режиме.
- Отключить site data в настройках приватности.

Разные браузеры дают **разные пути решения** — Chrome, Firefox и Safari требуют идти в разные места настроек. Демо должно это учитывать и не показывать пользователю Chrome-специфичную подсказку, если он в Firefox.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # +idb crate, +web-sys фичи
├── src/
│   ├── lib.rs                     # +pub mod cache_storage, idb_store, opfs;
│   ├── app.rs                     # только секция 18.17
│   ├── cache_storage.rs           # НОВЫЙ: Cache API
│   ├── idb_store.rs               # НОВЫЙ: IndexedDB через crate idb
│   ├── opfs.rs                    # НОВЫЙ: Origin Private File System
│   └── … остальные модули
└── tests/
    ├── wasm_cache_storage.rs      # НОВЫЙ
    ├── wasm_idb_store.rs          # НОВЫЙ
    └── wasm_opfs.rs               # НОВЫЙ
```

**Пояснения:**

- `cache_storage.rs` — обёртка над `caches.open` / `put` / `match` / `delete` / `keys`.
- `idb_store.rs` — использует crate `idb = "0.6"`. Хранит `{ id, name, mime, bytes }` в object store. Через `serde_wasm_bindgen` — сериализация и десериализация на границе IDB.
- `opfs.rs` — работает через `navigator.storage.getDirectory()` → `FileSystemDirectoryHandle` → `getFileHandle` → `createWritable`. Плюс функция `check_access()`, возвращающая структурированную ошибку.

---

## Ссылки для детального изучения

- **MDN: Cache API** — https://developer.mozilla.org/en-US/docs/Web/API/Cache
- **MDN: IndexedDB API** — https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
- **MDN: Origin Private File System (OPFS)** — https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system
- **MDN: StorageManager.estimate()** — https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/estimate
- **crate `idb`** — https://docs.rs/idb/latest/idb/
- **web-sys: Cache** — https://docs.rs/web-sys/latest/web_sys/struct.Cache.html
- **Chrome: Storage quotas** — https://developer.chrome.com/docs/capabilities/storage

---

## Обновление `Cargo.toml`

```toml
[dependencies]
# … существующие …
idb = "0.6"
serde-wasm-bindgen = "0.6"

[dependencies.web-sys]
version = "0.3"
features = [
    # … существующие …

    # 18.17: Cache API
    "Cache",
    "CacheStorage",

    # 18.17: StorageManager
    "StorageManager",

    # 18.17: OPFS (часть фич уже включена в 18.15)
    "FileSystemGetDirectoryOptions",
    "FileSystemGetFileOptions",
    "FileSystemCreateWritableOptions",
    "FileSystemRemoveOptions",
]
```

---

## `src/lib.rs`

```rust
// ---- Глава 18. File API ----
// … модули из предыдущих подглав …
pub mod cache_storage;   // 18.17
pub mod idb_store;       // 18.17
pub mod opfs;            // 18.17
```

---

## Модуль 1 — `src/cache_storage.rs`

```rust
//! Cache API: хранение Blob как Response.
//!
//! **Зачем.** Cache API — самый простой способ хранить «файлы как
//! ответы сервера». Работает поверх стандартного `Request`/`Response`,
//! поддерживает Range-запросы, MIME-типы и заголовки.
//!
//! **Про ключи.** Ключ для Cache API — это валидный URL. Подходят
//! `http://`, `https://` или относительные пути (`/path`), которые
//! резолвятся против текущего origin. Произвольные схемы вроде
//! `test://` не поддерживаются.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, Cache, CacheStorage, Request, RequestInit, Response};

/// Открывает (или создаёт) именованный кэш.
pub async fn open_cache(name: &str) -> Result<Cache, JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let caches: CacheStorage = window
        .caches()
        .map_err(|_| JsValue::from_str("caches недоступен"))?;

    let promise = caches.open(name);
    let value = JsFuture::from(promise).await?;
    return value
        .dyn_into::<Cache>()
        .map_err(|_| JsValue::from_str("не Cache"));
}

/// Превращает Blob в Response — то, что Cache API умеет хранить.
fn blob_to_response(blob: &Blob) -> Result<Response, JsValue> {
    return Response::new_with_opt_blob(Some(blob));
}

/// Кладёт Blob в кэш под указанным ключом (URL).
pub async fn put_blob(
    cache_name: &str,
    key: &str,
    blob: &Blob,
) -> Result<(), JsValue> {
    let cache = open_cache(cache_name).await?;

    let opts = RequestInit::new();
    opts.set_method("GET");
    let request = Request::new_with_str_and_init(key, &opts)?;

    let response = blob_to_response(blob)?;
    let promise = cache.put_with_request(&request, &response);
    let _ = JsFuture::from(promise).await?;
    return Ok(());
}

/// Читает Blob из кэша по ключу. `Ok(None)`, если записи нет.
pub async fn get_blob(
    cache_name: &str,
    key: &str,
) -> Result<Option<Blob>, JsValue> {
    let cache = open_cache(cache_name).await?;

    let opts = RequestInit::new();
    opts.set_method("GET");
    let request = Request::new_with_str_and_init(key, &opts)?;

    let promise = cache.match_with_request(&request);
    let value = JsFuture::from(promise).await?;

    if value.is_undefined() || value.is_null() {
        return Ok(None);
    }

    let response: Response = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Response"))?;
    let blob_promise = response.blob()?;
    let blob_value = JsFuture::from(blob_promise).await?;
    let blob: Blob = blob_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Blob"))?;
    return Ok(Some(blob));
}

/// Удаляет запись из кэша. Идемпотентно: `false`, если записи не было.
pub async fn delete_blob(
    cache_name: &str,
    key: &str,
) -> Result<bool, JsValue> {
    let cache = open_cache(cache_name).await?;

    let opts = RequestInit::new();
    opts.set_method("GET");
    let request = Request::new_with_str_and_init(key, &opts)?;

    let promise = cache.delete_with_request(&request);
    let value = JsFuture::from(promise).await?;
    return Ok(value.as_bool().unwrap_or(false));
}

/// Возвращает список URL, хранящихся в кэше.
pub async fn list_keys(cache_name: &str) -> Result<Vec<String>, JsValue> {
    let cache = open_cache(cache_name).await?;

    let promise = cache.keys();
    let value = JsFuture::from(promise).await?;
    let array: js_sys::Array = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("keys не Array"))?;

    let mut out: Vec<String> = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let item = array.get(i);
        if let Some(s) = item.as_string() {
            out.push(s);
            continue;
        }
        let req: Request = match item.dyn_into() {
            Ok(r) => r,
            Err(_) => continue,
        };
        out.push(req.url());
    }
    return Ok(out);
}

/// Удаляет весь кэш с указанным именем. Идемпотентно.
pub async fn delete_cache(name: &str) -> Result<bool, JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let caches = window
        .caches()
        .map_err(|_| JsValue::from_str("caches недоступен"))?;
    let promise = caches.delete(name);
    let value = JsFuture::from(promise).await?;
    return Ok(value.as_bool().unwrap_or(false));
}
```

**Ключевые моменты:**

- **`Response::new_with_opt_blob(Some(blob))`** — превращает Blob в Response, готовый к `cache.put`.
- **`cache.put_with_request` / `match_with_request` / `delete_with_request`** — принимают `Request` как ключ.
- **`cache.keys()`** возвращает `Array<Request>` — извлекаем `.url()`.
- Все операции идемпотентны: удаление несуществующего даёт `false`, но не ошибку.

---

## Модуль 2 — `src/idb_store.rs`

```rust
//! IndexedDB: структурированное хранилище через crate `idb`.
//!
//! **Модель.** Открываем базу (`hello-wasm-db`) с object store
//! `files` и keyPath `id`. В store хранятся записи
//! `{ id, name, mime, bytes }`.
//!
//! **Про сериализацию.** Crate `idb` работает с `JsValue`, а не с
//! Rust-структурами. Преобразование — через `serde_wasm_bindgen`
//! в обе стороны. На чтении `store.get()` возвращает
//! `Option<JsValue>` (а не `JsValue`), а `store.get_all()` —
//! `Vec<JsValue>` (а не единый Array). Каждый элемент
//! десериализуется отдельно.

use idb::{
    Database, DatabaseEvent, Factory, KeyPath, ObjectStoreParams,
    TransactionMode,
};
use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::Blob;

/// Запись, хранимая в IndexedDB.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct StoredEntry {
    pub id: String,
    pub name: String,
    pub mime: String,
    pub bytes: Vec<u8>,
}

/// Открывает базу данных с object store `files`.
pub async fn open_db() -> Result<Database, JsValue> {
    let factory = Factory::new()
        .map_err(|e| JsValue::from_str(&format!("Factory::new: {e:?}")))?;

    let mut open_request = factory
        .open("hello-wasm-db", Some(1))
        .map_err(|e| JsValue::from_str(&format!("open: {e:?}")))?;

    // `DatabaseEvent` нужен для метода `event.database()`.
    open_request.on_upgrade_needed(|event| {
        let db: Database = match event.database() {
            Ok(d) => d,
            Err(_) => return,
        };
        let mut params = ObjectStoreParams::new();
        params.key_path(Some(KeyPath::new_single("id")));
        let _ = db.create_object_store("files", params);
    });

    let db = open_request
        .await
        .map_err(|e| JsValue::from_str(&format!("await open: {e:?}")))?;
    return Ok(db);
}

/// Сохраняет запись. Перезаписывает по ключу `id`.
pub async fn put_entry(entry: &StoredEntry) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&["files"], TransactionMode::ReadWrite)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store("files")
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;

    // serde → JsValue: `store.put` принимает &JsValue, не Rust-структуру.
    let js_value = serde_wasm_bindgen::to_value(entry)
        .map_err(|e| JsValue::from_str(&format!("to_value: {e:?}")))?;

    store
        .put(&js_value, None)
        .map_err(|e| JsValue::from_str(&format!("put: {e:?}")))?;
    tx.commit()
        .map_err(|e| JsValue::from_str(&format!("commit: {e:?}")))?;
    return Ok(());
}

/// Читает запись по `id`. Возвращает `None`, если записи нет.
pub async fn get_entry(id: &str) -> Result<Option<StoredEntry>, JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&["files"], TransactionMode::ReadOnly)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store("files")
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;

    // Ключ — JsValue: `store.get` требует `impl Into<Query>`,
    // а `&str` этот трейт не реализует.
    let key: JsValue = JsValue::from_str(id);

    // idb 0.6.5: get() возвращает Option<JsValue>.
    let value_opt: Option<JsValue> = store
        .get(key)
        .map_err(|e| JsValue::from_str(&format!("get: {e:?}")))?
        .await
        .map_err(|e| JsValue::from_str(&format!("await get: {e:?}")))?;

    let value = match value_opt {
        Some(v) => v,
        None => return Ok(None),
    };

    let entry: StoredEntry = serde_wasm_bindgen::from_value(value)
        .map_err(|e| JsValue::from_str(&format!("from_value: {e:?}")))?;
    return Ok(Some(entry));
}

/// Возвращает все записи в store.
pub async fn list_entries() -> Result<Vec<StoredEntry>, JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&["files"], TransactionMode::ReadOnly)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store("files")
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;

    // idb 0.6.5: get_all() возвращает Vec<JsValue>.
    let values: Vec<JsValue> = store
        .get_all(None, None)
        .map_err(|e| JsValue::from_str(&format!("get_all: {e:?}")))?
        .await
        .map_err(|e| JsValue::from_str(&format!("await get_all: {e:?}")))?;

    let mut entries: Vec<StoredEntry> = Vec::with_capacity(values.len());
    for value in values {
        let entry: StoredEntry = serde_wasm_bindgen::from_value(value)
            .map_err(|e| JsValue::from_str(&format!("from_value: {e:?}")))?;
        entries.push(entry);
    }
    return Ok(entries);
}

/// Удаляет запись по `id`. Идемпотентно.
pub async fn delete_entry(id: &str) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&["files"], TransactionMode::ReadWrite)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store("files")
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;

    let key: JsValue = JsValue::from_str(id);
    store
        .delete(key)
        .map_err(|e| JsValue::from_str(&format!("delete: {e:?}")))?;
    tx.commit()
        .map_err(|e| JsValue::from_str(&format!("commit: {e:?}")))?;
    return Ok(());
}

/// Создаёт `StoredEntry` из Blob.
pub async fn entry_from_blob(
    id: &str,
    name: &str,
    blob: &Blob,
) -> Result<StoredEntry, JsValue> {
    let array_promise = blob.array_buffer();
    let value = wasm_bindgen_futures::JsFuture::from(array_promise).await?;
    let array_buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    let bytes = js_sys::Uint8Array::new(&array_buffer).to_vec();
    return Ok(StoredEntry {
        id: id.to_string(),
        name: name.to_string(),
        mime: blob.type_(),
        bytes,
    });
}

/// Восстанавливает Blob из `StoredEntry`.
pub fn blob_from_entry(entry: &StoredEntry) -> Result<Blob, JsValue> {
    let uint8 = js_sys::Uint8Array::from(&entry.bytes[..]);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type(&entry.mime);
    return Blob::new_with_u8_array_sequence_and_options(&parts, &options);
}
```

**Ключевые моменты:**

- **`idb` работает с `JsValue`, не с Rust-структурами.** Все данные сериализуются через `serde_wasm_bindgen::to_value` при записи и `from_value` при чтении.
- **`DatabaseEvent`** импортируется явно — трейт предоставляет `event.database()`.
- **`store.get()` возвращает `Option<JsValue>`**, а не `JsValue`. Отсутствие записи — `None`.
- **`store.get_all()` возвращает `Vec<JsValue>`** — десериализуем каждый элемент отдельно.
- **`let db: Database = ...`** — явная аннотация типа внутри замыкания `on_upgrade_needed`.

---

## Модуль 3 — `src/opfs.rs`

```rust
//! Origin Private File System (OPFS).
//!
//! **Зачем.** OPFS — настоящая файловая система внутри браузера.
//! Доступна по `navigator.storage.getDirectory()`. Подходит для
//! больших бинарных файлов: чтение и запись потоково, без
//! загрузки всего в память.
//!
//! **Особенности web-sys 0.3.105.** Методы `getFileHandle`,
//! `getDirectoryHandle`, `createWritable` вызываем через
//! `Reflect::get` + `Function::call*` — не зависим от точных
//! сигнатур.
//!
//! **Требования:** Chrome 86+, Firefox 111+, Safari 15.2+
//! + secure context. В Firefox OPFS недоступен в приватном
//! режиме — `check_access()` вернёт `SecurityError`.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::FileSystemDirectoryHandle;

/// Причина, по которой OPFS недоступен.
#[derive(Debug, Clone)]
pub enum OpfsError {
    /// В браузере нет `navigator.storage.getDirectory`.
    Unsupported,
    /// Браузер запретил доступ (настройки приватности, приватный режим).
    SecurityError,
    /// Прочие ошибки с сохранённым описанием.
    Other(String),
}

/// Возвращает корневой handle OPFS.
pub async fn get_root() -> Result<FileSystemDirectoryHandle, JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator = window.navigator();
    let nav_js: &JsValue = navigator.as_ref();

    let storage_js = js_sys::Reflect::get(nav_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() || storage_js.is_null() {
        return Err(JsValue::from_str("storage недоступен"));
    }

    let get_dir_fn_js = js_sys::Reflect::get(
        &storage_js,
        &JsValue::from_str("getDirectory"),
    )?;
    if get_dir_fn_js.is_undefined() {
        return Err(JsValue::from_str("storage.getDirectory недоступен"));
    }
    let get_dir_fn: js_sys::Function = get_dir_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getDirectory не функция"))?;

    let promise_js = get_dir_fn.call0(&storage_js)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let value = JsFuture::from(promise).await?;
    return value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"));
}

/// Проверяет, доступна ли OPFS в текущем окружении.
///
/// Возвращает типизированную ошибку — форматирование и подсказки
/// для пользователя делает UI-слой, где известен браузер.
pub async fn check_access() -> Result<(), OpfsError> {
    match get_root().await {
        Ok(_) => return Ok(()),
        Err(e) => {
            let raw = e.as_string().unwrap_or_else(|| format!("{e:?}"));
            if raw.contains("недоступен") {
                return Err(OpfsError::Unsupported);
            }
            if raw.contains("SecurityError") || raw.contains("Security error") {
                return Err(OpfsError::SecurityError);
            }
            return Err(OpfsError::Other(raw));
        }
    }
}

/// Записывает байты в файл OPFS. Если файл существует — перезаписывается.
pub async fn write_file(name: &str, bytes: &[u8]) -> Result<(), JsValue> {
    let root = get_root().await?;

    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("create"),
        &JsValue::from_bool(true),
    )?;

    let root_js: &JsValue = root.as_ref();
    let get_file_fn_js = js_sys::Reflect::get(
        root_js,
        &JsValue::from_str("getFileHandle"),
    )?;
    let get_file_fn: js_sys::Function = get_file_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFileHandle не функция"))?;

    let file_handle_js = get_file_fn.call2(
        root_js,
        &JsValue::from_str(name),
        &options,
    )?;
    let file_handle_promise: js_sys::Promise = file_handle_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let file_handle = JsFuture::from(file_handle_promise).await?;

    let file_handle_ref: &JsValue = &file_handle;
    let create_fn_js = js_sys::Reflect::get(
        file_handle_ref,
        &JsValue::from_str("createWritable"),
    )?;
    let create_fn: js_sys::Function = create_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("createWritable не функция"))?;
    let writable_js = create_fn.call0(file_handle_ref)?;
    let writable_promise: js_sys::Promise = writable_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let writable = JsFuture::from(writable_promise).await?;

    let uint8 = js_sys::Uint8Array::from(bytes);
    let writable_ref: &JsValue = &writable;
    let write_fn_js = js_sys::Reflect::get(
        writable_ref,
        &JsValue::from_str("write"),
    )?;
    let write_fn: js_sys::Function = write_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("write не функция"))?;
    let write_result_js = write_fn.call1(writable_ref, &uint8.buffer())?;
    let write_result_promise: js_sys::Promise = write_result_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    JsFuture::from(write_result_promise).await?;

    let close_fn_js = js_sys::Reflect::get(
        writable_ref,
        &JsValue::from_str("close"),
    )?;
    let close_fn: js_sys::Function = close_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("close не функция"))?;
    let close_result_js = close_fn.call0(writable_ref)?;
    let close_result_promise: js_sys::Promise = close_result_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    JsFuture::from(close_result_promise).await?;

    return Ok(());
}

/// Читает файл OPFS целиком в `Vec<u8>`.
pub async fn read_file(name: &str) -> Result<Vec<u8>, JsValue> {
    let root = get_root().await?;

    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("create"),
        &JsValue::from_bool(false),
    )?;

    let root_js: &JsValue = root.as_ref();
    let get_file_fn_js = js_sys::Reflect::get(
        root_js,
        &JsValue::from_str("getFileHandle"),
    )?;
    let get_file_fn: js_sys::Function = get_file_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFileHandle не функция"))?;

    let file_handle_js = get_file_fn.call2(
        root_js,
        &JsValue::from_str(name),
        &options,
    )?;
    let file_handle_promise: js_sys::Promise = file_handle_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let file_handle = JsFuture::from(file_handle_promise).await?;

    let file_handle_ref: &JsValue = &file_handle;
    let get_file_method_js = js_sys::Reflect::get(
        file_handle_ref,
        &JsValue::from_str("getFile"),
    )?;
    let get_file_method: js_sys::Function = get_file_method_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFile не функция"))?;
    let file_promise_js = get_file_method.call0(file_handle_ref)?;
    let file_promise: js_sys::Promise = file_promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let file_value = JsFuture::from(file_promise).await?;
    let file: web_sys::File = file_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не File"))?;

    let ab_promise = file.array_buffer();
    let ab_value = JsFuture::from(ab_promise).await?;
    let array_buffer: js_sys::ArrayBuffer = ab_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    return Ok(js_sys::Uint8Array::new(&array_buffer).to_vec());
}

/// Удаляет файл OPFS. Идемпотентно.
pub async fn remove_file(name: &str) -> Result<(), JsValue> {
    let root = get_root().await?;
    let root_js: &JsValue = root.as_ref();

    let remove_fn_js = js_sys::Reflect::get(
        root_js,
        &JsValue::from_str("removeEntry"),
    )?;
    let remove_fn: js_sys::Function = remove_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("removeEntry не функция"))?;
    let promise_js = remove_fn.call1(root_js, &JsValue::from_str(name))?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let _ = JsFuture::from(promise).await?;
    return Ok(());
}

/// Возвращает размер файла OPFS в байтах.
pub async fn file_size(name: &str) -> Result<f64, JsValue> {
    let root = get_root().await?;
    let options = js_sys::Object::new();
    js_sys::Reflect::set(
        &options,
        &JsValue::from_str("create"),
        &JsValue::from_bool(false),
    )?;
    let root_js: &JsValue = root.as_ref();
    let get_file_fn_js = js_sys::Reflect::get(
        root_js,
        &JsValue::from_str("getFileHandle"),
    )?;
    let get_file_fn: js_sys::Function = get_file_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFileHandle не функция"))?;

    let file_handle_js = get_file_fn.call2(
        root_js,
        &JsValue::from_str(name),
        &options,
    )?;
    let file_handle_promise: js_sys::Promise = file_handle_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let file_handle = JsFuture::from(file_handle_promise).await?;

    let file_handle_ref: &JsValue = &file_handle;
    let get_file_method_js = js_sys::Reflect::get(
        file_handle_ref,
        &JsValue::from_str("getFile"),
    )?;
    let get_file_method: js_sys::Function = get_file_method_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getFile не функция"))?;
    let file_promise_js = get_file_method.call0(file_handle_ref)?;
    let file_promise: js_sys::Promise = file_promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let file_value = JsFuture::from(file_promise).await?;
    let file: web_sys::File = file_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не File"))?;
    return Ok(file.size());
}

/// Возвращает `{ usage, quota }` в байтах.
pub async fn estimate() -> Result<(f64, f64), JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator = window.navigator();
    let nav_js: &JsValue = navigator.as_ref();

    let storage_js = js_sys::Reflect::get(nav_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() || storage_js.is_null() {
        return Err(JsValue::from_str("storage недоступен"));
    }

    let estimate_fn_js = js_sys::Reflect::get(
        &storage_js,
        &JsValue::from_str("estimate"),
    )?;
    let estimate_fn: js_sys::Function = estimate_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("estimate не функция"))?;
    let promise_js = estimate_fn.call0(&storage_js)?;
    let promise: js_sys::Promise = promise_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let value = JsFuture::from(promise).await?;

    let usage = js_sys::Reflect::get(&value, &JsValue::from_str("usage"))?
        .as_f64()
        .unwrap_or(0.0);
    let quota = js_sys::Reflect::get(&value, &JsValue::from_str("quota"))?
        .as_f64()
        .unwrap_or(0.0);
    return Ok((usage, quota));
}
```

---

## Тесты

### `tests/wasm_cache_storage.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::cache_storage::*;

wasm_bindgen_test_configure!(run_in_browser);

const CACHE_NAME: &str = "wasm-test-cache";

/// Ключи в Cache API — валидные URL. Используем относительные пути,
/// они резолвятся против текущего origin.
const KEY_1: &str = "/test-key-1";
const KEY_MISSING: &str = "/no-such-key";

#[wasm_bindgen_test]
async fn test_put_get_roundtrip() {
    let parts = js_sys::Array::of1(&"hello".into());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type("text/plain");
    let blob = web_sys::Blob::new_with_str_sequence_and_options(
        &parts, &options,
    )
    .expect("Blob::new_with_str_sequence_and_options");

    put_blob(CACHE_NAME, KEY_1, &blob)
        .await
        .expect("put_blob");

    let read = get_blob(CACHE_NAME, KEY_1)
        .await
        .expect("get_blob");
    assert!(read.is_some(), "ожидался Blob");

    let _ = delete_cache(CACHE_NAME).await;
}

#[wasm_bindgen_test]
async fn test_get_missing_returns_none() {
    let result = get_blob(CACHE_NAME, KEY_MISSING)
        .await
        .expect("get_blob");
    assert!(result.is_none(), "не должно быть записи");
}

#[wasm_bindgen_test]
async fn test_delete_idempotent() {
    let first = delete_blob(CACHE_NAME, KEY_MISSING)
        .await
        .expect("delete");
    assert!(!first, "первое удаление несуществующего — false");

    let second = delete_blob(CACHE_NAME, KEY_MISSING)
        .await
        .expect("delete");
    assert!(!second, "повторное удаление — тоже false");
}

#[wasm_bindgen_test]
async fn test_list_keys_returns_vec() {
    let keys = list_keys(CACHE_NAME).await.expect("list_keys");
    for k in &keys {
        assert!(!k.is_empty(), "ключ не пустой");
    }
}
```

### `tests/wasm_idb_store.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::idb_store::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_put_get_roundtrip() {
    let entry = StoredEntry {
        id: "test-1".to_string(),
        name: "test.txt".to_string(),
        mime: "text/plain".to_string(),
        bytes: b"hello".to_vec(),
    };
    put_entry(&entry).await.expect("put_entry");

    let read = get_entry("test-1").await.expect("get_entry");
    assert_eq!(read, Some(entry.clone()));

    let _ = delete_entry("test-1").await;
}

#[wasm_bindgen_test]
async fn test_get_missing_returns_none() {
    let read = get_entry("no-such-id").await.expect("get_entry");
    assert!(read.is_none(), "не должно быть записи");
}

#[wasm_bindgen_test]
async fn test_delete_idempotent() {
    delete_entry("no-such-id").await.expect("first delete");
    delete_entry("no-such-id").await.expect("second delete");
}

#[wasm_bindgen_test]
async fn test_list_contains_inserted() {
    let id = format!("list-test-{}", js_sys::Date::now() as u64);
    let entry = StoredEntry {
        id: id.clone(),
        name: "list.txt".to_string(),
        mime: "text/plain".to_string(),
        bytes: vec![1, 2, 3],
    };
    put_entry(&entry).await.expect("put_entry");

    let list = list_entries().await.expect("list_entries");
    let found = list.iter().any(|e| e.id == id);
    assert!(found, "запись {id} должна быть в списке");

    let _ = delete_entry(&id).await;
}
```

### `tests/wasm_opfs.rs`

```rust
use wasm_bindgen_test::*;
use hello_wasm02::opfs::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn test_write_read_roundtrip() {
    // Если OPFS недоступен — пропускаем (graceful skip).
    if check_access().await.is_err() {
        return;
    }

    let name = format!("test-{}.bin", js_sys::Date::now() as u64);
    let bytes: Vec<u8> = (0..=255).collect();

    write_file(&name, &bytes).await.expect("write_file");

    let read = read_file(&name).await.expect("read_file");
    assert_eq!(read, bytes);

    let size = file_size(&name).await.expect("file_size");
    assert_eq!(size as usize, bytes.len());

    let _ = remove_file(&name).await;
}

#[wasm_bindgen_test]
async fn test_read_missing_fails() {
    if check_access().await.is_err() {
        return;
    }
    let result = read_file("no-such-file.bin").await;
    assert!(result.is_err(), "ожидалась ошибка для отсутствующего файла");
}

#[wasm_bindgen_test]
async fn test_estimate_returns_pair() {
    let (usage, quota) = estimate().await.expect("estimate");
    assert!(quota > 0.0, "квота должна быть положительной");
    assert!(usage >= 0.0, "usage не может быть отрицательным");
}
```

---

## Команды запуска тестов

```bash
# Проверка типов
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests

# Тесты
wasm-pack test --headless --chrome --test wasm_cache_storage
wasm-pack test --headless --chrome --test wasm_idb_store
wasm-pack test --headless --chrome --test wasm_opfs

wasm-pack test --headless --firefox --test wasm_cache_storage
wasm-pack test --headless --firefox --test wasm_idb_store
wasm-pack test --headless --firefox --test wasm_opfs
```

**Ожидаемо:** `wasm_cache_storage` — 4 теста, `wasm_idb_store` — 4 теста, `wasm_opfs` — 3 теста. Все проходят.

---

## Демо-пример — `src/app.rs`

```rust
//! Демонстрация подглавы 18.17: Cache API, IndexedDB, OPFS.
//!
//! **Структура демо:**
//! 1. Диагностика — что доступно в браузере (с определением самого
//!    браузера), secure context, квота, асинхронная проверка OPFS.
//! 2. Cache API — сохранение/чтение Blob как Response.
//! 3. IndexedDB — структурированные записи `{ id, name, mime, bytes }`.
//! 4. OPFS — бинарный файл на 1 МБ.
//! 5. Сквозной тест — один Blob во все доступные хранилища.
//!
//! **Интеллектуальные детали:**
//! * Определяем браузер (Chrome/Firefox/Safari/Edge) и даём
//!   подсказки, специфичные для него.
//! * Проверяем OPFS асинхронно при старте — кнопки блокируются,
//!   пока проверка не завершится.
//! * Ошибки `SecurityError` объясняются понятным текстом, без
//!   stacktrace и `JsValue(...)`.
//! * Сквозной тест graceful skip недоступных API, не считая
//!   это ошибкой.
//! * Квота обновляется автоматически после каждой операции.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{cache_storage, dom, file_api, idb_store, log, opfs};

const CACHE_NAME: &str = "hello-wasm-cache";
const CACHE_KEY: &str = "/demo-cache-key";
const IDB_KEY: &str = "demo-entry";
const OPFS_NAME: &str = "data.bin";
const OPFS_SIZE: usize = 1024 * 1024;

/// Браузер пользователя — влияет на подсказки в UI.
#[derive(Debug, Clone, Copy, PartialEq)]
enum Browser {
    Chrome,
    Firefox,
    Safari,
    Edge,
    Unknown,
}

impl Browser {
    fn name(&self) -> &'static str {
        return match self {
            Browser::Chrome => "Chrome",
            Browser::Firefox => "Firefox",
            Browser::Safari => "Safari",
            Browser::Edge => "Edge",
            Browser::Unknown => "неизвестный",
        };
    }

    fn security_error_hint(&self) -> &'static str {
        return match self {
            Browser::Firefox =>
                "Firefox → Settings → Privacy & Security → Cookies and \
                 Site Data.\n\
                 Убедитесь, что вы не в приватном режиме: в Firefox OPFS \
                 недоступен в Private Browsing и при опции «Never remember \
                 history».",
            Browser::Chrome | Browser::Edge =>
                "Chrome → Settings → Privacy and security → Site Settings \
                 → Additional content settings → Cookies and site data.\n\
                 Сайт localhost должен быть разрешён.",
            Browser::Safari =>
                "Safari → Preferences → Privacy → Manage Website Data.\n\
                 Убедитесь, что не включён Private Browsing.",
            Browser::Unknown =>
                "Проверьте настройки приватности браузера: возможно, \
                 заблокированы cookies или site data.",
        };
    }
}

/// Определяет браузер по `navigator.userAgent`.
///
/// Порядок проверок важен: Edge содержит и «Edg/», и «Chrome/»,
/// Chrome содержит «Safari/» — проверяем в правильном порядке.
fn detect_browser() -> Browser {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return Browser::Unknown,
    };
    let ua: String = window.navigator().user_agent().unwrap_or_default();

    if ua.contains("Edg/") || ua.contains("Edge/") {
        return Browser::Edge;
    }
    if ua.contains("Firefox/") {
        return Browser::Firefox;
    }
    if ua.contains("Chrome/") || ua.contains("Chromium/") {
        return Browser::Chrome;
    }
    if ua.contains("Safari/") {
        return Browser::Safari;
    }
    return Browser::Unknown;
}

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Хранение: Cache API, IDB, OPFS")?;
    dom::append(&container, &heading)?;

    let intro = dom::create_paragraph(
        "Три независимых механизма браузерного хранилища. \
         Cache API — для HTTP-ответов и Blob; IndexedDB — для \
         структурированных данных; OPFS — для больших бинарных \
         файлов. Все три сосуществуют и делят общую квоту origin.",
    )?;
    dom::add_class(&dom::as_html(&intro), "hint")?;
    dom::append(&container, &intro)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    // Флаг: None = ещё проверяем, Some(true/false) = результат.
    let opfs_available: Rc<Cell<Option<bool>>> = Rc::new(Cell::new(None));
    let opfs_buttons: Rc<RefCell<Vec<web_sys::Element>>> =
        Rc::new(RefCell::new(Vec::new()));
    let opfs_status: Rc<RefCell<Option<web_sys::Element>>> =
        Rc::new(RefCell::new(None));

    build_diagnostics(&container, &closures, &opfs_status)?;
    build_cache_section(&container, &closures)?;
    build_idb_section(&container, &closures)?;
    build_opfs_section(&container, &closures, &opfs_buttons)?;
    build_cross_storage_section(&container, &closures, &opfs_available)?;

    // Асинхронная проверка OPFS. Результат обновляет флаг,
    // кнопки и статус в диагностике.
    {
        let opfs_available_c = opfs_available.clone();
        let opfs_buttons_c = opfs_buttons.clone();
        let opfs_status_c = opfs_status.clone();
        spawn_local(async move {
            let outcome = opfs::check_access().await;
            let (available, message): (bool, String) = match outcome {
                Ok(()) => (
                    true,
                    "✅ OPFS доступен: файловые операции работают."
                        .to_string(),
                ),
                Err(opfs::OpfsError::Unsupported) => {
                    let browser = detect_browser();
                    (
                        false,
                        format!(
                            "⚠️ OPFS недоступен в {}.\n\
                             Требуется Chrome 86+ / Firefox 111+ / \
                             Safari 15.2+ на secure context.",
                            browser.name(),
                        ),
                    )
                }
                Err(opfs::OpfsError::SecurityError) => {
                    let browser = detect_browser();
                    (
                        false,
                        format!(
                            "⚠️ OPFS недоступен: браузер запретил \
                             хранение данных (SecurityError).\n\n{}",
                            browser.security_error_hint(),
                        ),
                    )
                }
                Err(opfs::OpfsError::Other(msg)) => (
                    false,
                    format!("⚠️ OPFS недоступен: {msg}"),
                ),
            };
            opfs_available_c.set(Some(available));

            for btn in opfs_buttons_c.borrow().iter() {
                if available {
                    btn.remove_attribute("disabled").ok();
                } else {
                    btn.set_attribute("disabled", "").ok();
                }
            }

            if let Some(status) = opfs_status_c.borrow().as_ref() {
                status.set_text_content(Some(message.as_str()));
            }
        });
    }

    log!("Section 18.17 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ===============================================================
// Секция 0: диагностика и квота
// ===============================================================

fn build_diagnostics(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
    opfs_status_out: &Rc<RefCell<Option<web_sys::Element>>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("Диагностика хранилища"));
    dom::append(&section, &h2)?;

    let info = dom::create_element("div")?;
    info.set_attribute(
        "style",
        "margin: 12px 0; padding: 10px 14px; \
         background: #f0f4ff; border-left: 4px solid #4a7cff; \
         border-radius: 4px; font-family: monospace; \
         white-space: pre-line; line-height: 1.6;",
    )?;

    let protocol: String = web_sys::window()
        .and_then(|w| w.location().protocol().ok())
        .unwrap_or_default();
    let host: String = web_sys::window()
        .and_then(|w| w.location().host().ok())
        .unwrap_or_default();

    let browser = detect_browser();
    let secure = is_secure_context();
    let cache_ok = cache_supported();
    let idb_ok = idb_supported();

    let mut lines: Vec<String> = Vec::new();
    lines.push(format!("Protocol: {protocol}"));
    lines.push(format!("Host: {host}"));
    lines.push(format!("Browser: {}", browser.name()));

    let secure_line: String;
    if secure {
        secure_line = "Secure context: ✅".to_string();
    } else {
        secure_line =
            "Secure context: ⚠️ API требуют HTTPS или localhost".to_string();
    }
    lines.push(secure_line);

    let cache_line: String;
    if cache_ok {
        cache_line = "Cache API: ✅ доступен".to_string();
    } else {
        cache_line = "Cache API: ❌ недоступен".to_string();
    }
    lines.push(cache_line);

    let idb_line: String;
    if idb_ok {
        idb_line = "IndexedDB: ✅ доступен".to_string();
    } else {
        idb_line = "IndexedDB: ❌ недоступен".to_string();
    }
    lines.push(idb_line);

    info.set_text_content(Some(lines.join("\n").as_str()));
    dom::append(&section, &info)?;

    // Статус OPFS — отдельный элемент, не перетирает info.
    let opfs_status = dom::create_element("div")?;
    opfs_status.set_attribute(
        "style",
        "margin: 8px 0; padding: 8px 12px; \
         background: #fafafa; border-radius: 4px; \
         font-family: monospace; white-space: pre-line;",
    )?;
    opfs_status.set_text_content(Some("⏳ Проверяем доступность OPFS..."));
    dom::append(&section, &opfs_status)?;
    *opfs_status_out.borrow_mut() = Some(opfs_status);

    let refresh_btn = dom::create_element("button")?;
    refresh_btn.set_text_content(Some("Обновить квоту"));
    dom::append(&section, &refresh_btn)?;

    let quota = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&quota), "api-result")?;
    quota.set_text_content(Some("(нажмите «Обновить квоту»)"));
    dom::append(&section, &quota)?;

    dom::append(parent, &section)?;

    {
        let quota_c = quota.clone();
        callbacks::listen(
            &refresh_btn,
            "click",
            move |_ev: web_sys::Event| {
                refresh_quota(quota_c.clone());
            },
            closures,
        )?;
    }

    refresh_quota(quota);

    return Ok(());
}

fn refresh_quota(quota_element: web_sys::Element) {
    spawn_local(async move {
        match opfs::estimate().await {
            Ok((usage, quota)) => {
                let pct: f64;
                if quota > 0.0 {
                    pct = (usage / quota * 100.0).min(100.0);
                } else {
                    pct = 0.0;
                }
                let text = format!(
                    "Использовано: {} ({:.2}%)\n\
                     Квота: {}\n\
                     Свободно: {}",
                    file_api::format_file_size(usage),
                    pct,
                    file_api::format_file_size(quota),
                    file_api::format_file_size((quota - usage).max(0.0)),
                );
                quota_element.set_text_content(Some(text.as_str()));
            }
            Err(e) => {
                let msg = format!("⚠️ estimate недоступен: {}", format_js_error(&e));
                quota_element.set_text_content(Some(msg.as_str()));
            }
        }
    });
}

// ===============================================================
// Секция 1: Cache API
// ===============================================================

fn build_cache_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("1. Cache API"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Cache API хранит пары «URL → Response». Превращаем Blob \
         в Response и кладём под ключом-URL. Идеально для \
         кэширования HTTP-ответов и офлайн-ресурсов.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let save_btn = create_button("Сохранить тестовый Blob")?;
    let read_btn = create_button("Прочитать")?;
    let list_btn = create_button("Список ключей")?;
    let delete_btn = create_button("Удалить всё")?;

    dom::append(&section, &save_btn)?;
    dom::append(&section, &read_btn)?;
    dom::append(&section, &list_btn)?;
    dom::append(&section, &delete_btn)?;

    let result = make_result(&section)?;
    dom::append(parent, &section)?;

    {
        let result_c = result.clone();
        callbacks::listen(
            &save_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                result_c.set_text_content(Some("Сохраняем..."));
                spawn_local(async move {
                    let content = "Cache API test content\n";
                    let parts = js_sys::Array::of1(&JsValue::from_str(content));
                    let options = web_sys::BlobPropertyBag::new();
                    options.set_type("text/plain");
                    let blob = match web_sys::Blob::new_with_str_sequence_and_options(
                        &parts, &options,
                    ) {
                        Ok(b) => b,
                        Err(e) => {
                            let msg = format!("❌ Blob: {}", format_js_error(&e));
                            result_c.set_text_content(Some(msg.as_str()));
                            return;
                        }
                    };
                    match cache_storage::put_blob(CACHE_NAME, CACHE_KEY, &blob).await {
                        Ok(()) => {
                            let msg = format!(
                                "✅ Blob сохранён в Cache API\n\n\
                                 Кэш: {CACHE_NAME}\n\
                                 Ключ: {CACHE_KEY}\n\
                                 Размер: {} байт",
                                content.len(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ put_blob: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &read_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match cache_storage::get_blob(CACHE_NAME, CACHE_KEY).await {
                        Ok(Some(blob)) => {
                            let mime_display: String;
                            if blob.type_().is_empty() {
                                mime_display = "(не определён)".to_string();
                            } else {
                                mime_display = blob.type_();
                            }
                            let msg = format!(
                                "✅ Blob прочитан из Cache API\n\n\
                                 Размер: {} байт\n\
                                 MIME: {mime_display}",
                                blob.size(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Ok(None) => {
                            result_c.set_text_content(Some(
                                "Записи нет. Сначала нажмите «Сохранить»."));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ get_blob: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &list_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match cache_storage::list_keys(CACHE_NAME).await {
                        Ok(keys) => {
                            let mut out = format!("Ключей в кэше: {}\n\n", keys.len());
                            for k in &keys {
                                out.push_str(&format!("  {k}\n"));
                            }
                            result_c.set_text_content(Some(out.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ list_keys: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &delete_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match cache_storage::delete_cache(CACHE_NAME).await {
                        Ok(removed) => {
                            let status: &str;
                            if removed {
                                status = "удалён";
                            } else {
                                status = "не существовал";
                            }
                            let msg = format!(
                                "🗑 Кэш {CACHE_NAME}: {status}\n\
                                 (removed = {removed}) — идемпотентно",
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ delete_cache: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

// ===============================================================
// Секция 2: IndexedDB
// ===============================================================

fn build_idb_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("2. IndexedDB"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "IndexedDB хранит структурированные записи с ключами и \
         индексами. Мы сохраняем `{ id, name, mime, bytes }`. \
         Подходит для миллионов записей и сложных запросов.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let save_btn = create_button("Сохранить запись")?;
    let read_btn = create_button("Прочитать")?;
    let list_btn = create_button("Список записей")?;
    let delete_btn = create_button("Удалить")?;

    dom::append(&section, &save_btn)?;
    dom::append(&section, &read_btn)?;
    dom::append(&section, &list_btn)?;
    dom::append(&section, &delete_btn)?;

    let result = make_result(&section)?;
    dom::append(parent, &section)?;

    {
        let result_c = result.clone();
        callbacks::listen(
            &save_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                result_c.set_text_content(Some("Сохраняем..."));
                spawn_local(async move {
                    let bytes: Vec<u8> = (0..=255).collect();
                    let entry = idb_store::StoredEntry {
                        id: IDB_KEY.to_string(),
                        name: "demo.bin".to_string(),
                        mime: "application/octet-stream".to_string(),
                        bytes: bytes.clone(),
                    };
                    match idb_store::put_entry(&entry).await {
                        Ok(()) => {
                            let msg = format!(
                                "✅ Запись сохранена в IndexedDB\n\n\
                                 ID: {IDB_KEY}\n\
                                 Имя: demo.bin\n\
                                 Размер: {} байт",
                                bytes.len(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ put_entry: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &read_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match idb_store::get_entry(IDB_KEY).await {
                        Ok(Some(entry)) => {
                            let msg = format!(
                                "✅ Прочитана запись из IndexedDB\n\n\
                                 ID: {}\n\
                                 Имя: {}\n\
                                 MIME: {}\n\
                                 Размер: {} байт",
                                entry.id,
                                entry.name,
                                entry.mime,
                                entry.bytes.len(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Ok(None) => {
                            result_c.set_text_content(Some(
                                "Записи нет. Нажмите «Сохранить»."));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ get_entry: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &list_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match idb_store::list_entries().await {
                        Ok(entries) => {
                            let mut out = format!(
                                "Записей в store: {}\n\n",
                                entries.len(),
                            );
                            for e in entries.iter().take(20) {
                                out.push_str(&format!(
                                    "  {} · {} · {} байт\n",
                                    e.id, e.name, e.bytes.len(),
                                ));
                            }
                            if entries.len() > 20 {
                                out.push_str(&format!(
                                    "  … и ещё {}\n",
                                    entries.len() - 20,
                                ));
                            }
                            result_c.set_text_content(Some(out.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ list_entries: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &delete_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match idb_store::delete_entry(IDB_KEY).await {
                        Ok(()) => {
                            let msg = format!(
                                "🗑 Запись {IDB_KEY} удалена (идемпотентно)",
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ delete_entry: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

// ===============================================================
// Секция 3: OPFS
// ===============================================================

fn build_opfs_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
    opfs_buttons: &Rc<RefCell<Vec<web_sys::Element>>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("3. OPFS (Origin Private File System)"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "OPFS — настоящая файловая система в браузере. Доступна \
         через navigator.storage.getDirectory(). Идеальна для \
         больших бинарных файлов, которые не хочется грузить \
         в память целиком.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let status = dom::create_element("div")?;
    status.set_attribute(
        "style",
        "margin: 8px 0; padding: 8px 12px; background: #fafafa; \
         border-radius: 4px; font-family: monospace; \
         white-space: pre-line;",
    )?;
    status.set_text_content(Some(
        "⏳ Ожидаем проверку доступности OPFS...",
    ));
    dom::append(&section, &status)?;

    let write_btn = create_button("Записать 1 МБ")?;
    let read_btn = create_button("Прочитать")?;
    let size_btn = create_button("Размер файла")?;
    let delete_btn = create_button("Удалить файл")?;

    for btn in [&write_btn, &read_btn, &size_btn, &delete_btn] {
        btn.set_attribute("disabled", "").ok();
        opfs_buttons.borrow_mut().push(btn.clone());
    }

    dom::append(&section, &write_btn)?;
    dom::append(&section, &read_btn)?;
    dom::append(&section, &size_btn)?;
    dom::append(&section, &delete_btn)?;

    let result = make_result(&section)?;
    dom::append(parent, &section)?;

    {
        let result_c = result.clone();
        let status_c = status.clone();
        callbacks::listen(
            &write_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                let status_c = status_c.clone();
                result_c.set_text_content(Some("Записываем 1 МБ..."));
                spawn_local(async move {
                    let mut bytes: Vec<u8> = Vec::with_capacity(OPFS_SIZE);
                    for i in 0..OPFS_SIZE {
                        bytes.push((i % 256) as u8);
                    }
                    match opfs::write_file(OPFS_NAME, &bytes).await {
                        Ok(()) => {
                            let msg = format!(
                                "✅ Файл записан в OPFS\n\n\
                                 Имя: {OPFS_NAME}\n\
                                 Размер: {} байт",
                                bytes.len(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                            status_c.set_text_content(Some(
                                "OPFS: файл записан, готов к чтению.",
                            ));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ write_file: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &read_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match opfs::read_file(OPFS_NAME).await {
                        Ok(bytes) => {
                            let hex = format_hex_preview(&bytes, 16);
                            let msg = format!(
                                "✅ Прочитано из OPFS\n\n\
                                 Размер: {} байт\n\
                                 Первые байты: {hex}",
                                bytes.len(),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ read_file: {}\n\n\
                                 Возможно, файл не создан. \
                                 Нажмите «Записать 1 МБ».",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &size_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                spawn_local(async move {
                    match opfs::file_size(OPFS_NAME).await {
                        Ok(size) => {
                            let msg = format!(
                                "Размер файла {OPFS_NAME}: {} байт ({})",
                                size as u64,
                                file_api::format_file_size(size),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ file_size: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        let status_c = status.clone();
        callbacks::listen(
            &delete_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                let status_c = status_c.clone();
                spawn_local(async move {
                    match opfs::remove_file(OPFS_NAME).await {
                        Ok(()) => {
                            let msg = format!(
                                "🗑 Файл {OPFS_NAME} удалён из OPFS \
                                 (идемпотентно)",
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                            status_c.set_text_content(Some(
                                "OPFS: файла нет.",
                            ));
                        }
                        Err(e) => {
                            let msg = format!(
                                "❌ remove_file: {}",
                                format_js_error(&e),
                            );
                            result_c.set_text_content(Some(msg.as_str()));
                        }
                    }
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

// ===============================================================
// Секция 4: сквозной тест
// ===============================================================

fn build_cross_storage_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
    opfs_available: &Rc<Cell<Option<bool>>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let h2 = dom::create_element("h2")?;
    h2.set_text_content(Some("4. Сквозной тест: один Blob → хранилища"));
    dom::append(&section, &h2)?;

    let hint = dom::create_paragraph(
        "Сквозной сценарий: генерируем тестовый Blob, кладём его \
         во все доступные хранилища, читаем обратно и сравниваем \
         содержимое. Если OPFS недоступен — тест корректно его \
         пропустит, не выдавая ошибку.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let run_btn = create_button("Сквозной тест")?;
    let clear_btn = create_button("Очистить всё")?;

    dom::append(&section, &run_btn)?;
    dom::append(&section, &clear_btn)?;

    let result = make_result(&section)?;
    dom::append(parent, &section)?;

    {
        let result_c = result.clone();
        let opfs_available_c = opfs_available.clone();
        callbacks::listen(
            &run_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                let opfs_flag = opfs_available_c.get();
                result_c.set_text_content(Some("Выполняем сквозной тест..."));
                spawn_local(async move {
                    let opfs_ready = opfs_flag == Some(true);
                    let report = run_cross_storage_test(opfs_ready).await;
                    result_c.set_text_content(Some(report.as_str()));
                });
            },
            closures,
        )?;
    }

    {
        let result_c = result.clone();
        callbacks::listen(
            &clear_btn,
            "click",
            move |_ev: web_sys::Event| {
                let result_c = result_c.clone();
                result_c.set_text_content(Some("Очищаем..."));
                spawn_local(async move {
                    let mut lines: Vec<String> = Vec::new();
                    lines.push("Очистка всех хранилищ:".to_string());

                    match cache_storage::delete_cache(CACHE_NAME).await {
                        Ok(removed) => {
                            lines.push(format!(
                                "  Cache API: removed = {removed}"
                            ));
                        }
                        Err(e) => {
                            lines.push(format!(
                                "  Cache API: ❌ {}",
                                format_js_error(&e),
                            ));
                        }
                    }

                    match idb_store::delete_entry(IDB_KEY).await {
                        Ok(()) => lines.push(
                            "  IndexedDB: ✅ удалено".to_string()
                        ),
                        Err(e) => lines.push(format!(
                            "  IndexedDB: ❌ {}",
                            format_js_error(&e),
                        )),
                    }

                    match opfs::remove_file(OPFS_NAME).await {
                        Ok(()) => lines.push(
                            "  OPFS: ✅ удалено".to_string()
                        ),
                        Err(_) => lines.push(
                            "  OPFS: ⏭ пропущен (недоступен)".to_string()
                        ),
                    }

                    lines.push(String::new());
                    lines.push(
                        "Все операции идемпотентны: повторный клик \
                         вернёт те же значения."
                            .to_string(),
                    );

                    let text = lines.join("\n");
                    result_c.set_text_content(Some(text.as_str()));
                });
            },
            closures,
        )?;
    }

    return Ok(());
}

async fn run_cross_storage_test(opfs_ready: bool) -> String {
    let timestamp = js_sys::Date::now() as u64;
    let content = format!(
        "Cross-storage test\n\
         Timestamp: {timestamp}\n\
         Payload: ABCDEFGHIJKLMNOPQRSTUVWXYZ\n",
    );
    let bytes: Vec<u8> = content.as_bytes().to_vec();

    let mut lines: Vec<String> = Vec::new();
    lines.push(format!("Тестовые данные: {} байт", bytes.len()));
    lines.push(String::new());

    let mut cache_ok = false;
    let mut idb_ok = false;
    let mut opfs_ok = false;

    match put_cache_and_get_bytes(&bytes).await {
        Ok(read_back) => {
            let eq = read_back == bytes;
            let mark: &str;
            if eq {
                mark = "✅";
                cache_ok = true;
            } else {
                mark = "❌";
            }
            lines.push(format!("{mark} Cache API: round-trip"));
        }
        Err(e) => {
            lines.push(format!(
                "❌ Cache API: {}",
                format_js_error(&e),
            ));
        }
    }

    match put_idb_and_get_bytes(&bytes).await {
        Ok(read_back) => {
            let eq = read_back == bytes;
            let mark: &str;
            if eq {
                mark = "✅";
                idb_ok = true;
            } else {
                mark = "❌";
            }
            lines.push(format!("{mark} IndexedDB: round-trip"));
        }
        Err(e) => {
            lines.push(format!(
                "❌ IndexedDB: {}",
                format_js_error(&e),
            ));
        }
    }

    if opfs_ready {
        match put_opfs_and_get_bytes(&bytes).await {
            Ok(read_back) => {
                let eq = read_back == bytes;
                let mark: &str;
                if eq {
                    mark = "✅";
                    opfs_ok = true;
                } else {
                    mark = "❌";
                }
                lines.push(format!("{mark} OPFS: round-trip"));
            }
            Err(e) => {
                lines.push(format!(
                    "❌ OPFS: {}",
                    format_js_error(&e),
                ));
            }
        }
    } else {
        lines.push(
            "⏭ OPFS: пропущен (недоступен в этом браузере)".to_string(),
        );
    }

    lines.push(String::new());

    if cache_ok && idb_ok && opfs_ok {
        lines.push(
            "🎉 Все три хранилища сосуществуют и дают идентичный \
             результат. Данные независимы, квота — общая."
                .to_string(),
        );
    } else if cache_ok && idb_ok && !opfs_ready {
        lines.push(
            "ℹ️ Cache API и IndexedDB работают корректно. OPFS \
             недоступен — это не ошибка, а ограничение браузера."
                .to_string(),
        );
    } else {
        lines.push(
            "⚠️ Часть хранилищ вернула ошибки. Проверьте \
             диагностику выше."
                .to_string(),
        );
    }

    return lines.join("\n");
}

async fn put_cache_and_get_bytes(bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let options = web_sys::BlobPropertyBag::new();
    options.set_type("application/octet-stream");
    let blob = web_sys::Blob::new_with_u8_array_sequence_and_options(
        &parts, &options,
    )?;

    cache_storage::put_blob(CACHE_NAME, CACHE_KEY, &blob).await?;

    let read_blob = cache_storage::get_blob(CACHE_NAME, CACHE_KEY).await?
        .ok_or_else(|| JsValue::from_str("cache: запись не найдена"))?;

    let array_promise = read_blob.array_buffer();
    let value = wasm_bindgen_futures::JsFuture::from(array_promise).await?;
    let array_buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("cache: не ArrayBuffer"))?;
    return Ok(js_sys::Uint8Array::new(&array_buffer).to_vec());
}

async fn put_idb_and_get_bytes(bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    let entry = idb_store::StoredEntry {
        id: IDB_KEY.to_string(),
        name: "cross-test.bin".to_string(),
        mime: "application/octet-stream".to_string(),
        bytes: bytes.to_vec(),
    };
    idb_store::put_entry(&entry).await?;
    let read = idb_store::get_entry(IDB_KEY).await?
        .ok_or_else(|| JsValue::from_str("idb: запись не найдена"))?;
    return Ok(read.bytes);
}

async fn put_opfs_and_get_bytes(bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    opfs::write_file(OPFS_NAME, bytes).await?;
    return opfs::read_file(OPFS_NAME).await;
}

// ===============================================================
// Определение поддержки API
// ===============================================================

fn cache_supported() -> bool {
    return web_sys::window()
        .map(|w| w.caches().is_ok())
        .unwrap_or(false);
}

fn idb_supported() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let w_js: &JsValue = window.as_ref();
    return js_sys::Reflect::get(w_js, &JsValue::from_str("indexedDB"))
        .map(|v| !v.is_undefined() && !v.is_null())
        .unwrap_or(false);
}

fn is_secure_context() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let protocol: String = window
        .location()
        .protocol()
        .unwrap_or_default();
    let host: String = window
        .location()
        .host()
        .unwrap_or_default();
    return protocol == "https:"
        || host.starts_with("localhost")
        || host.starts_with("127.0.0.1");
}

// ===============================================================
// Helpers
// ===============================================================

fn create_button(text: &str) -> Result<web_sys::Element, JsValue> {
    let btn = dom::create_element("button")?;
    btn.set_text_content(Some(text));
    return Ok(btn);
}

fn make_result(parent: &web_sys::Element) -> Result<web_sys::Element, JsValue> {
    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(нажмите кнопку)"));
    dom::append(parent, &result)?;
    return Ok(result);
}

fn format_hex_preview(bytes: &[u8], n: usize) -> String {
    let limit = n.min(bytes.len());
    let mut out = String::new();
    for (i, b) in bytes[..limit].iter().enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format!("{:02X}", b));
    }
    if bytes.len() > limit {
        out.push_str(" …");
    }
    return out;
}

/// Форматирует `JsValue`-ошибку в понятную строку.
///
/// Для `SecurityError` — браузер-специфичная подсказка.
fn format_js_error(e: &JsValue) -> String {
    let browser = detect_browser();

    if let Some(exc) = e.dyn_ref::<web_sys::DomException>() {
        let name = exc.name();

        if name == "SecurityError" {
            return format!(
                "SecurityError ({}): браузер запретил доступ к хранилищу.\n\n{}",
                browser.name(),
                browser.security_error_hint(),
            );
        }
        if name == "QuotaExceededError" {
            return "QuotaExceededError: квота хранилища исчерпана. \
                    Удалите ненужные данные."
                .to_string();
        }
        if name == "AbortError" {
            return "AbortError: операция отменена.".to_string();
        }
        if name == "NotFoundError" {
            return "NotFoundError: запись не найдена.".to_string();
        }
        return format!("{name}: {}", exc.message());
    }

    let raw = e.as_string().unwrap_or_else(|| format!("{e:?}"));
    let first_line = raw.lines().next().unwrap_or("");
    let trimmed: &str;
    if first_line.len() > 200 {
        trimmed = &first_line[..200];
    } else {
        trimmed = first_line;
    }
    return trimmed.to_string();
}
```

---

## Команды запуска демо

```bash
npm run start
```

Открыть `http://localhost:5173`.

**Что проверить в браузере:**

1. **Диагностика:** Protocol, Host, **Browser**, secure context, статус Cache/IDB/OPFS.
2. **Квота:** нажать «Обновить» — увидеть `usage` / `quota`.
3. **Cache API:** «Сохранить тестовый Blob» → «Прочитать» → «Список ключей» → «Удалить всё».
4. **IndexedDB:** «Сохранить запись» → «Прочитать» → «Список записей» → «Удалить».
5. **OPFS:** кнопки активны, только если OPFS доступен. «Записать 1 МБ» → «Прочитать» → «Размер файла» → «Удалить файл».
6. **Сквозной тест:** один Blob → все доступные хранилища → сравнение.

---

## Что увидит пользователь

### Chrome на localhost — всё работает

```
Protocol: http:
Host: localhost:5173
Browser: Chrome
Secure context: ✅
Cache API: ✅ доступен
IndexedDB: ✅ доступен

✅ OPFS доступен: файловые операции работают.

Использовано: 1.02 MB (0.00%)
Квота: 210.00 GB
Свободно: 209.99 GB
```

### Firefox с заблокированным site data

```
Protocol: http:
Host: localhost:5173
Browser: Firefox
Secure context: ✅
Cache API: ✅ доступен
IndexedDB: ✅ доступен

⚠️ OPFS недоступен: браузер запретил хранение данных (SecurityError).

Firefox → Settings → Privacy & Security → Cookies and Site Data.
Убедитесь, что вы не в приватном режиме: в Firefox OPFS недоступен
в Private Browsing и при опции «Never remember history».
```

Кнопки OPFS — **disabled**. Сквозной тест покажет:

```
Тестовые данные: 80 байт

✅ Cache API: round-trip
✅ IndexedDB: round-trip
⏭ OPFS: пропущен (недоступен в этом браузере)

ℹ️ Cache API и IndexedDB работают корректно. OPFS недоступен —
это не ошибка, а ограничение браузера.
```

---

## Итоги подглавы 18.17

- **Cache API** — хранит пары «URL → Response». Ключ — валидный URL (`https://…` или `/path`). Работает с `Blob` в теле.
- **IndexedDB** — структурированное хранилище. Crate `idb` работает через `JsValue`; сериализация через `serde_wasm_bindgen`. `store.get()` → `Option<JsValue>`, `store.get_all()` → `Vec<JsValue>`.
- **OPFS** — файловая система в браузере. Методы вызываются через `Reflect::get` — не зависим от версии web-sys.
- **Диагностика** — определяем браузер и даём специфичные подсказки для `SecurityError`. Проверяем доступность API **асинхронно** при старте.
- **Graceful degradation** — если API недоступен, кнопки дизейблятся, сквозной тест пропускает шаг, а не падает.
- **Квота обновляется автоматически** после операций.
- **Идемпотентность удаления** — универсальное правило для всех трёх хранилищ.

## Замечания на будущее

**Правило проекта, зафиксированное в этой подглаве:**

1. **Диагностика должна знать контекст.** Одно и то же «браузер запретил хранение» требует разных путей в Chrome, Firefox, Safari. Подсказка «Chrome → Settings → …» в Firefox вводит в заблуждение. Определяйте `navigator.userAgent` (в порядке Edge → Firefox → Chrome → Safari) и подставляйте соответствующую инструкцию.

2. **Ошибки API типизируйте.** Пока ошибка — `OpfsError::SecurityError`, UI может подставить любую подсказку в зависимости от браузера. Строка `"SecurityError"` такой гибкости не даёт.

3. **Проверка доступности — асинхронная и реальная.** Наличие метода не гарантирует работу. Реальная проверка — вызов метода и обработка `Result`. Результат публикуйте в UI, пока проверка не завершена — блокируйте зависимые кнопки.

4. **`idb` работает с `JsValue`.** Все данные сериализуются через `serde_wasm_bindgen::to_value` при записи и `from_value` при чтении. Типы возврата в `idb 0.6.5`: `get` → `Option<JsValue>`, `get_all` → `Vec<JsValue>`.

5. **Ключ Cache API — валидный URL.** Относительный путь (`/path`) автоматически привязывается к текущему origin. Произвольные схемы (`test://…`) отклоняются.

6. **Методы web-sys вызывайте через Reflect, если сигнатуры отличаются от версии к версии.** `show_open_file_picker_with_options`, `getFileHandle`, `createWritable` — все три в 0.3.105 требуют обхода через `js_sys::Reflect::get` + `Function::call*`.

7. **Правило про `let`-биндинг до первого использования.** Если компилятор говорит `cannot find value X`, проверьте, что `let X = ...;` есть **в той же или внешней** области видимости. Для `match`-веток объявляйте `let` внутри каждой ветки, если переменная нужна только там.

8. **Никогда не берите ссылку от временного значения.** `window.navigator().as_ref()` даёт висячую ссылку. Сначала `let navigator = window.navigator();`, потом `navigator.as_ref()`. То же касается `window.location()`, `window.document()`, `window.crypto()`, `window.performance()`.