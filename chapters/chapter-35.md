# Глава 35. MediaDevices: камера и микрофон

`getUserMedia` — API из 2012 года, которое за тринадцать лет почти не изменилось. Его задача формулируется в одну строку: **запросить у пользователя доступ к камере и/или микрофону и вернуть `MediaStream`**. Но вокруг этого простого вызова выросла целая система: как узнать, какие устройства есть, как настроить качество, как переключить камеру на лету, что означают три независимых состояния трека. Об этом — глава.

Полезно сразу обозначить границы темы. В главе 23.8 работал `MediaStreamTrackProcessor`, `VideoFrame`, WASM-обработка кадров. Там предмет — **содержимое потока**. Глава 35 работает с **самим устройством**: какие камеры доступны, какие constraints к ним применить, как заменить активный трек на другой, не перезапуская весь pipeline. Это два разных уровня абстракции, и путать их — источник ошибок.

## 35.1. `enumerateDevices` и главная ловушка

Первое, что делает разработчик при работе с камерой — вызывает `enumerateDevices()`:

```javascript
const devices = await navigator.mediaDevices.enumerateDevices();
console.log(devices);
// [
//   { kind: "videoinput", deviceId: "", groupId: "", label: "" },
//   { kind: "audioinput", deviceId: "", groupId: "", label: "" }
// ]
```

Устройства есть, но `label`, `deviceId` и `groupId` — **пустые строки**. Это не баг и не ошибка окружения.

**Причина — приватность.** Метка камеры — это модель устройства. По ней можно опознать встроенный микрофон ноутбука, внешнюю USB-камеру, Bluetooth-гарнитуру. Если бы сайт мог получить labels без разрешения, это была бы утечка для fingerprinting'а: набор устройств в системе почти уникален.

**Спецификация.** До первого `getUserMedia` браузер обязан вернуть:
- `kind` — `"videoinput" | "audioinput" | "audiooutput"`
- `deviceId` — пустая строка
- `groupId` — пустая строка
- `label` — пустая строка

После успешного `getUserMedia` **того же origin** — labels и ID появляются. Уже разрешённый сайт видит полный список.

**Следствие для UI.** Показать список камер «до разрешения» можно, но там будут не имена, а placeholder'ы. Правильная последовательность:

1. `enumerateDevices()` → показываем «Камера 1, Камера 2» (индексы).
2. Пользователь нажимает «Начать».
3. `getUserMedia({video: true})` → диалог разрешения.
4. **После успеха — повторный `enumerateDevices()`** → labels уже заполнены.

Попытка обойти эту последовательность — использовать «серый» список до разрешения — приведёт к тому, что пользователь увидит «Камера 1» и «Камера 2», выберет одну, и в момент старта окажется, что за этим индексом стояло совсем другое устройство. Правило простое: **до разрешения — placeholder'ы, после — реальные метки, и обновление происходит автоматически**.

**Событие `devicechange`.** Когда пользователь подключает USB-камеру или отключает Bluetooth-наушники, браузер стреляет `mediaDevices.ondevicechange`. По нему надо перечитывать список.

```rust
let cb = Closure::<dyn FnMut()>::new(move || {
    enumerate_and_render(app.clone());
});
media_devices.add_event_listener_with_callback(
    "devicechange",
    cb.as_ref().unchecked_ref(),
)?;
cb.forget();
```

**Ещё один нюанс — стабильность `deviceId`.** В Chrome deviceId стабилен per-origin: перезапуск браузера его не меняет. В Firefox — **меняется при каждом запуске**. Полностью полагаться на deviceId нельзя — только сравнивать в пределах текущей сессии.

**Нумерация устройств в UI.** Распространённая ошибка — использовать глобальный индекс при отрисовке. Если в системе 2 камеры и 3 микрофона, глобальный счётчик даст «Микрофон 3, Микрофон 4, Микрофон 5». Правильно — **отдельный счётчик на каждый `kind`**:

```rust
let mut video_idx = 0u32;
let mut audio_idx = 0u32;
let mut other_idx = 0u32;

for (global_idx, d) in devices.iter().enumerate() {
    let local_idx = match d.kind {
        DeviceKind::VideoInput => { let i = video_idx; video_idx += 1; i }
        DeviceKind::AudioInput => { let i = audio_idx; audio_idx += 1; i }
        _ => { let i = other_idx; other_idx += 1; i }
    };
    // ...
}
```

Это даёт «Камера 1, Камера 2, Микрофон 1, Микрофон 2, Микрофон 3» — то, что ожидает пользователь.

## 35.2. Разрешение: два независимых уровня

Диалог `getUserMedia` — не единственный способ управлять доступом. Есть ещё Permissions API, и они работают параллельно.

**Уровень 1 — диалог.** Первый вызов `getUserMedia` открывает системный запрос: «Разрешить сайту использовать камеру?». Пользователь видит превью и выбирает.

**Уровень 2 — Permissions API.** После ответа `permissions.query({name: 'camera'})` возвращает `granted | denied | prompt`. До запроса — `prompt`.

**Ключевое отличие от других API.** Здесь **две независимые пермиссии** — `camera` и `microphone`. Можно разрешить камеру, но запретить микрофон. Одна выдача не означает другую. Это надо проверять отдельно и показывать в UI отдельными badge'ами.

```rust
async fn query_permission(name: &str) -> Option<String> {
    let promise = query_fn.call1(&permissions, &desc).ok()?;
    let promise: js_sys::Promise = promise.dyn_into().ok()?;
    let result = JsFuture::from(promise).await.ok()?;
    return Reflect::get(&result, &JsValue::from_str("state"))
        .ok()?
        .as_string();
}
```

**Четыре типичные ошибки.**

| Ошибка | Что значит | Что делать |
|---|---|---|
| `NotAllowedError` | Пользователь нажал «Block» | Не восстановить без действий в настройках |
| `NotFoundError` | Камеры/микрофона нет в системе | Сообщить прямо, не предлагать повтор |
| `NotReadableError` | Устройство занято другим приложением (Zoom, Teams, OBS) | Сказать, что закрыть и попробовать снова |
| `OverconstrainedError` | Constraints невыполнимы | Назвать, какое ограничение не удалось |

Сообщение `NotReadableError: Device in use undefined` — не для пользователя. Нужна челovekoчитаемая замена: «Устройство занято другим приложением (Zoom, Teams, OBS). Закройте приложения, использующие камеру, и попробуйте снова».

## 35.3. Constraints: `ideal` vs `exact`

`MediaTrackConstraints` — центральный элемент API. Пример:

```javascript
{
    video: {
        width: { ideal: 1920 },
        height: { ideal: 1080 },
        frameRate: { ideal: 30, max: 60 },
        facingMode: { ideal: 'environment' },
        deviceId: { exact: 'abc123...' }
    },
    audio: {
        echoCancellation: true,
        noiseSuppression: true
    }
}
```

**Ключевое различие.**

| Тип | Что значит | Если невыполнимо |
|---|---|---|
| `{ ideal: X }` | Хочу X, но готов на компромисс | Браузер вернёт ближайшее возможное |
| `{ exact: X }` | Требую ровно X | `OverconstrainedError`, поток не получен |
| `{ min: X, max: Y }` | Диапазон | `OverconstrainedError`, если вне диапазона |
| `X` (число) | Эквивалент `{ ideal: X }` | То же |

**Правило.** Почти всегда `ideal`. `exact` — только когда пользователь явно выбрал конкретное устройство из списка. Использовать `exact: width` означает, что при попытке на камере без 1920 получите ошибку вместо 1280.

**`facingMode` — только для mobile.** `'user'` (фронтальная) и `'environment'` (основная). На десктопе игнорируется. Всегда `ideal`, иначе на десктопе `OverconstrainedError`.

**`getSettings()` — что получилось на самом деле.** После получения трека можно узнать фактические параметры:

```rust
pub fn get_settings(track: &MediaStreamTrack) -> TrackSettings {
    let f = Reflect::get(track, &JsValue::from_str("getSettings"))
        .ok()
        .and_then(|v| v.dyn_into::<js_sys::Function>().ok())?;
    let result = f.call0(track.as_ref()).ok()?;
    TrackSettings::from_js(&result)
}
```

Это важно: `ideal: 1920` — это **пожелание**, а не гарантия. Реально камера может выдать 1280×720. `getSettings` покажет фактическое значение — именно его и нужно отображать в UI.

## 35.4. `MediaStreamTrack` — три состояния

Трек — не просто «включён» или «выключен». Есть **три независимых понятия**, и путать их — источник ошибок.

**1. `track.enabled = false` — mute, обратимо.** Трек продолжает жить, устройство остаётся занято, но в `MediaStream` идут чёрные кадры (для видео) или тишина (для аудио). Обратимо: `enabled = true` вернёт исходное. Применение: mute-кнопка в конференции.

**2. `track.stop()` — release, необратимо.** Устройство освобождается. Вернуть нельзя — нужно снова `getUserMedia`. Применение: кнопка «Stop».

**3. `track.onended` — событие.** Срабатывает, когда:
- пользователь отключил камеру через системный трей;
- другое приложение забрало устройство (USB unplug);
- другая вкладка заняла камеру (в некоторых браузерах).

После `onended` трек становится `readyState = 'ended'` — невалидным. Нужно снова `getUserMedia`.

**`muted` vs `enabled`.** Разные вещи:
- `enabled = false` — **вы** замьютили;
- `muted = true` — **браузер** замумл (например, другой процесс занял микрофон).

Три состояния независимы. Демо показывает их в отдельной панели: `video.enabled`, `video.muted`, `video.readyState`. Пользователь видит, что mute и readyState — не одно и то же.

**Слушатель `onended`.** В Rust — `Closure` в `set_onended`:

```rust
let cb = Closure::<dyn FnMut()>::new(move || {
    app.push_log(LogKind::Warn,
        format_str(format_args!("track {kind} завершён")));
    render_track_state(&app);
    set_status(&app.ui.status, "err",
        "трек завершён — перезапустите поток");
});
track.set_onended(Some(cb.as_ref().unchecked_ref()));
```

Closure обязательно хранить в структуре `App` (`track_closures: Rc<RefCell<Vec<Closure<...>>>>`). Иначе он соберётся GC и обработчик не сработает.

## 35.5. Hot-swap: переключение без перезапуска

Сценарий: пользователь в видео-звонке хочет сменить камеру. Нельзя перезапускать весь `getUserMedia` — потеряется аудио, сбросится соединение с пирами. Правильный паттерн:

```javascript
// 1. Запросить только новую камеру, без audio.
const newStream = await navigator.mediaDevices.getUserMedia({
    video: { deviceId: { exact: newDeviceId } },
});

// 2. Заменить только video track в старом stream.
const oldStream = videoElement.srcObject;
const oldTrack = oldStream.getVideoTracks()[0];
const newTrack = newStream.getVideoTracks()[0];
oldStream.removeTrack(oldTrack);
oldTrack.stop();
oldStream.addTrack(newTrack);

// 3. Перепривязать srcObject и явно перезапустить playback.
videoElement.srcObject = oldStream;
await videoElement.play();
```

**Ключевые детали.**

- **Новый `getUserMedia` — только для нужного kind.** Запрашиваем video (или audio) отдельно, не весь поток. Экономит ресурсы, не сбрасывает другой трек.
- **`removeTrack` + `addTrack` на существующем stream.** Не пересоздаём `MediaStream`. Другие потребители (WebRTC peer, MediaRecorder) продолжат работу.
- **Явный `.play()`.** `srcObject = oldStream` сам по себе playback не перезапускает в Chrome и Firefox.
- **`old_track.stop()` после `removeTrack`.** Иначе устройство остаётся занято.

**Защита от повторного клика.** Клик по активной камере не должен пересоздавать поток. Проверяем `current_track_device_id`:

```rust
let active_id = current_track_device_id(&app, "video");
if active_id == device.device_id {
    set_status(&app.ui.status, "busy", "эта камера уже активна");
    return;
}
```

**`kind` для `spawn_local`.** Замыкание требует `'static`, а параметр `kind: &str` живёт только на время функции. Решение — превратить в `String` до входа в async-блок:

```rust
fn hot_swap_track(app: Rc<App>, device: DeviceInfo, kind: &str) {
    let kind_owned = kind.to_string();  // ← до spawn_local
    let label = if device.has_label() {
        device.label.clone()
    } else {
        kind_owned.clone()
    };

    spawn_local(async move {
        let is_video = kind_owned == "video";
        // ...
    });
}
```

Это компромисс между строгостью lifetimes и `'static` из `spawn_local`. `Rc<App>` тоже передаётся по значению — клонирование Rc дёшево, а `&Rc<App>` в замыкание не поместится.

## 35.6. Практическое camera application

Собираем всё вместе. Демо — панель управления камерой, как в Zoom, но без WebRTC.

**Три колонки UI:**

1. **Управление** — Start / Stop / Mute, constraints panel, permissions badge, track state.
2. **Устройства** — группированный список камер и микрофонов с hot-swap по клику. Журнал событий.
3. **Превью** — `<video>` + фактические `getSettings()` + диагностика.

**Что демонстрирует:**

- **enumerate before permission** — пользователь видит «Камера 1, Камера 2», кликает «Начать», разрешает — labels заполняются автоматически.
- **Hot-swap** — клик по другой камере в списке меняет только video track, аудио остаётся.
- **Mute toggle** — `track.enabled = false/true` для соответствующего канала, кнопка меняет иконку.
- **Track state** — три состояния каждого трека в реальном времени: enabled, muted, readyState.
- **Диагностика** — снимок окружения по кнопке.

**Уникальный префикс `md-`** — не конфликтует с `gl-`, `ms-`, `cd-`, `aw-` из других глав.

## 35.7. Исходные файлы

**`src/media_devices.rs`** — модуль с `DeviceInfo`, `DeviceKind`, `StreamConstraints`, `FacingMode`, `TrackSettings`, `TrackState`, `enumerate_devices`, `request_stream`, `get_settings`, `read_track_state`, `format_media_error`. Все вызовы через `Reflect` для `enumerateDevices` и `getUserMedia` — это даёт независимость от версии web-sys.

**`tests/wasm_media_devices.rs`** — 15 тестов:
- feature detection (3 теста);
- `DeviceKind::from_str` roundtrip для известных вариантов;
- catch-all поведение `Unknown`;
- `display_label` с placeholder'ом и с реальной меткой;
- сборка constraints: `ideal` не `exact`, `deviceId: exact`, `facingMode: ideal`;
- парсинг `TrackSettings` из JS-объекта;
- форматирование трёх типов ошибок.

**`src/app.rs`** — демо-пример. Три колонки, общее состояние в `Rc<RefCell<...>>`, closures треков хранятся в `App`.

**`Cargo.toml`** — добавлены восемь фич:
```toml
"MediaDevices", "MediaDeviceInfo", "MediaDeviceKind",
"MediaStreamConstraints", "MediaTrackConstraints",
"MediaTrackSettings", "MediaTrackCapabilities",
"MediaTrackSupportedConstraints", "MediaTrackConstraintSet",
"InputDeviceInfo", "PermissionStatus",
```

## 35.8. Анализ решения

**Сильные стороны.**

1. **Разграничение с главой 23.8.** Там — pixel-level processing, здесь — device management. Не пересекаются.
2. **Показ главной ловушки.** Пустые labels до разрешения — центральная демонстрация. Пользователь видит «Камера 1, Камера 2», кликает «Начать», разрешает — список мгновенно перестраивается.
3. **Hot-swap для видео и аудио.** `removeTrack` + `addTrack` — рабочий паттерн для видеоконференций. Демонстрирует, что `MediaStream` — контейнер с изменяемым составом.
4. **Track lifecycle полностью.** `enabled`, `muted`, `readyState` — три состояния независимы, видны в UI.
5. **Человекочитаемые ошибки.** `NotReadableError` → конкретное действие. `OverconstrainedError` → какое ограничение не выполнено.

**Скрытые ошибки, которых удалось избежать.**

1. **`srcObject` без `.play()`.** Браузеры не перезапускают playback при переприсваивании `srcObject`. Явный `.play()` в `attach_preview`.
2. **`facingMode: exact` на десктопе.** `OverconstrainedError` при выборе `facingMode: {exact: 'environment'}`. Используем `{ideal: ...}`, чтобы на десктопе игнорировалось.
3. **`enumerateDevices` до `getUserMedia`.** Пустые labels. Демо показывает placeholder'ы и обновляет после разрешения.
4. **`stop()` перед заменой track.** В hot-swap сначала `old_track.stop()`, потом `addTrack(new)`. Иначе устройство остаётся занято.
5. **Closures треков в `App`.** Если не сохранить — соберутся GC, `onended` перестанет срабатывать.
6. **Неиспользуемые импорты.** Убраны `Document`, `Event` — они были только для заглушек.

**Правила проекта.**

- `format_str(format_args!(...))` в tail-позициях — везде.
- HTML через `push_str` при ≥2 кавычках.
- `esc_html` для всех динамических строк.
- Уникальный префикс `md-`.
- Отдельные функции для каждой группы операций.
- Общее состояние в `Rc<RefCell<...>>`.
- Функции, вызываемые из замыканий и async-блоков, принимают `Rc<App>` по значению. Read-only функции — `&Rc<App>`.

## 35.9. Итог

**MediaDevices — управление устройствами, не содержимым потока.** Четыре ключевых вопроса: какие устройства есть, какие constraints применить, как переключать без перезапуска, что означают три состояния трека.

**Главная ловушка — пустые labels до разрешения.** `enumerateDevices` заполняет `deviceId` и `label` только после успешного `getUserMedia`. Это защита от fingerprinting, не баг. Правильная последовательность: enumerate → getUserMedia → enumerate повторно.

**Constraints — `ideal` vs `exact`.** Почти всегда `ideal`. `exact` только когда пользователь явно выбрал конкретное устройство.

**`facingMode` — только mobile.** На десктопе игнорируется. Всегда `ideal`.

**Три состояния трека.** `enabled = false` — mute. `stop()` — release. `onended` — устройство отключено. Три независимых понятия.

**Hot-swap — `removeTrack` + `addTrack`.** Не пересоздаём `MediaStream`. Аудио остаётся живым.

**`srcObject` + `play()`.** Присваивание `srcObject` не перезапускает playback автоматически.

**Человекочитаемые ошибки.** `NotAllowedError`, `NotFoundError`, `NotReadableError`, `OverconstrainedError` — четыре разных сообщения, четыре разных действия.




## 35.10. Приложение. Исходные коды проекта.

**Cargo.toml**:
```
    # MediaDevices (глава 35)
    "MediaDevices",
    "MediaDeviceInfo",
    "MediaDeviceKind",
    "MediaStreamConstraints",
    "MediaTrackConstraints",
    "MediaTrackSettings",
    "MediaTrackCapabilities",
    "MediaTrackSupportedConstraints",
    "MediaTrackConstraintSet",
    "InputDeviceInfo",
    "PermissionStatus",
```

**media_devices.rs**:
```
//! 35. MediaDevices: камера и микрофон.
//!
//! Обёртки вокруг enumerateDevices / getUserMedia / getCapabilities.
//!
//! # Ключевое отличие от 23.8
//!
//! Глава 23.8 работает с **кадрами**: MediaStreamTrackProcessor →
//! WASM → MediaStreamTrackGenerator. Это пиксельный уровень.
//!
//! Глава 35 работает с **устройствами**: какие камеры доступны,
//! какие constraints применить, как переключать камеру на лету.
//! Это уровень device management.

use js_sys::{Array, Object, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{MediaStream, MediaStreamTrack};

use crate::format_str::format_str;

// ============================================================
//              FEATURE DETECTION
// ============================================================

pub fn has_media_devices() -> bool {
    return web_sys::window()
        .map(|w| {
            js_sys::Reflect::get(&w.navigator(), &JsValue::from_str("mediaDevices"))
                .map(|v| !v.is_undefined() && !v.is_null())
                .unwrap_or(false)
        })
        .unwrap_or(false);
}

pub fn has_get_user_media() -> bool {
    if !has_media_devices() {
        return false;
    }
    return web_sys::window()
        .and_then(|w| js_sys::Reflect::get(&w.navigator(), &JsValue::from_str("mediaDevices")).ok())
        .and_then(|md| js_sys::Reflect::get(&md, &JsValue::from_str("getUserMedia")).ok())
        .map(|v| v.is_function())
        .unwrap_or(false);
}

pub fn has_enumerate_devices() -> bool {
    if !has_media_devices() {
        return false;
    }
    return web_sys::window()
        .and_then(|w| js_sys::Reflect::get(&w.navigator(), &JsValue::from_str("mediaDevices")).ok())
        .and_then(|md| js_sys::Reflect::get(&md, &JsValue::from_str("enumerateDevices")).ok())
        .map(|v| v.is_function())
        .unwrap_or(false);
}

// ============================================================
//              DEVICE INFO
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum DeviceKind {
    VideoInput,
    AudioInput,
    AudioOutput,
    Unknown,
}

impl DeviceKind {
    pub fn from_str(s: &str) -> Self {
        return match s {
            "videoinput" => DeviceKind::VideoInput,
            "audioinput" => DeviceKind::AudioInput,
            "audiooutput" => DeviceKind::AudioOutput,
            _ => DeviceKind::Unknown,
        };
    }
    pub fn as_str(self) -> &'static str {
        return match self {
            DeviceKind::VideoInput => "videoinput",
            DeviceKind::AudioInput => "audioinput",
            DeviceKind::AudioOutput => "audiooutput",
            DeviceKind::Unknown => "unknown",
        };
    }
}

#[derive(Clone, Debug)]
pub struct DeviceInfo {
    pub kind: DeviceKind,
    pub device_id: String,
    pub group_id: String,
    pub label: String,
}

impl DeviceInfo {
    pub fn from_js(v: &JsValue) -> Self {
        return Self {
            kind: DeviceKind::from_str(
                &js_sys::Reflect::get(v, &JsValue::from_str("kind"))
                    .ok()
                    .and_then(|k| k.as_string())
                    .unwrap_or_default(),
            ),
            device_id: js_sys::Reflect::get(v, &JsValue::from_str("deviceId"))
                .ok()
                .and_then(|x| x.as_string())
                .unwrap_or_default(),
            group_id: js_sys::Reflect::get(v, &JsValue::from_str("groupId"))
                .ok()
                .and_then(|x| x.as_string())
                .unwrap_or_default(),
            label: js_sys::Reflect::get(v, &JsValue::from_str("label"))
                .ok()
                .and_then(|x| x.as_string())
                .unwrap_or_default(),
        };
    }

    pub fn has_label(&self) -> bool {
        return !self.label.is_empty();
    }

    /// Показывает label, либо «Камера N» для случая без разрешения.
    pub fn display_label(&self, index: usize) -> String {
        if self.has_label() {
            return self.label.clone();
        }
        return match self.kind {
            DeviceKind::VideoInput => format_str(format_args!("Камера {}", index + 1)),
            DeviceKind::AudioInput => format_str(format_args!("Микрофон {}", index + 1)),
            DeviceKind::AudioOutput => format_str(format_args!("Динамик {}", index + 1)),
            DeviceKind::Unknown => format_str(format_args!("Устройство {}", index + 1)),
        };
    }
}

/// Обёртка над `enumerateDevices()`.
pub async fn enumerate_devices() -> Result<Vec<DeviceInfo>, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let md = js_sys::Reflect::get(&window.navigator(), &JsValue::from_str("mediaDevices"))?;
    let f = js_sys::Reflect::get(&md, &JsValue::from_str("enumerateDevices"))?;
    let f: js_sys::Function = f
        .dyn_into()
        .map_err(|_| JsValue::from_str("enumerateDevices не функция"))?;
    let promise = f.call0(&md)?.dyn_into::<Promise>()?;
    let result = JsFuture::from(promise).await?;
    let array: Array = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("enumerateDevices не вернул Array"))?;
    let mut out = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        out.push(DeviceInfo::from_js(&array.get(i)));
    }
    return Ok(out);
}

// ============================================================
//              CONSTRAINTS
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum FacingMode {
    /// Любое лицо (по умолчанию).
    Any,
    /// Фронтальная камера (селфи).
    User,
    /// Основная камера (тыл).
    Environment,
}

impl FacingMode {
    pub fn as_str(self) -> &'static str {
        return match self {
            FacingMode::Any => "",
            FacingMode::User => "user",
            FacingMode::Environment => "environment",
        };
    }
}

#[derive(Clone, Debug)]
pub struct StreamConstraints {
    /// Запрашивать видео? Если false — параметры ниже игнорируются.
    pub video: bool,
    /// Запрашивать аудио?
    pub audio: bool,
    /// Идеальная ширина, пиксели.
    pub width: u32,
    /// Идеальная высота, пиксели.
    pub height: u32,
    /// Идеальный fps.
    pub frame_rate: u32,
    /// `facingMode` (только для мобильных).
    pub facing: FacingMode,
    /// Конкретный deviceId (exact). Пустая строка — не указывать.
    pub video_device_id: String,
    pub audio_device_id: String,
    /// Эхо-подавление (для аудио).
    pub echo_cancellation: bool,
}

impl Default for StreamConstraints {
    fn default() -> Self {
        return Self {
            video: true,
            audio: false,
            width: 1280,
            height: 720,
            frame_rate: 30,
            facing: FacingMode::Any,
            video_device_id: String::new(),
            audio_device_id: String::new(),
            echo_cancellation: true,
        };
    }
}

/// Собирает `MediaStreamConstraints` как JS-объект.
///
/// Для каждой характеристики используется `{ ideal: value }` —
/// браузер вернёт ближайшее доступное, не выбросит
/// `OverconstrainedError` при невыполнении.
pub fn build_constraints(c: &StreamConstraints) -> Result<JsValue, JsValue> {
    let root = Object::new();

    // --- video ---
    if c.video {
        let video = Object::new();
        Reflect::set(&video, &JsValue::from_str("width"), &ideal(c.width as f64)?)?;
        Reflect::set(
            &video,
            &JsValue::from_str("height"),
            &ideal(c.height as f64)?,
        )?;
        Reflect::set(
            &video,
            &JsValue::from_str("frameRate"),
            &ideal(c.frame_rate as f64)?,
        )?;
        if !c.video_device_id.is_empty() {
            let dev = Object::new();
            Reflect::set(
                &dev,
                &JsValue::from_str("exact"),
                &JsValue::from_str(&c.video_device_id),
            )?;
            Reflect::set(&video, &JsValue::from_str("deviceId"), &dev)?;
        }
        if c.facing != FacingMode::Any {
            let fm = Object::new();
            Reflect::set(
                &fm,
                &JsValue::from_str("ideal"),
                &JsValue::from_str(c.facing.as_str()),
            )?;
            Reflect::set(&video, &JsValue::from_str("facingMode"), &fm)?;
        }
        Reflect::set(&root, &JsValue::from_str("video"), &video)?;
    } else {
        Reflect::set(&root, &JsValue::from_str("video"), &JsValue::FALSE)?;
    }

    // --- audio ---
    if c.audio {
        let audio = Object::new();
        Reflect::set(
            &audio,
            &JsValue::from_str("echoCancellation"),
            &JsValue::from_bool(c.echo_cancellation),
        )?;
        Reflect::set(
            &audio,
            &JsValue::from_str("noiseSuppression"),
            &JsValue::TRUE,
        )?;
        if !c.audio_device_id.is_empty() {
            let dev = Object::new();
            Reflect::set(
                &dev,
                &JsValue::from_str("exact"),
                &JsValue::from_str(&c.audio_device_id),
            )?;
            Reflect::set(&audio, &JsValue::from_str("deviceId"), &dev)?;
        }
        Reflect::set(&root, &JsValue::from_str("audio"), &audio)?;
    } else {
        Reflect::set(&root, &JsValue::from_str("audio"), &JsValue::FALSE)?;
    }

    return Ok(root.into());
}

fn ideal(value: f64) -> Result<JsValue, JsValue> {
    let obj = Object::new();
    Reflect::set(&obj, &JsValue::from_str("ideal"), &JsValue::from_f64(value))?;
    return Ok(obj.into());
}

// ============================================================
//              GET USER MEDIA
// ============================================================

pub async fn request_stream(c: &StreamConstraints) -> Result<MediaStream, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let md = js_sys::Reflect::get(&window.navigator(), &JsValue::from_str("mediaDevices"))?;
    let f = js_sys::Reflect::get(&md, &JsValue::from_str("getUserMedia"))?;
    let f: js_sys::Function = f
        .dyn_into()
        .map_err(|_| JsValue::from_str("getUserMedia не функция"))?;
    let constraints = build_constraints(c)?;
    let promise = f.call1(&md, &constraints)?.dyn_into::<Promise>()?;
    let stream = JsFuture::from(promise).await?;
    return stream.dyn_into::<MediaStream>();
}

// ============================================================
//              TRACK INSPECTION
// ============================================================

#[derive(Clone, Debug, Default)]
pub struct TrackSettings {
    pub width: Option<u32>,
    pub height: Option<u32>,
    pub frame_rate: Option<f64>,
    pub device_id: Option<String>,
    pub facing_mode: Option<String>,
}

impl TrackSettings {
    pub fn from_js(v: &JsValue) -> Self {
        let read_num = |key: &str| -> Option<f64> {
            js_sys::Reflect::get(v, &JsValue::from_str(key))
                .ok()
                .and_then(|x| x.as_f64())
        };
        let read_str = |key: &str| -> Option<String> {
            js_sys::Reflect::get(v, &JsValue::from_str(key))
                .ok()
                .and_then(|x| x.as_string())
        };
        return Self {
            width: read_num("width").map(|n| n as u32),
            height: read_num("height").map(|n| n as u32),
            frame_rate: read_num("frameRate"),
            device_id: read_str("deviceId"),
            facing_mode: read_str("facingMode"),
        };
    }
}

pub fn get_settings(track: &MediaStreamTrack) -> TrackSettings {
    let f = js_sys::Reflect::get(track, &JsValue::from_str("getSettings"))
        .ok()
        .and_then(|v| v.dyn_into::<js_sys::Function>().ok());
    let Some(f) = f else {
        return TrackSettings::default();
    };
    let Ok(result) = f.call0(track.as_ref()) else {
        return TrackSettings::default();
    };
    return TrackSettings::from_js(&result);
}

/// Простой трек-статус для UI.
#[derive(Clone, Debug)]
pub struct TrackState {
    pub kind: String,
    pub label: String,
    pub enabled: bool,
    pub muted: bool,
    pub ready_state: String,
}

pub fn read_track_state(track: &MediaStreamTrack) -> TrackState {
    let read_bool = |key: &str| -> bool {
        js_sys::Reflect::get(track, &JsValue::from_str(key))
            .ok()
            .and_then(|v| v.as_bool())
            .unwrap_or(false)
    };
    let read_str = |key: &str| -> String {
        js_sys::Reflect::get(track, &JsValue::from_str(key))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default()
    };
    return TrackState {
        kind: read_str("kind"),
        label: read_str("label"),
        enabled: read_bool("enabled"),
        muted: read_bool("muted"),
        ready_state: read_str("readyState"),
    };
}

// ============================================================
//              ERRORS
// ============================================================

pub fn format_media_error(e: &JsValue) -> String {
    let name = js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    return match name.as_str() {
        "NotAllowedError" | "PermissionDeniedError" => "Доступ к камере/микрофону отклонён. \
             Разрешите его в настройках браузера (иконка замка \
             в адресной строке) и попробуйте снова."
            .to_string(),
        "NotFoundError" | "DevicesNotFoundError" => {
            "Устройство не найдено: камера или микрофон отсутствуют. \
             Подключите устройство и повторите."
                .to_string()
        }
        "NotReadableError" | "TrackStartError" => {
            "Устройство занято другим приложением (Zoom, Teams, OBS). \
             Закройте приложения, использующие камеру, и попробуйте снова."
                .to_string()
        }
        "OverconstrainedError" | "ConstraintNotSatisfiedError" => {
            let constraint = js_sys::Reflect::get(e, &JsValue::from_str("constraint"))
                .ok()
                .and_then(|v| v.as_string())
                .unwrap_or_default();
            format_str(format_args!(
                "Запрошенные параметры невыполнимы: {constraint}. \
                 Уменьшите разрешение или fps.",
            ))
        }
        "SecurityError" => "Нарушение политики безопасности. Проверьте, что страница \
             на HTTPS или localhost."
            .to_string(),
        "AbortError" => "Запрос прерван. Попробуйте снова.".to_string(),
        _ => {
            let msg = js_sys::Reflect::get(e, &JsValue::from_str("message"))
                .ok()
                .and_then(|v| v.as_string())
                .unwrap_or_else(|| String::from("неизвестная ошибка"));
            format_str(format_args!("getUserMedia: {name} — {msg}."))
        }
    };
}

// ============================================================
//              STOP
// ============================================================

/// Освобождает все треки потока. Идемпотентно.
pub fn stop_stream(stream: &MediaStream) {
    let tracks = stream.get_tracks();
    for i in 0..tracks.length() {
        if let Ok(track) = tracks.get(i).dyn_into::<MediaStreamTrack>() {
            track.stop();
        }
    }
}

```


**wasm_media_devices.rs**:
```
//! 35. Wasm-тесты media_devices: feature detection, constraints,
//! парсинг device info, форматирование ошибок.
//!
//! ПРАВИЛО: roundtrip from_str(as_str()) проверяем только для
//! известных вариантов. Catch-all вариант (Unknown) участвует в
//! отдельном тесте — он не хранит исходную строку и не может
//! восстановить её.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::media_devices::{
    self, DeviceInfo, DeviceKind, FacingMode, StreamConstraints, TrackSettings,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- Feature detection ----------------

#[wasm_bindgen_test]
fn wasm_media_devices_supported() {
    assert!(media_devices::has_media_devices());
}

#[wasm_bindgen_test]
fn wasm_get_user_media_supported() {
    assert!(media_devices::has_get_user_media());
}

#[wasm_bindgen_test]
fn wasm_enumerate_devices_supported() {
    assert!(media_devices::has_enumerate_devices());
}

// ---------------- Device kind ----------------

#[wasm_bindgen_test]
fn wasm_device_kind_known_roundtrip() {
    // Roundtrip from_str(as_str()) == x выполняется для всех
    // известных вариантов.
    let cases: [(&str, DeviceKind); 3] = [
        ("videoinput", DeviceKind::VideoInput),
        ("audioinput", DeviceKind::AudioInput),
        ("audiooutput", DeviceKind::AudioOutput),
    ];
    for (s, k) in cases {
        assert_eq!(DeviceKind::from_str(s), k);
        assert_eq!(k.as_str(), s);
    }
}

#[wasm_bindgen_test]
fn wasm_device_kind_unknown_catch_all() {
    // Неизвестная строка → Unknown. Обратного roundtrip нет:
    // Unknown.as_str() возвращает каноничный "unknown", а не исходную
    // строку. Проверяем именно это поведение.
    assert_eq!(DeviceKind::from_str("garbage"), DeviceKind::Unknown);
    assert_eq!(DeviceKind::from_str(""), DeviceKind::Unknown);
    assert_eq!(DeviceKind::from_str("VideoInput"), DeviceKind::Unknown);
    assert_eq!(DeviceKind::Unknown.as_str(), "unknown");
}

// ---------------- Device info ----------------

#[wasm_bindgen_test]
fn wasm_device_display_label_without_permission() {
    // До разрешения label пуст — показываем «Камера N».
    let d = DeviceInfo {
        kind: DeviceKind::VideoInput,
        device_id: String::new(),
        group_id: String::new(),
        label: String::new(),
    };
    assert!(!d.has_label());
    assert_eq!(d.display_label(0), "Камера 1");
    assert_eq!(d.display_label(2), "Камера 3");

    let d = DeviceInfo {
        kind: DeviceKind::AudioInput,
        device_id: String::new(),
        group_id: String::new(),
        label: String::new(),
    };
    assert_eq!(d.display_label(0), "Микрофон 1");
}

#[wasm_bindgen_test]
fn wasm_device_display_label_with_permission() {
    let d = DeviceInfo {
        kind: DeviceKind::VideoInput,
        device_id: "abc123".to_string(),
        group_id: "grp".to_string(),
        label: "FaceTime HD Camera".to_string(),
    };
    assert!(d.has_label());
    assert_eq!(d.display_label(0), "FaceTime HD Camera");
}

// ---------------- Constraints building ----------------

#[wasm_bindgen_test]
fn wasm_constraints_video_only() {
    let c = StreamConstraints {
        video: true,
        audio: false,
        ..Default::default()
    };
    let obj = media_devices::build_constraints(&c).expect("build");
    let video = js_sys::Reflect::get(
        &obj,
        &wasm_bindgen::JsValue::from_str("video"),
    )
    .unwrap();
    assert!(video.is_object());
    let audio = js_sys::Reflect::get(
        &obj,
        &wasm_bindgen::JsValue::from_str("audio"),
    )
    .unwrap();
    assert!(!audio.as_bool().unwrap_or(true));
}

#[wasm_bindgen_test]
fn wasm_constraints_ideal_not_exact() {
    // ideal — не бросает OverconstrainedError при невыполнении.
    let c = StreamConstraints::default();
    let obj = media_devices::build_constraints(&c).expect("build");
    let video = js_sys::Reflect::get(
        &obj,
        &wasm_bindgen::JsValue::from_str("video"),
    )
    .unwrap();
    let width = js_sys::Reflect::get(
        &video,
        &wasm_bindgen::JsValue::from_str("width"),
    )
    .unwrap();
    let ideal = js_sys::Reflect::get(
        &width,
        &wasm_bindgen::JsValue::from_str("ideal"),
    )
    .unwrap();
    assert_eq!(ideal.as_f64(), Some(1280.0));
    let exact = js_sys::Reflect::get(
        &width,
        &wasm_bindgen::JsValue::from_str("exact"),
    )
    .ok();
    assert!(exact.is_none() || exact.unwrap().is_undefined());
}

#[wasm_bindgen_test]
fn wasm_constraints_device_id_exact() {
    let c = StreamConstraints {
        video: true,
        video_device_id: "device-xyz".to_string(),
        ..Default::default()
    };
    let obj = media_devices::build_constraints(&c).expect("build");
    let video = js_sys::Reflect::get(
        &obj,
        &wasm_bindgen::JsValue::from_str("video"),
    )
    .unwrap();
    let dev = js_sys::Reflect::get(
        &video,
        &wasm_bindgen::JsValue::from_str("deviceId"),
    )
    .unwrap();
    let exact = js_sys::Reflect::get(
        &dev,
        &wasm_bindgen::JsValue::from_str("exact"),
    )
    .unwrap();
    assert_eq!(exact.as_string(), Some("device-xyz".to_string()));
}

#[wasm_bindgen_test]
fn wasm_constraints_facing_mode_user() {
    let c = StreamConstraints {
        facing: FacingMode::User,
        ..Default::default()
    };
    let obj = media_devices::build_constraints(&c).expect("build");
    let video = js_sys::Reflect::get(
        &obj,
        &wasm_bindgen::JsValue::from_str("video"),
    )
    .unwrap();
    let fm = js_sys::Reflect::get(
        &video,
        &wasm_bindgen::JsValue::from_str("facingMode"),
    )
    .unwrap();
    let ideal = js_sys::Reflect::get(
        &fm,
        &wasm_bindgen::JsValue::from_str("ideal"),
    )
    .unwrap();
    assert_eq!(ideal.as_string(), Some("user".to_string()));
}

// ---------------- Track settings parsing ----------------

#[wasm_bindgen_test]
fn wasm_track_settings_parses_numeric() {
    let v = js_sys::Object::new();
    js_sys::Reflect::set(
        &v,
        &wasm_bindgen::JsValue::from_str("width"),
        &1280.0.into(),
    )
    .unwrap();
    js_sys::Reflect::set(
        &v,
        &wasm_bindgen::JsValue::from_str("height"),
        &720.0.into(),
    )
    .unwrap();
    js_sys::Reflect::set(
        &v,
        &wasm_bindgen::JsValue::from_str("frameRate"),
        &30.0.into(),
    )
    .unwrap();
    let s = TrackSettings::from_js(&v);
    assert_eq!(s.width, Some(1280));
    assert_eq!(s.height, Some(720));
    assert_eq!(s.frame_rate, Some(30.0));
}

// ---------------- Error formatting ----------------

#[wasm_bindgen_test]
fn wasm_format_error_not_allowed_is_human() {
    let e = js_sys::Error::new("denied");
    js_sys::Reflect::set(
        &e,
        &wasm_bindgen::JsValue::from_str("name"),
        &"NotAllowedError".into(),
    )
    .unwrap();
    let msg = media_devices::format_media_error(&e);
    assert!(msg.contains("отклонён"));
    assert!(msg.contains("настройках"));
}

#[wasm_bindgen_test]
fn wasm_format_error_not_readable_mentions_other_apps() {
    let e = js_sys::Error::new("in use");
    js_sys::Reflect::set(
        &e,
        &wasm_bindgen::JsValue::from_str("name"),
        &"NotReadableError".into(),
    )
    .unwrap();
    let msg = media_devices::format_media_error(&e);
    assert!(msg.contains("занято"));
    assert!(msg.contains("Zoom") || msg.contains("Teams"));
}

#[wasm_bindgen_test]
fn wasm_format_error_overconstrained_mentions_constraint() {
    let e = js_sys::Error::new("too high");
    js_sys::Reflect::set(
        &e,
        &wasm_bindgen::JsValue::from_str("name"),
        &"OverconstrainedError".into(),
    )
    .unwrap();
    js_sys::Reflect::set(
        &e,
        &wasm_bindgen::JsValue::from_str("constraint"),
        &"width".into(),
    )
    .unwrap();
    let msg = media_devices::format_media_error(&e);
    assert!(msg.contains("width"));
}
```


**app.rs**:
```
//! Глава 35: MediaDevices — камера и микрофон.
//!
//! Практическое приложение управления камерой:
//!   1. Список устройств с группировкой по типу.
//!   2. Индикация «нет разрешения» (пустые labels).
//!   3. Start / Stop / Mute с визуальным состоянием.
//!   4. Constraints panel (width/height/fps/facing).
//!   5. Hot-swap видео и аудио без перезапуска потока.
//!   6. Track state: enabled / muted / readyState + onended.
//!   7. Permissions badge (camera / microphone).
//!   8. Диагностика окружения.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: srcObject + play() — не полагаемся на auto-play.
//! ПРАВИЛО: closures треков храним в App — иначе соберутся GC.

use std::cell::RefCell;
use std::rc::Rc;

use js_sys::Reflect;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;
use web_sys::{
    HtmlElement, HtmlSelectElement, HtmlVideoElement, MediaStream,
    MediaStreamTrack,
};

use crate::format_str::format_str;
use crate::media_devices::{
    self, DeviceInfo, DeviceKind, FacingMode, StreamConstraints, TrackState,
};

const LOG_LIMIT: usize = 12;

const HTML: &str = r#"
<style>
  :root {
    --md-info: #3498db; --md-ok: #27ae60; --md-err: #c0392b; --md-warn: #e67e22;
    --md-text: #222; --md-muted: #666;
    --md-bg: #fff; --md-soft: #fafafa; --md-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --md-text: #e6e6e6; --md-muted: #a0a0a0;
      --md-bg: #1e1e1e; --md-soft: #262626; --md-border: #3a3a3a;
    }
    body { background: #121212; }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .md-log-entry { background: #1e1e1e; }
    .md-anno { background: #1a2233; color: #9fc4ff; }
    .md-hint { background: #232014; color: #e6c98c; }
    .md-device-group { background: #1a1a1a; color: #8b949e; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--md-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .md-subtitle { color: var(--md-muted); font-size: 12px; margin: 0 0 12px; }
  .md-subtitle a { color: var(--md-info); }

  .md-grid {
    display: grid; gap: 12px;
    grid-template-columns: minmax(0, 1fr) minmax(0, 1fr) minmax(0, 1.2fr);
  }
  @media (max-width: 1100px) { .md-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px)  { .md-grid { grid-template-columns: 1fr; } }
  .md-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .md-panel {
    border: 1px solid var(--md-border); border-radius: 8px;
    padding: 10px 12px; background: var(--md-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .md-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px;
  }
  .md-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--md-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .md-anno {
    background: #eef4ff; border-left: 3px solid var(--md-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere;
  }
  .md-hint {
    background: #fff9e6; border-left: 3px solid var(--md-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #7d5a00;
  }

  .md-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; }
  .md-label { font-size: 11px; color: var(--md-muted); }

  input[type="number"], select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--md-bg); color: var(--md-text);
    font-family: ui-monospace, monospace;
  }
  input[type="number"] { width: 90px; }
  select { min-width: 0; flex: 1; }
  label.md-check {
    display: inline-flex; align-items: center; gap: 4px;
    font-size: 11px; color: var(--md-muted); cursor: pointer;
    user-select: none;
  }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--md-soft); color: var(--md-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.md-primary { background: #eaf4ff; border-color: var(--md-info); color: #1f4e79; }
  button.md-success { background: #e8f8ed; border-color: var(--md-ok); color: #1e8449; }
  button.md-danger  { background: #fdedec; border-color: var(--md-err); color: #922b21; }
  button.md-muted   { background: #fff3cd; border-color: var(--md-warn); color: #7d5a00; }

  .md-status {
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--md-soft); border-left: 3px solid #ccc;
    color: var(--md-muted); min-height: 22px; overflow-wrap: anywhere;
  }
  .md-status.ok   { background: #e8f8ed; border-color: var(--md-ok); color: #1e8449; }
  .md-status.err  { background: #fdedec; border-color: var(--md-err); color: #922b21; }
  .md-status.busy { background: #fff9e6; border-color: var(--md-warn); color: #7d5a00; }

  .md-preview {
    width: 100%; max-width: 480px;
    background: #000; border-radius: 6px;
    aspect-ratio: 4/3; object-fit: contain;
    display: block;
  }
  .md-preview--idle {
    background: #0d1117;
    display: flex; align-items: center; justify-content: center;
    color: #6e7681; font-family: ui-monospace, monospace;
    font-size: 12px; text-align: center; padding: 8px;
  }

  .md-device-list {
    max-height: 280px; overflow-y: auto;
    border: 1px solid var(--md-border); border-radius: 5px;
    background: var(--md-soft);
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .md-device-group {
    padding: 4px 10px; font-weight: 700; font-size: 10px;
    background: #eef4ff; color: #1f4e79;
    border-top: 1px solid rgba(128,128,128,0.15);
    border-bottom: 1px solid rgba(128,128,128,0.15);
    letter-spacing: 0.5px;
  }
  .md-device-group:first-child { border-top: none; }
  @media (prefers-color-scheme: dark) {
    .md-device-group { background: #23283a; color: #9fc4ff; }
  }
  .md-device-item {
    padding: 6px 10px; border-bottom: 1px solid rgba(128,128,128,0.1);
    cursor: pointer;
    display: flex; gap: 8px; align-items: center;
  }
  .md-device-item:hover { background: rgba(128,128,128,0.08); }
  .md-device-item.is-active { background: #e8f8ed; }
  .md-device-item.is-disabled { cursor: default; opacity: 0.6; }
  .md-device-item.is-disabled:hover { background: transparent; }
  .md-device-kind {
    display: inline-block; padding: 1px 6px; border-radius: 6px;
    background: #eef4ff; color: #1f4e79;
    font-weight: 700; font-size: 10px; flex-shrink: 0;
  }
  .md-device-label { flex: 1; overflow: hidden;
    text-overflow: ellipsis; white-space: nowrap; }
  .md-device-label.muted { color: #999; font-style: italic; }
  .md-device-grp {
    font-size: 9px; color: #999; margin-left: 4px;
  }

  .md-settings-grid {
    display: grid; grid-template-columns: auto 1fr;
    gap: 4px 12px; margin: 6px 0;
    font-family: ui-monospace, monospace; font-size: 12px;
  }
  .md-settings-grid .k { color: var(--md-muted); }
  .md-settings-grid .v { color: var(--md-text); }
  .md-settings-grid .v.ok  { color: #1e8449; }
  .md-settings-grid .v.err { color: #922b21; }
  .md-settings-grid .v.warn { color: #7d5a00; }

  .md-log {
    border: 1px solid var(--md-border); border-radius: 5px;
    background: var(--md-soft); max-height: 260px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .md-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .md-log-entry {
    display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent;
  }
  .md-log-entry.ok   { border-left-color: var(--md-ok); }
  .md-log-entry.err  { border-left-color: var(--md-err); }
  .md-log-entry.info { border-left-color: var(--md-info); }
  .md-log-entry.warn { border-left-color: var(--md-warn); }
  .md-log-entry .t { color: var(--md-muted); flex-shrink: 0; }
  .md-log-entry .m { flex: 1; overflow-wrap: anywhere; }
  .md-log-entry.err .m  { color: #922b21; }
  .md-log-entry.warn .m { color: #7d5a00; }

  .md-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
  }
  @media (prefers-color-scheme: dark) {
    .md-badge { background: #2a3a4a; color: #cfe0ff; }
  }
  .md-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .md-badge.err { background: #fdedec; color: #922b21; }
  .md-badge.warn { background: #fff9e6; color: #7d5a00; }

  .md-diag-panel { margin-top: 12px; }
  .md-diag-out {
    background: var(--md-soft); border: 1px solid var(--md-border);
    border-radius: 5px; padding: 8px 12px; margin-top: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    max-height: 280px; overflow-y: auto;
  }
  .md-diag-row {
    display: grid; grid-template-columns: minmax(200px, 42%) 1fr;
    gap: 8px; padding: 2px 0;
    border-bottom: 1px dashed rgba(128,128,128,0.15);
  }
  .md-diag-row .k { color: var(--md-muted); overflow-wrap: anywhere; }
  .md-diag-row .v { color: var(--md-text); overflow-wrap: anywhere; }
</style>

<h1>Глава 35: MediaDevices</h1>
<p class="md-subtitle">
  enumerateDevices, getUserMedia, constraints, MediaStreamTrack lifecycle.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="md-grid">

  <!-- ============ Колонка 1: Управление ============ -->
  <div class="md-col">
    <div class="md-panel">
      <h2 class="md-panel-title">
        <span class="md-num">1</span>Управление
      </h2>
      <div class="md-anno">
        Камера и микрофон требуют <b>secure context</b>. Разрешение
        запрашивается при первом <code>getUserMedia</code>. До разрешения
        <code>enumerateDevices</code> возвращает <b>пустые labels</b> —
        это не баг, а защита от fingerprinting.
      </div>

      <div class="md-row">
        <label class="md-check">
          <input type="checkbox" id="md-video" checked> видео
        </label>
        <label class="md-check">
          <input type="checkbox" id="md-audio"> аудио
        </label>
      </div>

      <div class="md-row">
        <span class="md-label">width:</span>
        <input type="number" id="md-width" value="1280" min="320" max="4096" step="160">
        <span class="md-label">height:</span>
        <input type="number" id="md-height" value="720" min="240" max="2160" step="120">
      </div>

      <div class="md-row">
        <span class="md-label">fps:</span>
        <input type="number" id="md-fps" value="30" min="1" max="60" step="1">
        <span class="md-label">facing:</span>
        <select id="md-facing">
          <option value="any">любое</option>
          <option value="user">фронтальная</option>
          <option value="environment">основная</option>
        </select>
      </div>

      <div class="md-row">
        <button id="md-start" class="md-success">▶ Начать</button>
        <button id="md-stop" class="md-danger" disabled>■ Стоп</button>
        <button id="md-mute-video">🔇 Видео</button>
        <button id="md-mute-audio" disabled>🔇 Аудио</button>
      </div>

      <div class="md-row">
        <button id="md-enumerate" class="md-primary">↻ Обновить список</button>
        <span class="md-label">permissions:</span>
        <span class="md-badge" id="md-perm-camera">camera: ?</span>
        <span class="md-badge" id="md-perm-mic">mic: ?</span>
      </div>

      <div id="md-status" class="md-status">готово</div>
    </div>

    <div class="md-panel">
      <h2 class="md-panel-title">Track state</h2>
      <div id="md-track-state" class="md-settings-grid">
        <span class="k">—</span>
        <span class="v">нет активного трека</span>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 2: Устройства ============ -->
  <div class="md-col">
    <div class="md-panel">
      <h2 class="md-panel-title">Устройства</h2>
      <div id="md-devices-hint" class="md-hint">
        Загружаем список устройств…
      </div>
      <div id="md-devices" class="md-device-list">
        <div class="md-log-empty">нажмите «Обновить список»</div>
      </div>
    </div>

    <div class="md-panel">
      <h2 class="md-panel-title">Журнал</h2>
      <div id="md-log" class="md-log">
        <div class="md-log-empty">здесь появятся события</div>
      </div>
    </div>
  </div>

  <!-- ============ Колонка 3: Превью и настройки ============ -->
  <div class="md-col">
    <div class="md-panel">
      <h2 class="md-panel-title">Превью</h2>
      <video id="md-preview" class="md-preview" autoplay muted playsinline></video>
      <div id="md-preview-idle" class="md-preview md-preview--idle">
        превью появится после «Начать»
      </div>
    </div>

    <div class="md-panel">
      <h2 class="md-panel-title">Фактические настройки (getSettings)</h2>
      <div id="md-settings" class="md-settings-grid">
        <span class="k">—</span>
        <span class="v">нет трека</span>
      </div>
    </div>
  </div>

</div>

<!-- ============ Диагностика (на всю ширину) ============ -->
<div class="md-panel md-diag-panel">
  <h2 class="md-panel-title">🔍 Диагностика окружения</h2>
  <div class="md-anno">
    Если превью не появляется или список устройств пуст — соберите
    снимок окружения. Типичные причины: не secure context, нет
    разрешения, устройство занято другим приложением.
  </div>
  <div class="md-row">
    <button id="md-diag-collect" class="md-primary">🔍 Собрать диагностику</button>
  </div>
  <div id="md-diag-out" class="md-diag-out">
    <div class="md-log-empty">нажмите «Собрать диагностику»</div>
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

struct Ui {
    video: web_sys::HtmlInputElement,
    audio: web_sys::HtmlInputElement,
    width: web_sys::HtmlInputElement,
    height: web_sys::HtmlInputElement,
    fps: web_sys::HtmlInputElement,
    facing: HtmlSelectElement,

    btn_start: HtmlElement,
    btn_stop: HtmlElement,
    btn_mute_video: HtmlElement,
    btn_mute_audio: HtmlElement,
    btn_enumerate: HtmlElement,
    btn_diag_collect: HtmlElement,

    status: web_sys::Element,
    devices_hint: web_sys::Element,
    devices: web_sys::Element,
    preview: HtmlVideoElement,
    preview_idle: web_sys::Element,
    settings: web_sys::Element,
    track_state: web_sys::Element,
    log: web_sys::Element,

    perm_camera: web_sys::Element,
    perm_mic: web_sys::Element,
    diag_out: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    devices: Rc<RefCell<Vec<DeviceInfo>>>,
    stream: Rc<RefCell<Option<MediaStream>>>,
    /// Closures onended / onmute, привязанные к трекам. Храним, чтобы
    /// не собрались GC.
    track_closures: Rc<RefCell<Vec<Closure<dyn FnMut()>>>>,
    ui: Ui,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        video: get_input("md-video")?,
        audio: get_input("md-audio")?,
        width: get_input("md-width")?,
        height: get_input("md-height")?,
        fps: get_input("md-fps")?,
        facing: get_el("md-facing")?.dyn_into()?,

        btn_start: get_el("md-start")?.dyn_into()?,
        btn_stop: get_el("md-stop")?.dyn_into()?,
        btn_mute_video: get_el("md-mute-video")?.dyn_into()?,
        btn_mute_audio: get_el("md-mute-audio")?.dyn_into()?,
        btn_enumerate: get_el("md-enumerate")?.dyn_into()?,
        btn_diag_collect: get_el("md-diag-collect")?.dyn_into()?,

        status: get_el("md-status")?,
        devices_hint: get_el("md-devices-hint")?,
        devices: get_el("md-devices")?,
        preview: get_el("md-preview")?.dyn_into()?,
        preview_idle: get_el("md-preview-idle")?,
        settings: get_el("md-settings")?,
        track_state: get_el("md-track-state")?,
        log: get_el("md-log")?,

        perm_camera: get_el("md-perm-camera")?,
        perm_mic: get_el("md-perm-mic")?,
        diag_out: get_el("md-diag-out")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        devices: Rc::new(RefCell::new(Vec::new())),
        stream: Rc::new(RefCell::new(None)),
        track_closures: Rc::new(RefCell::new(Vec::new())),
        ui,
    });

    app.render_log();

    if !media_devices::has_media_devices() {
        app.push_log(
            LogKind::Err,
            "MediaDevices API недоступен — нужен secure context".to_string(),
        );
        set_status(&app.ui.status, "err", "MediaDevices API недоступен");
        let _ = app.ui.btn_start.set_attribute("disabled", "");
        let _ = app.ui.btn_enumerate.set_attribute("disabled", "");
        return Ok(());
    }
    app.push_log(LogKind::Info, "MediaDevices API доступен".to_string());

    // Начальный enumerate — до разрешения, labels пусты.
    enumerate_and_render(app.clone());
    refresh_permissions_badge(app.clone());

    bind_buttons(&app)?;
    bind_devicechange(app.clone())?;

    Ok(())
}

// ============================================================
//                    BIND
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_enumerate, move || {
            enumerate_and_render(app_c.clone());
            refresh_permissions_badge(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_start, move || {
            let app_in = app_c.clone();
            spawn_local(async move {
                start_stream(app_in).await;
            });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_stop, move || {
            stop_all(&app_c);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_mute_video, move || {
            toggle_track_enabled(&app_c, "video");
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_mute_audio, move || {
            toggle_track_enabled(&app_c, "audio");
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_diag_collect, move || {
            collect_diagnostics(app_c.clone());
        });
    }
    Ok(())
}

fn bind_devicechange(app: Rc<App>) -> Result<(), JsValue> {
    let cb = Closure::<dyn FnMut()>::new(move || {
        app.push_log(
            LogKind::Info,
            "devicechange: список устройств изменился".to_string(),
        );
        enumerate_and_render(app.clone());
    });
    if let Some(win) = web_sys::window() {
        if let Ok(md) = Reflect::get(
            &win.navigator(),
            &JsValue::from_str("mediaDevices"),
        ) {
            if let Ok(target) = md.dyn_into::<web_sys::EventTarget>() {
                target.add_event_listener_with_callback(
                    "devicechange",
                    cb.as_ref().unchecked_ref(),
                )?;
            }
        }
    }
    cb.forget();
    Ok(())
}

// ============================================================
//                    ENUMERATE
// ============================================================

fn enumerate_and_render(app: Rc<App>) {
    spawn_local(async move {
        match media_devices::enumerate_devices().await {
            Ok(devices) => {
                let count = devices.len();
                let has_labels = devices.iter().any(|d| d.has_label());
                *app.devices.borrow_mut() = devices;
                render_device_list(&app);
                render_devices_hint(&app, has_labels);
                app.push_log(
                    LogKind::Info,
                    format_str(format_args!(
                        "enumerateDevices → {count}, labels: {}",
                        if has_labels { "есть" } else { "нет (нет разрешения)" },
                    )),
                );
                update_start_button(&app);
            }
            Err(e) => {
                let msg = format_str(format_args!("enumerateDevices: {e:?}"));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
            }
        }
    });
}

fn render_devices_hint(app: &Rc<App>, has_labels: bool) {
    if has_labels {
        app.ui.devices_hint.set_class_name("md-hint");
        app.ui.devices_hint.set_inner_html(
            "Клик по видео-устройству — <b>hot-swap</b>: камера меняется \
             без остановки аудио. Клик по аудио-устройству — переключение \
             микрофона.",
        );
    } else {
        app.ui.devices_hint.set_class_name("md-anno");
        app.ui.devices_hint.set_inner_html(
            "<b>Разрешения нет.</b> Показаны только типы устройств, \
             без имён (labels пусты — так браузер защищает от \
             fingerprinting). Нажмите «Начать», выдайте разрешение — \
             список обновится автоматически.",
        );
    }
}

fn render_device_list(app: &Rc<App>) {
    let devices = app.devices.borrow();
    if devices.is_empty() {
        app.ui.devices.set_inner_html(
            "<div class=\"md-log-empty\">устройств не найдено</div>",
        );
        return;
    }

    let active_video_id = current_track_device_id(app, "video");
    let active_audio_id = current_track_device_id(app, "audio");

    let mut video_idx = 0u32;
    let mut audio_idx = 0u32;
    let mut other_idx = 0u32;

    let mut html = String::with_capacity(768);
    let mut last_group_kind: Option<DeviceKind> = None;
    let mut first_group = true;

    for (global_idx, d) in devices.iter().enumerate() {
        // Заголовок группы.
        if last_group_kind != Some(d.kind) {
            let (title, count) = match d.kind {
                DeviceKind::VideoInput => (
                    "КАМЕРЫ",
                    devices.iter().filter(|x| x.kind == DeviceKind::VideoInput).count(),
                ),
                DeviceKind::AudioInput => (
                    "МИКРОФОНЫ",
                    devices.iter().filter(|x| x.kind == DeviceKind::AudioInput).count(),
                ),
                DeviceKind::AudioOutput => (
                    "ВЫВОД ЗВУКА",
                    devices.iter().filter(|x| x.kind == DeviceKind::AudioOutput).count(),
                ),
                DeviceKind::Unknown => ("ДРУГИЕ", 1),
            };
            if !first_group {
                // Ничего не делаем — граница группы сама по себе.
            }
            html.push_str("<div class=\"md-device-group\">");
            html.push_str(&esc_html(title));
            html.push_str(" · ");
            html.push_str(&count.to_string());
            html.push_str("</div>");
            last_group_kind = Some(d.kind);
            first_group = false;
        }

        // Локальный индекс в рамках своего kind.
        let local_idx = match d.kind {
            DeviceKind::VideoInput => {
                let i = video_idx;
                video_idx += 1;
                i
            }
            DeviceKind::AudioInput => {
                let i = audio_idx;
                audio_idx += 1;
                i
            }
            _ => {
                let i = other_idx;
                other_idx += 1;
                i
            }
        };

        let kind_label = match d.kind {
            DeviceKind::VideoInput => "VIDEO",
            DeviceKind::AudioInput => "AUDIO",
            DeviceKind::AudioOutput => "OUT",
            DeviceKind::Unknown => "???",
        };
        let label = d.display_label(local_idx as usize);
        let label_cls = if d.has_label() {
            "md-device-label"
        } else {
            "md-device-label muted"
        };

        let is_active = match d.kind {
            DeviceKind::VideoInput => {
                !active_video_id.is_empty() && active_video_id == d.device_id
            }
            DeviceKind::AudioInput => {
                !active_audio_id.is_empty() && active_audio_id == d.device_id
            }
            _ => false,
        };
        let clickable = !d.device_id.is_empty()
            && matches!(
                d.kind,
                DeviceKind::VideoInput | DeviceKind::AudioInput
            );

        let item_cls = if is_active {
            "md-device-item is-active"
        } else if clickable {
            "md-device-item"
        } else {
            "md-device-item is-disabled"
        };

        html.push_str("<div class=\"");
        html.push_str(item_cls);
        html.push_str("\" data-idx=\"");
        html.push_str(&global_idx.to_string());
        html.push_str("\" data-kind=\"");
        html.push_str(d.kind.as_str());
        html.push_str("\">");
        html.push_str("<span class=\"md-device-kind\">");
        html.push_str(kind_label);
        html.push_str("</span>");
        html.push_str("<span class=\"");
        html.push_str(label_cls);
        html.push_str("\" title=\"");
        html.push_str(&esc_html(&d.device_id));
        html.push_str("\">");
        html.push_str(&esc_html(&label));
        html.push_str("</span>");
        if !d.group_id.is_empty() {
            let short = short_id(&d.group_id);
            html.push_str("<span class=\"md-device-grp\" title=\"groupId\">");
            html.push_str(&esc_html(&short));
            html.push_str("</span>");
        }
        html.push_str("</div>");
    }

    app.ui.devices.set_inner_html(&html);

    // Обработчики кликов.
    if let Ok(items) = app.ui.devices.query_selector_all(".md-device-item") {
        for i in 0..items.length() {
            let Some(node) = items.get(i) else { continue };
            let Some(el) = node.dyn_into::<HtmlElement>().ok() else { continue };

            // el переместится в замыкание — предварительно клонируем
            // для последующего set_onclick.
            let el_for_closure = el.clone();
            let app_c = app.clone();
            let click = Closure::<dyn FnMut()>::new(move || {
                let idx_str = el_for_closure
                    .get_attribute("data-idx")
                    .unwrap_or_default();
                let kind = el_for_closure
                    .get_attribute("data-kind")
                    .unwrap_or_default();
                if let Ok(idx) = idx_str.parse::<usize>() {
                    switch_device(app_c.clone(), idx, &kind);
                }
            });
            el.set_onclick(Some(click.as_ref().unchecked_ref()));
            click.forget();
        }
    }
}

fn short_id(s: &str) -> String {
    if s.len() > 8 {
        return format_str(format_args!("{}…", &s[..6]));
    }
    return s.to_string();
}

// ============================================================
//                    PERMISSIONS
// ============================================================

fn refresh_permissions_badge(app: Rc<App>) {
    let el_cam = app.ui.perm_camera.clone();
    let el_mic = app.ui.perm_mic.clone();
    let log = app.log.clone();
    spawn_local(async move {
        let cam = query_permission("camera").await;
        let mic = query_permission("microphone").await;
        apply_permission_badge(&el_cam, "camera", cam.as_deref());
        apply_permission_badge(&el_mic, "mic", mic.as_deref());
        push_log_static(
            &log,
            LogKind::Info,
            format_str(format_args!(
                "permissions.query: camera={}, microphone={}",
                cam.unwrap_or_else(|| "n/a".into()),
                mic.unwrap_or_else(|| "n/a".into()),
            )),
        );
    });
}

fn apply_permission_badge(el: &web_sys::Element, name: &str, state: Option<&str>) {
    let (text, cls) = match state {
        Some("granted") => (
            format_str(format_args!("{name}: granted")),
            "md-badge ok",
        ),
        Some("denied") => (
            format_str(format_args!("{name}: denied")),
            "md-badge err",
        ),
        Some("prompt") => (
            format_str(format_args!("{name}: prompt")),
            "md-badge warn",
        ),
        _ => (
            format_str(format_args!("{name}: n/a")),
            "md-badge",
        ),
    };
    el.set_text_content(Some(&text));
    el.set_class_name(cls);
}

async fn query_permission(name: &str) -> Option<String> {
    use wasm_bindgen_futures::JsFuture;

    let global = js_sys::global();
    let nav = Reflect::get(&global, &JsValue::from_str("navigator")).ok()?;
    let permissions = Reflect::get(&nav, &JsValue::from_str("permissions")).ok()?;
    if permissions.is_undefined() || permissions.is_null() {
        return None;
    }
    let query: js_sys::Function =
        Reflect::get(&permissions, &JsValue::from_str("query"))
            .ok()?
            .dyn_into()
            .ok()?;
    let desc = js_sys::Object::new();
    Reflect::set(
        &desc,
        &JsValue::from_str("name"),
        &JsValue::from_str(name),
    )
    .ok()?;
    let promise = query.call1(&permissions, &desc).ok()?;
    let promise: js_sys::Promise = promise.dyn_into().ok()?;
    let result = JsFuture::from(promise).await.ok()?;
    return Reflect::get(&result, &JsValue::from_str("state"))
        .ok()?
        .as_string();
}

// ============================================================
//                    HELPERS
// ============================================================

fn is_stream_active(app: &Rc<App>) -> bool {
    return app.stream.borrow().is_some();
}

fn current_track_device_id(app: &Rc<App>, kind: &str) -> String {
    let stream_opt = app.stream.borrow();
    let Some(stream) = stream_opt.as_ref() else {
        return String::new();
    };
    let tracks = if kind == "video" {
        stream.get_video_tracks()
    } else {
        stream.get_audio_tracks()
    };
    if tracks.length() == 0 {
        return String::new();
    }
    let Some(t) = tracks.get(0).dyn_into::<MediaStreamTrack>().ok() else {
        return String::new();
    };
    return media_devices::get_settings(&t).device_id.unwrap_or_default();
}

fn read_constraints(app: &Rc<App>) -> StreamConstraints {
    let video = app.ui.video.checked();
    let audio = app.ui.audio.checked();
    let width = app.ui.width.value().parse::<u32>().unwrap_or(1280);
    let height = app.ui.height.value().parse::<u32>().unwrap_or(720);
    let fps = app.ui.fps.value().parse::<u32>().unwrap_or(30);
    let facing = match app.ui.facing.value().as_str() {
        "user" => FacingMode::User,
        "environment" => FacingMode::Environment,
        _ => FacingMode::Any,
    };
    return StreamConstraints {
        video,
        audio,
        width,
        height,
        frame_rate: fps,
        facing,
        video_device_id: String::new(),
        audio_device_id: String::new(),
        echo_cancellation: true,
    };
}

/// Читает состояние UI и синхронизирует доступность кнопки «Начать».
///
/// Принимает `&Rc<App>`, потому что вызывается из:
///   • `enumerate_and_render` (там `app: Rc<App>`, передаётся `&app`);
///   • `update_controls_state` (там `app: &Rc<App>`, передаётся `app`);
///   • `update_start_button` вызывается после каждой смены фазы,
///     поэтому borrow вместо clone экономит счётчик Rc.
fn update_start_button(app: &Rc<App>) {
    let has_video = app.ui.video.checked();
    let has_audio = app.ui.audio.checked();
    let can_start = (has_video || has_audio) && !is_stream_active(app);
    if can_start {
        let _ = app.ui.btn_start.remove_attribute("disabled");
    } else {
        let _ = app.ui.btn_start.set_attribute("disabled", "");
    }
}

// ============================================================
//                    START / STOP
// ============================================================

async fn start_stream(app: Rc<App>) {
    if is_stream_active(&app) {
        set_status(&app.ui.status, "busy", "поток уже запущен");
        return;
    }
    let constraints = read_constraints(&app);
    if !constraints.video && !constraints.audio {
        set_status(&app.ui.status, "err", "выберите видео или аудио");
        return;
    }

    set_status(&app.ui.status, "busy", "запрос доступа…");
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "getUserMedia(video={}, audio={}, {}×{}@{})",
            constraints.video, constraints.audio,
            constraints.width, constraints.height, constraints.frame_rate,
        )),
    );

    match media_devices::request_stream(&constraints).await {
        Ok(stream) => {
            attach_preview(&app, &stream);
            attach_track_listeners(&app, &stream);
            *app.stream.borrow_mut() = Some(stream);
            app.push_log(
                LogKind::Ok,
                "поток получен — разрешение выдано".to_string(),
            );
            set_status(&app.ui.status, "ok", "работает");
            update_controls_state(&app);
            render_settings(&app);
            render_track_state(&app);
            enumerate_and_render(app.clone());
            refresh_permissions_badge(app.clone());
        }
        Err(e) => {
            let msg = media_devices::format_media_error(&e);
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

fn attach_preview(app: &Rc<App>, stream: &MediaStream) {
    let _ = Reflect::set(
        &app.ui.preview,
        &JsValue::from_str("srcObject"),
        stream.as_ref(),
    );
    // srcObject не перезапускает playback автоматически.
    let play = Reflect::get(
        &app.ui.preview,
        &JsValue::from_str("play"),
    )
    .ok()
    .and_then(|v| v.dyn_into::<js_sys::Function>().ok());
    if let Some(play) = play {
        let _ = play.call0(&app.ui.preview);
    }
    app.ui.preview_idle.set_attribute("style", "display:none").ok();
    app.ui.preview.remove_attribute("style").ok();
}

fn detach_preview(app: &Rc<App>) {
    let _ = Reflect::set(
        &app.ui.preview,
        &JsValue::from_str("srcObject"),
        &JsValue::NULL,
    );
    app.ui.preview.set_attribute("style", "display:none").ok();
    app.ui.preview_idle.remove_attribute("style").ok();
    app.ui.preview_idle.set_inner_html(
        "превью появится после «Начать»",
    );
}

/// Навешивает onended на все треки. Closures сохраняются в App.
fn attach_track_listeners(app: &Rc<App>, stream: &MediaStream) {
    let tracks = stream.get_tracks();
    for i in 0..tracks.length() {
        let Some(track) = tracks.get(i).dyn_into::<MediaStreamTrack>().ok() else {
            continue;
        };
        let kind = track.kind();
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            app_c.push_log(
                LogKind::Warn,
                format_str(format_args!(
                    "track {kind} завершён: устройство отключено или занято",
                )),
            );
            render_track_state(&app_c);
            set_status(
                &app_c.ui.status,
                "err",
                "трек завершён — перезапустите поток",
            );
        });
        track.set_onended(Some(cb.as_ref().unchecked_ref()));
        app.track_closures.borrow_mut().push(cb);
    }
}

fn stop_all(app: &Rc<App>) {
    app.track_closures.borrow_mut().clear();
    let stream_opt = app.stream.borrow_mut().take();
    if let Some(stream) = stream_opt {
        media_devices::stop_stream(&stream);
        app.push_log(
            LogKind::Info,
            "все треки остановлены — устройства освобождены".to_string(),
        );
    }
    detach_preview(app);
    set_status(&app.ui.status, "ok", "остановлено");
    update_controls_state(app);
    clear_settings(app);
    clear_track_state(app);
    render_device_list(app);
    reset_mute_buttons(app);
}

fn reset_mute_buttons(app: &Rc<App>) {
    app.ui.btn_mute_video.set_text_content(Some("🔇 Видео"));
    app.ui.btn_mute_video.set_class_name("");
    app.ui.btn_mute_audio.set_text_content(Some("🔇 Аудио"));
    app.ui.btn_mute_audio.set_class_name("");
}

// ============================================================
//                    MUTE
// ============================================================

fn toggle_track_enabled(app: &Rc<App>, kind: &str) {
    let stream_opt = app.stream.borrow();
    let Some(stream) = stream_opt.as_ref() else {
        return;
    };
    let tracks = if kind == "video" {
        stream.get_video_tracks()
    } else {
        stream.get_audio_tracks()
    };
    if tracks.length() == 0 {
        return;
    }
    let Some(track) = tracks.get(0).dyn_into::<MediaStreamTrack>().ok() else {
        return;
    };
    let current = track.enabled();
    let new_state = !current;
    track.set_enabled(new_state);
    drop(stream_opt);

    // Кнопка отражает текущее состояние: «🔇 Видео» если выключен,
    // «🔊 Видео» если включён.
    let (btn, base) = if kind == "video" {
        (&app.ui.btn_mute_video, "Видео")
    } else {
        (&app.ui.btn_mute_audio, "Аудио")
    };
    if new_state {
        let label = format_str(format_args!("🔇 {base}"));
        btn.set_text_content(Some(&label));
        btn.set_class_name("");
    } else {
        let label = format_str(format_args!("🔊 {base}"));
        btn.set_text_content(Some(&label));
        btn.set_class_name("md-muted");
    }

    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "{kind} track: enabled = {new_state}",
        )),
    );
    render_track_state(app);
}

// ============================================================
//                    SWITCH DEVICE
// ============================================================

fn switch_device(app: Rc<App>, idx: usize, kind: &str) {
    let devices = app.devices.borrow();
    let Some(device) = devices.get(idx).cloned() else {
        return;
    };
    drop(devices);

    // Проверяем, что тип совпадает с data-kind.
    if device.kind.as_str() != kind {
        return;
    }

    match device.kind {
        DeviceKind::VideoInput => {
            if device.device_id.is_empty() {
                set_status(
                    &app.ui.status,
                    "err",
                    "нет разрешения — сначала запустите поток",
                );
                return;
            }
            if !is_stream_active(&app) {
                set_status(
                    &app.ui.status,
                    "busy",
                    "поток не запущен — используйте «Начать»",
                );
                return;
            }
            let active_id = current_track_device_id(&app, "video");
            if active_id == device.device_id {
                set_status(
                    &app.ui.status,
                    "busy",
                    "эта камера уже активна",
                );
                return;
            }
            hot_swap_track(app.clone(), device, "video");
        }
        DeviceKind::AudioInput => {
            if device.device_id.is_empty() {
                set_status(
                    &app.ui.status,
                    "err",
                    "нет разрешения — сначала запустите поток",
                );
                return;
            }
            if !is_stream_active(&app) {
                set_status(
                    &app.ui.status,
                    "busy",
                    "поток не запущен — используйте «Начать»",
                );
                return;
            }
            let active_id = current_track_device_id(&app, "audio");
            if active_id == device.device_id {
                set_status(
                    &app.ui.status,
                    "busy",
                    "этот микрофон уже активен",
                );
                return;
            }
            hot_swap_track(app.clone(), device, "audio");
        }
        _ => {}
    }
}

fn hot_swap_track(app: Rc<App>, device: DeviceInfo, kind: &str) {
    // Замыкание для spawn_local требует 'static. Превращаем
    // заимствованный &str в owned String до входа в async-блок.
    let kind_owned = kind.to_string();

    let label = if device.has_label() {
        device.label.clone()
    } else {
        kind_owned.clone()
    };
    app.push_log(
        LogKind::Info,
        format_str(format_args!("hot-swap {kind_owned} → {label}")),
    );
    set_status(&app.ui.status, "busy", "переключение устройства…");

    let _ = app.ui.btn_start.set_attribute("disabled", "");

    spawn_local(async move {
        let is_video = kind_owned == "video";

        let mut c = read_constraints(&app);
        if is_video {
            c.video = true;
            c.audio = false;
            c.video_device_id = device.device_id.clone();
        } else {
            c.video = false;
            c.audio = true;
            c.audio_device_id = device.device_id.clone();
        }

        match media_devices::request_stream(&c).await {
            Ok(new_stream) => {
                let new_track = get_first_track(&new_stream, &kind_owned);
                let Some(new_track) = new_track else {
                    set_status(&app.ui.status, "err", "новый поток без трека");
                    update_controls_state(&app);
                    return;
                };

                let old_stream = app.stream.borrow().clone();
                let Some(old_stream) = old_stream else {
                    return;
                };
                let old_tracks = if is_video {
                    old_stream.get_video_tracks()
                } else {
                    old_stream.get_audio_tracks()
                };
                if old_tracks.length() > 0 {
                    if let Ok(old_t) =
                        old_tracks.get(0).dyn_into::<MediaStreamTrack>()
                    {
                        let _ = old_stream.remove_track(&old_t);
                        old_t.stop();
                    }
                }
                let _ = old_stream.add_track(&new_track);

                attach_preview(&app, &old_stream);
                app.track_closures.borrow_mut().clear();
                attach_track_listeners(&app, &old_stream);

                app.push_log(
                    LogKind::Ok,
                    format_str(format_args!(
                        "{kind_owned} переключён → {label}",
                    )),
                );
                set_status(&app.ui.status, "ok", "устройство переключено");
                render_settings(&app);
                render_track_state(&app);
                render_device_list(&app);
                update_controls_state(&app);
            }
            Err(e) => {
                let msg = media_devices::format_media_error(&e);
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.status, "err", &msg);
                update_controls_state(&app);
            }
        }
    });
}

fn get_first_track(stream: &MediaStream, kind: &str) -> Option<MediaStreamTrack> {
    let tracks = if kind == "video" {
        stream.get_video_tracks()
    } else {
        stream.get_audio_tracks()
    };
    if tracks.length() == 0 {
        return None;
    }
    return tracks.get(0).dyn_into::<MediaStreamTrack>().ok();
}

// ============================================================
//                    RENDER
// ============================================================

fn render_settings(app: &Rc<App>) {
    let stream_opt = app.stream.borrow();
    let Some(stream) = stream_opt.as_ref() else {
        clear_settings(app);
        return;
    };
    let tracks = stream.get_video_tracks();
    if tracks.length() == 0 {
        // Если только аудио — покажем хотя бы устройство.
        let atracks = stream.get_audio_tracks();
        if atracks.length() > 0 {
            if let Ok(t) = atracks.get(0).dyn_into::<MediaStreamTrack>() {
                let s = media_devices::get_settings(&t);
                let mut html = String::with_capacity(128);
                if let Some(id) = s.device_id {
                    push_row(&mut html, "audio deviceId", &short_id(&id));
                }
                if html.is_empty() {
                    clear_settings(app);
                } else {
                    app.ui.settings.set_inner_html(&html);
                }
                return;
            }
        }
        clear_settings(app);
        return;
    }
    let Some(track) = tracks.get(0).dyn_into::<MediaStreamTrack>().ok() else {
        clear_settings(app);
        return;
    };
    let s = media_devices::get_settings(&track);

    let mut html = String::with_capacity(256);
    if let Some(w) = s.width {
        push_row(&mut html, "width", &format_str(format_args!("{w} px")));
    }
    if let Some(h) = s.height {
        push_row(&mut html, "height", &format_str(format_args!("{h} px")));
    }
    if let Some(fps) = s.frame_rate {
        push_row(&mut html, "frameRate", &format_str(format_args!("{fps:.1} fps")));
    }
    if let Some(id) = s.device_id {
        push_row(&mut html, "deviceId", &short_id(&id));
    }
    if let Some(fm) = s.facing_mode {
        push_row(&mut html, "facingMode", &fm);
    }
    if html.is_empty() {
        html.push_str("<span class=\"k\">—</span><span class=\"v\">нет данных</span>");
    }
    app.ui.settings.set_inner_html(&html);
}

fn clear_settings(app: &Rc<App>) {
    app.ui.settings.set_inner_html(
        "<span class=\"k\">—</span><span class=\"v\">нет трека</span>",
    );
}

fn render_track_state(app: &Rc<App>) {
    let stream_opt = app.stream.borrow();
    let Some(stream) = stream_opt.as_ref() else {
        clear_track_state(app);
        return;
    };

    let mut html = String::with_capacity(512);
    for (label, tracks) in [
        ("video", stream.get_video_tracks()),
        ("audio", stream.get_audio_tracks()),
    ] {
        if tracks.length() == 0 {
            continue;
        }
        let Some(t) = tracks.get(0).dyn_into::<MediaStreamTrack>().ok() else {
            continue;
        };
        let st: TrackState = media_devices::read_track_state(&t);

        // Строка заголовка: label трека или «—».
        let title = if st.label.is_empty() {
            "—".to_string()
        } else if st.label.len() > 24 {
            format_str(format_args!("{}…", &st.label[..24]))
        } else {
            st.label.clone()
        };
        push_row(&mut html, &format_str(format_args!("[{label}]")), &title);

        // enabled
        let (enabled_txt, enabled_cls) = if st.enabled {
            ("true", "v ok")
        } else {
            ("false", "v warn")
        };
        push_row_cls(
            &mut html,
            &format_str(format_args!("{label}.enabled")),
            enabled_txt,
            enabled_cls,
        );

        // muted
        let (muted_txt, muted_cls) = if st.muted {
            ("true", "v warn")
        } else {
            ("false", "v")
        };
        push_row_cls(
            &mut html,
            &format_str(format_args!("{label}.muted")),
            muted_txt,
            muted_cls,
        );

        // readyState
        let ready_cls = if st.ready_state == "live" { "v ok" } else { "v err" };
        push_row_cls(
            &mut html,
            &format_str(format_args!("{label}.readyState")),
            &st.ready_state,
            ready_cls,
        );
    }
    if html.is_empty() {
        clear_track_state(app);
        return;
    }
    app.ui.track_state.set_inner_html(&html);
}

fn clear_track_state(app: &Rc<App>) {
    app.ui.track_state.set_inner_html(
        "<span class=\"k\">—</span><span class=\"v\">нет активного трека</span>",
    );
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

fn update_controls_state(app: &Rc<App>) {
    let active = is_stream_active(app);
    if active {
        let _ = app.ui.btn_start.set_attribute("disabled", "");
        let _ = app.ui.btn_stop.remove_attribute("disabled");
        let _ = app.ui.btn_mute_video.remove_attribute("disabled");
    } else {
        let _ = app.ui.btn_start.remove_attribute("disabled");
        let _ = app.ui.btn_stop.set_attribute("disabled", "");
        let _ = app.ui.btn_mute_video.set_attribute("disabled", "");
    }

    let has_audio = app
        .stream
        .borrow()
        .as_ref()
        .map(|s| s.get_audio_tracks().length() > 0)
        .unwrap_or(false);
    if has_audio {
        let _ = app.ui.btn_mute_audio.remove_attribute("disabled");
    } else {
        let _ = app.ui.btn_mute_audio.set_attribute("disabled", "");
    }

    update_start_button(app);
}

// ============================================================
//                    DIAGNOSTICS
// ============================================================

fn collect_diagnostics(app: Rc<App>) {
    app.ui.diag_out.set_inner_html(
        "<div class=\"md-log-empty\">сбор…</div>",
    );
    app.push_log(LogKind::Info, "сбор диагностики…".to_string());
    spawn_local(async move {
        let mut rows: Vec<(String, String)> = Vec::new();
        let global = js_sys::global();

        let is_secure = Reflect::get(
            &global,
            &JsValue::from_str("isSecureContext"),
        )
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
        rows.push(("isSecureContext".into(), yes_no(is_secure)));
        rows.push((
            "MediaDevices API".into(),
            yes_no(media_devices::has_media_devices()),
        ));
        rows.push((
            "getUserMedia".into(),
            yes_no(media_devices::has_get_user_media()),
        ));
        rows.push((
            "enumerateDevices".into(),
            yes_no(media_devices::has_enumerate_devices()),
        ));

        let cam = query_permission("camera").await.unwrap_or_else(|| "n/a".into());
        let mic = query_permission("microphone").await.unwrap_or_else(|| "n/a".into());
        rows.push(("permissions.camera".into(), cam));
        rows.push(("permissions.microphone".into(), mic));

        rows.push((
            "device count".into(),
            app.devices.borrow().len().to_string(),
        ));

        if let Ok(nav) = Reflect::get(&global, &JsValue::from_str("navigator")) {
            if let Ok(ua) = Reflect::get(&nav, &JsValue::from_str("userAgent")) {
                if let Some(s) = ua.as_string() {
                    rows.push(("userAgent".into(), s));
                }
            }
        }

        let mut html = String::with_capacity(1024);
        for (k, v) in rows.iter() {
            html.push_str("<div class=\"md-diag-row\"><span class=\"k\">");
            html.push_str(&esc_html(k));
            html.push_str("</span><span class=\"v\">");
            html.push_str(&esc_html(v));
            html.push_str("</span></div>");
        }
        app.ui.diag_out.set_inner_html(&html);
        app.push_log(
            LogKind::Ok,
            format_str(format_args!("диагностика: {} строк", rows.len())),
        );
    });
}

fn yes_no(b: bool) -> String {
    return if b { "yes".into() } else { "no".into() };
}

// ============================================================
//                    UI HELPERS
// ============================================================

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "md-status ok",
        "err" => "md-status err",
        "busy" => "md-status busy",
        _ => "md-status",
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
                "<div class=\"md-log-empty\">здесь появятся события</div>",
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
            html.push_str("<div class=\"md-log-entry ");
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

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

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

fn get_input(id: &str) -> Result<web_sys::HtmlInputElement, JsValue> {
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
