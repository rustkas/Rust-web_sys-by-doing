# Глава 23.11. Muxing: EncodedVideoChunk → MP4

## 23.11.0. Обновлённая структура

```
src/
  container_parser.rs             ← из 23.9 (без изменений)
  mp4_demuxer.rs                  ← из 23.10 (без изменений)
  mp4_muxer.rs                    ← НОВЫЙ модуль 23.11
  video_frame.rs                  ← из 23.1 (format_str, set_prop)
  app.rs                          ← демо 23.11 (заменяется целиком)

tests/
  wasm_container_parser.rs        ← из 23.9
  wasm_mp4_demuxer.rs             ← из 23.10
  wasm_mp4_muxer.rs               ← НОВЫЙ тест (11 тестов)

src/lib.rs                        ← +1 строка: pub mod mp4_muxer;
```

**Комментарий.** Модуль `mp4_muxer.rs` — обратная операция к 23.10. Demuxer читал box-ы, muxer их пишет. `ftyp` → `moov` (mvhd + trak + mdia + minf + stbl) → `mdat`. Ключевая сложность — `stco` зависит от размера `moov`, но `moov` содержит `stco`. Решение — двухпроходная сборка. Префикс CSS — `mx-` (muxer), не пересекается с `dm-` (23.10), `cp-` (23.9), `ms-` (23.8), `vf-` (23.1).

**Ссылки для детального изучения:**

- ISO/IEC 14496-12: [ISO Base Media File Format](https://www.iso.org/standard/68960.html)
- Apple QuickTime File Format: [Movie Atoms](https://developer.apple.com/documentation/quicktime-file-format/movie_atoms)
- MP4RA: [Registered Box Types](https://mp4ra.org/#/atoms)
- W3C WebCodecs: [`EncodedVideoChunk`](https://www.w3.org/TR/webcodecs/#encodedvideochunk-interface)
- W3C WebCodecs: [`VideoEncoder`](https://www.w3.org/TR/webcodecs/#videoencoder-interface)

---

## 23.11.1. О чём эта подглава

23.10 закончилась на **чтении** MP4: файл → таблицы `stbl` → список `VideoSample`. Подглава 23.11 делает обратное: **список `EncodedVideoChunk` → валидный MP4-файл**.

**Что нужно, чтобы файл был валидным:**

1. **`ftyp`** — 32 байта, сигнатура + бренды.
2. **`moov`** — полное описание трека: `mvhd`, `trak`, `stbl` со всеми таблицами.
3. **`mdat`** — конкатенация всех encoded chunks в исходном порядке.

**Сложность, которая не видна снаружи:** `stco` содержит абсолютные offset-ы chunk-ов в файле. Первый offset = `ftyp_size + moov_size + 8` (8 — размер заголовка `mdat`). Но `moov_size` зависит от размера `stco`, а `stco` уже содержит этот offset. Циклическая зависимость.

**Решение — двухпроходная сборка:**

1. Собрать `moov` со **start-offset = 0**. Размер `moov` не зависит от содержимого `stco` — только от количества chunk-ов (одна u32 на каждый).
2. Вычислить `chunk_offset = ftyp.len() + moov.len() + 8`.
3. Пересобрать `moov` с реальным offset. Размер тот же.

**Подглава отвечает на вопросы:**

- Как построить каждый из ~15 обязательных box-ов MP4?
- Как из `avcC` (SPS/PPS) построить `stsd` с `avc1`?
- Как решить циклическую зависимость `stco ↔ moov_size`?
- Что делать, если Chrome не отдал `description` (SPS/PPS)?
- Как проверить, что получился валидный MP4?

**Демо:** пользователь кликает «Сгенерировать 30 кадров». Canvas рисует 30 анимационных кадров, `VideoEncoder` кодирует их в H.264. Muxer собирает MP4, `Blob` + `URL.createObjectURL` открывает результат в `<video>`. Дополнительные панели: Summary, Timing, Layout-bar, Hex-превью, Дерево, Round-trip.

---

## 23.11.2. Структура MP4-файла

```
┌──────────────────────────────────────┐
│ ftyp  — 32 B                          │  сигнатура + brands
├──────────────────────────────────────┤
│ moov                                  │  все метаданные
│   ├─ mvhd  (108 B)                    │    timescale, duration
│   └─ trak                             │
│       ├─ tkhd  (92 B)                 │    track_id, width, height
│       └─ mdia                         │
│           ├─ mdhd  (32 B)             │    media timescale
│           ├─ hdlr  (45 B)             │    handler_type = 'vide'
│           └─ minf                     │
│               ├─ vmhd  (20 B)         │    video media header
│               ├─ dinf → dref → url    │    data reference (self)
│               └─ stbl                 │
│                   ├─ stsd (130 B)     │    avc1 + avcC
│                   ├─ stts (24 B)      │    durations
│                   ├─ stsc (28 B)      │    samples-to-chunk
│                   ├─ stsz (20+4N B)   │    sample sizes
│                   ├─ stco (20 B)      │    chunk offset (1 chunk)
│                   └─ stss (12+4K B)   │    key frame indices
├──────────────────────────────────────┤
│ mdat                                  │  конкатенация всех chunks
│   [chunk 0][chunk 1]...[chunk N-1]    │
└──────────────────────────────────────┘
```

**Все chunk-и в один chunk MP4-а.** Это самый простой вариант: `stsc = [(1, N, 1)]`, `stco = [mdat_body_start]`. Разбиение на несколько chunk-ов улучшает производительность на больших файлах, но для 30 кадров не нужно.

---

## 23.11.3. Двухпроходная сборка `moov`

```rust
// 1. Строим moov с offset=0 для измерения.
let moov_stub = build_moov(chunks, config, 0);
// 2. Считаем реальный offset начала mdat body.
let mdat_body_start = (FTYP_SIZE + moov_stub.len() + 8) as u32;
// 3. Пересобираем moov с правильным offset. Размер тот же.
let moov = build_moov(chunks, config, mdat_body_start);
debug_assert_eq!(moov.len(), moov_stub.len());
```

**Правило.** В бинарных форматах, где есть обратные ссылки, сначала фиксируйте размеры, потом значения.

---

## 23.11.4. Решения по итогам практической работы

Этот раздел — самое ценное: конкретные ошибки, найденные при отладке на реальном Chrome, и их исправления.

### Баг 1. Chrome не отдаёт `description` (SPS/PPS)

**Симптом.** После кодирования 30 кадров `desc_cell` пустой, muxer падает с ошибкой «не получен description от VideoEncoder».

**Причина.** По умолчанию Chrome использует для H.264 формат **AnnexB** — SPS/PPS встраиваются в сам поток как start codes (`00 00 00 01`), а не выносятся отдельно в `metadata.decoderConfig.description`. Для MP4 нужен **AVCC** — SPS/PPS в отдельном box-е `avcC`, а сами NAL-и в chunks предварены length-prefix.

**Решение.** Двухуровневое:

1. Явно попросить Chrome использовать AVCC через `avc: { format: 'avc' }` в конфиге:
   ```rust
   let avc = Object::new();
   set_prop(&avc, "format", &JsValue::from_str("avc"))?;
   set_prop(&config, "avc", &avc)?;
   ```

2. **Fallback**, если Chrome всё равно не отдал `description`: извлечь SPS/PPS из первого key chunk (в AnnexB они там есть) и сконвертировать все chunks в AVCC:
   ```rust
   let nals = parse_annexb_nals(&first_key.data);
   // Ищем SPS (type 7) и PPS (type 8) среди NAL-ов.
   // Собираем avcC вручную: [1, profile, compat, level, 0xFF, 0xE1, ...]
   ```

**Правило.** Не полагайтесь на опциональные поля WebCodecs API. Если поле помечено как optional в спецификации — нужен fallback.

### Баг 2. `[u8]` вместо `Vec<u8>` в структурах

**Симптом.** Ошибка `expected Vec<u8>, found [u8]` при попытке сохранить `desc_cell` и `chunks_cell`.

**Причина.** `Uint8Array::copy_to(&mut Vec<u8>)` требует неявного приведения `&mut Vec<u8>` к `&mut [u8]`, которое rust-analyzer в WASM-контексте иногда не может вывести. Компилятор видит `copy_to` как метод, работающий с `[u8]`, и выводит тип результата как `[u8]` вместо `Vec<u8>`.

**Решение.** Использовать `Uint8Array::to_vec()` — метод, который **явно возвращает** `Vec<u8>`:

```rust
// Было (нестабильный вывод типа):
let mut buf = vec![0u8; desc_u8.length() as usize];
desc_u8.copy_to(&mut buf);   // ← buf выводится как [u8], а не Vec<u8>

// Стало:
let buf: Vec<u8> = desc_u8.to_vec();
```

То же для чанков:

```rust
let mut data: Vec<u8> = Vec::new();
let copy_to = get_fn(&chunk, "copyTo")?;
if let Ok(copy_fn) = copy_to.dyn_into::<js_sys::Function>() {
    let u8_buf = Uint8Array::new_with_length(byte_length as u32);
    if copy_fn.call1(&chunk, u8_buf.as_ref()).is_ok() {
        data = u8_buf.to_vec();  // ← явный Vec<u8>
    }
}
```

**Правило.** В WASM-коде для получения `Vec<u8>` из `Uint8Array` используйте `to_vec()`. Если нужен `copy_to` — делайте целевую переменную **явно типизированной** `Vec<u8>` и следите за выводом типа rust-analyzer-ом.

### Баг 3. `Uint8Array::from(&mp4[..])` — неоднозначность

**Симптом.** Ошибка `expected Vec<u8>, found [u8]` при создании `Uint8Array::from(&mp4[..])`.

**Причина.** `&mp4[..]` — срез без явного размера. Rust-analyzer в WASM-контексте не может вывести ожидаемый тип для `from`.

**Решение.** Использовать `.as_slice()` — метод, возвращающий `&[u8]` без неоднозначности:

```rust
// Было:
let uint8 = Uint8Array::from(&mp4[..]);

// Стало:
let uint8 = Uint8Array::from(mp4.as_slice());
```

Заодно конвертировать в `JsValue` явно, а не через `.as_ref()`:

```rust
let uint8_js: JsValue = uint8.into();
arr.push(&uint8_js);
```

### Баг 4. `Err(_) => Vec::new()` — не выводится тип

**Симптом.** Ошибка `type annotations needed` в `match parse_mp4(&mp4) { Ok(b) => b, Err(_) => Vec::new() }`.

**Причина.** Rust-analyzer не может вывести `Vec<BoxInfo>` из контекста.

**Решение.** Явная аннотация типа у переменной и в ветке `Err`:

```rust
let boxes: Vec<BoxInfo> = match parse_mp4(&mp4) {
    Ok(b) => b,
    Err(_) => Vec::<BoxInfo>::new(),
};
```

### Баг 5. Chrome отдаёт `description` в другом формате

**Симптом.** `description` приходит не как `Uint8Array`, а как `ArrayBuffer`. Код с `dyn_into::<Uint8Array>()` не срабатывает.

**Причина.** Спецификация WebCodecs разрешает любой `BufferSource` в `description`.

**Решение.** Поддержать оба варианта:

```rust
let buf: Vec<u8> = if let Ok(u8v) = desc_js.clone().dyn_into::<Uint8Array>() {
    u8v.to_vec()
} else if let Ok(ab) = desc_js.clone().dyn_into::<js_sys::ArrayBuffer>() {
    Uint8Array::new(&ab).to_vec()
} else {
    Vec::new()
};
```

### Баг 6. `find_avcc_in_stsd` — unused import

**Симптом.** После добавления аннотации `[avcC = N B]` в дерево оказалось, что `find_avcc_in_stsd` не был добавлен в `use` — warning.

**Решение.** Добавить в импорты и использовать в `box_annotation`.

### Баг 7. `Round-trip: 0.0 мс` — округление

**Симптом.** Round-trip занимает ~0.05 мс. Форматирование `{:.1}` даёт «0.0 мс» — вводит в заблуждение.

**Решение.** Хелпер `fmt_ms`:

```rust
fn fmt_ms(ms: f64) -> String {
    if ms < 0.05 { return String::from("<0.1"); }
    if ms < 1.0  { return format_str(format_args!("{ms:.2}")); }
    if ms < 10.0 { return format_str(format_args!("{ms:.1}")); }
    return format_str(format_args!("{ms:.0}"));
}
```

### Правила проекта, зафиксированные в этой подглаве

1. **Опциональные поля WebCodecs** требуют fallback. `description` — опционален.
2. **В WASM-контексте `Uint8Array → Vec<u8>`** используйте `to_vec()`, не `copy_to`.
3. **Явная типизация** в `match` с `Vec::<T>::new()` — избегает потери типа.
4. **Двухпроходная сборка** — общий паттерн для бинарных форматов с обратными ссылками.
5. **Round-trip через собственный demuxer** — сильнейший тест. Если mux + demux согласованы — формат валиден.
6. **Yield раз в кадр** через `TimeoutFuture::new(0).await` — UI обновляется, пользователь видит прогресс.

---

## 23.11.5. Исходные файлы

### `src/mp4_muxer.rs` — НОВЫЙ

```rust
//! 23.11. Muxing: EncodedVideoChunk → MP4.
//!
//! Минимальный MP4 muxer: `ftyp` + `moov` + `mdat`.
//!
//! ## Двухпроходная сборка moov
//!
//! `stco` содержит offset начала `mdat` — но offset зависит от размера
//! `moov`, который сам содержит `stco`. Решение: первый проход строит
//! `moov` с offset = 0, измеряет размер, второй — с реальным offset.
//! Размер не меняется: `stco` — фиксированной длины.
//!
//! ## Публичный API
//!
//! - `build_mp4(chunks, config) -> Result<Vec<u8>, String>`

use crate::video_frame::format_str;

// =============================================================================
// Константы
// =============================================================================

/// Track ID для единственного видео-трека.
pub const VIDEO_TRACK_ID: u32 = 1;
/// Timescale по умолчанию — микросекунды (1:1 с JS-timestamp).
pub const MOVIE_TIMESCALE: u32 = 1_000_000;
/// Размер `ftyp` box: 8 заголовок + 24 body.
pub const FTYP_SIZE: usize = 32;

// =============================================================================
// Входные структуры
// =============================================================================

/// Один encoded chunk на вход muxer-у.
#[derive(Debug, Clone)]
pub struct VideoChunk {
    pub is_key: bool,
    pub timestamp_us: u64,
    pub duration_us: u32,
    pub data: Vec<u8>,
}

/// Параметры сборки MP4.
#[derive(Debug, Clone)]
pub struct MuxConfig {
    pub width: u32,
    pub height: u32,
    /// avcC payload (SPS/PPS).
    pub avcc: Vec<u8>,
}

// =============================================================================
// Низкоуровневые хелперы
// =============================================================================

fn u16_be(v: u16) -> [u8; 2] {
    return v.to_be_bytes();
}

fn u32_be(v: u32) -> [u8; 4] {
    return v.to_be_bytes();
}

/// Собирает box: size (u32 BE) + type + body.
fn build_box(box_type: &[u8; 4], body: &[u8]) -> Vec<u8> {
    let size = (8 + body.len()) as u32;
    let mut out = Vec::with_capacity(size as usize);
    out.extend_from_slice(&u32_be(size));
    out.extend_from_slice(box_type);
    out.extend_from_slice(body);
    return out;
}

/// Full box: version+flags (u32) в начале body.
fn build_full_box(box_type: &[u8; 4], version: u8, flags: u32, body: &[u8]) -> Vec<u8> {
    let vf = ((version as u32) << 24) | (flags & 0xFFFFFF);
    let mut full_body = Vec::with_capacity(4 + body.len());
    full_body.extend_from_slice(&u32_be(vf));
    full_body.extend_from_slice(body);
    return build_box(box_type, &full_body);
}

/// Unity matrix (36 байт) — стандартная для видео.
fn unity_matrix() -> [u8; 36] {
    return [
        0x00, 0x01, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00,
        0x00, 0x01, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00,
        0x40, 0x00, 0x00, 0x00,
    ];
}

// =============================================================================
// ftyp
// =============================================================================

fn build_ftyp() -> Vec<u8> {
    let mut body = Vec::with_capacity(24);
    body.extend_from_slice(b"isom");
    body.extend_from_slice(&u32_be(0x200));
    body.extend_from_slice(b"isom");
    body.extend_from_slice(b"iso2");
    body.extend_from_slice(b"avc1");
    body.extend_from_slice(b"mp41");
    return build_box(b"ftyp", &body);
}

// =============================================================================
// mvhd
// =============================================================================

fn build_mvhd(duration: u32) -> Vec<u8> {
    let mut body = Vec::with_capacity(96);
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(MOVIE_TIMESCALE));
    body.extend_from_slice(&u32_be(duration));
    body.extend_from_slice(&u32_be(0x00010000));
    body.extend_from_slice(&u16_be(0x0100));
    body.extend_from_slice(&u16_be(0));
    body.extend_from_slice(&[0u8; 8]);
    body.extend_from_slice(&unity_matrix());
    body.extend_from_slice(&[0u8; 24]);
    body.extend_from_slice(&u32_be(VIDEO_TRACK_ID + 1));
    return build_full_box(b"mvhd", 0, 0, &body);
}

// =============================================================================
// tkhd
// =============================================================================

fn build_tkhd(duration: u32, width: u32, height: u32) -> Vec<u8> {
    let mut body = Vec::with_capacity(80);
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(VIDEO_TRACK_ID));
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(duration));
    body.extend_from_slice(&[0u8; 8]);
    body.extend_from_slice(&u16_be(0));
    body.extend_from_slice(&u16_be(0));
    body.extend_from_slice(&u16_be(0));
    body.extend_from_slice(&u16_be(0));
    body.extend_from_slice(&unity_matrix());
    body.extend_from_slice(&u32_be(width << 16));
    body.extend_from_slice(&u32_be(height << 16));
    return build_full_box(b"tkhd", 0, 0x07, &body);
}

// =============================================================================
// mdhd
// =============================================================================

fn build_mdhd(duration: u32) -> Vec<u8> {
    let mut body = Vec::with_capacity(20);
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(MOVIE_TIMESCALE));
    body.extend_from_slice(&u32_be(duration));
    body.extend_from_slice(&u16_be(0x55C4));
    body.extend_from_slice(&u16_be(0));
    return build_full_box(b"mdhd", 0, 0, &body);
}

// =============================================================================
// hdlr
// =============================================================================

fn build_hdlr() -> Vec<u8> {
    let mut body = Vec::with_capacity(24 + 13);
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(b"vide");
    body.extend_from_slice(&[0u8; 12]);
    body.extend_from_slice(b"VideoHandler");
    body.push(0);
    return build_full_box(b"hdlr", 0, 0, &body);
}

// =============================================================================
// vmhd / dinf / dref / url
// =============================================================================

fn build_vmhd() -> Vec<u8> {
    let mut body = Vec::with_capacity(8);
    body.extend_from_slice(&u16_be(0));
    body.extend_from_slice(&[0u8; 6]);
    return build_full_box(b"vmhd", 0, 0x01, &body);
}

fn build_url_() -> Vec<u8> {
    return build_full_box(b"url ", 0, 0x01, &[]);
}

fn build_dref() -> Vec<u8> {
    let url_box = build_url_();
    let mut body = Vec::with_capacity(8 + url_box.len());
    body.extend_from_slice(&u32_be(1));
    body.extend_from_slice(&url_box);
    return build_full_box(b"dref", 0, 0, &body);
}

fn build_dinf() -> Vec<u8> {
    let dref = build_dref();
    return build_box(b"dinf", &dref);
}

// =============================================================================
// stsd → avc1 + avcC
// =============================================================================

fn build_avc1(width: u32, height: u32, avcc: &[u8]) -> Vec<u8> {
    let mut body = Vec::with_capacity(78 + 8 + avcc.len());
    body.extend_from_slice(&[0u8; 6]);
    body.extend_from_slice(&u16_be(1));
    body.extend_from_slice(&[0u8; 16]);
    body.extend_from_slice(&u16_be(width as u16));
    body.extend_from_slice(&u16_be(height as u16));
    body.extend_from_slice(&u32_be(0x00480000));
    body.extend_from_slice(&u32_be(0x00480000));
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u16_be(1));
    body.extend_from_slice(&[0u8; 32]);
    body.extend_from_slice(&u16_be(0x0018));
    body.extend_from_slice(&u16_be(0xFFFF));
    body.extend_from_slice(&build_box(b"avcC", avcc));
    return build_box(b"avc1", &body);
}

fn build_stsd(width: u32, height: u32, avcc: &[u8]) -> Vec<u8> {
    let avc1 = build_avc1(width, height, avcc);
    let mut body = Vec::with_capacity(4 + avc1.len());
    body.extend_from_slice(&u32_be(1));
    body.extend_from_slice(&avc1);
    return build_full_box(b"stsd", 0, 0, &body);
}

// =============================================================================
// stts / stsc / stsz / stco / stss
// =============================================================================

fn build_stts(count: u32, delta: u32) -> Vec<u8> {
    let mut body = Vec::with_capacity(8);
    body.extend_from_slice(&u32_be(1));
    body.extend_from_slice(&u32_be(count));
    body.extend_from_slice(&u32_be(delta));
    return build_full_box(b"stts", 0, 0, &body);
}

fn build_stsc(count: u32) -> Vec<u8> {
    let mut body = Vec::with_capacity(12);
    body.extend_from_slice(&u32_be(1));
    body.extend_from_slice(&u32_be(1));
    body.extend_from_slice(&u32_be(count));
    body.extend_from_slice(&u32_be(1));
    return build_full_box(b"stsc", 0, 0, &body);
}

fn build_stsz(sizes: &[u32]) -> Vec<u8> {
    let mut body = Vec::with_capacity(8 + 4 * sizes.len());
    body.extend_from_slice(&u32_be(0));
    body.extend_from_slice(&u32_be(sizes.len() as u32));
    for &s in sizes {
        body.extend_from_slice(&u32_be(s));
    }
    return build_full_box(b"stsz", 0, 0, &body);
}

fn build_stco(mdat_body_start: u32) -> Vec<u8> {
    let mut body = Vec::with_capacity(8);
    body.extend_from_slice(&u32_be(1));
    body.extend_from_slice(&u32_be(mdat_body_start));
    return build_full_box(b"stco", 0, 0, &body);
}

fn build_stss(key_indices: &[u32]) -> Vec<u8> {
    let mut body = Vec::with_capacity(4 + 4 * key_indices.len());
    body.extend_from_slice(&u32_be(key_indices.len() as u32));
    for &idx in key_indices {
        body.extend_from_slice(&u32_be(idx));
    }
    return build_full_box(b"stss", 0, 0, &body);
}

// =============================================================================
// stbl / minf / mdia / trak
// =============================================================================

fn build_stbl(chunks: &[VideoChunk], config: &MuxConfig, mdat_body_start: u32) -> Vec<u8> {
    let total = chunks.len() as u32;
    let delta = chunks.first().map(|c| c.duration_us).unwrap_or(33_333);

    let sizes: Vec<u32> = chunks.iter().map(|c| c.data.len() as u32).collect();
    let key_indices: Vec<u32> = chunks
        .iter()
        .enumerate()
        .filter(|(_, c)| c.is_key)
        .map(|(i, _)| (i + 1) as u32)
        .collect();

    let mut body = Vec::new();
    body.extend_from_slice(&build_stsd(config.width, config.height, &config.avcc));
    body.extend_from_slice(&build_stts(total, delta));
    body.extend_from_slice(&build_stsc(total));
    body.extend_from_slice(&build_stsz(&sizes));
    body.extend_from_slice(&build_stco(mdat_body_start));
    body.extend_from_slice(&build_stss(&key_indices));
    return build_box(b"stbl", &body);
}

fn build_minf(chunks: &[VideoChunk], config: &MuxConfig, mdat_body_start: u32) -> Vec<u8> {
    let vmhd = build_vmhd();
    let dinf = build_dinf();
    let stbl = build_stbl(chunks, config, mdat_body_start);
    let mut body = Vec::with_capacity(vmhd.len() + dinf.len() + stbl.len());
    body.extend_from_slice(&vmhd);
    body.extend_from_slice(&dinf);
    body.extend_from_slice(&stbl);
    return build_box(b"minf", &body);
}

fn build_mdia(
    chunks: &[VideoChunk],
    config: &MuxConfig,
    mdat_body_start: u32,
    duration: u32,
) -> Vec<u8> {
    let mdhd = build_mdhd(duration);
    let hdlr = build_hdlr();
    let minf = build_minf(chunks, config, mdat_body_start);
    let mut body = Vec::with_capacity(mdhd.len() + hdlr.len() + minf.len());
    body.extend_from_slice(&mdhd);
    body.extend_from_slice(&hdlr);
    body.extend_from_slice(&minf);
    return build_box(b"mdia", &body);
}

fn build_trak(
    chunks: &[VideoChunk],
    config: &MuxConfig,
    mdat_body_start: u32,
    duration: u32,
) -> Vec<u8> {
    let tkhd = build_tkhd(duration, config.width, config.height);
    let mdia = build_mdia(chunks, config, mdat_body_start, duration);
    let mut body = Vec::with_capacity(tkhd.len() + mdia.len());
    body.extend_from_slice(&tkhd);
    body.extend_from_slice(&mdia);
    return build_box(b"trak", &body);
}

// =============================================================================
// moov / mdat
// =============================================================================

fn total_duration_us(chunks: &[VideoChunk]) -> u32 {
    let last_end = chunks
        .last()
        .map(|c| c.timestamp_us + c.duration_us as u64)
        .unwrap_or(0);
    return last_end.min(u32::MAX as u64) as u32;
}

fn build_moov(chunks: &[VideoChunk], config: &MuxConfig, mdat_body_start: u32) -> Vec<u8> {
    let duration = total_duration_us(chunks);
    let mvhd = build_mvhd(duration);
    let trak = build_trak(chunks, config, mdat_body_start, duration);
    let mut body = Vec::with_capacity(mvhd.len() + trak.len());
    body.extend_from_slice(&mvhd);
    body.extend_from_slice(&trak);
    return build_box(b"moov", &body);
}

fn build_mdat(chunks: &[VideoChunk]) -> Vec<u8> {
    let total: usize = chunks.iter().map(|c| c.data.len()).sum();
    let size = (8 + total) as u32;
    let mut out = Vec::with_capacity(size as usize);
    out.extend_from_slice(&u32_be(size));
    out.extend_from_slice(b"mdat");
    for c in chunks {
        out.extend_from_slice(&c.data);
    }
    return out;
}

// =============================================================================
// Публичный API
// =============================================================================

/// Собирает MP4 из массива encoded chunks.
pub fn build_mp4(chunks: &[VideoChunk], config: &MuxConfig) -> Result<Vec<u8>, String> {
    if chunks.is_empty() {
        return Err(String::from("build_mp4: пустой массив chunks"));
    }
    if config.width == 0 || config.height == 0 {
        return Err(String::from("build_mp4: width/height = 0"));
    }
    if config.avcc.len() < 4 {
        return Err(String::from("build_mp4: avcc слишком короткий"));
    }
    if config.avcc[0] != 1 {
        return Err(format_str(format_args!(
            "build_mp4: avcc.version = {}, ожидалось 1",
            config.avcc[0],
        )));
    }

    // Первый проход — измеряем moov.
    let moov_stub = build_moov(chunks, config, 0);
    let mdat_body_start = (FTYP_SIZE + moov_stub.len() + 8) as u32;

    // Второй проход — реальный offset. Размер moov не меняется.
    let ftyp = build_ftyp();
    let moov = build_moov(chunks, config, mdat_body_start);
    debug_assert_eq!(moov.len(), moov_stub.len());

    let mdat = build_mdat(chunks);

    let total = ftyp.len() + moov.len() + mdat.len();
    let check = ftyp.len() + moov_stub.len() + 8 + mdat.len() - 8;
    if total != check {
        return Err(String::from(
            "build_mp4: несоответствие размеров (moov изменился)",
        ));
    }

    let mut out = Vec::with_capacity(total);
    out.extend_from_slice(&ftyp);
    out.extend_from_slice(&moov);
    out.extend_from_slice(&mdat);
    return Ok(out);
}
```

### `src/lib.rs` — добавление

```rust
pub mod app;
pub mod container_parser;
pub mod mp4_demuxer;
pub mod mp4_muxer;          // ← 23.11
pub mod video_frame;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

### `tests/wasm_mp4_muxer.rs` — НОВЫЙ (11 тестов)

```rust
#![cfg(target_arch = "wasm32")]

//! Тесты 23.11 — MP4 muxer.

use wasm_bindgen_test::*;
use hello_wasm02::container_parser::{find_avcc_in_stsd, parse_mp4};
use hello_wasm02::mp4_demuxer::{build_samples, parse_mdhd_timescale, parse_sample_table};
use hello_wasm02::mp4_muxer::{build_mp4, MuxConfig, VideoChunk};

wasm_bindgen_test_configure!(run_in_browser);

fn test_avcc() -> Vec<u8> {
    return vec![
        0x01, 0x42, 0x00, 0x1E,
        0xFF, 0xE1,
        0x00, 0x04, 0x67, 0x42, 0x00, 0x1E,
        0x01,
        0x00, 0x02, 0x68, 0xCE,
    ];
}

fn test_config() -> MuxConfig {
    return MuxConfig {
        width: 640,
        height: 480,
        avcc: test_avcc(),
    };
}

fn make_chunks(n: usize) -> Vec<VideoChunk> {
    let mut chunks = Vec::with_capacity(n);
    for i in 0..n {
        let is_key = i == 0 || (n > 2 && i == n / 2);
        chunks.push(VideoChunk {
            is_key,
            timestamp_us: (i as u64) * 33_333,
            duration_us: 33_333,
            data: vec![(i & 0xFF) as u8; 100 + i * 10],
        });
    }
    return chunks;
}

#[wasm_bindgen_test]
fn mux_builds_valid_mp4_with_ftyp() {
    let chunks = make_chunks(5);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    assert!(mp4.len() >= 8);
    assert_eq!(&mp4[4..8], b"ftyp");
}

#[wasm_bindgen_test]
fn mux_output_parses_with_container_parser() {
    let chunks = make_chunks(5);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");
    let types: Vec<&str> = boxes.iter().map(|b| b.box_type.as_str()).collect();
    assert!(types.contains(&"ftyp"));
    assert!(types.contains(&"moov"));
    assert!(types.contains(&"mdat"));
}

#[wasm_bindgen_test]
fn mux_moov_contains_trak_and_stsd() {
    let chunks = make_chunks(5);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");

    let moov = boxes.iter().find(|b| b.box_type == "moov").expect("moov");
    assert!(moov.find("trak").is_some());

    let stsd = moov.find("stsd").expect("stsd");
    let avcc = find_avcc_in_stsd(stsd).expect("avcC в stsd");
    assert_eq!(avcc[0], 0x01);
    assert_eq!(avcc[1], 0x42);
}

#[wasm_bindgen_test]
fn mux_sample_table_matches_input() {
    let chunks = make_chunks(7);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");
    let moov = boxes.iter().find(|b| b.box_type == "moov").unwrap();
    let mdhd = moov.find("mdhd").unwrap();
    let timescale = parse_mdhd_timescale(mdhd).unwrap();
    let stbl = moov.find("stbl").unwrap();

    let table = parse_sample_table(stbl, timescale).unwrap();
    let samples = build_samples(&table).unwrap();

    assert_eq!(samples.len(), 7);
    for (i, s) in samples.iter().enumerate() {
        assert_eq!(s.size, chunks[i].data.len() as u32);
        assert_eq!(s.timestamp_us, chunks[i].timestamp_us);
        assert_eq!(s.is_key, chunks[i].is_key);
    }
}

#[wasm_bindgen_test]
fn mux_offsets_point_into_mdat() {
    let chunks = make_chunks(4);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");

    let mdat = boxes.iter().find(|b| b.box_type == "mdat").unwrap();
    let moov = boxes.iter().find(|b| b.box_type == "moov").unwrap();
    let stbl = moov.find("stbl").unwrap();
    let timescale = parse_mdhd_timescale(moov.find("mdhd").unwrap()).unwrap();

    let table = parse_sample_table(stbl, timescale).unwrap();
    let samples = build_samples(&table).unwrap();

    let mdat_start = mdat.offset as u64 + 8;
    let mdat_end = mdat.offset as u64 + mdat.size;
    for s in &samples {
        assert!(s.offset >= mdat_start);
        assert!(s.offset + s.size as u64 <= mdat_end);
    }
}

#[wasm_bindgen_test]
fn mux_key_frames_match_input() {
    let chunks = make_chunks(10);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");
    let moov = boxes.iter().find(|b| b.box_type == "moov").unwrap();
    let stbl = moov.find("stbl").unwrap();
    let timescale = parse_mdhd_timescale(moov.find("mdhd").unwrap()).unwrap();

    let table = parse_sample_table(stbl, timescale).unwrap();
    let samples = build_samples(&table).unwrap();

    let input_keys: Vec<bool> = chunks.iter().map(|c| c.is_key).collect();
    let output_keys: Vec<bool> = samples.iter().map(|s| s.is_key).collect();
    assert_eq!(input_keys, output_keys);
}

#[wasm_bindgen_test]
fn mux_total_size_matches_sum() {
    let chunks = make_chunks(5);
    let expected_mdat_body: usize = chunks.iter().map(|c| c.data.len()).sum();
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");
    let mdat = boxes.iter().find(|b| b.box_type == "mdat").unwrap();
    assert_eq!(mdat.size as usize, 8 + expected_mdat_body);
}

#[wasm_bindgen_test]
fn mux_rejects_empty_chunks() {
    let result = build_mp4(&[], &test_config());
    assert!(result.is_err());
}

#[wasm_bindgen_test]
fn mux_rejects_zero_width() {
    let chunks = make_chunks(2);
    let mut cfg = test_config();
    cfg.width = 0;
    assert!(build_mp4(&chunks, &cfg).is_err());
}

#[wasm_bindgen_test]
fn mux_rejects_short_avcc() {
    let chunks = make_chunks(2);
    let mut cfg = test_config();
    cfg.avcc = vec![0x01, 0x42];
    assert!(build_mp4(&chunks, &cfg).is_err());
}

#[wasm_bindgen_test]
fn mux_single_chunk_works() {
    let chunks = make_chunks(1);
    let mp4 = build_mp4(&chunks, &test_config()).expect("build_mp4");
    let boxes = parse_mp4(&mp4).expect("parse_mp4");
    assert!(boxes.iter().any(|b| b.box_type == "moov"));
    assert!(boxes.iter().any(|b| b.box_type == "mdat"));
}
```

### `src/app.rs` — полный текст

Полный текст `src/app.rs` прилагается к главе — около 1000 строк. Ключевые функции:

- **`parse_annexb_nals`, `annexb_to_avcc`, `build_avcc_from_annexb`** — fallback, если Chrome не отдал `description`.
- **`build_summary`** — параметры кодирования (Video/Duration/Bitrate target/Description) + факты о сборке (Chunks/Key frames/mdat/Total/Overhead).
- **`build_timing_html`** — Encode/Mux/Round-trip с хелпером `fmt_ms`.
- **`build_roundtrip`** — 4 проверки: sample count, sizes, timestamps, key flags.
- **`build_tree_html`** — дерево box-ов с аннотациями `[avcC = N B]` для `stsd`.
- **`build_layout_html`** — пропорциональный bar ftyp/moov/mdat.
- **`build_hex_preview`** — hex + ASCII первых 16 байт каждого top-level box-а.
- **`encode_and_mux`** — основной pipeline с AnnexB fallback.

---

## 23.11.6. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты muxer-а (11 тестов)
wasm-pack test --headless --chrome  --test wasm_mp4_muxer
wasm-pack test --headless --firefox --test wasm_mp4_muxer

# 3. Регрессия — тесты 23.9, 23.10
wasm-pack test --headless --chrome  --test wasm_container_parser
wasm-pack test --headless --chrome  --test wasm_mp4_demuxer

# 4. Демо
npm run wasm:dev && npm start
```

---

## 23.11.7.1. Итоговый вид демо

После клика «Сгенерировать 30 кадров → MP4»:

```
┌─ Summary ────────────────────────────────────────────────┐
│ Format:      AVCC (native)                               │
│ Video:       640×480 @ 30 fps                            │
│ Duration:    1.00 s                                      │
│ Bitrate target: 2.0 Mbps                                 │
│ Description: 20 B (avcC payload · box = 28 B)            │
│ ─────────────────────────────────────────────────────────│
│ Chunks:      30                                          │
│ Key frames:  1                                           │
│ mdat body:   86.7 KB                                     │
│ Avg chunk:   2.9 KB                                      │
│ Total file:  87.5 KB                                     │
│ Overhead:    795 B (0.9%)                                │
└──────────────────────────────────────────────────────────┘

┌─ Timing ─────────────────────────────────────────────────┐
│ Encode:       199 мс   (VideoEncoder)                    │
│ Mux:          1.0 мс   (ftyp + moov + mdat)              │
│ Round-trip:   <0.1 мс  (mux → demux)                     │
└──────────────────────────────────────────────────────────┘

┌─ Layout ─────────────────────────────────────────────────┐
│ [ftyp][moov]                      [mdat 99.2%]           │
└──────────────────────────────────────────────────────────┘

┌─ Hex preview ──────────────────────────────────────────────────────────────┐
│ — ftyp (offset=0) —                                                        │
│ 00000000: 00 00 00 20 66 74 79 70 69 73 6F 6D 00 00 02 00  ....ftypisom....│
│ — moov (offset=32) —                                                       │
│ 00000020: 00 00 02 F3 6D 6F 6F 76 00 00 00 6C 6D 76 68 64  ....moov...lmvhd│
│ — mdat (offset=787) —                                                      │
│ 00000313: 00 01 54 C4 6D 64 61 74 00 00 00 02 09 10 00 00  ...Tmdat........│
└────────────────────────────────────────────────────────────────────────────┘

┌─ Дерево ─────────────────────────────────────────────────┐
│ ftyp  offset=0    size=32 B                              │
│ moov  offset=32   size=755 B                             │
│   mvhd  offset=40   size=108 B                           │
│   trak  offset=148  size=639 B                           │
│     tkhd  offset=156  size=92 B                          │
│     mdia  offset=248  size=539 B                         │
│       mdhd  offset=256  size=32 B                        │
│       hdlr  offset=288  size=45 B                        │
│       minf  offset=333  size=454 B                       │
│         vmhd  offset=341  size=20 B                      │
│         dinf  offset=361  size=36 B                      │
│         stbl  offset=397  size=390 B                     │
│           stsd  offset=405  size=130 B  [avcC = 20 B]    │
│           stts  offset=535  size=24 B                    │
│           stsc  offset=559  size=28 B                    │
│           stsz  offset=587  size=140 B                   │
│           stco  offset=727  size=20 B                    │
│           stss  offset=747  size=16 B                    │
│ mdat  offset=787  size=87.5 KB                           │
└──────────────────────────────────────────────────────────┘
```

**Что видит студент:**

1. **Key sample** содержит **IDR slice** — то, с чего может начаться декодирование.
2. **`avcC = 20 B`** в дереве — совпадает с `Description: 20 B` в summary. Связь видна.
3. **`stco offset=727`** — это offset `mdat body` (787 + 8 = 795... нет, `stco` содержит value, а не offset). Значение `stco[0]` = начало `mdat body` = `787 + 8 = 795`. Проверяется: `ftyp (32) + moov (755) + mdat header (8) = 795` ✓.
4. **Timing**: encode 199 мс, mux 1.0 мс — muxer не bottleneck. Round-trip <0.1 мс — мгновенно.
5. **Layout-bar**: moov ~0.9%, mdat ~99.1%. Для видео это норма.
6. **Round-trip**: 4 зелёных галочки — sample count, sizes, timestamps, key flags совпадают.

23.11.7.b _src/app.rs` — полный текст

```
//! Глава 23.11 — Muxing: EncodedVideoChunk → MP4.
//!
//! Демо: 30 кадров на canvas → VideoEncoder → минимальный MP4 muxer
//! на Rust → валидный MP4-файл.
//!
//! ## Что демонстрируется
//!
//! - Генерация 30 кадров и кодирование через `VideoEncoder`.
//! - Сборка `ftyp` + `moov` + `mdat` вручную.
//! - Двухпроходная сборка `moov` (stco зависит от размера moov).
//! - Round-trip: разбор обратно через `mp4_demuxer` из 23.10.
//! - Дерево box-ов собранного MP4 — то же, что в 23.9.
//! - Layout-bar: доля ftyp / moov / mdat от размера файла.
//! - Hex-превью начала каждого top-level box-а.
//! - Timing: encode vs mux vs round-trip.
//!
//! ## AnnexB fallback
//!
//! Chrome может не отдавать `description` (SPS/PPS) в `metadata.decoderConfig`.
//! В этом случае chunks приходят в формате AnnexB (start codes `00 00 00 01`),
//! а для MP4 нужен AVCC (length-prefix). Демо:
//!   1. извлекает SPS/PPS из первого key chunk,
//!   2. строит avcC вручную,
//!   3. конвертирует все chunks AnnexB → AVCC.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect, Uint8Array};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Blob, BlobPropertyBag, Document, HtmlCanvasElement, HtmlElement, HtmlVideoElement};

use crate::container_parser::{find_avcc_in_stsd, format_bytes, parse_mp4, BoxInfo};
use crate::mp4_demuxer::{build_samples, parse_mdhd_timescale, parse_sample_table};
use crate::mp4_muxer::{build_mp4, MuxConfig, VideoChunk};
use crate::video_frame::{format_str, get_fn, set_prop};

// =============================================================================
// Константы
// =============================================================================

const FRAME_COUNT: u32 = 30;
const FPS: u32 = 30;
const FRAME_DURATION_US: u32 = 1_000_000 / FPS;
const CANVAS_W: u32 = 640;
const CANVAS_H: u32 = 480;
const BITRATE: u32 = 2_000_000;
const HEX_PREVIEW_BYTES: usize = 16;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.mx-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.mx-desc-line { margin:6px 0; }
.mx-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.mx-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.mx-btn:hover:not([disabled]) { background:#30363d; }
.mx-btn:disabled { opacity:0.5;cursor:not-allowed; }
.mx-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.mx-btn-primary:hover:not([disabled]) { background:#2ea043; }
.mx-btn-secondary { background:#8957e5;border-color:#a371f7;color:#fff;}
.mx-btn-secondary:hover:not([disabled]) { background:#a371f7; }
.mx-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}

.mx-canvas-wrap { max-width:900px;margin-bottom:16px;padding:10px;background:#0d1117;border:1px solid #21262d;border-radius:6px;}
.mx-canvas { display:block;background:#000;border-radius:4px;max-width:100%;width:100%;}

.mx-pair { max-width:900px;display:flex;gap:16px;flex-wrap:wrap;margin-bottom:16px;}
.mx-slot { flex:1 1 320px;min-width:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;padding:10px 12px;}
.mx-slot-title { color:#8b949e;font-family:ui-monospace,monospace;font-size:12px;font-weight:600;margin-bottom:8px;}
.mx-video { display:block;background:#000;border-radius:6px;max-width:100%;width:100%;aspect-ratio:4/3;object-fit:contain;}

.mx-summary { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;overflow-wrap:anywhere;}
.mx-summary .row { display:flex;gap:12px; }
.mx-summary .lbl { color:#8b949e;min-width:130px;flex-shrink:0; }
.mx-summary .val { color:#79c0ff; }
.mx-summary .ok { color:#3fb950; }
.mx-summary .warn { color:#f0883e; }
.mx-summary .err { color:#f85149; }
.mx-summary .muted { color:#6e7681; }
.mx-summary .sep { border-top:1px solid #21262d;margin:6px 0; }

.mx-progress { max-width:900px;margin-bottom:12px;}
.mx-progress-bar { height:8px;background:#21262d;border-radius:4px;overflow:hidden;}
.mx-progress-fill { height:100%;background:#3fb950;transition:width 0.1s linear;width:0%;}
.mx-progress-label { margin-top:4px;font-family:ui-monospace,monospace;font-size:11px;color:#8b949e;}

.mx-stats { max-width:900px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;overflow-wrap:anywhere;word-break:break-word;}
.mx-stats .lbl { color:#8b949e; }
.mx-stats .val { color:#79c0ff; }
.mx-stats .err { color:#f85149; }
.mx-stats .muted { color:#6e7681; }
.mx-stats .ok { color:#3fb950; }

.mx-tree { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:12px;line-height:1.55;white-space:pre;overflow-x:auto;color:#8b949e;}
.mx-tree .name { font-weight:600; }
.mx-tree .pos { color:#6e7681; }
.mx-tree .ann { color:#a371f7; }
.mx-box-ftyp { color:#58a6ff; }
.mx-box-moov, .mx-box-trak, .mx-box-mdia, .mx-box-minf, .mx-box-stbl, .mx-box-edts, .mx-box-dinf { color:#79c0ff; }
.mx-box-mdat { color:#f0883e; }
.mx-box-table { color:#a371f7; }
.mx-box-other { color:#8b949e; }

.mx-layout { max-width:900px;display:flex;height:32px;margin:0 0 16px 0;border-radius:6px;overflow:hidden;font-family:ui-monospace,monospace;font-size:11px;font-weight:600;}
.mx-layout-seg { display:flex;align-items:center;justify-content:center;min-width:3em;overflow:hidden;white-space:nowrap;padding:0 4px;}
.mx-seg-ftyp { background:#58a6ff; color:#0d1117; }
.mx-seg-moov { background:#a371f7; color:#0d1117; }
.mx-seg-mdat { background:#3fb950; color:#0d1117; }
.mx-seg-other { background:#30363d; color:#c9d1d9; }

.mx-hex { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;color:#8b949e;white-space:pre;overflow-x:auto;}
.mx-hex .row-label { color:#79c0ff; font-weight:600; }
.mx-hex .bytes { color:#c9d1d9; }
.mx-hex .ascii { color:#6e7681; }

.mx-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    phase: String,
    error: Option<String>,
    progress: f64,
    progress_label: String,
    format_note: String,
    summary_html: String,
    roundtrip_html: String,
    tree_html: String,
    layout_html: String,
    hex_html: String,
    timing: Option<(f64, f64, f64)>,
    mp4_bytes: Vec<u8>,
    blob_url: Option<String>,
    player_ready: bool,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            phase: String::from("простой"),
            error: None,
            progress: 0.0,
            progress_label: String::new(),
            format_note: String::new(),
            summary_html: String::new(),
            roundtrip_html: String::new(),
            tree_html: String::new(),
            layout_html: String::new(),
            hex_html: String::new(),
            timing: None,
            mp4_bytes: Vec::new(),
            blob_url: None,
            player_ready: false,
        };
    }

    fn reset(&mut self) {
        if let Some(url) = self.blob_url.take() {
            if let Some(window) = web_sys::window() {
                if let Ok(url_api) = Reflect::get(window.as_ref(), &JsValue::from_str("URL")) {
                    if let Ok(revoke) =
                        Reflect::get(&url_api, &JsValue::from_str("revokeObjectURL"))
                    {
                        if let Ok(revoke) = revoke.dyn_into::<js_sys::Function>() {
                            let _ = revoke.call1(&url_api, &JsValue::from_str(&url));
                        }
                    }
                }
            }
        }
        self.summary_html.clear();
        self.roundtrip_html.clear();
        self.tree_html.clear();
        self.layout_html.clear();
        self.hex_html.clear();
        self.format_note.clear();
        self.timing = None;
        self.mp4_bytes.clear();
        self.progress = 0.0;
        self.progress_label.clear();
        self.error = None;
        self.player_ready = false;
    }
}

// =============================================================================
// Хелперы строк
// =============================================================================

fn row(label: &str, value_html: &str) -> String {
    return format_str(format_args!(
        r#"<div class="row"><span class="lbl">{label}:</span><span class="val">{value}</span></div>"#,
        label = label,
        value = value_html,
    ));
}

/// Форматирует длительность в миллисекундах.
///
/// Малые значения (< 0.05) показываются как «<0.1 мс» — иначе
/// округление до одного знака даёт «0.0», что вводит в заблуждение.
fn fmt_ms(ms: f64) -> String {
    if ms < 0.05 {
        return String::from("<0.1");
    }
    if ms < 1.0 {
        return format_str(format_args!("{ms:.2}"));
    }
    if ms < 10.0 {
        return format_str(format_args!("{ms:.1}"));
    }
    return format_str(format_args!("{ms:.0}"));
}

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "mx-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(p) = old.parent_node() {
            p.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    return Ok(());
}

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>Muxing — обратная операция к demuxing.</strong> На входе — массив <code>EncodedVideoChunk</code> и <code>avcC</code> (SPS/PPS). На выходе — валидный MP4-файл.",
        "<strong>ftyp + moov + mdat.</strong> ftyp — сигнатура. moov — мета-информация (mvhd, trak, mdia, minf, stbl). mdat — конкатенация всех chunks.",
        "<strong>Циклическая зависимость.</strong> <code>stco[0]</code> зависит от размера <code>moov</code>, который содержит <code>stco</code>. Решение — двухпроходная сборка.",
        "<strong>AnnexB fallback.</strong> Chrome может отдать chunks без <code>description</code> — в формате AnnexB со start codes. Демо извлекает SPS/PPS из первого key chunk и конвертирует всё в AVCC.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("mx-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("mx-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    return Ok(block.unchecked_into());
}

// =============================================================================
// AnnexB ↔ AVCC
// =============================================================================

fn parse_annexb_nals(data: &[u8]) -> Vec<Vec<u8>> {
    let mut nals = Vec::new();
    let mut i = 0usize;
    let n = data.len();

    while i < n {
        let mut sc_len = 0usize;
        if i + 3 <= n && data[i] == 0 && data[i + 1] == 0 {
            if i + 4 <= n && data[i + 2] == 0 && data[i + 3] == 1 {
                sc_len = 4;
            } else if data[i + 2] == 1 {
                sc_len = 3;
            }
        }
        if sc_len == 0 {
            i += 1;
            continue;
        }

        let start = i + sc_len;
        let mut end = start;
        while end + 3 <= n {
            if data[end] == 0 && data[end + 1] == 0 {
                if end + 4 <= n && data[end + 2] == 0 && data[end + 3] == 1 {
                    break;
                }
                if data[end + 2] == 1 {
                    break;
                }
            }
            end += 1;
        }
        if end + 3 > n {
            end = n;
        }
        if end > start {
            nals.push(data[start..end].to_vec());
        }
        i = end;
    }
    return nals;
}

fn annexb_to_avcc(data: &[u8]) -> Vec<u8> {
    let nals = parse_annexb_nals(data);
    let mut out = Vec::with_capacity(data.len());
    for nal in &nals {
        let len = nal.len() as u32;
        out.extend_from_slice(&len.to_be_bytes());
        out.extend_from_slice(nal);
    }
    return out;
}

fn build_avcc_from_annexb(data: &[u8]) -> Option<Vec<u8>> {
    let nals = parse_annexb_nals(data);
    let mut sps: Option<&[u8]> = None;
    let mut pps: Option<&[u8]> = None;

    for nal in &nals {
        if nal.is_empty() {
            continue;
        }
        let nal_type = nal[0] & 0x1F;
        if nal_type == 7 && sps.is_none() {
            sps = Some(nal);
        }
        if nal_type == 8 && pps.is_none() {
            pps = Some(nal);
        }
    }

    let sps = sps?;
    let pps = pps?;
    if sps.len() < 4 {
        return None;
    }

    let mut out = Vec::with_capacity(11 + sps.len() + pps.len());
    out.push(1);
    out.push(sps[1]);
    out.push(sps[2]);
    out.push(sps[3]);
    out.push(0xFF);
    out.push(0xE1);
    out.extend_from_slice(&(sps.len() as u16).to_be_bytes());
    out.extend_from_slice(sps);
    out.push(1);
    out.extend_from_slice(&(pps.len() as u16).to_be_bytes());
    out.extend_from_slice(pps);
    return Some(out);
}

// =============================================================================
// Рисование кадра
// =============================================================================

fn draw_frame(canvas: &HtmlCanvasElement, i: u32) -> Result<(), JsValue> {
    let ctx_js = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?;

    let t = (i as f64) / FRAME_COUNT as f64;
    let hue = t * 360.0;

    let create_grad = Reflect::get(&ctx_js, &JsValue::from_str("createLinearGradient"))?
        .dyn_into::<js_sys::Function>()?;
    let g = create_grad.call4(
        &ctx_js,
        &JsValue::from_f64(0.0),
        &JsValue::from_f64(0.0),
        &JsValue::from_f64(CANVAS_W as f64),
        &JsValue::from_f64(CANVAS_H as f64),
    )?;

    let add_color =
        Reflect::get(&g, &JsValue::from_str("addColorStop"))?.dyn_into::<js_sys::Function>()?;
    let _ = add_color.call2(
        &g,
        &JsValue::from_f64(0.0),
        &JsValue::from_str(&format_str(format_args!("hsl({hue:.0}, 70%, 40%)"))),
    );
    let _ = add_color.call2(
        &g,
        &JsValue::from_f64(1.0),
        &JsValue::from_str(&format_str(format_args!(
            "hsl({:.0}, 70%, 20%)",
            (hue + 60.0) % 360.0,
        ))),
    );

    set_prop(&ctx_js, "fillStyle", &g)?;
    let fill_rect =
        Reflect::get(&ctx_js, &JsValue::from_str("fillRect"))?.dyn_into::<js_sys::Function>()?;
    let _ = fill_rect.call4(
        &ctx_js,
        &JsValue::from_f64(0.0),
        &JsValue::from_f64(0.0),
        &JsValue::from_f64(CANVAS_W as f64),
        &JsValue::from_f64(CANVAS_H as f64),
    );

    let cx = CANVAS_W as f64 / 2.0 + (t * std::f64::consts::TAU).sin() * 150.0;
    let cy = CANVAS_H as f64 / 2.0 + (t * std::f64::consts::TAU).cos() * 100.0;

    let begin_path =
        Reflect::get(&ctx_js, &JsValue::from_str("beginPath"))?.dyn_into::<js_sys::Function>()?;
    let _ = begin_path.call0(&ctx_js);

    let arc = Reflect::get(&ctx_js, &JsValue::from_str("arc"))?.dyn_into::<js_sys::Function>()?;
    let _ = arc.call5(
        &ctx_js,
        &JsValue::from_f64(cx),
        &JsValue::from_f64(cy),
        &JsValue::from_f64(40.0),
        &JsValue::from_f64(0.0),
        &JsValue::from_f64(std::f64::consts::TAU),
    );

    set_prop(&ctx_js, "fillStyle", &JsValue::from_str("#f0f6fc"))?;
    let fill = Reflect::get(&ctx_js, &JsValue::from_str("fill"))?.dyn_into::<js_sys::Function>()?;
    let _ = fill.call0(&ctx_js);

    return Ok(());
}

// =============================================================================
// Summary / timing / round-trip
// =============================================================================

fn build_summary(
    chunks: &[VideoChunk],
    mp4: &[u8],
    avcc_len: usize,
) -> String {
    let total_size = mp4.len();
    let mdat_body: usize = chunks.iter().map(|c| c.data.len()).sum();
    let key_count = chunks.iter().filter(|c| c.is_key).count();
    let avg_size = if chunks.is_empty() {
        0
    } else {
        mdat_body / chunks.len()
    };

    let total_us: u64 = chunks
        .last()
        .map(|c| c.timestamp_us + c.duration_us as u64)
        .unwrap_or(0);
    let duration_s = total_us as f64 / 1_000_000.0;

    let mut out = String::new();

    // --- Параметры кодирования ---
    out.push_str(&row(
        "Video",
        &format_str(format_args!(
            r#"<span class="val">{w}×{h}</span> <span class="muted">@ {fps} fps</span>"#,
            w = CANVAS_W,
            h = CANVAS_H,
            fps = FPS,
        )),
    ));
    out.push_str(&row(
        "Duration",
        &format_str(format_args!(
            r#"<span class="val">{dur:.2} s</span>"#,
            dur = duration_s,
        )),
    ));
    out.push_str(&row(
        "Bitrate target",
        &format_str(format_args!(
            r#"<span class="val">{:.1} Mbps</span>"#,
            BITRATE as f64 / 1_000_000.0,
        )),
    ));
    out.push_str(&row(
        "Description",
        &format_str(format_args!(
            r#"<span class="val">{len} B</span> <span class="muted">(avcC payload · box = {box_len} B)</span>"#,
            len = avcc_len,
            box_len = avcc_len + 8,
        )),
    ));

    // Разделитель.
    out.push_str(r#"<div class="sep"></div>"#);

    // --- Факты о сборке ---
    out.push_str(&row(
        "Chunks",
        &format_str(format_args!(r#"<span class="val">{}</span>"#, chunks.len(),)),
    ));
    out.push_str(&row(
        "Key frames",
        &format_str(format_args!(r#"<span class="ok">{key_count}</span>"#,)),
    ));
    out.push_str(&row(
        "mdat body",
        &format_str(format_args!(
            r#"<span class="val">{}</span>"#,
            format_bytes(mdat_body),
        )),
    ));
    out.push_str(&row(
        "Avg chunk",
        &format_str(format_args!(
            r#"<span class="val">{}</span>"#,
            format_bytes(avg_size),
        )),
    ));
    out.push_str(&row(
        "Total file",
        &format_str(format_args!(
            r#"<span class="ok">{}</span>"#,
            format_bytes(total_size),
        )),
    ));
    out.push_str(&row(
        "Overhead",
        &format_str(format_args!(
            r#"<span class="muted">{}</span> <span class="val">({:.1}%)</span>"#,
            format_bytes(total_size - mdat_body),
            (total_size - mdat_body) as f64 / total_size as f64 * 100.0,
        )),
    ));
    return out;
}

fn build_timing_html(timing: (f64, f64, f64)) -> String {
    let (enc, mux, rt) = timing;
    let mut out = String::new();
    out.push_str(&row(
        "Encode",
        &format_str(format_args!(
            r#"<span class="val">{v} мс</span> <span class="muted">(VideoEncoder)</span>"#,
            v = fmt_ms(enc),
        )),
    ));
    out.push_str(&row(
        "Mux",
        &format_str(format_args!(
            r#"<span class="ok">{v} мс</span> <span class="muted">(ftyp + moov + mdat)</span>"#,
            v = fmt_ms(mux),
        )),
    ));
    out.push_str(&row(
        "Round-trip",
        &format_str(format_args!(
            r#"<span class="val">{v} мс</span> <span class="muted">(mux → demux)</span>"#,
            v = fmt_ms(rt),
        )),
    ));
    return out;
}

fn build_roundtrip(mp4: &[u8], input_chunks: &[VideoChunk]) -> String {
    let boxes = match parse_mp4(mp4) {
        Ok(b) => b,
        Err(e) => {
            return format_str(format_args!(r#"<div class="err">parse_mp4: {e}</div>"#));
        }
    };
    let Some(moov) = boxes.iter().find(|b| b.box_type == "moov") else {
        return format_str(format_args!(r#"<div class="err">moov не найден</div>"#));
    };
    let Some(mdhd) = moov.find("mdhd") else {
        return format_str(format_args!(r#"<div class="err">mdhd не найден</div>"#));
    };
    let Some(timescale) = parse_mdhd_timescale(mdhd) else {
        return format_str(format_args!(r#"<div class="err">timescale = 0</div>"#));
    };
    let Some(stbl) = moov.find("stbl") else {
        return format_str(format_args!(r#"<div class="err">stbl не найден</div>"#));
    };

    let table = match parse_sample_table(stbl, timescale) {
        Ok(t) => t,
        Err(e) => {
            return format_str(format_args!(r#"<div class="err">sample table: {e}</div>"#));
        }
    };
    let samples = match build_samples(&table) {
        Ok(s) => s,
        Err(e) => {
            return format_str(format_args!(r#"<div class="err">build_samples: {e}</div>"#));
        }
    };

    let count_ok = samples.len() == input_chunks.len();
    let size_ok = samples
        .iter()
        .zip(input_chunks.iter())
        .all(|(s, c)| s.size == c.data.len() as u32);
    let ts_ok = samples
        .iter()
        .zip(input_chunks.iter())
        .all(|(s, c)| s.timestamp_us == c.timestamp_us);
    let key_ok = samples
        .iter()
        .zip(input_chunks.iter())
        .all(|(s, c)| s.is_key == c.is_key);

    let line = |label: &str, ok: bool, val: &str| -> String {
        let cls = if ok { "ok" } else { "err" };
        let mark = if ok { "✓" } else { "✗" };
        return format_str(format_args!(
            r#"<div class="row"><span class="lbl">{label}:</span><span class="{cls}">{mark} {val}</span></div>"#,
            label = label,
            cls = cls,
            mark = mark,
            val = val,
        ));
    };

    let mut out = String::new();
    out.push_str(&format_str(format_args!(
        r#"<div class="row"><span class="lbl">Round-trip:</span><span class="val">mux → demux</span></div>"#
    )));
    out.push_str(&line(
        "Samples",
        count_ok,
        &format_str(format_args!("{}/{}", samples.len(), input_chunks.len())),
    ));
    out.push_str(&line("Sizes", size_ok, "совпадают"));
    out.push_str(&line("Timestamps", ts_ok, "совпадают"));
    out.push_str(&line("Key flags", key_ok, "совпадают"));
    return out;
}

// =============================================================================
// Tree / layout / hex-preview
// =============================================================================

fn box_class(box_type: &str) -> &'static str {
    match box_type {
        "ftyp" => "mx-box-ftyp",
        "moov" | "trak" | "mdia" | "minf" | "stbl" | "edts" | "dinf" => "mx-box-moov",
        "mdat" => "mx-box-mdat",
        "stsd" | "stco" | "stsz" | "stss" | "stsc" | "avcC" | "stts" => "mx-box-table",
        _ => "mx-box-other",
    }
}

/// Возвращает HTML-аннотацию для box-а, если она есть.
///
/// Сейчас: `stsd` — показывает размер извлечённого `avcC` из payload.
/// Это связывает summary с деревом: видно, что description
/// действительно лежит внутри stsd.
fn box_annotation(b: &BoxInfo) -> String {
    if b.box_type == "stsd" {
        if let Some(avcc) = find_avcc_in_stsd(b) {
            return format_str(format_args!(
                r#"  <span class="ann">[avcC = {len} B]</span>"#,
                len = avcc.len(),
            ));
        }
    }
    return String::new();
}

fn render_tree_recursive(boxes: &[BoxInfo], depth: usize, out: &mut String) {
    for b in boxes {
        let indent = "  ".repeat(depth);
        let cls = box_class(&b.box_type);
        let ann = box_annotation(b);
        out.push_str(&indent);
        out.push_str(&format_str(format_args!(
            r#"<span class="name {cls}">{name}</span> <span class="pos">offset={off} size={sz}</span>{ann}"#,
            cls = cls,
            name = b.box_type,
            off = b.offset,
            sz = format_bytes(b.size as usize),
            ann = ann,
        )));
        out.push('\n');
        if !b.children.is_empty() {
            render_tree_recursive(&b.children, depth + 1, out);
        }
    }
}

fn build_tree_html(boxes: &[BoxInfo]) -> String {
    let mut inner = String::new();
    render_tree_recursive(boxes, 0, &mut inner);
    return format_str(format_args!(r#"<div class="mx-tree">{inner}</div>"#));
}

fn build_layout_html(boxes: &[BoxInfo]) -> String {
    if boxes.is_empty() {
        return String::new();
    }
    let mut html = String::from(r#"<div class="mx-layout">"#);
    for b in boxes {
        let cls = match b.box_type.as_str() {
            "ftyp" => "mx-seg-ftyp",
            "moov" => "mx-seg-moov",
            "mdat" => "mx-seg-mdat",
            _ => "mx-seg-other",
        };
        html.push_str(&format_str(format_args!(
            r#"<div class="mx-layout-seg {cls}" style="flex:{flex}" title="{name} ({hr})">{name}</div>"#,
            cls = cls,
            flex = b.size.max(1),
            name = b.box_type,
            hr = format_bytes(b.size as usize),
        )));
    }
    html.push_str("</div>");
    return html;
}

fn hex_row(bytes: &[u8], offset: usize) -> String {
    let n = bytes.len().min(HEX_PREVIEW_BYTES);
    let mut hex_part = String::with_capacity(50);
    let mut ascii_part = String::with_capacity(16);
    for i in 0..HEX_PREVIEW_BYTES {
        if i < n {
            hex_part.push_str(&format_str(format_args!("{:02X} ", bytes[i])));
            let ch = bytes[i];
            let c = if ch.is_ascii_graphic() || ch == b' ' {
                ch as char
            } else {
                '.'
            };
            ascii_part.push(c);
        } else {
            hex_part.push_str("   ");
            ascii_part.push(' ');
        }
        if i == 7 {
            hex_part.push(' ');
        }
    }
    return format_str(format_args!(
        r#"<span class="row-label">{off:08X}:</span> <span class="bytes">{hex}</span> <span class="ascii">{ascii}</span>"#,
        off = offset,
        hex = hex_part,
        ascii = ascii_part,
    ));
}

fn hex_block(label: &str, offset: usize, mp4: &[u8]) -> String {
    if offset + HEX_PREVIEW_BYTES > mp4.len() {
        return String::new();
    }
    let mut out = String::new();
    out.push_str(&format_str(format_args!(
        r#"<span class="row-label">— {label} (offset={off}) —</span>"#,
        label = label,
        off = offset,
    )));
    out.push('\n');
    out.push_str(&hex_row(&mp4[offset..offset + HEX_PREVIEW_BYTES], offset));
    out.push('\n');
    return out;
}

fn build_hex_preview(mp4: &[u8], boxes: &[BoxInfo]) -> String {
    if mp4.len() < HEX_PREVIEW_BYTES {
        return String::new();
    }
    let mut out = String::from(r#"<div class="mx-hex">"#);
    for b in boxes {
        if b.box_type == "ftyp" || b.box_type == "moov" || b.box_type == "mdat" {
            out.push_str(&hex_block(&b.box_type, b.offset, mp4));
        }
    }
    out.push_str("</div>");
    return out;
}

// =============================================================================
// Кодирование + muxing
// =============================================================================

async fn encode_and_mux(doc: Document, st: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let global = js_sys::global();
    let ctor = match Reflect::get(&global, &JsValue::from_str("VideoEncoder")) {
        Ok(v) if v.is_function() => v,
        _ => {
            return Err(JsValue::from_str("VideoEncoder недоступен"));
        }
    };

    let encode_t0 = js_sys::Date::now();

    let chunks_cell: Rc<RefCell<Vec<VideoChunk>>> = Rc::new(RefCell::new(Vec::new()));
    let desc_cell: Rc<RefCell<Option<Vec<u8>>>> = Rc::new(RefCell::new(None));
    let err_cell: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));

    let chunks_out = chunks_cell.clone();
    let desc_out = desc_cell.clone();
    let err_out = err_cell.clone();

    let output_cb: Closure<dyn FnMut(JsValue, JsValue)> =
        Closure::new(move |chunk: JsValue, metadata: JsValue| {
            if desc_out.borrow().is_none() {
                if let Ok(cfg) = Reflect::get(&metadata, &JsValue::from_str("decoderConfig")) {
                    if !cfg.is_undefined() && !cfg.is_null() {
                        if let Ok(desc_js) = Reflect::get(&cfg, &JsValue::from_str("description")) {
                            if !desc_js.is_undefined() && !desc_js.is_null() {
                                let buf: Vec<u8> =
                                    if let Ok(u8v) = desc_js.clone().dyn_into::<Uint8Array>() {
                                        u8v.to_vec()
                                    } else if let Ok(ab) =
                                        desc_js.clone().dyn_into::<js_sys::ArrayBuffer>()
                                    {
                                        Uint8Array::new(&ab).to_vec()
                                    } else {
                                        Vec::new()
                                    };
                                if !buf.is_empty() {
                                    *desc_out.borrow_mut() = Some(buf);
                                }
                            }
                        }
                    }
                }
            }

            let is_key = Reflect::get(&chunk, &JsValue::from_str("type"))
                .ok()
                .and_then(|v| v.as_string())
                .map(|s| s == "key")
                .unwrap_or(false);

            let timestamp_us = Reflect::get(&chunk, &JsValue::from_str("timestamp"))
                .ok()
                .and_then(|v| v.as_f64())
                .unwrap_or(0.0) as u64;

            let duration_us = Reflect::get(&chunk, &JsValue::from_str("duration"))
                .ok()
                .and_then(|v| v.as_f64())
                .unwrap_or(FRAME_DURATION_US as f64) as u32;

            let byte_length = Reflect::get(&chunk, &JsValue::from_str("byteLength"))
                .ok()
                .and_then(|v| v.as_f64())
                .unwrap_or(0.0) as usize;

            let mut data: Vec<u8> = Vec::new();
            let copy_to = match Reflect::get(&chunk, &JsValue::from_str("copyTo")) {
                Ok(f) => f,
                Err(_) => return,
            };
            if let Ok(copy_fn) = copy_to.dyn_into::<js_sys::Function>() {
                let u8_buf = Uint8Array::new_with_length(byte_length as u32);
                if copy_fn.call1(&chunk, u8_buf.as_ref()).is_ok() {
                    data = u8_buf.to_vec();
                }
            }

            chunks_out.borrow_mut().push(VideoChunk {
                is_key,
                timestamp_us,
                duration_us,
                data,
            });
        });

    let err_cb: Closure<dyn FnMut(JsValue)> = Closure::new(move |e: JsValue| {
        let msg = Reflect::get(&e, &JsValue::from_str("message"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| String::from("unknown encoder error"));
        *err_out.borrow_mut() = Some(msg);
    });

    let init = Object::new();
    set_prop(&init, "output", output_cb.as_ref())?;
    set_prop(&init, "error", err_cb.as_ref())?;

    let encoder = {
        let args = Array::new();
        args.push(&init);
        Reflect::construct(&ctor.dyn_into::<js_sys::Function>()?, &args)?
    };

    let config = Object::new();
    set_prop(&config, "codec", &JsValue::from_str("avc1.42001E"))?;
    set_prop(&config, "width", &JsValue::from_f64(CANVAS_W as f64))?;
    set_prop(&config, "height", &JsValue::from_f64(CANVAS_H as f64))?;
    set_prop(&config, "bitrate", &JsValue::from_f64(BITRATE as f64))?;
    set_prop(&config, "framerate", &JsValue::from_f64(FPS as f64))?;

    let avc = Object::new();
    set_prop(&avc, "format", &JsValue::from_str("avc"))?;
    set_prop(&config, "avc", &avc)?;

    let configure = get_fn(&encoder, "configure")?;
    let _ = configure.call1(&encoder, &config)?;

    let canvas: HtmlCanvasElement = doc
        .get_element_by_id("mx-canvas")
        .and_then(|e| e.dyn_into::<HtmlCanvasElement>().ok())
        .ok_or_else(|| JsValue::from_str("no #mx-canvas"))?;

    let vf_ctor =
        Reflect::get(&global, &JsValue::from_str("VideoFrame"))?.dyn_into::<js_sys::Function>()?;
    let encode_fn = get_fn(&encoder, "encode")?;

    for i in 0..FRAME_COUNT {
        if let Some(e) = err_cell.borrow().as_ref() {
            return Err(JsValue::from_str(e));
        }

        draw_frame(&canvas, i)?;

        let ts_us = (i as f64) * FRAME_DURATION_US as f64;

        let init = Object::new();
        set_prop(&init, "timestamp", &JsValue::from_f64(ts_us))?;
        set_prop(
            &init,
            "duration",
            &JsValue::from_f64(FRAME_DURATION_US as f64),
        )?;

        let args = Array::new();
        args.push(canvas.as_ref());
        args.push(&init);
        let frame = Reflect::construct(&vf_ctor, &args)?;

        let opts = Object::new();
        let is_key = i == 0;
        set_prop(&opts, "keyFrame", &JsValue::from_bool(is_key))?;
        let _ = encode_fn.call2(&encoder, &frame, &opts);

        if let Ok(close) = Reflect::get(&frame, &JsValue::from_str("close")) {
            if let Ok(close) = close.dyn_into::<js_sys::Function>() {
                let _ = close.call0(&frame);
            }
        }

        {
            let mut s = st.borrow_mut();
            s.progress = (i + 1) as f64 / FRAME_COUNT as f64;
            s.progress_label = format_str(format_args!(
                "Кодирование: {}/{} кадров",
                i + 1,
                FRAME_COUNT,
            ));
        }
        update_progress(&doc, &st.borrow());

        TimeoutFuture::new(0).await;
    }

    let flush_fn = get_fn(&encoder, "flush")?;
    let flush_promise = flush_fn.call0(&encoder)?.dyn_into::<js_sys::Promise>()?;
    wasm_bindgen_futures::JsFuture::from(flush_promise).await?;

    if let Ok(close) = get_fn(&encoder, "close") {
        let _ = close.call0(&encoder);
    }
    output_cb.forget();
    err_cb.forget();

    let encode_ms = js_sys::Date::now() - encode_t0;

    let chunks_raw = chunks_cell.borrow().clone();
    let desc_raw = desc_cell.borrow().clone();

    if chunks_raw.is_empty() {
        return Err(JsValue::from_str("не закодировано ни одного chunk-а"));
    }

    let (avcc, chunks, format_note) = match desc_raw {
        Some(d) if !d.is_empty() => (d, chunks_raw, String::from("AVCC (native)")),
        _ => {
            let first_key = chunks_raw.iter().find(|c| c.is_key);
            let Some(first_key) = first_key else {
                return Err(JsValue::from_str(
                    "нет key chunk — не из чего извлечь SPS/PPS",
                ));
            };
            let Some(avcc) = build_avcc_from_annexb(&first_key.data) else {
                return Err(JsValue::from_str(
                    "не удалось извлечь SPS/PPS из key chunk (AnnexB fallback)",
                ));
            };
            let converted: Vec<VideoChunk> = chunks_raw
                .into_iter()
                .map(|c| VideoChunk {
                    is_key: c.is_key,
                    timestamp_us: c.timestamp_us,
                    duration_us: c.duration_us,
                    data: annexb_to_avcc(&c.data),
                })
                .collect();
            (avcc, converted, String::from("AnnexB → AVCC (fallback)"))
        }
    };

    let avcc_len = avcc.len();

    let config = MuxConfig {
        width: CANVAS_W,
        height: CANVAS_H,
        avcc,
    };

    let mux_t0 = js_sys::Date::now();
    let mp4 = match build_mp4(&chunks, &config) {
        Ok(b) => b,
        Err(e) => {
            return Err(JsValue::from_str(&e));
        }
    };
    let mux_ms = js_sys::Date::now() - mux_t0;

    let uint8 = Uint8Array::from(mp4.as_slice());
    let arr = Array::new();
    let uint8_js: JsValue = uint8.into();
    arr.push(&uint8_js);
    let opts = BlobPropertyBag::new();
    opts.set_type("video/mp4");
    let blob = Blob::new_with_u8_array_sequence_and_options(&arr, &opts)?;

    let url = {
        let url_api = Reflect::get(&global, &JsValue::from_str("URL"))?;
        let create = Reflect::get(&url_api, &JsValue::from_str("createObjectURL"))?
            .dyn_into::<js_sys::Function>()?;
        let blob_js: JsValue = blob.into();
        create
            .call1(&url_api, &blob_js)?
            .as_string()
            .ok_or_else(|| JsValue::from_str("createObjectURL returned non-string"))?
    };

    let summary = build_summary(&chunks, &mp4, avcc_len);

    let rt_t0 = js_sys::Date::now();
    let roundtrip = build_roundtrip(&mp4, &chunks);
    let rt_ms = js_sys::Date::now() - rt_t0;

    let boxes: Vec<BoxInfo> = match parse_mp4(&mp4) {
        Ok(b) => b,
        Err(_) => Vec::<BoxInfo>::new(),
    };
    let tree_html = build_tree_html(&boxes);
    let layout_html = build_layout_html(&boxes);
    let hex_html = build_hex_preview(&mp4, &boxes);

    {
        let mut s = st.borrow_mut();
        s.phase = String::from("готово");
        s.progress = 1.0;
        s.progress_label = String::from("MP4 собран");
        s.format_note = format_note;
        s.summary_html = summary;
        s.roundtrip_html = roundtrip;
        s.tree_html = tree_html;
        s.layout_html = layout_html;
        s.hex_html = hex_html;
        s.timing = Some((encode_ms, mux_ms, rt_ms));
        s.mp4_bytes = mp4;
        s.blob_url = Some(url);
        s.player_ready = true;
    }

    if let Some(video) = doc
        .get_element_by_id("mx-player")
        .and_then(|e| e.dyn_into::<HtmlVideoElement>().ok())
    {
        if let Some(url) = st.borrow().blob_url.as_ref() {
            set_prop(video.as_ref(), "src", &JsValue::from_str(url))?;
            let _ = video.load();
        }
    }

    update_ui(&doc, &st.borrow());
    return Ok(());
}

// =============================================================================
// UI update
// =============================================================================

fn update_progress(doc: &Document, st: &DemoState) {
    if let Some(el) = doc.get_element_by_id("mx-progress-fill") {
        let pct = format_str(format_args!("{:.0}%", st.progress * 100.0));
        if let Ok(html_el) = el.dyn_into::<HtmlElement>() {
            let _ = html_el.style().set_property("width", &pct);
        }
    }
    if let Some(el) = doc.get_element_by_id("mx-progress-label") {
        el.set_text_content(Some(&st.progress_label));
    }
}

fn update_ui(doc: &Document, st: &DemoState) {
    if let Some(el) = doc.get_element_by_id("mx-summary") {
        let format_line = if st.format_note.is_empty() {
            String::new()
        } else {
            row("Format", &format_str(format_args!(
                r#"<span class="val">{fmt}</span>"#,
                fmt = st.format_note,
            )))
        };
        el.set_inner_html(&if st.summary_html.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#"<div class="mx-summary">{fmt}{body}</div>"#,
                fmt = format_line,
                body = st.summary_html,
            ))
        });
    }

    if let Some(el) = doc.get_element_by_id("mx-timing") {
        el.set_inner_html(&match st.timing {
            Some(t) => format_str(format_args!(
                r#"<div class="mx-summary">{}</div>"#,
                build_timing_html(t),
            )),
            None => String::new(),
        });
    }

    if let Some(el) = doc.get_element_by_id("mx-roundtrip") {
        el.set_inner_html(&if st.roundtrip_html.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#"<div class="mx-summary">{}</div>"#,
                st.roundtrip_html,
            ))
        });
    }

    if let Some(el) = doc.get_element_by_id("mx-layout") {
        el.set_inner_html(&st.layout_html);
    }
    if let Some(el) = doc.get_element_by_id("mx-tree") {
        el.set_inner_html(&st.tree_html);
    }
    if let Some(el) = doc.get_element_by_id("mx-hex") {
        el.set_inner_html(&st.hex_html);
    }

    if let Some(el) = doc.get_element_by_id("mx-stats") {
        let err_line = st
            .error
            .as_ref()
            .map(|m| format_str(format_args!(r#"<div class="err">{m}</div>"#)))
            .unwrap_or_default();
        let html = format_str(format_args!(
            r#"{err}<div><span class="lbl">Статус:</span> <span class="val">{phase}</span></div>"#,
            err = err_line,
            phase = st.phase,
        ));
        el.set_inner_html(&html);
    }

    let enable_download = st.player_ready && !st.mp4_bytes.is_empty();
    if let Some(el) = doc.get_element_by_id("mx-download") {
        if enable_download {
            let _ = el.remove_attribute("disabled");
        } else {
            let _ = el.set_attribute("disabled", "");
        }
    }
    if let Some(el) = doc.get_element_by_id("mx-generate") {
        if st.phase == "кодирование" {
            let _ = el.set_attribute("disabled", "");
        } else {
            let _ = el.remove_attribute("disabled");
        }
    }
}

// =============================================================================
// Скачивание
// =============================================================================

fn trigger_download(doc: &Document, st: &Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let url = match st.borrow().blob_url.as_ref() {
        Some(u) => u.clone(),
        None => return Ok(()),
    };

    let a = doc.create_element("a")?;
    a.set_attribute("href", &url)?;
    a.set_attribute("download", "hello-wasm02-23.11.mp4")?;
    let head = doc.head().ok_or_else(|| JsValue::from_str("no head"))?;
    head.append_child(&a)?;
    let a_el: HtmlElement = a.dyn_into()?;
    a_el.click();
    if let Some(parent) = a_el.parent_node() {
        let _ = parent.remove_child(&a_el);
    }
    return Ok(());
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("23.11 — Muxing: EncodedVideoChunk → MP4"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("mx-block");
    controls.set_inner_html(
        r#"<button class="mx-btn mx-btn-primary" id="mx-generate">Сгенерировать 30 кадров → MP4</button>
           <button class="mx-btn mx-btn-secondary" id="mx-download" disabled>Скачать MP4</button>
           <span class="mx-label">ftyp + moov + mdat</span>"#,
    );
    root.append_child(&controls)?;

    let stats = document.create_element("div")?;
    stats.set_class_name("mx-stats");
    stats.set_id("mx-stats");
    stats.set_inner_html(
        r#"<div><span class="lbl">Статус:</span> <span class="val">простой</span></div>"#,
    );
    root.append_child(&stats)?;

    let progress = document.create_element("div")?;
    progress.set_class_name("mx-progress");
    progress.set_inner_html(
        r#"<div class="mx-progress-bar"><div class="mx-progress-fill" id="mx-progress-fill"></div></div>
           <div class="mx-progress-label" id="mx-progress-label"></div>"#,
    );
    root.append_child(&progress)?;

    let canvas_wrap = document.create_element("div")?;
    canvas_wrap.set_class_name("mx-canvas-wrap");
    canvas_wrap.set_inner_html(
        r#"<canvas id="mx-canvas" class="mx-canvas" width="640" height="480"></canvas>"#,
    );
    root.append_child(&canvas_wrap)?;

    let pair = document.create_element("div")?;
    pair.set_class_name("mx-pair");
    pair.set_inner_html(
        r#"<div class="mx-slot">
             <div class="mx-slot-title">Результат (MP4 через Blob URL)</div>
             <video id="mx-player" class="mx-video" controls playsinline></video>
           </div>
           <div class="mx-slot">
             <div class="mx-slot-title">Round-trip проверка</div>
             <div id="mx-roundtrip"></div>
           </div>"#,
    );
    root.append_child(&pair)?;

    let summary = document.create_element("div")?;
    summary.set_id("mx-summary");
    root.append_child(&summary)?;

    let timing = document.create_element("div")?;
    timing.set_id("mx-timing");
    root.append_child(&timing)?;

    let layout = document.create_element("div")?;
    layout.set_id("mx-layout");
    root.append_child(&layout)?;

    let hex_el = document.create_element("div")?;
    hex_el.set_id("mx-hex");
    root.append_child(&hex_el)?;

    let tree = document.create_element("div")?;
    tree.set_id("mx-tree");
    root.append_child(&tree)?;

    let footnote = document.create_element("p")?;
    footnote.set_class_name("mx-footnote");
    footnote.set_text_content(Some(
        "VideoEncoder кодирует 30 кадров H.264. Muxer собирает ftyp + moov + mdat вручную. \
         Если Chrome не отдаёт description — извлекаем SPS/PPS из первого key chunk \
         и конвертируем chunks из AnnexB в AVCC. Round-trip проверяет, что всё сходится.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("mx-generate")
            .ok_or_else(|| JsValue::from_str("no #mx-generate"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            {
                let mut s = st.borrow_mut();
                s.reset();
                s.phase = String::from("кодирование");
            }
            update_ui(&doc, &st.borrow());
            spawn_local(async move {
                match encode_and_mux(doc_async.clone(), st_async.clone()).await {
                    Ok(()) => {}
                    Err(e) => {
                        let mut s = st_async.borrow_mut();
                        s.phase = String::from("ошибка");
                        s.error = Some(format_str(format_args!("{e:?}")));
                        drop(s);
                        update_ui(&doc_async, &st_async.borrow());
                    }
                }
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("mx-download")
            .ok_or_else(|| JsValue::from_str("no #mx-download"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let _ = trigger_download(&doc, &st);
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let st = state.borrow();
        update_ui(&document, &st);
        update_progress(&document, &st);
    }

    return Ok(());
}
```

## 23.11.8. Итог

**Muxing — обратная операция к demuxing.** На вход — массив `EncodedVideoChunk` и `avcC`. На выход — валидный MP4-файл.

**ftyp + moov + mdat.** 32 байта сигнатуры, все метаданные трека, конкатенация chunks.

**Циклическая зависимость `stco ↔ moov_size`** решается двухпроходной сборкой: сначала фиксируем размеры, потом значения.

**AnnexB fallback.** Chrome может не отдать `description` — тогда chunks приходят в AnnexB. Демо извлекает SPS/PPS из первого key chunk, строит `avcC` вручную и конвертирует всё в AVCC.

**Round-trip** — самый сильный тест. Собранный muxer-ом MP4 разбирается demuxer-ом из 23.10. Если sample count, sizes, timestamps и key flags совпадают — формат валиден.

**Правило проекта.** Демо имитирует реальный workflow: canvas → encode → mux → preview + download + round-trip. Не отдельные вызовы API, а полный путь.

**Дальше — 23.12 (Практический проект: MP4 → обработка → MP4)** — соединим 23.10 и 23.11 в сквозной pipeline: файл на входе, обработка пикселей между ними, файл на выходе.