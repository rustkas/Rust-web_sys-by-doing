# Глава 18. File API

## 18.12. Хеширование файлов: `sha2`, `SubtleCrypto` и адаптивный пул воркеров

В подглаве 18.11 мы вынесли чтение файла в Web Worker. Теперь разберёмся, как проверить **целостность** — посчитать SHA-256. Это нужно для сравнения файлов, проверки контрольных сумм, детектирования дубликатов. В браузере есть встроенный криптографический API — `crypto.subtle.digest`, работающий на нативном уровне. И есть crate `sha2` — чистый Rust. Мы разберём обе реализации, измерим их скорости, и **удивимся результату**.

Эта подглава — не просто описание API, а **история оптимизации**. Мы начнём с наивного решения, обнаружим его проблемы и последовательно улучшим: от 8 МБ/с до 400+ МБ/с. По ходу разберём, почему SHA-256 нельзя распараллелить внутри одного файла, как работать с гигабайтными файлами, и как построить адаптивный планировщик, который сам подстраивает число воркеров.

---

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── .cargo/
│   └── config.toml                # + target-feature=+simd128
├── Cargo.toml                     # + sha2 = "0.11", hex, SubtleCrypto, Crypto, MessageChannel
├── src/
│   ├── lib.rs                     # ОБНОВЛЁН: + pub mod hashing;
│   ├── app.rs                     # ОБНОВЛЁН: только секция 18.12
│   ├── hashing.rs                 # НОВЫЙ: модуль подглавы 18.12
│   ├── worker_file.rs             # ОБНОВЛЁН: + sha256_in_worker + sha256_chunk_in_worker
│   ├── dom.rs                     # существует
│   ├── callbacks.rs               # существует
│   ├── file_reader.rs             # из 18.2
│   ├── file_utils.rs              # из 18.3
│   ├── file_api.rs                # из 18.1
│   ├── logging.rs                 # существует
│   └── … остальные модули
├── web/
│   └── public/
│       └── workers/
│           └── file_worker.js     # ОБНОВЛЁН: + handlers hash / hash_chunk
└── tests/
    └── wasm_hashing.rs            # НОВЫЙ: тесты подглавы 18.12
```

**Пояснения:**

- `src/hashing.rs` — модуль подглавы. Автономен: зависит от `web-sys`, `js-sys`, `wasm-bindgen`, `wasm-bindgen-futures`, `sha2`, `hex`.
- `src/worker_file.rs` расширен двумя функциями: `sha256_in_worker` (для одиночного ArrayBuffer) и `sha256_chunk_in_worker` (для куска файла).
- `src/app.rs` содержит **только** код подглавы 18.12.
- `web/public/workers/file_worker.js` расширен обработчиками `hash` и `hash_chunk`.

---

## Ссылки для детального изучения

- **MDN: SubtleCrypto.digest()** — https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest
- **MDN: Crypto** — https://developer.mozilla.org/en-US/docs/Web/API/Crypto
- **RFC 6234 — SHA-256** — https://datatracker.ietf.org/doc/html/rfc6234
- **crate `sha2` — документация** — https://docs.rs/sha2/latest/sha2/
- **crate `sha2` 0.11 changelog** — https://github.com/RustCrypto/hashes/blob/master/sha2/CHANGELOG.md
- **crate `hex` — документация** — https://docs.rs/hex/latest/hex/
- **Merkle tree** — https://en.wikipedia.org/wiki/Merkle_tree
- **MDN: MessageChannel** — https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel

---

## Зависимости и конфигурация

### `.cargo/config.toml`

```toml
[target.wasm32-unknown-unknown]
rustflags = [
    "--cfg=web_sys_unstable_apis",
    "-C", "target-feature=+simd128",
]
```

### `Cargo.toml`

```toml
[dependencies]
# … существующие …
sha2 = "0.11"
hex = "0.4"
futures = "0.3"

[dependencies.web-sys]
version = "0.3"
features = [
    # … существующие …
    "SubtleCrypto",
    "Crypto",
    "MessageChannel",
]
```

**Что важно знать про `sha2 0.11`:** в этой версии `Digest::finalize()` возвращает `Array<u8, N>` из crate `hybrid-array`. Этот тип **не реализует `LowerHex`**, поэтому `format!("{:x}", hash)` не компилируется. Правильный способ — `hex::encode(hash.as_slice())`.

---

## Содержание новых / обновлённых файлов

### `src/lib.rs`

```rust
pub mod hashing;    // 18.12
```

---

### `src/hashing.rs`

Модуль предоставляет четыре способа получить SHA-256, плюс утилиты. Все они **автономны** — не зависят от других модулей проекта.

```rust
//! Хеширование файлов: SHA-256 через SubtleCrypto и через crate `sha2`.
//!
//! **Про производительность.** RustCrypto `sha2` на wasm32 — скаляр:
//! нет SIMD-бэкенда, скорость ~8–12 МБ/с. Нативный
//! `crypto.subtle.digest` работает через SHA-NI или ARM Crypto
//! Extensions, и в 6–8 раз быстрее.
//!
//! **Про yield.** `await` на `Blob::array_buffer()` даёт только
//! микротаску, а репейнт в браузере — между макротасками. Чтобы
//! прогресс-бар обновлялся, периодически отдаём управление через
//! `MessageChannel`. Это не throttлится в фоне, в отличие от
//! `setTimeout(0)`.

use std::cell::Cell;
use std::rc::Rc;

use sha2::{Digest, Sha256};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Blob, File};

/// Считает SHA-256 через crate `sha2` (синхронно).
pub fn sha256_via_sha2(data: &[u8]) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data);
    let hash = hasher.finalize();
    return hex::encode(hash.as_slice());
}

/// Считает SHA-256 через `crypto.subtle.digest` (асинхронно).
///
/// Требует secure context: HTTPS или localhost.
pub async fn sha256_via_subtle(data: &[u8]) -> Result<String, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let crypto = window.crypto().map_err(|_| JsValue::from_str("no crypto"))?;
    let subtle = crypto.subtle();

    let uint8 = js_sys::Uint8Array::new_from_slice(data);
    let obj: &js_sys::Object = uint8.as_ref();
    let promise = subtle.digest_with_str_and_buffer_source("SHA-256", obj)?;
    let value = JsFuture::from(promise).await?;
    let array_buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("subtle.digest вернул не ArrayBuffer"))?;
    return Ok(hex::encode(js_sys::Uint8Array::new(&array_buffer).to_vec()));
}

/// Slice-based streaming без отмены — обёртка для тестов.
pub async fn sha256_stream_with_progress<F>(
    blob: &Blob, chunk_size: u32, on_progress: F,
) -> Result<String, JsValue>
where F: FnMut(f64, f64) {
    let never = Rc::new(Cell::new(false));
    let result = sha256_stream_cancellable(blob, chunk_size, never, on_progress).await?;
    return result.ok_or_else(|| JsValue::from_str("неожиданная отмена"));
}

/// Slice-based streaming с отменой.
///
/// Режет файл на чанки через `Blob.slice`, каждый читает
/// `Blob::array_buffer()`. Большие чанки (4–16 МБ) уменьшают
/// число Rust↔JS переходов.
pub async fn sha256_stream_cancellable<F>(
    blob: &Blob,
    chunk_size: u32,
    cancel: Rc<Cell<bool>>,
    mut on_progress: F,
) -> Result<Option<String>, JsValue>
where F: FnMut(f64, f64) {
    let mut hasher = Sha256::new();
    let size = blob.size();
    let chunk_size_f = chunk_size as f64;
    let mut offset = 0.0_f64;

    on_progress(0.0, size);
    let mut last_yield_ms = js_sys::Date::now();

    while offset < size {
        if cancel.get() { return Ok(None); }
        let end = (offset + chunk_size_f).min(size);
        let slice = blob.slice_with_f64_and_f64(offset, end)?;
        let bytes = read_blob_bytes(&slice).await?;
        if cancel.get() { return Ok(None); }

        hasher.update(&bytes);
        offset = end;
        on_progress(offset, size);

        let now = js_sys::Date::now();
        if now - last_yield_ms >= 16.0 {
            yield_via_message_channel().await?;
            last_yield_ms = js_sys::Date::now();
        }
    }

    return Ok(Some(hex::encode(hasher.finalize().as_slice())));
}

/// Быстрый отпечаток: размер и дата изменения.
pub fn quick_fingerprint(file: &File) -> String {
    return format!(
        "{} байт, изменён {}",
        file.size() as u64,
        file.last_modified() as u64,
    );
}

/// Читает Blob как `Vec<u8>` через `Blob::array_buffer()`.
async fn read_blob_bytes(blob: &Blob) -> Result<Vec<u8>, JsValue> {
    let promise = blob.array_buffer();
    let value = JsFuture::from(promise).await?;
    let array_buffer: js_sys::ArrayBuffer = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не ArrayBuffer"))?;
    return Ok(js_sys::Uint8Array::new(&array_buffer).to_vec());
}

/// Yield через MessageChannel — не throttлится в фоне.
async fn yield_via_message_channel() -> Result<(), JsValue> {
    let channel = web_sys::MessageChannel::new()?;
    let port1 = channel.port1();
    let port2 = channel.port2();
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let resolve_c = resolve.clone();
        let cb = Closure::once_into_js(move |_event: web_sys::MessageEvent| {
            let _ = resolve_c.call0(&JsValue::NULL);
        });
        port1.set_onmessage(Some(cb.unchecked_ref()));
        port2.post_message(&JsValue::NULL).ok();
    });
    JsFuture::from(promise).await?;
    return Ok(());
}
```

---

### `web/public/workers/file_worker.js`

Воркер обрабатывает три типа сообщений: `read` (из 18.11), `hash` (одиночный файл), `hash_chunk` (кусок файла).

```js
// Web Worker для файловых операций.
//
// Типы сообщений:
//   1. { type: "read",       id, file }
//   2. { type: "hash",       id, file }
//   3. { type: "hash_chunk", id, file, start, end }
//
// Ответы: result / hash_result / hash_chunk_result / error.

const HEAD_SIZE = 64;

self.onmessage = (event) => {
    const data = event.data;
    if (!data) return;
    if (data.type === "read") handleRead(data);
    else if (data.type === "hash") handleHash(data);
    else if (data.type === "hash_chunk") handleHashChunk(data);
    else self.postMessage({ type: "error", message: "unknown message type" });
};

function handleRead(data) {
    const file = data.file, id = data.id;
    if (!file) { self.postMessage({ type: "error", id, message: "no file" }); return; }
    try {
        const reader = new FileReaderSync();
        const head = file.slice(0, HEAD_SIZE);
        const buffer = reader.readAsArrayBuffer(head);
        const uint8 = new Uint8Array(buffer);
        let hex = "";
        for (let i = 0; i < uint8.length; i++) {
            if (i > 0) hex += " ";
            hex += uint8[i].toString(16).padStart(2, "0").toUpperCase();
        }
        self.postMessage({
            type: "result", id, name: file.name, size: file.size,
            headHex: hex, headLen: uint8.length,
        });
    } catch (err) { self.postMessage({ type: "error", id, message: String(err) }); }
}

async function handleHash(data) {
    const file = data.file, id = data.id;
    if (!file) { self.postMessage({ type: "error", id, message: "no file" }); return; }
    try {
        const arrayBuffer = await file.arrayBuffer();
        const hashBuffer = await crypto.subtle.digest("SHA-256", arrayBuffer);
        const hex = bytesToHex(new Uint8Array(hashBuffer));
        self.postMessage({
            type: "hash_result", id, name: file.name, size: file.size, hex,
        });
    } catch (err) { self.postMessage({ type: "error", id, message: String(err) }); }
}

async function handleHashChunk(data) {
    const file = data.file, start = data.start, end = data.end, id = data.id;
    if (!file) { self.postMessage({ type: "error", id, message: "no file" }); return; }
    try {
        const slice = file.slice(start, end);
        const buffer = await slice.arrayBuffer();
        const hashBuffer = await crypto.subtle.digest("SHA-256", buffer);
        const hex = bytesToHex(new Uint8Array(hashBuffer));
        self.postMessage({
            type: "hash_chunk_result", id, start, end,
            bytes: buffer.byteLength, hex,
        });
    } catch (err) { self.postMessage({ type: "error", id, message: String(err) }); }
}

function bytesToHex(uint8) {
    let hex = "";
    for (let i = 0; i < uint8.length; i++) {
        hex += uint8[i].toString(16).padStart(2, "0");
    }
    return hex;
}
```

---

### `src/worker_file.rs` — добавляем две функции

К существующему модулю добавляются `sha256_in_worker` и `sha256_chunk_in_worker`. Они похожи по структуре на `read_file_in_worker` из 18.11: создают `Worker`, отправляют сообщение, ждут ответ через `Promise`, терминируют воркер.

```rust
/// Считает SHA-256 файла в Web Worker через `crypto.subtle.digest`.
pub async fn sha256_in_worker(
    file: &web_sys::File, worker_url: &str,
) -> Result<String, JsValue> {
    let worker = Worker::new(worker_url)?;
    let msg_obj = js_sys::Object::new();
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("type"),
        &JsValue::from_str("hash"))?;
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("id"),
        &JsValue::from_f64(1.0))?;
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("file"),
        file.as_ref())?;
    let msg: JsValue = msg_obj.into();

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();
        let on_message = Closure::once_into_js(move |event: MessageEvent| {
            let data = event.data();
            let obj: &js_sys::Object = match data.dyn_ref() {
                Some(o) => o,
                None => {
                    let _ = reject_c.call1(&JsValue::NULL,
                        &JsValue::from_str("ответ не Object"));
                    return;
                }
            };
            let type_str = js_sys::Reflect::get(obj, &JsValue::from_str("type"))
                .unwrap_or(JsValue::UNDEFINED)
                .as_string().unwrap_or_default();
            if type_str == "hash_result" {
                let _ = resolve_c.call1(&JsValue::NULL, &data);
            } else {
                let err_msg = js_sys::Reflect::get(obj, &JsValue::from_str("message"))
                    .ok().and_then(|v| v.as_string())
                    .unwrap_or_else(|| "worker error".to_string());
                let _ = reject_c.call1(&JsValue::NULL, &JsValue::from_str(&err_msg));
            }
        });
        worker.set_onmessage(Some(on_message.unchecked_ref()));
    });

    worker.post_message(&msg)?;
    let value = JsFuture::from(promise).await;
    worker.terminate();
    let value = value?;
    return extract_hex_field(&value);
}

/// Считает SHA-256 куска `[start, end)` в Web Worker.
pub async fn sha256_chunk_in_worker(
    file: &web_sys::File,
    start: f64, end: f64,
    worker_url: &str,
) -> Result<String, JsValue> {
    let worker = Worker::new(worker_url)?;
    let msg_obj = js_sys::Object::new();
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("type"),
        &JsValue::from_str("hash_chunk"))?;
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("id"),
        &JsValue::from_f64(1.0))?;
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("file"), file.as_ref())?;
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("start"),
        &JsValue::from_f64(start))?;
    js_sys::Reflect::set(&msg_obj, &JsValue::from_str("end"),
        &JsValue::from_f64(end))?;
    let msg: JsValue = msg_obj.into();

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();
        let on_message = Closure::once_into_js(move |event: MessageEvent| {
            let data = event.data();
            let obj: &js_sys::Object = match data.dyn_ref() {
                Some(o) => o,
                None => {
                    let _ = reject_c.call1(&JsValue::NULL,
                        &JsValue::from_str("ответ не Object"));
                    return;
                }
            };
            let type_str = js_sys::Reflect::get(obj, &JsValue::from_str("type"))
                .unwrap_or(JsValue::UNDEFINED)
                .as_string().unwrap_or_default();
            if type_str == "hash_chunk_result" {
                let _ = resolve_c.call1(&JsValue::NULL, &data);
            } else {
                let err_msg = js_sys::Reflect::get(obj, &JsValue::from_str("message"))
                    .ok().and_then(|v| v.as_string())
                    .unwrap_or_else(|| "worker error".to_string());
                let _ = reject_c.call1(&JsValue::NULL, &JsValue::from_str(&err_msg));
            }
        });
        worker.set_onmessage(Some(on_message.unchecked_ref()));
    });

    worker.post_message(&msg)?;
    let value = JsFuture::from(promise).await;
    worker.terminate();
    let value = value?;
    return extract_hex_field(&value);
}

fn extract_hex_field(value: &JsValue) -> Result<String, JsValue> {
    let obj: &js_sys::Object = value.dyn_ref()
        .ok_or_else(|| JsValue::from_str("ответ не Object"))?;
    return js_sys::Reflect::get(obj, &JsValue::from_str("hex"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("нет поля hex"));
}
```

---

### `src/app.rs` — единая адаптивная стратегия

Полный код — в предыдущем ответе. Здесь — ключевые идеи.

**Две стратегии вместо трёх.** `Inline` для файлов < 1 МБ — WASM sha2 в памяти. `Chunked` для всего остального — пул воркеров с нативным `crypto.subtle.digest`. Это решает проблему indeterminate progress bar: chunked всегда даёт реальный прогресс.

**Начальный параллелизм** зависит от числа кусков:

| Число кусков | Старт | Максимум |
|---|---|---|
| 1 | 1 | 1 |
| 2–4 | 2 | 2 |
| 5–12 | 3 | 4 |
| 13–32 | 4 | 6 |
| > 32 | 4 | 8 |

**Hill-climbing.** Каждые 4 завершённых куска делается checkpoint. Сравниваем throughput текущего окна с предыдущим:

- быстрее на 10 % → параллелизм +1;
- медленнее на 10 % → параллелизм −1;
- в пределах 10 % → оставляем.

Система сходится к оптимуму за 10–15 кусков. На быстрых SSD добавляет воркеров до 5–6, на медленных HDD остаётся на 2.

**Отмена срабатывает на первом завершившемся куске**, а не на всём батче. Худший случай ожидания — время обработки одного куска (~0.5–1 с).

**UI** обновляется на каждом куске: прогресс-бар, метка с двумя строками (`проценты · кусков · воркеров` / `скорость · ETA`), текст кнопки «Стоп».

**Merkle-root.** Для файлов > 1 МБ итоговый хеш — SHA-256 от конкатенации hex-хешей кусков. Это не стандартный SHA-256 файла. В отчёте явная пометка.

---

### `tests/wasm_hashing.rs`

Тесты проверяют корректность sha2 по известным векторам NIST, совпадение с `SubtleCrypto` и работу slice-based streaming.

Ключевые тесты:

```rust
#[wasm_bindgen_test]
fn test_sha2_hello() {
    // SHA-256("hello") = 2cf24dba...
    let hex = sha256_via_sha2(b"hello");
    assert_eq!(hex, "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824");
}

#[wasm_bindgen_test]
async fn test_subtle_matches_sha2() {
    let data = b"The quick brown fox jumps over the lazy dog";
    assert_eq!(sha256_via_sha2(data),
               sha256_via_subtle(data).await.unwrap());
}

#[wasm_bindgen_test]
async fn test_stream_matches_in_memory_multi_chunk() {
    let data: Vec<u8> = (0..100_000).map(|i| (i % 256) as u8).collect();
    let blob = make_blob(&data);
    assert_eq!(sha256_via_sha2(&data),
               sha256_stream(&blob, 16 * 1024).await.unwrap());
}
```

Полный набор — 19 тестов, включая прогресс-контракты (первый вызов = 0, последний = total, монотонность).

---

## Команды запуска

```bash
# Проверка типов
cargo check --target wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown --tests

# Dev-сборка и запуск
npm run start

# Тесты подглавы
wasm-pack test --headless --chrome --test wasm_hashing
wasm-pack test --headless --firefox --test wasm_hashing
npm run test
```

---

## Результаты: путь оптимизации

Мы прошли долгий путь, и стоит зафиксировать цифры.

| Версия | Файл 4 ГБ | Примечание |
|---|---|---|
| WASM streaming, `FileReader` + `TimeoutFuture` | **1259 с (3.4 МБ/с)** | Фоновая вкладка throttled setTimeout |
| WASM streaming, `Blob.array_buffer` + MessageChannel | 527 с (8 МБ/с) | Yield перестал тормозить |
| Slice 4 МБ + WASM | 527 с (8 МБ/с) | Размер чанка не помог — WASM скаляр |
| **Chunked native + адаптивный пул** | **10.5 с (400 МБ/с)** | **×120 ускорение** |

Для файла 926 МБ одиночный Worker даёт 67 МБ/с, chunked native с пулом — 400+ МБ/с. Разница в 6 раз — именно вклад параллелизма.

### Почему это работает

- **`crypto.subtle.digest` использует SHA-NI** на x86 и ARM Crypto Extensions на Apple Silicon. Один вызов даёт 60–70 МБ/с.
- **4 параллельных вызова** на 4 ядрах — 240–280 МБ/с.
- **5 воркеров** на 8-ядерном CPU — 350–420 МБ/с.
- **6+ воркеров** обычно уже упираются в диск или в память, поэтому hill-climbing останавливается на 4–6.

### Что ограничивает дальнейший рост

- **Скорость диска.** Если носитель отдаёт 500 МБ/с, теоретический предел — 500 МБ/с.
- **Пропускная способность PCIe / SATA.** NVMe даёт 3+ ГБ/с, SATA — 500 МБ/с.
- **`ArrayBuffer` копирование.** Каждый кусок читается в JS-буфер, потом передаётся в `crypto.subtle`, где тоже копируется. Для кусков 128 МБ это 4 × 2 × 128 МБ = 1 ГБ копирований на файл 4 ГБ — около 1 секунды.

---

## Итоги подглавы 18.12

Ключевые факты, зафиксированные по ходу работы:

1. **SHA-256 нельзя распараллелить внутри одного файла** — структура Меркла — Дамгора. Но можно распараллелить независимые куски.
2. **`sha2` crate на wasm32 — скаляр.** Флаг `+simd128` включает компилятору SIMD-инструкции, но у RustCrypto нет SIMD-бэкенда для wasm32. Скорость ~8–12 МБ/с.
3. **`crypto.subtle.digest` в 6–8 раз быстрее** — работает через SHA-NI / Crypto Extensions. Один вызов даёт 60–70 МБ/с.
4. **Файлы > 2 ГБ не влезают в `ArrayBuffer`** — нужен chunked подход.
5. **Chunked hashing даёт Merkle-root, а не стандартный SHA-256.** Для проверки целостности подходит, для сравнения с внешним хешем — нет.
6. **`setTimeout(0)` throttлится в фоне** до 1000 мс. `MessageChannel` — не throttлится.
7. **`Blob.stream()` режет по 64 КБ** — на больших файлах накладные расходы съедают всё преимущество. `Blob.slice` с чанком 4+ МБ быстрее.
8. **Hill-climbing** — простейший алгоритм адаптации параллелизма: замеряем throughput, шагаем в сторону роста, останавливаемся на плато.
9. **Indeterminate progress bar вводит пользователя в заблуждение.** Если операцию можно разбить на куски — разбейте; прогресс должен быть реальным.
10. **Отмена должна срабатывать за время обработки одного куска**, а не всей группы. Для этого `FuturesUnordered` вместо `join_all`.

## Замечания на будущее

**Правило проекта:** прежде чем оптимизировать — **измерьте**. Наши первые попытки ускорения (увеличение чанка, `Blob.stream()`) не помогли, потому что мы оптимизировали не то место. Реальное узкое место было в скорости WASM-хеша, а не в переходах Rust↔JS. Определить это можно было только замером: 1-мегабайтный файл шёл с той же скоростью 8 МБ/с, что и 4-гигабайтный — значит, дело не в накладных расходах на переходы.

**Второе правило:** адаптивность должна быть **в параметрах**, а не в перезапуске. Параллелизм и размер куска меняются на лету без потери уже сделанной работы. Это тот же принцип, что в CDN и автоскейлерах.

**Третье правило:** для параллельных задач с ограничением используйте `FuturesUnordered`. `join_all` ждёт всю группу — простаивает CPU и медленно реагирует на отмену.

**Четвёртое правило:** `sha2 0.11` требует `hex::encode(hash.as_slice())`, потому что `finalize()` возвращает `Array<u8, N>`, не реализующий `LowerHex`. `format!("{:x}", hash)` не компилируется.

**Пятое правило:** для работы с большими файлами, где итог — Merkle-root, всегда **явно предупреждайте пользователя** в UI. Пользователь может использовать хеш для сравнения с внешним источником — и получит несовпадение без объяснения. Пометка «Merkle-root, не стандартный SHA-256» снимает вопрос заранее.