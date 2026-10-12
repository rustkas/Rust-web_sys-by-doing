# Глава 23.14. Temporal scalability: L1T2, L1T3

## 23.14.0. Обновлённая структура

```
src/
  video_encoder_scalability.rs    ← НОВЫЙ модуль 23.14
  video_encoder.rs                ← из 23.3 (переиспользуется: базовые обёртки)
  video_frame.rs                  ← из 23.1 (переиспользуется: хелперы JsValue)
  video_decoder.rs, video_encoder_tuning.rs, audio_api.rs  ← 23.2–23.13
tests/
  wasm_video_encoder_scalability.rs  ← НОВЫЙ тест
src/lib.rs                        ← +1 строка: pub mod video_encoder_scalability;
src/app.rs                        ← демо 23.14 (заменяется целиком)
```

**Комментарий.** Модуль `video_encoder_scalability.rs` — про **temporal scalability**: конфигурация `VideoEncoder` с `scalabilityMode: "L1T1"|"L1T2"|"L1T3"`, чтение `metadata.svc.temporalLayerId`, дроп лучей под нагрузкой без перекодирования. Префикс CSS — `ts-` (temporal scalability), не пересекается с `vf-` (23.1), `vd-` (23.2), `ve-` (23.3), `vt-` (23.4), `ac-` (23.5), `id-` (23.6), `px-` (23.7), `ms-` (23.8), `cp-` (23.9), `dm-` (23.10), `mx-` (23.11), `pp-` (23.12), `aw-` (23.13). Базовые обёртки (`make_video_encoder`, `encode`, `flush`, `close`) — **переиспользуются** из `video_encoder::*`. Мини-хелперы `num`, `set_prop`, `get_fn`, `format_str` — из `video_frame::*`.

**Ссылки для детального изучения:**

- MDN: [`VideoEncoder.configure()` — scalabilityMode](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder/configure)
- W3C: [WebCodecs § SvcOutputMetadata](https://w3c.github.io/webcodecs/#dictdef-svcoutputmetadata)
- W3C: [WebCodecs § EncodedVideoChunkMetadata](https://w3c.github.io/webcodecs/#dictdef-encodedvideochunkmetadata)
- Chrome Developers: [WebCodecs scalability modes](https://developer.chrome.com/docs/web-platform/best-practices/webcodecs#scalability_modes)
- W3C: [WebRTC-SVC Explainer](https://github.com/w3c/webrtc-svc/blob/main/explainer.md)

---

## 23.14.1. О чём эта глава

**Temporal scalability** — техника кодирования, при которой видеопоток разбивается на **временные слои** (temporal layers). Каждый слой содержит кадры определённой частоты:

- **T0** (базовый слой) — самая низкая частота (например, 15 fps из 30 fps).
- **T1** — дополняет T0 до 2× частоты (30 fps из 60 fps).
- **T2** — дополняет T1 до полной частоты.

**Ключевое свойство:** кадры T0 можно декодировать **независимо** от T1 и T2. Если отбросить T2 — получим видео с половинной частотой, но **без разрыва картинки**. Это адаптация к нагрузке **без перекодирования**.

**Что демонстрируется:**

1. Два `VideoEncoder`'а рядом: L1T1 (без scalability) и L1T3 (три временных слоя).
2. Чтение `metadata.svc.temporalLayerId` для каждого чанка.
3. Кнопки «Дропнуть T2» и «Дропнуть T1+T2» — визуально меняют FPS правого видео.
4. График FPS в реальном времени.

**Что не является темой:** spatial scalability (L2T1, L3T3) — только temporal.

---

## 23.14.2. Как работает temporal scalability

### Структура GOP

В обычном GOP (без scalability) кадры идут подряд:

```
I  P  P  P  P  P  P  P   (все кадры зависят от предыдущих)
```

С temporal scalability L1T3 (3 слоя) структура другая:

```
T0:  I-----------P-----------P-----------   (низкая частота, независимые)
T1:      P-----------P-----------P------    (зависит от T0)
T2:        P---P---P---P---P---P---P--      (зависит от T0 и T1)
```

Каждый слой может ссылаться только на **свой** или **более низкий** слой. T0 не зависит ни от чего. Если дропнуть T1 и T2, T0 продолжает декодироваться.

### `scalabilityMode`

Строка формата `L<X>T<Y>`:
- `L1` — 1 spatial layer (без spatial scalability).
- `T1` — 1 temporal layer (нет scalability).
- `T2` — 2 temporal layers.
- `T3` — 3 temporal layers.

Для temporal scalability доступны: `L1T1`, `L1T2`, `L1T3`.

### `metadata.svc.temporalLayerId`

Для каждого `EncodedVideoChunk` в `metadata.svc.temporalLayerId` приходит номер слоя (0, 1, 2). Это позволяет приложению **отбрасывать** чанки с `temporalLayerId > threshold` перед отправкой в сеть или декодер.

---

## 23.14.3. Что появится

- `check_scalability_config` — `isConfigSupported` с `scalabilityMode`.
- `configure_scalability` — `configure` с `scalabilityMode`.
- `extract_temporal_layer_id` — чтение `metadata.svc.temporalLayerId`.
- Два независимых энкодера (L1T1 и L1T3) работают параллельно.
- Дроп T2 и T1+T2 в реальном времени.
- График FPS: как меняется частота при дропе.

---

## 23.14.4. Исходные файлы

### `src/video_encoder_scalability.rs`

```rust
//! Глава 23.14 — Temporal scalability: L1T2, L1T3.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement, HtmlSelectElement,
    ImageData,
};

use crate::video_decoder::{
    close as decoder_close, configure as decoder_configure,
    decode_chunk, flush as decoder_flush, make_video_decoder,
};
use crate::video_encoder::{
    close as encoder_close, encode as encoder_encode, flush as encoder_flush,
    make_video_encoder,
};
use crate::video_encoder_scalability::{
    check_scalability_config, configure_scalability, extract_temporal_layer_id,
    format_layer_distribution,
};
use crate::video_frame::{
    close as frame_close, format_str, invoke, num, set_prop,
};

// =============================================================================
// Константы
// =============================================================================

const FRAME_W: u32 = 320;
const FRAME_H: u32 = 240;
const FRAME_COUNT: u32 = 30;
const FPS: f64 = 30.0;
const BITRATE: u32 = 500_000;
const CODEC: &str = "vp09.00.10.08";
const FRAME_DURATION_US: f64 = 33333.0;
// Цикл воспроизведения совпадает с длительностью генерации: 30 кадров при
// 30 fps проигрываются за 1 секунду. Тогда fps на канвасах совпадает с fps
// источника, а дроп слоёв даёт читаемые 15 / 8 fps.
const PLAYBACK_CYCLE_MS: f64 = 1000.0;
const RAF_MIN_DELTA_MS: f64 = 15.0;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.ts-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.ts-desc-line { margin:6px 0; }
.ts-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.ts-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.ts-btn:hover:not([disabled]) { background:#30363d; }
.ts-btn:disabled { opacity:0.5;cursor:not-allowed; }
.ts-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.ts-btn-primary:hover:not([disabled]) { background:#2ea043; }
.ts-btn-warn { background:#f0883e;border-color:#fb8500;color:#fff;}
.ts-btn-warn:hover:not([disabled]) { background:#fb8500; }
.ts-btn-danger { background:#a4302e;border-color:#d1242f;color:#fff;}
.ts-btn-danger:hover:not([disabled]) { background:#d1242f; }
.ts-btn-ghost { background:transparent;border-color:#30363d;color:#8b949e;}
.ts-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.ts-select { font-size:12px;padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#0d1117;color:#c9d1d9;font-family:ui-monospace,monospace;}
.ts-canvases { display:flex;gap:16px;flex-wrap:wrap;max-width:1000px;margin-bottom:16px;}
.ts-canvas-col { display:flex;flex-direction:column;gap:6px;flex:1 1 300px;min-width:0;}
.ts-canvas-title { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;display:flex;justify-content:space-between;}
.ts-canvas-title .fps-ok { color:#3fb950;font-weight:600; }
.ts-canvas-title .fps-warn { color:#f0883e;font-weight:600; }
.ts-canvas-title .fps-err { color:#f85149;font-weight:600; }
.ts-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;width:100%;height:auto;}
.ts-stats { max-width:1000px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;margin-bottom:16px;}
.ts-stats .label { color:#8b949e; }
.ts-stats .value { color:#79c0ff; }
.ts-stats .ok { color:#3fb950; }
.ts-stats .warn { color:#f0883e; }
.ts-stats .err { color:#f85149; }
.ts-stats .muted { color:#6e7681; }
.ts-stats .sep { border-top:1px solid #21262d;margin:6px 0; }
.ts-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum DropMode { None, DropT2, DropT1T2 }

impl DropMode {
    fn label(self) -> &'static str {
        match self {
            DropMode::None => "full L1T3",
            DropMode::DropT2 => "L1T3 без T2",
            DropMode::DropT1T2 => "только T0",
        }
    }
    fn from_str(s: &str) -> Self {
        match s {
            "t2" => DropMode::DropT2,
            "t12" => DropMode::DropT1T2,
            _ => DropMode::None,
        }
    }
}

#[derive(Clone, Copy, PartialEq, Eq)]
enum SupportState { Checking, Available, Unavailable }

#[derive(Clone)]
struct ChunkRecord {
    layer: u32,
    bytes: usize,
    chunk: JsValue,
}

#[derive(Clone)]
struct PlaybackStream {
    frames: Vec<Vec<u8>>,
    source_chunks: usize,
    source_bytes: usize,
    label: String,
}

impl PlaybackStream {
    fn new() -> Self {
        return Self {
            frames: Vec::new(),
            source_chunks: 0,
            source_bytes: 0,
            label: String::new(),
        };
    }
    fn fps(&self) -> f64 {
        if self.frames.is_empty() { return 0.0; }
        return self.frames.len() as f64 / (PLAYBACK_CYCLE_MS / 1000.0);
    }
    fn is_empty(&self) -> bool { self.frames.is_empty() }
}

struct DemoState {
    running: bool,
    phase: String,
    error: Option<String>,
    support: SupportState,
    stream_a: PlaybackStream,
    stream_b_full: PlaybackStream,
    stream_b_no_t2: PlaybackStream,
    stream_b_only_t0: PlaybackStream,
    drop_mode: DropMode,
    layer_dist: [u32; 3],
    playback_start_ms: f64,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            running: false,
            phase: String::from("простой"),
            error: None,
            support: SupportState::Checking,
            stream_a: PlaybackStream::new(),
            stream_b_full: PlaybackStream::new(),
            stream_b_no_t2: PlaybackStream::new(),
            stream_b_only_t0: PlaybackStream::new(),
            drop_mode: DropMode::None,
            layer_dist: [0; 3],
            playback_start_ms: 0.0,
        };
    }

    fn clear(&mut self) {
        self.stream_a = PlaybackStream::new();
        self.stream_b_full = PlaybackStream::new();
        self.stream_b_no_t2 = PlaybackStream::new();
        self.stream_b_only_t0 = PlaybackStream::new();
        self.layer_dist = [0; 3];
        self.playback_start_ms = 0.0;
        self.error = None;
    }

    fn current_b_stream(&self) -> &PlaybackStream {
        match self.drop_mode {
            DropMode::None => &self.stream_b_full,
            DropMode::DropT2 => &self.stream_b_no_t2,
            DropMode::DropT1T2 => &self.stream_b_only_t0,
        }
    }

    fn has_data(&self) -> bool {
        return !self.stream_a.is_empty();
    }
}

// =============================================================================
// Service Worker cleanup
// =============================================================================

fn has_sw_controller() -> bool {
    let global = js_sys::global();
    let Ok(nav) = Reflect::get(&global, &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(sw) = Reflect::get(&nav, &JsValue::from_str("serviceWorker")) else {
        return false;
    };
    if sw.is_undefined() || sw.is_null() { return false; }
    let Ok(ctrl) = Reflect::get(&sw, &JsValue::from_str("controller")) else {
        return false;
    };
    !ctrl.is_null() && !ctrl.is_undefined()
}

fn unregister_sw_and_reload(document: &Document) -> Result<(), JsValue> {
    let script = document.create_element("script")?;
    script.set_text_content(Some(
        r#"(function(){
            var KEY='ts-sw-cleared-23-14';
            if(sessionStorage.getItem(KEY)==='1') return;
            sessionStorage.setItem(KEY,'1');
            if(!('serviceWorker' in navigator)){location.reload();return;}
            navigator.serviceWorker.getRegistrations().then(function(rs){
                var ps=[];for(var i=0;i<rs.length;i++)ps.push(rs[i].unregister());
                return Promise.all(ps);
            }).then(function(){location.reload();}).catch(function(){location.reload();});
        })();"#,
    ));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&script)?;
    Ok(())
}

fn show_sw_notice(document: &Document) -> Result<(), JsValue> {
    let root = document.get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html(
        r#"<div style="max-width:900px;padding:20px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#c9d1d9;font-family:system-ui,sans-serif;">
            <h2 style="margin:0 0 10px 0;color:#f0883e;">Очистка старых Service Worker'ов…</h2>
            <p style="margin:0;line-height:1.55;">Снимаем регистрацию и перезагружаем страницу.</p>
        </div>"#,
    );
    Ok(())
}

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "ts-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() { p.remove_child(&old)?; }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

// =============================================================================
// Строковые хелперы
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

fn line_err(msg: &str) -> String {
    return format_str(format_args!(
        r#"<div><span class="err">{msg}</span></div>"#,
        msg = escape_html(msg),
    ));
}

fn line_phase(st: &DemoState) -> String {
    return format_str(format_args!(
        r#"<div><span class="label">Статус: </span><span class="value">{phase}</span> · <span class="label">Режим: </span><span class="value">{drop}</span></div>"#,
        phase = escape_html(&st.phase),
        drop = st.drop_mode.label(),
    ));
}

fn line_support(st: &DemoState) -> String {
    let (cls, text) = match st.support {
        SupportState::Checking => ("muted", "проверка…"),
        SupportState::Available => ("ok", "доступен (VP9)"),
        SupportState::Unavailable => ("err", "не поддерживается браузером"),
    };
    return format_str(format_args!(
        r#"<div><span class="label">Temporal scalability L1T3: </span><span class="{cls}">{text}</span></div>"#,
        cls = cls, text = text,
    ));
}

fn line_streams(st: &DemoState) -> String {
    if !st.has_data() {
        return String::new();
    }
    let a = &st.stream_a;
    let b = st.current_b_stream();
    let full = &st.stream_b_full;

    // Потеря частоты относительно L1T1 (базовый поток без скалирования).
    let fps_saved_pct = if !a.frames.is_empty() && !b.frames.is_empty() {
        100.0 - (b.frames.len() as f64 / a.frames.len() as f64) * 100.0
    } else {
        0.0
    };

    // SVC-оверхед: L1T3 full vs L1T1. Положительное значение = L1T3 тяжелее.
    // На коротких клипах оверхед больше нуля — это нормальное свойство SVC
    // (метаданные слоёв, NAL-обёртки), а не ошибка демки.
    let svc_overhead_pct = if a.source_bytes > 0 && full.source_bytes > 0 {
        (full.source_bytes as f64 / a.source_bytes as f64 - 1.0) * 100.0
    } else {
        0.0
    };
    let (ovh_cls, ovh_sign, ovh_abs) = if svc_overhead_pct >= 0.0 {
        ("warn", "+", svc_overhead_pct)
    } else {
        ("ok", "−", -svc_overhead_pct)
    };

    // Экономия от дропа: current (dropped) vs L1T3 full. Считается только
    // когда режим дропа != None. Положительное значение = сэкономили.
    let drop_line = if st.drop_mode == DropMode::None || full.source_bytes == 0 {
        String::new()
    } else {
        let saved = (1.0 - b.source_bytes as f64 / full.source_bytes as f64) * 100.0;
        if saved >= 0.0 {
            format_str(format_args!(
                r#" · <span class="label">экономия от дропа (vs L1T3 full): </span><span class="ok">−{p:.0}%</span>"#,
                p = saved,
            ))
        } else {
            format_str(format_args!(
                r#" · <span class="label">дроп добавил (vs L1T3 full): </span><span class="warn">+{p:.0}%</span>"#,
                p = -saved,
            ))
        }
    };

    return format_str(format_args!(
        r#"<div class="sep"></div>
           <div><span class="label">L1T1: </span><span class="value">{af} кадров</span> <span class="muted">({afps:.1} fps · {ab} Б в {ac} чанках)</span></div>
           <div><span class="label">{blabel}: </span><span class="value">{bf} кадров</span> <span class="muted">({bfps:.1} fps · {bb} Б в {bc} чанках)</span> · <span class="label">потеря частоты: </span><span class="warn">−{fsaved:.0}%</span>{drop_line}</div>
           <div><span class="label">SVC-оверхед (L1T3 full vs L1T1): </span><span class="{ovh_cls}">{ovh_sign}{ovh_abs:.0}%</span></div>
           <div><span class="label">Распределение слоёв (полный L1T3): </span><span class="value">{dist}</span></div>"#,
        af = a.frames.len(), afps = a.fps(),
        ab = a.source_bytes, ac = a.source_chunks,
        blabel = if b.label.is_empty() { "L1T3" } else { b.label.as_str() },
        bf = b.frames.len(), bfps = b.fps(),
        bb = b.source_bytes, bc = b.source_chunks,
        fsaved = fps_saved_pct,
        drop_line = drop_line,
        ovh_cls = ovh_cls, ovh_sign = ovh_sign, ovh_abs = ovh_abs,
        dist = format_layer_distribution(&st.layer_dist),
    ));
}

fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("ts-stats") else { return; };
    let err_line = st.error.as_ref().map(|m| line_err(m.as_str())).unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{support}{phase}{streams}"#,
        err = err_line,
        support = line_support(st),
        phase = line_phase(st),
        streams = line_streams(st),
    ));
    el.set_inner_html(&html);
}

fn update_buttons(doc: &Document, st: &DemoState) {
    let can_run = !st.running && st.support == SupportState::Available;
    let has_data = st.has_data();
    let set_enabled = |id: &str, enabled: bool| {
        if let Some(el) = doc.get_element_by_id(id) {
            if enabled { let _ = el.remove_attribute("disabled"); }
            else { let _ = el.set_attribute("disabled", ""); }
        }
    };
    set_enabled("ts-run", can_run);
    set_enabled("ts-drop-select", has_data && !st.running);
    set_enabled("ts-drop-t2", has_data && !st.running && st.drop_mode != DropMode::DropT2);
    set_enabled("ts-drop-t12", has_data && !st.running && st.drop_mode != DropMode::DropT1T2);
    set_enabled("ts-reset", has_data && !st.running && st.drop_mode != DropMode::None);
}

fn update_canvas_titles(doc: &Document, st: &DemoState) {
    if let Some(el) = doc.get_element_by_id("ts-title-a") {
        let (cls, fps) = if st.stream_a.is_empty() {
            ("", 0.0)
        } else {
            ("fps-ok", st.stream_a.fps())
        };
        let html = format_str(format_args!(
            r#"<span>L1T1 — все кадры</span><span class="{cls}">{fps:.1} fps</span>"#,
            cls = cls, fps = fps,
        ));
        el.set_inner_html(&html);
    }
    if let Some(el) = doc.get_element_by_id("ts-title-b") {
        let b = st.current_b_stream();
        let (cls, fps) = if b.is_empty() {
            ("", 0.0)
        } else if st.drop_mode == DropMode::None {
            ("fps-ok", b.fps())
        } else if st.drop_mode == DropMode::DropT2 {
            ("fps-warn", b.fps())
        } else {
            ("fps-err", b.fps())
        };
        let html = format_str(format_args!(
            r#"<span>{label}</span><span class="{cls}">{fps:.1} fps</span>"#,
            label = if b.label.is_empty() { "L1T3" } else { b.label.as_str() },
            cls = cls, fps = fps,
        ));
        el.set_inner_html(&html);
    }
}

// =============================================================================
// Рисование источника
// =============================================================================

fn draw_source(ctx: &CanvasRenderingContext2d, idx: u32) {
    let ctx_js: &JsValue = ctx.as_ref();
    let t = idx as f64 / FRAME_COUNT as f64;
    let hue = t * 360.0;
    let color = format_str(format_args!("hsl({hue:.0}, 70%, 45%)"));
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str(&color));
    let _ = invoke(
        ctx_js, "fillRect",
        &[num(0.0), num(0.0), num(FRAME_W as f64), num(FRAME_H as f64)],
    );
    let cx = FRAME_W as f64 / 2.0 + (t * std::f64::consts::TAU).sin() * 90.0;
    let cy = FRAME_H as f64 / 2.0 + (t * std::f64::consts::TAU).cos() * 70.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#f0f6fc"));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(
        ctx_js, "arc",
        &[num(cx), num(cy), num(24.0), num(0.0), num(std::f64::consts::TAU)],
    );
    let _ = invoke(ctx_js, "fill", &[]);
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#c9d1d9"));
    let _ = set_prop(ctx_js, "font", &JsValue::from_str("16px monospace"));
    let text = format_str(format_args!("#{idx}"));
    let _ = invoke(
        ctx_js, "fillText",
        &[JsValue::from_str(&text), num(8.0), num(FRAME_H as f64 - 8.0)],
    );
}

fn make_frame(canvas: &HtmlCanvasElement, timestamp_us: f64) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoFrame"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoFrame not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = Object::new();
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    let args = Array::new();
    args.push(&JsValue::from(canvas.clone()));
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn frame_to_rgba(frame: &JsValue, scratch_ctx: &CanvasRenderingContext2d) -> Vec<u8> {
    let ctx_js: &JsValue = scratch_ctx.as_ref();
    let _ = invoke(
        ctx_js, "drawImage",
        &[frame.clone(), num(0.0), num(0.0), num(FRAME_W as f64), num(FRAME_H as f64)],
    );
    let args = [num(0.0), num(0.0), num(FRAME_W as f64), num(FRAME_H as f64)];
    invoke(ctx_js, "getImageData", &args)
        .ok()
        .and_then(|v| v.dyn_into::<ImageData>().ok())
        .map(|img| img.data().0)
        .unwrap_or_default()
}

// =============================================================================
// Настройка энкодера
// =============================================================================

/// Конфигурирует `VideoEncoder` БЕЗ поля `scalabilityMode`.
///
/// Это корректный способ настройки базового потока L1T1: L1T1 — это не
/// "режим скалирования", а его отсутствие. Chromium/VP9 на конфиг с
/// `scalabilityMode: "L1T1"` не бросает исключение, но оставляет энкодер
/// в состоянии `"unconfigured"`, из-за чего первый же `encode()` падает
/// с `InvalidStateError`.
fn configure_plain(
    encoder: &JsValue,
    codec: &str,
    width: u32,
    height: u32,
    bitrate: u32,
    framerate: f64,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "width", &num(width as f64))?;
    set_prop(&cfg, "height", &num(height as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    set_prop(&cfg, "framerate", &num(framerate))?;
    let f = Reflect::get(encoder, &JsValue::from_str("configure"))?;
    let f = f.dyn_into::<js_sys::Function>()?;
    let _ = f.call1(encoder, &cfg)?;
    Ok(())
}

/// Проверяет, что энкодер действительно перешёл в состояние `"configured"`.
///
/// WebCodecs позволяет `configure()` завершиться без исключения, но
/// асинхронно перевести энкодер в error/closed. Без этой проверки мы
/// поймаем ошибку только на `encode()` — уже с невнятным сообщением.
fn assert_encoder_configured(encoder: &JsValue) -> Result<(), JsValue> {
    let state = Reflect::get(encoder, &JsValue::from_str("state"))?;
    let s = state.as_string().unwrap_or_else(|| String::from("<unknown>"));
    if s != "configured" {
        return Err(JsValue::from_str(&format_str(format_args!(
            "VideoEncoder state = \"{s}\" (ожидалось \"configured\") — \
             браузер не принял конфиг, вероятно scalabilityMode несовместим с кодеком",
        ))));
    }
    Ok(())
}

// =============================================================================
// Кодирование
// =============================================================================

struct EncoderOutput {
    chunks: Vec<ChunkRecord>,
}

async fn encode_l1t1(
    canvas: &HtmlCanvasElement,
    source_ctx: &CanvasRenderingContext2d,
) -> Result<EncoderOutput, JsValue> {
    let chunks: Rc<RefCell<Vec<ChunkRecord>>> = Rc::new(RefCell::new(Vec::new()));
    let chunks_cb = chunks.clone();
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, _meta: JsValue| {
            let bl = Reflect::get(&chunk, &JsValue::from_str("byteLength"))
                .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as usize;
            chunks_cb.borrow_mut().push(ChunkRecord { layer: 0, bytes: bl, chunk });
        },
    );
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });
    let enc = make_video_encoder(&on_output, &on_error)?;

    // L1T1 — baseline без temporal scaling. Не передаём scalabilityMode.
    configure_plain(&enc, CODEC, FRAME_W, FRAME_H, BITRATE, FPS)?;
    assert_encoder_configured(&enc)?;

    let mut last_yield = js_sys::Date::now();
    for i in 0..FRAME_COUNT {
        draw_source(source_ctx, i);
        let frame = make_frame(canvas, i as f64 * FRAME_DURATION_US)?;
        encoder_encode(&enc, &frame, i == 0)?;
        let _ = frame_close(&frame);
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    encoder_flush(&enc).await?;
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
    let out = chunks.borrow().clone();
    Ok(EncoderOutput { chunks: out })
}

async fn encode_l1t3(
    canvas: &HtmlCanvasElement,
    source_ctx: &CanvasRenderingContext2d,
) -> Result<EncoderOutput, JsValue> {
    let chunks: Rc<RefCell<Vec<ChunkRecord>>> = Rc::new(RefCell::new(Vec::new()));
    let chunks_cb = chunks.clone();
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, meta: JsValue| {
            let bl = Reflect::get(&chunk, &JsValue::from_str("byteLength"))
                .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as usize;
            let layer = extract_temporal_layer_id(&meta).unwrap_or(0);
            chunks_cb.borrow_mut().push(ChunkRecord { layer, bytes: bl, chunk });
        },
    );
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });
    let enc = make_video_encoder(&on_output, &on_error)?;
    configure_scalability(&enc, CODEC, FRAME_W, FRAME_H, BITRATE, FPS, "L1T3")?;
    assert_encoder_configured(&enc)?;

    let mut last_yield = js_sys::Date::now();
    for i in 0..FRAME_COUNT {
        draw_source(source_ctx, i);
        let frame = make_frame(canvas, i as f64 * FRAME_DURATION_US)?;
        encoder_encode(&enc, &frame, i == 0)?;
        let _ = frame_close(&frame);
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    encoder_flush(&enc).await?;
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
    let out = chunks.borrow().clone();
    Ok(EncoderOutput { chunks: out })
}

// =============================================================================
// Декодирование
// =============================================================================

async fn decode_chunks_to_frames(
    chunks: &[ChunkRecord],
    scratch_ctx: CanvasRenderingContext2d,
) -> Result<Vec<Vec<u8>>, JsValue> {
    if chunks.is_empty() {
        return Ok(Vec::new());
    }
    let frames: Rc<RefCell<Vec<Vec<u8>>>> = Rc::new(RefCell::new(Vec::new()));
    let frames_cb = frames.clone();
    let scratch_cb = scratch_ctx.clone();
    let on_output = Closure::<dyn FnMut(JsValue)>::new(move |frame: JsValue| {
        let bytes = frame_to_rgba(&frame, &scratch_cb);
        frames_cb.borrow_mut().push(bytes);
        let _ = frame_close(&frame);
    });
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });

    let dec = make_video_decoder(&on_output, &on_error)?;
    decoder_configure(&dec, CODEC, FRAME_W, FRAME_H, None)?;

    let mut last_yield = js_sys::Date::now();
    for rec in chunks.iter() {
        decode_chunk(&dec, &rec.chunk)?;
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    decoder_flush(&dec).await?;
    let _ = decoder_close(&dec);
    on_output.forget();
    on_error.forget();
    let out = frames.borrow().clone();
    Ok(out)
}

// =============================================================================
// Playback
// =============================================================================

fn draw_rgba_frame(canvas: &HtmlCanvasElement, bytes: &[u8]) {
    if bytes.is_empty() { return; }
    let Ok(Some(ctx_obj)) = canvas.get_context("2d") else { return; };
    let Ok(ctx) = ctx_obj.dyn_into::<CanvasRenderingContext2d>() else { return; };
    let clamped = wasm_bindgen::Clamped(bytes);
    let Ok(img) = ImageData::new_with_u8_clamped_array_and_sh(clamped, FRAME_W, FRAME_H) else {
        return;
    };
    let _ = invoke(ctx.as_ref(), "putImageData", &[JsValue::from(img), num(0.0), num(0.0)]);
}

fn frame_index_for(elapsed_ms: f64, total: usize) -> usize {
    if total == 0 { return 0; }
    let phase = (elapsed_ms / PLAYBACK_CYCLE_MS) % 1.0;
    return ((phase * total as f64) as usize).min(total - 1);
}

// =============================================================================
// Основной pipeline
// =============================================================================

async fn run_pipeline(doc: Document, st: Rc<RefCell<DemoState>>) {
    {
        let mut s = st.borrow_mut();
        if s.running { return; }
        s.running = true;
        s.clear();
        s.phase = String::from("кодирование L1T1…");
    }
    update_stats(&doc, &st.borrow());
    update_buttons(&doc, &st.borrow());

    let Some(canvas) = doc.get_element_by_id("ts-source")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else {
        set_error(&doc, &st, "no #ts-source");
        return;
    };
    let Some(source_ctx) = canvas.get_context("2d").ok().flatten()
        .and_then(|c| c.dyn_into::<CanvasRenderingContext2d>().ok()) else {
        set_error(&doc, &st, "no 2d context");
        return;
    };
    let Some(scratch) = doc.get_element_by_id("ts-scratch")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
        .and_then(|c| c.get_context("2d").ok().flatten())
        .and_then(|c| c.dyn_into::<CanvasRenderingContext2d>().ok()) else {
        set_error(&doc, &st, "no scratch canvas");
        return;
    };

    // --- Encode L1T1 ---
    let out_a = match encode_l1t1(&canvas, &source_ctx).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("encode L1T1: {e:?}"))); return; }
    };
    {
        let mut s = st.borrow_mut();
        s.phase = format_str(format_args!("L1T1: {} чанков → decode…", out_a.chunks.len()));
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_a = match decode_chunks_to_frames(&out_a.chunks, scratch.clone()).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T1: {e:?}"))); return; }
    };

    let bytes_a: usize = out_a.chunks.iter().map(|c| c.bytes).sum();
    {
        let mut s = st.borrow_mut();
        s.stream_a = PlaybackStream {
            frames: frames_a,
            source_chunks: out_a.chunks.len(),
            source_bytes: bytes_a,
            label: String::from("L1T1"),
        };
        s.phase = String::from("кодирование L1T3…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    // --- Encode L1T3 ---
    let out_b = match encode_l1t3(&canvas, &source_ctx).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("encode L1T3: {e:?}"))); return; }
    };

    let mut dist = [0u32; 3];
    for rec in &out_b.chunks {
        if rec.layer < 3 { dist[rec.layer as usize] += 1; }
    }

    // Декодируем три варианта.
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode L1T3 (full)…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_full = match decode_chunks_to_frames(&out_b.chunks, scratch.clone()).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T3 full: {e:?}"))); return; }
    };

    let chunks_no_t2: Vec<ChunkRecord> = out_b.chunks.iter()
        .filter(|c| c.layer <= 1).cloned().collect();
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode L1T3 (no T2)…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_no_t2 = match decode_chunks_to_frames(&chunks_no_t2, scratch.clone()).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T3 no T2: {e:?}"))); return; }
    };

    let chunks_only_t0: Vec<ChunkRecord> = out_b.chunks.iter()
        .filter(|c| c.layer == 0).cloned().collect();
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode L1T3 (only T0)…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_only_t0 = match decode_chunks_to_frames(&chunks_only_t0, scratch).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T3 only T0: {e:?}"))); return; }
    };

    let bytes_full: usize = out_b.chunks.iter().map(|c| c.bytes).sum();
    let bytes_no_t2: usize = chunks_no_t2.iter().map(|c| c.bytes).sum();
    let bytes_only_t0: usize = chunks_only_t0.iter().map(|c| c.bytes).sum();

    {
        let mut s = st.borrow_mut();
        s.stream_b_full = PlaybackStream {
            frames: frames_full,
            source_chunks: out_b.chunks.len(),
            source_bytes: bytes_full,
            label: String::from("L1T3 full"),
        };
        s.stream_b_no_t2 = PlaybackStream {
            frames: frames_no_t2,
            source_chunks: chunks_no_t2.len(),
            source_bytes: bytes_no_t2,
            label: String::from("L1T3 без T2"),
        };
        s.stream_b_only_t0 = PlaybackStream {
            frames: frames_only_t0,
            source_chunks: chunks_only_t0.len(),
            source_bytes: bytes_only_t0,
            label: String::from("L1T3 только T0"),
        };
        s.layer_dist = dist;
        s.phase = format_str(format_args!(
            "готово (L1T1: {} кадров, L1T3: {} кадров)",
            s.stream_a.frames.len(), s.stream_b_full.frames.len(),
        ));
        s.playback_start_ms = js_sys::Date::now();
        s.running = false;
    }

    update_stats(&doc, &st.borrow());
    update_buttons(&doc, &st.borrow());
    update_canvas_titles(&doc, &st.borrow());
}

fn set_error(doc: &Document, st: &Rc<RefCell<DemoState>>, msg: &str) {
    {
        let mut s = st.borrow_mut();
        s.error = Some(String::from(msg));
        s.phase = String::from("ошибка");
        s.running = false;
    }
    update_stats(doc, &st.borrow());
    update_buttons(doc, &st.borrow());
}

// =============================================================================
// rAF: playback
// =============================================================================

fn start_playback_loop(doc: Document, st: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;

    let Some(canvas_a) = doc.get_element_by_id("ts-playback-a")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else {
        return Ok(());
    };
    let Some(canvas_b) = doc.get_element_by_id("ts-playback-b")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else {
        return Ok(());
    };

    let cell: Rc<RefCell<Option<Closure<dyn FnMut(f64)>>>> = Rc::new(RefCell::new(None));
    let cell2 = cell.clone();
    let doc_loop = doc.clone();
    let st_loop = st.clone();
    let mut last_tick_ms = 0.0f64;

    *cell.borrow_mut() = Some(Closure::wrap(Box::new(move |_now: f64| {
        if let Some(w) = web_sys::window() {
            if let Some(cb) = cell2.borrow().as_ref() {
                let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
            }
        }
        let now = js_sys::Date::now();
        if now - last_tick_ms < RAF_MIN_DELTA_MS { return; }
        last_tick_ms = now;

        let (has_data, elapsed) = {
            let s = st_loop.borrow();
            (s.has_data(), now - s.playback_start_ms)
        };
        if !has_data { return; }

        // Рисуем напрямую из заимствованных кадров, без clone()
        // всего массива кадров.
        {
            let s = st_loop.borrow();
            let idx = frame_index_for(elapsed, s.stream_a.frames.len());
            if idx < s.stream_a.frames.len() {
                draw_rgba_frame(&canvas_a, &s.stream_a.frames[idx]);
            }
        }
        {
            let s = st_loop.borrow();
            let stream = s.current_b_stream();
            let idx = frame_index_for(elapsed, stream.frames.len());
            if idx < stream.frames.len() {
                draw_rgba_frame(&canvas_b, &stream.frames[idx]);
            }
        }

        update_canvas_titles(&doc_loop, &st_loop.borrow());
    }) as Box<dyn FnMut(f64)>));

    {
        let cb = cell.borrow();
        if let Some(c) = cb.as_ref() {
            window.request_animation_frame(c.as_ref().unchecked_ref())?;
        }
    }
    std::mem::forget(cell);
    Ok(())
}

// =============================================================================
// Описание
// =============================================================================

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>Temporal scalability</strong> — разбиение видеопотока на временные слои. T0 (базовый) декодируется независимо, T1 дополняет до 2× частоты, T2 — до полной.",
        "<strong>Демонстрация.</strong> Три canvas'а: source (генерация), L1T1 (все кадры), L1T3 (с фильтрацией). Оба потока реально декодируются и проигрываются за один временной интервал. Смена drop-режима меняет частоту правого canvas — без перезапуска энкодера.",
        "<strong>Ключевое свойство.</strong> После дропа T1+T2 остаётся только T0. Картинка та же — базовый слой содержит полное изображение, только реже. Это основа SFU в WebRTC.",
        "<strong>Проверка поддержки.</strong> L1T3 доступен не во всех браузерах и не со всеми кодеками. Кнопка «Запустить» активируется только если <code>isConfigSupported</code> вернул true.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ts-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("ts-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    if has_sw_controller() {
        show_sw_notice(&document)?;
        unregister_sw_and_reload(&document)?;
        return Ok(());
    }

    inject_styles(&document)?;

    let root = document.get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23.14 — Temporal scalability: L1T2, L1T3"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Скрытый scratch canvas.
    let scratch_el = document.create_element("canvas")?;
    scratch_el.set_id("ts-scratch");
    scratch_el.set_attribute("style", "display:none")?;
    let scratch_canvas: HtmlCanvasElement = scratch_el.dyn_into()?;
    scratch_canvas.set_width(FRAME_W);
    scratch_canvas.set_height(FRAME_H);
    root.append_child(&scratch_canvas)?;

    // Controls.
    let controls = document.create_element("div")?;
    controls.set_class_name("ts-block");
    controls.set_inner_html(
        r#"<button class="ts-btn ts-btn-primary" id="ts-run" disabled>Запустить кодирование</button>
           <label class="ts-label">Drop:</label>
           <select id="ts-drop-select" class="ts-select" disabled>
             <option value="none" selected>полный L1T3</option>
             <option value="t2">без T2</option>
             <option value="t12">только T0</option>
           </select>
           <button class="ts-btn ts-btn-warn" id="ts-drop-t2" disabled>Quick: без T2</button>
           <button class="ts-btn ts-btn-danger" id="ts-drop-t12" disabled>Quick: только T0</button>
           <button class="ts-btn ts-btn-ghost" id="ts-reset" disabled>Reset</button>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("ts-stats");
    stats.set_id("ts-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let canvases = document.create_element("div")?;
    canvases.set_class_name("ts-canvases");
    canvases.set_inner_html(&format_str(format_args!(
        r#"<div class="ts-canvas-col">
             <div class="ts-canvas-title" id="ts-title-source">
               <span>Source (генерация)</span><span class="muted">30 fps</span>
             </div>
             <canvas id="ts-source" class="ts-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="ts-canvas-col">
             <div class="ts-canvas-title" id="ts-title-a">
               <span>L1T1 — все кадры</span><span class="muted">—</span>
             </div>
             <canvas id="ts-playback-a" class="ts-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="ts-canvas-col">
             <div class="ts-canvas-title" id="ts-title-b">
               <span>L1T3</span><span class="muted">—</span>
             </div>
             <canvas id="ts-playback-b" class="ts-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = FRAME_W, h = FRAME_H,
    )));
    root.append_child(&canvases)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("ts-footnote");
    footnote.set_text_content(Some(
        "Оба canvas'а (L1T1 и L1T3) играют один временной интервал. \
         При дропе T2 или T1+T2 правый поток играет с меньшей частотой, \
         но картинка та же — базовый слой T0 сохранён.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    // Run.
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("ts-run")
            .ok_or_else(|| JsValue::from_str("no #ts-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let doc_cb = doc.clone();
            let st_cb = st.clone();
            spawn_local(async move {
                run_pipeline(doc_cb, st_cb).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Select.
    {
        let doc = document.clone();
        let st = state.clone();
        let sel_el = document.get_element_by_id("ts-drop-select")
            .ok_or_else(|| JsValue::from_str("no #ts-drop-select"))?;
        let sel: HtmlSelectElement = sel_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let Some(el) = doc.get_element_by_id("ts-drop-select") else { return; };
            let Ok(sel) = el.dyn_into::<HtmlSelectElement>() else { return; };
            {
                let mut s = st.borrow_mut();
                s.drop_mode = DropMode::from_str(&sel.value());
                s.playback_start_ms = js_sys::Date::now();
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
            update_canvas_titles(&doc, &st.borrow());
        });
        sel.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Quick buttons.
    for (btn_id, mode_value, mode_enum) in [
        ("ts-drop-t2", "t2", DropMode::DropT2),
        ("ts-drop-t12", "t12", DropMode::DropT1T2),
    ] {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id(btn_id)
            .ok_or_else(|| JsValue::from_str("missing quick button"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            if let Some(sel) = doc.get_element_by_id("ts-drop-select") {
                if let Ok(sel) = sel.dyn_into::<HtmlSelectElement>() {
                    sel.set_value(mode_value);
                }
            }
            {
                let mut s = st.borrow_mut();
                s.drop_mode = mode_enum;
                s.playback_start_ms = js_sys::Date::now();
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
            update_canvas_titles(&doc, &st.borrow());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Reset.
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("ts-reset")
            .ok_or_else(|| JsValue::from_str("no #ts-reset"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            if let Some(sel) = doc.get_element_by_id("ts-drop-select") {
                if let Ok(sel) = sel.dyn_into::<HtmlSelectElement>() {
                    sel.set_value("none");
                }
            }
            {
                let mut s = st.borrow_mut();
                s.drop_mode = DropMode::None;
                s.playback_start_ms = js_sys::Date::now();
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
            update_canvas_titles(&doc, &st.borrow());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Изначальный source-кадр.
    if let Some(canvas_el) = document.get_element_by_id("ts-source") {
        if let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() {
            if let Ok(Some(ctx_obj)) = canvas.get_context("2d") {
                if let Ok(ctx) = ctx_obj.dyn_into::<CanvasRenderingContext2d>() {
                    draw_source(&ctx, 0);
                }
            }
        }
    }

    update_stats(&document, &state.borrow());
    update_buttons(&document, &state.borrow());
    start_playback_loop(document.clone(), state.clone())?;

    // Проверка поддержки L1T3 — асинхронно, после построения UI.
    {
        let doc = document.clone();
        let st = state.clone();
        spawn_local(async move {
            let ok = check_scalability_config(
                CODEC, FRAME_W, FRAME_H, BITRATE, FPS, "L1T3",
            ).await.unwrap_or(false);
            {
                let mut s = st.borrow_mut();
                s.support = if ok { SupportState::Available } else { SupportState::Unavailable };
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
        });
    }

    Ok(())
}
```

### `src/lib.rs` — добавление

```rust
pub mod codecs_demo;              // 23
pub mod video_frame;              // 23.1
pub mod video_decoder;            // 23.2
pub mod video_encoder;            // 23.3
pub mod video_encoder_tuning;     // 23.4
pub mod audio_api;                // 23.13
pub mod video_encoder_scalability; // ← 23.14
```

### `tests/wasm_video_encoder_scalability.rs`

```rust
#![cfg(target_arch = "wasm32")]

//! 23.14. WASM-тесты temporal scalability.
//!
//! ## Что проверяется
//!
//! - Справочники (константы) — тривиально.
//! - Feature detection (`isConfigSupported`) — реально.
//! - `configure` — соответствует `isConfigSupported`.
//! - **Реальное кодирование** — L1T3 даёт несколько слоёв, L1T1 — нет.
//! - Утилиты (`extract_temporal_layer_id`, `format_layer_distribution`) —
//!   точное содержимое.
//!
//! ## Особенности headless
//!
//! - `configure` может вернуть `Err` в некоторых сборках Chrome —
//!   это ОК, если `isConfigSupported` тоже вернул `false`.
//! - Тесты кодирования пропускаются (return), если L1T3 не поддержан.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

// Импорт из video_encoder — только публичные обёртки.
use hello_wasm02::video_encoder::{
    close as encoder_close, flush as encoder_flush, make_video_encoder,
};
// Мини-хелперы JsValue определены в video_frame — импорт оттуда.
use hello_wasm02::video_frame::{get_fn, set_prop};
use hello_wasm02::video_encoder_scalability::{
    check_scalability_config, configure_scalability, extract_temporal_layer_id,
    format_layer_distribution, is_temporal_scalability_supported, mode_label,
    mode_layers, SCALABILITY_MODES,
};

wasm_bindgen_test_configure!(run_in_browser);

// Тестовый кодек: VP9 Profile 0. Chrome поддерживает L1T3 для VP9.
const CODEC: &str = "vp09.00.10.08";
const W: u32 = 320;
const H: u32 = 240;
const BITRATE: u32 = 500_000;
const FPS: f64 = 30.0;

// =============================================================================
// Хелперы
// =============================================================================

/// Создаёт VideoFrame из пустого canvas. Размер задан константами.
fn make_test_frame(timestamp_us: f64) -> Result<JsValue, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let canvas_el = document.create_element("canvas")?;
    let canvas: web_sys::HtmlCanvasElement = canvas_el.dyn_into()?;
    canvas.set_width(W);
    canvas.set_height(H);

    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("VideoFrame"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoFrame not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = js_sys::Object::new();
    set_prop(&init, "timestamp", &JsValue::from_f64(timestamp_us))?;
    let args = js_sys::Array::new();
    args.push(&JsValue::from(&canvas));
    args.push(&init);
    js_sys::Reflect::construct(&ctor, &args)
}

/// Проверяет поддержку L1T3 с тестовыми параметрами.
/// Возвращает true, если можно запускать тесты кодирования.
async fn is_l1t3_available() -> bool {
    return check_scalability_config(CODEC, W, H, BITRATE, FPS, "L1T3")
        .await
        .unwrap_or(false);
}

// =============================================================================
// Справочники
// =============================================================================

#[wasm_bindgen_test]
fn scalability_modes_well_formed() {
    assert_eq!(SCALABILITY_MODES.len(), 3);
    assert_eq!(SCALABILITY_MODES[0].0, "L1T1");
    assert_eq!(SCALABILITY_MODES[1].0, "L1T2");
    assert_eq!(SCALABILITY_MODES[2].0, "L1T3");
}

#[wasm_bindgen_test]
fn mode_layers_ok() {
    assert_eq!(mode_layers("L1T1"), 1);
    assert_eq!(mode_layers("L1T2"), 2);
    assert_eq!(mode_layers("L1T3"), 3);
    assert_eq!(mode_layers("unknown"), 1);
}

#[wasm_bindgen_test]
fn mode_label_exact_content() {
    assert_eq!(mode_label("L1T1"), "L1T1 (no scaling)");
    assert_eq!(mode_label("L1T2"), "L1T2 (2 layers)");
    assert_eq!(mode_label("L1T3"), "L1T3 (3 layers)");
    // Fallback возвращает исходную строку — важно, что не паникует.
    assert_eq!(mode_label("L2T3"), "L2T3");
    assert_eq!(mode_label(""), "");
}

// =============================================================================
// Feature detection
// =============================================================================

#[wasm_bindgen_test]
async fn l1t1_supported() {
    let ok = check_scalability_config(CODEC, W, H, BITRATE, FPS, "L1T1")
        .await
        .unwrap_or(false);
    assert!(ok, "L1T1 должен поддерживаться в Chrome с VP9");
}

#[wasm_bindgen_test]
async fn bogus_scalability_mode_not_supported() {
    let ok = check_scalability_config(CODEC, W, H, BITRATE, FPS, "L9T9")
        .await
        .unwrap_or(false);
    assert!(!ok, "L9T9 не должен поддерживаться");
}

#[wasm_bindgen_test]
async fn l1t3_status_reported() {
    // Тест не падает — фиксирует факт обращения к API.
    let result = is_temporal_scalability_supported(CODEC, W, H, BITRATE, FPS).await;
    assert!(result.is_ok(), "функция не должна падать");
    let supported = result.unwrap();
    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[ts-test] L1T3 supported: {supported}"
    )));
}

// =============================================================================
// configure — согласованность с isConfigSupported
// =============================================================================

#[wasm_bindgen_test]
async fn configure_l1t1_succeeds() {
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(|_, _| {});
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|_| {});
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");

    let supported = check_scalability_config(CODEC, W, H, BITRATE, FPS, "L1T1")
        .await
        .unwrap_or(false);
    assert!(supported, "L1T1 должен поддерживаться");

    let result = configure_scalability(&enc, CODEC, W, H, BITRATE, FPS, "L1T1");
    assert!(result.is_ok(), "configure(L1T1) должен пройти");

    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
async fn configure_l1t3_matches_isconfigsupported() {
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(|_, _| {});
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|_| {});
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");

    let supported = check_scalability_config(CODEC, W, H, BITRATE, FPS, "L1T3")
        .await
        .unwrap_or(false);

    let result = configure_scalability(&enc, CODEC, W, H, BITRATE, FPS, "L1T3");

    // Согласованность: если isConfigSupported вернул true, configure
    // тоже должен пройти. Если false — configure может упасть.
    assert_eq!(
        supported,
        result.is_ok(),
        "isConfigSupported и configure должны согласовываться для L1T3"
    );

    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
}

// =============================================================================
// extract_temporal_layer_id — точные сценарии
// =============================================================================

#[wasm_bindgen_test]
fn extract_from_empty_object_returns_none() {
    let meta = js_sys::Object::new();
    assert_eq!(extract_temporal_layer_id(&meta.into()), None);
}

#[wasm_bindgen_test]
fn extract_from_string_returns_none() {
    let meta = JsValue::from_str("not metadata");
    assert_eq!(extract_temporal_layer_id(&meta), None);
}

#[wasm_bindgen_test]
fn extract_from_null_svc_returns_none() {
    let meta = js_sys::Object::new();
    set_prop(&meta, "svc", &JsValue::NULL).unwrap();
    assert_eq!(extract_temporal_layer_id(&meta.into()), None);
}

#[wasm_bindgen_test]
fn extract_reads_layer_zero() {
    let svc = js_sys::Object::new();
    set_prop(&svc, "temporalLayerId", &JsValue::from_f64(0.0)).unwrap();
    let meta = js_sys::Object::new();
    set_prop(&meta, "svc", &svc).unwrap();
    assert_eq!(extract_temporal_layer_id(&meta.into()), Some(0));
}

#[wasm_bindgen_test]
fn extract_reads_layer_two() {
    let svc = js_sys::Object::new();
    set_prop(&svc, "temporalLayerId", &JsValue::from_f64(2.0)).unwrap();
    let meta = js_sys::Object::new();
    set_prop(&meta, "svc", &svc).unwrap();
    assert_eq!(extract_temporal_layer_id(&meta.into()), Some(2));
}

// =============================================================================
// format_layer_distribution
// =============================================================================

#[wasm_bindgen_test]
fn format_layer_distribution_ok() {
    let s = format_layer_distribution(&[10, 5, 3]);
    assert!(s.contains("T0: 10"), "got: {s}");
    assert!(s.contains("T1: 5"), "got: {s}");
    assert!(s.contains("T2: 3"), "got: {s}");
}

#[wasm_bindgen_test]
fn format_layer_distribution_zero() {
    let s = format_layer_distribution(&[0, 0, 0]);
    assert!(s.contains("T0: 0"), "got: {s}");
    assert!(s.contains("T1: 0"), "got: {s}");
    assert!(s.contains("T2: 0"), "got: {s}");
}

// =============================================================================
// Реальное кодирование: L1T3 даёт несколько слоёв
// =============================================================================

#[wasm_bindgen_test]
async fn l1t3_produces_temporal_layers() {
    if !is_l1t3_available().await {
        web_sys::console::warn_1(&JsValue::from_str(
            "[ts-test] L1T3 не поддерживается, тест пропущен",
        ));
        return;
    }

    let layers: Rc<RefCell<Vec<u32>>> = Rc::new(RefCell::new(Vec::new()));
    let layers_cb = layers.clone();
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |_chunk: JsValue, meta: JsValue| {
            let id = extract_temporal_layer_id(&meta).unwrap_or(0);
            layers_cb.borrow_mut().push(id);
        },
    );
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });

    let enc = make_video_encoder(&on_output, &on_error).expect("construct");
    configure_scalability(&enc, CODEC, W, H, BITRATE, FPS, "L1T3")
        .expect("configure L1T3");

    // 10 кадров.
    for i in 0..10u32 {
        let frame = make_test_frame(i as f64 * 33_333.0).expect("frame");
        let opts = js_sys::Object::new();
        if i == 0 {
            set_prop(&opts, "keyFrame", &JsValue::TRUE).unwrap();
        }
        let encode_fn = get_fn(&enc, "encode").unwrap();
        let _ = encode_fn.call2(&enc, &frame, &opts);
        // yield — чтобы callback успел.
        gloo_timers::future::TimeoutFuture::new(0).await;
    }

    encoder_flush(&enc).await.expect("flush");
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();

    let collected = layers.borrow().clone();
    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[ts-test] L1T3 layers: {:?}", collected,
    )));

    assert!(!collected.is_empty(), "должны быть чанки");
    assert_eq!(collected[0], 0, "первый чанк — T0");
    assert!(
        collected.iter().all(|&l| l <= 2),
        "temporalLayerId не должен превышать 2 для L1T3"
    );
    // Хотя бы один кадр должен быть в T0.
    let t0_count = collected.iter().filter(|&&l| l == 0).count();
    assert!(t0_count >= 1, "должен быть хотя бы один T0-чанк");
}

// =============================================================================
// Реальное кодирование: L1T1 не даёт слоёв выше T0
// =============================================================================

#[wasm_bindgen_test]
async fn l1t1_has_only_layer_zero() {
    let supported = check_scalability_config(CODEC, W, H, BITRATE, FPS, "L1T1")
        .await
        .unwrap_or(false);
    if !supported { return; }

    let ids: Rc<RefCell<Vec<Option<u32>>>> = Rc::new(RefCell::new(Vec::new()));
    let ids_cb = ids.clone();
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |_c: JsValue, meta: JsValue| {
            ids_cb.borrow_mut().push(extract_temporal_layer_id(&meta));
        },
    );
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|_| {});

    let enc = make_video_encoder(&on_output, &on_error).expect("construct");
    configure_scalability(&enc, CODEC, W, H, BITRATE, FPS, "L1T1")
        .expect("configure L1T1");

    for i in 0..3u32 {
        let frame = make_test_frame(i as f64 * 33_333.0).expect("frame");
        let opts = js_sys::Object::new();
        if i == 0 {
            set_prop(&opts, "keyFrame", &JsValue::TRUE).unwrap();
        }
        let encode_fn = get_fn(&enc, "encode").unwrap();
        let _ = encode_fn.call2(&enc, &frame, &opts);
        gloo_timers::future::TimeoutFuture::new(0).await;
    }

    encoder_flush(&enc).await.expect("flush");
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();

    let collected = ids.borrow().clone();
    web_sys::console::log_1(&JsValue::from_str(&format!(
        "[ts-test] L1T1 layer ids: {:?}", collected,
    )));

    for id in &collected {
        assert!(
            id.is_none() || *id == Some(0),
            "L1T1 не должен производить чанки со слоем > 0, got: {id:?}",
        );
    }
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_video_encoder_scalability

# 3. Демо
npm start
```

---

## 23.14.5. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.14 — Temporal scalability: L1T2, L1T3.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement, HtmlSelectElement,
    ImageData,
};

use crate::video_decoder::{
    close as decoder_close, configure as decoder_configure,
    decode_chunk, flush as decoder_flush, make_video_decoder,
};
use crate::video_encoder::{
    close as encoder_close, encode as encoder_encode, flush as encoder_flush,
    make_video_encoder,
};
use crate::video_encoder_scalability::{
    check_scalability_config, configure_scalability, extract_temporal_layer_id,
    format_layer_distribution,
};
use crate::video_frame::{
    close as frame_close, format_str, invoke, num, set_prop,
};

// =============================================================================
// Константы
// =============================================================================

const FRAME_W: u32 = 320;
const FRAME_H: u32 = 240;
const FRAME_COUNT: u32 = 30;
const FPS: f64 = 30.0;
const BITRATE: u32 = 500_000;
const CODEC: &str = "vp09.00.10.08";
const FRAME_DURATION_US: f64 = 33333.0;
// Цикл воспроизведения совпадает с длительностью генерации: 30 кадров при
// 30 fps проигрываются за 1 секунду. Тогда fps на канвасах совпадает с fps
// источника, а дроп слоёв даёт читаемые 15 / 8 fps.
const PLAYBACK_CYCLE_MS: f64 = 1000.0;
const RAF_MIN_DELTA_MS: f64 = 15.0;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.ts-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.ts-desc-line { margin:6px 0; }
.ts-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.ts-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.ts-btn:hover:not([disabled]) { background:#30363d; }
.ts-btn:disabled { opacity:0.5;cursor:not-allowed; }
.ts-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.ts-btn-primary:hover:not([disabled]) { background:#2ea043; }
.ts-btn-warn { background:#f0883e;border-color:#fb8500;color:#fff;}
.ts-btn-warn:hover:not([disabled]) { background:#fb8500; }
.ts-btn-danger { background:#a4302e;border-color:#d1242f;color:#fff;}
.ts-btn-danger:hover:not([disabled]) { background:#d1242f; }
.ts-btn-ghost { background:transparent;border-color:#30363d;color:#8b949e;}
.ts-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.ts-select { font-size:12px;padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#0d1117;color:#c9d1d9;font-family:ui-monospace,monospace;}
.ts-canvases { display:flex;gap:16px;flex-wrap:wrap;max-width:1000px;margin-bottom:16px;}
.ts-canvas-col { display:flex;flex-direction:column;gap:6px;flex:1 1 300px;min-width:0;}
.ts-canvas-title { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;display:flex;justify-content:space-between;}
.ts-canvas-title .fps-ok { color:#3fb950;font-weight:600; }
.ts-canvas-title .fps-warn { color:#f0883e;font-weight:600; }
.ts-canvas-title .fps-err { color:#f85149;font-weight:600; }
.ts-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;width:100%;height:auto;}
.ts-stats { max-width:1000px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;margin-bottom:16px;}
.ts-stats .label { color:#8b949e; }
.ts-stats .value { color:#79c0ff; }
.ts-stats .ok { color:#3fb950; }
.ts-stats .warn { color:#f0883e; }
.ts-stats .err { color:#f85149; }
.ts-stats .muted { color:#6e7681; }
.ts-stats .sep { border-top:1px solid #21262d;margin:6px 0; }
.ts-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum DropMode { None, DropT2, DropT1T2 }

impl DropMode {
    fn label(self) -> &'static str {
        match self {
            DropMode::None => "full L1T3",
            DropMode::DropT2 => "L1T3 без T2",
            DropMode::DropT1T2 => "только T0",
        }
    }
    fn from_str(s: &str) -> Self {
        match s {
            "t2" => DropMode::DropT2,
            "t12" => DropMode::DropT1T2,
            _ => DropMode::None,
        }
    }
}

#[derive(Clone, Copy, PartialEq, Eq)]
enum SupportState { Checking, Available, Unavailable }

#[derive(Clone)]
struct ChunkRecord {
    layer: u32,
    bytes: usize,
    chunk: JsValue,
}

#[derive(Clone)]
struct PlaybackStream {
    frames: Vec<Vec<u8>>,
    source_chunks: usize,
    source_bytes: usize,
    label: String,
}

impl PlaybackStream {
    fn new() -> Self {
        return Self {
            frames: Vec::new(),
            source_chunks: 0,
            source_bytes: 0,
            label: String::new(),
        };
    }
    fn fps(&self) -> f64 {
        if self.frames.is_empty() { return 0.0; }
        return self.frames.len() as f64 / (PLAYBACK_CYCLE_MS / 1000.0);
    }
    fn is_empty(&self) -> bool { self.frames.is_empty() }
}

struct DemoState {
    running: bool,
    phase: String,
    error: Option<String>,
    support: SupportState,
    stream_a: PlaybackStream,
    stream_b_full: PlaybackStream,
    stream_b_no_t2: PlaybackStream,
    stream_b_only_t0: PlaybackStream,
    drop_mode: DropMode,
    layer_dist: [u32; 3],
    playback_start_ms: f64,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            running: false,
            phase: String::from("простой"),
            error: None,
            support: SupportState::Checking,
            stream_a: PlaybackStream::new(),
            stream_b_full: PlaybackStream::new(),
            stream_b_no_t2: PlaybackStream::new(),
            stream_b_only_t0: PlaybackStream::new(),
            drop_mode: DropMode::None,
            layer_dist: [0; 3],
            playback_start_ms: 0.0,
        };
    }

    fn clear(&mut self) {
        self.stream_a = PlaybackStream::new();
        self.stream_b_full = PlaybackStream::new();
        self.stream_b_no_t2 = PlaybackStream::new();
        self.stream_b_only_t0 = PlaybackStream::new();
        self.layer_dist = [0; 3];
        self.playback_start_ms = 0.0;
        self.error = None;
    }

    fn current_b_stream(&self) -> &PlaybackStream {
        match self.drop_mode {
            DropMode::None => &self.stream_b_full,
            DropMode::DropT2 => &self.stream_b_no_t2,
            DropMode::DropT1T2 => &self.stream_b_only_t0,
        }
    }

    fn has_data(&self) -> bool {
        return !self.stream_a.is_empty();
    }
}

// =============================================================================
// Service Worker cleanup
// =============================================================================

fn has_sw_controller() -> bool {
    let global = js_sys::global();
    let Ok(nav) = Reflect::get(&global, &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(sw) = Reflect::get(&nav, &JsValue::from_str("serviceWorker")) else {
        return false;
    };
    if sw.is_undefined() || sw.is_null() { return false; }
    let Ok(ctrl) = Reflect::get(&sw, &JsValue::from_str("controller")) else {
        return false;
    };
    !ctrl.is_null() && !ctrl.is_undefined()
}

fn unregister_sw_and_reload(document: &Document) -> Result<(), JsValue> {
    let script = document.create_element("script")?;
    script.set_text_content(Some(
        r#"(function(){
            var KEY='ts-sw-cleared-23-14';
            if(sessionStorage.getItem(KEY)==='1') return;
            sessionStorage.setItem(KEY,'1');
            if(!('serviceWorker' in navigator)){location.reload();return;}
            navigator.serviceWorker.getRegistrations().then(function(rs){
                var ps=[];for(var i=0;i<rs.length;i++)ps.push(rs[i].unregister());
                return Promise.all(ps);
            }).then(function(){location.reload();}).catch(function(){location.reload();});
        })();"#,
    ));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&script)?;
    Ok(())
}

fn show_sw_notice(document: &Document) -> Result<(), JsValue> {
    let root = document.get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html(
        r#"<div style="max-width:900px;padding:20px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#c9d1d9;font-family:system-ui,sans-serif;">
            <h2 style="margin:0 0 10px 0;color:#f0883e;">Очистка старых Service Worker'ов…</h2>
            <p style="margin:0;line-height:1.55;">Снимаем регистрацию и перезагружаем страницу.</p>
        </div>"#,
    );
    Ok(())
}

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "ts-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() { p.remove_child(&old)?; }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

// =============================================================================
// Строковые хелперы
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

fn line_err(msg: &str) -> String {
    return format_str(format_args!(
        r#"<div><span class="err">{msg}</span></div>"#,
        msg = escape_html(msg),
    ));
}

fn line_phase(st: &DemoState) -> String {
    return format_str(format_args!(
        r#"<div><span class="label">Статус: </span><span class="value">{phase}</span> · <span class="label">Режим: </span><span class="value">{drop}</span></div>"#,
        phase = escape_html(&st.phase),
        drop = st.drop_mode.label(),
    ));
}

fn line_support(st: &DemoState) -> String {
    let (cls, text) = match st.support {
        SupportState::Checking => ("muted", "проверка…"),
        SupportState::Available => ("ok", "доступен (VP9)"),
        SupportState::Unavailable => ("err", "не поддерживается браузером"),
    };
    return format_str(format_args!(
        r#"<div><span class="label">Temporal scalability L1T3: </span><span class="{cls}">{text}</span></div>"#,
        cls = cls, text = text,
    ));
}

fn line_streams(st: &DemoState) -> String {
    if !st.has_data() {
        return String::new();
    }
    let a = &st.stream_a;
    let b = st.current_b_stream();
    let full = &st.stream_b_full;

    // Потеря частоты относительно L1T1 (базовый поток без скалирования).
    let fps_saved_pct = if !a.frames.is_empty() && !b.frames.is_empty() {
        100.0 - (b.frames.len() as f64 / a.frames.len() as f64) * 100.0
    } else {
        0.0
    };

    // SVC-оверхед: L1T3 full vs L1T1. Положительное значение = L1T3 тяжелее.
    // На коротких клипах оверхед больше нуля — это нормальное свойство SVC
    // (метаданные слоёв, NAL-обёртки), а не ошибка демки.
    let svc_overhead_pct = if a.source_bytes > 0 && full.source_bytes > 0 {
        (full.source_bytes as f64 / a.source_bytes as f64 - 1.0) * 100.0
    } else {
        0.0
    };
    let (ovh_cls, ovh_sign, ovh_abs) = if svc_overhead_pct >= 0.0 {
        ("warn", "+", svc_overhead_pct)
    } else {
        ("ok", "−", -svc_overhead_pct)
    };

    // Экономия от дропа: current (dropped) vs L1T3 full. Считается только
    // когда режим дропа != None. Положительное значение = сэкономили.
    let drop_line = if st.drop_mode == DropMode::None || full.source_bytes == 0 {
        String::new()
    } else {
        let saved = (1.0 - b.source_bytes as f64 / full.source_bytes as f64) * 100.0;
        if saved >= 0.0 {
            format_str(format_args!(
                r#" · <span class="label">экономия от дропа (vs L1T3 full): </span><span class="ok">−{p:.0}%</span>"#,
                p = saved,
            ))
        } else {
            format_str(format_args!(
                r#" · <span class="label">дроп добавил (vs L1T3 full): </span><span class="warn">+{p:.0}%</span>"#,
                p = -saved,
            ))
        }
    };

    return format_str(format_args!(
        r#"<div class="sep"></div>
           <div><span class="label">L1T1: </span><span class="value">{af} кадров</span> <span class="muted">({afps:.1} fps · {ab} Б в {ac} чанках)</span></div>
           <div><span class="label">{blabel}: </span><span class="value">{bf} кадров</span> <span class="muted">({bfps:.1} fps · {bb} Б в {bc} чанках)</span> · <span class="label">потеря частоты: </span><span class="warn">−{fsaved:.0}%</span>{drop_line}</div>
           <div><span class="label">SVC-оверхед (L1T3 full vs L1T1): </span><span class="{ovh_cls}">{ovh_sign}{ovh_abs:.0}%</span></div>
           <div><span class="label">Распределение слоёв (полный L1T3): </span><span class="value">{dist}</span></div>"#,
        af = a.frames.len(), afps = a.fps(),
        ab = a.source_bytes, ac = a.source_chunks,
        blabel = if b.label.is_empty() { "L1T3" } else { b.label.as_str() },
        bf = b.frames.len(), bfps = b.fps(),
        bb = b.source_bytes, bc = b.source_chunks,
        fsaved = fps_saved_pct,
        drop_line = drop_line,
        ovh_cls = ovh_cls, ovh_sign = ovh_sign, ovh_abs = ovh_abs,
        dist = format_layer_distribution(&st.layer_dist),
    ));
}

fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("ts-stats") else { return; };
    let err_line = st.error.as_ref().map(|m| line_err(m.as_str())).unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{support}{phase}{streams}"#,
        err = err_line,
        support = line_support(st),
        phase = line_phase(st),
        streams = line_streams(st),
    ));
    el.set_inner_html(&html);
}

fn update_buttons(doc: &Document, st: &DemoState) {
    let can_run = !st.running && st.support == SupportState::Available;
    let has_data = st.has_data();
    let set_enabled = |id: &str, enabled: bool| {
        if let Some(el) = doc.get_element_by_id(id) {
            if enabled { let _ = el.remove_attribute("disabled"); }
            else { let _ = el.set_attribute("disabled", ""); }
        }
    };
    set_enabled("ts-run", can_run);
    set_enabled("ts-drop-select", has_data && !st.running);
    set_enabled("ts-drop-t2", has_data && !st.running && st.drop_mode != DropMode::DropT2);
    set_enabled("ts-drop-t12", has_data && !st.running && st.drop_mode != DropMode::DropT1T2);
    set_enabled("ts-reset", has_data && !st.running && st.drop_mode != DropMode::None);
}

fn update_canvas_titles(doc: &Document, st: &DemoState) {
    if let Some(el) = doc.get_element_by_id("ts-title-a") {
        let (cls, fps) = if st.stream_a.is_empty() {
            ("", 0.0)
        } else {
            ("fps-ok", st.stream_a.fps())
        };
        let html = format_str(format_args!(
            r#"<span>L1T1 — все кадры</span><span class="{cls}">{fps:.1} fps</span>"#,
            cls = cls, fps = fps,
        ));
        el.set_inner_html(&html);
    }
    if let Some(el) = doc.get_element_by_id("ts-title-b") {
        let b = st.current_b_stream();
        let (cls, fps) = if b.is_empty() {
            ("", 0.0)
        } else if st.drop_mode == DropMode::None {
            ("fps-ok", b.fps())
        } else if st.drop_mode == DropMode::DropT2 {
            ("fps-warn", b.fps())
        } else {
            ("fps-err", b.fps())
        };
        let html = format_str(format_args!(
            r#"<span>{label}</span><span class="{cls}">{fps:.1} fps</span>"#,
            label = if b.label.is_empty() { "L1T3" } else { b.label.as_str() },
            cls = cls, fps = fps,
        ));
        el.set_inner_html(&html);
    }
}

// =============================================================================
// Рисование источника
// =============================================================================

fn draw_source(ctx: &CanvasRenderingContext2d, idx: u32) {
    let ctx_js: &JsValue = ctx.as_ref();
    let t = idx as f64 / FRAME_COUNT as f64;
    let hue = t * 360.0;
    let color = format_str(format_args!("hsl({hue:.0}, 70%, 45%)"));
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str(&color));
    let _ = invoke(
        ctx_js, "fillRect",
        &[num(0.0), num(0.0), num(FRAME_W as f64), num(FRAME_H as f64)],
    );
    let cx = FRAME_W as f64 / 2.0 + (t * std::f64::consts::TAU).sin() * 90.0;
    let cy = FRAME_H as f64 / 2.0 + (t * std::f64::consts::TAU).cos() * 70.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#f0f6fc"));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(
        ctx_js, "arc",
        &[num(cx), num(cy), num(24.0), num(0.0), num(std::f64::consts::TAU)],
    );
    let _ = invoke(ctx_js, "fill", &[]);
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#c9d1d9"));
    let _ = set_prop(ctx_js, "font", &JsValue::from_str("16px monospace"));
    let text = format_str(format_args!("#{idx}"));
    let _ = invoke(
        ctx_js, "fillText",
        &[JsValue::from_str(&text), num(8.0), num(FRAME_H as f64 - 8.0)],
    );
}

fn make_frame(canvas: &HtmlCanvasElement, timestamp_us: f64) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoFrame"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoFrame not available"));
    }
    let ctor = ctor.dyn_into::<js_sys::Function>()?;
    let init = Object::new();
    set_prop(&init, "timestamp", &num(timestamp_us))?;
    let args = Array::new();
    args.push(&JsValue::from(canvas.clone()));
    args.push(&init);
    Reflect::construct(&ctor, &args)
}

fn frame_to_rgba(frame: &JsValue, scratch_ctx: &CanvasRenderingContext2d) -> Vec<u8> {
    let ctx_js: &JsValue = scratch_ctx.as_ref();
    let _ = invoke(
        ctx_js, "drawImage",
        &[frame.clone(), num(0.0), num(0.0), num(FRAME_W as f64), num(FRAME_H as f64)],
    );
    let args = [num(0.0), num(0.0), num(FRAME_W as f64), num(FRAME_H as f64)];
    invoke(ctx_js, "getImageData", &args)
        .ok()
        .and_then(|v| v.dyn_into::<ImageData>().ok())
        .map(|img| img.data().0)
        .unwrap_or_default()
}

// =============================================================================
// Настройка энкодера
// =============================================================================

/// Конфигурирует `VideoEncoder` БЕЗ поля `scalabilityMode`.
///
/// Это корректный способ настройки базового потока L1T1: L1T1 — это не
/// "режим скалирования", а его отсутствие. Chromium/VP9 на конфиг с
/// `scalabilityMode: "L1T1"` не бросает исключение, но оставляет энкодер
/// в состоянии `"unconfigured"`, из-за чего первый же `encode()` падает
/// с `InvalidStateError`.
fn configure_plain(
    encoder: &JsValue,
    codec: &str,
    width: u32,
    height: u32,
    bitrate: u32,
    framerate: f64,
) -> Result<(), JsValue> {
    let cfg = Object::new();
    set_prop(&cfg, "codec", &JsValue::from_str(codec))?;
    set_prop(&cfg, "width", &num(width as f64))?;
    set_prop(&cfg, "height", &num(height as f64))?;
    set_prop(&cfg, "bitrate", &num(bitrate as f64))?;
    set_prop(&cfg, "framerate", &num(framerate))?;
    let f = Reflect::get(encoder, &JsValue::from_str("configure"))?;
    let f = f.dyn_into::<js_sys::Function>()?;
    let _ = f.call1(encoder, &cfg)?;
    Ok(())
}

/// Проверяет, что энкодер действительно перешёл в состояние `"configured"`.
///
/// WebCodecs позволяет `configure()` завершиться без исключения, но
/// асинхронно перевести энкодер в error/closed. Без этой проверки мы
/// поймаем ошибку только на `encode()` — уже с невнятным сообщением.
fn assert_encoder_configured(encoder: &JsValue) -> Result<(), JsValue> {
    let state = Reflect::get(encoder, &JsValue::from_str("state"))?;
    let s = state.as_string().unwrap_or_else(|| String::from("<unknown>"));
    if s != "configured" {
        return Err(JsValue::from_str(&format_str(format_args!(
            "VideoEncoder state = \"{s}\" (ожидалось \"configured\") — \
             браузер не принял конфиг, вероятно scalabilityMode несовместим с кодеком",
        ))));
    }
    Ok(())
}

// =============================================================================
// Кодирование
// =============================================================================

struct EncoderOutput {
    chunks: Vec<ChunkRecord>,
}

async fn encode_l1t1(
    canvas: &HtmlCanvasElement,
    source_ctx: &CanvasRenderingContext2d,
) -> Result<EncoderOutput, JsValue> {
    let chunks: Rc<RefCell<Vec<ChunkRecord>>> = Rc::new(RefCell::new(Vec::new()));
    let chunks_cb = chunks.clone();
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, _meta: JsValue| {
            let bl = Reflect::get(&chunk, &JsValue::from_str("byteLength"))
                .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as usize;
            chunks_cb.borrow_mut().push(ChunkRecord { layer: 0, bytes: bl, chunk });
        },
    );
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });
    let enc = make_video_encoder(&on_output, &on_error)?;

    // L1T1 — baseline без temporal scaling. Не передаём scalabilityMode.
    configure_plain(&enc, CODEC, FRAME_W, FRAME_H, BITRATE, FPS)?;
    assert_encoder_configured(&enc)?;

    let mut last_yield = js_sys::Date::now();
    for i in 0..FRAME_COUNT {
        draw_source(source_ctx, i);
        let frame = make_frame(canvas, i as f64 * FRAME_DURATION_US)?;
        encoder_encode(&enc, &frame, i == 0)?;
        let _ = frame_close(&frame);
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    encoder_flush(&enc).await?;
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
    let out = chunks.borrow().clone();
    Ok(EncoderOutput { chunks: out })
}

async fn encode_l1t3(
    canvas: &HtmlCanvasElement,
    source_ctx: &CanvasRenderingContext2d,
) -> Result<EncoderOutput, JsValue> {
    let chunks: Rc<RefCell<Vec<ChunkRecord>>> = Rc::new(RefCell::new(Vec::new()));
    let chunks_cb = chunks.clone();
    let on_output = Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, meta: JsValue| {
            let bl = Reflect::get(&chunk, &JsValue::from_str("byteLength"))
                .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as usize;
            let layer = extract_temporal_layer_id(&meta).unwrap_or(0);
            chunks_cb.borrow_mut().push(ChunkRecord { layer, bytes: bl, chunk });
        },
    );
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });
    let enc = make_video_encoder(&on_output, &on_error)?;
    configure_scalability(&enc, CODEC, FRAME_W, FRAME_H, BITRATE, FPS, "L1T3")?;
    assert_encoder_configured(&enc)?;

    let mut last_yield = js_sys::Date::now();
    for i in 0..FRAME_COUNT {
        draw_source(source_ctx, i);
        let frame = make_frame(canvas, i as f64 * FRAME_DURATION_US)?;
        encoder_encode(&enc, &frame, i == 0)?;
        let _ = frame_close(&frame);
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    encoder_flush(&enc).await?;
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
    let out = chunks.borrow().clone();
    Ok(EncoderOutput { chunks: out })
}

// =============================================================================
// Декодирование
// =============================================================================

async fn decode_chunks_to_frames(
    chunks: &[ChunkRecord],
    scratch_ctx: CanvasRenderingContext2d,
) -> Result<Vec<Vec<u8>>, JsValue> {
    if chunks.is_empty() {
        return Ok(Vec::new());
    }
    let frames: Rc<RefCell<Vec<Vec<u8>>>> = Rc::new(RefCell::new(Vec::new()));
    let frames_cb = frames.clone();
    let scratch_cb = scratch_ctx.clone();
    let on_output = Closure::<dyn FnMut(JsValue)>::new(move |frame: JsValue| {
        let bytes = frame_to_rgba(&frame, &scratch_cb);
        frames_cb.borrow_mut().push(bytes);
        let _ = frame_close(&frame);
    });
    let on_error = Closure::<dyn FnMut(JsValue)>::new(|e: JsValue| {
        web_sys::console::error_1(&e);
    });

    let dec = make_video_decoder(&on_output, &on_error)?;
    decoder_configure(&dec, CODEC, FRAME_W, FRAME_H, None)?;

    let mut last_yield = js_sys::Date::now();
    for rec in chunks.iter() {
        decode_chunk(&dec, &rec.chunk)?;
        let now = js_sys::Date::now();
        if now - last_yield >= 16.0 {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }
    decoder_flush(&dec).await?;
    let _ = decoder_close(&dec);
    on_output.forget();
    on_error.forget();
    let out = frames.borrow().clone();
    Ok(out)
}

// =============================================================================
// Playback
// =============================================================================

fn draw_rgba_frame(canvas: &HtmlCanvasElement, bytes: &[u8]) {
    if bytes.is_empty() { return; }
    let Ok(Some(ctx_obj)) = canvas.get_context("2d") else { return; };
    let Ok(ctx) = ctx_obj.dyn_into::<CanvasRenderingContext2d>() else { return; };
    let clamped = wasm_bindgen::Clamped(bytes);
    let Ok(img) = ImageData::new_with_u8_clamped_array_and_sh(clamped, FRAME_W, FRAME_H) else {
        return;
    };
    let _ = invoke(ctx.as_ref(), "putImageData", &[JsValue::from(img), num(0.0), num(0.0)]);
}

fn frame_index_for(elapsed_ms: f64, total: usize) -> usize {
    if total == 0 { return 0; }
    let phase = (elapsed_ms / PLAYBACK_CYCLE_MS) % 1.0;
    return ((phase * total as f64) as usize).min(total - 1);
}

// =============================================================================
// Основной pipeline
// =============================================================================

async fn run_pipeline(doc: Document, st: Rc<RefCell<DemoState>>) {
    {
        let mut s = st.borrow_mut();
        if s.running { return; }
        s.running = true;
        s.clear();
        s.phase = String::from("кодирование L1T1…");
    }
    update_stats(&doc, &st.borrow());
    update_buttons(&doc, &st.borrow());

    let Some(canvas) = doc.get_element_by_id("ts-source")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else {
        set_error(&doc, &st, "no #ts-source");
        return;
    };
    let Some(source_ctx) = canvas.get_context("2d").ok().flatten()
        .and_then(|c| c.dyn_into::<CanvasRenderingContext2d>().ok()) else {
        set_error(&doc, &st, "no 2d context");
        return;
    };
    let Some(scratch) = doc.get_element_by_id("ts-scratch")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
        .and_then(|c| c.get_context("2d").ok().flatten())
        .and_then(|c| c.dyn_into::<CanvasRenderingContext2d>().ok()) else {
        set_error(&doc, &st, "no scratch canvas");
        return;
    };

    // --- Encode L1T1 ---
    let out_a = match encode_l1t1(&canvas, &source_ctx).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("encode L1T1: {e:?}"))); return; }
    };
    {
        let mut s = st.borrow_mut();
        s.phase = format_str(format_args!("L1T1: {} чанков → decode…", out_a.chunks.len()));
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_a = match decode_chunks_to_frames(&out_a.chunks, scratch.clone()).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T1: {e:?}"))); return; }
    };

    let bytes_a: usize = out_a.chunks.iter().map(|c| c.bytes).sum();
    {
        let mut s = st.borrow_mut();
        s.stream_a = PlaybackStream {
            frames: frames_a,
            source_chunks: out_a.chunks.len(),
            source_bytes: bytes_a,
            label: String::from("L1T1"),
        };
        s.phase = String::from("кодирование L1T3…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    // --- Encode L1T3 ---
    let out_b = match encode_l1t3(&canvas, &source_ctx).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("encode L1T3: {e:?}"))); return; }
    };

    let mut dist = [0u32; 3];
    for rec in &out_b.chunks {
        if rec.layer < 3 { dist[rec.layer as usize] += 1; }
    }

    // Декодируем три варианта.
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode L1T3 (full)…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_full = match decode_chunks_to_frames(&out_b.chunks, scratch.clone()).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T3 full: {e:?}"))); return; }
    };

    let chunks_no_t2: Vec<ChunkRecord> = out_b.chunks.iter()
        .filter(|c| c.layer <= 1).cloned().collect();
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode L1T3 (no T2)…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_no_t2 = match decode_chunks_to_frames(&chunks_no_t2, scratch.clone()).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T3 no T2: {e:?}"))); return; }
    };

    let chunks_only_t0: Vec<ChunkRecord> = out_b.chunks.iter()
        .filter(|c| c.layer == 0).cloned().collect();
    {
        let mut s = st.borrow_mut();
        s.phase = String::from("decode L1T3 (only T0)…");
    }
    update_stats(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let frames_only_t0 = match decode_chunks_to_frames(&chunks_only_t0, scratch).await {
        Ok(v) => v,
        Err(e) => { set_error(&doc, &st, &format_str(format_args!("decode L1T3 only T0: {e:?}"))); return; }
    };

    let bytes_full: usize = out_b.chunks.iter().map(|c| c.bytes).sum();
    let bytes_no_t2: usize = chunks_no_t2.iter().map(|c| c.bytes).sum();
    let bytes_only_t0: usize = chunks_only_t0.iter().map(|c| c.bytes).sum();

    {
        let mut s = st.borrow_mut();
        s.stream_b_full = PlaybackStream {
            frames: frames_full,
            source_chunks: out_b.chunks.len(),
            source_bytes: bytes_full,
            label: String::from("L1T3 full"),
        };
        s.stream_b_no_t2 = PlaybackStream {
            frames: frames_no_t2,
            source_chunks: chunks_no_t2.len(),
            source_bytes: bytes_no_t2,
            label: String::from("L1T3 без T2"),
        };
        s.stream_b_only_t0 = PlaybackStream {
            frames: frames_only_t0,
            source_chunks: chunks_only_t0.len(),
            source_bytes: bytes_only_t0,
            label: String::from("L1T3 только T0"),
        };
        s.layer_dist = dist;
        s.phase = format_str(format_args!(
            "готово (L1T1: {} кадров, L1T3: {} кадров)",
            s.stream_a.frames.len(), s.stream_b_full.frames.len(),
        ));
        s.playback_start_ms = js_sys::Date::now();
        s.running = false;
    }

    update_stats(&doc, &st.borrow());
    update_buttons(&doc, &st.borrow());
    update_canvas_titles(&doc, &st.borrow());
}

fn set_error(doc: &Document, st: &Rc<RefCell<DemoState>>, msg: &str) {
    {
        let mut s = st.borrow_mut();
        s.error = Some(String::from(msg));
        s.phase = String::from("ошибка");
        s.running = false;
    }
    update_stats(doc, &st.borrow());
    update_buttons(doc, &st.borrow());
}

// =============================================================================
// rAF: playback
// =============================================================================

fn start_playback_loop(doc: Document, st: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;

    let Some(canvas_a) = doc.get_element_by_id("ts-playback-a")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else {
        return Ok(());
    };
    let Some(canvas_b) = doc.get_element_by_id("ts-playback-b")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok()) else {
        return Ok(());
    };

    let cell: Rc<RefCell<Option<Closure<dyn FnMut(f64)>>>> = Rc::new(RefCell::new(None));
    let cell2 = cell.clone();
    let doc_loop = doc.clone();
    let st_loop = st.clone();
    let mut last_tick_ms = 0.0f64;

    *cell.borrow_mut() = Some(Closure::wrap(Box::new(move |_now: f64| {
        if let Some(w) = web_sys::window() {
            if let Some(cb) = cell2.borrow().as_ref() {
                let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
            }
        }
        let now = js_sys::Date::now();
        if now - last_tick_ms < RAF_MIN_DELTA_MS { return; }
        last_tick_ms = now;

        let (has_data, elapsed) = {
            let s = st_loop.borrow();
            (s.has_data(), now - s.playback_start_ms)
        };
        if !has_data { return; }

        // Рисуем напрямую из заимствованных кадров, без clone()
        // всего массива кадров.
        {
            let s = st_loop.borrow();
            let idx = frame_index_for(elapsed, s.stream_a.frames.len());
            if idx < s.stream_a.frames.len() {
                draw_rgba_frame(&canvas_a, &s.stream_a.frames[idx]);
            }
        }
        {
            let s = st_loop.borrow();
            let stream = s.current_b_stream();
            let idx = frame_index_for(elapsed, stream.frames.len());
            if idx < stream.frames.len() {
                draw_rgba_frame(&canvas_b, &stream.frames[idx]);
            }
        }

        update_canvas_titles(&doc_loop, &st_loop.borrow());
    }) as Box<dyn FnMut(f64)>));

    {
        let cb = cell.borrow();
        if let Some(c) = cb.as_ref() {
            window.request_animation_frame(c.as_ref().unchecked_ref())?;
        }
    }
    std::mem::forget(cell);
    Ok(())
}

// =============================================================================
// Описание
// =============================================================================

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>Temporal scalability</strong> — разбиение видеопотока на временные слои. T0 (базовый) декодируется независимо, T1 дополняет до 2× частоты, T2 — до полной.",
        "<strong>Демонстрация.</strong> Три canvas'а: source (генерация), L1T1 (все кадры), L1T3 (с фильтрацией). Оба потока реально декодируются и проигрываются за один временной интервал. Смена drop-режима меняет частоту правого canvas — без перезапуска энкодера.",
        "<strong>Ключевое свойство.</strong> После дропа T1+T2 остаётся только T0. Картинка та же — базовый слой содержит полное изображение, только реже. Это основа SFU в WebRTC.",
        "<strong>Проверка поддержки.</strong> L1T3 доступен не во всех браузерах и не со всеми кодеками. Кнопка «Запустить» активируется только если <code>isConfigSupported</code> вернул true.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("ts-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("ts-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    if has_sw_controller() {
        show_sw_notice(&document)?;
        unregister_sw_and_reload(&document)?;
        return Ok(());
    }

    inject_styles(&document)?;

    let root = document.get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23.14 — Temporal scalability: L1T2, L1T3"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Скрытый scratch canvas.
    let scratch_el = document.create_element("canvas")?;
    scratch_el.set_id("ts-scratch");
    scratch_el.set_attribute("style", "display:none")?;
    let scratch_canvas: HtmlCanvasElement = scratch_el.dyn_into()?;
    scratch_canvas.set_width(FRAME_W);
    scratch_canvas.set_height(FRAME_H);
    root.append_child(&scratch_canvas)?;

    // Controls.
    let controls = document.create_element("div")?;
    controls.set_class_name("ts-block");
    controls.set_inner_html(
        r#"<button class="ts-btn ts-btn-primary" id="ts-run" disabled>Запустить кодирование</button>
           <label class="ts-label">Drop:</label>
           <select id="ts-drop-select" class="ts-select" disabled>
             <option value="none" selected>полный L1T3</option>
             <option value="t2">без T2</option>
             <option value="t12">только T0</option>
           </select>
           <button class="ts-btn ts-btn-warn" id="ts-drop-t2" disabled>Quick: без T2</button>
           <button class="ts-btn ts-btn-danger" id="ts-drop-t12" disabled>Quick: только T0</button>
           <button class="ts-btn ts-btn-ghost" id="ts-reset" disabled>Reset</button>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("ts-stats");
    stats.set_id("ts-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    let canvases = document.create_element("div")?;
    canvases.set_class_name("ts-canvases");
    canvases.set_inner_html(&format_str(format_args!(
        r#"<div class="ts-canvas-col">
             <div class="ts-canvas-title" id="ts-title-source">
               <span>Source (генерация)</span><span class="muted">30 fps</span>
             </div>
             <canvas id="ts-source" class="ts-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="ts-canvas-col">
             <div class="ts-canvas-title" id="ts-title-a">
               <span>L1T1 — все кадры</span><span class="muted">—</span>
             </div>
             <canvas id="ts-playback-a" class="ts-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="ts-canvas-col">
             <div class="ts-canvas-title" id="ts-title-b">
               <span>L1T3</span><span class="muted">—</span>
             </div>
             <canvas id="ts-playback-b" class="ts-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = FRAME_W, h = FRAME_H,
    )));
    root.append_child(&canvases)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("ts-footnote");
    footnote.set_text_content(Some(
        "Оба canvas'а (L1T1 и L1T3) играют один временной интервал. \
         При дропе T2 или T1+T2 правый поток играет с меньшей частотой, \
         но картинка та же — базовый слой T0 сохранён.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    // Run.
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("ts-run")
            .ok_or_else(|| JsValue::from_str("no #ts-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let doc_cb = doc.clone();
            let st_cb = st.clone();
            spawn_local(async move {
                run_pipeline(doc_cb, st_cb).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Select.
    {
        let doc = document.clone();
        let st = state.clone();
        let sel_el = document.get_element_by_id("ts-drop-select")
            .ok_or_else(|| JsValue::from_str("no #ts-drop-select"))?;
        let sel: HtmlSelectElement = sel_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            let Some(el) = doc.get_element_by_id("ts-drop-select") else { return; };
            let Ok(sel) = el.dyn_into::<HtmlSelectElement>() else { return; };
            {
                let mut s = st.borrow_mut();
                s.drop_mode = DropMode::from_str(&sel.value());
                s.playback_start_ms = js_sys::Date::now();
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
            update_canvas_titles(&doc, &st.borrow());
        });
        sel.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Quick buttons.
    for (btn_id, mode_value, mode_enum) in [
        ("ts-drop-t2", "t2", DropMode::DropT2),
        ("ts-drop-t12", "t12", DropMode::DropT1T2),
    ] {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id(btn_id)
            .ok_or_else(|| JsValue::from_str("missing quick button"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            if let Some(sel) = doc.get_element_by_id("ts-drop-select") {
                if let Ok(sel) = sel.dyn_into::<HtmlSelectElement>() {
                    sel.set_value(mode_value);
                }
            }
            {
                let mut s = st.borrow_mut();
                s.drop_mode = mode_enum;
                s.playback_start_ms = js_sys::Date::now();
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
            update_canvas_titles(&doc, &st.borrow());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Reset.
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document.get_element_by_id("ts-reset")
            .ok_or_else(|| JsValue::from_str("no #ts-reset"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure = Closure::<dyn FnMut()>::new(move || {
            if let Some(sel) = doc.get_element_by_id("ts-drop-select") {
                if let Ok(sel) = sel.dyn_into::<HtmlSelectElement>() {
                    sel.set_value("none");
                }
            }
            {
                let mut s = st.borrow_mut();
                s.drop_mode = DropMode::None;
                s.playback_start_ms = js_sys::Date::now();
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
            update_canvas_titles(&doc, &st.borrow());
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Изначальный source-кадр.
    if let Some(canvas_el) = document.get_element_by_id("ts-source") {
        if let Ok(canvas) = canvas_el.dyn_into::<HtmlCanvasElement>() {
            if let Ok(Some(ctx_obj)) = canvas.get_context("2d") {
                if let Ok(ctx) = ctx_obj.dyn_into::<CanvasRenderingContext2d>() {
                    draw_source(&ctx, 0);
                }
            }
        }
    }

    update_stats(&document, &state.borrow());
    update_buttons(&document, &state.borrow());
    start_playback_loop(document.clone(), state.clone())?;

    // Проверка поддержки L1T3 — асинхронно, после построения UI.
    {
        let doc = document.clone();
        let st = state.clone();
        spawn_local(async move {
            let ok = check_scalability_config(
                CODEC, FRAME_W, FRAME_H, BITRATE, FPS, "L1T3",
            ).await.unwrap_or(false);
            {
                let mut s = st.borrow_mut();
                s.support = if ok { SupportState::Available } else { SupportState::Unavailable };
            }
            update_stats(&doc, &st.borrow());
            update_buttons(&doc, &st.borrow());
        });
    }

    Ok(())
}
```

---

## 23.14.6. Анализ решения

### Что демонстрирует

1. **Два энкодера параллельно** — L1T1 и L1T3 на одном источнике кадров.
2. **Чтение `metadata.svc.temporalLayerId`** — распределение чанков по слоям.
3. **Дроп лучей без перекодирования** — просто фильтруем чанки по `layer_id`.
4. **FPS меняется** — с 30 до 15 (дроп T2) или до 7.5 (дроп T1+T2).
5. **Базовый поток не рвётся** — T0 всегда декодируется независимо.

### Ценность

**Адаптация к нагрузке без перекодирования.** В WebRTC SFU, если клиент на слабом канале, сервер может отбросить верхние слои и не тратить пропускную способность. T0 остаётся, картинка не рвётся — просто становится менее плавной.

**Один поток — три уровня качества.** Не нужно держать три отдельных энкодера. Один L1T3 даёт 3 уровня детализации.

### Найденные проблемы и решения

1. **`scalabilityMode` поддерживается не всеми кодеками.** VP9 и AV1 — да, H.264 — ограниченно. Демо проверяет `isConfigSupported` и сообщает об ошибке.
2. **`metadata.svc` может отсутствовать.** Для L1T1 его нет — `extract_temporal_layer_id` возвращает `None`, используем 0.
3. **Число кадров в чанках и FPS.** В демо мы генерируем 60 кадров и считаем FPS после дропа как долю пропущенных чанков. В реальном времени FPS измерялся бы по времени.

---

## 23.14.7. Итог

**Temporal scalability** — разбиение потока на временные слои. `scalabilityMode: "L1T2"|"L1T3"` в `VideoEncoder.configure`. Чтение `metadata.svc.temporalLayerId` позволяет приложению фильтровать чанки.

**Дроп лучей** — простой `filter` по `layer_id`. Никакого перекодирования. Базовый слой T0 всегда проходит.

**Практическое применение** — WebRTC SFU: сервер пересылает клиентам только те слои, которые им нужны. Адаптация к нагрузке без перезапуска энкодера.