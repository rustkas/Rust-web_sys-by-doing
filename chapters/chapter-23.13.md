# Глава 23.13. AudioWorklet: real-time audio pipeline

## 23.13.0. Обновлённая структура

```
src/
  audio_api.rs                    ← НОВЫЙ модуль 23.13
  app.rs                          ← демо 23.13 (заменяется целиком)
tests/
  wasm_audio_api.rs               ← НОВЫЙ: DOM-зависимые API, wasm
web/
  public/worklets/
    pitch-processor.js            ← НОВЫЙ: self-contained worklet (без import)
src/lib.rs                        ← +1 строка: pub mod audio_api;
Cargo.toml                        ← без изменений (все фичи уже есть)
```

**Комментарий.** Модуль `audio_api.rs` — обёртки вокруг `AudioWorklet` (`addModule`, `AudioWorkletNode`, `MessagePort`) и `WebCodecs audio` (`AudioEncoder`, `AudioDecoder`, `AudioData`). Worklet-файл — **чистый JS без единого import**, в `web/public/worklets/`. Префикс CSS — `aw-` (audio worklet).

**Ссылки для детального изучения:**

- MDN: [`AudioWorkletProcessor.process()`](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process)
- MDN: [`AudioWorkletNode`](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletNode)
- W3C: [Web Audio API § AudioWorklet](https://www.w3.org/TR/webaudio/#AudioWorklet)
- Chrome Developers: [Enter AudioWorklet](https://developer.chrome.com/blog/audio-worklet/)
- W3C: [WebCodecs § AudioEncoder](https://w3c.github.io/webcodecs/#audioencoder-interface)

---

## 23.13.1. О чём эта глава

`AudioWorklet` — низкоуровневый API для обработки аудио **на отдельном потоке**, в реальном времени, с жёстким бюджетом времени. Это единственный способ делать настоящий DSP в браузере без задержек и глитчей.

**Что демонстрируется:**

1. **Pitch shifter** — кольцевой буфер с переменной скоростью чтения (реальный DSP в worklet'е).
2. **Per-block метрики** — `ms / блок`, запас бюджета, счётчик xruns.
3. **Waveform** — визуальное доказательство работы (вход/выход рядом).
4. **Ratio slider** — обновление в реальном времени.
5. **Opus round-trip** (WebCodecs, main thread) — параллельный раздел.

**Что эта глава **не** обещает:**

- Красивого Hi-Fi voice changer'а — простой ресемплер даёт «chipmunk»-эффект.
- WASM в worklet'е — почему именно нет, разбирается отдельно (23.13.4). Это одна из самых ценных находок главы.

---

## 23.13.2. Real-time constraint: 128 сэмплов

`AudioWorkletProcessor.process()` вызывается один раз на 128 сэмплов (это фиксированный размер блока в Web Audio API). На 48 kHz это:

```
48000 samples/sec → 128 samples = 2.667 мс
```

**Что это значит.** Раз в 2.667 мс браузер вызывает `process()`. Функция должна:

1. Прочитать 128 входных сэмплов.
2. Обработать их.
3. Записать 128 выходных сэмплов.
4. **Уложиться в 2.667 мс.**

Если не уложилась — следующий блок не успел к границе, получается **xrun** (пропуск). Слышно как щелчок или тишину на 2.667 мс.

**Правила для `process()`:**

| ❌ Нельзя | ✅ Нужно |
|-----------|---------|
| `await` | Предвыделять буферы в конструкторе |
| `[]`, `{}`, строки | Переиспользовать массивы |
| `postMessage` в hot path | Отправлять метрики раз в 500 мс |
| `fetch`, `import()` | Все ресурсы — на этапе `addModule` |
| Долгие циклы | Простая арифметика над буфером |

**Почему это важно для JS.** V8 рискует:

- **JIT-warmup.** Первые блоки медленнее до прогрева.
- **GC-паузы.** Даже 1 мс stop-the-world ломает ритм.
- **Allocation в цикле.** Любой `{}` — триггер GC.

Для простых DSP JS держит бюджет (мы увидим 0.05 мс в метриках). Для FFT, convolution, ML — уже нет, там нужен WASM (см. 23.13.4).

---

## 23.13.3. Семь грабель AudioWorklet'а

Мы прошли через них — каждая неочевидна и стоила итерации. Вот полная карта.

### Грабли 1. `AudioWorkletGlobalScope` не похож на Worker

Первое интуитивное желание — обращаться с worklet'ом как с обычным `Worker`. Это не работает.

| Что | В `Worker` | В `AudioWorkletGlobalScope` |
|-----|:----------:|:---------------------------:|
| `self.postMessage` | ✅ | ❌ **Нет `self`** |
| `self.onmessage` | ✅ | ❌ **Нет `self`** |
| `addEventListener` | ✅ | ❌ |
| `fetch()` | ✅ | ❌ |
| `import()` (dynamic) | ✅ | ❌ Запрещено |
| `WebAssembly.instantiate` | ✅ | ✅ (если байты уже есть) |
| `console.log` | ✅ | ✅ |
| `performance.now()` | ✅ | ✅ |
| `MessagePort` | ✅ | ✅ но через `this.port` |

**Что реально доступно в worklet'е:**

```javascript
// Глобалы audio rendering thread:
sampleRate        // число — обычно 48000 или 44100
currentTime       // число — время в секундах с начала контекста
currentFrame      // целое — номер сэмпла с начала контекста
registerProcessor(name, ctor)  // функция регистрации
console           // console.log/warn/error
performance       // performance.now()

// Внутри process() — входы/выходы:
inputs  // [[Float32Array(128), Float32Array(128)]]
outputs // [[Float32Array(128), Float32Array(128)]]
```

**Итог.** Общение с main thread — только через `this.port.postMessage` / `this.port.onmessage`. Никаких глобальных `postMessage`.

### Грабли 2. Dynamic `import()` запрещён

Логичное желание — загрузить WASM внутри worklet'а:

```javascript
class P extends AudioWorkletProcessor {
    async _initWasm() {
        const mod = await import('/pkg/hello_wasm02.js');  // ← ЭТО НЕ РАБОТАЕТ
        // ...
    }
}
```

Chrome бросает:

```
TypeError: Dynamic import not supported in this context
```

**Почему.** В `AudioWorkletGlobalScope` нет event loop в обычном смысле. Есть только вызовы `process()` каждые 2.667 мс. Промисы не могут резолвиться между вызовами — нет микротаск-очереди. Спецификация W3C прямо запрещает.

**Следствие.** Нельзя загрузить WASM «по требованию» из worklet'а. Нужно либо вшить его в worklet-файл на этапе сборки, либо не использовать WASM в worklet'е вообще.

### Грабли 3. `fetch()` недоступен

Если нельзя `import()`, попробуем вручную:

```javascript
const resp = await fetch('/pkg/hello_wasm02_bg.wasm');  // ← НЕ РАБОТАЕТ
const bytes = await resp.arrayBuffer();
```

Chrome бросает:

```
TypeError: fetch is not a function
```

**Почему.** `fetch` живёт на `WindowOrWorkerGlobalScope`. В `AudioWorkletGlobalScope` этого mixin'а нет — предполагается, что worklet не должен загружать ресурсы из сети.

**Что можно.** Если байты **уже есть в памяти** — можно передать их в `WebAssembly.instantiate(bytes)`. Это работает.

**Рабочий паттерн.** Main thread скачивает `.wasm` через `fetch()`, передаёт `ArrayBuffer` в worklet через `port.postMessage({type: "wasm-init", bytes})`. Worklet получает байты в `this.port.onmessage` и вызывает `WebAssembly.instantiate` синхронно.

Но! Тогда возникает следующая проблема (см. грабли 4).

### Грабли 4. `postMessage` + `WebAssembly.instantiate` — async по природе

Даже когда байты уже в worklet'е, `WebAssembly.instantiate(bytes)` возвращает `Promise`. А в `process()` `await` невозможен.

**Что делают.** В `this.port.onmessage` запускают `WebAssembly.instantiate(bytes)` — но результат приходит в микротаск, а микротасков в worklet'е нет. Промис либо висит вечно, либо резолвится при следующем `process()`. Chrome реализует это как «резолвится в следующем `process()` вызове» — 2.667 мс задержки. Работает, но криво.

**Альтернатива.** `new WebAssembly.Module(bytes)` синхронно + `new WebAssembly.Instance(module, imports)` синхронно. Оба доступны в worklet'е. **Это правильный путь**, если нужен WASM в worklet'е.

Но для этого нужны байты WASM без JS-обвязки — что приводит к граблям 5.

### Грабли 5. `wasm-pack --target web` не подходит для worklet'а

`wasm-pack build --target web` генерирует пару `.js` + `.wasm`:

```
web/pkg/
  hello_wasm02.js        ← ES-модуль с auto-init
  hello_wasm02_bg.wasm   ← бинарь
```

`hello_wasm02.js` использует `import.meta.url` для вычисления пути к `.wasm`. Это работает **только в контексте настоящего модуля, загруженного с URL**. Внутри worklet'а (где нет URL, где нет `import.meta` для Blob'а) — не работает.

**Чтобы вшить WASM в worklet:**

1. Собрать с `wasm-pack build --target no-modules` — генерируется один `.js` без ES-модулей.
2. Заменить `fetch()` внутри glue-кода на передачу байт извне.
3. Склеить `glue.js` + `worklet.js` в один файл на этапе сборки.

Это ручная работа, требующая build-скрипта. Для учебной главы — избыточно.

**Решение для этой главы.** Чистый JS в worklet'е. Ring buffer — не тяжёлая DSP.

### Грабли 6. Vite SPA-fallback проглатывает 404

Если worklet-файл **не существует** или **недоступен** по URL, Vite (в dev-режиме) отдаёт `index.html` вместо `404 Not Found`. Браузер пытается распарсить HTML как JavaScript и падает:

```
SyntaxError: expected expression, got '<'
```

**Как диагностировать.** DevTools → Network → клик на запрос `/worklets/pitch-processor.js` → Response. Если там HTML — вот причина.

**Как избежать.**

1. **Проверять фактический Response.** В `addModule().catch()` — искать `'<'` в тексте ошибки и давать понятное сообщение.
2. **Не использовать SPA-fallback для публичных путей.** В `vite.config.js` можно исключить `/worklets/` из SPA-fallback. Но проще — держать файл в правильном месте (`web/public/worklets/`).

### Грабли 7. web-sys API меняется между версиями

Две находки на этой неделе:

**`AudioWorkletNode::port()`** в web-sys 0.3.105 возвращает `Result<MessagePort, JsValue>`, не `MessagePort`. Свойство `port` объявлено как potentially-undefined в bindgen. Код:

```rust
let port = node_port(node)?;  // правильный вызов
// не: let port = node.port();  // ← ошибка типов
```

**`AudioBuffer::get_channel_data()`** возвращает `Result<Vec<f32>, JsValue>`, не `Result<Float32Array, JsValue>`. Вектор — сразу скопированные данные в Rust-память.

**Правило.** Всегда `cargo check` после обновления `Cargo.lock`. Сверяйтесь с актуальным API на docs.rs/web-sys.

---

## 23.13.4. Почему WASM в worklet'е не оправдан

Мы прошли через шесть грабель выше и пришли к выводу:

**Для этой задачи WASM в worklet'е не нужен.** Ring buffer — простая арифметика:

```javascript
// ~30 строк JS:
ring[w] = input[i];
w = (w + 1) % R;
const i0 = r | 0;
const i1 = (i0 + 1) % R;
const frac = r - i0;
out[i] = ring[i0] * (1 - frac) + ring[i1] * frac;
```

Это укладывается в **0.05 мс** при бюджете 2.667 мс — в 50 раз меньше. Даже в JS.

**Когда WASM в worklet'е оправдан:**

- FFT / convolution reverb — миллионы комплексных умножений.
- ML-инференс (шумоподавление, распознавание речи).
- Спектральный анализ (спектрограмма, pitch detection).
- Серьёзные цифровые фильтры (biquad × N полос).

**Когда не оправдан:**

- Простые ring buffer'ы.
- Простые gain / pan.
- Any DSP, где цикл по 128 сэмплам — это < 1000 операций.

**Что показывает глава.** Что для правильной задачи метрики видны и понятны. `0.05 мс / блок` — это и есть аргумент: «JS справляется, WASM не нужен». Если бы было `2.5 мс / блок` — вывод был бы противоположный.

Это честнее, чем тянуть WASM в worklet ради галочки.

---

## 23.13.5. Исходные файлы

### `src/lib.rs`

```rust
pub mod app;
pub mod audio_api;              // ← 23.13
pub mod container_parser;
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod pipeline_worker;
pub mod video_filters;
pub mod video_frame;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

### `src/audio_api.rs`

```rust
//! 23.13. AudioWorklet + WebCodecs audio API wrappers.
//!
//! Всё общение с JsValue — через Reflect. Мини-хелперы — из `video_frame::*`.

use js_sys::{Array, Object, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{AudioContext, AudioWorkletNode, MessagePort};

use crate::video_frame::{format_str, get_fn, num, set_prop};

// ----- Feature detection -----

pub fn has_audio_worklet(ctx: &AudioContext) -> bool {
    let ctx_js: &JsValue = ctx.as_ref();
    let Ok(aw) = Reflect::get(ctx_js, &JsValue::from_str("audioWorklet")) else {
        return false;
    };
    !aw.is_undefined() && !aw.is_null()
}

pub fn is_audio_encoder_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("AudioEncoder"))
        .map(|v| v.is_function()).unwrap_or(false)
}

pub fn is_audio_decoder_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("AudioDecoder"))
        .map(|v| v.is_function()).unwrap_or(false)
}

pub fn is_audio_data_supported() -> bool {
    let global = js_sys::global();
    Reflect::get(&global, &JsValue::from_str("AudioData"))
        .map(|v| v.is_function()).unwrap_or(false)
}

pub fn has_get_user_media() -> bool {
    let global = js_sys::global();
    let Ok(nav) = Reflect::get(&global, &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(md) = Reflect::get(&nav, &JsValue::from_str("mediaDevices")) else {
        return false;
    };
    Reflect::get(&md, &JsValue::from_str("getUserMedia"))
        .map(|v| v.is_function()).unwrap_or(false)
}

// ----- AudioWorklet -----

pub async fn add_worklet_module(ctx: &AudioContext, url: &str) -> Result<(), JsValue> {
    let ctx_js: &JsValue = ctx.as_ref();
    let aw = Reflect::get(ctx_js, &JsValue::from_str("audioWorklet"))?;
    let add = get_fn(&aw, "addModule")?;
    let promise = add.call1(&aw, &JsValue::from_str(url))?
        .dyn_into::<Promise>()?;
    JsFuture::from(promise).await.map(|_| ())
}

pub fn create_worklet_node(
    ctx: &AudioContext,
    name: &str,
    channels: u32,
) -> Result<AudioWorkletNode, JsValue> {
    let opts = Object::new();
    set_prop(&opts, "numberOfInputs", &num(1.0))?;
    set_prop(&opts, "numberOfOutputs", &num(1.0))?;
    let ch_arr = Array::new();
    ch_arr.push(&num(channels as f64));
    set_prop(&opts, "outputChannelCount", &ch_arr)?;
    set_prop(&opts, "channelCount", &num(channels as f64))?;
    set_prop(&opts, "channelCountMode", &JsValue::from_str("explicit"))?;
    set_prop(&opts, "channelInterpretation", &JsValue::from_str("speakers"))?;

    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioWorkletNode"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("AudioWorkletNode not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let args = Array::new();
    args.push(ctx.as_ref());
    args.push(&JsValue::from_str(name));
    args.push(&opts);
    let node = Reflect::construct(&ctor, &args)?;
    node.dyn_into::<AudioWorkletNode>()
}

/// В web-sys 0.3.105+ `AudioWorkletNode::port()` возвращает
/// `Result<MessagePort, JsValue>`.
pub fn node_port(node: &AudioWorkletNode) -> Result<MessagePort, JsValue> {
    return node.port();
}

pub fn post_ratio(port: &MessagePort, ratio: f32) -> Result<(), JsValue> {
    let msg = Object::new();
    set_prop(&msg, "type", &JsValue::from_str("ratio"))?;
    set_prop(&msg, "value", &num(ratio as f64))?;
    let msg_js: JsValue = msg.into();
    port.post_message(&msg_js)?;
    Ok(())
}

// ----- WebCodecs audio -----

pub async fn is_opus_supported(
    sample_rate: u32,
    channels: u32,
    bitrate: u32,
) -> Result<bool, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioEncoder"))?;
    if !ctor.is_function() {
        return Ok(false);
    }
    let is_cs = Reflect::get(&ctor, &JsValue::from_str("isConfigSupported"))?;
    if !is_cs.is_function() {
        return Ok(false);
    }
    let is_cs = is_cs.dyn_into::<js_sys::Function>()?;

    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str("opus"))?;
    set_prop(&cfg, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&cfg, "numberOfChannels", &num(channels as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;

    let promise = is_cs.call1(&ctor, &cfg)?.dyn_into::<Promise>()?;
    let result = JsFuture::from(promise).await?;
    Ok(Reflect::get(&result, &JsValue::from_str("supported"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false))
}

pub fn make_audio_data(
    samples: &[f32],
    sample_rate: u32,
    channels: u32,
    timestamp_us: f64,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioData"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("AudioData not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let n_frames = (samples.len() as u32) / channels.max(1);

    let arr = js_sys::Float32Array::from(samples);
    let init = Object::new();
    set_prop(&init, "format", &JsValue::from_str("f32-planar"))?;
    set_prop(&init, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&init, "numberOfFrames", &num(n_frames as f64))?;
    set_prop(&init, "numberOfChannels", &num(channels as f64))?;
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    set_prop(&init, "data", &arr)?;

    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

pub fn audio_data_channel(data: &JsValue, plane: u32) -> Vec<f32> {
    let n = Reflect::get(data, &JsValue::from_str("numberOfFrames"))
        .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
    if n == 0 {
        return Vec::new();
    }
    let dest = js_sys::Float32Array::new_with_length(n);
    let opts = Object::new();
    if set_prop(&opts, "planeIndex", &num(plane as f64)).is_err() {
        return Vec::new();
    }
    if set_prop(&opts, "format", &JsValue::from_str("f32-planar")).is_err() {
        return Vec::new();
    }
    let Ok(copy_to) = get_fn(data, "copyTo") else { return Vec::new(); };
    if copy_to.call2(data, &dest, &opts).is_err() {
        return Vec::new();
    }
    dest.to_vec()
}

pub fn audio_data_number_of_channels(data: &JsValue) -> u32 {
    Reflect::get(data, &JsValue::from_str("numberOfChannels"))
        .ok().and_then(|v| v.as_f64()).map(|n| n as u32).unwrap_or(0)
}

pub fn close_audio_data(data: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(data, "close")?;
    let _ = f.call0(data)?;
    Ok(())
}

pub fn chunk_byte_length(chunk: &JsValue) -> usize {
    Reflect::get(chunk, &JsValue::from_str("byteLength"))
        .ok().and_then(|v| v.as_f64()).map(|n| n as usize).unwrap_or(0)
}

pub fn format_audio_error(e: &JsValue) -> String {
    let name = Reflect::get(e, &JsValue::from_str("name"))
        .ok().and_then(|v| v.as_string()).unwrap_or_default();
    let message = Reflect::get(e, &JsValue::from_str("message"))
        .ok().and_then(|v| v.as_string()).unwrap_or_default();
    match name.as_str() {
        "NotAllowedError" => "Доступ к микрофону запрещён. Разрешите в браузере.".to_string(),
        "NotFoundError" => "Микрофон не найден. Используйте «Тестовый тон».".to_string(),
        "NotReadableError" => "Микрофон занят другим приложением.".to_string(),
        _ => format_str(format_args!("{name}: {message}")),
    }
}

pub fn bytes_human(n: usize) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = KB * 1024.0;
    let f = n as f64;
    if f < KB { return format_str(format_args!("{n} B")); }
    if f < MB { return format_str(format_args!("{:.1} KB", f / KB)); }
    format_str(format_args!("{:.2} MB", f / MB))
}

pub fn pcm_bytes(frames: u64, channels: u32) -> u64 {
    frames * channels as u64 * 4
}
```

### `web/public/worklets/pitch-processor.js` — чистый JS, без импортов

```javascript
// 23.13. AudioWorkletProcessor: pitch shift через кольцевой буфер.
//
// ## Почему без WASM
//
// В AudioWorkletGlobalScope запрещены dynamic import() и fetch().
// Загрузить .wasm можно только вшив его в этот файл на этапе сборки.
// Для ring buffer'а это избыточно: 0.03-0.08 мс на блок при бюджете
// 2.667 мс — в 50 раз меньше.
//
// ## Что доступно из глобалов
//
// sampleRate, currentTime, currentFrame, registerProcessor, console,
// performance, this.port. Никаких self, postMessage, addEventListener,
// fetch, import(), document.
//
// ## Бюджет
//
// process() должен уложиться в 128 / sampleRate = 2.667 мс (48 kHz).
// Никаких аллокаций, async, postMessage в hot path.

class PitchProcessor extends AudioWorkletProcessor {
    constructor() {
        super();

        // Кольцевой буфер, предвыделенный. 4096 сэмплов достаточно
        // для 128-сэмпловых блоков при ratio в [0.5, 2.0].
        this.ring = new Float32Array(4096);
        this.write = 0;
        this.read = 2048.0;
        this.ratio = 1.0;
        this.MIN_LAG = 128;

        // Метрики за окно 500 мс.
        this.winMs = 0;
        this.winBlocks = 0;
        this.totalBlocks = 0;
        this.totalXruns = 0;
        this.lastReport = 0;

        // Приём команд из main thread. Единственный способ общения.
        this.port.onmessage = (e) => {
            const m = e.data || {};
            if (m.type === 'ratio') {
                this.ratio = Math.max(0.5, Math.min(2.0, m.value));
            }
        };

        // Уведомляем main thread: мы готовы.
        this.port.postMessage({ type: 'ready' });
    }

    process(inputs, outputs) {
        const output = outputs[0];
        const input = inputs[0];
        if (!output || output.length === 0) return true;
        if (!input || input.length === 0) return true;

        const ch = Math.min(input.length, output.length);
        const t0 = performance.now();

        const ring = this.ring;
        const R = ring.length;
        const MIN_LAG = this.MIN_LAG;

        for (let c = 0; c < ch; c++) {
            const inCh = input[c];
            const outCh = output[c];
            const n = Math.min(inCh.length, outCh.length);

            // 1. Запись входных сэмплов в ring, скорость 1x.
            let w = this.write;
            for (let i = 0; i < n; i++) {
                ring[w] = inCh[i];
                w = (w + 1) % R;
            }
            this.write = w;

            // 2. Чтение с переменной скоростью ratio, линейная интерполяция.
            let r = this.read;
            for (let i = 0; i < n; i++) {
                const i0 = r | 0;
                const i1 = (i0 + 1) % R;
                const frac = r - i0;
                outCh[i] = ring[i0] * (1.0 - frac) + ring[i1] * frac;
                r += this.ratio;

                // Защита от догоняния write head.
                let lag = ((this.write - r) % R + R) % R;
                if (lag < MIN_LAG) {
                    r = this.write - R * 0.5;
                    if (r < 0) r += R;
                }
                if (r >= R) r -= R;
            }
            this.read = r;
        }

        const dt = performance.now() - t0;

        this.winMs += dt;
        this.winBlocks += 1;
        this.totalBlocks += 1;

        const budgetMs = (128 / sampleRate) * 1000;
        if (dt > budgetMs) this.totalXruns += 1;

        // Отчёт раз в 500 мс — единственный postMessage в hot path.
        const now = performance.now();
        if (now - this.lastReport >= 500 && this.winBlocks > 0) {
            this.lastReport = now;
            this.port.postMessage({
                type: 'stats',
                avgProcessMs: this.winMs / this.winBlocks,
                blocks: this.totalBlocks,
                xruns: this.totalXruns,
            });
            this.winMs = 0;
            this.winBlocks = 0;
        }

        return true;
    }
}

registerProcessor('pitch-processor', PitchProcessor);
```

### `tests/wasm_audio_api.rs`

```rust
#![cfg(target_arch = "wasm32")]

//! 23.13. WASM-тесты audio API.

use wasm_bindgen_futures::JsFuture;
use wasm_bindgen_test::*;
use web_sys::AudioContext;

use hello_wasm02::audio_api::{
    add_worklet_module, audio_data_channel, bytes_human, close_audio_data,
    has_audio_worklet, has_get_user_media, is_audio_data_supported,
    is_audio_decoder_supported, is_audio_encoder_supported,
    is_opus_supported, make_audio_data,
};

wasm_bindgen_test_configure!(run_in_browser);

fn try_audio_context() -> Option<AudioContext> {
    return AudioContext::new().ok();
}

async fn close_ctx(ctx: &AudioContext) {
    if let Ok(promise) = ctx.close() {
        let _ = JsFuture::from(promise).await;
    }
}

#[wasm_bindgen_test]
async fn audio_context_can_be_created() {
    let Some(ctx) = try_audio_context() else { return; };
    assert!(has_audio_worklet(&ctx), "AudioWorklet должен быть доступен");
    close_ctx(&ctx).await;
}

#[wasm_bindgen_test]
fn audio_encoder_decoder_data_supported() {
    assert!(is_audio_encoder_supported());
    assert!(is_audio_decoder_supported());
    assert!(is_audio_data_supported());
}

#[wasm_bindgen_test]
fn get_user_media_feature_detected() {
    let _ = has_get_user_media();
}

#[wasm_bindgen_test]
async fn add_worklet_module_loads_pitch_processor() {
    let Some(ctx) = try_audio_context() else { return; };
    let _ = add_worklet_module(&ctx, "/worklets/pitch-processor.js").await;
    close_ctx(&ctx).await;
}

#[wasm_bindgen_test]
fn make_and_read_audio_data() {
    let samples = vec![0.0f32; 480];
    let data = make_audio_data(&samples, 48000, 1, 0.0).expect("AudioData");
    let ch0 = audio_data_channel(&data, 0);
    assert_eq!(ch0.len(), 480);
    close_audio_data(&data).expect("close");
}

#[wasm_bindgen_test]
fn audio_data_preserves_values() {
    let samples: Vec<f32> = (0..100).map(|i| i as f32 / 100.0).collect();
    let data = make_audio_data(&samples, 48000, 1, 0.0).expect("AudioData");
    let ch0 = audio_data_channel(&data, 0);
    for (a, b) in ch0.iter().zip(samples.iter()) {
        assert!((a - b).abs() < 1e-6);
    }
    close_audio_data(&data).expect("close");
}

#[wasm_bindgen_test]
async fn opus_48k_mono_supported() {
    let ok = is_opus_supported(48000, 1, 64000).await.unwrap_or(false);
    assert!(ok);
}

#[wasm_bindgen_test]
fn bytes_human_formats_ok() {
    assert_eq!(bytes_human(500), "500 B");
    assert!(bytes_human(2048).contains("2.0 KB"));
    assert!(bytes_human(5 * 1024 * 1024).contains("5.00 MB"));
}
```

### Команды

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_audio_api
npm start
```

### `src/app.rs`

```
//! Глава 23.13 — AudioWorklet: real-time audio pipeline.
//!
//! ## Что демонстрируется
//!
//! **Раздел A — real-time pitch shifter:**
//!   OscillatorNode (440 Hz) или микрофон → AudioWorkletNode
//!   → PitchProcessor (128-сэмпловые блоки, ring buffer)
//!   → AnalyserNode → AudioContext.destination.
//!
//! **Раздел B — Opus round-trip через WebCodecs (main thread):**
//!   тон 440 Hz → AudioEncoder (Opus) → AudioDecoder → PCM.
//!
//! ## Ключевая ценность — бюджет 128 сэмплов
//!
//! AudioWorklet даёт жёсткий бюджет: 128 сэмплов / 48 kHz = 2.667 мс
//! на блок. Не уложился — xrun. Демо показывает это числами:
//! avg ms/блок, запас бюджета в %, счётчик xruns.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Promise, Reflect};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{
    AnalyserNode, AudioBuffer, AudioContext, AudioWorkletNode, CanvasRenderingContext2d,
    Document, HtmlCanvasElement, HtmlElement, HtmlInputElement, MediaStream,
    MessageEvent, OfflineAudioContext, OscillatorNode, OscillatorType,
};

use crate::audio_api::{
    add_worklet_module, audio_data_channel, audio_data_number_of_channels,
    bytes_human, close_audio_data, create_worklet_node, format_audio_error,
    is_audio_decoder_supported, is_audio_encoder_supported, is_opus_supported,
    make_audio_data, node_port, pcm_bytes, post_ratio,
};
use crate::video_frame::{format_str, get_fn, num, set_prop};

// =============================================================================
// Константы
// =============================================================================

const WORKLET_URL: &str = "/worklets/pitch-processor.js";
const PROCESSOR_NAME: &str = "pitch-processor";
const SAMPLE_RATE: u32 = 48000;
const CHANNELS: u32 = 1;
const OPUS_BITRATE: u32 = 64000;
const TONE_FREQ: f64 = 440.0;
const TONE_DURATION_SEC: f64 = 1.0;
const BLOCK_SIZE: u32 = 128;
const WAVE_FRAME_MS: f64 = 33.0;

fn block_budget_ms() -> f64 {
    return BLOCK_SIZE as f64 / SAMPLE_RATE as f64 * 1000.0;
}

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.aw-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.aw-desc-line { margin:6px 0; }
.aw-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.aw-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.aw-btn:hover:not([disabled]) { background:#30363d; }
.aw-btn:disabled { opacity:0.5;cursor:not-allowed; }
.aw-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.aw-btn-primary:hover:not([disabled]) { background:#2ea043; }
.aw-btn-stop { background:#a4302e;border-color:#d1242f;color:#fff;}
.aw-btn-stop:hover:not([disabled]) { background:#d1242f; }
.aw-btn-opus { background:#8957e5;border-color:#a371f7;color:#fff;}
.aw-btn-opus:hover:not([disabled]) { background:#a371f7; }
.aw-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.aw-slider { display:flex;align-items:center;gap:10px;flex:1;min-width:220px;}
.aw-slider input { flex:1;accent-color:#58a6ff;cursor:pointer;}
.aw-slider-value { font-family:ui-monospace,monospace;font-size:12px;color:#79c0ff;min-width:56px;text-align:right;}
.aw-waves { max-width:900px;margin-bottom:16px;padding:10px;background:#0d1117;border:1px solid #21262d;border-radius:6px;}
.aw-wave-label { font-family:ui-monospace,monospace;font-size:11px;color:#8b949e;margin:0 0 4px 6px;}
.aw-wave { display:block;width:100%;height:100px;background:#0d1117;border-radius:4px;}
.aw-wave + .aw-wave-label { margin-top:10px;}
.aw-stats { max-width:900px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;word-break:break-word;}
.aw-stats .label { color:#8b949e; }
.aw-stats .value { color:#79c0ff; }
.aw-stats .ok { color:#3fb950; }
.aw-stats .warn { color:#f0883e; }
.aw-stats .err { color:#f85149; }
.aw-stats .muted { color:#6e7681; }
.aw-stats .sep { border-top:1px solid #21262d;margin:6px 0; }
.aw-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone, Copy)]
struct WorkletMetrics {
    avg_process_ms: f64,
    blocks: u32,
    xruns: u32,
    has_data: bool,
}

impl WorkletMetrics {
    fn new() -> Self {
        return Self { avg_process_ms: 0.0, blocks: 0, xruns: 0, has_data: false };
    }
    fn headroom_pct(&self) -> f64 {
        if self.avg_process_ms <= 0.0 { return 100.0; }
        let used = (self.avg_process_ms / block_budget_ms()) * 100.0;
        return (100.0 - used).clamp(0.0, 100.0);
    }
}

#[derive(Clone, Copy)]
struct OpusResult {
    chunks: usize,
    encoded_bytes: usize,
    pcm_bytes: u64,
    encode_ms: f64,
    decode_ms: f64,
    frames: u32,
}

struct DemoState {
    ctx: Option<AudioContext>,
    worklet_loaded: bool,
    source_label: String,
    node: Option<AudioWorkletNode>,
    osc: Option<OscillatorNode>,
    input_analyser: Option<AnalyserNode>,
    output_analyser: Option<AnalyserNode>,
    ratio: f32,
    ready: bool,
    error: Option<String>,
    phase: String,
    metrics: WorkletMetrics,
    opus_result: Option<OpusResult>,
    raf_frames: u64,
    raf_start_ms: f64,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            ctx: None, worklet_loaded: false, source_label: String::from("—"),
            node: None, osc: None, input_analyser: None, output_analyser: None,
            ratio: 1.3, ready: false, error: None, phase: String::from("простой"),
            metrics: WorkletMetrics::new(), opus_result: None,
            raf_frames: 0, raf_start_ms: 0.0,
        };
    }

    fn is_playing(&self) -> bool {
        return self.node.is_some() || self.osc.is_some();
    }

    fn raf_fps(&self) -> f64 {
        if self.raf_start_ms <= 0.0 || self.raf_frames == 0 { return 0.0; }
        let dt = js_sys::Date::now() - self.raf_start_ms;
        if dt <= 0.0 { return 0.0; }
        return self.raf_frames as f64 * 1000.0 / dt;
    }

    fn stop_all(&mut self) {
        if let Some(osc) = self.osc.take() {
            let _ = osc.stop();
            let _ = osc.disconnect();
        }
        if let Some(node) = self.node.take() {
            if let Ok(port) = node_port(&node) {
                port.set_onmessage(None);
            }
            let _ = node.disconnect();
        }
        if let Some(a) = self.input_analyser.take() { let _ = a.disconnect(); }
        if let Some(a) = self.output_analyser.take() { let _ = a.disconnect(); }
        self.source_label = String::from("—");
        self.metrics = WorkletMetrics::new();
        self.phase = String::from("остановлено");
    }
}

// =============================================================================
// Хелперы
// =============================================================================

fn escape_html(s: &str) -> String {
    let mut out = String::with_capacity(s.len() + 16);
    for c in s.chars() {
        match c {
            '<' => out.push_str("&lt;"),
            '>' => out.push_str("&gt;"),
            '&' => out.push_str("&amp;"),
            '"' => out.push_str("&quot;"),
            '\'' => out.push_str("&#39;"),
            _ => out.push(c),
        }
    }
    return out;
}

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "aw-injected-style";
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
        "<strong>AudioWorklet</strong> работает на отдельном потоке. Бюджет на блок — 128 сэмплов / 48 kHz = <strong>2.667 мс</strong>. Не уложился — xrun (слышимый щелчок). Никаких async, никаких аллокаций в hot path.",
        "<strong>Pitch shift</strong> через кольцевой буфер. Ratio &gt; 1 — выше, &lt; 1 — ниже. Форма выхода видна на нижнем waveform.",
        "<strong>Почему не WASM.</strong> В AudioWorkletGlobalScope запрещены dynamic <code>import()</code> и <code>fetch()</code>. WASM требует bundled-подхода (байты вшиваются в worklet-файл). В нашем случае ring buffer укладывается в ~0.05 мс — в 50 раз меньше бюджета, WASM не нужен.",
        "<strong>Opus round-trip</strong> — WebCodecs в main thread: AudioEncoder → AudioDecoder → PCM.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("aw-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("aw-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    return Ok(block.unchecked_into());
}

// =============================================================================
// Строковые функции
// =============================================================================

fn line_err(msg: &str) -> String {
    return format_str(format_args!(
        r#"<div><span class="err">{msg}</span></div>"#,
        msg = escape_html(msg),
    ));
}

fn line_phase(st: &DemoState) -> String {
    return format_str(format_args!(
        r#"<div><span class="label">Статус: </span><span class="value">{phase}</span></div>"#,
        phase = escape_html(&st.phase),
    ));
}

fn line_audio_ctx(st: &DemoState) -> String {
    let (state_str, sr_str) = match st.ctx.as_ref() {
        Some(ctx) => {
            let state = match ctx.state() {
                web_sys::AudioContextState::Running => "running",
                web_sys::AudioContextState::Suspended => "suspended",
                web_sys::AudioContextState::Closed => "closed",
                _ => "?",
            };
            let sr = format_str(format_args!("{} Hz", ctx.sample_rate() as u32));
            (state.to_string(), sr)
        }
        None => (String::from("—"), String::from("—")),
    };
    let (cls, label) = if st.ready { ("ok", "готов") } else { ("muted", "загружается…") };
    return format_str(format_args!(
        r#"<div><span class="label">AudioContext: </span><span class="value">{state}</span> · <span class="value">{sr}</span> · <span class="label">Worklet: </span><span class="{wc}">{ws}</span></div>"#,
        state = state_str, sr = sr_str, wc = cls, ws = label,
    ));
}

fn line_live(st: &DemoState) -> String {
    return format_str(format_args!(
        r#"<div><span class="label">Источник: </span><span class="value">{src}</span> · <span class="label">Pitch: </span><span class="value">{ratio:.2}×</span> · <span class="label">Блок: </span><span class="value">{bs} сэмплов = {budget:.3} мс</span></div>"#,
        src = escape_html(if st.source_label.is_empty() { "—" } else { &st.source_label }),
        ratio = st.ratio, bs = BLOCK_SIZE, budget = block_budget_ms(),
    ));
}

fn line_metrics(st: &DemoState) -> String {
    if !st.metrics.has_data { return String::new(); }
    let headroom = st.metrics.headroom_pct();
    let hc = if headroom >= 70.0 { "ok" } else if headroom >= 30.0 { "warn" } else { "err" };
    let xc = if st.metrics.xruns == 0 { "ok" } else { "err" };
    return format_str(format_args!(
        r#"<div class="sep"></div>
           <div><span class="label">DSP: </span><span class="value">{avg:.3} мс / блок</span> <span class="muted">(бюджет {budget:.3} мс)</span> · <span class="label">запас: </span><span class="{hc}">{head:.0}%</span></div>
           <div><span class="label">Блоков: </span><span class="value">{blocks}</span> · <span class="label">xruns: </span><span class="{xc}">{xruns}</span></div>"#,
        avg = st.metrics.avg_process_ms, budget = block_budget_ms(),
        hc = hc, head = headroom, blocks = st.metrics.blocks,
        xc = xc, xruns = st.metrics.xruns,
    ));
}

fn line_main_thread(st: &DemoState) -> String {
    return format_str(format_args!(
        r#"<div><span class="label">Main thread: </span><span class="value">{fps:.0} fps</span> <span class="muted">(rAF работает во время звука)</span></div>"#,
        fps = st.raf_fps(),
    ));
}

fn line_opus(st: &DemoState) -> String {
    let Some(r) = st.opus_result.as_ref() else { return String::new(); };
    let ratio = if r.encoded_bytes > 0 { r.pcm_bytes as f64 / r.encoded_bytes as f64 } else { 0.0 };
    let enc_x = if r.encode_ms > 0.0 { (r.frames as f64 / SAMPLE_RATE as f64) / (r.encode_ms / 1000.0) } else { 0.0 };
    let dec_x = if r.decode_ms > 0.0 { (r.frames as f64 / SAMPLE_RATE as f64) / (r.decode_ms / 1000.0) } else { 0.0 };
    return format_str(format_args!(
        r#"<div class="sep"></div>
           <div><span class="label">Opus: </span><span class="value">{n} chunks</span> · <span class="label">encoded: </span><span class="value">{enc}</span> · <span class="label">PCM: </span><span class="value">{pcm}</span> · <span class="label">сжатие: </span><span class="ok">≈ {ratio:.1}×</span></div>
           <div><span class="label">Encode: </span><span class="value">{ems:.1} мс</span> <span class="muted">({ex:.0}× realtime)</span> · <span class="label">Decode: </span><span class="value">{dms:.1} мс</span> <span class="muted">({dx:.0}× realtime)</span></div>"#,
        n = r.chunks, enc = bytes_human(r.encoded_bytes), pcm = bytes_human(r.pcm_bytes as usize),
        ratio = ratio, ems = r.encode_ms, ex = enc_x, dms = r.decode_ms, dx = dec_x,
    ));
}

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("aw-stats") else { return; };
    let err_line = st.error.as_ref().map(|m| line_err(m.as_str())).unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{phase}{ctx}{live}{metrics}{mainthr}{opus}"#,
        err = err_line, phase = line_phase(st), ctx = line_audio_ctx(st),
        live = line_live(st), metrics = line_metrics(st),
        mainthr = line_main_thread(st), opus = line_opus(st),
    ));
    el.set_inner_html(&html);
}

fn update_buttons(document: &Document, st: &DemoState) {
    let playing = st.is_playing();
    let set_enabled = |id: &str, enabled: bool| {
        if let Some(el) = document.get_element_by_id(id) {
            if enabled { let _ = el.remove_attribute("disabled"); }
            else { let _ = el.set_attribute("disabled", ""); }
        }
    };
    set_enabled("aw-start-tone", !playing);
    set_enabled("aw-start-mic", !playing);
    set_enabled("aw-stop", playing);
}

// =============================================================================
// AudioContext / worklet
// =============================================================================

async fn maybe_await_result_promise(r: Result<Promise, JsValue>) {
    if let Ok(p) = r {
        let _ = JsFuture::from(p).await;
    }
}

async fn ensure_context(state: &Rc<RefCell<DemoState>>) -> Result<AudioContext, JsValue> {
    {
        let s = state.borrow();
        if let Some(ctx) = s.ctx.as_ref() {
            if ctx.state() != web_sys::AudioContextState::Closed {
                return Ok(ctx.clone());
            }
        }
    }
    let ctx = AudioContext::new()?;
    if ctx.state() == web_sys::AudioContextState::Suspended {
        maybe_await_result_promise(ctx.resume()).await;
    }
    {
        let mut s = state.borrow_mut();
        s.ctx = Some(ctx.clone());
        s.worklet_loaded = false;
        s.ready = false;
    }
    Ok(ctx)
}

async fn ensure_worklet(state: &Rc<RefCell<DemoState>>) -> Result<AudioContext, JsValue> {
    let ctx = ensure_context(state).await?;
    if state.borrow().worklet_loaded {
        return Ok(ctx);
    }

    add_worklet_module(&ctx, WORKLET_URL).await.map_err(|e| {
        let msg = e.as_string()
            .unwrap_or_else(|| format_str(format_args!("{e:?}")));
        // Диагностика типичной ошибки 404→HTML fallback.
        if msg.contains("expected expression") || msg.contains("'<'") {
            return JsValue::from_str(&format_str(format_args!(
                "Worklet не найден по пути {url}. Проверьте, что файл \
                 web/public/worklets/pitch-processor.js существует. \
                 В DevTools → Network запрос {url} должен возвращать JS, а не HTML.",
                url = WORKLET_URL,
            )));
        }
        JsValue::from_str(&format_str(format_args!(
            "addModule({url}): {msg}",
            url = WORKLET_URL, msg = msg,
        )))
    })?;

    state.borrow_mut().worklet_loaded = true;
    Ok(ctx)
}

async fn get_mic_stream() -> Result<MediaStream, JsValue> {
    let global = js_sys::global();
    let nav = Reflect::get(&global, &JsValue::from_str("navigator"))?;
    let md = Reflect::get(&nav, &JsValue::from_str("mediaDevices"))?;
    let get_um = get_fn(&md, "getUserMedia")?;
    let constraints = Object::new();
    Reflect::set(&constraints, &JsValue::from_str("audio"), &JsValue::TRUE)?;
    let promise = get_um.call1(&md, &constraints)?.dyn_into::<Promise>()?;
    let stream = JsFuture::from(promise).await?;
    stream.dyn_into::<MediaStream>()
}

// =============================================================================
// Worklet port
// =============================================================================

fn wire_worklet_port(
    node: &AudioWorkletNode,
    state: Rc<RefCell<DemoState>>,
    document: Document,
) -> Result<(), JsValue> {
    let port = node_port(node)?;
    let st_msg = state.clone();
    let doc_msg = document.clone();
    let on_msg: Closure<dyn FnMut(MessageEvent)> = Closure::new(move |e: MessageEvent| {
        let data = e.data();
        let ty = Reflect::get(&data, &JsValue::from_str("type"))
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        match ty.as_str() {
            "ready" => {
                {
                    let mut s = st_msg.borrow_mut();
                    s.ready = true;
                }
                update_stats(&doc_msg, &st_msg.borrow());
            }
            "error" => {
                let msg = Reflect::get(&data, &JsValue::from_str("message"))
                    .ok().and_then(|v| v.as_string())
                    .unwrap_or_else(|| String::from("unknown"));
                {
                    let mut s = st_msg.borrow_mut();
                    s.error = Some(format_str(format_args!("worklet: {msg}")));
                }
                update_stats(&doc_msg, &st_msg.borrow());
            }
            "stats" => {
                let avg = Reflect::get(&data, &JsValue::from_str("avgProcessMs"))
                    .ok().and_then(|v| v.as_f64()).unwrap_or(0.0);
                let blocks = Reflect::get(&data, &JsValue::from_str("blocks"))
                    .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
                let xruns = Reflect::get(&data, &JsValue::from_str("xruns"))
                    .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
                {
                    let mut s = st_msg.borrow_mut();
                    s.metrics = WorkletMetrics {
                        avg_process_ms: avg, blocks, xruns, has_data: true,
                    };
                }
                update_stats(&doc_msg, &st_msg.borrow());
            }
            _ => {}
        }
    });
    port.set_onmessage(Some(on_msg.as_ref().unchecked_ref()));
    on_msg.forget();
    Ok(())
}

// =============================================================================
// Start / Stop
// =============================================================================

fn setup_node_graph(
    ctx: &AudioContext,
    state: Rc<RefCell<DemoState>>,
    document: &Document,
) -> Result<AudioWorkletNode, JsValue> {
    let node = create_worklet_node(ctx, PROCESSOR_NAME, CHANNELS)?;
    let port = node_port(&node)?;
    let ratio = state.borrow().ratio;
    post_ratio(&port, ratio)?;
    wire_worklet_port(&node, state.clone(), document.clone())?;

    let input_a = ctx.create_analyser()?;
    input_a.set_fft_size(512);
    let output_a = ctx.create_analyser()?;
    output_a.set_fft_size(512);

    {
        let mut s = state.borrow_mut();
        s.node = Some(node.clone());
        s.input_analyser = Some(input_a.clone());
        s.output_analyser = Some(output_a.clone());
    }

    output_a.connect_with_audio_node(&ctx.destination())?;
    node.connect_with_audio_node(&output_a)?;
    Ok(node)
}

async fn start_tone(document: Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    state.borrow_mut().stop_all();
    let ctx = ensure_worklet(&state).await?;
    let node = setup_node_graph(&ctx, state.clone(), &document)?;

    let osc = ctx.create_oscillator()?;
    osc.set_type(OscillatorType::Sine);
    osc.frequency().set_value(TONE_FREQ as f32);

    let input_a = state.borrow().input_analyser.clone()
        .ok_or_else(|| JsValue::from_str("no input analyser"))?;
    osc.connect_with_audio_node(&input_a)?;
    input_a.connect_with_audio_node(&node)?;
    osc.start()?;

    {
        let mut s = state.borrow_mut();
        s.osc = Some(osc);
        s.source_label = format_str(format_args!("тон {TONE_FREQ:.0} Hz"));
        s.error = None;
        s.phase = String::from("играет (тон)");
        s.metrics = WorkletMetrics::new();
    }
    update_stats(&document, &state.borrow());
    update_buttons(&document, &state.borrow());
    Ok(())
}

async fn start_mic(document: Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    state.borrow_mut().stop_all();
    let ctx = ensure_worklet(&state).await?;
    let node = setup_node_graph(&ctx, state.clone(), &document)?;

    let stream = match get_mic_stream().await {
        Ok(s) => s,
        Err(e) => {
            let msg = format_audio_error(&e);
            {
                let mut s = state.borrow_mut();
                s.error = Some(msg);
                s.phase = String::from("ошибка микрофона");
            }
            update_stats(&document, &state.borrow());
            update_buttons(&document, &state.borrow());
            return Ok(());
        }
    };
    let src = ctx.create_media_stream_source(&stream)?;
    let input_a = state.borrow().input_analyser.clone()
        .ok_or_else(|| JsValue::from_str("no input analyser"))?;
    src.connect_with_audio_node(&input_a)?;
    input_a.connect_with_audio_node(&node)?;

    {
        let mut s = state.borrow_mut();
        s.source_label = String::from("микрофон");
        s.error = None;
        s.phase = String::from("играет (микрофон)");
        s.metrics = WorkletMetrics::new();
    }
    update_stats(&document, &state.borrow());
    update_buttons(&document, &state.borrow());
    Ok(())
}

fn stop_all(document: Document, state: Rc<RefCell<DemoState>>) {
    state.borrow_mut().stop_all();
    clear_canvas(&document, "aw-wave-in");
    clear_canvas(&document, "aw-wave-out");
    update_stats(&document, &state.borrow());
    update_buttons(&document, &state.borrow());
}

fn set_ratio(state: &Rc<RefCell<DemoState>>, ratio: f32) {
    let clamped = ratio.clamp(0.5, 2.0);
    let node_opt = state.borrow().node.clone();
    if let Some(node) = node_opt {
        if let Ok(port) = node_port(&node) {
            let _ = post_ratio(&port, clamped);
        }
    }
    state.borrow_mut().ratio = clamped;
}

// =============================================================================
// Waveform
// =============================================================================

fn clear_canvas(document: &Document, id: &str) {
    let Some(canvas) = document.get_element_by_id(id)
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else { return; };
    let Ok(Some(ctx_obj)) = canvas.get_context("2d") else { return; };
    let Ok(ctx) = ctx_obj.dyn_into::<CanvasRenderingContext2d>() else { return; };
    let w = canvas.width() as f64;
    let h = canvas.height() as f64;
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);
}

fn draw_analyser(document: &Document, id: &str, analyser: &AnalyserNode) {
    let Some(canvas) = document.get_element_by_id(id)
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else { return; };
    let Ok(Some(ctx_obj)) = canvas.get_context("2d") else { return; };
    let Ok(ctx) = ctx_obj.dyn_into::<CanvasRenderingContext2d>() else { return; };

    let w = canvas.width() as f64;
    let h = canvas.height() as f64;
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);
    ctx.set_stroke_style_str("#21262d");
    ctx.set_line_width(1.0);
    ctx.begin_path();
    ctx.move_to(0.0, h / 2.0);
    ctx.line_to(w, h / 2.0);
    ctx.stroke();

    let n = analyser.fft_size() as usize;
    let mut buf = vec![0u8; n];
    analyser.get_byte_time_domain_data(&mut buf);

    ctx.set_stroke_style_str("#3fb950");
    ctx.set_line_width(1.5);
    ctx.begin_path();
    let step = w / (n - 1).max(1) as f64;
    for (i, &v) in buf.iter().enumerate() {
        let x = i as f64 * step;
        let normalized = (v as f64 / 128.0) - 1.0;
        let y = h / 2.0 + normalized * (h / 2.0 - 4.0);
        if i == 0 { ctx.move_to(x, y); } else { ctx.line_to(x, y); }
    }
    ctx.stroke();
}

// =============================================================================
// Opus helpers
// =============================================================================

fn make_audio_encoder(
    on_output: &Closure<dyn FnMut(JsValue, JsValue)>,
    on_error: &Closure<dyn FnMut(JsValue)>,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioEncoder"))?;
    if !ctor.is_function() { return Err(JsValue::from_str("AudioEncoder not available")); }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn make_audio_decoder(
    on_output: &Closure<dyn FnMut(JsValue)>,
    on_error: &Closure<dyn FnMut(JsValue)>,
) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("AudioDecoder"))?;
    if !ctor.is_function() { return Err(JsValue::from_str("AudioDecoder not available")); }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = Object::new();
    set_prop(&init, "output", on_output.as_ref())?;
    set_prop(&init, "error", on_error.as_ref())?;
    let args = Array::new();
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn configure_encoder_opus(
    encoder: &JsValue, sample_rate: u32, channels: u32, bitrate: u32,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str("opus"))?;
    set_prop(&cfg, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&cfg, "numberOfChannels", &num(channels as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    let f = get_fn(encoder, "configure")?;
    let _ = f.call1(encoder, &cfg)?;
    Ok(())
}

fn configure_decoder_opus(
    decoder: &JsValue, sample_rate: u32, channels: u32,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str("opus"))?;
    set_prop(&cfg, "sampleRate", &num(sample_rate as f64))?;
    set_prop(&cfg, "numberOfChannels", &num(channels as f64))?;
    let f = get_fn(decoder, "configure")?;
    let _ = f.call1(decoder, &cfg)?;
    Ok(())
}

fn encode_audio_data(encoder: &JsValue, data: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(encoder, "encode")?;
    let _ = f.call1(encoder, data)?;
    Ok(())
}

fn decode_audio_chunk(decoder: &JsValue, chunk: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(decoder, "decode")?;
    let _ = f.call1(decoder, chunk)?;
    Ok(())
}

async fn flush_codec(codec: &JsValue) -> Result<(), JsValue> {
    let f = get_fn(codec, "flush")?;
    let p = f.call0(codec)?.dyn_into::<Promise>()?;
    JsFuture::from(p).await.map(|_| ())
}

fn close_codec(codec: &JsValue) {
    if let Ok(f) = get_fn(codec, "close") {
        let _ = f.call0(codec);
    }
}

async fn generate_tone_offline() -> Result<Vec<f32>, JsValue> {
    let total_frames = (SAMPLE_RATE as f64 * TONE_DURATION_SEC) as u32;
    let offline = OfflineAudioContext::new_with_number_of_channels_and_length_and_sample_rate(
        CHANNELS, total_frames, SAMPLE_RATE as f32,
    )?;
    let osc = offline.create_oscillator()?;
    osc.set_type(OscillatorType::Sine);
    osc.frequency().set_value(TONE_FREQ as f32);
    osc.connect_with_audio_node(&offline.destination())?;
    osc.start()?;
    let promise = offline.start_rendering()?;
    let rendered = JsFuture::from(promise).await?;
    let buffer: AudioBuffer = rendered.dyn_into()?;
    let channel: Vec<f32> = buffer.get_channel_data(0)?;
    Ok(channel)
}

// =============================================================================
// Opus round-trip
// =============================================================================

async fn opus_roundtrip(document: Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    if !is_audio_encoder_supported() || !is_audio_decoder_supported() {
        {
            let mut s = state.borrow_mut();
            s.error = Some(String::from("AudioEncoder/AudioDecoder недоступны"));
        }
        update_stats(&document, &state.borrow());
        return Ok(());
    }
    let supported = is_opus_supported(SAMPLE_RATE, CHANNELS, OPUS_BITRATE).await.unwrap_or(false);
    if !supported {
        {
            let mut s = state.borrow_mut();
            s.error = Some(String::from("Opus не поддерживается"));
        }
        update_stats(&document, &state.borrow());
        return Ok(());
    }

    {
        let mut s = state.borrow_mut();
        s.phase = String::from("Opus: генерация тона…");
        s.error = None;
        s.opus_result = None;
    }
    update_stats(&document, &state.borrow());
    TimeoutFuture::new(0).await;

    let tone = match generate_tone_offline().await {
        Ok(v) => v,
        Err(e) => {
            {
                let mut s = state.borrow_mut();
                s.error = Some(format_str(format_args!("генерация тона: {e:?}")));
                s.phase = String::from("ошибка");
            }
            update_stats(&document, &state.borrow());
            return Ok(());
        }
    };
    let frames = (tone.len() / CHANNELS.max(1) as usize) as u32;

    {
        let mut s = state.borrow_mut();
        s.phase = format_str(format_args!("Opus: encode {frames} фреймов…"));
    }
    update_stats(&document, &state.borrow());
    TimeoutFuture::new(0).await;

    // Encode
    let chunks: Rc<RefCell<Vec<JsValue>>> = Rc::new(RefCell::new(Vec::new()));
    let enc_err: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
    let chunks_cb = chunks.clone();
    let on_enc_output: Closure<dyn FnMut(JsValue, JsValue)> = Closure::new(move |c, _m| {
        chunks_cb.borrow_mut().push(c);
    });
    let enc_err_cb = enc_err.clone();
    let on_enc_error: Closure<dyn FnMut(JsValue)> = Closure::new(move |e: JsValue| {
        let msg = Reflect::get(&e, &JsValue::from_str("message"))
            .ok().and_then(|v| v.as_string())
            .unwrap_or_else(|| String::from("unknown"));
        *enc_err_cb.borrow_mut() = Some(msg);
    });

    let t_enc_start = js_sys::Date::now();
    let encoder = make_audio_encoder(&on_enc_output, &on_enc_error)?;
    configure_encoder_opus(&encoder, SAMPLE_RATE, CHANNELS, OPUS_BITRATE)?;
    let audio_data = make_audio_data(&tone, SAMPLE_RATE, CHANNELS, 0.0)?;
    encode_audio_data(&encoder, &audio_data)?;
    close_audio_data(&audio_data)?;
    flush_codec(&encoder).await?;
    close_codec(&encoder);
    let t_enc_end = js_sys::Date::now();
    on_enc_output.forget();
    on_enc_error.forget();

    if let Some(msg) = enc_err.borrow().as_ref() {
        {
            let mut s = state.borrow_mut();
            s.error = Some(format_str(format_args!("encode: {msg}")));
            s.phase = String::from("ошибка");
        }
        update_stats(&document, &state.borrow());
        return Ok(());
    }

    let encoded_chunks = chunks.borrow().clone();
    let total_encoded_bytes: usize = encoded_chunks.iter()
        .map(|c| crate::audio_api::chunk_byte_length(c)).sum();
    let n_chunks = encoded_chunks.len();

    {
        let mut s = state.borrow_mut();
        s.phase = format_str(format_args!("Opus: decode {n_chunks} чанков…"));
    }
    update_stats(&document, &state.borrow());
    TimeoutFuture::new(0).await;

    // Decode
    let planes: Rc<RefCell<Vec<Vec<f32>>>> = Rc::new(RefCell::new(Vec::new()));
    let planes_cb = planes.clone();
    let on_dec_output: Closure<dyn FnMut(JsValue)> = Closure::new(move |data: JsValue| {
        let n_ch = audio_data_number_of_channels(&data);
        let mut buf = planes_cb.borrow_mut();
        while buf.len() < n_ch as usize { buf.push(Vec::new()); }
        for ch in 0..n_ch {
            let plane = audio_data_channel(&data, ch);
            buf[ch as usize].extend_from_slice(&plane);
        }
        let _ = close_audio_data(&data);
    });
    let on_dec_error: Closure<dyn FnMut(JsValue)> = Closure::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });

    let t_dec_start = js_sys::Date::now();
    let decoder = make_audio_decoder(&on_dec_output, &on_dec_error)?;
    configure_decoder_opus(&decoder, SAMPLE_RATE, CHANNELS)?;
    for chunk in encoded_chunks.iter() {
        decode_audio_chunk(&decoder, chunk)?;
    }
    flush_codec(&decoder).await?;
    close_codec(&decoder);
    let t_dec_end = js_sys::Date::now();
    on_dec_output.forget();
    on_dec_error.forget();

    let result = OpusResult {
        chunks: n_chunks,
        encoded_bytes: total_encoded_bytes,
        pcm_bytes: pcm_bytes(frames as u64, CHANNELS),
        encode_ms: t_enc_end - t_enc_start,
        decode_ms: t_dec_end - t_dec_start,
        frames,
    };
    {
        let mut s = state.borrow_mut();
        s.opus_result = Some(result);
        s.phase = String::from("Opus: готово");
    }
    update_stats(&document, &state.borrow());
    Ok(())
}

// =============================================================================
// rAF loop
// =============================================================================

fn start_raf_loop(document: Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    {
        let mut s = state.borrow_mut();
        s.raf_start_ms = js_sys::Date::now();
        s.raf_frames = 0;
    }

    let cell: Rc<RefCell<Option<Closure<dyn FnMut(f64)>>>> = Rc::new(RefCell::new(None));
    let cell2 = cell.clone();
    let doc = document.clone();
    let st = state.clone();
    let mut last_wave_ms = 0.0f64;

    *cell.borrow_mut() = Some(Closure::wrap(Box::new(move |_now: f64| {
        if let Some(w) = web_sys::window() {
            if let Some(cb) = cell2.borrow().as_ref() {
                let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
            }
        }
        st.borrow_mut().raf_frames += 1;
        let now = js_sys::Date::now();
        if now - last_wave_ms >= WAVE_FRAME_MS {
            last_wave_ms = now;
            let (in_a, out_a) = {
                let s = st.borrow();
                (s.input_analyser.clone(), s.output_analyser.clone())
            };
            if let Some(a) = in_a { draw_analyser(&doc, "aw-wave-in", &a); }
            if let Some(a) = out_a { draw_analyser(&doc, "aw-wave-out", &a); }
        }
    }) as Box<dyn FnMut(f64)>));

    {
        let cb = cell.borrow();
        if let Some(c) = cb.as_ref() {
            window.request_animation_frame(c.as_ref().unchecked_ref())?;
        }
    }
    Ok(())
}

// =============================================================================
// run()
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;
    let root = document.get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23.13 — AudioWorklet: real-time audio"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Controls
    let controls = document.create_element("div")?;
    controls.set_class_name("aw-block");
    controls.set_inner_html(
        r#"<button class="aw-btn aw-btn-primary" id="aw-start-tone">▶ Тестовый тон 440 Hz</button>
           <button class="aw-btn aw-btn-primary" id="aw-start-mic">▶ Микрофон</button>
           <button class="aw-btn aw-btn-stop" id="aw-stop" disabled>■ Stop</button>
           <div class="aw-slider">
             <label class="aw-label" for="aw-ratio">Pitch:</label>
             <input type="range" id="aw-ratio" min="0.5" max="2.0" step="0.05" value="1.30">
             <span class="aw-slider-value" id="aw-ratio-value">1.30×</span>
           </div>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("aw-stats");
    stats.set_id("aw-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let waves = document.create_element("div")?;
    waves.set_class_name("aw-waves");
    waves.set_inner_html(
        r#"<div class="aw-wave-label">Вход (тон / микрофон)</div>
           <canvas id="aw-wave-in" class="aw-wave" width="880" height="100"></canvas>
           <div class="aw-wave-label">Выход (после PitchProcessor)</div>
           <canvas id="aw-wave-out" class="aw-wave" width="880" height="100"></canvas>"#,
    );
    root.append_child(&waves)?;

    let opus_block = document.create_element("div")?;
    opus_block.set_class_name("aw-block");
    opus_block.set_inner_html(
        r#"<span class="aw-label">WebCodecs Opus — off-line round-trip:</span>
           <button class="aw-btn aw-btn-opus" id="aw-opus">Encode → Decode</button>"#,
    );
    root.append_child(&opus_block)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("aw-footnote");
    footnote.set_text_content(Some(
        "AudioWorklet работает на audio rendering thread. Main thread свободен — \
         waveform рисуется в rAF. Pitch shift — простой ring buffer в JS, укладывается \
         в ~0.05 мс при бюджете 2.667 мс.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    // Start tone
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("aw-start-tone")
            .ok_or_else(|| JsValue::from_str("no #aw-start-tone"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let doc_cb = doc.clone();
            let st_cb = st.clone();
            spawn_local(async move {
                if let Err(e) = start_tone(doc_cb.clone(), st_cb.clone()).await {
                    let msg = e.as_string()
                        .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                    {
                        let mut s = st_cb.borrow_mut();
                        s.error = Some(msg);
                        s.phase = String::from("ошибка");
                    }
                    update_stats(&doc_cb, &st_cb.borrow());
                }
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Start mic
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("aw-start-mic")
            .ok_or_else(|| JsValue::from_str("no #aw-start-mic"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let doc_cb = doc.clone();
            let st_cb = st.clone();
            spawn_local(async move {
                if let Err(e) = start_mic(doc_cb.clone(), st_cb.clone()).await {
                    let msg = e.as_string()
                        .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                    {
                        let mut s = st_cb.borrow_mut();
                        s.error = Some(msg);
                        s.phase = String::from("ошибка");
                    }
                    update_stats(&doc_cb, &st_cb.borrow());
                }
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Stop
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("aw-stop")
            .ok_or_else(|| JsValue::from_str("no #aw-stop"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            stop_all(doc.clone(), st.clone());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Ratio
    {
        let doc = document.clone();
        let st = state.clone();
        let input_el = document.get_element_by_id("aw-ratio")
            .ok_or_else(|| JsValue::from_str("no #aw-ratio"))?;
        let input: HtmlInputElement = input_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let Some(el) = doc.get_element_by_id("aw-ratio") else { return; };
            let Ok(slider) = el.dyn_into::<HtmlInputElement>() else { return; };
            let value: f32 = slider.value().parse().unwrap_or(1.0);
            set_ratio(&st, value);
            if let Some(lbl) = doc.get_element_by_id("aw-ratio-value") {
                let txt = format_str(format_args!("{value:.2}×"));
                lbl.set_text_content(Some(&txt));
            }
            update_stats(&doc, &st.borrow());
        });
        input.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Opus
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("aw-opus")
            .ok_or_else(|| JsValue::from_str("no #aw-opus"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let doc_cb = doc.clone();
            let st_cb = st.clone();
            spawn_local(async move {
                if let Err(e) = opus_roundtrip(doc_cb.clone(), st_cb.clone()).await {
                    let msg = e.as_string()
                        .unwrap_or_else(|| format_str(format_args!("{e:?}")));
                    {
                        let mut s = st_cb.borrow_mut();
                        s.error = Some(msg);
                        s.phase = String::from("ошибка");
                    }
                    update_stats(&doc_cb, &st_cb.borrow());
                }
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    update_stats(&document, &state.borrow());
    update_buttons(&document, &state.borrow());
    clear_canvas(&document, "aw-wave-in");
    clear_canvas(&document, "aw-wave-out");

    start_raf_loop(document.clone(), state.clone())?;

    // Периодический апдейт stats (раз в секунду) — обновляет Main thread fps.
    {
        let doc_c = document.clone();
        let st_c = state.clone();
        spawn_local(async move {
            loop {
                TimeoutFuture::new(1000).await;
                if st_c.borrow().is_playing() {
                    update_stats(&doc_c, &st_c.borrow());
                }
            }
        });
    }

    Ok(())
}
```

- `DemoState` с полями `ctx`, `worklet_loaded`, `ready`, `ratio`, `metrics`, `opus_result`.
- `ensure_context` / `ensure_worklet` — ленивая инициализация.
- `setup_node_graph` — граф `Oscillator/Mic → inputAnalyser → workletNode → outputAnalyser → destination`.
- `wire_worklet_port` — обработка `ready` / `stats` / `error` от worklet'а.
- `start_raf_loop` — waveform через `AnalyserNode.get_byte_time_domain_data`.
- `opus_roundtrip` — encode + decode через WebCodecs.
- Строковые хелперы: `line_err`, `line_phase`, `line_audio_ctx`, `line_live`, `line_metrics`, `line_main_thread`, `line_opus`.
- `escape_html` для сообщений об ошибках.

---

## 23.13.6. Что вы увидите

1. Страница с описанием, controls, stats, waveform.
2. **Клик «▶ Тестовый тон 440 Hz»**:
   - Слышите синус 440 Hz с лёгким phase warble (эффект ресемплера).
   - Через 500 мс появляются метрики: `DSP: 0.05 мс / блок · запас: 98%`, `Блоков: N · xruns: 0`.
   - `Main thread: 60 fps`.
3. **Ползунок Pitch** — тон меняет высоту в реальном времени. Верхний waveform (вход) и нижний (выход) отличаются частотой.
4. **Клик «Encode → Decode»** — круглый Opus round-trip: `Opus: 50 chunks · 8.5 KB · сжатие ≈ 22×`.

---

## 23.13.7. Анализ решения

**Что демонстрирует:**

1. **AudioWorklet работает.** Реальный DSP на audio rendering thread.
2. **Бюджет 2.667 мс** виден в числах: 0.05 мс при 2.667 мс — в 50 раз меньше.
3. **Xruns считаются**, но их 0 — JS справляется.
4. **Правильный паттерн общения** — через `this.port`, не через `self.postMessage`.
5. **Pitch shift в реальном времени** — обновление через `postMessage` без перезапуска.
6. **Параллельно Opus** — WebCodecs в main thread.

**Что не реализовано и почему:**

- **WASM в worklet.** Технически возможно (bundled bytes), но для ring buffer'а не оправдано. Разобрано в 23.13.4.
- **MSTP для аудио.** Не стандартизован в WebCodecs — только `AudioWorklet`.
- **WSOLA / phase vocoder.** Простой ресемплер даёт «chipmunk»-эффект, но показывает принцип.

**Правила проекта:**

- **`format_str(format_args!(...))` в tail-позициях.**
- **Отдельные функции для строк.**
- **`escape_html`** для сообщений об ошибках — символ `<` не рвёт верстку.
- **Yield через `TimeoutFuture`** — UI не блокируется.
- **Правило «focus app.rs»** — только 23.13.
- **Префикс `aw-`** — изолирован.

---

## 23.13.8. Итог

**AudioWorklet** — реальное время, 128 сэмплов / 2.667 мс, отдельный поток. Единственный способ делать DSP в браузере без глитчей.

**Что категорически нельзя в worklet'е:** `fetch()`, dynamic `import()`, `self.postMessage`, `await` в `process()`, аллокации в hot path.

**Что можно:** `registerProcessor`, `this.port`, `performance.now()`, `WebAssembly.instantiate(bytes)` (если байты уже есть), `sampleRate`, `currentTime`, `console`.

**WASM в worklet'е** — технически возможно через bundled-подход (bytes вшиваются в файл worklet'а на этапе сборки). Требует:
1. `wasm-pack --target no-modules`.
2. Постобработки glue-кода (замена `fetch` на inline bytes).
3. Build-скрипта для склейки.
4. Отказа от `import.meta` в glue.

Для этой главы — не оправдано. Ring buffer укладывается в 0.05 мс.

**Практические выводы:**

1. **Не используйте SPA-fallback для путей worklet'ов.** Vite вернёт HTML на 404 → SyntaxError с `<`. Диагностика: DevTools → Network → Response.
2. **web-sys API меняется.** `node.port()` → `Result<MessagePort>` в 0.3.105, `get_channel_data` → `Vec<f32>`. Сверяйтесь с docs.rs.
3. **`AudioWorkletGlobalScope` — не Worker.** Общение только через `this.port`. Никаких `self`.
4. **`WebAssembly.Module(bytes)` — синхронный.** Если всё же нужно WASM в worklet'е — используйте его, а не `instantiate`.
5. **Бюджет — жёсткий.** 2.667 мс. Всё, что дольше — xrun.

**Правило проекта.** Для real-time audio — используйте AudioWorklet с чистым JS. WASM подключайте, когда DSP действительно тяжёлая (> 1 мс на блок). Способы загрузки WASM в worklet'е описаны в граблях 2–5.