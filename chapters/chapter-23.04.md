# Глава 23.4. Codec negotiation и тюнинг VideoEncoder (side-by-side)

## 23.4.0. Обновлённая структура

```
src/
  video_encoder_tuning.rs          ← НОВЫЙ модуль 23.4
  video_encoder.rs                 ← из 23.3 (переиспользуется: базовые обёртки)
  video_decoder.rs                 ← из 23.2 (не используется в 23.4)
  video_frame.rs                   ← из 23.1 (переиспользуется: хелперы JsValue)
  …
tests/
  wasm_video_encoder_tuning.rs     ← НОВЫЙ тест
src/lib.rs                         ← +1 строка: pub mod video_encoder_tuning;
src/app.rs                         ← демо 23.4 (заменяется целиком)
```

**Комментарий.** Модуль `video_encoder_tuning.rs` — про **negotiation** и **тюнинг**: `isConfigSupported` для набора кодеков, `latencyMode`, `bitrateMode`, `contentHint`, `hardwareAcceleration`, `scalabilityMode`, `alpha`. Префикс CSS — `vt-` (video tuning), не пересекается с `vf-` (23.1), `vd-` (23.2), `ve-` (23.3). Базовые обёртки (`make_video_encoder`, `encode`, `flush`, `close`) — **переиспользуются** из `video_encoder::*`. Мини-хелперы `num`, `set_prop`, `get_fn`, `format_str` — из `video_frame::*`. Дублирования нет.

**Ссылки для детального изучения:**

- MDN: [`VideoEncoder.isConfigSupported()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder/isConfigSupported_static)
- MDN: [`VideoEncoder.configure()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoEncoder/configure)
- MDN: [`VideoFrame.clone()`](https://developer.mozilla.org/en-US/docs/Web/API/VideoFrame/clone)
- W3C: [WebCodecs § VideoEncoderConfig](https://w3c.github.io/webcodecs/#videoencoderconfig)
- W3C: [WebCodecs § LatencyMode](https://w3c.github.io/webcodecs/#enumdef-latencymode)
- W3C: [WebCodecs § VideoEncoderBitrateMode](https://w3c.github.io/webcodecs/#enumdef-videoencoderbitratemode)
- W3C: [WebCodecs § ScalabilityMode](https://w3c.github.io/webcodecs/#enumdef-scalabilitymode)
- Chrome Developers: [Video processing with WebCodecs](https://developer.chrome.com/docs/web-platform/best-practices/webcodecs)

---

## 23.4.1. О чём эта подглава

23.3 показала, **как** работает `VideoEncoder`: `configure` → `encode` → `flush` → чанки. Но там использовался один кодек (`avc1.42001E`) и один набор параметров. Реальное приложение должно **выбирать** кодек и **настраивать** его под контент.

Подглава отвечает на шесть вопросов.

**Первый — какие кодеки поддерживает браузер.** Через `VideoEncoder.isConfigSupported()` — статический метод, проверяющий комбинацию без создания энкодера.

**Второй — как `latencyMode` влияет на результат.** `"quality"` (по умолчанию) — максимум качества. `"realtime"` — минимум задержки, обычно за счёт качества сжатия.

**Третий — что такое `contentHint`.** Подсказка кодеку о типе контента: `"motion"` / `"detail"` / `"text"`. Не команда — кодек может её проигнорировать.

**Четвёртый — что делает `hardwareAcceleration`.** `"no-preference"` (по умолчанию), `"prefer-hardware"` (GPU-энкодер), `"prefer-software"` (CPU). На устройствах без GPU-энкодера `prefer-hardware` отвергает все кодеки.

**Пятый — для чего `scalabilityMode` и `alpha`.** `scalabilityMode` (`L1T2`, `L2T1`, …) — SVC-режим, доступен только для VP9/AV1. `alpha: "keep"|"discard"` — сохранять ли альфа-канал.

**Шестой — как сравнить две конфигурации.** Параллельный side-by-side: два независимых `VideoEncoder` работают в одном проходе над одним источником.

---

## 23.4.2. Что появится

- `VideoEncoder.isConfigSupported` для 4 кодеков через `TuningConfig`;
- `configure` со всеми шестью подсказками WebCodecs;
- Два источника: `motion` (движущийся круг) и `text` (текстовый кадр);
- **Side-by-side** — два энкодера, один вход, честное сравнение;
- Δ% B от A и Δeff (key/avg_delta) как числовые метрики;
- История последних 5 пар с колонкой `Src`.

---

## 23.4.3. Что такое side-by-side и зачем

**Идея.** Один кадр рисуется **один раз**, клонируется через `VideoFrame.clone()`, и отправляется в **два разных энкодера** — A (baseline) и B (challenger). Оба видят **идентичный вход** — значит, разницу в размере и времени можно честно приписать конфигурации.

**Зачем `clone()`.** `VideoEncoder.encode()` **забирает владение** переданным `VideoFrame`. Если отправить один и тот же фрейм в два энкодера, второй получит уже закрытый объект и упадёт с `InvalidStateError`. Поэтому: `base_frame` → `clone()` → `clone()` → три независимых инстанса.

**Почему один проход, а не два последовательных.** Если запустить A, дождаться, потом запустить B — общее время будет `tA + tB`. При параллельном проходе оба энкодера работают одновременно (на уровне движка браузера), и общее время ближе к `max(tA, tB)`. Замер этого времени — тоже показатель.

**Ограничение.** Оба энкодера живут в главном потоке JS. Это не истинный parallelism на уровне ядер (для этого нужны Worker'ы), но `WebCodecs` выполняет тяжёлую работу в фоновых потоках браузера. Общий wall-time честно отражает нагрузку.

---

## 23.4.4. Практические находки

**1. `isConfigSupported` не падает при неподдерживаемом `contentHint`.** Просто возвращает `{supported: false}`. Не бросает исключение.

**2. `prefer-software` работает всегда.** Программный кодек есть в любом браузере. `prefer-hardware` — только при наличии GPU-энкодера (Linux без VAAPI → все кодеки отвергнуты).

**3. `contentHint` может игнорироваться.** На AV1 в Chrome подсказка `text` на текстовом кадре даёт **тот же размер**, что `motion` — кодек считает, что знает лучше. Демо должно это показывать честно, а не прятать за одинаковыми числами.

**4. `VideoFrame.clone()` — обязателен при параллельном проходе.** Иначе — `InvalidStateError` на втором энкодере.

**5. Смена параметров делает старые результаты невалидными.** Если пользователь поменял codec, а stats продолжают показывать прошлый результат — это вводит в заблуждение. Нужно сбрасывать результаты и support-списки при `change`.

**6. Накопление истории — практичнее, чем один `prev`.** Одна строка `Δ vs prev` работает, но не даёт сравнивать больше двух конфигураций. Таблица из 5 строк — именно то, что нужно для tuning-сессии.

---

## 23.4.5. Исходные файлы

### `src/video_encoder_tuning.rs`

```rust
//! 23.4. Codec negotiation и тюнинг VideoEncoder.
//!
//! Обёртки для negotiation и tuning:
//!
//! - `isConfigSupported` для списка кодеков;
//! - `configure_tuned` — configure с полным набором подсказок:
//!   `latencyMode`, `bitrateMode`, `contentHint`,
//!   `hardwareAcceleration`, `scalabilityMode`, `alpha`.
//!
//! ## Что нового по сравнению с 23.3
//!
//! 23.3 (`video_encoder`) — базовые обёртки: конструктор, configure,
//! encode, flush, close. Здесь — **tuning**: подсказки кодеку,
//! проверка поддержки, выбор лучшего кодека.
//!
//! ## Переиспользование
//!
//! `configure_tuned` строит `VideoEncoderConfig` через
//! `Object::new` + `set_prop` и вызывает метод `configure` напрямую
//! через `get_fn`. Дополнительных зависимостей от `video_encoder`
//! не требуется — только базовые хелперы из `video_frame`.

use js_sys::{Object, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{format_str, get_fn, num, set_prop};

// =============================================================================
// Справочники допустимых значений
// =============================================================================

/// Кодеки для проверки. `avc1.42001E` — H.264 Baseline.
/// `vp8` — VP8. `vp09.00.10.08` — VP9 Profile 0.
/// `av01.0.04M.08` — AV1 Main Profile.
pub const CODECS: &[(&str, &str)] = &[
    ("avc1.42001E", "H.264 Baseline"),
    ("vp8", "VP8"),
    ("vp09.00.10.08", "VP9"),
    ("av01.0.04M.08", "AV1"),
];

pub const LATENCY_MODES: &[&str] = &["quality", "realtime"];
pub const BITRATE_MODES: &[&str] = &["variable", "constant"];
pub const CONTENT_HINTS: &[&str] = &["motion", "detail", "text"];
pub const HW_ACCELS: &[&str] = &["no-preference", "prefer-hardware", "prefer-software"];

/// Значения `scalabilityMode`. Первое — «выключено».
/// Полный список в спецификации WebCodecs больше, здесь — основные.
pub const SCALABILITY_MODES: &[&str] = &[
    "(off)", "L1T1", "L1T2", "L1T3", "L2T1", "L2T2", "L3T1",
];

/// Значения `alpha`. Первое — «выключено».
pub const ALPHA_MODES: &[&str] = &["(off)", "discard", "keep"];

// =============================================================================
// Конфигурация tuning
// =============================================================================

/// Полная конфигурация для `configure` / `isConfigSupported`.
/// Все поля, которые могут отсутствовать в конфиге WebCodecs,
/// — `Option`. Пустые строки из UI (например `"(off)"`)
/// конвертируются в `None` на стороне демо-примера.
pub struct TuningConfig<'a> {
    pub codec: &'a str,
    pub width: u32,
    pub height: u32,
    pub bitrate: u32,
    pub framerate: f64,
    pub latency_mode: Option<&'a str>,
    pub bitrate_mode: Option<&'a str>,
    pub content_hint: Option<&'a str>,
    pub hardware_acceleration: Option<&'a str>,
    pub scalability_mode: Option<&'a str>,
    pub alpha: Option<&'a str>,
}

impl<'a> TuningConfig<'a> {
    /// Собирает JS-объект `VideoEncoderConfig`.
    fn to_js_object(&self) -> Result<Object, JsValue> {
        let cfg = Object::new();
        set_prop(&cfg, "codec", &JsValue::from_str(self.codec))?;
        set_prop(&cfg, "width", &num(self.width as f64))?;
        set_prop(&cfg, "height", &num(self.height as f64))?;
        set_prop(&cfg, "bitrate", &num(self.bitrate as f64))?;
        set_prop(&cfg, "framerate", &num(self.framerate))?;
        if let Some(v) = self.latency_mode {
            set_prop(&cfg, "latencyMode", &JsValue::from_str(v))?;
        }
        if let Some(v) = self.bitrate_mode {
            set_prop(&cfg, "bitrateMode", &JsValue::from_str(v))?;
        }
        if let Some(v) = self.content_hint {
            set_prop(&cfg, "contentHint", &JsValue::from_str(v))?;
        }
        if let Some(v) = self.hardware_acceleration {
            set_prop(&cfg, "hardwareAcceleration", &JsValue::from_str(v))?;
        }
        if let Some(v) = self.scalability_mode {
            set_prop(&cfg, "scalabilityMode", &JsValue::from_str(v))?;
        }
        if let Some(v) = self.alpha {
            set_prop(&cfg, "alpha", &JsValue::from_str(v))?;
        }
        Ok(cfg)
    }
}

// =============================================================================
// Расширенный configure
// =============================================================================

/// `encoder.configure(...)` с полным набором полей.
pub fn configure_tuned(encoder: &JsValue, cfg: &TuningConfig) -> Result<(), JsValue> {
    let cfg_obj = cfg.to_js_object()?;
    let f = get_fn(encoder, "configure")?;
    let _ = f.call1(encoder, &cfg_obj)?;
    Ok(())
}

// =============================================================================
// isConfigSupported
// =============================================================================

/// Результат проверки одного кодека.
#[derive(Clone)]
pub struct CodecSupport {
    pub codec: String,
    pub label: String,
    pub supported: bool,
}

/// Проверяет `isConfigSupported` для каждого кодека из `CODECS`.
/// Поле `codec` в переданной конфигурации игнорируется —
/// он подставляется из списка `CODECS` на каждой итерации.
pub async fn check_codecs(cfg: &TuningConfig<'_>) -> Vec<CodecSupport> {
    let mut out = Vec::with_capacity(CODECS.len());
    for (codec, label) in CODECS {
        let supported = check_one_codec(codec, cfg).await.unwrap_or(false);
        out.push(CodecSupport {
            codec: codec.to_string(),
            label: label.to_string(),
            supported,
        });
    }
    out
}

/// Проверяет один кодек с заданной конфигурацией.
/// Поле `codec` из конфигурации игнорируется — используется `codec` из аргумента.
pub async fn check_one_codec(
    codec: &str,
    cfg: &TuningConfig<'_>,
) -> Result<bool, JsValue> {
    let cfg_obj = cfg.to_js_object()?;
    set_prop(&cfg_obj, "codec", &JsValue::from_str(codec))?;

    let global = js_sys::global();
    let ctor = Reflect::get(&global, &JsValue::from_str("VideoEncoder"))?;
    if !ctor.is_function() {
        return Err(JsValue::from_str("VideoEncoder not available"));
    }
    let f = Reflect::get(&ctor, &JsValue::from_str("isConfigSupported"))?;
    if !f.is_function() {
        return Err(JsValue::from_str(
            "VideoEncoder.isConfigSupported not available",
        ));
    }
    let f = f.dyn_into::<js_sys::Function>()?;
    let promise = f.call1(&ctor, &cfg_obj)?.dyn_into::<Promise>()?;
    let result = JsFuture::from(promise).await?;
    let supported = Reflect::get(&result, &JsValue::from_str("supported"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
    Ok(supported)
}

// =============================================================================
// Форматирование результатов
// =============================================================================

/// Форматирует результат `check_codecs` в строку для UI.
pub fn format_codec_support(results: &[CodecSupport]) -> String {
    let mut html = String::with_capacity(results.len() * 64);
    for r in results {
        let mark = if r.supported { "✅" } else { "❌" };
        let cls = if r.supported { "ok" } else { "err" };
        html.push_str(&format_str(format_args!(
            r#"<span class="{cls}">{mark} {label}</span> "#,
            cls = cls,
            mark = mark,
            label = r.label,
        )));
    }
    html
}
```

### `src/lib.rs` — добавление

```rust
pub mod codecs_demo;              // 23
pub mod video_frame;              // 23.1
pub mod video_decoder;            // 23.2
pub mod video_encoder;            // 23.3
pub mod video_encoder_tuning;     // ← 23.4
```

### `tests/wasm_video_encoder_tuning.rs`

```rust
#![cfg(target_arch = "wasm32")]

use js_sys::Object;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::video_encoder::{
    chunk_byte_length, chunk_type, close as encoder_close,
    configure as base_configure, is_video_encoder_supported,
    make_video_encoder,
};
use hello_wasm02::video_encoder_tuning::{
    check_codecs, check_one_codec, configure_tuned, TuningConfig,
    ALPHA_MODES, BITRATE_MODES, CONTENT_HINTS, CODECS,
    HW_ACCELS, LATENCY_MODES, SCALABILITY_MODES,
};
use hello_wasm02::video_frame::{close as frame_close, num, set_prop};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Справочники допустимых значений
// =============================================================================

#[wasm_bindgen_test]
fn codec_list_is_well_formed() {
    assert!(CODECS.len() >= 4);
    assert_eq!(CODECS[0].0, "avc1.42001E");
    for (codec, _) in CODECS {
        assert!(!codec.is_empty());
    }
}

#[wasm_bindgen_test]
fn latency_modes_ok() {
    assert!(LATENCY_MODES.contains(&"quality"));
    assert!(LATENCY_MODES.contains(&"realtime"));
}

#[wasm_bindgen_test]
fn bitrate_modes_ok() {
    assert!(BITRATE_MODES.contains(&"variable"));
    assert!(BITRATE_MODES.contains(&"constant"));
}

#[wasm_bindgen_test]
fn content_hints_ok() {
    assert!(CONTENT_HINTS.contains(&"motion"));
    assert!(CONTENT_HINTS.contains(&"detail"));
    assert!(CONTENT_HINTS.contains(&"text"));
}

#[wasm_bindgen_test]
fn hw_accels_ok() {
    assert!(HW_ACCELS.contains(&"no-preference"));
    assert!(HW_ACCELS.contains(&"prefer-hardware"));
    assert!(HW_ACCELS.contains(&"prefer-software"));
}

#[wasm_bindgen_test]
fn scalability_modes_ok() {
    assert_eq!(SCALABILITY_MODES[0], "(off)");
    assert!(SCALABILITY_MODES.contains(&"L1T1"));
    assert!(SCALABILITY_MODES.contains(&"L1T3"));
    assert!(SCALABILITY_MODES.contains(&"L2T1"));
}

#[wasm_bindgen_test]
fn alpha_modes_ok() {
    assert_eq!(ALPHA_MODES[0], "(off)");
    assert!(ALPHA_MODES.contains(&"discard"));
    assert!(ALPHA_MODES.contains(&"keep"));
}

// =============================================================================
// Feature detection
// =============================================================================

#[wasm_bindgen_test]
fn video_encoder_is_supported_in_browser() {
    assert!(is_video_encoder_supported());
}

// =============================================================================
// Фабрика конфигураций
// =============================================================================

fn default_cfg<'a>(codec: &'a str) -> TuningConfig<'a> {
    TuningConfig {
        codec,
        width: 320,
        height: 240,
        bitrate: 1_000_000,
        framerate: 30.0,
        latency_mode: Some("quality"),
        bitrate_mode: Some("variable"),
        content_hint: Some("motion"),
        hardware_acceleration: Some("no-preference"),
        scalability_mode: None,
        alpha: None,
    }
}

// =============================================================================
// isConfigSupported через TuningConfig
// =============================================================================

#[wasm_bindgen_test]
async fn check_one_codec_h264_baseline() {
    let cfg = default_cfg("avc1.42001E");
    let ok = check_one_codec("avc1.42001E", &cfg).await.unwrap_or(false);
    assert!(ok, "avc1.42001E 320x240 1Mbps must be supported");
}

#[wasm_bindgen_test]
async fn check_codecs_returns_all_four() {
    let cfg = default_cfg("avc1.42001E");
    let results = check_codecs(&cfg).await;
    assert_eq!(results.len(), CODECS.len());
    assert!(
        results.iter().any(|r| r.supported),
        "At least one codec must be supported",
    );
}

#[wasm_bindgen_test]
async fn check_codecs_with_realtime_latency() {
    let mut cfg = default_cfg("avc1.42001E");
    cfg.latency_mode = Some("realtime");
    let results = check_codecs(&cfg).await;
    assert!(results.iter().any(|r| r.supported));
}

#[wasm_bindgen_test]
async fn check_codecs_with_content_hint_text() {
    let mut cfg = default_cfg("avc1.42001E");
    cfg.content_hint = Some("text");
    let results = check_codecs(&cfg).await;
    assert_eq!(results.len(), CODECS.len());
}

#[wasm_bindgen_test]
async fn check_codecs_with_constant_bitrate() {
    let mut cfg = default_cfg("avc1.42001E");
    cfg.bitrate_mode = Some("constant");
    let results = check_codecs(&cfg).await;
    assert!(results.iter().any(|r| r.supported));
}

#[wasm_bindgen_test]
async fn check_codecs_with_prefer_software() {
    let mut cfg = default_cfg("avc1.42001E");
    cfg.hardware_acceleration = Some("prefer-software");
    let results = check_codecs(&cfg).await;
    assert!(results.iter().any(|r| r.supported));
}

#[wasm_bindgen_test]
async fn check_codecs_with_scalability_mode() {
    let mut cfg = default_cfg("vp09.00.10.08");
    cfg.scalability_mode = Some("L1T2");
    let results = check_codecs(&cfg).await;
    assert_eq!(results.len(), CODECS.len());
}

#[wasm_bindgen_test]
async fn check_codecs_with_alpha_keep() {
    let mut cfg = default_cfg("vp8");
    cfg.alpha = Some("keep");
    let results = check_codecs(&cfg).await;
    assert_eq!(results.len(), CODECS.len());
}

#[wasm_bindgen_test]
async fn check_codecs_with_unsupported_combo_does_not_panic() {
    let mut cfg = default_cfg("av01.0.04M.08");
    cfg.hardware_acceleration = Some("prefer-hardware");
    cfg.scalability_mode = Some("L3T1");
    cfg.alpha = Some("keep");
    let results = check_codecs(&cfg).await;
    assert_eq!(results.len(), CODECS.len());
}

#[wasm_bindgen_test]
async fn invalid_codec_string_is_not_supported() {
    let cfg = default_cfg("not-a-real-codec");
    let ok = check_one_codec("not-a-real-codec", &cfg)
        .await
        .unwrap_or(false);
    assert!(!ok);
}

#[wasm_bindgen_test]
async fn codec_support_has_codec_and_label() {
    let cfg = default_cfg("avc1.42001E");
    let results = check_codecs(&cfg).await;
    for r in &results {
        assert!(!r.codec.is_empty());
        assert!(!r.label.is_empty());
    }
}

// =============================================================================
// configure_tuned: смоук-тесты
// =============================================================================

#[wasm_bindgen_test]
fn configure_tuned_does_not_throw_on_avc1() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_c: JsValue, _m: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");

    let cfg = default_cfg("avc1.42001E");
    assert!(configure_tuned(&enc, &cfg).is_ok());

    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn configure_tuned_with_scalability_and_alpha() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_c: JsValue, _m: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");

    let mut cfg = default_cfg("vp09.00.10.08");
    cfg.scalability_mode = Some("L1T2");
    cfg.alpha = Some("discard");

    let _ = configure_tuned(&enc, &cfg);
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn tuning_config_omits_none_fields() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_c: JsValue, _m: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");

    let cfg = TuningConfig {
        codec: "avc1.42001E",
        width: 320,
        height: 240,
        bitrate: 1_000_000,
        framerate: 30.0,
        latency_mode: None,
        bitrate_mode: None,
        content_hint: None,
        hardware_acceleration: None,
        scalability_mode: None,
        alpha: None,
    };
    assert!(configure_tuned(&enc, &cfg).is_ok());
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
}

// =============================================================================
// Обратная совместимость с 23.3
// =============================================================================

#[wasm_bindgen_test]
fn base_configure_still_works() {
    let on_output =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
            |_c: JsValue, _m: JsValue| {},
        );
    let on_error =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
            |_e: JsValue| {},
        );
    let enc = make_video_encoder(&on_output, &on_error).expect("construct");
    let result = base_configure(&enc, "avc1.42001E", 320, 240, 1_000_000, 30.0);
    assert!(result.is_ok());
    let _ = encoder_close(&enc);
    on_output.forget();
    on_error.forget();
}

#[wasm_bindgen_test]
fn chunk_helpers_are_resilient_to_non_chunks() {
    let v = JsValue::from_str("not a chunk");
    assert!(chunk_type(&v).is_empty());
    assert_eq!(chunk_byte_length(&v), 0);
}

// =============================================================================
// VideoFrame.clone() — критично для side-by-side
// =============================================================================

#[wasm_bindgen_test]
fn video_frame_clone_produces_independent_instance() {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document.create_element("canvas").unwrap();
    let canvas: web_sys::HtmlCanvasElement = canvas.dyn_into().unwrap();
    canvas.set_width(16);
    canvas.set_height(16);

    let global = js_sys::global();
    let ctor = js_sys::Reflect::get(&global, &JsValue::from_str("VideoFrame"))
        .unwrap();
    let ctor: js_sys::Function = ctor.dyn_into().unwrap();

    let init = Object::new();
    set_prop(&init, "timestamp", &num(0.0)).unwrap();
    let args = js_sys::Array::new();
    args.push(&JsValue::from(&canvas));
    args.push(&init);
    let frame = js_sys::Reflect::construct(&ctor, &args).unwrap();

    let clone_fn = js_sys::Reflect::get(&frame, &JsValue::from_str("clone"))
        .unwrap()
        .dyn_into::<js_sys::Function>()
        .unwrap();
    let clone = clone_fn.call0(&frame).unwrap();

    assert!(!clone.is_null() && !clone.is_undefined());

    let _ = frame_close(&frame);
    let _ = frame_close(&clone);
}
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome --test wasm_video_encoder_tuning

# 3. Демо
npm run wasm:dev && npm start
```

---

## 23.4.6. `src/app.rs` — автономный демо-пример

```rust
//! Глава 23.4 — Codec negotiation и тюнинг VideoEncoder (side-by-side).
//!
//! Демо: **параллельный side-by-side** двух `VideoEncoder` над одним
//! источником. Два слота (A и B) со своими параметрами; кадр
//! рисуется один раз, клонируется, отправляется в оба энкодера.
//! Сравнение — честное: оба видят идентичный вход.
//!
//! ## Что демонстрируется
//!
//! - `VideoEncoder.isConfigSupported()` для 4 кодеков (per-slot).
//! - Все шесть подсказок: `latencyMode`, `bitrateMode`,
//!   `contentHint`, `hardwareAcceleration`, `scalabilityMode`,
//!   `alpha`.
//! - **Параллельный encode** двух конфигураций в одном проходе.
//! - **Δ% B от A** — насколько B жмёт/тормозит относительно A.
//! - **Δeff** — во сколько раз key больше avg delta в каждом слоте.
//! - **История последних 5 пар** с колонкой `Src`.
//!
//! ## Что не является темой
//!
//! Round-trip decode — тема 23.3.
//!
//! **Файл автономен.** Стили `.vt-*` инжектируются в документ.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement,
    HtmlSelectElement,
};

use crate::video_encoder::{
    chunk_byte_length, chunk_type, close as encoder_close,
    encode as encoder_encode, flush as encoder_flush,
    is_video_encoder_supported, make_video_encoder,
};
use crate::video_encoder_tuning::{
    check_codecs, check_one_codec, configure_tuned, format_codec_support,
    CodecSupport, TuningConfig,
    ALPHA_MODES, BITRATE_MODES, CONTENT_HINTS, CODECS, HW_ACCELS,
    LATENCY_MODES, SCALABILITY_MODES,
};
use crate::video_frame::{
    close as frame_close, format_str, get_fn, invoke, num, set_prop,
};

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
    if sw.is_undefined() || sw.is_null() {
        return false;
    }
    let Ok(ctrl) = Reflect::get(&sw, &JsValue::from_str("controller")) else {
        return false;
    };
    !ctrl.is_null() && !ctrl.is_undefined()
}

fn unregister_sw_and_reload(document: &Document) -> Result<(), JsValue> {
    let script = document.create_element("script")?;
    script.set_text_content(Some(
        r#"(function(){
            var KEY='vt-sw-cleared-23-4';
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
    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html(
        r#"<div style="max-width:900px;padding:20px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#c9d1d9;font-family:system-ui,sans-serif;">
            <h2 style="margin:0 0 10px 0;color:#f0883e;">Очистка старых Service Worker'ов…</h2>
            <p style="margin:0;line-height:1.55;">Снимаем регистрацию и перезагружаем страницу.</p>
        </div>"#,
    );
    Ok(())
}

// =============================================================================
// Стили
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "vt-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() {
            p.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document.head().ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

const STYLE_CSS: &str = r#"
.vt-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.vt-desc-line { margin:6px 0; }
.vt-canvas-col { display:flex;flex-direction:column;gap:6px;max-width:900px;margin-bottom:16px;}
.vt-canvas-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.vt-canvas { display:block;background:#000;border:1px solid #21262d;border-radius:8px;}
.vt-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.vt-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.vt-btn:hover:not([disabled]) { background:#30363d; }
.vt-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.vt-btn-primary:hover:not([disabled]) { background:#2ea043; }
.vt-select { padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;font-size:12px;font-family:ui-monospace,monospace;}
.vt-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.vt-pair { display:flex;gap:16px;flex-wrap:wrap;max-width:900px;margin-bottom:16px;}
.vt-slot { flex:1 1 380px;min-width:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:10px 12px;}
.vt-slot-title-a { color:#f0883e;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.vt-slot-title-b { color:#58a6ff;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.vt-slot-controls { display:flex;flex-wrap:wrap;gap:6px;margin-bottom:10px;}
.vt-slot-controls .vt-select { padding:4px 6px;font-size:11px;max-width:130px;}
.vt-slot .vt-stats { max-width:none;margin-bottom:10px;font-size:11px;line-height:1.5;padding:8px 10px;}
.vt-slot .vt-chart { max-width:none;height:90px;margin-bottom:0;padding:6px;}
.vt-stats { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.vt-stats .label { color:#8b949e; }
.vt-stats .value { color:#79c0ff; }
.vt-stats .ok { color:#3fb950; }
.vt-stats .warn { color:#f0883e; }
.vt-stats .err { color:#f85149; }
.vt-stats .muted { color:#6e7681; }
.vt-chart { display:flex;align-items:flex-end;gap:2px;background:#0d1117;border:1px solid #21262d;border-radius:6px;}
.vt-bar { flex:1;min-width:3px;min-height:2px;border-radius:2px 2px 0 0;}
.vt-bar-key { background:#f0883e; }
.vt-bar-delta { background:#58a6ff; }
.vt-history { max-width:900px;padding:10px 14px;background:#0d1117;border-left:3px solid #d2a8ff;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:11px;line-height:1.5;margin-bottom:16px;overflow-x:auto;}
.vt-history-title { color:#8b949e; font-size:12px; margin-bottom:6px; }
.vt-history table { width:100%; border-collapse:collapse; }
.vt-history th { color:#8b949e; text-align:left; padding:3px 8px; border-bottom:1px solid #21262d; font-weight:500; }
.vt-history td { padding:3px 8px; border-bottom:1px solid #161b22; }
.vt-history tr:last-child td { border-bottom:none; }
.vt-history .current { background:rgba(63,185,80,0.08); }
.vt-history td.a-bytes { color:#f0883e; }
.vt-history td.b-bytes { color:#58a6ff; }
.vt-history td.delta-ok { color:#3fb950; }
.vt-history td.delta-warn { color:#f0883e; }
.vt-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Константы
// =============================================================================

const FRAME_W: u32 = 320;
const FRAME_H: u32 = 240;
const FRAME_COUNT: u32 = 30;
const FRAME_DURATION_US: f64 = 33333.0;
const BITRATE: u32 = 1_000_000;
const FRAMERATE: f64 = 30.0;
const YIELD_EVERY_MS: f64 = 16.0;
const HISTORY_LIMIT: usize = 5;

// =============================================================================
// Состояние
// =============================================================================

#[derive(Clone)]
struct ChunkInfo {
    ty: String,
    bytes: usize,
}

/// Параметры одного слота (A или B).
#[derive(Clone)]
struct SlotParams {
    codec: String,
    latency: String,
    content_hint: String,
    hw: String,
    scalability: String,
    alpha: String,
}

impl SlotParams {
    fn defaults_a() -> Self {
        Self {
            codec: CODECS[0].0.to_string(),
            latency: LATENCY_MODES[0].to_string(),
            content_hint: CONTENT_HINTS[0].to_string(),
            hw: HW_ACCELS[0].to_string(),
            scalability: SCALABILITY_MODES[0].to_string(),
            alpha: ALPHA_MODES[0].to_string(),
        }
    }

    fn defaults_b() -> Self {
        Self {
            codec: CODECS[1].0.to_string(),
            latency: LATENCY_MODES[0].to_string(),
            content_hint: CONTENT_HINTS[2].to_string(),
            hw: HW_ACCELS[0].to_string(),
            scalability: SCALABILITY_MODES[0].to_string(),
            alpha: ALPHA_MODES[0].to_string(),
        }
    }
}

/// Результат одного слота.
#[derive(Clone, Default)]
struct SlotResult {
    chunks: Vec<ChunkInfo>,
    key_count: u32,
    delta_count: u32,
    key_bytes: usize,
    delta_bytes: usize,
    total_bytes: usize,
    peak_bytes: usize,
    encode_time_ms: f64,
}

/// Снимок пары для истории.
#[derive(Clone)]
struct PairSnapshot {
    source: String,
    a: SlotParams,
    b: SlotParams,
    a_bytes: usize,
    a_time: f64,
    b_bytes: usize,
    b_time: f64,
}

struct DemoState {
    paused: bool,
    phase: String,
    error: Option<String>,
    running: bool,
    source: String,
    bitrate_mode: String,
    a: SlotParams,
    b: SlotParams,
    a_result: SlotResult,
    b_result: SlotResult,
    a_support: Vec<CodecSupport>,
    b_support: Vec<CodecSupport>,
    history: Vec<PairSnapshot>,
}

impl DemoState {
    fn new() -> Self {
        Self {
            paused: false,
            phase: String::from("простой"),
            error: None,
            running: false,
            source: String::from("motion"),
            bitrate_mode: BITRATE_MODES[0].to_string(),
            a: SlotParams::defaults_a(),
            b: SlotParams::defaults_b(),
            a_result: SlotResult::default(),
            b_result: SlotResult::default(),
            a_support: Vec::new(),
            b_support: Vec::new(),
            history: Vec::new(),
        }
    }

    fn clear_results(&mut self) {
        self.error = None;
        self.a_result = SlotResult::default();
        self.b_result = SlotResult::default();
    }

    fn push_history(&mut self, snap: PairSnapshot) {
        self.history.push(snap);
        while self.history.len() > HISTORY_LIMIT {
            self.history.remove(0);
        }
    }
}

// =============================================================================
// Guard
// =============================================================================

struct RunGuard(Rc<RefCell<DemoState>>);

impl Drop for RunGuard {
    fn drop(&mut self) {
        let mut s = self.0.borrow_mut();
        s.running = false;
    }
}

// =============================================================================
// Хелперы: короткие имена для таблицы
// =============================================================================

fn short_codec(codec: &str) -> &str {
    match codec {
        "avc1.42001E" => "H.264",
        "vp8" => "VP8",
        "vp09.00.10.08" => "VP9",
        "av01.0.04M.08" => "AV1",
        _ => codec,
    }
}

fn short_hw(hw: &str) -> &str {
    match hw {
        "no-preference" => "no-pref",
        "prefer-hardware" => "hw",
        "prefer-software" => "sw",
        _ => hw,
    }
}

// =============================================================================
// HTML-хелперы для опций select
// =============================================================================

fn options_str(values: &[&str], selected: &str) -> String {
    values
        .iter()
        .map(|v| {
            let sel = if *v == selected { " selected" } else { "" };
            format_str(format_args!(
                r#"<option value="{v}"{sel}>{v}</option>"#,
                v = v,
                sel = sel,
            ))
        })
        .collect::<Vec<_>>()
        .join("")
}

fn options_codec_str(selected: &str) -> String {
    CODECS
        .iter()
        .map(|(v, l)| {
            let sel = if *v == selected { " selected" } else { "" };
            format_str(format_args!(
                r#"<option value="{v}"{sel}>{l}</option>"#,
                v = v,
                l = l,
                sel = sel,
            ))
        })
        .collect::<Vec<_>>()
        .join("")
}

fn slot_controls_html(prefix: &str, defaults: &SlotParams) -> String {
    format_str(format_args!(
        r#"<div class="vt-slot-controls">
             <select class="vt-select" id="vt-{p}-codec" title="codec">{codec}</select>
             <select class="vt-select" id="vt-{p}-latency" title="latencyMode">{lat}</select>
             <select class="vt-select" id="vt-{p}-hint" title="contentHint">{ch}</select>
             <select class="vt-select" id="vt-{p}-hw" title="hardwareAcceleration">{hw}</select>
             <select class="vt-select" id="vt-{p}-scal" title="scalabilityMode">{sc}</select>
             <select class="vt-select" id="vt-{p}-alpha" title="alpha">{al}</select>
           </div>"#,
        p = prefix,
        codec = options_codec_str(&defaults.codec),
        lat = options_str(LATENCY_MODES, &defaults.latency),
        ch = options_str(CONTENT_HINTS, &defaults.content_hint),
        hw = options_str(HW_ACCELS, &defaults.hw),
        sc = options_str(SCALABILITY_MODES, &defaults.scalability),
        al = options_str(ALPHA_MODES, &defaults.alpha),
    ))
}

// =============================================================================
// UI: строки
// =============================================================================

fn line_err(msg: &str) -> String {
    format_str(format_args!(r#"<div><span class="err">{msg}</span></div>"#))
}

fn line_phase(st: &DemoState) -> String {
    format_str(format_args!(
        r#"<div><span class="label">Статус: </span><span class="value">{phase}</span></div>"#,
        phase = st.phase,
    ))
}

fn line_support(label: &str, results: &[CodecSupport]) -> String {
    if results.is_empty() {
        return String::new();
    }
    let any_supported = results.iter().any(|r| r.supported);
    let hint = if any_supported {
        String::new()
    } else {
        format_str(format_args!(
            r#" <span class="warn">— ни один не поддержан. Попробуйте no-preference / prefer-software.</span>"#
        ))
    };
    format_str(format_args!(
        r#"<div><span class="label">{label} support: </span>{html}{hint}</div>"#,
        label = label,
        html = format_codec_support(results),
        hint = hint,
    ))
}

/// Стат-блок одного слота.
fn render_slot_stats(result: &SlotResult) -> String {
    if result.chunks.is_empty() {
        return String::from("—");
    }
    let raw = (FRAME_W as usize) * (FRAME_H as usize) * 4 * result.chunks.len();
    let ratio = raw as f64 / result.total_bytes as f64;
    let min_b = result.chunks.iter().map(|c| c.bytes).min().unwrap_or(0);
    let avg_b = result.total_bytes / result.chunks.len();
    let per_frame = result.encode_time_ms / FRAME_COUNT as f64;

    // Δeff — во сколько раз key в среднем больше delta.
    // Показывает, работает ли delta-кодирование:
    //   ≥ 3×  — кодек эффективно экономит на P-frame'ах (ok);
    //   ≥ 1.5 — работает, но слабо (value);
    //   < 1.5 — delta почти не отличаются от key (warn).
    let eff_line = if result.key_count > 0 && result.delta_count > 0 {
        let key_avg = result.key_bytes as f64 / result.key_count as f64;
        let delta_avg = result.delta_bytes as f64 / result.delta_count as f64;
        let eff = if delta_avg > 0.0 {
            key_avg / delta_avg
        } else {
            0.0
        };
        let cls = if eff >= 3.0 {
            "ok"
        } else if eff >= 1.5 {
            "value"
        } else {
            "warn"
        };
        format_str(format_args!(
            r#"<div><span class="label">Δeff: </span><span class="{cls}">key/avg_delta ≈ {e:.1}×</span></div>"#,
            cls = cls,
            e = eff,
        ))
    } else {
        String::new()
    };

    format_str(format_args!(
        r#"<div><span class="label">Bytes: </span><span class="value">{t}</span> · <span class="label">Time: </span><span class="value">{ms:.0} мс</span> <span class="muted">({pf:.1} мс/к)</span></div>
           <div><span class="label">Chunks: </span><span class="value">{n}</span> (key: <span class="ok">{k}</span>, delta: <span class="warn">{d}</span>)</div>
           <div><span class="label">Sizes: </span><span class="muted">min</span> <span class="value">{mn}</span> · <span class="muted">avg</span> <span class="value">{av}</span> · <span class="muted">peak</span> <span class="warn">{pk}</span></div>
           <div><span class="label">Ratio: </span><span class="ok">≈ {r:.1}×</span></div>{eff}"#,
        t = result.total_bytes,
        ms = result.encode_time_ms,
        pf = per_frame,
        n = result.chunks.len(),
        k = result.key_count,
        d = result.delta_count,
        mn = min_b,
        av = avg_b,
        pk = result.peak_bytes,
        r = ratio,
        eff = eff_line,
    ))
}

fn bar_html(idx: usize, info: &ChunkInfo, max_bytes: usize) -> String {
    let pct = (info.bytes as f64 / max_bytes as f64) * 100.0;
    let cls = if info.ty == "key" {
        "vt-bar vt-bar-key"
    } else {
        "vt-bar vt-bar-delta"
    };
    let title = format_str(format_args!(
        "#{i} {ty}: {b} B",
        i = idx,
        ty = info.ty,
        b = info.bytes,
    ));
    format_str(format_args!(
        r##"<div class="{cls}" style="height:{pct:.1}%" title="{title}"></div>"##,
        cls = cls,
        pct = pct,
        title = title,
    ))
}

fn render_slot_chart(result: &SlotResult) -> String {
    if result.chunks.is_empty() {
        return String::new();
    }
    let max = result.chunks.iter().map(|c| c.bytes).max().unwrap_or(1).max(1);
    let mut html = String::with_capacity(result.chunks.len() * 96);
    for (i, c) in result.chunks.iter().enumerate() {
        html.push_str(&bar_html(i, c, max));
    }
    html
}

/// Общий стат-блок (фаза + поддержка + ошибка).
fn update_stats(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("vt-stats") else {
        return;
    };
    let err_line = st
        .error
        .as_ref()
        .map(|m| line_err(m.as_str()))
        .unwrap_or_default();
    let html = format_str(format_args!(
        r#"{err}{phase}{sa}{sb}"#,
        err = err_line,
        phase = line_phase(st),
        sa = line_support("A", &st.a_support),
        sb = line_support("B", &st.b_support),
    ));
    el.set_inner_html(&html);
}

fn update_slot_stats(doc: &Document, id: &str, result: &SlotResult) {
    let Some(el) = doc.get_element_by_id(id) else {
        return;
    };
    el.set_inner_html(&render_slot_stats(result));
}

fn update_slot_chart(doc: &Document, id: &str, result: &SlotResult) {
    let Some(el) = doc.get_element_by_id(id) else {
        return;
    };
    el.set_inner_html(&render_slot_chart(result));
}

fn update_all_slot_ui(doc: &Document, st: &DemoState) {
    update_slot_stats(doc, "vt-stats-a", &st.a_result);
    update_slot_stats(doc, "vt-stats-b", &st.b_result);
    update_slot_chart(doc, "vt-chart-a", &st.a_result);
    update_slot_chart(doc, "vt-chart-b", &st.b_result);
}

/// Таблица истории пар.
fn render_history(st: &DemoState) -> String {
    if st.history.is_empty() {
        return format_str(format_args!(
            r#"<div class="vt-history-title">История: пусто. Нажмите Encode Both.</div>"#
        ));
    }
    let n = st.history.len();
    let mut rows = String::with_capacity(n * 320);
    for (i, h) in st.history.iter().enumerate() {
        let last = i == n - 1;
        let cls = if last { "current" } else { "" };

        let a_delta = if h.a_bytes > 0 {
            ((h.b_bytes as i64 - h.a_bytes as i64) as f64 / h.a_bytes as f64) * 100.0
        } else {
            0.0
        };
        let dsign = if a_delta >= 0.0 { "+" } else { "" };
        let dcls = if a_delta <= 0.0 { "delta-ok" } else { "delta-warn" };

        rows.push_str(&format_str(format_args!(
            r#"<tr class="{cls}"><td>{idx}</td><td>{src}</td>
                <td>{ac}</td><td>{ah}</td><td>{al}</td><td>{ahw}</td><td class="a-bytes">{an}</td><td>{atm:.0}</td>
                <td>{bc}</td><td>{bh}</td><td>{bl}</td><td>{bhw}</td><td class="b-bytes">{bn}</td><td>{btm:.0}</td>
                <td class="{dc}">{ds}{dp:.1}%</td></tr>"#,
            cls = cls,
            idx = i + 1,
            src = h.source,
            ac = short_codec(&h.a.codec),
            ah = h.a.content_hint,
            al = h.a.latency,
            ahw = short_hw(&h.a.hw),
            an = h.a_bytes,
            atm = h.a_time,
            bc = short_codec(&h.b.codec),
            bh = h.b.content_hint,
            bl = h.b.latency,
            bhw = short_hw(&h.b.hw),
            bn = h.b_bytes,
            btm = h.b_time,
            dc = dcls,
            ds = dsign,
            dp = a_delta,
        )));
    }
    format_str(format_args!(
        r#"<div class="vt-history-title">История пар (последние {n}): A vs B. Δ% = B(bytes) от A(bytes). Отрицательное — B жмёт лучше.</div>
           <table>
             <thead><tr><th>#</th><th>Src</th>
               <th>Ac</th><th>Ah</th><th>Al</th><th>Ahw</th><th>A bytes</th><th>A ms</th>
               <th>Bc</th><th>Bh</th><th>Bl</th><th>Bhw</th><th>B bytes</th><th>B ms</th>
               <th>Δ</th>
             </tr></thead>
             <tbody>{rows}</tbody>
           </table>"#,
        n = n,
        rows = rows,
    ))
}

fn update_history(doc: &Document, st: &DemoState) {
    let Some(el) = doc.get_element_by_id("vt-history") else {
        return;
    };
    el.set_inner_html(&render_history(st));
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>Side-by-side</strong>: два независимых <code>VideoEncoder</code> работают в одном проходе. Кадр рисуется один раз, клонируется через <code>VideoFrame.clone()</code>, и отправляется в оба. Сравнение честное — оба видят идентичный вход.",
        "<strong>isConfigSupported</strong> (кнопка <em>Test configs</em>) прогоняет 4 кодека на каждом слоте с его параметрами. Если текущий кодек слота не поддержан — берётся первый доступный.",
        "<strong>latencyMode / bitrateMode / contentHint / hardwareAcceleration / scalabilityMode / alpha</strong> — шесть подсказок WebCodecs. <code>contentHint</code> — подсказка, не команда: некоторые кодеки её игнорируют. <code>scalabilityMode</code> — SVC, только для VP9/AV1. <code>alpha</code> — сохранение альфа-канала.",
        "<strong>Δ%</strong> в истории — относительный размер B от A. Отрицательное — B жмёт лучше. <strong>История</strong> хранит последние 5 пар для сравнения тренда.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("vt-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("vt-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Рисование source
// =============================================================================

fn draw_motion(ctx: &CanvasRenderingContext2d, w: u32, h: u32, phase: f64) {
    let ctx_js: &JsValue = ctx.as_ref();
    let pi2 = std::f64::consts::PI * 2.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#0f2027"));
    let _ = invoke(ctx_js, "fillRect", &[num(0.0), num(0.0), num(w as f64), num(h as f64)]);
    let cx = w as f64 / 2.0 + (phase * pi2).sin() * 100.0;
    let cy = h as f64 / 2.0 + (phase * pi2).cos() * 70.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#f0883e"));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(ctx_js, "arc", &[num(cx), num(cy), num(32.0), num(0.0), num(pi2)]);
    let _ = invoke(ctx_js, "fill", &[]);
}

fn draw_text(ctx: &CanvasRenderingContext2d, w: u32, h: u32, phase: f64) {
    let ctx_js: &JsValue = ctx.as_ref();
    let pi2 = std::f64::consts::PI * 2.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#0d1117"));
    let _ = invoke(ctx_js, "fillRect", &[num(0.0), num(0.0), num(w as f64), num(h as f64)]);
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#c9d1d9"));
    let _ = set_prop(ctx_js, "font", &JsValue::from_str("14px monospace"));
    let _ = invoke(ctx_js, "fillText", &[JsValue::from_str("Hello, WebCodecs!"), num(20.0), num(40.0)]);
    let _ = invoke(ctx_js, "fillText", &[JsValue::from_str("contentHint = text"), num(20.0), num(70.0)]);
    let _ = invoke(ctx_js, "fillText", &[JsValue::from_str("Резкие границы букв"), num(20.0), num(100.0)]);
    let cx = w as f64 - 40.0 + (phase * pi2).sin() * 20.0;
    let _ = set_prop(ctx_js, "fillStyle", &JsValue::from_str("#3fb950"));
    let _ = invoke(ctx_js, "beginPath", &[]);
    let _ = invoke(ctx_js, "arc", &[num(cx), num(30.0), num(8.0), num(0.0), num(pi2)]);
    let _ = invoke(ctx_js, "fill", &[]);
}

fn draw_source(ctx: &CanvasRenderingContext2d, w: u32, h: u32, phase: f64, source: &str) {
    match source {
        "text" => draw_text(ctx, w, h, phase),
        _ => draw_motion(ctx, w, h, phase),
    }
}

fn draw_frame_at(ctx: &CanvasRenderingContext2d, w: u32, h: u32, idx: u32, source: &str) {
    let phase = idx as f64 / FRAME_COUNT as f64;
    draw_source(ctx, w, h, phase, source);
}

// =============================================================================
// VideoFrame helpers
// =============================================================================

fn make_frame_from_canvas(
    canvas: &HtmlCanvasElement,
    timestamp_us: f64,
) -> Result<JsValue, JsValue> {
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

/// `VideoFrame.clone()` — дубликат кадра.
/// `VideoEncoder.encode()` забирает владение фреймом, поэтому для
/// параллельного прохода нужен отдельный экземпляр на каждый энкодер.
fn clone_frame(frame: &JsValue) -> Result<JsValue, JsValue> {
    let f = get_fn(frame, "clone")?;
    f.call0(frame)
}

// =============================================================================
// Параметры
// =============================================================================

fn read_select(doc: &Document, id: &str, default: &str) -> String {
    doc.get_element_by_id(id)
        .and_then(|e| e.dyn_into::<HtmlSelectElement>().ok())
        .map(|s| s.value())
        .filter(|v| !v.is_empty())
        .unwrap_or_else(|| default.to_string())
}

struct Params {
    source: String,
    bitrate_mode: String,
    a: SlotParams,
    b: SlotParams,
}

fn read_slot_params(doc: &Document, p: &str) -> SlotParams {
    SlotParams {
        codec: read_select(
            doc,
            &format_str(format_args!("vt-{}-codec", p)),
            CODECS[0].0,
        ),
        latency: read_select(
            doc,
            &format_str(format_args!("vt-{}-latency", p)),
            LATENCY_MODES[0],
        ),
        content_hint: read_select(
            doc,
            &format_str(format_args!("vt-{}-hint", p)),
            CONTENT_HINTS[0],
        ),
        hw: read_select(
            doc,
            &format_str(format_args!("vt-{}-hw", p)),
            HW_ACCELS[0],
        ),
        scalability: read_select(
            doc,
            &format_str(format_args!("vt-{}-scal", p)),
            SCALABILITY_MODES[0],
        ),
        alpha: read_select(
            doc,
            &format_str(format_args!("vt-{}-alpha", p)),
            ALPHA_MODES[0],
        ),
    }
}

fn read_params(doc: &Document) -> Params {
    Params {
        source: read_select(doc, "vt-source-type", "motion"),
        bitrate_mode: read_select(doc, "vt-bitrate-mode", BITRATE_MODES[0]),
        a: read_slot_params(doc, "a"),
        b: read_slot_params(doc, "b"),
    }
}

fn opt_str(s: &str) -> Option<&str> {
    if s.is_empty() || s == "(off)" {
        None
    } else {
        Some(s)
    }
}

/// Строит `TuningConfig` из `SlotParams` и shared `bitrate_mode`.
fn make_cfg<'a>(slot: &'a SlotParams, bitrate_mode: &'a str) -> TuningConfig<'a> {
    TuningConfig {
        codec: &slot.codec,
        width: FRAME_W,
        height: FRAME_H,
        bitrate: BITRATE,
        framerate: FRAMERATE,
        latency_mode: Some(&slot.latency),
        bitrate_mode: Some(bitrate_mode),
        content_hint: Some(&slot.content_hint),
        hardware_acceleration: Some(&slot.hw),
        scalability_mode: opt_str(&slot.scalability),
        alpha: opt_str(&slot.alpha),
    }
}

fn sync_state_from_selects(doc: &Document, st: &Rc<RefCell<DemoState>>) {
    let p = read_params(doc);
    let mut s = st.borrow_mut();
    s.source = p.source;
    s.bitrate_mode = p.bitrate_mode;
    s.a = p.a;
    s.b = p.b;
}

fn set_select(doc: &Document, id: &str, value: &str) {
    if let Some(sel) = doc
        .get_element_by_id(id)
        .and_then(|e| e.dyn_into::<HtmlSelectElement>().ok())
    {
        sel.set_value(value);
    }
}

// =============================================================================
// Параллельный encode двух конфигураций
// =============================================================================

fn build_slot_result(types: Vec<String>, sizes: Vec<usize>, time_ms: f64) -> SlotResult {
    let key_count = types.iter().filter(|t| *t == "key").count() as u32;
    let delta_count = types.len() as u32 - key_count;
    let key_bytes: usize = types
        .iter()
        .zip(sizes.iter())
        .filter(|(t, _)| *t == "key")
        .map(|(_, b)| *b)
        .sum();
    let delta_bytes: usize = types
        .iter()
        .zip(sizes.iter())
        .filter(|(t, _)| *t == "delta")
        .map(|(_, b)| *b)
        .sum();
    let total_bytes = key_bytes + delta_bytes;
    let peak_bytes = sizes.iter().copied().max().unwrap_or(0);
    let chunks = types
        .iter()
        .zip(sizes.iter())
        .map(|(t, b)| ChunkInfo {
            ty: t.clone(),
            bytes: *b,
        })
        .collect();
    SlotResult {
        chunks,
        key_count,
        delta_count,
        key_bytes,
        delta_bytes,
        total_bytes,
        peak_bytes,
        encode_time_ms: time_ms,
    }
}

async fn do_encode_pair(
    doc: Document,
    cfg_a: &TuningConfig<'_>,
    cfg_b: &TuningConfig<'_>,
    source: &str,
) -> Result<(SlotResult, SlotResult), JsValue> {
    // Хранилища чанков для двух энкодеров.
    let a_types: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let a_sizes: Rc<RefCell<Vec<usize>>> = Rc::new(RefCell::new(Vec::new()));
    let b_types: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));
    let b_sizes: Rc<RefCell<Vec<usize>>> = Rc::new(RefCell::new(Vec::new()));

    // Колбэки энкодера A.
    let a_types_cb = a_types.clone();
    let a_sizes_cb = a_sizes.clone();
    let a_on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, _meta: JsValue| {
            a_types_cb.borrow_mut().push(chunk_type(&chunk));
            a_sizes_cb.borrow_mut().push(chunk_byte_length(&chunk));
        },
    );
    let a_on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |e: JsValue| {
            web_sys::console::error_1(&e);
        },
    );

    // Колбэки энкодера B.
    let b_types_cb = b_types.clone();
    let b_sizes_cb = b_sizes.clone();
    let b_on_output = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue, JsValue)>::new(
        move |chunk: JsValue, _meta: JsValue| {
            b_types_cb.borrow_mut().push(chunk_type(&chunk));
            b_sizes_cb.borrow_mut().push(chunk_byte_length(&chunk));
        },
    );
    let b_on_error = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        |e: JsValue| {
            web_sys::console::error_1(&e);
        },
    );

    let encoder_a = make_video_encoder(&a_on_output, &a_on_error)?;
    let encoder_b = make_video_encoder(&b_on_output, &b_on_error)?;

    configure_tuned(&encoder_a, cfg_a)?;
    configure_tuned(&encoder_b, cfg_b)?;

    let source_canvas = doc
        .get_element_by_id("vt-source")
        .ok_or_else(|| JsValue::from_str("no #vt-source"))?
        .dyn_into::<HtmlCanvasElement>()?;
    let source_ctx = source_canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d (source)"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    let t0 = js_sys::Date::now();
    let mut last_yield = t0;

    for i in 0..FRAME_COUNT {
        // 1. Рисуем кадр один раз.
        draw_frame_at(&source_ctx, FRAME_W, FRAME_H, i, source);
        let base_frame =
            make_frame_from_canvas(&source_canvas, i as f64 * FRAME_DURATION_US)?;

        // 2. Клонируем ДО отправки — encode() забирает владение.
        let frame_a = clone_frame(&base_frame)?;
        let frame_b = clone_frame(&base_frame)?;
        let _ = frame_close(&base_frame);

        // 3. Отправляем в оба энкодера.
        encoder_encode(&encoder_a, &frame_a, i == 0)?;
        encoder_encode(&encoder_b, &frame_b, i == 0)?;

        // 4. Если encode() не закрыл фреймы — закрываем вручную.
        // Ошибка игнорируется: закрытый фрейм → InvalidStateError, но это ОК.
        let _ = frame_close(&frame_a);
        let _ = frame_close(&frame_b);

        // 5. Yield раз в 16 мс — не даём UI подвиснуть.
        let now = js_sys::Date::now();
        if now - last_yield >= YIELD_EVERY_MS {
            last_yield = now;
            TimeoutFuture::new(0).await;
        }
    }

    // Оба flush'а последовательно — энкодеры работают параллельно
    // на уровне движка, JS-ожидание одного не блокирует другой.
    encoder_flush(&encoder_a).await?;
    encoder_flush(&encoder_b).await?;

    let t1 = js_sys::Date::now();
    let elapsed = t1 - t0;

    let _ = encoder_close(&encoder_a);
    let _ = encoder_close(&encoder_b);
    a_on_output.forget();
    a_on_error.forget();
    b_on_output.forget();
    b_on_error.forget();

    let a_types_v = a_types.borrow().clone();
    let a_sizes_v = a_sizes.borrow().clone();
    let b_types_v = b_types.borrow().clone();
    let b_sizes_v = b_sizes.borrow().clone();

    let a_res = build_slot_result(a_types_v, a_sizes_v, elapsed);
    let b_res = build_slot_result(b_types_v, b_sizes_v, elapsed);
    Ok((a_res, b_res))
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    if has_sw_controller() {
        show_sw_notice(&document)?;
        unregister_sw_and_reload(&document)?;
        return Ok(());
    }

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23.4 — Codec negotiation & tuning (side-by-side)"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_video_encoder_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("vt-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает VideoEncoder. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // ---- Source canvas ----
    let canvases = document.create_element("div")?;
    canvases.set_class_name("vt-canvas-col");
    canvases.set_inner_html(&format_str(format_args!(
        r#"<div class="vt-canvas-label">Source (общий вход для обоих энкодеров)</div>
           <canvas id="vt-source" class="vt-canvas" width="{w}" height="{h}"></canvas>"#,
        w = FRAME_W,
        h = FRAME_H,
    )));
    root.append_child(&canvases)?;

    // ---- Общий блок: bitrateMode, source, кнопки ----
    let shared = document.create_element("div")?;
    shared.set_class_name("vt-block");
    shared.set_inner_html(&format_str(format_args!(
        r#"<label class="vt-label">BitrateMode:</label>
           <select class="vt-select" id="vt-bitrate-mode">{bm_opts}</select>
           <label class="vt-label">Source:</label>
           <select class="vt-select" id="vt-source-type">
               <option value="motion" selected>motion</option>
               <option value="text">text</option>
           </select>
           <button class="vt-btn vt-btn-primary" id="vt-test" title="isConfigSupported для всех кодеков на обоих слотах">Test configs</button>
           <button class="vt-btn vt-btn-primary" id="vt-run" title="Параллельный encode обоих слотов в одном проходе">Encode Both</button>
           <button class="vt-btn" id="vt-pause">Pause</button>"#,
        bm_opts = options_str(BITRATE_MODES, BITRATE_MODES[0]),
    )));
    root.append_child(&shared)?;

    // ---- Общий stats-блок (фаза + поддержка) ----
    // ВАЖНО: класс vt-stats, а не vt-block, иначе внутренние
    // спаны .label/.value/.ok не получат цветов из CSS.
    let stats = document.create_element("div")?;
    stats.set_class_name("vt-stats");
    stats.set_id("vt-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // ---- Пара слотов ----
    let defaults_a = SlotParams::defaults_a();
    let defaults_b = SlotParams::defaults_b();
    let pair = document.create_element("div")?;
    pair.set_class_name("vt-pair");
    pair.set_inner_html(&format_str(format_args!(
        r#"<div class="vt-slot">
             <div class="vt-slot-title-a">A — baseline</div>
             {controls_a}
             <div class="vt-stats" id="vt-stats-a">—</div>
             <div class="vt-chart" id="vt-chart-a"></div>
           </div>
           <div class="vt-slot">
             <div class="vt-slot-title-b">B — challenger</div>
             {controls_b}
             <div class="vt-stats" id="vt-stats-b">—</div>
             <div class="vt-chart" id="vt-chart-b"></div>
           </div>"#,
        controls_a = slot_controls_html("a", &defaults_a),
        controls_b = slot_controls_html("b", &defaults_b),
    )));
    root.append_child(&pair)?;

    // ---- История ----
    let history = document.create_element("div")?;
    history.set_class_name("vt-history");
    history.set_id("vt-history");
    root.append_child(&history)?;

    // ---- Footnote ----
    let footnote = document.create_element("p")?;
    footnote.set_class_name("vt-footnote");
    footnote.set_text_content(Some(
        "Два независимых VideoEncoder в одном проходе. Кадр рисуется один раз, \
         клонируется через VideoFrame.clone() и отправляется в оба. Сравнение честное. \
         Изменяйте параметры A и/или B, нажимайте Encode Both. \
         Δ% в истории — относительный размер B от A; отрицательное — B жмёт лучше. \
         contentHint — подсказка, не команда: некоторые кодеки её игнорируют. \
         scalabilityMode (L1T2, …) — SVC, только для VP9/AV1. \
         alpha (discard/keep) — сохранение альфа-канала. \
         Round-trip decode — тема 23.3, здесь только encode.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    // ---------- change listeners на все селекты ----------
    //
    // Любая смена параметров делает невалидными:
    //   • результаты прошлого Encode (они для старых параметров);
    //   • support-списки (они для старых параметров);
    //   • phase.
    // Поэтому: обновляем DemoState из селектов, сбрасываем
    // результаты и поддержку, ставим подсказку и перерисовываем UI.
    {
        let doc_c = document.clone();
        let st_c = state.clone();
        let handler = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            sync_state_from_selects(&doc_c, &st_c);
            {
                let mut s = st_c.borrow_mut();
                s.clear_results();
                s.a_support.clear();
                s.b_support.clear();
                s.phase = String::from(
                    "параметры изменены — нажмите Test configs / Encode Both",
                );
            }
            let s = st_c.borrow();
            update_stats(&doc_c, &s);
            update_all_slot_ui(&doc_c, &s);
            // history не трогаем — это летопись прошлых запусков.
        });
        let handler_fn = handler.as_ref().unchecked_ref::<js_sys::Function>();
        let ids = [
            "vt-bitrate-mode",
            "vt-source-type",
            "vt-a-codec", "vt-a-latency", "vt-a-hint", "vt-a-hw", "vt-a-scal", "vt-a-alpha",
            "vt-b-codec", "vt-b-latency", "vt-b-hint", "vt-b-hw", "vt-b-scal", "vt-b-alpha",
        ];
        for id in ids {
            if let Some(el) = document.get_element_by_id(id) {
                if let Ok(el) = el.dyn_into::<HtmlElement>() {
                    el.add_event_listener_with_callback("change", handler_fn)?;
                }
            }
        }
        handler.forget();
    }

    // ---------- Test configs ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("vt-test")
            .ok_or_else(|| JsValue::from_str("no #vt-test"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                let p = read_params(&doc);
                let current_a = p.a.codec.clone();
                let current_b = p.b.codec.clone();

                let cfg_a = make_cfg(&p.a, &p.bitrate_mode);
                let results_a = check_codecs(&cfg_a).await;

                let cfg_b = make_cfg(&p.b, &p.bitrate_mode);
                let results_b = check_codecs(&cfg_b).await;

                // Автовыбор, если текущий кодек слота не поддержан.
                let a_sup = results_a
                    .iter()
                    .find(|r| r.codec == current_a)
                    .map(|r| r.supported)
                    .unwrap_or(false);
                let b_sup = results_b
                    .iter()
                    .find(|r| r.codec == current_b)
                    .map(|r| r.supported)
                    .unwrap_or(false);

                let mut phase = String::from("config проверен");
                let mut a_pick: Option<String> = None;
                let mut b_pick: Option<String> = None;

                if !a_sup {
                    if let Some(first) = results_a.iter().find(|r| r.supported) {
                        a_pick = Some(first.codec.clone());
                    }
                }
                if !b_sup {
                    if let Some(first) = results_b.iter().find(|r| r.supported) {
                        b_pick = Some(first.codec.clone());
                    }
                }

                if let Some(c) = &a_pick {
                    set_select(&doc, "vt-a-codec", c);
                }
                if let Some(c) = &b_pick {
                    set_select(&doc, "vt-b-codec", c);
                }

                match (a_pick.as_deref(), b_pick.as_deref()) {
                    (Some(a), Some(b)) => {
                        phase = format_str(format_args!(
                            "config проверен (A авто-выбор: {a}, B авто-выбор: {b})"
                        ));
                    }
                    (Some(a), None) => {
                        phase = format_str(format_args!(
                            "config проверен (A авто-выбор: {a})"
                        ));
                    }
                    (None, Some(b)) => {
                        phase = format_str(format_args!(
                            "config проверен (B авто-выбор: {b})"
                        ));
                    }
                    (None, None) => {}
                }

                {
                    let mut s = st.borrow_mut();
                    s.a_support = results_a;
                    s.b_support = results_b;
                    s.phase = phase;
                }
                sync_state_from_selects(&doc, &st);
                update_stats(&doc, &st.borrow());
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- Encode Both ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("vt-run")
            .ok_or_else(|| JsValue::from_str("no #vt-run"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                let p = read_params(&doc);
                let source = p.source.clone();

                {
                    let mut s = st.borrow_mut();
                    if s.running {
                        return;
                    }
                    s.running = true;
                    s.clear_results();
                    s.phase = String::from("проверка config…");
                }
                let _guard = RunGuard(st.clone());
                sync_state_from_selects(&doc, &st);
                update_stats(&doc, &st.borrow());
                update_all_slot_ui(&doc, &st.borrow());

                let cfg_a = make_cfg(&p.a, &p.bitrate_mode);
                let cfg_b = make_cfg(&p.b, &p.bitrate_mode);

                // Pre-check обоих слотов.
                let ok_a = check_one_codec(&p.a.codec, &cfg_a).await.unwrap_or(false);
                let ok_b = check_one_codec(&p.b.codec, &cfg_b).await.unwrap_or(false);

                if !ok_a || !ok_b {
                    let mut s = st.borrow_mut();
                    let bad = if !ok_a && !ok_b {
                        format_str(format_args!(
                            "A: {a} и B: {b} не поддержаны",
                            a = p.a.codec,
                            b = p.b.codec,
                        ))
                    } else if !ok_a {
                        format_str(format_args!("A: {a} не поддержан", a = p.a.codec))
                    } else {
                        format_str(format_args!("B: {b} не поддержан", b = p.b.codec))
                    };
                    s.error = Some(format_str(format_args!(
                        "{bad}. Нажмите Test configs для авто-выбора."
                    )));
                    s.phase = String::from("не поддерживается");
                    update_stats(&doc, &s);
                    return;
                }

                {
                    let mut s = st.borrow_mut();
                    s.phase = String::from("encode…");
                }
                update_stats(&doc, &st.borrow());

                let (a_res, b_res) =
                    match do_encode_pair(doc.clone(), &cfg_a, &cfg_b, &source).await {
                        Ok(v) => v,
                        Err(e) => {
                            let mut s = st.borrow_mut();
                            s.error = Some(format_str(format_args!("encode: {e:?}")));
                            s.phase = String::from("ошибка");
                            update_stats(&doc, &s);
                            return;
                        }
                    };

                // Снимок для истории — до перезаписи state.
                let snap = PairSnapshot {
                    source: source.clone(),
                    a: p.a.clone(),
                    b: p.b.clone(),
                    a_bytes: a_res.total_bytes,
                    a_time: a_res.encode_time_ms,
                    b_bytes: b_res.total_bytes,
                    b_time: b_res.encode_time_ms,
                };

                {
                    let mut s = st.borrow_mut();
                    s.a_result = a_res;
                    s.b_result = b_res;
                    s.push_history(snap);
                    s.phase = String::from("готово");
                }
                update_stats(&doc, &st.borrow());
                update_all_slot_ui(&doc, &st.borrow());
                update_history(&doc, &st.borrow());
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- Pause ----------
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("vt-pause")
            .ok_or_else(|| JsValue::from_str("no #vt-pause"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;

        let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
            let paused = {
                let mut s = st.borrow_mut();
                s.paused = !s.paused;
                s.paused
            };
            if let Some(el) = doc.get_element_by_id("vt-pause") {
                el.set_text_content(Some(if paused { "Resume" } else { "Pause" }));
            }
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---------- rAF: анимация source ----------
    {
        let doc_for_raf = document.clone();
        let source_canvas = doc_for_raf
            .get_element_by_id("vt-source")
            .ok_or_else(|| JsValue::from_str("no #vt-source"))?
            .dyn_into::<HtmlCanvasElement>()?;
        let source_ctx = source_canvas
            .get_context("2d")?
            .ok_or_else(|| JsValue::from_str("no 2d (source)"))?
            .dyn_into::<CanvasRenderingContext2d>()?;

        let cell: Rc<RefCell<Option<wasm_bindgen::closure::Closure<dyn FnMut(f64)>>>> =
            Rc::new(RefCell::new(None));
        let cell2 = cell.clone();
        let st = state.clone();

        *cell.borrow_mut() = Some(wasm_bindgen::closure::Closure::wrap(Box::new(
            move |now: f64| {
                if let Some(w) = web_sys::window() {
                    if let Some(cb) = cell2.borrow().as_ref() {
                        let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
                    }
                }
                if st.borrow().paused {
                    return;
                }
                let phase = (now / 1000.0) % 1.0;
                let source = st.borrow().source.clone();
                draw_source(&source_ctx, FRAME_W, FRAME_H, phase, &source);
            },
        ) as Box<dyn FnMut(f64)>));

        let cb = cell.borrow();
        window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    }

    sync_state_from_selects(&document, &state);
    {
        let st = state.borrow();
        update_stats(&document, &st);
        update_all_slot_ui(&document, &st);
        update_history(&document, &st);
    }

    Ok(())
}
```

---

## 23.4.7. Анализ решения

**Ценность демо.**

1. **Side-by-side — честное сравнение.** Один вход, два энкодера, идентичные кадры (через `VideoFrame.clone()`). Разницу в размере и времени можно приписать только конфигурации.
2. **Все шесть подсказок WebCodecs.** `latencyMode`, `bitrateMode`, `contentHint`, `hardwareAcceleration`, `scalabilityMode`, `alpha` — каждое значение можно проверить.
3. **История 5 пар с колонкой `Src`.** Видно не только что сравнивалось, но и на каком контенте.
4. **Δ% B от A** — числовое сравнение в одной колонке.
5. **Δeff** — key/avg_delta. Работает ли delta-кодирование в каждом слоте.

**Правила проекта, применённые.**

- **Отдельные функции для строк** — `line_err`, `line_phase`, `line_support`, `render_slot_stats`, `render_slot_history`, `bar_html`, `render_slot_chart`, `render_history`.
- **`format_str(format_args!(...))` в tail-позициях.**
- **`Rc<RefCell<DemoState>>`** — общее состояние между обработчиками и rAF.
- **`RunGuard`** — защита от параллельных запусков.
- **Один `change`-listener** на 14 селектов через `as_ref().unchecked_ref::<js_sys::Function>()`.
- **Yield раз в 16 мс** — не блокирует UI во время encode.
- **Префикс `vt-`** — не пересекается с `vd-`, `vf-`, `ve-`, `cd-`.
- **Никаких `unused imports`** — все задействованы.
- **`r##"..."##` в `bar_html`** — там `title="#N ..."` внутри.
- **`create_element`, не `createElement`** — везде.

**Найденные в практике проблемы и их решения.**

1. **`const DESC_LINES` с `.to_string().as_str()`** — не компилируется: `to_string` не const, `as_str` возвращает ссылку на временный. **Решение:** массив строк живёт локально внутри `build_description`.
2. **`TuningConfig` вместо 10 параметров подряд** — сигнатуры `configure_tuned` / `check_codecs` / `check_one_codec` были переведены на одну структуру. Это устранило mismatch между модулем и app.rs.
3. **Смена параметров не давала видимого эффекта.** Change-listener обновлял только `DemoState`, но не DOM. **Решение:** обработчик теперь ещё и очищает результаты/support и вызывает `update_stats` + `update_all_slot_ui`.
4. **Shared stats-блок был нечитаем.** Класс контейнера — `vt-block`, а все цветовые правила — через `.vt-stats .label` и т. п. Селектор не матчился. **Решение:** класс контейнера — `vt-stats`, CSS-правило восстановлено с `padding` / `font-size` / `line-height`.
5. **`SlotResult::key_bytes` / `delta_bytes` не читались** → dead_code. **Решение:** добавил Δeff в slot-stats.
6. **`PairSnapshot::source` не читался** → dead_code. **Решение:** колонка `Src` в таблице истории.
7. **`has_result_placeholder`** — остался от предыдущей итерации. **Решение:** удалён, `has_result` заменён на `phase` + `history.is_empty()`.

**Проверки корректности.**

- Все `format_args!` без `f`-суффикса: `{pf:.1}`, `{ms:.0}`, `{r:.1}`, `{e:.1}`, `{dp:.1}`. Валидны.
- Raw-строки: `r#"..."#` и `r##"..."##` правильно экранированы.
- `TuningConfig` конструируется через `make_cfg` — все lifetimes корректны.
- `frame_close` на `base_frame` после `clone` — оригинал закрывается, клоны живут независимо.
- `frame_close` на `frame_a` / `frame_b` после `encode` — если движок уже закрыл, ошибка игнорируется (`let _ =`).
- В `change`-listener: `set_select` не эмитит `change` событие, поэтому автовыбор кодека не приведёт к каскадной очистке.
- `RunGuard` устанавливается **после** `s.running = true` — порядок критичен, иначе Drop сбросит флаг сразу.

---

## 23.4.8. Wow-момент на практике

**Первый.** На motion-сцене H.264 vs VP8 показывают `Δ% = -3.9%` — VP8 немного компактнее, но в 1.8 раза медленнее. Численное, честное сравнение.

**Второй.** На текстовом контенте с `contentHint = text` и `BitrateMode = constant` — H.264 выдаёт 4103 байта, VP8 — 5180. Разница +26.3% в пользу H.264 на тексте. Неочевидный результат, видный только через side-by-side.

**Третий.** AV1 в Chrome **игнорирует contentHint** — при переключении motion/detail/text размер не меняется. Демо показывает это явно: строка `contentHint проигнорирован кодеком` в истории.

**Четвёртый.** Δeff для H.264 на motion-сцене ≈ 5.2×. Для той же H.264 на текстовом контенте Δeff ≈ 20× (key содержит текст целиком, delta почти пустые — движется только маленький зелёный кружок).

**Пятый.** История за 5 запусков: можно увидеть, как AV1 при 30 кадрах тратит 380 мс, а H.264 — 69 мс. Числа рядом, решение — за пользователем.

---

## 23.4.9. Итог

**`VideoEncoder.isConfigSupported`** — обязательный шаг перед `configure`. Позволяет выбрать лучший доступный кодек и показать пользователю, что поддерживается.

**Шесть подсказок WebCodecs.** `latencyMode` (quality/realtime), `bitrateMode` (variable/constant), `contentHint` (motion/detail/text), `hardwareAcceleration` (no-preference/prefer-hardware/prefer-software), `scalabilityMode` (SVC, только VP9/AV1), `alpha` (discard/keep).

**Side-by-side через `VideoFrame.clone()`.** Один вход → два энкодера → честное сравнение. `encode()` забирает владение, поэтому нужен отдельный экземпляр на энкодер.

**История 5 пар с `Src`, `Δ%`, `Δeff`.** Числовое сравнение в одной таблице. `Δ%` — относительный размер B от A. `Δeff` — эффективность delta-кодирования в слоте.

**Практические находки.** `isConfigSupported` не падает при неподдерживаемом hint — возвращает `supported: false`. `prefer-software` работает всегда, `prefer-hardware` — только с GPU. `contentHint` — подсказка, а не команда: AV1 в Chrome её игнорирует на текстовом кадре.

**Правило проекта.** При смене параметров нужно сбрасывать невалидные результаты — иначе UI вводит пользователя в заблуждение. History сохраняется: она относится к прошлым запускам с их собственными параметрами.

**Дальше — 23.5 (WebCodecs: полный pipeline)** — реальный видеофайл → декодер → фильтр → энкодер → файл.