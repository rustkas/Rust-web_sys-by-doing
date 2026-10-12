# Глава 23.10. Demuxing: MP4 → EncodedVideoChunk

## 23.10.0. Обновлённая структура

```
src/
  container_parser.rs             ← из 23.9 (небольшая правка keep_payload)
  mp4_demuxer.rs                  ← НОВЫЙ модуль 23.10
  video_frame.rs                  ← из 23.1 (format_str, set_prop)
  app.rs                          ← демо 23.10 (заменяется целиком)

tests/
  wasm_container_parser.rs        ← из 23.9 (без изменений)
  wasm_mp4_demuxer.rs             ← НОВЫЙ тест (24 теста)

src/lib.rs                        ← +1 строка: pub mod mp4_demuxer;
```

**Комментарий.** Модуль `mp4_demuxer.rs` — про **перевод декларативного описания в конкретные байты файла**. 23.9 показала дерево box-ов; 23.10 идёт дальше: из таблиц `stbl` (`stco`, `stsc`, `stsz`, `stts`, `stss`) вычисляются **смещения и размеры каждого encoded-chunk-а** в `mdat` — ровно то, что нужно `VideoDecoder.decode(chunk)`. Префикс CSS — `dm-` (demuxer), не пересекается с `cp-` (23.9), `ms-` (23.8), `vf-` (23.1).

**Ссылки для детального изучения:**

- ISO/IEC 14496-12: [Sample Table Boxes (§8.5)](https://www.iso.org/standard/68960.html)
- Apple QuickTime File Format: [Sample Table Atoms](https://developer.apple.com/documentation/quicktime-file-format/sample_table_atoms)
- MP4RA: [Registered Box Types](https://mp4ra.org/#/atoms)
- W3C WebCodecs: [`EncodedVideoChunk`](https://www.w3.org/TR/webcodecs/#encodedvideochunk-interface)
- W3C WebCodecs: [`VideoDecoder.configure`](https://www.w3.org/TR/webcodecs/#dom-videodecoder-configure)
- `mp4box.js`: [Production MP4 parser](https://github.com/gpac/mp4box.js)

---

## 23.10.1. О чём эта подглава

23.9 научила нас читать структуру контейнера: какие box-ы существуют, где что лежит. Но там же остался нерешённый вопрос: **где именно в файле находятся закодированные кадры?** Мы видели `mdat offset=60843 size=25.0 MB` — и всё.

**Проблема.** `mdat` — плоский поток байтов. Внутри перемешаны video-сэмплы, audio-сэмплы, служебные NAL-юниты. Чтобы отдать `VideoDecoder.decode(chunk)`, нужно знать для каждого видео-сэмпла:

- **offset** — где в файле начинается этот sample.
- **size** — сколько байт он занимает.
- **timestamp** — media time этого sample.
- **duration** — сколько длится.
- **is_key** — это key frame (IDR) или delta?

Все эти данные есть в **таблицах сэмплов** внутри `stbl`. Но таблицы — реляционные: они хранят **чunk-ы** (группы сэмплов) и **сэмплы** в разных таблицах. Связь через `stsc`.

**Подглава отвечает на вопросы:**

- Как устроены пять таблиц `stbl` и как они связаны?
- Как из них вычислить offset/size/timestamp каждого sample-а?
- Как определить key frame через `stss`?
- Что делать, если `stss` пустой (all-key)?
- Как выглядит `description` для `VideoDecoder.configure`?
- Что за NAL-юниты лежат внутри sample-а?

**Демо** показывает: пользователь загружает MP4, видит число чанков, число key-кадров, время длительности, hex `description`, **карту key frames**, **NAL-разбор первых samples**, таблицу первых 30 сэмплов с offset/size/timestamp/is_key. Кнопка «Проверить конфиг» строит `VideoDecoder.isConfigSupported({codec, description})` — проверяет, что описание из `avcC` действительно валидно для WebCodecs.

---

## 23.10.2. Таблицы stbl

`stbl` содержит пять (иногда шесть) таблиц. Каждая — параллельный массив:

### `stco` — Chunk Offset Box

Список **абсолютных** смещений chunk-ов в файле. Chunk — группа сэмплов, лежащих подряд в `mdat`.

```
[version+flags: 4]
[entry_count: 4]
[offset_0: 4][offset_1: 4]...[offset_N: 4]
```

**Зачем chunk-ы.** В `mdat` сэмплы группируются так, чтобы размер одной операции чтения был не слишком маленьким. Например, если каждый video-сэмпл ~12 КБ, то по 10 сэмплов в чанк = 120 КБ за операцию. Так быстрее, чем 12 КБ × 10 раз.

### `stsc` — Sample-to-Chunk Box

Правило: какие сэмплы в каком чанке. Хранится сжато — runs:

```
[first_chunk: 4][samples_per_chunk: 4][sample_description_index: 4]
```

Например: `[(1, 10, 1)]` — с 1-го чанка и до конца по 10 сэмплов. Или `[(1, 10, 1), (5, 8, 1)]` — с 1-го по 4-й чанк по 10 сэмплов, с 5-го и дальше — по 8.

### `stsz` — Sample Size Box

Два варианта:

```
[version+flags: 4][sample_size: 4][sample_count: 4]
```

Если `sample_size != 0` — все сэмплы одного размера (редко для видео, чаще для аудио). Если `0` — далее идёт `sample_count` четвёрок: размер каждого сэмпла.

### `stts` — Time-to-Sample Box

Пары `(sample_count, sample_delta)` — сколько сэмплов имеют данную длительность. Если frame rate постоянный — одна пара `(N, 1000)` для 30 fps при timescale=30000. Если VFR — несколько пар.

### `stss` — Sync Sample Box

Список 1-based индексов **key frames**. Может отсутствовать — тогда **все** сэмплы ключевые (например, для animation).

### `mdhd` — Media Header Box

Лежит в `mdia`, рядом с `hdlr`. Содержит **timescale** — знаменатель для timestamp-ов.

```
version 0: [ver+flags: 4][ctime: 4][mtime: 4][timescale: 4][duration: 4]
version 1: 64-битные ctime/mtime — timescale на offset 20.
```

Если `mdhd.timescale = 30000`, то `sample_delta = 1000` означает 1000/30000 = 33.333 мс.

---

## 23.10.3. Алгоритм: stbl → список samples

**Шаг 1.** Пройти по chunk-ам. Для chunk-а `k` (1-based) узнать число сэмплов через `stsc`. Offset первого сэмпла = `stco[k-1]`.

**Шаг 2.** Внутри chunk-а пройти по сэмплам, суммируя размеры из `stsz`. Offset `i`-го сэмпла в chunk-е = `chunk_offset + Σ stsz[предыдущих]`.

**Шаг 3.** Timestamp. Обходим `stts` слева направо, накапливая время. Каждый сэмпл получает `cumulative_time * 1_000_000 / timescale` в микросекундах.

**Шаг 4.** `is_key`: если `stss` пуст — все true. Иначе `(i+1) ∈ stss`.

---

## 23.10.4. Решения по итогам практической работы

### Решение 1. Не расширять `container_parser.rs` без нужды

Для demuxer-а нужны payload-ы **`stts`** и **`mdhd`**, которых не было в 23.9. Правка — одна строка в `keep_payload`:

```rust
let keep_payload = matches!(
    box_type.as_str(),
    "ftyp" | "stsd" | "avcC" | "hdlr" | "mdhd"
    | "stco" | "stsz" | "stss" | "stsc" | "stts"
);
```

**Правило.** Один модуль — одна ответственность. Demuxer читает payload, но не парсит контейнер сам.

### Решение 2. Один разбор таблиц — одна функция

Пять таблиц парсятся пятью функциями с одинаковой сигнатурой `fn parse_xxx(payload: &[u8]) -> Result<Vec<...>, String>`. Форматы записей разные (4/8/12 байт), поэтому общий хелпер не помогает — экономия на спичках.

### Решение 3. `samples_in_chunk` — линейный поиск с конца по stsc

`stsc` — маленькая таблица (обычно < 5 записей). Линейный поиск с конца дешевле бинарного и понятнее:

```rust
fn samples_in_chunk(stsc: &[StscEntry], chunk_1based: usize) -> u32 {
    for i in (0..stsc.len()).rev() {
        if stsc[i].first_chunk as usize <= chunk_1based {
            return stsc[i].samples_per_chunk;
        }
    }
    0
}
```

### Решение 4. `stss` пустой = все ключевые

```rust
let is_key = if stss.is_empty() {
    true
} else {
    stss.binary_search(&(index as u32 + 1)).is_ok()
};
```

### Решение 5. Проверка границ — до построения UI

Все offset-ы из `stco` должны быть **внутри** диапазона `mdat`. Если нет — sample table повреждена. Проверка стоит один проход:

```rust
pub fn verify_offsets(
    samples: &[VideoSample],
    mdat_start: u64,
    mdat_end: u64,
) -> Result<(), String> { ... }
```

Это самопроверка: если demuxer ошибся в арифметике, demo сразу показывает красный флаг.

### Решение 6. Codec string для `isConfigSupported`

Для H.264 — `avc1.PPCCLL`, где PP/CC/LL — байты 1..4 из `avcC`:

```rust
fn avc_codec_string(avcc: &[u8]) -> Option<String> {
    if avcc.len() < 4 { return None; }
    Some(format_str(format_args!(
        "avc1.{:02X}{:02X}{:02X}",
        avcc[1], avcc[2], avcc[3],
    )))
}
```

### Решение 7. `parse_avcc_nals` — length-prefix вместо start codes

В MP4 H.264 sample-ы хранятся в формате **AVCC**: каждый NAL предваряется length-prefix длиной `nal_length_size` байт (`avcC[4] & 0x03 + 1`). Это не Annex B с `00 00 01` start codes. Чтобы показать NAL-структуру, нужно читать length-prefix, а не искать старт-код.

### Решение 8. Grouping в keymap

Для 2175 samples один символ на sample — 2175 символов, которые бесполезны в UI. Grouping по 10: символ показывает, есть ли в группе key frames. Плюс перенос строки раз в 80 символов.

### Баг 1. Тест VFR ожидал неверный timestamp

**Симптом.** `multi_entry_stts_supports_vfr` падал: ожидал `166_666`, парсер вернул `116_666`.

**Причина.** В комментарии к тесту была ошибка: «2\*33.333 + 2\*50.000». Но к моменту s3 прошёл только **один** 50-мс интервал (между s2 и s3). Второй наступит после s3 и относится к s4, которого нет.

**Решение.** Правильное ожидание — `116_666`, полный набор timestamp-ов для наглядности:

```rust
assert_eq!(samples[0].timestamp_us, 0);
assert_eq!(samples[1].timestamp_us, 33_333);
assert_eq!(samples[2].timestamp_us, 66_666);
assert_eq!(samples[3].timestamp_us, 116_666);
```

### Баг 2. `(u64, u8)` в ветке `size_u32 == 0`

**Симптом.** Ошибка компиляции `expected u8, found u64` в `parse_mp4_at`.

**Причина.** В ветке `size_u32 == 0` элементы кортежа шли в порядке `(size, header_size)`, а в двух других — `(header_size, size)`.

**Решение.** Единый порядок `(u8, u64)` во всех ветках:

```rust
} else if size_u32 == 0 {
    (8u8, (data.len() - offset) as u64)
}
```

**Правило.** Если функция возвращает кортеж, порядок элементов должен быть одинаков во всех ветках.

### Баг 3. `return` в хелперах WASM-кода

**Симптом.** `expected Vec<u8>, found ()` в `v0_flags()` тестов.

**Причина.** Rust-analyzer теряет тип у tail-выражения функции, если оно состоит из макроса или литерала без явной типизации, в WASM-контексте.

**Решение.** Добавлять `return` в конце:

```rust
fn v0_flags() -> Vec<u8> {
    return vec![0u8; 4];
}
```

**Правило проекта.** В WASM-контексте в хелперах используйте `return` перед tail-выражением, если оно — макрос, конструктор структуры или вызов функции без явной типизации. Это симметрично правилу про `format_str(format_args!(...))`.

### Баг 4. Утечка памяти в NalUnit

**Симптом.** `field offset_in_sample is never read`.

**Причина.** Поле избыточно: offset NAL-а вычисляется из предыдущих `pos + nal_length_size + length`, а в UI показывается только `index`, `length`, `nal_type`.

**Решение.** Удалить поле.

**Правило.** Не хранить в структурах данные, которые не читаются. `dead_code` — не безобидное предупреждение.

### Баг 5. `document.createElement` (JS-стиль) вместо `document.create_element`

**Симптом.** `no method createElement in web_sys::Document`.

**Причина.** В JS DOM-методы camelCase, в Rust с web-sys — snake_case.

**Решение.** Все DOM-методы — snake_case: `create_element`, `get_element_by_id`, `set_attribute`, `add_event_listener_with_callback`.

### Решение 9. `TimeoutFuture::new(0).await` между фазами

В `process_bytes` — три yield-точки: после `parse_mp4`, после `build_samples`, перед обновлением UI. Для файла 25 MB это незаметно, но для 100k+ samples блокировка UI была бы ощутимой.

**Правило проекта.** Любая длительная async-операция, обновляющая UI, должна явно отдавать управление через `TimeoutFuture::new(0).await`.

---

## 23.10.5. Исходные файлы

### `src/container_parser.rs` — правка (одна строка)

Найдите в `parse_mp4_at`:

```rust
let keep_payload = matches!(
    box_type.as_str(),
    "ftyp" | "stsd" | "avcC" | "hdlr" | "mdhd"
    | "stco" | "stsz" | "stss" | "stsc" | "stts"
);
```

Если в предыдущей версии было без `mdhd` и `stts` — добавьте их. Остальной файл — без изменений.

### `src/mp4_demuxer.rs` — НОВЫЙ

```rust
//! 23.10. Demuxing: MP4 → EncodedVideoChunk.
//!
//! Превращает таблицы `stbl` в плоский список encoded-chunks
//! (samples), готовых для `VideoDecoder.decode(chunk)`.
//!
//! ## Пять таблиц
//!
//! - `stco` — chunk offsets (абсолютные, в файле).
//! - `stsc` — samples per chunk (runs).
//! - `stsz` — размер каждого sample (или общий fixed size).
//! - `stts` — (sample_count, sample_delta) — длительности.
//! - `stss` — 1-based индексы key frames (или пусто = все key).
//!
//! ## Timescale
//!
//! Отдельно передаётся из `mdhd`. Все timestamp-ы выражаются
//! в его единицах и потом конвертируются в микросекунды.
//!
//! ## Публичный API
//!
//! - `parse_sample_table(stbl, timescale) -> SampleTable`
//! - `build_samples(&SampleTable) -> Vec<VideoSample>`
//! - `verify_offsets(&[VideoSample], mdat_start, mdat_end) -> Result<(), String>`
//! - `avc_codec_string(avcc) -> Option<String>`

use crate::container_parser::BoxInfo;
use crate::video_frame::format_str;

// =============================================================================
// Структуры
// =============================================================================

/// Все таблицы stbl, необходимые для demuxing-а.
#[derive(Debug, Clone)]
pub struct SampleTable {
    /// Timescale из `mdhd` — знаменатель timestamp-ов.
    pub timescale: u32,
    /// Chunk offsets из `stco` (абсолютные).
    pub stco: Vec<u32>,
    /// Rules из `stsc`.
    pub stsc: Vec<StscEntry>,
    /// Размеры сэмплов из `stsz` (если stsz_fixed = None).
    pub stsz: Vec<u32>,
    /// Фиксированный размер сэмплов (если stsz version 0 и sample_size != 0).
    pub stsz_fixed: Option<u32>,
    /// Длительности из `stts`.
    pub stts: Vec<SttsEntry>,
    /// Key frame indices (1-based). Пустой = все ключевые.
    pub stss: Vec<u32>,
}

#[derive(Debug, Clone, Copy)]
pub struct StscEntry {
    pub first_chunk: u32,
    pub samples_per_chunk: u32,
}

#[derive(Debug, Clone, Copy)]
pub struct SttsEntry {
    pub sample_count: u32,
    pub sample_delta: u32,
}

/// Один encoded chunk — что подаётся в `VideoDecoder.decode`.
#[derive(Debug, Clone)]
pub struct VideoSample {
    pub index: usize,
    pub offset: u64,
    pub size: u32,
    pub timestamp_us: u64,
    pub duration_us: u32,
    pub is_key: bool,
}

impl SampleTable {
    /// Общее число сэмплов.
    pub fn total_samples(&self) -> usize {
        match self.stsz_fixed {
            Some(_) => {
                let mut total = 0u32;
                let chunks = self.stco.len();
                for k in 1..=chunks {
                    total += samples_in_chunk(&self.stsc, k);
                }
                total as usize
            }
            None => self.stsz.len(),
        }
    }
}

// =============================================================================
// Парсинг отдельных таблиц
// =============================================================================

fn read_u32_be(data: &[u8], off: usize) -> u32 {
    return u32::from_be_bytes([data[off], data[off + 1], data[off + 2], data[off + 3]]);
}

/// `stco`: [version+flags: 4][entry_count: 4][offset: 4 × N]
fn parse_stco(p: &[u8]) -> Result<Vec<u32>, String> {
    if p.len() < 8 {
        return Err(String::from("stco too short"));
    }
    let count = read_u32_be(p, 4) as usize;
    if p.len() < 8 + count * 4 {
        return Err(String::from("stco truncated"));
    }
    let mut out = Vec::with_capacity(count);
    for i in 0..count {
        out.push(read_u32_be(p, 8 + i * 4));
    }
    return Ok(out);
}

/// `stsc`: [version+flags: 4][entry_count: 4][(fc,spc,sdi): 12 × N]
fn parse_stsc(p: &[u8]) -> Result<Vec<StscEntry>, String> {
    if p.len() < 8 {
        return Err(String::from("stsc too short"));
    }
    let count = read_u32_be(p, 4) as usize;
    if p.len() < 8 + count * 12 {
        return Err(String::from("stsc truncated"));
    }
    let mut out = Vec::with_capacity(count);
    for i in 0..count {
        let base = 8 + i * 12;
        out.push(StscEntry {
            first_chunk: read_u32_be(p, base),
            samples_per_chunk: read_u32_be(p, base + 4),
        });
    }
    return Ok(out);
}

/// `stsz`: [ver+flags: 4][sample_size: 4][sample_count: 4][sizes × N if sample_size == 0]
fn parse_stsz(p: &[u8]) -> Result<(Option<u32>, Vec<u32>), String> {
    if p.len() < 12 {
        return Err(String::from("stsz too short"));
    }
    let sample_size = read_u32_be(p, 4);
    let count = read_u32_be(p, 8) as usize;
    if sample_size != 0 {
        return Ok((Some(sample_size), Vec::new()));
    }
    if p.len() < 12 + count * 4 {
        return Err(String::from("stsz truncated"));
    }
    let mut out = Vec::with_capacity(count);
    for i in 0..count {
        out.push(read_u32_be(p, 12 + i * 4));
    }
    return Ok((None, out));
}

/// `stts`: [version+flags: 4][entry_count: 4][(count,delta): 8 × N]
fn parse_stts(p: &[u8]) -> Result<Vec<SttsEntry>, String> {
    if p.len() < 8 {
        return Err(String::from("stts too short"));
    }
    let count = read_u32_be(p, 4) as usize;
    if p.len() < 8 + count * 8 {
        return Err(String::from("stts truncated"));
    }
    let mut out = Vec::with_capacity(count);
    for i in 0..count {
        let base = 8 + i * 8;
        out.push(SttsEntry {
            sample_count: read_u32_be(p, base),
            sample_delta: read_u32_be(p, base + 4),
        });
    }
    return Ok(out);
}

/// `stss`: [version+flags: 4][entry_count: 4][sample_number: 4 × N]
fn parse_stss(p: &[u8]) -> Result<Vec<u32>, String> {
    if p.len() < 8 {
        return Err(String::from("stss too short"));
    }
    let count = read_u32_be(p, 4) as usize;
    if p.len() < 8 + count * 4 {
        return Err(String::from("stss truncated"));
    }
    let mut out = Vec::with_capacity(count);
    for i in 0..count {
        out.push(read_u32_be(p, 8 + i * 4));
    }
    return Ok(out);
}

// =============================================================================
// Публичные функции
// =============================================================================

/// Извлекает все таблицы из `stbl`.
pub fn parse_sample_table(stbl: &BoxInfo, timescale: u32) -> Result<SampleTable, String> {
    let stco = stbl.find("stco").ok_or_else(|| String::from("no stco"))?;
    let stsc = stbl.find("stsc").ok_or_else(|| String::from("no stsc"))?;
    let stsz = stbl.find("stsz").ok_or_else(|| String::from("no stsz"))?;
    let stts = stbl.find("stts").ok_or_else(|| String::from("no stts"))?;
    let stss = stbl.find("stss");

    let (stsz_fixed, stsz_vec) = parse_stsz(&stsz.payload)?;

    return Ok(SampleTable {
        timescale,
        stco: parse_stco(&stco.payload)?,
        stsc: parse_stsc(&stsc.payload)?,
        stsz: stsz_vec,
        stsz_fixed,
        stts: parse_stts(&stts.payload)?,
        stss: match stss {
            Some(b) => parse_stss(&b.payload)?,
            None => Vec::new(),
        },
    });
}

/// Сколько сэмплов в chunk-е `chunk_1based`.
fn samples_in_chunk(stsc: &[StscEntry], chunk_1based: usize) -> u32 {
    for i in (0..stsc.len()).rev() {
        if stsc[i].first_chunk as usize <= chunk_1based {
            return stsc[i].samples_per_chunk;
        }
    }
    return 0;
}

/// Строит список `VideoSample` — по одному на encoded chunk.
pub fn build_samples(t: &SampleTable) -> Result<Vec<VideoSample>, String> {
    let num_chunks = t.stco.len();
    let total = t.total_samples();
    let mut samples = Vec::with_capacity(total);

    // Проходим по chunk-ам, внутри — по сэмплам.
    let mut sample_index: usize = 0;
    for chunk_1based in 1..=num_chunks {
        if sample_index >= total {
            break;
        }
        let chunk_offset = t.stco[chunk_1based - 1] as u64;
        let n = samples_in_chunk(&t.stsc, chunk_1based) as usize;
        let mut cur_off = chunk_offset;

        for _ in 0..n {
            if sample_index >= total {
                break;
            }
            let size = match t.stsz_fixed {
                Some(s) => s,
                None => t.stsz[sample_index],
            };
            samples.push(VideoSample {
                index: sample_index,
                offset: cur_off,
                size,
                timestamp_us: 0,
                duration_us: 0,
                is_key: false,
            });
            cur_off += size as u64;
            sample_index += 1;
        }
    }

    if samples.len() != total {
        return Err(format_str(format_args!(
            "построено {} сэмплов, ожидалось {}",
            samples.len(),
            total,
        )));
    }

    // Timestamp-ы: обходим stts, накапливая media time.
    let ts = t.timescale.max(1) as u64;
    let mut cum_time: u64 = 0;
    let mut idx: usize = 0;
    'outer: for entry in &t.stts {
        let delta = entry.sample_delta as u64;
        for _ in 0..entry.sample_count {
            if idx >= samples.len() {
                break 'outer;
            }
            let ts_us = cum_time * 1_000_000 / ts;
            let dur_us = (delta * 1_000_000 / ts) as u32;
            samples[idx].timestamp_us = ts_us;
            samples[idx].duration_us = dur_us;
            cum_time += delta;
            idx += 1;
        }
    }

    // Key frames: если stss пуст — все ключевые.
    if t.stss.is_empty() {
        for s in &mut samples {
            s.is_key = true;
        }
    } else {
        for s in &mut samples {
            s.is_key = t.stss.binary_search(&(s.index as u32 + 1)).is_ok();
        }
    }

    return Ok(samples);
}

/// Проверяет, что все sample offset-ы попадают в диапазон `mdat`.
pub fn verify_offsets(
    samples: &[VideoSample],
    mdat_start: u64,
    mdat_end: u64,
) -> Result<(), String> {
    for s in samples {
        if s.offset < mdat_start || s.offset + s.size as u64 > mdat_end {
            return Err(format_str(format_args!(
                "sample #{}: offset {} size {} выходит за mdat ({}..{})",
                s.index, s.offset, s.size, mdat_start, mdat_end,
            )));
        }
    }
    return Ok(());
}

/// Читает `timescale` из payload `mdhd`.
pub fn parse_mdhd_timescale(mdhd: &BoxInfo) -> Option<u32> {
    let p = &mdhd.payload;
    if p.len() < 16 {
        return None;
    }
    let version = p[0];
    let off = if version == 1 { 20 } else { 12 };
    if p.len() < off + 4 {
        return None;
    }
    let ts = read_u32_be(p, off);
    if ts == 0 {
        return None;
    }
    return Some(ts);
}

/// Строит строку codec-а для `VideoDecoder.configure({codec})`.
pub fn avc_codec_string(avcc: &[u8]) -> Option<String> {
    if avcc.len() < 4 {
        return None;
    }
    return Some(format_str(format_args!(
        "avc1.{:02X}{:02X}{:02X}",
        avcc[1],
        avcc[2],
        avcc[3],
    )));
}
```

### Как добавить в `src/lib.rs`

```rust
pub mod app;
pub mod container_parser;
pub mod mp4_demuxer;        // ← 23.10
pub mod video_frame;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

### `tests/wasm_mp4_demuxer.rs` — НОВЫЙ (24 теста)

Полный текст `tests/wasm_mp4_demuxer.rs`:

```rust
#![cfg(target_arch = "wasm32")]

//! Тесты 23.10 — MP4 demuxer.
//!
//! ## Что проверяется
//!
//! - Парсинг пяти таблиц `stbl`: `stco`, `stsc`, `stsz`, `stts`, `stss`.
//! - Преобразование таблиц в плоский список `VideoSample`.
//! - Накопление offset-ов внутри chunk-а и переход между chunk-ами.
//! - Multi-entry `stsc` (chunk-ы разной ёмкости).
//! - Fixed-size `stsz` (`sample_size != 0`).
//! - Multi-entry `stts` (VFR — variable frame rate).
//! - Проверка границ `verify_offsets` против диапазона `mdat`.
//! - Codec-string из `avcC` и timescale из `mdhd` (v0 и v1).
//! - Негативные случаи: усечённые payload-ы, отсутствующие таблицы.
//! - Порядок таблиц внутри `stbl` не важен.

use wasm_bindgen_test::*;
use hello_wasm02::container_parser::{parse_mp4, BoxInfo};
use hello_wasm02::mp4_demuxer::{
    avc_codec_string, build_samples, parse_mdhd_timescale, parse_sample_table,
    verify_offsets,
};

wasm_bindgen_test_configure!(run_in_browser);

// =============================================================================
// Сборка тестовых данных
// =============================================================================

fn mp4_box(box_type: &[u8; 4], body: &[u8]) -> Vec<u8> {
    let size = (8 + body.len()) as u32;
    let mut out = Vec::with_capacity(size as usize);
    out.extend_from_slice(&size.to_be_bytes());
    out.extend_from_slice(box_type);
    out.extend_from_slice(body);
    return out;
}

fn v0_flags() -> Vec<u8> {
    return vec![0u8; 4];
}

fn make_stco(offsets: &[u32]) -> Vec<u8> {
    let mut body = v0_flags();
    body.extend_from_slice(&(offsets.len() as u32).to_be_bytes());
    for &o in offsets {
        body.extend_from_slice(&o.to_be_bytes());
    }
    return body;
}

fn make_stsc(entries: &[(u32, u32)]) -> Vec<u8> {
    let mut body = v0_flags();
    body.extend_from_slice(&(entries.len() as u32).to_be_bytes());
    for &(first_chunk, samples_per_chunk) in entries {
        body.extend_from_slice(&first_chunk.to_be_bytes());
        body.extend_from_slice(&samples_per_chunk.to_be_bytes());
        body.extend_from_slice(&1u32.to_be_bytes());
    }
    return body;
}

fn make_stsz_per_sample(sizes: &[u32]) -> Vec<u8> {
    let mut body = v0_flags();
    body.extend_from_slice(&0u32.to_be_bytes());
    body.extend_from_slice(&(sizes.len() as u32).to_be_bytes());
    for &s in sizes {
        body.extend_from_slice(&s.to_be_bytes());
    }
    return body;
}

fn make_stsz_fixed(sample_size: u32, sample_count: u32) -> Vec<u8> {
    let mut body = v0_flags();
    body.extend_from_slice(&sample_size.to_be_bytes());
    body.extend_from_slice(&sample_count.to_be_bytes());
    return body;
}

fn make_stts(entries: &[(u32, u32)]) -> Vec<u8> {
    let mut body = v0_flags();
    body.extend_from_slice(&(entries.len() as u32).to_be_bytes());
    for &(sample_count, sample_delta) in entries {
        body.extend_from_slice(&sample_count.to_be_bytes());
        body.extend_from_slice(&sample_delta.to_be_bytes());
    }
    return body;
}

fn make_stss(sample_numbers: &[u32]) -> Vec<u8> {
    let mut body = v0_flags();
    body.extend_from_slice(&(sample_numbers.len() as u32).to_be_bytes());
    for &s in sample_numbers {
        body.extend_from_slice(&s.to_be_bytes());
    }
    return body;
}

fn build_stbl_with(
    stco: &[u8], stsc: &[u8], stsz: &[u8], stts: &[u8], stss: Option<&[u8]>,
) -> BoxInfo {
    let mut body = Vec::new();
    body.extend_from_slice(&mp4_box(b"stco", stco));
    body.extend_from_slice(&mp4_box(b"stsc", stsc));
    body.extend_from_slice(&mp4_box(b"stsz", stsz));
    body.extend_from_slice(&mp4_box(b"stts", stts));
    if let Some(stss_body) = stss {
        body.extend_from_slice(&mp4_box(b"stss", stss_body));
    }
    let bytes = mp4_box(b"stbl", &body);
    return parse_mp4(&bytes)
        .expect("stbl parse")
        .into_iter()
        .next()
        .expect("stbl должен быть одним верхнеуровневым box-ом");
}

fn build_stbl(
    stco: &[u32], stsc: &[(u32, u32)], stsz: &[u32],
    stts: &[(u32, u32)], stss: &[u32],
) -> BoxInfo {
    return build_stbl_with(
        &make_stco(stco),
        &make_stsc(stsc),
        &make_stsz_per_sample(stsz),
        &make_stts(stts),
        Some(&make_stss(stss)),
    );
}

fn build_stbl_fixed(
    stco: &[u32], stsc: &[(u32, u32)],
    sample_size: u32, sample_count: u32, stts: &[(u32, u32)],
) -> BoxInfo {
    return build_stbl_with(
        &make_stco(stco),
        &make_stsc(stsc),
        &make_stsz_fixed(sample_size, sample_count),
        &make_stts(stts),
        None,
    );
}

const TS_30K: u32 = 30_000;

// =============================================================================
// Базовые свойства
// =============================================================================

#[wasm_bindgen_test]
fn sample_count_matches_stsz_length() {
    let stbl = build_stbl(&[100, 200, 300], &[(1, 1)], &[10, 20, 30], &[(3, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).expect("parse table");
    assert_eq!(table.total_samples(), 3);
    let samples = build_samples(&table).expect("build samples");
    assert_eq!(samples.len(), 3);
}

// =============================================================================
// Арифметика offset-ов
// =============================================================================

#[wasm_bindgen_test]
fn offsets_accumulate_within_chunk() {
    let stbl = build_stbl(&[1000], &[(1, 4)], &[10, 20, 30, 40], &[(4, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples[0].offset, 1000);
    assert_eq!(samples[1].offset, 1010);
    assert_eq!(samples[2].offset, 1030);
    assert_eq!(samples[3].offset, 1060);
}

#[wasm_bindgen_test]
fn offsets_reset_between_chunks() {
    let stbl = build_stbl(&[1000, 5000], &[(1, 2)], &[10, 20, 30, 40], &[(4, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples[0].offset, 1000);
    assert_eq!(samples[1].offset, 1010);
    assert_eq!(samples[2].offset, 5000);
    assert_eq!(samples[3].offset, 5030);
}

#[wasm_bindgen_test]
fn multi_entry_stsc_supports_variable_chunk_capacity() {
    let sizes = [10u32; 10];
    let stbl = build_stbl(
        &[100, 200, 300, 400],
        &[(1, 4), (3, 1)],
        &sizes,
        &[(10, 1000)],
        &[],
    );
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    assert_eq!(table.total_samples(), 10);
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples[0].offset, 100);
    assert_eq!(samples[3].offset, 130);
    assert_eq!(samples[4].offset, 200);
    assert_eq!(samples[7].offset, 230);
    assert_eq!(samples[8].offset, 300);
    assert_eq!(samples[9].offset, 400);
}

// =============================================================================
// Fixed-size stsz
// =============================================================================

#[wasm_bindgen_test]
fn fixed_size_stsz_uses_constant_sample_size() {
    let stbl = build_stbl_fixed(&[1000, 2000], &[(1, 2)], 100, 4, &[(4, 1000)]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    assert_eq!(table.stsz_fixed, Some(100));
    assert!(table.stsz.is_empty());
    assert_eq!(table.total_samples(), 4);
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples.len(), 4);
    for s in &samples {
        assert_eq!(s.size, 100);
    }
    assert_eq!(samples[0].offset, 1000);
    assert_eq!(samples[1].offset, 1100);
    assert_eq!(samples[2].offset, 2000);
    assert_eq!(samples[3].offset, 2100);
}

// =============================================================================
// Key frames
// =============================================================================

#[wasm_bindgen_test]
fn stss_marks_only_listed_samples_as_key() {
    let stbl = build_stbl(
        &[0, 100, 200, 300, 400, 500],
        &[(1, 1)],
        &[10; 6],
        &[(6, 1000)],
        &[1, 5],
    );
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples.len(), 6);
    assert!(samples[0].is_key);
    assert!(!samples[1].is_key);
    assert!(!samples[2].is_key);
    assert!(!samples[3].is_key);
    assert!(samples[4].is_key);
    assert!(!samples[5].is_key);
}

#[wasm_bindgen_test]
fn empty_stss_means_all_frames_are_key() {
    let stbl = build_stbl(&[0, 100, 200], &[(1, 1)], &[10, 20, 30], &[(3, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert!(samples.iter().all(|s| s.is_key));
}

// =============================================================================
// Timestamps и durations
// =============================================================================

#[wasm_bindgen_test]
fn timestamps_accumulate_with_constant_delta() {
    let stbl = build_stbl(&[0, 100, 200], &[(1, 1)], &[10, 20, 30], &[(3, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples[0].timestamp_us, 0);
    assert_eq!(samples[0].duration_us, 33_333);
    assert_eq!(samples[1].timestamp_us, 33_333);
    assert_eq!(samples[2].timestamp_us, 66_666);
}

#[wasm_bindgen_test]
fn multi_entry_stts_supports_vfr() {
    let stbl = build_stbl(&[0, 100, 200, 300], &[(1, 1)], &[10; 4], &[(2, 1000), (2, 1500)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples[0].duration_us, 33_333);
    assert_eq!(samples[1].duration_us, 33_333);
    assert_eq!(samples[2].duration_us, 50_000);
    assert_eq!(samples[3].duration_us, 50_000);
    assert_eq!(samples[0].timestamp_us, 0);
    assert_eq!(samples[1].timestamp_us, 33_333);
    assert_eq!(samples[2].timestamp_us, 66_666);
    assert_eq!(samples[3].timestamp_us, 116_666);
}

#[wasm_bindgen_test]
fn timestamps_scale_with_timescale() {
    let stbl = build_stbl(&[0, 100], &[(1, 1)], &[10, 10], &[(2, 1000)], &[]);
    let table = parse_sample_table(&stbl, 1000).unwrap();
    let samples = build_samples(&table).unwrap();
    assert_eq!(samples[0].duration_us, 1_000_000);
    assert_eq!(samples[1].timestamp_us, 1_000_000);
}

// =============================================================================
// verify_offsets
// =============================================================================

#[wasm_bindgen_test]
fn verify_offsets_accepts_valid_range() {
    let stbl = build_stbl(&[1000, 2000], &[(1, 2)], &[10, 20, 30, 40], &[(4, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    assert!(verify_offsets(&samples, 1000, 2070).is_ok());
}

#[wasm_bindgen_test]
fn verify_offsets_rejects_sample_after_mdat_end() {
    let stbl = build_stbl(&[1000, 2000], &[(1, 2)], &[10, 20, 30, 40], &[(4, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    let result = verify_offsets(&samples, 1000, 1500);
    assert!(result.is_err());
    let msg = result.unwrap_err();
    assert!(msg.contains("sample #2"), "получено: {msg}");
}

#[wasm_bindgen_test]
fn verify_offsets_rejects_sample_before_mdat_start() {
    let stbl = build_stbl(&[500, 2000], &[(1, 2)], &[10, 20, 30, 40], &[(4, 1000)], &[]);
    let table = parse_sample_table(&stbl, TS_30K).unwrap();
    let samples = build_samples(&table).unwrap();
    let result = verify_offsets(&samples, 1000, 3000);
    assert!(result.is_err());
    let msg = result.unwrap_err();
    assert!(msg.contains("sample #0"), "получено: {msg}");
}

// =============================================================================
// Codec string
// =============================================================================

#[wasm_bindgen_test]
fn codec_string_from_avcc_baseline() {
    let avcc = &[
        0x01, 0x42, 0x00, 0x1E, 0xFF, 0xE1,
        0x00, 0x04, 0x67, 0x42, 0x00, 0x1E, 0x01,
        0x00, 0x02, 0x68, 0xCE,
    ];
    assert_eq!(avc_codec_string(avcc).as_deref(), Some("avc1.42001E"));
}

#[wasm_bindgen_test]
fn codec_string_from_avcc_high_profile() {
    let avcc = &[0x01, 0x64, 0x00, 0x28, 0xFF, 0xE1];
    assert_eq!(avc_codec_string(avcc).as_deref(), Some("avc1.640028"));
}

#[wasm_bindgen_test]
fn codec_string_rejects_too_short_avcc() {
    assert!(avc_codec_string(&[0x01, 0x42]).is_none());
}

// =============================================================================
// mdhd timescale
// =============================================================================

fn make_mdhd_v0(timescale: u32) -> BoxInfo {
    let mut payload = vec![0u8; 12];
    payload.extend_from_slice(&timescale.to_be_bytes());
    payload.extend_from_slice(&0u32.to_be_bytes());
    return BoxInfo {
        box_type: String::from("mdhd"),
        offset: 0,
        size: 8 + payload.len() as u64,
        header_size: 8,
        children: Vec::new(),
        payload,
    };
}

fn make_mdhd_v1(timescale: u32) -> BoxInfo {
    let mut payload = vec![1u8, 0, 0, 0];
    payload.extend_from_slice(&[0u8; 8]);
    payload.extend_from_slice(&[0u8; 8]);
    payload.extend_from_slice(&timescale.to_be_bytes());
    payload.extend_from_slice(&[0u8; 8]);
    return BoxInfo {
        box_type: String::from("mdhd"),
        offset: 0,
        size: 8 + payload.len() as u64,
        header_size: 8,
        children: Vec::new(),
        payload,
    };
}

#[wasm_bindgen_test]
fn mdhd_timescale_v0_reads_offset_12() {
    assert_eq!(parse_mdhd_timescale(&make_mdhd_v0(30_000)), Some(30_000));
}

#[wasm_bindgen_test]
fn mdhd_timescale_v1_reads_offset_20() {
    assert_eq!(parse_mdhd_timescale(&make_mdhd_v1(90_000)), Some(90_000));
}

#[wasm_bindgen_test]
fn mdhd_rejects_zero_timescale() {
    assert!(parse_mdhd_timescale(&make_mdhd_v0(0)).is_none());
}

#[wasm_bindgen_test]
fn mdhd_rejects_truncated_payload() {
    let mdhd = BoxInfo {
        box_type: String::from("mdhd"),
        offset: 0,
        size: 16,
        header_size: 8,
        children: Vec::new(),
        payload: vec![0u8; 8],
    };
    assert!(parse_mdhd_timescale(&mdhd).is_none());
}

// =============================================================================
// Негативные случаи
// =============================================================================

#[wasm_bindgen_test]
fn missing_stco_returns_error() {
    let body = mp4_box(b"stsc", &make_stsc(&[(1, 1)]));
    let bytes = mp4_box(b"stbl", &body);
    let stbl = parse_mp4(&bytes)
        .expect("stbl parse")
        .into_iter()
        .next()
        .expect("one box");
    let result = parse_sample_table(&stbl, TS_30K);
    assert!(result.is_err());
    assert!(result.unwrap_err().contains("stco"));
}

#[wasm_bindgen_test]
fn truncated_stsz_returns_error() {
    let mut stsz_body = v0_flags();
    stsz_body.extend_from_slice(&0u32.to_be_bytes());
    stsz_body.extend_from_slice(&5u32.to_be_bytes());
    let stbl = build_stbl_with(
        &make_stco(&[0, 100]),
        &make_stsc(&[(1, 1)]),
        &stsz_body,
        &make_stts(&[(5, 1000)]),
        None,
    );
    let result = parse_sample_table(&stbl, TS_30K);
    assert!(result.is_err());
    assert!(result.unwrap_err().contains("stsz"));
}

#[wasm_bindgen_test]
fn truncated_stco_returns_error() {
    let mut stco_body = v0_flags();
    stco_body.extend_from_slice(&3u32.to_be_bytes());
    stco_body.extend_from_slice(&100u32.to_be_bytes());
    let stbl = build_stbl_with(
        &stco_body,
        &make_stsc(&[(1, 1)]),
        &make_stsz_per_sample(&[10, 20, 30]),
        &make_stts(&[(3, 1000)]),
        None,
    );
    let result = parse_sample_table(&stbl, TS_30K);
    assert!(result.is_err());
    assert!(result.unwrap_err().contains("stco"));
}

// =============================================================================
// Порядок таблиц внутри stbl
// =============================================================================

#[wasm_bindgen_test]
fn table_order_inside_stbl_does_not_matter() {
    let mut body = Vec::new();
    body.extend_from_slice(&mp4_box(b"stss", &make_stss(&[1])));
    body.extend_from_slice(&mp4_box(b"stts", &make_stts(&[(2, 1000)])));
    body.extend_from_slice(&mp4_box(b"stsz", &make_stsz_per_sample(&[10, 20])));
    body.extend_from_slice(&mp4_box(b"stsc", &make_stsc(&[(1, 1)])));
    body.extend_from_slice(&mp4_box(b"stco", &make_stco(&[100, 200])));
    let bytes = mp4_box(b"stbl", &body);
    let stbl = parse_mp4(&bytes)
        .expect("stbl parse")
        .into_iter()
        .next()
        .expect("one box");
    let table = parse_sample_table(&stbl, TS_30K).expect("parse table");
    let samples = build_samples(&table).expect("build samples");
    assert_eq!(samples.len(), 2);
    assert_eq!(samples[0].offset, 100);
    assert_eq!(samples[1].offset, 200);
    assert!(samples[0].is_key);
    assert!(!samples[1].is_key);
}
```

### `src/app.rs` — полный текст

Полный текст `src/app.rs` (около 1000 строк).

```
//! Глава 23.10 — Demuxing: MP4 → EncodedVideoChunk.
//!
//! Демо: загрузка MP4, извлечение таблиц `stbl`, построение
//! плоского списка `VideoSample` — прямой аналог `EncodedVideoChunk`.
//!
//! ## Что демонстрируется
//!
//! - `stco` + `stsc` + `stsz` → offset и size каждого chunk-а.
//! - `stts` → timestamp и duration.
//! - `stss` → key frame.
//! - `avcC` → `description` (SPS/PPS) и codec-string `avc1.PPCCLL`.
//! - **NAL-разбор**: содержимое первых samples — SPS/PPS/IDR/delta.
//! - **Карта key frames** — GOP-структура на одном экране.
//! - Проверка через `VideoDecoder.isConfigSupported({codec, description})`.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::{Object, Reflect, Uint8Array};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, File, FileReader, HtmlElement, HtmlInputElement};

use crate::container_parser::{find_avcc_in_stsd, format_bytes, hex_dump, parse_mp4, BoxInfo};
use crate::mp4_demuxer::{
    avc_codec_string, build_samples, parse_mdhd_timescale, parse_sample_table,
    verify_offsets, VideoSample,
};
use crate::video_frame::{format_str, set_prop};

// =============================================================================
// Константы
// =============================================================================

const MAX_SAMPLES_PREVIEW: usize = 30;
const MAX_DESCRIPTION_HEX: usize = 64;
const MAX_NAME_CHARS: usize = 50;
const MAX_NALS_PER_SAMPLE: usize = 12;
const NAL_PREVIEW_BYTES: usize = 32;
/// Порог: до этого числа samples в keymap — один символ на sample.
/// Выше — группируем по `KEYMAP_GROUP_SIZE` samples в один символ.
const KEYMAP_SINGLE_LIMIT: usize = 200;
/// Размер группы символа keymap, если samples больше лимита.
const KEYMAP_GROUP_SIZE: usize = 10;
/// Максимум групп в keymap (защита от гигантских файлов).
const KEYMAP_MAX_GROUPS: usize = 500;
/// Длина строки в keymap (символов). Раз в 80 символов — перенос.
const KEYMAP_ROW_LENGTH: usize = 80;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.dm-desc { max-width:900px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.dm-desc-line { margin:6px 0; }
.dm-block { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.dm-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.dm-btn:hover:not([disabled]) { background:#30363d; }
.dm-btn:disabled { opacity:0.5;cursor:not-allowed; }
.dm-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.dm-btn-primary:hover:not([disabled]) { background:#2ea043; }
.dm-btn-check { background:#8957e5;border-color:#a371f7;color:#fff;}
.dm-btn-check:hover:not([disabled]) { background:#a371f7; }
.dm-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}

.dm-summary { max-width:900px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;overflow-wrap:anywhere;}
.dm-summary .row { display:flex;gap:12px; }
.dm-summary .lbl { color:#8b949e;min-width:130px;flex-shrink:0; }
.dm-summary .val { color:#79c0ff; }
.dm-summary .ok { color:#3fb950; }
.dm-summary .warn { color:#f0883e; }
.dm-summary .err { color:#f85149; }
.dm-summary .muted { color:#6e7681; }

.dm-hex { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:12px;color:#79c0ff;white-space:pre-wrap;overflow-wrap:anywhere;}

.dm-keymap { max-width:900px;margin-bottom:16px;padding:10px 14px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:14px;line-height:1.5;color:#8b949e;overflow-x:auto;}
.dm-keymap .k { color:#3fb950; font-weight:700; }
.dm-keymap .x { color:#f0883e; font-weight:700; }
.dm-keymap .d { color:#484f58; }
.dm-keymap .legend { color:#6e7681; margin-top:6px; font-size:11px; }

.dm-nals { max-width:900px;margin-bottom:16px;padding:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:12px;color:#c9d1d9;}
.dm-nals .chunk { padding:10px 14px; border-bottom:1px solid #21262d; }
.dm-nals .chunk:last-child { border-bottom:none; }
.dm-nals .chunk-title { color:#8b949e; margin-bottom:6px; }
.dm-nals .chunk-title .kind { font-weight:700; }
.dm-nals .chunk-title .kind-key { color:#3fb950; }
.dm-nals .chunk-title .kind-delta { color:#8b949e; }
.dm-nals .chunk-title .num { color:#79c0ff; }
.dm-nals .nal { padding:2px 0; }
.dm-nals .nal .idx { color:#6e7681; }
.dm-nals .nal .len { color:#79c0ff; }
.dm-nals .nal .type-id { color:#6e7681; }
.dm-nals .nal .type-name { color:#a371f7; }
.dm-nals .nal .type-sps { color:#f0883e; font-weight:700; }
.dm-nals .nal .type-pps { color:#f0883e; font-weight:700; }
.dm-nals .nal .type-idr { color:#3fb950; font-weight:700; }
.dm-nals .nal .type-nonidr { color:#8b949e; }
.dm-nals .hex { margin-top:6px; color:#6e7681; }
.dm-nals .no-nals { color:#f0883e; padding:10px 14px; }

.dm-table { max-width:900px;margin-bottom:16px;padding:0;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;color:#c9d1d9;overflow:hidden;}
.dm-table table { width:100%; border-collapse:collapse; }
.dm-table th { text-align:left; color:#8b949e; font-weight:500; padding:8px 12px; border-bottom:1px solid #21262d; }
.dm-table td { padding:6px 12px; border-bottom:1px solid #161b22; }
.dm-table tr:last-child td { border-bottom:none; }
.dm-table .key { color:#3fb950; font-weight:600; }
.dm-table .delta { color:#8b949e; }
.dm-table .idx { color:#6e7681; }
.dm-table .num { color:#79c0ff; }
.dm-table .ts { color:#a371f7; }
.dm-table .more { text-align:center; color:#6e7681; font-style:italic; padding:10px; }

.dm-stats { max-width:900px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;overflow-wrap:anywhere;word-break:break-word;}
.dm-stats .lbl { color:#8b949e; }
.dm-stats .val { color:#79c0ff; }
.dm-stats .err { color:#f85149; }
.dm-stats .muted { color:#6e7681; }
.dm-stats .ok { color:#3fb950; }

.dm-footnote { max-width:900px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    phase: String,
    error: Option<String>,
    file_name: String,
    file_size: usize,
    summary_html: String,
    description_hex: String,
    keymap_html: String,
    nals_html: String,
    samples_html: String,
    codec_string: Option<String>,
    avcc_bytes: Vec<u8>,
    config_supported: Option<Result<String, String>>,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            phase: String::from("простой"),
            error: None,
            file_name: String::new(),
            file_size: 0,
            summary_html: String::new(),
            description_hex: String::new(),
            keymap_html: String::new(),
            nals_html: String::new(),
            samples_html: String::new(),
            codec_string: None,
            avcc_bytes: Vec::new(),
            config_supported: None,
        };
    }

    fn clear(&mut self) {
        self.summary_html.clear();
        self.description_hex.clear();
        self.keymap_html.clear();
        self.nals_html.clear();
        self.samples_html.clear();
        self.codec_string = None;
        self.avcc_bytes.clear();
        self.config_supported = None;
    }

    fn can_check_config(&self) -> bool {
        return self.codec_string.is_some() && !self.avcc_bytes.is_empty();
    }
}

// =============================================================================
// Хелперы имён и строк
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

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "dm-injected-style";
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
    return Ok(());
}

fn build_description_block(document: &Document) -> Result<web_sys::Node, JsValue> {
    let lines: &[&str] = &[
        "<strong>stbl</strong> — пять таблиц. <code>stco</code> — offset chunk-ов, <code>stsc</code> — сколько сэмплов в каждом, <code>stsz</code> — размеры, <code>stts</code> — длительности, <code>stss</code> — key frames.",
        "<strong>EncodedVideoChunk ≈ sample.</strong> Offset, size, timestamp, is_key — четыре поля, которые превращают <code>(mdat, таблицы)</code> в один вызов <code>VideoDecoder.decode(chunk)</code>.",
        "<strong>NAL units.</strong> Внутри H.264 sample-а — length-prefixed NAL: <code>SPS</code> (type 7), <code>PPS</code> (type 8), <code>IDR slice</code> (type 5), <code>non-IDR slice</code> (type 1). Это то, что видит декодер.",
        "<strong>description = avcC</strong> (SPS/PPS). Codec-string <code>avc1.PPCCLL</code> — три байта из того же avcC.",
    ];
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("dm-desc");
    for line in lines {
        let p = document.create_element("p")?;
        p.set_class_name("dm-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    return Ok(block.unchecked_into());
}

// =============================================================================
// Разбор NAL-юнитов (AVCC — length-prefixed)
// =============================================================================

#[derive(Debug, Clone, Copy)]
struct NalUnit {
    index: usize,
    length: u32,
    nal_type: u8,
}

fn nal_type_name(nal_type: u8) -> &'static str {
    match nal_type {
        1 => "non-IDR slice",
        2 | 3 | 4 => "slice partition",
        5 => "IDR slice",
        6 => "SEI",
        7 => "SPS",
        8 => "PPS",
        9 => "AUD",
        10 => "end of seq",
        11 => "end of stream",
        12 => "filler",
        _ => "?",
    }
}

fn nal_type_css(nal_type: u8) -> &'static str {
    match nal_type {
        5 => "type-idr",
        7 => "type-sps",
        8 => "type-pps",
        1 => "type-nonidr",
        _ => "type-name",
    }
}

/// Разбирает AVCC NAL units внутри sample-а.
///
/// Формат: `[len: nal_length_size][nal][len: nal_length_size][nal]...`
/// `nal_length_size` обычно 4 (из `avcC[4] & 0x03 + 1`).
fn parse_avcc_nals(sample_bytes: &[u8], nal_length_size: usize) -> Vec<NalUnit> {
    let mut result = Vec::new();
    if nal_length_size == 0 || nal_length_size > 4 {
        return result;
    }
    let mut pos = 0usize;
    let mut idx = 0usize;
    while pos + nal_length_size <= sample_bytes.len() && idx < MAX_NALS_PER_SAMPLE {
        let mut len: u32 = 0;
        for i in 0..nal_length_size {
            len = (len << 8) | sample_bytes[pos + i] as u32;
        }
        if len == 0 || pos + nal_length_size + len as usize > sample_bytes.len() {
            break;
        }
        let nal_byte = sample_bytes[pos + nal_length_size];
        let nal_type = nal_byte & 0x1F;
        result.push(NalUnit {
            index: idx,
            length: len,
            nal_type,
        });
        pos += nal_length_size + len as usize;
        idx += 1;
    }
    return result;
}

// =============================================================================
// Поиск video trak в moov
// =============================================================================

fn extract_hdlr_type(hdlr: &BoxInfo) -> &'static str {
    let p = &hdlr.payload;
    if p.len() < 12 {
        return "other";
    }
    match &p[8..12] {
        b"vide" => "video",
        b"soun" => "audio",
        _ => "other",
    }
}

fn find_video_trak<'a>(moov: &'a BoxInfo) -> Option<&'a BoxInfo> {
    for trak in &moov.children {
        if trak.box_type != "trak" {
            continue;
        }
        if let Some(hdlr) = trak.find("hdlr") {
            if extract_hdlr_type(hdlr) == "video" {
                return Some(trak);
            }
        }
    }
    return None;
}

fn find_mdat_range(boxes: &[BoxInfo]) -> Option<(u64, u64)> {
    for b in boxes {
        if b.box_type == "mdat" {
            let start = b.offset as u64;
            return Some((start, start + b.size));
        }
    }
    return None;
}

fn extract_sample_bytes(bytes: &[u8], sample: &VideoSample) -> Option<Vec<u8>> {
    let start = sample.offset as usize;
    let end = start.checked_add(sample.size as usize)?;
    if end <= bytes.len() {
        return Some(bytes[start..end].to_vec());
    }
    return None;
}

// =============================================================================
// Построение UI-панелей
// =============================================================================

fn build_summary(
    samples: &[VideoSample],
    num_key: usize,
    timescale: u32,
    total_duration_us: u64,
    description_len: usize,
    codec_string: Option<&str>,
    offsets_ok: bool,
    mdat_range: Option<(u64, u64)>,
    nal_length_size: Option<usize>,
) -> String {
    let mut out = String::new();

    out.push_str(&row(
        "Samples",
        &format_str(format_args!(
            r#"<span class="num">{}</span>"#,
            samples.len(),
        )),
    ));

    out.push_str(&row(
        "Key frames",
        &format_str(format_args!(
            r#"<span class="ok">{num_key}</span> <span class="muted">({:.1}%)</span>"#,
            if samples.is_empty() {
                0.0
            } else {
                num_key as f64 / samples.len() as f64 * 100.0
            },
        )),
    ));

    out.push_str(&row(
        "Timescale",
        &format_str(format_args!("{timescale}")),
    ));

    out.push_str(&row(
        "Duration",
        &format_str(format_args!(
            r#"<span class="num">{:.2} s</span>"#,
            total_duration_us as f64 / 1_000_000.0,
        )),
    ));

    // Средний FPS и средний bitrate видео-дорожки.
    if !samples.is_empty() && total_duration_us > 0 {
        let avg_dur_us = total_duration_us as f64 / samples.len() as f64;
        let fps = if avg_dur_us > 0.0 {
            1_000_000.0 / avg_dur_us
        } else {
            0.0
        };
        out.push_str(&row(
            "Frame rate",
            &format_str(format_args!(
                r#"<span class="num">{fps:.2}</span> <span class="muted">fps (avg)</span>"#
            )),
        ));

        let total_bytes: u64 = samples.iter().map(|s| s.size as u64).sum();
        let duration_s = total_duration_us as f64 / 1_000_000.0;
        let mbps = (total_bytes as f64 * 8.0) / duration_s / 1_000_000.0;
        out.push_str(&row(
            "Bitrate",
            &format_str(format_args!(
                r#"<span class="num">{mbps:.2}</span> <span class="muted">Mbps (video)</span>"#
            )),
        ));
    }

    if let Some(cs) = codec_string {
        out.push_str(&row("Codec", cs));
    }

    if let Some(ls) = nal_length_size {
        out.push_str(&row(
            "NAL format",
            &format_str(format_args!(
                r#"AVCC <span class="muted">(length-prefix = {ls} B)</span>"#
            )),
        ));
    }

    out.push_str(&row(
        "Description",
        &format_str(format_args!(
            r#"<span class="num">{description_len} B</span>"#
        )),
    ));

    if let Some((start, end)) = mdat_range {
        out.push_str(&row(
            "mdat range",
            &format_str(format_args!(r#"<span class="muted">{start}..{end}</span>"#)),
        ));
    }

    let (cls, txt) = if offsets_ok {
        ("ok", "✓ все offset-ы внутри mdat")
    } else {
        ("err", "⚠ есть offset-ы вне mdat")
    };
    out.push_str(&format_str(format_args!(
        r#"<div class="row"><span class="lbl">Offsets:</span><span class="{cls}">{txt}</span></div>"#,
        cls = cls,
        txt = txt,
    )));

    return out;
}

/// Строит визуальную карту key frames.
///
/// До `KEYMAP_SINGLE_LIMIT` samples — по одному символу на sample.
/// Больше — группируем по `KEYMAP_GROUP_SIZE`: символ показывает,
/// есть ли в группе key frames. Раз в `KEYMAP_ROW_LENGTH` символов —
/// жёсткий перенос строки, чтобы карта читалась как таблица.
fn build_keyframe_map(samples: &[VideoSample]) -> String {
    if samples.is_empty() {
        return String::new();
    }

    let n = samples.len();
    let group_size = if n <= KEYMAP_SINGLE_LIMIT {
        1usize
    } else {
        KEYMAP_GROUP_SIZE
    };
    let total_groups = (n + group_size - 1) / group_size;
    let show_groups = total_groups.min(KEYMAP_MAX_GROUPS);

    let mut rows_html = String::with_capacity(show_groups * 5);
    let mut chars_in_row = 0usize;

    for g in 0..show_groups {
        let start = g * group_size;
        let end = ((g + 1) * group_size).min(n);
        let total = end - start;

        let mut keys = 0usize;
        for s in &samples[start..end] {
            if s.is_key {
                keys += 1;
            }
        }

        // Перенос строки раз в KEYMAP_ROW_LENGTH символов.
        if chars_in_row >= KEYMAP_ROW_LENGTH {
            rows_html.push_str("<br>");
            chars_in_row = 0;
        }

        let ch_html = if keys == 0 {
            r#"<span class="d">·</span>"#
        } else if keys == total {
            r#"<span class="x">X</span>"#
        } else {
            r#"<span class="k">K</span>"#
        };
        rows_html.push_str(ch_html);
        chars_in_row += 1;
    }

    let mut html = String::from(r#"<div class="dm-keymap">"#);
    html.push_str(&rows_html);

    if total_groups > show_groups {
        html.push_str(&format_str(format_args!(
            r#"<br><span class="d">… показано {}/{}</span>"#,
            show_groups * group_size,
            n,
        )));
    }

    html.push_str(&format_str(format_args!(
        r#"<div class="legend">1 символ = {} sample(ов) · X = все key · K = есть key · · = все delta</div>"#,
        group_size,
    )));
    html.push_str("</div>");
    return html;
}

fn render_chunk_block(
    sample: &VideoSample,
    nals: &[NalUnit],
    kind: &str,
    hex_preview: &str,
) -> String {
    let (kind_cls, kind_text) = if kind == "key" {
        ("kind-key", "key")
    } else {
        ("kind-delta", "delta")
    };

    // Счётчик NAL: показывает, один ли NAL в sample (типично для
    // AVCC с SPS/PPS в description) или несколько.
    let nals_count_html = if nals.is_empty() {
        String::from(r#" · <span class="num">NAL: —</span>"#)
    } else {
        format_str(format_args!(
            r#" · <span class="num">NAL: {n}</span>"#,
            n = nals.len(),
        ))
    };

    let mut html = format_str(format_args!(
        r#"<div class="chunk">
             <div class="chunk-title">
               Chunk #{idx} <span class="{kcls}">{kind}</span>
               · offset=<span class="num">{off}</span>
               · size=<span class="num">{sz}</span> B
               · ts=<span class="num">{ts:.2}</span> мс
               {nals_count}
             </div>"#,
        idx = sample.index,
        kcls = kind_cls,
        kind = kind_text,
        off = sample.offset,
        sz = sample.size,
        ts = sample.timestamp_us as f64 / 1000.0,
        nals_count = nals_count_html,
    ));

    if nals.is_empty() {
        html.push_str(
            r#"<div class="no-nals">NAL не разобраны (не H.264 или пустой sample)</div>"#,
        );
    } else {
        for n in nals {
            html.push_str(&format_str(format_args!(
                r#"<div class="nal">
                     <span class="idx">NAL #{idx}</span>
                     <span class="type-id">type=</span><span class="{tcls}">{tname}</span>
                     <span class="len">len={len}</span>
                   </div>"#,
                idx = n.index,
                tcls = nal_type_css(n.nal_type),
                tname = nal_type_name(n.nal_type),
                len = n.length,
            )));
        }
    }

    if !hex_preview.is_empty() {
        html.push_str(&format_str(format_args!(
            r#"<div class="hex">hex[0..{n}]: {h}</div>"#,
            n = NAL_PREVIEW_BYTES,
            h = hex_preview,
        )));
    }

    html.push_str("</div>");
    return html;
}

fn build_nals_panel(
    nals_key: &[NalUnit],
    nals_delta: &[NalUnit],
    sample_key: Option<&VideoSample>,
    sample_delta: Option<&VideoSample>,
    key_hex: &str,
    delta_hex: &str,
) -> String {
    if sample_key.is_none() && sample_delta.is_none() {
        return String::new();
    }
    let mut html = String::from(r#"<div class="dm-nals">"#);
    if let Some(s) = sample_key {
        html.push_str(&render_chunk_block(s, nals_key, "key", key_hex));
    }
    if let Some(s) = sample_delta {
        html.push_str(&render_chunk_block(s, nals_delta, "delta", delta_hex));
    }
    html.push_str("</div>");
    return html;
}

fn build_samples_table(samples: &[VideoSample]) -> String {
    if samples.is_empty() {
        return String::from(r#"<div class="more">нет samples</div>"#);
    }
    let n = samples.len().min(MAX_SAMPLES_PREVIEW);
    let mut html = String::from(
        r#"<div class="dm-table"><table>
           <thead><tr>
             <th style="width:60px">#</th>
             <th style="width:120px">offset</th>
             <th style="width:100px">size</th>
             <th style="width:110px">timestamp</th>
             <th style="width:100px">duration</th>
             <th>kind</th>
           </tr></thead><tbody>"#,
    );
    for s in &samples[..n] {
        let (cls, kind) = if s.is_key {
            ("key", "key")
        } else {
            ("delta", "delta")
        };
        html.push_str(&format_str(format_args!(
            r#"<tr>
                 <td class="idx">{idx}</td>
                 <td class="num">{off}</td>
                 <td class="num">{sz}</td>
                 <td class="ts">{ts} мс</td>
                 <td class="ts">{dur} мс</td>
                 <td class="{cls}">{kind}</td>
               </tr>"#,
            idx = s.index,
            off = s.offset,
            sz = s.size,
            ts = format_str(format_args!("{:.2}", s.timestamp_us as f64 / 1000.0)),
            dur = format_str(format_args!("{:.2}", s.duration_us as f64 / 1000.0)),
            cls = cls,
            kind = kind,
        )));
    }
    if samples.len() > n {
        html.push_str(&format_str(format_args!(
            r#"<tr><td colspan="6" class="more">… всего {} samples, показаны первые {}</td></tr>"#,
            samples.len(),
            n,
        )));
    }
    html.push_str("</tbody></table></div>");
    return html;
}

fn set_error(
    doc: &Document,
    st: Rc<RefCell<DemoState>>,
    name: String,
    size: usize,
    error: String,
) {
    let mut s = st.borrow_mut();
    s.clear();
    s.phase = String::from("ошибка");
    s.error = Some(error);
    s.file_name = name;
    s.file_size = size;
    drop(s);
    update_ui(doc, &st.borrow());
}

// =============================================================================
// Обработка файла (async, с yield-точками)
// =============================================================================

async fn process_bytes(
    doc: &Document,
    st: Rc<RefCell<DemoState>>,
    name: String,
    bytes: Vec<u8>,
) {
    let is_mp4 = bytes.len() >= 8 && &bytes[4..8] == b"ftyp";
    if !is_mp4 {
        set_error(
            doc,
            st,
            name,
            bytes.len(),
            String::from("не MP4: сигнатура ftyp не найдена"),
        );
        return;
    }

    // 1. Парсим структуру.
    let boxes = match parse_mp4(&bytes) {
        Ok(b) => b,
        Err(e) => {
            set_error(doc, st, name, bytes.len(), e);
            return;
        }
    };
    TimeoutFuture::new(0).await;

    // 2. moov + video-trak + mdhd + stbl.
    let Some(moov) = boxes.iter().find(|b| b.box_type == "moov") else {
        set_error(
            doc,
            st,
            name,
            bytes.len(),
            String::from("нет moov — возможно, fragmented MP4"),
        );
        return;
    };
    let Some(video_trak) = find_video_trak(moov) else {
        set_error(
            doc,
            st,
            name,
            bytes.len(),
            String::from("нет video-трека в moov"),
        );
        return;
    };
    let Some(mdhd) = video_trak.find("mdhd") else {
        set_error(doc, st, name, bytes.len(), String::from("нет mdhd"));
        return;
    };
    let Some(timescale) = parse_mdhd_timescale(mdhd) else {
        set_error(doc, st, name, bytes.len(), String::from("mdhd.timescale = 0"));
        return;
    };
    let Some(stbl) = video_trak.find("stbl") else {
        set_error(doc, st, name, bytes.len(), String::from("нет stbl"));
        return;
    };

    // 3. Sample table + samples.
    let table = match parse_sample_table(stbl, timescale) {
        Ok(t) => t,
        Err(e) => {
            set_error(doc, st, name, bytes.len(), e);
            return;
        }
    };
    let samples = match build_samples(&table) {
        Ok(s) => s,
        Err(e) => {
            set_error(doc, st, name, bytes.len(), e);
            return;
        }
    };
    TimeoutFuture::new(0).await;

    // 4. Проверка: offset-ы внутри mdat.
    let mdat_range = find_mdat_range(&boxes);
    let offsets_ok = mdat_range
        .map(|(s, e)| verify_offsets(&samples, s, e).is_ok())
        .unwrap_or(false);

    // 5. Codec-string и description.
    let avcc_bytes = video_trak
        .find("stsd")
        .and_then(find_avcc_in_stsd)
        .unwrap_or_default();
    let codec_string = avc_codec_string(&avcc_bytes);

    // 6. NAL length-size: avcC[4] & 0x03 + 1.
    let nal_length_size = if avcc_bytes.len() >= 5 {
        Some(((avcc_bytes[4] & 0x03) as usize) + 1)
    } else {
        None
    };

    // 7. Извлекаем первые key и delta samples для NAL-разбора.
    let sample_key = samples.iter().find(|s| s.is_key).cloned();
    let sample_delta = samples.iter().find(|s| !s.is_key).cloned();

    let key_bytes = sample_key
        .as_ref()
        .and_then(|s| extract_sample_bytes(&bytes, s));
    let delta_bytes = sample_delta
        .as_ref()
        .and_then(|s| extract_sample_bytes(&bytes, s));

    // NAL-разбор только для H.264 (avc1.*).
    let parse_nals = codec_string
        .as_deref()
        .map(|c| c.starts_with("avc1"))
        .unwrap_or(false);
    let nls = nal_length_size.unwrap_or(4);

    let nals_key = if parse_nals {
        key_bytes
            .as_deref()
            .map(|b| parse_avcc_nals(b, nls))
            .unwrap_or_default()
    } else {
        Vec::new()
    };
    let nals_delta = if parse_nals {
        delta_bytes
            .as_deref()
            .map(|b| parse_avcc_nals(b, nls))
            .unwrap_or_default()
    } else {
        Vec::new()
    };

    let key_hex = key_bytes
        .as_deref()
        .map(|b| hex_dump(b, NAL_PREVIEW_BYTES))
        .unwrap_or_default();
    let delta_hex = delta_bytes
        .as_deref()
        .map(|b| hex_dump(b, NAL_PREVIEW_BYTES))
        .unwrap_or_default();

    TimeoutFuture::new(0).await;

    // 8. UI.
    let total_duration_us: u64 = samples
        .last()
        .map(|s| s.timestamp_us + s.duration_us as u64)
        .unwrap_or(0);
    let num_key = samples.iter().filter(|s| s.is_key).count();

    let summary = build_summary(
        &samples,
        num_key,
        timescale,
        total_duration_us,
        avcc_bytes.len(),
        codec_string.as_deref(),
        offsets_ok,
        mdat_range,
        nal_length_size,
    );
    let description_hex = if avcc_bytes.is_empty() {
        String::from("—")
    } else {
        hex_dump(&avcc_bytes, MAX_DESCRIPTION_HEX)
    };
    let keymap_html = build_keyframe_map(&samples);
    let nals_html = build_nals_panel(
        &nals_key,
        &nals_delta,
        sample_key.as_ref(),
        sample_delta.as_ref(),
        &key_hex,
        &delta_hex,
    );
    let samples_html = build_samples_table(&samples);

    {
        let mut s = st.borrow_mut();
        s.clear();
        s.phase = String::from("файл разобран");
        s.error = None;
        s.file_name = name;
        s.file_size = bytes.len();
        s.summary_html = summary;
        s.description_hex = description_hex;
        s.keymap_html = keymap_html;
        s.nals_html = nals_html;
        s.samples_html = samples_html;
        s.codec_string = codec_string;
        s.avcc_bytes = avcc_bytes;
        s.config_supported = None;
    }
    update_ui(doc, &st.borrow());
}

// =============================================================================
// UI update
// =============================================================================

fn update_ui(doc: &Document, st: &DemoState) {
    if let Some(el) = doc.get_element_by_id("dm-summary") {
        el.set_inner_html(&if st.summary_html.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#"<div class="dm-summary">{}</div>"#,
                st.summary_html,
            ))
        });
    }
    if let Some(el) = doc.get_element_by_id("dm-hex") {
        el.set_inner_html(&if st.description_hex.is_empty() {
            String::new()
        } else {
            format_str(format_args!(
                r#"<span class="lbl">description (avcC) = </span>{hex}"#,
                hex = st.description_hex,
            ))
        });
    }
    if let Some(el) = doc.get_element_by_id("dm-keymap") {
        el.set_inner_html(&st.keymap_html);
    }
    if let Some(el) = doc.get_element_by_id("dm-nals") {
        el.set_inner_html(&st.nals_html);
    }
    if let Some(el) = doc.get_element_by_id("dm-samples") {
        el.set_inner_html(&st.samples_html);
    }

    if let Some(el) = doc.get_element_by_id("dm-check") {
        if st.can_check_config() {
            let _ = el.remove_attribute("disabled");
        } else {
            let _ = el.set_attribute("disabled", "");
        }
    }

    if let Some(el) = doc.get_element_by_id("dm-stats") {
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
        let check_info = match &st.config_supported {
            Some(Ok(msg)) => format_str(format_args!(r#" · <span class="ok">{msg}</span>"#)),
            Some(Err(msg)) => format_str(format_args!(r#" · <span class="err">{msg}</span>"#)),
            None => String::new(),
        };
        let html = format_str(format_args!(
            r#"{err}<div><span class="lbl">Статус:</span> <span class="val">{phase}</span>{file}{check}</div>"#,
            err = err_line,
            phase = st.phase,
            file = file_info,
            check = check_info,
        ));
        el.set_inner_html(&html);
    }
}

// =============================================================================
// Проверка конфига VideoDecoder
// =============================================================================

async fn check_config(doc: Document, st: Rc<RefCell<DemoState>>) {
    let (codec, description) = {
        let s = st.borrow();
        match (&s.codec_string, &s.avcc_bytes) {
            (Some(c), v) if !v.is_empty() => (c.clone(), v.clone()),
            _ => return,
        }
    };

    let global = js_sys::global();
    let ctor = match Reflect::get(&global, &JsValue::from_str("VideoDecoder")) {
        Ok(v) if v.is_function() => v,
        _ => {
            let mut s = st.borrow_mut();
            s.config_supported = Some(Err(String::from("VideoDecoder недоступен")));
            drop(s);
            update_ui(&doc, &st.borrow());
            return;
        }
    };

    let is_supported = match Reflect::get(&ctor, &JsValue::from_str("isConfigSupported")) {
        Ok(f) => f,
        Err(_) => {
            let mut s = st.borrow_mut();
            s.config_supported = Some(Err(String::from("isConfigSupported отсутствует")));
            drop(s);
            update_ui(&doc, &st.borrow());
            return;
        }
    };
    let Ok(is_supported) = is_supported.dyn_into::<js_sys::Function>() else {
        return;
    };

    let config = Object::new();
    set_prop(&config, "codec", &JsValue::from_str(&codec)).ok();
    let arr = Uint8Array::from(&description[..]);
    set_prop(&config, "description", arr.as_ref()).ok();

    let promise = match is_supported.call1(&ctor, &config) {
        Ok(p) => p,
        Err(e) => {
            let mut s = st.borrow_mut();
            s.config_supported = Some(Err(format_str(format_args!(
                "isConfigSupported: {e:?}"
            ))));
            drop(s);
            update_ui(&doc, &st.borrow());
            return;
        }
    };
    let Ok(promise) = promise.dyn_into::<js_sys::Promise>() else {
        return;
    };

    TimeoutFuture::new(0).await;
    let result = wasm_bindgen_futures::JsFuture::from(promise).await;

    let msg = match result {
        Ok(v) => {
            let supported = Reflect::get(&v, &JsValue::from_str("supported"))
                .ok()
                .and_then(|x| x.as_bool())
                .unwrap_or(false);
            if supported {
                Ok(format_str(format_args!("✓ {codec} supported")))
            } else {
                Err(format_str(format_args!("✗ {codec} not supported")))
            }
        }
        Err(e) => Err(format_str(format_args!("isConfigSupported error: {e:?}"))),
    };

    {
        let mut s = st.borrow_mut();
        s.config_supported = Some(msg);
    }
    update_ui(&doc, &st.borrow());
}

// =============================================================================
// Чтение файла
// =============================================================================

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
    let ab = result
        .dyn_into::<js_sys::ArrayBuffer>()
        .map_err(|_| JsValue::from_str("result is not ArrayBuffer"))?;
    return Ok(Uint8Array::new(&ab).to_vec());
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
    h2.set_text_content(Some("23.10 — Demuxing: MP4 → EncodedVideoChunk"));
    root.append_child(&h2)?;
    root.append_child(&build_description_block(&document)?)?;

    // Controls.
    let controls = document.create_element("div")?;
    controls.set_class_name("dm-block");
    controls.set_inner_html(
        r#"<input type="file" id="dm-file" accept=".mp4,.m4v,.mov,video/mp4" style="display:none">
           <button class="dm-btn dm-btn-primary" id="dm-open">Открыть MP4</button>
           <button class="dm-btn dm-btn-check" id="dm-check" disabled>Проверить конфиг</button>
           <span class="dm-label">stbl → EncodedVideoChunk</span>"#,
    );
    root.append_child(&controls)?;

    // Stats.
    let stats = document.create_element("div")?;
    stats.set_class_name("dm-stats");
    stats.set_id("dm-stats");
    stats.set_inner_html(
        r#"<div><span class="lbl">Статус:</span> <span class="val">простой</span></div>"#,
    );
    root.append_child(&stats)?;

    // Summary.
    let summary = document.create_element("div")?;
    summary.set_id("dm-summary");
    root.append_child(&summary)?;

    // Description hex.
    let hex_el = document.create_element("div")?;
    hex_el.set_class_name("dm-hex");
    hex_el.set_id("dm-hex");
    root.append_child(&hex_el)?;

    // Key frame map.
    let keymap = document.create_element("div")?;
    keymap.set_id("dm-keymap");
    root.append_child(&keymap)?;

    // NALs panel.
    let nals = document.create_element("div")?;
    nals.set_id("dm-nals");
    root.append_child(&nals)?;

    // Samples table.
    let samples = document.create_element("div")?;
    samples.set_id("dm-samples");
    root.append_child(&samples)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("dm-footnote");
    footnote.set_text_content(Some(
        "stco + stsc + stsz → список (offset, size). stts → timestamps. \
         stss → key frames. avcC → description и codec-string. \
         Внутри key sample — SPS, PPS, IDR. Внутри delta — non-IDR slice.",
    ));
    root.append_child(&footnote)?;

    let state = Rc::new(RefCell::new(DemoState::new()));

    let input_el = document
        .get_element_by_id("dm-file")
        .ok_or_else(|| JsValue::from_str("no #dm-file"))?;
    let input: HtmlInputElement = input_el.dyn_into()?;

    // Кнопка «Открыть MP4».
    {
        let input_for_btn = input.clone();
        let btn_el = document
            .get_element_by_id("dm-open")
            .ok_or_else(|| JsValue::from_str("no #dm-open"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            input_for_btn.click();
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // Кнопка «Проверить конфиг».
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("dm-check")
            .ok_or_else(|| JsValue::from_str("no #dm-check"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc = doc.clone();
            let st = st.clone();
            spawn_local(async move {
                check_config(doc, st).await;
            });
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
                        TimeoutFuture::new(0).await;
                        process_bytes(&doc_cb, st_cb, name, bytes).await;
                    }
                    Err(e) => {
                        set_error(
                            &doc_cb,
                            st_cb,
                            name,
                            0,
                            format_str(format_args!("read: {e:?}")),
                        );
                    }
                }
            });
        });
        input.add_event_listener_with_callback("change", onchange.as_ref().unchecked_ref())?;
        onchange.forget();
    }

    return Ok(());
}
```


Ключевые функции:

- `process_bytes` — верхний уровень: парсинг + анализ + UI, с тремя `TimeoutFuture::new(0).await`.
- `parse_avcc_nals` — разбирает AVCC-структуру внутри sample-а.
- `find_video_trak`, `extract_hdlr_type`, `find_mdat_range`, `extract_sample_bytes` — навигация по дереву.
- `build_summary` — метрики: samples, key frames, FPS, bitrate, codec, NAL format, offsets OK.
- `build_keyframe_map` — карта с группировкой по 10 и переносом каждые 80 символов.
- `build_nals_panel`, `render_chunk_block` — NAL-разбор первых key и delta samples.
- `build_samples_table` — таблица первых 30 samples.
- `check_config` — `VideoDecoder.isConfigSupported` через Reflect.
- `read_file` — FileReader с 0-аргументными closures.

---

## 23.10.6. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Тесты demuxer-а (24 теста)
wasm-pack test --headless --chrome  --test wasm_mp4_demuxer
wasm-pack test --headless --firefox --test wasm_mp4_demuxer

# 3. Тесты 23.9 (не сломались?)
wasm-pack test --headless --chrome  --test wasm_container_parser

# 4. Демо
npm run wasm:dev && npm start
```

---

## 23.10.7. Итоговый вид демо

Для загруженного MP4 (25 МБ, 2175 samples, 30 fps):

```
┌─ Summary ────────────────────────────────────────────────┐
│ Samples:     2175                                        │
│ Key frames:  15 (0.7%)                                   │
│ Timescale:   15360                                       │
│ Duration:    72.58 s                                     │
│ Frame rate:  30.00 fps (avg)                             │
│ Bitrate:     2.81 Mbps (video)                           │
│ Codec:       avc1.64001F                                 │
│ NAL format:  AVCC (length-prefix = 4 B)                  │
│ Description: 51 B                                        │
│ mdat range:  60751..26259331                             │
│ Offsets:     ✓ все offset-ы внутри mdat                  │
└──────────────────────────────────────────────────────────┘

description (avcC) = 01 64 00 1F FF E1 00 1E 67 64 00 1F …

┌─ Keymap ─────────────────────────────────────────────────┐
│ K·········K·········K·········K·········K·········K······│
│ ··········K·········K·········K··························│
│ ··········K·········                                     │
│ 1 символ = 10 sample(ов) · X = все key · K = есть key    │
└──────────────────────────────────────────────────────────┘

┌─ Chunk #0 key · offset=60759 · size=69370 B · ts=0.00 мс · NAL: 1───┐
│   NAL #0 type=IDR slice len=69366                                   │
│   hex[0..32]: 00 01 0E F6 65 88 80 20 0F F8 2F 46 01 2A …           │
│                                                                     │
│ Chunk #1 delta · offset=130156 · size=23958 B · ts=33.33 мс · NAL: 1│
│   NAL #0 type=non-IDR slice len=23954                               │
│   hex[0..32]: 00 00 5D 92 61 E1 02 32 BF D8 0E 6E 00 84 …           │
───────────└──────────────────────────────────────────────────────────┘

┌─ Samples table (первые 30) ──────────────────────────────┐
│ #  offset   size    timestamp  duration  kind            │
│ 0  60759    69370   0.00 мс    33.33 мс  key             │
│ 1  130156   23958   33.33 мс   33.33 мс  delta           │
│ …                                                        │
└──────────────────────────────────────────────────────────┘
```

**Что видит студент:**

1. **Key sample** содержит **IDR slice** — то, с чего может начаться декодирование.
2. **Delta sample** содержит **non-IDR slice** — минимальный размер, зависит от предыдущих.
3. **Timescale 15360, delta 512** → 33.33 мс → 30 fps.
4. **AVCC length-prefix = 4 байта** — то, что описано в `avcC[4] & 0x03 + 1`.
5. **Проверка смещений** — все sample попадают в `mdat`, автоматическая валидация.
6. **`VideoDecoder.isConfigSupported` возвращает `true`** — конфиг с SPS/PPS валиден.

---

## 23.10.8. Итог

**`stbl` — реляционная схема, а не плоский список.** Пять таблиц связаны через `stsc`, `stco` даёт позиции chunk-ов, `stsz` — размеры, `stts` — длительности, `stss` — key frames. Только вместе они дают полную картину.

**Алгоритм:** пройти по chunk-ам → внутри chunk-а по сэмплам, накапливая offset → наложить timestamp-ы из `stts` → пометить key из `stss`.

**`mdhd.timescale`** — обязательный знаменатель timestamp-ов.

**`description` для `VideoDecoder.configure`** — это payload `avcC`. Codec-string — `avc1.PPCCLL` из байт 1–3 `avcC`.

**NAL-разбор** — length-prefixed AVCC, не Annex B start codes. В sample лежит `[len][nal][len][nal]...`. Тип NAL — младшие 5 бит первого байта NAL: `0x65 & 0x1F = 5` = IDR slice.

**Самопроверка** — все offset-ы должны попадать в `mdat`. Один проход `verify_offsets` — и любая ошибка в арифметике всплывает сразу.

**Правило проекта.** Демо показывает **реальный workflow**: файл → таблицы → chunk-list → готовый к декодированию. Не отдельные вызовы API, а полный путь.

**Дальше — 23.11 (Muxing: EncodedVideoChunk → MP4)** — обратная операция: из `VideoSample` собрать `stbl` и записать валидный MP4.