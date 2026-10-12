# Глава 39. Real-Time Browser Architecture

Три главы назад мы разобрали **WebSocket** — надёжный упорядоченный канал. Две главы назад — **WebTransport** — мультиплексирование поверх QUIC. И ещё раньше — **WebRTC** — прямое peer-to-peer соединение. Три транспорта, три разных уровня абстракции. Но до сих пор мы рассматривали их по отдельности, как самостоятельные API.

Эта глава — **сводная**. Её задача — показать, что все три транспорта решают одну задачу: **доставить байты от одной стороны до другой с минимальной задержкой**. И ответить на три вопроса, которые возникают у каждого, кто строит real-time систему:

1. **Какой транспорт выбрать?** WebSocket, WebTransport или WebRTC — что подходит под конкретную задачу.
2. **Что именно передавать?** JSON, MessagePack или свой бинарный протокол.
3. **Как организовать код?** Чтобы смена транспорта не требовала переписывать логику приложения.

Ответы на эти вопросы — не «правильный выбор», а **архитектурные решения**. Их принимают один раз, и они определяют структуру кода на годы вперёд.

## 39.1. Три транспорта — три семантики

Сравним три транспорта по критериям, которые реально влияют на архитектуру.

```text
┌────────────────┬──────────────┬────────────────┬─────────────────┐
│                │ WebSocket    │ WebTransport   │ WebRTC          │
├────────────────┼──────────────┼────────────────┼─────────────────┤
│ Транспорт      │ TCP          │ QUIC (UDP)     │ DTLS/SCTP (UDP) │
│ Соединение     │ клиент-серв  │ клиент-сервер  │ peer-to-peer    │
│ Потоков        │ 1            │ N              │ N + media       │
│ Надёжность     │ reliable     │ reliable +     │ reliable +      │
│                │              │ unreliable     │ unreliable      │
│ Порядок        │ ordered      │ ordered +      │ configurable    │
│                │              │ unordered      │                 │
│ HOL blocking   │ есть         │ нет            │ нет             │
│ Latency        │ ~50-200 мс   │ ~20-100 мс     │ ~5-50 мс        │
│ Signaling      │ встроенный   │ встроенный     │ внешний         │
│ Поддержка      │ universal    │ Chrome/Edge/   │ universal       │
│                │              │ Safari 26.4+   │                 │
│ Setup сложность│ минимум       │ сертификат +   │ STUN/TURN +     │
│                │              │ QUIC-сервер    │ signaling       │
└────────────────┴──────────────┴────────────────┴─────────────────┘
```

**Ключевое различие — не скорость, а семантика.** WebSocket предлагает одну надёжную упорядоченную трубу. WebTransport и WebRTC позволяют выбрать надёжность **per-message**. Если приложению нужны только упорядоченные сообщения — WebSocket проще. Если нужны разные приоритеты данных — WebTransport.

## 39.2. Когда что выбирать

Практический критерий прост. Сначала спросите: **«Соединение между двумя клиентами или между клиентом и сервером?»**

**Peer-to-peer → WebRTC.** Единственный транспорт, у которого нет центрального сервера после установки соединения. Видеозвонки, совместное редактирование, файлообмен между устройствами.

**Клиент-сервер → WebSocket или WebTransport.**

Дальше — по данным:

| Приложение | Транспорт | Почему |
|---|---|---|
| Чат, уведомления | WebSocket | Одна упорядоченная труба, всё работает из коробки |
| Файловая синхронизация | WebSocket | Надёжность и порядок обязательны |
| Совместное редактирование | WebTransport | Курсоры — датаграммы, документ — потоки |
| Игровой netcode | WebTransport | Позиции — датаграммы, чат — потоки |
| Live-котировки | WebTransport | Отдельные потоки на инструмент |
| REST поверх WebSocket | WebSocket | Один канал, последовательные запросы |

**WebSocket везде, где нет явной причины для другого.** Он проще, работает во всех браузерах, не требует сертификата. WebTransport — дополнение, а не замена.

## 39.3. Бинарный протокол: зачем и как

Раз транспорт выбран, следующий вопрос — **что именно передавать**. Три варианта:

| Формат | Размер | Парсинг | Применение |
|---|---|---|---|
| **JSON** | +30-100% | `JSON.parse`, ~1 мкс | Быстрый старт, отладка |
| **MessagePack / CBOR** | +5-15% | ~3 мкс | Компромисс |
| **Свой бинарный** | базовый | ~0.5 мкс | Real-time, игры |

**JSON — не для real-time.** Причины:

1. **Размер.** `{"type":"position","x":1.5,"y":2.7}` — 35 байт. То же бинарно — 12 байт.
2. **Скорость парсинга.** JSON на 60 сообщениях в секунду съедает десятки микросекунд. Кажется мало, но на WASM с плотным потоком — ощутимо.
3. **Схема неявная.** Поле `"x"` может быть числом, строкой, отсутствовать — отлавливается в рантайме.
4. **Нет frame-границ.** WebSocket даёт границы сам, но TCP-поток — нет. Нужен явный framing.

**Свой бинарный протокол** решает всё это. Минимальный набор:

```text
┌──────┬────────┬──────┬───────┬───────┬──────────────┐
│ MAGIC│ VERSION│ KIND │  SEQ  │  LEN  │   PAYLOAD    │
│ 2 B  │  1 B   │ 1 B  │  4 B  │  4 B  │   N байт     │
└──────┴────────┴──────┴───────┴───────┴──────────────┘
  0x52  0x01      ...   u32 BE  u32 BE   ...
  0x54
```

**12 байт заголовка.** Фиксированная длина — байты читаются по смещениям, без парсера. `seq` — номер для сопоставления (запрос-ответ, RTT). `len` — размер payload — позволяет отбрасывать кадры заранее, если они слишком большие. Big-endian для сетевого порядка — не важно в браузере, но правильно.

**Frame-границы** — самое важное. TCP/QUIC не сохраняет границы сообщений: два сообщения могут прийти одним куском или одно — двумя. **Декодер обязан накапливать** — буфер, из которого читаются полные кадры по мере поступления байтов.

## 39.4. Rust protocol engine

Протокол-движок — **транспорт-независимый код**. Он не знает, идут байты через WebSocket, WebTransport или WebRTC. Только:

- `encode(frame) -> Vec<u8>` — сериализация.
- `FrameDecoder::push(bytes) -> Vec<Frame>` — разбор потока.

```rust
pub const MAGIC: [u8; 2] = [0x52, 0x54]; // "RT"
pub const VERSION: u8 = 1;
pub const HEADER_LEN: usize = 12;

#[derive(Clone, Copy, PartialEq, Eq)]
pub enum FrameKind {
    Text, Binary, Ping, Pong, Close, Error,
}

pub struct Frame {
    pub kind: FrameKind,
    pub seq: u32,
    pub payload: Vec<u8>,
}

pub fn encode(frame: &Frame) -> Result<Vec<u8>, String> {
    // 12 байт заголовка + payload.
}

pub struct FrameDecoder {
    buf: Vec<u8>,
    // статистика
}

impl FrameDecoder {
    /// Принимает произвольный кусок байт, возвращает готовые кадры.
    /// Неполный кадр остаётся в буфере до следующего вызова.
    pub fn push(&mut self, chunk: &[u8]) -> Vec<Result<Frame, DecodeError>> {
        // ...
    }
}
```

**`FrameDecoder` — stateful.** Он держит буфер между вызовами. Поток может дать половину заголовка, а через секунду — остаток. Декодер это переживает.

## 39.5. Transport layer: единый интерфейс

Протокол-движок не должен знать про конкретный транспорт. Он работает с абстракцией `Transport`:

```rust
pub enum TransportKind {
    WebSocket,
    WebTransport,
    WebRtc,
}

pub enum TransportState {
    Idle, Connecting, Ready, Closed, Failed,
}

pub enum TransportEvent {
    Connected,
    Disconnected(String),
    Bytes(Vec<u8>),
    Error(String),
}

pub struct Transport {
    kind: TransportKind,
    backend: Backend,
}

enum Backend {
    WebSocket(Rc<WsClient>),
    WebTransport(Rc<WtClient>),
}
```

**Публичный API не зависит от конкретного транспорта:**

```rust
impl Transport {
    pub fn ws(url: &str, on_event: Rc<dyn Fn(TransportEvent)>) -> Rc<Transport>;
    pub fn wt(url: &str, on_event: Rc<dyn Fn(TransportEvent)>) -> Rc<Transport>;

    pub fn kind(&self) -> TransportKind;
    pub fn state(&self) -> TransportState;
    pub fn connect(&self) -> Result<(), String>;
    pub fn send_binary(&self, bytes: &[u8]) -> Result<(), String>;
    pub fn close(&self);
}
```

Приложение использует `transport.send_binary(&encoded_frame)` — и не меняет ни строки, если переключиться с WebSocket на WebTransport. Внутри — разные модули (`ws_client`, `wt_client`), но поверх — единый интерфейс.

**Практическая ценность.** На старте проекта выбирается WebSocket (проще). Когда вырастают требования (нужен unreliable-канал, мультиплексирование), транспорт меняется на WebTransport за один день — переписывается только конфигурация, логика остаётся.

## 39.6. Структура проекта

```text
src/
  rta_protocol.rs                ← НОВЫЙ модуль: frame codec
  rta_transport.rs               ← НОВЫЙ модуль: unified transport
  app.rs                         ← демо (заменяется целиком)
tests/
  wasm_rta.rs                    ← тесты главы
src/lib.rs                       ← +2 строки: pub mod rta_protocol; pub mod rta_transport;
```

**Ссылки для изучения:**

- [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455) — WebSocket protocol, frame format.
- [RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000) — QUIC transport, framing.
- [Designing Data-Intensive Applications](https://dataintensive.net/) — главы про кодирование данных.
- [Cap'n Proto](https://capnproto.org/) — пример высокопроизводительного бинарного протокола.
- [FlatBuffers](https://flatbuffers.dev/) — zero-copy сериализация для real-time.

**Префикс CSS** — `rta-` (real-time architecture), не конфликтует с `gl-`, `ms-`, `cd-`, `aw-`, `md-`, `ws-`, `wt-`, `rtc-`.

**Cargo.toml** — новых фич не требуется. Используются фичи из глав 36, 37 и стандартные.

## 39.7. Модуль `rta_protocol.rs`

```rust
//! 39. Real-Time protocol engine.
//!
//! Транспорт-независимый кодек для real-time сообщений.
//! Работает поверх WebSocket, WebTransport или WebRTC без изменений.
//!
//! # Формат кадра
//!
//! ```text
//! ┌──────┬────────┬──────┬───────┬───────┬──────────────┐
//! │ MAGIC│ VERSION│ KIND │  SEQ  │  LEN  │   PAYLOAD    │
//! │ 2 B  │  1 B   │ 1 B  │  4 B  │  4 B  │   N байт     │
//! └──────┴────────┴──────┴───────┴───────┴──────────────┘
//!  0x52 0x54  0x01   ...   u32 BE  u32 BE   ...
//! ```
//!
//! Заголовок — 12 байт, big-endian для сетевого порядка.
//! Payload — произвольные байты, до MAX_PAYLOAD.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

use crate::format_str::format_str;

pub const MAGIC: [u8; 2] = [0x52, 0x54]; // "RT"
pub const VERSION: u8 = 1;
pub const HEADER_LEN: usize = 12;
pub const MAX_PAYLOAD: usize = 1024 * 1024; // 1 MiB

// ============================================================
//                    ТИПЫ КАДРОВ
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum FrameKind {
    Text = 0x01,
    Binary = 0x02,
    Ping = 0x03,
    Pong = 0x04,
    Close = 0x05,
    Error = 0x06,
}

impl FrameKind {
    pub fn from_byte(b: u8) -> Option<Self> {
        return match b {
            0x01 => Some(FrameKind::Text),
            0x02 => Some(FrameKind::Binary),
            0x03 => Some(FrameKind::Ping),
            0x04 => Some(FrameKind::Pong),
            0x05 => Some(FrameKind::Close),
            0x06 => Some(FrameKind::Error),
            _ => None,
        };
    }
    pub fn as_byte(self) -> u8 {
        return self as u8;
    }
    pub fn as_str(self) -> &'static str {
        return match self {
            FrameKind::Text => "text",
            FrameKind::Binary => "binary",
            FrameKind::Ping => "ping",
            FrameKind::Pong => "pong",
            FrameKind::Close => "close",
            FrameKind::Error => "error",
        };
    }
}

// ============================================================
//                    КАДР
// ============================================================

#[derive(Clone, Debug)]
pub struct Frame {
    pub kind: FrameKind,
    pub seq: u32,
    pub payload: Vec<u8>,
}

impl Frame {
    pub fn text(seq: u32, s: &str) -> Self {
        return Self {
            kind: FrameKind::Text,
            seq,
            payload: s.as_bytes().to_vec(),
        };
    }
    pub fn binary(seq: u32, bytes: &[u8]) -> Self {
        return Self {
            kind: FrameKind::Binary,
            seq,
            payload: bytes.to_vec(),
        };
    }
    pub fn ping(seq: u32) -> Self {
        return Self {
            kind: FrameKind::Ping,
            seq,
            payload: Vec::new(),
        };
    }
    pub fn pong(seq: u32) -> Self {
        return Self {
            kind: FrameKind::Pong,
            seq,
            payload: Vec::new(),
        };
    }
    pub fn total_size(&self) -> usize {
        return HEADER_LEN + self.payload.len();
    }
    pub fn as_text(&self) -> String {
        return String::from_utf8_lossy(&self.payload).into_owned();
    }
}

// ============================================================
//                    ENCODE
// ============================================================

pub fn encode(frame: &Frame) -> Result<Vec<u8>, String> {
    if frame.payload.len() > MAX_PAYLOAD {
        return Err(format_str(format_args!(
            "payload too large: {} > {}",
            frame.payload.len(),
            MAX_PAYLOAD,
        )));
    }
    let mut out = Vec::with_capacity(frame.total_size());
    out.extend_from_slice(&MAGIC);
    out.push(VERSION);
    out.push(frame.kind.as_byte());
    out.extend_from_slice(&frame.seq.to_be_bytes());
    out.extend_from_slice(&(frame.payload.len() as u32).to_be_bytes());
    out.extend_from_slice(&frame.payload);
    return Ok(out);
}

// ============================================================
//                    DECODE
// ============================================================

#[derive(Debug, Clone, PartialEq)]
pub enum DecodeError {
    BadMagic { got: [u8; 2] },
    BadVersion { got: u8 },
    BadKind { got: u8 },
    PayloadTooLarge { len: u32 },
    IncompleteHeader { have: usize },
    IncompletePayload { have: usize, need: usize },
}

impl DecodeError {
    pub fn as_str(&self) -> String {
        return match self {
            DecodeError::BadMagic { got } => format_str(format_args!(
                "bad magic: {:02x} {:02x}",
                got[0],
                got[1],
            )),
            DecodeError::BadVersion { got } => format_str(format_args!(
                "bad version: {got}",
            )),
            DecodeError::BadKind { got } => format_str(format_args!(
                "unknown kind: 0x{got:02x}",
            )),
            DecodeError::PayloadTooLarge { len } => format_str(format_args!(
                "payload too large: {len}",
            )),
            DecodeError::IncompleteHeader { have } => format_str(format_args!(
                "incomplete header: {have}/{HEADER_LEN}",
            )),
            DecodeError::IncompletePayload { have, need } => format_str(format_args!(
                "incomplete payload: {have}/{need}",
            )),
        };
    }
    pub fn is_incomplete(&self) -> bool {
        return matches!(
            self,
            DecodeError::IncompleteHeader { .. }
                | DecodeError::IncompletePayload { .. }
        );
    }
}

pub fn try_decode_one(buf: &[u8]) -> Result<(Frame, usize), DecodeError> {
    if buf.len() < HEADER_LEN {
        return Err(DecodeError::IncompleteHeader { have: buf.len() });
    }
    if buf[0] != MAGIC[0] || buf[1] != MAGIC[1] {
        return Err(DecodeError::BadMagic { got: [buf[0], buf[1]] });
    }
    if buf[2] != VERSION {
        return Err(DecodeError::BadVersion { got: buf[2] });
    }
    let kind = FrameKind::from_byte(buf[3])
        .ok_or(DecodeError::BadKind { got: buf[3] })?;
    let mut seq_bytes = [0u8; 4];
    seq_bytes.copy_from_slice(&buf[4..8]);
    let seq = u32::from_be_bytes(seq_bytes);
    let mut len_bytes = [0u8; 4];
    len_bytes.copy_from_slice(&buf[8..12]);
    let len = u32::from_be_bytes(len_bytes);
    if len as usize > MAX_PAYLOAD {
        return Err(DecodeError::PayloadTooLarge { len });
    }
    let total = HEADER_LEN + len as usize;
    if buf.len() < total {
        return Err(DecodeError::IncompletePayload {
            have: buf.len() - HEADER_LEN,
            need: len as usize,
        });
    }
    let payload = buf[HEADER_LEN..total].to_vec();
    return Ok((Frame { kind, seq, payload }, total));
}

// ============================================================
//                    STATEFUL DECODER
// ============================================================

pub struct FrameDecoder {
    buf: Vec<u8>,
    frames_emitted: u64,
    bytes_consumed: u64,
}

impl FrameDecoder {
    pub fn new() -> Self {
        return Self {
            buf: Vec::with_capacity(1024),
            frames_emitted: 0,
            bytes_consumed: 0,
        };
    }

    pub fn frames_emitted(&self) -> u64 {
        return self.frames_emitted;
    }
    pub fn bytes_consumed(&self) -> u64 {
        return self.bytes_consumed;
    }
    pub fn pending_bytes(&self) -> usize {
        return self.buf.len();
    }

    pub fn push(&mut self, chunk: &[u8]) -> Vec<Result<Frame, DecodeError>> {
        self.buf.extend_from_slice(chunk);
        let mut out: Vec<Result<Frame, DecodeError>> = Vec::new();
        loop {
            match try_decode_one(&self.buf) {
                Ok((frame, consumed)) => {
                    self.buf.drain(0..consumed);
                    self.frames_emitted += 1;
                    self.bytes_consumed += consumed as u64;
                    out.push(Ok(frame));
                }
                Err(e) if e.is_incomplete() => {
                    break;
                }
                Err(e) => {
                    self.buf.clear();
                    out.push(Err(e));
                    break;
                }
            }
        }
        return out;
    }

    pub fn reset(&mut self) {
        self.buf.clear();
    }
}

impl Default for FrameDecoder {
    fn default() -> Self {
        return Self::new();
    }
}

// ============================================================
//                    UI HELPERS
// ============================================================

pub fn hex_preview(bytes: &[u8], max: usize) -> String {
    let n = bytes.len().min(max);
    let mut out = String::with_capacity(n * 3 + 16);
    for (i, b) in bytes.iter().take(n).enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format_str(format_args!("{:02x}", b)));
    }
    if bytes.len() > max {
        out.push_str(" …");
    }
    return out;
}

pub struct FrameAnatomy {
    pub magic: [u8; 2],
    pub version: u8,
    pub kind_byte: u8,
    pub seq: u32,
    pub len: u32,
    pub payload_preview: String,
}

pub fn anatomy(bytes: &[u8]) -> Option<FrameAnatomy> {
    if bytes.len() < HEADER_LEN {
        return None;
    }
    let mut seq_bytes = [0u8; 4];
    seq_bytes.copy_from_slice(&bytes[4..8]);
    let mut len_bytes = [0u8; 4];
    len_bytes.copy_from_slice(&bytes[8..12]);
    return Some(FrameAnatomy {
        magic: [bytes[0], bytes[1]],
        version: bytes[2],
        kind_byte: bytes[3],
        seq: u32::from_be_bytes(seq_bytes),
        len: u32::from_be_bytes(len_bytes),
        payload_preview: hex_preview(&bytes[HEADER_LEN..], 16),
    });
}
```

## 39.8. Модуль `rta_transport.rs`

```rust
//! 39. Unified transport layer.
//!
//! Абстракция над WebSocket, WebTransport и WebRTC.
//! Протокол-движок (`rta_protocol`) не знает, по какому каналу
//! идут байты — только `send_binary` и callback приёма.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

use std::rc::Rc;

use crate::format_str::format_str;
use crate::ws_client::{WsClient, WsConfig, WsEvent, WsState};
use crate::wt_client::{WtClient, WtConfig, WtEvent, WtState};

// ============================================================
//                    ТИП ТРАНСПОРТА
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum TransportKind {
    WebSocket,
    WebTransport,
    WebRtc,
}

impl TransportKind {
    pub fn as_str(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "websocket",
            TransportKind::WebTransport => "webtransport",
            TransportKind::WebRtc => "webrtc",
        };
    }
    pub fn from_str(s: &str) -> Self {
        return match s {
            "websocket" => TransportKind::WebSocket,
            "webtransport" => TransportKind::WebTransport,
            "webrtc" => TransportKind::WebRtc,
            _ => TransportKind::WebSocket,
        };
    }
    pub fn default_url(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "wss://echo.websocket.org",
            TransportKind::WebTransport => "https://127.0.0.1:4433/echo",
            TransportKind::WebRtc => "—",
        };
    }
    pub fn reliability(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "reliable + ordered",
            TransportKind::WebTransport => "reliable + unreliable",
            TransportKind::WebRtc => "configurable per channel",
        };
    }
    pub fn multiplexing(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "нет (1 поток)",
            TransportKind::WebTransport => "да (N streams + datagrams)",
            TransportKind::WebRtc => "да (data channels + media)",
        };
    }
    pub fn latency(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "~50-200 мс",
            TransportKind::WebTransport => "~20-100 мс",
            TransportKind::WebRtc => "~5-50 мс",
        };
    }
    pub fn browser_support(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "все браузеры",
            TransportKind::WebTransport => "Chrome / Edge / Safari 26.4+",
            TransportKind::WebRtc => "все браузеры",
        };
    }
    pub fn setup(self) -> &'static str {
        return match self {
            TransportKind::WebSocket => "минимум",
            TransportKind::WebTransport => "сертификат + QUIC-сервер",
            TransportKind::WebRtc => "STUN/TURN + signaling",
        };
    }
}

// ============================================================
//                    СОСТОЯНИЕ
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum TransportState {
    Idle,
    Connecting,
    Ready,
    Closed,
    Failed,
}

impl TransportState {
    pub fn as_str(self) -> &'static str {
        return match self {
            TransportState::Idle => "idle",
            TransportState::Connecting => "connecting",
            TransportState::Ready => "ready",
            TransportState::Closed => "closed",
            TransportState::Failed => "failed",
        };
    }
}

// ============================================================
//                    СОБЫТИЯ
// ============================================================

#[derive(Clone, Debug)]
pub enum TransportEvent {
    Connected,
    Disconnected(String),
    Bytes(Vec<u8>),
    Error(String),
}

// ============================================================
//                    TRANSPORT
// ============================================================

enum Backend {
    WebSocket(Rc<WsClient>),
    WebTransport(Rc<WtClient>),
}

pub struct Transport {
    kind: TransportKind,
    backend: Backend,
}

impl Transport {
    pub fn ws(url: &str, on_event: Rc<dyn Fn(TransportEvent)>) -> Rc<Transport> {
        let cb = on_event.clone();
        let adapter: Rc<dyn Fn(WsEvent)> = Rc::new(move |ev: WsEvent| {
            match ev {
                WsEvent::Opened => cb(TransportEvent::Connected),
                WsEvent::Closed { code, reason, was_clean } => {
                    // Строка формируется в отдельной функции,
                    // чтобы избежать rust-analyzer-проблемы с
                    // format! в сложной позиции.
                    let info = closed_info(code, &reason, was_clean);
                    cb(TransportEvent::Disconnected(info));
                }
                WsEvent::Binary(v) => cb(TransportEvent::Bytes(v)),
                WsEvent::Text(s) => cb(TransportEvent::Bytes(s.into_bytes())),
                WsEvent::Error(e) => cb(TransportEvent::Error(e)),
                WsEvent::Reconnecting { attempt, delay_ms } => {
                    let msg = reconnecting_info(attempt, delay_ms);
                    cb(TransportEvent::Error(msg));
                }
                WsEvent::ReconnectExhausted => {
                    cb(TransportEvent::Error(
                        "reconnect exhausted".to_string(),
                    ));
                }
            }
        });
        let client = WsClient::new(WsConfig::new(url), adapter);
        return Rc::new(Transport {
            kind: TransportKind::WebSocket,
            backend: Backend::WebSocket(client),
        });
    }

    pub fn wt(url: &str, on_event: Rc<dyn Fn(TransportEvent)>) -> Rc<Transport> {
        let cb = on_event.clone();
        let adapter: Rc<dyn Fn(WtEvent)> = Rc::new(move |ev: WtEvent| {
            match ev {
                WtEvent::Ready => cb(TransportEvent::Connected),
                WtEvent::Closed { reason } => {
                    cb(TransportEvent::Disconnected(reason));
                }
                WtEvent::Datagram(v) => cb(TransportEvent::Bytes(v)),
                WtEvent::StreamData(v) => cb(TransportEvent::Bytes(v)),
                WtEvent::Error(e) => cb(TransportEvent::Error(e)),
                WtEvent::Text(s) => cb(TransportEvent::Error(s)),
                WtEvent::Connecting | WtEvent::Stats => {}
            }
        });
        let client = WtClient::new(WtConfig::new(url), adapter);
        return Rc::new(Transport {
            kind: TransportKind::WebTransport,
            backend: Backend::WebTransport(client),
        });
    }

    pub fn kind(&self) -> TransportKind {
        return self.kind;
    }

    pub fn state(&self) -> TransportState {
        return match &self.backend {
            Backend::WebSocket(c) => match c.state() {
                WsState::Connecting => TransportState::Connecting,
                WsState::Open => TransportState::Ready,
                WsState::Closing => TransportState::Connecting,
                WsState::Closed => TransportState::Closed,
            },
            Backend::WebTransport(c) => match c.state() {
                WtState::Idle => TransportState::Idle,
                WtState::Connecting => TransportState::Connecting,
                WtState::Ready => TransportState::Ready,
                WtState::Closed => TransportState::Closed,
                WtState::Failed => TransportState::Failed,
            },
        };
    }

    pub fn connect(&self) -> Result<(), String> {
        return match &self.backend {
            Backend::WebSocket(c) => c
                .connect()
                .map_err(|e| format_str(format_args!("ws connect: {e:?}"))),
            Backend::WebTransport(c) => c
                .connect()
                .map_err(|e| format_str(format_args!("wt connect: {e:?}"))),
        };
    }

    pub fn send_binary(&self, bytes: &[u8]) -> Result<(), String> {
        return match &self.backend {
            Backend::WebSocket(c) => c
                .send_binary(bytes)
                .map_err(|e| format_str(format_args!("ws send: {e:?}"))),
            Backend::WebTransport(c) => c
                .send_datagram(bytes)
                .map_err(|e| format_str(format_args!("wt send: {e:?}"))),
        };
    }

    pub fn close(&self) {
        match &self.backend {
            Backend::WebSocket(c) => c.close(),
            Backend::WebTransport(c) => c.close(),
        }
    }
}

// ============================================================
//                    HELPERS
// ============================================================

/// Формирует строку для WsEvent::Closed в отдельной функции —
/// иначе rust-analyzer теряет тип у format! внутри if/else.
fn closed_info(code: u16, reason: &str, was_clean: bool) -> String {
    if reason.is_empty() {
        return format_str(format_args!(
            "closed code={code} clean={was_clean}",
        ));
    }
    return format_str(format_args!(
        "closed code={code} reason={reason} clean={was_clean}",
    ));
}

/// Формирует строку для WsEvent::Reconnecting.
fn reconnecting_info(attempt: u32, delay_ms: u32) -> String {
    return format_str(format_args!(
        "reconnecting: attempt={attempt} delay={delay_ms}мс",
    ));
}
```

## 39.9. Тесты `tests/wasm_rta.rs`

```rust
//! 39. Wasm-тесты rta_protocol: encode/decode, partial reads,
//! потоковая обработка.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::rta_protocol::{
    anatomy, encode, hex_preview, try_decode_one, DecodeError,
    Frame, FrameDecoder, FrameKind, HEADER_LEN, MAGIC, MAX_PAYLOAD,
    VERSION,
};
use hello_wasm02::rta_transport::{TransportKind, TransportState};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- FrameKind ----------------

#[wasm_bindgen_test]
fn wasm_frame_kind_roundtrip() {
    let cases: [(&str, u8, FrameKind); 6] = [
        ("text", 0x01, FrameKind::Text),
        ("binary", 0x02, FrameKind::Binary),
        ("ping", 0x03, FrameKind::Ping),
        ("pong", 0x04, FrameKind::Pong),
        ("close", 0x05, FrameKind::Close),
        ("error", 0x06, FrameKind::Error),
    ];
    for (s, b, k) in cases {
        assert_eq!(FrameKind::from_byte(b), Some(k));
        assert_eq!(k.as_byte(), b);
        assert_eq!(k.as_str(), s);
    }
}

#[wasm_bindgen_test]
fn wasm_frame_kind_unknown() {
    assert_eq!(FrameKind::from_byte(0x00), None);
    assert_eq!(FrameKind::from_byte(0x99), None);
    assert_eq!(FrameKind::from_byte(0xff), None);
}

// ---------------- Encode ----------------

#[wasm_bindgen_test]
fn wasm_encode_text_frame_basic() {
    let f = Frame::text(42, "hello");
    let bytes = encode(&f).expect("encode");
    assert_eq!(bytes.len(), HEADER_LEN + 5);
    assert_eq!(&bytes[0..2], &MAGIC);
    assert_eq!(bytes[2], VERSION);
    assert_eq!(bytes[3], FrameKind::Text.as_byte());
    assert_eq!(&bytes[4..8], &42u32.to_be_bytes());
    assert_eq!(&bytes[8..12], &5u32.to_be_bytes());
    assert_eq!(&bytes[12..], b"hello");
}

#[wasm_bindgen_test]
fn wasm_encode_ping_frame_empty_payload() {
    let f = Frame::ping(1);
    let bytes = encode(&f).expect("encode");
    assert_eq!(bytes.len(), HEADER_LEN);
    assert_eq!(bytes[3], FrameKind::Ping.as_byte());
}

#[wasm_bindgen_test]
fn wasm_encode_rejects_oversized() {
    let big = vec![0u8; MAX_PAYLOAD + 1];
    let f = Frame::binary(1, &big);
    assert!(encode(&f).is_err());
}

// ---------------- Decode ----------------

#[wasm_bindgen_test]
fn wasm_decode_roundtrip() {
    let orig = Frame::text(123, "привет");
    let bytes = encode(&orig).expect("encode");
    let (parsed, consumed) = try_decode_one(&bytes).expect("decode");
    assert_eq!(consumed, bytes.len());
    assert_eq!(parsed.kind, FrameKind::Text);
    assert_eq!(parsed.seq, 123);
    assert_eq!(parsed.as_text(), "привет");
}

#[wasm_bindgen_test]
fn wasm_decode_bad_magic() {
    let mut bytes = encode(&Frame::text(1, "x")).expect("encode");
    bytes[0] = 0xff;
    let err = try_decode_one(&bytes).unwrap_err();
    match err {
        DecodeError::BadMagic { got } => assert_eq!(got, [0xff, 0x54]),
        _ => panic!("unexpected error: {err:?}"),
    }
}

#[wasm_bindgen_test]
fn wasm_decode_bad_version() {
    let mut bytes = encode(&Frame::text(1, "x")).expect("encode");
    bytes[2] = 99;
    let err = try_decode_one(&bytes).unwrap_err();
    match err {
        DecodeError::BadVersion { got } => assert_eq!(got, 99),
        _ => panic!("unexpected error: {err:?}"),
    }
}

#[wasm_bindgen_test]
fn wasm_decode_bad_kind() {
    let mut bytes = encode(&Frame::text(1, "x")).expect("encode");
    bytes[3] = 0x77;
    let err = try_decode_one(&bytes).unwrap_err();
    match err {
        DecodeError::BadKind { got } => assert_eq!(got, 0x77),
        _ => panic!("unexpected error: {err:?}"),
    }
}

#[wasm_bindgen_test]
fn wasm_decode_incomplete_header() {
    let bytes = encode(&Frame::text(1, "x")).expect("encode");
    let err = try_decode_one(&bytes[..5]).unwrap_err();
    match err {
        DecodeError::IncompleteHeader { have } => assert_eq!(have, 5),
        _ => panic!("unexpected error: {err:?}"),
    }
    assert!(err.is_incomplete());
}

#[wasm_bindgen_test]
fn wasm_decode_incomplete_payload() {
    let bytes = encode(&Frame::text(1, "hello")).expect("encode");
    // Обрезаем — payload целиком не влезает.
    let err = try_decode_one(&bytes[..HEADER_LEN + 2]).unwrap_err();
    match err {
        DecodeError::IncompletePayload { have, need } => {
            assert_eq!(have, 2);
            assert_eq!(need, 5);
        }
        _ => panic!("unexpected error: {err:?}"),
    }
    assert!(err.is_incomplete());
}

// ---------------- FrameDecoder ----------------

#[wasm_bindgen_test]
fn wasm_decoder_single_frame() {
    let mut d = FrameDecoder::new();
    let bytes = encode(&Frame::text(1, "hello")).expect("encode");
    let results = d.push(&bytes);
    assert_eq!(results.len(), 1);
    let frame = results[0].as_ref().unwrap();
    assert_eq!(frame.as_text(), "hello");
    assert_eq!(d.frames_emitted(), 1);
    assert_eq!(d.pending_bytes(), 0);
}

#[wasm_bindgen_test]
fn wasm_decoder_two_frames_in_one_chunk() {
    let mut d = FrameDecoder::new();
    let mut bytes = encode(&Frame::text(1, "a")).expect("encode");
    bytes.extend_from_slice(&encode(&Frame::text(2, "b")).expect("encode"));
    let results = d.push(&bytes);
    assert_eq!(results.len(), 2);
    assert_eq!(results[0].as_ref().unwrap().seq, 1);
    assert_eq!(results[1].as_ref().unwrap().seq, 2);
}

#[wasm_bindgen_test]
fn wasm_decoder_partial_header() {
    let mut d = FrameDecoder::new();
    let bytes = encode(&Frame::text(1, "hello")).expect("encode");
    // Разбиваем на 3 части: 5, 10, остаток.
    let r1 = d.push(&bytes[..5]);
    assert!(r1.is_empty());
    let r2 = d.push(&bytes[5..10]);
    assert!(r2.is_empty());
    let r3 = d.push(&bytes[10..]);
    assert_eq!(r3.len(), 1);
    assert_eq!(r3[0].as_ref().unwrap().as_text(), "hello");
}

#[wasm_bindgen_test]
fn wasm_decoder_byte_by_byte() {
    let mut d = FrameDecoder::new();
    let bytes = encode(&Frame::ping(7)).expect("encode");
    let mut total = Vec::new();
    for b in &bytes {
        let mut r = d.push(&[*b]);
        total.append(&mut r);
    }
    assert_eq!(total.len(), 1);
    assert_eq!(total[0].as_ref().unwrap().seq, 7);
}

#[wasm_bindgen_test]
fn wasm_decoder_fatal_error_clears_buffer() {
    let mut d = FrameDecoder::new();
    // Мусор, не начинающийся с MAGIC.
    let results = d.push(&[0xff, 0xff, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]);
    assert_eq!(results.len(), 1);
    assert!(results[0].is_err());
    assert_eq!(d.pending_bytes(), 0);
}

// ---------------- Anatomy ----------------

#[wasm_bindgen_test]
fn wasm_anatomy_basic() {
    let bytes = encode(&Frame::text(42, "abc")).expect("encode");
    let a = anatomy(&bytes).expect("anatomy");
    assert_eq!(a.magic, MAGIC);
    assert_eq!(a.version, VERSION);
    assert_eq!(a.kind_byte, FrameKind::Text.as_byte());
    assert_eq!(a.seq, 42);
    assert_eq!(a.len, 3);
    assert_eq!(a.payload_preview, "61 62 63");
}

#[wasm_bindgen_test]
fn wasm_anatomy_too_short() {
    assert!(anatomy(&[1, 2, 3]).is_none());
}

// ---------------- hex_preview ----------------

#[wasm_bindgen_test]
fn wasm_hex_preview_basic() {
    assert_eq!(hex_preview(&[0xde, 0xad, 0xbe, 0xef], 8), "de ad be ef");
}

#[wasm_bindgen_test]
fn wasm_hex_preview_truncates() {
    let bytes = [0x01, 0x02, 0x03, 0x04, 0x05, 0x06];
    let s = hex_preview(&bytes, 3);
    assert!(s.starts_with("01 02 03"));
    assert!(s.ends_with('…'));
}

#[wasm_bindgen_test]
fn wasm_hex_preview_empty() {
    assert_eq!(hex_preview(&[], 8), "");
}

// ---------------- Transport meta ----------------

#[wasm_bindgen_test]
fn wasm_transport_kind_roundtrip() {
    let cases: [(&str, TransportKind); 3] = [
        ("websocket", TransportKind::WebSocket),
        ("webtransport", TransportKind::WebTransport),
        ("webrtc", TransportKind::WebRtc),
    ];
    for (s, k) in cases {
        assert_eq!(TransportKind::from_str(s), k);
        assert_eq!(k.as_str(), s);
    }
}

#[wasm_bindgen_test]
fn wasm_transport_kind_unknown_defaults_to_ws() {
    assert_eq!(TransportKind::from_str("garbage"), TransportKind::WebSocket);
}

#[wasm_bindgen_test]
fn wasm_transport_state_as_str() {
    assert_eq!(TransportState::Idle.as_str(), "idle");
    assert_eq!(TransportState::Ready.as_str(), "ready");
    assert_eq!(TransportState::Failed.as_str(), "failed");
}

#[wasm_bindgen_test]
fn wasm_transport_meta_non_empty() {
    for k in [
        TransportKind::WebSocket,
        TransportKind::WebTransport,
        TransportKind::WebRtc,
    ] {
        assert!(!k.reliability().is_empty());
        assert!(!k.multiplexing().is_empty());
        assert!(!k.latency().is_empty());
        assert!(!k.browser_support().is_empty());
        assert!(!k.setup().is_empty());
    }
}
```

**Команды запуска:**

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_rta
npm run wasm:dev && npm start
```

## 39.10. Демо-пример

Демо показывает **один протокол над двумя транспортами**. Пользователь выбирает WebSocket или WebTransport, подключается, отправляет сообщения — а внутри всё идёт через один и тот же frame codec.

**Три колонки UI:**

1. **Транспорт** — выбор (WebSocket / WebTransport), URL, Connect, состояние. Ниже — Compose: сообщение, Send.
2. **Инспектор кадров** — последний отправленный кадр (слева) и последний полученный (справа). Оба показываются как hex-байты с разбором полей заголовка: magic, version, kind, seq, len, payload preview.
3. **Журнал** — все события: подключения, кадры, ошибки, метрики.

**Под колонками** — матрица выбора транспорта: сравнение WebSocket, WebTransport и WebRTC по шести критериям.

**Что демонстрирует:**

1. **Бинарный протокол в действии.** Каждое сообщение кодируется в 12-байтный заголовок + payload. Видно, что «привет» превращается в `52 54 01 01 00 00 00 01 00 00 00 06 70 72 69 76 65 74`.
2. **Один кодек — два транспорта.** Меняете транспорт — формат кадра не меняется. Тот же 12-байтный заголовок идёт и через WebSocket, и через WebTransport.
3. **Инкремент seq.** Каждое сообщение получает порядковый номер. Echo возвращает тот же seq — вычисляется RTT.
4. **Frame boundaries.** Если отправить 10 сообщений подряд, echo вернёт их единым куском. `FrameDecoder` разберёт корректно.

## 39.11. Что стоит запомнить

**Правило.** **Транспорт — не выбор «один раз и навсегда».** WebSocket на старте, WebTransport когда потребуется мультиплексирование, WebRTC когда нужен peer-to-peer. Абстракция `Transport` позволяет менять транспорт без переписывания логики.

**Правило.** **Бинарный формат — не «сложнее», а «проще».** 12 байт заголовка против 30 байт JSON. Нулевая стоимость парсинга против парсера JSON. Явная схема против `"field": maybe_number_or_string`.

**Правило.** **Frame boundaries — не проблема транспорта.** TCP/QUIC дают поток байт, не поток сообщений. Декодер обязан накапливать — `FrameDecoder` именно это и делает.

**Правило.** **`seq` — дешёвый, но незаменимый.** Одно 4-байтное поле даёт RTT, сопоставление запрос-ответ, обнаружение пропаж.

**Правило.** **Reliability per-message — самое сильное преимущество QUIC.** WebSocket надёжен и упорядочен всегда. WebTransport позволяет выбрать: отправьте позицию игрока как датаграмму (потеря не страшна), а текстуру — как поток (потеря критична).

**Правило.** **Бинарный протокол работает и поверх WebRTC.** Data channels передают `ArrayBuffer` — тот же кодек подходит без изменений.

## 39.12. Итог

**Три транспорта — три семантики.** WebSocket — одна надёжная упорядоченная труба. WebTransport — мультиплексирование с выбором надёжности. WebRTC — peer-to-peer без сервера.

**Бинарный протокол — минимальный и расширяемый.** 12-байтный заголовок: magic, version, kind, seq, len. Фиксированная длина, нулевая стоимость парсинга. `seq` даёт RTT и сопоставление.

**Транспорт-абстракция — архитектурный минимум.** Код протокола не должен знать про WebSocket. `Transport` инкапсулирует выбор и позволяет менять бэкенд без переписывания.

**Frame boundaries — ответственность приложения.** Транспорт даёт поток байт. `FrameDecoder` собирает из него кадры, переживая частичные приёмы.

**Матрица выбора — на одном экране.** Не надо гуглить «WebSocket vs WebTransport vs WebRTC». Шесть критериев — тип соединения, надёжность, мультиплексирование, задержка, поддержка, сложность setup — дают ответ за 30 секунд.

**Правило архитектуры real-time приложения.** Сначала выберите транспорт по семантике (server / peer). Потом — формат (binary / JSON). Потом — изолируйте транспорт за абстракцией. Только после этого пишите логику.

Следующая тема — **Web Push**: уведомления от сервера через VAPID, без активной вкладки.

## 39.12. Приложение. `app.rs`

```
//! Глава 39: Real-Time Browser Architecture.
//!
//! Один бинарный протокол — два транспорта.
//! Демонстрирует:
//!   1. 12-байтный кадр (magic / version / kind / seq / len / payload).
//!   2. Единый `Transport` поверх WebSocket или WebTransport.
//!   3. Frame inspector: hex + разбор полей.
//!   4. Сравнение размера JSON vs Binary.
//!   5. Round-trip через совпадение seq.
//!   6. Матрицу выбора транспорта по 6 критериям.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: callback захватывает Weak<App> — избегаем Rc-цикла.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use web_sys::{
    HtmlElement, HtmlInputElement, HtmlSelectElement, HtmlTextAreaElement,
};

use crate::format_str::format_str;
use crate::rta_protocol::{
    anatomy, encode, hex_preview, Frame, FrameDecoder, FrameKind,
};
use crate::rta_transport::{
    Transport, TransportEvent, TransportKind, TransportState,
};

const LOG_LIMIT: usize = 25;

const HTML: &str = r#"
<style>
  :root {
    --rta-info: #3498db; --rta-ok: #27ae60; --rta-err: #c0392b; --rta-warn: #e67e22;
    --rta-text: #222; --rta-muted: #666;
    --rta-bg: #fff; --rta-soft: #fafafa; --rta-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --rta-text: #e6e6e6; --rta-muted: #a0a0a0;
      --rta-bg: #1e1e1e; --rta-soft: #262626; --rta-border: #3a3a3a;
    }
    body { background: #121212; }
    select, input, textarea { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .rta-log-entry { background: #1e1e1e; }
    .rta-hint { background: #232014; color: #e6c98c; }
    .rta-hex { background: #1a1a1a; color: #9ed09e; border-color: #333; }
    .rta-matrix th.active { background: #23283a; color: #9fc4ff; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--rta-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .rta-subtitle { color: var(--rta-muted); font-size: 12px; margin: 0 0 12px; }
  .rta-subtitle a { color: var(--rta-info); }

  .rta-grid {
    display: grid; gap: 12px;
    grid-template-columns: minmax(0, 1fr) minmax(0, 1.15fr) minmax(0, 1fr);
  }
  @media (max-width: 1100px) { .rta-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px)  { .rta-grid { grid-template-columns: 1fr; } }
  .rta-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .rta-panel {
    border: 1px solid var(--rta-border); border-radius: 8px;
    padding: 10px 12px; background: var(--rta-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .rta-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px;
  }
  .rta-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--rta-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }

  .rta-row { display: flex; gap: 6px; flex-wrap: wrap; margin: 6px 0; align-items: center; }
  .rta-label { font-size: 11px; color: var(--rta-muted); }

  select, input[type="text"], textarea {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--rta-bg); color: var(--rta-text);
    font-family: ui-monospace, monospace;
    width: 100%;
  }
  textarea { height: 60px; resize: vertical; }
  select { cursor: pointer; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--rta-soft); color: var(--rta-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.rta-primary { background: #eaf4ff; border-color: var(--rta-info); color: #1f4e79; }
  button.rta-success { background: #e8f8ed; border-color: var(--rta-ok); color: #1e8449; }
  button.rta-danger  { background: #fdedec; border-color: var(--rta-err); color: #922b21; }

  .rta-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
    margin-left: auto;
  }
  .rta-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .rta-badge.err { background: #fdedec; color: #922b21; }
  .rta-badge.warn { background: #fff9e6; color: #7d5a00; }

  .rta-hint {
    background: #fff9e6; border-left: 3px solid var(--rta-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 0;
    color: #7d5a00;
  }
  .rta-hint code { background: rgba(0,0,0,0.06); padding: 1px 4px; border-radius: 3px; }
  @media (prefers-color-scheme: dark) { .rta-hint code { background: rgba(255,255,255,0.08); } }

  .rta-size {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--rta-muted); margin-left: auto;
  }

  .rta-frame-empty {
    color: var(--rta-muted); font-style: italic;
    font-family: ui-monospace, monospace; font-size: 11px;
    padding: 4px 0;
  }
  .rta-hex {
    background: #f6f8fa; border: 1px solid var(--rta-border);
    border-radius: 4px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #2d5a2d; overflow-wrap: anywhere;
    margin: 4px 0;
  }
  @media (prefers-color-scheme: dark) {
    .rta-hex { background: #1a1a1a; color: #9ed09e; border-color: #333; }
  }
  .rta-anatomy {
    display: grid; grid-template-columns: auto 1fr;
    gap: 2px 12px;
    font-family: ui-monospace, monospace; font-size: 11px;
    margin-top: 4px;
  }
  .rta-anatomy .k { color: var(--rta-muted); }
  .rta-anatomy .v { color: var(--rta-text); overflow-wrap: anywhere; }
  .rta-anatomy .v .kind { color: var(--rta-info); font-weight: 700; }
  .rta-anatomy .v .payload-text { color: var(--rta-ok); }

  .rta-metrics {
    display: grid; grid-template-columns: auto 1fr;
    gap: 3px 12px;
    font-family: ui-monospace, monospace; font-size: 12px;
  }
  .rta-metrics .k { color: var(--rta-muted); }
  .rta-metrics .v { color: var(--rta-text); }
  .rta-metrics .v.ok { color: var(--rta-ok); }

  .rta-log {
    border: 1px solid var(--rta-border); border-radius: 5px;
    background: var(--rta-soft); max-height: 300px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rta-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .rta-log-entry {
    display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent;
  }
  .rta-log-entry.ok   { border-left-color: var(--rta-ok); }
  .rta-log-entry.err  { border-left-color: var(--rta-err); }
  .rta-log-entry.info { border-left-color: var(--rta-info); }
  .rta-log-entry.warn { border-left-color: var(--rta-warn); }
  .rta-log-entry .t { color: var(--rta-muted); flex-shrink: 0; }
  .rta-log-entry .m { flex: 1; overflow-wrap: anywhere; }
  .rta-log-entry.err .m  { color: #922b21; }
  .rta-log-entry.warn .m { color: #7d5a00; }

  .rta-matrix-panel { margin-top: 12px; }
  .rta-matrix {
    width: 100%; border-collapse: collapse;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rta-matrix th, .rta-matrix td {
    padding: 6px 10px; text-align: left;
    border-bottom: 1px solid var(--rta-border);
    vertical-align: top;
  }
  .rta-matrix th {
    color: var(--rta-muted); font-weight: 700; font-size: 11px;
    background: var(--rta-soft);
  }
  .rta-matrix th.active { background: #eef4ff; color: #1f4e79; }
  @media (prefers-color-scheme: dark) {
    .rta-matrix th.active { background: #23283a; color: #9fc4ff; }
  }
  .rta-matrix td.row-label { color: var(--rta-muted); white-space: nowrap; }
  .rta-matrix td { color: var(--rta-text); }
</style>

<h1>Глава 39: Real-Time Browser Architecture</h1>
<p class="rta-subtitle">
  Один бинарный протокол, два транспорта. WebSocket, WebTransport, WebRTC.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="rta-grid">

  <!-- ============ Колонка 1 ============ -->
  <div class="rta-col">
    <div class="rta-panel">
      <h2 class="rta-panel-title">
        <span class="rta-num">1</span>Транспорт
      </h2>
      <div class="rta-row">
        <select id="rta-transport">
          <option value="websocket">WebSocket — TCP, reliable, ordered</option>
          <option value="webtransport">WebTransport — QUIC, multiplexing</option>
          <option value="webrtc" disabled>WebRTC — peer-to-peer (глава 38)</option>
        </select>
      </div>
      <div class="rta-row">
        <input type="text" id="rta-url" value="wss://echo.websocket.org">
      </div>
      <div class="rta-row">
        <button id="rta-connect" class="rta-success">▶ Connect</button>
        <button id="rta-disconnect" class="rta-danger" disabled>■ Disconnect</button>
        <button id="rta-reset">↺ Reset</button>
        <span class="rta-badge" id="rta-state">idle</span>
      </div>
    </div>

    <div class="rta-hint" id="rta-hint"></div>

    <div class="rta-panel">
      <h2 class="rta-panel-title">
        <span class="rta-num">2</span>Сообщение
      </h2>
      <div class="rta-row">
        <textarea id="rta-msg">Привет из real-time!</textarea>
      </div>
      <div class="rta-row">
        <button id="rta-send" class="rta-primary" disabled>➤ Отправить</button>
        <span class="rta-size" id="rta-size">JSON ≈ 0 B · Binary 0 B</span>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 2: Frame Inspector ============ -->
  <div class="rta-col">
    <div class="rta-panel">
      <h2 class="rta-panel-title">📤 Последний отправленный</h2>
      <div id="rta-sent-frame">
        <div class="rta-frame-empty">—</div>
      </div>
    </div>
    <div class="rta-panel">
      <h2 class="rta-panel-title">📥 Последний полученный</h2>
      <div id="rta-received-frame">
        <div class="rta-frame-empty">—</div>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 3: Метрики + журнал ============ -->
  <div class="rta-col">
    <div class="rta-panel">
      <h2 class="rta-panel-title">Метрики</h2>
      <div id="rta-metrics" class="rta-metrics">
        <span class="k">—</span><span class="v">нет данных</span>
      </div>
    </div>
    <div class="rta-panel">
      <h2 class="rta-panel-title">Журнал</h2>
      <div id="rta-log" class="rta-log">
        <div class="rta-log-empty">здесь появятся события</div>
      </div>
    </div>
  </div>

</div>

<!-- ============ Матрица выбора транспорта ============ -->
<div class="rta-panel rta-matrix-panel">
  <h2 class="rta-panel-title">Матрица выбора транспорта</h2>
  <table class="rta-matrix">
    <thead>
      <tr>
        <th></th>
        <th id="rta-th-websocket">WebSocket</th>
        <th id="rta-th-webtransport">WebTransport</th>
        <th id="rta-th-webrtc">WebRTC</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td class="row-label">Транспорт</td>
        <td>TCP</td>
        <td>QUIC (UDP)</td>
        <td>DTLS/SCTP (UDP)</td>
      </tr>
      <tr>
        <td class="row-label">Соединение</td>
        <td>client → server</td>
        <td>client → server</td>
        <td>peer ↔ peer</td>
      </tr>
      <tr>
        <td class="row-label">Надёжность</td>
        <td>reliable + ordered</td>
        <td>+ unreliable datagrams</td>
        <td>per-channel</td>
      </tr>
      <tr>
        <td class="row-label">Мультиплексирование</td>
        <td>нет (1 поток)</td>
        <td>да (N streams + datagrams)</td>
        <td>да (data channels + media)</td>
      </tr>
      <tr>
        <td class="row-label">Latency (типичная)</td>
        <td>~50–200 мс</td>
        <td>~20–100 мс</td>
        <td>~5–50 мс</td>
      </tr>
      <tr>
        <td class="row-label">Браузеры</td>
        <td>все</td>
        <td>Chrome / Edge / Safari 26.4+</td>
        <td>все</td>
      </tr>
      <tr>
        <td class="row-label">Setup</td>
        <td>минимум</td>
        <td>сертификат + QUIC-сервер</td>
        <td>STUN/TURN + signaling</td>
      </tr>
    </tbody>
  </table>
</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info, Warn }

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

#[derive(Default)]
struct Metrics {
    frames_sent: u64,
    frames_received: u64,
    bytes_sent: u64,
    bytes_received: u64,
    last_rtt_ms: Option<f64>,
}

struct SentInfo {
    bytes: Vec<u8>,
    seq: u32,
    sent_at_ms: f64,
}

struct ReceivedInfo {
    bytes: Vec<u8>,
    rtt_ms: Option<f64>,
}

struct Ui {
    transport_sel: HtmlSelectElement,
    url: HtmlInputElement,
    btn_connect: HtmlElement,
    btn_disconnect: HtmlElement,
    btn_reset: HtmlElement,
    badge_state: web_sys::Element,
    hint: web_sys::Element,

    msg: HtmlTextAreaElement,
    btn_send: HtmlElement,
    size_estimate: web_sys::Element,

    sent_frame: web_sys::Element,
    received_frame: web_sys::Element,

    metrics: web_sys::Element,
    log: web_sys::Element,

    th_ws: web_sys::Element,
    th_wt: web_sys::Element,
    th_rtc: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    transport: RefCell<Option<Rc<Transport>>>,
    decoder: RefCell<FrameDecoder>,
    next_seq: Rc<Cell<u32>>,
    metrics: Rc<RefCell<Metrics>>,
    last_sent: RefCell<Option<SentInfo>>,
    last_received: RefCell<Option<ReceivedInfo>>,
    generation: Rc<Cell<u32>>,
    ui: Ui,
}

// ============================================================
//                    ЗАПУСК
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        transport_sel: get_el("rta-transport")?.dyn_into()?,
        url: get_input("rta-url")?,
        btn_connect: get_el("rta-connect")?.dyn_into()?,
        btn_disconnect: get_el("rta-disconnect")?.dyn_into()?,
        btn_reset: get_el("rta-reset")?.dyn_into()?,
        badge_state: get_el("rta-state")?,
        hint: get_el("rta-hint")?,

        msg: get_el("rta-msg")?.dyn_into()?,
        btn_send: get_el("rta-send")?.dyn_into()?,
        size_estimate: get_el("rta-size")?,

        sent_frame: get_el("rta-sent-frame")?,
        received_frame: get_el("rta-received-frame")?,

        metrics: get_el("rta-metrics")?,
        log: get_el("rta-log")?,

        th_ws: get_el("rta-th-websocket")?,
        th_wt: get_el("rta-th-webtransport")?,
        th_rtc: get_el("rta-th-webrtc")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        transport: RefCell::new(None),
        decoder: RefCell::new(FrameDecoder::new()),
        next_seq: Rc::new(Cell::new(1)),
        metrics: Rc::new(RefCell::new(Metrics::default())),
        last_sent: RefCell::new(None),
        last_received: RefCell::new(None),
        generation: Rc::new(Cell::new(0)),
        ui,
    });

    app.render_log();
    render_state(&app);
    render_hint(&app);
    render_metrics(&app);
    render_frames(&app);
    render_size_estimate(&app);
    render_matrix_highlight(&app);
    update_controls(&app);

    bind_buttons(&app)?;
    bind_live_input(&app)?;

    Ok(())
}

// ============================================================
//                    BIND
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_connect, move || {
            connect(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_disconnect, move || {
            disconnect(&app_c);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_reset, move || {
            do_reset(&app_c);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_send, move || {
            send_message(app_c.clone());
        });
    }
    // Смена транспорта.
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            on_transport_change(app_c.clone());
        });
        app.ui
            .transport_sel
            .add_event_listener_with_callback("change", cb.as_ref().unchecked_ref())?;
        cb.forget();
    }
    Ok(())
}

fn bind_live_input(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        render_size_estimate(&app_c);
    });
    app.ui
        .msg
        .add_event_listener_with_callback("input", cb.as_ref().unchecked_ref())?;
    cb.forget();
    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn current_kind(app: &Rc<App>) -> TransportKind {
    return TransportKind::from_str(&app.ui.transport_sel.value());
}

fn on_transport_change(app: Rc<App>) {
    // При смене транспорта закрываем текущую сессию.
    if app.transport.borrow().is_some() {
        disconnect(&app);
    }
    let kind = current_kind(&app);
    app.ui.url.set_value(kind.default_url());
    render_hint(&app);
    render_matrix_highlight(&app);
    update_controls(&app);
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "транспорт: {} — {}",
            kind.as_str(),
            kind.reliability(),
        )),
    );
}

fn connect(app: Rc<App>) {
    if app.transport.borrow().is_some() {
        return;
    }
    let kind = current_kind(&app);
    if kind == TransportKind::WebRtc {
        set_status(
            &app.ui.badge_state,
            "warn",
            "WebRTC — не в этом демо (глава 45)",
        );
        return;
    }
    let url = app.ui.url.value();
    if url.is_empty() {
        set_status(&app.ui.badge_state, "err", "URL пуст");
        return;
    }

    // Weak<App> — избегаем цикла App → Transport → callback → App.
    let app_weak = Rc::downgrade(&app);
    let on_event: Rc<dyn Fn(TransportEvent)> = Rc::new(move |ev| {
        if let Some(app) = app_weak.upgrade() {
            on_transport_event(app, ev);
        }
    });

    let t = match kind {
        TransportKind::WebSocket => Transport::ws(&url, on_event),
        TransportKind::WebTransport => Transport::wt(&url, on_event),
        TransportKind::WebRtc => unreachable!("проверено выше"),
    };

    // Новое поколение: async-задачи прошлой сессии игнорируются.
    app.generation.set(app.generation.get().wrapping_add(1));
    // Сброс состояния кадров.
    *app.last_sent.borrow_mut() = None;
    *app.last_received.borrow_mut() = None;
    app.decoder.borrow_mut().reset();
    app.next_seq.set(1);
    render_frames(&app);

    *app.transport.borrow_mut() = Some(t.clone());
    app.push_log(
        LogKind::Info,
        format_str(format_args!("connect: {} → {url}", kind.as_str())),
    );
    set_status(&app.ui.badge_state, "warn", "connecting…");
    update_controls(&app);

    if let Err(e) = t.connect() {
        app.push_log(LogKind::Err, e.clone());
        set_status(&app.ui.badge_state, "err", &e);
        *app.transport.borrow_mut() = None;
        update_controls(&app);
    }
}

fn disconnect(app: &Rc<App>) {
    let t = app.transport.borrow_mut().take();
    if let Some(t) = t {
        t.close();
        app.push_log(LogKind::Info, "disconnected".to_string());
    }
    // Сбрасываем decoder — обрыв потока не должен оставить
    // полукадр в буфере для следующей сессии.
    app.decoder.borrow_mut().reset();
    render_state(app);
    update_controls(app);
}

fn do_reset(app: &Rc<App>) {
    disconnect(app);
    app.log.borrow_mut().clear();
    app.render_log();
    *app.metrics.borrow_mut() = Metrics::default();
    *app.last_sent.borrow_mut() = None;
    *app.last_received.borrow_mut() = None;
    app.decoder.borrow_mut().reset();
    app.next_seq.set(1);
    render_frames(app);
    render_metrics(app);
    render_size_estimate(app);
    set_status(&app.ui.badge_state, "idle", "сброшено");
    update_controls(app);
}

fn send_message(app: Rc<App>) {
    let text = app.ui.msg.value();
    if text.is_empty() {
        return;
    }
    let Some(t) = app.transport.borrow().clone() else {
        set_status(&app.ui.badge_state, "err", "не подключено");
        return;
    };
    if t.state() != TransportState::Ready {
        set_status(&app.ui.badge_state, "warn", "не готово — ждём соединения");
        return;
    }

    let seq = app.next_seq.get();
    app.next_seq.set(seq.wrapping_add(1));

    let frame = Frame::text(seq, &text);
    let bytes = match encode(&frame) {
        Ok(b) => b,
        Err(e) => {
            app.push_log(LogKind::Err, e);
            return;
        }
    };

    match t.send_binary(&bytes) {
        Ok(()) => {
            {
                let mut m = app.metrics.borrow_mut();
                m.frames_sent += 1;
                m.bytes_sent += bytes.len() as u64;
            }
            *app.last_sent.borrow_mut() = Some(SentInfo {
                bytes: bytes.clone(),
                seq,
                sent_at_ms: js_sys::Date::now(),
            });
            app.push_log(
                LogKind::Info,
                format_str(format_args!(
                    "→ frame seq={seq} {} B",
                    bytes.len(),
                )),
            );
            render_frames(&app);
            render_metrics(&app);
        }
        Err(e) => {
            app.push_log(LogKind::Err, e.clone());
            set_status(&app.ui.badge_state, "err", &e);
        }
    }
}

// ============================================================
//                    ТРАНСПОРТНЫЕ СОБЫТИЯ
// ============================================================

fn on_transport_event(app: Rc<App>, ev: TransportEvent) {
    match ev {
        TransportEvent::Connected => {
            app.push_log(
                LogKind::Ok,
                format_str(format_args!(
                    "connected ({})",
                    current_kind(&app).as_str(),
                )),
            );
            set_status(&app.ui.badge_state, "ok", "ready");
            update_controls(&app);
        }
        TransportEvent::Disconnected(reason) => {
            app.push_log(
                LogKind::Warn,
                format_str(format_args!("disconnected: {reason}")),
            );
            set_status(&app.ui.badge_state, "err", &reason);
            *app.transport.borrow_mut() = None;
            update_controls(&app);
        }
        TransportEvent::Error(msg) => {
            app.push_log(LogKind::Err, msg.clone());
            push_error_hint(&app, &msg);
            set_status(&app.ui.badge_state, "err", &msg);
        }
        TransportEvent::Bytes(bytes) => {
            // Декодируем весь пришедший кусок.
            // Borrow должен быть коротким: handle_received_frame
            // может захотеть decoder.borrow_mut().reset().
            let results = {
                let mut d = app.decoder.borrow_mut();
                d.push(&bytes)
            };
            for r in results {
                match r {
                    Ok(frame) => handle_received_frame(app.clone(), frame),
                    Err(e) => {
                        app.push_log(LogKind::Err, e.as_str());
                        app.decoder.borrow_mut().reset();
                    }
                }
            }
        }
    }
}

fn handle_received_frame(app: Rc<App>, frame: Frame) {
    // Считаем RTT, если seq совпадает с последним отправленным.
    let sent = app.last_sent.borrow();
    let rtt = sent
        .as_ref()
        .filter(|s| s.seq == frame.seq)
        .map(|s| js_sys::Date::now() - s.sent_at_ms);
    drop(sent);

    if let Some(rtt_ms) = rtt {
        let mut m = app.metrics.borrow_mut();
        m.last_rtt_ms = Some(rtt_ms);
    }

    // Восстанавливаем байты (payload и заголовок) для hex-дампа.
    let bytes = encode(&frame).unwrap_or_default();

    {
        let mut m = app.metrics.borrow_mut();
        m.frames_received += 1;
        m.bytes_received += bytes.len() as u64;
    }

    let short: String = frame.as_text().chars().take(40).collect();
    let rtt_part = match rtt {
        Some(ms) => format_str(format_args!(" · RTT {ms:.1} мс")),
        None => String::new(),
    };
    app.push_log(
        LogKind::Ok,
        format_str(format_args!(
            "← frame seq={} kind={} {short}{rtt_part}",
            frame.seq,
            frame.kind.as_str(),
        )),
    );

    *app.last_received.borrow_mut() = Some(ReceivedInfo {
        bytes,
        rtt_ms: rtt,
    });

    render_frames(&app);
    render_metrics(&app);
}

/// Дописывает подсказку для типичных ошибок.
fn push_error_hint(app: &Rc<App>, msg: &str) {
    let lower = msg.to_lowercase();
    let hint: Option<&str> = if lower.contains("connection rejected")
        || lower.contains("connection refused")
        || lower.contains("failed to establish")
        || lower.contains("connection reset")
    {
        match current_kind(app) {
            TransportKind::WebTransport => Some(
                "WebTransport: (1) echo-сервер из главы 37 запущен на \
                 127.0.0.1:4433; (2) сертификат принят — откройте \
                 https://127.0.0.1:4433 в новой вкладке и нажмите \
                 «Принять»; (3) Chrome / Edge, не Firefox.",
            ),
            _ => Some(
                "WebSocket: проверьте URL (wss:// для HTTPS-страниц). \
                 Публичный echo.websocket.org иногда недоступен — \
                 попробуйте wss://echo.websocket.events.",
            ),
        }
    } else if lower.contains("certificate") || lower.contains("tls") {
        Some(
            "Проблема с TLS-сертификатом сервера. Для WebTransport \
             используйте mkcert (глава 37).",
        )
    } else {
        None
    };
    if let Some(h) = hint {
        app.push_log(LogKind::Warn, h.to_string());
    }
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_state(app: &Rc<App>) {
    let state = app
        .transport
        .borrow()
        .as_ref()
        .map(|t| t.state())
        .unwrap_or(TransportState::Idle);
    let (cls, label) = match state {
        TransportState::Idle => ("warn", "idle"),
        TransportState::Connecting => ("warn", "connecting"),
        TransportState::Ready => ("ok", "ready"),
        TransportState::Closed => ("warn", "closed"),
        TransportState::Failed => ("err", "failed"),
    };
    set_status(&app.ui.badge_state, cls, label);
}

fn render_hint(app: &Rc<App>) {
    let kind = current_kind(app);
    let text = match kind {
        TransportKind::WebSocket => {
            "Публичный echo-сервер. По умолчанию \
             <code>wss://echo.websocket.org</code>. Если не отвечает — \
             <code>wss://echo.websocket.events</code>."
        }
        TransportKind::WebTransport => {
            "Нужен локальный QUIC-сервер из главы 37: \
             <code>cargo run --example hello</code> в папке wtransport. \
             Перед подключением примите сертификат — откройте \
             <code>https://127.0.0.1:4433</code> в новой вкладке."
        }
        TransportKind::WebRtc => {
            "В этом демо WebRTC не реализован. См. главу 45 — \
             там manual signaling между двумя вкладками."
        }
    };
    app.ui.hint.set_inner_html(text);
}

fn render_matrix_highlight(app: &Rc<App>) {
    let kind = current_kind(app);
    for (el, k) in [
        (&app.ui.th_ws, TransportKind::WebSocket),
        (&app.ui.th_wt, TransportKind::WebTransport),
        (&app.ui.th_rtc, TransportKind::WebRtc),
    ] {
        if kind == k {
            el.set_class_name("active");
        } else {
            el.set_class_name("");
        }
    }
}

fn render_frames(app: &Rc<App>) {
    // Отправленный.
    let sent_html = {
        let s = app.last_sent.borrow();
        match s.as_ref() {
            Some(s) => render_frame_html(&s.bytes, None),
            None => "<div class=\"rta-frame-empty\">—</div>".to_string(),
        }
    };
    app.ui.sent_frame.set_inner_html(&sent_html);

    // Полученный.
    let recv_html = {
        let r = app.last_received.borrow();
        match r.as_ref() {
            Some(r) => {
                let note = r.rtt_ms.map(|ms| {
                    format_str(format_args!("← echo, RTT {ms:.1} мс"))
                });
                render_frame_html(&r.bytes, note)
            }
            None => "<div class=\"rta-frame-empty\">—</div>".to_string(),
        }
    };
    app.ui.received_frame.set_inner_html(&recv_html);
}

/// Рендерит один кадр: hex-дамп + таблица полей.
///
/// HTML формируется через push_str — кавычек много, format_args!
/// не подходит.
fn render_frame_html(bytes: &[u8], note: Option<String>) -> String {
    if bytes.is_empty() {
        return "<div class=\"rta-frame-empty\">—</div>".to_string();
    }

    let mut html = String::with_capacity(512);

    // Размер + заметка.
    let size_label = format_str(format_args!("{} байт", bytes.len()));
    html.push_str("<div class=\"rta-row\"><span class=\"rta-label\">");
    html.push_str(&esc_html(&size_label));
    html.push_str("</span>");
    if let Some(n) = note {
        html.push_str("<span class=\"rta-size\">");
        html.push_str(&esc_html(&n));
        html.push_str("</span>");
    }
    html.push_str("</div>");

    // Hex-строка.
    let hex = hex_preview(bytes, 32);
    html.push_str("<div class=\"rta-hex\">");
    html.push_str(&esc_html(&hex));
    html.push_str("</div>");

    // Анатомия.
    if let Some(a) = anatomy(bytes) {
        let magic_hex = format_str(format_args!(
            "{:02x} {:02x}",
            a.magic[0], a.magic[1],
        ));
        let kind_str = FrameKind::from_byte(a.kind_byte)
            .map(|k| k.as_str().to_string())
            .unwrap_or_else(|| "?".to_string());

        html.push_str("<div class=\"rta-anatomy\">");
        push_kv(&mut html, "magic", &magic_hex);
        push_kv(&mut html, "version", &format_str(format_args!("{:02x}", a.version)));

        let kind_v = format_str(format_args!(
            "{:02x} ({kind_str})",
            a.kind_byte,
        ));
        html.push_str("<span class=\"k\">kind</span><span class=\"v\"><span class=\"kind\">");
        html.push_str(&esc_html(&kind_v));
        html.push_str("</span></span>");

        push_kv(
            &mut html,
            "seq",
            &format_str(format_args!("{}", a.seq)),
        );
        push_kv(
            &mut html,
            "len",
            &format_str(format_args!("{} байт", a.len)),
        );

        // Payload с текстовым представлением, если это Text-кадр.
        if a.kind_byte == FrameKind::Text.as_byte() && !a.payload_preview.is_empty() {
            let text_preview = String::from_utf8_lossy(&bytes[12..])
                .chars()
                .take(40)
                .collect::<String>();
            let v_html = format_str(format_args!(
                "{}  <span class=\"payload-text\">«{text_preview}»</span>",
                a.payload_preview,
            ));
            html.push_str("<span class=\"k\">payload</span><span class=\"v\">");
            html.push_str(&esc_html(&v_html));
            html.push_str("</span>");
        } else {
            push_kv(&mut html, "payload", &a.payload_preview);
        }
        html.push_str("</div>");
    }

    return html;
}

fn push_kv(html: &mut String, k: &str, v: &str) {
    html.push_str("<span class=\"k\">");
    html.push_str(&esc_html(k));
    html.push_str("</span><span class=\"v\">");
    html.push_str(&esc_html(v));
    html.push_str("</span>");
}

fn render_size_estimate(app: &Rc<App>) {
    let text = app.ui.msg.value();
    let text_bytes = text.len();
    let binary_bytes = 12 + text_bytes; // HEADER_LEN + payload
    // Грубая оценка JSON: {"seq":N,"text":"..."} — 24 симв. каркаса + контент.
    let json_bytes = 24 + text_bytes;
    let savings = if json_bytes > 0 {
        ((json_bytes as f64 - binary_bytes as f64) / json_bytes as f64) * 100.0
    } else {
        0.0
    };
    let label = format_str(format_args!(
        "JSON ≈ {json_bytes} B · Binary {binary_bytes} B · −{savings:.0}%",
    ));
    app.ui.size_estimate.set_text_content(Some(&label));
}

fn render_metrics(app: &Rc<App>) {
    let m = app.metrics.borrow();
    let mut html = String::with_capacity(256);
    push_metric(&mut html, "frames sent", &format_str(format_args!("{}", m.frames_sent)));
    push_metric(&mut html, "frames recv", &format_str(format_args!("{}", m.frames_received)));
    push_metric(&mut html, "bytes sent", &format_str(format_args!("{} B", m.bytes_sent)));
    push_metric(&mut html, "bytes recv", &format_str(format_args!("{} B", m.bytes_received)));
    match m.last_rtt_ms {
        Some(ms) => {
            html.push_str("<span class=\"k\">last RTT</span>");
            html.push_str("<span class=\"v ok\">");
            html.push_str(&esc_html(&format_str(format_args!("{ms:.1} мс"))));
            html.push_str("</span>");
        }
        None => push_metric(&mut html, "last RTT", "—"),
    }
    drop(m);
    app.ui.metrics.set_inner_html(&html);
}

fn push_metric(html: &mut String, k: &str, v: &str) {
    html.push_str("<span class=\"k\">");
    html.push_str(&esc_html(k));
    html.push_str("</span><span class=\"v\">");
    html.push_str(&esc_html(v));
    html.push_str("</span>");
}

fn update_controls(app: &Rc<App>) {
    let t_opt = app.transport.borrow().clone();
    let (has_transport, is_ready) = match t_opt.as_ref() {
        Some(t) => (true, t.state() == TransportState::Ready),
        None => (false, false),
    };
    set_disabled(&app.ui.btn_connect, has_transport);
    set_disabled(&app.ui.btn_disconnect, !has_transport);
    set_disabled(&app.ui.btn_send, !is_ready);
    // Селект транспорта — не блокируем. При смене автоматически
    // закроется текущая сессия (см. on_transport_change).
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "rta-badge ok",
        "err" => "rta-badge err",
        "warn" => "rta-badge warn",
        _ => "rta-badge",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn set_disabled(el: &HtmlElement, disabled: bool) {
    if disabled {
        let _ = el.set_attribute("disabled", "");
    } else {
        let _ = el.remove_attribute("disabled");
    }
}

// ============================================================
//                    LOG
// ============================================================

fn push_log_static(log: &Rc<RefCell<Vec<LogEntry>>>, kind: LogKind, text: String) {
    let time = fmt_now_hms();
    let mut v = log.borrow_mut();
    v.push(LogEntry { time, text, kind });
    if v.len() > LOG_LIMIT {
        let drop_n = v.len() - LOG_LIMIT;
        v.drain(0..drop_n);
    }
}

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        push_log_static(&self.log, kind, text);
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"rta-log-empty\">здесь появятся события</div>",
            );
            return;
        }
        let mut html = String::with_capacity(512);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
                LogKind::Warn => "warn",
            };
            html.push_str("<div class=\"rta-log-entry ");
            html.push_str(cls);
            html.push_str("\"><span class=\"t\">");
            html.push_str(&esc_html(&e.time));
            html.push_str("</span><span class=\"m\">");
            html.push_str(&esc_html(&e.text));
            html.push_str("</span></div>");
        }
        self.ui.log.set_inner_html(&html);
        self.ui
            .log
            .set_scroll_top(self.ui.log.scroll_height() as f64);
    }
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(),
        d.get_minutes(),
        d.get_seconds(),
    ));
}

fn esc_html(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    return web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")));
}

fn get_input(id: &str) -> Result<HtmlInputElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не input")));
}

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut() + 'static,
{
    let cb = Closure::<dyn FnMut()>::new(f);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```