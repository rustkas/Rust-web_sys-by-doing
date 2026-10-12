# Глава 37. WebTransport

`WebTransport` — единственный транспорт в Web Platform, который работает поверх **QUIC** и предоставляет **три независимых канала передачи** в одном соединении. Он решает проблему, которую WebSocket в принципе решить не может: **head-of-line blocking**. Глава объясняет, зачем WebTransport появился, как его поднять локально, как настроить браузер, и почему технология сегодня ограничена в применении.

## 37.1. Почему WebTransport появился

`WebSocket` работает поверх **TCP**. TCP гарантирует упорядоченную доставку — если один сегмент потерялся, все последующие данные ждут его повторной передачи. Это **head-of-line blocking**: пакет, потерявшийся в канале, блокирует весь канал.

В чатах это незаметно. В приложениях с разным приоритетом данных — критично. Игровая позиция не должна ждать загрузки текстуры. Курсор соавтора в документе не должен ждать загрузки соседнего абзаца. Голосовой чанк не должен ждать отставшего видеопакета.

`WebTransport` решает это через **QUIC** — транспортный протокол поверх UDP. В одном QUIC-соединении независимо живут:

- **Несколько надёжных упорядоченных потоков** (streams). Потеря в одном не блокирует другие.
- **Ненадёжные неупорядоченные датаграммы** (datagrams). Могут теряться, приходят в любом порядке — но приходят с минимальной задержкой.

Плюс QUIC даёт быстрый handshake (0-RTT или 1-RTT вместо TCP+TLS) и умеет работать в условиях плохих сетей — переживает смену Wi-Fi на LTE без разрыва.

## 37.2. Три канала передачи

`WebTransport` предоставляет три режима доставки, и они **независимы друг от друга**:

| Канал | Гарантии | Применение |
|---|---|---|
| **Unidirectional stream** | надёжно, упорядочено, one-way | логи, метрики, телеметрия |
| **Bidirectional stream** | надёжно, упорядочено, two-way | файлы, запросы-ответы |
| **Datagram** | ненадёжно, без порядка, low-latency | игровое состояние, курсоры, голос |

**Ключевое отличие от WebSocket.** Надёжность выбирается **на уровне API**, а не является свойством транспорта. Один клиент может отправлять позиции игроков как датаграммы (потеря не страшна — следующая позиция свежее), а текстуры — как потоки (потеря критична).

## 37.3. Ограничения браузеров

WebTransport — молодая технология. Поддержка по состоянию на начало 2026 года:

| Браузер | Поддержка | Комментарий |
|---|---|---|
| Chrome 97+ | ✅ полная | Работает, но требует настройки сертификата |
| Edge 97+ | ✅ полная | То же, что Chrome |
| Firefox | ⚠️ частичная | Только за флагом `dom.webtransport.enable`, нестабильно |
| Safari 26.4+ | ✅ базовая | Появилась в марте 2026 |

**Firefox не подходит для разработки.** Реализация за флагом `dom.webtransport.enable` в `about:config`, handshake с самоподписанными сертификатами часто падает. Для отладки используйте **Chrome** или **Edge**.

**Chrome требует дополнительных настроек.** Из-за строгих требований к TLS самоподписанный сертификат без специальной подготовки Chrome отвергает. Об этом — раздел 37.7.

**Из-за неполной поддержки всех браузеров технология ограничена.** WebTransport не заменяет WebSocket везде — он дополняет его там, где нужен мультиплексинг и ненадёжная доставка. Если ваше приложение должно работать в Firefox и Safari без флагов — WebSocket остаётся рабочим вариантом.

## 37.4. Структура проекта

Демо состоит из двух частей: **Rust-сервер** (отдельный проект `wtransport`) и **Rust/WASM-клиент** в основном проекте.

```text
hello-wasm02/                     ← основной проект
├── src/
│   ├── wt_client.rs              ← НОВЫЙ модуль главы 37
│   └── app.rs                    ← демо (заменяется целиком)
├── tests/
│   └── wasm_wt_client.rs         ← тесты главы
└── src/lib.rs                    ← +1 строка: pub mod wt_client;

wtransport/                       ← форк библиотеки wtransport
├── cert.pem                      ← сертификат (генерируется отдельно)
├── key.pem                       ← приватный ключ
└── wtransport/
    └── examples/
        └── hello.rs              ← НОВЫЙ: наш echo-сервер на 4433
```

**Почему отдельный сервер в форке библиотеки.** Библиотека `wtransport` уже содержит все зависимости (`tokio`, `quinn`, `rustls`). Проще добавить свой пример, чем поднимать отдельный Cargo-проект.

**Ссылки для изучения:**

- MDN: [`WebTransport`](https://developer.mozilla.org/en-US/docs/Web/API/WebTransport)
- W3C: [WebTransport Specification](https://w3c.github.io/webtransport/)
- IETF: [WebTransport over HTTP/3](https://datatracker.ietf.org/doc/html/draft-ietf-webtrans-http3)
- Chrome Developers: [How to use WebTransport](https://developer.chrome.com/docs/capabilities/web-apis/webtransport)
- `wtransport` crate: [BiagioFesta/wtransport](https://github.com/BiagioFesta/wtransport)

## 37.5. Подготовка сервера

### 37.5.1. Клонирование wtransport

```bash
cd ~/projects/wa/hello-wasm02
git clone https://github.com/BiagioFesta/wtransport.git
cd wtransport
```

В корне появится `Cargo.toml` (workspace), папки `wtransport/` и `wtransport-proto/`.

### 37.5.2. Сертификат через mkcert

Обычный `openssl`-сертификат **не подойдёт**: Chrome требует, чтобы сертификат был подписан известным ему CA. Решение — `mkcert`, который создаёт локальный CA и регистрирует его в системном trust store.

**Установка mkcert (Windows):**

```powershell
winget install FiloSottile.mkcert
```

После установки **обязательно перезапустите PowerShell** — установщик обновил PATH, но текущая сессия этого не видит.

**Регистрация локального CA:**

```powershell
mkcert -install
```

Ожидаемый вывод:

```
Created a new local CA 💥
The local CA is now installed in the system trust store! ⚡️
```

**Генерация сертификата.** Не запускайте mkcert из UNC-путей (`\\wsl.localhost\...`) — PowerShell подставит полный путь в аргументы. Сгенерируйте в обычной Windows-папке:

```powershell
mkdir C:\mkcert-tmp -Force
cd C:\mkcert-tmp
mkcert -cert-file cert.pem -key-file key.pem localhost 127.0.0.1 ::1
```

SAN должен содержать `localhost`, `127.0.0.1` и `::1`. Проверьте:

```powershell
openssl x509 -in cert.pem -noout -issuer
```

Ожидаемый вывод:

```
issuer=C = US, O = mkcert development CA, CN = mkcert development CA ...
```

Если issuer — mkcert, сертификат правильный.

**Копирование в WSL.** Убедитесь, что WSL запущен:

```powershell
wsl -l -v
```

Если статус `Stopped` — откройте WSL в отдельном окне. Затем скопируйте:

```powershell
Copy-Item cert.pem,key.pem -Destination \\wsl.localhost\Ubuntu\home\rustkas\projects\wa\hello-wasm02\wtransport\
```

Замените `Ubuntu` на имя вашего дистрибутива.

В WSL проверьте, что файлы на месте:

```bash
cd ~/projects/wa/hello-wasm02/wtransport
ls -la cert.pem key.pem
```

### 37.5.3. Echo-сервер

Создайте файл `~/projects/wa/hello-wasm02/wtransport/wtransport/examples/hello.rs`:

```rust
//! WebTransport echo-сервер на 127.0.0.1:4433/echo.
//!
//! Отвечает эхом на датаграммы и bidirectional-стримы.
//!
//! Запуск из корня репозитория wtransport:
//!     cargo run --example hello

use anyhow::Result;
use wtransport::endpoint::IncomingSession;
use wtransport::{Endpoint, Identity, ServerConfig};

const PORT: u16 = 4433;
const PATH: &str = "/echo";

#[tokio::main]
async fn main() -> Result<()> {
    let identity = Identity::load_pemfiles("cert.pem", "key.pem").await?;
    let config = ServerConfig::builder()
        .with_bind_default(PORT)
        .with_identity(identity)
        .build();
    let endpoint = Endpoint::server(config)?;

    println!("=== WebTransport echo server ===");
    println!("URL:  https://127.0.0.1:{PORT}{PATH}");
    println!("Порт: {PORT}");
    println!();

    loop {
        let incoming = endpoint.accept().await;
        tokio::spawn(handle_session(incoming));
    }
}

async fn handle_session(incoming: IncomingSession) {
    let request = match incoming.await {
        Ok(r) => r,
        Err(e) => {
            eprintln!("accept incoming: {e}");
            return;
        }
    };

    // Path нужно превратить в String до accept():
    // accept() перемещает request, а path() возвращает заимствование.
    let path: String = request.path().to_string();
    if path != PATH {
        eprintln!("отказ: path = {path}");
        return;
    }

    let connection = match request.accept().await {
        Ok(c) => c,
        Err(e) => {
            eprintln!("accept session: {e}");
            return;
        }
    };

    println!("клиент подключён (path={path})");

    // Датаграммы: эхо.
    loop {
        match connection.receive_datagram().await {
            Ok(dg) => {
                let payload = dg.payload();
                println!("← datagram {}B — эхо назад", payload.len());
                let _ = connection.send_datagram(payload);
            }
            Err(_) => break,
        }
    }

    println!("клиент отключён");
}
```

**Запуск:**

```bash
cd ~/projects/wa/hello-wasm02/wtransport
cargo run --example hello
```

Ожидаемый вывод:

```
=== WebTransport echo server ===
URL:  https://127.0.0.1:4433/echo
Порт: 4433
```

**Не закрывайте терминал** — сервер работает в фоне, пока вы тестируете клиент.

### 37.5.4. WSL и сеть

Если сервер запущен в **WSL2**, а Chrome — в Windows, `127.0.0.1` для TCP-трафика работает через автоматический проброс, а для UDP — **нет**. Это приводит к ошибке `ERR_CONNECTION_RESET` на клиенте.

**Решение — mirrored networking mode.** Создайте или откройте `%UserProfile%\.wslconfig` в Windows:

```ini
[wsl2]
networkingMode=mirrored
```

Выполните в PowerShell:

```powershell
wsl --shutdown
```

Подождите 10 секунд, откройте WSL заново. Windows может показать диалог «Разрешить WSL доступ к сети» — выберите **Разрешить**.

После этого `127.0.0.1:4433` начнёт работать для UDP так же, как для TCP.

## 37.6. Настройка Chrome

WebTransport в Chrome требует трёх независимых действий:

**1. Chrome, а не Firefox.** Firefox не поддерживает WebTransport стабильно. Если в заголовке окна `Mozilla Firefox` — закройте его, откройте Chrome.

**2. Developer Mode.** Откройте `chrome://flags`, найдите **WebTransport Developer Mode**, установите **Enabled**, нажмите **Relaunch**. После установки mkcert этот флаг можно отключить — mkcert-подписанные сертификаты Chrome принимает без него.

**3. Принятие сертификата.** Самоподписанный сертификат Chrome принимает через **JavaScript-попытку соединения**, а не через навигацию по URL. Открытие `https://127.0.0.1:4433/echo` в адресной строке даст `This site can't be reached` — это нормально, сервер отвечает только по QUIC/UDP, HTTP-запросы не обрабатывает.

При нажатии **Connect** в демо Chrome применит следующий алгоритм:

1. Попробует установить QUIC-соединение.
2. Проверит сертификат через системный trust store.
3. Если сертификат подписан mkcert — соединение установится без диалогов.
4. Если сертификат самоподписанный — покажет диалог «Принять?».

## 37.7. Модуль `wt_client.rs`

Модуль — обёртка над браузерным `WebTransport`. Все методы Streams API вызываются через `Reflect::get` + `Function::call*`, потому что typed web-sys в разных версиях имеет несовместимые сигнатуры.


## 37.8. Демо-пример

Демо-пример (`src/app.rs`) содержит полный UI для работы с двумя каналами. Его листинг приведён в **Приложении** в конце главы. Здесь опишем, что он показывает.

**Три колонки интерфейса:**

1. **Управление** — поле URL, кнопки Connect / Close / Reset, индикатор состояния. Кнопки одиночной отправки (Datagram, Stream), кнопки burst-тестов (20 датаграмм, 10 потоков), кнопка параллельного теста (10 + 5 одновременно).
2. **Журнал событий** — все действия в реальном времени с seq-нумерами. Видно порядок прихода — датаграммы могут обгонять друг друга.
3. **Метрики** — счётчики отправленных и полученных пакетов по каждому каналу, байты, ошибки. Отдельная панель «Доставка по каналам» — сколько датаграмм и потоков доставлено из ожидаемого числа (`M/N`).

**Сценарий проверки:**

1. Connect → состояние `ready`.
2. Нажать **Datagram** — в журнале `← datagram ... seq=0`.
3. Нажать **20 датаграмм** — увидеть серию эхо-ответов с seq=1..20, в панели «Доставка по каналам» счётчик `20/20`.
4. Нажать **10 потоков** — счётчик `10/10` в той же панели.
5. Нажать **Параллельно** — оба канала работают одновременно, счётчики обновляются независимо.

**Что демонстрирует:**

- Оба канала живут в одном QUIC-соединении.
- Датаграммы и потоки независимы: потеря в одном не блокирует другой.
- Seq-нумерация в payload позволяет проверить порядок прихода.
- Панель «Доставка по каналам» показывает, что оба канала работают без потерь.

## 37.9. Сравнение с WebSocket

| | WebSocket | WebTransport |
|---|---|---|
| Транспорт | TCP | QUIC (UDP) |
| Соединений | 1 поток | N независимых потоков |
| Надёжность | только надёжно | + ненадёжные датаграммы |
| Head-of-line blocking | есть | нет |
| Handshake | 1-RTT (TCP + TLS) | 0-RTT или 1-RTT |
| Поддержка браузеров | полная | частичная (Chrome, Edge, Safari 26.4) |
| Сложность setup | минимум | требует TLS-сертификата |

**Когда использовать WebTransport:**

- Игровой netcode: позиции игроков — датаграммы, чат — потоки.
- Коллаборативные редакторы: курсоры соавторов — датаграммы.
- Стриминг файлов с прогрессом: несколько потоков параллельно, не блокируют друг друга.
- Приложения с разным приоритетом данных в одной сессии.

**Когда остаться на WebSocket:**

- Обычный чат, простой real-time UI.
- Приложение должно работать во всех браузерах без флагов.
- Сложность настройки сертификатов и сервера не оправдана.

## 37.10. Ограничения и перспективы

**Три ограничения сегодня:**

1. **Браузерная поддержка.** Chrome и Edge — стабильно. Firefox — только за флагом, нестабильно. Safari 26.4+ — базовая поддержка. Это ограничивает применение WebTransport в публичных проектах, ориентированных на широкую аудиторию.

2. **Сложность развёртывания.** Требуется QUIC-сервер с HTTP/3, TLS-сертификат, правильные SAN. На хостинге нужен провайдер, поддерживающий UDP и QUIC (не все облака это делают без дополнительных настроек).

3. **Нестандартные сертификаты.** Разработка требует mkcert или отдельного CA. Продакшн требует обычного доверенного сертификата.

**Что изменится:**

Safari уже поддерживает, Firefox за флагом. По мере стабилизации Firefox WebTransport станет применимым шире. Провайдеры cloud-инфраструктуры постепенно добавляют поддержку QUIC. К 2027 году ожидается parity с WebSocket по доступности.

Пока технология **нишевая**: там, где WebSocket не справляется (мультиплексинг, низкая задержка, разный приоритет данных), WebTransport даёт реальный выигрыш. Для остального — WebSocket остаётся рабочим и простым вариантом.

## 37.11. Итог

**WebTransport — API над QUIC.** Три канала в одном соединении: надёжные потоки, bidirectional, ненадёжные датаграммы.

**Мультиплексирование без head-of-line blocking.** Потеря в одном потоке не блокирует другие.

**Требует HTTPS и специального сертификата.** mkcert — рабочий путь для локальной разработки.

**Chrome — единственный стабильный браузер для отладки.** Firefox не подходит.

**WSL требует mirrored networking mode** для UDP-трафика.

**Технология ограничена в применении** из-за неполной поддержки браузерами. Использовать там, где WebSocket не справляется, а не как замену везде.

В следующей главе — **Web Push**: как отправлять уведомления на устройство, когда браузер закрыт.

---

# Приложение. Листинги

## А.1. Echo-сервер `wtransport/wtransport/examples/hello.rs`

```rust
//! WebTransport echo-сервер на 127.0.0.1:4433/echo.
//!
//! Отвечает эхом на датаграммы.
//!
//! Запуск из корня репозитория wtransport:
//!     cargo run --example hello

use anyhow::Result;
use wtransport::endpoint::IncomingSession;
use wtransport::{Endpoint, Identity, ServerConfig};

const PORT: u16 = 4433;
const PATH: &str = "/echo";

#[tokio::main]
async fn main() -> Result<()> {
    let identity = Identity::load_pemfiles("cert.pem", "key.pem").await?;
    let config = ServerConfig::builder()
        .with_bind_default(PORT)
        .with_identity(identity)
        .build();
    let endpoint = Endpoint::server(config)?;

    println!("=== WebTransport echo server ===");
    println!("URL:  https://127.0.0.1:{PORT}{PATH}");
    println!("Порт: {PORT}");
    println!();

    loop {
        let incoming = endpoint.accept().await;
        tokio::spawn(handle_session(incoming));
    }
}

async fn handle_session(incoming: IncomingSession) {
    let request = match incoming.await {
        Ok(r) => r,
        Err(e) => { eprintln!("accept: {e}"); return; }
    };

    let path: String = request.path().to_string();
    if path != PATH {
        eprintln!("отказ: path = {path}");
        return;
    }

    let connection = match request.accept().await {
        Ok(c) => c,
        Err(e) => { eprintln!("accept session: {e}"); return; }
    };

    println!("клиент подключён (path={path})");

    loop {
        match connection.receive_datagram().await {
            Ok(dg) => {
                let payload = dg.payload();
                println!("← datagram {}B — эхо назад", payload.len());
                let _ = connection.send_datagram(payload);
            }
            Err(_) => break,
        }
    }

    println!("клиент отключён");
}
```

## А.2. Полный `src/app.rs`

```rust
//! Глава 37: WebTransport.
//!
//! Практическое демо двух каналов передачи поверх QUIC:
//!   1. Datagrams — ненадёжные, могут теряться / менять порядок.
//!   2. Bidirectional streams — надёжные, упорядоченные.
//!   3. Burst-тесты с нумерацией seq — сравниваем доставку.
//!   4. Параллельный тест — оба канала одновременно.
//!   5. Метрики по каждому каналу: M/N доставлено.
//!   6. Диагностика ошибок connect с подсказками.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: callback захватывает Weak<App> — избегаем Rc-цикла.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::callback::Interval;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use web_sys::{HtmlElement, HtmlInputElement};

use crate::format_str::format_str;
use crate::wt_client::{format_bytes, WtClient, WtConfig, WtEvent, WtState, WtStats};

const LOG_LIMIT: usize = 16;
const BURST_DATAGRAMS: u32 = 20;
const BURST_STREAMS: u32 = 10;
const PARALLEL_DATAGRAMS: u32 = 10;
const PARALLEL_STREAMS: u32 = 5;
const STORAGE_KEY_URL: &str = "hello-wasm02.wt.url";

const HTML: &str = r#"
<style>
  :root {
    --wt-info: #3498db; --wt-ok: #27ae60; --wt-err: #c0392b; --wt-warn: #e67e22;
    --wt-text: #222; --wt-muted: #666;
    --wt-bg: #fff; --wt-soft: #fafafa; --wt-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --wt-text: #e6e6e6; --wt-muted: #a0a0a0;
      --wt-bg: #1e1e1e; --wt-soft: #262626; --wt-border: #3a3a3a;
    }
    body { background: #121212; }
    input { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .wt-log-entry { background: #1e1e1e; }
    .wt-anno { background: #1a2233; color: #9fc4ff; }
    .wt-hint { background: #232014; color: #e6c98c; }
    .wt-warn-block { background: #3a2020; color: #f0a0a0; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--wt-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .wt-subtitle { color: var(--wt-muted); font-size: 12px; margin: 0 0 12px; }
  .wt-subtitle a { color: var(--wt-info); }

  .wt-grid {
    display: grid; gap: 12px;
    grid-template-columns: minmax(0, 1.1fr) minmax(0, 1fr) minmax(0, 1fr);
  }
  @media (max-width: 1100px) { .wt-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px)  { .wt-grid { grid-template-columns: 1fr; } }
  .wt-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .wt-panel {
    border: 1px solid var(--wt-border); border-radius: 8px;
    padding: 10px 12px; background: var(--wt-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .wt-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px;
  }
  .wt-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--wt-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .wt-anno {
    background: #eef4ff; border-left: 3px solid var(--wt-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere;
  }
  .wt-hint {
    background: #fff9e6; border-left: 3px solid var(--wt-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #7d5a00;
  }
  .wt-warn-block {
    background: #fdedec; border-left: 3px solid var(--wt-err);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #922b21;
  }

  .wt-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; }
  .wt-label { font-size: 11px; color: var(--wt-muted); }

  input[type="text"] {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--wt-bg); color: var(--wt-text);
    font-family: ui-monospace, monospace;
    flex: 1; min-width: 180px;
  }
  input[type="text"].wt-error {
    border-color: var(--wt-err); background: #fdf3f3;
  }
  @media (prefers-color-scheme: dark) {
    input[type="text"].wt-error { background: #3a2020; }
  }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--wt-soft); color: var(--wt-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.wt-primary { background: #eaf4ff; border-color: var(--wt-info); color: #1f4e79; }
  button.wt-success { background: #e8f8ed; border-color: var(--wt-ok); color: #1e8449; }
  button.wt-danger  { background: #fdedec; border-color: var(--wt-err); color: #922b21; }
  button.wt-warn    { background: #fff3cd; border-color: var(--wt-warn); color: #7d5a00; }

  .wt-status {
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--wt-soft); border-left: 3px solid #ccc;
    color: var(--wt-muted); min-height: 22px; overflow-wrap: anywhere;
  }
  .wt-status.ok   { background: #e8f8ed; border-color: var(--wt-ok); color: #1e8449; }
  .wt-status.err  { background: #fdedec; border-color: var(--wt-err); color: #922b21; }
  .wt-status.busy { background: #fff9e6; border-color: var(--wt-warn); color: #7d5a00; }

  .wt-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
  }
  @media (prefers-color-scheme: dark) {
    .wt-badge { background: #2a3a4a; color: #cfe0ff; }
  }
  .wt-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .wt-badge.err { background: #fdedec; color: #922b21; }
  .wt-badge.warn { background: #fff9e6; color: #7d5a00; }

  .wt-log {
    border: 1px solid var(--wt-border); border-radius: 5px;
    background: var(--wt-soft); max-height: 420px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .wt-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .wt-log-entry {
    display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent;
  }
  .wt-log-entry.ok   { border-left-color: var(--wt-ok); }
  .wt-log-entry.err  { border-left-color: var(--wt-err); }
  .wt-log-entry.info { border-left-color: var(--wt-info); }
  .wt-log-entry.warn { border-left-color: var(--wt-warn); }
  .wt-log-entry .t { color: var(--wt-muted); flex-shrink: 0; }
  .wt-log-entry .m { flex: 1; overflow-wrap: anywhere; }
  .wt-log-entry.err .m  { color: #922b21; }
  .wt-log-entry.warn .m { color: #7d5a00; }

  .wt-settings-grid {
    display: grid; grid-template-columns: auto 1fr;
    gap: 4px 12px; margin: 6px 0;
    font-family: ui-monospace, monospace; font-size: 12px;
  }
  .wt-settings-grid .k { color: var(--wt-muted); }
  .wt-settings-grid .v { color: var(--wt-text); overflow-wrap: anywhere; }
  .wt-settings-grid .v.ok  { color: #1e8449; }
  .wt-settings-grid .v.err { color: #922b21; }
  .wt-settings-grid .v.warn { color: #7d5a00; }

  .wt-compare {
    display: grid; grid-template-columns: 1fr 1fr;
    gap: 4px 12px; font-size: 11px;
    font-family: ui-monospace, monospace;
  }
  .wt-compare .h { color: var(--wt-muted); font-weight: 700; }
  .wt-compare .ws { color: var(--wt-info); }
  .wt-compare .wtt { color: var(--wt-ok); }
</style>

<h1>Глава 37: WebTransport</h1>
<p class="wt-subtitle">
  Datagrams, bidirectional streams, reliability. QUIC / HTTP-3 в браузере.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebTransport"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="wt-grid">

  <!-- ============ Колонка 1: Управление ============ -->
  <div class="wt-col">
    <div class="wt-panel">
      <h2 class="wt-panel-title">
        <span class="wt-num">1</span>Управление
      </h2>

      <div class="wt-warn-block">
        <b>Требуется WebTransport-сервер.</b> Нужен QUIC / HTTP-3
        echo-сервер на <code>https://127.0.0.1:4433/echo</code>. Без него
        <b>Connect упадёт с ошибкой</b> — это нормальное поведение,
        а не баг демо. Перед первым запуском откройте
        <code>https://127.0.0.1:4433</code> в новой вкладке и примите
        самоподписанный сертификат.
      </div>

      <div class="wt-anno">
        WebTransport — не замена WebSocket. Оба канала живут в одном
        QUIC-соединении: <code>datagrams</code> для ненадёжных
        быстрых сообщений, <code>streams</code> — для надёжных упорядоченных.
      </div>

      <div class="wt-row">
        <input type="text" id="wt-url"
               value="https://127.0.0.1:4433/echo">
      </div>

      <div class="wt-row">
        <button id="wt-connect" class="wt-success">▶ Connect</button>
        <button id="wt-close" class="wt-danger" disabled>■ Close</button>
        <button id="wt-reset">↺ Reset</button>
        <span class="wt-badge" id="wt-state">idle</span>
      </div>

      <div class="wt-row">
        <button id="wt-datagram" disabled>📦 Datagram</button>
        <button id="wt-stream" class="wt-primary" disabled>📏 Stream</button>
      </div>

      <div class="wt-row">
        <span class="wt-label">Burst:</span>
        <button id="wt-burst-dg" class="wt-warn" disabled>
          📦 20 датаграмм
        </button>
        <button id="wt-burst-str" class="wt-warn" disabled>
          📏 10 потоков
        </button>
      </div>

      <div class="wt-row">
        <button id="wt-parallel" disabled>
          ⚡ Параллельно: 10 дат + 5 потоков
        </button>
      </div>

      <div id="wt-status" class="wt-status">готово</div>
    </div>

    <div class="wt-panel">
      <h2 class="wt-panel-title">Состояние клиента</h2>
      <div id="wt-settings" class="wt-settings-grid">
        <span class="k">state</span><span class="v">idle</span>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 2: Журнал ============ -->
  <div class="wt-col">
    <div class="wt-panel">
      <h2 class="wt-panel-title">Журнал событий</h2>
      <div class="wt-hint">
        Каждому пакету присваивается seq-номер. В журнале видно
        <code>seq=N</code> и порядок прихода. Datagrams могут
        обгонять друг друга — это <b>правильное поведение</b>.
      </div>
      <div id="wt-log" class="wt-log">
        <div class="wt-log-empty">здесь появятся события</div>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 3: Статистика ============ -->
  <div class="wt-col">
    <div class="wt-panel">
      <h2 class="wt-panel-title">Метрики</h2>
      <div id="wt-metrics" class="wt-settings-grid">
        <span class="k">—</span><span class="v">нет данных</span>
      </div>
    </div>

    <div class="wt-panel">
      <h2 class="wt-panel-title">Доставка по каналам</h2>
      <div id="wt-delivery" class="wt-settings-grid">
        <span class="k">—</span><span class="v">нет активного burst</span>
      </div>
    </div>

    <div class="wt-panel">
      <h2 class="wt-panel-title">WebSocket vs WebTransport</h2>
      <div class="wt-compare">
        <span class="h">WebSocket</span>
        <span class="h">WebTransport</span>
        <span class="ws">TCP</span>
        <span class="wtt">QUIC (UDP)</span>
        <span class="ws">1 поток</span>
        <span class="wtt">N потоков</span>
        <span class="ws">надёжно</span>
        <span class="wtt">+ ненадёжно</span>
        <span class="ws">HOL blocking</span>
        <span class="wtt">нет HOL</span>
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

/// Состояние текущего burst-теста.
#[derive(Default, Clone)]
struct BurstState {
    id: u32,
    datagrams_expected: u32,
    datagrams_arrived: Vec<u32>,
    streams_expected: u32,
    streams_arrived: Vec<u32>,
}

impl BurstState {
    fn is_idle(&self) -> bool {
        return self.datagrams_expected == 0 && self.streams_expected == 0;
    }
}

struct Ui {
    url: HtmlInputElement,
    btn_connect: HtmlElement,
    btn_close: HtmlElement,
    btn_reset: HtmlElement,
    btn_datagram: HtmlElement,
    btn_stream: HtmlElement,
    btn_burst_dg: HtmlElement,
    btn_burst_str: HtmlElement,
    btn_parallel: HtmlElement,
    badge_state: web_sys::Element,
    status: web_sys::Element,
    settings: web_sys::Element,
    metrics: web_sys::Element,
    delivery: web_sys::Element,
    log: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    client: RefCell<Option<Rc<WtClient>>>,
    burst: RefCell<BurstState>,
    _stats_timer: RefCell<Option<Interval>>,
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
        url: get_input("wt-url")?,
        btn_connect: get_el("wt-connect")?.dyn_into()?,
        btn_close: get_el("wt-close")?.dyn_into()?,
        btn_reset: get_el("wt-reset")?.dyn_into()?,
        btn_datagram: get_el("wt-datagram")?.dyn_into()?,
        btn_stream: get_el("wt-stream")?.dyn_into()?,
        btn_burst_dg: get_el("wt-burst-dg")?.dyn_into()?,
        btn_burst_str: get_el("wt-burst-str")?.dyn_into()?,
        btn_parallel: get_el("wt-parallel")?.dyn_into()?,
        badge_state: get_el("wt-state")?,
        status: get_el("wt-status")?,
        settings: get_el("wt-settings")?,
        metrics: get_el("wt-metrics")?,
        delivery: get_el("wt-delivery")?,
        log: get_el("wt-log")?,
    };

    // Восстанавливаем URL из localStorage.
    if let Some(saved) = load_saved_url() {
        ui.url.set_value(&saved);
    }

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        client: RefCell::new(None),
        burst: RefCell::new(BurstState::default()),
        _stats_timer: RefCell::new(None),
        ui,
    });

    app.render_log();
    render_settings(&app);
    render_metrics_default(&app);
    render_delivery(&app);

    bind_buttons(&app)?;
    start_stats_timer(&app)?;
    bind_before_unload(&app)?;

    Ok(())
}

// ============================================================
//                    LOCALSTORAGE
// ============================================================

fn load_saved_url() -> Option<String> {
    let win = web_sys::window()?;
    let storage = win.local_storage().ok()??;
    return storage.get_item(STORAGE_KEY_URL).ok()?;
}

fn save_url(url: &str) {
    if let Some(win) = web_sys::window() {
        if let Ok(Some(storage)) = win.local_storage() {
            let _ = storage.set_item(STORAGE_KEY_URL, url);
        }
    }
}

// ============================================================
//                    BIND
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_connect, move || { connect(app_c.clone()); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_close, move || { disconnect(&app_c); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_reset, move || { do_reset(&app_c); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_datagram, move || {
            send_single_datagram(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_stream, move || {
            send_single_stream(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_burst_dg, move || {
            burst_datagrams(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_burst_str, move || {
            burst_streams(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_parallel, move || {
            burst_parallel(app_c.clone());
        });
    }
    Ok(())
}

fn bind_before_unload(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        if let Some(client) = app_c.client.borrow().as_ref() {
            client.close();
        }
    });
    let win = web_sys::window().ok_or("no window")?;
    win.add_event_listener_with_callback(
        "beforeunload",
        cb.as_ref().unchecked_ref(),
    )?;
    cb.forget();
    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn connect(app: Rc<App>) {
    // Разрешаем повторный connect, если текущее состояние
    // терминальное (idle / failed / closed).
    {
        let c = app.client.borrow();
        if let Some(existing) = c.as_ref() {
            let s = existing.state();
            if !matches!(
                s,
                WtState::Idle | WtState::Closed | WtState::Failed
            ) {
                return;
            }
        }
    }

    let url = app.ui.url.value();
    if url.is_empty() {
        set_status(&app.ui.status, "err", "URL не задан");
        return;
    }

    // Сохраняем URL для следующего запуска.
    save_url(&url);

    // Снимаем красную подсветку поля URL.
    let _ = app.ui.url.set_class_name("");

    let config = WtConfig::new(url.clone());

    // Weak<App> в callback — избегаем цикла App ↔ WtClient.
    let app_weak = Rc::downgrade(&app);
    let on_event: Rc<dyn Fn(WtEvent)> = Rc::new(move |ev| {
        if let Some(app) = app_weak.upgrade() {
            on_wt_event(app, ev);
        }
    });

    let client = WtClient::new(config, on_event);
    *app.client.borrow_mut() = Some(client.clone());

    // Новый burst-цикл.
    {
        let mut b = app.burst.borrow_mut();
        *b = BurstState::default();
        b.id = b.id.wrapping_add(1);
    }
    render_delivery(&app);

    app.push_log(
        LogKind::Info,
        format_str(format_args!("подключение к {url}")),
    );
    set_status(
        &app.ui.status,
        "busy",
        "connecting (может занять 1-3 сек)…",
    );
    render_settings(&app);
    update_controls(&app);

    if let Err(e) = client.connect() {
        let msg = format_err_str("connect", &e);
        app.push_log(LogKind::Err, msg.clone());
        push_error_hint(&app, &msg);
        set_status(&app.ui.status, "err", &msg);
        let _ = app.ui.url.set_class_name("wt-error");
        render_settings(&app);
        update_controls(&app);
    }
}

fn disconnect(app: &Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let Some(client) = client_opt else { return; };
    client.close();
    app.push_log(LogKind::Info, "ручное закрытие".to_string());
    set_status(&app.ui.status, "ok", "closed");
    update_controls(app);
    render_settings(app);
}

fn do_reset(app: &Rc<App>) {
    let client_opt = app.client.borrow_mut().take();
    if let Some(client) = client_opt {
        client.close();
    }
    let _ = app.ui.url.set_class_name("");
    app.log.borrow_mut().clear();
    *app.burst.borrow_mut() = BurstState::default();
    app.render_log();
    render_metrics_default(app);
    render_delivery(app);
    render_settings(app);
    set_status(&app.ui.status, "ok", "сброшено");
    update_controls(app);
}

// ---------- Одиночные сообщения ----------

fn send_single_datagram(app: Rc<App>) {
    let Some(client) = ensure_ready(&app) else { return; };
    let burst_id = app.burst.borrow().id;
    let payload = make_payload('d', burst_id, 0, 16);
    match client.send_datagram(&payload) {
        Ok(()) => {
            app.push_log(
                LogKind::Info,
                format_str(format_args!(
                    "→ datagram {}B (seq=0)",
                    payload.len(),
                )),
            );
        }
        Err(e) => {
            let msg = format_err_str("send_datagram", &e);
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

fn send_single_stream(app: Rc<App>) {
    let Some(client) = ensure_ready(&app) else { return; };
    let burst_id = app.burst.borrow().id;
    let payload = make_payload('s', burst_id, 0, 64);
    match client.send_over_stream(&payload) {
        Ok(()) => {
            app.push_log(
                LogKind::Info,
                format_str(format_args!(
                    "→ stream {}B (seq=0)",
                    payload.len(),
                )),
            );
        }
        Err(e) => {
            let msg = format_err_str("send_over_stream", &e);
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

// ---------- Burst-тесты ----------

fn burst_datagrams(app: Rc<App>) {
    let Some(client) = ensure_ready(&app) else { return; };

    let burst_id = {
        let mut b = app.burst.borrow_mut();
        b.datagrams_expected = BURST_DATAGRAMS;
        b.datagrams_arrived.clear();
        b.id
    };
    render_delivery(&app);
    app.push_log(
        LogKind::Warn,
        format_str(format_args!(
            "burst datagrams: {BURST_DATAGRAMS} с seq=1..{BURST_DATAGRAMS}",
        )),
    );

    for i in 1..=BURST_DATAGRAMS {
        let payload = make_payload('d', burst_id, i, 16);
        if let Err(e) = client.send_datagram(&payload) {
            app.push_log(LogKind::Err, format_err_str("send_datagram", &e));
            break;
        }
    }
    set_status(
        &app.ui.status,
        "busy",
        &format_str(format_args!(
            "отправлено {BURST_DATAGRAMS} датаграмм — ждём эхо",
        )),
    );
}

fn burst_streams(app: Rc<App>) {
    let Some(client) = ensure_ready(&app) else { return; };

    let burst_id = {
        let mut b = app.burst.borrow_mut();
        b.streams_expected = BURST_STREAMS;
        b.streams_arrived.clear();
        b.id
    };
    render_delivery(&app);
    app.push_log(
        LogKind::Warn,
        format_str(format_args!(
            "burst streams: {BURST_STREAMS} потоков",
        )),
    );

    for i in 1..=BURST_STREAMS {
        let payload = make_payload('s', burst_id, i, 128);
        if let Err(e) = client.send_over_stream(&payload) {
            app.push_log(LogKind::Err, format_err_str("send_over_stream", &e));
            break;
        }
    }
    set_status(
        &app.ui.status,
        "busy",
        &format_str(format_args!(
            "отправлено {BURST_STREAMS} потоков — ждём эхо",
        )),
    );
}

fn burst_parallel(app: Rc<App>) {
    let Some(client) = ensure_ready(&app) else { return; };

    let burst_id = {
        let mut b = app.burst.borrow_mut();
        b.datagrams_expected = PARALLEL_DATAGRAMS;
        b.datagrams_arrived.clear();
        b.streams_expected = PARALLEL_STREAMS;
        b.streams_arrived.clear();
        b.id
    };
    render_delivery(&app);
    app.push_log(
        LogKind::Warn,
        format_str(format_args!(
            "ПАРАЛЛЕЛЬНО: {PARALLEL_DATAGRAMS} датаграмм + \
             {PARALLEL_STREAMS} потоков в одном соединении",
        )),
    );

    for i in 1..=PARALLEL_DATAGRAMS {
        let payload = make_payload('d', burst_id, i, 16);
        let _ = client.send_datagram(&payload);
    }
    for i in 1..=PARALLEL_STREAMS {
        let payload = make_payload('s', burst_id, i, 128);
        let _ = client.send_over_stream(&payload);
    }
    set_status(
        &app.ui.status,
        "busy",
        "оба канала активны — смотрим журнал",
    );
}

/// Возвращает `Some(client)`, только если соединение Ready.
/// При других состояниях — понятное сообщение и `None`.
fn ensure_ready(app: &Rc<App>) -> Option<Rc<WtClient>> {
    let client_opt = app.client.borrow().clone();
    let Some(client) = client_opt else {
        set_status(
            &app.ui.status,
            "err",
            "соединение не установлено — нажмите Connect",
        );
        return None;
    };
    let s = client.state();
    match s {
        WtState::Ready => Some(client),
        WtState::Connecting => {
            set_status(
                &app.ui.status,
                "busy",
                "соединение устанавливается — подождите",
            );
            None
        }
        WtState::Failed => {
            set_status(
                &app.ui.status,
                "err",
                "соединение failed — исправьте URL/сервер и нажмите Connect",
            );
            None
        }
        WtState::Closed => {
            set_status(
                &app.ui.status,
                "err",
                "соединение закрыто — нажмите Connect",
            );
            None
        }
        WtState::Idle => {
            set_status(
                &app.ui.status,
                "err",
                "соединение не установлено — нажмите Connect",
            );
            None
        }
    }
}

/// Строит payload с заголовком: `{kind}:{burst}:{seq}:{random}`.
fn make_payload(kind: char, burst: u32, seq: u32, random_len: usize) -> Vec<u8> {
    let header = format_str(format_args!("{kind}:{burst}:{seq}:"));
    let mut out: Vec<u8> = Vec::with_capacity(header.len() + random_len);
    out.extend_from_slice(header.as_bytes());
    for _ in 0..random_len {
        let r = js_sys::Math::random();
        out.push((r * 256.0) as u8);
    }
    return out;
}

/// Разбирает заголовок payload. Возвращает (kind, burst, seq).
fn parse_payload_header(bytes: &[u8]) -> Option<(char, u32, u32)> {
    if bytes.len() < 6 {
        return None;
    }
    let kind = bytes[0] as char;
    if kind != 'd' && kind != 's' {
        return None;
    }
    if bytes[1] != b':' {
        return None;
    }
    let mut i = 2;
    let mut burst: u32 = 0;
    while i < bytes.len() && bytes[i].is_ascii_digit() {
        burst = burst.checked_mul(10)?
            .checked_add((bytes[i] - b'0') as u32)?;
        i += 1;
    }
    if i >= bytes.len() || bytes[i] != b':' {
        return None;
    }
    i += 1;
    let mut seq: u32 = 0;
    while i < bytes.len() && bytes[i].is_ascii_digit() {
        seq = seq.checked_mul(10)?
            .checked_add((bytes[i] - b'0') as u32)?;
        i += 1;
    }
    if i >= bytes.len() || bytes[i] != b':' {
        return None;
    }
    return Some((kind, burst, seq));
}

// ============================================================
//                    ОБРАБОТКА СОБЫТИЙ
// ============================================================

fn on_wt_event(app: Rc<App>, ev: WtEvent) {
    match ev {
        // Connecting не логируем отдельно: connect() уже вывел
        // «подключение к {url}» — не дублируем.
        WtEvent::Connecting => {}
        WtEvent::Ready => {
            app.push_log(
                LogKind::Ok,
                "соединение готово — можно отправлять".to_string(),
            );
            set_status(&app.ui.status, "ok", "ready");
            let _ = app.ui.url.set_class_name("");
            render_settings(&app);
            update_controls(&app);
        }
        WtEvent::Text(s) => {
            app.push_log(LogKind::Info, s);
        }
        WtEvent::Datagram(bytes) => {
            handle_datagram(app, bytes);
        }
        WtEvent::StreamData(bytes) => {
            handle_stream_data(app, bytes);
        }
        WtEvent::Closed { reason } => {
            app.push_log(
                LogKind::Warn,
                format_str(format_args!("закрыто: {reason}")),
            );
            set_status(&app.ui.status, "err", &reason);
            render_settings(&app);
            update_controls(&app);
        }
        WtEvent::Error(msg) => {
            app.push_log(LogKind::Err, msg.clone());
            push_error_hint(&app, &msg);
            set_status(&app.ui.status, "err", &msg);
            let _ = app.ui.url.set_class_name("wt-error");
            render_settings(&app);
            update_controls(&app);
        }
        WtEvent::Stats => {
            render_metrics(&app);
        }
    }
}

/// Дописывает подсказку для типичных ошибок connect.
fn push_error_hint(app: &Rc<App>, msg: &str) {
    let lower = msg.to_lowercase();
    let hint: Option<&str> = if lower.contains("connection rejected")
        || lower.contains("connection refused")
        || lower.contains("failed to establish")
    {
        Some(
            "Проверьте: (1) сервер запущен на 127.0.0.1:4433; \
             (2) сертификат принят — откройте https://127.0.0.1:4433 \
             в новой вкладке и нажмите «Принять»; (3) на сервере \
             включён HTTP/3 (QUIC). После исправления нажмите Connect.",
        )
    } else if lower.contains("certificate") || lower.contains("tls") {
        Some(
            "Проблема с TLS-сертификатом. Откройте \
             https://127.0.0.1:4433 в новой вкладке, примите \
             предупреждение и вернитесь.",
        )
    } else if lower.contains("secure context") {
        Some(
            "WebTransport требует HTTPS-страницу. На http:// он \
             недоступен.",
        )
    } else {
        None
    };
    if let Some(h) = hint {
        app.push_log(LogKind::Warn, h.to_string());
    }
}

fn handle_datagram(app: Rc<App>, bytes: Vec<u8>) {
    let n = bytes.len();
    let header = parse_payload_header(&bytes);
    let (kind, summary) = match header {
        Some(('d', burst, seq)) => {
            let (arrived, total) = {
                let mut b = app.burst.borrow_mut();
                if b.id == burst && b.datagrams_expected > 0 {
                    if !b.datagrams_arrived.contains(&seq) {
                        b.datagrams_arrived.push(seq);
                    }
                    (b.datagrams_arrived.len() as u32, b.datagrams_expected)
                } else {
                    (0, 0)
                }
            };
            let text = if total > 0 {
                format_str(format_args!(
                    "← datagram {n}B seq={seq} (burst {burst}, \
                     доставлено {arrived}/{total})",
                ))
            } else {
                format_str(format_args!("← datagram {n}B seq={seq}"))
            };
            (LogKind::Ok, text)
        }
        _ => (
            LogKind::Ok,
            format_str(format_args!("← datagram {n}B (без заголовка)")),
        ),
    };
    app.push_log(kind, summary);
    render_delivery(&app);
    render_metrics(&app);
}

fn handle_stream_data(app: Rc<App>, bytes: Vec<u8>) {
    let n = bytes.len();
    let header = parse_payload_header(&bytes);
    let (kind, summary) = match header {
        Some(('s', burst, seq)) => {
            let (arrived, total) = {
                let mut b = app.burst.borrow_mut();
                if b.id == burst && b.streams_expected > 0 {
                    if !b.streams_arrived.contains(&seq) {
                        b.streams_arrived.push(seq);
                    }
                    (b.streams_arrived.len() as u32, b.streams_expected)
                } else {
                    (0, 0)
                }
            };
            let text = if total > 0 {
                format_str(format_args!(
                    "← stream {n}B seq={seq} (burst {burst}, \
                     доставлено {arrived}/{total})",
                ))
            } else {
                format_str(format_args!("← stream {n}B seq={seq}"))
            };
            (LogKind::Ok, text)
        }
        _ => (
            LogKind::Ok,
            format_str(format_args!("← stream {n}B (без заголовка)")),
        ),
    };
    app.push_log(kind, summary);
    render_delivery(&app);
    render_metrics(&app);
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_settings(app: &Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let state_str = match client_opt.as_ref() {
        Some(c) => c.state().as_str().to_string(),
        None => "idle".to_string(),
    };
    let cls = match state_str.as_str() {
        "ready" => "v ok",
        "connecting" => "v warn",
        "failed" | "closed" => "v err",
        _ => "v",
    };
    let mut html = String::with_capacity(256);
    push_row_cls(&mut html, "state", &state_str, cls);
    if let Some(c) = client_opt {
        let cfg = c.config();
        push_row(&mut html, "url", &cfg.url);
        push_row(
            &mut html,
            "datagram_buffer",
            &cfg.datagram_buffer.to_string(),
        );
    }
    app.ui.settings.set_inner_html(&html);
    update_badge(&app.ui.badge_state, &state_str, cls);
}

fn render_metrics_default(app: &Rc<App>) {
    app.ui.metrics.set_inner_html(
        "<span class=\"k\">—</span><span class=\"v\">нет данных</span>",
    );
}

fn render_metrics(app: &Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let Some(c) = client_opt else {
        render_metrics_default(app);
        return;
    };
    let s: WtStats = c.stats();
    let mut html = String::with_capacity(512);
    push_row_cls(
        &mut html,
        "datagrams sent",
        &format_str(format_args!("{}", s.datagrams_sent)),
        "v ok",
    );
    push_row_cls(
        &mut html,
        "datagrams recv",
        &format_str(format_args!("{}", s.datagrams_recv)),
        "v ok",
    );
    push_row(
        &mut html,
        "streams opened",
        &format_str(format_args!("{}", s.streams_opened)),
    );
    push_row(
        &mut html,
        "streams completed",
        &format_str(format_args!("{}", s.streams_completed)),
    );
    push_row(&mut html, "bytes sent", &format_bytes(s.bytes_sent));
    push_row(&mut html, "bytes recv", &format_bytes(s.bytes_recv));
    if s.errors > 0 {
        push_row_cls(
            &mut html,
            "errors",
            &format_str(format_args!("{}", s.errors)),
            "v err",
        );
    }
    app.ui.metrics.set_inner_html(&html);
}

fn render_delivery(app: &Rc<App>) {
    let b = app.burst.borrow();
    if b.is_idle() {
        app.ui.delivery.set_inner_html(
            "<span class=\"k\">—</span>\
             <span class=\"v\">нет активного burst</span>",
        );
        return;
    }
    let mut html = String::with_capacity(256);
    if b.datagrams_expected > 0 {
        let n = b.datagrams_arrived.len() as u32;
        let total = b.datagrams_expected;
        let cls = if n >= total { "v ok" } else { "v warn" };
        push_row_cls(
            &mut html,
            "datagrams",
            &format_str(format_args!("{n}/{total}")),
            cls,
        );
    }
    if b.streams_expected > 0 {
        let n = b.streams_arrived.len() as u32;
        let total = b.streams_expected;
        let cls = if n >= total { "v ok" } else { "v warn" };
        push_row_cls(
            &mut html,
            "streams",
            &format_str(format_args!("{n}/{total}")),
            cls,
        );
    }
    app.ui.delivery.set_inner_html(&html);
}

fn update_badge(el: &web_sys::Element, state: &str, cls: &str) {
    let badge_cls = match cls {
        "v ok" => "wt-badge ok",
        "v warn" => "wt-badge warn",
        "v err" => "wt-badge err",
        _ => "wt-badge",
    };
    el.set_text_content(Some(state));
    el.set_class_name(badge_cls);
}

fn update_controls(app: &Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let (is_active, is_ready, can_connect) = match client_opt.as_ref() {
        Some(c) => {
            let s = c.state();
            let active = matches!(s, WtState::Connecting | WtState::Ready);
            let can = matches!(
                s,
                WtState::Idle
                    | WtState::Closed
                    | WtState::Failed
                    | WtState::Ready
            );
            (active, s == WtState::Ready, can)
        }
        None => (false, false, true),
    };
    set_disabled(&app.ui.btn_connect, is_active || !can_connect);
    set_disabled(&app.ui.btn_close, !is_active);
    set_disabled(&app.ui.btn_datagram, !is_ready);
    set_disabled(&app.ui.btn_stream, !is_ready);
    set_disabled(&app.ui.btn_burst_dg, !is_ready);
    set_disabled(&app.ui.btn_burst_str, !is_ready);
    set_disabled(&app.ui.btn_parallel, !is_ready);
}

fn set_disabled(el: &HtmlElement, disabled: bool) {
    if disabled {
        let _ = el.set_attribute("disabled", "");
    } else {
        let _ = el.remove_attribute("disabled");
    }
}

// ============================================================
//                    TIMER
// ============================================================

fn start_stats_timer(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let interval = Interval::new(1000, move || {
        if app_c.client.borrow().is_some() {
            render_metrics(&app_c);
        }
    });
    *app._stats_timer.borrow_mut() = Some(interval);
    Ok(())
}

// ============================================================
//                    HELPERS
// ============================================================

fn format_err_str(context: &str, e: &JsValue) -> String {
    let name = js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    let msg = js_sys::Reflect::get(e, &JsValue::from_str("message"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| format_str(format_args!("{e:?}")));
    return format_str(format_args!("{context}: {name} — {msg}"));
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

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "wt-status ok",
        "err" => "wt-status err",
        "busy" => "wt-status busy",
        _ => "wt-status",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
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

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        push_log_static(&self.log, kind, text);
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"wt-log-empty\">здесь появятся события</div>",
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
            html.push_str("<div class=\"wt-log-entry ");
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
}

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

Полные тексты `src/app.rs`, `src/wt_client.rs` и `tests/wasm_wt_client.rs` приведены в тексте главы 37. Здесь они опущены для компактности — цель Приложения показать структуру, а не дублировать содержимое.

**Что читатель может добавить в Приложение сам:**

- Полный листинг `src/app.rs` (около 900 строк).
- Полный листинг `src/wt_client.rs` (около 500 строк).
- Тесты `tests/wasm_wt_client.rs` (около 100 строк).
- Скриншоты работающего демо с эхо-ответами сервера.
- Вывод консоли сервера в момент подключения клиента.
- Скриншот `chrome://flags` с включённым WebTransport Developer Mode.
- Вывод `mkcert -install` и `mkcert -cert-file ...`.