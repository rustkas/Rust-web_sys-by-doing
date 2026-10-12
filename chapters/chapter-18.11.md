# Глава 18. File API

## 18.11. Web Workers для файловой обработки

До сих пор всё, что мы делали с файлами, выполнялось в **главном потоке** — том же, что рисует страницу, обрабатывает клики и анимации. Пока операции короткие, это незаметно. Но стоит запустить чтение 500-мегабайтного файла или цикл конвертации на несколько секунд — UI замрёт: кнопки не нажимаются, прокрутка не работает, браузер показывает «страница не отвечает». Решение — **Web Worker**: отдельный поток, в котором можно выполнять тяжёлую работу, не блокируя UI.

В этой подглаве разберём, как передать `File` в воркер, как читать его через `FileReaderSync` (доступен только в воркере) и как запустить несколько воркеров параллельно на разные файлы.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── Cargo.toml                     # без изменений в этой подглаве
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod worker_file;
│   ├── app.rs                     # ОБНОВЛЁН: только секция 18.11
│   ├── worker_file.rs             # НОВЫЙ: модуль подглавы 18.11
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── logging.rs                 # существует
│   └── … остальные модули проекта без изменений
├── web/
│   └── public/
│       └── workers/
│           └── file_worker.js     # НОВЫЙ: JS-воркер
└── tests/
    └── wasm_worker_file.rs        # НОВЫЙ: тесты подглавы 18.11
```

**Пояснения:**

- `src/worker_file.rs` — Rust-модуль: создаёт `Worker`, отправляет `File` через `postMessage`, ждёт ответ.
- `web/public/workers/file_worker.js` — JS-воркер: получает `File`, читает через `FileReaderSync`, отправляет результат обратно. Vite раздаёт `web/public/*` по корню.
- `src/app.rs` — **только** код подглавы 18.11. Другие секции главы 18 в этом файле не присутствуют.
- `src/lib.rs` — дополняется строкой `pub mod worker_file;`.
- `tests/wasm_worker_file.rs` — тесты подглавы.

Все нужные фичи web-sys (`Worker`, `MessageEvent`, `Blob`, `Url`) уже включены в `Cargo.toml`.

---

## Ссылки для детального изучения

- **MDN: Web Workers API** — https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API
- **MDN: Worker** — https://developer.mozilla.org/en-US/docs/Web/API/Worker
- **MDN: DedicatedWorkerGlobalScope** — https://developer.mozilla.org/en-US/docs/Web/API/DedicatedWorkerGlobalScope
- **MDN: FileReaderSync** — https://developer.mozilla.org/en-US/docs/Web/API/FileReaderSync
- **MDN: Transferable objects** — https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects
- **web-sys: Worker** — https://docs.rs/web-sys/latest/web_sys/struct.Worker.html
- **web-sys: MessageEvent** — https://docs.rs/web-sys/latest/web_sys/struct.MessageEvent.html

---

## Содержание новых / обновлённых файлов

### `src/lib.rs`

Добавьте одну строку:

```rust
pub mod worker_file;    // 18.11
```

---

### `web/public/workers/file_worker.js`

Классический (не модульный) воркер. Слушает `message`, читает переданный `File` синхронно через `FileReaderSync`, отправляет результат обратно.

```js
// Web Worker для чтения файлов.
//
// Принимает сообщение { type: "read", id, file }, читает первые
// 64 байта через FileReaderSync (доступен только в воркере)
// и отправляет ответ { type: "result", id, name, size, headHex, headLen }.
//
// При ошибке — { type: "error", id, message }.

const HEAD_SIZE = 64;

self.onmessage = (event) => {
    const data = event.data;
    if (!data || data.type !== "read") {
        self.postMessage({ type: "error", message: "unknown message type" });
        return;
    }

    const file = data.file;
    const id = data.id;

    if (!file) {
        self.postMessage({ type: "error", id, message: "no file" });
        return;
    }

    try {
        // FileReaderSync доступен только внутри Worker — в главном
        // потоке его вызов бросает InvalidStateError.
        const reader = new FileReaderSync();
        const head = file.slice(0, HEAD_SIZE);
        const buffer = reader.readAsArrayBuffer(head);
        const uint8 = new Uint8Array(buffer);

        let hex = "";
        for (let i = 0; i < uint8.length; i++) {
            if (i > 0) {
                hex += " ";
            }
            hex += uint8[i].toString(16).padStart(2, "0").toUpperCase();
        }

        self.postMessage({
            type: "result",
            id,
            name: file.name,
            size: file.size,
            headHex: hex,
            headLen: uint8.length,
        });
    } catch (err) {
        self.postMessage({
            type: "error",
            id,
            message: String(err),
        });
    }
};
```

**Ключевое:**

- **`FileReaderSync` доступен только в воркере.** В главном потоке его вызов бросает `InvalidStateError`.
- **`file.slice(0, HEAD_SIZE)`** — читаем только заголовок. Для полноценной обработки можно читать весь файл.
- **Никаких `importScripts`** — воркер самодостаточен.

---

### `src/worker_file.rs`

Rust-обёртка над `Worker`. Создаёт воркер, отправляет `File` через `postMessage`, оборачивает ответ в `Promise`, ждёт результат, терминирует воркер.

```rust
//! Web Workers для файловой обработки.
//!
//! Модуль автономен: зависит только от `web-sys`, `js-sys`,
//! `wasm-bindgen`, `wasm-bindgen-futures`.
//!
//! `File` передаётся в воркер через `postMessage` по structured
//! clone. Это дешёвая операция: в браузере `File` — это ссылка
//! на blob-объект, а не копия содержимого.
//!
//! `FileReaderSync` доступен только внутри воркера — в главном
//! потоке его вызов бросает `InvalidStateError`. Поэтому чтение
//! вынесено в отдельный поток: главный остаётся отзывчивым.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{MessageEvent, Worker};

/// Результат чтения файла в воркере.
#[derive(Debug, Clone, PartialEq)]
pub struct WorkerReadResult {
    /// Имя файла.
    pub name: String,
    /// Размер файла в байтах.
    pub size: f64,
    /// Первые 64 байта в hex-представлении (через пробел).
    pub head_hex: String,
    /// Сколько байт реально прочитано (может быть меньше 64,
    /// если файл короче).
    pub head_len: u32,
}

/// Читает `File` в отдельном Worker и возвращает первые 64 байта.
///
/// Внутри воркера используется `FileReaderSync` — синхронное
/// чтение, доступное только в Worker-контексте. Воркер создаётся
/// на каждый вызов и терминируется после получения ответа.
pub async fn read_file_in_worker(
    file: &web_sys::File,
    worker_url: &str,
) -> Result<WorkerReadResult, JsValue> {
    let worker = Worker::new(worker_url)?;

    // Формируем сообщение: { type: "read", id: 1, file }.
    let msg_obj = js_sys::Object::new();
    js_sys::Reflect::set(
        &msg_obj,
        &JsValue::from_str("type"),
        &JsValue::from_str("read"),
    )?;
    js_sys::Reflect::set(
        &msg_obj,
        &JsValue::from_str("id"),
        &JsValue::from_f64(1.0),
    )?;
    js_sys::Reflect::set(
        &msg_obj,
        &JsValue::from_str("file"),
        file.as_ref(),
    )?;
    let msg: JsValue = msg_obj.into();

    // Оборачиваем ответ воркера в Promise.
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();
        let on_message = Closure::once_into_js(move |event: MessageEvent| {
            let data = event.data();
            let obj: &js_sys::Object = match data.dyn_ref() {
                Some(o) => o,
                None => {
                    let _ = reject_c
                        .call1(&JsValue::NULL, &JsValue::from_str("ответ не Object"));
                    return;
                }
            };
            let type_js = js_sys::Reflect::get(obj, &JsValue::from_str("type"))
                .unwrap_or(JsValue::UNDEFINED);
            let type_str = type_js.as_string().unwrap_or_default();

            if type_str == "result" {
                let _ = resolve_c.call1(&JsValue::NULL, &data);
            } else {
                let err_msg = js_sys::Reflect::get(
                    obj,
                    &JsValue::from_str("message"),
                )
                .ok()
                .and_then(|v| v.as_string())
                .unwrap_or_else(|| "worker error".to_string());
                let _ = reject_c.call1(&JsValue::NULL, &JsValue::from_str(&err_msg));
            }
        });
        worker.set_onmessage(Some(on_message.unchecked_ref()));
    });

    worker.post_message(&msg)?;
    let value = JsFuture::from(promise).await;

    // Терминируем воркер — освобождаем ресурсы.
    worker.terminate();

    let value = value?;
    return extract_result(&value);
}

/// Извлекает поля из ответа воркера.
fn extract_result(value: &JsValue) -> Result<WorkerReadResult, JsValue> {
    let obj: &js_sys::Object = value
        .dyn_ref()
        .ok_or_else(|| JsValue::from_str("ответ не Object"))?;

    let name = js_sys::Reflect::get(obj, &JsValue::from_str("name"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("нет поля name"))?;
    let size = js_sys::Reflect::get(obj, &JsValue::from_str("size"))?
        .as_f64()
        .ok_or_else(|| JsValue::from_str("нет поля size"))?;
    let head_hex = js_sys::Reflect::get(obj, &JsValue::from_str("headHex"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("нет поля headHex"))?;
    let head_len_f = js_sys::Reflect::get(obj, &JsValue::from_str("headLen"))?
        .as_f64()
        .ok_or_else(|| JsValue::from_str("нет поля headLen"))?;

    return Ok(WorkerReadResult {
        name,
        size,
        head_hex,
        head_len: head_len_f as u32,
    });
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — для `JsValue`, `Closure`.
- `wasm_bindgen::JsCast` — для `.dyn_ref()`, `.unchecked_ref()`.
- `wasm_bindgen_futures::JsFuture` — для `.await` на `Promise`.
- `web_sys::{MessageEvent, Worker}` — оба используются.
- Неиспользуемых импортов нет.

**Ключевые моменты:**

- **`Worker::new(url)`** — создаёт воркер из URL. URL должен быть того же origin.
- **`post_message(&JsValue)`** — отправляет сообщение. `File` вкладывается в `js_sys::Object` — structured clone, но сам `Blob` не копируется, только ссылка.
- **`set_onmessage`** — обработчик резолвит `Promise`. `Closure::once_into_js` — одноразовый, вызывается один раз и сам освобождается.
- **`worker.terminate()`** — обязателен после получения ответа.

---

### `src/app.rs` — только демонстрация 18.11

```rust
//! Демонстрация подглавы 18.11: Web Workers для файловой обработки.
//!
//! Секция: выбрать несколько файлов → каждый уйдёт в отдельный
//! Web Worker → FileReaderSync прочитает первые 64 КБ → результат.
//!
//! Пока воркеры работают, UI остаётся отзывчивым: страницу можно
//! прокручивать, кнопки — нажимать.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;

use crate::callbacks::{self, EventClosures};
use crate::{dom, file_api, log, worker_file};

/// URL воркера. Vite раздаёт файлы из `web/public/` по корню,
/// поэтому JS-воркер доступен как `/workers/file_worker.js`.
const WORKER_URL: &str = "/workers/file_worker.js";

pub fn run() -> Result<(), JsValue> {
    let container = dom::get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("container #app not found"))?;

    let heading = dom::create_heading(1, "Web Workers — файловая обработка")?;
    dom::append(&container, &heading)?;

    let closures = Rc::new(RefCell::new(EventClosures::new()));

    build_worker_section(&container, &closures)?;

    log!("Section 18.11 ready");
    std::mem::forget(closures);
    return Ok(());
}

// ---------------------------------------------------------------
// Секция 18.11: Web Workers
// ---------------------------------------------------------------

fn build_worker_section(
    parent: &web_sys::Element,
    closures: &Rc<RefCell<EventClosures>>,
) -> Result<(), JsValue> {
    let section = dom::create_element("section")?;
    dom::add_class(&dom::as_html(&section), "storage-section")?;

    let hint = dom::create_paragraph(
        "Выберите несколько файлов — каждый уйдёт в отдельный \
         Web Worker. В воркере FileReaderSync прочитает первые \
         64 КБ и вернёт hex. Пока воркеры работают, UI остаётся \
         отзывчивым: страницу можно прокручивать, кнопки — нажимать.",
    )?;
    dom::add_class(&dom::as_html(&hint), "hint")?;
    dom::append(&section, &hint)?;

    let input: web_sys::HtmlInputElement = dom::create_element("input")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не input"))?;
    input.set_type("file");
    input.set_multiple(true);
    dom::append(&section, &input)?;

    let btn = dom::create_element("button")?;
    btn.set_text_content(Some("Прочитать все в воркерах"));
    dom::append(&section, &btn)?;

    let result = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result), "api-result")?;
    result.set_text_content(Some("(выберите файлы и нажмите кнопку)"));
    dom::append(&section, &result)?;

    dom::append(parent, &section)?;

    let input_c = input.clone();
    let result_c = result.clone();

    callbacks::listen(
        &btn,
        "click",
        move |_ev: web_sys::Event| {
            let files = match input_c.files() {
                Some(f) => f,
                None => {
                    result_c.set_text_content(Some("Сначала выберите файлы."));
                    return;
                }
            };
            let count = files.length();
            if count == 0 {
                result_c.set_text_content(Some("Файлы не выбраны."));
                return;
            }

            // Собираем (имя, File) заранее, чтобы имена не зависели
            // от времени жизни files.
            let mut entries: Vec<(String, web_sys::File)> = Vec::new();
            for i in 0..count {
                if let Some(f) = files.get(i) {
                    let name = f.name();
                    entries.push((name, f));
                }
            }

            let result_c = result_c.clone();
            result_c.set_text_content(Some("Читаем в воркерах..."));
            spawn_local(async move {
                // Запускаем по одному воркеру на файл параллельно.
                let tasks = entries.into_iter().map(|(name, file)| {
                    async move {
                        let res = worker_file::read_file_in_worker(
                            &file,
                            WORKER_URL,
                        )
                        .await;
                        (name, res)
                    }
                });
                let outcomes = futures::future::join_all(tasks).await;
                let message = format_worker_report(&outcomes);
                result_c.set_text_content(Some(&message));
            });
        },
        closures,
    )?;

    return Ok(());
}

// ---------------------------------------------------------------
// Вспомогательные функции
// ---------------------------------------------------------------

/// Формирует отчёт по всем результатам обработки в воркерах.
///
/// Ветки `match` собирают строку в `let block: String`, tail —
/// через явный `return`. Это безопасный паттерн для окружения
/// проекта (см. предыдущие подглавы).
fn format_worker_report(
    outcomes: &[(String, Result<worker_file::WorkerReadResult, JsValue>)],
) -> String {
    if outcomes.is_empty() {
        return "Нет результатов.".to_string();
    }

    let mut out = format!("Обработано файлов: {}\n\n", outcomes.len());
    for (i, (name, res)) in outcomes.iter().enumerate() {
        let block: String;
        match res {
            Ok(r) => {
                block = format!(
                    "{}. {}\n   Размер: {} ({:.0} байт)\n   Первые {} байт: {}",
                    i + 1,
                    name,
                    file_api::format_file_size(r.size),
                    r.size,
                    r.head_len,
                    r.head_hex,
                );
            }
            Err(e) => {
                block = format!("{}. {} — ❌ ошибка: {:?}", i + 1, name, e);
            }
        }
        out.push_str(&block);
        out.push_str("\n\n");
    }
    return out;
}
```

**Проверка импортов:**

- `RefCell`, `Rc` — для `EventClosures`.
- `wasm_bindgen::prelude::*` — для `JsValue`.
- `wasm_bindgen::JsCast` — для `.dyn_into()`.
- `wasm_bindgen_futures::spawn_local` — для `async`-блока.
- `callbacks`, `EventClosures`, `dom`, `file_api`, `log`, `worker_file` — все используются.
- `futures::future::join_all` — полный путь, без импорта. Crate `futures = "0.3"` уже в `Cargo.toml`.
- Неиспользуемых импортов нет.

**Особенности:**

- **`app.rs` содержит только секцию 18.11.** Никаких других секций главы 18 — они находятся в своих подглавах.
- **Параллельный запуск через `join_all`.** Результаты идут в порядке файлов.
- **Один воркер на файл.** Браузер распределяет воркеры по ядрам.
- **UI не блокируется.** `spawn_local` возвращает управление на каждом `.await`.

---

### `tests/wasm_worker_file.rs`

Тесты проверяют, что `Worker` создаётся, что `postMessage` работает, и что `File` передаётся в воркер без потери данных. Реальный `file_worker.js` требует загрузки с сервера, поэтому в тестах используем inline-воркер через blob URL.

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// Создаёт Worker из inline-кода через blob URL.
fn create_inline_worker(code: &str) -> (web_sys::Worker, String) {
    let array = js_sys::Array::of1(&JsValue::from_str(code));
    let blob = web_sys::Blob::new_with_str_sequence(&array)
        .expect("Blob::new_with_str_sequence");
    let url = web_sys::Url::create_object_url_with_blob(&blob)
        .expect("create_object_url_with_blob");
    let worker = web_sys::Worker::new(&url).expect("Worker::new");
    return (worker, url);
}

#[wasm_bindgen_test]
async fn test_worker_echo() {
    // Воркер возвращает то, что получил.
    let code = "self.onmessage = (e) => self.postMessage(e.data);";
    let (worker, url) = create_inline_worker(code);

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();
        let on_message = Closure::once_into_js(move |event: web_sys::MessageEvent| {
            let _ = resolve_c.call1(&JsValue::NULL, &event.data());
        });
        let on_error = Closure::once_into_js(move |err: JsValue| {
            let _ = reject_c.call1(&JsValue::NULL, &err);
        });
        worker.set_onmessage(Some(on_message.unchecked_ref()));
        worker.set_onerror(Some(on_error.unchecked_ref()));
    });

    worker
        .post_message(&JsValue::from_str("hello"))
        .expect("post_message");
    let result = JsFuture::from(promise).await.expect("await");

    assert_eq!(result.as_string().as_deref(), Some("hello"));

    worker.terminate();
    web_sys::Url::revoke_object_url(&url).ok();
}

#[wasm_bindgen_test]
async fn test_worker_file_transfer() {
    // Воркер получает File и возвращает его имя и размер.
    // Проверяем, что File корректно передаётся через structured clone.
    let code = r#"
        self.onmessage = (e) => {
            const f = e.data.file;
            self.postMessage({ name: f.name, size: f.size });
        };
    "#;
    let (worker, url) = create_inline_worker(code);

    let file = web_sys::File::new_with_str_sequence(
        &js_sys::Array::of1(&JsValue::from_str("test content")),
        "sample.txt",
    )
    .expect("File::new_with_str_sequence");

    let msg_obj = js_sys::Object::new();
    js_sys::Reflect::set(
        &msg_obj,
        &JsValue::from_str("file"),
        file.as_ref(),
    )
    .expect("Reflect::set file");
    let msg: JsValue = msg_obj.into();

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();
        let on_message = Closure::once_into_js(move |event: web_sys::MessageEvent| {
            let _ = resolve_c.call1(&JsValue::NULL, &event.data());
        });
        let on_error = Closure::once_into_js(move |err: JsValue| {
            let _ = reject_c.call1(&JsValue::NULL, &err);
        });
        worker.set_onmessage(Some(on_message.unchecked_ref()));
        worker.set_onerror(Some(on_error.unchecked_ref()));
    });

    worker.post_message(&msg).expect("post_message");
    let result = JsFuture::from(promise).await.expect("await");

    let obj: &js_sys::Object = result.dyn_ref().expect("ответ не Object");
    let name = js_sys::Reflect::get(obj, &JsValue::from_str("name"))
        .expect("get name")
        .as_string()
        .expect("name не string");
    let size = js_sys::Reflect::get(obj, &JsValue::from_str("size"))
        .expect("get size")
        .as_f64()
        .expect("size не number");

    assert_eq!(name, "sample.txt");
    assert_eq!(size, 12.0); // длина "test content"

    worker.terminate();
    web_sys::Url::revoke_object_url(&url).ok();
}

#[wasm_bindgen_test]
async fn test_worker_terminate_no_panic() {
    let code = "self.onmessage = () => {};";
    let (worker, url) = create_inline_worker(code);
    worker.terminate();
    web_sys::Url::revoke_object_url(&url).ok();
}
```

**Проверка импортов:**

- `wasm_bindgen::prelude::*` — `JsValue`, `Closure`.
- `wasm_bindgen::JsCast` — `.dyn_ref()`, `.unchecked_ref()`.
- `wasm_bindgen_futures::JsFuture` — `.await` на `Promise`.
- `wasm_bindgen_test::*` — `wasm_bindgen_test`, `wasm_bindgen_test_configure`.

Полные пути `web_sys::*`, `js_sys::*` — без импортов. Неиспользуемых импортов нет.

**Что проверяем:**

- **`test_worker_echo`** — воркер создаётся из blob URL, получает строку, возвращает её обратно. Базовый контракт `postMessage` / `onmessage`.
- **`test_worker_file_transfer`** — `File` передаётся через `postMessage` без искажений: имя и размер совпадают. Это ключевая проверка structured clone для `File`.
- **`test_worker_terminate_no_panic`** — терминирование воркера не вызывает панику.

**Что не проверяем:** реальный `file_worker.js` требует загрузки с сервера, а не с blob URL. В headless-тестах это усложняет конфигурацию. Интеграционная проверка делается вручную через демо-секцию.

---

## Команды запуска

### Сборка и запуск демо

```bash
# Проверка типов
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests

# Dev-сборка WASM и запуск Vite
npm run start
```

**Важно:** убедитесь, что файл `web/public/workers/file_worker.js` существует. Vite раздаёт `web/public/*` по корню, поэтому URL `/workers/file_worker.js` ведёт ровно к этому файлу.

### Тесты подглавы 18.11

```bash
wasm-pack test --headless --chrome --test wasm_worker_file
wasm-pack test --headless --firefox --test wasm_worker_file

npm run test
```

### Что ожидать в браузере

После `npm run start` появится одна секция — **«Web Workers — файловая обработка»** с полем выбора нескольких файлов и кнопкой.

**Сценарий 1 — один файл:**
1. Выберите `document.pdf`.
2. Нажмите «Прочитать все в воркерах».
3. Пока идёт обработка, попробуйте прокрутить страницу, нажать другую кнопку, переключить вкладку — всё работает, UI не замерзает.
4. Появится результат:

```
Обработано файлов: 1

1. document.pdf
   Размер: 234.56 KB (240192 байт)
   Первые 64 байт: 25 50 44 46 2D 31 2E 37 0A 25 E2 E3 CF D3 0A ...
```

**Сценарий 2 — четыре файла параллельно:**
1. Выберите 4 больших файла.
2. Нажмите кнопку.
3. Все четыре воркера стартуют одновременно. Результаты появятся вместе, но каждый воркер работал в своём потоке.

**Сценарий 3 — воркер не подключается:**
Если файл `web/public/workers/file_worker.js` отсутствует, все задачи вернут ошибку:

```
1. document.pdf — ❌ ошибка: JsValue("worker error")
```

Проверьте, что файл на месте и URL правильный.

---

## Итоги подглавы 18.11

- **Web Worker** — отдельный поток, в котором тяжёлая работа не блокирует UI.
- **`File` передаётся через `postMessage`** по structured clone: содержимое не копируется.
- **`FileReaderSync` доступен только в воркере** — в главном потоке его вызов бросает `InvalidStateError`.
- **`Worker::new(url)` + `postMessage` + `onmessage`** — базовый цикл.
- **`worker.terminate()`** — обязателен после завершения.
- **Параллелизм через `join_all`** — несколько воркеров на разные файлы, браузер распределяет по ядрам.

Модуль `worker_file.rs` автономен, все tail-функции через явный `return`, неиспользуемых импортов нет. JS-воркер самодостаточен. `app.rs` содержит **только** код подглавы 18.11 — никаких других секций.

## Замечания на будущее

Четыре правила, зафиксированные в этой подглаве:

1. **`FileReaderSync` работает только в воркере.** В главном потоке — `InvalidStateError`.

2. **`File` в `postMessage` не копируется.** Structured clone обрабатывает `File` как ссылку на blob. Передача 4-гигабайтного файла в воркер занимает микросекунды.

3. **`Worker::new(url)` требует того же origin.** URL воркера должен быть на том же домене. `web/public/workers/file_worker.js` через Vite удовлетворяет этому требованию.

4. **`app.rs` — только код текущей подглавы.** Предыдущие секции не накапливаются. Читатель получает фокус на изучаемой теме.