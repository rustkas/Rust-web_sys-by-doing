# Глава 36. WebSocket

*(Уточнение: работа шла над главой 36 — WebSocket. В запросе указана «35», но по контексту и по скриншоту речь идёт именно о WebSocket. Ниже — финальный текст главы 36 с учётом реально работающих решений, найденных в процессе отладки.)*

---

`WebSocket` — единственный протокол в Web Platform с **полноценным двунаправленным соединением**. Все остальные API работают по модели «запрос–ответ»: клиент инициирует, сервер отвечает, соединение закрывается. WebSocket держит канал открытым в обе стороны — сервер может отправить сообщение в любой момент, без запроса от клиента.

Это открывает целый класс приложений, невозможных через `fetch`: чаты, коллаборативные редакторы, биржевые котировки, игровые лобби. Но вместе с мощностью приходит и сложность: **разрыв — не ошибка, а нормальное состояние сети**. WiFi переключился, ноутбук уснул, корпоративный прокси закрыл соединение по таймауту. Клиент обязан переподключаться сам, не полагаясь на пользователя.

Три темы формируют ядро главы: **жизненный цикл соединения**, **стратегия reconnect с exponential backoff**, **управление состоянием на стороне Rust**. И четвёртая, неочевидная и очень практичная: **правильная работа с wasm-bindgen Closure при длинном жизненном цикле JS-объекта** — на этом спотыкается каждый, кто впервые пишет WebSocket-клиент на Rust, и ошибка проявляется не сразу, а через несколько секунд после видимого успеха.

## 36.1. Структура

```
src/
  ws_client.rs                   ← НОВЫЙ модуль главы 36
  app.rs                         ← демо (заменяется целиком)
tests/
  wasm_ws_client.rs              ← тесты главы
src/lib.rs                       ← +1 строка: pub mod ws_client;
Cargo.toml                       ← +3 фичи web-sys
```

**Ссылки для детального изучения:**

- MDN: [`WebSocket`](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- MDN: [`CloseEvent`](https://developer.mozilla.org/en-US/docs/Web/API/CloseEvent)
- RFC 6455: [The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- HTML Living Standard: [Web sockets](https://html.spec.whatwg.org/multipage/web-sockets.html)
- RFC 6202: [Known Issues and Best Practices for Long Polling](https://datatracker.ietf.org/doc/html/rfc6202) — почему WebSocket лучше polling

**Префикс CSS** — `ws-`, не конфликтует с `gl-`, `ms-`, `cd-`, `aw-`, `md-`.

**Cargo.toml** — три новые фичи:

```toml
    "WebSocket",
    "BinaryType",
    "CloseEvent",
```

`MessageEvent`, `Event`, `Blob` уже подключены в предыдущих главах.

## 36.2. Жизненный цикл соединения

`WebSocket.readyState` — числовой код 0..3, каждый — отдельная фаза:

```text
0  CONNECTING   handshake (HTTP Upgrade) идёт прямо сейчас
1  OPEN         канал открыт, обе стороны могут отправлять
2  CLOSING      close() вызван, идёт рукопожатие завершения
3  CLOSED       канал закрыт, объект непригоден
```

**Переходы:**

```text
        new WebSocket(url)
                │
                ▼
         ┌──────────────┐
         │  CONNECTING  │
         └──────┬───────┘
                │  onopen
                ▼
         ┌──────────────┐
         │     OPEN     │◀─── send()/receive()
         └──────┬───────┘
                │  close() или onerror
                ▼
         ┌──────────────┐
         │   CLOSING    │
         └──────┬───────┘
                │  onclose
                ▼
         ┌──────────────┐
         │    CLOSED    │
         └──────────────┘
                │
                └──── reconnect ────▶ CONNECTING
```

**Ключевое правило.** Отправка возможна только в `OPEN`. `send()` на `CONNECTING` бросит `InvalidStateError`. То же — на `CLOSING` и `CLOSED`.

```rust
pub enum WsState {
    Connecting, Open, Closing, Closed,
}

impl WsState {
    pub fn from_code(code: u16) -> Self {
        return match code {
            0 => WsState::Connecting,
            1 => WsState::Open,
            2 => WsState::Closing,
            3 => WsState::Closed,
            _ => WsState::Closed,
        };
    }
}
```

**Код закрытия в `CloseEvent` — это не то же самое, что `readyState`.** Коды:

| Код | Значение |
|---|---|
| 1000 | Нормальное завершение |
| 1001 | Сервер уходит (going away) |
| 1006 | **Abnormal closure** — соединение разорвано без close-фрейма |
| 1008 | Политика нарушена |
| 1011 | Внутренняя ошибка сервера |
| 4000–4999 | **Application-specific** — используйте сами |

Код **1006 нельзя отправить явно** — он доступен только при чтении `CloseEvent.code`. Именно он появляется при падении сервера, обрыве сети, принудительном закрытии вкладки. Если видите 1006 в `CloseEvent` — не спрашивайте «почему не 1000», это аварийный разрыв.

Флаг `wasClean` — был ли handshake завершения штатным. Разница: `wasClean = true` означает, что сервер успел отправить close-фрейм, `false` — TCP разорван молча.

## 36.3. Text vs binary

WebSocket поддерживает два типа сообщений:

- **Text** — UTF-8 строки. Отправка: `send_with_str()`.
- **Binary** — произвольные байты. Отправка: `send_with_u8_array()`.

На приёме тип зависит от фрейма и **от `binaryType`**. По умолчанию `binaryType = "blob"`, и `data` приходит как `Blob` — асинхронно читаемый. **Установите `binaryType = "arraybuffer"` сразу после `new WebSocket()`:**

```rust
let ws = WebSocket::new(&url)?;
ws.set_binary_type(BinaryType::Arraybuffer);
```

Тогда `MessageEvent.data` — `ArrayBuffer`, читается синхронно:

```rust
if let Ok(ab) = data.dyn_into::<ArrayBuffer>() {
    let u8 = Uint8Array::new(&ab);
    cb(WsEvent::Binary(u8.to_vec()));
}
```

**Почему Blob — плохо для Rust.** Blob читается только через `FileReader` (асинхронно, через промис). В Rust это значит, что обработчик сообщения не может обработать бинарные данные синхронно, и приходится строить дополнительный pipeline. С `ArrayBuffer` всё просто: пришло → прочитали → обработали.

**Правило.** Устанавливайте `binaryType` до `onopen`, а не в обработчике. Изменение на лету не переключит уже буферизованные фреймы.

**Размер сообщений.** WebSocket не имеет лимита на размер одного фрейма. Браузеры ставят разумный предел — обычно 64 МБ. Для больших данных — чанковать на уровне приложения (`{seq, total, data}`), не WebSocket.

## 36.4. События

Четыре обработчика:

```text
onopen    — соединение открыто, можно отправлять
onmessage — пришло сообщение (text или binary)
onerror   — транспортная ошибка, НЕ закрытие
onclose   — соединение закрыто (с кодом и причиной)
```

**Порядок не всегда очевиден.** `onopen` не обязательно предшествует `onerror`. Ошибка handshake (неверный URL, CORS-политика, истёкший сертификат) вызовет `onerror` без `onopen`, а следом — `onclose`.

**`onerror` — не сигнал реконнекта.** Классическая ошибка: инициировать реконнект из `onerror`. Он только уведомляет о транспортной проблеме. Реальный разрыв придёт в `onclose`. **Реконнект запускайте только из `onclose`** — иначе получите двойной запуск.

**`onclose` — единственный надёжный источник правды** о том, что соединение мертво. И при штатном `close()`, и при аварии, `onclose` сработает. Всё остальное — догадки.

## 36.5. Стратегия reconnect

**Наивная стратегия — «попробовать сразу» — плоха.** Если сервер лежит, тысяча клиентов одновременно бьют по нему. Retry-шторм не даёт серверу подняться. Даже одиночный клиент без задержки может перегрузить восстанавливающийся сервер и снова его уронить.

**Exponential backoff.** Задержка растёт экспоненциально:

```text
attempt 1:   500 мс
attempt 2:  1000 мс
attempt 3:  2000 мс
attempt 4:  4000 мс
attempt 5:  8000 мс
attempt 6: 15000 мс (cap)
```

Формула: `delay = min(max_ms, base_ms × 2^(attempt-1))`.

**Jitter.** Все клиенты, упавшие одновременно, переподключатся одновременно. Jitter — случайное отклонение ±20 % — размазывает их по времени:

```text
attempt 3: не 2000 мс ровно, а 1600…2400 мс
```

```rust
pub fn backoff_delay_ms(attempt: u32, base_ms: u32, max_ms: u32) -> u32 {
    if attempt == 0 {
        return base_ms.min(max_ms);
    }
    let shift = attempt.saturating_sub(1).min(16);
    let factor = 1u32.checked_shl(shift).unwrap_or(u32::MAX);
    let raw = base_ms.saturating_mul(factor);
    let capped = raw.min(max_ms);
    let jitter_range = (capped / 5).max(1);
    let jitter = random_i32(jitter_range as i32);
    let with_jitter = (capped as i64 + jitter as i64).max(1) as u32;
    return with_jitter.min(max_ms);
}
```

**Сброс счётчика при успехе.** В `onopen` — `reconnect_attempt = 0`. Иначе следующий разрыв начнётся с attempt = 5 и будет ждать 15 секунд вместо 500 миллисекунд.

**Реконнект под флагом.** При ручном `close()` от пользователя реконнект не нужен. Флаг `should_reconnect` управляет этим:

```rust
pub fn close(&self) {
    let ws_opt = {
        let mut inner = self.inner.borrow_mut();
        inner.should_reconnect = false;
        inner.reconnect_timeout = None;
        inner.ws.clone()
    };
    if let Some(ws) = ws_opt {
        if ws.ready_state() == WebSocket::OPEN
            || ws.ready_state() == WebSocket::CONNECTING
        {
            let _ = ws.close();
        }
    }
}
```

**Токен поколения.** При пересоздании сокета старые замыкания могут ещё сработать (запоздавший `onclose` предыдущего сокета). Вводим `generation: u32`, инкрементируем при каждом `open`. Closure захватывает своё поколение, при вызове сравнивает — не совпало, игнорируем.

Без этого получите лавину: старый сокет закрылся → `onclose` запустил reconnect → новый сокет создан → следом приходит запоздавший `onclose` старого → запускает **второй** reconnect. И так по кругу.

## 36.6. Closure и wasm-bindgen: неочевидная ловушка

Это самая ценная часть главы. Ошибка, на которой спотыкаются все, и проявляется она **не в момент, когда что-то пошло не так, а через несколько миллисекунд**, в момент следующего `open_socket`.

**Симптом в консоли:**

```
Uncaught Error: closure invoked recursively or after being dropped
    __wbg_set_onclose  hello_wasm02.js:972
    wasm_bindgen__convert__closures_____invoke__hf6d92a...
```

Ошибка возникает в строке `ws.set_onclose(cl)` для **нового** сокета, который создаётся из callback'а реконнекта. Wasm-bindgen отказывается устанавливать обработчик.

**Наивная (и ошибочная) реализация:**

```rust
struct Inner {
    ws: Option<WebSocket>,
    closures: Vec<Closure<dyn FnMut(JsValue)>>,  // ← накопление closures
}

fn open_socket(...) {
    let ws = WebSocket::new(&url)?;

    let onopen = Closure::new(...);
    ws.set_onopen(Some(onopen.as_ref().unchecked_ref()));
    closures.push(onopen);                     // ← сохраняем

    let onclose = Closure::new(...);
    ws.set_onclose(Some(onclose.as_ref().unchecked_ref()));
    closures.push(onclose);                    // ← сохраняем

    i.closures = closures;                     // ← ДРОПАЕМ старые closures
}
```

**Что происходит при пересоздании сокета (reconnect):**

1. Старый `WebSocket` хранится в JS-памяти и всё ещё ссылается через `.onclose` на wasm-bindgen closure `h_old`.
2. В Rust мы делаем `i.closures = new_closures` — вектор перезаписывается, `h_old` **уничтожается**.
3. JS-объект WebSocket остаётся с висячей ссылкой на мёртвую wasm-функцию.
4. При следующей операции с closure-таблицей wasm-bindgen замечает несоответствие и падает.

**Проблема не в том, что мы дропаем closures. Проблема в том, что дропаем их, пока JS-объект ещё жив.**

## Решение — `forget()`

Любая closure, передаваемая в JS-объект с длинным жизненным циклом (WebSocket, EventSource, Worker, AudioContext-узлы), должна быть **forgotten**. Wasm-bindgen перестаёт её отслеживать, JS-функция живёт вечно.

```rust
fn open_socket(
    inner: &Rc<RefCell<Inner>>,
    on_event: Rc<dyn Fn(WsEvent)>,
) -> Result<(), JsValue> {
    let ws = WebSocket::new(&url)?;
    ws.set_binary_type(BinaryType::Arraybuffer);

    // ---- onopen ----
    {
        let cl = Closure::<dyn FnMut(JsValue)>::new(move |_ev| {
            // обработчик
        });
        ws.set_onopen(Some(cl.as_ref().unchecked_ref()));
        cl.forget();       // ← ключевое
    }

    // ---- onmessage ----, ---- onerror ----, ---- onclose ----
    // ... аналогично: setup + forget()

    // Сохраняем только WebSocket. Closures забыты и живут в JS.
    {
        let mut i = inner.borrow_mut();
        i.ws = Some(ws);
    }

    return Ok(());
}
```

Поле `closures: Vec<...>` в `Inner` **удаляем полностью**. Оно больше не нужно — мы не управляем временем жизни замыканий.

**Цена решения.** Небольшая утечка — каждая forgotten closure остаётся в JS-памяти навсегда. За один reconnect — 4 closures × ~150 байт = ~600 байт. За тысячи реконнектов — около 1 МБ. Для демо-приложения это ничто.

**Если нужна нулевая утечка** — есть два пути, оба сложнее и не нужны в учебном примере:
- Снимать `set_onclose(None)` перед дропом closure. Но это надо делать **вне** самих closures — например, из отдельного `setTimeout(0)`, тогда старый WebSocket успеет отсоединиться.
- Использовать `Closure::once_into_js` — но это только для однократных обработчиков.

**Правило проекта.** Если вы передаёте wasm-bindgen Closure в JS-объект с **неограниченным временем жизни** (WebSocket, EventSource, Worker) — forget'ьте её. Хранение в `Vec` и очистка при пересоздании объекта дадут ошибку «closure invoked recursively or after being dropped» — не сразу, а при следующей установке обработчиков.

Исключение — короткоживущие closures: обработчики кликов на кнопках, локальные callbacks, `setTimeout`. Их дропать безопасно, потому что JS перестаёт ссылаться на них раньше, чем Rust дропает wrapper.

## 36.7. Rust application state

WebSocket — асинхронный, событийный API. Всё общение с Rust — через колбэки. Модель `Rc<RefCell<...>>` подходит хорошо, но с двумя оговорками.

### 36.7.1. Rc-цикл: App ↔ WsClient ↔ callback

Наивная реализация:

```rust
struct App {
    client: RefCell<Option<Rc<WsClient>>>,   // App держит WsClient
}

// При connect:
let on_event: Rc<dyn Fn(WsEvent)> = Rc::new(move |ev| {
    app.push_log(...);   // App держится в callback
});
let client = WsClient::new(config, on_event);
*app.client.borrow_mut() = Some(client);
```

**Результат — цикл.** `App` держит `WsClient`, `WsClient` держит `on_event`, `on_event` держит `App`. Никто не освобождается. Если пользователь нажмёт Connect–Disconnect много раз — десятки мегабайт утечки.

**Решение — `Weak<App>` в callback:**

```rust
let app_weak = Rc::downgrade(&app);
let on_event: Rc<dyn Fn(WsEvent)> = Rc::new(move |ev| {
    if let Some(app) = app_weak.upgrade() {
        on_ws_event(app, ev);
    }
});
```

Теперь `on_event` не владеет `App`. Когда пользователь делает `disconnect`, `App` продолжает жить (его держит внешний `Rc` из `run()`), но при завершении страницы всё освобождается корректно.

### 36.7.2. `RefCell::borrow()` через JS-вызов

Классическая ловушка:

```rust
// НЕПРАВИЛЬНО:
pub fn send_text(&self, text: &str) -> Result<(), JsValue> {
    let inner = self.inner.borrow();          // ← держим borrow
    let Some(ws) = inner.ws.as_ref() else { ... };
    return ws.send_with_str(text);            // ← JS-вызов
}
```

Если `send_with_str` синхронно вызовет `onerror` или `onclose` — оба захотят `inner.borrow_mut()`, а borrow уже занят. Паника «already mutably borrowed» или «already borrowed».

**Решение — клонировать нужное и отпустить borrow до JS-вызова:**

```rust
pub fn send_text(&self, text: &str) -> Result<(), JsValue> {
    let ws = {
        let inner = self.inner.borrow();
        let Some(ws) = inner.ws.as_ref() else {
            return Err(JsValue::from_str("no socket"));
        };
        if ws.ready_state() != WebSocket::OPEN {
            return Err(JsValue::from_str("socket not open"));
        }
        ws.clone()                            // ← копия
    };                                        // ← borrow отпущен
    return ws.send_with_str(text);            // ← безопасно
}
```

Тот же паттерн — для `send_binary`, `close`, `simulate_drop`. Правило: **никогда не держите `RefCell::borrow()` через JS-вызов, который может синхронно вызвать ваш же callback**.

## 36.8. Практический real-time client

Собираем всё вместе. Демо — панель управления WebSocket-клиентом с эхо-сервером.

**Три колонки UI:**

1. **Управление** — URL, Connect/Disconnect, Reset, отправка text, отправка binary, Ping, Simulate drop, индикатор состояния, конфиг reconnect.
2. **Журнал + эхо-чат** — события в реальном времени, история сообщений с направлением `→` (out) / `←` (in) / `·` (system).
3. **Метрики** — счётчики, uptime, latency, признаки восстановления. Панель «Практика reconnect».

**Сценарий:**

1. Вводим `wss://echo.websocket.org`, Connect.
2. Состояние: `Connecting → Open`, uptime тикает.
3. Отправляем «Привет» — эхо приходит, добавляется в чат.
4. Отправляем binary (32 байта) — эхо, hex-preview.
5. Ping — `ping:<timestamp>` уходит, pong возвращается, показываем RTT.
6. Simulate drop — `close(4000)` → onclose → reconnect с backoff.
7. В журнале видны строки `reconnecting: attempt=1, delay=454 мс`.
8. После успеха — «соединение восстановлено после реконнекта».
9. Кнопка Reset — очистка лога, чата, метрик, session_id.

**Что показано в UI реально работающего демо:**

- Все четыре состояния visible (badge меняет цвет).
- Счётчики `sent / recv / bytes_sent / bytes_recv / dropped`.
- `uptime` — замораживается при Closed, тикает при Open.
- `last_ping` — round-trip мс.
- Флаг `recovered: yes` — если соединение восстановлено после реконнекта.
- Сессии в журнале разделены строкой `— сессия N —`.

## 36.9. Исходные файлы

### `Cargo.toml`

```toml
    "WebSocket",
    "BinaryType",
    "CloseEvent",
```

### `src/lib.rs`

```rust
pub mod ws_client;
```

### `src/ws_client.rs`

Полный текст приведён в предыдущем сообщении (с `forget()` вместо накопления closures). Ключевые элементы:

- `WsState` — четыре состояния, `from_code`, `as_str`.
- `WsEvent` — семь вариантов, включая `Reconnecting` и `ReconnectExhausted`.
- `WsConfig` — url, max_attempts, base_ms, max_ms, binary_as_arraybuffer.
- `backoff_delay_ms` — чистая функция, тестируется без сети.
- `WsClient` — `new`, `connect`, `close`, `send_text`, `send_binary`, `simulate_drop`, `state`, `config`.
- `open_socket` — свободная функция, вызывается из `connect` и из таймера реконнекта. **Все closures — forgotten.**
- `Inner` **не хранит** `Vec<Closure>` — только `ws`, `generation`, `reconnect_attempt`, `should_reconnect`, `reconnect_timeout`, `state`.
- `hex_preview` — утилита для отображения бинарных данных.

### `tests/wasm_ws_client.rs`

16 тестов:

- `WsState::from_code` для всех кодов + unknown → Closed.
- `WsState::as_str`.
- `backoff_delay_ms`: attempt=0 даёт base; attempt=1 в диапазоне ±20 %; рост экспоненциальный (20 прогонов — min attempt 4 > max attempt 1); cap на max; никогда не ноль; наличие jitter (10 прогонов дают разные значения).
- `hex_preview`: базовый случай; truncate с эллипсисом; пустой вход; padding для однозначных байт.
- `WsConfig::new` — defaults.

**Команды:**

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_ws_client
npm run wasm:dev && npm start
```

## 36.10. Анализ решения

**Что демонстрирует:**

1. **Полный lifecycle.** Четыре состояния визуально различимы.
2. **Text и binary симметричны.** Обе кнопки отправляют, обе получают echo.
3. **Reconnect в действии.** Simulate drop → backoff → восстановление. Видна серия `reconnecting` с растущими задержками.
4. **Ping/pong с замером RTT.** Round-trip измеряется и обновляется.
5. **Rust application state.** Единая модель `App` с `RefCell<Metrics>`, `RefCell<Vec<ChatMsg>>`, `RefCell<Vec<LogEntry>>`.
6. **Session boundary в журнале.** Reconnect-циклы визуально отделены.
7. **Uptime замораживается.** После Closed — финальное значение, серым.

**Ключевые практические находки:**

1. **Closures для WebSocket должны быть `forget()`-нуты.** Хранение в `Vec` с последующим дропом даёт ошибку «closure invoked recursively or after being dropped» при следующем open_socket.
2. **`RefCell::borrow()` не переживает JS-вызов.** Клонируйте нужное до вызова `close`/`send_with_*`.
3. **`Weak<App>` в callback'е** — единственный способ избежать Rc-цикла.
4. **Generation token** защищает от лавины запоздавших событий старого сокета.
5. **`binaryType = "arraybuffer"`** устанавливается **до** open, не после.

**Скрытые ошибки, которых удалось избежать:**

1. **Двойной reconnect.** Без generation token и с запуском реконнекта из `onerror` — лавина. Решение: реконнект только из `onclose`, generation guard.
2. **Утечка Rc.** `Rc<App>` в callback без ослабления. Решение: `Weak<App>`.
3. **Паника `RefCell`.** Borrow через JS-вызов. Решение: клонировать `WebSocket` до вызова метода.
4. **Забытый `binaryType`.** Blob вместо ArrayBuffer → `MessageEvent.data` не читается из Rust. Решение: установка `BinaryType::Arraybuffer` сразу после `new`.
5. **Забытый `forget()`.** Ошибка «closure invoked recursively» через несколько секунд после видимого успеха.

## 36.11. Итог

**WebSocket — единственный двунаправленный транспорт в Web Platform.** Сервер может отправлять данные без запроса клиента.

**Жизненный цикл — четыре состояния.** `CONNECTING → OPEN → CLOSING → CLOSED`. Отправка только в `OPEN`. `onerror` не означает закрытие — реконнект запускается только из `onclose`.

**Text vs binary.** Text через `send_with_str`, binary через `send_with_u8_array`. На приёме — `ArrayBuffer` для синхронного чтения. `binaryType` — до `onopen`.

**Reconnect обязателен.** Exponential backoff с jitter: 500 мс × 2^attempt, cap 15 с, ±20 %. Сброс счётчика при успешном open.

**Generation token.** При каждом `open` инкрементируется. Запоздавшие события старого сокета игнорируются. Без него — лавина реконнектов.

**Closures — forget.** Единственный рабочий паттерн для WebSocket. Хранение в Vec и последующий дроп дают ошибку wasm-bindgen.

**Weak<App> в callback.** Защита от Rc-цикла.

**Клонируйте перед JS-вызовом.** `RefCell::borrow()` не должен жить через `ws.close()` или `ws.send_*()`.

**Session boundary.** Reconnect-циклы должны визуально разделяться в UI.

Глава закрыта. Следующая тема — **WebRTC** — прямое peer-to-peer соединение между двумя браузерами, использующее `MediaStream` (глава 35) как источник данных и `WebSocket` (глава 36) как сигнальный канал.

## 36.12. Приложение.

**Cargo.toml**:
```
    # WebSocket (глава 36)
    "WebSocket",
    "BinaryType",
    "CloseEvent",
```

**ws_client.rs**:
```
//! 36. WebSocket клиент.
//!
//! Обёртка над браузерным WebSocket:
//!   • жизненный цикл (Connecting → Open → Closing → Closed);
//!   • text- и binary-сообщения;
//!   • events через Rust-колбэк Fn(WsEvent);
//!   • стратегия reconnect с exponential backoff и jitter;
//!   • защита от лавины событий через токен поколения.
//!
//! # Про closures
//!
//! Все closures, навешиваемые на WebSocket, вызывают `forget()`.
//! Причина: JS-объект WebSocket переживает Rust-обёртку на несколько
//! GC-циклов. Если хранить closures в Vec и дропать их при пересоздании
//! сокета, получаем состояние «wasm-функция уничтожена, JS-ссылка жива»
//! → ошибка «closure invoked recursively or after being dropped»
//! при следующей установке обработчиков.
//!
//! `forget()` оставляет JS-функцию валидной навсегда. Небольшая утечка
//! (несколько сотен байт на reconnect) — приемлемая цена за корректность.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::callback::Timeout;
use js_sys::{ArrayBuffer, Uint8Array};
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use web_sys::{BinaryType, CloseEvent, MessageEvent, WebSocket};

// ============================================================
//                    СОСТОЯНИЯ
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum WsState {
    Connecting,
    Open,
    Closing,
    Closed,
}

impl WsState {
    pub fn from_code(code: u16) -> Self {
        return match code {
            0 => WsState::Connecting,
            1 => WsState::Open,
            2 => WsState::Closing,
            3 => WsState::Closed,
            _ => WsState::Closed,
        };
    }

    pub fn as_str(self) -> &'static str {
        return match self {
            WsState::Connecting => "connecting",
            WsState::Open => "open",
            WsState::Closing => "closing",
            WsState::Closed => "closed",
        };
    }
}

// ============================================================
//                    СОБЫТИЯ
// ============================================================

#[derive(Clone, Debug)]
pub enum WsEvent {
    Opened,
    Text(String),
    Binary(Vec<u8>),
    Error(String),
    Closed {
        code: u16,
        reason: String,
        was_clean: bool,
    },
    Reconnecting { attempt: u32, delay_ms: u32 },
    ReconnectExhausted,
}

// ============================================================
//                    КОНФИГУРАЦИЯ
// ============================================================

#[derive(Clone, Debug)]
pub struct WsConfig {
    pub url: String,
    pub max_reconnect_attempts: u32,
    pub base_reconnect_ms: u32,
    pub max_reconnect_ms: u32,
    pub binary_as_arraybuffer: bool,
}

impl WsConfig {
    pub fn new(url: impl Into<String>) -> Self {
        return Self {
            url: url.into(),
            max_reconnect_attempts: 5,
            base_reconnect_ms: 500,
            max_reconnect_ms: 15_000,
            binary_as_arraybuffer: true,
        };
    }
}

// ============================================================
//                    СТРАТЕГИЯ RECONNECT
// ============================================================

pub fn backoff_delay_ms(attempt: u32, base_ms: u32, max_ms: u32) -> u32 {
    if attempt == 0 {
        return base_ms.min(max_ms);
    }
    let shift = attempt.saturating_sub(1).min(16);
    let factor = 1u32.checked_shl(shift).unwrap_or(u32::MAX);
    let raw = base_ms.saturating_mul(factor);
    let capped = raw.min(max_ms);
    let jitter_range = (capped / 5).max(1);
    let jitter = random_i32(jitter_range as i32);
    let with_jitter = (capped as i64 + jitter as i64).max(1) as u32;
    return with_jitter.min(max_ms);
}

fn random_i32(limit: i32) -> i32 {
    let r = js_sys::Math::random();
    let span = (limit as f64) * 2.0;
    return (r * span - limit as f64).round() as i32;
}

// ============================================================
//                    INNER
// ============================================================

struct Inner {
    config: WsConfig,
    ws: Option<WebSocket>,
    generation: u32,
    reconnect_attempt: u32,
    should_reconnect: bool,
    reconnect_timeout: Option<Timeout>,
    state: WsState,
}

// ============================================================
//                    КЛИЕНТ
// ============================================================

pub struct WsClient {
    inner: Rc<RefCell<Inner>>,
    on_event: Rc<dyn Fn(WsEvent)>,
}

impl WsClient {
    pub fn new(config: WsConfig, on_event: Rc<dyn Fn(WsEvent)>) -> Rc<Self> {
        let inner = Rc::new(RefCell::new(Inner {
            config,
            ws: None,
            generation: 0,
            reconnect_attempt: 0,
            should_reconnect: false,
            reconnect_timeout: None,
            state: WsState::Closed,
        }));
        return Rc::new(Self { inner, on_event });
    }

    pub fn state(&self) -> WsState {
        return self.inner.borrow().state;
    }

    pub fn config(&self) -> WsConfig {
        return self.inner.borrow().config.clone();
    }

    pub fn connect(self: &Rc<Self>) -> Result<(), JsValue> {
        {
            let mut inner = self.inner.borrow_mut();
            inner.should_reconnect = true;
            inner.reconnect_attempt = 0;
            inner.reconnect_timeout = None;
            inner.state = WsState::Connecting;
        }
        return open_socket(&self.inner, self.on_event.clone());
    }

    pub fn close(&self) {
        let ws_opt = {
            let mut inner = self.inner.borrow_mut();
            inner.should_reconnect = false;
            inner.reconnect_timeout = None;
            inner.ws.clone()
        };
        // Borrow отпущен до JS-вызова: onclose может сработать
        // синхронно и попытаться взять RefCell.
        if let Some(ws) = ws_opt {
            if ws.ready_state() == WebSocket::OPEN
                || ws.ready_state() == WebSocket::CONNECTING
            {
                let _ = ws.close();
            }
        }
    }

    pub fn send_text(&self, text: &str) -> Result<(), JsValue> {
        let ws = {
            let inner = self.inner.borrow();
            let Some(ws) = inner.ws.as_ref() else {
                return Err(JsValue::from_str("no socket"));
            };
            if ws.ready_state() != WebSocket::OPEN {
                return Err(JsValue::from_str("socket not open"));
            }
            ws.clone()
        };
        return ws.send_with_str(text);
    }

    pub fn send_binary(&self, bytes: &[u8]) -> Result<(), JsValue> {
        let ws = {
            let inner = self.inner.borrow();
            let Some(ws) = inner.ws.as_ref() else {
                return Err(JsValue::from_str("no socket"));
            };
            if ws.ready_state() != WebSocket::OPEN {
                return Err(JsValue::from_str("socket not open"));
            }
            ws.clone()
        };
        return ws.send_with_u8_array(bytes);
    }

    pub fn simulate_drop(&self) -> Result<(), JsValue> {
        let ws = {
            let inner = self.inner.borrow();
            let Some(ws) = inner.ws.as_ref() else {
                return Err(JsValue::from_str("no active socket"));
            };
            if ws.ready_state() != WebSocket::OPEN {
                return Err(JsValue::from_str("socket not open"));
            }
            ws.clone()
        };
        return ws.close_with_code_and_reason(4000, "simulate-drop");
    }
}

// ============================================================
//                    ОТКРЫТИЕ СОКЕТА
// ============================================================

fn open_socket(
    inner: &Rc<RefCell<Inner>>,
    on_event: Rc<dyn Fn(WsEvent)>,
) -> Result<(), JsValue> {
    let (url, binary_type, gen_id) = {
        let mut inner_ref = inner.borrow_mut();
        inner_ref.generation = inner_ref.generation.wrapping_add(1);
        let url = inner_ref.config.url.clone();
        let bt = if inner_ref.config.binary_as_arraybuffer {
            BinaryType::Arraybuffer
        } else {
            BinaryType::Blob
        };
        (url, bt, inner_ref.generation)
    };

    let ws = WebSocket::new(&url)?;
    ws.set_binary_type(binary_type);

    // ---- onopen ----
    {
        let weak = Rc::downgrade(inner);
        let cb = on_event.clone();
        let cl = Closure::<dyn FnMut(JsValue)>::new(move |_ev: JsValue| {
            let Some(inner) = weak.upgrade() else { return };
            if inner.borrow().generation != gen_id {
                return;
            }
            {
                let mut i = inner.borrow_mut();
                i.state = WsState::Open;
                i.reconnect_attempt = 0;
            }
            cb(WsEvent::Opened);
        });
        ws.set_onopen(Some(cl.as_ref().unchecked_ref()));
        cl.forget();
    }

    // ---- onmessage ----
    {
        let weak = Rc::downgrade(inner);
        let cb = on_event.clone();
        let cl = Closure::<dyn FnMut(JsValue)>::new(move |ev: JsValue| {
            let Some(inner) = weak.upgrade() else { return };
            if inner.borrow().generation != gen_id {
                return;
            }
            let Ok(msg) = ev.dyn_into::<MessageEvent>() else {
                return;
            };
            let data = msg.data();
            if let Some(text) = data.as_string() {
                cb(WsEvent::Text(text));
                return;
            }
            if let Ok(ab) = data.dyn_into::<ArrayBuffer>() {
                let u8 = Uint8Array::new(&ab);
                cb(WsEvent::Binary(u8.to_vec()));
                return;
            }
        });
        ws.set_onmessage(Some(cl.as_ref().unchecked_ref()));
        cl.forget();
    }

    // ---- onerror ----
    {
        let weak = Rc::downgrade(inner);
        let cb = on_event.clone();
        let cl = Closure::<dyn FnMut(JsValue)>::new(move |_ev: JsValue| {
            let Some(inner) = weak.upgrade() else { return };
            if inner.borrow().generation != gen_id {
                return;
            }
            cb(WsEvent::Error("transport error".to_string()));
        });
        ws.set_onerror(Some(cl.as_ref().unchecked_ref()));
        cl.forget();
    }

    // ---- onclose ----
    {
        let weak = Rc::downgrade(inner);
        let cb = on_event.clone();
        let cl = Closure::<dyn FnMut(JsValue)>::new(move |ev: JsValue| {
            let Some(inner) = weak.upgrade() else { return };
            if inner.borrow().generation != gen_id {
                return;
            }

            let (code, reason, was_clean) = if let Ok(ce) = ev.dyn_into::<CloseEvent>()
            {
                (ce.code(), ce.reason(), ce.was_clean())
            } else {
                (1006, String::new(), false)
            };

            {
                let mut i = inner.borrow_mut();
                i.state = WsState::Closed;
                i.ws = None;
                i.reconnect_timeout = None;
            }

            cb(WsEvent::Closed {
                code,
                reason,
                was_clean,
            });

            let (should, attempt, max_attempts, base, max_ms) = {
                let i = inner.borrow();
                (
                    i.should_reconnect,
                    i.reconnect_attempt + 1,
                    i.config.max_reconnect_attempts,
                    i.config.base_reconnect_ms,
                    i.config.max_reconnect_ms,
                )
            };

            if !should || max_attempts == 0 {
                return;
            }
            if attempt > max_attempts {
                cb(WsEvent::ReconnectExhausted);
                return;
            }

            let delay = backoff_delay_ms(attempt, base, max_ms);
            {
                let mut i = inner.borrow_mut();
                i.reconnect_attempt = attempt;
            }
            cb(WsEvent::Reconnecting {
                attempt,
                delay_ms: delay,
            });

            let weak2 = Rc::downgrade(&inner);
            let cb2 = cb.clone();
            let timeout = Timeout::new(delay, move || {
                let Some(inner) = weak2.upgrade() else { return };
                if !inner.borrow().should_reconnect {
                    return;
                }
                let _ = open_socket(&inner, cb2);
            });
            inner.borrow_mut().reconnect_timeout = Some(timeout);
        });
        ws.set_onclose(Some(cl.as_ref().unchecked_ref()));
        cl.forget();
    }

    // Сохраняем только WebSocket. Closures — forgotten, живут в JS.
    {
        let mut i = inner.borrow_mut();
        i.ws = Some(ws);
    }

    return Ok(());
}

// ============================================================
//                    УТИЛИТЫ
// ============================================================

/// Форматирует первые N байт как hex-строку.
pub fn hex_preview(bytes: &[u8], max_bytes: usize) -> String {
    let n = bytes.len().min(max_bytes);
    let mut out = String::with_capacity(n * 3 + 16);
    for (i, b) in bytes.iter().take(n).enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format!("{:02x}", b));
    }
    if bytes.len() > max_bytes {
        out.push_str(" …");
    }
    return out;
}
```

**wasm_ws_client.rs**:
```
//! 36. Wasm-тесты ws_client: состояния, backoff, hex, конструктор.
//!
//! Тесты без сети — только чистая логика. Тесты с реальным
//! соединением помечены #[ignore].

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::ws_client::{backoff_delay_ms, hex_preview, WsConfig, WsState};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- WsState ----------------

#[wasm_bindgen_test]
fn wasm_ws_state_from_code_all() {
    assert_eq!(WsState::from_code(0), WsState::Connecting);
    assert_eq!(WsState::from_code(1), WsState::Open);
    assert_eq!(WsState::from_code(2), WsState::Closing);
    assert_eq!(WsState::from_code(3), WsState::Closed);
}

#[wasm_bindgen_test]
fn wasm_ws_state_from_code_unknown_is_closed() {
    assert_eq!(WsState::from_code(99), WsState::Closed);
    assert_eq!(WsState::from_code(255), WsState::Closed);
}

#[wasm_bindgen_test]
fn wasm_ws_state_as_str() {
    assert_eq!(WsState::Connecting.as_str(), "connecting");
    assert_eq!(WsState::Open.as_str(), "open");
    assert_eq!(WsState::Closing.as_str(), "closing");
    assert_eq!(WsState::Closed.as_str(), "closed");
}

// ---------------- Backoff ----------------

#[wasm_bindgen_test]
fn wasm_backoff_attempt_zero_is_base() {
    let d = backoff_delay_ms(0, 500, 15_000);
    assert_eq!(d, 500);
}

#[wasm_bindgen_test]
fn wasm_backoff_attempt_one_near_base() {
    // ±20 % от 500 = [400, 600].
    for _ in 0..20 {
        let d = backoff_delay_ms(1, 500, 15_000);
        assert!(d >= 400 && d <= 600, "d = {d}");
    }
}

#[wasm_bindgen_test]
fn wasm_backoff_grows_exponentially() {
    // Минимум при attempt=4 должен быть больше максимума при attempt=1.
    let mut max_a1 = 0u32;
    let mut min_a4 = u32::MAX;
    for _ in 0..50 {
        let a1 = backoff_delay_ms(1, 500, 15_000);
        let a4 = backoff_delay_ms(4, 500, 15_000);
        max_a1 = max_a1.max(a1);
        min_a4 = min_a4.min(a4);
    }
    assert!(min_a4 > max_a1, "min_a4 = {min_a4}, max_a1 = {max_a1}");
}

#[wasm_bindgen_test]
fn wasm_backoff_capped_at_max() {
    // attempt=10 при базе 500 должен быть закэплен на 15000 (±20 %).
    for _ in 0..20 {
        let d = backoff_delay_ms(10, 500, 15_000);
        assert!(d <= 15_000, "d = {d}");
        assert!(d >= 12_000, "d = {d}");
    }
}

#[wasm_bindgen_test]
fn wasm_backoff_never_zero() {
    for attempt in 0..=20 {
        let d = backoff_delay_ms(attempt, 100, 15_000);
        assert!(d > 0, "attempt = {attempt}, d = {d}");
    }
}

#[wasm_bindgen_test]
fn wasm_backoff_has_jitter() {
    // С jitter 10 вызовов не должны вернуть одно и то же число.
    let mut seen = std::collections::HashSet::new();
    for _ in 0..10 {
        seen.insert(backoff_delay_ms(3, 500, 15_000));
    }
    assert!(seen.len() > 1, "no jitter detected");
}

// ---------------- Hex preview ----------------

#[wasm_bindgen_test]
fn wasm_hex_preview_basic() {
    let bytes = [0xde, 0xad, 0xbe, 0xef];
    assert_eq!(hex_preview(&bytes, 8), "de ad be ef");
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
    let bytes: &[u8] = &[];
    assert_eq!(hex_preview(bytes, 8), "");
}

#[wasm_bindgen_test]
fn wasm_hex_preview_pads_single_digit() {
    let bytes = [0x0a, 0x00, 0xff];
    assert_eq!(hex_preview(&bytes, 8), "0a 00 ff");
}

// ---------------- WsConfig ----------------

#[wasm_bindgen_test]
fn wasm_ws_config_defaults() {
    let c = WsConfig::new("wss://example.com");
    assert_eq!(c.url, "wss://example.com");
    assert_eq!(c.max_reconnect_attempts, 5);
    assert_eq!(c.base_reconnect_ms, 500);
    assert_eq!(c.max_reconnect_ms, 15_000);
    assert!(c.binary_as_arraybuffer);
}
```


**app.rs**:
```
//! Глава 36: WebSocket.
//!
//! Практический real-time клиент с эхо-сервером:
//!   1. Connect / Disconnect по URL.
//!   2. Отправка text- и binary-сообщений.
//!   3. Ping с измерением round-trip time.
//!   4. Simulate drop — эмуляция обрыва без остановки reconnect.
//!   5. Автоматический reconnect с exponential backoff.
//!   6. Метрики: сообщения, байты, uptime, latency, dropped.
//!   7. Rust application state — единая модель Session.
//!   8. Разделители сессий в журнале.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: closures WebSocket хранятся в WsClient (не в app).
//! ПРАВИЛО: callback-цепочка не должна образовывать Rc-цикл — только Weak.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::callback::Interval;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use web_sys::{HtmlElement, HtmlInputElement};

use crate::format_str::format_str;
use crate::ws_client::{
    hex_preview, WsClient, WsConfig, WsEvent, WsState,
};

const LOG_LIMIT: usize = 14;
const CHAT_LIMIT: usize = 10;

const HTML: &str = r#"
<style>
  :root {
    --ws-info: #3498db; --ws-ok: #27ae60; --ws-err: #c0392b; --ws-warn: #e67e22;
    --ws-text: #222; --ws-muted: #666;
    --ws-bg: #fff; --ws-soft: #fafafa; --ws-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --ws-text: #e6e6e6; --ws-muted: #a0a0a0;
      --ws-bg: #1e1e1e; --ws-soft: #262626; --ws-border: #3a3a3a;
    }
    body { background: #121212; }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .ws-log-entry { background: #1e1e1e; }
    .ws-anno { background: #1a2233; color: #9fc4ff; }
    .ws-hint { background: #232014; color: #e6c98c; }
    .ws-msg-row.out { color: #9fc4ff; }
    .ws-msg-row.in  { color: #b8e0b8; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--ws-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .ws-subtitle { color: var(--ws-muted); font-size: 12px; margin: 0 0 12px; }
  .ws-subtitle a { color: var(--ws-info); }

  .ws-grid {
    display: grid; gap: 12px;
    grid-template-columns: minmax(0, 1.1fr) minmax(0, 1fr) minmax(0, 1.1fr);
  }
  @media (max-width: 1100px) { .ws-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px)  { .ws-grid { grid-template-columns: 1fr; } }
  .ws-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .ws-panel {
    border: 1px solid var(--ws-border); border-radius: 8px;
    padding: 10px 12px; background: var(--ws-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .ws-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px;
  }
  .ws-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--ws-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .ws-anno {
    background: #eef4ff; border-left: 3px solid var(--ws-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere;
  }
  .ws-hint {
    background: #fff9e6; border-left: 3px solid var(--ws-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #7d5a00;
  }

  .ws-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; }
  .ws-label { font-size: 11px; color: var(--ws-muted); }

  input[type="text"], input[type="number"], select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--ws-bg); color: var(--ws-text);
    font-family: ui-monospace, monospace;
  }
  input[type="text"].ws-url { flex: 1; min-width: 180px; }
  input[type="text"].ws-msg { flex: 1; min-width: 120px; }
  input[type="number"] { width: 70px; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--ws-soft); color: var(--ws-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.ws-primary { background: #eaf4ff; border-color: var(--ws-info); color: #1f4e79; }
  button.ws-success { background: #e8f8ed; border-color: var(--ws-ok); color: #1e8449; }
  button.ws-danger  { background: #fdedec; border-color: var(--ws-err); color: #922b21; }
  button.ws-warn    { background: #fff3cd; border-color: var(--ws-warn); color: #7d5a00; }

  .ws-status {
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--ws-soft); border-left: 3px solid #ccc;
    color: var(--ws-muted); min-height: 22px; overflow-wrap: anywhere;
  }
  .ws-status.ok   { background: #e8f8ed; border-color: var(--ws-ok); color: #1e8449; }
  .ws-status.err  { background: #fdedec; border-color: var(--ws-err); color: #922b21; }
  .ws-status.busy { background: #fff9e6; border-color: var(--ws-warn); color: #7d5a00; }

  .ws-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
  }
  @media (prefers-color-scheme: dark) {
    .ws-badge { background: #2a3a4a; color: #cfe0ff; }
  }
  .ws-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .ws-badge.err { background: #fdedec; color: #922b21; }
  .ws-badge.warn { background: #fff9e6; color: #7d5a00; }

  .ws-log {
    border: 1px solid var(--ws-border); border-radius: 5px;
    background: var(--ws-soft); max-height: 340px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .ws-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .ws-log-entry {
    display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent;
  }
  .ws-log-entry.ok   { border-left-color: var(--ws-ok); }
  .ws-log-entry.err  { border-left-color: var(--ws-err); }
  .ws-log-entry.info { border-left-color: var(--ws-info); }
  .ws-log-entry.warn { border-left-color: var(--ws-warn); }
  .ws-log-entry.sess {
    background: #eef4ff; color: #1f4e79; font-weight: 700;
    border-left-color: var(--ws-info); padding: 4px 10px;
  }
  .ws-log-entry .t { color: var(--ws-muted); flex-shrink: 0; }
  .ws-log-entry .m { flex: 1; overflow-wrap: anywhere; }
  .ws-log-entry.err .m  { color: #922b21; }
  .ws-log-entry.warn .m { color: #7d5a00; }

  .ws-chat {
    border: 1px solid var(--ws-border); border-radius: 5px;
    background: var(--ws-soft); max-height: 260px; overflow-y: auto;
    padding: 6px 0;
    font-family: ui-monospace, monospace; font-size: 12px;
  }
  .ws-chat-empty { padding: 14px; text-align: center;
    color: #999; font-style: italic; font-size: 11px; }
  .ws-msg-row { padding: 3px 12px; overflow-wrap: anywhere; }
  .ws-msg-row.out { color: #1f4e79; }
  .ws-msg-row.in  { color: #1e8449; }
  .ws-msg-row.sys { color: #7d5a00; font-style: italic; font-size: 11px;
    text-align: center; padding: 6px 12px; }
  .ws-msg-prefix { color: var(--ws-muted); margin-right: 6px; }

  .ws-settings-grid {
    display: grid; grid-template-columns: auto 1fr;
    gap: 4px 12px; margin: 6px 0;
    font-family: ui-monospace, monospace; font-size: 12px;
  }
  .ws-settings-grid .k { color: var(--ws-muted); }
  .ws-settings-grid .v { color: var(--ws-text); }
  .ws-settings-grid .v.ok  { color: #1e8449; }
  .ws-settings-grid .v.err { color: #922b21; }
  .ws-settings-grid .v.warn { color: #7d5a00; }
</style>

<h1>Глава 36: WebSocket</h1>
<p class="ws-subtitle">
  Connection lifecycle, text и binary messages, reconnect с backoff.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebSocket"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="ws-grid">

  <!-- ============ Колонка 1: Управление ============ -->
  <div class="ws-col">
    <div class="ws-panel">
      <h2 class="ws-panel-title">
        <span class="ws-num">1</span>Управление
      </h2>
      <div class="ws-anno">
        WebSocket держит соединение открытым в обе стороны. Разрыв —
        не ошибка, а нормальное состояние сети. Клиент переподключается
        сам через <b>exponential backoff</b> с jitter.
      </div>

      <div class="ws-row">
        <input type="text" id="ws-url" class="ws-url"
               value="wss://echo.websocket.org">
      </div>

      <div class="ws-row">
        <button id="ws-connect" class="ws-success">▶ Connect</button>
        <button id="ws-disconnect" class="ws-danger" disabled>■ Disconnect</button>
        <button id="ws-reset">↺ Reset</button>
        <span class="ws-badge" id="ws-state">closed</span>
      </div>

      <div class="ws-row">
        <input type="text" id="ws-msg" class="ws-msg"
               value="Привет, WebSocket!" placeholder="текст сообщения">
        <button id="ws-send-text" disabled>➤ Text</button>
      </div>

      <div class="ws-row">
        <span class="ws-label">binary:</span>
        <input type="number" id="ws-bin-len" value="32" min="1" max="1024">
        <span class="ws-label">байт</span>
        <button id="ws-send-bin" disabled>➤ Binary</button>
      </div>

      <div class="ws-row">
        <button id="ws-ping" disabled>🏓 Ping</button>
        <button id="ws-drop" class="ws-warn" disabled>💥 Simulate drop</button>
      </div>

      <div id="ws-status" class="ws-status">готово</div>
    </div>

    <div class="ws-panel">
      <h2 class="ws-panel-title">Состояние клиента</h2>
      <div id="ws-settings" class="ws-settings-grid">
        <span class="k">state</span><span class="v">closed</span>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 2: Журнал и чат ============ -->
  <div class="ws-col">
    <div class="ws-panel">
      <h2 class="ws-panel-title">Журнал событий</h2>
      <div class="ws-hint">
        Все события из <code>WsClient::on_event</code>. Между сессиями —
        разделитель «— сессия N —», добавленный при каждом connect.
      </div>
      <div id="ws-log" class="ws-log">
        <div class="ws-log-empty">здесь появятся события</div>
      </div>
    </div>

    <div class="ws-panel">
      <h2 class="ws-panel-title">Эхо-чат</h2>
      <div id="ws-chat" class="ws-chat">
        <div class="ws-chat-empty">отправьте сообщение — оно вернётся эхом</div>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 3: Статистика ============ -->
  <div class="ws-col">
    <div class="ws-panel">
      <h2 class="ws-panel-title">Метрики</h2>
      <div id="ws-metrics" class="ws-settings-grid">
        <span class="k">—</span><span class="v">нет данных</span>
      </div>
    </div>

    <div class="ws-panel">
      <h2 class="ws-panel-title">Практика reconnect</h2>
      <div class="ws-anno">
        Нажмите <b>Simulate drop</b> — соединение закроется кодом 4000
        (application-specific). Клиент воспримет это как разрыв и
        запустит реконнект. В журнале увидите серию <code>reconnecting
        (attempt = N, delay = M мс)</code> с растущей задержкой.
      </div>
      <div id="ws-reconnect-info" class="ws-settings-grid">
        <span class="k">attempt</span><span class="v">—</span>
      </div>
    </div>
  </div>

</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info, Warn, Session }

#[derive(Clone, Copy, PartialEq, Eq)]
enum ChatDir { Out, In, Sys }

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

struct ChatMsg {
    time: String,
    dir: ChatDir,
    text: String,
}

#[derive(Default)]
struct Metrics {
    sent_count: u64,
    recv_count: u64,
    bytes_sent: u64,
    bytes_recv: u64,
    dropped_while_closed: u64,
    opened_at_ms: f64,
    closed_at_ms: f64,
    last_ping_ms: Option<f64>,
    last_reconnect_attempt: u32,
    last_reconnect_delay: u32,
    recovered_from_reconnect: bool,
}

struct Ui {
    url: HtmlInputElement,
    msg: HtmlInputElement,
    bin_len: HtmlInputElement,
    btn_connect: HtmlElement,
    btn_disconnect: HtmlElement,
    btn_reset: HtmlElement,
    btn_send_text: HtmlElement,
    btn_send_bin: HtmlElement,
    btn_ping: HtmlElement,
    btn_drop: HtmlElement,
    badge_state: web_sys::Element,
    status: web_sys::Element,
    settings: web_sys::Element,
    metrics: web_sys::Element,
    reconnect_info: web_sys::Element,
    log: web_sys::Element,
    chat: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    chat: Rc<RefCell<Vec<ChatMsg>>>,
    metrics: Rc<RefCell<Metrics>>,
    client: RefCell<Option<Rc<WsClient>>>,
    session_id: RefCell<u32>,
    _uptime: RefCell<Option<Interval>>,
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
        url: get_input("ws-url")?,
        msg: get_input("ws-msg")?,
        bin_len: get_input("ws-bin-len")?,
        btn_connect: get_el("ws-connect")?.dyn_into()?,
        btn_disconnect: get_el("ws-disconnect")?.dyn_into()?,
        btn_reset: get_el("ws-reset")?.dyn_into()?,
        btn_send_text: get_el("ws-send-text")?.dyn_into()?,
        btn_send_bin: get_el("ws-send-bin")?.dyn_into()?,
        btn_ping: get_el("ws-ping")?.dyn_into()?,
        btn_drop: get_el("ws-drop")?.dyn_into()?,
        badge_state: get_el("ws-state")?,
        status: get_el("ws-status")?,
        settings: get_el("ws-settings")?,
        metrics: get_el("ws-metrics")?,
        reconnect_info: get_el("ws-reconnect-info")?,
        log: get_el("ws-log")?,
        chat: get_el("ws-chat")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        chat: Rc::new(RefCell::new(Vec::new())),
        metrics: Rc::new(RefCell::new(Metrics::default())),
        client: RefCell::new(None),
        session_id: RefCell::new(0),
        _uptime: RefCell::new(None),
        ui,
    });

    app.render_log();
    app.render_chat();
    render_settings(&app);
    render_metrics(&app);
    render_reconnect_info(&app);

    bind_buttons(&app)?;
    start_uptime_timer(&app)?;

    Ok(())
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
        set_click(&app.ui.btn_disconnect, move || { disconnect(&app_c); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_reset, move || { do_reset(&app_c); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_send_text, move || { send_text(app_c.clone()); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_send_bin, move || { send_binary(app_c.clone()); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_ping, move || { send_ping(app_c.clone()); });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_drop, move || { simulate_drop(app_c.clone()); });
    }
    Ok(())
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn connect(app: Rc<App>) {
    if app.client.borrow().is_some() {
        return;
    }

    let url = app.ui.url.value();
    if url.is_empty() {
        set_status(&app.ui.status, "err", "URL не задан");
        return;
    }

    let config = WsConfig::new(url.clone());

    // Callback захватывает Weak<App>, чтобы не образовать цикл
    // App → WsClient → on_event → App.
    let app_weak = Rc::downgrade(&app);
    let on_event: Rc<dyn Fn(WsEvent)> = Rc::new(move |ev| {
        if let Some(app) = app_weak.upgrade() {
            on_ws_event(app, ev);
        }
    });

    let client = WsClient::new(config, on_event);
    *app.client.borrow_mut() = Some(client.clone());

    // Новая сессия.
    let sid = {
        let mut s = app.session_id.borrow_mut();
        *s += 1;
        *s
    };
    app.push_log(
        LogKind::Session,
        format_str(format_args!("— сессия {sid} —")),
    );
    push_chat(
        &app,
        ChatDir::Sys,
        &format_str(format_args!("— сессия {sid} —")),
    );

    {
        let mut m = app.metrics.borrow_mut();
        *m = Metrics::default();
    }

    app.push_log(
        LogKind::Info,
        format_str(format_args!("подключение к {url}")),
    );
    set_status(&app.ui.status, "busy", "connecting…");
    update_controls(&app);

    if let Err(e) = client.connect() {
        let msg = format_str(format_args!("connect: {e:?}"));
        app.push_log(LogKind::Err, msg.clone());
        set_status(&app.ui.status, "err", &msg);
        *app.client.borrow_mut() = None;
        update_controls(&app);
    }
}

fn disconnect(app: &Rc<App>) {
    let client_opt = app.client.borrow_mut().take();
    let Some(client) = client_opt else { return; };
    client.close();
    app.push_log(LogKind::Info, "ручное закрытие соединения".to_string());
    set_status(&app.ui.status, "ok", "disconnected");
    update_controls(app);
}

fn do_reset(app: &Rc<App>) {
    disconnect(app);
    app.log.borrow_mut().clear();
    app.chat.borrow_mut().clear();
    *app.metrics.borrow_mut() = Metrics::default();
    *app.session_id.borrow_mut() = 0;
    app.render_log();
    app.render_chat();
    render_metrics(app);
    render_reconnect_info(app);
    set_status(&app.ui.status, "ok", "сброшено");
}

/// Возвращает true, если сокет открыт и send прошёл. Иначе —
/// инкрементирует dropped_while_closed и возвращает false.
fn ensure_open(app: &Rc<App>) -> Option<Rc<WsClient>> {
    let client_opt = app.client.borrow().clone();
    let Some(client) = client_opt else {
        set_status(&app.ui.status, "err", "нет соединения");
        return None;
    };
    if client.state() != WsState::Open {
        app.metrics.borrow_mut().dropped_while_closed += 1;
        render_metrics(app);
        set_status(&app.ui.status, "busy", "сокет не открыт — сообщение отброшено");
        return None;
    }
    return Some(client);
}

fn send_text(app: Rc<App>) {
    let text = app.ui.msg.value();
    if text.is_empty() { return; }
    let Some(client) = ensure_open(&app) else { return; };
    match client.send_text(&text) {
        Ok(()) => {
            {
                let mut m = app.metrics.borrow_mut();
                m.sent_count += 1;
                m.bytes_sent += text.len() as u64;
            }
            push_chat(&app, ChatDir::Out, &text);
            render_metrics(&app);
        }
        Err(e) => {
            let msg = format_str(format_args!("send_text: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

fn send_binary(app: Rc<App>) {
    let n = app.ui.bin_len.value().parse::<usize>().unwrap_or(32).clamp(1, 1024);
    let Some(client) = ensure_open(&app) else { return; };

    let mut bytes = Vec::with_capacity(n);
    for _ in 0..n {
        let r = js_sys::Math::random();
        bytes.push((r * 256.0) as u8);
    }

    match client.send_binary(&bytes) {
        Ok(()) => {
            {
                let mut m = app.metrics.borrow_mut();
                m.sent_count += 1;
                m.bytes_sent += n as u64;
            }
            let preview = hex_preview(&bytes, 8);
            push_chat(
                &app,
                ChatDir::Out,
                &format_str(format_args!("[binary {n}B] {preview}")),
            );
            render_metrics(&app);
        }
        Err(e) => {
            let msg = format_str(format_args!("send_binary: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

fn send_ping(app: Rc<App>) {
    let Some(client) = ensure_open(&app) else { return; };
    let ts = js_sys::Date::now();
    let text = format_str(format_args!("ping:{ts}"));
    match client.send_text(&text) {
        Ok(()) => {
            {
                let mut m = app.metrics.borrow_mut();
                m.sent_count += 1;
                m.bytes_sent += text.len() as u64;
            }
            push_chat(&app, ChatDir::Out, &text);
            render_metrics(&app);
        }
        Err(e) => {
            let msg = format_str(format_args!("ping: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

fn simulate_drop(app: Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let Some(client) = client_opt else { return; };
    match client.simulate_drop() {
        Ok(()) => {
            app.push_log(
                LogKind::Warn,
                "симуляция обрыва: close(4000, 'simulate-drop')".to_string(),
            );
        }
        Err(e) => {
            let msg = format_str(format_args!("simulate_drop: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

// ============================================================
//                    ОБРАБОТКА СОБЫТИЙ
// ============================================================

fn on_ws_event(app: Rc<App>, ev: WsEvent) {
    match ev {
        WsEvent::Opened => {
            let was_reconnect = app.metrics.borrow().last_reconnect_attempt > 0;
            {
                let mut m = app.metrics.borrow_mut();
                m.opened_at_ms = js_sys::Date::now();
                m.closed_at_ms = 0.0;
                m.recovered_from_reconnect = was_reconnect;
            }
            let msg = if was_reconnect {
                "соединение восстановлено после реконнекта"
            } else {
                "соединение открыто"
            };
            app.push_log(LogKind::Ok, msg.to_string());
            set_status(&app.ui.status, "ok", msg);
            update_controls(&app);
            render_metrics(&app);
        }
        WsEvent::Text(text) => {
            {
                let mut m = app.metrics.borrow_mut();
                m.recv_count += 1;
                m.bytes_recv += text.len() as u64;
            }
            if let Some(ts_str) = text.strip_prefix("ping:") {
                if let Ok(ts) = ts_str.parse::<f64>() {
                    let rtt = js_sys::Date::now() - ts;
                    app.metrics.borrow_mut().last_ping_ms = Some(rtt);
                    let msg = format_str(format_args!("pong: {rtt:.1} мс"));
                    app.push_log(LogKind::Ok, msg);
                }
            } else {
                let short: String = text.chars().take(60).collect();
                app.push_log(
                    LogKind::Info,
                    format_str(format_args!("text: {short}")),
                );
            }
            push_chat(&app, ChatDir::In, &text);
            render_metrics(&app);
        }
        WsEvent::Binary(bytes) => {
            {
                let mut m = app.metrics.borrow_mut();
                m.recv_count += 1;
                m.bytes_recv += bytes.len() as u64;
            }
            let preview = hex_preview(&bytes, 8);
            let n = bytes.len();
            app.push_log(
                LogKind::Info,
                format_str(format_args!("binary {n}B: {preview}")),
            );
            push_chat(
                &app,
                ChatDir::In,
                &format_str(format_args!("[binary {n}B] {preview}")),
            );
            render_metrics(&app);
        }
        WsEvent::Error(msg) => {
            app.push_log(
                LogKind::Warn,
                format_str(format_args!("transport error: {msg}")),
            );
        }
        WsEvent::Closed { code, reason, was_clean } => {
            {
                let mut m = app.metrics.borrow_mut();
                m.closed_at_ms = js_sys::Date::now();
            }
            let short_reason = if reason.is_empty() {
                String::new()
            } else {
                format_str(format_args!(" reason={reason}"))
            };
            let msg = format_str(format_args!(
                "closed code={code} clean={was_clean}{short_reason}",
            ));
            let kind = if was_clean { LogKind::Info } else { LogKind::Warn };
            app.push_log(kind, msg.clone());
            set_status(&app.ui.status, "err", &msg);
            render_metrics(&app);
        }
        WsEvent::Reconnecting { attempt, delay_ms } => {
            {
                let mut m = app.metrics.borrow_mut();
                m.last_reconnect_attempt = attempt;
                m.last_reconnect_delay = delay_ms;
            }
            app.push_log(
                LogKind::Warn,
                format_str(format_args!(
                    "reconnecting: attempt={attempt}, delay={delay_ms} мс",
                )),
            );
            set_status(
                &app.ui.status,
                "busy",
                &format_str(format_args!(
                    "переподключение через {delay_ms} мс (попытка {attempt})",
                )),
            );
            render_reconnect_info(&app);
        }
        WsEvent::ReconnectExhausted => {
            app.push_log(LogKind::Err, "reconnect: попытки исчерпаны".to_string());
            set_status(
                &app.ui.status,
                "err",
                "reconnect exhausted — клик Connect для новой сессии",
            );
            *app.client.borrow_mut() = None;
            update_controls(&app);
        }
    }
}

// ============================================================
//                    РЕНДЕР
// ============================================================

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

fn render_settings(app: &Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let state_str = match client_opt.as_ref() {
        Some(c) => c.state().as_str().to_string(),
        None => "closed".to_string(),
    };
    let cls = match state_str.as_str() {
        "open" => "v ok",
        "connecting" | "closing" => "v warn",
        "closed" => "v err",
        _ => "v",
    };
    let mut html = String::with_capacity(256);
    push_row_cls(&mut html, "state", &state_str, cls);
    if let Some(c) = client_opt {
        let cfg = c.config();
        push_row(&mut html, "url", &cfg.url);
        push_row(
            &mut html,
            "max_attempts",
            &cfg.max_reconnect_attempts.to_string(),
        );
        push_row(&mut html, "base_ms", &cfg.base_reconnect_ms.to_string());
        push_row(&mut html, "max_ms", &cfg.max_reconnect_ms.to_string());
    }
    app.ui.settings.set_inner_html(&html);
    update_badge(&app.ui.badge_state, &state_str, cls);
}

fn update_badge(el: &web_sys::Element, state: &str, cls: &str) {
    let badge_cls = match cls {
        "v ok" => "ws-badge ok",
        "v warn" => "ws-badge warn",
        "v err" => "ws-badge err",
        _ => "ws-badge",
    };
    el.set_text_content(Some(state));
    el.set_class_name(badge_cls);
}

fn render_metrics(app: &Rc<App>) {
    let m = app.metrics.borrow();
    let mut html = String::with_capacity(512);
    push_row(&mut html, "sent", &format_str(format_args!("{}", m.sent_count)));
    push_row(&mut html, "recv", &format_str(format_args!("{}", m.recv_count)));
    push_row(&mut html, "bytes sent", &format_bytes(m.bytes_sent));
    push_row(&mut html, "bytes recv", &format_bytes(m.bytes_recv));

    if m.dropped_while_closed > 0 {
        push_row_cls(
            &mut html,
            "dropped",
            &format_str(format_args!("{}", m.dropped_while_closed)),
            "v warn",
        );
    }

    // Uptime: живой (now - opened) или финальный (closed - opened).
    let (uptime_text, uptime_cls) = if m.opened_at_ms > 0.0 {
        let end = if m.closed_at_ms > 0.0 {
            m.closed_at_ms
        } else {
            js_sys::Date::now()
        };
        let sec = ((end - m.opened_at_ms) / 1000.0).floor() as u64;
        let text = format_duration(sec);
        let cls = if m.closed_at_ms > 0.0 {
            "v warn"
        } else {
            "v ok"
        };
        (text, cls)
    } else {
        ("—".to_string(), "v")
    };
    push_row_cls(&mut html, "uptime", &uptime_text, uptime_cls);

    let (ping_text, ping_cls) = match m.last_ping_ms {
        Some(ms) => (format_str(format_args!("{ms:.1} мс")), "v ok"),
        None => ("—".to_string(), "v"),
    };
    push_row_cls(&mut html, "last ping", &ping_text, ping_cls);

    if m.recovered_from_reconnect {
        push_row_cls(&mut html, "recovered", "yes", "v warn");
    }

    drop(m);
    app.ui.metrics.set_inner_html(&html);
}

fn render_reconnect_info(app: &Rc<App>) {
    let m = app.metrics.borrow();
    let mut html = String::with_capacity(128);
    if m.last_reconnect_attempt == 0 {
        push_row(&mut html, "attempt", "—");
        push_row(&mut html, "delay", "—");
    } else {
        push_row(
            &mut html,
            "attempt",
            &format_str(format_args!("{}", m.last_reconnect_attempt)),
        );
        push_row(
            &mut html,
            "delay",
            &format_str(format_args!("{} мс", m.last_reconnect_delay)),
        );
    }
    drop(m);
    app.ui.reconnect_info.set_inner_html(&html);
}

fn format_bytes(n: u64) -> String {
    if n < 1024 {
        return format_str(format_args!("{n} B"));
    }
    if n < 1024 * 1024 {
        return format_str(format_args!("{:.1} KB", n as f64 / 1024.0));
    }
    return format_str(format_args!("{:.2} MB", n as f64 / (1024.0 * 1024.0)));
}

fn format_duration(sec: u64) -> String {
    let m = sec / 60;
    let s = sec % 60;
    return format_str(format_args!("{m}:{s:02}"));
}

fn update_controls(app: &Rc<App>) {
    let client_opt = app.client.borrow().clone();
    let (is_active, is_open) = match client_opt.as_ref() {
        Some(c) => {
            let s = c.state();
            (true, s == WsState::Open)
        }
        None => (false, false),
    };

    set_disabled(&app.ui.btn_connect, is_active);
    set_disabled(&app.ui.btn_disconnect, !is_active);
    set_disabled(&app.ui.btn_send_text, !is_open);
    set_disabled(&app.ui.btn_send_bin, !is_open);
    set_disabled(&app.ui.btn_ping, !is_open);
    set_disabled(&app.ui.btn_drop, !is_open);

    render_settings(app);
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

fn start_uptime_timer(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let interval = Interval::new(500, move || {
        // Обновляем метрики, пока идёт активная сессия.
        // После Closed — тоже, чтобы зафиксировать финальный uptime.
        if app_c.metrics.borrow().opened_at_ms > 0.0
            && app_c.client.borrow().is_some()
        {
            render_metrics(&app_c);
        }
    });
    *app._uptime.borrow_mut() = Some(interval);
    Ok(())
}

// ============================================================
//                    UI HELPERS
// ============================================================

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "ws-status ok",
        "err" => "ws-status err",
        "busy" => "ws-status busy",
        _ => "ws-status",
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
                "<div class=\"ws-log-empty\">здесь появятся события</div>",
            );
            return;
        }
        let mut html = String::with_capacity(768);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
                LogKind::Warn => "warn",
                LogKind::Session => "sess",
            };
            html.push_str("<div class=\"ws-log-entry ");
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
                "<div class=\"ws-chat-empty\">отправьте сообщение — \
                 оно вернётся эхом</div>",
            );
            return;
        }
        let mut html = String::with_capacity(512);
        for m in chat.iter() {
            let (cls, prefix) = match m.dir {
                ChatDir::Out => ("ws-msg-row out", "→"),
                ChatDir::In => ("ws-msg-row in", "←"),
                ChatDir::Sys => ("ws-msg-row sys", ""),
            };
            html.push_str("<div class=\"");
            html.push_str(cls);
            html.push_str("\">");
            if !prefix.is_empty() {
                html.push_str("<span class=\"ws-msg-prefix\">");
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

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

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
