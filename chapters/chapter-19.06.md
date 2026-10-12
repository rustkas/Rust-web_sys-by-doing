# Глава 19.6. Большие файлы: `adaptive chunk`, backpressure, копирование

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── lib.rs                       # +2 pub mod fsa_large_*
│   ├── app.rs                       # демо 19.6: два блока + ввод размера источника
│   ├── fsa_large_chunk.rs           # план чанков, live-скорость, метрики (+progress_calls)
│   └── fsa_large_copy.rs            # потоковое копирование с отменой, SHA-256, CopyOptions
└── tests/
    └── wasm_fsa_large.rs            # тесты 19.6 (+4 новых)
```

**Краткий комментарий.** В проекте есть `fsa_api_large.rs` — две функции: `adaptive_chunk_size(total)` (одно число) и `copy_file(src, dst, …)` (простой цикл). Эта подглава идёт **на уровень модели и измеримости**:

1. **Chunk plan как объект.** Не «число 64 КБ», а полный план: размер чанка, ожидаемое число итераций, peak memory, экономия по сравнению с «прочитать весь файл в память».
2. **Backpressure — не абстрактное слово, а измеримая доля времени.** Читаем и пишем последовательно: `read chunk → write chunk → read next`. Диск сам диктует темп; параллелить read и write нельзя — буферы растут, memory bound рушится.
3. **Live-скорость через EMA.** Сырая скорость в первые 100 мс — шум (0 или 999 МБ/с). Экспоненциальная скользящая средняя сглаживает и даёт стабильное число.
4. **Отмена не оставляет мусора.** До `close()` destination-поток откатывается через `abort()` — источник нетронут, получатель без частичных данных.
5. **Counterfactual — главный образовательный приём.** Демо даёт две версии одного и того же копирования: с yield (UI плавный) и без yield (UI зависает). Плюс панель «прочитать весь файл в память vs читать и писать чанками»: сколько RAM сэкономили. Без counterfactual правило «yield необходим» остаётся словами.
6. **Проверка — не догадка.** Совпадение размеров — необходимое, но не достаточное условие. SHA-256 обоих файлов даёт **достаточное** доказательство побитовой идентичности.
7. **Размер источника — параметр UI.** 1…500 МБ. Позволяет за один прогон увидеть, как `ChunkPlan` реагирует на маленький и большой файл, и как наивный путь упирается в память вкладки, а чанковый — нет.

Префикс `fsa_large_*` — уникален; с `fsa_api_large` не пересекается.

### Терминология UI

Слово «наивно» (калька с англ. *naive*) в интерфейсе не используется — это жаргон, который легко спутать с «нативно». Вместо него всюду применяются описательные формулировки:

* **«прочитать весь файл в память»** — прямолинейный путь: сначала получить все байты файла в `ArrayBuffer`, потом записать их в приёмник. Пик памяти = размер файла. Работает с той же File System Access API, что и чанковый путь — разница только в способе использования.
* **«читать и писать чанками»** — реализованный streaming-путь: `read chunk → write chunk → read next`. Пик памяти = размер одного чанка, не зависит от размера файла.

Эти два термина вводятся в help-тексте и используются единообразно по всему демо.

## Ссылки

- [MDN: Backpressure in streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Concepts#backpressure)
- [MDN: ReadableStreamDefaultReader](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStreamDefaultReader)
- [MDN: FileSystemWritableFileStream](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemWritableFileStream)
- [web.dev: Streams — processing without loading](https://web.dev/streams/)
- [WHATWG Streams spec](https://streams.spec.whatwg.org/)
- [Chrome: File System Access — performance](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access)
- [MDN: SubtleCrypto / SHA-256](https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/digest)

---

## Исходные модули

### `src/fsa_large_chunk.rs`

```rust
//! Модель плана копирования: размер чанка, число итераций, live-скорость.
//!
//! Модуль не читает и не пишет файл (это делает `fsa_large_copy`) —
//! здесь только предметная модель:
//! * `ChunkPlan` — размер чанка под объём, ожидаемое число итераций,
//!   peak memory, экономия по сравнению с «прочитать весь файл в память».
//! * `SpeedEma` — экспоненциальная средняя для сглаживания скачков.
//! * `CopyStats` — метрики копирования, раздельно read / write time,
//!   число yield'ов и число вызовов `on_progress`.
//!
//! `progress_calls` — educational-поле: с yield'ом оно десятки, без yield'а — 1.
//! Разница между двумя режимами видна в UI и в тестах.

use crate::fsa_choose_options::format_str;

/// Целевое число итераций при выборе размера чанка.
pub const TARGET_ITERATIONS: u32 = 200;
pub const MIN_CHUNK: usize = 64 * 1024;        // 64 КиБ
pub const MAX_CHUNK: usize = 4 * 1024 * 1024;  // 4 МиБ

/// План копирования под заданный размер файла.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct ChunkPlan {
    pub total_bytes: u64,
    pub chunk_size: usize,
    pub expected_chunks: u32,
    pub memory_bound_bytes: u64,
}

impl ChunkPlan {
    pub fn for_total(total: u64) -> Self {
        let total_usize: usize = usize::try_from(total).unwrap_or(usize::MAX);
        let raw: usize = total_usize / TARGET_ITERATIONS as usize;
        let chunk_size: usize = raw.clamp(MIN_CHUNK, MAX_CHUNK);
        let expected_chunks: u32 = if total == 0 {
            0
        } else {
            ((total + chunk_size as u64 - 1) / chunk_size as u64) as u32
        };
        return ChunkPlan {
            total_bytes: total,
            chunk_size,
            expected_chunks,
            memory_bound_bytes: chunk_size as u64,
        };
    }

    /// Сколько байт сэкономлено по сравнению с «прочитать весь файл в память».
    pub fn memory_saved(&self) -> u64 {
        if self.total_bytes > self.memory_bound_bytes {
            return self.total_bytes - self.memory_bound_bytes;
        }
        return 0;
    }

    pub fn describe(&self) -> String {
        let total: String = format_bytes(self.total_bytes);
        let chunk: String = format_bytes(self.chunk_size as u64);
        let bound: String = format_bytes(self.memory_bound_bytes);
        let saved: String = format_bytes(self.memory_saved());
        let s: String = format_str(format_args!(
            "План: {total} · chunk {chunk} · ~{} итераций\n\
             Пик памяти: {bound} (прочитать весь файл: {total}, экономия {saved})",
            self.expected_chunks,
        ));
        return s;
    }
}

/// Экспоненциальная скользящая средняя скорости.
///
/// Сырая скорость в первые 100 мс — шум: то 0 МБ/с, то 999 МБ/с.
/// EMA с `alpha ≈ 0.25` сглаживает и даёт стабильное число для UI.
#[derive(Debug, Clone)]
pub struct SpeedEma {
    alpha: f64,
    value: f64,
    initialised: bool,
}

impl SpeedEma {
    pub fn new(alpha: f64) -> Self {
        return SpeedEma { alpha, value: 0.0, initialised: false };
    }

    pub fn update(&mut self, bytes: u64, elapsed_ms: f64) {
        if elapsed_ms <= 0.0 {
            return;
        }
        let mbps: f64 = bytes as f64 / 1_048_576.0 / (elapsed_ms / 1000.0);
        if !self.initialised {
            self.value = mbps;
            self.initialised = true;
        } else {
            self.value = self.alpha * mbps + (1.0 - self.alpha) * self.value;
        }
    }

    pub fn value_mbps(&self) -> f64 {
        return self.value;
    }
}

/// Итоговые метрики копирования.
#[derive(Debug, Clone, PartialEq)]
pub struct CopyStats {
    pub bytes_copied: u64,
    pub chunks: u32,
    pub yields: u32,
    /// Сколько раз вызвался `on_progress` (с yield'ом — много, без — 1).
    /// Educational-поле: разница между режимами видна в UI и тестах.
    pub progress_calls: u32,
    pub read_ms: f64,
    pub write_ms: f64,
    pub total_ms: f64,
    pub chunk_size: usize,
    pub cancelled: bool,
    pub first_chunk_sizes: Vec<usize>,
}

impl CopyStats {
    pub fn avg_speed_mbps(&self) -> f64 {
        if self.total_ms <= 0.0 {
            return 0.0;
        }
        return self.bytes_copied as f64 / 1_048_576.0 / (self.total_ms / 1000.0);
    }

    /// Доля времени на чтение.
    pub fn read_share(&self) -> f64 {
        let sum: f64 = self.read_ms + self.write_ms;
        if sum <= 0.0 {
            return 0.0;
        }
        return self.read_ms / sum;
    }

    pub fn write_share(&self) -> f64 {
        return 1.0 - self.read_share();
    }

    /// Совпадает ли прочитанное с ожидаемым размером.
    pub fn matches(&self, expected: u64) -> bool {
        return !self.cancelled && self.bytes_copied == expected;
    }

    /// Узкое место: чтение, запись или баланс.
    pub fn bottleneck(&self) -> &'static str {
        let r: f64 = self.read_share();
        if r > 0.6 {
            return "чтение";
        }
        if r < 0.4 {
            return "запись";
        }
        return "баланс";
    }
}

// ---------------------------------------------------------------------------
// Форматтеры для UI
// ---------------------------------------------------------------------------

pub fn format_bytes(n: u64) -> String {
    if n < 1024 {
        return format_str(format_args!("{n} Б"));
    }
    if n < 1024 * 1024 {
        let k: f64 = n as f64 / 1024.0;
        return format_str(format_args!("{k:.1} КБ"));
    }
    if n < 1024 * 1024 * 1024 {
        let m: f64 = n as f64 / (1024.0 * 1024.0);
        return format_str(format_args!("{m:.2} МБ"));
    }
    let g: f64 = n as f64 / (1024.0 * 1024.0 * 1024.0);
    return format_str(format_args!("{g:.2} ГБ"));
}

pub fn format_chunk_sizes(sizes: &[usize]) -> String {
    if sizes.is_empty() {
        return "(нет)".to_string();
    }
    let mut parts: Vec<String> = Vec::with_capacity(sizes.len());
    for s in sizes {
        parts.push(format_str(format_args!("{}", format_bytes(*s as u64))));
    }
    return parts.join(", ");
}

/// Развёрнутый отчёт для UI: статус, объём, время, скорость, узкое место,
/// число обновлений прогресса и yield'ов.
pub fn format_stats(stats: &CopyStats, expected: u64) -> String {
    let status: String = if stats.cancelled {
        "⚠ Отменено — destination откатан".to_string()
    } else if stats.matches(expected) {
        "✓ Скопировано полностью, размер совпадает".to_string()
    } else {
        format_str(format_args!(
            "⚠ Скопировано {} из {} — расхождение",
            format_bytes(stats.bytes_copied),
            format_bytes(expected),
        ))
    };

    let read_pct: f64 = stats.read_share() * 100.0;
    let write_pct: f64 = stats.write_share() * 100.0;

    let mut lines: Vec<String> = Vec::with_capacity(11);
    lines.push(status);
    lines.push(format_str(format_args!(
        "Скопировано:  {} · чанков: {}",
        format_bytes(stats.bytes_copied), stats.chunks,
    )));
    lines.push(format_str(format_args!(
        "Progress:     {} обновлений · yield'ов: {}",
        stats.progress_calls, stats.yields,
    )));
    lines.push(format_str(format_args!(
        "Chunk size:   {} (adaptive)",
        format_bytes(stats.chunk_size as u64),
    )));
    lines.push(format_str(format_args!(
        "Время:        total {:.0} ms · read {:.0} ms ({:.0}%) · write {:.0} ms ({:.0}%)",
        stats.total_ms, stats.read_ms, read_pct,
        stats.write_ms, write_pct,
    )));
    lines.push(format_str(format_args!(
        "Скорость:     {:.2} МБ/с · узкое место: {}",
        stats.avg_speed_mbps(), stats.bottleneck(),
    )));
    lines.push(format_str(format_args!(
        "Размеры первых чанков: {}",
        format_chunk_sizes(&stats.first_chunk_sizes),
    )));
    return lines.join("\n");
}
```

### `src/fsa_large_copy.rs`

```rust
//! Потоковое копирование `File` → `FileSystemFileHandle` без полной
//! буферизации.
//!
//! Схема:
//! 1. Открыть writable у destination.
//! 2. Получить reader у source.
//! 3. Цикл: read один чанк → write его → следующий.
//! 4. Yield раз в `opts.yield_interval_ms` мс, чтобы UI успевал отрисовать.
//! 5. `commit()` фиксирует; `abort()` откатывает при отмене.
//!
//! **Backpressure реальна.** `writer.write(chunk).await` блокирует,
//! пока диск не готов принять следующий блок. Мы не читаем вперёд —
//! читаем ровно столько, сколько записали. Это даёт memory bound
//! в один `chunk_size`.
//!
//! **Параллелить read и write нельзя.** `join!(read, write)` превратит
//! pipeline в два независимых потока данных: буферы растут, memory
//! bound рушится, а выигрыш в throughput нулевой — диск всё равно один.
//!
//! **Yield управляем параметром.** `yield_interval_ms = 16.0` — плавный
//! UI, `0.0` — синхронный цикл (UI зависает до конца). Оба варианта
//! оставлены намеренно: демо показывает разницу.
//!
//! **Проверка — не догадка.** `sha256_file` даёт потоковый SHA-256
//! файла с memory O(1); `format_hash_verdict` сравнивает два хеша
//! и возвращает человекочитаемый вердикт.

use gloo_timers::future::TimeoutFuture;
use js_sys::Uint8Array;
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{File, FileSystemFileHandle, ReadableStreamDefaultReader};

use crate::fsa_choose_options::format_str;
use crate::fsa_large_chunk::{ChunkPlan, CopyStats, SpeedEma};
use crate::fsa_write_ops::WriteSession;

/// Опции копирования.
#[derive(Debug, Clone, Copy)]
pub struct CopyOptions {
    /// Интервал yield'а в мс. `16.0` — плавный UI, `0.0` — без yield.
    pub yield_interval_ms: f64,
}

impl CopyOptions {
    pub fn with_yield() -> Self {
        return CopyOptions { yield_interval_ms: 16.0 };
    }
    pub fn without_yield() -> Self {
        return CopyOptions { yield_interval_ms: 0.0 };
    }
}

/// Итог копирования.
pub enum CopyOutcome {
    Done(CopyStats),
    Cancelled(CopyStats),
}

/// Копирование с дефолтным yield'ом (16 мс).
///
/// `on_progress(frac, speed_mbps)` вызывается не чаще 16 мс.
/// `is_cancelled()` проверяется на каждой итерации и перед записью.
pub async fn copy_stream(
    src: &File,
    dst_handle: &FileSystemFileHandle,
    on_progress: impl FnMut(f64, f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<CopyOutcome, String> {
    return copy_stream_with_options(
        src,
        dst_handle,
        CopyOptions::with_yield(),
        on_progress,
        is_cancelled,
    )
    .await;
}

/// Копирование с явными опциями.
///
/// `opts.yield_interval_ms = 0.0` отключает yield: цикл становится
/// полностью синхронным, `on_progress` вызовется ровно один раз —
/// в самом конце. Это counterfactual-режим для демонстрации
/// необходимости yield'а.
pub async fn copy_stream_with_options(
    src: &File,
    dst_handle: &FileSystemFileHandle,
    opts: CopyOptions,
    mut on_progress: impl FnMut(f64, f64),
    is_cancelled: impl Fn() -> bool,
) -> Result<CopyOutcome, String> {
    let plan: ChunkPlan = ChunkPlan::for_total(src.size() as u64);

    let stream = src.stream();
    let reader = ReadableStreamDefaultReader::new(&stream)
        .map_err(|e| format_str(format_args!("reader: {e:?}")))?;

    // Открываем writable. keepExistingData=false: destination создаётся
    // с нуля — при отмене ничего не останется.
    let mut session = WriteSession::open(dst_handle, false)
        .await
        .map_err(|e| e.explain())?;

    let total: u64 = src.size() as u64;
    let mut bytes_copied: u64 = 0;
    let mut chunks: u32 = 0;
    let mut yields: u32 = 0;
    let mut progress_calls: u32 = 0;
    let mut read_ms: f64 = 0.0;
    let mut write_ms: f64 = 0.0;
    let mut first_chunk_sizes: Vec<usize> = Vec::with_capacity(8);
    let mut last_yield: f64 = 0.0;
    let t_total: f64 = js_sys::Date::now();
    let mut speed = SpeedEma::new(0.25);
    let mut last_speed_t: f64 = t_total;
    let mut last_speed_b: u64 = 0;
    let mut cancelled_flag: bool = false;
    let do_yield: bool = opts.yield_interval_ms > 0.0;

    loop {
        if is_cancelled() {
            cancelled_flag = true;
            break;
        }

        // --- Read ---
        let t_read: f64 = js_sys::Date::now();
        let result = JsFuture::from(reader.read())
            .await
            .map_err(|e| format_str(format_args!("read: {e:?}")))?;
        read_ms += js_sys::Date::now() - t_read;

        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))
            .ok()
            .and_then(|v| v.as_bool())
            .unwrap_or(true);
        if done {
            break;
        }

        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))
            .map_err(|e| format_str(format_args!("value: {e:?}")))?;
        let chunk = Uint8Array::new(&value);
        let bytes: Vec<u8> = chunk.to_vec();
        let len: usize = bytes.len();

        if first_chunk_sizes.len() < 8 {
            first_chunk_sizes.push(len);
        }

        // --- Write (with backpressure) ---
        let t_write: f64 = js_sys::Date::now();
        session
            .write(&bytes)
            .await
            .map_err(|e| e.explain())?;
        write_ms += js_sys::Date::now() - t_write;

        bytes_copied += len as u64;
        chunks += 1;

        // --- Yield раз в opts.yield_interval_ms + обновление прогресса ---
        if do_yield {
            let now: f64 = js_sys::Date::now();
            if now - last_yield >= opts.yield_interval_ms {
                let frac: f64 = if total > 0 {
                    bytes_copied as f64 / total as f64
                } else {
                    1.0
                };

                let dt: f64 = now - last_speed_t;
                let db: u64 = bytes_copied - last_speed_b;
                speed.update(db, dt);
                last_speed_t = now;
                last_speed_b = bytes_copied;

                on_progress(frac, speed.value_mbps());
                progress_calls += 1;
                TimeoutFuture::new(0).await;
                last_yield = now;
                yields += 1;
            }
        }
    }

    // --- Отмена: abort, destination не остаётся с частичными данными ---
    if cancelled_flag {
        let _ = session.abort().await;
        let total_ms: f64 = js_sys::Date::now() - t_total;
        let stats = CopyStats {
            bytes_copied, chunks, yields, progress_calls,
            read_ms, write_ms, total_ms,
            chunk_size: plan.chunk_size,
            cancelled: true,
            first_chunk_sizes,
        };
        return Ok(CopyOutcome::Cancelled(stats));
    }

    // --- Commit: сливаем буфер, закрываем поток ---
    let t_commit: f64 = js_sys::Date::now();
    let _ = session.commit().await.map_err(|e| e.explain())?;
    write_ms += js_sys::Date::now() - t_commit;

    on_progress(1.0, speed.value_mbps());
    progress_calls += 1;
    let total_ms: f64 = js_sys::Date::now() - t_total;
    let stats = CopyStats {
        bytes_copied, chunks, yields, progress_calls,
        read_ms, write_ms, total_ms,
        chunk_size: plan.chunk_size,
        cancelled: false,
        first_chunk_sizes,
    };
    return Ok(CopyOutcome::Done(stats));
}

// ---------------------------------------------------------------------------
// SHA-256 файла
// ---------------------------------------------------------------------------

/// Потоковый SHA-256 файла. Память O(1) — не зависит от размера.
///
/// Совпадение размеров источника и приёмника — необходимое, но не
/// достаточное условие корректности копии. SHA-256 даёт достаточное.
pub async fn sha256_file(handle: &FileSystemFileHandle) -> Result<String, String> {
    use sha2::{Digest, Sha256};

    let file = crate::fsa_api_file::get_file(handle)
        .await
        .map_err(|e| format_str(format_args!("get_file: {e:?}")))?;

    let stream = file.stream();
    let reader = ReadableStreamDefaultReader::new(&stream)
        .map_err(|e| format_str(format_args!("reader: {e:?}")))?;

    let mut hasher = Sha256::new();

    loop {
        let result = JsFuture::from(reader.read())
            .await
            .map_err(|e| format_str(format_args!("read: {e:?}")))?;
        let done = js_sys::Reflect::get(&result, &JsValue::from_str("done"))
            .ok()
            .and_then(|v| v.as_bool())
            .unwrap_or(true);
        if done {
            break;
        }
        let value = js_sys::Reflect::get(&result, &JsValue::from_str("value"))
            .map_err(|e| format_str(format_args!("value: {e:?}")))?;
        let chunk = Uint8Array::new(&value);
        hasher.update(&chunk.to_vec());
    }

    return Ok(hex::encode(hasher.finalize()));
}

/// Сравнить хеши источника и приёмника, вернуть человекочитаемый вердикт.
pub fn format_hash_verdict(src_hash: &str, dst_hash: &str) -> String {
    let match_line: String = if src_hash == dst_hash {
        "✓ Хеши совпадают — копия побитово идентична источнику.".to_string()
    } else {
        "✗ Хеши различаются — копия повреждена.".to_string()
    };
    let src_short: String = if src_hash.len() >= 16 { src_hash[..16].to_string() } else { src_hash.to_string() };
    let dst_short: String = if dst_hash.len() >= 16 { dst_hash[..16].to_string() } else { dst_hash.to_string() };
    let s: String = format_str(format_args!(
        "{match_line}\n\
         Источник:  {src_short}…\n\
         Приёмник:  {dst_short}…",
    ));
    return s;
}
```

---

## Как добавить в `lib.rs`

```rust
pub mod fsa_large_chunk;
pub mod fsa_large_copy;
```

---

## Тесты

### `tests/wasm_fsa_large.rs`

```rust
//! Тесты `fsa_large_*` — план чанков, EMA скорости, копирование OPFS → OPFS,
//! counterfactual без yield, SHA-256 источника и приёмника.

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
            .map_err(|_| JsValue::from_str("не Promise"))?;
        let value = JsFuture::from(promise).await?;
        value
            .dyn_into::<FileSystemDirectoryHandle>()
            .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
    }
}

use hello_wasm02::fsa_large_chunk::{
    format_bytes, format_chunk_sizes, format_stats, ChunkPlan, CopyStats,
    SpeedEma, MAX_CHUNK, MIN_CHUNK,
};
use hello_wasm02::fsa_large_copy::{
    copy_stream, copy_stream_with_options, format_hash_verdict, sha256_file,
    CopyOptions, CopyOutcome,
};
use hello_wasm02::fsa_write_ops::{read_bytes_now, WriteSession};
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// ChunkPlan
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn plan_small_file_uses_min_chunk() {
    let plan = ChunkPlan::for_total(4 * 1024);
    assert_eq!(plan.chunk_size, MIN_CHUNK);
    assert_eq!(plan.expected_chunks, 1);
}

#[wasm_bindgen_test]
fn plan_mid_file_within_bounds() {
    let plan = ChunkPlan::for_total(200 * 1024 * 1024);
    assert!(plan.chunk_size >= MIN_CHUNK);
    assert!(plan.chunk_size <= MAX_CHUNK);
    assert_eq!(plan.expected_chunks, 200);
}

#[wasm_bindgen_test]
fn plan_huge_file_clamps_to_max() {
    let plan = ChunkPlan::for_total(10 * 1024 * 1024 * 1024);
    assert_eq!(plan.chunk_size, MAX_CHUNK);
}

#[wasm_bindgen_test]
fn plan_zero_size() {
    let plan = ChunkPlan::for_total(0);
    assert_eq!(plan.expected_chunks, 0);
    assert_eq!(plan.memory_saved(), 0);
}

#[wasm_bindgen_test]
fn plan_memory_saved_positive_for_large() {
    let plan = ChunkPlan::for_total(100 * 1024 * 1024);
    assert!(plan.memory_saved() > 0);
    assert_eq!(plan.memory_saved(), plan.total_bytes - plan.chunk_size as u64);
}

#[wasm_bindgen_test]
fn plan_describe_contains_all_parts() {
    let plan = ChunkPlan::for_total(100 * 1024 * 1024);
    let s = plan.describe();
    assert!(s.contains("План"));
    assert!(s.contains("Пик памяти"));
    assert!(s.contains("экономия"));
}

// ---------------------------------------------------------------------------
// SpeedEma
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn ema_first_value_is_raw() {
    let mut ema = SpeedEma::new(0.25);
    ema.update(1_048_576, 1000.0);
    assert!((ema.value_mbps() - 1.0).abs() < 1e-6);
}

#[wasm_bindgen_test]
fn ema_smooths_subsequent_values() {
    let mut ema = SpeedEma::new(0.5);
    ema.update(1_048_576, 1000.0);
    ema.update(3 * 1_048_576, 1000.0);
    assert!((ema.value_mbps() - 2.0).abs() < 1e-6);
}

#[wasm_bindgen_test]
fn ema_ignores_zero_elapsed() {
    let mut ema = SpeedEma::new(0.25);
    ema.update(1_048_576, 1000.0);
    let before = ema.value_mbps();
    ema.update(1_048_576, 0.0);
    assert_eq!(ema.value_mbps(), before);
}

// ---------------------------------------------------------------------------
// CopyStats
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn stats_bottleneck_read() {
    let s = CopyStats {
        bytes_copied: 100, chunks: 1, yields: 0, progress_calls: 0,
        read_ms: 80.0, write_ms: 20.0, total_ms: 100.0,
        chunk_size: 100, cancelled: false, first_chunk_sizes: vec![100],
    };
    assert_eq!(s.bottleneck(), "чтение");
    assert!((s.read_share() - 0.8).abs() < 1e-6);
}

#[wasm_bindgen_test]
fn stats_bottleneck_write() {
    let s = CopyStats {
        bytes_copied: 100, chunks: 1, yields: 0, progress_calls: 0,
        read_ms: 20.0, write_ms: 80.0, total_ms: 100.0,
        chunk_size: 100, cancelled: false, first_chunk_sizes: vec![100],
    };
    assert_eq!(s.bottleneck(), "запись");
}

#[wasm_bindgen_test]
fn stats_matches_requires_not_cancelled() {
    let s = CopyStats {
        bytes_copied: 100, chunks: 1, yields: 0, progress_calls: 0,
        read_ms: 1.0, write_ms: 1.0, total_ms: 2.0,
        chunk_size: 100, cancelled: true, first_chunk_sizes: vec![100],
    };
    assert!(!s.matches(100), "cancel отменяет match");
}

// ---------------------------------------------------------------------------
// Форматтеры
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn format_bytes_units() {
    assert_eq!(format_bytes(0), "0 Б");
    assert_eq!(format_bytes(512), "512 Б");
    assert_eq!(format_bytes(2048), "2.0 КБ");
    assert_eq!(format_bytes(5 * 1024 * 1024), "5.00 МБ");
}

#[wasm_bindgen_test]
fn format_chunk_sizes_empty() {
    assert_eq!(format_chunk_sizes(&[]), "(нет)");
}

#[wasm_bindgen_test]
fn format_chunk_sizes_nonempty() {
    let s = format_chunk_sizes(&[1024, 2048]);
    assert!(s.contains("1.0 КБ"));
    assert!(s.contains("2.0 КБ"));
}

// ---------------------------------------------------------------------------
// Копирование OPFS → OPFS
// ---------------------------------------------------------------------------

async fn prepare_src(
    root: &web_sys::FileSystemDirectoryHandle,
    name: &str,
    size: usize,
) -> web_sys::FileSystemFileHandle {
    let _ = hello_wasm02::fsa_api_directory::remove_entry(root, name, true).await;
    let handle = hello_wasm02::fsa_api_directory::get_file_handle(root, name, true)
        .await
        .expect("create src");
    let data: Vec<u8> = (0..size).map(|i| (i % 251) as u8).collect();
    let mut s = WriteSession::open(&handle, false).await.expect("open src");
    s.write(&data).await.expect("write src");
    s.commit().await.expect("commit src");
    return handle;
}

async fn prepare_dst(
    root: &web_sys::FileSystemDirectoryHandle,
    name: &str,
) -> web_sys::FileSystemFileHandle {
    let _ = hello_wasm02::fsa_api_directory::remove_entry(root, name, true).await;
    hello_wasm02::fsa_api_directory::get_file_handle(root, name, true)
        .await
        .expect("create dst")
}

#[wasm_bindgen_test]
async fn copy_small_file_bitwise_equal() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let size = 256 * 1024;
    let src_handle = prepare_src(&root, "lg_src_small.bin", size).await;
    let dst_handle = prepare_dst(&root, "lg_dst_small.bin").await;

    let src_file = hello_wasm02::fsa_api_file::get_file(&src_handle)
        .await
        .expect("get src file");

    let outcome = copy_stream(&src_file, &dst_handle, |_, _| {}, || false)
        .await
        .expect("copy");

    match outcome {
        CopyOutcome::Done(stats) => {
            assert_eq!(stats.bytes_copied, size as u64);
            assert!(stats.matches(size as u64));
            assert!(!stats.cancelled);
        }
        CopyOutcome::Cancelled(_) => panic!("не должно быть отменено"),
    }

    let src_bytes = read_bytes_now(&src_handle).await.expect("read src");
    let dst_bytes = read_bytes_now(&dst_handle).await.expect("read dst");
    assert_eq!(src_bytes, dst_bytes, "побитовое совпадение");

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_src_small.bin", true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_dst_small.bin", true).await;
}

#[wasm_bindgen_test]
async fn copy_4mb_bitwise_equal() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let size = 4 * 1024 * 1024;
    let src_handle = prepare_src(&root, "lg_src_4mb.bin", size).await;
    let dst_handle = prepare_dst(&root, "lg_dst_4mb.bin").await;

    let src_file = hello_wasm02::fsa_api_file::get_file(&src_handle)
        .await
        .expect("get src file");

    let progress_calls = std::rc::Rc::new(std::cell::Cell::new(0u32));
    let pc = progress_calls.clone();
    let outcome = copy_stream(
        &src_file,
        &dst_handle,
        move |_frac, _speed| pc.set(pc.get() + 1),
        || false,
    )
    .await
    .expect("copy");

    match outcome {
        CopyOutcome::Done(stats) => {
            assert!(stats.matches(size as u64));
            assert!(stats.yields >= 1, "yield должен сработать на 4 МБ");
            assert!(stats.chunks >= 1);
        }
        CopyOutcome::Cancelled(_) => panic!("не должно быть отменено"),
    }
    assert!(progress_calls.get() >= 1, "on_progress вызывался");

    let src_bytes = read_bytes_now(&src_handle).await.expect("read src");
    let dst_bytes = read_bytes_now(&dst_handle).await.expect("read dst");
    assert_eq!(src_bytes.len(), dst_bytes.len());
    assert_eq!(src_bytes, dst_bytes);

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_src_4mb.bin", true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_dst_4mb.bin", true).await;
}

#[wasm_bindgen_test]
async fn copy_cancel_discards_destination() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let size = 2 * 1024 * 1024;
    let src_handle = prepare_src(&root, "lg_src_cancel.bin", size).await;
    let dst_handle = prepare_dst(&root, "lg_dst_cancel.bin").await;

    let src_file = hello_wasm02::fsa_api_file::get_file(&src_handle)
        .await
        .expect("get src file");

    let cancel = std::rc::Rc::new(std::cell::Cell::new(true));
    let c = cancel.clone();
    let outcome = copy_stream(&src_file, &dst_handle, |_, _| {}, move || c.get())
        .await
        .expect("copy");

    match outcome {
        CopyOutcome::Cancelled(stats) => {
            assert!(stats.cancelled);
        }
        CopyOutcome::Done(_) => panic!("должно быть отменено"),
    }

    let src_bytes = read_bytes_now(&src_handle).await.expect("read src");
    assert_eq!(src_bytes.len(), size);

    let dst_size = match hello_wasm02::fsa_api_file::get_file(&dst_handle).await {
        Ok(f) => f.size() as u64,
        Err(_) => 0,
    };
    assert_eq!(dst_size, 0, "destination не остаётся с частичными данными");

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_src_cancel.bin", true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_dst_cancel.bin", true).await;
}

#[wasm_bindgen_test]
async fn copy_progress_monotonic() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let size = 512 * 1024;
    let src_handle = prepare_src(&root, "lg_src_prog.bin", size).await;
    let dst_handle = prepare_dst(&root, "lg_dst_prog.bin").await;

    let src_file = hello_wasm02::fsa_api_file::get_file(&src_handle)
        .await
        .expect("get src file");

    let values: std::rc::Rc<std::cell::RefCell<Vec<f64>>> =
        std::rc::Rc::new(std::cell::RefCell::new(Vec::new()));
    let v = values.clone();
    let _ = copy_stream(
        &src_file,
        &dst_handle,
        move |frac, _speed| v.borrow_mut().push(frac),
        || false,
    )
    .await
    .expect("copy");

    let vec = values.borrow();
    for w in vec.windows(2) {
        assert!(w[1] + 1e-9 >= w[0], "прогресс монотонный");
    }
    let last = *vec.last().unwrap_or(&0.0);
    assert!((last - 1.0).abs() < 1e-6, "финальный прогресс = 1.0");

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_src_prog.bin", true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_dst_prog.bin", true).await;
}

// ---------------------------------------------------------------------------
// no-yield vs with-yield
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn copy_no_yield_has_one_progress_call() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let size = 512 * 1024;
    let src_handle = prepare_src(&root, "lg_src_noyield.bin", size).await;
    let dst_handle = prepare_dst(&root, "lg_dst_noyield.bin").await;

    let src_file = hello_wasm02::fsa_api_file::get_file(&src_handle)
        .await
        .expect("get src file");

    let calls = std::rc::Rc::new(std::cell::Cell::new(0u32));
    let c = calls.clone();
    let outcome = copy_stream_with_options(
        &src_file,
        &dst_handle,
        CopyOptions::without_yield(),
        move |_, _| c.set(c.get() + 1),
        || false,
    )
    .await
    .expect("copy");

    match outcome {
        CopyOutcome::Done(stats) => {
            assert_eq!(stats.yields, 0, "без yield не должно быть yield'ов");
            assert_eq!(stats.progress_calls, 1, "on_progress только в конце");
        }
        _ => panic!("должно быть Done"),
    }

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_src_noyield.bin", true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_dst_noyield.bin", true).await;
}

// ---------------------------------------------------------------------------
// sha256_file
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn sha256_matches_for_identical_files() {
    let root = match opfs::root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let size = 256 * 1024;
    let src_handle = prepare_src(&root, "lg_src_hash.bin", size).await;
    let dst_handle = prepare_dst(&root, "lg_dst_hash.bin").await;

    let src_file = hello_wasm02::fsa_api_file::get_file(&src_handle)
        .await
        .expect("get src file");

    let _ = copy_stream(
        &src_file,
        &dst_handle,
        |_, _| {},
        || false,
    )
    .await
    .expect("copy");

    let src_hash = sha256_file(&src_handle).await.expect("hash src");
    let dst_hash = sha256_file(&dst_handle).await.expect("hash dst");

    assert_eq!(src_hash, dst_hash, "хеши источника и приёмника совпадают");
    assert_eq!(src_hash.len(), 64);

    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_src_hash.bin", true).await;
    let _ = hello_wasm02::fsa_api_directory::remove_entry(&root, "lg_dst_hash.bin", true).await;
}

#[wasm_bindgen_test]
fn hash_verdict_matches() {
    let h = "abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789";
    let s = format_hash_verdict(h, h);
    assert!(s.contains("✓"));
    assert!(s.contains("abcdef0123456789"));
}

#[wasm_bindgen_test]
fn hash_verdict_mismatch() {
    let a = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
    let b = "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb";
    let s = format_hash_verdict(a, b);
    assert!(s.contains("✗"));
}
```

Итого тестов: **23** (19 исходных + 4 новых: `copy_no_yield_has_one_progress_call`, `sha256_matches_for_identical_files`, `hash_verdict_matches`, `hash_verdict_mismatch`).

---

## Демо-пример в `src/app.rs`

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
    wire_block_b(&b, state);

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

            // Пересоздаём источник.
            let _ = fsa_api_directory::remove_entry(&root, SRC_NAME, true).await;
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
    let mut lines: Vec<String> = Vec::with_capacity(18);
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
         ограничена буфером, а не размером файла)."
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

**Строка ввода размера источника.**
Пользователь задаёт любое значение от 1 до 500 МБ. Под полем — live-подсказка, которая обновляется на каждый `input`:
```
→ при 24 МБ: chunk 64.0 КБ · ~384 итераций
Прочитать весь файл в память:  peak 24.00 МБ
Читать и писать чанками:       peak 64.0 КБ
```
При 1 МБ chunk упирается в нижнюю границу (64 КБ) — итераций ~16. При 200 МБ chunk растёт до 1 МБ — итераций ровно 200. При 500 МБ chunk упирается в верхнюю границу 4 МиБ — итераций ~125.

**Блок A: плавное копирование.**
Прогресс и скорость обновляются десятки раз. Метрики: `Progress: N обновлений · yield'ов: M`. Кнопка «Проверить SHA-256» доказывает, что копия побитово идентична источнику.

**Блок B: counterfactual без yield.**
Тот же файл, тот же цикл, но `opts.yield_interval_ms = 0.0`. Прогресс-бар прыгает с 0 % на 100 %, кнопка «Отмена» не срабатывает, UI замирает на время копирования. Метрики: `Progress: 1 обновление · yield'ов: 0` + предупреждение «UI не обновлялся во время копирования».

**Counterfactual памяти.**
После подготовки источника — жёлтая панель с описательными формулировками:
```
Counterfactual (для 24 МБ):
• Прочитать весь файл в память: peak 24.00 МБ RAM
• Читать и писать чанками:      peak 64.0 КБ RAM
• Экономия памяти:              23.94 МБ
Попробуйте задать 500 МБ: прочитать такой файл целиком уже не получится — а чанковое чтение справится.
```

**Backpressure.**
Цикл — `read chunk → write chunk → read next`. Никакого предчтения. Метрики показывают раздельно `read_ms` и `write_ms`: если write занимает 70 % — узкое место «запись»; если read 70 % — «чтение».

**Отмена.**
Кнопка «Отмена» ставит флаг, цикл останавливается на ближайшем yield. `WriteSession::abort()` откатывает приёмник — файл не остаётся с половиной данных. Источник нетронут. Тест `copy_cancel_discards_destination` это подтверждает.

**Генерация источника не съедает память.**
Источник пишется чанками по 64 КиБ (`write_source_chunked`). Даже при выборе 500 МБ пик памяти при подготовке — 64 КиБ, а не 500 МБ. Без этого шага поле ввода размера было бы опасно: `Vec<u8>` на 500 МБ в Wasm-linear-memory вполне может уронить вкладку до начала самого эксперимента.

## Правила проекта, зафиксированные в 19.6

1. **`app.rs` содержит только демо 19.6** — предыдущие подглавы не накапливаются.
2. **`read chunk → write chunk` — последовательно.** Параллелить нельзя: буферы растут, memory bound рушится, а выигрыш нулевой — диск всё равно один.
3. **Backpressure реальна и измерима.** `writer.write(chunk).await` блокирует, пока диск не готов; это видно в `write_ms` метрик.
4. **Yield — параметр, а не константа.** `CopyOptions::with_yield()` для UI, `without_yield()` для counterfactual. Разница в один вызов `TimeoutFuture::new(0).await` становится видимой только при сравнении двух режимов рядом.
5. **Adaptive chunk — параметр, а не стратегия.** Одно число, зависящее от объёма: `clamp(total / 200, 64 КиБ, 4 МиБ)`.
6. **Live-скорость через EMA.** Сырые значения в первые 100 мс — шум.
7. **Отмена не оставляет мусора.** `abort()` откатывает destination; source нетронут.
8. **Проверка — не догадка.** Размер — необходимое, но не достаточное условие. SHA-256 обоих файлов даёт достаточное доказательство побитовой идентичности.
9. **Counterfactual — главный образовательный приём.** Показать «как было бы без yield» / «как было бы без streaming» эффективнее, чем рассказать.
10. **Размер источника — параметр UI.** 1…500 МБ. Генерация источника — чанками, чтобы подготовка не съедала память. Live-подсказка показывает план для текущего значения.
11. **Терминология UI — описательная.** Вместо жаргонного «наивно» (легко спутать с «нативно») используется «прочитать весь файл в память» vs «читать и писать чанками». Определения даны в help-тексте.
12. **Не делать chunk size sweep.** Браузерный `ReadableStreamDefaultReader` не даёт контролировать размер читаемых чанков — мы получаем, что даёт браузер. Sweep был бы самообманом.
13. **Все строки через `format_str` или `Vec<String>.join`.**
14. **`return` в хвостах макросов** (`matches!`, `format!`).
15. **Внутри `Closure::new(move || …)` элементы клонируются до `move`.**
16. **Внутри `src/app.rs` — только `crate::…`, не `hello_wasm02::…`.** Прямое обращение к собственному крейту по имени не резолвится.
17. **`cargo check` без предупреждений.**

## Команды

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_fsa_large
npm start
```

Ожидаемый результат тестов: **23 passed, 0 failed**.