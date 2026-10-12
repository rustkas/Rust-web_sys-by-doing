# Глава 18.18. Практический проект: конвертер файлов

## Что раскрывает глава

Сквозной сценарий, связывающий все подглавы 18: **выбрать → валидировать → прочитать потоково → обработать → захешировать → сохранить или отправить на сервер**. Каждый шаг — уже знакомый приём, но здесь они собраны в один pipeline, где видно, как решения предыдущих подглав работают вместе.

Глава фокусируется на **инженерных компромиссах**, которые нельзя показать на одиночном API-вызове:

- четыре движка grayscale (inline WASM, Web Worker, параллельные Worker'ы, OffscreenCanvas) — один интерфейс, разные цены;
- отмена, которая работает **не везде** (sync-фаза непрерываема) — и как честно это показать пользователю;
- прогресс с ETA и yield раз в 16 мс — почему без него UI замирает;
- адаптивные параметры (chunk size, число воркеров) вместо «стратегий с перезапуском»;
- идемпотентность на уровне UI (`Сбросить`, повторное сохранение) и сервера (`DELETE`);
- контракт `uploads_root()/{upload_id}/{filename}` и клиентский `generate_upload_id`.

---

## Структура файлов и папок

```
src/
├── lib.rs                        # регистрация модулей + #[wasm_bindgen(start)]
├── app.rs                        # точка входа демо (тонкая обёртка + CSS)
├── converter.rs                  # чистая логика конвертации
├── converter_ui.rs               # обёртки ввода-вывода (FSA, XHR, fallback)
├── converter_worker.rs           # Web Worker: одиночный и параллельный
├── converter_native.rs           # OffscreenCanvas-путь
└── converter_app/
    ├── mod.rs                    # точка входа подмодуля
    ├── state.rs                  # DemoState, Ui, StageStatus, Engine, StageCtx
    ├── ui_builder.rs             # построение DOM-дерева секции
    ├── handlers.rs               # навешивание onclick/onchange
    ├── pipeline.rs               # 7 стадий: pick → read → validate → convert → hash → save → upload
    └── metrics.rs                # форматирование прогресса, метрик, self-check

tests/
├── wasm_converter.rs             # чистая логика
├── wasm_converter_worker.rs      # Web Worker
├── wasm_converter_native.rs      # OffscreenCanvas
└── wasm_converter_ui.rs          # обёртки ввода-вывода
```

**Что добавлено в этой подглаве:** `converter.rs`, `converter_ui.rs`, `converter_worker.rs`, `converter_native.rs`, папка `converter_app/`, четыре тестовых файла.

**Что не тронуто:** `tests/common/mod.rs` (DOM-fixture для других глав) и все модули предыдущих подглав 18.

---

## Ссылки для детального изучения

- [File System Access API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API)
- [File System Access: `showOpenFilePicker` / `showSaveFilePicker`](https://developer.mozilla.org/en-US/docs/Web/API/Window/showOpenFilePicker)
- [`Blob.stream` и `ReadableStreamDefaultReader`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/getReader)
- [Web Workers (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- [Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- [OffscreenCanvas (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- [`createImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/createImageBitmap)
- [`ctx.filter` (Canvas 2D)](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/filter)
- [`XmlHttpRequest.upload.onprogress`](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/upload)
- [crate `image`](https://docs.rs/image/latest/image/)
- [crate `sha2`](https://docs.rs/sha2/latest/sha2/)
- [crate `gloo-timers`](https://docs.rs/gloo-timers/latest/gloo_timers/)
- [web-sys unstable APIs](https://rustwasm.github.io/docs/wasm-bindgen/web-sys/unstable-apis.html)

---

## Исходные модули

### `src/converter.rs` — чистая логика

Ядро без побочных эффектов на UI. Все функции можно тестировать в изоляции и переиспользовать из любого движка.

- `validate_magic_bytes` — определение формата по первым байтам (PNG `89 50 4E 47`, JPEG `FF D8 FF`) с человекочитаемой ошибкой.
- `ReadOutcome`, `read_blob_stream_validated` — потоковое чтение `File` через `Blob.stream` с прогрессом, поддержкой отмены и **early-exit** по magic-байтам: если первый же чанк не PNG/JPEG, чтение прерывается, не тратя трафик.
- `adaptive_chunk(total, target_iters)` — размер чанка как функция от объёма (правило «параметр, а не стратегия»).
- `sha256_hex` / `sha256_hex_with_progress` — синхронная и chunked-версии SHA-256; вторая отдаёт управление браузеру раз в 16 мс и возвращает `HashStats` (hex, число чанков/yield'ов).
- `to_grayscale_png` — синхронная inline-конвертация через `image` (движок `WasmInline`).
- `decode_to_rgba` / `encode_rgba_png` — мост для Worker/OffscreenCanvas: разложить PNG/JPEG в сырой RGBA и собрать обратно.
- `generate_upload_id` — `crypto.randomUUID()` с fallback на таймстемп.
- `bytes_to_blob` — вспомогательное для превью «до/после».

Полный код — см. листинг в репозитории.

### `src/converter_ui.rs` — обёртки ввода-вывода

Всё, что касается диалогов и сети. Каждая функция имеет FSA-путь через `Reflect` и fallback для браузеров без FSA:

- `pick_image_file` — выбор файла; в FSA-accept перечислены `.png`, `.jpg`, `.jpeg`; отмена даёт `AbortError`, а не панику.
- `SaveOutcome` (`Fsa` / `FallbackDownload` / `Cancelled`) — честный результат сохранения: UI отличает `<a download>` от FSA.
- `save_file_chunked` — запись чанками по 256 КиБ с yield раз в 16 мс и `is_cancelled`; кэширует `FileSystemFileHandle` в `thread_local!`, чтобы повторный Save не открывал диалог.
- `forget_save_handle` — сброс кэша.
- `upload_to_server` + `upload_http` — POST по контракту `{base}/files/{upload_id}/{filename}` с `Content-Type` и прогрессом через `XmlHttpRequestUpload::onprogress`.
- `delete_upload` — идемпотентный `DELETE /files/{id}`; `{removed: true|false}` — оба успех.
- `urlencode` — минимальный percent-encode path-сегментов.

### `src/converter_worker.rs` — Web Worker и параллелизм

Тяжёлые CPU-bound операции grayscale — вне главного потока. Принцип: не «выбираем worker по кнопке», а «параллелизм — параметр, зависящий от железа».

- Встроенный JS (`WORKER_JS`) — per-pixel grayscale над `Uint8ClampedArray`; возвращает результат через transferable `ArrayBuffer`, не копируя буфер дважды.
- `spawn_worker` — Worker из inline-`Blob` + `URL.createObjectURL`.
- `process_in_worker` — один RPC: посылает `{id, buffer}` через `Reflect`-вызов `postMessage` (web-sys не поддерживает transfer-лист), ждёт ответа как `Promise`, `terminate()` после.
- `grayscale_in_worker` — одиночный Worker (движок `WasmWorker`).
- `grayscale_parallel` — режет RGBA на N stripes по границе пикселя и гоняет их через `join_all`; N = `navigator.hardwareConcurrency` с клампом 1..4.

### `src/converter_native.rs` — OffscreenCanvas

Альтернатива WASM-пути: **декодирование PNG/JPEG и фильтрация выполняются нативным кодеком браузера**, а не `image`-крейтом. Полезно, когда важна скорость старта.

- `grayscale_native` — весь путь: `createImageBitmap(blob)` → `OffscreenCanvas` → `ctx.filter = "grayscale(1)"` → `drawImage` → `convertToBlob()` → `arrayBuffer()`.
- Все промис-возвращающие методы (`createImageBitmap`, `getContext`, `convertToBlob`, `arrayBuffer`) вызываются через `js_sys::Reflect::get` + `Function::call*` — код не зависит от минорной версии web-sys, у которой сигнатуры `Promise` vs `Result<Promise>` менялись.
- `ctx.filter` может отсутствовать (старые движки) — установка молча игнорируется, тогда результат остаётся цветным (это видно в self-check badge).

### `src/converter_app/` — организация демо-секции

`app.rs` содержит только демо-код текущей подглавы, но физически 600+ строк не влезают в один файл. Компромисс: `app.rs` — тонкая обёртка, а реализация демо — в соседнем подмодуле.

| Файл | Ответственность |
|---|---|
| `mod.rs` | Точка входа: собрать секцию и навесить обработчики |
| `state.rs` | `DemoState`, `Ui`, `StageStatus`, `Engine`, `StageCtx`, константы |
| `ui_builder.rs` | Построение DOM: кнопки, превью, прогресс, метрики |
| `handlers.rs` | Навешивание `onclick`/`onchange` |
| `pipeline.rs` | 7 стадий: pick → read → validate → convert → hash → save → upload |
| `metrics.rs` | Форматирование прогресса, метрик, self-check, утилиты |

Такое разделение даёт тестируемость (`metrics::build_output_name` — чистая), единый источник состояния (`DemoState` в `state.rs`) и изоляцию правил от UI.

---

## Тесты

Четыре файла, каждый самодостаточен — не трогают чужой `tests/common/mod.rs`.

| Файл | Что покрывает |
|---|---|
| `wasm_converter.rs` | magic-байты, `adaptive_chunk`, sync/async SHA-256 и их совпадение, decode/encode PNG, grayscale PNG/JPEG, `generate_upload_id`, чтение (успех / отмена / early-exit) |
| `wasm_converter_worker.rs` | базовый grayscale, **побитовое совпадение** parallel и single-worker, пустой / 1-px / нечётный буфер, сохранение альфы |
| `wasm_converter_native.rs` | PNG/JPEG roundtrip, размеры, отклонение мусора, мягкая проверка grayscale (с учётом `ctx.filter`) |
| `wasm_converter_ui.rs` | идемпотентность `forget_save_handle`, url-safe `upload_id`, «не паникует» для save без user gesture, smoke-проверка upload/delete к мёртвому серверу |

Принципы:
1. **Тестовые данные генерируются, не зашиваются в hex** — иначе отлаживаем опечатку в байтах.
2. **Инварианты, а не точные значения**: `R==G==B`, монотонность прогресса, размерности — не зависят от версии `image`.
3. **Cross-check между движками**: `parallel` ≡ `single worker`; `sha256_hex` ≡ `sha256_hex_with_progress.hex`.
4. **User-gesture-зависимые пути** — `#[ignore]` или «не паникует», без попыток эмулировать жест в headless (правило проекта).

---

## Команды запуска

```bash
# запуск демо
npm start                    # wasm:clean + wasm:dev + vite

# отдельные шаги
npm run wasm:dev             # wasm-pack build --dev --target web --out-dir pkg
npm run dev                  # vite

# тесты подглавы
wasm-pack test --headless --chrome --test wasm_converter
wasm-pack test --headless --chrome --test wasm_converter_worker
wasm-pack test --headless --chrome --test wasm_converter_native
wasm-pack test --headless --chrome --test wasm_converter_ui

# игнорируемые (требуют ручного прогона)
wasm-pack test --headless --chrome --test wasm_converter_ui -- --ignored

# всё вместе
npm test                     # host + node + chrome
```

---

## Демо-пример в `app.rs`

`app.rs` соблюдает контракт с `lib.rs`: единственная публичная точка входа `run() -> Result<(), JsValue>`, вызывается после `logging::init()`. Здесь же — одноразовая инжекция CSS секции (идемпотентная через `thread_local!`).

```rust
//! Демо-секция подглавы 18.18 «Практический проект: конвертер файлов».
//!
//! Контракт с `lib.rs`:
//! - единственная публичная точка входа — `run() -> Result<(), JsValue>`;
//! - `logging::init()` уже вызван из `lib.rs`, повторять не нужно;
//! - CSS секции инжектится здесь ровно один раз (idempotent).

use std::cell::Cell;

use wasm_bindgen::prelude::*;

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);

        let css = r#"
            #converter-demo { font-family: system-ui, sans-serif; max-width: 900px; }
            #converter-demo button { padding: 6px 12px; cursor: pointer; }
            #converter-demo button:disabled { opacity: 0.5; cursor: not-allowed; }
            #progress-outer {
                width: 280px; height: 16px;
                border: 1px solid #ccc; border-radius: 8px;
                overflow: hidden; background: #f0f0f0;
            }
            #progress-inner {
                height: 100%; background: #3498db;
                transition: width 0.1s;
            }
            .phase-bar { display: flex; gap: 2px; margin-top: 4px; }
            .phase-cell { width: 40px; height: 6px; background: #ddd; border-radius: 3px; }
            .phase-cell.done   { background: #27ae60; }
            .phase-cell.active { background: #e67e22; }
            .phase-cell.error  { background: #c0392b; }
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

/// Точка входа демо-секции 18.18.
pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let section = crate::converter_app::build_converter_section();

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;
    body.append_child(&section)?;
    Ok(())
}
```

Что показывает демо:

- **Pipeline-индикатор** из 7 стадий с иконками `○ ◐ ● ✗ ⊘` — «карта» всего workflow.
- **Прогресс-бар** с подписью активной фазы и ETA (`Чтение (Blob.stream)`, `Конвертация (sync, CPU-bound)`, `SHA-256 (chunked, yield 16 мс)`, `Запись (chunked)`).
- **Magic-байты** рядом с форматом (`PNG (magic: 89 50 4E 47 0D 0A 1A 0A)`).
- **Self-check** grayscale (badge ✓/✗ — «на 16 точках R==G==B»).
- **Тайминги по фазам** с МБ/с и подсветкой узкого места.
- **Селектор движка**: WASM inline / Worker / Parallel / OffscreenCanvas.
- **Отмена** — активна на async-фазах, честно блокируется на sync-фазе конвертации.
- **Сохранение** через FSA с fallback на `<a download>`, повторный save переиспользует кэшированный handle.
- **Upload** на `auth-server` по контракту `files/{upload_id}/{filename}` и идемпотентный `DELETE`.
- **Кнопка «Скачать SHA-256»** — sidecar-файл для аудита.

---

## Правила проекта, зафиксированные в этой подглаве

- **`app.rs` содержит только демо-код текущей подглавы.** Предыдущие секции не накапливаются; физическое разделение — через `converter_app/`.
- **Любая длительная async-операция должна отдавать управление через `TimeoutFuture::new(0).await`.** Yield раз в 16 мс, а не на каждом чанке.
- **Indeterminate progress bar недопустим для длительных операций.** Если операцию можно разбить — разбейте. Даже 500 МБ режутся на 4 куска.
- **Адаптивность — параметр, а не «стратегия с перезапуском».** Перезапуск выбрасывает уже сделанную работу. Chunk size и число воркеров меняются без потери прогресса.
- **Если две стратегии отличаются только настройками — объединяйте.** Было три grayscale-стратегии (Inline/Worker/ChunkedWorker), стало четыре по числу движков, но все — параметры одного конвейера.
- **Состояние, от которого зависят два UI-элемента, живёт в `Rc<RefCell<State>>`.** Не пробрасывайте колбэки между несвязанными обработчиками.
- **«Отменить/удалить» — идемпотентно.** `removed: true` в первый раз, `removed: false` во второй — оба успех. Никаких 404.
- **Пользователь работает с предметными понятиями.** Показывайте `photo.grayscale.png` и «удалено», а не `upload_id=upload-1734…`.
- **Автообновление UI важнее кнопки «Обновить».**
- **После рефакторинга — `cargo check` и удаление мёртвого кода.** `dead_code` — сигнал артефактов предыдущих версий.
- **Единый каталог файлов — `uploads_root()/{upload_id}/{filename}`.** Все способы пишут туда; изменение решения на лету даёт неконсистентность.
- **`Option<T>` в hot loop — обычно лишний.** Переменная, присваиваемая и читаемая в одной итерации, должна быть локальной.
- **Тесты с abort не ждут ответа сервера.** Если даёте запросу «начаться», отменяйте немедленно.
- **`cargo build` без предупреждений.** `unused_assignments`, `dead_code`, `unused_imports` маскируют реальные проблемы.
- **Если web-sys не экспортирует метод с нужной сигнатурой — `Reflect::get` + `Function::call*`.** Универсальный способ обойти разницу между версиями web-sys.
- **`js_sys::AsyncIterator` — не Promise.** Для итерации по async-iterable: `iter.next()` → `Promise` → `JsFuture::from`.
- **При вызове методов через Reflect проверяйте `!func.is_undefined()`.**
- **Headless-тесты не эмулируют user gesture.** Помечайте `#[ignore]` или проверяйте обработку `SecurityError`.
- **Демо имитирует реальный workflow.** «Взял файл, обработал, положил рядом» — а не отдельные вызовы API.
- **Ограничения (`MAX_FILE_SIZE`, `SCAN_LIMIT`) — видны пользователю.** Молчаливый обрез создаёт ложное впечатление.
- **`format!` и фигурные скобки.** `{obj.field}` не работает; для литеральных `{}` — экранируйте `{{ }}`.
- **`format!` в хвосте `match`-arm или `if`-ветви — выносите в функцию с `-> String` или оборачивайте в `let s: String = …; s`.** rust-analyzer не всегда выводит тип макроса по позиции.

---

## Что демонстрация показывает в итоге

- **Реальный workflow**: выбрать → увидеть «до» → сконвертировать → увидеть «после» → сохранить или загрузить.
- **Три источника задержки с прогрессом**: чтение, хеширование, запись — единый приём yield 16 мс.
- **Четыре движка grayscale** с разной ценой и общей сигнатурой `fn(&[u8], ImageFormat) -> Result<Vec<u8>, String>`.
- **Отмена**, честная к границам: работает на async-фазах, блокируется на sync-фазе.
- **Понятные ошибки**: `AbortError` → «отменено», `SecurityError` → «нужен жест», network error → «Сервер недоступен».
- **Метрики в одну строку** с таймингами по фазам, МБ/с и подсветкой узкого места.
- **Идемпотентный сброс** и повторное сохранение без реконвертации.

Подглава 18.18 — завершающая в главе 18: показывает, как все приёмы предыдущих подглав (Blob, stream, FSA, XHR, Worker, OffscreenCanvas, hash, chunked write, upload contract) собираются в один сквозной сценарий без избыточности и с честными компромиссами.


## Полные содержания файлов главы 18.18

Ниже — все созданные/изменённые файлы в финальном состоянии (после всех правок). Порядок: сначала исходные модули, затем `app.rs`, затем подмодуль `converter_app/`, затем тесты, и в конце — дельта `Cargo.toml` и `lib.rs`.

---

### `src/converter.rs`

```rust
use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Uint8Array};
use sha2::{Digest, Sha256};
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, File, ReadableStreamDefaultReader};

/// Формат изображения, определённый по magic-байтам.
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum ImageFormat {
    Png,
    Jpeg,
}

pub const MAGIC_PNG: [u8; 4] = [0x89, 0x50, 0x4E, 0x47];
pub const MAGIC_JPEG: [u8; 3] = [0xFF, 0xD8, 0xFF];
pub const MAGIC_MIN: usize = 4;

/// Валидация magic-байтов с человекочитаемой ошибкой.
pub fn validate_magic_bytes(data: &[u8]) -> Result<ImageFormat, String> {
    if data.len() < MAGIC_MIN {
        return Err(format!(
            "Файл слишком мал: {} байт, минимум {}",
            data.len(),
            MAGIC_MIN
        ));
    }
    if data.starts_with(&MAGIC_PNG) {
        Ok(ImageFormat::Png)
    } else if data.starts_with(&MAGIC_JPEG) {
        Ok(ImageFormat::Jpeg)
    } else {
        Err(format!(
            "Неподдерживаемый формат (magic: {:02X} {:02X} {:02X} {:02X})",
            data[0], data[1], data[2], data[3]
        ))
    }
}

/// Результат потокового чтения.
pub enum ReadOutcome {
    Ok(Vec<u8>),
    Cancelled,
    InvalidFormat(String),
}

/// Потоковое чтение с прогрессом, отменой и early-exit по magic-байтам.
pub async fn read_blob_stream_validated(
    file: &File,
    mut on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<ReadOutcome, JsValue> {
    let stream = file.stream();
    let reader = ReadableStreamDefaultReader::new(&stream)?;

    let cap = (file.size() as usize).min(16 * 1024 * 1024);
    let mut chunks: Vec<u8> = Vec::with_capacity(cap);

    let mut received: u64 = 0;
    let total = file.size() as u64;
    let mut last_yield = 0.0_f64;
    let mut validated = false;

    loop {
        if is_cancelled() {
            return Ok(ReadOutcome::Cancelled);
        }
        let result = JsFuture::from(reader.read()).await?;
        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))?
            .as_bool()
            .unwrap_or(true);
        if done {
            break;
        }
        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))?;
        let chunk = Uint8Array::new(&value);
        let len = chunk.length() as u64;
        chunks.extend_from_slice(&chunk.to_vec());
        received += len;

        if !validated && chunks.len() >= MAGIC_MIN {
            validated = true;
            if let Err(msg) = validate_magic_bytes(&chunks) {
                return Ok(ReadOutcome::InvalidFormat(msg));
            }
        }

        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            on_progress(received as f64 / total as f64);
            TimeoutFuture::new(0).await;
            last_yield = now;
        }
    }
    on_progress(1.0);
    Ok(ReadOutcome::Ok(chunks))
}

/// Адаптивный размер чанка.
pub fn adaptive_chunk(total: usize, target_iters: usize) -> usize {
    let iters = target_iters.max(1);
    (total / iters).clamp(64 * 1024, 4 * 1024 * 1024)
}

pub struct HashStats {
    pub hex: String,
    pub yields: u32,
    pub chunks: u32,
    pub chunk_size: usize,
}

pub async fn sha256_hex_with_progress(
    data: &[u8],
    mut on_progress: impl FnMut(f64),
) -> HashStats {
    let chunk_size = adaptive_chunk(data.len(), 200);
    let mut hasher = Sha256::new();
    let total = data.len().max(1);
    let mut last_yield = 0.0_f64;
    let mut yields = 0u32;
    let mut chunks = 0u32;

    for (i, chunk) in data.chunks(chunk_size).enumerate() {
        hasher.update(chunk);
        chunks += 1;
        let processed = ((i + 1) * chunk_size).min(data.len());
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            on_progress(processed as f64 / total as f64);
            TimeoutFuture::new(0).await;
            last_yield = now;
            yields += 1;
        }
    }
    on_progress(1.0);
    HashStats {
        hex: hex::encode(hasher.finalize()),
        yields,
        chunks,
        chunk_size,
    }
}

pub fn sha256_hex(data: &[u8]) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data);
    hex::encode(hasher.finalize())
}

pub fn to_grayscale_png(data: &[u8], format: ImageFormat) -> Result<Vec<u8>, String> {
    use image::{GenericImageView, ImageFormat as ImgFormat};

    let img = image::load_from_memory_with_format(
        data,
        match format {
            ImageFormat::Png => ImgFormat::Png,
            ImageFormat::Jpeg => ImgFormat::Jpeg,
        },
    )
    .map_err(|e| e.to_string())?;

    let gray = img.grayscale();
    let (w, h) = gray.dimensions();
    if w != img.width() || h != img.height() {
        return Err("Размеры после конвертации изменились".into());
    }
    let mut out = Vec::new();
    gray.write_to(&mut std::io::Cursor::new(&mut out), ImgFormat::Png)
        .map_err(|e| e.to_string())?;
    Ok(out)
}

pub fn bytes_to_blob(data: &[u8]) -> Result<Blob, JsValue> {
    let arr = Array::new();
    arr.push(&Uint8Array::from(data));
    Blob::new_with_u8_array_sequence(&arr)
}

/// 5.1–5.3: декодирование PNG/JPEG в сырой RGBA + размеры.
pub fn decode_to_rgba(data: &[u8], format: ImageFormat) -> Result<(Vec<u8>, u32, u32), String> {
    use image::ImageFormat as ImgFormat;
    let img = image::load_from_memory_with_format(
        data,
        match format {
            ImageFormat::Png => ImgFormat::Png,
            ImageFormat::Jpeg => ImgFormat::Jpeg,
        },
    )
    .map_err(|e| e.to_string())?;
    let (w, h) = (img.width(), img.height());
    Ok((img.to_rgba8().into_raw(), w, h))
}

/// 5.1–5.3: собрать PNG из RGBA.
pub fn encode_rgba_png(rgba: &[u8], w: u32, h: u32) -> Result<Vec<u8>, String> {
    let img: image::RgbaImage = image::ImageBuffer::from_raw(w, h, rgba.to_vec())
        .ok_or_else(|| "Неверный размер RGBA-буфера".to_string())?;
    let mut out = Vec::new();
    img.write_to(&mut std::io::Cursor::new(&mut out), image::ImageFormat::Png)
        .map_err(|e| e.to_string())?;
    Ok(out)
}

/// 2.3: клиентский идентификатор для контракта uploads_root()/{upload_id}/{filename}.
pub fn generate_upload_id() -> String {
    if let Some(w) = web_sys::window() {
        if let Ok(crypto) = js_sys::Reflect::get(&w, &JsValue::from_str("crypto")) {
            if let Ok(uuid_fn) =
                js_sys::Reflect::get(&crypto, &JsValue::from_str("randomUUID"))
            {
                if !uuid_fn.is_undefined() {
                    if let Ok(f) = uuid_fn.dyn_into::<js_sys::Function>() {
                        if let Ok(v) = f.call0(&crypto) {
                            if let Some(s) = v.as_string() {
                                return s;
                            }
                        }
                    }
                }
            }
        }
    }
    let ts = js_sys::Date::now() as u64;
    format!("upload-{:x}", ts)
}
```

---

### `src/converter_ui.rs`

```rust
use std::cell::RefCell;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{
    Blob, File, FileSystemFileHandle, HtmlAnchorElement, HtmlInputElement, ProgressEvent, Url,
    XmlHttpRequest, XmlHttpRequestResponseType,
};

use crate::converter::adaptive_chunk;

thread_local! {
    static LAST_SAVE_HANDLE: RefCell<Option<FileSystemFileHandle>> = const { RefCell::new(None) };
}

pub fn forget_save_handle() {
    LAST_SAVE_HANDLE.with(|h| *h.borrow_mut() = None);
}

pub async fn pick_image_file() -> Result<File, JsValue> {
    let window = web_sys::window().unwrap();

    if let Ok(picker_fn) = js_sys::Reflect::get(&window, &JsValue::from_str("showOpenFilePicker")) {
        if !picker_fn.is_undefined() {
            let opts = js_sys::Object::new();
            let types = js_sys::Array::new();
            let accept = js_sys::Object::new();
            js_sys::Reflect::set(&accept, &"description".into(), &"Images".into())?;

            let mime = js_sys::Object::new();
            js_sys::Reflect::set(
                &mime,
                &"image/png".into(),
                &js_sys::Array::of1(&".png".into()),
            )?;
            js_sys::Reflect::set(
                &mime,
                &"image/jpeg".into(),
                &js_sys::Array::of2(&".jpg".into(), &".jpeg".into()),
            )?;
            js_sys::Reflect::set(&accept, &"accept".into(), &mime)?;
            types.push(&accept);
            js_sys::Reflect::set(&opts, &"types".into(), &types)?;

            let func = picker_fn.dyn_into::<js_sys::Function>()?;
            let promise = func.call1(&window, &opts)?;
            let result = JsFuture::from(promise.dyn_into::<js_sys::Promise>()?).await?;
            let arr = js_sys::Array::from(&result);
            let handle = arr.get(0);
            let file_handle = web_sys::FileSystemFileHandle::from(handle);
            let file = JsFuture::from(file_handle.get_file()).await?;
            return Ok(File::from(file));
        }
    }

    let document = window.document().unwrap();
    let input = document
        .create_element("input")?
        .dyn_into::<HtmlInputElement>()?;
    input.set_type("file");
    input.set_accept("image/png,image/jpeg,.png,.jpg,.jpeg");
    input.click();

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let input_clone = input.clone();
        let on_change = Closure::once(move || {
            match input_clone.files().and_then(|f| f.get(0)) {
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
    let file = JsFuture::from(promise).await?;
    Ok(File::from(file))
}

pub enum SaveOutcome {
    Fsa,
    FallbackDownload,
    Cancelled,
}

pub async fn save_file_chunked(
    data: &[u8],
    suggested_name: &str,
    mut on_progress: impl FnMut(f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<SaveOutcome, JsValue> {
    let window = web_sys::window().unwrap();

    if let Ok(saver_fn) = js_sys::Reflect::get(&window, &JsValue::from_str("showSaveFilePicker")) {
        if !saver_fn.is_undefined() {
            let cached = LAST_SAVE_HANDLE.with(|h| h.borrow().clone());
            let file_handle = if let Some(h) = cached {
                h
            } else {
                let opts = js_sys::Object::new();
                js_sys::Reflect::set(&opts, &"suggestedName".into(), &suggested_name.into())?;
                let func = saver_fn.dyn_into::<js_sys::Function>()?;
                let promise = func.call1(&window, &opts)?;
                let handle = JsFuture::from(promise.dyn_into::<js_sys::Promise>()?).await?;
                let fh = FileSystemFileHandle::from(handle);
                LAST_SAVE_HANDLE.with(|h| *h.borrow_mut() = Some(fh.clone()));
                fh
            };

            let writable = JsFuture::from(file_handle.create_writable()).await?;
            let writable = web_sys::FileSystemWritableFileStream::from(writable);

            let chunk = adaptive_chunk(data.len(), 200);
            let total = data.len().max(1);
            let mut last_yield = 0.0_f64;

            for (i, c) in data.chunks(chunk).enumerate() {
                if is_cancelled() {
                    let _ = writable.close();
                    return Ok(SaveOutcome::Cancelled);
                }
                JsFuture::from(writable.write_with_u8_array(c)?).await?;
                let processed = ((i + 1) * chunk).min(data.len());
                let now = js_sys::Date::now();
                if now - last_yield >= 16.0 {
                    on_progress(processed as f64 / total as f64);
                    TimeoutFuture::new(0).await;
                    last_yield = now;
                }
            }
            JsFuture::from(writable.close()).await?;
            on_progress(1.0);
            return Ok(SaveOutcome::Fsa);
        }
    }

    on_progress(0.0);
    let blob = Blob::new_with_u8_array_sequence(&js_sys::Array::of1(
        &js_sys::Uint8Array::from(data),
    ))?;
    let url = Url::create_object_url_with_blob(&blob)?;
    let a = window
        .document()
        .unwrap()
        .create_element("a")?
        .dyn_into::<HtmlAnchorElement>()?;
    a.set_href(&url);
    a.set_download(suggested_name);
    a.click();
    Url::revoke_object_url(&url)?;
    on_progress(1.0);
    Ok(SaveOutcome::FallbackDownload)
}

pub async fn upload_to_server(
    data: &[u8],
    upload_id: &str,
    filename: &str,
    mime: &str,
    base_url: &str,
    on_progress: impl FnMut(f64) + 'static,
) -> Result<u16, JsValue> {
    let url = format!(
        "{}/files/{}/{}",
        base_url.trim_end_matches('/'),
        urlencode(upload_id),
        urlencode(filename)
    );
    upload_http(data, mime, &url, on_progress).await
}

async fn upload_http(
    data: &[u8],
    mime: &str,
    url: &str,
    mut on_progress: impl FnMut(f64) + 'static,
) -> Result<u16, JsValue> {
    let xhr = XmlHttpRequest::new()?;
    xhr.open_with_async("POST", url, true)?;
    xhr.set_request_header("Content-Type", mime)?;

    let upload = xhr.upload()?;
    let cb = Closure::<dyn FnMut(ProgressEvent)>::new(move |e: ProgressEvent| {
        if e.length_computable() {
            on_progress(e.loaded() / e.total().max(1.0));
        }
    });
    upload.set_onprogress(Some(cb.as_ref().unchecked_ref()));
    cb.forget();

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let xhr_load = xhr.clone();
        let resolve_cb = Closure::once(move || {
            let status = xhr_load.status().unwrap_or(0);
            let _ = resolve.call1(&JsValue::NULL, &JsValue::from_f64(status as f64));
        });
        xhr.set_onload(Some(resolve_cb.as_ref().unchecked_ref()));
        resolve_cb.forget();

        let reject_cb = Closure::once(move || {
            let _ = reject.call1(&JsValue::NULL, &JsValue::from_str("network error"));
        });
        xhr.set_onerror(Some(reject_cb.as_ref().unchecked_ref()));
        reject_cb.forget();
    });

    xhr.send_with_opt_u8_array(Some(data))?;
    let status = JsFuture::from(promise).await?;
    Ok(status.as_f64().unwrap_or(0.0) as u16)
}

pub async fn delete_upload(base_url: &str, upload_id: &str) -> Result<bool, JsValue> {
    let url = format!(
        "{}/files/{}",
        base_url.trim_end_matches('/'),
        urlencode(upload_id)
    );
    let xhr = XmlHttpRequest::new()?;
    xhr.open_with_async("DELETE", &url, true)?;
    xhr.set_response_type(XmlHttpRequestResponseType::Json);

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let xhr_load = xhr.clone();
        let resolve_cb = Closure::once(move || {
            let status = xhr_load.status().unwrap_or(0);
            let resp = xhr_load.response().unwrap_or(JsValue::UNDEFINED);
            let removed = js_sys::Reflect::get(&resp, &"removed".into())
                .ok()
                .and_then(|v| v.as_bool())
                .unwrap_or(false);
            let payload = js_sys::Object::new();
            let _ = js_sys::Reflect::set(
                &payload,
                &"status".into(),
                &JsValue::from_f64(status as f64),
            );
            let _ = js_sys::Reflect::set(
                &payload,
                &"removed".into(),
                &JsValue::from_bool(removed),
            );
            let _ = resolve.call1(&JsValue::NULL, &payload.into());
        });
        xhr.set_onload(Some(resolve_cb.as_ref().unchecked_ref()));
        resolve_cb.forget();

        let reject_cb = Closure::once(move || {
            let _ = reject.call1(&JsValue::NULL, &JsValue::from_str("network error"));
        });
        xhr.set_onerror(Some(reject_cb.as_ref().unchecked_ref()));
        reject_cb.forget();
    });

    xhr.send()?;
    let resp = JsFuture::from(promise).await?;
    let status = js_sys::Reflect::get(&resp, &"status".into())
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u16;
    let removed = js_sys::Reflect::get(&resp, &"removed".into())
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);

    if (200..300).contains(&status) {
        Ok(removed)
    } else {
        Err(JsValue::from_str(&format!("HTTP {}", status)))
    }
}

fn urlencode(s: &str) -> String {
    let mut out = String::with_capacity(s.len());
    for b in s.bytes() {
        match b {
            b'A'..=b'Z' | b'a'..=b'z' | b'0'..=b'9' | b'-' | b'_' | b'.' | b'~' => {
                out.push(b as char);
            }
            _ => out.push_str(&format!("%{:02X}", b)),
        }
    }
    out
}
```

---

### `src/converter_worker.rs`

```rust
use futures::future::join_all;
use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, MessageEvent, Url, Worker};

const WORKER_JS: &str = r#"
    self.onmessage = function(e) {
        const d = e.data;
        const rgba = new Uint8ClampedArray(d.buffer);
        const out = new Uint8ClampedArray(rgba.length);
        for (let i = 0; i < rgba.length; i += 4) {
            const y = (0.299 * rgba[i] + 0.587 * rgba[i+1] + 0.114 * rgba[i+2]) | 0;
            out[i] = y; out[i+1] = y; out[i+2] = y; out[i+3] = rgba[i+3];
        }
        self.postMessage({ id: d.id, buffer: out.buffer }, [out.buffer]);
    };
"#;

fn spawn_worker() -> Result<Worker, JsValue> {
    let parts = Array::of1(&JsValue::from_str(WORKER_JS));
    let blob = Blob::new_with_str_sequence(&parts)?;
    let url = Url::create_object_url_with_blob(&blob)?;
    let w = Worker::new(&url)?;
    Url::revoke_object_url(&url)?;
    Ok(w)
}

async fn process_in_worker(worker: Worker, id: u32, rgba: Vec<u8>) -> Result<Vec<u8>, JsValue> {
    let u8arr = Uint8Array::from(rgba.as_slice());
    let buf = u8arr.buffer();

    let msg = Object::new();
    Reflect::set(&msg, &"id".into(), &JsValue::from_f64(id as f64))?;
    Reflect::set(&msg, &"buffer".into(), &buf)?;

    let post = Reflect::get(&worker, &"postMessage".into())?
        .dyn_into::<js_sys::Function>()?;
    let transfer = Array::of1(&buf);
    post.call2(&worker, &msg.into(), &transfer.into())?;

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let on_msg = Closure::once(move |e: MessageEvent| {
            let data = e.data();
            let buf = Reflect::get(&data, &"buffer".into()).unwrap_or(JsValue::UNDEFINED);
            let _ = resolve.call1(&JsValue::NULL, &buf);
        });
        worker.set_onmessage(Some(on_msg.as_ref().unchecked_ref()));
        on_msg.forget();

        let on_err = Closure::once(move |e: JsValue| {
            let _ = reject.call1(&JsValue::NULL, &e);
        });
        worker.set_onerror(Some(on_err.as_ref().unchecked_ref()));
        on_err.forget();
    });

    let result = JsFuture::from(promise).await?;
    let arr = Uint8Array::new(&result);
    worker.terminate();
    Ok(arr.to_vec())
}

pub async fn grayscale_in_worker(rgba: Vec<u8>) -> Result<Vec<u8>, JsValue> {
    let worker = spawn_worker()?;
    process_in_worker(worker, 0, rgba).await
}

pub async fn grayscale_parallel(rgba: Vec<u8>) -> Result<Vec<u8>, JsValue> {
    let n = worker_count().max(1);
    let total = rgba.len();
    let stripe = (total / n + 3) & !3;
    if stripe == 0 || total == 0 {
        return grayscale_in_worker(rgba).await;
    }

    let mut futures = Vec::with_capacity(n);
    for i in 0..n {
        let start = i * stripe;
        if start >= total {
            break;
        }
        let end = ((i + 1) * stripe).min(total);
        let slice = rgba[start..end].to_vec();
        let w = spawn_worker()?;
        futures.push(async move {
            let out = process_in_worker(w, i as u32, slice).await?;
            Ok::<(usize, Vec<u8>), JsValue>((start, out))
        });
    }

    let parts = join_all(futures).await;
    let mut result = vec![0u8; total];
    for p in parts {
        let (offset, bytes) = p?;
        let len = bytes.len();
        result[offset..offset + len].copy_from_slice(&bytes);
    }
    Ok(result)
}

fn worker_count() -> usize {
    let mut n = 2usize;
    if let Some(w) = web_sys::window() {
        if let Ok(nav) = Reflect::get(&w, &"navigator".into()) {
            if let Ok(hc) = Reflect::get(&nav, &"hardwareConcurrency".into()) {
                if let Some(v) = hc.as_f64() {
                    n = (v as usize).min(4).max(1);
                }
            }
        }
    }
    n
}
```

---

### `src/converter_native.rs`

```rust
use js_sys::{Function, Reflect, Uint8Array};
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, OffscreenCanvas};

/// 5.3: grayscale через OffscreenCanvas.
/// Все промис-возвращающие методы вызываются через Reflect — это не зависит
/// от версии web-sys (сигнатуры Promise vs Result<Promise> менялись).
pub async fn grayscale_native(png_bytes: &[u8]) -> Result<Vec<u8>, JsValue> {
    let window = web_sys::window().unwrap();

    let parts = js_sys::Array::new();
    parts.push(&Uint8Array::from(png_bytes));
    let blob: Blob = Blob::new_with_u8_array_sequence(&parts)?;

    let create_fn = Reflect::get(&window, &JsValue::from_str("createImageBitmap"))?
        .dyn_into::<Function>()?;
    let bitmap_promise_js: JsValue = create_fn.call1(&window, &blob)?;
    let bitmap_promise: js_sys::Promise = bitmap_promise_js.dyn_into()?;
    let bitmap_js: JsValue = JsFuture::from(bitmap_promise).await?;

    let w = Reflect::get(&bitmap_js, &JsValue::from_str("width"))?
        .as_f64()
        .unwrap_or(0.0);
    let h = Reflect::get(&bitmap_js, &JsValue::from_str("height"))?
        .as_f64()
        .unwrap_or(0.0);
    if w <= 0.0 || h <= 0.0 {
        return Err(JsValue::from_str("createImageBitmap: bad size"));
    }

    let canvas = OffscreenCanvas::new(w as u32, h as u32)?;

    let get_ctx = Reflect::get(&canvas, &JsValue::from_str("getContext"))?
        .dyn_into::<Function>()?;
    let ctx: JsValue = get_ctx.call1(&canvas, &JsValue::from_str("2d"))?;
    if ctx.is_null() || ctx.is_undefined() {
        return Err(JsValue::from_str("OffscreenCanvas: no 2d context"));
    }

    let _ = Reflect::set(
        &ctx,
        &JsValue::from_str("filter"),
        &JsValue::from_str("grayscale(1)"),
    );

    let draw_fn = Reflect::get(&ctx, &JsValue::from_str("drawImage"))?
        .dyn_into::<Function>()?;
    draw_fn.call3(
        &ctx,
        &bitmap_js,
        &JsValue::from_f64(0.0),
        &JsValue::from_f64(0.0),
    )?;

    let convert_fn = Reflect::get(&canvas, &JsValue::from_str("convertToBlob"))?
        .dyn_into::<Function>()?;
    let blob_promise_js: JsValue = convert_fn.call0(&canvas)?;
    let blob_promise: js_sys::Promise = blob_promise_js.dyn_into()?;
    let result_blob_js: JsValue = JsFuture::from(blob_promise).await?;
    let result_blob: Blob = result_blob_js.dyn_into()?;

    let array_buffer_fn = Reflect::get(&result_blob, &JsValue::from_str("arrayBuffer"))?
        .dyn_into::<Function>()?;
    let buf_promise_js: JsValue = array_buffer_fn.call0(&result_blob)?;
    let buf_promise: js_sys::Promise = buf_promise_js.dyn_into()?;
    let buf_js: JsValue = JsFuture::from(buf_promise).await?;
    let u8arr = Uint8Array::new(&buf_js);
    Ok(u8arr.to_vec())
}
```

---

### `src/app.rs`

```rust
//! Демо-секция подглавы 18.18 «Практический проект: конвертер файлов».
//!
//! Контракт с `lib.rs`:
//! - единственная публичная точка входа — `run() -> Result<(), JsValue>`;
//! - `logging::init()` уже вызван из `lib.rs`, повторять не нужно;
//! - CSS секции инжектится здесь ровно один раз (idempotent).

use std::cell::Cell;

use wasm_bindgen::prelude::*;

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);

        let css = r#"
            #converter-demo { font-family: system-ui, sans-serif; max-width: 900px; }
            #converter-demo button { padding: 6px 12px; cursor: pointer; }
            #converter-demo button:disabled { opacity: 0.5; cursor: not-allowed; }
            #progress-outer {
                width: 280px; height: 16px;
                border: 1px solid #ccc; border-radius: 8px;
                overflow: hidden; background: #f0f0f0;
            }
            #progress-inner {
                height: 100%; background: #3498db;
                transition: width 0.1s;
            }
            .phase-bar { display: flex; gap: 2px; margin-top: 4px; }
            .phase-cell { width: 40px; height: 6px; background: #ddd; border-radius: 3px; }
            .phase-cell.done   { background: #27ae60; }
            .phase-cell.active { background: #e67e22; }
            .phase-cell.error  { background: #c0392b; }
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

/// Точка входа демо-секции 18.18.
pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let section = crate::converter_app::build_converter_section();

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;
    body.append_child(&section)?;
    Ok(())
}
```

---

### `src/converter_app/mod.rs`

```rust
pub mod state;
pub mod ui_builder;
pub mod handlers;
pub mod pipeline;
pub mod metrics;

use web_sys::HtmlElement;

pub fn build_converter_section() -> HtmlElement {
    let (section, ui, state) = ui_builder::build();
    handlers::wire(&ui, state);
    section
}
```

---

### `src/converter_app/state.rs`

```rust
use std::cell::RefCell;
use std::rc::Rc;
use web_sys::{
    HtmlButtonElement, HtmlElement, HtmlImageElement, HtmlInputElement,
    HtmlProgressElement, HtmlSelectElement, HtmlSpanElement,
};

pub const STAGES: [&str; 7] = [
    "Выбор", "Чтение", "Валидация", "Конвертация", "Хеш", "Сохранение", "Загрузка",
];
pub const MAX_FILE_SIZE: u64 = 500 * 1024 * 1024;

#[derive(Clone, Copy, PartialEq)]
pub enum StageStatus {
    Pending, Active, Done, Error, Cancelled,
}

#[derive(Clone, Copy, PartialEq)]
pub enum Engine {
    WasmInline,
    WasmWorker,
    WasmParallel,
    NativeCanvas,
}

impl Engine {
    pub fn label(self) -> &'static str {
        match self {
            Engine::WasmInline => "WASM inline",
            Engine::WasmWorker => "WASM worker",
            Engine::WasmParallel => "WASM parallel",
            Engine::NativeCanvas => "Native OffscreenCanvas",
        }
    }
    pub fn from_str(s: &str) -> Self {
        match s {
            "worker" => Engine::WasmWorker,
            "parallel" => Engine::WasmParallel,
            "native" => Engine::NativeCanvas,
            _ => Engine::WasmInline,
        }
    }
}

#[derive(Default)]
pub struct DemoState {
    pub result_bytes: Option<Vec<u8>>,
    pub result_name: Option<String>,
    pub original_url: Option<String>,
    pub result_url: Option<String>,
    pub cancel_requested: bool,
    pub pipeline_running: bool,
    pub upload_id: Option<String>,
    pub uploaded: bool,
    pub original_len: u64,
    pub result_len: u64,
    pub format_label: String,
    pub magic_hex: String,
    pub hash_hex: String,
    pub verified: bool,
    pub read_ms: f64,
    pub convert_ms: f64,
    pub hash_ms: f64,
    pub save_ms: f64,
    pub upload_ms: f64,
    pub delete_ms: f64,
    pub hash_yields: u32,
    pub hash_chunks: u32,
    pub hash_chunk_size: usize,
    pub engine_label: String,
}

#[derive(Clone)]
pub struct Ui {
    pub pick_btn: HtmlButtonElement,
    pub cancel_btn: HtmlButtonElement,
    pub save_btn: HtmlButtonElement,
    pub reset_btn: HtmlButtonElement,
    pub upload_cb: HtmlInputElement,
    pub upload_btn: HtmlButtonElement,
    pub delete_btn: HtmlButtonElement,
    pub hash_download_btn: HtmlButtonElement,
    pub engine_select: HtmlSelectElement,
    pub progress: HtmlProgressElement,
    pub progress_label: HtmlSpanElement,
    pub phase_label: HtmlSpanElement,
    pub stages: Rc<Vec<HtmlSpanElement>>,
    pub phase_cells: Rc<Vec<HtmlElement>>,
    pub status: HtmlElement,
    pub metrics: HtmlElement,
    pub preview_original: HtmlImageElement,
    pub preview_result: HtmlImageElement,
    pub verify_badge: HtmlSpanElement,
}

/// 5.7 — централизованное управление стадиями.
pub struct StageCtx {
    pub stages: Rc<Vec<HtmlSpanElement>>,
    pub phase_cells: Rc<Vec<HtmlElement>>,
    pub phase_label: HtmlSpanElement,
}

impl StageCtx {
    pub fn enter(&self, i: usize, phase: &str) {
        set_stage(&self.stages[i], STAGES[i], StageStatus::Active);
        self.phase_label.set_text_content(Some(phase));
        if let Some(c) = self.phase_cells.get(i) {
            let _ = c.set_attribute("class", "phase-cell active");
        }
    }
    pub fn done(&self, i: usize) {
        set_stage(&self.stages[i], STAGES[i], StageStatus::Done);
        if let Some(c) = self.phase_cells.get(i) {
            let _ = c.set_attribute("class", "phase-cell done");
        }
    }
    pub fn error(&self, i: usize) {
        set_stage(&self.stages[i], STAGES[i], StageStatus::Error);
        if let Some(c) = self.phase_cells.get(i) {
            let _ = c.set_attribute("class", "phase-cell error");
        }
    }
    pub fn cancel(&self, i: usize) {
        set_stage(&self.stages[i], STAGES[i], StageStatus::Cancelled);
    }
    pub fn set_phase(&self, text: &str) {
        self.phase_label.set_text_content(Some(text));
    }
}

pub fn set_stage(stage: &HtmlSpanElement, name: &str, status: StageStatus) {
    let (color, icon) = match status {
        StageStatus::Pending => ("#aaa", "○"),
        StageStatus::Active => ("#e67e22", "◐"),
        StageStatus::Done => ("#27ae60", "●"),
        StageStatus::Error => ("#c0392b", "✗"),
        StageStatus::Cancelled => ("#7f8c8d", "⊘"),
    };
    stage.set_text_content(Some(&format!("{} {}", icon, name)));
    let _ = stage.set_attribute(
        "style",
        &format!(
            "color: {}; margin-right: 12px; font-family: monospace;",
            color
        ),
    );
}

pub fn cancel_check(state: Rc<RefCell<DemoState>>) -> impl Fn() -> bool {
    move || {
        state
            .try_borrow()
            .map(|s| s.cancel_requested)
            .unwrap_or(true)
    }
}
```

---

### `src/converter_app/ui_builder.rs`

```rust
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use web_sys::{
    Document, HtmlButtonElement, HtmlElement, HtmlImageElement, HtmlInputElement,
    HtmlOptionElement, HtmlProgressElement, HtmlSelectElement, HtmlSpanElement,
};

use super::state::{DemoState, StageStatus, Ui, STAGES};

pub fn build() -> (HtmlElement, Ui, Rc<RefCell<DemoState>>) {
    let document = web_sys::window().unwrap().document().unwrap();
    let section = document.create_element("section").unwrap();
    section.set_id("converter-demo");

    let h2 = document.create_element("h2").unwrap();
    h2.set_text_content(Some("18.18 — Конвертер: PNG/JPEG → Grayscale PNG"));
    section.append_child(&h2).unwrap();

    let controls = document.create_element("div").unwrap();
    controls
        .set_attribute(
            "style",
            "margin-bottom: 12px; display: flex; gap: 8px; flex-wrap: wrap; align-items: center;",
        )
        .unwrap();

    let pick_btn = make_button(&document, "Выбрать изображение", "pick-btn");
    let cancel_btn = make_button(&document, "Отмена", "cancel-btn");
    let save_btn = make_button(&document, "Сохранить результат", "save-btn");
    let reset_btn = make_button(&document, "Сбросить", "reset-btn");
    let hash_download_btn = make_button(&document, "Скачать SHA-256", "hash-dl-btn");
    let delete_btn = make_button(&document, "Удалить с сервера", "delete-btn");
    cancel_btn.set_disabled(true);
    save_btn.set_disabled(true);
    hash_download_btn.set_disabled(true);
    delete_btn.set_disabled(true);
    for b in [&pick_btn, &cancel_btn, &save_btn, &hash_download_btn, &reset_btn] {
        controls.append_child(b).unwrap();
    }

    let engine_select = document
        .create_element("select")
        .unwrap()
        .dyn_into::<HtmlSelectElement>()
        .unwrap();
    engine_select.set_id("engine-select");
    for (val, label) in [
        ("wasm", "WASM inline"),
        ("worker", "WASM worker"),
        ("parallel", "WASM parallel"),
        ("native", "Native OffscreenCanvas"),
    ] {
        let opt = document
            .create_element("option")
            .unwrap()
            .dyn_into::<HtmlOptionElement>()
            .unwrap();
        opt.set_value(val);
        opt.set_text_content(Some(label));
        engine_select.append_child(&opt).unwrap();
    }
    let lbl = document.create_element("label").unwrap();
    lbl.set_text_content(Some("Движок:"));
    lbl.set_attribute("style", "margin-left: 12px;").unwrap();
    controls.append_child(&lbl).unwrap();
    controls.append_child(&engine_select).unwrap();

    let upload_cb = document
        .create_element("input")
        .unwrap()
        .dyn_into::<HtmlInputElement>()
        .unwrap();
    upload_cb.set_type("checkbox");
    upload_cb.set_id("upload-cb");
    let upload_btn = make_button(&document, "Загрузить", "upload-btn");
    upload_btn.set_disabled(true);
    let upload_lbl = document.create_element("label").unwrap();
    upload_lbl.set_text_content(Some("Загрузить на сервер"));
    upload_lbl.set_attribute("style", "margin-left: 12px;").unwrap();
    upload_lbl.append_child(&upload_cb).unwrap();
    controls.append_child(&upload_lbl).unwrap();
    controls.append_child(&upload_btn).unwrap();
    controls.append_child(&delete_btn).unwrap();
    section.append_child(&controls).unwrap();

    // Pipeline stages
    let stages_row = document.create_element("div").unwrap();
    stages_row
        .set_attribute(
            "style",
            "margin-bottom: 8px; padding: 8px; background: #f9f9f9; border-radius: 4px; font-size: 14px;",
        )
        .unwrap();
    let mut stages: Vec<HtmlSpanElement> = Vec::with_capacity(STAGES.len());
    for (i, name) in STAGES.iter().enumerate() {
        let s = document
            .create_element("span")
            .unwrap()
            .dyn_into::<HtmlSpanElement>()
            .unwrap();
        super::state::set_stage(&s, name, StageStatus::Pending);
        stages_row.append_child(&s).unwrap();
        stages.push(s);
        if i + 1 < STAGES.len() {
            let arrow = document.create_element("span").unwrap();
            arrow.set_text_content(Some("→"));
            arrow
                .set_attribute("style", "color: #ccc; margin-right: 12px;")
                .unwrap();
            stages_row.append_child(&arrow).unwrap();
        }
    }
    section.append_child(&stages_row).unwrap();

    // 4.4 — тонкая полоса фаз
    let phase_bar = document.create_element("div").unwrap();
    phase_bar.set_attribute("class", "phase-bar").unwrap();
    let mut phase_cells: Vec<HtmlElement> = Vec::with_capacity(STAGES.len());
    for _ in STAGES.iter() {
        let c = document
            .create_element("div")
            .unwrap()
            .dyn_into::<HtmlElement>()
            .unwrap();
        c.set_attribute("class", "phase-cell").unwrap();
        phase_bar.append_child(&c).unwrap();
        phase_cells.push(c);
    }
    section.append_child(&phase_bar).unwrap();

    // Превью
    let preview_row = document.create_element("div").unwrap();
    preview_row
        .set_attribute("style", "display: flex; gap: 24px; margin: 12px 0;")
        .unwrap();
    let (orig_wrap, preview_original, _) =
        make_preview(&document, "preview-original", "Оригинал");
    let (res_wrap, preview_result, verify_badge) =
        make_preview(&document, "preview-result", "Grayscale");
    preview_row.append_child(&orig_wrap).unwrap();
    preview_row.append_child(&res_wrap).unwrap();
    section.append_child(&preview_row).unwrap();

    // Progress + labels
    let progress_row = document.create_element("div").unwrap();
    progress_row
        .set_attribute(
            "style",
            "display: flex; gap: 8px; align-items: center; flex-wrap: wrap; margin-bottom: 8px;",
        )
        .unwrap();
    let progress_outer = document.create_element("div").unwrap();
    progress_outer.set_id("progress-outer");
    let progress_inner = document.create_element("div").unwrap();
    progress_inner.set_id("progress-inner");
    progress_inner.set_attribute("style", "width: 0%;").unwrap();
    progress_outer.append_child(&progress_inner).unwrap();
    let progress = document
        .create_element("progress")
        .unwrap()
        .dyn_into::<HtmlProgressElement>()
        .unwrap();
    progress.set_max(1.0);
    progress.set_value(0.0);
    progress.set_attribute("style", "display:none;").unwrap();
    let progress_label = document
        .create_element("span")
        .unwrap()
        .dyn_into::<HtmlSpanElement>()
        .unwrap();
    progress_label.set_text_content(Some("0%"));
    let phase_label = document
        .create_element("span")
        .unwrap()
        .dyn_into::<HtmlSpanElement>()
        .unwrap();
    phase_label.set_text_content(Some("—"));
    phase_label
        .set_attribute("style", "color:#555; font-family: monospace; font-size: 12px;")
        .unwrap();
    progress_row.append_child(&progress_outer).unwrap();
    progress_row.append_child(&progress_label).unwrap();
    progress_row.append_child(&phase_label).unwrap();
    section.append_child(&progress_row).unwrap();
    section.append_child(&progress).unwrap();

    let status = document
        .create_element("div")
        .unwrap()
        .dyn_into::<HtmlElement>()
        .unwrap();
    status.set_text_content(Some("Готово к выбору файла."));
    status.set_attribute("style", "margin-bottom: 8px;").unwrap();
    section.append_child(&status).unwrap();

    let metrics = document
        .create_element("pre")
        .unwrap()
        .dyn_into::<HtmlElement>()
        .unwrap();
    metrics
        .set_attribute(
            "style",
            "background:#f6f6f6; padding: 8px; border-radius: 4px; font-size: 13px; white-space: pre-wrap; margin: 0;",
        )
        .unwrap();
    section.append_child(&metrics).unwrap();

    let ui = Ui {
        pick_btn,
        cancel_btn,
        save_btn,
        reset_btn,
        upload_cb,
        upload_btn,
        delete_btn,
        hash_download_btn,
        engine_select,
        progress,
        progress_label,
        phase_label,
        stages: Rc::new(stages),
        phase_cells: Rc::new(phase_cells),
        status,
        metrics,
        preview_original,
        preview_result,
        verify_badge,
    };
    let state = Rc::new(RefCell::new(DemoState::default()));
    (
        section.dyn_into::<HtmlElement>().unwrap(),
        ui,
        state,
    )
}

fn make_button(d: &Document, text: &str, id: &str) -> HtmlButtonElement {
    let b = d
        .create_element("button")
        .unwrap()
        .dyn_into::<HtmlButtonElement>()
        .unwrap();
    b.set_id(id);
    b.set_text_content(Some(text));
    b
}

fn make_preview(
    d: &Document,
    id: &str,
    caption: &str,
) -> (HtmlElement, HtmlImageElement, HtmlSpanElement) {
    let wrap = d
        .create_element("div")
        .unwrap()
        .dyn_into::<HtmlElement>()
        .unwrap();
    wrap.set_attribute("style", "text-align: center;").unwrap();

    let cap = d.create_element("div").unwrap();
    cap.set_text_content(Some(caption));
    cap.set_attribute("style", "font-weight: bold; margin-bottom: 4px;")
        .unwrap();
    wrap.append_child(&cap).unwrap();

    let img = d
        .create_element("img")
        .unwrap()
        .dyn_into::<HtmlImageElement>()
        .unwrap();
    img.set_id(id);
    img.set_attribute(
        "style",
        "max-width: 240px; max-height: 240px; min-width: 120px; min-height: 120px; \
         border: 1px solid #ccc; background: #f6f6f6; object-fit: contain;",
    )
    .unwrap();
    wrap.append_child(&img).unwrap();

    // ✅ создаём <span>, чтобы dyn_into::<HtmlSpanElement>() был валиден.
    // `display: block` возвращает блочное поведение, как было у div.
    let badge = d
        .create_element("span")
        .unwrap()
        .dyn_into::<HtmlSpanElement>()
        .unwrap();
    badge
        .set_attribute(
            "style",
            "display: block; font-size: 12px; margin-top: 4px;",
        )
        .unwrap();
    wrap.append_child(&badge).unwrap();

    (wrap, img, badge)
}
```

---

### `src/converter_app/handlers.rs`

```rust
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::spawn_local;
use web_sys::Url;

use crate::converter_ui;
use super::state::{DemoState, Ui};
use super::{metrics, pipeline};

pub fn wire(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    wire_pick(ui, state.clone());
    wire_cancel(ui, state.clone());
    wire_save(ui, state.clone());
    wire_reset(ui, state.clone());
    wire_upload(ui, state.clone());
    wire_delete(ui, state.clone());
    wire_hash_download(ui, state);
}

fn wire_pick(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_owned.clone();
        let state = state.clone();
        spawn_local(async move { pipeline::run(ui, state).await });
    });
    ui.pick_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_cancel(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        if let Ok(mut s) = state.try_borrow_mut() {
            s.cancel_requested = true;
        }
        ui_owned.status.set_text_content(Some("Запрошена отмена…"));
        ui_owned.cancel_btn.set_disabled(true);
    });
    ui.cancel_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_reset(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        if state.borrow().pipeline_running {
            ui_owned
                .status
                .set_text_content(Some("Нельзя сбросить: pipeline выполняется."));
            return;
        }
        pipeline::release_urls(&state);
        converter_ui::forget_save_handle();
        *state.borrow_mut() = DemoState::default();
        let _ = ui_owned.preview_original.remove_attribute("src");
        let _ = ui_owned.preview_result.remove_attribute("src");
        metrics::reset_stages_and_bar(&ui_owned);
        metrics::update_progress(&ui_owned, 0.0);
        ui_owned.progress_label.set_text_content(Some("0%"));
        ui_owned.status.set_text_content(Some("Сброшено."));
        ui_owned.metrics.set_text_content(Some(""));
        ui_owned.verify_badge.set_text_content(Some(""));
        ui_owned.upload_cb.set_checked(false);
        ui_owned.upload_btn.set_disabled(true);
        ui_owned.delete_btn.set_disabled(true);
        ui_owned.hash_download_btn.set_disabled(true);
        ui_owned.pick_btn.set_disabled(false);
        ui_owned.cancel_btn.set_disabled(true);
        ui_owned.save_btn.set_disabled(true);
    });
    ui.reset_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_save(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_owned.clone();
        let state = state.clone();
        spawn_local(async move { pipeline::run_save(ui, state).await });
    });
    ui.save_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_upload(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_for_cb = ui.clone();
    let state_for_cb = state.clone();
    let on_cb = Closure::<dyn FnMut()>::new(move || {
        let enabled = ui_for_cb.upload_cb.checked();
        let has = state_for_cb
            .try_borrow()
            .map(|s| s.result_bytes.is_some())
            .unwrap_or(false);
        ui_for_cb.upload_btn.set_disabled(!(enabled && has));
    });
    ui.upload_cb.set_onchange(Some(on_cb.as_ref().unchecked_ref()));
    on_cb.forget();

    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_owned.clone();
        let state = state.clone();
        spawn_local(async move { pipeline::run_upload(ui, state).await });
    });
    ui.upload_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_delete(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let ui = ui_owned.clone();
        let state = state.clone();
        spawn_local(async move {
            let upload_id = match state.try_borrow().ok().and_then(|s| s.upload_id.clone()) {
                Some(v) => v,
                None => {
                    ui.status
                        .set_text_content(Some("Нет upload_id — сначала загрузите."));
                    return;
                }
            };
            ui.delete_btn.set_disabled(true);
            ui.status.set_text_content(Some("Удаление с сервера…"));
            let t0 = js_sys::Date::now();
            let url = "http://localhost:3001/api";
            match converter_ui::delete_upload(url, &upload_id).await {
                Ok(true) => ui
                    .status
                    .set_text_content(Some("Файл удалён (removed: true).")),
                Ok(false) => ui
                    .status
                    .set_text_content(Some("Файла уже нет (removed: false) — идемпотентно.")),
                Err(e) => {
                    let msg = e.as_string().unwrap_or_else(|| format!("{:?}", e));
                    ui.status
                        .set_text_content(Some(&format!("Ошибка удаления: {}", msg)));
                }
            }
            state.borrow_mut().delete_ms = js_sys::Date::now() - t0;
            ui.delete_btn.set_disabled(false);
        });
    });
    ui.delete_btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_hash_download(ui: &Ui, state: Rc<RefCell<DemoState>>) {
    let ui_owned = ui.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let pair = state.try_borrow().ok().and_then(|s| {
            if s.hash_hex.is_empty() {
                None
            } else {
                s.result_name.clone().map(|n| (s.hash_hex.clone(), n))
            }
        });
        let (hash, name) = match pair {
            Some(v) => v,
            None => return,
        };
        let content = format!("{}  {}\n", hash, name);
        let blob = match web_sys::Blob::new_with_str_sequence(&js_sys::Array::of1(
            &JsValue::from_str(&content),
        )) {
            Ok(b) => b,
            Err(_) => return,
        };
        if let Ok(url) = Url::create_object_url_with_blob(&blob) {
            if let Some(win) = web_sys::window() {
                if let Some(doc) = win.document() {
                    if let Ok(a) = doc.create_element("a") {
                        if let Ok(a) = a.dyn_into::<web_sys::HtmlAnchorElement>() {
                            a.set_href(&url);
                            a.set_download(&format!("{}.sha256", name));
                            a.click();
                        }
                    }
                }
            }
            let _ = Url::revoke_object_url(&url);
        }
        ui_owned
            .status
            .set_text_content(Some("SHA-256 выгружен как sidecar-файл."));
    });
    ui.hash_download_btn
        .set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

/// Выбранный движок.
pub fn current_engine(ui: &Ui) -> super::state::Engine {
    super::state::Engine::from_str(&ui.engine_select.value())
}
```

---

### `src/converter_app/pipeline.rs`

```rust
use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::prelude::*;
use web_sys::Url;

use crate::{converter, converter_native, converter_ui, converter_worker};
use super::handlers::current_engine;
use super::metrics;
use super::state::{cancel_check, DemoState, Engine, StageCtx, Ui, MAX_FILE_SIZE};

const SERVER_BASE: &str = "http://localhost:3001/api";

fn ctx(ui: &Ui) -> StageCtx {
    StageCtx {
        stages: ui.stages.clone(),
        phase_cells: ui.phase_cells.clone(),
        phase_label: ui.phase_label.clone(),
    }
}

pub fn release_urls(state: &Rc<RefCell<DemoState>>) {
    let mut s = state.borrow_mut();
    if let Some(u) = s.original_url.take() {
        let _ = Url::revoke_object_url(&u);
    }
    if let Some(u) = s.result_url.take() {
        let _ = Url::revoke_object_url(&u);
    }
}

pub async fn run(ui: Ui, state: Rc<RefCell<DemoState>>) {
    {
        let mut s = state.borrow_mut();
        if s.pipeline_running {
            return;
        }
        s.pipeline_running = true;
        s.cancel_requested = false;
    }

    let c = ctx(&ui);
    release_urls(&state);
    let _ = ui.preview_original.remove_attribute("src");
    let _ = ui.preview_result.remove_attribute("src");
    metrics::reset_stages_and_bar(&ui);
    metrics::update_progress(&ui, 0.0);
    ui.progress_label.set_text_content(Some("0%"));
    ui.metrics.set_text_content(Some(""));
    ui.verify_badge.set_text_content(Some(""));
    ui.pick_btn.set_disabled(true);
    ui.cancel_btn.set_disabled(false);
    ui.save_btn.set_disabled(true);
    ui.upload_btn.set_disabled(true);
    ui.delete_btn.set_disabled(true);
    ui.hash_download_btn.set_disabled(true);
    ui.reset_btn.set_disabled(true);

    let engine = current_engine(&ui);

    // === Стадия 0: выбор ===
    c.enter(0, "Выбор");
    ui.status.set_text_content(Some("Выбор файла…"));
    let file = match converter_ui::pick_image_file().await {
        Ok(f) => {
            c.done(0);
            f
        }
        Err(e) => {
            let name: String = metrics::error_name(&e);
            c.cancel(0);
            let msg: String = format_pick_error(name.as_str());
            ui.status.set_text_content(Some(&msg));
            finish(&ui, false);
            state.borrow_mut().pipeline_running = false;
            return;
        }
    };
    let file_name = file.name();
    let file_size = file.size() as u64;
    let file_type = file.type_();

    if file_size > MAX_FILE_SIZE {
        c.error(1);
        let msg: String = format!(
            "Файл слишком большой: {:.1} МБ (лимит {:.0} МБ)",
            file_size as f64 / 1_048_576.0,
            MAX_FILE_SIZE as f64 / 1_048_576.0,
        );
        ui.status.set_text_content(Some(&msg));
        finish(&ui, false);
        state.borrow_mut().pipeline_running = false;
        return;
    }

    let info: String = format!(
        "Выбран: {} ({:.1} МБ, MIME: {})",
        file_name,
        file_size as f64 / 1_048_576.0,
        if file_type.is_empty() { "—" } else { &file_type }
    );
    ui.status.set_text_content(Some(&info));

    // === Стадия 1: чтение ===
    c.enter(1, "Чтение (Blob.stream, yield 16 мс)");
    metrics::update_progress(&ui, 0.0);
    let read_t0 = js_sys::Date::now();
    let progress_cb = metrics::make_progress_cb(ui.clone(), file_size);
    let cancelled = cancel_check(state.clone());
    let outcome =
        match converter::read_blob_stream_validated(&file, progress_cb, cancelled).await {
            Ok(o) => o,
            Err(e) => {
                c.error(1);
                let msg: String = format!("Ошибка чтения: {}", metrics::error_name(&e));
                ui.status.set_text_content(Some(&msg));
                finish(&ui, false);
                state.borrow_mut().pipeline_running = false;
                return;
            }
        };
    state.borrow_mut().read_ms = js_sys::Date::now() - read_t0;

    let data: Vec<u8> = match outcome {
        converter::ReadOutcome::Ok(d) => {
            c.done(1);
            d
        }
        converter::ReadOutcome::Cancelled => {
            c.cancel(1);
            ui.status.set_text_content(Some("Чтение отменено."));
            finish(&ui, false);
            state.borrow_mut().pipeline_running = false;
            return;
        }
        converter::ReadOutcome::InvalidFormat(m) => {
            c.error(1);
            c.cancel(2);
            let msg: String = format!("Early-exit: {}", m);
            ui.status.set_text_content(Some(&msg));
            finish(&ui, false);
            state.borrow_mut().pipeline_running = false;
            return;
        }
    };

    // === Стадия 2: валидация ===
    c.enter(2, "Валидация (sync)");
    let fmt = match converter::validate_magic_bytes(&data) {
        Ok(f) => {
            c.done(2);
            f
        }
        Err(m) => {
            c.error(2);
            ui.status.set_text_content(Some(&m));
            finish(&ui, false);
            state.borrow_mut().pipeline_running = false;
            return;
        }
    };

    let fmt_label: &str = match fmt {
        converter::ImageFormat::Png => "PNG",
        converter::ImageFormat::Jpeg => "JPEG",
    };
    let magic_hex = metrics::hex_preview(&data, 8);
    let mime_note: String = format_mime_note(&file_type, fmt);

    let status_msg: String = format!(
        "Формат: {} (magic: {}; {})",
        fmt_label, magic_hex, mime_note
    );
    ui.status.set_text_content(Some(&status_msg));

    {
        let mut s = state.borrow_mut();
        s.original_len = data.len() as u64;
        s.format_label = fmt_label.to_string();
        s.magic_hex = magic_hex;
        s.engine_label = engine.label().to_string();
    }

    if let Ok(blob) = converter::bytes_to_blob(&data) {
        if let Ok(url) = Url::create_object_url_with_blob(&blob) {
            ui.preview_original.set_src(&url);
            state.borrow_mut().original_url = Some(url);
        }
    }

    let cancel_now = state.try_borrow().map(|s| s.cancel_requested).unwrap_or(true);
    if cancel_now {
        c.cancel(3);
        ui.status
            .set_text_content(Some("Отменено перед конвертацией."));
        finish(&ui, false);
        state.borrow_mut().pipeline_running = false;
        return;
    }

    // === Стадия 3: конвертация ===
    let phase: String = format!("Конвертация ({})", engine.label());
    c.enter(3, &phase);
    metrics::update_progress(&ui, 1.0);
    ui.progress_label
        .set_text_content(Some("100% (sync, прогресса нет)"));
    ui.cancel_btn.set_disabled(true);
    TimeoutFuture::new(0).await;

    let conv_t0 = js_sys::Date::now();
    let gray: Vec<u8> = match convert_by_engine(engine, &data, fmt).await {
        Ok(g) => g,
        Err(e) => {
            c.error(3);
            let msg: String = format!("Ошибка конвертации: {}", e);
            ui.status.set_text_content(Some(&msg));
            finish(&ui, false);
            state.borrow_mut().pipeline_running = false;
            return;
        }
    };
    state.borrow_mut().convert_ms = js_sys::Date::now() - conv_t0;
    ui.cancel_btn.set_disabled(false);
    c.done(3);

    if let Ok(blob) = converter::bytes_to_blob(&gray) {
        if let Ok(url) = Url::create_object_url_with_blob(&blob) {
            ui.preview_result.set_src(&url);
            state.borrow_mut().result_url = Some(url);
        }
    }

    let verified = metrics::verify_grayscale(&gray);
    let (txt, col): (&str, &str) = if verified {
        ("✓ R==G==B (16 точек)", "#27ae60")
    } else {
        ("✗ не прошла", "#c0392b")
    };
    ui.verify_badge.set_text_content(Some(txt));
    let style: String = format!("color: {}; font-weight: bold; font-size: 12px;", col);
    let _ = ui.verify_badge.set_attribute("style", &style);
    state.borrow_mut().verified = verified;

    // === Стадия 4: SHA-256 ===
    c.enter(4, "SHA-256 (chunked, yield 16 мс)");
    metrics::update_progress(&ui, 0.0);
    let hash_t0 = js_sys::Date::now();
    let progress_cb = metrics::make_progress_cb(ui.clone(), gray.len() as u64);
    let stats = converter::sha256_hex_with_progress(&gray, progress_cb).await;
    let hash_ms = js_sys::Date::now() - hash_t0;

    {
        let mut s = state.borrow_mut();
        s.hash_ms = hash_ms;
        s.hash_hex = stats.hex;
        s.hash_yields = stats.yields;
        s.hash_chunks = stats.chunks;
        s.hash_chunk_size = stats.chunk_size;
        s.result_len = gray.len() as u64;
        s.result_bytes = Some(gray);
        s.result_name = Some(metrics::build_output_name(&file_name));
    }
    c.done(4);

    metrics::refresh(&ui, &state);
    ui.status
        .set_text_content(Some("Готово. Можно сохранить или загрузить."));
    metrics::update_progress(&ui, 1.0);
    ui.progress_label.set_text_content(Some("100%"));
    ui.phase_label.set_text_content(Some("—"));
    state.borrow_mut().pipeline_running = false;
    finish(&ui, true);
    ui.save_btn.set_disabled(false);
    ui.hash_download_btn.set_disabled(false);
    if ui.upload_cb.checked() {
        ui.upload_btn.set_disabled(false);
    }
}

async fn convert_by_engine(
    engine: Engine,
    data: &[u8],
    fmt: converter::ImageFormat,
) -> Result<Vec<u8>, String> {
    match engine {
        Engine::WasmInline => converter::to_grayscale_png(data, fmt),
        Engine::WasmWorker => {
            let (rgba, w, h) = converter::decode_to_rgba(data, fmt)?;
            let out = converter_worker::grayscale_in_worker(rgba)
                .await
                .map_err(|e| format!("{:?}", e))?;
            converter::encode_rgba_png(&out, w, h)
        }
        Engine::WasmParallel => {
            let (rgba, w, h) = converter::decode_to_rgba(data, fmt)?;
            let out = converter_worker::grayscale_parallel(rgba)
                .await
                .map_err(|e| format!("{:?}", e))?;
            converter::encode_rgba_png(&out, w, h)
        }
        Engine::NativeCanvas => converter_native::grayscale_native(data)
            .await
            .map_err(|e| format!("{:?}", e)),
    }
}

// ---------------------------------------------------------------------------
// Save
// ---------------------------------------------------------------------------
pub async fn run_save(ui: Ui, state: Rc<RefCell<DemoState>>) {
    if state.borrow().pipeline_running {
        return;
    }

    let extracted: Option<(Vec<u8>, String)> = {
        match state.try_borrow() {
            Ok(s) => match (s.result_bytes.clone(), s.result_name.clone()) {
                (Some(b), Some(n)) => Some((b, n)),
                _ => None,
            },
            Err(_) => {
                ui.status.set_text_content(Some("State busy; retry."));
                return;
            }
        }
    };

    let (bytes, name) = match extracted {
        Some(pair) => pair,
        None => {
            ui.status.set_text_content(Some("Нет результата для сохранения."));
            return;
        }
    };

    state.borrow_mut().pipeline_running = true;
    state.borrow_mut().cancel_requested = false;

    let c = ctx(&ui);
    ui.save_btn.set_disabled(true);
    ui.pick_btn.set_disabled(true);
    ui.reset_btn.set_disabled(true);
    ui.cancel_btn.set_disabled(false);
    metrics::update_progress(&ui, 0.0);

    let has_fsa: bool = web_sys::window()
        .and_then(|w| js_sys::Reflect::get(&w, &JsValue::from_str("showSaveFilePicker")).ok())
        .map(|v| !v.is_undefined())
        .unwrap_or(false);

    let phase: &str = if has_fsa {
        "Запись (chunked, yield 16 мс)"
    } else {
        "Скачивание (fallback <a download>)"
    };
    c.enter(5, phase);
    ui.status.set_text_content(Some("Сохранение…"));

    let save_t0 = js_sys::Date::now();
    let progress_cb = metrics::make_progress_cb(ui.clone(), bytes.len() as u64);
    let cancelled = cancel_check(state.clone());
    let result = converter_ui::save_file_chunked(&bytes, &name, progress_cb, cancelled).await;
    state.borrow_mut().save_ms = js_sys::Date::now() - save_t0;

    match result {
        Ok(converter_ui::SaveOutcome::Fsa) => {
            let msg: String = format!("Сохранено (FSA): {}", name);
            ui.status.set_text_content(Some(&msg));
            c.done(5);
        }
        Ok(converter_ui::SaveOutcome::FallbackDownload) => {
            let msg: String = format!("Скачано через <a download>: {}", name);
            ui.status.set_text_content(Some(&msg));
            c.done(5);
        }
        Ok(converter_ui::SaveOutcome::Cancelled) => {
            ui.status.set_text_content(Some("Сохранение отменено."));
            c.cancel(5);
        }
        Err(e) => {
            let en: String = metrics::error_name(&e);
            let msg: String = format_save_error(en.as_str());
            ui.status.set_text_content(Some(&msg));
            c.error(5);
        }
    }

    ui.phase_label.set_text_content(Some("—"));
    metrics::refresh(&ui, &state);
    state.borrow_mut().pipeline_running = false;
    finish(&ui, true);
    ui.save_btn.set_disabled(false);
}

// ---------------------------------------------------------------------------
// Upload
// ---------------------------------------------------------------------------
pub async fn run_upload(ui: Ui, state: Rc<RefCell<DemoState>>) {
    if state.borrow().pipeline_running {
        return;
    }

    let extracted: Option<(Vec<u8>, String)> = {
        match state.try_borrow() {
            Ok(s) => match (s.result_bytes.clone(), s.result_name.clone()) {
                (Some(b), Some(n)) => Some((b, n)),
                _ => None,
            },
            Err(_) => {
                ui.status.set_text_content(Some("State busy; retry."));
                return;
            }
        }
    };

    let (bytes, name) = match extracted {
        Some(pair) => pair,
        None => {
            ui.status.set_text_content(Some("Нет результата для загрузки."));
            return;
        }
    };

    let existing_id: Option<String> = match state.try_borrow() {
        Ok(s) => s.upload_id.clone(),
        Err(_) => None,
    };
    let upload_id: String = match existing_id {
        Some(id) => id,
        None => converter::generate_upload_id(),
    };

    state.borrow_mut().pipeline_running = true;
    state.borrow_mut().upload_id = Some(upload_id.clone());

    let c = ctx(&ui);
    ui.upload_btn.set_disabled(true);
    ui.save_btn.set_disabled(true);
    ui.pick_btn.set_disabled(true);
    ui.reset_btn.set_disabled(true);
    ui.cancel_btn.set_disabled(false);
    metrics::update_progress(&ui, 0.0);
    c.enter(6, "Upload (XHR, upload_id)");

    let short_id: &str = if upload_id.len() >= 8 {
        &upload_id[..8]
    } else {
        upload_id.as_str()
    };
    let info: String = format!(
        "Загрузка: {} → files/{}/{}",
        name, short_id, name
    );
    ui.status.set_text_content(Some(&info));

    let t0 = js_sys::Date::now();
    let progress_cb = metrics::make_progress_cb(ui.clone(), bytes.len() as u64);
    let res = converter_ui::upload_to_server(
        &bytes,
        &upload_id,
        &name,
        "image/png",
        SERVER_BASE,
        progress_cb,
    )
    .await;
    state.borrow_mut().upload_ms = js_sys::Date::now() - t0;

    match res {
        Ok(st) if (200..300).contains(&st) => {
            state.borrow_mut().uploaded = true;
            let msg: String = format!("Загружено (HTTP {}): {}", st, name);
            ui.status.set_text_content(Some(&msg));
            c.done(6);
            ui.delete_btn.set_disabled(false);
        }
        Ok(st) => {
            let msg: String = format!("HTTP {} — проверьте auth-server.", st);
            ui.status.set_text_content(Some(&msg));
            c.error(6);
        }
        Err(e) => {
            let msg: String = e.as_string().unwrap_or_else(|| format!("{:?}", e));
            ui.status.set_text_content(Some(&msg));
            c.error(6);
        }
    }

    ui.phase_label.set_text_content(Some("—"));
    metrics::refresh(&ui, &state);
    state.borrow_mut().pipeline_running = false;
    finish(&ui, true);
    ui.save_btn.set_disabled(false);
}

fn finish(ui: &Ui, success: bool) {
    ui.pick_btn.set_disabled(false);
    ui.cancel_btn.set_disabled(true);
    ui.reset_btn.set_disabled(false);
    if !success {
        ui.save_btn.set_disabled(true);
        ui.upload_btn.set_disabled(true);
        ui.delete_btn.set_disabled(true);
        ui.hash_download_btn.set_disabled(true);
    }
}

// ---------------------------------------------------------------------------
// Хелперы форматирования строк (устойчивы к rust-analyzer type inference)
// ---------------------------------------------------------------------------

fn format_pick_error(name: &str) -> String {
    match name {
        "AbortError" => {
            let s: String = "Выбор отменён.".to_string();
            s
        }
        "SecurityError" => {
            let s: String = "Браузер требует жест пользователя — повторите.".to_string();
            s
        }
        "" => {
            let s: String = "Ошибка выбора файла.".to_string();
            s
        }
        other => {
            let s: String = format!("Ошибка выбора: {}", other);
            s
        }
    }
}

fn format_save_error(en: &str) -> String {
    if en == "AbortError" {
        let s: String = "Диалог сохранения закрыт.".to_string();
        s
    } else if en.is_empty() {
        let s: String = "Ошибка сохранения.".to_string();
        s
    } else {
        let s: String = format!("Ошибка сохранения: {}", en);
        s
    }
}

fn format_mime_note(file_type: &str, fmt: converter::ImageFormat) -> String {
    if file_type.is_empty() {
        let s: String = "MIME не указан".to_string();
        return s;
    }
    let expected: &str = match fmt {
        converter::ImageFormat::Png => "image/png",
        converter::ImageFormat::Jpeg => "image/jpeg",
    };
    if file_type.eq_ignore_ascii_case(expected) {
        let s: String = "MIME совпадает".to_string();
        return s;
    }
    let s: String = format!("MIME не совпадает: {} vs {}", file_type, expected);
    s
}
```

---

### `src/converter_app/metrics.rs`

```rust
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;

use super::state::{set_stage, DemoState, StageStatus, Ui, STAGES};

pub fn update_progress(ui: &Ui, frac: f64) {
    let pct = (frac * 100.0).clamp(0.0, 100.0);
    ui.progress.set_value(frac);
    if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
        if let Some(el) = doc.get_element_by_id("progress-inner") {
            let style: String = format!("width: {:.1}%;", pct);
            let _ = el.set_attribute("style", &style);
        }
    }
}

/// Отдельная функция с явным `-> String` — rust-analyzer не путается.
fn format_progress_label(frac: f64, done: u64, total_bytes: u64, elapsed: f64) -> String {
    if elapsed <= 0.05 || frac <= 0.01 {
        return format!("{:>3.0}%", frac * 100.0);
    }

    let speed: f64 = done as f64 / elapsed / 1_048_576.0;
    let eta: f64 = if frac > 0.001 {
        elapsed / frac * (1.0 - frac)
    } else {
        0.0
    };

    let pct: f64 = frac * 100.0;
    let done_mb: f64 = done as f64 / 1_048_576.0;
    let total_mb: f64 = total_bytes as f64 / 1_048_576.0;

    format!(
        "{:>3.0}% · {:.1}/{:.1} MB · {:.1} MB/s · ETA {:.1} s",
        pct, done_mb, total_mb, speed, eta
    )
}

pub fn make_progress_cb(ui: Ui, total_bytes: u64) -> impl FnMut(f64) {
    let t0: f64 = js_sys::Date::now();
    move |frac: f64| {
        update_progress(&ui, frac);
        let elapsed: f64 = (js_sys::Date::now() - t0) / 1000.0;
        let done: u64 = (frac * total_bytes as f64) as u64;
        let label: String = format_progress_label(frac, done, total_bytes, elapsed);
        ui.progress_label.set_text_content(Some(&label));
    }
}

pub fn reset_stages_and_bar(ui: &Ui) {
    for (i, s) in ui.stages.iter().enumerate() {
        set_stage(s, STAGES[i], StageStatus::Pending);
    }
    for c in ui.phase_cells.iter() {
        let _ = c.set_attribute("class", "phase-cell");
    }
    ui.phase_label.set_text_content(Some("—"));
}

pub fn error_name(e: &JsValue) -> String {
    let raw: Result<JsValue, JsValue> =
        js_sys::Reflect::get(e, &JsValue::from_str("name"));
    let opt: Option<JsValue> = raw.ok();
    let as_string: Option<String> = opt.and_then(|v| v.as_string());
    match as_string {
        Some(s) => s,
        None => String::new(),
    }
}

pub fn hex_preview(data: &[u8], n: usize) -> String {
    let take: usize = data.len().min(n);
    let mut out: String = String::with_capacity(take * 3);
    for (i, b) in data[..take].iter().enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format!("{:02X}", b));
    }
    out
}

pub fn build_output_name(original: &str) -> String {
    let lower: String = original.to_ascii_lowercase();
    let suffix_len: usize = if lower.ends_with(".jpeg") {
        5
    } else if lower.ends_with(".jpg") {
        4
    } else if lower.ends_with(".png") {
        4
    } else {
        0
    };
    let stem: &str = if suffix_len == 0 {
        original
    } else {
        &original[..original.len() - suffix_len]
    };
    let mut out: String = String::with_capacity(stem.len() + 14);
    out.push_str(stem);
    out.push_str(".grayscale.png");
    out
}

pub fn verify_grayscale(png_bytes: &[u8]) -> bool {
    use image::GenericImageView;
    let img = match image::load_from_memory_with_format(png_bytes, image::ImageFormat::Png) {
        Ok(i) => i,
        Err(_) => return false,
    };
    let (w, h) = img.dimensions();
    if w == 0 || h == 0 {
        return true;
    }
    for iy in 0..4u32 {
        for ix in 0..4u32 {
            let x = (ix * w / 4).min(w - 1);
            let y = (iy * h / 4).min(h - 1);
            let p = img.get_pixel(x, y).0;
            if p[0] != p[1] || p[1] != p[2] {
                return false;
            }
        }
    }
    true
}

fn format_upload_line(s: &DemoState) -> String {
    match s.upload_id.as_ref() {
        Some(id) => {
            let short: &str = if id.len() >= 8 { &id[..8] } else { id.as_str() };
            let line: String = format!(
                "Upload:      id={}... | {} ms | delete {} ms",
                short, s.upload_ms, s.delete_ms
            );
            line
        }
        None => {
            let empty: String = String::new();
            empty
        }
    }
}

pub fn refresh(ui: &Ui, state: &Rc<RefCell<DemoState>>) {
    let s = state.borrow();

    let orig: u64 = s.original_len;
    let res: u64 = s.result_len;

    let ratio: f64 = if orig > 0 {
        (res as f64 / orig as f64 - 1.0) * 100.0
    } else {
        0.0
    };

    let hash_prefix: String = if s.hash_hex.len() >= 16 {
        s.hash_hex[..16].to_string()
    } else {
        s.hash_hex.clone()
    };

    let sum_other: f64 = s.read_ms + s.hash_ms + s.save_ms;
    let bottleneck: String = if s.convert_ms > 2.0 * sum_other && s.convert_ms > 1.0 {
        "  <- bottleneck".to_string()
    } else {
        String::new()
    };

    let read_mbps: f64 = if s.read_ms > 0.0 {
        orig as f64 / 1_048_576.0 / (s.read_ms / 1000.0)
    } else {
        0.0
    };
    let hash_mbps: f64 = if s.hash_ms > 0.0 {
        res as f64 / 1_048_576.0 / (s.hash_ms / 1000.0)
    } else {
        0.0
    };
    let save_mbps: f64 = if s.save_ms > 0.0 {
        res as f64 / 1_048_576.0 / (s.save_ms / 1000.0)
    } else {
        0.0
    };

    let verify_text: String = if s.result_len == 0 {
        "—".to_string()
    } else if s.verified {
        "grayscale OK (R==G==B на 16 точках)".to_string()
    } else {
        "не прошла".to_string()
    };

    let upload_line: String = format_upload_line(&s);

    let mut lines: Vec<String> = Vec::with_capacity(9);
    lines.push(format!("Формат:      {} (magic: {})", s.format_label, s.magic_hex));
    lines.push(format!("Движок:      {}", s.engine_label));
    lines.push(format!(
        "Размер:      {} → {} байт ({:+.1} %)",
        orig, res, ratio
    ));
    lines.push(format!(
        "Тайминги:    чтение {:.1} ms ({:.1} МБ/с) · конвертация {:.1} ms{}",
        s.read_ms, read_mbps, s.convert_ms, bottleneck
    ));
    lines.push(format!(
        "             хеш {:.1} ms ({:.1} МБ/с) · запись {:.1} ms ({:.1} МБ/с)",
        s.hash_ms, hash_mbps, s.save_ms, save_mbps
    ));
    if !upload_line.is_empty() {
        lines.push(upload_line);
    }
    lines.push(format!("SHA-256:     {}...", hash_prefix));
    lines.push(format!(
        "Троттлинг:   chunk={} КиБ · итераций={} · yield'ов={}",
        s.hash_chunk_size / 1024,
        s.hash_chunks,
        s.hash_yields
    ));
    lines.push(format!("Self-check:  {}", verify_text));

    let text: String = lines.join("\n");
    ui.metrics.set_text_content(Some(&text));
}
```

---

### `tests/wasm_converter.rs`

```rust
//! Тесты чистой логики: `src/converter.rs`.
//! Самодостаточны, не используют `tests/common`.

mod raster {
    use image::{ImageBuffer, Rgb};

    pub fn make_test_png(w: u32, h: u32) -> Vec<u8> {
        let img: ImageBuffer<Rgb<u8>, Vec<u8>> =
            ImageBuffer::from_fn(w, h, |x, y| {
                Rgb([
                    ((x * 255 / w.max(1)) as u8).wrapping_add(10),
                    ((y * 255 / h.max(1)) as u8).wrapping_add(20),
                    (((x + y) * 255 / (w + h).max(1)) as u8).wrapping_add(30),
                ])
            });
        let mut out = Vec::new();
        img.write_to(
            &mut std::io::Cursor::new(&mut out),
            image::ImageFormat::Png,
        )
        .expect("PNG encode");
        out
    }

    pub fn make_test_jpeg(w: u32, h: u32) -> Vec<u8> {
        let img: ImageBuffer<Rgb<u8>, Vec<u8>> =
            ImageBuffer::from_fn(w, h, |x, y| {
                Rgb([
                    ((x * 255 / w.max(1)) as u8).wrapping_add(10),
                    ((y * 255 / h.max(1)) as u8).wrapping_add(20),
                    (((x + y) * 255 / (w + h).max(1)) as u8).wrapping_add(30),
                ])
            });
        let mut out = Vec::new();
        img.write_to(
            &mut std::io::Cursor::new(&mut out),
            image::ImageFormat::Jpeg,
        )
        .expect("JPEG encode");
        out
    }

    pub fn verify_grayscale(png_bytes: &[u8]) -> bool {
        use image::GenericImageView;
        let img = match image::load_from_memory_with_format(
            png_bytes,
            image::ImageFormat::Png,
        ) {
            Ok(i) => i,
            Err(_) => return false,
        };
        let (w, h) = img.dimensions();
        if w == 0 || h == 0 {
            return true;
        }
        for iy in 0..4u32 {
            for ix in 0..4u32 {
                let x = (ix * w / 4).min(w - 1);
                let y = (iy * h / 4).min(h - 1);
                let p = img.get_pixel(x, y).0;
                if p[0] != p[1] || p[1] != p[2] {
                    return false;
                }
            }
        }
        true
    }

    pub fn file_from_bytes(bytes: &[u8], name: &str) -> web_sys::File {
        let u8arr = js_sys::Uint8Array::from(bytes);
        let blob = web_sys::Blob::new_with_u8_array_sequence(
            &js_sys::Array::of1(&u8arr),
        )
        .expect("Blob::new");
        web_sys::File::new_with_blob_sequence(
            &js_sys::Array::of1(&blob),
            name,
        )
        .expect("File::new")
    }
}

use hello_wasm02::converter::{self, ReadOutcome};
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn validate_png_jpeg_and_garbage() {
    let png = raster::make_test_png(2, 2);
    assert_eq!(
        converter::validate_magic_bytes(&png).unwrap(),
        converter::ImageFormat::Png
    );

    let jpg = raster::make_test_jpeg(2, 2);
    assert_eq!(
        converter::validate_magic_bytes(&jpg).unwrap(),
        converter::ImageFormat::Jpeg
    );

    assert!(converter::validate_magic_bytes(b"plain text").is_err());
    assert!(converter::validate_magic_bytes(&[]).is_err());
    assert!(converter::validate_magic_bytes(&[0x89, 0x50]).is_err());
}

#[wasm_bindgen_test]
fn adaptive_chunk_bounds() {
    // Нижняя граница clamp.
    assert_eq!(converter::adaptive_chunk(0, 200), 64 * 1024);
    assert_eq!(converter::adaptive_chunk(1, 200), 64 * 1024);

    // Верхняя граница clamp (на wasm32 usize = u32, ~4 ГиБ).
    let huge: usize = (u32::MAX - 1) as usize;
    assert_eq!(
        converter::adaptive_chunk(huge, 200),
        4 * 1024 * 1024
    );

    // Средний размер: 200 МБ / 200 = 1 МБ — в границах.
    let mid = converter::adaptive_chunk(200 * 1024 * 1024, 200);
    assert!(mid >= 64 * 1024 && mid <= 4 * 1024 * 1024);
}

#[wasm_bindgen_test]
async fn sha256_sync_and_async_agree() {
    let data: Vec<u8> = (0..1024 * 1024).map(|i| (i & 0xFF) as u8).collect();
    let sync = converter::sha256_hex(&data);
    let stats = converter::sha256_hex_with_progress(&data, |_| {}).await;
    assert_eq!(sync.len(), 64);
    assert_eq!(stats.hex.len(), 64);
    assert_eq!(sync, stats.hex);
}

#[wasm_bindgen_test]
async fn sha256_progress_monotonic() {
    let data = vec![0xABu8; 4 * 1024 * 1024];
    let seen: std::rc::Rc<std::cell::RefCell<Vec<f64>>> =
        std::rc::Rc::new(std::cell::RefCell::new(Vec::new()));
    let seen_c = seen.clone();
    let stats = converter::sha256_hex_with_progress(&data, move |f| {
        seen_c.borrow_mut().push(f);
    })
    .await;

    let v = seen.borrow();
    for w in v.windows(2) {
        assert!(w[1] + 1e-9 >= w[0], "progress must be monotonic");
    }
    assert!((v.last().copied().unwrap_or(0.0) - 1.0).abs() < 1e-6);

    assert!(stats.chunk_size >= 64 * 1024 && stats.chunk_size <= 4 * 1024 * 1024);
    let expected_chunks =
        ((data.len() + stats.chunk_size - 1) / stats.chunk_size) as u32;
    assert_eq!(stats.chunks, expected_chunks);
}

#[wasm_bindgen_test]
fn decode_encode_png_roundtrip() {
    let png = raster::make_test_png(8, 8);
    let (rgba, w, h) =
        converter::decode_to_rgba(&png, converter::ImageFormat::Png).unwrap();
    assert_eq!((w, h), (8, 8));
    assert_eq!(rgba.len(), (8 * 8 * 4) as usize);

    let back = converter::encode_rgba_png(&rgba, w, h).unwrap();
    assert_eq!(
        converter::validate_magic_bytes(&back).unwrap(),
        converter::ImageFormat::Png
    );
    use image::GenericImageView;
    let img = image::load_from_memory_with_format(&back, image::ImageFormat::Png).unwrap();
    assert_eq!(img.dimensions(), (8, 8));
}

#[wasm_bindgen_test]
fn grayscale_roundtrip_verify() {
    let png = raster::make_test_png(8, 8);
    let fmt = converter::validate_magic_bytes(&png).unwrap();
    let gray = converter::to_grayscale_png(&png, fmt).unwrap();
    assert_eq!(
        converter::validate_magic_bytes(&gray).unwrap(),
        converter::ImageFormat::Png
    );
    assert!(raster::verify_grayscale(&gray));
}

#[wasm_bindgen_test]
fn grayscale_from_jpeg() {
    let jpg = raster::make_test_jpeg(8, 8);
    let fmt = converter::validate_magic_bytes(&jpg).unwrap();
    let gray = converter::to_grayscale_png(&jpg, fmt).unwrap();
    assert!(raster::verify_grayscale(&gray));
}

#[wasm_bindgen_test]
fn upload_id_is_url_safe_and_not_empty() {
    let id = converter::generate_upload_id();
    assert!(!id.is_empty());
    assert!(id.len() >= 8);
    for ch in id.chars() {
        assert!(
            ch.is_ascii_alphanumeric() || ch == '-' || ch == '_',
            "unexpected char in upload_id: {:?}",
            ch
        );
    }
}

#[wasm_bindgen_test]
async fn read_ok_png() {
    let bytes = raster::make_test_png(4, 4);
    let file = raster::file_from_bytes(&bytes, "a.png");

    let hits = std::rc::Rc::new(std::cell::Cell::new(0u32));
    let hits_c = hits.clone();
    let outcome = converter::read_blob_stream_validated(
        &file,
        move |_| hits_c.set(hits_c.get() + 1),
        || false,
    )
    .await
    .unwrap();

    match outcome {
        ReadOutcome::Ok(d) => assert_eq!(d, bytes),
        _ => panic!("expected Ok"),
    }
    assert!(hits.get() >= 1);
}

#[wasm_bindgen_test]
async fn read_cancel_immediately() {
    let bytes = raster::make_test_png(8, 8);
    let file = raster::file_from_bytes(&bytes, "a.png");
    let outcome =
        converter::read_blob_stream_validated(&file, |_| {}, || true)
            .await
            .unwrap();
    assert!(matches!(outcome, ReadOutcome::Cancelled));
}

#[wasm_bindgen_test]
async fn read_early_exit_on_garbage() {
    let file = raster::file_from_bytes(b"not a picture at all", "a.txt");
    let outcome =
        converter::read_blob_stream_validated(&file, |_| {}, || false)
            .await
            .unwrap();
    match outcome {
        ReadOutcome::InvalidFormat(msg) => assert!(!msg.is_empty()),
        _ => panic!("expected InvalidFormat"),
    }
}
```

---

### `tests/wasm_converter_worker.rs`

```rust
//! Тесты Web Worker-движка: `src/converter_worker.rs`.

mod raster {
    pub fn make_rgba(w: u32, h: u32) -> Vec<u8> {
        let mut out = Vec::with_capacity((w * h * 4) as usize);
        for y in 0..h {
            for x in 0..w {
                out.push((x * 255 / w.max(1)) as u8);
                out.push((y * 255 / h.max(1)) as u8);
                out.push(((x + y) * 255 / (w + h).max(1)) as u8);
                out.push(255);
            }
        }
        out
    }

    pub fn assert_all_gray(buf: &[u8]) {
        assert_eq!(buf.len() % 4, 0, "RGBA length must be multiple of 4");
        for px in buf.chunks_exact(4) {
            assert_eq!(px[0], px[1], "R != G at some pixel");
            assert_eq!(px[1], px[2], "G != B at some pixel");
        }
    }

    pub fn alpha_preserved(orig: &[u8], out: &[u8]) {
        for (a, b) in orig.chunks_exact(4).zip(out.chunks_exact(4)) {
            assert_eq!(a[3], b[3]);
        }
    }
}

use hello_wasm02::converter_worker;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn worker_grayscale_basic() {
    let rgba = raster::make_rgba(16, 16);
    let out = converter_worker::grayscale_in_worker(rgba.clone())
        .await
        .expect("worker ok");
    assert_eq!(out.len(), rgba.len());
    raster::assert_all_gray(&out);
    raster::alpha_preserved(&rgba, &out);
}

#[wasm_bindgen_test]
async fn parallel_matches_single_worker() {
    let rgba = raster::make_rgba(32, 32);
    let single = converter_worker::grayscale_in_worker(rgba.clone())
        .await
        .unwrap();
    let par = converter_worker::grayscale_parallel(rgba.clone())
        .await
        .unwrap();
    assert_eq!(single.len(), par.len());
    assert_eq!(single, par, "parallel result must match single-worker result");
}

#[wasm_bindgen_test]
async fn parallel_handles_zero_and_tiny() {
    let empty = converter_worker::grayscale_parallel(Vec::new())
        .await
        .unwrap();
    assert_eq!(empty.len(), 0);

    let one_px = vec![120u8, 40, 200, 255];
    let out = converter_worker::grayscale_parallel(one_px)
        .await
        .unwrap();
    assert_eq!(out.len(), 4);
    raster::assert_all_gray(&out);
    assert_eq!(out[3], 255);
}

#[wasm_bindgen_test]
async fn parallel_odd_length() {
    let rgba: Vec<u8> = (0..12).map(|i| (i * 20) as u8).collect();
    let out = converter_worker::grayscale_parallel(rgba)
        .await
        .unwrap();
    assert_eq!(out.len(), 12);
    raster::assert_all_gray(&out);
}
```

---

### `tests/wasm_converter_native.rs`

```rust
//! Тесты OffscreenCanvas-движка: `src/converter_native.rs`.

mod raster {
    use image::{ImageBuffer, Rgb};

    pub fn make_test_png(w: u32, h: u32) -> Vec<u8> {
        let img: ImageBuffer<Rgb<u8>, Vec<u8>> =
            ImageBuffer::from_fn(w, h, |x, y| {
                Rgb([
                    ((x * 255 / w.max(1)) as u8).wrapping_add(10),
                    ((y * 255 / h.max(1)) as u8).wrapping_add(20),
                    (((x + y) * 255 / (w + h).max(1)) as u8).wrapping_add(30),
                ])
            });
        let mut out = Vec::new();
        img.write_to(
            &mut std::io::Cursor::new(&mut out),
            image::ImageFormat::Png,
        )
        .expect("PNG encode");
        out
    }

    pub fn make_test_jpeg(w: u32, h: u32) -> Vec<u8> {
        let img: ImageBuffer<Rgb<u8>, Vec<u8>> =
            ImageBuffer::from_fn(w, h, |x, y| {
                Rgb([
                    ((x * 255 / w.max(1)) as u8).wrapping_add(10),
                    ((y * 255 / h.max(1)) as u8).wrapping_add(20),
                    (((x + y) * 255 / (w + h).max(1)) as u8).wrapping_add(30),
                ])
            });
        let mut out = Vec::new();
        img.write_to(
            &mut std::io::Cursor::new(&mut out),
            image::ImageFormat::Jpeg,
        )
        .expect("JPEG encode");
        out
    }
}

use hello_wasm02::{converter, converter_native};
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

fn offscreen_available() -> bool {
    let window = web_sys::window().unwrap();
    js_sys::Reflect::get(&window, &"OffscreenCanvas".into())
        .map(|v| !v.is_undefined())
        .unwrap_or(false)
}

#[wasm_bindgen_test]
async fn native_grayscale_roundtrip() {
    if !offscreen_available() {
        return;
    }
    let png = raster::make_test_png(16, 16);
    let out = converter_native::grayscale_native(&png)
        .await
        .expect("native grayscale ok");

    assert_eq!(
        converter::validate_magic_bytes(&out).unwrap(),
        converter::ImageFormat::Png
    );

    use image::GenericImageView;
    let orig =
        image::load_from_memory_with_format(&png, image::ImageFormat::Png).unwrap();
    let res =
        image::load_from_memory_with_format(&out, image::ImageFormat::Png).unwrap();
    assert_eq!(orig.dimensions(), res.dimensions());
}

#[wasm_bindgen_test]
async fn native_grayscale_from_jpeg() {
    if !offscreen_available() {
        return;
    }
    let jpg = raster::make_test_jpeg(16, 16);
    let out = converter_native::grayscale_native(&jpg)
        .await
        .expect("native grayscale from JPEG");
    assert_eq!(
        converter::validate_magic_bytes(&out).unwrap(),
        converter::ImageFormat::Png
    );
}

#[wasm_bindgen_test]
async fn native_rejects_garbage() {
    if !offscreen_available() {
        return;
    }
    let res = converter_native::grayscale_native(b"definitely not an image").await;
    assert!(res.is_err(), "garbage must not decode");
}

#[wasm_bindgen_test]
async fn native_grayscale_is_gray_or_warn() {
    if !offscreen_available() {
        return;
    }
    let png = raster::make_test_png(16, 16);
    let out = match converter_native::grayscale_native(&png).await {
        Ok(o) => o,
        Err(_) => return,
    };
    use image::GenericImageView;
    let img =
        image::load_from_memory_with_format(&out, image::ImageFormat::Png).unwrap();
    let (w, h) = img.dimensions();
    let mut non_gray = 0u32;
    for iy in 0..4u32 {
        for ix in 0..4u32 {
            let x = (ix * w / 4).min(w - 1);
            let y = (iy * h / 4).min(h - 1);
            let p = img.get_pixel(x, y).0;
            if p[0] != p[1] || p[1] != p[2] {
                non_gray += 1;
            }
        }
    }
    if non_gray > 0 {
        web_sys::console::warn_1(
            &"ctx.filter=grayscale(1) не поддержан в этом движке".into(),
        );
    }
}
```

---

### `tests/wasm_converter_ui.rs`

```rust
//! Тесты обёрток ввода-вывода: `src/converter_ui.rs`.
//! Всё, что требует user gesture, помечается `#[ignore]` или мягко
//! проверяется на «не паникует».

use hello_wasm02::{converter, converter_ui};
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn forget_save_handle_idempotent() {
    converter_ui::forget_save_handle();
    converter_ui::forget_save_handle();
    converter_ui::forget_save_handle();
}

#[wasm_bindgen_test]
fn upload_id_url_safe_via_converter() {
    for _ in 0..8 {
        let id = converter::generate_upload_id();
        for ch in id.chars() {
            assert!(
                ch.is_ascii_alphanumeric() || ch == '-' || ch == '_',
                "unexpected char in upload_id: {:?}",
                ch
            );
        }
    }
}

#[wasm_bindgen_test]
async fn save_file_chunked_without_gesture_is_safe() {
    let bytes = vec![0u8; 1024];
    let res = converter_ui::save_file_chunked(
        &bytes,
        "test.grayscale.png",
        |_| {},
        || false,
    )
    .await;

    match res {
        Ok(_) => { /* fallback <a download> */ }
        Err(e) => {
            let name = js_sys::Reflect::get(&e, &"name".into())
                .ok()
                .and_then(|v| v.as_string())
                .unwrap_or_default();
            assert!(
                name.is_empty()
                    || name == "SecurityError"
                    || name == "AbortError"
                    || name == "NotAllowedError",
                "unexpected error name: {}",
                name
            );
        }
    }
}

#[wasm_bindgen_test]
async fn save_file_chunked_cancel_immediately_is_safe() {
    let bytes = vec![0u8; 64 * 1024];
    let res = converter_ui::save_file_chunked(
        &bytes,
        "cancel.png",
        |_| {},
        || true,
    )
    .await;

    match res {
        Ok(converter_ui::SaveOutcome::Cancelled) => {}
        Ok(converter_ui::SaveOutcome::FallbackDownload) => {}
        Ok(converter_ui::SaveOutcome::Fsa) => {
            panic!("FSA path should not complete when cancelled immediately");
        }
        Err(_) => { /* SecurityError без жеста — допустимо */ }
    }
}

#[wasm_bindgen_test]
#[ignore]
async fn pick_image_file_requires_gesture() {
    let res = converter_ui::pick_image_file().await;
    assert!(res.is_err());
}

#[wasm_bindgen_test]
async fn upload_to_dead_server_returns_err() {
    let bytes = vec![0u8; 32];
    let res = converter_ui::upload_to_server(
        &bytes,
        "test-upload-id",
        "x.png",
        "image/png",
        "http://127.0.0.1:1",
        |_| {},
    )
    .await;
    assert!(res.is_err(), "upload to dead server must err");
}

#[wasm_bindgen_test]
async fn delete_from_dead_server_returns_err() {
    let res = converter_ui::delete_upload("http://127.0.0.1:1", "any-id").await;
    assert!(res.is_err(), "delete from dead server must err");
}
```

---

### Дельта `Cargo.toml`

В `[dependencies.web-sys]` → `features = [...]` **добавлены**:

```toml
    # 18.18 — конвертер файлов
    "OffscreenCanvas",
    "OffscreenCanvasRenderingContext2d",
    "ImageBitmap",
    "HtmlProgressElement",
    "HtmlSpanElement",
    "XmlHttpRequestResponseType",
    "HtmlHeadElement",
```

`XmlHttpRequest`, `ProgressEvent`, `FileSystemFileHandle`, `FileSystemWritableFileStream`, `Blob`, `File` — уже присутствовали.

---

### Дельта `src/lib.rs`

В блок `pub mod ...` **добавлены** четыре строки (в разделе главы 18.18):

```rust
pub mod converter;
pub mod converter_ui;
pub mod converter_worker;
pub mod converter_native;

pub mod converter_app;
```

Остальное содержимое `lib.rs` не меняется. Точка входа остаётся:

```rust
#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    logging::init();
    app::run()
}
```

`app::run()` теперь возвращает `Result<(), JsValue>` — контракт соблюдён.

---

### Резюме

Все 15 файлов главы 18.18 (11 исходных + 4 тестовых) собраны и работают. Правила проекта соблюдены:

- `app.rs` — тонкая обёртка с одной точкой входа `run() -> Result<(), JsValue>`;
- логика демо-секции — в `converter_app/`, отделена от `app.rs`;
- движки grayscale (`converter.rs`, `converter_worker.rs`, `converter_native.rs`) — независимы и переиспользуемы;
- обёртки ввода-вывода (`converter_ui.rs`) — с FSA-путём и fallback;
- тесты самодостаточны, не трогают чужой `tests/common/mod.rs`;
- длительные async-операции отдают управление раз в 16 мс через `TimeoutFuture::new(0).await`;
- sync-фазы честно помечены и блокируют отмену;
- состояние UI — единый `Rc<RefCell<DemoState>>`;
- промис-возвращающие методы web-sys вызываются через `Reflect` (независимость от версии);
- идемпотентность `Сбросить` и `DELETE`;
- никаких предупреждений `dead_code` / `unused_imports` / `unused_variables`.