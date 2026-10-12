# Глава 18. File API

## 18.14. Прогресс, отмена и интеллектуальная отправка

В подглаве 18.13 мы отправляли файл через `fetch` — просто, но с двумя принципиальными ограничениями: **не было прогресса отправки** и **не было способа отменить запрос в любой момент**. Оба ограничения — свойства `fetch`. В этой подглаве мы закроем их и пойдём дальше: построим **интеллектуальный помощник**, который сам выбирает оптимальный способ отправки для конкретного файла и сети, а также **систему управления загрузками**, где пользователь оперирует только именами файлов.

**Три способа — три компромисса:**

| Способ | Сильные стороны | Слабые стороны |
|---|---|---|
| `fetch + FormData` | Простота, `async/await` | Нет прогресса отправки |
| `XMLHttpRequest` | `upload.onprogress`, отмена | Callback-стиль |
| Chunked upload | Докачка, любые размеры | Сложность, состояние на сервере |

### Хронология проблем и решений

Эта подглава потребовала нескольких итераций — и каждая выявила важный урок.

1. **Первая версия: in-memory chunked upload.** Сервер держал чанки в `HashMap<String, Vec<u8>>`. Загрузка 4-гигабайтного файла съедала 8 ГБ RAM и убивала WSL2 через OOM-killer.

2. **Дисковое хранилище.** Чанки пишутся в файлы `{temp_dir}/hello-wasm-uploads/{upload_id}/{index:08}.part`. Память — O(chunk_size). Пик — ~16 КБ на любом размере. **Лимитов по размеру больше нет.**

3. **Проблема с `Content-Type`.** При отправке `ArrayBuffer` через `fetch` браузер не устанавливает `Content-Type`. Axum-экстрактор `Bytes` требует `application/octet-stream` и отвечал `415`. Фикс: явный заголовок для каждого запроса с телом.

4. **Проблема с abort-гонкой.** `xhr.abort()`, вызванный до подписки на события, оставлял промис зависшим. Причина — `readyState` может стать `UNSENT` или `DONE`, и мы проверяли только `DONE`. Фикс: **subscribe-then-check** — сначала подписка, потом проверка `readyState` на **оба** терминальных состояния.

5. **Проблема с одиночными загрузками.** Файлы, отправленные через `fetch`/`XHR`, не появлялись в списке, потому что серверный обработчик `/api/upload` не сохранял их. Фикс: **унифицированное хранилище** — все способы сохраняют в один каталог.

6. **Проблема с UI.** Пользователь не понимал, что такое `upload_id`. Фикс: показать **только имена файлов**, кнопка «Удалить» рядом с каждым.

**Всё это отражено в коде ниже.**

---

## Обновлённая структура файлов

```
hello-wasm02/                       # клиентский crate
├── src/
│   ├── lib.rs                     # +upload_progress, +chunked_upload, +upload_strategy
│   ├── app.rs                     # только секция 18.14
│   ├── upload_progress.rs         # XHR с прогрессом, отменой, subscribe-then-check
│   ├── chunked_upload.rs          # chunked upload + список + удаление
│   ├── upload_strategy.rs         # интеллектуальный выбор метода
│   ├── file_upload.rs             # из 18.13 (стратегия Fetch)
│   └── … остальные модули
├── auth-server/
│   ├── Cargo.toml                 # tokio +features: time, fs, io-util
│   └── src/
│       ├── main.rs                # +chunk-эндпоинты, +TTL-задача
│       ├── handlers.rs            # disk-backed upload + list + delete
│       └── auth.rs                # без изменений
└── tests/
    ├── wasm_upload_progress.rs    # тесты XHR
    └── wasm_chunked_upload.rs     # тесты chunked + list + lifecycle
```

**Расположение файлов на сервере:**

```
$TMPDIR/hello-wasm-uploads/
├── upload-1734567890-123456789/
│   └── myfile.pdf              # собранный файл
└── upload-1734567890-987654321/
    └── video.mp4
```

`std::env::temp_dir()` даёт `/tmp` в Linux/macOS и `%TEMP%` в Windows. В WSL2 — `/tmp` внутри VM, очищается при перезапуске.

---

## Ссылки для детального изучения

- **MDN: XMLHttpRequest** — https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
- **MDN: XMLHttpRequest.upload** — https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/upload
- **MDN: ProgressEvent** — https://developer.mozilla.org/en-US/docs/Web/API/ProgressEvent
- **MDN: Network Information API** — https://developer.mozilla.org/en-US/docs/Web/API/Network_Information_API
- **MDN: AbortController** — https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- **TUS — Resumable upload protocol** — https://tus.io/protocols/resumable-upload
- **AWS S3 multipart upload** — https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- **tokio::fs** — https://docs.rs/tokio/latest/tokio/fs/index.html

---

## Содержание файлов

### 1. `auth-server/Cargo.toml`

```toml
[dependencies]
axum = { version = "0.8.9", features = ["multipart"] }
tokio = { version = "1", features = [
    "macros",
    "rt-multi-thread",
    "time",
    "fs",
    "io-util",     # AsyncReadExt / AsyncWriteExt
] }
tower-http = { version = "0.7.1", features = ["cors"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
chrono = { version = "0.4", features = ["serde"] }
dotenvy = "0.15"
tokio-stream = "0.1"
futures = "0.3"
async-stream = "0.3"
sha2 = "0.10"
hex = "0.4"
```

---

### 2. `auth-server/src/handlers.rs` — секция 18.14

**Импорты в шапке файла** должны включать `use std::time::Duration;` и `use serde::Deserialize;`.

**Секция 18.14** (добавьте в конец файла):

```rust
// ============================================================
// 18.14: Chunked upload и управление загрузками
// ============================================================

use std::collections::{HashMap, HashSet};
use std::path::PathBuf;
use std::sync::Arc;
use std::time::Duration;

use axum::body::Bytes;
use axum::extract::{Path, Query};
use tokio::sync::Mutex;

/// Хранилище метаданных активных загрузок.
///
/// **В памяти — только метаданные.** Сами байты — на диске.
/// Память O(number_of_uploads), не O(size).
pub type ChunkStore = Arc<Mutex<HashMap<String, UploadState>>>;

pub struct UploadState {
    pub filename: String,
    pub total_size: usize,
    pub total_chunks: usize,
    pub dir: PathBuf,
    pub received: HashSet<usize>,
}

/// Корневая директория для всех загрузок.
pub fn uploads_root() -> PathBuf {
    return std::env::temp_dir().join("hello-wasm-uploads");
}

/// Проверяет, что `upload_id` безопасен как имя директории.
fn is_safe_id(id: &str) -> bool {
    !id.is_empty()
        && id.len() <= 64
        && id.chars().all(|c| {
            c.is_ascii_alphanumeric() || c == '-' || c == '_'
        })
}

/// Генерирует уникальный upload_id.
fn generate_upload_id() -> String {
    use std::sync::atomic::{AtomicU64, Ordering};
    static COUNTER: AtomicU64 = AtomicU64::new(0);
    let n = COUNTER.fetch_add(1, Ordering::SeqCst);
    let ts = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .map(|d| d.as_millis() as u64)
        .unwrap_or(0);
    return format!("upload-{ts}-{n}");
}

#[derive(Deserialize)]
pub struct ChunkQuery {
    pub upload_id: String,
    pub chunk_index: usize,
    pub total_chunks: usize,
    pub filename: String,
    pub total_size: usize,
}

/// Принимает один чанк файла и записывает его на диск.
pub async fn upload_chunk(
    axum::extract::Extension(store): axum::extract::Extension<ChunkStore>,
    Query(q): Query<ChunkQuery>,
    body: Bytes,
) -> Result<Json<Value>, StatusCode> {
    if !is_safe_id(&q.upload_id) {
        return Err(StatusCode::BAD_REQUEST);
    }
    if q.chunk_index >= q.total_chunks {
        return Err(StatusCode::BAD_REQUEST);
    }

    let dir = uploads_root().join(&q.upload_id);

    {
        let mut guard = store.lock().await;
        let state = guard.entry(q.upload_id.clone()).or_insert_with(|| {
            UploadState {
                filename: q.filename.clone(),
                total_size: q.total_size,
                total_chunks: q.total_chunks,
                dir: dir.clone(),
                received: HashSet::new(),
            }
        });
        if state.total_chunks != q.total_chunks
            || state.total_size != q.total_size
            || state.filename != q.filename
        {
            return Err(StatusCode::BAD_REQUEST);
        }
    }

    tokio::fs::create_dir_all(&dir)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    let path = dir.join(format!("{:08}.part", q.chunk_index));
    tokio::fs::write(&path, &body)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let received = {
        let mut guard = store.lock().await;
        let state = guard
            .get_mut(&q.upload_id)
            .ok_or(StatusCode::INTERNAL_SERVER_ERROR)?;
        state.received.insert(q.chunk_index);
        state.received.len()
    };

    Ok(Json(json!({ "received": received })))
}

#[derive(Deserialize)]
pub struct FinalizeRequest {
    pub upload_id: String,
}

/// Финализирует загрузку: склеивает чанки потоково, считает SHA-256.
pub async fn finalize_upload(
    axum::extract::Extension(store): axum::extract::Extension<ChunkStore>,
    Json(req): Json<FinalizeRequest>,
) -> Result<Json<Value>, StatusCode> {
    if !is_safe_id(&req.upload_id) {
        return Err(StatusCode::BAD_REQUEST);
    }

    let state = {
        let mut guard = store.lock().await;
        guard.remove(&req.upload_id).ok_or(StatusCode::NOT_FOUND)?
    };

    if state.received.len() != state.total_chunks {
        return Err(StatusCode::BAD_REQUEST);
    }

    let output = state.dir.join(&state.filename);
    let mut out = tokio::fs::File::create(&output)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let mut hasher = Sha256::new();
    let mut total_written: usize = 0;

    for i in 0..state.total_chunks {
        let chunk_path = state.dir.join(format!("{:08}.part", i));
        let mut f = tokio::fs::File::open(&chunk_path)
            .await
            .map_err(|_| StatusCode::BAD_REQUEST)?;

        let mut buf = [0u8; 8192];
        loop {
            let n = tokio::io::AsyncReadExt::read(&mut f, &mut buf)
                .await
                .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
            if n == 0 { break; }
            hasher.update(&buf[..n]);
            tokio::io::AsyncWriteExt::write_all(&mut out, &buf[..n])
                .await
                .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
            total_written += n;
        }
    }
    tokio::io::AsyncWriteExt::flush(&mut out)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    drop(out);

    // Убираем .part-файлы.
    for i in 0..state.total_chunks {
        let path = state.dir.join(format!("{:08}.part", i));
        tokio::fs::remove_file(&path).await.ok();
    }

    if total_written != state.total_size {
        return Err(StatusCode::BAD_REQUEST);
    }

    let sha256_hex = hex::encode(hasher.finalize());

    Ok(Json(json!({
        "status": "ok",
        "filename": state.filename,
        "size": total_written,
        "chunks": state.total_chunks,
        "sha256": sha256_hex,
    })))
}

/// Возвращает метаданные одной загрузки.
pub async fn upload_status(
    Path(upload_id): Path<String>,
) -> Result<Json<Value>, StatusCode> {
    if !is_safe_id(&upload_id) {
        return Err(StatusCode::BAD_REQUEST);
    }

    let dir = uploads_root().join(&upload_id);
    if !dir.exists() {
        return Err(StatusCode::NOT_FOUND);
    }

    let mut entries = tokio::fs::read_dir(&dir)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let mut files: Vec<String> = Vec::new();
    let mut total_size: u64 = 0;
    while let Some(e) = entries
        .next_entry()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
    {
        files.push(e.file_name().to_string_lossy().to_string());
        if let Ok(meta) = e.metadata().await {
            total_size += meta.len();
        }
    }

    Ok(Json(json!({
        "upload_id": upload_id,
        "files": files,
        "total_size": total_size,
    })))
}

/// Возвращает список всех финализированных загрузок.
///
/// Для каждой директории в `uploads_root()` ищет итоговый файл
/// (не `.part`). Возвращает `[{upload_id, filename, size}, ...]`,
/// отсортированный по имени файла.
pub async fn list_uploads() -> Result<Json<Value>, StatusCode> {
    let root = uploads_root();
    if !root.exists() {
        return Ok(Json(json!({ "uploads": [] })));
    }

    let mut entries = tokio::fs::read_dir(&root)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let mut uploads: Vec<Value> = Vec::new();

    while let Some(entry) = entries
        .next_entry()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
    {
        let upload_id = entry.file_name().to_string_lossy().to_string();
        let dir = entry.path();

        let mut inner = match tokio::fs::read_dir(&dir).await {
            Ok(r) => r,
            Err(_) => continue,
        };

        while let Some(item) = inner.next_entry().await.ok().flatten() {
            let name = item.file_name().to_string_lossy().to_string();
            if name.ends_with(".part") {
                continue;
            }
            if let Ok(meta) = item.metadata().await {
                uploads.push(json!({
                    "upload_id": upload_id,
                    "filename": name,
                    "size": meta.len(),
                }));
            }
            break;
        }
    }

    uploads.sort_by(|a, b| {
        let na = a.get("filename").and_then(|v| v.as_str()).unwrap_or("");
        let nb = b.get("filename").and_then(|v| v.as_str()).unwrap_or("");
        na.cmp(nb)
    });

    return Ok(Json(json!({ "uploads": uploads })));
}

/// Удаляет загрузку: директорию со всем содержимым.
pub async fn delete_upload(
    axum::extract::Extension(store): axum::extract::Extension<ChunkStore>,
    Path(upload_id): Path<String>,
) -> Result<Json<Value>, StatusCode> {
    if !is_safe_id(&upload_id) {
        return Err(StatusCode::BAD_REQUEST);
    }

    {
        let mut guard = store.lock().await;
        guard.remove(&upload_id);
    }

    let dir = uploads_root().join(&upload_id);
    let existed = dir.exists();
    if existed {
        tokio::fs::remove_dir_all(&dir)
            .await
            .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    }

    Ok(Json(json!({
        "status": "ok",
        "upload_id": upload_id,
        "removed": existed,
    })))
}

// ============================================================
// TTL-очистка
// ============================================================

pub async fn cleanup_old_uploads(ttl: Duration) {
    let root = uploads_root();
    loop {
        tokio::time::sleep(Duration::from_secs(3600)).await;
        let _ = cleanup_once(&root, ttl).await;
    }
}

async fn cleanup_once(
    root: &std::path::Path,
    ttl: Duration,
) -> std::io::Result<()> {
    if !root.exists() {
        return Ok(());
    }
    let now = std::time::SystemTime::now();
    let mut entries = tokio::fs::read_dir(root).await?;
    while let Some(entry) = entries.next_entry().await? {
        let path = entry.path();
        if let Ok(metadata) = entry.metadata().await {
            if let Ok(modified) = metadata.modified() {
                if let Ok(age) = now.duration_since(modified) {
                    if age > ttl {
                        tokio::fs::remove_dir_all(&path).await.ok();
                    }
                }
            }
        }
    }
    Ok(())
}
```

**И секция 18.13 `upload` — тоже disk-backed:**

```rust
/// Принимает файл через multipart/form-data и сохраняет на диск.
///
/// **Унифицированное хранилище.** Файл сохраняется в тот же каталог
/// `uploads_root()/{upload_id}/{filename}`, что использует chunked
/// upload. Все загрузки — fetch, XHR, chunked — попадают в общий
/// список `/api/upload/chunk/list` и могут быть удалены через
/// `/api/upload/chunk/{upload_id}`.
pub async fn upload(mut multipart: Multipart) -> Result<Json<Value>, StatusCode> {
    let mut file_name = String::from("unnamed");
    let mut content_type = String::from("application/octet-stream");
    let mut size: usize = 0;
    let mut hasher = Sha256::new();
    let mut found_file = false;

    let upload_id = generate_upload_id();
    let dir = uploads_root().join(&upload_id);

    loop {
        let field_opt = multipart
            .next_field()
            .await
            .map_err(|_| StatusCode::BAD_REQUEST)?;
        let field = match field_opt {
            Some(f) => f,
            None => break,
        };

        let name = field.name().map(String::from).unwrap_or_default();
        if name != "file" { continue; }
        found_file = true;

        if let Some(fname) = field.file_name() {
            file_name = fname.to_string();
        }
        if let Some(ct) = field.content_type() {
            content_type = ct.to_string();
        }

        tokio::fs::create_dir_all(&dir)
            .await
            .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
        let output_path = dir.join(&file_name);
        let mut out_file = tokio::fs::File::create(&output_path)
            .await
            .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

        let mut field = field;
        loop {
            let chunk_opt = field
                .chunk()
                .await
                .map_err(|_| StatusCode::BAD_REQUEST)?;
            let chunk = match chunk_opt {
                Some(c) => c,
                None => break,
            };
            size += chunk.len();
            hasher.update(&chunk);
            tokio::io::AsyncWriteExt::write_all(&mut out_file, &chunk)
                .await
                .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
        }
        tokio::io::AsyncWriteExt::flush(&mut out_file)
            .await
            .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    }

    if !found_file {
        let _ = tokio::fs::remove_dir_all(&dir).await;
        return Err(StatusCode::BAD_REQUEST);
    }

    let sha256_hex = hex::encode(hasher.finalize());

    Ok(Json(json!({
        "status": "ok",
        "upload_id": upload_id,
        "filename": file_name,
        "size": size,
        "content_type": content_type,
        "sha256": sha256_hex,
    })))
}
```

---

### 3. `auth-server/src/main.rs`

```rust
mod auth;
mod handlers;

use std::collections::HashMap;
use std::sync::Arc;
use std::time::Duration;

use tokio::sync::Mutex;

use axum::{
    extract::{DefaultBodyLimit, Extension},
    routing::{get, post},
    Router,
};
use tower_http::cors::CorsLayer;

#[tokio::main(flavor = "multi_thread", worker_threads = 4)]
async fn main() {
    dotenvy::dotenv().ok();

    let chunk_store: handlers::ChunkStore =
        Arc::new(Mutex::new(HashMap::new()));

    // Фоновая TTL-очистка: раз в час удаляет загрузки старше 24 часов.
    tokio::spawn(handlers::cleanup_old_uploads(Duration::from_secs(
        24 * 3600,
    )));

    let app = Router::new()
        // … все существующие маршруты …
        .route(
            "/api/upload",
            post(handlers::upload).layer(DefaultBodyLimit::disable()),
        )
        .route(
            "/api/upload/chunk",
            post(handlers::upload_chunk).layer(DefaultBodyLimit::disable()),
        )
        .route(
            "/api/upload/chunk/finalize",
            post(handlers::finalize_upload),
        )
        .route(
            "/api/upload/chunk/list",
            get(handlers::list_uploads),
        )
        .route(
            "/api/upload/chunk/{upload_id}",
            get(handlers::upload_status).delete(handlers::delete_upload),
        )
        .layer(Extension(chunk_store))
        .layer(CorsLayer::permissive());

    let addr = std::env::var("BIND_ADDRESS").unwrap_or("127.0.0.1:3000".into());
    println!("Auth server listening on {addr}");
    let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**Важно:** `/list` и `/finalize` — статические, объявлены **до** `/{upload_id}`. В axum 0.8 статические маршруты имеют приоритет, но лучше явно.

---

### 4. `src/upload_progress.rs`

```rust
//! XHR-загрузка с прогрессом и отменой.
//!
//! **Subscribe-then-check.** XHR может завершиться (или быть
//! отменённым) до подписки на события. Порядок критичен:
//! подписаться → проверить `ready_state`. Если запрос уже
//! в терминальном состоянии (`DONE` или `UNSENT`), обрабатываем
//! вручную — иначе промис зависнет.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{File, FormData, ProgressEvent, XmlHttpRequest};

pub fn start_upload<F>(
    file: &File,
    url: &str,
    mut on_progress: F,
) -> Result<XmlHttpRequest, JsValue>
where
    F: FnMut(f64, f64) + 'static,
{
    let xhr = XmlHttpRequest::new()?;
    xhr.open("POST", url)?;

    let on_progress_cb = Closure::wrap(Box::new(move |e: ProgressEvent| {
        if e.length_computable() {
            on_progress(e.loaded(), e.total());
        } else {
            on_progress(e.loaded(), 0.0);
        }
    }) as Box<dyn FnMut(ProgressEvent)>);
    xhr.upload()?
        .set_onprogress(Some(on_progress_cb.as_ref().unchecked_ref()));
    on_progress_cb.forget();

    let form_data = FormData::new()?;
    form_data.append_with_blob("file", file)?;
    xhr.send_with_opt_form_data(Some(&form_data))?;

    return Ok(xhr);
}

pub async fn await_finish(xhr: &XmlHttpRequest) -> Result<String, JsValue> {
    let xhr_c = xhr.clone();
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();

        let xhr_for_load = xhr_c.clone();
        let on_load = Closure::once_into_js(move || {
            let text: String = xhr_for_load
                .response_text()
                .ok()
                .flatten()
                .unwrap_or_default();
            let _ = resolve_c.call1(&JsValue::NULL, &JsValue::from_str(&text));
        });

        let reject_c2 = reject_c.clone();
        let on_error = Closure::once_into_js(move || {
            let _ = reject_c2.call1(&JsValue::NULL, &JsValue::from_str("network error"));
        });

        let reject_c3 = reject_c.clone();
        let on_abort = Closure::once_into_js(move || {
            let _ = reject_c3.call1(&JsValue::NULL, &JsValue::from_str("aborted"));
        });

        xhr_c.set_onload(Some(on_load.unchecked_ref()));
        xhr_c.set_onerror(Some(on_error.unchecked_ref()));
        xhr_c.set_onabort(Some(on_abort.unchecked_ref()));

        // Проверка после подписки — оба терминальных состояния.
        let state = xhr_c.ready_state();
        let terminal = state == XmlHttpRequest::DONE
            || state == XmlHttpRequest::UNSENT;
        if terminal {
            let status = xhr_c.status().unwrap_or(0);
            if status == 0 {
                let _ = reject.call1(&JsValue::NULL, &JsValue::from_str("aborted"));
            } else {
                let text: String = xhr_c
                    .response_text()
                    .ok()
                    .flatten()
                    .unwrap_or_default();
                let _ = resolve.call1(&JsValue::NULL, &JsValue::from_str(&text));
            }
        }
    });
    let value = JsFuture::from(promise).await?;
    return value
        .as_string()
        .ok_or_else(|| JsValue::from_str("ответ не строка"));
}

pub fn is_abort_error(e: &JsValue) -> bool {
    if let Some(s) = e.as_string() {
        return s == "aborted";
    }
    return false;
}

pub async fn upload_with_progress<F>(
    file: &File,
    url: &str,
    on_progress: F,
) -> Result<String, JsValue>
where
    F: FnMut(f64, f64) + 'static,
{
    let xhr = start_upload(file, url, on_progress)?;
    return await_finish(&xhr).await;
}
```

---

### 5. `src/chunked_upload.rs`

```rust
//! Chunked upload: файл режется через `Blob.slice`, чанки уходят
//! через fetch, прогресс — по завершённым чанкам.
//!
//! **Content-Type обязателен.** axum 0.8 строго проверяет:
//! `Bytes` требует `application/octet-stream`, `Json` — `application/json`.
//! Браузер при отправке `ArrayBuffer` его не ставит — устанавливаем явно.

use std::cell::Cell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{AbortController, Blob, File, Request, RequestInit, Response};

pub const DEFAULT_CHUNK_SIZE: u32 = 1024 * 1024;
pub const CHUNK_TIMEOUT_MS: u32 = 30_000;

#[derive(Debug, Clone)]
pub struct UploadResult {
    pub upload_id: String,
    pub response: String,
}

#[derive(Debug, Clone)]
pub struct UploadInfo {
    pub upload_id: String,
    pub filename: String,
    pub size: f64,
}

pub fn chunk_count(size: f64, chunk_size: u32) -> u32 {
    if size <= 0.0 || chunk_size == 0 { return 0; }
    return (size / chunk_size as f64).ceil() as u32;
}

fn generate_upload_id() -> String {
    let ts = js_sys::Date::now() as u64;
    let r = (js_sys::Math::random() * 1_000_000_000.0) as u64;
    return format!("upload-{ts}-{r}");
}

pub async fn upload_chunked<F>(
    file: &File,
    base_url: &str,
    chunk_size: u32,
    cancel: Rc<Cell<bool>>,
    mut on_progress: F,
) -> Result<UploadResult, JsValue>
where
    F: FnMut(u32, u32) + 'static,
{
    let total_size = file.size();
    let total_chunks = chunk_count(total_size, chunk_size);
    if total_chunks == 0 {
        return Err(JsValue::from_str("пустой файл"));
    }

    let upload_id = generate_upload_id();
    let filename = file.name();

    for i in 0..total_chunks {
        if cancel.get() {
            return Err(JsValue::from_str("aborted"));
        }

        let start = (i as f64) * (chunk_size as f64);
        let end = (start + chunk_size as f64).min(total_size);
        let slice = file.slice_with_f64_and_f64(start, end)?;
        let bytes = blob_to_bytes(&slice).await?;

        let url = format!(
            "{base_url}?upload_id={upload_id}&chunk_index={i}\
             &total_chunks={total_chunks}&filename={filename}\
             &total_size={}",
            total_size as u64,
        );

        let _ = send_with_timeout(
            &url,
            &bytes,
            "POST",
            Some("application/octet-stream"),
            CHUNK_TIMEOUT_MS,
        )
        .await
        .map_err(|e| {
            JsValue::from_str(&format!("чанк {i} не отправлен: {e:?}"))
        })?;

        on_progress(i + 1, total_chunks);
    }

    let finalize_url = format!("{base_url}/finalize");
    let body_obj = js_sys::Object::new();
    js_sys::Reflect::set(
        &body_obj,
        &JsValue::from_str("upload_id"),
        &JsValue::from_str(&upload_id),
    )?;
    let body_json = js_sys::JSON::stringify(&body_obj.into())?;
    let body_str = body_json
        .as_string()
        .ok_or_else(|| JsValue::from_str("не stringify"))?;

    let response = send_with_timeout(
        &finalize_url,
        body_str.as_bytes(),
        "POST",
        Some("application/json"),
        CHUNK_TIMEOUT_MS,
    )
    .await?;

    return Ok(UploadResult { upload_id, response });
}

pub async fn delete_chunked_upload(
    base_url: &str,
    upload_id: &str,
) -> Result<String, JsValue> {
    let url = format!("{base_url}/{upload_id}");
    return send_with_timeout(&url, &[], "DELETE", None, CHUNK_TIMEOUT_MS)
        .await;
}

pub async fn get_chunked_upload_status(
    base_url: &str,
    upload_id: &str,
) -> Result<String, JsValue> {
    let url = format!("{base_url}/{upload_id}");
    return send_with_timeout(&url, &[], "GET", None, CHUNK_TIMEOUT_MS).await;
}

pub async fn list_chunked_uploads(
    base_url: &str,
) -> Result<Vec<UploadInfo>, JsValue> {
    let url = format!("{base_url}/list");
    let text = send_with_timeout(&url, &[], "GET", None, CHUNK_TIMEOUT_MS)
        .await?;

    let parsed: JsValue = js_sys::JSON::parse(&text)?;
    let uploads_js = js_sys::Reflect::get(
        &parsed,
        &JsValue::from_str("uploads"),
    )?;
    let arr = uploads_js
        .dyn_into::<js_sys::Array>()
        .map_err(|_| JsValue::from_str("uploads не массив"))?;

    let mut out: Vec<UploadInfo> = Vec::with_capacity(arr.length() as usize);
    for i in 0..arr.length() {
        let item = arr.get(i);
        let upload_id = js_sys::Reflect::get(
            &item, &JsValue::from_str("upload_id"))?
            .as_string()
            .ok_or_else(|| JsValue::from_str("нет upload_id"))?;
        let filename = js_sys::Reflect::get(
            &item, &JsValue::from_str("filename"))?
            .as_string()
            .ok_or_else(|| JsValue::from_str("нет filename"))?;
        let size = js_sys::Reflect::get(
            &item, &JsValue::from_str("size"))?
            .as_f64()
            .ok_or_else(|| JsValue::from_str("нет size"))?;
        out.push(UploadInfo { upload_id, filename, size });
    }
    return Ok(out);
}

async fn send_with_timeout(
    url: &str,
    body: &[u8],
    method: &str,
    content_type: Option<&str>,
    timeout_ms: u32,
) -> Result<String, JsValue> {
    let controller = AbortController::new()?;
    let signal = controller.signal();

    {
        let controller_c = controller.clone();
        spawn_local(async move {
            TimeoutFuture::new(timeout_ms).await;
            controller_c.abort();
        });
    }

    let opts = RequestInit::new();
    opts.set_method(method);
    opts.set_signal(Some(&signal));

    if let Some(ct) = content_type {
        let headers = web_sys::Headers::new()?;
        headers.set("Content-Type", ct)?;
        opts.set_headers(&headers);
    }

    if !body.is_empty() {
        let uint8 = js_sys::Uint8Array::from(body);
        opts.set_body(&uint8.buffer());
    }

    let request = Request::new_with_str_and_init(url, &opts)?;
    return send_and_read_text(&request).await;
}

async fn blob_to_bytes(blob: &Blob) -> Result<Vec<u8>, JsValue> {
    let promise = blob.array_buffer();
    let value = JsFuture::from(promise).await?;
    let array_buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    return Ok(js_sys::Uint8Array::new(&array_buffer).to_vec());
}

async fn send_and_read_text(request: &Request) -> Result<String, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let promise = window.fetch_with_request(request);
    let resp_value = JsFuture::from(promise).await?;
    let resp: Response = resp_value
        .dyn_into()
        .map_err(|_| JsValue::from_str("ответ не Response"))?;

    if !resp.ok() {
        return Err(JsValue::from_str(&format!(
            "HTTP {} {}", resp.status(), resp.status_text(),
        )));
    }
    let text_promise = resp.text()?;
    let text_value = JsFuture::from(text_promise).await?;
    return text_value
        .as_string()
        .ok_or_else(|| JsValue::from_str("ответ не строка"));
}
```

---

### 6. `src/upload_strategy.rs`

```rust
//! Интеллектуальный выбор способа отправки.
//!
//! Без лимитов по размеру — сервер пишет чанки на диск.

use wasm_bindgen::prelude::*;

pub const SIZE_FETCH_MAX: f64 = 5.0 * 1024.0 * 1024.0;
pub const SIZE_XHR_MAX: f64 = 100.0 * 1024.0 * 1024.0;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum UploadMethod { Fetch, Xhr, Chunked }

impl UploadMethod {
    pub fn label(&self) -> &'static str {
        return match self {
            UploadMethod::Fetch => "fetch + FormData",
            UploadMethod::Xhr => "XMLHttpRequest + прогресс",
            UploadMethod::Chunked => "Chunked upload (по 1 МБ)",
        };
    }
    pub fn tag(&self) -> &'static str {
        return match self {
            UploadMethod::Fetch => "fetch",
            UploadMethod::Xhr => "xhr",
            UploadMethod::Chunked => "chunked",
        };
    }
    pub fn from_tag(tag: &str) -> Option<Self> {
        return match tag {
            "fetch" => Some(UploadMethod::Fetch),
            "xhr" => Some(UploadMethod::Xhr),
            "chunked" => Some(UploadMethod::Chunked),
            _ => None,
        };
    }
}

#[derive(Debug, Clone)]
pub struct Strategy {
    pub method: UploadMethod,
    pub reason: String,
    pub connection_type: String,
    pub size_mb: f64,
}

pub fn detect_connection_type() -> String {
    let window = match web_sys::window() { Some(w) => w, None => return "unknown".into() };
    let navigator = window.navigator();
    let nav_js: &JsValue = navigator.as_ref();
    let conn = match js_sys::Reflect::get(nav_js, &JsValue::from_str("connection")) {
        Ok(v) => v, Err(_) => return "unknown".into(),
    };
    if conn.is_undefined() || conn.is_null() { return "unknown".into(); }
    return match js_sys::Reflect::get(&conn, &JsValue::from_str("effectiveType")) {
        Ok(v) => v.as_string().unwrap_or_else(|| "unknown".into()),
        Err(_) => "unknown".into(),
    };
}

fn is_slow_connection(c: &str) -> bool {
    return matches!(c, "slow-2g" | "2g");
}

pub fn pick_strategy(file_size: f64, connection: &str) -> Strategy {
    let size_mb = file_size / (1024.0 * 1024.0);

    if is_slow_connection(connection) {
        return Strategy {
            method: UploadMethod::Chunked,
            reason: format!(
                "Соединение медленное ({connection}) — chunked upload \
                 переживает обрывы и умеет докачку.",
            ),
            connection_type: connection.to_string(),
            size_mb,
        };
    }
    if file_size < SIZE_FETCH_MAX {
        return Strategy {
            method: UploadMethod::Fetch,
            reason: format!(
                "Файл маленький ({:.1} МБ) — прогресс не нужен, \
                 простой fetch быстрее и проще.", size_mb,
            ),
            connection_type: connection.to_string(),
            size_mb,
        };
    }
    if file_size < SIZE_XHR_MAX {
        return Strategy {
            method: UploadMethod::Xhr,
            reason: format!(
                "Файл средний ({:.1} МБ) — нужен прогресс отправки \
                 и возможность отмены.", size_mb,
            ),
            connection_type: connection.to_string(),
            size_mb,
        };
    }
    return Strategy {
        method: UploadMethod::Chunked,
        reason: format!(
            "Файл большой ({:.1} МБ) — chunked upload на диске \
             работает для любого размера.", size_mb,
        ),
        connection_type: connection.to_string(),
        size_mb,
    };
}
```

---

### 7. `src/app.rs` — краткая схема

Полный код — в предыдущем ответе. Ключевые моменты:

- **RAII-охрана `XhrSlot`** — очищает слот `current_xhr` при выходе из `spawn_local`, даже при раннем `return`.
- **Кнопка «Стоп»** дизейблится после клика, раздизейблится при новой загрузке.
- **При отмене** прогресс сбрасывается в 0, метка — «отменено».
- **Список загруженных файлов** обновляется после **любой** успешной загрузки (fetch, XHR, chunked).
- **Пользователь видит только имена файлов** — `upload_id` скрыт внутри замыканий.
- **Автообновление** — после успеха `refresh_uploads_list`.
- **Кнопка «Обновить»** — ручная синхронизация на случай внешних изменений.

---

## Команды запуска

```bash
# Сервер (отдельный терминал)
cd auth-server && cargo build && cd ..
npm run server

# Клиент
npm run start

# Тесты
wasm-pack test --headless --chrome --test wasm_upload_progress
wasm-pack test --headless --chrome --test wasm_chunked_upload
```

### Проверка через curl

```bash
# Multipart (fetch/XHR-стратегия)
echo "hello" > /tmp/test.txt
curl -X POST http://localhost:3000/api/upload -F "file=@/tmp/test.txt"
# → {"status":"ok","upload_id":"upload-...","filename":"test.txt",...}

# Список
curl http://localhost:3000/api/upload/chunk/list
# → {"uploads":[{"upload_id":"...","filename":"test.txt","size":6}]}

# Удаление
curl -X DELETE "http://localhost:3000/api/upload/chunk/upload-..."  # подставить ID
rm /tmp/test.txt
```

---

## Что увидит пользователь

**Отправка файла:**
1. Выбор файла.
2. Панель рекомендации объясняет выбор системы.
3. Кнопка «Отправить» → прогресс → результат.
4. **Файл появляется в списке «Загруженные файлы»** — независимо от того, fetch это, XHR или chunked.

**Управление загрузками:**
```
📄 dataset.csv  ·  47.68 MB                 [Удалить]
📄 video.mp4    ·  1.20 GB                  [Удалить]
📄 notes.txt    ·  2.00 KB                  [Удалить]
```

Одна кнопка рядом с каждым файлом. Клик «Удалить» — файл исчезает из списка. Никаких `upload_id`, никаких полей ввода.

**Отмена:**
1. Запуск XHR-загрузки большого файла.
2. Нажатие «■» на 30 %.
3. Прогресс сбрасывается в 0, метка «отменено», кнопка «Стоп» неактивна.
4. Новая загрузка — кнопка снова активна, прогресс с нуля.

---

## Итоги подглавы 18.14

Десять уроков, вынесенных из этой подглавы:

1. **Память пропорциональна размеру чанка, не файла.** In-memory chunked upload не масштабируется. Дисковое хранилище — единственный способ поддержать произвольные размеры. Пик — 16 КБ на любой файл.

2. **Унифицированное хранилище.** Если UI показывает общий список объектов, все источники (fetch, XHR, chunked) должны писать в **одно место**. Иначе пользователь видит неконсистентную картину.

3. **Content-Type обязателен для fetch с телом.** Браузер не ставит его для `ArrayBuffer`. Axum-экстракторы строги: `Bytes` требует `application/octet-stream`, `Json` — `application/json`.

4. **Subscribe-then-check для XHR.** Сначала подписки, потом проверка `ready_state`. И проверять **оба** терминальных состояния: `DONE` и `UNSENT`. Иначе abort-до-подписки оставит промис зависшим.

5. **RAII-охрана для общего mutable-состояния.** `XhrSlot::drop` очищает слот — даже при раннем `return`.

6. **Отмена сбрасывает UI.** Прогресс → 0, метка «отменено». Не оставляйте застывший «47 %» — это путает.

7. **Кнопка действия дизейблится после клика.** Снимается при старте новой операции. Предотвращает двойные клики.

8. **Пользователь работает с предметными понятиями.** Показывайте имена файлов, а не `upload_id`. Кнопка «Удалить» — рядом с именем.

9. **Идемпотентность DELETE.** Повторное удаление возвращает `removed: false`, а не 404. Упрощает клиент — retry не ломается.

10. **TTL-задача — страховка.** Раз в час удаляет загрузки старше 24 часов. Temp-директория даёт дополнительный уровень защиты при перезагрузке ОС.

## Замечания на будущее

**Правило проекта:** при проектировании API загрузки **сразу** решите, где хранятся файлы и как они попадают в UI. Изменение этого решения на лету приводит к неконсистентности: часть файлов есть, часть нет. Наш финальный вариант — единый каталог `uploads_root()/{upload_id}/{filename}`, все способы пишут туда.

**Второе правило:** `Option<T>` в hot loop — обычно лишний. Если переменная присваивается и читается в одной итерации — делайте её локальной. Обёртка добавляет `unwrap()` и шум.

**Третье правило:** тесты с abort не должны ждать ответа сервера. Если даёте запросу «начаться», отменяйте немедленно — иначе тест становится хрупким.

**Четвёртое правило:** держите `cargo build` без предупреждений. `unused_assignments`, `dead_code`, `unused_imports` накапливаются и маскируют реальные проблемы.

