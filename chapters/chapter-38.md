# Глава 45. WebRTC

`WebRTC` — единственная технология в Web Platform, позволяющая **двум браузерам соединиться напрямую**, минуя сервер. После установки соединения голос, видео и произвольные данные идут peer-to-peer, сервер нужен только для первоначального «рукопожатия». Это фундаментально отличает WebRTC от WebSocket, WebTransport, Web Push — все они работают по модели клиент-сервер. WebRTC — первая децентрализованная технология в браузере.

За мощью скрывается цена. WebRTC — самый сложный API в Web Platform. Три независимых слоя: **signaling** (как стороны находят друг друга), **ICE** (как пробить NAT), **SDP** (что именно стороны будут передавать). Все три нужно понять, чтобы получить рабочее соединение. И один из них — signaling — **вообще не входит в WebRTC API**.

Глава объясняет каждый слой и заканчивается рабочим демо: две вкладки браузера соединяются вручную через copy-paste, обмениваются SDP, устанавливают peer-to-peer соединение и общаются через `RTCDataChannel`.

## 45.1. Три слоя WebRTC

```text
┌────────────────────────────────────────────────────────────────┐
│  SIGNALING  — как стороны обмениваются метаданными             │
│  Не входит в WebRTC API! Выбор за разработчиком:               │
│  WebSocket, HTTP, BroadcastChannel, ручной обмен.              │
├────────────────────────────────────────────────────────────────┤
│  ICE        — как пробить NAT и найти путь                     │
│  STUN-сервер: «какой у меня публичный IP?»                     │
│  TURN-сервер: «перешли пакеты, прямое соединение невозможно»   │
├────────────────────────────────────────────────────────────────┤
│  SDP        — что именно стороны будут передавать              │
│  Кодеки, media tracks, data channels, DTLS fingerprints        │
└────────────────────────────────────────────────────────────────┘
```

**Signaling — не часть WebRTC.** В спецификации нет API для сигналинга. Это осознанно: разные приложения используют разные транспорты. Чат — WebSocket. Игровое лобби — HTTP long-polling. Быстрое демо — `BroadcastChannel` между вкладками. Мы в главе используем **manual copy-paste**: две вкладки обмениваются SDP через буфер обмена, вручную. Это самый чистый способ показать, что signaling — отдельная задача, решаемая приложением.

**ICE (Interactive Connectivity Establishment)** — протокол, который перебирает все возможные пути и находит работающий:

1. **Host candidates** — локальные IP-адреса (`192.168.1.5`). Работают в одной сети.
2. **Server-reflexive (srflx)** — публичный IP, полученный от **STUN-сервера**. Работают через NAT.
3. **Relay candidates** — адреса **TURN-сервера**. Работают всегда, но трафик идёт через посредника.

Каждая сторона генерирует кандидатов и отправляет их через signaling собеседнику. Оба набирают пул парных соединений и одновременно пробуют — кто первый ответит, тот и используется.

**SDP (Session Description Protocol)** — текстовый формат описания медиа-сессии. Содержит кодеки, media tracks, data channels, DTLS-fingerprint. Три типа SDP: **offer** (что я умею), **answer** (что я поддерживаю), **pranswer** (промежуточный, редко).

## 45.2. Обмен SDP: offer / answer

```text
A: createOffer → setLocalDescription(offer)
A → B: SDP offer
B: setRemoteDescription(offer) → createAnswer → setLocalDescription(answer)
B → A: SDP answer
A: setRemoteDescription(answer)
```

После этого обе стороны начинают ICE handshake. Параллельно с SDP.

## 45.3. Vanilla ICE: ждём всё в SDP

Два подхода к передаче ICE-кандидатов:

**Trickle ICE.** Каждый кандидат отправляется через signaling **по мере генерации**. Быстрее, но требует постоянного канала signaling. Используется в production.

**Vanilla ICE.** Дожидаемся `iceGatheringState === "complete"` — **все кандидаты уже внутри SDP**. Медленнее на 1–3 секунды, но весь SDP передаётся **одним куском**. Идеален для copy-paste.

Мы используем vanilla: в signaling-сообщении нет отдельных ICE-пакетов, только SDP.

## 45.4. Data channels

После установки соединения через SDP и ICE открывается транспорт. Поверх него — два вида данных: **media tracks** (аудио/видео, из главы 35) и **data channels** (произвольные данные). Мы используем data channels — это самое простое и наглядное.

У `RTCDataChannel` три режима доставки:

| Режим | Гарантии |
|---|---|
| `reliable + ordered` | Как TCP (по умолчанию) |
| `reliable + unordered` | Гарантированно, порядок не важен |
| `unreliable` | Как UDP — максимум попыток, без гарантий |

Для демо используем `reliable + ordered` — простейший случай.

## 45.5. Жизненный цикл соединения

Два независимых state machine одновременно:

**`RTCPeerConnection.connectionState`:**

```text
new → connecting → connected → disconnected → failed → closed
```

**`RTCPeerConnection.iceConnectionState`:**

```text
new → checking → connected → completed → disconnected → failed → closed
```

`connectionState` — общий статус. `iceConnectionState` — статус именно ICE. Они коррелируют, но не совпадают: `connectionState = connected` наступает после `iceConnectionState = connected` или `completed`. Для UI достаточно `connectionState`, для диагностики полезны оба.

## 45.6. Структура проекта

```text
src/
  rtc_peer.rs                   ← НОВЫЙ модуль главы 45
  app.rs                        ← демо (заменяется целиком)
tests/
  wasm_rtc_peer.rs              ← тесты главы
src/lib.rs                      ← +1 строка: pub mod rtc_peer;
Cargo.toml                      ← +12 фич web-sys
```

**Ссылки для изучения:**

- MDN: [`RTCPeerConnection`](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection)
- MDN: [`RTCDataChannel`](https://developer.mozilla.org/en-US/docs/Web/API/RTCDataChannel)
- W3C: [WebRTC 1.0 Specification](https://w3c.github.io/webrtc-pc/)
- WebRTC for the Curious: [Signaling](https://webrtcforthecurious.com/docs/02-signaling/)

**Префикс CSS** — `rtc-`, не конфликтует с `gl-`, `ms-`, `cd-`, `aw-`, `md-`, `ws-`, `wt-`.

**Cargo.toml** — добавьте в `[dependencies.web-sys]`:

```toml
    # WebRTC (глава 45)
    "RtcPeerConnection",
    "RtcPeerConnectionState",
    "RtcIceConnectionState",
    "RtcIceGatheringState",
    "RtcSessionDescription",
    "RtcSessionDescriptionInit",
    "RtcSdpType",
    "RtcIceCandidate",
    "RtcIceCandidateInit",
    "RtcConfiguration",
    "RtcIceServer",
    "RtcDataChannel",
    "RtcDataChannelInit",
    "RtcDataChannelEvent",
    "RtcDataChannelEventInit",
    "RtcPeerConnectionIceEvent",
```

## 45.7. Модуль `rtc_peer.rs`

Модуль — обёртка над `RTCPeerConnection`. Все dictionary-типы (`RtcConfiguration`, `RtcSessionDescriptionInit`, `RtcIceCandidateInit`) создаются через `js_sys::Object` + `Reflect::set` и приводятся через `unchecked_into`. Это работает независимо от версии web-sys.

```rust
//! 45. WebRTC peer connection wrapper.
//!
//! # Особенности web-sys
//!
//! • `create_offer`, `create_answer`, `set_*_description`,
//!   `add_ice_candidate*` возвращают `Promise` напрямую (без
//!   `Result<Promise>`). Знак `?` к ним не применяется.
//! • `RtcConfiguration`, `RtcSessionDescriptionInit`,
//!   `RtcIceCandidateInit` — dictionary-типы. Создаём их через
//!   `js_sys::Object` + `Reflect::set`, конвертируем через
//!   `unchecked_into`. `dyn_into` для dictionary не работает.
//! • Поле `readyState` читаем через `Reflect`, потому что
//!   фича `RtcDataChannelState` не подключена в Cargo.toml.

use gloo_timers::future::TimeoutFuture;
use js_sys::{Array, Object, Reflect};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;
use web_sys::{
    RtcDataChannel, RtcDataChannelEvent, RtcIceCandidate,
    RtcPeerConnection, RtcPeerConnectionIceEvent,
};

// ... полный код модуля — приведён в разделе «Приложение» ...
```

Ключевые публичные функции:

- `create_peer_connection(cfg)` — создаёт peer с конфигурацией ICE-серверов.
- `create_offer(pc)` / `create_answer(pc)` — async, возвращают SDP-init.
- `set_local_description(pc, desc)` / `set_remote_description(pc, desc)` — async.
- `get_local_description(pc)` — читает SDP после `setLocalDescription`.
- `wait_ice_gathering_complete(pc)` — ждёт `iceGatheringState === "complete"`.
- `sdp_to_json_string(desc)` / `sdp_from_json_string(str)` / `sdp_type(desc)` — упаковка/распаковка SDP для copy-paste.
- `create_data_channel(pc, label)` — создаёт канал.
- `data_channel_send(dc, text)` / `data_channel_state(dc)` / `data_channel_label(dc)`.
- `on_ice_candidate(pc, cb)` / `on_data_channel(pc, cb)` / `on_connection_state_change(pc, cb)` / `on_ice_connection_state_change(pc, cb)` / `on_data_channel_open(dc, cb)` / `on_data_channel_message(dc, cb)` — обработчики событий.
- `close_peer_connection(pc)` / `close_data_channel(dc)`.

## 45.8. Тесты `tests/wasm_rtc_peer.rs`

15 тестов, все в headless Chrome:

- `wasm_rtc_is_supported` — API доступен.
- `wasm_rtc_state_roundtrip_known` / `wasm_rtc_state_unknown` / `wasm_rtc_state_terminal` — состояние connection.
- `wasm_ice_state_roundtrip_known` / `wasm_ice_state_unknown` — состояние ICE.
- `wasm_peer_config_loopback_empty` / `wasm_peer_config_with_stun_has_google` — конфигурация.
- `wasm_create_loopback_peer` — создание peer с начальным состоянием `new`.
- `wasm_create_peer_with_stun` — с STUN-сервером.
- `wasm_create_data_channel` — создание канала, начальное состояние `connecting`.
- `wasm_create_offer_produces_sdp` — offer содержит `v=0`.
- `wasm_offer_answer_roundtrip` — полный цикл SDP между двумя peer'ами в одной вкладке.

**Команды запуска:**

```bash
# Проверка компиляции
cargo check --target wasm32-unknown-unknown

# Тесты главы 45
wasm-pack test --headless --chrome --test wasm_rtc_peer

# Демо
npm run wasm:dev && npm start
```

## 45.9. Демо-пример: общение между двумя вкладками

Демо реализует **manual signaling**: два независимых экземпляра приложения на разных портах (`:5173` и `:5174`) договариваются об SDP через буфер обмена. Каждая вкладка может стать либо **инициатором**, либо **ответчиком**.

**Почему разные порты — это разные origins.** `BroadcastChannel` и `localStorage` между `localhost:5173` и `localhost:5174` **не работают** — браузер считает их разными источниками. Поэтому единственный способ без сервера — copy-paste.

**Сценарий:**

1. Открыть `http://127.0.0.1:5173` и `http://127.0.0.1:5174` в двух вкладках.
2. **Вкладка 1:** нажать «Создать offer». Появится SDP, badge `offer ✓`.
3. **Вкладка 1:** нажать «📋 Копировать SDP» — кнопка на 1.8 сек покажет «✓ Скопировано».
4. **Вкладка 2:** вставить в «Входящий SDP», badge покажет `offer ✓`. Нажать «Применить входящий SDP».
5. **Вкладка 2:** роль автоматически меняется на «ответчик». Нажать «Создать answer». Скопировать SDP.
6. **Вкладка 1:** вставить в «Входящий SDP» (`answer ✓`), нажать «Применить».
7. Через 1–2 секунды в обеих вкладках `connectionState → connected`, `dataChannel → open`. Обмен сообщениями работает.

**Что показывает UI:**

- **Панель состояния** — `role`, `connectionState`, `iceConnectionState`, `dataChannel`, `uptime`.
- **DataChannel «chat»** — поле ввода и история сообщений (стрелки `→` и `←`).
- **Подсказка о DataChannel** — контекстное сообщение, зависящее от состояния и роли: «Ожидаем answer от собеседника» (для инициатора), «Peer создан, но DataChannel ещё не открыт» (для ответчика), «Канал открыт — можно отправлять сообщения».
- **Метрики** — ICE-кандидаты по типам (`host`, `srflx`, `relay`), сообщения, байты, размеры SDP.
- **Журнал** — все события с временными метками.
- **Live-badges** — валидация SDP обновляется по мере ввода: `offer ✓`, `answer ✓`, `не JSON`, `нет type`.

## 45.10. Production: что добавить

Loopback и manual signaling работают без инфраструктуры, но это учебные режимы. Для **реального соединения между двумя браузерами** нужно:

**1. Signaling-сервер.** Посредник, через который стороны обмениваются SDP и ICE-кандидатами до установки прямого соединения. Минимальные требования: двусторонний канал (WebSocket), идентификация peer'ов (комнаты/логины), пересылка трёх типов сообщений (`offer`, `answer`, `ice`). Такой сервер на Rust + `tokio-tungstenite` занимает ~80 строк.

**2. STUN-сервер.** Публичные адреса для NAT-обхода. Google предоставляет бесплатно: `stun:stun.l.google.com:19302`. Без STUN работает только loopback и локальные сети.

**3. TURN-сервер.** Для случаев, когда прямое соединение невозможно (симметричный NAT, корпоративный файрвол). Open-source реализация — `coturn`. Около 15–20% соединений в публичной сети требуют TURN.

**4. Trickle ICE.** В production используйте trickle, не vanilla: каждая сторона отправляет кандидатов через signaling по мере генерации. Соединение устанавливается на 1–3 секунды быстрее.

**5. HTTPS.** WebRTC требует secure context. `localhost` — исключение, для реального домена обязателен сертификат.

## 45.11. Сравнение подходов signaling

| Подход | Сфера | Требует |
|---|---|---|
| **Loopback** | Одна вкладка | Ничего |
| **Manual copy-paste** | Любые вкладки, любые порты | Только буфер обмена |
| **BroadcastChannel** | Тот же origin | Ничего |
| **WebSocket-сервер** | Любые браузеры, любые машины | Сервер + STUN (TURN для 15–20%) |

Manual signaling — не production-решение, но **самый педагогически ценный способ** показать, что signaling не входит в WebRTC API. Все остальные подходы скрывают эту истину за инфраструктурой.

## 45.12. Итог

**WebRTC — самый сложный API в Web Platform.** Три независимых слоя: signaling, ICE, SDP. Каждый нужно понять.

**Signaling — не часть WebRTC.** Выбор транспорта за разработчиком. WebSocket — самый частый, manual copy-paste — самый наглядный.

**ICE пробивает NAT.** Host, srflx (через STUN), relay (через TURN). STUN нужен почти всегда, TURN — для 15–20% случаев.

**Vanilla ICE vs trickle.** Vanilla ждёт полного SDP, передаёт одним куском. Trickle отправляет кандидатов по одному. Для copy-paste необходим vanilla.

**Data channels — независимые каналы данных.** Reliable+ordered по умолчанию, можно настроить unreliable.

**Жизненный цикл — два state machine.** `connectionState` (общий) и `iceConnectionState` (ICE).

**Manual signaling работает без сервера.** Два независимых клиента обмениваются SDP через буфер обмена. Единственный сценарий WebRTC, не требующий инфраструктуры.

**Production требует:**
- Signaling-сервер (WebSocket).
- STUN — обязателен.
- TURN — для 15–20% случаев.
- HTTPS — обязателен.
- Trickle ICE — для скорости.

Следующая тема — **Web Push**: уведомления от сервера, когда браузер закрыт.

---

# Приложение. Листинги

## A.1. `src/rtc_peer.rs`

```rust
//! 45. WebRTC peer connection wrapper.
//!
//! Обёртка над RTCPeerConnection:
//!   • создание peer с конфигурацией ICE-серверов;
//!   • offer/answer через SDP;
//!   • ICE candidates — обработка и обмен;
//!   • data channels — создание, приём, отправка;
//!   • lifecycle — connectionState и iceConnectionState.
//!
//! # Особенности web-sys
//!
//! • `create_offer`, `create_answer`, `set_*_description`,
//!   `add_ice_candidate*` возвращают `Promise` напрямую (без
//!   `Result<Promise>`). Знак `?` к ним не применяется.
//! • `RtcConfiguration`, `RtcSessionDescriptionInit`,
//!   `RtcIceCandidateInit` — dictionary-типы. Создаём их через
//!   `js_sys::Object` + `Reflect::set`, конвертируем через
//!   `unchecked_into`. `dyn_into` для dictionary не работает.
//! • Поле `readyState` читаем через `Reflect`, потому что
//!   фича `RtcDataChannelState` не подключена в Cargo.toml.

use js_sys::{Array, Object, Reflect};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;
use web_sys::{
    RtcDataChannel, RtcDataChannelEvent, RtcIceCandidate,
    RtcPeerConnection, RtcPeerConnectionIceEvent,
};

// ============================================================
//                    СОСТОЯНИЯ
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum RtcState {
    New,
    Connecting,
    Connected,
    Disconnected,
    Failed,
    Closed,
    Unknown,
}

impl RtcState {
    pub fn from_str(s: &str) -> Self {
        return match s {
            "new" => RtcState::New,
            "connecting" => RtcState::Connecting,
            "connected" => RtcState::Connected,
            "disconnected" => RtcState::Disconnected,
            "failed" => RtcState::Failed,
            "closed" => RtcState::Closed,
            _ => RtcState::Unknown,
        };
    }
    pub fn as_str(self) -> &'static str {
        return match self {
            RtcState::New => "new",
            RtcState::Connecting => "connecting",
            RtcState::Connected => "connected",
            RtcState::Disconnected => "disconnected",
            RtcState::Failed => "failed",
            RtcState::Closed => "closed",
            RtcState::Unknown => "unknown",
        };
    }
    pub fn is_terminal(self) -> bool {
        return matches!(self, RtcState::Closed | RtcState::Failed);
    }
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum IceState {
    New,
    Checking,
    Connected,
    Completed,
    Disconnected,
    Failed,
    Closed,
    Unknown,
}

impl IceState {
    pub fn from_str(s: &str) -> Self {
        return match s {
            "new" => IceState::New,
            "checking" => IceState::Checking,
            "connected" => IceState::Connected,
            "completed" => IceState::Completed,
            "disconnected" => IceState::Disconnected,
            "failed" => IceState::Failed,
            "closed" => IceState::Closed,
            _ => IceState::Unknown,
        };
    }
    pub fn as_str(self) -> &'static str {
        return match self {
            IceState::New => "new",
            IceState::Checking => "checking",
            IceState::Connected => "connected",
            IceState::Completed => "completed",
            IceState::Disconnected => "disconnected",
            IceState::Failed => "failed",
            IceState::Closed => "closed",
            IceState::Unknown => "unknown",
        };
    }
}

// ============================================================
//                    КОНФИГУРАЦИЯ
// ============================================================

#[derive(Clone, Debug)]
pub struct IceServer {
    pub urls: Vec<String>,
    pub username: Option<String>,
    pub credential: Option<String>,
}

#[derive(Clone, Debug, Default)]
pub struct PeerConfig {
    pub ice_servers: Vec<IceServer>,
}

impl PeerConfig {
    /// Для loopback (одна вкладка, два peer'а) — без STUN.
    pub fn loopback() -> Self {
        return Self { ice_servers: vec![] };
    }
    /// С публичным Google STUN. Для соединения через интернет.
    pub fn with_stun() -> Self {
        return Self {
            ice_servers: vec![IceServer {
                urls: vec!["stun:stun.l.google.com:19302".into()],
                username: None,
                credential: None,
            }],
        };
    }
}

fn build_configuration(cfg: &PeerConfig) -> Result<JsValue, JsValue> {
    let config = Object::new();
    let servers = Array::new();
    for s in &cfg.ice_servers {
        let server = Object::new();
        let urls = Array::new();
        for u in &s.urls {
            urls.push(&JsValue::from_str(u));
        }
        Reflect::set(&server, &JsValue::from_str("urls"), &urls)?;
        if let Some(u) = &s.username {
            Reflect::set(
                &server,
                &JsValue::from_str("username"),
                &JsValue::from_str(u),
            )?;
        }
        if let Some(c) = &s.credential {
            Reflect::set(
                &server,
                &JsValue::from_str("credential"),
                &JsValue::from_str(c),
            )?;
        }
        servers.push(&server);
    }
    Reflect::set(&config, &JsValue::from_str("iceServers"), &servers)?;
    Reflect::set(
        &config,
        &JsValue::from_str("iceTransportPolicy"),
        &JsValue::from_str("all"),
    )?;
    return Ok(config.into());
}

pub fn create_peer_connection(cfg: &PeerConfig) -> Result<RtcPeerConnection, JsValue> {
    let config_js = build_configuration(cfg)?;
    // RtcConfiguration — dictionary. dyn_into не работает,
    // используем unchecked_into.
    let config: web_sys::RtcConfiguration = config_js.unchecked_into();
    return RtcPeerConnection::new_with_configuration(&config);
}

// ============================================================
//                    SDP: OFFER / ANSWER
// ============================================================

pub fn get_local_description(pc: &RtcPeerConnection) -> Result<JsValue, JsValue> {
    let desc = pc
        .local_description()
        .ok_or_else(|| JsValue::from_str("no local description"))?;
    let desc_js: &JsValue = desc.unchecked_ref();
    let sdp = Reflect::get(desc_js, &JsValue::from_str("sdp"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("no sdp"))?;
    let typ = Reflect::get(desc_js, &JsValue::from_str("type"))?
        .as_string()
        .ok_or_else(|| JsValue::from_str("no type"))?;
    let init = Object::new();
    Reflect::set(&init, &JsValue::from_str("type"), &JsValue::from_str(&typ))?;
    Reflect::set(&init, &JsValue::from_str("sdp"), &JsValue::from_str(&sdp))?;
    return Ok(init.into());
}

pub async fn create_offer(pc: &RtcPeerConnection) -> Result<JsValue, JsValue> {
    // create_offer возвращает Promise напрямую.
    let promise = pc.create_offer();
    let result = JsFuture::from(promise).await?;
    return Ok(result);
}

pub async fn create_answer(pc: &RtcPeerConnection) -> Result<JsValue, JsValue> {
    let promise = pc.create_answer();
    let result = JsFuture::from(promise).await?;
    return Ok(result);
}

pub async fn set_local_description(
    pc: &RtcPeerConnection,
    desc: &JsValue,
) -> Result<(), JsValue> {
    // RtcSessionDescriptionInit — dictionary. unchecked_into.
    let init: web_sys::RtcSessionDescriptionInit = desc.clone().unchecked_into();
    // Метод возвращает Promise напрямую.
    let promise = pc.set_local_description(&init);
    JsFuture::from(promise).await?;
    return Ok(());
}

pub async fn set_remote_description(
    pc: &RtcPeerConnection,
    desc: &JsValue,
) -> Result<(), JsValue> {
    let init: web_sys::RtcSessionDescriptionInit = desc.clone().unchecked_into();
    let promise = pc.set_remote_description(&init);
    JsFuture::from(promise).await?;
    return Ok(());
}

// ============================================================
//                    ICE CANDIDATES
// ============================================================

pub fn candidate_to_init(c: &RtcIceCandidate) -> Result<JsValue, JsValue> {
    let init = Object::new();
    let c_js: &JsValue = c.unchecked_ref();
    for key in ["candidate", "sdpMid", "usernameFragment"] {
        if let Ok(v) = Reflect::get(c_js, &JsValue::from_str(key)) {
            Reflect::set(&init, &JsValue::from_str(key), &v)?;
        }
    }
    if let Ok(v) = Reflect::get(c_js, &JsValue::from_str("sdpMLineIndex")) {
        Reflect::set(&init, &JsValue::from_str("sdpMLineIndex"), &v)?;
    }
    return Ok(init.into());
}

/// Форматирует кандидата для лога: `typ/protocol`.
pub fn candidate_summary(c: &RtcIceCandidate) -> String {
    let c_js: &JsValue = c.unchecked_ref();
    let s = Reflect::get(c_js, &JsValue::from_str("candidate"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    let typ = extract_token(&s, "typ").unwrap_or_else(|| "?".into());
    let proto = extract_token(&s, "protocol").unwrap_or_else(|| "?".into());
    return format!("{typ}/{proto}");
}

fn extract_token(s: &str, key: &str) -> Option<String> {
    let parts: Vec<&str> = s.split_whitespace().collect();
    for i in 0..parts.len().saturating_sub(1) {
        if parts[i] == key {
            return Some(parts[i + 1].to_string());
        }
    }
    return None;
}

pub fn add_ice_candidate(pc: &RtcPeerConnection, init: &JsValue) -> Result<(), JsValue> {
    let cand_init: web_sys::RtcIceCandidateInit = init.clone().unchecked_into();
    // Метод возвращает Promise напрямую.
    let promise = pc.add_ice_candidate_with_opt_rtc_ice_candidate_init(Some(&cand_init));
    // Fire-and-forget.
    wasm_bindgen_futures::spawn_local(async move {
        let _ = JsFuture::from(promise).await;
    });
    return Ok(());
}

// ============================================================
//                    DATA CHANNELS
// ============================================================

pub fn create_data_channel(
    pc: &RtcPeerConnection,
    label: &str,
) -> Result<RtcDataChannel, JsValue> {
    // create_data_channel возвращает RtcDataChannel напрямую.
    return Ok(pc.create_data_channel(label));
}

pub fn data_channel_state(dc: &RtcDataChannel) -> &'static str {
    // RtcDataChannelState не подключён в Cargo.toml —
    // читаем строку через Reflect.
    let s = Reflect::get(
        dc.unchecked_ref::<JsValue>(),
        &JsValue::from_str("readyState"),
    )
    .ok()
    .and_then(|v| v.as_string())
    .unwrap_or_default();
    return match s.as_str() {
        "connecting" => "connecting",
        "open" => "open",
        "closing" => "closing",
        "closed" => "closed",
        _ => "unknown",
    };
}

pub fn data_channel_send(dc: &RtcDataChannel, text: &str) -> Result<(), JsValue> {
    if data_channel_state(dc) != "open" {
        return Err(JsValue::from_str("data channel not open"));
    }
    return dc.send_with_str(text);
}

pub fn data_channel_label(dc: &RtcDataChannel) -> String {
    return dc.label();
}

// ============================================================
//                    EVENT LISTENERS
// ============================================================

pub fn on_ice_candidate<F>(pc: &RtcPeerConnection, mut cb: F) -> Result<(), JsValue>
where
    F: FnMut(Option<RtcIceCandidate>) + 'static,
{
    let closure = Closure::<dyn FnMut(RtcPeerConnectionIceEvent)>::new(
        move |ev: RtcPeerConnectionIceEvent| {
            cb(ev.candidate());
        },
    );
    pc.set_onicecandidate(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    return Ok(());
}

pub fn on_data_channel<F>(pc: &RtcPeerConnection, mut cb: F) -> Result<(), JsValue>
where
    F: FnMut(RtcDataChannel) + 'static,
{
    let closure = Closure::<dyn FnMut(RtcDataChannelEvent)>::new(
        move |ev: RtcDataChannelEvent| {
            cb(ev.channel());
        },
    );
    pc.set_ondatachannel(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    return Ok(());
}

pub fn on_connection_state_change<F>(
    pc: &RtcPeerConnection,
    mut cb: F,
) -> Result<(), JsValue>
where
    F: FnMut(RtcState) + 'static,
{
    let pc_clone = pc.clone();
    let closure = Closure::<dyn FnMut(web_sys::Event)>::new(
        move |_ev: web_sys::Event| {
            let s = Reflect::get(
                pc_clone.unchecked_ref::<JsValue>(),
                &JsValue::from_str("connectionState"),
            )
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| "unknown".into());
            cb(RtcState::from_str(&s));
        },
    );
    pc.set_onconnectionstatechange(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    return Ok(());
}

pub fn on_ice_connection_state_change<F>(
    pc: &RtcPeerConnection,
    mut cb: F,
) -> Result<(), JsValue>
where
    F: FnMut(IceState) + 'static,
{
    let pc_clone = pc.clone();
    let closure = Closure::<dyn FnMut(web_sys::Event)>::new(
        move |_ev: web_sys::Event| {
            let s = Reflect::get(
                pc_clone.unchecked_ref::<JsValue>(),
                &JsValue::from_str("iceConnectionState"),
            )
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| "unknown".into());
            cb(IceState::from_str(&s));
        },
    );
    pc.set_oniceconnectionstatechange(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    return Ok(());
}

pub fn on_data_channel_message<F>(
    dc: &RtcDataChannel,
    mut cb: F,
) -> Result<(), JsValue>
where
    F: FnMut(String) + 'static,
{
    let closure = Closure::<dyn FnMut(web_sys::MessageEvent)>::new(
        move |ev: web_sys::MessageEvent| {
            if let Some(text) = ev.data().as_string() {
                cb(text);
            }
        },
    );
    dc.set_onmessage(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    return Ok(());
}

pub fn on_data_channel_open<F>(
    dc: &RtcDataChannel,
    mut cb: F,
) -> Result<(), JsValue>
where
    F: FnMut() + 'static,
{
    let closure = Closure::<dyn FnMut(web_sys::Event)>::new(
        move |_ev: web_sys::Event| {
            cb();
        },
    );
    dc.set_onopen(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    return Ok(());
}

// ============================================================
//                    CLOSE
// ============================================================

pub fn close_peer_connection(pc: &RtcPeerConnection) {
    let _ = pc.close();
}

pub fn close_data_channel(dc: &RtcDataChannel) {
    let _ = dc.close();
}

// ============================================================
//                    FEATURE DETECTION
// ============================================================

pub fn is_supported() -> bool {
    let global = js_sys::global();
    return Reflect::get(&global, &JsValue::from_str("RTCPeerConnection"))
        .map(|v| v.is_function())
        .unwrap_or(false);
}

use gloo_timers::future::TimeoutFuture;

/// Ждёт, пока iceGatheringState станет "complete".
///
/// Используется в vanilla-ICE-подходе: не отправляем ICE-кандидатов
/// по одному через signaling, а дожидаемся, пока все они окажутся
/// в SDP. После этого весь SDP передаётся одним куском.
pub async fn wait_ice_gathering_complete(
    pc: &RtcPeerConnection,
) -> Result<(), JsValue> {
    for _ in 0..200 {
        let state = Reflect::get(
            pc.unchecked_ref::<JsValue>(),
            &JsValue::from_str("iceGatheringState"),
        )
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
        if state == "complete" {
            return Ok(());
        }
        TimeoutFuture::new(50).await;
    }
    return Err(JsValue::from_str("ICE gathering timeout (10s)"));
}

/// SDP → строка JSON. Формат: `{"type":"offer","sdp":"..."}`.
pub fn sdp_to_json_string(desc: &JsValue) -> Result<String, JsValue> {
    let js_str = js_sys::JSON::stringify(desc)?;
    return Ok(js_str.as_string().unwrap_or_default());
}

/// Строка JSON → SDP-объект `{type, sdp}`.
pub fn sdp_from_json_string(s: &str) -> Result<JsValue, JsValue> {
    return js_sys::JSON::parse(s);
}

/// Читает `type` из SDP-объекта: "offer" | "answer" | "".
pub fn sdp_type(desc: &JsValue) -> String {
    return Reflect::get(desc, &JsValue::from_str("type"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
}
```

## A.2. `tests/wasm_rtc_peer.rs`

```rust
//! 45. Wasm-тесты rtc_peer.
//!
//! Проверяют логику, не требующую реального сетевого соединения.
//! Полноценное SDP+ICE-согласование тестируется через демо.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

use hello_wasm02::rtc_peer::{self, IceState, PeerConfig, RtcState};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- Feature detection ----------------

#[wasm_bindgen_test]
fn wasm_rtc_is_supported() {
    assert!(rtc_peer::is_supported());
}

// ---------------- Состояния ----------------

#[wasm_bindgen_test]
fn wasm_rtc_state_roundtrip_known() {
    let cases: [(&str, RtcState); 6] = [
        ("new", RtcState::New),
        ("connecting", RtcState::Connecting),
        ("connected", RtcState::Connected),
        ("disconnected", RtcState::Disconnected),
        ("failed", RtcState::Failed),
        ("closed", RtcState::Closed),
    ];
    for (s, k) in cases {
        assert_eq!(RtcState::from_str(s), k);
        assert_eq!(k.as_str(), s);
    }
}

#[wasm_bindgen_test]
fn wasm_rtc_state_unknown() {
    assert_eq!(RtcState::from_str("garbage"), RtcState::Unknown);
    assert_eq!(RtcState::Unknown.as_str(), "unknown");
}

#[wasm_bindgen_test]
fn wasm_rtc_state_terminal() {
    assert!(RtcState::Closed.is_terminal());
    assert!(RtcState::Failed.is_terminal());
    assert!(!RtcState::Connected.is_terminal());
    assert!(!RtcState::Connecting.is_terminal());
}

#[wasm_bindgen_test]
fn wasm_ice_state_roundtrip_known() {
    let cases: [(&str, IceState); 7] = [
        ("new", IceState::New),
        ("checking", IceState::Checking),
        ("connected", IceState::Connected),
        ("completed", IceState::Completed),
        ("disconnected", IceState::Disconnected),
        ("failed", IceState::Failed),
        ("closed", IceState::Closed),
    ];
    for (s, k) in cases {
        assert_eq!(IceState::from_str(s), k);
        assert_eq!(k.as_str(), s);
    }
}

#[wasm_bindgen_test]
fn wasm_ice_state_unknown() {
    assert_eq!(IceState::from_str("garbage"), IceState::Unknown);
}

// ---------------- PeerConfig ----------------

#[wasm_bindgen_test]
fn wasm_peer_config_loopback_empty() {
    let c = PeerConfig::loopback();
    assert!(c.ice_servers.is_empty());
}

#[wasm_bindgen_test]
fn wasm_peer_config_with_stun_has_google() {
    let c = PeerConfig::with_stun();
    assert_eq!(c.ice_servers.len(), 1);
    assert!(c.ice_servers[0].urls[0].contains("stun.l.google.com"));
}

// ---------------- RTCPeerConnection creation ----------------

#[wasm_bindgen_test]
fn wasm_create_loopback_peer() {
    let pc = rtc_peer::create_peer_connection(&PeerConfig::loopback())
        .expect("create pc");

    // Начальный state — "new". Читаем через Reflect,
    // потому что RtcPeerConnectionState — не подключённая фича.
    let state = js_sys::Reflect::get(
        pc.unchecked_ref::<wasm_bindgen::JsValue>(),
        &wasm_bindgen::JsValue::from_str("connectionState"),
    )
    .ok()
    .and_then(|v| v.as_string())
    .unwrap_or_default();
    assert_eq!(state, "new");

    rtc_peer::close_peer_connection(&pc);
}

#[wasm_bindgen_test]
fn wasm_create_peer_with_stun() {
    let pc = rtc_peer::create_peer_connection(&PeerConfig::with_stun())
        .expect("create pc");
    rtc_peer::close_peer_connection(&pc);
}

// ---------------- Data channel creation ----------------

#[wasm_bindgen_test]
fn wasm_create_data_channel() {
    let pc = rtc_peer::create_peer_connection(&PeerConfig::loopback())
        .expect("create pc");
    let dc = rtc_peer::create_data_channel(&pc, "test-channel")
        .expect("create dc");
    assert_eq!(rtc_peer::data_channel_label(&dc), "test-channel");
    // До соединения state — "connecting".
    assert_eq!(rtc_peer::data_channel_state(&dc), "connecting");
    rtc_peer::close_data_channel(&dc);
    rtc_peer::close_peer_connection(&pc);
}

// ---------------- Offer generation ----------------

#[wasm_bindgen_test]
async fn wasm_create_offer_produces_sdp() {
    let pc = rtc_peer::create_peer_connection(&PeerConfig::loopback())
        .expect("create pc");
    // Нужен data channel, чтобы SDP содержал application m-line.
    let _dc = rtc_peer::create_data_channel(&pc, "test").expect("dc");
    let offer = rtc_peer::create_offer(&pc).await.expect("offer");
    // Проверяем, что offer содержит SDP.
    let sdp = js_sys::Reflect::get(
        &offer,
        &wasm_bindgen::JsValue::from_str("sdp"),
    )
    .ok()
    .and_then(|v| v.as_string())
    .unwrap_or_default();
    assert!(!sdp.is_empty());
    assert!(sdp.contains("v=0"), "sdp = {sdp}");
    rtc_peer::close_peer_connection(&pc);
}

#[wasm_bindgen_test]
async fn wasm_offer_answer_roundtrip() {
    let a = rtc_peer::create_peer_connection(&PeerConfig::loopback())
        .expect("pc A");
    let b = rtc_peer::create_peer_connection(&PeerConfig::loopback())
        .expect("pc B");

    let _dc = rtc_peer::create_data_channel(&a, "chat").expect("dc");
    let offer = rtc_peer::create_offer(&a).await.expect("offer");
    rtc_peer::set_local_description(&a, &offer)
        .await
        .expect("set local A");

    let offer_full = rtc_peer::get_local_description(&a).expect("get local A");
    rtc_peer::set_remote_description(&b, &offer_full)
        .await
        .expect("set remote B");

    let answer = rtc_peer::create_answer(&b).await.expect("answer");
    rtc_peer::set_local_description(&b, &answer)
        .await
        .expect("set local B");

    let answer_full = rtc_peer::get_local_description(&b).expect("get local B");
    rtc_peer::set_remote_description(&a, &answer_full)
        .await
        .expect("set remote A");

    rtc_peer::close_peer_connection(&a);
    rtc_peer::close_peer_connection(&b);
}
```

## A.3. `src/app.rs`

Полный листинг `src/app.rs` приведён в основном тексте главы 45. Здесь он опущен, чтобы не дублировать — см. раздел **45.9 «Демо-пример»** и соответствующий раздел в предыдущем сообщении, где приведён полный код с ручным signaling, ролевым индикатором, live-badges, сводкой ICE-кандидатов, uptime-таймером, подсказкой DataChannel и фидбэком при копировании.

```
//! Глава 45: WebRTC — manual signaling demo.
//!
//! Два независимых сценария в одном файле:
//!   1. **Инициатор** — создаёт offer, ждёт answer.
//!   2. **Ответчик** — принимает offer, создаёт answer.
//!
//! SDP передаётся через копипаст: пользователь копирует свой SDP
//! из textarea и вставляет в textarea собеседника.
//!
//! Используется **vanilla ICE**: не отправляем кандидатов по одному,
//! дожидаемся `iceGatheringState === "complete"` — все кандидаты
//! уже в SDP.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: closures хранятся через forget(), поколение — через Cell<u32>.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::callback::{Interval, Timeout};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use web_sys::{
    HtmlElement, HtmlInputElement, HtmlTextAreaElement,
    RtcDataChannel, RtcPeerConnection,
};

use crate::format_str::format_str;
use crate::rtc_peer::{self, IceState, PeerConfig, RtcState};

const LOG_LIMIT: usize = 30;
const CHAT_LIMIT: usize = 20;
const COPY_FEEDBACK_MS: u32 = 1800;

const HTML: &str = r#"
<style>
  :root {
    --rtc-info: #3498db; --rtc-ok: #27ae60; --rtc-err: #c0392b; --rtc-warn: #e67e22;
    --rtc-text: #222; --rtc-muted: #666;
    --rtc-bg: #fff; --rtc-soft: #fafafa; --rtc-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --rtc-text: #e6e6e6; --rtc-muted: #a0a0a0;
      --rtc-bg: #1e1e1e; --rtc-soft: #262626; --rtc-border: #3a3a3a;
    }
    body { background: #121212; }
    input[type="text"], textarea { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .rtc-log-entry { background: #1e1e1e; }
    .rtc-anno { background: #1a2233; color: #9fc4ff; }
    .rtc-hint { background: #232014; color: #e6c98c; }
    .rtc-chat { background: #1a1a1a; }
    .rtc-msg-row.out { color: #9fc4ff; }
    .rtc-msg-row.in { color: #b8e0b8; }
    .rtc-badge { background: #2a3a4a; color: #cfe0ff; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--rtc-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .rtc-subtitle { color: var(--rtc-muted); font-size: 12px; margin: 0 0 12px; }
  .rtc-subtitle a { color: var(--rtc-info); }

  .rtc-grid {
    display: grid; gap: 12px;
    grid-template-columns: minmax(0, 1.25fr) minmax(0, 1fr) minmax(0, 0.85fr);
  }
  @media (max-width: 1200px) { .rtc-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px)  { .rtc-grid { grid-template-columns: 1fr; } }
  .rtc-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .rtc-panel {
    border: 1px solid var(--rtc-border); border-radius: 8px;
    padding: 10px 12px; background: var(--rtc-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .rtc-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap;
  }
  .rtc-step {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--rtc-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .rtc-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700;
    background: #eef4ff; color: #1f4e79;
    margin-left: auto;
  }
  .rtc-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .rtc-badge.err { background: #fdedec; color: #922b21; }
  .rtc-badge.warn { background: #fff9e6; color: #7d5a00; }

  .rtc-anno {
    background: #eef4ff; border-left: 3px solid var(--rtc-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.55; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere;
  }
  .rtc-hint {
    background: #fff9e6; border-left: 3px solid var(--rtc-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #7d5a00;
  }

  .rtc-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; }
  .rtc-label { font-size: 11px; color: var(--rtc-muted); }

  input[type="text"] {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--rtc-bg); color: var(--rtc-text);
    font-family: ui-monospace, monospace;
    flex: 1; min-width: 100px;
  }
  textarea {
    display: block; width: 100%; box-sizing: border-box;
    padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--rtc-bg); color: var(--rtc-text);
    resize: vertical;
    line-height: 1.35;
  }
  textarea#rtc-local-sdp { height: 150px; }
  textarea#rtc-remote-sdp { height: 150px; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--rtc-soft); color: var(--rtc-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.rtc-primary { background: #eaf4ff; border-color: var(--rtc-info); color: #1f4e79; }
  button.rtc-success { background: #e8f8ed; border-color: var(--rtc-ok); color: #1e8449; }
  button.rtc-danger  { background: #fdedec; border-color: var(--rtc-err); color: #922b21; }
  button.rtc-copied  { background: #e8f8ed; border-color: var(--rtc-ok); color: #1e8449; }

  .rtc-status {
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--rtc-soft); border-left: 3px solid #ccc;
    color: var(--rtc-muted); min-height: 22px; overflow-wrap: anywhere;
  }
  .rtc-status.ok   { background: #e8f8ed; border-color: var(--rtc-ok); color: #1e8449; }
  .rtc-status.err  { background: #fdedec; border-color: var(--rtc-err); color: #922b21; }
  .rtc-status.busy { background: #fff9e6; border-color: var(--rtc-warn); color: #7d5a00; }

  .rtc-peer-grid {
    display: grid; grid-template-columns: auto 1fr;
    gap: 4px 12px;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rtc-peer-grid .k { color: var(--rtc-muted); }
  .rtc-peer-grid .v { color: var(--rtc-text); }
  .rtc-peer-grid .v.ok  { color: #1e8449; }
  .rtc-peer-grid .v.err { color: #922b21; }
  .rtc-peer-grid .v.warn { color: #7d5a00; }

  .rtc-dc-hint {
    margin-top: 6px; padding: 4px 8px; border-radius: 4px;
    font-size: 11px; line-height: 1.4;
    background: var(--rtc-soft); border-left: 3px solid #ccc;
    color: var(--rtc-muted);
  }
  .rtc-dc-hint.ok   { background: #e8f8ed; border-color: var(--rtc-ok); color: #1e8449; }
  .rtc-dc-hint.warn { background: #fff9e6; border-color: var(--rtc-warn); color: #7d5a00; }
  .rtc-dc-hint.err  { background: #fdedec; border-color: var(--rtc-err); color: #922b21; }

  .rtc-chat {
    border: 1px solid var(--rtc-border); border-radius: 5px;
    background: var(--rtc-soft); max-height: 200px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rtc-chat-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; font-size: 11px; }
  .rtc-msg-row { padding: 3px 10px; overflow-wrap: anywhere; }
  .rtc-msg-row.out { color: #1f4e79; }
  .rtc-msg-row.in  { color: #1e8449; }
  .rtc-msg-row.sys { color: #7d5a00; font-style: italic;
    font-size: 11px; text-align: center; padding: 6px 10px; }
  .rtc-msg-time { color: var(--rtc-muted); margin-right: 6px; }

  .rtc-log {
    border: 1px solid var(--rtc-border); border-radius: 5px;
    background: var(--rtc-soft); max-height: 460px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rtc-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .rtc-log-entry {
    display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent;
  }
  .rtc-log-entry.ok   { border-left-color: var(--rtc-ok); }
  .rtc-log-entry.err  { border-left-color: var(--rtc-err); }
  .rtc-log-entry.info { border-left-color: var(--rtc-info); }
  .rtc-log-entry.warn { border-left-color: var(--rtc-warn); }
  .rtc-log-entry .t { color: var(--rtc-muted); flex-shrink: 0; }
  .rtc-log-entry .m { flex: 1; overflow-wrap: anywhere; }
  .rtc-log-entry.err .m  { color: #922b21; }
  .rtc-log-entry.warn .m { color: #7d5a00; }
</style>

<h1>Глава 45: WebRTC</h1>
<p class="rtc-subtitle">
  Manual signaling: общение между двумя вкладками/браузерами.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="rtc-grid">

  <!-- ============ Колонка 1: SDP ============ -->
  <div class="rtc-col">
    <div class="rtc-panel">
      <div class="rtc-anno">
        <b>Как пользоваться.</b> Откройте эту страницу в двух разных
        вкладках или браузерах. В <b>одной</b> вкладке нажмите
        «Создать offer» — она станет <b>инициатором</b>. В <b>другой</b>
        вкладке вставьте полученный SDP в «Входящий» и нажмите
        «Применить» — она станет <b>ответчиком</b>.
      </div>
    </div>

    <div class="rtc-panel">
      <h2 class="rtc-panel-title">
        <span class="rtc-step">1</span>Инициатор
      </h2>
      <div class="rtc-hint">
        Доступно до создания peer'а. После — используйте Reset,
        чтобы начать заново.
      </div>
      <div class="rtc-row">
        <button id="rtc-create-offer" class="rtc-success">
          Создать offer
        </button>
      </div>
    </div>

    <div class="rtc-panel">
      <h2 class="rtc-panel-title">
        <span class="rtc-step">2</span>Мой SDP (отправить собеседнику)
        <span class="rtc-badge" id="rtc-local-badge">—</span>
      </h2>
      <textarea id="rtc-local-sdp" readonly
        placeholder="здесь появится ваш SDP после нажатия «Создать offer/answer»"></textarea>
      <div class="rtc-row">
        <button id="rtc-copy" disabled>📋 Копировать SDP</button>
        <span class="rtc-label" id="rtc-local-size">0 байт</span>
      </div>
    </div>

    <div class="rtc-panel">
      <h2 class="rtc-panel-title">
        <span class="rtc-step">3</span>Входящий SDP (от собеседника)
        <span class="rtc-badge" id="rtc-remote-badge">—</span>
      </h2>
      <textarea id="rtc-remote-sdp"
        placeholder="вставьте сюда SDP из соседней вкладки (Ctrl+V)"></textarea>
      <div class="rtc-row">
        <button id="rtc-apply" class="rtc-primary" disabled>
          Применить входящий SDP
        </button>
      </div>
    </div>

    <div class="rtc-panel">
      <h2 class="rtc-panel-title">
        <span class="rtc-step">4</span>Ответчик
      </h2>
      <div class="rtc-hint">
        Доступно только после применения входящего offer. Нажмите —
        получите свой SDP, скопируйте его обратно инициатору.
      </div>
      <div class="rtc-row">
        <button id="rtc-create-answer" disabled>Создать answer</button>
      </div>
    </div>

    <div class="rtc-row">
      <button id="rtc-reset" class="rtc-danger">↺ Reset</button>
    </div>

    <div id="rtc-status" class="rtc-status">
      Начните с «Создать offer» в этой вкладке — или вставьте SDP
      из соседней вкладки в блок 3.
    </div>
  </div>

  <!-- ============ Колонка 2: Состояние и DataChannel ============ -->
  <div class="rtc-col">
    <div class="rtc-panel">
      <h2 class="rtc-panel-title">Состояние соединения</h2>
      <div id="rtc-states" class="rtc-peer-grid">
        <span class="k">role</span><span class="v">—</span>
        <span class="k">connectionState</span><span class="v">new</span>
        <span class="k">iceConnectionState</span><span class="v">new</span>
        <span class="k">dataChannel</span><span class="v">—</span>
        <span class="k">uptime</span><span class="v">—</span>
      </div>
    </div>

    <div class="rtc-panel">
      <h2 class="rtc-panel-title">DataChannel «chat»</h2>
      <div class="rtc-row">
        <input type="text" id="rtc-msg" value="Привет через WebRTC!">
        <button id="rtc-send" disabled>➤ Send</button>
      </div>
      <div id="rtc-chat" class="rtc-chat">
        <div class="rtc-chat-empty">канал откроется после обмена SDP</div>
      </div>
      <div id="rtc-dc-hint" class="rtc-dc-hint">
        Канал ещё не создан — начните с «Создать offer» или примите
        входящий offer.
      </div>
    </div>

    <div class="rtc-panel">
      <h2 class="rtc-panel-title">Метрики</h2>
      <div id="rtc-metrics" class="rtc-peer-grid">
        <span class="k">—</span><span class="v">нет данных</span>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 3: Журнал ============ -->
  <div class="rtc-col">
    <div class="rtc-panel">
      <h2 class="rtc-panel-title">Журнал событий</h2>
      <div id="rtc-log" class="rtc-log">
        <div class="rtc-log-empty">здесь появятся события</div>
      </div>
    </div>
  </div>

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

#[derive(Clone, Copy, PartialEq)]
enum ChatDir { Out, In, Sys }

struct ChatMsg {
    time: String,
    dir: ChatDir,
    text: String,
}

#[derive(Clone, Copy, PartialEq)]
enum Role {
    None,
    Initiator,
    Responder,
}

impl Role {
    fn as_str(self) -> &'static str {
        return match self {
            Role::None => "—",
            Role::Initiator => "инициатор",
            Role::Responder => "ответчик",
        };
    }
}

#[derive(Default)]
struct Metrics {
    ice_host: u32,
    ice_srflx: u32,
    ice_relay: u32,
    ice_other: u32,
    messages_sent: u32,
    messages_recv: u32,
    bytes_sent: u64,
    bytes_recv: u64,
    sdp_size_local: usize,
    sdp_size_remote: usize,
}

struct Ui {
    btn_create_offer: HtmlElement,
    btn_create_answer: HtmlElement,
    btn_apply: HtmlElement,
    btn_copy: HtmlElement,
    btn_send: HtmlElement,
    btn_reset: HtmlElement,
    local_sdp: HtmlTextAreaElement,
    remote_sdp: HtmlTextAreaElement,
    local_badge: web_sys::Element,
    remote_badge: web_sys::Element,
    local_size: web_sys::Element,
    msg: HtmlInputElement,
    status: web_sys::Element,
    states: web_sys::Element,
    dc_hint: web_sys::Element,
    chat: web_sys::Element,
    metrics: web_sys::Element,
    log: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    chat: Rc<RefCell<Vec<ChatMsg>>>,
    metrics: Rc<RefCell<Metrics>>,
    peer: RefCell<Option<RtcPeerConnection>>,
    dc: RefCell<Option<RtcDataChannel>>,
    state: Rc<RefCell<(RtcState, IceState)>>,
    role: Rc<Cell<Role>>,
    generation: Rc<Cell<u32>>,
    connected_since_ms: Rc<Cell<f64>>,
    answer_ready: Rc<Cell<bool>>,
    _uptime_timer: RefCell<Option<Interval>>,
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
        btn_create_offer: get_el("rtc-create-offer")?.dyn_into()?,
        btn_create_answer: get_el("rtc-create-answer")?.dyn_into()?,
        btn_apply: get_el("rtc-apply")?.dyn_into()?,
        btn_copy: get_el("rtc-copy")?.dyn_into()?,
        btn_send: get_el("rtc-send")?.dyn_into()?,
        btn_reset: get_el("rtc-reset")?.dyn_into()?,
        local_sdp: get_el("rtc-local-sdp")?.dyn_into()?,
        remote_sdp: get_el("rtc-remote-sdp")?.dyn_into()?,
        local_badge: get_el("rtc-local-badge")?,
        remote_badge: get_el("rtc-remote-badge")?,
        local_size: get_el("rtc-local-size")?,
        msg: get_input("rtc-msg")?,
        status: get_el("rtc-status")?,
        states: get_el("rtc-states")?,
        dc_hint: get_el("rtc-dc-hint")?,
        chat: get_el("rtc-chat")?,
        metrics: get_el("rtc-metrics")?,
        log: get_el("rtc-log")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        chat: Rc::new(RefCell::new(Vec::new())),
        metrics: Rc::new(RefCell::new(Metrics::default())),
        peer: RefCell::new(None),
        dc: RefCell::new(None),
        state: Rc::new(RefCell::new((RtcState::New, IceState::New))),
        role: Rc::new(Cell::new(Role::None)),
        generation: Rc::new(Cell::new(0)),
        connected_since_ms: Rc::new(Cell::new(0.0)),
        answer_ready: Rc::new(Cell::new(false)),
        _uptime_timer: RefCell::new(None),
        ui,
    });

    if !rtc_peer::is_supported() {
        set_status(&app.ui.status, "err", "WebRTC API недоступен");
        return Ok(());
    }

    app.push_log(LogKind::Info, "WebRTC API доступен".to_string());
    app.render_log();
    app.render_chat();
    render_states(&app);
    render_metrics(&app);
    render_dc_hint(&app);
    update_controls(&app);
    update_badges(&app);

    bind_buttons(&app)?;
    bind_live_input(&app)?;

    // Uptime-таймер.
    {
        let app_c = app.clone();
        let interval = Interval::new(1000, move || {
            if app_c.connected_since_ms.get() > 0.0 {
                render_states(&app_c);
                render_metrics(&app_c);
            }
        });
        *app._uptime_timer.borrow_mut() = Some(interval);
    }

    Ok(())
}

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_create_offer, move || {
            create_offer_clicked(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_create_answer, move || {
            create_answer_clicked(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_apply, move || {
            apply_remote_clicked(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_copy, move || {
            copy_local_sdp(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_send, move || {
            send_message(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_reset, move || {
            do_reset(&app_c);
        });
    }
    Ok(())
}

fn bind_live_input(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        update_badges(&app_c);
        update_controls(&app_c);
    });
    app.ui
        .remote_sdp
        .add_event_listener_with_callback("input", cb.as_ref().unchecked_ref())?;
    cb.forget();
    Ok(())
}

// ============================================================
//                    HANDLERS
// ============================================================

fn create_offer_clicked(app: Rc<App>) {
    reset_inner(&app);
    app.role.set(Role::Initiator);

    let pc = match rtc_peer::create_peer_connection(&PeerConfig::loopback()) {
        Ok(p) => p,
        Err(e) => {
            let msg = format_str(format_args!("create pc: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
            return;
        }
    };

    attach_state_listeners(&app, &pc);
    attach_ondatachannel_listener(&app, &pc);

    let dc = match rtc_peer::create_data_channel(&pc, "chat") {
        Ok(d) => d,
        Err(e) => {
            let msg = format_str(format_args!("create dc: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
            return;
        }
    };
    attach_dc_listeners(&app, &dc);
    *app.dc.borrow_mut() = Some(dc);
    *app.peer.borrow_mut() = Some(pc.clone());

    let gen = app.generation.get();
    app.push_log(
        LogKind::Warn,
        "=== Инициатор: создаём offer ===".to_string(),
    );
    set_status(&app.ui.status, "busy", "создание offer…");
    render_states(&app);
    render_dc_hint(&app);
    update_controls(&app);

    wasm_bindgen_futures::spawn_local(async move {
        let offer = match rtc_peer::create_offer(&pc).await {
            Ok(o) => o,
            Err(e) => {
                if app.generation.get() == gen {
                    let msg = format_str(format_args!("create_offer: {e:?}"));
                    app.push_log(LogKind::Err, msg.clone());
                    set_status(&app.ui.status, "err", &msg);
                }
                return;
            }
        };
        if let Err(e) = rtc_peer::set_local_description(&pc, &offer).await {
            if app.generation.get() == gen {
                let msg = format_str(format_args!("setLocalDescription: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
            }
            return;
        }

        if app.generation.get() == gen {
            set_status(&app.ui.status, "busy", "собираем ICE-кандидатов…");
        }

        if let Err(e) = rtc_peer::wait_ice_gathering_complete(&pc).await {
            if app.generation.get() == gen {
                let msg = format_str(format_args!("ICE gathering: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
            }
            return;
        }

        if app.generation.get() != gen {
            return;
        }

        let full = match rtc_peer::get_local_description(&pc) {
            Ok(f) => f,
            Err(e) => {
                let msg = format_str(format_args!("get_local_description: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
                return;
            }
        };
        let text = match rtc_peer::sdp_to_json_string(&full) {
            Ok(t) => t,
            Err(e) => {
                let msg = format_str(format_args!("sdp_to_json: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
                return;
            }
        };

        app.ui.local_sdp.set_value(&text);
        update_local_size(&app);
        update_badges(&app);

        app.push_log(
            LogKind::Ok,
            format_str(format_args!(
                "offer готов ({} байт, ICE-кандидаты внутри)",
                text.len(),
            )),
        );
        set_status(
            &app.ui.status,
            "ok",
            "1) скопируйте SDP  2) вставьте в соседнюю вкладку  3) ждите answer",
        );
        update_controls(&app);
    });
}

fn create_answer_clicked(app: Rc<App>) {
    if !app.answer_ready.get() {
        set_status(&app.ui.status, "err", "сначала примените входящий offer");
        return;
    }
    let pc = match app.peer.borrow().clone() {
        Some(p) => p,
        None => {
            set_status(&app.ui.status, "err", "peer не создан");
            return;
        }
    };

    let gen = app.generation.get();
    app.push_log(
        LogKind::Warn,
        "=== Ответчик: создаём answer ===".to_string(),
    );
    set_status(&app.ui.status, "busy", "создание answer…");
    update_controls(&app);

    wasm_bindgen_futures::spawn_local(async move {
        let answer = match rtc_peer::create_answer(&pc).await {
            Ok(a) => a,
            Err(e) => {
                if app.generation.get() == gen {
                    let msg = format_str(format_args!("create_answer: {e:?}"));
                    app.push_log(LogKind::Err, msg.clone());
                    set_status(&app.ui.status, "err", &msg);
                }
                return;
            }
        };
        if let Err(e) = rtc_peer::set_local_description(&pc, &answer).await {
            if app.generation.get() == gen {
                let msg = format_str(format_args!("setLocalDescription: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
            }
            return;
        }
        if app.generation.get() == gen {
            set_status(&app.ui.status, "busy", "собираем ICE-кандидатов…");
        }

        if let Err(e) = rtc_peer::wait_ice_gathering_complete(&pc).await {
            if app.generation.get() == gen {
                let msg = format_str(format_args!("ICE gathering: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
            }
            return;
        }
        if app.generation.get() != gen {
            return;
        }

        let full = match rtc_peer::get_local_description(&pc) {
            Ok(f) => f,
            Err(e) => {
                let msg = format_str(format_args!("get_local_description: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
                return;
            }
        };
        let text = match rtc_peer::sdp_to_json_string(&full) {
            Ok(t) => t,
            Err(e) => {
                let msg = format_str(format_args!("sdp_to_json: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
                return;
            }
        };

        app.ui.local_sdp.set_value(&text);
        update_local_size(&app);
        update_badges(&app);

        app.push_log(
            LogKind::Ok,
            format_str(format_args!(
                "answer готов ({} байт, ICE-кандидаты внутри)",
                text.len(),
            )),
        );
        set_status(
            &app.ui.status,
            "ok",
            "скопируйте answer и вставьте его в первую вкладку",
        );
        update_controls(&app);
    });
}

fn apply_remote_clicked(app: Rc<App>) {
    let text = app.ui.remote_sdp.value();
    if text.trim().is_empty() {
        set_status(&app.ui.status, "err", "входящий SDP пуст");
        return;
    }
    let parsed = match rtc_peer::sdp_from_json_string(&text) {
        Ok(v) => v,
        Err(e) => {
            let msg = format_str(format_args!(
                "не удалось распарсить SDP: ожидался JSON вида \
                 {{\"type\":\"offer|answer\",\"sdp\":\"...\"}}. Ошибка: {e:?}",
            ));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
            return;
        }
    };
    let typ = rtc_peer::sdp_type(&parsed);
    if typ != "offer" && typ != "answer" {
        set_status(
            &app.ui.status,
            "err",
            "во входящем SDP поле type не offer и не answer",
        );
        return;
    }

    let pc = ensure_peer(&app);
    app.metrics.borrow_mut().sdp_size_remote = text.len();

    if typ == "offer" {
        app.role.set(Role::Responder);
    }

    let gen = app.generation.get();
    let app_c = app.clone();
    set_status(
        &app.ui.status,
        "busy",
        &format_str(format_args!("применяем входящий {typ}…")),
    );

    wasm_bindgen_futures::spawn_local(async move {
        if let Err(e) = rtc_peer::set_remote_description(&pc, &parsed).await {
            if app_c.generation.get() == gen {
                let msg = format_str(format_args!("setRemoteDescription: {e:?}"));
                app_c.push_log(LogKind::Err, msg.clone());
                set_status(&app_c.ui.status, "err", &msg);
            }
            return;
        }
        if app_c.generation.get() != gen {
            return;
        }

        if typ == "offer" {
            app_c.answer_ready.set(true);
            app_c.push_log(
                LogKind::Ok,
                "входящий offer применён — вы ответчик".to_string(),
            );
            set_status(
                &app_c.ui.status,
                "ok",
                "нажмите «Создать answer» → скопируйте → отправьте инициатору",
            );
        } else {
            app_c.push_log(
                LogKind::Ok,
                "входящий answer применён — ICE-соединение устанавливается"
                    .to_string(),
            );
            set_status(
                &app_c.ui.status,
                "busy",
                "ICE handshake: ждём установки соединения…",
            );
        }
        render_states(&app_c);
        render_dc_hint(&app_c);
        update_badges(&app_c);
        update_controls(&app_c);
    });
}

fn copy_local_sdp(app: Rc<App>) {
    let text = app.ui.local_sdp.value();
    if text.is_empty() {
        return;
    }
    let success = try_copy_to_clipboard(&text);
    if success {
        app.push_log(
            LogKind::Ok,
            format_str(format_args!(
                "SDP скопирован в буфер ({} байт)",
                text.len(),
            )),
        );
        set_status(
            &app.ui.status,
            "ok",
            "SDP в буфере — вставьте его в соседнюю вкладку (Ctrl+V)",
        );
        show_copy_feedback(&app);
    } else {
        set_status(
            &app.ui.status,
            "err",
            "Clipboard API недоступен — выделите SDP и нажмите Ctrl+C",
        );
    }
}

fn try_copy_to_clipboard(text: &str) -> bool {
    let Some(win) = web_sys::window() else {
        return false;
    };
    let nav = win.navigator();
    let Ok(clipboard) = js_sys::Reflect::get(
        &nav,
        &JsValue::from_str("clipboard"),
    ) else {
        return false;
    };
    let Ok(write_fn) = js_sys::Reflect::get(
        &clipboard,
        &JsValue::from_str("writeText"),
    )
    .and_then(|v| v.dyn_into::<js_sys::Function>())
    else {
        return false;
    };
    let _ = write_fn.call1(&clipboard, &JsValue::from_str(text));
    return true;
}

fn show_copy_feedback(app: &Rc<App>) {
    // Меняем текст кнопки на «✓ Скопировано» на 1.8 секунды.
    let btn = app.ui.btn_copy.clone();
    btn.set_text_content(Some("✓ Скопировано"));
    btn.set_class_name("rtc-copied");
    let btn2 = btn.clone();
    Timeout::new(COPY_FEEDBACK_MS, move || {
        btn2.set_text_content(Some("📋 Копировать SDP"));
        btn2.set_class_name("");
    })
    .forget();
}

fn send_message(app: Rc<App>) {
    let text = app.ui.msg.value();
    if text.is_empty() {
        return;
    }
    let Some(dc) = app.dc.borrow().clone() else {
        set_status(&app.ui.status, "err", "DataChannel не создан");
        return;
    };
    match rtc_peer::data_channel_send(&dc, &text) {
        Ok(()) => {
            let n = text.len();
            {
                let mut m = app.metrics.borrow_mut();
                m.messages_sent += 1;
                m.bytes_sent += n as u64;
            }
            push_chat(&app, ChatDir::Out, &text);
            render_metrics(&app);
        }
        Err(e) => {
            let msg = format_str(format_args!("send: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

// ============================================================
//                    PEER & LISTENERS
// ============================================================

fn ensure_peer(app: &Rc<App>) -> RtcPeerConnection {
    if let Some(pc) = app.peer.borrow().as_ref() {
        return pc.clone();
    }
    let pc = rtc_peer::create_peer_connection(&PeerConfig::loopback())
        .expect("create pc");
    attach_state_listeners(app, &pc);
    attach_ondatachannel_listener(app, &pc);
    *app.peer.borrow_mut() = Some(pc.clone());
    return pc;
}

fn attach_state_listeners(app: &Rc<App>, pc: &RtcPeerConnection) {
    let gen = app.generation.get();

    {
        let app_c = app.clone();
        let state_c = app.state.clone();
        let connected_c = app.connected_since_ms.clone();
        rtc_peer::on_connection_state_change(pc, move |s| {
            if app_c.generation.get() != gen {
                return;
            }
            {
                let mut st = state_c.borrow_mut();
                st.0 = s;
            }
            if s == RtcState::Connected {
                if connected_c.get() <= 0.0 {
                    connected_c.set(js_sys::Date::now());
                }
            } else {
                connected_c.set(0.0);
            }
            app_c.push_log(
                LogKind::Info,
                format_str(format_args!("connectionState → {}", s.as_str())),
            );
            render_states(&app_c);
            update_controls(&app_c);
        })
        .ok();
    }

    {
        let app_c = app.clone();
        let state_c = app.state.clone();
        rtc_peer::on_ice_connection_state_change(pc, move |s| {
            if app_c.generation.get() != gen {
                return;
            }
            {
                let mut st = state_c.borrow_mut();
                st.1 = s;
            }
            app_c.push_log(
                LogKind::Info,
                format_str(format_args!("iceConnectionState → {}", s.as_str())),
            );
            render_states(&app_c);
        })
        .ok();
    }

    {
        let app_c = app.clone();
        rtc_peer::on_ice_candidate(pc, move |cand| {
            if app_c.generation.get() != gen {
                return;
            }
            match cand {
                Some(c) => {
                    let c_str = js_sys::Reflect::get(
                        c.unchecked_ref::<JsValue>(),
                        &JsValue::from_str("candidate"),
                    )
                    .ok()
                    .and_then(|v| v.as_string())
                    .unwrap_or_default();
                    let typ = ice_candidate_type(&c_str);
                    let mut m = app_c.metrics.borrow_mut();
                    match typ {
                        "host" => m.ice_host += 1,
                        "srflx" | "prflx" => m.ice_srflx += 1,
                        "relay" => m.ice_relay += 1,
                        _ => m.ice_other += 1,
                    }
                }
                None => {
                    let (h, s, r, o) = {
                        let m = app_c.metrics.borrow();
                        (m.ice_host, m.ice_srflx, m.ice_relay, m.ice_other)
                    };
                    app_c.push_log(
                        LogKind::Info,
                        format_str(format_args!(
                            "ICE gathering завершён: host={h} srflx={s} \
                             relay={r} other={o}",
                        )),
                    );
                }
            }
            render_metrics(&app_c);
        })
        .ok();
    }
}

fn attach_ondatachannel_listener(app: &Rc<App>, pc: &RtcPeerConnection) {
    let gen = app.generation.get();
    let app_c = app.clone();
    rtc_peer::on_data_channel(pc, move |dc| {
        if app_c.generation.get() != gen {
            return;
        }
        app_c.push_log(
            LogKind::Ok,
            format_str(format_args!(
                "получен DataChannel «{}» от собеседника",
                rtc_peer::data_channel_label(&dc),
            )),
        );
        attach_dc_listeners(&app_c, &dc);
        *app_c.dc.borrow_mut() = Some(dc);
        update_controls(&app_c);
        render_states(&app_c);
        render_dc_hint(&app_c);
    })
    .ok();
}

fn attach_dc_listeners(app: &Rc<App>, dc: &RtcDataChannel) {
    let gen = app.generation.get();

    {
        let app_c = app.clone();
        rtc_peer::on_data_channel_open(dc, move || {
            if app_c.generation.get() != gen {
                return;
            }
            app_c.push_log(
                LogKind::Ok,
                "DataChannel открыт — можно отправлять сообщения".to_string(),
            );
            set_status(
                &app_c.ui.status,
                "ok",
                "соединение установлено, DataChannel открыт",
            );
            push_chat(&app_c, ChatDir::Sys, "— соединение установлено —");
            update_controls(&app_c);
            render_states(&app_c);
            render_dc_hint(&app_c);
        })
        .ok();
    }

    {
        let app_c = app.clone();
        rtc_peer::on_data_channel_message(dc, move |text| {
            if app_c.generation.get() != gen {
                return;
            }
            let n = text.len();
            {
                let mut m = app_c.metrics.borrow_mut();
                m.messages_recv += 1;
                m.bytes_recv += n as u64;
            }
            push_chat(&app_c, ChatDir::In, &text);
            render_metrics(&app_c);
        })
        .ok();
    }
}

// ============================================================
//                    RESET
// ============================================================

fn do_reset(app: &Rc<App>) {
    reset_inner(app);
    set_status(
        &app.ui.status,
        "ok",
        "Начните с «Создать offer» в этой вкладке — или вставьте SDP \
         из соседней вкладки в блок 3.",
    );
    app.push_log(LogKind::Warn, "--- reset ---".to_string());
    update_controls(app);
}

fn reset_inner(app: &Rc<App>) {
    app.generation.set(app.generation.get().wrapping_add(1));

    if let Some(dc) = app.dc.borrow_mut().take() {
        rtc_peer::close_data_channel(&dc);
    }
    if let Some(pc) = app.peer.borrow_mut().take() {
        rtc_peer::close_peer_connection(&pc);
    }

    *app.metrics.borrow_mut() = Metrics::default();
    *app.state.borrow_mut() = (RtcState::New, IceState::New);
    app.role.set(Role::None);
    app.connected_since_ms.set(0.0);
    app.answer_ready.set(false);

    app.ui.local_sdp.set_value("");
    app.ui.remote_sdp.set_value("");
    app.chat.borrow_mut().clear();

    update_local_size(app);
    render_states(app);
    render_metrics(app);
    render_dc_hint(app);
    update_badges(app);
    app.render_chat();
}

// ============================================================
//                    УТИЛИТЫ
// ============================================================

fn ice_candidate_type(s: &str) -> &'static str {
    let parts: Vec<&str> = s.split_whitespace().collect();
    for i in 0..parts.len().saturating_sub(1) {
        if parts[i] == "typ" {
            return match parts[i + 1] {
                "host" => "host",
                "srflx" => "srflx",
                "prflx" => "prflx",
                "relay" => "relay",
                _ => "other",
            };
        }
    }
    return "other";
}

fn format_uptime(sec: u64) -> String {
    let m = sec / 60;
    let s = sec % 60;
    return format_str(format_args!("{m}:{s:02}"));
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn update_local_size(app: &Rc<App>) {
    let n = app.ui.local_sdp.value().len();
    app.metrics.borrow_mut().sdp_size_local = n;
    let text = format_str(format_args!("{n} байт"));
    app.ui.local_size.set_text_content(Some(&text));
    render_metrics(app);
}

fn update_badges(app: &Rc<App>) {
    // Локальный badge.
    let local_text = app.ui.local_sdp.value();
    let (local_label, local_cls) = if local_text.trim().is_empty() {
        ("—".to_string(), "rtc-badge")
    } else {
        match rtc_peer::sdp_from_json_string(&local_text) {
            Ok(v) => {
                let typ = rtc_peer::sdp_type(&v);
                if typ == "offer" || typ == "answer" {
                    (format!("{typ} ✓"), "rtc-badge ok")
                } else {
                    ("нет type".to_string(), "rtc-badge warn")
                }
            }
            Err(_) => ("формат?".to_string(), "rtc-badge warn"),
        }
    };
    app.ui.local_badge.set_text_content(Some(&local_label));
    app.ui.local_badge.set_class_name(local_cls);

    // Входящий badge.
    let remote_text = app.ui.remote_sdp.value();
    let (remote_label, remote_cls) = if remote_text.trim().is_empty() {
        ("—".to_string(), "rtc-badge")
    } else {
        match rtc_peer::sdp_from_json_string(&remote_text) {
            Ok(v) => {
                let typ = rtc_peer::sdp_type(&v);
                if typ == "offer" || typ == "answer" {
                    (format!("{typ} ✓"), "rtc-badge ok")
                } else {
                    ("нет type".to_string(), "rtc-badge warn")
                }
            }
            Err(_) => ("не JSON".to_string(), "rtc-badge err"),
        }
    };
    app.ui.remote_badge.set_text_content(Some(&remote_label));
    app.ui.remote_badge.set_class_name(remote_cls);
}

fn render_states(app: &Rc<App>) {
    let st = *app.state.borrow();
    let mut html = String::with_capacity(384);
    push_row(&mut html, "role", app.role.get().as_str());
    push_row_cls(
        &mut html,
        "connectionState",
        st.0.as_str(),
        tone_conn(st.0),
    );
    push_row_cls(
        &mut html,
        "iceConnectionState",
        st.1.as_str(),
        tone_ice(st.1),
    );
    let dc_state = app
        .dc
        .borrow()
        .as_ref()
        .map(|dc| rtc_peer::data_channel_state(dc))
        .unwrap_or("—");
    let dc_cls = if dc_state == "open" { "v ok" } else { "v warn" };
    push_row_cls(&mut html, "dataChannel", dc_state, dc_cls);

    let since = app.connected_since_ms.get();
    let uptime = if since > 0.0 {
        let sec = ((js_sys::Date::now() - since) / 1000.0).floor() as u64;
        format_uptime(sec)
    } else {
        "—".to_string()
    };
    let uptime_cls = if since > 0.0 { "v ok" } else { "v" };
    push_row_cls(&mut html, "uptime", &uptime, uptime_cls);

    app.ui.states.set_inner_html(&html);
}

/// Подсказка о состоянии DataChannel — поясняет «connecting»
/// и указывает, что делать дальше.
fn render_dc_hint(app: &Rc<App>) {
    let dc_opt = app.dc.borrow().clone();
    let peer_exists = app.peer.borrow().is_some();
    let role = app.role.get();

    let (cls, text) = match dc_opt.as_ref() {
        Some(dc) => {
            let state = rtc_peer::data_channel_state(dc);
            match state {
                "open" => (
                    "rtc-dc-hint ok",
                    "Канал открыт — можно отправлять сообщения.",
                ),
                "connecting" => {
                    if role == Role::Initiator {
                        (
                            "rtc-dc-hint warn",
                            "Ожидаем answer от собеседника. Канал откроется \
                             после обмена SDP и ICE-согласования.",
                        )
                    } else {
                        (
                            "rtc-dc-hint warn",
                            "Ожидаем, пока инициатор применит answer. \
                             Канал откроется после ICE-согласования.",
                        )
                    }
                }
                "closing" => ("rtc-dc-hint warn", "Канал закрывается."),
                "closed" => ("rtc-dc-hint err", "Канал закрыт. Нажмите Reset."),
                _ => ("rtc-dc-hint", "Неизвестное состояние канала."),
            }
        }
        None => {
            if peer_exists {
                (
                    "rtc-dc-hint warn",
                    "Peer создан, но DataChannel ещё не открыт. \
                     Дождитесь установки соединения.",
                )
            } else {
                (
                    "rtc-dc-hint",
                    "Канал ещё не создан — начните с «Создать offer» \
                     или примите входящий offer.",
                )
            }
        }
    };
    app.ui.dc_hint.set_class_name(cls);
    app.ui.dc_hint.set_text_content(Some(text));
}

fn render_metrics(app: &Rc<App>) {
    let m = app.metrics.borrow();
    let mut html = String::with_capacity(512);
    push_row(
        &mut html,
        "ICE host",
        &format_str(format_args!("{}", m.ice_host)),
    );
    push_row(
        &mut html,
        "ICE srflx",
        &format_str(format_args!("{}", m.ice_srflx)),
    );
    push_row(
        &mut html,
        "ICE relay",
        &format_str(format_args!("{}", m.ice_relay)),
    );
    push_row(
        &mut html,
        "messages sent",
        &format_str(format_args!("{}", m.messages_sent)),
    );
    push_row(
        &mut html,
        "messages recv",
        &format_str(format_args!("{}", m.messages_recv)),
    );
    push_row(
        &mut html,
        "bytes sent",
        &format_str(format_args!("{}", m.bytes_sent)),
    );
    push_row(
        &mut html,
        "bytes recv",
        &format_str(format_args!("{}", m.bytes_recv)),
    );
    push_row(
        &mut html,
        "my SDP size",
        &format_str(format_args!("{} B", m.sdp_size_local)),
    );
    push_row(
        &mut html,
        "incoming SDP",
        &format_str(format_args!("{} B", m.sdp_size_remote)),
    );
    drop(m);
    app.ui.metrics.set_inner_html(&html);
}

fn tone_conn(s: RtcState) -> &'static str {
    return match s {
        RtcState::Connected => "v ok",
        RtcState::Failed | RtcState::Closed => "v err",
        RtcState::Connecting | RtcState::Disconnected => "v warn",
        _ => "v",
    };
}

fn tone_ice(s: IceState) -> &'static str {
    return match s {
        IceState::Connected | IceState::Completed => "v ok",
        IceState::Failed | IceState::Closed => "v err",
        IceState::Checking | IceState::Disconnected => "v warn",
        _ => "v",
    };
}

fn update_controls(app: &Rc<App>) {
    let has_local_sdp = !app.ui.local_sdp.value().is_empty();
    let remote_text = app.ui.remote_sdp.value();
    let has_remote_sdp = !remote_text.trim().is_empty();
    let peer_exists = app.peer.borrow().is_some();
    let answer_ready = app.answer_ready.get();
    let dc_open = app
        .dc
        .borrow()
        .as_ref()
        .map(|dc| rtc_peer::data_channel_state(dc) == "open")
        .unwrap_or(false);

    let remote_valid = if has_remote_sdp {
        match rtc_peer::sdp_from_json_string(&remote_text) {
            Ok(v) => {
                let typ = rtc_peer::sdp_type(&v);
                typ == "offer" || typ == "answer"
            }
            Err(_) => false,
        }
    } else {
        false
    };

    set_disabled(&app.ui.btn_create_offer, peer_exists);
    set_disabled(
        &app.ui.btn_create_answer,
        !(peer_exists && answer_ready && !has_local_sdp),
    );
    set_disabled(&app.ui.btn_apply, !remote_valid);
    set_disabled(&app.ui.btn_copy, !has_local_sdp);
    set_disabled(&app.ui.btn_send, !dc_open);
}

fn set_disabled(el: &HtmlElement, disabled: bool) {
    if disabled {
        let _ = el.set_attribute("disabled", "");
    } else {
        let _ = el.remove_attribute("disabled");
    }
}

// ============================================================
//                    UI HELPERS
// ============================================================

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "rtc-status ok",
        "err" => "rtc-status err",
        "busy" => "rtc-status busy",
        _ => "rtc-status",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn push_row(html: &mut String, k: &str, v: &str) {
    push_row_cls(html, k, v, "v");
}

fn push_row_cls(html: &mut String, k: &str, v: &str, v_cls: &str) {
    html.push_str("<span class=\"k\">");
    html.push_str(&esc_html(k));
    html.push_str("</span><span class=\"");
    html.push_str(v_cls);
    html.push_str("\">");
    html.push_str(&esc_html(v));
    html.push_str("</span>");
}

fn push_log_static(log: &Rc<RefCell<Vec<LogEntry>>>, kind: LogKind, text: String) {
    let time = fmt_now_hms();
    let mut v = log.borrow_mut();
    v.push(LogEntry { time, text, kind });
    if v.len() > LOG_LIMIT {
        let drop_n = v.len() - LOG_LIMIT;
        v.drain(0..drop_n);
    }
}

fn push_chat(app: &Rc<App>, dir: ChatDir, text: &str) {
    let time = fmt_now_hms();
    let mut chat = app.chat.borrow_mut();
    chat.push(ChatMsg {
        time,
        dir,
        text: text.to_string(),
    });
    if chat.len() > CHAT_LIMIT {
        let drop_n = chat.len() - CHAT_LIMIT;
        chat.drain(0..drop_n);
    }
    drop(chat);
    app.render_chat();
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
                "<div class=\"rtc-log-empty\">здесь появятся события</div>",
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
            html.push_str("<div class=\"rtc-log-entry ");
            html.push_str(cls);
            html.push_str("\"><span class=\"t\">");
            html.push_str(&esc_html(&e.time));
            html.push_str("</span><span class=\"m\">");
            html.push_str(&esc_html(&e.text));
            html.push_str("</span></div>");
        }
        self.ui.log.set_inner_html(&html);
        self.ui.log.set_scroll_top(self.ui.log.scroll_height() as f64);
    }

    fn render_chat(&self) {
        let chat = self.chat.borrow();
        if chat.is_empty() {
            self.ui.chat.set_inner_html(
                "<div class=\"rtc-chat-empty\">канал откроется после обмена SDP</div>",
            );
            return;
        }
        let mut html = String::with_capacity(512);
        for m in chat.iter() {
            let (cls, prefix) = match m.dir {
                ChatDir::Out => ("rtc-msg-row out", "→"),
                ChatDir::In => ("rtc-msg-row in", "←"),
                ChatDir::Sys => ("rtc-msg-row sys", ""),
            };
            html.push_str("<div class=\"");
            html.push_str(cls);
            html.push_str("\">");
            if !prefix.is_empty() {
                html.push_str("<span class=\"rtc-msg-time\">");
                html.push_str(&esc_html(&m.time));
                html.push(' ');
                html.push_str(prefix);
                html.push_str("</span>");
            }
            html.push_str(&esc_html(&m.text));
            html.push_str("</div>");
        }
        self.ui.chat.set_inner_html(&html);
        self.ui
            .chat
            .set_scroll_top(self.ui.chat.scroll_height() as f64);
    }
}

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(), d.get_minutes(), d.get_seconds(),
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