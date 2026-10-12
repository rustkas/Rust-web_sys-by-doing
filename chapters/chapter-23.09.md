# Глава 23.9. Структура контейнеров: MP4 и WebM

## 23.9.0. Обновлённая структура

```
src/
  container_parser.rs             ← НОВЫЙ модуль 23.9
  video_frame.rs                  ← из 23.1 (переиспользуется: format_str)
  app.rs                          ← демо 23.9 (заменяется целиком)

tests/
  wasm_container_parser.rs        ← НОВЫЙ тест (11 тестов)

src/lib.rs                        ← +1 строка: pub mod container_parser;

web/main.js                       ← без изменений
vite.config.js                    ← без изменений
Cargo.toml                        ← без изменений
```

**Комментарий.** Модуль `container_parser.rs` — про **два бинарных формата контейнеров** с принципиально разной структурой. MP4 — вложенные box-ы с 32-битными размерами и FourCC-типами. WebM — EBML-дерево с variable-length integer ID и size. Задача — показать, **как читать чужие бинарные форматы** без внешних крейтов: только `js_sys::Uint8Array` и байтовые срезы. Префикс CSS — `cp-` (container parser), не пересекается с `vf-` (23.1), `ms-` (23.8).

**Ссылки для детального изучения:**

- ISO/IEC 14496-12: [ISO Base Media File Format (MP4 box structure)](https://www.iso.org/standard/68960.html)
- RFC 9559: [Matroska Media Container Format Specification](https://www.rfc-editor.org/rfc/rfc9559)
- RFC 8794: [Extensible Binary Meta Language (EBML)](https://www.rfc-editor.org/rfc/rfc8794)
- MDN: [`Uint8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array)
- FFmpeg Wiki: [How to check if Fast Start is enabled](https://trac.ffmpeg.org/wiki/HowToCheckIfFaststartIsEnabledForPlayback)
- `mp4box.js`: [JavaScript MP4 parser](https://github.com/gpac/mp4box.js)
- `webm-muxer`: [TypeScript WebM muxer](https://github.com/Vanilagy/webm-muxer)

---

## 23.9.1. О чём эта подглава

До сих пор WebCodecs работал с **данными** — кадрами и чанками. Но откуда берутся чанки? Из контейнеров. MP4 и WebM — два самых распространённых формата, и у них принципиально разная архитектура.

**MP4** — это дерево **box-ов** (называемых также atom-ами). Каждый box имеет:

- `size: u32` — полный размер в байтах, включая заголовок.
- `type: FourCC` — четыре байта, идентифицирующих тип (`ftyp`, `moov`, `mdat`, `trak`, `stbl`, `stsd`, `avcC`).
- `data` — содержимое (для контейнерных box-ов — вложенные box-ы).

Если `size == 1`, реальный размер хранится в следующих 8 байтах (`largesize: u64`). Это позволяет box-ам превышать 4 ГБ. Если `size == 0`, box тянется до конца файла.

**WebM** — это **EBML** (Extensible Binary Meta Language). Каждый элемент имеет:

- `id: VINT` — variable-length integer, 1–4 байта. Первый байт содержит ведущие нули, указывающие длину.
- `size: VINT` — variable-length integer, длина содержимого.
- `data` — содержимое (для мастер-элементов — вложенные элементы).

**Ключевые вопросы:**

- Как устроено дерево box-ов в MP4?
- Как читать variable-length integer в EBML?
- Чем `fast-start` MP4 отличается от обычного?
- Где в MP4 лежат SPS/PPS для H.264?
- Как извлечь параметры audio-трека из `stsd`?

**Демо** показывает **ручной парсинг** на уровне байтов: пользователь загружает MP4 или WebM, видит дерево box-ов/элементов, размеры, абсолютные offset-ы, ключевые метаданные, layout-bar, контекстные подсказки.

---

## 23.9.2. MP4: box-ы

### Базовый формат box-а

```
┌──────────────┬──────────────┬──────────────────────────┐
│ size (u32)   │ type (4 cc)  │ data (size - 8 байт)     │
└──────────────┴──────────────┴──────────────────────────┘
```

- **size** — полный размер box-а, включая заголовок.
- **type** — четыре байта ASCII.
- **data** — содержимое. Для контейнерных box-ов — вложенные box-ы.

### Расширенный размер

Если `size == 1`, после `type` идут 8 байт `largesize: u64`. Полный размер = 16 байт заголовка + `largesize`.

### Ключевые box-ы MP4

| Box | Назначение |
|-----|------------|
| `ftyp` | File Type Box — сигнатура файла, major brand, совместимые бренды |
| `moov` | Movie Box — все метаданные (треки, таблицы сэмплов) |
| `mdat` | Media Data Box — сами закодированные данные |
| `trak` | Track Box — один трек (видео или аудио) |
| `hdlr` | Handler Reference — тип трека (`vide`/`soun`/`hint`/`meta`) |
| `stbl` | Sample Table Box — таблицы для навигации по сэмплам |
| `stsd` | Sample Description Box — кодек и его параметры |
| `avcC` | AVC Configuration Box — SPS/PPS для H.264 |
| `stco` | Chunk Offset Box — смещения чанков в файле |
| `stsz` | Sample Size Box — размеры каждого сэмпла |
| `stsc` | Sample-to-Chunk Box — какие сэмплы в каком чанке |
| `stss` | Sync Sample Box — индексы ключевых кадров |

### Fast-start

В обычном MP4 `moov` стоит **после** `mdat`:

```
ftyp → mdat → moov
```

Плеер вынужден сначала скачать весь `mdat`, чтобы найти `moov`. При стриминге это долго.

**Fast-start** (web-optimized) — `moov` **перед** `mdat`:

```
ftyp → moov → mdat
```

Плеер сразу читает метаданные и может начинать воспроизведение, не дожидаясь всего файла.

### AudioSampleEntry

Внутри `stsd` для audio-трека лежит `AudioSampleEntry` с фиксированной раскладкой:

```
[0..4]   version+flags
[4..8]   entry_count
entry box:
  [8..12]   entry.size
  [12..16]  entry.type
  SampleEntry body (8 байт):
    [16..22] reserved (6)
    [22..24] data_reference_index (2)
  AudioSampleEntry additional body:
    [24..32] reserved (8)
    [32..34] channel_count
    [34..36] sample_size
    [36..38] pre_defined
    [38..40] reserved
    [40..44] sample_rate (fixed 16.16 — целая часть в старших 16 битах)
```

Для `ac-3` и `ec-3` есть расширения после базовой части, но базовые поля те же.

---

## 23.9.3. WebM: EBML

### Variable-length integer (VINT)

| Байт 1 | Длина | Диапазон ID |
|--------|-------|-------------|
| `1xxxxxxx` | 1 | 0x80–0xFE |
| `01xxxxxx` | 2 | 0x4000–0x7FFE |
| `001xxxxx` | 3 | 0x200000–0x3FFFFE |
| `0001xxxx` | 4 | 0x10000000–0x1FFFFFFE |

**ID** кодируется как есть (с ведущими битами). **Size** — ведущие биты отбрасываются.

### Ключевые элементы WebM

| ID (hex) | Название | Назначение |
|----------|----------|------------|
| `0x1A45DFA3` | EBML | Заголовок EBML |
| `0x18538067` | Segment | Корень |
| `0x1654AE6B` | Tracks | Описание потоков |
| `0xAE` | TrackEntry | Один трек |
| `0x1F43B675` | Cluster | Группа медиа-данных |
| `0xA3` | SimpleBlock | Один кадр |
| `0x86` | CodecID | Кодек (`V_VP8`, `A_OPUS`) |
| `0x4282` | DocType | `webm` или `matroska` |
| `0xB0` | PixelWidth | Ширина |
| `0xBA` | PixelHeight | Высота |

---

## 23.9.4. Решения по итогам практической работы

Этот раздел — самое ценное в подглаве: конкретные ошибки, найденные при отладке на реальных MP4-файлах, и их исправления.

### Баг 1. `(u64, u8)` вместо `(u8, u64)`

**Симптом.** Ошибка компиляции `expected u8, found u64` в `parse_mp4_at`.

**Причина.** В ветке `size_u32 == 0` (box тянется до конца файла) элементы кортежа шли в порядке `(size, header_size)`, а в двух других ветках — `(header_size, size)`.

**Решение.** Единый порядок `(u8, u64)` во всех трёх ветках:

```rust
} else if size_u32 == 0 {
    (8u8, (data.len() - offset) as u64)   // ← header_size, size
}
```

**Правило.** Если функция возвращает кортеж, порядок элементов должен быть одинаков во всех ветках. Компилятор это проверяет, но лучше — одна аннотация типа на все ветки:

```rust
let (header_size, size): (u8, u64) = if … { … } else if … { … } else { … };
```

### Баг 2. Относительные offset-ы вместо абсолютных

**Симптом.** В дереве `mvhd offset=0` — хотя `mvhd` реально на позиции 40.

**Причина.** `parse_mp4` вызывал сам себя рекурсивно с локальным буфером, и `offset` считался **от начала body родителя**, а не от начала файла.

**Решение.** Введена приватная `parse_mp4_at(data, base_offset)`, публичная `parse_mp4` — обёртка с `base_offset = 0`:

```rust
pub fn parse_mp4(data: &[u8]) -> Result<Vec<BoxInfo>, String> {
    parse_mp4_at(data, 0)
}

fn parse_mp4_at(data: &[u8], base_offset: usize) -> Result<Vec<BoxInfo>, String> {
    …
    if is_container {
        parse_mp4_at(body, base_offset + body_start)?  // ← передаём абсолютную базу
    }
    …
    offset: base_offset + offset  // ← абсолютный offset
}
```

То же для WebM: `parse_webm` / `parse_webm_at`.

**Правило.** Если структура вложенная и у каждого элемента есть позиция — позиция должна быть **абсолютной**. Относительные offset-ы сбивают с толку и не позволяют проверить в hex-редакторе.

### Баг 3. `avcC` не находился через `find("avcC")`

**Симптом.** В summary никогда не появлялась строка `avcC`, хотя дерево показывает `stsd`.

**Причина.** `avcC` лежит **внутри payload `stsd`**, а не как отдельный дочерний box. `parse_mp4` не рекурсирует внутрь `stsd` (он в списке не-контейнеров), поэтому `find("avcC")` всегда возвращает `None`.

**Решение.** Функция `find_avcc_in_stsd(stsd)` — ищет по смещению после VisualSampleEntry (8 + 8 + 78 = 94 байта от начала payload):

```rust
pub fn find_avcc_in_stsd(stsd: &BoxInfo) -> Option<Vec<u8>> {
    let p = &stsd.payload;
    let mut pos = 8 + 8 + 78;
    while pos + 8 <= p.len() {
        let size = u32::from_be_bytes([p[pos], p[pos+1], p[pos+2], p[pos+3]]) as usize;
        if size < 8 || pos + size > p.len() { break; }
        if &p[pos+4..pos+8] == b"avcC" {
            return Some(p[pos+8..pos+size].to_vec());
        }
        pos += size;
    }
    None
}
```

### Баг 4. `Tracks: 1 video, 1 audio` не отображалось

**Симптом.** В summary не было строки о треках, хотя на дереве два `trak`.

**Причина.** `analyze_mp4` вызывал `extract_hdlr_type(hdlr)`, но `payload` у `hdlr` был пустой: в `parse_mp4_at` список `keep_payload` не включал `"hdlr"`. Функция возвращала `"other"` для обоих треков.

**Решение.** Добавить `"hdlr"` в `keep_payload`:

```rust
let keep_payload = matches!(
    box_type.as_str(),
    "ftyp" | "stsd" | "avcC" | "stco" | "stsz" | "stss" | "stsc" | "hdlr"
);
```

**Правило.** Список `keep_payload` должен включать **все** box-ы, чей payload нужен вышестоящему коду. Легко забыть при добавлении нового анализа.

### Баг 5. `Chars` не реализует `ExactSizeIterator`

**Симптом.** Ошибка компиляции в `truncate_name`: `the trait bound Chars<'_>: ExactSizeIterator is not satisfied`.

**Причина.** Цепочка `stem.chars().rev().take(n).rev().collect()` не работает: `Rev<Chars>` не `DoubleEndedIterator`.

**Решение.** Собрать символы в `Vec<char>` и работать с индексами:

```rust
let stem_chars: Vec<char> = stem.chars().collect();
let head: String = stem_chars.iter().take(keep_head).collect();
let tail_start = stem_chars.len().saturating_sub(keep_tail);
let tail: String = stem_chars[tail_start..].iter().collect();
```

### Баг 6. `Closure<dyn FnMut()>` требует 0 аргументов

**Симптом.** Ошибка компиляции `closure is expected to take 0 arguments, but it takes 1 argument` в `read_file`.

**Причина.** Для `FileReader::set_onload` и `set_onerror` использовались замыкания вида `|_e: JsValue| …`. Но `Closure<dyn FnMut()>` ожидает ровно 0 аргументов.

**Решение.** Оба callback-а — без аргументов. JS-движок вызывает их с одним аргументом (Event), но Rust-замыкание без аргументов это игнорирует:

```rust
let onload: Closure<dyn FnMut()> = Closure::new(move || {
    let _ = resolve.call0(&JsValue::NULL);
});
let onerror: Closure<dyn FnMut()> = Closure::new(move || {
    let _ = reject.call1(&JsValue::NULL, &JsValue::from_str("read error"));
});
```

### Баг 7. `format!` теряет тип в tail-позиции

**Симптом.** Ошибка `expected String, found ()` в функциях, где последним выражением был `format!(…)` без обёртки.

**Решение.** Все `format!` в tail-позиции заменить на `format_str(format_args!(…))`:

```rust
fn line_err(msg: &str) -> String {
    format_str(format_args!(r#"<span class="err">{msg}</span>"#))
}
```

**Правило.** В WASM-коде используйте `format_str(format_args!(…))` для любой строковой интерполяции в tail-позиции функции, если она не в обычном `let`-биндинге.

### Баг 8. `matches!` в tail-позиции функции теряет тип

**Симптом.** Ошибка `expected bool, found ()` в `is_visual_codec`.

**Решение.** Заменить `matches!` на явный `match`:

```rust
fn is_visual_codec(codec: &str) -> bool {
    match codec {
        "avc1" | "avc3" | "hvc1" | "hev1" | "av01" | "vp09" | "mp4v" => true,
        _ => false,
    }
}
```

### Баг 9. Тесты WebM: VINT size не соответствует содержимому

**Симптом.** Тест `webm_segment_and_tracks` падал: `Tracks должен быть внутри Segment`.

**Причина.** В тестовых данных VINT-size у элементов был на 1 больше фактической длины. Например, `CodecID size=6` при payload `"V_VP8"` длиной 5.

**Решение.** Явно проверить соответствие и пересчитать:

```rust
let segment: &[u8] = &[
    0x18, 0x53, 0x80, 0x67,  // Segment id
    0x8E,                    // Segment size = 14
    0x16, 0x54, 0xAE, 0x6B,  // Tracks id
    0x89,                    // Tracks size = 9
    0xAE,                    // TrackEntry id
    0x87,                    // TrackEntry size = 7
    0x86,                    // CodecID id
    0x85,                    // CodecID size = 5
    b'V', b'_', b'V', b'P', b'8',
];
assert_eq!(segment.len(), 19, "тестовые данные должны быть 19 байт");
```

**Правило.** При ручной сборке EBML-структур в тестах **всегда** проверяйте через `assert_eq!(len(), expected)`. Одна лишняя единица в size — и парсер честно рассыпает дерево.

---

## 23.9.5. Исходные файлы

### `src/container_parser.rs`

```rust
//! 23.9. Структура контейнеров: MP4 и WebM.
//!
//! Ручной парсинг двух бинарных форматов без внешних крейтов:
//!
//! - **MP4** — дерево box-ов с 32-битными размерами и FourCC.
//! - **WebM** — EBML-дерево с variable-length integer ID и size.
//!
//! ## Публичный API
//!
//! - `parse_mp4(data) -> Result<Vec<BoxInfo>, String>` — дерево MP4 box-ов.
//! - `parse_webm(data) -> Result<Vec<EbmlElement>, String>` — дерево WebM.
//! - `is_fast_start(boxes) -> bool` — moov перед mdat.
//! - `parse_avcc(avcc) -> Result<(Vec<u8>, Vec<u8>), String>` — SPS/PPS.
//! - `find_avcc_in_stsd(stsd) -> Option<Vec<u8>>` — payload avcC из stsd.
//! - `format_bytes(n) -> String`, `hex_dump(data, limit) -> String`.
//!
//! ## Абсолютные offset-ы
//!
//! Внутренние `parse_mp4_at`/`parse_webm_at` принимают `base_offset` —
//! позицию текущего буфера в файле. Все `BoxInfo::offset` и
//! `EbmlElement::offset` — **абсолютные позиции в исходном файле**.

use crate::video_frame::format_str;

// =============================================================================
// MP4
// =============================================================================

/// Информация об одном MP4 box-е.
#[derive(Debug, Clone)]
pub struct BoxInfo {
    pub box_type: String,
    pub offset: usize,
    pub size: u64,
    pub header_size: u8,
    pub children: Vec<BoxInfo>,
    pub payload: Vec<u8>,
}

impl BoxInfo {
    pub fn find(&self, box_type: &str) -> Option<&BoxInfo> {
        if self.box_type == box_type {
            return Some(self);
        }
        for child in &self.children {
            if let Some(found) = child.find(box_type) {
                return Some(found);
            }
        }
        None
    }
}

/// Разбирает MP4 из байтов. Offset-ы — абсолютные (от начала файла).
pub fn parse_mp4(data: &[u8]) -> Result<Vec<BoxInfo>, String> {
    parse_mp4_at(data, 0)
}

fn parse_mp4_at(data: &[u8], base_offset: usize) -> Result<Vec<BoxInfo>, String> {
    let mut boxes = Vec::new();
    let mut offset = 0usize;

    while offset + 8 <= data.len() {
        let size_u32 = u32::from_be_bytes([
            data[offset],
            data[offset + 1],
            data[offset + 2],
            data[offset + 3],
        ]) as u64;

        let box_type = std::str::from_utf8(&data[offset + 4..offset + 8])
            .map_err(|e| format_str(format_args!("invalid box type: {e}")))?
            .to_string();

        // Порядок кортежа — (header_size: u8, size: u64) во ВСЕХ ветках.
        let (header_size, size): (u8, u64) = if size_u32 == 1 {
            if offset + 16 > data.len() {
                return Err(String::from("largesize truncated"));
            }
            let largesize = u64::from_be_bytes([
                data[offset + 8],
                data[offset + 9],
                data[offset + 10],
                data[offset + 11],
                data[offset + 12],
                data[offset + 13],
                data[offset + 14],
                data[offset + 15],
            ]);
            (16u8, largesize)
        } else if size_u32 == 0 {
            (8u8, (data.len() - offset) as u64)
        } else {
            (8u8, size_u32)
        };

        if size < header_size as u64 || offset as u64 + size > data.len() as u64 {
            return Err(format_str(format_args!(
                "invalid box size: type={box_type}, size={size}, offset={offset}"
            )));
        }

        let body_start = offset + header_size as usize;
        let body_end = offset + size as usize;
        let body = &data[body_start..body_end];

        let is_container = matches!(
            box_type.as_str(),
            "moov" | "trak" | "mdia" | "minf" | "stbl" | "edts" | "dinf"
        );

        let (children, payload) = if is_container {
            match parse_mp4_at(body, base_offset + body_start) {
                Ok(ch) => (ch, Vec::new()),
                Err(_) => (Vec::new(), body.to_vec()),
            }
        } else {
            // hdlr нужен для определения типа трека.
            let keep_payload = matches!(
                box_type.as_str(),
                "ftyp" | "stsd" | "avcC" | "stco" | "stsz" | "stss" | "stsc" | "hdlr"
            );
            (Vec::new(), if keep_payload { body.to_vec() } else { Vec::new() })
        };

        boxes.push(BoxInfo {
            box_type,
            offset: base_offset + offset,
            size,
            header_size,
            children,
            payload,
        });

        offset += size as usize;
    }

    Ok(boxes)
}

/// Проверяет fast-start: `moov` перед `mdat`.
pub fn is_fast_start(boxes: &[BoxInfo]) -> bool {
    let mut moov_pos = None;
    let mut mdat_pos = None;
    for (i, b) in boxes.iter().enumerate() {
        if b.box_type == "moov" {
            moov_pos = Some(i);
        }
        if b.box_type == "mdat" && mdat_pos.is_none() {
            mdat_pos = Some(i);
        }
    }
    match (moov_pos, mdat_pos) {
        (Some(m), Some(d)) => m < d,
        _ => false,
    }
}

/// Извлекает SPS/PPS из `avcC`.
pub fn parse_avcc(avcc: &[u8]) -> Result<(Vec<u8>, Vec<u8>), String> {
    if avcc.len() < 7 {
        return Err(String::from("avcC too short"));
    }
    let num_sps = (avcc[5] & 0x1F) as usize;
    let mut pos = 6usize;

    if num_sps == 0 {
        return Err(String::from("avcC: no SPS"));
    }
    if pos + 2 > avcc.len() {
        return Err(String::from("avcC: SPS length truncated"));
    }
    let sps_len = u16::from_be_bytes([avcc[pos], avcc[pos + 1]]) as usize;
    pos += 2;
    if pos + sps_len > avcc.len() {
        return Err(String::from("avcC: SPS truncated"));
    }
    let sps = avcc[pos..pos + sps_len].to_vec();
    pos += sps_len;

    if pos >= avcc.len() {
        return Err(String::from("avcC: no PPS count"));
    }
    let num_pps = avcc[pos] as usize;
    pos += 1;
    if num_pps == 0 {
        return Err(String::from("avcC: no PPS"));
    }
    if pos + 2 > avcc.len() {
        return Err(String::from("avcC: PPS length truncated"));
    }
    let pps_len = u16::from_be_bytes([avcc[pos], avcc[pos + 1]]) as usize;
    pos += 2;
    if pos + pps_len > avcc.len() {
        return Err(String::from("avcC: PPS truncated"));
    }
    let pps = avcc[pos..pos + pps_len].to_vec();

    Ok((sps, pps))
}

/// Ищет payload `avcC` внутри `stsd` для video-track.
///
/// `stsd` — не чистый контейнер. Его payload:
///   [version+flags: 4]
///   [entry_count: 4]
///   entry[0]:
///     [size: 4][type: 4]              ← 8 байт
///     VisualSampleEntry: 78 байт
///     child boxes:                     ← здесь лежит avcC
pub fn find_avcc_in_stsd(stsd: &BoxInfo) -> Option<Vec<u8>> {
    let p = &stsd.payload;
    // 8 (stsd header) + 8 (entry header) + 78 (VisualSampleEntry).
    let mut pos = 8 + 8 + 78;
    while pos + 8 <= p.len() {
        let size = u32::from_be_bytes([
            p[pos],
            p[pos + 1],
            p[pos + 2],
            p[pos + 3],
        ]) as usize;
        if size < 8 || pos + size > p.len() {
            break;
        }
        if &p[pos + 4..pos + 8] == b"avcC" {
            return Some(p[pos + 8..pos + size].to_vec());
        }
        pos += size;
    }
    None
}

// =============================================================================
// WebM (EBML)
// =============================================================================

#[derive(Debug, Clone)]
pub struct EbmlElement {
    pub id: u64,
    pub name: String,
    pub offset: usize,
    pub size: u64,
    pub total_size: u64,
    pub children: Vec<EbmlElement>,
    pub payload: Vec<u8>,
}

impl EbmlElement {
    pub fn find(&self, name: &str) -> Option<&EbmlElement> {
        if self.name == name {
            return Some(self);
        }
        for c in &self.children {
            if let Some(f) = c.find(name) {
                return Some(f);
            }
        }
        None
    }
}

fn read_vint(data: &[u8]) -> Option<(u64, usize)> {
    if data.is_empty() {
        return None;
    }
    let first = data[0];
    let length = first.leading_zeros() as usize + 1;
    if length > 8 || data.len() < length {
        return None;
    }
    let mut value: u64 = 0;
    for &b in &data[..length] {
        value = (value << 8) | b as u64;
    }
    Some((value, length))
}

fn read_id(data: &[u8]) -> Option<(u64, usize)> {
    read_vint(data)
}

fn read_size(data: &[u8]) -> Option<(u64, usize)> {
    let (v, len) = read_vint(data)?;
    if len > 4 {
        return None;
    }
    let marker_bit = 1u64 << (7 * len);
    Some((v & (marker_bit - 1), len))
}

fn ebml_name(id: u64) -> &'static str {
    match id {
        0x1A45DFA3 => "EBML",
        0x18538067 => "Segment",
        0x1654AE6B => "Tracks",
        0xAE => "TrackEntry",
        0x86 => "CodecID",
        0xE0 => "Video",
        0xB0 => "PixelWidth",
        0xBA => "PixelHeight",
        0x1F43B675 => "Cluster",
        0xE7 => "Timestamp",
        0xA3 => "SimpleBlock",
        0xEC => "Void",
        0xBF => "CRC-32",
        0x42F7 => "EBMLVersion",
        0x4286 => "EBMLReadVersion",
        0x42F2 => "EBMLMaxIDLength",
        0x42F3 => "EBMLMaxSizeLength",
        0x4282 => "DocType",
        0x4287 => "DocTypeVersion",
        0x4285 => "DocTypeReadVersion",
        _ => "Unknown",
    }
}

fn is_master(id: u64) -> bool {
    match id {
        0x1A45DFA3
        | 0x18538067
        | 0x1654AE6B
        | 0xAE
        | 0xE0
        | 0x1F43B675
        => true,
        _ => false,
    }
}

/// Разбирает WebM/EBML из байтов. Offset-ы — абсолютные.
pub fn parse_webm(data: &[u8]) -> Result<Vec<EbmlElement>, String> {
    parse_webm_at(data, 0)
}

fn parse_webm_at(data: &[u8], base_offset: usize) -> Result<Vec<EbmlElement>, String> {
    let mut elements = Vec::new();
    let mut offset = 0usize;

    while offset < data.len() {
        let (id, id_len) = read_id(&data[offset..])
            .ok_or_else(|| format_str(format_args!("invalid EBML id at offset {offset}")))?;

        let (size, size_len) = read_size(&data[offset + id_len..])
            .ok_or_else(|| format_str(format_args!("invalid EBML size at offset {offset}")))?;

        let total_header = id_len + size_len;
        let body_start = offset + total_header;
        let body_end = body_start
            .checked_add(size as usize)
            .ok_or_else(|| String::from("EBML size overflow"))?;

        if body_end > data.len() {
            return Err(format_str(format_args!(
                "EBML element truncated: id=0x{id:X}, size={size}, offset={offset}"
            )));
        }

        let body = &data[body_start..body_end];
        let name = ebml_name(id).to_string();

        let (children, payload) = if is_master(id) {
            match parse_webm_at(body, base_offset + body_start) {
                Ok(ch) => (ch, Vec::new()),
                Err(_) => (Vec::new(), body.to_vec()),
            }
        } else {
            (Vec::new(), body.to_vec())
        };

        elements.push(EbmlElement {
            id,
            name,
            offset: base_offset + offset,
            size,
            total_size: (total_header + size as usize) as u64,
            children,
            payload,
        });

        offset = body_end;
    }

    Ok(elements)
}

// =============================================================================
// Форматирование
// =============================================================================

/// Возвращает hex-строку байтов (до `limit`).
pub fn hex_dump(data: &[u8], limit: usize) -> String {
    let n = data.len().min(limit);
    let mut out = String::with_capacity(n * 3 + 2);
    for (i, b) in data[..n].iter().enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format_str(format_args!("{b:02X}")));
    }
    if data.len() > limit {
        out.push_str(" …");
    }
    out
}

/// Возвращает человекочитаемый размер: 512 B, 3.4 KB, 8.1 MB.
pub fn format_bytes(n: usize) -> String {
    const KB: f64 = 1024.0;
    const MB: f64 = KB * 1024.0;
    const GB: f64 = MB * 1024.0;
    let f = n as f64;
    if f < KB {
        format_str(format_args!("{n} B"))
    } else if f < MB {
        format_str(format_args!("{:.1} KB", f / KB))
    } else if f < GB {
        format_str(format_args!("{:.1} MB", f / MB))
    } else {
        format_str(format_args!("{:.2} GB", f / GB))
    }
}
```

### Как добавить в `src/lib.rs`

```rust
pub mod app;
pub mod container_parser;   // ← 23.9
pub mod video_frame;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

### `tests/wasm_container_parser.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;
use hello_wasm02::container_parser::{
    is_fast_start, parse_avcc, parse_mp4, parse_webm,
};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Вспомогательные конструкторы
// =============================================================================

fn mp4_box(box_type: &[u8; 4], body: &[u8]) -> Vec<u8> {
    let size = (8 + body.len()) as u32;
    let mut out = Vec::with_capacity(size as usize);
    out.extend_from_slice(&size.to_be_bytes());
    out.extend_from_slice(box_type);
    out.extend_from_slice(body);
    out
}

fn mp4_container(box_type: &[u8; 4], children: &[Vec<u8>]) -> Vec<u8> {
    let mut body = Vec::new();
    for c in children {
        body.extend_from_slice(c);
    }
    mp4_box(box_type, &body)
}

fn concat(parts: &[Vec<u8>]) -> Vec<u8> {
    let mut out = Vec::new();
    for p in parts {
        out.extend_from_slice(p);
    }
    out
}

// =============================================================================
// MP4
// =============================================================================

#[wasm_bindgen_test]
fn mp4_parses_ftyp_in_first_32_bytes() {
    let ftyp = mp4_box(b"ftyp", b"isom\x00\x00\x02\x00isomiso2avc1mp41");
    let boxes = parse_mp4(&ftyp).expect("parse ftyp");
    assert_eq!(boxes.len(), 1);
    assert_eq!(boxes[0].box_type, "ftyp");
    assert_eq!(boxes[0].offset, 0);
    assert!(boxes[0].size <= 32);
}

#[wasm_bindgen_test]
fn mp4_parses_mdat_size() {
    let payload = vec![0u8; 100];
    let mdat = mp4_box(b"mdat", &payload);
    let boxes = parse_mp4(&mdat).expect("parse mdat");
    assert_eq!(boxes[0].box_type, "mdat");
    assert_eq!(boxes[0].size, 108); // 8 + 100
}

#[wasm_bindgen_test]
fn mp4_parses_moov_tree() {
    let stsd = mp4_box(b"stsd", b"\x00\x00\x00\x00");
    let stbl = mp4_container(b"stbl", &[stsd]);
    let minf = mp4_container(b"minf", &[stbl]);
    let mdia = mp4_container(b"mdia", &[minf]);
    let trak = mp4_container(b"trak", &[mdia]);
    let moov = mp4_container(b"moov", &[trak]);

    let boxes = parse_mp4(&moov).expect("parse moov");
    assert_eq!(boxes[0].box_type, "moov");
    assert_eq!(boxes[0].children.len(), 1);
    assert_eq!(boxes[0].children[0].box_type, "trak");

    let stsd_found = boxes[0].find("stsd");
    assert!(stsd_found.is_some(), "stsd должен быть в дереве");
}

#[wasm_bindgen_test]
fn mp4_fast_start_detection() {
    let ftyp = mp4_box(b"ftyp", b"isom");
    let moov = mp4_box(b"moov", b"");
    let mdat = mp4_box(b"mdat", b"");

    let fast_bytes = concat(&[ftyp.clone(), moov.clone(), mdat.clone()]);
    let slow_bytes = concat(&[ftyp.clone(), mdat.clone(), moov.clone()]);

    let b1 = parse_mp4(&fast_bytes).unwrap();
    let b2 = parse_mp4(&slow_bytes).unwrap();

    assert!(is_fast_start(&b1));
    assert!(!is_fast_start(&b2));
}

#[wasm_bindgen_test]
fn mp4_avcc_extracts_sps_pps() {
    let avcc: &[u8] = &[
        0x01, 0x42, 0x00, 0x1E,
        0xFF,
        0xE1,
        0x00, 0x04, 0x67, 0x42, 0x00, 0x1E,
        0x01,
        0x00, 0x02, 0x68, 0xCE,
    ];
    let (sps, pps) = parse_avcc(avcc).expect("parse avcC");
    assert_eq!(sps.len(), 4);
    assert_eq!(pps.len(), 2);
    assert_eq!(sps[0], 0x67);
    assert_eq!(pps[0], 0x68);
}

// =============================================================================
// WebM
// =============================================================================

#[wasm_bindgen_test]
fn webm_starts_with_ebml_header() {
    let data: &[u8] = &[
        0x1A, 0x45, 0xDF, 0xA3,
        0x84,
        0x42, 0x86, 0x81, 0x01,
    ];
    let elems = parse_webm(data).expect("parse EBML");
    assert_eq!(elems.len(), 1);
    assert_eq!(elems[0].id, 0x1A45DFA3);
    assert_eq!(elems[0].name, "EBML");
}

#[wasm_bindgen_test]
fn webm_segment_and_tracks() {
    let segment: &[u8] = &[
        0x18, 0x53, 0x80, 0x67,
        0x8E,
        0x16, 0x54, 0xAE, 0x6B,
        0x89,
        0xAE,
        0x87,
        0x86,
        0x85,
        b'V', b'_', b'V', b'P', b'8',
    ];
    assert_eq!(segment.len(), 19, "тестовые данные должны быть 19 байт");

    let elems = parse_webm(segment).expect("parse Segment");
    assert_eq!(elems.len(), 1);
    assert_eq!(elems[0].id, 0x18538067);
    assert_eq!(elems[0].name, "Segment");

    let tracks_found = elems[0].find("Tracks");
    assert!(tracks_found.is_some(), "Tracks должен быть внутри Segment");

    let codec_id = elems[0].find("CodecID");
    assert!(codec_id.is_some());
    assert_eq!(codec_id.unwrap().payload, b"V_VP8");
}

#[wasm_bindgen_test]
fn webm_parses_multibyte_id() {
    let data: &[u8] = &[
        0x1F, 0x43, 0xB6, 0x75,
        0x80,
    ];
    let elems = parse_webm(data).expect("parse Cluster");
    assert_eq!(elems.len(), 1);
    assert_eq!(elems[0].id, 0x1F43B675);
    assert_eq!(elems[0].name, "Cluster");
    assert_eq!(elems[0].size, 0);
}

#[wasm_bindgen_test]
fn webm_simple_block() {
    let data: &[u8] = &[
        0xA3, 0x84, 0x81, 0x00, 0x00, 0x00,
    ];
    let elems = parse_webm(data).expect("parse SimpleBlock");
    assert_eq!(elems.len(), 1);
    assert_eq!(elems[0].id, 0xA3);
    assert_eq!(elems[0].name, "SimpleBlock");
    assert_eq!(elems[0].size, 4);
}

#[wasm_bindgen_test]
fn webm_rejects_truncated_element() {
    let data: &[u8] = &[0xA3, 0x8A];
    assert!(parse_webm(data).is_err());
}

#[wasm_bindgen_test]
fn mp4_rejects_invalid_box_size() {
    let data: &[u8] = &[0xFF, 0xFF, 0xFF, 0xFF, b'f', b't', b'y', b'p'];
    assert!(parse_mp4(data).is_err());
}
```

### `src/app.rs` — полный текст

Полный текст `app.rs`.

```
//! Глава 23.9 — Структура контейнеров: MP4 и WebM.
//!
//! Демо: загрузка MP4/WebM файла, ручной разбор на уровне
//! box-ов / EBML-элементов, вывод дерева, определение fast-start,
//! извлечение SPS/PPS из avcC, определение кодека и разрешения.
//!
//! ## Что демонстрируется
//!
//! - `FileReader` → `ArrayBuffer` → `Uint8Array` → `Vec<u8>`.
//! - Рекурсивный парсинг дерева MP4 box-ов (абсолютные offset-ы).
//! - Рекурсивный парсинг дерева WebM EBML-элементов.
//! - Определение fast-start (`moov` перед `mdat`).
//! - Извлечение SPS/PPS из `avcC` (внутри `stsd` video-трека).
//! - Извлечение параметров audio-трека (`mp4a`, sample rate, channels).
//! - Пропорциональный layout-bar с семантическими цветами.
//! - Контекстные подсказки — что означают увиденные структуры.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::Uint8Array;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    Document, File, FileReader, HtmlElement, HtmlInputElement,
};

use crate::container_parser::{
    find_avcc_in_stsd, format_bytes, hex_dump, is_fast_start, parse_avcc,
    parse_mp4, parse_webm, BoxInfo, EbmlElement,
};
use crate::video_frame::format_str;

// =============================================================================
// Константы
// =============================================================================

const MAX_TREE_NODES: usize = 400;
const MAX_NAME_CHARS: usize = 50;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.cp-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.cp-desc-line { margin:6px 0; }
.cp-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.cp-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.cp-btn:hover:not([disabled]) { background:#30363d; }
.cp-btn:disabled { opacity:0.5;cursor:not-allowed; }
.cp-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.cp-btn-primary:hover:not([disabled]) { background:#2ea043; }
.cp-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}

.cp-summary { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;overflow-wrap:anywhere;}
.cp-summary .row { display:flex;gap:12px; }
.cp-summary .lbl { color:#8b949e;min-width:130px;flex-shrink:0; }
.cp-summary .val { color:#79c0ff; }
.cp-summary .ok { color:#3fb950; }
.cp-summary .warn { color:#f0883e; }
.cp-summary .err { color:#f85149; }
.cp-summary .muted { color:#6e7681; }

.cp-layout { max-width:900px;display:flex;height:32px;margin:0 0 16px 0;border-radius:6px;overflow:hidden;font-family:ui-monospace,monospace;font-size:11px;font-weight:600;}
.cp-layout-seg { display:flex;align-items:center;justify-content:center;min-width:2.5em;overflow:hidden;white-space:nowrap;padding:0 4px;}
.cp-seg-ftyp { background:#58a6ff; color:#0d1117; }
.cp-seg-moov { background:#a371f7; color:#0d1117; }
.cp-seg-mdat { background:#3fb950; color:#0d1117; }
.cp-seg-free { background:#484f58; color:#c9d1d9; }
.cp-seg-other { background:#30363d; color:#c9d1d9; }

.cp-tree { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:12px;line-height:1.55;white-space:pre;overflow-x:auto;color:#8b949e;}
.cp-tree .name { font-weight:600; }
.cp-tree .pos { color:#6e7681; }
.cp-tree .ann { color:#a371f7; }
.cp-tree .muted { color:#6e7681; font-style:italic; }

/* MP4 */
.cp-box-ftyp { color:#58a6ff; }
.cp-box-moov, .cp-box-trak, .cp-box-mdia, .cp-box-minf, .cp-box-stbl, .cp-box-edts, .cp-box-dinf { color:#79c0ff; }
.cp-box-mdat { color:#f0883e; }
.cp-box-table { color:#a371f7; }
.cp-box-other { color:#8b949e; }

/* WebM */
.cp-ebml-header { color:#58a6ff; }
.cp-ebml-segment { color:#79c0ff; }
.cp-ebml-tracks, .cp-ebml-trackentry, .cp-ebml-video { color:#a371f7; }
.cp-ebml-cluster { color:#f0883e; }
.cp-ebml-prop { color:#3fb950; }
.cp-ebml-other { color:#8b949e; }

.cp-hints { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;}
.cp-hints .hint { margin:2px 0; }

.cp-stats { max-width:900px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;overflow-wrap:anywhere;word-break:break-word;}
.cp-stats .lbl { color:#8b949e; }
.cp-stats .val { color:#79c0ff; }
.cp-stats .err { color:#f85149; }
.cp-stats .muted { color:#6e7681; }

.cp-hex { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;color:#8b949e;white-space:pre;overflow-x:auto;}
.cp-hex .row-label { color:#79c0ff; font-weight:600; }
.cp-hex .bytes { color:#c9d1d9; }
.cp-hex .ascii { color:#6e7681; }

.cp-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    phase: String,
    error: Option<String>,
    /// `"MP4"` | `"WebM"` | `""`.
    format: String,
    file_name: String,
    file_size: usize,
    summary_html: String,
    layout_html: String,
    tree_html: String,
    hex_html: String,
    hints: Vec<String>,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            phase: String::from("простой"),
            error: None,
            format: String::new(),
            file_name: String::new(),
            file_size: 0,
            summary_html: String::new(),
            layout_html: String::new(),
            tree_html: String::new(),
            hex_html: String::new(),
            hints: Vec::new(),
        };
    }

    fn clear(&mut self) {
        self.summary_html.clear();
        self.layout_html.clear();
        self.tree_html.clear();
        self.hex_html.clear();
        self.hints.clear();
    }
}

// =============================================================================
// Хелперы строк
// =============================================================================

fn truncate_name(name: &str) -> String {
    let chars_count = name.chars().count();
    if chars_count <= MAX_NAME_CHARS {
        return name.to_string();
    }
    let (stem, ext) = match name.rfind('.') {
        Some(pos) if pos > 0 && name.len() - pos <= 8 => (&name[..pos], &name[pos..]),
        _ => (name, ""),
    };
    let stem_chars: Vec<char> = stem.chars().collect();
    if stem_chars.len() <= MAX_NAME_CHARS {
        return name.to_string();
    }
    let ext_chars: Vec<char> = ext.chars().collect();
    let keep_head = 32usize;
    let keep_tail = MAX_NAME_CHARS.saturating_sub(keep_head + 3 + ext_chars.len());
    let head: String = stem_chars.iter().take(keep_head).collect();
    let tail_start = stem_chars.len().saturating_sub(keep_tail);
    let tail: String = stem_chars[tail_start..].iter().collect();
    let ext_str: String = ext_chars.iter().collect();
    return format_str(format_args!("{head}…{tail}{ext_str}"));
}

fn row(label: &str, value_html: &str) -> String {
    return format_str(format_args!(
        r#"<div class="row"><span class="lbl">{label}:</span><span class="val">{value}</span></div>"#,
        label = label,
        value = value_html,
    ));
}

/// Осмысленные имена кодека.
fn codec_meaning(codec: &str) -> &'static str {
    match codec {
        // Video
        "avc1" | "avc3" => "H.264",
        "hvc1" | "hev1" => "H.265/HEVC",
        "av01" => "AV1",
        "vp09" => "VP9",
        "mp4v" => "MPEG-4",
        // Audio
        "mp4a" => "AAC",
        "ac-3" => "AC-3",
        "ec-3" => "E-AC-3",
        "alac" => "ALAC",
        "Opus" => "Opus",
        // WebM
        "V_VP8" => "VP8",
        "V_VP9" => "VP9",
        "V_AV1" => "AV1",
        "V_MPEG4/ISO/AVC" => "H.264",
        "A_OPUS" => "Opus",
        "A_VORBIS" => "Vorbis",
        _ => "",
    }
}

/// Известные video-кодеки с VisualSampleEntry layout.
fn is_visual_codec(codec: &str) -> bool {
    match codec {
        "avc1" | "avc3" | "hvc1" | "hev1" | "av01" | "vp09" | "mp4v" => true,
        _ => false,
    }
}

/// Формирует строку с кодеком и опциональным пояснением.
fn codec_with_meaning(codec: &str) -> String {
    let meaning = codec_meaning(codec);
    if meaning.is_empty() {
        return codec.to_string();
    }
    return format_str(format_args!("{codec} ({meaning})"));
}

/// Строка «Tracks: N video, M audio» или пустая, если треков нет.
fn tracks_row(video: usize, audio: usize) -> String {
    if video == 0 && audio == 0 {
        return String::new();
    }
    let parts = match (video, audio) {
        (v, 0) => format_str(format_args!("{v} video")),
        (0, a) => format_str(format_args!("{a} audio")),
        (v, a) => format_str(format_args!("{v} video, {a} audio")),
    };
    return row("Tracks", &parts);
}

// =============================================================================
// Факты о файле
// =============================================================================

struct Mp4Facts {
    brand: Option<String>,
    fast_start: bool,
    video_tracks: usize,
    audio_tracks: usize,
    codec: Option<String>,
    width: Option<u32>,
    height: Option<u32>,
    audio_codec: Option<String>,
    audio_sample_rate: Option<u32>,
    audio_channels: Option<u16>,
    avcc_info: Option<String>,
}

struct WebmFacts {
    doctype: Option<String>,
    codec: Option<String>,
    width: Option<u32>,
    height: Option<u32>,
}

// =============================================================================
// Анализ MP4
// =============================================================================

fn extract_brand(ftyp: &BoxInfo) -> Option<String> {
    if ftyp.payload.len() < 4 {
        return None;
    }
    return std::str::from_utf8(&ftyp.payload[..4]).ok().map(String::from);
}

/// Читает `handler_type` из `hdlr` (`vide` / `soun` / `hint` / `meta`).
fn extract_hdlr_type(hdlr: &BoxInfo) -> &'static str {
    let p = &hdlr.payload;
    if p.len() < 12 {
        return "other";
    }
    match &p[8..12] {
        b"vide" => "video",
        b"soun" => "audio",
        b"hint" => "hint",
        b"meta" => "meta",
        _ => "other",
    }
}

/// Извлекает кодек + размеры из `stsd` (video-трек).
fn extract_stsd_info(stsd: &BoxInfo) -> Option<(String, Option<u32>, Option<u32>)> {
    let p = &stsd.payload;
    if p.len() < 16 {
        return None;
    }
    let codec = std::str::from_utf8(&p[12..16]).ok()?.to_string();

    if !is_visual_codec(&codec) || p.len() < 44 {
        return Some((codec, None, None));
    }
    let width = u16::from_be_bytes([p[16 + 24], p[16 + 25]]) as u32;
    let height = u16::from_be_bytes([p[16 + 26], p[16 + 27]]) as u32;
    if width > 0 && height > 0 {
        return Some((codec, Some(width), Some(height)));
    }
    return Some((codec, None, None));
}

/// Извлекает кодек, sample rate и channels из `stsd` audio-трека.
fn extract_audio_info(stsd: &BoxInfo) -> Option<(String, Option<u32>, Option<u16>)> {
    let p = &stsd.payload;
    if p.len() < 16 {
        return None;
    }
    let codec = std::str::from_utf8(&p[12..16]).ok()?.to_string();
    if p.len() < 44 {
        return Some((codec, None, None));
    }
    let channels = u16::from_be_bytes([p[32], p[33]]);
    let sample_rate_fixed = u32::from_be_bytes([p[40], p[41], p[42], p[43]]);
    let sample_rate = sample_rate_fixed >> 16;
    return Some((
        codec,
        if sample_rate > 0 { Some(sample_rate) } else { None },
        if channels > 0 { Some(channels) } else { None },
    ));
}

fn analyze_mp4(boxes: &[BoxInfo]) -> Mp4Facts {
    let mut brand = None;
    let mut codec = None;
    let mut width = None;
    let mut height = None;
    let mut audio_codec = None;
    let mut audio_sample_rate = None;
    let mut audio_channels = None;
    let mut avcc_info = None;
    let mut video_tracks = 0usize;
    let mut audio_tracks = 0usize;

    for b in boxes {
        if b.box_type == "ftyp" {
            brand = extract_brand(b);
        }

        if b.box_type == "moov" {
            for trak in &b.children {
                if trak.box_type != "trak" {
                    continue;
                }
                let handler = trak
                    .find("hdlr")
                    .map(extract_hdlr_type)
                    .unwrap_or("other");

                match handler {
                    "video" => {
                        video_tracks += 1;
                        if codec.is_none() {
                            if let Some(stsd) = trak.find("stsd") {
                                if let Some((c, w, h)) = extract_stsd_info(stsd) {
                                    codec = Some(c);
                                    width = w;
                                    height = h;
                                }
                                if let Some(avcc_bytes) = find_avcc_in_stsd(stsd) {
                                    avcc_info = Some(match parse_avcc(&avcc_bytes) {
                                        Ok((sps, pps)) => format_str(format_args!(
                                            "SPS={sps_len} B · PPS={pps_len} B · SPS[0..4]={hex}",
                                            sps_len = sps.len(),
                                            pps_len = pps.len(),
                                            hex = hex_dump(&sps, 4),
                                        )),
                                        Err(e) => format_str(format_args!("ошибка: {e}")),
                                    });
                                }
                            }
                        }
                    }
                    "audio" => {
                        audio_tracks += 1;
                        if audio_codec.is_none() {
                            if let Some(stsd) = trak.find("stsd") {
                                if let Some((c, rate, ch)) = extract_audio_info(stsd) {
                                    audio_codec = Some(c);
                                    audio_sample_rate = rate;
                                    audio_channels = ch;
                                }
                            }
                        }
                    }
                    _ => {}
                }
            }
        }
    }

    return Mp4Facts {
        brand,
        fast_start: is_fast_start(boxes),
        video_tracks,
        audio_tracks,
        codec,
        width,
        height,
        audio_codec,
        audio_sample_rate,
        audio_channels,
        avcc_info,
    };
}

/// Формирует «, 48000 Hz, 2 ch» из sample_rate/channels.
fn audio_details(rate: Option<u32>, channels: Option<u16>) -> String {
    match (rate, channels) {
        (Some(sr), Some(ch)) => format_str(format_args!(", {sr} Hz, {ch} ch")),
        (Some(sr), None) => format_str(format_args!(", {sr} Hz")),
        (None, Some(ch)) => format_str(format_args!(", {ch} ch")),
        (None, None) => String::new(),
    }
}

fn hints_mp4(f: &Mp4Facts) -> Vec<String> {
    let mut h = Vec::new();

    if f.fast_start {
        h.push(String::from(
            "✓ moov (метаданные) перед mdat (данные) — файл готов к стримингу, \
             плеер читает метаданные сразу.",
        ));
    } else {
        h.push(String::from(
            "⚠ mdat (данные) перед moov (метаданные). Плееру придётся скачать \
             весь mdat, чтобы найти метаданные.",
        ));
    }

    if let Some(brand) = &f.brand {
        let meaning = match brand.as_str() {
            "isom" => "ISO Base Media (стандартный)",
            "mp41" | "mp42" => "MP4 v1 / v2",
            "qt  " => "QuickTime (Apple)",
            "M4V " => "MP4 Video",
            "iso2" | "iso5" | "iso6" => "ISO BMFF, современный",
            _ => "неизвестный бренд",
        };
        h.push(format_str(format_args!("Бренд: {brand} — {meaning}.")));
    }

    if let (Some(c), Some(w), Some(hh)) = (&f.codec, f.width, f.height) {
        h.push(format_str(format_args!(
            "Видео: {}, {w}×{hh}.",
            codec_with_meaning(c),
        )));
    } else if let Some(c) = &f.codec {
        h.push(format_str(format_args!("Видео-кодек: {}.", codec_with_meaning(c))));
    }

    if let Some(c) = &f.audio_codec {
        let details = audio_details(f.audio_sample_rate, f.audio_channels);
        h.push(format_str(format_args!(
            "Аудио: {}{details}.",
            codec_with_meaning(c),
        )));
    }

    if f.avcc_info.is_some() {
        h.push(String::from(
            "SPS (Sequence Parameter Set) и PPS (Picture Parameter Set) — \
             обязательные параметры H.264. Без них VideoDecoder.configure() \
             не запустит декодер.",
        ));
    }

    return h;
}

// =============================================================================
// Анализ WebM
// =============================================================================

fn collect_webm_text(elems: &[EbmlElement], name: &str, out: &mut Option<String>) {
    for e in elems {
        if e.name == name && out.is_none() && !e.payload.is_empty() {
            *out = Some(String::from_utf8_lossy(&e.payload).to_string());
        }
        collect_webm_text(&e.children, name, out);
    }
}

fn collect_webm_u32(elems: &[EbmlElement], name: &str, out: &mut Option<u32>) {
    for e in elems {
        if e.name == name && out.is_none() && !e.payload.is_empty() && e.payload.len() <= 4 {
            let mut v: u32 = 0;
            for &b in &e.payload {
                v = (v << 8) | b as u32;
            }
            *out = Some(v);
        }
        collect_webm_u32(&e.children, name, out);
    }
}

fn analyze_webm(elems: &[EbmlElement]) -> WebmFacts {
    let mut doctype = None;
    collect_webm_text(elems, "DocType", &mut doctype);

    let mut codec = None;
    collect_webm_text(elems, "CodecID", &mut codec);

    let mut width = None;
    collect_webm_u32(elems, "PixelWidth", &mut width);

    let mut height = None;
    collect_webm_u32(elems, "PixelHeight", &mut height);

    return WebmFacts {
        doctype,
        codec,
        width,
        height,
    };
}

fn hints_webm(f: &WebmFacts) -> Vec<String> {
    let mut h = Vec::new();
    if let Some(dt) = &f.doctype {
        let meaning = match dt.as_str() {
            "webm" => "WebM",
            "matroska" => "Matroska",
            _ => "неизвестный",
        };
        h.push(format_str(format_args!(
            "DocType: {dt} — контейнер {meaning}."
        )));
    }
    if let Some(c) = &f.codec {
        h.push(format_str(format_args!(
            "CodecID: {}.",
            codec_with_meaning(c),
        )));
    }
    if let (Some(w), Some(hh)) = (f.width, f.height) {
        h.push(format_str(format_args!("Разрешение видео: {w}×{hh}.")));
    }
    if h.is_empty() {
        h.push(String::from(
            "Структура EBML разобрана. Ключевые элементы: EBML → Segment → Tracks / Cluster.",
        ));
    }
    return h;
}

// =============================================================================
// Tree / layout / hex
// =============================================================================

fn mp4_box_class(box_type: &str) -> &'static str {
    match box_type {
        "ftyp" => "cp-box-ftyp",
        "moov" | "trak" | "mdia" | "minf" | "stbl" | "edts" | "dinf" => "cp-box-moov",
        "mdat" => "cp-box-mdat",
        "stsd" | "stco" | "stsz" | "stss" | "stsc" | "avcC" | "stts" => "cp-box-table",
        _ => "cp-box-other",
    }
}

fn webm_el_class(name: &str) -> &'static str {
    match name {
        "EBML" => "cp-ebml-header",
        "Segment" => "cp-ebml-segment",
        "Tracks" | "TrackEntry" | "Video" => "cp-ebml-tracks",
        "Cluster" => "cp-ebml-cluster",
        "CodecID" | "PixelWidth" | "PixelHeight" | "DocType" => "cp-ebml-prop",
        _ => "cp-ebml-other",
    }
}

/// Аннотация к box-у: `[avcC = N B]` для `stsd`.
fn mp4_box_annotation(b: &BoxInfo) -> String {
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

fn render_tree_mp4(boxes: &[BoxInfo], depth: usize, out: &mut String, count: &mut usize) {
    if *count >= MAX_TREE_NODES {
        return;
    }
    for b in boxes {
        if *count >= MAX_TREE_NODES {
            return;
        }
        *count += 1;

        let indent = "  ".repeat(depth);
        let cls = mp4_box_class(&b.box_type);
        let ann = mp4_box_annotation(b);
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
            render_tree_mp4(&b.children, depth + 1, out, count);
        }
    }
}

fn render_tree_webm(elems: &[EbmlElement], depth: usize, out: &mut String, count: &mut usize) {
    if *count >= MAX_TREE_NODES {
        return;
    }
    for e in elems {
        if *count >= MAX_TREE_NODES {
            return;
        }
        *count += 1;

        let indent = "  ".repeat(depth);
        let cls = webm_el_class(&e.name);

        // Для CodecID / DocType — показать payload как текст.
        let extra = if (e.name == "CodecID" || e.name == "DocType") && !e.payload.is_empty() {
            let s = String::from_utf8_lossy(&e.payload);
            format_str(format_args!(r#"  <span class="ann">= "{s}"</span>"#))
        } else {
            String::new()
        };

        out.push_str(&indent);
        out.push_str(&format_str(format_args!(
            r#"<span class="name {cls}">{name}</span> <span class="pos">offset={off} size={sz}</span>{extra}"#,
            cls = cls,
            name = e.name,
            off = e.offset,
            sz = format_bytes(e.size as usize),
            extra = extra,
        )));
        out.push('\n');

        if !e.children.is_empty() {
            render_tree_webm(&e.children, depth + 1, out, count);
        }
    }
}

fn build_tree_mp4(boxes: &[BoxInfo]) -> String {
    let mut inner = String::new();
    let mut count = 0usize;
    render_tree_mp4(boxes, 0, &mut inner, &mut count);
    if count >= MAX_TREE_NODES {
        inner.push_str(&format_str(format_args!(
            r#"<span class="muted">… показаны первые {MAX_TREE_NODES} узлов</span>"#
        )));
        inner.push('\n');
    }
    return format_str(format_args!(r#"<div class="cp-tree">{inner}</div>"#));
}

fn build_tree_webm(elems: &[EbmlElement]) -> String {
    let mut inner = String::new();
    let mut count = 0usize;
    render_tree_webm(elems, 0, &mut inner, &mut count);
    if count >= MAX_TREE_NODES {
        inner.push_str(&format_str(format_args!(
            r#"<span class="muted">… показаны первые {MAX_TREE_NODES} узлов</span>"#
        )));
        inner.push('\n');
    }
    return format_str(format_args!(r#"<div class="cp-tree">{inner}</div>"#));
}

fn seg_class(name: &str) -> &'static str {
    match name {
        "ftyp" => "cp-seg-ftyp",
        "moov" => "cp-seg-moov",
        "mdat" => "cp-seg-mdat",
        "moof" => "cp-seg-moov",
        "free" | "skip" | "wide" => "cp-seg-free",
        "EBML" => "cp-seg-ftyp",
        "Segment" => "cp-seg-moov",
        "Tracks" => "cp-seg-other",
        "Cluster" => "cp-seg-mdat",
        _ => "cp-seg-other",
    }
}

fn build_layout_html(parts: &[(String, u64)]) -> String {
    if parts.is_empty() {
        return String::new();
    }
    let mut html = String::from(r#"<div class="cp-layout">"#);
    for (name, size) in parts.iter() {
        html.push_str(&format_str(format_args!(
            r#"<div class="cp-layout-seg {cls}" style="flex:{flex}" title="{name} ({hr})">{name}</div>"#,
            cls = seg_class(name),
            flex = (*size).max(1),
            name = name,
            hr = format_bytes(*size as usize),
        )));
    }
    html.push_str("</div>");
    return html;
}

// =============================================================================
// Hex-превью
// =============================================================================

fn hex_row(bytes: &[u8], offset: usize) -> String {
    let n = bytes.len().min(16);
    let mut hex_part = String::with_capacity(50);
    let mut ascii_part = String::with_capacity(16);
    for i in 0..16 {
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

fn hex_block(label: &str, offset: usize, bytes: &[u8]) -> String {
    if offset + 16 > bytes.len() {
        return String::new();
    }
    let mut out = String::new();
    out.push_str(&format_str(format_args!(
        r#"<span class="row-label">— {label} (offset={off}) —</span>"#,
        label = label,
        off = offset,
    )));
    out.push('\n');
    out.push_str(&hex_row(&bytes[offset..offset + 16], offset));
    out.push('\n');
    return out;
}

fn build_hex_preview_mp4(bytes: &[u8], boxes: &[BoxInfo]) -> String {
    if bytes.len() < 16 {
        return String::new();
    }
    let mut out = String::from(r#"<div class="cp-hex">"#);
    for b in boxes {
        if b.box_type == "ftyp" || b.box_type == "moov" || b.box_type == "mdat" {
            out.push_str(&hex_block(&b.box_type, b.offset, bytes));
        }
    }
    out.push_str("</div>");
    return out;
}

fn build_hex_preview_webm(bytes: &[u8], elems: &[EbmlElement]) -> String {
    if bytes.len() < 16 {
        return String::new();
    }
    let mut out = String::from(r#"<div class="cp-hex">"#);
    for e in elems {
        if e.name == "EBML" || e.name == "Segment" {
            out.push_str(&hex_block(&e.name, e.offset, bytes));
        }
    }
    out.push_str("</div>");
    return out;
}

// =============================================================================
// UI helpers
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "cp-injected-style";
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
        "<strong>MP4</strong> — дерево box-ов. Каждый box имеет <code>size: u32</code> и <code>type: FourCC</code>. Ключевые: <code>ftyp</code> (сигнатура), <code>moov</code> (метаданные), <code>mdat</code> (данные).",
        "<strong>WebM</strong> — EBML-дерево. Каждый элемент имеет variable-length <code>id</code> и <code>size</code>. Корень — <code>Segment</code>, внутри — <code>Tracks</code> и <code>Cluster</code>.",
        "<strong>Fast-start</strong> — если <code>moov</code> стоит перед <code>mdat</code>, файл оптимизирован для стриминга. Layout-bar показывает это визуально.",
        "<strong>SPS/PPS</strong> — извлекаются из <code>avcC</code> внутри <code>stsd</code>. Без них <code>VideoDecoder</code> не инициализируется для H.264.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("cp-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("cp-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    return Ok(block.unchecked_into());
}

// =============================================================================
// Обработка файла
// =============================================================================

fn process_bytes(doc: &Document, st: Rc<RefCell<DemoState>>, name: String, bytes: Vec<u8>) {
    let is_mp4 = bytes.len() >= 8 && &bytes[4..8] == b"ftyp";
    let is_webm = bytes.len() >= 4 && &bytes[..4] == b"\x1A\x45\xDF\xA3";

    if is_mp4 {
        match parse_mp4(&bytes) {
            Ok(boxes) => {
                let facts = analyze_mp4(&boxes);

                let mut summary = String::new();
                let brand = facts.brand.clone().unwrap_or_else(|| "—".into());
                summary.push_str(&row(
                    "Format",
                    &format_str(format_args!(
                        r#"MP4 <span class="muted">(major brand: {brand})</span>"#
                    )),
                ));
                let fs = if facts.fast_start { "да" } else { "нет" };
                let fs_class = if facts.fast_start { "ok" } else { "warn" };
                summary.push_str(&format_str(format_args!(
                    r#"<div class="row"><span class="lbl">Fast-start:</span><span class="{cls}">{fs}</span></div>"#,
                    cls = fs_class,
                    fs = fs,
                )));
                summary.push_str(&tracks_row(facts.video_tracks, facts.audio_tracks));
                if let Some(c) = &facts.codec {
                    let dims = match (facts.width, facts.height) {
                        (Some(w), Some(h)) => format_str(format_args!("{w}×{h}")),
                        _ => "—".to_string(),
                    };
                    summary.push_str(&row(
                        "Video",
                        &format_str(format_args!("{}, {dims}", codec_with_meaning(c))),
                    ));
                }
                if let Some(c) = &facts.audio_codec {
                    let details = audio_details(facts.audio_sample_rate, facts.audio_channels);
                    summary.push_str(&row(
                        "Audio",
                        &format_str(format_args!("{}{details}", codec_with_meaning(c))),
                    ));
                }
                if let Some(info) = &facts.avcc_info {
                    summary.push_str(&row("avcC", info));
                }

                let hints = hints_mp4(&facts);

                let top: Vec<(String, u64)> = boxes
                    .iter()
                    .map(|b| (b.box_type.clone(), b.size))
                    .collect();
                let layout_html = build_layout_html(&top);
                let tree_html = build_tree_mp4(&boxes);
                let hex_html = build_hex_preview_mp4(&bytes, &boxes);

                let mut s = st.borrow_mut();
                s.clear();
                s.phase = String::from("файл разобран");
                s.format = String::from("MP4");
                s.file_name = name;
                s.file_size = bytes.len();
                s.summary_html = summary;
                s.layout_html = layout_html;
                s.tree_html = tree_html;
                s.hex_html = hex_html;
                s.hints = hints;
                drop(s);
                update_ui(doc, &st.borrow());
            }
            Err(e) => set_error(doc, st, name, bytes.len(), "MP4", e),
        }
    } else if is_webm {
        match parse_webm(&bytes) {
            Ok(elems) => {
                let facts = analyze_webm(&elems);

                let mut summary = String::new();
                summary.push_str(&row("Format", "WebM (EBML)"));
                let dt = facts.doctype.clone().unwrap_or_else(|| "—".into());
                summary.push_str(&row("DocType", &dt));
                if let Some(c) = &facts.codec {
                    summary.push_str(&row("CodecID", &codec_with_meaning(c)));
                }
                if let (Some(w), Some(h)) = (facts.width, facts.height) {
                    summary.push_str(&row(
                        "Video",
                        &format_str(format_args!("{w}×{h}")),
                    ));
                }

                let hints = hints_webm(&facts);

                let top: Vec<(String, u64)> = elems
                    .iter()
                    .map(|e| (e.name.clone(), e.size))
                    .collect();
                let layout_html = build_layout_html(&top);
                let tree_html = build_tree_webm(&elems);
                let hex_html = build_hex_preview_webm(&bytes, &elems);

                let mut s = st.borrow_mut();
                s.clear();
                s.phase = String::from("файл разобран");
                s.format = String::from("WebM");
                s.file_name = name;
                s.file_size = bytes.len();
                s.summary_html = summary;
                s.layout_html = layout_html;
                s.tree_html = tree_html;
                s.hex_html = hex_html;
                s.hints = hints;
                drop(s);
                update_ui(doc, &st.borrow());
            }
            Err(e) => set_error(doc, st, name, bytes.len(), "WebM", e),
        }
    } else {
        set_error(
            doc,
            st,
            name,
            bytes.len(),
            "unknown",
            String::from("неизвестный формат: ожидается ftyp (MP4) или 0x1A45DFA3 (WebM)"),
        );
    }
}

fn set_error(
    doc: &Document,
    st: Rc<RefCell<DemoState>>,
    name: String,
    size: usize,
    format: &str,
    error: String,
) {
    let mut s = st.borrow_mut();
    s.clear();
    s.phase = String::from("ошибка");
    s.error = Some(error);
    s.format = format.to_string();
    s.file_name = name;
    s.file_size = size;
    drop(s);
    update_ui(doc, &st.borrow());
}

// =============================================================================
// UI update
// =============================================================================

fn update_ui(doc: &Document, st: &DemoState) {
    // Summary.
    if let Some(el) = doc.get_element_by_id("cp-summary") {
        el.set_inner_html(&if st.summary_html.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#"<div class="cp-summary">{}</div>"#,
                st.summary_html,
            ))
        });
    }

    // Layout.
    if let Some(el) = doc.get_element_by_id("cp-layout") {
        el.set_inner_html(&st.layout_html);
    }

    // Hints.
    if let Some(el) = doc.get_element_by_id("cp-hints") {
        if st.hints.is_empty() {
            el.set_inner_html("");
        } else {
            let mut html = String::new();
            for h in &st.hints {
                html.push_str(&format_str(format_args!(
                    r#"<div class="hint">{h}</div>"#
                )));
            }
            el.set_inner_html(&html);
        }
    }

    // Tree.
    if let Some(el) = doc.get_element_by_id("cp-tree") {
        if st.tree_html.is_empty() {
            el.set_inner_html(
                r#"<span class="muted">Загрузите MP4 или WebM файл…</span>"#,
            );
        } else {
            el.set_inner_html(&st.tree_html);
        }
    }

    // Hex.
    if let Some(el) = doc.get_element_by_id("cp-hex") {
        el.set_inner_html(&st.hex_html);
    }

    // Stats.
    if let Some(el) = doc.get_element_by_id("cp-stats") {
        let err_line = st
            .error
            .as_ref()
            .map(|m| format_str(format_args!(r#"<div class="err">{m}</div>"#)))
            .unwrap_or_default();
        let file_info = if st.file_name.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#" · <span class="val">{name}</span> <span class="muted">({size})</span>"#,
                name = truncate_name(&st.file_name),
                size = format_bytes(st.file_size),
            ))
        };
        let fmt_info = if st.format.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#" · <span class="val">{fmt}</span>"#,
                fmt = st.format,
            ))
        };
        let html = format_str(format_args!(
            r#"{err}<div><span class="lbl">Статус:</span> <span class="val">{phase}</span>{fmt}{file}</div>"#,
            err = err_line,
            phase = st.phase,
            fmt = fmt_info,
            file = file_info,
        ));
        el.set_inner_html(&html);
    }
}

// =============================================================================
// Чтение файла
// =============================================================================

/// Читает файл через FileReader + Promise.
async fn read_file(file: File) -> Result<Vec<u8>, JsValue> {
    let reader = FileReader::new()?;

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
    let array_buffer = result
        .dyn_into::<js_sys::ArrayBuffer>()
        .map_err(|_| JsValue::from_str("result is not ArrayBuffer"))?;
    let uint8 = Uint8Array::new(&array_buffer);
    return Ok(uint8.to_vec());
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
    h2.set_text_content(Some("23.9 — Структура контейнеров: MP4 и WebM"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Controls.
    let controls = document.create_element("div")?;
    controls.set_class_name("cp-block");
    controls.set_inner_html(
        r#"<input type="file" id="cp-file" accept=".mp4,.m4v,.mov,.webm,.mkv,video/mp4,video/webm" style="display:none">
           <button class="cp-btn cp-btn-primary" id="cp-open">Открыть файл</button>
           <span class="cp-label">Парсер: MP4 boxes + EBML VINT</span>"#,
    );
    root.append_child(&controls)?;

    // Stats.
    let stats = document.create_element("div")?;
    stats.set_class_name("cp-stats");
    stats.set_id("cp-stats");
    stats.set_inner_html(
        r#"<div><span class="lbl">Статус:</span> <span class="val">простой</span></div>"#,
    );
    root.append_child(&stats)?;

    // Summary.
    let summary = document.create_element("div")?;
    summary.set_id("cp-summary");
    root.append_child(&summary)?;

    // Layout-bar.
    let layout = document.create_element("div")?;
    layout.set_id("cp-layout");
    root.append_child(&layout)?;

    // Hints.
    let hints = document.create_element("div")?;
    hints.set_class_name("cp-hints");
    hints.set_id("cp-hints");
    hints.set_inner_html("");
    root.append_child(&hints)?;

    // Hex preview.
    let hex_el = document.create_element("div")?;
    hex_el.set_id("cp-hex");
    root.append_child(&hex_el)?;

    // Tree.
    let tree = document.create_element("div")?;
    tree.set_id("cp-tree");
    tree.set_inner_html(r#"<span class="muted">Загрузите MP4 или WebM файл…</span>"#);
    root.append_child(&tree)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("cp-footnote");
    footnote.set_text_content(Some(
        "MP4: ftyp → moov → mdat. WebM: EBML → Segment → Tracks/Cluster. \
         Fast-start: moov перед mdat — видно на layout-bar. \
         SPS/PPS извлекаются из avcC (внутри stsd).",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    let input_el = document
        .get_element_by_id("cp-file")
        .ok_or_else(|| JsValue::from_str("no #cp-file"))?;
    let input: HtmlInputElement = input_el.dyn_into()?;

    // Кнопка «Открыть файл».
    {
        let input_for_btn = input.clone();
        let btn_el = document
            .get_element_by_id("cp-open")
            .ok_or_else(|| JsValue::from_str("no #cp-open"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            input_for_btn.click();
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Change на input.
    {
        let doc = document.clone();
        let st = state.clone();
        let input_clone = input.clone();
        let onchange: Closure<dyn FnMut()> = Closure::new(move || {
            let Some(files) = input_clone.files() else {
                return;
            };
            let Some(file) = files.get(0) else {
                return;
            };
            let name = file.name();

            {
                let mut s = st.borrow_mut();
                s.clear();
                s.phase = format_str(format_args!("чтение {name}…"));
                s.error = None;
            }
            update_ui(&doc, &st.borrow());

            let doc_cb = doc.clone();
            let st_cb = st.clone();
            spawn_local(async move {
                match read_file(file).await {
                    Ok(bytes) => {
                        // Даём браузеру отрисовать «чтение …».
                        TimeoutFuture::new(0).await;
                        process_bytes(&doc_cb, st_cb, name, bytes);
                    }
                    Err(e) => {
                        set_error(
                            &doc_cb,
                            st_cb,
                            name,
                            0,
                            "",
                            format_str(format_args!("read: {e:?}")),
                        );
                    }
                }
            });
        });
        input.add_event_listener_with_callback("change", onchange.as_ref().unchecked_ref())?;
        onchange.forget();
    }

    // Первичная отрисовка.
    {
        let st = state.borrow();
        update_ui(&document, &st);
    }

    return Ok(());
}
```

Ключевые функции:

- `extract_hdlr_type(hdlr) -> &'static str` — определяет `video`/`audio`/`hint`/`meta` по handler_type.
- `extract_stsd_info(stsd)` — для video-трека: кодек + width + height.
- `extract_audio_info(stsd)` — для audio-трека: кодек + sample_rate + channels.
- `find_avcc_in_stsd(stsd)` — из `container_parser`, вызывается для первого video-трека.
- `mp4_box_annotation(b)` — `[avc1]` / `[mp4a]` в дереве.
- `render_layout_bar(parts)` — пропорциональная полоса с семантическими цветами.
- `render_tree_mp4` / `render_tree_webm` — рекурсивный вывод с аннотациями и подсветкой.
- `truncate_name(name)` — обрезка длинных имён файлов.
- `process_bytes` — верхний уровень: парсинг + анализ + update UI.
- `read_file` — `FileReader` с 0-аргументными closures.

---

## 23.9.6. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты подглавы
wasm-pack test --headless --chrome  --test wasm_container_parser
wasm-pack test --headless --firefox --test wasm_container_parser

# 3. Демо
npm run wasm:dev && npm start
```

---

## 23.9.7. Итоговый вид демо

Для загруженного MP4-файла:

```
┌─ Summary ────────────────────────────────────────────────┐
│ Format:      MP4 (major brand: isom)                     │
│ Fast-start:  да                                          │
│ Tracks:      1 video, 1 audio                            │
│ Video:       avc1 (H.264), 720×1280                      │
│ Audio:       mp4a (AAC), 44100 Hz, 2 ch                  │
│ avcC:        SPS=30 B · PPS=6 B · SPS[0..4]=67 64 00 1F …│
└──────────────────────────────────────────────────────────┘

┌─ Layout bar ───────────────────────────────────┐
│ [ftyp][moov][free]      [mdat ……… 99.8%]       │
└────────────────────────────────────────────────┘

┌─ Hints ────────────────────────────────────────┐
│ ✓ moov (метаданные) перед mdat (данные) — файл │
│   готов к стримингу, плеер читает метаданные   │
│   сразу.                                       │
│ mdat занимает 99.8% файла — это и есть сами    │
│ видео-данные.                                  │
│ Бренд: isom — ISO Base Media (стандартный).    │
│ Видео: avc1 (H.264), 720×1280.                 │
│ Аудио: mp4a (AAC), 44100 Hz, 2 ch.             │
│ SPS (Sequence Parameter Set) и PPS (Picture    │
│ Parameter Set) — обязательные параметры H.264. │
│ Без них VideoDecoder.configure() не запустит   │
│ декодер.                                       │
└────────────────────────────────────────────────┘

┌─ Tree ─────────────────────────────────────────┐
│ ftyp  offset=0     size=32 B                   │
│ moov  offset=32    size=59.3 KB                │
│   mvhd  offset=40    size=108 B                │
│   trak  offset=148   size=46.5 KB              │
│     tkhd  offset=156   size=92 B               │
│     mdia  offset=284   size=46.4 KB            │
│       minf  offset=369   size=46.3 KB          │
│         stbl  offset=433   size=46.2 KB        │
│           stsd  offset=441   size=180 B [avc1] │
│           stts  offset=621   size=24 B         │
│           …                                    │
│ free  offset=60835 size=8 B                    │
│ mdat  offset=60843 size=25.0 MB                │
└────────────────────────────────────────────────┘
```

---

## 23.9.8. Итог

**MP4 и WebM — два принципиально разных контейнера.** MP4 — дерево box-ов с фиксированными типами. WebM — EBML-дерево с variable-length integer ID и size. Один парсер не подходит для обоих, но общий приём — «читать длину и тип, рекурсировать в контейнерные элементы» — работает для обоих.

**Абсолютные offset-ы.** Если структура вложенная и у каждого элемента есть позиция в исходном файле — позиция должна быть абсолютной. Относительные offset-ы сбивают с толку и не позволяют проверить в hex-редакторе.

**Fast-start** — это позиция `moov` относительно `mdat`. До `mdat` — файл оптимизирован для стриминга.

**SPS/PPS** лежат в `avcC`, а `avcC` — внутри payload `stsd`, а не как отдельный дочерний box. Искать `find("avcC")` бесполезно — нужен специальный проход по payload `stsd` с известным смещением.

**AudioSampleEntry** имеет фиксированную раскладку: `channel_count` на 32–34, `sample_rate` (fixed 16.16) на 40–44. Для большинства аудио-кодеков (`mp4a`, `Opus`, `alac`) этого достаточно.

**Правило проекта.** Демо показывает **реальный workflow**: файл → байты → парсер → дерево → анализ. Не отдельные вызовы API, а полный путь. Студент видит, что «магический» формат MP4 — это просто последовательность box-ов с size и type.

**Дальше — 23.10 (Demuxing: MP4 → EncodedVideoChunk)** — из таблиц `stbl` (`stco`, `stsc`, `stsz`, `stss`) извлекаются реальные видеочанки, готовые для `VideoDecoder`.