# Глава 23.12. Практический проект: MP4 → обработка → MP4

## 23.12.0. Обновлённая структура

```
src/
  pipeline_worker.rs              ← НОВЫЙ модуль 23.12-W
  mp4_muxer.rs                    ← из 23.11 (обновлён: per-chunk durations)
  mp4_demuxer.rs                  ← из 23.10 (обновлён: audio track)
  container_parser.rs             ← из 23.9
  video_filters.rs                ← из 23.12 (базовый, без изменений)
  video_frame.rs                  ← из 23.1
  app.rs                          ← демо 23.12 (переписан)
tests/
  wasm_mp4_muxer.rs               ← из 23.11
  wasm_mp4_demuxer.rs             ← из 23.10
web/
  public/workers/
    mp4_worker.js                 ← НОВЫЙ worker (загрузчик)
src/lib.rs                        ← +1 строка: pub mod pipeline_worker;
Cargo.toml                        ← +features: Worker, OffscreenCanvas, ImageBitmap
```

**Комментарий.** Глава объединяет всё, что было в 23.9–23.11, в сквозной pipeline: **MP4 → обработка пикселей → MP4**. Отличие от ранних версий — **N параллельных воркеров** с разбивкой по GOP'ам, **GPU-фильтр через OffscreenCanvas**, **audio passthrough**, **Baseline Profile для encoder'а** (ключевое открытие этой главы), **график FPS в реальном времени**.

**Ссылки для детального изучения:**

- W3C WebCodecs: [`VideoEncoder`](https://w3c.github.io/webcodecs/#videoencoder-interface)
- W3C WebCodecs: [`EncodedVideoChunk`](https://w3c.github.io/webcodecs/#encodedvideochunk-interface)
- W3C WebCodecs: [`VideoFrame`](https://w3c.github.io/webcodecs/#videoframe-interface)
- ISO/IEC 14496-12: [ISO Base Media File Format — ctts box](https://www.iso.org/standard/68960.html) §8.6.1.3
- Wikipedia: [H.264 Profiles (Baseline vs High, B-frames)](https://en.wikipedia.org/wiki/Advanced_Video_Coding#Profiles)
- MDN: [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- MDN: [`CanvasRenderingContext2D.filter`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/filter)

---

## 23.12.1. О чём эта глава

Полный цикл обработки видеофайла в браузере без ffmpeg.wasm:

```
input.mp4 ──► demux ──► EncodedVideoChunk[] ──► N worker'ов
                                                    │
                                       decode → filter → encode
                                                    │
                                                    ▼
output.mp4 ◄── mux ◄── EncodedVideoChunk[] ◄────────┘
              (audio passthrough)
```

**Что демонстрируется:**

- Разбивка исходного потока на независимые GOP'ы.
- Распределение GOP'ов по N воркерам (каждый со своим decoder/encoder/OffscreenCanvas).
- GPU-фильтр: `ctx.filter = "grayscale(100%)"` + `drawImage(frame)` + `transferToImageBitmap` + `new VideoFrame(bitmap)`.
- Audio passthrough: сжатые байты audio-трека копируются в новый `mdat` как есть.
- Mux с SPS/PPS **от encoder'а** (не от source).
- График скорости обработки в реальном времени.

**Что НЕ демонстрируется:** MP4 с переменным frame rate, B-frames (см. 23.12.7 — почему), HDR, 10-bit.

---

## 23.12.2. Архитектура

### N параллельных воркеров

Задача: обработать 2175 кадров 720×1280 grayscale'ом.

Наивный подход — один воркер — даёт ~22 fps. Но у браузера есть несколько ядер, и Chrome даёт каждому `Worker`-у отдельные квоты на GPU и encoder.

**Ключевое наблюдение:** H.264 video — это набор **GOP**'ов (Group of Pictures). Каждый GOP начинается с key-кадра (IDR) и содержит N delta-кадров, которые зависят только от **своего** key-кадра. Значит GOP'ы **независимы** — можно декодировать параллельно.

Разбивка:
```
samples: [K D D D K D D D K D D D ...]     (K=key, D=delta)
   │        │       │       │
   ▼        ▼       ▼       ▼
 GOP#0    GOP#1   GOP#2   GOP#3
   │        │       │       │
   └────────┴───────┴───────┘
              │
              ▼
        distribute по N воркерам
        (round-robin)
```

Каждый воркер получает **свой список chunks**, свой `VideoDecoder`, свой `VideoEncoder`, свой `OffscreenCanvas`. Обрабатывает независимо, отдаёт результат. Main thread ждёт **всех**, объединяет, сортирует по timestamp, mux'ит.

**Ускорение:** на 8 воркерах с 15 GOP'ами получилось ~41 fps avg против ~22 fps на одном. Ускорение не линейное по N — есть overhead на message-passing и финальный mux, но 2× достижимо.

### Модель отмены

Воркеры живут в `Vec<WorkerHandle>`. `worker.terminate()` уничтожает воркер **немедленно** — не нужны session-проверки внутри, не нужны ack-сообщения. При отмене:
1. `terminate()` всех воркеров.
2. Spawn новых — на следующий запуск.

Это **проще**, чем session-инкременты, потому что `terminate()` гарантированно убивает всё.

### Границы данных

Каждый chunk — `{is_key, ts, dur, data: ArrayBuffer}`. Передаётся через `postMessage` с **transfer** — zero-copy. Воркер возвращает массив `EncodedVideoChunk` как JS-объекты, тоже через transfer.

---

## 23.12.3. Ключевое открытие: B-frames и ctts

Это открытие стоило нескольких итераций и является самым важным практическим уроком главы.

### Симптом

После успешного encoding'а и mux'а видео **играется, но дёргается** — кадры идут не в том порядке, скачут вперёд-назад, повторяются. Скорость воспроизведения в порядке, звук синхронный. Просто визуально рвано.

### Диагноз

Encoder Chrome'а по умолчанию использует **High Profile** (`avc1.64001F`), а с ним — **B-frames**. B-frame — двунаправленный: он ссылается на предыдущий **и следующий** кадр.

В декодере это означает: кадр должен быть **декодирован** раньше, чем его "предыдущий" сосед (потому что нужен для его реконструкции). Но **показан** он должен позже.

Возникает расхождение **PTS ≠ DTS**:

```
Порядок декодирования (DTS):  I  P  B  B  P  B  B
Порядок отображения (PTS):    I  B  B  P  B  B  P
```

Для MP4 это выражается в отдельном box'е **`ctts` (Composition Time to Sample)** — таблица, которая для каждого сэмпла говорит, **насколько сдвинуть его PTS относительно DTS**.

**У нашего muxer'а `ctts` нет.** Он пишет только `stts` (decode durations). Плеер предполагает `PTS == DTS` и показывает кадры в порядке чтения — но при B-frames этот порядок ≠ порядок отображения.

Результат — дёрганье.

### Решение

**Принудительно использовать Baseline Profile для encoder'а.**

```rust
const ENCODER_CODEC: &str = "avc1.42E01E"; // Baseline 3.1
```

Baseline Profile **не поддерживает B-frames** по спецификации. Следовательно, `PTS == DTS`, `ctts` не нужен, и всё сходится.

Декодер продолжает использовать **source**-avcC (в исходном файле может быть High Profile — плеер это умеет декодировать). А вот **encoder** теперь всегда Baseline, и его avcC пишется в output.

### Проверка в логах

```
[pipeline-worker] encoder forced to avc1.42E01E (Baseline, no B-frames)
[pipeline-worker] bucket done: 300 chunks, avcc=42 B, codec=avc1.42E01E
[main] muxing with encoder avcc: 42 B, codec: avc1.42E01E
[main] first 5 chunks after sort: ts=0 k=1 dur=33333 | ts=33333 k=0 | ts=66666 k=0 | ...
```

- `avc1.42E01E` — Baseline 3.1.
- `avcc=42 B` — короткий avcC (у Baseline проще SPS/PPS).
- Монотонные timestamp'ы — PTS идут ровно через 33333 µs.

**Что было бы, если оставить High:** `avcc=43 B`, `codec=avc1.64001F`, и в файле — не-монотонные PTS. Плеер "перескакивал" бы кадры.

### Альтернативы

| Решение | Плюсы | Минусы |
|---------|-------|--------|
| **Baseline encoder** (наше) | Просто, работает, ~2% потери качества | Нет B-frames = чуть хуже компрессия |
| High + писать `ctts` | Полное качество High | Нужно расширить muxer, порядок сэмплов отличается от декодера |
| `latencyMode: "realtime"` | Может отключить B-frames | Не гарантировано, зависит от Chrome |

Выбрали первое — минимальная инженерная стоимость при том же результате.

---

## 23.12.4. Быстрый фильтр: OffscreenCanvas + ctx.filter

### Что было (медленный путь)

```
frame.copyTo(rgba)              ← readback, 10-30 ms
apply_filter(rgba) в WASM       ← CPU, 5-15 ms
new VideoFrame(rgba, {format})  ← upload, 10-30 ms
```

Три операции GPU↔CPU на кадр. На 720×1280 при 30 fps — десятки миллисекунд оверхеда.

### Что стало (быстрый путь)

```rust
ctx.set_filter("grayscale(100%)");        // строка, GPU-фильтр
ctx.draw_image(frame, 0, 0, w, h);        // GPU → GPU
let bitmap = canvas.transfer_to_image_bitmap(); // GPU-handle
let new_frame = new VideoFrame(bitmap, { ts, dur }); // GPU → GPU
```

**Всё на GPU.** `ctx.filter` поддерживается Chrome начиная с версии ~52, и его значение применяется к `drawImage` **на уровне GPU-шейдера**. `transferToImageBitmap()` не читает пиксели — передаёт GPU-handle.

**Ускорение:** filter-стадия упала с ~25 ms/кадр до ~6 ms/кадр (в 4×).

### Fallback

Если `ctx.filter` не поддерживается, автоматически используется CPU-путь. Feature detection:

```rust
let gpu_path = {
    let _ = set_prop(ctx.as_ref(), "filter", &JsValue::from_str("grayscale(100%)"));
    let back = Reflect::get(ctx.as_ref(), &JsValue::from_str("filter"))
        .ok().and_then(|v| v.as_string()).unwrap_or_default();
    back.contains("grayscale")
};
```

---

## 23.12.5. Audio passthrough

### Проблема

В первых итерациях audio-трек **терялся** — muxer умел только video.

Простое "перекодировать audio" не подходит: нужны `AudioDecoder`, `AudioEncoder`, обработка `AudioData`. Это отдельный пласт, и для фильтра графisйла звук вообще не трогается.

### Решение: passthrough

Сжатые байты audio-сэмплов **копируются в новый `mdat` как есть**. В `moov` создаётся второй `trak` с `soun`-handler'ом. `stsd` audio-трека берётся **из исходного файла** — там уже готовый `esds`-box с AAC-параметрами.

**Что копируется as-is:**
- `stsd` payload (AAC-specific: sample rate, channels, esds).
- Сами байты сэмплов.

**Что пересчитывается:**
- `stts`, `stsc`, `stsz`, `stco` — по факту.
- `mdhd.duration` — в единицах audio-timscale.

### Критично: `mdhd.duration` в единицах timescale

**Первый баг** прошлых итераций: `mdhd.duration` и `stts.sample_delta` для audio писались в **микросекундах**, а не в единицах `timescale` (44100).

Плеер видел: `sum(stts) ≈ 72_000_000`, а `mdhd.duration = 3_197_250` (правильно) — или наоборот, `duration = 72_500_000` при timescale 44100. В любом случае трек не сходился, и Chrome его **молча отбрасывал**.

**Fix:**
```rust
let media_duration_ts = if audio.timescale == 0 {
    0
} else {
    ((movie_duration_us as u64) * (audio.timescale as u64) / 1_000_000) as u32
};
let mdhd = build_mdhd(audio.timescale, media_duration_ts);
```

Аналогично в `stts`:
```rust
let durations_ts: Vec<u32> = audio.chunks.iter().map(|c| {
    ((c.duration_us as u64) * (audio.timescale as u64) / 1_000_000) as u32
}).collect();
```

После fix'а звук появился.

---

## 23.12.6. Исходные файлы

### `Cargo.toml` — фичи

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    # ... из предыдущих глав ...
    # 23.12 — worker + OffscreenCanvas
    "Worker",
    "WorkerOptions",
    "WorkerType",
    "MessageEvent",
    "OffscreenCanvas",
    "OffscreenCanvasRenderingContext2d",
    "ImageBitmap",
    "Blob",
    "BlobPropertyBag",
]
```

### `src/lib.rs`

```rust
pub mod app;
pub mod container_parser;
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod pipeline_worker;   // ← 23.12-W
pub mod video_filters;
pub mod video_frame;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

### `web/public/workers/mp4_worker.js`

```javascript
// 23.12-W. Загрузчик pipeline-worker'а.
//
// Импортирует wasm-pack сборку и вызывает Rust-функцию worker_run(),
// которая регистрирует self.onmessage.

self.postMessage({ type: "boot" });

(async () => {
    try {
        const mod = await import("/pkg/hello_wasm02.js");
        await mod.default();
        if (typeof mod.worker_run !== "function") {
            throw new Error("worker_run not exported from WASM module");
        }
        mod.worker_run();
        self.postMessage({ type: "ready" });
    } catch (err) {
        self.postMessage({
            type: "boot_error",
            message: String(err) + (err && err.stack ? " | " + err.stack : ""),
        });
    }
})();
```

### `src/mp4_demuxer.rs` — добавления

В конец файла `src/mp4_demuxer.rs` добавлено (не изменяя существующего):

```rust
use wasm_bindgen::JsValue;

// =============================================================================
// Audio track (passthrough)
// =============================================================================

#[derive(Debug, Clone)]
pub struct AudioSample {
    pub index: usize,
    pub offset: u64,
    pub size: u32,
    pub timestamp_us: u64,
    pub duration_us: u32,
}

pub struct AudioTrackInfo {
    /// Полный payload stsd-box'а — копируется как есть.
    pub stsd_payload: Vec<u8>,
    pub timescale: u32,
    pub sample_rate: u32,
    pub channels: u16,
    pub samples: Vec<AudioSample>,
}

/// Извлекает audio-трек из moov. None, если audio-трека нет.
pub fn extract_audio_track(moov: &BoxInfo) -> Result<Option<AudioTrackInfo>, String> {
    let mut audio_trak: Option<&BoxInfo> = None;
    for trak in &moov.children {
        if trak.box_type != "trak" { continue; }
        if let Some(hdlr) = trak.find("hdlr") {
            let p = &hdlr.payload;
            if p.len() >= 12 && &p[8..12] == b"soun" {
                audio_trak = Some(trak);
                break;
            }
        }
    }
    let Some(audio_trak) = audio_trak else {
        return Ok(None);
    };

    let mdhd = audio_trak.find("mdhd")
        .ok_or_else(|| String::from("audio: нет mdhd"))?;
    let timescale = parse_mdhd_timescale(mdhd)
        .ok_or_else(|| String::from("audio: mdhd.timescale = 0"))?;

    let stbl = audio_trak.find("stbl")
        .ok_or_else(|| String::from("audio: нет stbl"))?;
    let table = parse_sample_table(stbl, timescale)?;
    let raw_samples = build_samples(&table)?;

    let samples: Vec<AudioSample> = raw_samples.into_iter().map(|s| AudioSample {
        index: s.index, offset: s.offset, size: s.size,
        timestamp_us: s.timestamp_us, duration_us: s.duration_us,
    }).collect();

    let stsd = audio_trak.find("stsd")
        .ok_or_else(|| String::from("audio: нет stsd"))?;
    let stsd_payload = stsd.payload.clone();

    let mut sample_rate: u32 = 0;
    let mut channels: u16 = 0;
    if stsd_payload.len() >= 44 {
        channels = u16::from_be_bytes([stsd_payload[32], stsd_payload[33]]);
        let sr_fixed = u32::from_be_bytes([
            stsd_payload[40], stsd_payload[41],
            stsd_payload[42], stsd_payload[43],
        ]);
        sample_rate = sr_fixed >> 16;
    }
    if sample_rate == 0 || channels == 0 {
        return Err(String::from("audio: sample_rate или channels = 0"));
    }

    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[demux] audio: {} samples, {} Hz, {} ch, timescale={}",
        samples.len(), sample_rate, channels, timescale,
    )));

    Ok(Some(AudioTrackInfo {
        stsd_payload, timescale, sample_rate, channels, samples,
    }))
}
```

### `src/mp4_muxer.rs` — ключевые изменения

**Изменение 1:** `build_video_stbl` использует per-chunk durations для `stts` (а не первый duration для всех):

```rust
fn build_video_stbl(chunks: &[VideoChunk], config: &MuxConfig, mdat_body_start: u32) -> Vec<u8> {
    let total = chunks.len() as u32;
    let sizes: Vec<u32> = chunks.iter().map(|c| c.data.len() as u32).collect();
    let key_indices: Vec<u32> = chunks.iter().enumerate()
        .filter(|(_, c)| c.is_key).map(|(i, _)| (i + 1) as u32).collect();

    let stts = if chunks.is_empty() {
        build_stts(&[])
    } else {
        let first_dur = chunks[0].duration_us;
        let all_same = chunks.iter().all(|c| c.duration_us == first_dur);
        if all_same {
            build_stts(&[(total, first_dur)])
        } else {
            let durations: Vec<u32> = chunks.iter().map(|c| c.duration_us).collect();
            build_stts_from_durations_ts(&durations)
        }
    };

    let mut body = Vec::new();
    body.extend_from_slice(&build_video_stsd(config.width, config.height, &config.avcc));
    body.extend_from_slice(&stts);
    body.extend_from_slice(&build_stsc(total));
    body.extend_from_slice(&build_stsz(&sizes));
    body.extend_from_slice(&build_stco(mdat_body_start));
    body.extend_from_slice(&build_stss(&key_indices));
    build_box(b"stbl", &body)
}
```

**Изменение 2:** `build_audio_stbl` пересчитывает durations в единицах `audio.timescale`:

```rust
fn build_audio_stbl(audio: &AudioTrackInput, mdat_body_start: u32) -> Vec<u8> {
    let total = audio.chunks.len() as u32;
    let sizes: Vec<u32> = audio.chunks.iter().map(|c| c.data.len() as u32).collect();

    let durations_ts: Vec<u32> = audio.chunks.iter().map(|c| {
        if audio.timescale == 0 { 0 } else {
            ((c.duration_us as u64) * (audio.timescale as u64) / 1_000_000) as u32
        }
    }).collect();

    let mut body = Vec::new();
    body.extend_from_slice(&build_audio_stsd(&audio.stsd_payload));
    body.extend_from_slice(&build_stts_from_durations_ts(&durations_ts));
    body.extend_from_slice(&build_stsc(total));
    body.extend_from_slice(&build_stsz(&sizes));
    body.extend_from_slice(&build_stco(mdat_body_start));
    build_box(b"stbl", &body)
}
```

**Изменение 3:** `build_audio_mdia` — `mdhd.duration` в единицах `audio.timescale`:

```rust
fn build_audio_mdia(
    audio: &AudioTrackInput, mdat_body_start: u32, movie_duration_us: u32,
) -> Vec<u8> {
    let media_duration_ts = if audio.timescale == 0 { 0 } else {
        ((movie_duration_us as u64) * (audio.timescale as u64) / 1_000_000) as u32
    };
    let mut body = Vec::new();
    body.extend_from_slice(&build_mdhd(audio.timescale, media_duration_ts));
    body.extend_from_slice(&build_hdlr(b"soun", b"SoundHandler"));
    body.extend_from_slice(&build_audio_minf(audio, mdat_body_start));
    build_box(b"mdia", &body)
}
```

### `src/pipeline_worker.rs` — полный текст

```rust
//! 23.12-W. Pipeline worker: decode → filter → encode для группы chunks.
//!
//! ## Роль
//!
//! Main thread делает demux, разбивает chunks по GOP'ам, распределяет
//! между N воркерами. Каждый воркер обрабатывает свой bucket
//! независимо (свой decoder, свой encoder, свой OffscreenCanvas).
//!
//! ## Ключевой fix
//!
//! Encoder принудительно **Baseline Profile** (`avc1.42E01E`) — он
//! не поддерживает B-frames, для которых нужен `ctts`-box в MP4
//! (PTS ≠ DTS). У нашего muxer'а ctts нет, поэтому High/Main
//! привели бы к дёрганью видео.

use std::cell::RefCell;
use std::collections::{HashMap, VecDeque};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Function, Object, Promise, Reflect, Uint8Array};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{ImageBitmap, OffscreenCanvas, OffscreenCanvasRenderingContext2d};

use crate::video_filters::{apply_filter, FILTER_GRAYSCALE, FILTER_INVERT, FILTER_SEPIA};
use crate::video_frame::{format_str, get_fn, invoke, num, set_prop};

const QUEUE_HIGH_WATER: u32 = 8;
const YIELD_AFTER_MS: f64 = 8.0;
const PROGRESS_INTERVAL_MS: f64 = 120.0;
/// Baseline Profile 3.1 — без B-frames.
const ENCODER_CODEC: &str = "avc1.42E01E";

struct WorkerSharedState { session: u64 }
impl WorkerSharedState { fn new() -> Self { Self { session: 0 } } }

#[wasm_bindgen]
pub fn worker_run() -> Result<(), JsValue> {
    let global = js_sys::global();
    web_sys::console::log_1(&JsValue::from_str("[pipeline-worker] registering onmessage"));

    let state = Rc::new(RefCell::new(WorkerSharedState::new()));
    let onmessage: Closure<dyn FnMut(JsValue)> = Closure::new({
        let state = state.clone();
        move |event: JsValue| {
            let data = match Reflect::get(&event, &JsValue::from_str("data")) {
                Ok(v) => v, Err(_) => return,
            };
            if data.is_undefined() || data.is_null() { return; }
            if get_str(&data, "type") == "run" {
                handle_run(data, state.clone());
            }
        }
    });
    Reflect::set(&global, &JsValue::from_str("onmessage"), onmessage.as_ref())?;
    onmessage.forget();
    Ok(())
}

fn handle_run(data: JsValue, state: Rc<RefCell<WorkerSharedState>>) {
    let session = get_f64(&data, "session") as u64;
    state.borrow_mut().session = session;
    spawn_local(async move {
        let result = run_bucket(data, session, state.clone()).await;
        if let Err(e) = result {
            let s = e.as_string().unwrap_or_else(|| format_str(format_args!("{e:?}")));
            if !s.is_empty() && s != "cancelled" { post_error(&s, session); }
        }
    });
}

// ----- Post messages -----

fn post_message(msg: &JsValue) {
    let global = js_sys::global();
    let Ok(post) = Reflect::get(&global, &JsValue::from_str("postMessage")) else { return; };
    let Ok(post) = post.dyn_into::<Function>() else { return; };
    let _ = post.call1(&global, msg);
}

#[allow(clippy::too_many_arguments)]
fn post_progress(
    session: u64, frames: u64, total: u64, fps: f64, ms_per_frame: f64,
    filter_ms_avg: f64, encode_ms_avg: f64, gpu_path: bool,
) {
    let msg = Object::new();
    let _ = set_prop(&msg, "type", &JsValue::from_str("progress"));
    let _ = set_prop(&msg, "session", &JsValue::from_f64(session as f64));
    let _ = set_prop(&msg, "frames", &JsValue::from_f64(frames as f64));
    let _ = set_prop(&msg, "total", &JsValue::from_f64(total as f64));
    let _ = set_prop(&msg, "fps", &JsValue::from_f64(fps));
    let _ = set_prop(&msg, "ms_per_frame", &JsValue::from_f64(ms_per_frame));
    let _ = set_prop(&msg, "filter_ms_avg", &JsValue::from_f64(filter_ms_avg));
    let _ = set_prop(&msg, "encode_ms_avg", &JsValue::from_f64(encode_ms_avg));
    let _ = set_prop(&msg, "gpu_path", &JsValue::from_bool(gpu_path));
    post_message(&msg);
}

fn post_error(message: &str, session: u64) {
    let msg = Object::new();
    let _ = set_prop(&msg, "type", &JsValue::from_str("error"));
    let _ = set_prop(&msg, "session", &JsValue::from_f64(session as f64));
    let _ = set_prop(&msg, "message", &JsValue::from_str(message));
    post_message(&msg);
}

fn post_chunks(session: u64, chunks: &[EncodedOut], avcc: &[u8], codec: &str) {
    let arr = Array::new();
    let transfer = Array::new();

    for c in chunks {
        let u8 = Uint8Array::from(c.data.as_slice());
        let buffer = u8.buffer();
        let obj = Object::new();
        let _ = set_prop(&obj, "is_key", &JsValue::from_bool(c.is_key));
        let _ = set_prop(&obj, "ts", &JsValue::from_f64(c.timestamp_us as f64));
        let _ = set_prop(&obj, "dur", &JsValue::from_f64(c.duration_us as f64));
        let _ = set_prop(&obj, "data", buffer.as_ref());
        arr.push(&obj);
        transfer.push(buffer.as_ref());
    }

    let avcc_u8 = Uint8Array::from(avcc);
    let avcc_buffer = avcc_u8.buffer();

    let msg = Object::new();
    let _ = set_prop(&msg, "type", &JsValue::from_str("chunks"));
    let _ = set_prop(&msg, "session", &JsValue::from_f64(session as f64));
    let _ = set_prop(&msg, "chunks", &arr);
    let _ = set_prop(&msg, "avcc", avcc_buffer.as_ref());
    let _ = set_prop(&msg, "codec", &JsValue::from_str(codec));
    transfer.push(avcc_buffer.as_ref());

    let global = js_sys::global();
    let Ok(post) = Reflect::get(&global, &JsValue::from_str("postMessage")) else { return; };
    let Ok(post) = post.dyn_into::<Function>() else { return; };
    let _ = post.call2(&global, &msg, &transfer);
}

// ----- Хелперы -----

fn get_str(target: &JsValue, key: &str) -> String {
    Reflect::get(target, &JsValue::from_str(key))
        .ok().and_then(|v| v.as_string()).unwrap_or_default()
}
fn get_f64(target: &JsValue, key: &str) -> f64 {
    Reflect::get(target, &JsValue::from_str(key))
        .ok().and_then(|v| v.as_f64()).unwrap_or(0.0)
}
fn vf_timestamp(frame: &JsValue) -> f64 { get_f64(frame, "timestamp") }
fn vf_duration(frame: &JsValue) -> f64 { get_f64(frame, "duration") }
fn read_u32(obj: &JsValue, name: &str) -> u32 { get_f64(obj, name) as u32 }

fn close_video_frame(frame: &JsValue) {
    if frame.is_undefined() || frame.is_null() { return; }
    if let Ok(f) = get_fn(frame, "close") { let _ = f.call0(frame); }
}
fn close_all_frames(queue: &Rc<RefCell<VecDeque<JsValue>>>) {
    let mut q = queue.borrow_mut();
    while let Some(f) = q.pop_front() { close_video_frame(&f); }
}
fn try_call(obj: &JsValue, method: &str) {
    if let Ok(f) = get_fn(obj, method) { let _ = f.call0(obj); }
}

async fn wait_for_queue(
    codec_obj: &JsValue, queue_prop: &str,
    state: &Rc<RefCell<WorkerSharedState>>, session: u64,
) -> Result<(), JsValue> {
    loop {
        if state.borrow().session != session {
            return Err(JsValue::from_str("cancelled"));
        }
        if read_u32(codec_obj, queue_prop) < QUEUE_HIGH_WATER { return Ok(()); }
        TimeoutFuture::new(0).await;
    }
}

// ----- Chunks -----

#[derive(Clone)]
struct EncodedIn {
    is_key: bool, timestamp_us: u64, duration_us: u32, data: Vec<u8>,
}
#[derive(Clone)]
struct EncodedOut {
    is_key: bool, timestamp_us: u64, duration_us: u32, data: Vec<u8>,
}

// ----- WebCodecs -----

fn make_encoded_chunk(c: &EncodedIn) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("EncodedVideoChunk"))?
        .dyn_into::<Function>()?;
    let init = Object::new();
    set_prop(&init, "type", &JsValue::from_str(if c.is_key { "key" } else { "delta" }))?;
    set_prop(&init, "timestamp", &num(c.timestamp_us as f64))?;
    set_prop(&init, "duration", &num(c.duration_us as f64))?;
    let u8 = Uint8Array::from(c.data.as_slice());
    let u8_js: JsValue = u8.into();
    set_prop(&init, "data", &u8_js)?;
    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn make_video_frame_from_bitmap(
    bitmap: &ImageBitmap, timestamp_us: f64, duration_us: f64,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoFrame"))?
        .dyn_into::<Function>()?;
    let init = Object::new();
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    if duration_us > 0.0 { set_prop(&init, "duration", &num(duration_us))?; }
    let args = Array::new();
    args.push(bitmap.as_ref());
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn make_video_frame_from_rgba(
    rgba: &[u8], width: u32, height: u32, timestamp_us: f64, duration_us: f64,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoFrame"))?
        .dyn_into::<Function>()?;
    let u8 = Uint8Array::from(rgba);
    let u8_js: JsValue = u8.into();
    let init = Object::new();
    set_prop(&init, "format", &JsValue::from_str("RGBA"))?;
    set_prop(&init, "codedWidth", &num(width as f64))?;
    set_prop(&init, "codedHeight", &num(height as f64))?;
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    if duration_us > 0.0 { set_prop(&init, "duration", &num(duration_us))?; }
    let args = Array::new();
    args.push(&u8_js);
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn make_decoder_config(codec: &str, description: &[u8]) -> Result<JsValue, JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    if !description.is_empty() {
        let u8 = Uint8Array::from(description);
        let u8_js: JsValue = u8.into();
        set_prop(&cfg, "description", &u8_js)?;
    }
    Ok(cfg.into())
}

fn make_encoder_config(
    codec: &str, width: u32, height: u32, bitrate: u32,
    framerate: u32, latency_mode: &str,
) -> Result<JsValue, JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "width", &num(width as f64))?;
    set_prop(&cfg, "height", &num(height as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    set_prop(&cfg, "framerate", &num(framerate as f64))?;
    if !latency_mode.is_empty() {
        set_prop(&cfg, "latencyMode", &JsValue::from_str(latency_mode))?;
    }
    let avc = Object::new();
    set_prop(&avc, "format", &JsValue::from_str("avc"))?;
    set_prop(&cfg, "avc", &avc)?;
    Ok(cfg.into())
}

// ----- Filter paths -----

async fn process_frame_gpu(
    frame: &JsValue, canvas: &OffscreenCanvas,
    ctx: &OffscreenCanvasRenderingContext2d,
    css_filter: &str, width: u32, height: u32,
) -> Result<JsValue, JsValue> {
    let ts = vf_timestamp(frame);
    let dur = vf_duration(frame);
    set_prop(ctx.as_ref(), "filter", &JsValue::from_str(css_filter))?;
    invoke(ctx.as_ref(), "drawImage", &[
        frame.clone(), num(0.0), num(0.0),
        num(width as f64), num(height as f64),
    ])?;
    let bitmap = canvas.transfer_to_image_bitmap()?;
    let new_frame = make_video_frame_from_bitmap(&bitmap, ts, dur)?;
    let _ = bitmap.close();
    Ok(new_frame)
}

async fn process_frame_cpu(
    frame: &JsValue, width: u32, height: u32, filter_id: u32,
) -> Result<JsValue, JsValue> {
    let ts = vf_timestamp(frame);
    let dur = vf_duration(frame);
    let bytes_len = (width as usize) * (height as usize) * 4;
    let u8 = Uint8Array::new_with_length(bytes_len as u32);
    let opts = Object::new();
    set_prop(&opts, "format", &JsValue::from_str("RGBA"))?;
    let copy_fn = get_fn(frame, "copyTo")?;
    let copy_ret = copy_fn.call2(frame, u8.as_ref(), opts.as_ref())?;
    if let Ok(promise) = copy_ret.clone().dyn_into::<Promise>() {
        let _ = wasm_bindgen_futures::JsFuture::from(promise).await;
    }
    let rgba = u8.to_vec();
    let filtered = apply_filter(&rgba, width, height, filter_id);
    make_video_frame_from_rgba(&filtered, width, height, ts, dur)
}

// ----- Bucket processing -----

async fn run_bucket(
    data: JsValue, session: u64, state: Rc<RefCell<WorkerSharedState>>,
) -> Result<(), JsValue> {
    let codec_string = get_str(&data, "codec_string");
    let width = get_f64(&data, "width") as u32;
    let height = get_f64(&data, "height") as u32;
    let latency_mode = get_str(&data, "latency_mode");
    let filter_id = get_f64(&data, "filter_id") as u32;
    let bitrate = get_f64(&data, "bitrate") as u32;
    let bitrate = if bitrate < 100_000 { 2_000_000 } else { bitrate };

    let avcc_js = Reflect::get(&data, &JsValue::from_str("avcc"))?;
    let source_avcc: Vec<u8> = if let Ok(ab) = avcc_js.dyn_into::<js_sys::ArrayBuffer>() {
        Uint8Array::new(&ab).to_vec()
    } else { Vec::new() };

    let chunks_js = Reflect::get(&data, &JsValue::from_str("chunks"))?;
    let chunks_arr: Array = chunks_js.dyn_into()?;
    let n = chunks_arr.length();

    let mut input_chunks: Vec<EncodedIn> = Vec::with_capacity(n as usize);
    for i in 0..n {
        let obj = chunks_arr.get(i);
        let is_key = Reflect::get(&obj, &JsValue::from_str("is_key"))
            .ok().and_then(|v| v.as_bool()).unwrap_or(false);
        let ts = get_f64(&obj, "ts") as u64;
        let dur = get_f64(&obj, "dur") as u32;
        let data_js = Reflect::get(&obj, &JsValue::from_str("data"))?;
        let bytes: Vec<u8> = if let Ok(ab) = data_js.dyn_into::<js_sys::ArrayBuffer>() {
            Uint8Array::new(&ab).to_vec()
        } else { Vec::new() };
        input_chunks.push(EncodedIn {
            is_key, timestamp_us: ts, duration_us: dur, data: bytes,
        });
    }
    let total = input_chunks.len() as u64;

    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[pipeline-worker] bucket: {} chunks, {}x{}, codec={}, latency={}, bitrate={}",
        total, width, height, codec_string, latency_mode, bitrate,
    )));

    if input_chunks.is_empty() {
        post_chunks(session, &[], &source_avcc, &codec_string);
        return Ok(());
    }

    // Сохраняем duration и is_key из входных chunks. Chrome VideoEncoder
    // НЕ всегда копирует duration на выходные chunks.
    let mut duration_map: HashMap<u64, u32> = HashMap::with_capacity(input_chunks.len());
    for c in &input_chunks { duration_map.insert(c.timestamp_us, c.duration_us); }
    let duration_map = Rc::new(duration_map);
    let key_flags: Rc<Vec<bool>> = Rc::new(input_chunks.iter().map(|c| c.is_key).collect());

    // OffscreenCanvas.
    let canvas = OffscreenCanvas::new(width, height)?;
    let ctx_obj = canvas.get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d on OffscreenCanvas"))?;
    let ctx: OffscreenCanvasRenderingContext2d = ctx_obj.dyn_into()?;

    let gpu_path = {
        let _ = set_prop(ctx.as_ref(), "filter", &JsValue::from_str("grayscale(100%)"));
        let back = Reflect::get(ctx.as_ref(), &JsValue::from_str("filter"))
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        back.contains("grayscale")
    };

    let css_filter = match filter_id {
        FILTER_GRAYSCALE => "grayscale(100%)",
        FILTER_INVERT => "invert(100%)",
        FILTER_SEPIA => "sepia(100%)",
        _ => "none",
    };

    // Decoder.
    let pending: Rc<RefCell<VecDeque<JsValue>>> = Rc::new(RefCell::new(VecDeque::new()));
    let decode_err: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));

    let pending_cb = pending.clone();
    let dec_output_cb: Closure<dyn FnMut(JsValue)> = Closure::new(move |frame: JsValue| {
        pending_cb.borrow_mut().push_back(frame);
    });
    let decode_err_cb = decode_err.clone();
    let dec_err_cb: Closure<dyn FnMut(JsValue)> = Closure::new(move |e: JsValue| {
        let msg = Reflect::get(&e, &JsValue::from_str("message"))
            .ok().and_then(|v| v.as_string())
            .unwrap_or_else(|| String::from("unknown decoder error"));
        *decode_err_cb.borrow_mut() = Some(msg);
    });

    let global = js_sys::global();
    let dec_ctor = Reflect::get(&global, &JsValue::from_str("VideoDecoder"))?;
    if !dec_ctor.is_function() { return Err(JsValue::from_str("VideoDecoder недоступен")); }
    let dec_ctor = dec_ctor.dyn_into::<Function>()?;
    let dec_init = Object::new();
    set_prop(&dec_init, "output", dec_output_cb.as_ref())?;
    set_prop(&dec_init, "error", dec_err_cb.as_ref())?;
    let dec_args = Array::new();
    dec_args.push(&dec_init);
    let decoder = Reflect::construct(&dec_ctor, &dec_args)?;

    let dec_cfg = make_decoder_config(&codec_string, &source_avcc)?;
    get_fn(&decoder, "configure")?.call1(&decoder, &dec_cfg)?;
    let dec_decode = get_fn(&decoder, "decode")?;
    let dec_flush = get_fn(&decoder, "flush")?;

    // Encoder.
    let enc_output: Rc<RefCell<Vec<EncodedOut>>> = Rc::new(RefCell::new(Vec::new()));
    let enc_err: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
    let enc_avcc: Rc<RefCell<Option<Vec<u8>>>> = Rc::new(RefCell::new(None));
    let enc_codec: Rc<RefCell<String>> = Rc::new(RefCell::new(String::new()));

    let enc_output_cb = enc_output.clone();
    let duration_map_cb = duration_map.clone();
    let enc_avcc_cb = enc_avcc.clone();
    let enc_codec_cb = enc_codec.clone();
    let output_cb: Closure<dyn FnMut(JsValue, JsValue)> = Closure::new(move |chunk: JsValue, meta: JsValue| {
        if enc_avcc_cb.borrow().is_none() {
            if let Ok(cfg) = Reflect::get(&meta, &JsValue::from_str("decoderConfig")) {
                if !cfg.is_undefined() && !cfg.is_null() {
                    if let Ok(codec_val) = Reflect::get(&cfg, &JsValue::from_str("codec")) {
                        if let Some(s) = codec_val.as_string() {
                            *enc_codec_cb.borrow_mut() = s;
                        }
                    }
                    if let Ok(desc) = Reflect::get(&cfg, &JsValue::from_str("description")) {
                        if !desc.is_undefined() && !desc.is_null() {
                            let bytes: Vec<u8> = if let Ok(u8v) = desc.clone().dyn_into::<Uint8Array>() {
                                u8v.to_vec()
                            } else if let Ok(ab) = desc.clone().dyn_into::<js_sys::ArrayBuffer>() {
                                Uint8Array::new(&ab).to_vec()
                            } else { Vec::new() };
                            if !bytes.is_empty() { *enc_avcc_cb.borrow_mut() = Some(bytes); }
                        }
                    }
                }
            }
        }

        let is_key = Reflect::get(&chunk, &JsValue::from_str("type"))
            .ok().and_then(|v| v.as_string()).map(|s| s == "key").unwrap_or(false);
        let ts = get_f64(&chunk, "timestamp") as u64;
        let dur = duration_map_cb.get(&ts).copied()
            .filter(|&d| d > 0)
            .or_else(|| {
                let d = get_f64(&chunk, "duration") as u32;
                if d > 0 { Some(d) } else { None }
            })
            .unwrap_or(33_333);

        let bl = get_f64(&chunk, "byteLength") as usize;
        let mut data: Vec<u8> = Vec::new();
        if let Ok(copy_to) = Reflect::get(&chunk, &JsValue::from_str("copyTo")) {
            if let Ok(copy_fn) = copy_to.dyn_into::<Function>() {
                let u8_buf = Uint8Array::new_with_length(bl as u32);
                if copy_fn.call1(&chunk, u8_buf.as_ref()).is_ok() {
                    data = u8_buf.to_vec();
                }
            }
        }
        enc_output_cb.borrow_mut().push(EncodedOut {
            is_key, timestamp_us: ts, duration_us: dur, data,
        });
    });
    let enc_err_cb_out = enc_err.clone();
    let err_cb: Closure<dyn FnMut(JsValue)> = Closure::new(move |e: JsValue| {
        let msg = Reflect::get(&e, &JsValue::from_str("message"))
            .ok().and_then(|v| v.as_string())
            .unwrap_or_else(|| String::from("unknown encoder error"));
        *enc_err_cb_out.borrow_mut() = Some(msg);
    });

    let enc_ctor = Reflect::get(&global, &JsValue::from_str("VideoEncoder"))?;
    if !enc_ctor.is_function() {
        try_call(&decoder, "close");
        return Err(JsValue::from_str("VideoEncoder недоступен"));
    }
    let enc_ctor = enc_ctor.dyn_into::<Function>()?;
    let enc_init = Object::new();
    set_prop(&enc_init, "output", output_cb.as_ref())?;
    set_prop(&enc_init, "error", err_cb.as_ref())?;
    let enc_args = Array::new();
    enc_args.push(&enc_init);
    let encoder = Reflect::construct(&enc_ctor, &enc_args)?;

    // **КЛЮЧЕВОЙ FIX**: Baseline Profile.
    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[pipeline-worker] encoder forced to {} (Baseline, no B-frames)",
        ENCODER_CODEC,
    )));

    let framerate = 30u32;
    let enc_cfg = make_encoder_config(
        ENCODER_CODEC, width, height, bitrate, framerate, &latency_mode,
    )?;
    get_fn(&encoder, "configure")?.call1(&encoder, &enc_cfg)?;
    let enc_encode = get_fn(&encoder, "encode")?;
    let enc_flush = get_fn(&encoder, "flush")?;

    // Feed + drain loop.
    let t_start = js_sys::Date::now();
    let mut frames_processed: u64 = 0;
    let mut filter_ms_accum: f64 = 0.0;
    let mut encode_ms_accum: f64 = 0.0;
    let mut last_progress_ms = t_start;
    let mut last_yield_ms = t_start;

    macro_rules! drain_one {
        ($frame:expr) => {{
            let frame: JsValue = $frame;
            let t_filter_start = js_sys::Date::now();
            let new_frame = if gpu_path {
                match process_frame_gpu(frame.as_ref(), &canvas, &ctx, css_filter, width, height).await {
                    Ok(f) => f,
                    Err(_) => {
                        close_video_frame(&frame);
                        process_frame_cpu(frame.as_ref(), width, height, filter_id).await?
                    }
                }
            } else {
                process_frame_cpu(frame.as_ref(), width, height, filter_id).await?
            };
            close_video_frame(&frame);
            let t_filter_end = js_sys::Date::now();

            if wait_for_queue(&encoder, "encodeQueueSize", &state, session).await.is_err() {
                close_video_frame(&new_frame);
                return Err(JsValue::from_str("cancelled"));
            }

            let t_encode_start = js_sys::Date::now();
            let is_key = key_flags.get(frames_processed as usize)
                .copied().unwrap_or(frames_processed == 0);
            let opts = Object::new();
            set_prop(&opts, "keyFrame", &JsValue::from_bool(is_key))?;
            enc_encode.call2(&encoder, &new_frame, &opts)?;
            close_video_frame(&new_frame);
            let t_encode_end = js_sys::Date::now();

            filter_ms_accum += t_filter_end - t_filter_start;
            encode_ms_accum += t_encode_end - t_encode_start;
            frames_processed += 1;

            let now = js_sys::Date::now();
            if now - last_progress_ms >= PROGRESS_INTERVAL_MS {
                last_progress_ms = now;
                let elapsed_ms = now - t_start;
                let fps = if elapsed_ms > 0.0 { frames_processed as f64 * 1000.0 / elapsed_ms } else { 0.0 };
                let ms_per_frame = if frames_processed > 0 { elapsed_ms / frames_processed as f64 } else { 0.0 };
                let nf = frames_processed as f64;
                post_progress(session, frames_processed, total, fps, ms_per_frame,
                    if nf > 0.0 { filter_ms_accum / nf } else { 0.0 },
                    if nf > 0.0 { encode_ms_accum / nf } else { 0.0 },
                    gpu_path);
            }

            if now - last_yield_ms >= YIELD_AFTER_MS {
                last_yield_ms = now;
                TimeoutFuture::new(0).await;
            }
        }};
    }

    for input in input_chunks.iter() {
        if state.borrow().session != session {
            close_all_frames(&pending);
            try_call(&decoder, "close");
            try_call(&encoder, "close");
            return Ok(());
        }
        if let Some(msg) = decode_err.borrow().as_ref() {
            close_all_frames(&pending);
            try_call(&decoder, "close");
            try_call(&encoder, "close");
            return Err(JsValue::from_str(&format_str(format_args!("decode: {msg}"))));
        }
        if wait_for_queue(&decoder, "decodeQueueSize", &state, session).await.is_err() {
            close_all_frames(&pending);
            try_call(&decoder, "close");
            try_call(&encoder, "close");
            return Ok(());
        }
        let chunk_js = make_encoded_chunk(input)?;
        if let Err(e) = dec_decode.call1(&decoder, &chunk_js) {
            close_all_frames(&pending);
            try_call(&decoder, "close");
            try_call(&encoder, "close");
            return Err(JsValue::from_str(&format_str(format_args!("decode: {e:?}"))));
        }
        loop {
            let frame = match pending.borrow_mut().pop_front() {
                Some(f) => f, None => break,
            };
            if state.borrow().session != session {
                close_video_frame(&frame);
                close_all_frames(&pending);
                try_call(&decoder, "close");
                try_call(&encoder, "close");
                return Ok(());
            }
            drain_one!(frame);
        }
    }

    // Final flush decoder.
    let flush_p = dec_flush.call0(&decoder)?.dyn_into::<Promise>()?;
    let _ = wasm_bindgen_futures::JsFuture::from(flush_p).await;

    if let Some(msg) = decode_err.borrow().as_ref() {
        close_all_frames(&pending);
        try_call(&decoder, "close");
        try_call(&encoder, "close");
        return Err(JsValue::from_str(&format_str(format_args!("decode: {msg}"))));
    }

    loop {
        let frame = match pending.borrow_mut().pop_front() {
            Some(f) => f, None => break,
        };
        if state.borrow().session != session {
            close_video_frame(&frame);
            close_all_frames(&pending);
            try_call(&decoder, "close");
            try_call(&encoder, "close");
            return Ok(());
        }
        drain_one!(frame);
    }

    let enc_flush_p = enc_flush.call0(&encoder)?.dyn_into::<Promise>()?;
    wasm_bindgen_futures::JsFuture::from(enc_flush_p).await?;

    try_call(&decoder, "close");
    try_call(&encoder, "close");

    if let Some(msg) = enc_err.borrow().as_ref() {
        return Err(JsValue::from_str(&format_str(format_args!("encode: {msg}"))));
    }

    let out: Vec<EncodedOut> = enc_output.borrow().clone();
    let final_avcc = enc_avcc.borrow().clone().unwrap_or(source_avcc);
    let final_codec = {
        let c = enc_codec.borrow().clone();
        if c.is_empty() { codec_string.clone() } else { c }
    };

    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[pipeline-worker] bucket done: {} chunks, avcc={} B, codec={}, time={} ms",
        out.len(), final_avcc.len(), final_codec,
        (js_sys::Date::now() - t_start) as u64,
    )));
    post_chunks(session, &out, &final_avcc, &final_codec);
    Ok(())
}
```

### `src/app.rs` — полный текст

(Полный текст app.rs — прилагается отдельно в приложении к главе. Он ~1100 строк. Ключевые компоненты см. ниже.)

Ключевые части:

**Структура state:**

```rust
struct DemoState {
    session: u64,
    stage: Stage,
    local_progress: f64,
    frames_processed: u64,
    total_frames: u64,
    fps: f64,
    ms_per_frame: f64,
    filter_ms_avg: f64,
    encode_ms_avg: f64,
    gpu_path: bool,
    chart_fps: VecDeque<f64>,        // FPS-график
    chart_last_push_ms: f64,
    pipeline_start_ms: f64,
    total_elapsed_ms: f64,           // фиксируется в момент done
    pool: Vec<WorkerHandle>,
    num_workers: usize,
    latency_mode: String,
    workers_dispatched: usize,
    worker_results_count: usize,
    worker_errors: u32,
    source_bytes: Option<Vec<u8>>,
    source_info: Option<SourceInfo>,
    // ...
}
```

**Разбивка по GOP'ам:**

```rust
fn split_by_gops(samples: &[VideoSample]) -> Vec<Vec<usize>> {
    let mut gops: Vec<Vec<usize>> = Vec::new();
    let mut current: Vec<usize> = Vec::new();
    for (i, s) in samples.iter().enumerate() {
        let is_key = i == 0 || s.is_key;
        if is_key && !current.is_empty() {
            gops.push(std::mem::take(&mut current));
        }
        current.push(i);
    }
    if !current.is_empty() { gops.push(current); }
    gops
}

fn distribute_gops(gops: Vec<Vec<usize>>, n_workers: usize) -> Vec<Vec<usize>> {
    let mut buckets: Vec<Vec<usize>> = vec![Vec::new(); n_workers];
    for (i, gop) in gops.into_iter().enumerate() {
        buckets[i % n_workers].extend(gop);
    }
    buckets
}
```

**Mux с encoder avcC:**

```rust
let mux_result = build_mp4_with_audio(
    &all_chunks,
    &MuxConfig {
        width: info.width,
        height: info.height,
        avcc: enc_avcc.clone(),        // ← avcC от encoder'а, не source
    },
    info.audio_input.as_ref(),
);
```

**График FPS:** рисуется в `render_chart()` — canvas 1000×80, хранит последние 240 точек FPS (раз в 200 мс), с автомасштабом и подписью текущего значения.

**Полный текст app.rs**

```
//! Глава 23.12 — Практический проект: MP4 → обработка → MP4.
//!
//! ## Архитектура
//!
//! Main thread: demux → разбивка chunks по GOP'ам → распределение
//! по N воркерам → сбор → mux → Blob URL.
//!
//! ## Ключевой fix дёрганья
//!
//! 1. Mux использует **avcC (SPS/PPS) от encoder'а**, не от source-файла.
//! 2. Encoder принудительно **Baseline Profile** — без B-frames.
//!    B-frames требуют ctts-box в MP4 (PTS ≠ DTS), которого у нас нет.

use std::cell::RefCell;
use std::collections::VecDeque;
use std::rc::Rc;

use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    Blob, BlobPropertyBag, Document, HtmlCanvasElement, HtmlElement,
    HtmlInputElement, HtmlVideoElement, MessageEvent, Worker, WorkerOptions,
    WorkerType,
};

use crate::container_parser::{find_avcc_in_stsd, parse_mp4, BoxInfo};
use crate::mp4_demuxer::{
    avc_codec_string, build_samples, extract_audio_track, parse_mdhd_timescale,
    parse_sample_table, VideoSample,
};
use crate::mp4_muxer::{
    build_mp4_with_audio, AudioChunk, AudioTrackInput, MuxConfig, VideoChunk,
};
use crate::video_filters::{
    FILTER_GRAYSCALE, FILTER_INVERT, FILTER_NONE, FILTER_SEPIA,
};
use crate::video_frame::{format_str, set_prop};

// =============================================================================
// Константы
// =============================================================================

const WORKER_URL: &str = "/workers/mp4_worker.js";
const MAX_NAME_CHARS: usize = 50;
const CHART_SAMPLES: usize = 240;
const DEFAULT_NUM_WORKERS: usize = 4;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.pp-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.pp-desc-line { margin:6px 0; }
.pp-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.pp-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.pp-btn:hover:not([disabled]) { background:#30363d; }
.pp-btn:disabled { opacity:0.5;cursor:not-allowed; }
.pp-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.pp-btn-primary:hover:not([disabled]) { background:#2ea043; }
.pp-btn-danger { background:#a4302e;border-color:#d1242f;color:#fff;}
.pp-btn-danger:hover:not([disabled]) { background:#d1242f; }
.pp-btn-secondary { background:#8957e5;border-color:#a371f7;color:#fff;}
.pp-btn-secondary:hover:not([disabled]) { background:#a371f7; }
.pp-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.pp-check { display:inline-flex;align-items:center;gap:6px;font-family:ui-monospace,monospace;font-size:12px;color:#c9d1d9;cursor:pointer;padding:4px 6px;border-radius:4px;}
.pp-check:hover { background:#21262d; }
.pp-check input { cursor:pointer; }
.pp-select { font-size:12px;padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#0d1117;color:#c9d1d9;font-family:ui-monospace,monospace;}
.pp-progress { max-width:1000px;margin-bottom:12px;}
.pp-progress-bar { height:10px;background:#21262d;border-radius:5px;overflow:hidden;}
.pp-progress-fill { height:100%;background:#3fb950;transition:width 0.15s linear;width:0%;}
.pp-progress-label { margin-top:4px;font-family:ui-monospace,monospace;font-size:11px;color:#8b949e;display:flex;justify-content:space-between;gap:12px;}
.pp-progress-label .stage-name { color:#79c0ff; font-weight:600; }
.pp-progress-label .stage-stats { color:#8b949e; }
.pp-speed { max-width:1000px;margin-bottom:12px;padding:8px 14px;background:#0d1117;border-left:3px solid #d2a8ff;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.6;}
.pp-speed .big { color:#79c0ff; font-weight:600; font-size:14px; }
.pp-speed .ok { color:#3fb950; }
.pp-speed .warn { color:#f0883e; }
.pp-speed .muted { color:#6e7681; }
.pp-chart-wrap { max-width:1000px;margin-bottom:12px;padding:8px;background:#0d1117;border:1px solid #21262d;border-radius:4px;}
.pp-chart-title { font-family:ui-monospace,monospace;font-size:11px;color:#8b949e;margin-bottom:6px;display:flex;justify-content:space-between;}
.pp-chart { display:block;width:100%;height:80px;background:#0d1117;}
.pp-pair { max-width:1000px;display:flex;gap:16px;flex-wrap:wrap;margin-bottom:16px;}
.pp-slot { flex:1 1 380px;min-width:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:10px 12px;}
.pp-slot-title { color:#8b949e;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.pp-slot-title-res { color:#a371f7; }
.pp-video { display:block;background:#000;border-radius:6px;max-width:100%;width:100%;aspect-ratio:16/9;object-fit:contain;}
.pp-summary { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;overflow-wrap:anywhere;}
.pp-summary .row { display:flex;gap:12px; }
.pp-summary .lbl { color:#8b949e;min-width:150px;flex-shrink:0; }
.pp-summary .val { color:#79c0ff; }
.pp-summary .ok { color:#3fb950; }
.pp-summary .warn { color:#f0883e; }
.pp-summary .err { color:#f85149; }
.pp-summary .muted { color:#6e7681; }
.pp-summary .sep { border-top:1px solid #21262d;margin:6px 0; }
.pp-stats { max-width:1000px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;overflow-wrap:anywhere;word-break:break-word;}
.pp-stats .lbl { color:#8b949e; }
.pp-stats .val { color:#79c0ff; }
.pp-stats .err { color:#f85149; }
.pp-stats .muted { color:#6e7681; }
.pp-stats .ok { color:#3fb950; }
.pp-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Stage
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum Stage { Idle, Demux, Pipeline, Mux, Done, Cancelled }

impl Stage {
    fn label(&self) -> &'static str {
        match self {
            Stage::Idle => "простой",
            Stage::Demux => "demux (main)",
            Stage::Pipeline => "pipeline (N воркеров параллельно)",
            Stage::Mux => "mux (main)",
            Stage::Done => "готово",
            Stage::Cancelled => "отменено",
        }
    }
    fn range(&self) -> (f64, f64) {
        match self {
            Stage::Idle | Stage::Cancelled => (0.0, 0.0),
            Stage::Demux => (0.0, 0.05),
            Stage::Pipeline => (0.05, 0.95),
            Stage::Mux => (0.95, 1.00),
            Stage::Done => (1.0, 1.0),
        }
    }
    fn interpolate(&self, local: f64) -> f64 {
        let (a, b) = self.range();
        let l = local.max(0.0).min(1.0);
        return a + (b - a) * l;
    }
}

// =============================================================================
// WorkerHandle
// =============================================================================

struct WorkerHandle {
    worker: Worker,
    ready: bool,
    boot_error: Option<String>,
    result: Option<Vec<VideoChunk>>,
    result_avcc: Option<Vec<u8>>,
    result_codec: String,
    frames_done: u64,
    frames_total: u64,
    fps: f64,
    ms_per_frame: f64,
    filter_ms_avg: f64,
    encode_ms_avg: f64,
    gpu_path: bool,
    _on_message: Option<Closure<dyn FnMut(MessageEvent)>>,
    _on_error: Option<Closure<dyn FnMut(JsValue)>>,
}

impl WorkerHandle {
    fn new(worker: Worker) -> Self {
        return Self {
            worker, ready: false, boot_error: None, result: None,
            result_avcc: None, result_codec: String::new(),
            frames_done: 0, frames_total: 0, fps: 0.0, ms_per_frame: 0.0,
            filter_ms_avg: 0.0, encode_ms_avg: 0.0, gpu_path: true,
            _on_message: None, _on_error: None,
        };
    }
    fn terminate(&self) {
        self.worker.set_onmessage(None);
        self.worker.set_onerror(None);
        let _ = self.worker.terminate();
    }
    fn reset_run_state(&mut self) {
        self.result = None;
        self.result_avcc = None;
        self.result_codec.clear();
        self.frames_done = 0;
        self.frames_total = 0;
        self.fps = 0.0;
        self.ms_per_frame = 0.0;
        self.filter_ms_avg = 0.0;
        self.encode_ms_avg = 0.0;
    }
}

// =============================================================================
// SourceInfo
// =============================================================================

struct SourceInfo {
    width: u32,
    height: u32,
    audio_input: Option<AudioTrackInput>,
    duration_us: u64,
    avg_framerate: u32,
    source_size: usize,
}

// =============================================================================
// DemoState
// =============================================================================

struct DemoState {
    session: u64,
    stage: Stage,
    local_progress: f64,
    frames_processed: u64,
    total_frames: u64,
    fps: f64,
    ms_per_frame: f64,
    filter_ms_avg: f64,
    encode_ms_avg: f64,
    gpu_path: bool,
    chart_fps: VecDeque<f64>,
    chart_last_push_ms: f64,
    pipeline_start_ms: f64,
    total_elapsed_ms: f64,
    pool: Vec<WorkerHandle>,
    num_workers: usize,
    latency_mode: String,
    workers_dispatched: usize,
    worker_results_count: usize,
    worker_errors: u32,
    source_bytes: Option<Vec<u8>>,
    source_info: Option<SourceInfo>,
    source_name: String,
    source_url: Option<String>,
    result_url: Option<String>,
    result_bytes: Vec<u8>,
    summary_html: String,
    ready: bool,
    error: Option<String>,
    filter_grayscale: bool,
    filter_invert: bool,
    filter_sepia: bool,
}

impl DemoState {
    fn new(num_workers: usize) -> Self {
        return Self {
            session: 0,
            stage: Stage::Idle,
            local_progress: 0.0,
            frames_processed: 0,
            total_frames: 0,
            fps: 0.0,
            ms_per_frame: 0.0,
            filter_ms_avg: 0.0,
            encode_ms_avg: 0.0,
            gpu_path: true,
            chart_fps: VecDeque::with_capacity(CHART_SAMPLES),
            chart_last_push_ms: 0.0,
            pipeline_start_ms: 0.0,
            total_elapsed_ms: 0.0,
            pool: Vec::new(),
            num_workers,
            latency_mode: String::from("quality"),
            workers_dispatched: 0,
            worker_results_count: 0,
            worker_errors: 0,
            source_bytes: None,
            source_info: None,
            source_name: String::new(),
            source_url: None,
            result_url: None,
            result_bytes: Vec::new(),
            summary_html: String::new(),
            ready: false,
            error: None,
            filter_grayscale: false,
            filter_invert: false,
            filter_sepia: false,
        };
    }

    fn progress(&self) -> f64 { return self.stage.interpolate(self.local_progress); }

    fn filter_id(&self) -> u32 {
        if self.filter_grayscale { return FILTER_GRAYSCALE; }
        if self.filter_invert { return FILTER_INVERT; }
        if self.filter_sepia { return FILTER_SEPIA; }
        return FILTER_NONE;
    }

    fn eta_sec(&self) -> Option<f64> {
        if self.stage == Stage::Done { return Some(0.0); }
        if self.fps <= 0.0 || self.total_frames == 0 { return None; }
        let remaining = self.total_frames.saturating_sub(self.frames_processed);
        return Some(remaining as f64 / self.fps);
    }

    fn final_fps(&self) -> f64 {
        if self.total_elapsed_ms > 0.0 && self.frames_processed > 0 {
            return self.frames_processed as f64 * 1000.0 / self.total_elapsed_ms;
        }
        return self.fps;
    }
    fn final_ms_per_frame(&self) -> f64 {
        if self.total_elapsed_ms > 0.0 && self.frames_processed > 0 {
            return self.total_elapsed_ms / self.frames_processed as f64;
        }
        return self.ms_per_frame;
    }

    fn push_chart_sample(&mut self, now_ms: f64) {
        if now_ms - self.chart_last_push_ms < 200.0 { return; }
        self.chart_last_push_ms = now_ms;
        if self.chart_fps.len() >= CHART_SAMPLES { self.chart_fps.pop_front(); }
        self.chart_fps.push_back(self.fps);
    }

    fn is_active(&self) -> bool {
        return matches!(self.stage, Stage::Demux | Stage::Pipeline | Stage::Mux);
    }

    fn pool_ready(&self) -> bool {
        if self.pool.is_empty() { return false; }
        return self.pool.iter().all(|w| w.ready);
    }

    fn kill_pool(&mut self) {
        for w in self.pool.iter() { w.terminate(); }
        self.pool.clear();
        self.workers_dispatched = 0;
        self.worker_results_count = 0;
        self.worker_errors = 0;
    }

    fn cancel(&mut self) {
        self.session = self.session.wrapping_add(1);
        self.kill_pool();
        self.stage = Stage::Cancelled;
        self.local_progress = 0.0;
        self.ready = false;
        self.summary_html.clear();
        if let Some(url) = self.result_url.take() { revoke_url(&url); }
        self.result_bytes.clear();
        self.source_info = None;
        self.total_elapsed_ms = 0.0;
        self.pipeline_start_ms = 0.0;
        self.chart_fps.clear();
        self.chart_last_push_ms = 0.0;
    }

    fn aggregate_metrics(&mut self) {
        let mut frames = 0u64;
        let mut total = 0u64;
        let mut filter_acc = 0.0;
        let mut enc_acc = 0.0;
        let mut gpu_any = false;
        let mut n = 0u32;
        for w in self.pool.iter() {
            frames += w.frames_done;
            total += w.frames_total;
            if w.frames_done > 0 {
                filter_acc += w.filter_ms_avg;
                enc_acc += w.encode_ms_avg;
                n += 1;
            }
            if w.gpu_path { gpu_any = true; }
        }
        self.frames_processed = frames;
        self.total_frames = total;
        self.filter_ms_avg = if n > 0 { filter_acc / n as f64 } else { 0.0 };
        self.encode_ms_avg = if n > 0 { enc_acc / n as f64 } else { 0.0 };
        self.gpu_path = gpu_any;

        if !matches!(self.stage, Stage::Done | Stage::Cancelled) {
            let now = js_sys::Date::now();
            if self.pipeline_start_ms > 0.0 {
                let elapsed = now - self.pipeline_start_ms;
                if elapsed > 0.0 && frames > 0 {
                    self.fps = frames as f64 * 1000.0 / elapsed;
                    self.ms_per_frame = elapsed / frames as f64;
                }
            }
        }
        self.local_progress = if total > 0 { frames as f64 / total as f64 } else { 0.0 };
    }
}

// =============================================================================
// Хелперы строк
// =============================================================================

fn row(label: &str, value_html: &str) -> String {
    return format_str(format_args!(
        r#"<div class="row"><span class="lbl">{label}:</span><span class="val">{value}</span></div>"#,
        label = label, value = value_html,
    ));
}

fn truncate_name(name: &str) -> String {
    let chars_count = name.chars().count();
    if chars_count <= MAX_NAME_CHARS { return name.to_string(); }
    let stem_chars: Vec<char> = name.chars().collect();
    let head: String = stem_chars.iter().take(32).collect();
    let tail_start = stem_chars.len().saturating_sub(15);
    let tail: String = stem_chars[tail_start..].iter().collect();
    return format_str(format_args!("{head}…{tail}"));
}

fn fmt_eta(sec: f64) -> String {
    if !sec.is_finite() || sec < 0.0 { return String::from("—"); }
    let s = sec.round() as u64;
    if s < 60 { return format_str(format_args!("{s} с")); }
    let m = s / 60;
    let r = s % 60;
    if m < 60 { return format_str(format_args!("{m}:{r:02}")); }
    let h = m / 60;
    let mr = m % 60;
    return format_str(format_args!("{h}:{mr:02}:{r:02}"));
}

fn fmt_bytes_short(n: usize) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = KB * 1024.0;
    let f = n as f64;
    if f < KB { return format_str(format_args!("{n} B")); }
    if f < MB { return format_str(format_args!("{:.1} KB", f / KB)); }
    return format_str(format_args!("{:.1} MB", f / MB));
}

// =============================================================================
// DOM / JS helpers
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "pp-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() { p.remove_child(&old)?; }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    return Ok(());
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>N воркеров параллельно.</strong> Main thread: demux → разбивка chunks по <em>GOP'ам</em> → распределение по N воркерам → сбор → mux → Blob. Каждый воркер — свой decoder, encoder, OffscreenCanvas.",
        "<strong>Разбивка по GOP'ам.</strong> Delta-chunk зависит от своего key-chunk. Потому chunks делятся на группы, начинающиеся с key. Группы декодируются независимо, параллельно.",
        "<strong>Fast filter.</strong> В каждом воркере: <code>ctx.filter</code> на <code>OffscreenCanvas</code> + <code>transferToImageBitmap</code> + <code>new VideoFrame(bitmap)</code>.",
        "<strong>Baseline Profile для encoder'а.</strong> High/Main могут использовать B-frames — для них нужен <code>ctts</code>-box в MP4 (PTS ≠ DTS). У нас его нет, поэтому <em>принудительно Baseline</em>: без B-frames, PTS = DTS.",
        "<strong>Mux с encoder avcC.</strong> Воркеры перекодируют видео своим encoder'ом — у него свой SPS/PPS. Mux использует <em>его</em>, а не source.",
        "<strong>Audio passthrough.</strong> Байты audio-трека копируются как есть.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("pp-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("pp-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    return Ok(block.unchecked_into());
}

fn revoke_url(url: &str) {
    let Some(window) = web_sys::window() else { return; };
    let Ok(url_api) = Reflect::get(window.as_ref(), &JsValue::from_str("URL")) else { return; };
    let Ok(revoke) = Reflect::get(&url_api, &JsValue::from_str("revokeObjectURL")) else { return; };
    let Ok(revoke) = revoke.dyn_into::<js_sys::Function>() else { return; };
    let _ = revoke.call1(&url_api, &JsValue::from_str(url));
}

fn create_object_url(blob: &Blob) -> Result<String, JsValue> {
    let global = js_sys::global();
    let url_api = Reflect::get(&global, &JsValue::from_str("URL"))?;
    let create = Reflect::get(&url_api, &JsValue::from_str("createObjectURL"))?
        .dyn_into::<js_sys::Function>()?;
    let blob_js: JsValue = blob.into();
    return create.call1(&url_api, &blob_js)?
        .as_string()
        .ok_or_else(|| JsValue::from_str("createObjectURL returned non-string"));
}

fn detach_video(video: &HtmlVideoElement) {
    let _ = set_prop(video.as_ref(), "srcObject", &JsValue::NULL);
    let _ = video.remove_attribute("src");
    let _ = video.load();
}

fn get_f64(target: &JsValue, key: &str) -> f64 {
    return Reflect::get(target, &JsValue::from_str(key))
        .ok().and_then(|v| v.as_f64()).unwrap_or(0.0);
}
fn get_str(target: &JsValue, key: &str) -> String {
    return Reflect::get(target, &JsValue::from_str(key))
        .ok().and_then(|v| v.as_string()).unwrap_or_default();
}
fn get_bool(target: &JsValue, key: &str) -> bool {
    return Reflect::get(target, &JsValue::from_str(key))
        .ok().and_then(|v| v.as_bool()).unwrap_or(false);
}

// =============================================================================
// Разбор MP4
// =============================================================================

struct VideoInfo {
    width: u32,
    height: u32,
    avcc: Vec<u8>,
    codec_string: String,
    samples: Vec<VideoSample>,
    duration_us: u64,
    avg_framerate: u32,
}

fn extract_video_info(boxes: &[BoxInfo]) -> Result<VideoInfo, String> {
    let moov = boxes.iter().find(|b| b.box_type == "moov")
        .ok_or_else(|| String::from("нет moov (fragmented MP4 не поддерживается)"))?;

    let mut video_trak: Option<&BoxInfo> = None;
    for trak in &moov.children {
        if trak.box_type != "trak" { continue; }
        if let Some(hdlr) = trak.find("hdlr") {
            let p = &hdlr.payload;
            if p.len() >= 12 && &p[8..12] == b"vide" {
                video_trak = Some(trak);
                break;
            }
        }
    }
    let video_trak = video_trak.ok_or_else(|| String::from("нет video-трека"))?;

    let mdhd = video_trak.find("mdhd").ok_or_else(|| String::from("нет mdhd"))?;
    let timescale = parse_mdhd_timescale(mdhd)
        .ok_or_else(|| String::from("mdhd.timescale = 0"))?;

    let stbl = video_trak.find("stbl").ok_or_else(|| String::from("нет stbl"))?;
    let table = parse_sample_table(stbl, timescale)?;
    let samples = build_samples(&table)?;

    let stsd = video_trak.find("stsd").ok_or_else(|| String::from("нет stsd"))?;
    let avcc = find_avcc_in_stsd(stsd).ok_or_else(|| String::from("нет avcC"))?;
    let codec_string = avc_codec_string(&avcc).ok_or_else(|| String::from("bad avcC"))?;

    let p = &stsd.payload;
    if p.len() < 44 { return Err(String::from("stsd payload короче 44 байт")); }
    let width = u16::from_be_bytes([p[16 + 24], p[16 + 25]]) as u32;
    let height = u16::from_be_bytes([p[16 + 26], p[16 + 27]]) as u32;
    if width == 0 || height == 0 { return Err(String::from("width/height = 0")); }

    let duration_us: u64 = samples.last()
        .map(|s| s.timestamp_us + s.duration_us as u64).unwrap_or(0);
    let first_dur = samples.first().map(|s| s.duration_us).unwrap_or(33_333);
    let avg_framerate = if first_dur > 0 {
        (1_000_000u64 / first_dur as u64).max(1) as u32
    } else { 30 };

    return Ok(VideoInfo {
        width, height, avcc, codec_string, samples,
        duration_us, avg_framerate,
    });
}

fn build_audio_chunks(
    audio_info: &crate::mp4_demuxer::AudioTrackInfo, source: &[u8],
) -> Result<Vec<AudioChunk>, String> {
    let mut out = Vec::with_capacity(audio_info.samples.len());
    for s in &audio_info.samples {
        let start = s.offset as usize;
        let end = start.checked_add(s.size as usize)
            .ok_or_else(|| String::from("audio sample offset overflow"))?;
        if end > source.len() {
            return Err(format_str(format_args!(
                "audio sample #{} выходит за пределы файла", s.index,
            )));
        }
        out.push(AudioChunk {
            timestamp_us: s.timestamp_us,
            duration_us: s.duration_us,
            data: source[start..end].to_vec(),
        });
    }
    return Ok(out);
}

// =============================================================================
// GOP splitting
// =============================================================================

fn split_by_gops(samples: &[VideoSample]) -> Vec<Vec<usize>> {
    let mut gops: Vec<Vec<usize>> = Vec::new();
    let mut current: Vec<usize> = Vec::new();
    for (i, s) in samples.iter().enumerate() {
        let is_key = i == 0 || s.is_key;
        if is_key && !current.is_empty() {
            gops.push(std::mem::take(&mut current));
        }
        current.push(i);
    }
    if !current.is_empty() { gops.push(current); }
    return gops;
}

fn distribute_gops(gops: Vec<Vec<usize>>, n_workers: usize) -> Vec<Vec<usize>> {
    let mut buckets: Vec<Vec<usize>> = vec![Vec::new(); n_workers];
    for (i, gop) in gops.into_iter().enumerate() {
        buckets[i % n_workers].extend(gop);
    }
    return buckets;
}

// =============================================================================
// Чтение файла
// =============================================================================

async fn read_file(file: web_sys::File) -> Result<Vec<u8>, JsValue> {
    let reader = web_sys::FileReader::new()?;
    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let reader_clone = reader.clone();
        let onload: Closure<dyn FnMut()> = Closure::new(move || {
            let _ = resolve.call0(&JsValue::NULL);
        });
        let onerror: Closure<dyn FnMut()> = Closure::new(move || {
            let _ = reject.call1(&JsValue::NULL, &JsValue::from_str("read error"));
        });
        reader_clone.set_onload(Some(onload.as_ref().unchecked_ref()));
        reader_clone.set_onerror(Some(onerror.as_ref().unchecked_ref()));
        onload.forget();
        onerror.forget();
        let _ = reader_clone.read_as_array_buffer(&file);
    });
    wasm_bindgen_futures::JsFuture::from(promise).await?;
    let result = reader.result()?;
    let ab = result.dyn_into::<js_sys::ArrayBuffer>()
        .map_err(|_| JsValue::from_str("result is not ArrayBuffer"))?;
    return Ok(Uint8Array::new(&ab).to_vec());
}

// =============================================================================
// Worker pool
// =============================================================================

fn spawn_one_worker(state: Rc<RefCell<DemoState>>, index: usize) -> Result<WorkerHandle, JsValue> {
    let opts = WorkerOptions::new();
    opts.set_type(WorkerType::Module);
    let worker = Worker::new_with_options(WORKER_URL, &opts)?;

    let st_msg = state.clone();
    let onmessage: Closure<dyn FnMut(MessageEvent)> = Closure::new(move |event: MessageEvent| {
        let data = event.data();
        let ty = get_str(&data, "type");
        match ty.as_str() {
            "boot" => {}
            "ready" => {
                {
                    let mut s = st_msg.borrow_mut();
                    if index < s.pool.len() {
                        s.pool[index].ready = true;
                    }
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    update_ui(&doc, &st_msg.borrow());
                }
            }
            "boot_error" => {
                let msg = get_str(&data, "message");
                web_sys::console::error_1(&JsValue::from_str(&format!(
                    "[worker #{index}] boot error: {msg}"
                )));
                {
                    let mut s = st_msg.borrow_mut();
                    if index < s.pool.len() {
                        s.pool[index].ready = false;
                        s.pool[index].boot_error = Some(msg.clone());
                    }
                    if s.error.is_none() {
                        s.error = Some(format_str(format_args!("Worker #{index} boot: {msg}")));
                    }
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    update_ui(&doc, &st_msg.borrow());
                }
            }
            "progress" => {
                let session = get_f64(&data, "session") as u64;
                let frames = get_f64(&data, "frames") as u64;
                let total = get_f64(&data, "total") as u64;
                let fps = get_f64(&data, "fps");
                let ms_per_frame = get_f64(&data, "ms_per_frame");
                let filter_ms_avg = get_f64(&data, "filter_ms_avg");
                let encode_ms_avg = get_f64(&data, "encode_ms_avg");
                let gpu_path = get_bool(&data, "gpu_path");

                {
                    let mut s = st_msg.borrow_mut();
                    if s.session != session { return; }
                    if index >= s.pool.len() { return; }
                    s.pool[index].frames_done = frames;
                    s.pool[index].frames_total = total;
                    s.pool[index].fps = fps;
                    s.pool[index].ms_per_frame = ms_per_frame;
                    s.pool[index].filter_ms_avg = filter_ms_avg;
                    s.pool[index].encode_ms_avg = encode_ms_avg;
                    s.pool[index].gpu_path = gpu_path;
                    if s.stage == Stage::Demux && frames > 0 {
                        s.stage = Stage::Pipeline;
                    }
                    s.aggregate_metrics();
                    let now = js_sys::Date::now();
                    s.push_chart_sample(now);
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    update_ui(&doc, &st_msg.borrow());
                }
            }
            "chunks" => {
                let session = get_f64(&data, "session") as u64;
                let chunks_js = Reflect::get(&data, &JsValue::from_str("chunks"))
                    .unwrap_or(JsValue::UNDEFINED);
                let chunks_arr: Array = match chunks_js.dyn_into() {
                    Ok(a) => a, Err(_) => return,
                };

                let mut out: Vec<VideoChunk> = Vec::with_capacity(chunks_arr.length() as usize);
                for i in 0..chunks_arr.length() {
                    let obj = chunks_arr.get(i);
                    let is_key = get_bool(&obj, "is_key");
                    let ts = get_f64(&obj, "ts") as u64;
                    let dur = get_f64(&obj, "dur") as u32;
                    let data_js = Reflect::get(&obj, &JsValue::from_str("data"))
                        .unwrap_or(JsValue::UNDEFINED);
                    let bytes: Vec<u8> = if let Ok(ab) = data_js.dyn_into::<js_sys::ArrayBuffer>() {
                        Uint8Array::new(&ab).to_vec()
                    } else { Vec::new() };
                    out.push(VideoChunk {
                        is_key, timestamp_us: ts, duration_us: dur, data: bytes,
                    });
                }

                let avcc_js = Reflect::get(&data, &JsValue::from_str("avcc"))
                    .unwrap_or(JsValue::UNDEFINED);
                let result_avcc: Vec<u8> = if let Ok(ab) = avcc_js.dyn_into::<js_sys::ArrayBuffer>() {
                    Uint8Array::new(&ab).to_vec()
                } else { Vec::new() };
                let result_codec = get_str(&data, "codec");

                web_sys::console::log_1(&JsValue::from_str(&format!(
                    "[main] worker #{} returned {} chunks, avcc={} B, codec={}",
                    index, out.len(), result_avcc.len(), result_codec,
                )));

                let all_done = {
                    let mut s = st_msg.borrow_mut();
                    if s.session != session { return; }
                    if index >= s.pool.len() { return; }
                    s.pool[index].frames_done = out.len() as u64;
                    s.pool[index].result = Some(out);
                    s.pool[index].result_avcc = Some(result_avcc);
                    s.pool[index].result_codec = result_codec;
                    s.worker_results_count += 1;
                    s.workers_dispatched > 0 && s.worker_results_count >= s.workers_dispatched
                };

                if !all_done { return; }

                let (mut all_chunks, source_info, enc_avcc, enc_codec, worker_errors) = {
                    let mut s = st_msg.borrow_mut();
                    let mut all: Vec<VideoChunk> = Vec::new();
                    for w in s.pool.iter_mut() {
                        if let Some(chunks) = w.result.take() {
                            all.extend(chunks);
                        }
                    }
                    let info = s.source_info.take();
                    let avcc = s.pool.iter()
                        .find_map(|w| w.result_avcc.clone())
                        .unwrap_or_default();
                    let codec = s.pool.iter()
                        .find(|w| !w.result_codec.is_empty())
                        .map(|w| w.result_codec.clone())
                        .unwrap_or_default();
                    (all, info, avcc, codec, s.worker_errors)
                };

                all_chunks.sort_by_key(|c| c.timestamp_us);

                // **Диагностика**: первые 5 chunks — ts, is_key.
                if !all_chunks.is_empty() {
                    let sample: Vec<String> = all_chunks.iter().take(5).map(|c| {
                        format_str(format_args!(
                            "ts={} k={} dur={}",
                            c.timestamp_us, c.is_key as u8, c.duration_us
                        ))
                    }).collect();
                    web_sys::console::log_1(&JsValue::from_str(&format!(
                        "[main] first 5 chunks after sort: {}",
                        sample.join(" | ")
                    )));
                }

                if worker_errors > 0 {
                    let mut s = st_msg.borrow_mut();
                    s.error = Some(format_str(format_args!(
                        "{} воркер(ов) упали — pipeline прерван",
                        worker_errors,
                    )));
                    s.stage = Stage::Idle;
                    drop(s);
                    if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                        update_ui(&doc, &st_msg.borrow());
                    }
                    return;
                }

                let Some(info) = source_info else {
                    let mut s = st_msg.borrow_mut();
                    s.error = Some(String::from("source_info потерян"));
                    s.stage = Stage::Idle;
                    drop(s);
                    if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                        update_ui(&doc, &st_msg.borrow());
                    }
                    return;
                };

                if enc_avcc.is_empty() {
                    let mut s = st_msg.borrow_mut();
                    s.error = Some(String::from(
                        "encoder не вернул SPS/PPS (avcC) — нечего mux'ить",
                    ));
                    s.stage = Stage::Idle;
                    drop(s);
                    if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                        update_ui(&doc, &st_msg.borrow());
                    }
                    return;
                }

                web_sys::console::log_1(&JsValue::from_str(&format!(
                    "[main] muxing with encoder avcc: {} B, codec: {}",
                    enc_avcc.len(), enc_codec,
                )));

                let elapsed_ms = {
                    let s = st_msg.borrow();
                    if s.pipeline_start_ms > 0.0 {
                        js_sys::Date::now() - s.pipeline_start_ms
                    } else { 0.0 }
                };
                {
                    let mut s = st_msg.borrow_mut();
                    s.total_elapsed_ms = elapsed_ms;
                    s.frames_processed = all_chunks.len() as u64;
                    s.total_frames = all_chunks.len() as u64;
                    s.stage = Stage::Mux;
                    s.local_progress = 1.0;
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    update_ui(&doc, &st_msg.borrow());
                }

                let mux_result = build_mp4_with_audio(
                    &all_chunks,
                    &MuxConfig {
                        width: info.width,
                        height: info.height,
                        avcc: enc_avcc.clone(),
                    },
                    info.audio_input.as_ref(),
                );

                match mux_result {
                    Ok(mp4) => {
                        let arr = Array::new();
                        let u8 = Uint8Array::from(mp4.as_slice());
                        let u8_js: JsValue = u8.into();
                        arr.push(&u8_js);
                        let opts = BlobPropertyBag::new();
                        opts.set_type("video/mp4");
                        if let Ok(blob) = Blob::new_with_u8_array_sequence_and_options(&arr, &opts) {
                            if let Ok(url) = create_object_url(&blob) {
                                let summary = build_final_summary(
                                    &all_chunks, &info, mp4.len(),
                                    elapsed_ms, enc_avcc.len(), &enc_codec,
                                );

                                {
                                    let mut s = st_msg.borrow_mut();
                                    s.result_url = Some(url.clone());
                                    s.result_bytes = mp4;
                                    s.summary_html = summary;
                                    s.ready = true;
                                    s.stage = Stage::Done;
                                    s.local_progress = 1.0;
                                }

                                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                                    if let Some(video) = doc.get_element_by_id("pp-result")
                                        .and_then(|e| e.dyn_into::<HtmlVideoElement>().ok())
                                    {
                                        let _ = set_prop(
                                            video.as_ref(), "src", &JsValue::from_str(&url),
                                        );
                                        let _ = video.load();
                                    }
                                    update_ui(&doc, &st_msg.borrow());
                                }
                            }
                        }
                    }
                    Err(e) => {
                        {
                            let mut s = st_msg.borrow_mut();
                            s.error = Some(format_str(format_args!("mux: {e}")));
                            s.stage = Stage::Idle;
                        }
                        if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                            update_ui(&doc, &st_msg.borrow());
                        }
                    }
                }
            }
            "error" => {
                let session = get_f64(&data, "session") as u64;
                let message = get_str(&data, "message");
                let all_done = {
                    let mut s = st_msg.borrow_mut();
                    if s.session != session { return; }
                    s.worker_errors += 1;
                    s.worker_results_count += 1;
                    if s.error.is_none() {
                        s.error = Some(format_str(format_args!(
                            "Worker #{index}: {message}"
                        )));
                    }
                    s.workers_dispatched > 0 && s.worker_results_count >= s.workers_dispatched
                };
                if all_done {
                    let mut s = st_msg.borrow_mut();
                    let elapsed_ms = if s.pipeline_start_ms > 0.0 {
                        js_sys::Date::now() - s.pipeline_start_ms
                    } else { 0.0 };
                    s.total_elapsed_ms = elapsed_ms;
                    s.stage = Stage::Idle;
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    update_ui(&doc, &st_msg.borrow());
                }
            }
            _ => {}
        }
    });
    worker.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));

    let st_err = state.clone();
    let onerror: Closure<dyn FnMut(JsValue)> = Closure::new(move |e: JsValue| {
        let msg = get_str(&e, "message");
        let msg = if msg.is_empty() { format_str(format_args!("{e:?}")) } else { msg };
        web_sys::console::error_1(&JsValue::from_str(&format!(
            "[worker #{index}] global error: {msg}"
        )));
        {
            let mut s = st_err.borrow_mut();
            if index < s.pool.len() {
                s.pool[index].boot_error = Some(msg.clone());
            }
            if s.error.is_none() {
                s.error = Some(format_str(format_args!("Worker #{index}: {msg}")));
            }
        }
        if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
            update_ui(&doc, &st_err.borrow());
        }
    });
    worker.set_onerror(Some(onerror.as_ref().unchecked_ref()));

    let mut h = WorkerHandle::new(worker);
    h._on_message = Some(onmessage);
    h._on_error = Some(onerror);
    return Ok(h);
}

fn spawn_pool(state: Rc<RefCell<DemoState>>, n: usize) {
    let mut handles: Vec<Result<WorkerHandle, JsValue>> = Vec::with_capacity(n);
    for i in 0..n {
        handles.push(spawn_one_worker(state.clone(), i));
    }
    let mut s = state.borrow_mut();
    for h in handles {
        match h {
            Ok(w) => s.pool.push(w),
            Err(e) => {
                if s.error.is_none() {
                    s.error = Some(format_str(format_args!("spawn worker: {e:?}")));
                }
            }
        }
    }
}

fn restart_pool(state: Rc<RefCell<DemoState>>, n: usize) {
    {
        let mut s = state.borrow_mut();
        s.kill_pool();
        s.num_workers = n;
    }
    spawn_pool(state, n);
}

// =============================================================================
// send_bucket
// =============================================================================

#[allow(clippy::too_many_arguments)]
fn send_bucket(
    worker: &Worker,
    session: u64,
    codec_string: &str,
    avcc: &[u8],
    width: u32,
    height: u32,
    bitrate: u32,
    latency_mode: &str,
    filter_id: u32,
    samples: &[VideoSample],
    indices: &[usize],
    source: &[u8],
) -> Result<(), JsValue> {
    let chunks_arr = Array::new();
    let transfer = Array::new();

    for &idx in indices {
        let sample = &samples[idx];
        let start = sample.offset as usize;
        let end = start
            .checked_add(sample.size as usize)
            .ok_or_else(|| JsValue::from_str("sample offset overflow"))?;
        if end > source.len() {
            return Err(JsValue::from_str("sample вне файла"));
        }
        let bytes = &source[start..end];
        let u8 = Uint8Array::from(bytes);
        let buffer = u8.buffer();

        let obj = Object::new();
        let is_key = idx == 0 || sample.is_key;
        set_prop(&obj, "is_key", &JsValue::from_bool(is_key))?;
        set_prop(&obj, "ts", &JsValue::from_f64(sample.timestamp_us as f64))?;
        set_prop(&obj, "dur", &JsValue::from_f64(sample.duration_us as f64))?;
        set_prop(&obj, "data", buffer.as_ref())?;

        chunks_arr.push(&obj);
        transfer.push(buffer.as_ref());
    }

    let avcc_u8 = Uint8Array::from(avcc);
    let avcc_buffer = avcc_u8.buffer();

    let msg = Object::new();
    set_prop(&msg, "type", &JsValue::from_str("run"))?;
    set_prop(&msg, "session", &JsValue::from_f64(session as f64))?;
    set_prop(&msg, "codec_string", &JsValue::from_str(codec_string))?;
    set_prop(&msg, "avcc", avcc_buffer.as_ref())?;
    set_prop(&msg, "width", &JsValue::from_f64(width as f64))?;
    set_prop(&msg, "height", &JsValue::from_f64(height as f64))?;
    set_prop(&msg, "bitrate", &JsValue::from_f64(bitrate as f64))?;
    set_prop(&msg, "latency_mode", &JsValue::from_str(latency_mode))?;
    set_prop(&msg, "filter_id", &JsValue::from_f64(filter_id as f64))?;
    set_prop(&msg, "chunks", &chunks_arr)?;

    transfer.push(avcc_buffer.as_ref());
    worker.post_message_with_transfer(&msg, &transfer)?;
    return Ok(());
}

// =============================================================================
// Итоговый summary
// =============================================================================

fn build_final_summary(
    chunks: &[VideoChunk],
    info: &SourceInfo,
    result_size: usize,
    total_elapsed_ms: f64,
    enc_avcc_len: usize,
    enc_codec: &str,
) -> String {
    let key_count = chunks.iter().filter(|c| c.is_key).count();
    let duration_s = info.duration_us as f64 / 1_000_000.0;

    let elapsed_s = total_elapsed_ms / 1000.0;
    let fps_avg = if elapsed_s > 0.0 { chunks.len() as f64 / elapsed_s } else { 0.0 };
    let ms_per_frame_avg = if fps_avg > 0.0 { 1000.0 / fps_avg } else { 0.0 };

    let mut summary = String::new();
    summary.push_str(&row("Workers", &format_str(format_args!(
        r#"<span class="ok">N воркеров</span> <span class="muted">(параллельно по GOP)</span>"#
    ))));
    summary.push_str(&row("Video", &format_str(format_args!(
        r#"<span class="val">{w}×{h}</span> <span class="muted">@ {fr} fps</span>"#,
        w = info.width, h = info.height, fr = info.avg_framerate,
    ))));
    summary.push_str(&row("Duration", &format_str(format_args!(
        r#"<span class="val">{dur:.2} s</span>"#, dur = duration_s,
    ))));
    summary.push_str(&row("Frames", &format_str(format_args!(
        r#"<span class="val">{}</span>"#, chunks.len(),
    ))));
    summary.push_str(&row("Key frames", &format_str(format_args!(
        r#"<span class="ok">{key_count}</span>"#,
    ))));
    summary.push_str(&row("Total time", &format_str(format_args!(
        r#"<span class="ok">{elapsed}</span> <span class="muted">· {fps:.1} fps avg · {ms:.1} ms/frame</span>"#,
        elapsed = fmt_eta(elapsed_s), fps = fps_avg, ms = ms_per_frame_avg,
    ))));
    summary.push_str(&row("Encoder codec", &format_str(format_args!(
        r#"<span class="val">{codec}</span> <span class="muted">(Baseline — без B-frames)</span>"#,
        codec = enc_codec,
    ))));

    match &info.audio_input {
        Some(a) => {
            summary.push_str(&row("Audio", &format_str(format_args!(
                r#"<span class="ok">passthrough</span> <span class="muted">{sr} Hz, {ch} ch, {n} chunks, ts={ts}</span>"#,
                sr = a.sample_rate, ch = a.channels, n = a.chunks.len(), ts = a.timescale,
            ))));
        }
        None => {
            summary.push_str(&row("Audio", &format_str(format_args!(
                r#"<span class="muted">нет audio-трека</span>"#
            ))));
        }
    }

    summary.push_str(r#"<div class="sep"></div>"#);
    summary.push_str(&row("Source size", &format_str(format_args!(
        r#"<span class="val">{}</span>"#, fmt_bytes_short(info.source_size),
    ))));
    summary.push_str(&row("Result size", &format_str(format_args!(
        r#"<span class="val">{}</span>"#, fmt_bytes_short(result_size),
    ))));
    let ratio = if info.source_size > 0 {
        result_size as f64 / info.source_size as f64
    } else { 0.0 };
    let ratio_cls = if ratio <= 1.5 { "ok" } else if ratio <= 2.0 { "warn" } else { "err" };
    summary.push_str(&row("Ratio", &format_str(format_args!(
        r#"<span class="{cls}">{ratio:.2}×</span>"#,
        cls = ratio_cls, ratio = ratio,
    ))));
    summary.push_str(&row("Description", &format_str(format_args!(
        r#"<span class="val">{}</span> <span class="muted">(avcC от encoder'а)</span>"#,
        fmt_bytes_short(enc_avcc_len),
    ))));

    return summary;
}

// =============================================================================
// UI
// =============================================================================

fn update_progress(doc: &Document, st: &DemoState) {
    let pct = (st.progress() * 100.0).max(0.0).min(100.0);
    let pct_str = format_str(format_args!("{pct:.0}%"));
    if let Some(el) = doc.get_element_by_id("pp-progress-fill") {
        if let Ok(html_el) = el.dyn_into::<HtmlElement>() {
            let _ = html_el.style().set_property("width", &pct_str);
        }
    }
    if let Some(el) = doc.get_element_by_id("pp-progress-label") {
        let stage = st.stage.label();
        let local = if st.stage == Stage::Pipeline {
            format_str(format_args!("{}/{} кадров", st.frames_processed, st.total_frames))
        } else { String::new() };
        let left = format_str(format_args!(r#"<span class="stage-name">{stage}</span>"#));
        let right = format_str(format_args!(
            r#"<span class="stage-stats">{local} · {pct_str}</span>"#
        ));
        el.set_inner_html(&format_str(format_args!("{left}{right}")));
    }
}

fn update_speed_panel(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("pp-speed") else { return; };
    if st.stage != Stage::Pipeline && st.stage != Stage::Done && st.stage != Stage::Mux {
        el.set_inner_html("");
        return;
    }
    if st.frames_processed == 0 {
        el.set_inner_html(r#"<div><span class="muted">Скорость: инициализация…</span></div>"#);
        return;
    }

    let (fps, ms_per_frame) = if st.stage == Stage::Done {
        (st.final_fps(), st.final_ms_per_frame())
    } else {
        (st.fps, st.ms_per_frame)
    };

    let path_label = if st.gpu_path {
        r#"<span class="ok">GPU (OffscreenCanvas + filter)</span>"#
    } else {
        r#"<span class="warn">CPU (copyTo + WASM)</span>"#
    };

    let eta_html = if st.stage == Stage::Done {
        format_str(format_args!(
            r#"<span class="muted">завершено за</span> <span class="big">{}</span>"#,
            fmt_eta(st.total_elapsed_ms / 1000.0),
        ))
    } else {
        match st.eta_sec() {
            Some(sec) => format_str(format_args!(
                r#"<span class="muted">осталось ~</span><span class="big">{}</span>"#,
                fmt_eta(sec),
            )),
            None => String::new(),
        }
    };

    let total = st.filter_ms_avg + st.encode_ms_avg;
    let filter_pct = if total > 0.0 { st.filter_ms_avg / total * 100.0 } else { 0.0 };
    let encode_pct = if total > 0.0 { st.encode_ms_avg / total * 100.0 } else { 0.0 };

    let n_workers = st.pool.iter().filter(|w| w.ready).count();
    let worker_progresses: Vec<String> = st.pool.iter().map(|w| {
        if w.frames_total > 0 {
            format_str(format_args!("{}/{}", w.frames_done, w.frames_total))
        } else { String::from("—") }
    }).collect();
    let workers_line = worker_progresses.join(" · ");

    let html = format_str(format_args!(
        r#"<div>
             <span class="muted">Скорость:</span>
             <span class="big">{fps:.1} fps</span>
             <span class="muted">·</span>
             <span class="big">{ms:.1} ms/кадр</span>
             <span class="muted">·</span>
             <span class="muted">путь:</span> {path} {eta}
           </div>
           <div class="muted" style="margin-top:4px;">
             Средние стадии (ms/кадр на воркер):
             filter <span class="big">{flt:.1}</span> (<span class="muted">{flt_p:.0}%</span>)
             · encode <span class="big">{enc:.1}</span> (<span class="muted">{enc_p:.0}%</span>)
           </div>
           <div class="muted" style="margin-top:4px;">
             Воркеры ({n_ready}/{n_total}): {workers}
           </div>"#,
        fps = fps, ms = ms_per_frame, path = path_label, eta = eta_html,
        flt = st.filter_ms_avg, enc = st.encode_ms_avg,
        flt_p = filter_pct, enc_p = encode_pct,
        n_ready = n_workers, n_total = st.pool.len(), workers = workers_line,
    ));
    el.set_inner_html(&html);
}

fn render_chart(doc: &Document, st: &DemoState) {
    let Some(canvas_el) = doc.get_element_by_id("pp-chart") else { return; };
    let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() else { return; };
    let Ok(Some(ctx_obj)) = canvas.get_context("2d") else { return; };
    let Ok(ctx) = ctx_obj.dyn_into::<web_sys::CanvasRenderingContext2d>() else { return; };

    let w = canvas.width() as f64;
    let h = canvas.height() as f64;

    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);

    let samples: Vec<f64> = st.chart_fps.iter().cloned().collect();

    if samples.len() < 2 {
        ctx.set_fill_style_str("#484f58");
        ctx.set_font("11px ui-monospace");
        let _ = ctx.fill_text("график появится после «Обработать»", 20.0, h / 2.0 + 4.0);
        return;
    }

    ctx.set_stroke_style_str("#21262d");
    ctx.set_line_width(1.0);
    for i in 0..=4 {
        let y = (h * i as f64 / 4.0).round() + 0.5;
        ctx.begin_path();
        ctx.move_to(0.0, y);
        ctx.line_to(w, y);
        ctx.stroke();
    }

    let max_v = samples.iter().cloned().fold(0.0_f64, f64::max).max(30.0);

    ctx.set_fill_style_str("#484f58");
    ctx.set_font("10px ui-monospace");
    let label_max = format_str(format_args!("{max_v:.0} fps"));
    let _ = ctx.fill_text(&label_max, 4.0, 11.0);

    let n = samples.len();
    let x_step = w / (CHART_SAMPLES - 1).max(1) as f64;
    let x_start = w - (n - 1) as f64 * x_step;

    ctx.set_fill_style_str("rgba(63, 185, 80, 0.12)");
    ctx.begin_path();
    ctx.move_to(x_start, h);
    for (i, &v) in samples.iter().enumerate() {
        let x = x_start + i as f64 * x_step;
        let y = h - (v / max_v) * (h - 4.0) - 2.0;
        ctx.line_to(x, y);
    }
    let x_end = x_start + (n - 1) as f64 * x_step;
    ctx.line_to(x_end, h);
    ctx.close_path();
    ctx.fill();

    ctx.set_stroke_style_str("#3fb950");
    ctx.set_line_width(2.0);
    ctx.begin_path();
    for (i, &v) in samples.iter().enumerate() {
        let x = x_start + i as f64 * x_step;
        let y = h - (v / max_v) * (h - 4.0) - 2.0;
        if i == 0 { ctx.move_to(x, y); } else { ctx.line_to(x, y); }
    }
    ctx.stroke();

    if let Some(&last) = samples.last() {
        ctx.set_fill_style_str("#79c0ff");
        ctx.set_font("bold 11px ui-monospace");
        let label = format_str(format_args!("{last:.1} fps"));
        let _ = ctx.fill_text(&label, w - 80.0, h / 2.0 + 4.0);
    }
}

fn update_ui(doc: &Document, st: &DemoState) {
    update_progress(doc, st);
    update_speed_panel(doc, st);
    render_chart(doc, st);

    if let Some(el) = doc.get_element_by_id("pp-stats") {
        let err_line = st.error.as_ref()
            .map(|m| format_str(format_args!(r#"<div class="err">{m}</div>"#)))
            .unwrap_or_default();
        let file_line = if st.source_name.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#" · <span class="val">{name}</span> <span class="muted">({size})</span>"#,
                name = truncate_name(&st.source_name),
                size = st.source_bytes.as_ref()
                    .map(|b| fmt_bytes_short(b.len()))
                    .unwrap_or_else(|| String::from("—")),
            ))
        };
        let workers_ready = st.pool.iter().filter(|w| w.ready).count();
        let workers_total = st.pool.len();
        let ws = if workers_total == 0 {
            String::from("воркеры: инициализация…")
        } else {
            format_str(format_args!("воркеры: {workers_ready}/{workers_total}"))
        };
        let html = format_str(format_args!(
            r#"{err}<div><span class="lbl">Статус:</span> <span class="val">{stage}</span>{file} <span class="muted">· {ws}</span></div>"#,
            err = err_line, stage = st.stage.label(), file = file_line, ws = ws,
        ));
        el.set_inner_html(&html);
    }

    if let Some(el) = doc.get_element_by_id("pp-summary") {
        el.set_inner_html(&if st.summary_html.is_empty() {
            String::new()
        } else {
            format_str(format_args!(r#"<div class="pp-summary">{}</div>"#, st.summary_html))
        });
    }

    let active = st.is_active();
    let has_file = st.source_bytes.is_some();
    let any_filter = st.filter_grayscale || st.filter_invert || st.filter_sepia;
    let can_process = has_file && !active && any_filter && st.pool_ready();
    let can_cancel = active;
    let can_download = st.ready && !st.result_bytes.is_empty();

    if let Some(el) = doc.get_element_by_id("pp-process") {
        if can_process { let _ = el.remove_attribute("disabled"); }
        else { let _ = el.set_attribute("disabled", ""); }
    }
    if let Some(el) = doc.get_element_by_id("pp-cancel") {
        if can_cancel { let _ = el.remove_attribute("disabled"); }
        else { let _ = el.set_attribute("disabled", ""); }
    }
    if let Some(el) = doc.get_element_by_id("pp-download") {
        if can_download { let _ = el.remove_attribute("disabled"); }
        else { let _ = el.set_attribute("disabled", ""); }
    }
    if let Some(el) = doc.get_element_by_id("pp-open") {
        if active { let _ = el.set_attribute("disabled", ""); }
        else { let _ = el.remove_attribute("disabled"); }
    }
    for id in ["pp-f-gray", "pp-f-inv", "pp-f-sep"] {
        if let Some(el) = doc.get_element_by_id(id) {
            if let Ok(input) = el.dyn_into::<HtmlInputElement>() {
                input.set_disabled(active);
            }
        }
    }
    if let Some(el) = doc.get_element_by_id("pp-workers") {
        if active { let _ = el.set_attribute("disabled", ""); }
        else { let _ = el.remove_attribute("disabled"); }
    }
    if let Some(el) = doc.get_element_by_id("pp-latency") {
        if active { let _ = el.set_attribute("disabled", ""); }
        else { let _ = el.remove_attribute("disabled"); }
    }
}

// =============================================================================
// Start processing
// =============================================================================

fn start_processing(doc: Document, st: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    if !st.borrow().pool_ready() {
        return Err(JsValue::from_str("воркеры ещё не готовы"));
    }

    let (source, filter_id, latency_mode, num_workers) = {
        let s = st.borrow();
        let src = match s.source_bytes.clone() { Some(b) => b, None => return Ok(()) };
        (src, s.filter_id(), s.latency_mode.clone(), s.num_workers)
    };

    let boxes = match parse_mp4(&source) {
        Ok(b) => b,
        Err(e) => {
            {
                let mut s = st.borrow_mut();
                s.error = Some(format_str(format_args!("parse: {e}")));
                s.stage = Stage::Idle;
            }
            update_ui(&doc, &st.borrow());
            return Ok(());
        }
    };
    let info = match extract_video_info(&boxes) {
        Ok(i) => i,
        Err(e) => {
            {
                let mut s = st.borrow_mut();
                s.error = Some(e);
                s.stage = Stage::Idle;
            }
            update_ui(&doc, &st.borrow());
            return Ok(());
        }
    };
    if info.samples.is_empty() {
        {
            let mut s = st.borrow_mut();
            s.error = Some(String::from("нет video-сэмплов"));
            s.stage = Stage::Idle;
        }
        update_ui(&doc, &st.borrow());
        return Ok(());
    }

    let audio_input: Option<AudioTrackInput> = {
        let moov = boxes.iter().find(|b| b.box_type == "moov");
        match moov.and_then(|m| extract_audio_track(m).ok().flatten()) {
            Some(ai) => match build_audio_chunks(&ai, &source) {
                Ok(chunks) => Some(AudioTrackInput {
                    stsd_payload: ai.stsd_payload,
                    timescale: ai.timescale,
                    sample_rate: ai.sample_rate,
                    channels: ai.channels,
                    chunks,
                }),
                Err(e) => {
                    web_sys::console::warn_1(&JsValue::from_str(&format!("audio: {e}")));
                    None
                }
            },
            None => None,
        }
    };

    let duration_s = info.duration_us as f64 / 1_000_000.0;
    let src_bitrate = if duration_s > 0.0 {
        (source.len() as f64 * 8.0 / duration_s) as u32
    } else { 4_000_000 };
    let bitrate = src_bitrate.max(1_000_000).min(20_000_000);

    let gops = split_by_gops(&info.samples);
    let buckets = distribute_gops(gops, num_workers);
    let total_chunks: usize = buckets.iter().map(|b| b.len()).sum();

    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[main] demux: {} samples, {} buckets of sizes {:?}, bitrate={}",
        info.samples.len(), num_workers,
        buckets.iter().map(|b| b.len()).collect::<Vec<_>>(),
        bitrate,
    )));

    let session = {
        let mut s = st.borrow_mut();
        s.session = s.session.wrapping_add(1);
        s.stage = Stage::Pipeline;
        s.local_progress = 0.0;
        s.frames_processed = 0;
        s.total_frames = total_chunks as u64;
        s.fps = 0.0;
        s.ms_per_frame = 0.0;
        s.filter_ms_avg = 0.0;
        s.encode_ms_avg = 0.0;
        s.chart_fps.clear();
        s.chart_last_push_ms = 0.0;
        s.pipeline_start_ms = js_sys::Date::now();
        s.total_elapsed_ms = 0.0;
        s.error = None;
        s.ready = false;
        if let Some(url) = s.result_url.take() { revoke_url(&url); }
        s.result_bytes.clear();
        s.summary_html.clear();
        s.worker_results_count = 0;
        s.workers_dispatched = 0;
        s.worker_errors = 0;
        for w in s.pool.iter_mut() { w.reset_run_state(); }
        s.source_info = Some(SourceInfo {
            width: info.width,
            height: info.height,
            audio_input,
            duration_us: info.duration_us,
            avg_framerate: info.avg_framerate,
            source_size: source.len(),
        });
        s.session
    };
    update_ui(&doc, &st.borrow());

    let ready_workers: Vec<Worker> = {
        let s = st.borrow();
        s.pool.iter().filter(|h| h.ready).map(|h| h.worker.clone()).collect()
    };

    let nonempty_buckets: Vec<Vec<usize>> = buckets.into_iter()
        .filter(|b| !b.is_empty())
        .collect();

    let dispatched = ready_workers.len().min(nonempty_buckets.len());
    if dispatched == 0 {
        {
            let mut s = st.borrow_mut();
            s.error = Some(String::from("нет готовых воркеров"));
            s.stage = Stage::Idle;
        }
        update_ui(&doc, &st.borrow());
        return Ok(());
    }

    {
        let mut s = st.borrow_mut();
        s.workers_dispatched = dispatched;
    }

    for (bi, bucket) in nonempty_buckets.iter().take(dispatched).enumerate() {
        let worker = &ready_workers[bi];
        if let Err(e) = send_bucket(
            worker, session,
            &info.codec_string, &info.avcc,
            info.width, info.height, bitrate,
            &latency_mode, filter_id,
            &info.samples, bucket, &source,
        ) {
            {
                let mut s = st.borrow_mut();
                s.session = s.session.wrapping_add(1);
                s.error = Some(format_str(format_args!("send bucket #{bi}: {e:?}")));
                s.stage = Stage::Idle;
            }
            update_ui(&doc, &st.borrow());
            return Ok(());
        }
    }

    return Ok(());
}

// =============================================================================
// Скачивание
// =============================================================================

fn trigger_download(doc: &Document, st: &Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let url = match st.borrow().result_url.as_ref() {
        Some(u) => u.clone(), None => return Ok(()),
    };
    let a = doc.create_element("a")?;
    a.set_attribute("href", &url)?;
    a.set_attribute("download", "processed-23.12.mp4")?;
    let head = doc.head().ok_or_else(|| JsValue::from_str("no head"))?;
    head.append_child(&a)?;
    let a_el: HtmlElement = a.dyn_into()?;
    a_el.click();
    if let Some(parent) = a_el.parent_node() { let _ = parent.remove_child(&a_el); }
    return Ok(());
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document.get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23.12 — Практический проект: MP4 → обработка → MP4"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("pp-block");
    controls.set_inner_html(
        r#"<input type="file" id="pp-file" accept=".mp4,.m4v,.mov,video/mp4" style="display:none">
           <button class="pp-btn pp-btn-secondary" id="pp-open">Открыть MP4</button>
           <label class="pp-check" id="pp-lbl-gray"><input type="checkbox" id="pp-f-gray"> Grayscale</label>
           <label class="pp-check" id="pp-lbl-inv"><input type="checkbox" id="pp-f-inv"> Invert</label>
           <label class="pp-check" id="pp-lbl-sep"><input type="checkbox" id="pp-f-sep"> Sepia</label>
           <label class="pp-label">Воркеры:</label>
           <select id="pp-workers" class="pp-select">
             <option value="1">1</option>
             <option value="2">2</option>
             <option value="4" selected>4</option>
             <option value="8">8</option>
           </select>
           <label class="pp-label">LatencyMode:</label>
           <select id="pp-latency" class="pp-select">
             <option value="realtime">realtime</option>
             <option value="quality" selected>quality</option>
           </select>
           <button class="pp-btn pp-btn-primary" id="pp-process" disabled>Обработать</button>
           <button class="pp-btn pp-btn-danger" id="pp-cancel" disabled>Отмена</button>
           <button class="pp-btn" id="pp-download" disabled>Скачать MP4</button>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("pp-stats");
    stats.set_id("pp-stats");
    stats.set_inner_html(
        r#"<div><span class="lbl">Статус:</span> <span class="val">простой</span> <span class="muted">· воркеры: инициализация…</span></div>"#,
    );
    root.append_child(&stats)?;

    let progress = document.create_element("div")?;
    progress.set_class_name("pp-progress");
    progress.set_inner_html(
        r#"<div class="pp-progress-bar"><div class="pp-progress-fill" id="pp-progress-fill"></div></div>
           <div class="pp-progress-label" id="pp-progress-label"></div>"#,
    );
    root.append_child(&progress)?;

    let speed = document.create_element("div")?;
    speed.set_class_name("pp-speed");
    speed.set_id("pp-speed");
    root.append_child(&speed)?;

    let chart_wrap = document.create_element("div")?;
    chart_wrap.set_class_name("pp-chart-wrap");
    chart_wrap.set_inner_html(
        r#"<div class="pp-chart-title">
             <span>Скорость обработки (FPS)</span>
             <span>← прошлое · сейчас →</span>
           </div>
           <canvas id="pp-chart" class="pp-chart" width="1000" height="80"></canvas>"#,
    );
    root.append_child(&chart_wrap)?;

    let pair = document.create_element("div")?;
    pair.set_class_name("pp-pair");
    pair.set_inner_html(
        r#"<div class="pp-slot">
             <div class="pp-slot-title">Источник (MP4 из файла)</div>
             <video id="pp-source" class="pp-video" controls muted playsinline></video>
           </div>
           <div class="pp-slot">
             <div class="pp-slot-title pp-slot-title-res">Результат (обработанный MP4)</div>
             <video id="pp-result" class="pp-video" controls playsinline></video>
           </div>"#,
    );
    root.append_child(&pair)?;

    let summary = document.create_element("div")?;
    summary.set_id("pp-summary");
    root.append_child(&summary)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("pp-footnote");
    footnote.set_text_content(Some(
        "N воркеров параллельно: main thread делает demux, разбивает chunks по GOP'ам, \
         распределяет по воркерам. Encoder принудительно Baseline — без B-frames \
         (иначе нужен ctts-box в MP4, которого у нас нет). Mux использует avcC \
         от encoder'а. Отмена — terminate() всех воркеров.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new(DEFAULT_NUM_WORKERS)));
    spawn_pool(state.clone(), DEFAULT_NUM_WORKERS);
    update_ui(&document, &state.borrow());

    let input_el = document.get_element_by_id("pp-file")
        .ok_or_else(|| JsValue::from_str("no #pp-file"))?;
    let input: HtmlInputElement = input_el.dyn_into()?;

    {
        let input_for_btn = input.clone();
        let btn_el = document.get_element_by_id("pp-open")
            .ok_or_else(|| JsValue::from_str("no #pp-open"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || { input_for_btn.click(); });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let input_clone = input.clone();
        let onchange: Closure<dyn FnMut()> = Closure::new(move || {
            let Some(files) = input_clone.files() else { return; };
            let Some(file) = files.get(0) else { return; };
            let name = file.name();
            {
                let mut s = st.borrow_mut();
                s.error = None;
                s.source_name = name.clone();
                if let Some(url) = s.result_url.take() { revoke_url(&url); }
                s.result_bytes.clear();
                s.summary_html.clear();
                s.ready = false;
                s.stage = Stage::Idle;
            }
            update_ui(&doc, &st.borrow());

            let doc_cb = doc.clone();
            let st_cb = st.clone();
            let input_for_reset = input_clone.clone();
            spawn_local(async move {
                match read_file(file).await {
                    Ok(bytes) => {
                        let arr = Array::new();
                        let u8 = Uint8Array::from(bytes.as_slice());
                        let u8_js: JsValue = u8.into();
                        arr.push(&u8_js);
                        let opts = BlobPropertyBag::new();
                        opts.set_type("video/mp4");
                        if let Ok(blob) = Blob::new_with_u8_array_sequence_and_options(&arr, &opts) {
                            if let Ok(url) = create_object_url(&blob) {
                                if let Some(video) = doc_cb.get_element_by_id("pp-source")
                                    .and_then(|e| e.dyn_into::<HtmlVideoElement>().ok())
                                {
                                    let _ = set_prop(video.as_ref(), "src", &JsValue::from_str(&url));
                                    let _ = video.load();
                                }
                                {
                                    let mut s = st_cb.borrow_mut();
                                    if let Some(old) = s.source_url.take() { revoke_url(&old); }
                                    s.source_url = Some(url);
                                    s.source_bytes = Some(bytes);
                                }
                            }
                        }
                        update_ui(&doc_cb, &st_cb.borrow());
                    }
                    Err(e) => {
                        let mut s = st_cb.borrow_mut();
                        s.error = Some(format_str(format_args!("read: {e:?}")));
                        s.source_bytes = None;
                        drop(s);
                        update_ui(&doc_cb, &st_cb.borrow());
                    }
                }
                input_for_reset.set_value("");
            });
        });
        input.add_event_listener_with_callback("change", onchange.as_ref().unchecked_ref())?;
        onchange.forget();
    }

    for (id, setter) in [
        ("pp-f-gray", (|s: &mut DemoState, v| s.filter_grayscale = v) as fn(&mut DemoState, bool)),
        ("pp-f-inv", |s, v| s.filter_invert = v),
        ("pp-f-sep", |s, v| s.filter_sepia = v),
    ] {
        let doc = document.clone();
        let st = state.clone();
        let el = document.get_element_by_id(id)
            .ok_or_else(|| JsValue::from_str("missing checkbox"))?;
        let input_el: HtmlInputElement = el.dyn_into()?;
        let input_for_cb = input_el.clone();
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let checked = input_for_cb.checked();
            {
                let mut s = st.borrow_mut();
                s.filter_grayscale = false;
                s.filter_invert = false;
                s.filter_sepia = false;
                setter(&mut s, checked);
            }
            for other in ["pp-f-gray", "pp-f-inv", "pp-f-sep"] {
                if let Some(el) = doc.get_element_by_id(other) {
                    if let Ok(input) = el.dyn_into::<HtmlInputElement>() {
                        if other != id { input.set_checked(false); }
                    }
                }
            }
            update_ui(&doc, &st.borrow());
        });
        input_el.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let el = document.get_element_by_id("pp-workers")
            .ok_or_else(|| JsValue::from_str("no #pp-workers"))?;
        let sel: web_sys::HtmlSelectElement = el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let Some(el) = doc.get_element_by_id("pp-workers") else { return; };
            let Ok(sel) = el.dyn_into::<web_sys::HtmlSelectElement>() else { return; };
            let n: usize = sel.value().parse().unwrap_or(DEFAULT_NUM_WORKERS);
            if n == 0 { return; }
            restart_pool(st.clone(), n);
            update_ui(&doc, &st.borrow());
        });
        sel.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let el = document.get_element_by_id("pp-latency")
            .ok_or_else(|| JsValue::from_str("no #pp-latency"))?;
        let sel: web_sys::HtmlSelectElement = el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let Some(el) = doc.get_element_by_id("pp-latency") else { return; };
            let Ok(sel) = el.dyn_into::<web_sys::HtmlSelectElement>() else { return; };
            st.borrow_mut().latency_mode = sel.value();
        });
        sel.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("pp-process")
            .ok_or_else(|| JsValue::from_str("no #pp-process"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            if let Err(e) = start_processing(doc.clone(), st.clone()) {
                {
                    let mut s = st.borrow_mut();
                    s.error = Some(format_str(format_args!("start: {e:?}")));
                    s.stage = Stage::Idle;
                }
                update_ui(&doc, &st.borrow());
            }
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("pp-cancel")
            .ok_or_else(|| JsValue::from_str("no #pp-cancel"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let n = {
                let mut s = st.borrow_mut();
                s.cancel();
                s.num_workers
            };
            spawn_pool(st.clone(), n);
            if let Some(video) = doc.get_element_by_id("pp-result")
                .and_then(|e| e.dyn_into::<HtmlVideoElement>().ok())
            {
                detach_video(&video);
            }
            update_ui(&doc, &st.borrow());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("pp-download")
            .ok_or_else(|| JsValue::from_str("no #pp-download"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let _ = trigger_download(&doc, &st);
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    return Ok(());
}
```

---

## 23.12.7. Практические находки (по ходу итераций)

Список грабель, найденных при разработке:

### 1. Key-chunk первым

**Симптом:** `DataError: VideoDecoder needs a key chunk`.

**Причина:** `VideoDecoder` требует, чтобы первый `decode()` после `configure()` получил `key`-chunk.

**Что было не так:** первая версия разбивала по индексу без учёта key — первый sample мог быть delta (например, если `stss` пропускал нулевой sample).

**Fix:** перед отправкой в воркер всегда форсируем первый chunk как `is_key = true`. Внутри воркера ещё раз: `keyFrame = frames_processed == 0 || key_flags[idx]`.

### 2. `flush()` между батчами ломает decoder

**Симптом:** второй батч падал с `DataError: needs a key chunk`.

**Причина:** `await decoder.flush()` переводит `VideoDecoder` в состояние **awaiting key chunk**. Следующий `decode()` обязан быть key.

**Fix:** никаких промежуточных `flush()`. Только финальный — после всех chunks.

### 3. Хост `new VideoFrame(canvas)` делает readback

**Симптом:** медленно (~10-15 ms/кадр на 720×1280).

**Причина:** Chrome неявно делает GPU→CPU readback при `VideoFrame(canvas)`.

**Fix:** `OffscreenCanvas.transferToImageBitmap()` + `new VideoFrame(bitmap)`. Все операции GPU↔GPU.

### 4. Audio терялось

**Симптом:** нет звука.

**Причина:** `mdhd.duration` и `stts.sample_delta` для audio были в **микросекундах**, а не в единицах `timescale`. Плеер видел несогласованный трек и отбрасывал его.

**Fix:** пересчёт в единицах `audio.timescale`:
```rust
let duration_ts = duration_us * timescale / 1_000_000;
```

### 5. Дёрганье из-за B-frames

**Симптом:** видео играется, но кадры рвано идут.

**Причина:** Chrome encoder по умолчанию — High Profile с B-frames. Для B-frames нужен `ctts`-box. У нас его нет, PTS ≠ DTS, плеер путает порядок.

**Fix:** `ENCODER_CODEC = "avc1.42E01E"` — Baseline. Без B-frames, PTS = DTS.

### 6. `worker_errors` не учитывался

**Симптом:** pipeline мог зависнуть, ожидая ответа от упавшего воркера.

**Fix:** в `"error"` handler — `s.worker_results_count += 1`. При `all_done` если `worker_errors > 0` — mux **не запускаем**, показываем ошибку.

### 7. Borrow conflict в spawn pool

**Симптом:** panic при спавне.

**Причина:** `for i in 0..n { spawn_one_worker(state.clone(), i) }` внутри `state.borrow_mut()`.

**Fix:** `spawn_pool` сначала **все** handles собирает локально, потом одним `borrow_mut()` добавляет в pool.

### 8. Race при dispatch

**Симптом:** иногда воркер успевал ответить раньше, чем `workers_dispatched` устанавливался — в итоге `all_done` не срабатывал.

**Fix:** `workers_dispatched` устанавливается **до** цикла отправки.

### 9. `frames_processed` для коротких bucket'ов

**Симптом:** если bucket короткий и воркер не прислал ни одного progress-сообщения, `frames_done` = 0, счётчики сбивались.

**Fix:** в `"chunks"` handler фиксируем `frames_done = out.len()` до агрегации.

---

## 23.12.8. Ожидаемые логи

**Успешный запуск (8 воркеров):**

```
[pipeline-worker] registering onmessage
[demux] audio: 1564 samples, 44100 Hz, 2 ch, timescale=44100
[main] demux: 2175 samples, 8 buckets of sizes [300, 300, 300, 300, 300, 300, 225, 150], bitrate=2897559
[pipeline-worker] bucket: 300 chunks, 720x1280, codec=avc1.64001F, latency=quality, bitrate=2897559
[pipeline-worker] encoder forced to avc1.42E01E (Baseline, no B-frames)
[pipeline-worker] bucket done: 300 chunks, avcc=42 B, codec=avc1.42E01E, time=51700 ms
[main] worker #0 returned 300 chunks, avcc=42 B, codec=avc1.42E01E
...
[main] first 5 chunks after sort: ts=0 k=1 dur=33333 | ts=33333 k=0 | ts=66666 k=0 | ts=100000 k=0 | ts=133333 k=0
[main] muxing with encoder avcc: 42 B, codec: avc1.42E01E
```

**Что важно проверить:**
- `codec=avc1.42E01E` — Baseline применился.
- `avcc=42 B` — короткий Baseline SPS/PPS.
- `first 5 chunks after sort: ts=0 k=1 dur=33333 | ts=33333 k=0 | ...` — монотонно.

**Если `codec=avc1.64001F`** — fix не применился, был кэширован старый `pkg/`. Очистите и пересоберите: `rm -rf web/pkg && npm run wasm:dev`.

---

## 23.12.9. Итог

**Что получилось:**

- Полный цикл MP4 → grayscale/sepia/invert → MP4.
- N параллельных воркеров с разбивкой по GOP'ам (~41 fps avg против ~22 fps на одном).
- GPU-фильтр через `OffscreenCanvas + ctx.filter` — без readback.
- Audio passthrough — стерео, синхронно.
- График FPS в реальном времени.
- Total time фиксируется и показывается в summary.

**Что не реализовано (задачи на будущее):**

- **Multiple GOP'ы на один worker — round-robin.** Сейчас простой round-robin, но можно умнее: greedily распределять по времени.
- **Параллельный mux.** Пока mux в main thread — быстро, но при 100k+ chunks можно распараллелить.
- **Variable frame rate.** Сейчас `stts` строим по фактическим durations, но если source — VFR, muxer не идеален.
- **B-frames.** Оставили за скобками — сложно, требует `ctts`, обсуждалось в 23.12.3.

**Правило проекта, зафиксированное в главе.** Все критичные решения имеют **комментарии в коде** с объяснением причины. Особенно те, что неочевидны и выглядят как «магия» — Baseline profile, refill duration map, key-chunk первым. Через полгода никто не помнит, почему оно так, но комментарий даст контекст.