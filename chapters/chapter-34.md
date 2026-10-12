# Глава 34. Geolocation

Геолокация — редкий случай в Web Platform, когда API обращается к **аппаратному обеспечению** пользователя и требует **явного согласия** на доступ к приватным данным. От Notifications он отличается длиной цепочки: приложение → браузер → ОС → GPS/сеть → провайдер. На каждом шаге что-то может отказать, и API рапортует об ошибке **одним из трёх кодов**, за которыми скрываются совершенно разные проблемы.

В этой главе мы построим практический location-компонент: от запроса одной позиции до непрерывного трекинга с фильтрацией GPS-шума. И разберём главную ловушку, в которую попадает каждый, кто впервые работает с геолокацией: **callback приходит регулярно, но это ещё не значит, что координаты меняются**.

## 34.1. Три метода, одна задача

`navigator.geolocation` предоставляет три метода, и путать их назначение — источник большинства ошибок:

```text
┌─────────────────────────────────────────────────────────────────┐
│  navigator.geolocation                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  getCurrentPosition(success, error?, options?)                  │
│    Одноразовый запрос. Возвращает void, не Promise.             │
│    Результат — только через callbacks.                          │
│                                                                 │
│  watchPosition(success, error?, options?)                       │
│    Непрерывное отслеживание. Возвращает watch_id: i32.          │
│    Клик по «Стоп» → clearWatch(watch_id).                       │
│                                                                 │
│  clearWatch(watch_id)                                           │
│    Остановка. Идемпотентна: повторный вызов — не ошибка.        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

`getCurrentPosition` не возвращает `Promise` — это **callback-based API**, спроектированный до эры промисов. На Rust/web-sys работаем через `Closure` + Reflect (об этом ниже).

## 34.2. PositionOptions — три параметра, которые меняют всё

Без `PositionOptions` API ведёт себя как чёрный ящик: timeout по умолчанию **бесконечный**, точность «как получится», кеш не определён. Управление лежит в трёх полях:

```rust
pub struct GeoOptions {
    pub high_accuracy: bool,  // GPS вместо Wi-Fi/IP-геолокации
    pub timeout_ms: u32,      // 0 = бесконечно (почти всегда ошибка)
    pub max_age_ms: u32,      // допустимый возраст кеша
}
```

**`enableHighAccuracy`** переключает источник данных. `false` — быстрый ответ по IP/Wi-Fi, точность ~1000 м. `true` — GPS, точность ~10 м, но время получения — секунды, а на десктопе без GPS-чипа может не работать вовсе.

**`timeout`** — критичный параметр. `Infinity` по умолчанию означает, что при недоступности GPS колбэк может не вызваться **никогда**. Правило: всегда ставьте явный timeout. На десктопе практичное значение — **12000 мс**, не меньше. Меньшие значения дают `POSITION_UNAVAILABLE` на первой попытке, потому что Wi-Fi-геолокации нужно время получить свежий набор точек.

**`maximumAge`** — окно кеша. Если ОС получила позицию 30 секунд назад, а `maximumAge = 60000`, браузер вернёт её мгновенно, не запуская GPS. Для карт это нормально, для навигации — опасно: покажете устаревшую точку.

```rust
impl GeoOptions {
    pub fn to_web_sys(&self) -> Result<web_sys::PositionOptions, JsValue> {
        let o = web_sys::PositionOptions::new();
        o.set_enable_high_accuracy(self.high_accuracy);
        o.set_timeout(self.timeout_ms);
        o.set_maximum_age(self.max_age_ms);
        return Ok(o);
    }
}
```

**PositionOptions — это dictionary, а не interface.** web-sys генерирует только сеттеры, геттеров не существует. Проверить значения обратно нельзя — валидирует их браузер при вызове `getCurrentPosition`. Это относится ко всем options-объектам Web Platform (`RequestInit`, `ShareData` и т.д.).

## 34.3. Ошибки: три кода, два типа

`PositionError` содержит `code: u16` и `message: String`. Кодов ровно три:

| Код | Константа | Что значит | Что делать |
|---|---|---|---|
| 1 | `PERMISSION_DENIED` | Отказ пользователя или политика | Показать инструкцию, как включить |
| 2 | `POSITION_UNAVAILABLE` | Нет GPS, нет сети, нет источника | Отключить high accuracy, подождать |
| 3 | `TIMEOUT` | Не успели за `timeout` мс | Увеличить timeout или переключиться |

Из практики: **все три кода надо разделить на два класса** — восстанавливаемые и фатальные:

```rust
impl GeoErrorKind {
    pub fn from_code(code: u16) -> Self {
        return match code {
            1 => GeoErrorKind::PermissionDenied,
            2 => GeoErrorKind::PositionUnavailable,
            3 => GeoErrorKind::Timeout,
            _ => GeoErrorKind::Unknown,
        };
    }

    /// Может ли ошибка исчезнуть при следующей попытке.
    pub fn is_recoverable(self) -> bool {
        return matches!(
            self,
            GeoErrorKind::Timeout | GeoErrorKind::PositionUnavailable
        );
    }
}
```

Разница принципиальна для UI:

- **Recoverable.** `watchPosition` продолжает работать после timeout — браузер повторит попытку. Не пугаем пользователя красным статусом, показываем жёлтый «ожидание позиции», трекинг не останавливаем.
- **Fatal.** `PermissionDenied` sticky — JS не может повторно вызвать диалог. Пользователь должен зайти в настройки вручную. Останавливаем трекинг, показываем инструкцию.

`PERMISSION_DENIED` в частности требует именно такого разделения: в момент отказа мы не знаем, пользователь нажал «Block» (навсегда) или просто закрыл диалог (временный `Prompt`). Пока политика явно не запретит, Permission API покажет `prompt` — браузер спросит ещё раз. Только `denied` — окончательно.

## 34.4. Проектный приём: Reflect + call3

В web-sys `Geolocation` есть методы `get_current_position_with_error_callback` (без options) и `get_current_position` (без error callback). Метода, который принимает **и** error callback, **и** `PositionOptions`, — нет.

Есть два выхода: либо работать без options (терять timeout и точность), либо вызвать метод через `js_sys::Reflect`. Второй вариант — универсальный и документированный:

```rust
fn call_geo_method(
    geo: &web_sys::Geolocation,
    method_name: &str,
    success: &js_sys::Function,
    error: Option<&js_sys::Function>,
    options: &web_sys::PositionOptions,
) -> Result<JsValue, JsValue> {
    let method: js_sys::Function =
        js_sys::Reflect::get(geo, &JsValue::from_str(method_name))?
            .dyn_into()
            .map_err(|_| JsValue::from_str("Geolocation method is not a function"))?;

    let geo_val: &JsValue = geo.unchecked_ref();
    let success_val: &JsValue = success.unchecked_ref();
    let error_val: JsValue = match error {
        Some(e) => e.unchecked_ref::<JsValue>().clone(),
        None => JsValue::UNDEFINED,
    };
    let options_val: &JsValue = options.unchecked_ref();

    return method.call3(geo_val, success_val, &error_val, options_val);
}
```

Здесь важен один приём: все web-sys типы реализуют `JsCast`, а значит через `unchecked_ref::<JsValue>()` мы получаем `&JsValue` без аллокаций. Если error не задан, передаём `JsValue::UNDEFINED` — ровно то, что ожидает JS API в качестве отсутствующего callback.

## 34.5. Практический location-компонент

Собираем всё вместе. Полный модуль `geo_core` включает:

- `Permission` enum и `permission_state()` через Permissions API.
- `GeoOptions`, `GeoPosition`, `GeoError`, `GeoErrorKind`.
- `haversine_m`, `bearing_deg`, `smooth_ema`, `GeoFormat`.
- `get_current_position`, `watch_position`, `clear_watch`.

Ключевая часть `get_current_position`:

```rust
pub fn get_current_position<FS, FE>(
    opts: &GeoOptions,
    mut on_success: FS,
    mut on_error: FE,
) -> Result<(), JsValue>
where
    FS: FnMut(GeoPosition) + 'static,
    FE: FnMut(GeoError) + 'static,
{
    let geo = web_sys::window()
        .ok_or("no window")?
        .navigator()
        .geolocation()?;

    let success = Closure::<dyn FnMut(web_sys::Position)>::new(
        move |pos: web_sys::Position| {
            on_success(GeoPosition::from_js(&pos));
        },
    );
    let error = Closure::<dyn FnMut(web_sys::PositionError)>::new(
        move |err: web_sys::PositionError| {
            on_error(GeoError::from_js(&err));
        },
    );

    let options = opts.to_web_sys()?;
    call_geo_method(
        &geo,
        "getCurrentPosition",
        success.as_ref().unchecked_ref(),
        Some(error.as_ref().unchecked_ref()),
        &options,
    )?;

    // Одноразовый запрос — closures можно forget, они не понадобятся.
    success.forget();
    error.forget();
    return Ok(());
}
```

**Важная деталь для `watch_position`.** Он возвращает `GeoHandle`, который **держит closures живыми**:

```rust
pub struct GeoHandle {
    watch_id: i32,
    _success: Closure<dyn FnMut(web_sys::Position)>,
    _error: Closure<dyn FnMut(web_sys::PositionError)>,
}
```

Если `Closure` собирается GC, браузерный callback продолжает вызываться, но Rust-функция уже уничтожена — получите либо панику, либо молчание. Пока `GeoHandle` жив — closures не собираются. Явно вызывайте `clear_watch(handle.watch_id())` перед drop, если хотите остановить браузерный трекинг сразу.

**Ещё одна ловушка.** В замыкании нельзя перемещать захваченный `on_success` наружу:

```rust
// ❌ Closure станет FnOnce:
move |pos| {
    let mut f = on_success;   // ← move из захвата
    f(GeoPosition::from_js(&pos));
}

// ✅ Closure останется FnMut:
move |pos| {
    on_success(GeoPosition::from_js(&pos));   // прямой вызов через &mut
}
```

`Closure::new` требует `FnMut`, потому что браузер может вызывать callback сколько угодно раз. Перемещение захвата наружу превращает замыкание в `FnOnce`, и компилятор откажется.

## 34.6. Отладка: как отделить шум от движения

Здесь начинается самая интересная часть. Пользователь сидит на месте, ничего не делает — а трекер показывает «прошёл 5 метров за 3 секунды». Это не движение, это **шум GPS**.

При `highAccuracy = false` браузер отдаёт координаты с точностью ±100–500 м. Два подряд сэмпла, взятых в одной точке, будут отличаться на десятки метров — просто потому, что Wi-Fi-геолокация работает с такой погрешностью. Если суммировать расстояния между всеми сэмплами как «пройденный путь», получите сотни метров, стоя на месте.

Решение — **фильтр по точности**: считаем расстояние движением, только если оно больше текущей погрешности.

```rust
if let Some(prev_pos) = &prev {
    let d = geo_core::haversine_m(
        prev_pos.latitude, prev_pos.longitude,
        pos.latitude, pos.longitude,
    );

    if d < pos.accuracy_m {
        // Расстояние меньше погрешности — это дрожание датчика,
        // а не движение. Не накапливаем дистанцию.
        filtered = true;
        app.watch_filtered.set(app.watch_filtered.get() + 1);
    } else {
        app.watch_accepted.set(app.watch_accepted.get() + 1);
        app.total_distance_m.set(app.total_distance_m.get() + d);
        // ... расчёт bearing, скорости
    }
}
```

Счётчики `smp / acc / flt` показывают, сколько сэмплов пришло, сколько принято, сколько отфильтровано. При стоянии на месте картина: 12 сэмплов, 1 принят, 11 отфильтровано. Это не «баг в коде», это **физика Wi-Fi-геолокации**.

**Траекторию тоже надо фильтровать.** Если складывать все точки в массив и рисовать SVG-полилинию, при стоянии на месте получится зигзаг из шума, визуально неотличимый от настоящего движения. Правило: **в траекторию идут только принятые точки**, плюс самая первая — чтобы был маркер «трек начался»:

```rust
let traj_len = app.traj_points.borrow().len();
if traj_len == 0 || !filtered {
    app.traj_points
        .borrow_mut()
        .push((pos.latitude, pos.longitude));
}
```

**Ещё один независимый детектор** — `moving / stationary`. Он отличается от фильтра шума: тот отбрасывает одиночные сэмплы, а этот смотрит на несколько сэмплов подряд. Если за последние 5 попыток расстояние ни разу не превысило `2 × accuracy`, статус переключается на «стою». Только если пользователь реально сдвинулся, метка сменится на «двигаюсь».

```rust
fn update_moving_status(
    app: &Rc<App>,
    distance_from_prev: Option<f64>,
    accuracy_m: f64,
) {
    let is_moving_now = match distance_from_prev {
        Some(d) => d > 2.0 * accuracy_m,
        None => false,
    };
    if is_moving_now {
        app.stationary_streak.set(0);
        app.moving.set(true);
    } else {
        let streak = app.stationary_streak.get() + 1;
        app.stationary_streak.set(streak);
        if streak >= 5 {
            app.moving.set(false);
        }
    }
}
```

Пороги `2 × accuracy` и `5 сэмплов` подобраны на практике: точность 131 м даёт порог 262 м, что отсекает типичный Wi-Fi-шум и пропускает реальное движение.

## 34.7. Прочие практические приёмы

**Warm-up.** Первые 2–3 сэмпла после старта трекинга обычно менее точные, чем последующие: GPS-чип прогревается, Wi-Fi-сканирование собирает свежие данные. Помечаем их маркером `[warm-up]` в журнале и другим цветом строки — пользователь видит, что первые данные могут быть неточными.

**Две скорости.** GPS-датчик часто отдаёт `speed` напрямую, но на десктопе это поле `null`. Считаем скорость из двух сэмплов: `d / dt`. Если расчётная скорость превышает 300 км/ч — это не движение, а сбой, отбрасываем как аномалию.

**Два азимута.** Аналогично, `heading` от GPS — `null` на десктопе. Считаем `bearing` из двух точек по формуле `atan2(sin(Δλ)cos(φ2), cos(φ1)sin(φ2) − sin(φ1)cos(φ2)cos(Δλ))`. Результат [0, 360), 0° — север.

**EMA-сглаживание.** Простая экспоненциальная скользящая средняя `smooth_ema(prev, current, 0.3)` даёт визуально стабильную траекторию без изобретения Kalman-фильтра. Для отображения достаточно.

**Сессионный таймер.** Сколько времени прошло от старта трекинга. Полезно для понимания, как долго живёт `watchPosition` — на десктопе Firefox может выдать всего 1–2 сэмпла за минуту.

**Сохранение финального времени.** При остановке трекинга `session_start_ms` сбрасывается, но финальное значение фиксируется в `session_final_s` — после остановки в бейдже видно «сколько длилось».

## 34.8. Диагностика окружения

Как и в главе про уведомления, диагностика — часть демо, а не опция. Панель собирает снимок окружения:

```rust
fn collect_diagnostics(app: Rc<App>) {
    // ...
    rows.push(("isSecureContext", yes_no(is_secure)));
    rows.push(("Geolocation API", yes_no(geo_core::is_supported())));
    rows.push(("Permissions API (geolocation)",
        geo_core::permission_state().await.unwrap_or_else(|| "n/a".into())));
    rows.push(("userAgent", ua));
    rows.push(("document.visibilityState", vis));
    rows.push(("document.hasFocus()", yes_no(has_focus)));
}
```

Что здесь важно:

- **`isSecureContext`** — фундаментальное требование. Без HTTPS или `localhost` Geolocation API недоступен.
- **Permissions API** — надёжнее, чем `Notification.permission` (аналог из главы 33). Показывает `granted / denied / prompt`.
- **`document.visibilityState`** — если вкладка в фоне, браузер может приостановить `watchPosition`. На десктопе это особенно заметно: сэмплы приходят реже или совсем перестают.
- **`document.hasFocus()`** — если окно не в фокусе, некоторые системы вообще глушат геолокацию, экономя батарею.

## 34.9. Что мы построили

Полноценное демо главы включает:

1. **Модуль `geo_core`** — обёртка над API, ошибки, геометрия, форматирование.
2. **Компонент `app`** — три колонки UI: управление, позиция + визуализация, журнал.
3. **15 практических доработок:** multi-sample, счётчики, pause/resume, фильтр шума, детектор движения, две скорости, два азимута, warm-up, статистика точности, session timer, копирование, DD/DMS, диагностика, `onchange`-подписка, полный сброс.
4. **Диагностический инструментарий** — снимок окружения.

Ключевые правила, вынесенные из главы:

1. **`timeout = 0` — почти всегда ошибка.** Бесконечное ожидание GPS на десктопе без чипа — это «зависший» колбэк.
2. **`enableHighAccuracy = true` — не всегда лучше.** На десктопе без GPS это гарантированный timeout.
3. **`PERMISSION_DENIED` — sticky.** После отказа диалог не появится. Нужна инструкция для пользователя.
4. **`clearWatch` идемпотентна.** Повторный вызов с тем же ID — не ошибка.
5. **`altitude`, `heading`, `speed` часто `null`.** Проверяйте `Option`, не выводите «NaN м/с».
6. **Фильтр шума обязателен.** `distance < accuracy` — это дрожание датчика, а не движение.
7. **Траектория — только принятые точки.** Иначе SVG рисует зигзаг из шума.
8. **Recoverable ≠ Fatal.** Timeout и `PositionUnavailable` — восстанавливаемые, трекинг продолжается. `PermissionDenied` — фатальная, стоп.
9. **Closure для `FnMut` нельзя перемещать наружу.** `let mut f = callback; f(x);` — ошибка компиляции. `callback(x);` — правильно.
10. **PositionOptions — dictionary.** Геттеров не существует, только сеттеры.

В следующей главе — Web Share API: как поделиться координатами с другими приложениями устройства.

## 34.10. Приложение. Исходные коды

**Cargo.toml**:

```
    "Geolocation",
    "Position",
    "PositionError",
    "PositionOptions",
    "Coordinates",
    "VisibilityState",
```

**geo_core.rs**
```
//! Геолокация: getCurrentPosition, watchPosition, clearWatch.
//!
//! ПРАВИЛО: callbacks — единственный способ получить результат.
//! getCurrentPosition возвращает void, не Promise.
//!
//! ПРАВИЛО: Closure должны жить, пока идёт запрос. Для одноразового
//! запроса — forget(), для трекинга — храним в GeoHandle.
//!
//! ПРАВИЛО: timeout = 0 означает «бесконечно» — почти всегда ошибка.
//!
//! ПРАВИЛО: web-sys не генерирует `*_with_error_callback_and_options`
//! для Geolocation. Вызываем через Reflect + Function::call3.
//!
//! ПРАВИЛО: в замыкании для Closure нельзя перемещать захваченный
//! FnMut наружу, иначе замыкание станет FnOnce.
//!
//! ТРЕБУЕТСЯ В Cargo.toml (features web-sys):
//!   "Geolocation", "Position", "PositionError",
//!   "PositionOptions", "Coordinates"

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};

// ============================================================
//                    ОШИБКИ
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum GeoErrorKind {
    PermissionDenied,
    PositionUnavailable,
    Timeout,
    Unknown,
}

impl GeoErrorKind {
    /// Маппинг кода PositionError в наш enum.
    pub fn from_code(code: u16) -> Self {
        return match code {
            1 => GeoErrorKind::PermissionDenied,
            2 => GeoErrorKind::PositionUnavailable,
            3 => GeoErrorKind::Timeout,
            _ => GeoErrorKind::Unknown,
        };
    }

    /// Может ли ошибка исчезнуть при следующей попытке.
    ///
    /// - `Timeout` — браузер продолжит пытаться получить позицию.
    /// - `PositionUnavailable` — временное отсутствие сигнала.
    /// - `PermissionDenied` — sticky, без действий пользователя не исправить.
    /// - `Unknown` — неизвестно, считаем фатальной.
    pub fn is_recoverable(self) -> bool {
        return matches!(
            self,
            GeoErrorKind::Timeout | GeoErrorKind::PositionUnavailable
        );
    }
}

#[derive(Clone, Debug)]
pub struct GeoError {
    pub kind: GeoErrorKind,
    pub message: String,
}

impl GeoError {
    pub fn from_js(err: &web_sys::PositionError) -> Self {
        return Self {
            kind: GeoErrorKind::from_code(err.code()),
            message: err.message(),
        };
    }

    pub fn user_hint(&self) -> &'static str {
        return match self.kind {
            GeoErrorKind::PermissionDenied => {
                "Доступ к геолокации запрещён. Разрешите его в настройках браузера \
                 (значок замка в адресной строке → Разрешения)."
            }
            GeoErrorKind::PositionUnavailable => {
                "Не удалось определить позицию: нет GPS-сигнала и недоступна \
                 геолокация по сети. Попробуйте отключить высокую точность \
                 или ввести координаты вручную."
            }
            GeoErrorKind::Timeout => {
                "Истекло время ожидания. Увеличьте timeout или отключите \
                 высокую точность."
            }
            GeoErrorKind::Unknown => {
                "Неизвестная ошибка геолокации. Проверьте консоль браузера."
            }
        };
    }
}

// ============================================================
//                    ПОЗИЦИЯ
// ============================================================

#[derive(Clone, Debug)]
pub struct GeoPosition {
    pub latitude: f64,
    pub longitude: f64,
    pub accuracy_m: f64,
    pub altitude_m: Option<f64>,
    pub heading_deg: Option<f64>,
    pub speed_mps: Option<f64>,
    pub timestamp_ms: f64,
}

impl GeoPosition {
    pub fn from_js(pos: &web_sys::Position) -> Self {
        let c = pos.coords();
        return Self {
            latitude: c.latitude(),
            longitude: c.longitude(),
            accuracy_m: c.accuracy(),
            altitude_m: c.altitude(),
            heading_deg: c.heading(),
            speed_mps: c.speed(),
            timestamp_ms: pos.timestamp(),
        };
    }

    pub fn accuracy_label(&self) -> &'static str {
        return if self.accuracy_m < 20.0 {
            "высокая (GPS)"
        } else if self.accuracy_m < 200.0 {
            "средняя (сеть)"
        } else {
            "низкая (IP)"
        };
    }
}

// ============================================================
//                    ГЕОМЕТРИЯ
// ============================================================

/// Расстояние между двумя точками по большому кругу (haversine), метры.
///
/// Радиус Земли — средний, 6 371 000 м. Для последовательных точек
/// GPS-трекинга (десятки метров друг от друга) ошибка < 0.5%.
pub fn haversine_m(lat1: f64, lon1: f64, lat2: f64, lon2: f64) -> f64 {
    const R: f64 = 6_371_000.0;
    let dlat = (lat2 - lat1).to_radians();
    let dlon = (lon2 - lon1).to_radians();
    let a = (dlat / 2.0).sin().powi(2)
        + lat1.to_radians().cos() * lat2.to_radians().cos()
            * (dlon / 2.0).sin().powi(2);
    return 2.0 * R * a.sqrt().asin();
}

/// Начальный азимут из точки 1 в точку 2, градусы [0, 360).
///
/// 0° — север, 90° — восток. Используется, когда GPS не отдаёт
/// `heading` (на десктопе почти всегда `null`).
pub fn bearing_deg(lat1: f64, lon1: f64, lat2: f64, lon2: f64) -> f64 {
    let phi1 = lat1.to_radians();
    let phi2 = lat2.to_radians();
    let dlon = (lon2 - lon1).to_radians();
    let x = dlon.sin() * phi2.cos();
    let y = phi1.cos() * phi2.sin() - phi1.sin() * phi2.cos() * dlon.cos();
    let deg = x.atan2(y).to_degrees();
    return if deg < 0.0 { deg + 360.0 } else { deg };
}

/// EMA-сглаживание: `alpha` — вес нового значения (0..1).
///
/// `alpha = 0.3` — заметное сглаживание, но реакция на движение
/// сохраняется. `alpha = 0.05` — почти статика. `alpha = 1.0` —
/// отключение сглаживания.
pub fn smooth_ema(prev: f64, current: f64, alpha: f64) -> f64 {
    return alpha * current + (1.0 - alpha) * prev;
}

// ============================================================
//                    ФОРМАТ КООРДИНАТ
// ============================================================

/// Формат отображения координат.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum GeoFormat {
    /// Десятичные градусы: `12.871196° N`.
    Decimal,
    /// Градусы-минуты-секунды: `12° 52' 16.3" N`.
    Dms,
}

impl GeoFormat {
    pub fn format_lat(&self, lat: f64) -> String {
        let dir = if lat >= 0.0 { "N" } else { "S" };
        return self.format(lat.abs(), dir);
    }

    pub fn format_lon(&self, lon: f64) -> String {
        let dir = if lon >= 0.0 { "E" } else { "W" };
        return self.format(lon.abs(), dir);
    }

    fn format(&self, value: f64, dir: &str) -> String {
        return match self {
            GeoFormat::Decimal => {
                crate::format_str::format_str(format_args!("{:.6}° {dir}", value))
            }
            GeoFormat::Dms => {
                let deg = value.floor();
                let min_total = (value - deg) * 60.0;
                let min = min_total.floor();
                let sec = (min_total - min) * 60.0;
                crate::format_str::format_str(format_args!(
                    "{:.0}° {:.0}' {:.1}\" {dir}",
                    deg, min, sec,
                ))
            }
        };
    }
}

// ============================================================
//                    ОПЦИИ
// ============================================================

#[derive(Clone, Copy, Debug)]
pub struct GeoOptions {
    pub high_accuracy: bool,
    pub timeout_ms: u32,
    pub max_age_ms: u32,
}

impl Default for GeoOptions {
    fn default() -> Self {
        return Self {
            high_accuracy: false,
            timeout_ms: 12000,
            max_age_ms: 0,
        };
    }
}

impl GeoOptions {
    pub fn to_web_sys(&self) -> Result<web_sys::PositionOptions, JsValue> {
        let o = web_sys::PositionOptions::new();
        o.set_enable_high_accuracy(self.high_accuracy);
        o.set_timeout(self.timeout_ms);
        o.set_maximum_age(self.max_age_ms);
        return Ok(o);
    }
}

// ============================================================
//                    WATCH HANDLE
// ============================================================

pub struct GeoHandle {
    watch_id: i32,
    _success: Closure<dyn FnMut(web_sys::Position)>,
    _error: Closure<dyn FnMut(web_sys::PositionError)>,
}

impl GeoHandle {
    pub fn watch_id(&self) -> i32 {
        return self.watch_id;
    }
}

// ============================================================
//                    PUBLIC API
// ============================================================

pub fn is_supported() -> bool {
    return web_sys::window()
        .and_then(|w| w.navigator().geolocation().ok())
        .is_some();
}

fn call_geo_method(
    geo: &web_sys::Geolocation,
    method_name: &str,
    success: &js_sys::Function,
    error: Option<&js_sys::Function>,
    options: &web_sys::PositionOptions,
) -> Result<JsValue, JsValue> {
    let method: js_sys::Function =
        js_sys::Reflect::get(geo, &JsValue::from_str(method_name))?
            .dyn_into()
            .map_err(|_| JsValue::from_str("Geolocation method is not a function"))?;

    let geo_val: &JsValue = geo.unchecked_ref();
    let success_val: &JsValue = success.unchecked_ref();
    let error_val: JsValue = match error {
        Some(e) => e.unchecked_ref::<JsValue>().clone(),
        None => JsValue::UNDEFINED,
    };
    let options_val: &JsValue = options.unchecked_ref();

    return method.call3(geo_val, success_val, &error_val, options_val);
}

pub fn get_current_position<FS, FE>(
    opts: &GeoOptions,
    mut on_success: FS,
    mut on_error: FE,
) -> Result<(), JsValue>
where
    FS: FnMut(GeoPosition) + 'static,
    FE: FnMut(GeoError) + 'static,
{
    let geo = web_sys::window()
        .ok_or("no window")?
        .navigator()
        .geolocation()?;

    let success = Closure::<dyn FnMut(web_sys::Position)>::new(
        move |pos: web_sys::Position| {
            on_success(GeoPosition::from_js(&pos));
        },
    );
    let error = Closure::<dyn FnMut(web_sys::PositionError)>::new(
        move |err: web_sys::PositionError| {
            on_error(GeoError::from_js(&err));
        },
    );

    let options = opts.to_web_sys()?;
    call_geo_method(
        &geo,
        "getCurrentPosition",
        success.as_ref().unchecked_ref(),
        Some(error.as_ref().unchecked_ref()),
        &options,
    )?;

    success.forget();
    error.forget();
    return Ok(());
}

pub fn watch_position<FS, FE>(
    opts: &GeoOptions,
    mut on_success: FS,
    mut on_error: FE,
) -> Result<GeoHandle, JsValue>
where
    FS: FnMut(GeoPosition) + 'static,
    FE: FnMut(GeoError) + 'static,
{
    let geo = web_sys::window()
        .ok_or("no window")?
        .navigator()
        .geolocation()?;

    let success = Closure::<dyn FnMut(web_sys::Position)>::new(
        move |pos: web_sys::Position| {
            on_success(GeoPosition::from_js(&pos));
        },
    );
    let error = Closure::<dyn FnMut(web_sys::PositionError)>::new(
        move |err: web_sys::PositionError| {
            on_error(GeoError::from_js(&err));
        },
    );

    let options = opts.to_web_sys()?;
    let result = call_geo_method(
        &geo,
        "watchPosition",
        success.as_ref().unchecked_ref(),
        Some(error.as_ref().unchecked_ref()),
        &options,
    )?;

    let watch_id = result
        .as_f64()
        .ok_or_else(|| JsValue::from_str("watchPosition did not return a number"))?
        as i32;

    return Ok(GeoHandle {
        watch_id,
        _success: success,
        _error: error,
    });
}

pub fn clear_watch(watch_id: i32) -> Result<(), JsValue> {
    let geo = web_sys::window()
        .ok_or("no window")?
        .navigator()
        .geolocation()?;
    geo.clear_watch(watch_id);
    return Ok(());
}

/// Читает статус разрешения через Permissions API.
pub async fn permission_state() -> Option<String> {
    use wasm_bindgen_futures::JsFuture;

    let global = js_sys::global();
    let nav = js_sys::Reflect::get(&global, &JsValue::from_str("navigator")).ok()?;
    let permissions = js_sys::Reflect::get(&nav, &JsValue::from_str("permissions")).ok()?;
    if permissions.is_undefined() || permissions.is_null() {
        return None;
    }
    let query: js_sys::Function =
        js_sys::Reflect::get(&permissions, &JsValue::from_str("query"))
            .ok()?
            .dyn_into()
            .ok()?;
    let desc = js_sys::Object::new();
    js_sys::Reflect::set(
        &desc,
        &JsValue::from_str("name"),
        &JsValue::from_str("geolocation"),
    )
    .ok()?;
    let promise = query.call1(&permissions, &desc).ok()?;
    let promise: js_sys::Promise = promise.dyn_into().ok()?;
    let result = JsFuture::from(promise).await.ok()?;
    return js_sys::Reflect::get(&result, &JsValue::from_str("state"))
        .ok()?
        .as_string();
}
```

**wasm_geo_core.rs**:
```
//! Wasm-тесты geo_core: доступность API, коды ошибок, options,
//! геометрия, форматирование координат.
//!
//! ПРАВИЛО: PositionError нельзя сконструировать в JS — только
//! получить от браузера. Маппинг кодов тестируется через from_code.
//!
//! ПРАВИЛО: PositionOptions — dictionary. web-sys не генерирует
//! getter'ы для dictionary, только setters.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::geo_core::{
    self, GeoError, GeoErrorKind, GeoFormat, GeoOptions, GeoPosition,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------- Доступность API ----------------

#[wasm_bindgen_test]
fn wasm_geolocation_api_is_supported() {
    assert!(geo_core::is_supported());
}

// ---------------- GeoOptions ----------------

#[wasm_bindgen_test]
fn wasm_default_options_have_finite_timeout() {
    let o = GeoOptions::default();
    assert!(o.timeout_ms > 0);
    assert!(o.timeout_ms <= 30_000);
}

#[wasm_bindgen_test]
fn wasm_options_convert_to_web_sys() {
    let opts = GeoOptions {
        high_accuracy: true,
        timeout_ms: 5000,
        max_age_ms: 1000,
    };
    // Dictionary: getters отсутствуют, проверяем только успешность.
    assert!(opts.to_web_sys().is_ok());
}

// ---------------- Маппинг кодов ошибок ----------------

#[wasm_bindgen_test]
fn wasm_error_kind_from_code_all() {
    assert_eq!(GeoErrorKind::from_code(1), GeoErrorKind::PermissionDenied);
    assert_eq!(GeoErrorKind::from_code(2), GeoErrorKind::PositionUnavailable);
    assert_eq!(GeoErrorKind::from_code(3), GeoErrorKind::Timeout);
    assert_eq!(GeoErrorKind::from_code(0), GeoErrorKind::Unknown);
    assert_eq!(GeoErrorKind::from_code(99), GeoErrorKind::Unknown);
}

#[wasm_bindgen_test]
fn wasm_error_recoverable_flags() {
    assert!(GeoErrorKind::Timeout.is_recoverable());
    assert!(GeoErrorKind::PositionUnavailable.is_recoverable());
    assert!(!GeoErrorKind::PermissionDenied.is_recoverable());
    assert!(!GeoErrorKind::Unknown.is_recoverable());
}

#[wasm_bindgen_test]
fn wasm_user_hints_are_non_empty_for_all_kinds() {
    for kind in [
        GeoErrorKind::PermissionDenied,
        GeoErrorKind::PositionUnavailable,
        GeoErrorKind::Timeout,
        GeoErrorKind::Unknown,
    ] {
        let e = GeoError {
            kind,
            message: "test".to_string(),
        };
        assert!(!e.user_hint().is_empty(), "kind = {:?}", kind);
    }
}

// ---------------- Haversine ----------------

#[wasm_bindgen_test]
fn wasm_haversine_zero_for_same_point() {
    let d = geo_core::haversine_m(12.87, 100.89, 12.87, 100.89);
    assert!(d < 0.001, "distance = {d}");
}

#[wasm_bindgen_test]
fn wasm_haversine_pattaya_to_bangkok() {
    // Паттайя (12.9236, 100.8824) — Бангкок (13.7563, 100.5018).
    // Реальное расстояние ≈ 100 км (по прямой).
    let d = geo_core::haversine_m(12.9236, 100.8824, 13.7563, 100.5018);
    assert!(d > 90_000.0, "too short: {d}");
    assert!(d < 110_000.0, "too long: {d}");
}

#[wasm_bindgen_test]
fn wasm_haversine_equator_one_degree() {
    // 1° по долготе на экваторе ≈ 111 195 м.
    let d = geo_core::haversine_m(0.0, 0.0, 0.0, 1.0);
    assert!((d - 111_195.0).abs() < 100.0, "d = {d}");
}

#[wasm_bindgen_test]
fn wasm_haversine_symmetric() {
    let d1 = geo_core::haversine_m(10.0, 20.0, 30.0, 40.0);
    let d2 = geo_core::haversine_m(30.0, 40.0, 10.0, 20.0);
    assert!((d1 - d2).abs() < 0.001);
}

// ---------------- Bearing ----------------

#[wasm_bindgen_test]
fn wasm_bearing_north_is_zero() {
    // Из (0,0) в (1,0) — строго на север.
    let b = geo_core::bearing_deg(0.0, 0.0, 1.0, 0.0);
    assert!(b < 0.5 || b > 359.5, "b = {b}");
}

#[wasm_bindgen_test]
fn wasm_bearing_east_is_ninety() {
    // Из (0,0) в (0,1) — строго на восток.
    let b = geo_core::bearing_deg(0.0, 0.0, 0.0, 1.0);
    assert!((b - 90.0).abs() < 0.5, "b = {b}");
}

#[wasm_bindgen_test]
fn wasm_bearing_south_is_180() {
    let b = geo_core::bearing_deg(1.0, 0.0, 0.0, 0.0);
    assert!((b - 180.0).abs() < 0.5, "b = {b}");
}

#[wasm_bindgen_test]
fn wasm_bearing_in_range() {
    for (lat1, lon1, lat2, lon2) in [
        (10.0, 20.0, 30.0, 40.0),
        (-5.0, 100.0, 5.0, -100.0),
        (45.0, 0.0, 45.0, 179.9),
    ] {
        let b = geo_core::bearing_deg(lat1, lon1, lat2, lon2);
        assert!(b >= 0.0 && b < 360.0, "b = {b}");
    }
}

// ---------------- Smoothing ----------------

#[wasm_bindgen_test]
fn wasm_smooth_ema_identity_at_one() {
    // alpha = 1.0 — новое значение полностью перетирает старое.
    assert!((geo_core::smooth_ema(0.0, 10.0, 1.0) - 10.0).abs() < 1e-9);
}

#[wasm_bindgen_test]
fn wasm_smooth_ema_no_change_at_zero() {
    // alpha = 0.0 — старое значение сохраняется.
    assert!((geo_core::smooth_ema(5.0, 10.0, 0.0) - 5.0).abs() < 1e-9);
}

#[wasm_bindgen_test]
fn wasm_smooth_ema_midpoint() {
    // alpha = 0.5 — среднее.
    assert!((geo_core::smooth_ema(0.0, 10.0, 0.5) - 5.0).abs() < 1e-9);
}

// ---------------- Формат координат ----------------

#[wasm_bindgen_test]
fn wasm_format_decimal_lat() {
    let s = GeoFormat::Decimal.format_lat(12.871196);
    assert!(s.contains("N"), "s = {s}");
    assert!(s.contains("12.871196"), "s = {s}");
}

#[wasm_bindgen_test]
fn wasm_format_decimal_lon_west() {
    let s = GeoFormat::Decimal.format_lon(-100.5);
    assert!(s.contains("W"), "s = {s}");
    assert!(s.contains("100.5"), "s = {s}");
}

#[wasm_bindgen_test]
fn wasm_format_dms_lat() {
    // 12.871196° = 12° 52' 16.3" N.
    let s = GeoFormat::Dms.format_lat(12.871196);
    assert!(s.contains("12°"), "s = {s}");
    assert!(s.contains("52'"), "s = {s}");
    assert!(s.contains("N"), "s = {s}");
}

#[wasm_bindgen_test]
fn wasm_format_dms_zero() {
    let s = GeoFormat::Dms.format_lat(0.0);
    assert!(s.contains("0° 0' 0.0\""), "s = {s}");
}

// ---------------- Accuracy labels ----------------

#[wasm_bindgen_test]
fn wasm_accuracy_labels_boundaries() {
    let make = |acc: f64| GeoPosition {
        latitude: 0.0,
        longitude: 0.0,
        accuracy_m: acc,
        altitude_m: None,
        heading_deg: None,
        speed_mps: None,
        timestamp_ms: 0.0,
    };
    assert_eq!(make(0.0).accuracy_label(), "высокая (GPS)");
    assert_eq!(make(19.99).accuracy_label(), "высокая (GPS)");
    assert_eq!(make(20.0).accuracy_label(), "средняя (сеть)");
    assert_eq!(make(199.99).accuracy_label(), "средняя (сеть)");
    assert_eq!(make(200.0).accuracy_label(), "низкая (IP)");
    assert_eq!(make(5000.0).accuracy_label(), "низкая (IP)");
}

// ---------------- clear_watch идемпотентен ----------------

#[wasm_bindgen_test]
fn wasm_clear_watch_is_idempotent() {
    let _ = geo_core::clear_watch(99999);
    let _ = geo_core::clear_watch(99999);
}

// ---------------- Permissions API ----------------

#[wasm_bindgen_test]
fn wasm_permission_state_returns_something_or_none() {
    let _ = geo_core::permission_state();
}
```

**app.rs**:
```
//! Демо Главы 34: Geolocation API.
//!
//! Практический location-компонент:
//!   1.  Multi-sample: 3× getCurrentPosition параллельно.
//!   2.  Счётчики smp / acc / flt / err.
//!   3.  Pause / resume (clearWatch + пересоздание watch).
//!   4.  GPS-фильтр шума (distance > accuracy).
//!   5.  Индикатор moving / stationary.
//!   6.  Скорость из двух сэмплов vs GPS-скорость.
//!   7.  Азимут (bearing) из двух точек.
//!   8.  Warm-up marking первых 3 сэмплов.
//!   9.  Статистика точности (SVG-график).
//!   10. Session timer с сохранением финального времени.
//!   11. Копировать координаты в буфер.
//!   12. DD / DMS формат.
//!   13. Диагностика окружения.
//!   14. permission.onchange с дедупликацией логов.
//!   15. Полный сброс.
//!   16. Траектория фильтрует шум: при стоянии на месте точки не
//!       добавляются — SVG не рисует ложный зигзаг.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: две части UI, зависящие от одного состояния, — через
//! общее Rc<Cell<...>>, не через проброс колбэков.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::callback::Interval;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;

use crate::format_str::format_str;
use crate::geo_core::{self, GeoError, GeoFormat, GeoHandle, GeoOptions, GeoPosition};

const LOG_LIMIT: usize = 14;
const ACCURACY_WARN_THRESHOLD_M: f64 = 200.0;
const SPEED_ANOMALY_MPS: f64 = 83.3; // ≈ 300 км/ч
const WARMUP_SAMPLES: u32 = 3;
const SMOOTH_ALPHA: f64 = 0.3;

/// Порог «жёлтый» для badge age — трекинг молчит >5 сек.
const AGE_WARN_S: i64 = 5;
/// Порог «красный» для badge age — трекинг молчит >15 сек.
const AGE_ERR_S: i64 = 15;

const HTML: &str = r#"
<style>
  :root {
    --gl-info: #3498db; --gl-ok: #27ae60; --gl-err: #c0392b; --gl-warn: #e67e22;
    --gl-text: #222; --gl-muted: #666;
    --gl-bg: #fff; --gl-soft: #fafafa; --gl-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --gl-text: #e6e6e6; --gl-muted: #a0a0a0;
      --gl-bg: #1e1e1e; --gl-soft: #262626; --gl-border: #3a3a3a;
    }
    body { background: #121212; }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .gl-log-entry { background: #1e1e1e; }
    .gl-anno { background: #1a2233; color: #9fc4ff; }
    .gl-hint { background: #232014; color: #e6c98c; border-left-color: #5a4400; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--gl-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .gl-subtitle { color: var(--gl-muted); font-size: 12px; margin: 0 0 12px; }
  .gl-subtitle a { color: var(--gl-info); }

  .gl-grid {
    display: grid; gap: 12px;
    grid-template-columns: minmax(0, 1.15fr) minmax(0, 1fr) minmax(0, 1fr);
  }
  @media (max-width: 1100px) { .gl-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px)  { .gl-grid { grid-template-columns: 1fr; } }
  .gl-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .gl-panel {
    border: 1px solid var(--gl-border); border-radius: 8px;
    padding: 10px 12px; background: var(--gl-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .gl-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px;
  }
  .gl-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--gl-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .gl-anno {
    background: #eef4ff; border-left: 3px solid var(--gl-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere;
  }
  .gl-hint {
    background: #fff9e6; border-left: 3px solid var(--gl-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #7d5a00;
  }
  .gl-hint--hidden { display: none; }

  .gl-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; }
  input[type="number"] {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--gl-bg); color: var(--gl-text);
    font-family: ui-monospace, monospace; width: 100px;
  }
  label.gl-check {
    display: inline-flex; align-items: center; gap: 4px;
    font-size: 11px; color: var(--gl-muted); cursor: pointer;
    user-select: none;
  }
  .gl-field-label {
    font-size: 11px; color: var(--gl-muted); margin-right: 4px;
  }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--gl-soft); color: var(--gl-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.gl-primary { background: #eaf4ff; border-color: var(--gl-info); color: #1f4e79; }
  button.gl-success { background: #e8f8ed; border-color: var(--gl-ok); color: #1e8449; }
  button.gl-danger  { background: #fdedec; border-color: var(--gl-err); color: #922b21; }

  .gl-status {
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--gl-soft); border-left: 3px solid #ccc;
    color: var(--gl-muted); min-height: 22px; overflow-wrap: anywhere;
  }
  .gl-status.ok   { background: #e8f8ed; border-color: var(--gl-ok); color: #1e8449; }
  .gl-status.err  { background: #fdedec; border-color: var(--gl-err); color: #922b21; }
  .gl-status.busy { background: #fff9e6; border-color: var(--gl-warn); color: #7d5a00; }

  .gl-coord-grid {
    display: grid; grid-template-columns: auto 1fr;
    gap: 4px 12px; margin: 6px 0;
    font-family: ui-monospace, monospace; font-size: 12px;
  }
  .gl-coord-grid .k { color: var(--gl-muted); }
  .gl-coord-grid .v { color: var(--gl-text); }

  .gl-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
  }
  @media (prefers-color-scheme: dark) {
    .gl-badge { background: #2a3a4a; color: #cfe0ff; }
  }
  .gl-badge--ok  { background: #e8f8ed; color: #1e8449; }
  .gl-badge--err { background: #fdedec; color: #922b21; }
  .gl-badge--warn { background: #fff9e6; color: #7d5a00; }

  .gl-map-link {
    display: inline-block; margin-top: 6px;
    font-size: 12px; color: var(--gl-info); text-decoration: none;
  }
  .gl-map-link:hover { text-decoration: underline; }

  .gl-svg {
    background: var(--gl-soft);
    border: 1px solid var(--gl-border);
    border-radius: 4px;
    width: 100%;
    display: block;
  }

  .gl-log {
    border: 1px solid var(--gl-border); border-radius: 5px;
    background: var(--gl-soft); max-height: 420px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .gl-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .gl-log-entry {
    display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent;
  }
  .gl-log-entry.ok   { border-left-color: var(--gl-ok); }
  .gl-log-entry.err  { border-left-color: var(--gl-err); }
  .gl-log-entry.info { border-left-color: var(--gl-info); }
  .gl-log-entry.warn { border-left-color: var(--gl-warn); }
  .gl-log-entry .t { color: var(--gl-muted); flex-shrink: 0; }
  .gl-log-entry .m { flex: 1; overflow-wrap: anywhere; }
  .gl-log-entry.err .m  { color: #922b21; }
  .gl-log-entry.warn .m { color: #7d5a00; }

  .gl-diag-panel { margin-top: 12px; }
  .gl-diag-out {
    background: var(--gl-soft); border: 1px solid var(--gl-border);
    border-radius: 5px; padding: 8px 12px; margin-top: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    max-height: 360px; overflow-y: auto;
  }
  .gl-diag-row {
    display: grid; grid-template-columns: minmax(200px, 42%) 1fr;
    gap: 8px; padding: 2px 0;
    border-bottom: 1px dashed rgba(128,128,128,0.15);
  }
  .gl-diag-row .k { color: var(--gl-muted); overflow-wrap: anywhere; }
  .gl-diag-row .v { color: var(--gl-text); overflow-wrap: anywhere; }
</style>

<h1>Глава 34: Geolocation</h1>
<p class="gl-subtitle">
  getCurrentPosition, watchPosition, clearWatch, обработка ошибок.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="gl-grid">

  <!-- ============ Колонка 1: Управление ============ -->
  <div class="gl-col">
    <div class="gl-panel">
      <h2 class="gl-panel-title">
        <span class="gl-num">1</span>Управление
      </h2>
      <div class="gl-anno">
        Геолокация требует <b>secure context</b>. Разрешение — <b>sticky
        per origin</b>. После отказа диалог не появится — нужно менять
        вручную в настройках браузера.
      </div>

      <div class="gl-row">
        <label class="gl-check">
          <input type="checkbox" id="gl-high-acc"> enableHighAccuracy
        </label>
      </div>
      <div class="gl-hint gl-hint--hidden" id="gl-high-acc-hint">
        Включает GPS. Точность растёт, но на десктопе без чипа
        запрос может занять весь timeout.
      </div>

      <div class="gl-row">
        <span class="gl-field-label">timeout:</span>
        <input type="number" id="gl-timeout" value="12000" min="1000" max="60000" step="1000">
        <span class="gl-field-label">мс</span>
      </div>
      <div class="gl-row">
        <span class="gl-field-label">maxAge:</span>
        <input type="number" id="gl-maxage" value="0" min="0" max="600000" step="1000">
        <span class="gl-field-label">мс</span>
      </div>

      <div class="gl-row">
        <button id="gl-get-once" class="gl-primary">📍 Определить позицию</button>
        <button id="gl-get-3x">📊 3× замер</button>
      </div>

      <div class="gl-row">
        <button id="gl-watch-toggle" class="gl-success">▶ Начать трекинг</button>
        <button id="gl-watch-stop" class="gl-danger" disabled>■ Стоп</button>
      </div>

      <div class="gl-row">
        <span class="gl-badge" id="gl-badge-state">—</span>
        <span class="gl-badge" id="gl-badge-move" title="детектор движения">—</span>
      </div>
      <div class="gl-row">
        <span class="gl-badge" id="gl-badge-count" title="всего сэмплов от watchPosition">0 smp</span>
        <span class="gl-badge" id="gl-badge-accepted" title="принято (не отфильтровано)">0 acc</span>
        <span class="gl-badge" id="gl-badge-filtered" title="отфильтровано шумом">0 flt</span>
        <span class="gl-badge" id="gl-badge-errors" title="ошибок">0 err</span>
      </div>
      <div class="gl-row">
        <span class="gl-badge" id="gl-badge-session" title="длительность сессии">0:00</span>
        <span class="gl-badge" id="gl-badge-age" title="время с последнего сэмпла">—</span>
        <span class="gl-badge" id="gl-badge-perm" title="permissions.query">—</span>
      </div>

      <div class="gl-row">
        <label class="gl-check">
          <input type="checkbox" id="gl-smooth"> EMA-сглаживание
        </label>
        <label class="gl-check">
          <input type="checkbox" id="gl-dms"> формат DMS
        </label>
      </div>

      <div class="gl-row">
        <button id="gl-refresh-perm">↻ Разрешение</button>
        <button id="gl-reset">↺ Сбросить</button>
      </div>

      <div id="gl-status" class="gl-status">готово</div>
    </div>
  </div>

  <!-- ============ Колонка 2: Позиция + визуализация ============ -->
  <div class="gl-col">
    <div class="gl-panel">
      <h2 class="gl-panel-title">
        <span class="gl-num">2</span>Последняя позиция
      </h2>
      <div class="gl-hint">
        Altitude, heading, speed часто <code>null</code> на десктопе.
        Дистанция — по прямой (haversine), с фильтром шума.
      </div>
      <div id="gl-position" class="gl-coord-grid">
        <span class="k">—</span>
        <span class="v">ещё не определено</span>
      </div>
      <div class="gl-row">
        <button id="gl-copy">📋 Копировать координаты</button>
      </div>
      <div id="gl-map-link"></div>
    </div>

    <div class="gl-panel">
      <h2 class="gl-panel-title">Визуализация</h2>
      <div class="gl-row">
        <span class="gl-field-label">Траектория:</span>
      </div>
      <svg id="gl-traj" class="gl-svg" viewBox="0 0 300 100" height="100"></svg>
      <div class="gl-row">
        <span class="gl-field-label">Точность (м):</span>
      </div>
      <svg id="gl-acc" class="gl-svg" viewBox="0 0 300 60" height="60"></svg>
    </div>
  </div>

  <!-- ============ Колонка 3: Журнал ============ -->
  <div class="gl-col">
    <div class="gl-panel">
      <h2 class="gl-panel-title">Журнал</h2>
      <div id="gl-log" class="gl-log">
        <div class="gl-log-empty">здесь появятся события</div>
      </div>
    </div>
  </div>

</div>

<!-- ============ Диагностика (на всю ширину) ============ -->
<div class="gl-panel gl-diag-panel">
  <h2 class="gl-panel-title">🔍 Диагностика окружения</h2>
  <div class="gl-anno">
    Если позиция не появляется — соберите снимок окружения. Типичные
    причины: не secure context, PERMISSION_DENIED в настройках, GPS
    отключён в ОС, слабый сигнал (PositionUnavailable), истёк timeout.
  </div>
  <div class="gl-row">
    <button id="gl-diag-collect" class="gl-primary">🔍 Собрать диагностику</button>
  </div>
  <div id="gl-diag-out" class="gl-diag-out">
    <div class="gl-log-empty">нажмите «Собрать диагностику»</div>
  </div>
</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info, Warn }

#[derive(Clone, Copy, PartialEq)]
enum TrackingState { Stopped, Running, Paused }

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

struct Ui {
    high_acc: web_sys::HtmlInputElement,
    high_acc_hint: web_sys::Element,
    timeout: web_sys::HtmlInputElement,
    max_age: web_sys::HtmlInputElement,
    smooth: web_sys::HtmlInputElement,
    dms: web_sys::HtmlInputElement,

    btn_get_once: web_sys::HtmlButtonElement,
    btn_get_3x: web_sys::HtmlButtonElement,
    btn_watch_toggle: web_sys::HtmlButtonElement,
    btn_watch_stop: web_sys::HtmlButtonElement,
    btn_refresh_perm: web_sys::HtmlButtonElement,
    btn_reset: web_sys::HtmlButtonElement,
    btn_copy: web_sys::HtmlButtonElement,

    badge_state: web_sys::Element,
    badge_move: web_sys::Element,
    badge_count: web_sys::Element,
    badge_accepted: web_sys::Element,
    badge_errors: web_sys::Element,
    badge_filtered: web_sys::Element,
    badge_session: web_sys::Element,
    badge_age: web_sys::Element,
    badge_perm: web_sys::Element,

    status: web_sys::Element,
    position: web_sys::Element,
    map_link: web_sys::Element,
    traj: web_sys::Element,
    acc: web_sys::Element,
    log: web_sys::Element,

    btn_diag_collect: web_sys::HtmlButtonElement,
    diag_out: web_sys::Element,
}

struct App {
    log: Rc<RefCell<Vec<LogEntry>>>,
    watch_handle: Rc<RefCell<Option<GeoHandle>>>,
    tracking: Rc<Cell<TrackingState>>,

    watch_ok: Rc<Cell<u32>>,
    watch_accepted: Rc<Cell<u32>>,
    watch_err: Rc<Cell<u32>>,
    watch_filtered: Rc<Cell<u32>>,
    /// true — только что стартовал трекинг, первый сэмпл не сравниваем
    /// с позицией от предыдущей сессии / getCurrentPosition.
    watch_first_sample: Rc<Cell<bool>>,
    last_update_ms: Rc<Cell<f64>>,
    session_start_ms: Rc<Cell<f64>>,
    session_final_s: Rc<Cell<u32>>,

    last_position: Rc<RefCell<Option<GeoPosition>>>,
    total_distance_m: Rc<Cell<f64>>,
    stationary_streak: Rc<Cell<u32>>,
    moving: Rc<Cell<bool>>,

    acc_samples: Rc<RefCell<Vec<f64>>>,
    traj_points: Rc<RefCell<Vec<(f64, f64)>>>,

    smooth_prev: Rc<Cell<Option<(f64, f64)>>>,
    last_coords: Rc<Cell<Option<(f64, f64)>>>,

    /// Последний известный статус разрешения — для дедупликации
    /// сообщений в журнале.
    last_permission: Rc<RefCell<Option<String>>>,

    _age_timer: RefCell<Option<Interval>>,
    ui: Ui,
}

// ============================================================
//                    PERMISSIONS
// ============================================================

/// Читает статус разрешения через Permissions API и обновляет badge.
///
/// `silent = true` — не писать в журнал, если статус не изменился
/// с прошлого раза. Используется при visibilitychange и onchange,
/// чтобы избежать спама из одинаковых `permissions.query → granted`.
fn refresh_permission(app: Rc<App>, silent: bool) {
    let el = app.ui.badge_perm.clone();
    let log = app.log.clone();
    let last = app.last_permission.clone();

    spawn_local(async move {
        let Some(s) = geo_core::permission_state().await else {
            el.set_text_content(Some("permissions: n/a"));
            el.set_class_name("gl-badge");
            if !silent {
                push_log_static(
                    &log,
                    LogKind::Info,
                    "Permissions API недоступен для geolocation".to_string(),
                );
            }
            return;
        };

        let short = match s.as_str() {
            "granted" => "permissions: granted",
            "denied" => "permissions: denied",
            "prompt" => "permissions: prompt",
            _ => "permissions: ?",
        };
        el.set_text_content(Some(short));
        el.set_class_name("gl-badge");

        let changed = {
            let mut guard = last.borrow_mut();
            let was = guard.clone();
            *guard = Some(s.clone());
            was.as_deref() != Some(s.as_str())
        };

        if changed {
            push_log_static(
                &log,
                LogKind::Info,
                format_str(format_args!("permissions.query → {s}")),
            );
        }
    });
}

// ============================================================
//                    RUN
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        high_acc: get_input("gl-high-acc")?,
        high_acc_hint: get_el("gl-high-acc-hint")?,
        timeout: get_input("gl-timeout")?,
        max_age: get_input("gl-maxage")?,
        smooth: get_input("gl-smooth")?,
        dms: get_input("gl-dms")?,

        btn_get_once: get_button("gl-get-once")?,
        btn_get_3x: get_button("gl-get-3x")?,
        btn_watch_toggle: get_button("gl-watch-toggle")?,
        btn_watch_stop: get_button("gl-watch-stop")?,
        btn_refresh_perm: get_button("gl-refresh-perm")?,
        btn_reset: get_button("gl-reset")?,
        btn_copy: get_button("gl-copy")?,

        badge_state: get_el("gl-badge-state")?,
        badge_move: get_el("gl-badge-move")?,
        badge_count: get_el("gl-badge-count")?,
        badge_accepted: get_el("gl-badge-accepted")?,
        badge_errors: get_el("gl-badge-errors")?,
        badge_filtered: get_el("gl-badge-filtered")?,
        badge_session: get_el("gl-badge-session")?,
        badge_age: get_el("gl-badge-age")?,
        badge_perm: get_el("gl-badge-perm")?,

        status: get_el("gl-status")?,
        position: get_el("gl-position")?,
        map_link: get_el("gl-map-link")?,
        traj: get_el("gl-traj")?,
        acc: get_el("gl-acc")?,
        log: get_el("gl-log")?,

        btn_diag_collect: get_button("gl-diag-collect")?,
        diag_out: get_el("gl-diag-out")?,
    };

    let app = Rc::new(App {
        log: Rc::new(RefCell::new(Vec::new())),
        watch_handle: Rc::new(RefCell::new(None)),
        tracking: Rc::new(Cell::new(TrackingState::Stopped)),

        watch_ok: Rc::new(Cell::new(0)),
        watch_accepted: Rc::new(Cell::new(0)),
        watch_err: Rc::new(Cell::new(0)),
        watch_filtered: Rc::new(Cell::new(0)),
        watch_first_sample: Rc::new(Cell::new(false)),
        last_update_ms: Rc::new(Cell::new(0.0)),
        session_start_ms: Rc::new(Cell::new(0.0)),
        session_final_s: Rc::new(Cell::new(0)),

        last_position: Rc::new(RefCell::new(None)),
        total_distance_m: Rc::new(Cell::new(0.0)),
        stationary_streak: Rc::new(Cell::new(0)),
        moving: Rc::new(Cell::new(false)),

        acc_samples: Rc::new(RefCell::new(Vec::new())),
        traj_points: Rc::new(RefCell::new(Vec::new())),

        smooth_prev: Rc::new(Cell::new(None)),
        last_coords: Rc::new(Cell::new(None)),

        last_permission: Rc::new(RefCell::new(None)),

        _age_timer: RefCell::new(None),
        ui,
    });

    app.render_log();
    app.update_badges();

    if !geo_core::is_supported() {
        app.push_log(
            LogKind::Err,
            "Geolocation API недоступен — нужен secure context (HTTPS или localhost)"
                .to_string(),
        );
        set_status(&app.ui.status, "err", "Geolocation API недоступен");
        app.ui.btn_get_once.set_disabled(true);
        app.ui.btn_get_3x.set_disabled(true);
        app.ui.btn_watch_toggle.set_disabled(true);
        return Ok(());
    }
    app.push_log(LogKind::Info, "Geolocation API доступен".to_string());

    // Таймер: обновляет badge_age и badge_session раз в секунду.
    // Цвет age-бейджа отражает молчание трекинга: жёлтый с 5 сек,
    // красный с 15 — сигнал, что сэмплы не приходят.
    {
        let tracking = app.tracking.clone();
        let last_update = app.last_update_ms.clone();
        let session_start = app.session_start_ms.clone();
        let session_final = app.session_final_s.clone();
        let el_age = app.ui.badge_age.clone();
        let el_session = app.ui.badge_session.clone();
        let interval = Interval::new(1000, move || {
            let state = tracking.get();

            if state != TrackingState::Running {
                el_age.set_text_content(Some("—"));
                el_age.set_class_name("gl-badge");
            } else {
                let last = last_update.get();
                if last <= 0.0 {
                    el_age.set_text_content(Some("ждём fix…"));
                    el_age.set_class_name("gl-badge gl-badge--warn");
                } else {
                    let age_s = ((js_sys::Date::now() - last) / 1000.0).floor() as i64;
                    let text = format_str(format_args!("{age_s}с"));
                    el_age.set_text_content(Some(&text));
                    let cls = if age_s >= AGE_ERR_S {
                        "gl-badge gl-badge--err"
                    } else if age_s >= AGE_WARN_S {
                        "gl-badge gl-badge--warn"
                    } else {
                        "gl-badge"
                    };
                    el_age.set_class_name(cls);
                }
            }

            if state == TrackingState::Stopped {
                let s_total = session_final.get();
                let m = s_total / 60;
                let s = s_total % 60;
                let text = format_str(format_args!("{m}:{s:02}"));
                el_session.set_text_content(Some(&text));
            } else {
                let start = session_start.get();
                if start <= 0.0 {
                    el_session.set_text_content(Some("0:00"));
                } else {
                    let total_s =
                        ((js_sys::Date::now() - start) / 1000.0).floor() as i64;
                    let m = total_s / 60;
                    let s = total_s % 60;
                    let text = format_str(format_args!("{m}:{s:02}"));
                    el_session.set_text_content(Some(&text));
                }
            }
        });
        *app._age_timer.borrow_mut() = Some(interval);
    }

    // Initial permission check — с логированием.
    refresh_permission(app.clone(), false);

    // visibilitychange → refresh только при возврате в видимость.
    //
    // web-sys не отдаёт `visibility_state()` без фичи "VisibilityState",
    // поэтому читаем строку через Reflect: "visible" | "hidden" | "prerender".
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
            let visible = web_sys::window()
                .and_then(|w| w.document())
                .and_then(|d| {
                    js_sys::Reflect::get(&d, &JsValue::from_str("visibilityState"))
                        .ok()
                        .and_then(|v| v.as_string())
                })
                .map(|s| s == "visible")
                .unwrap_or(false);
            if visible {
                // silent = true: логируем только при реальной смене.
                refresh_permission(app_c.clone(), true);
            }
        });
        if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
            doc.add_event_listener_with_callback(
                "visibilitychange",
                cb.as_ref().unchecked_ref(),
            )?;
        }
        cb.forget();
    }

    // Подписка на permission.onchange — не логирует стартовое значение.
    {
        let app_c = app.clone();
        spawn_local(async move {
            subscribe_permission_change(app_c).await;
        });
    }

    // Hint для enableHighAccuracy.
    {
        let hint = app.ui.high_acc_hint.clone();
        let el = app.ui.high_acc.clone();
        let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
            let cls = if el.checked() {
                "gl-hint"
            } else {
                "gl-hint gl-hint--hidden"
            };
            hint.set_class_name(cls);
        });
        app.ui.high_acc.add_event_listener_with_callback(
            "change",
            cb.as_ref().unchecked_ref(),
        )?;
        cb.forget();
    }

    bind_buttons(&app)?;

    return Ok(());
}

// ============================================================
//                    BIND BUTTONS
// ============================================================

fn bind_buttons(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_get_once, move |_| {
            do_single_query(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_get_3x, move |_| {
            do_multi_query(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_watch_toggle, move |_| {
            let app_in = app_c.clone();
            match app_in.tracking.get() {
                TrackingState::Stopped => start_tracking(app_in),
                TrackingState::Running => pause_tracking(app_in),
                TrackingState::Paused => resume_tracking(app_in),
            }
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_watch_stop, move |_| {
            stop_tracking(&app_c.clone(), true);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_refresh_perm, move |_| {
            refresh_permission(app_c.clone(), false);
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_copy, move |_| {
            copy_coordinates(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_reset, move |_| {
            do_reset(app_c.clone());
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.btn_diag_collect, move |_| {
            collect_diagnostics(app_c.clone());
        });
    }
    return Ok(());
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

fn do_single_query(app: Rc<App>) {
    let opts = read_options(&app);
    set_status(&app.ui.status, "busy", "запрос позиции…");
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "getCurrentPosition (high_acc={}, timeout={}мс, maxAge={}мс)",
            opts.high_accuracy, opts.timeout_ms, opts.max_age_ms,
        )),
    );

    let app_ok = app.clone();
    let app_err = app.clone();
    let r = geo_core::get_current_position(
        &opts,
        move |pos| { on_position(app_ok.clone(), pos, false); },
        move |err| { on_error(app_err.clone(), err, false); },
    );
    if let Err(e) = r {
        let msg = format_str(format_args!("getCurrentPosition: {e:?}"));
        app.push_log(LogKind::Err, msg.clone());
        set_status(&app.ui.status, "err", &msg);
    }
}

fn do_multi_query(app: Rc<App>) {
    set_status(&app.ui.status, "busy", "3 запроса позиции…");
    app.push_log(
        LogKind::Info,
        "3× getCurrentPosition параллельно".to_string(),
    );

    for _ in 1..=3 {
        let opts = read_options(&app);
        let app_ok = app.clone();
        let app_err = app.clone();
        let r = geo_core::get_current_position(
            &opts,
            move |pos| { on_position(app_ok.clone(), pos, false); },
            move |err| { on_error(app_err.clone(), err, false); },
        );
        if let Err(e) = r {
            let msg = format_str(format_args!("getCurrentPosition: {e:?}"));
            app.push_log(LogKind::Err, msg);
        }
    }
}

fn start_tracking(app: Rc<App>) {
    if app.watch_handle.borrow().is_some() {
        return;
    }
    let opts = read_options(&app);
    set_status(&app.ui.status, "busy", "запуск трекинга…");
    app.push_log(
        LogKind::Info,
        format_str(format_args!(
            "watchPosition (high_acc={}, timeout={}мс)",
            opts.high_accuracy, opts.timeout_ms,
        )),
    );

    app.watch_ok.set(0);
    app.watch_accepted.set(0);
    app.watch_err.set(0);
    app.watch_filtered.set(0);
    app.watch_first_sample.set(true);
    app.last_update_ms.set(0.0);
    app.session_start_ms.set(js_sys::Date::now());
    app.session_final_s.set(0);
    app.total_distance_m.set(0.0);
    app.stationary_streak.set(0);
    app.moving.set(false);
    app.acc_samples.borrow_mut().clear();
    app.traj_points.borrow_mut().clear();
    app.smooth_prev.set(None);
    app.update_badges();

    launch_watch(app.clone(), opts);
}

fn resume_tracking(app: Rc<App>) {
    let opts = read_options(&app);
    // Первый сэмпл после resume тоже не сравниваем с прошлым —
    // прошлая точка может быть на большом расстоянии.
    app.watch_first_sample.set(true);
    launch_watch(app.clone(), opts);
}

fn launch_watch(app: Rc<App>, opts: GeoOptions) {
    let app_ok = app.clone();
    let app_err = app.clone();
    let r = geo_core::watch_position(
        &opts,
        move |pos| { on_position(app_ok.clone(), pos, true); },
        move |err| { on_error(app_err.clone(), err, true); },
    );
    match r {
        Ok(handle) => {
            let log_msg = format_str(format_args!(
                "трекинг запущен (watch_id = {})",
                handle.watch_id(),
            ));
            app.push_log(LogKind::Ok, log_msg);
            set_status(&app.ui.status, "busy", "ожидание первой позиции…");
            *app.watch_handle.borrow_mut() = Some(handle);
            app.tracking.set(TrackingState::Running);
            update_toggle_button(&app);
            app.ui.btn_watch_stop.set_disabled(false);
        }
        Err(e) => {
            let msg = format_str(format_args!("watchPosition: {e:?}"));
            app.push_log(LogKind::Err, msg.clone());
            set_status(&app.ui.status, "err", &msg);
        }
    }
}

fn pause_tracking(app: Rc<App>) {
    let handle = app.watch_handle.borrow_mut().take();
    if let Some(h) = handle {
        let id = h.watch_id();
        drop(h);
        let _ = geo_core::clear_watch(id);
    }
    app.tracking.set(TrackingState::Paused);
    set_status(&app.ui.status, "busy", "пауза — данные не поступают");
    app.push_log(LogKind::Info, "трекинг поставлен на паузу".to_string());
    update_toggle_button(&app);
    app.update_badges();
}

fn stop_tracking(app: &Rc<App>, log_event: bool) {
    // Фиксируем финальное время до сброса session_start.
    let start = app.session_start_ms.get();
    if start > 0.0 {
        let s = ((js_sys::Date::now() - start) / 1000.0).floor() as u32;
        app.session_final_s.set(s);
    }

    let handle = app.watch_handle.borrow_mut().take();
    if let Some(h) = handle {
        let id = h.watch_id();
        drop(h);
        let _ = geo_core::clear_watch(id);
        if log_event {
            let msg = format_str(format_args!(
                "трекинг остановлен (watch_id = {id}, smp: {}, acc: {}, flt: {}, err: {})",
                app.watch_ok.get(),
                app.watch_accepted.get(),
                app.watch_filtered.get(),
                app.watch_err.get(),
            ));
            app.push_log(LogKind::Info, msg.clone());
            set_status(&app.ui.status, "ok", &msg);
        }
    } else if log_event {
        set_status(&app.ui.status, "busy", "трекинг не запущен");
    }
    app.tracking.set(TrackingState::Stopped);
    app.session_start_ms.set(0.0);
    update_toggle_button(app);
    app.ui.btn_watch_stop.set_disabled(true);
    app.update_badges();
}

fn do_reset(app: Rc<App>) {
    stop_tracking(&app, false);

    app.watch_ok.set(0);
    app.watch_accepted.set(0);
    app.watch_err.set(0);
    app.watch_filtered.set(0);
    app.watch_first_sample.set(false);
    app.last_update_ms.set(0.0);
    app.session_start_ms.set(0.0);
    app.session_final_s.set(0);
    app.total_distance_m.set(0.0);
    app.stationary_streak.set(0);
    app.moving.set(false);
    *app.last_position.borrow_mut() = None;
    app.acc_samples.borrow_mut().clear();
    app.traj_points.borrow_mut().clear();
    app.smooth_prev.set(None);
    app.last_coords.set(None);

    app.update_badges();
    app.ui.position.set_inner_html(
        "<span class=\"k\">—</span><span class=\"v\">ещё не определено</span>",
    );
    app.ui.map_link.set_inner_html("");
    render_trajectory(&app.ui.traj, &[]);
    render_accuracy(&app.ui.acc, &[]);

    app.log.borrow_mut().clear();
    app.render_log();

    set_status(&app.ui.status, "ok", "сброшено");
}

fn copy_coordinates(app: Rc<App>) {
    let coords = app.last_coords.get();
    let Some((lat, lon)) = coords else {
        app.push_log(
            LogKind::Warn,
            "нет координат для копирования".to_string(),
        );
        return;
    };
    let text = format_str(format_args!("{lat:.6}, {lon:.6}"));
    let nav = match web_sys::window() {
        Some(w) => w.navigator(),
        None => return,
    };
    let Ok(clipboard) =
        js_sys::Reflect::get(&nav, &JsValue::from_str("clipboard"))
    else {
        app.push_log(LogKind::Err, "Clipboard API недоступен".to_string());
        return;
    };
    if clipboard.is_undefined() || clipboard.is_null() {
        app.push_log(LogKind::Err, "Clipboard API недоступен".to_string());
        return;
    }
    let Ok(write_fn) =
        js_sys::Reflect::get(&clipboard, &JsValue::from_str("writeText"))
            .and_then(|v| v.dyn_into::<js_sys::Function>())
    else {
        app.push_log(
            LogKind::Err,
            "clipboard.writeText не найден".to_string(),
        );
        return;
    };
    let _ = write_fn.call1(&clipboard, &JsValue::from_str(&text));

    // Подтверждение — только в журнал. Статус трекинга не трогаем:
    // «скопировано» — это мелочь, важный статус не должен пропадать.
    app.push_log(
        LogKind::Ok,
        format_str(format_args!("скопировано в буфер: {text}")),
    );
}

// ============================================================
//                    POSITION HANDLER
// ============================================================

fn on_position(app: Rc<App>, pos: GeoPosition, is_watch: bool) {
    let display_coords = apply_smoothing(&app, &pos);

    // Порядок важен: первый сэмпл трекинга не сравниваем с prev,
    // даже если в last_position лежит результат getCurrentPosition.
    let first_in_session = is_watch && app.watch_first_sample.get();
    if first_in_session {
        app.watch_first_sample.set(false);
    }
    let prev: Option<GeoPosition> = if first_in_session {
        None
    } else {
        app.last_position.borrow().clone()
    };

    let mut distance_from_prev: Option<f64> = None;
    let mut bearing: Option<f64> = None;
    let mut sample_speed: Option<f64> = None;
    let mut filtered = false;

    if is_watch {
        if let Some(prev_pos) = &prev {
            let d = geo_core::haversine_m(
                prev_pos.latitude, prev_pos.longitude,
                pos.latitude, pos.longitude,
            );
            let dt = (pos.timestamp_ms - prev_pos.timestamp_ms) / 1000.0;

            if d < pos.accuracy_m {
                // Расстояние меньше погрешности — вероятно, дрожание
                // координат. Не накапливаем дистанцию.
                filtered = true;
                app.watch_filtered.set(app.watch_filtered.get() + 1);
            } else {
                app.watch_accepted.set(app.watch_accepted.get() + 1);
                distance_from_prev = Some(d);
                app.total_distance_m.set(app.total_distance_m.get() + d);
                bearing = Some(geo_core::bearing_deg(
                    prev_pos.latitude, prev_pos.longitude,
                    pos.latitude, pos.longitude,
                ));
                if dt > 0.1 {
                    let sp = d / dt;
                    if sp < SPEED_ANOMALY_MPS {
                        sample_speed = Some(sp);
                    } else {
                        app.push_log(
                            LogKind::Warn,
                            format_str(format_args!(
                                "аномалия: расчётная скорость {:.0} км/ч — \
                                 отбрасываем сэмпл",
                                sp * 3.6,
                            )),
                        );
                    }
                }
            }
        }

        update_moving_status(&app, distance_from_prev, pos.accuracy_m);

        let n = app.watch_ok.get() + 1;
        app.watch_ok.set(n);
        app.last_update_ms.set(js_sys::Date::now());

        if n == 1 && pos.accuracy_m > ACCURACY_WARN_THRESHOLD_M {
            app.push_log(
                LogKind::Warn,
                format_str(format_args!(
                    "точность ±{:.0}м > {}м — дельта может быть шумом",
                    pos.accuracy_m, ACCURACY_WARN_THRESHOLD_M as i64,
                )),
            );
        }
        if n == 1 {
            set_status(
                &app.ui.status,
                "ok",
                "трекинг активен — позиции поступают",
            );
        }

        app.acc_samples.borrow_mut().push(pos.accuracy_m);
        // В траекторию идёт только первый сэмпл и принятые (не шум).
        // Иначе при стоянии на месте SVG рисует зигзаг из шума и
        // выглядит, как будто пользователь движется.
        let traj_len = app.traj_points.borrow().len();
        if traj_len == 0 || !filtered {
            app.traj_points
                .borrow_mut()
                .push((pos.latitude, pos.longitude));
        }
    }

    *app.last_position.borrow_mut() = Some(pos.clone());
    app.last_coords.set(Some((pos.latitude, pos.longitude)));
    app.update_badges();

    render_position(
        &app.ui.position,
        &app.ui.map_link,
        &pos,
        &read_format(&app),
        distance_from_prev,
        app.total_distance_m.get(),
        bearing,
        sample_speed,
        filtered,
        app.watch_filtered.get(),
    );

    render_trajectory(&app.ui.traj, &app.traj_points.borrow());
    render_accuracy(&app.ui.acc, &app.acc_samples.borrow());

    let warm_marker = if is_watch && app.watch_ok.get() <= WARMUP_SAMPLES {
        " [warm-up]"
    } else {
        ""
    };
    let kind = if is_watch && app.watch_ok.get() <= WARMUP_SAMPLES {
        LogKind::Warn
    } else {
        LogKind::Ok
    };
    let msg = if is_watch {
        format_str(format_args!(
            "позиция #{}: {:.5}, {:.5} (±{:.0}м, {}){warm_marker}",
            app.watch_ok.get(),
            pos.latitude, pos.longitude,
            pos.accuracy_m, pos.accuracy_label(),
        ))
    } else {
        format_str(format_args!(
            "позиция: {:.5}, {:.5} (±{:.0}м, {})",
            pos.latitude, pos.longitude,
            pos.accuracy_m, pos.accuracy_label(),
        ))
    };
    app.push_log(kind, msg.clone());

    if !is_watch {
        set_status(&app.ui.status, "ok", &msg);
    }

    if let Some((lat, lon)) = display_coords {
        app.smooth_prev.set(Some((lat, lon)));
    }
}

fn apply_smoothing(app: &Rc<App>, pos: &GeoPosition) -> Option<(f64, f64)> {
    if !app.ui.smooth.checked() {
        return None;
    }
    return match app.smooth_prev.get() {
        Some((plat, plon)) => {
            let nlat = geo_core::smooth_ema(plat, pos.latitude, SMOOTH_ALPHA);
            let nlon = geo_core::smooth_ema(plon, pos.longitude, SMOOTH_ALPHA);
            Some((nlat, nlon))
        }
        None => Some((pos.latitude, pos.longitude)),
    };
}

fn update_moving_status(
    app: &Rc<App>,
    distance_from_prev: Option<f64>,
    accuracy_m: f64,
) {
    let is_moving_now = match distance_from_prev {
        Some(d) => d > 2.0 * accuracy_m,
        None => false,
    };
    if is_moving_now {
        app.stationary_streak.set(0);
        app.moving.set(true);
    } else {
        let streak = app.stationary_streak.get() + 1;
        app.stationary_streak.set(streak);
        if streak >= 5 {
            app.moving.set(false);
        }
    }
}

fn on_error(app: Rc<App>, err: GeoError, is_watch: bool) {
    let recoverable = err.kind.is_recoverable();
    let tracking = is_watch && app.tracking.get() == TrackingState::Running;

    if is_watch {
        app.watch_err.set(app.watch_err.get() + 1);
        app.update_badges();
    }

    let kind = if recoverable && tracking {
        LogKind::Warn
    } else {
        LogKind::Err
    };
    app.push_log(kind, format_str(format_args!("ошибка: {}", err.message)));

    if !recoverable {
        stop_tracking(&app, false);
        app.push_log(LogKind::Info, err.user_hint().to_string());
        set_status(&app.ui.status, "err", err.user_hint());
        return;
    }

    if tracking {
        app.push_log(
            LogKind::Info,
            "трекинг продолжается — браузер повторит попытку".to_string(),
        );
        set_status(
            &app.ui.status,
            "busy",
            "ожидание позиции — браузер продолжит попытки",
        );
    } else {
        app.push_log(LogKind::Info, err.user_hint().to_string());
        set_status(&app.ui.status, "err", err.user_hint());
    }
}

// ============================================================
//                    PERMISSION ONCHANGE
// ============================================================

async fn subscribe_permission_change(app: Rc<App>) {
    use wasm_bindgen_futures::JsFuture;

    let global = js_sys::global();
    let Ok(nav) = js_sys::Reflect::get(&global, &JsValue::from_str("navigator")) else {
        return;
    };
    let Ok(permissions) = js_sys::Reflect::get(&nav, &JsValue::from_str("permissions"))
    else {
        return;
    };
    if permissions.is_undefined() || permissions.is_null() {
        return;
    }
    let Ok(query_fn) = js_sys::Reflect::get(&permissions, &JsValue::from_str("query")) else {
        return;
    };
    let Ok(query_fn) = query_fn.dyn_into::<js_sys::Function>() else {
        return;
    };
    let desc = js_sys::Object::new();
    let _ = js_sys::Reflect::set(
        &desc,
        &JsValue::from_str("name"),
        &JsValue::from_str("geolocation"),
    );
    let Ok(promise) = query_fn.call1(&permissions, &desc) else {
        return;
    };
    let Ok(promise) = promise.dyn_into::<js_sys::Promise>() else {
        return;
    };
    let Ok(status) = JsFuture::from(promise).await else {
        return;
    };

    // Читаем начальное значение и запоминаем — первое событие onchange
    // в Firefox приходит сразу после подписки, его логировать не нужно.
    if let Ok(init_state) =
        js_sys::Reflect::get(&status, &JsValue::from_str("state"))
            .and_then(|v| v.dyn_into::<js_sys::JsString>())
    {
        *app.last_permission.borrow_mut() = Some(init_state.as_string().unwrap_or_default());
    }

    let app_c = app.clone();
    let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
        let app_in = app_c.clone();
        spawn_local(async move {
            let Some(s) = geo_core::permission_state().await else {
                return;
            };
            // Дедупликация: логируем только реальное изменение.
            let changed = {
                let mut guard = app_in.last_permission.borrow_mut();
                let was = guard.clone();
                *guard = Some(s.clone());
                was.as_deref() != Some(s.as_str())
            };
            if !changed {
                return;
            }
            app_in.push_log(
                LogKind::Info,
                format_str(format_args!("permission изменился → {s}")),
            );
            if s == "denied" && app_in.tracking.get() != TrackingState::Stopped {
                stop_tracking(&app_in, false);
                app_in.push_log(
                    LogKind::Warn,
                    "трекинг остановлен: разрешение отозвано".to_string(),
                );
            }
        });
    });

    let _ = js_sys::Reflect::set(
        &status,
        &JsValue::from_str("onchange"),
        cb.as_ref().unchecked_ref(),
    );
    cb.forget();
}

// ============================================================
//                    DIAGNOSTICS
// ============================================================

fn collect_diagnostics(app: Rc<App>) {
    app.ui.diag_out.set_inner_html(
        "<div class=\"gl-log-empty\">сбор…</div>",
    );
    app.push_log(LogKind::Info, "сбор диагностики…".to_string());
    spawn_local(async move {
        let mut rows: Vec<(String, String)> = Vec::new();
        let global = js_sys::global();

        let is_secure = js_sys::Reflect::get(
            &global,
            &JsValue::from_str("isSecureContext"),
        )
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
        rows.push(("isSecureContext".into(), yes_no(is_secure)));
        rows.push(("Geolocation API".into(), yes_no(geo_core::is_supported())));

        let perm = geo_core::permission_state()
            .await
            .unwrap_or_else(|| "n/a".into());
        rows.push(("Permissions API (geolocation)".into(), perm));

        if let Ok(nav) = js_sys::Reflect::get(&global, &JsValue::from_str("navigator")) {
            if let Ok(ua) = js_sys::Reflect::get(&nav, &JsValue::from_str("userAgent")) {
                if let Some(s) = ua.as_string() {
                    rows.push(("userAgent".into(), s));
                }
            }
        }
        if let Ok(doc) = js_sys::Reflect::get(&global, &JsValue::from_str("document")) {
            if let Ok(vis) =
                js_sys::Reflect::get(&doc, &JsValue::from_str("visibilityState"))
            {
                if let Some(s) = vis.as_string() {
                    rows.push(("document.visibilityState".into(), s));
                }
            }
            let has_focus = js_sys::Reflect::get(&doc, &JsValue::from_str("hasFocus"))
                .ok()
                .and_then(|f| f.dyn_into::<js_sys::Function>().ok())
                .and_then(|f| f.call0(&doc).ok())
                .and_then(|v| v.as_bool())
                .unwrap_or(false);
            rows.push(("document.hasFocus()".into(), yes_no(has_focus)));
        }

        let mut html = String::with_capacity(1024);
        for (k, v) in rows.iter() {
            html.push_str("<div class=\"gl-diag-row\"><span class=\"k\">");
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

// ============================================================
//                    РЕНДЕР
// ============================================================

#[allow(clippy::too_many_arguments)]
fn render_position(
    el: &web_sys::Element,
    map_link: &web_sys::Element,
    pos: &GeoPosition,
    fmt: &GeoFormat,
    distance_from_prev: Option<f64>,
    total_distance_m: f64,
    bearing: Option<f64>,
    sample_speed: Option<f64>,
    filtered: bool,
    filtered_total: u32,
) {
    let mut html = String::with_capacity(768);
    push_coord(&mut html, "Широта", &fmt.format_lat(pos.latitude));
    push_coord(&mut html, "Долгота", &fmt.format_lon(pos.longitude));
    push_coord(
        &mut html,
        "Точность",
        &format_str(format_args!(
            "±{:.0} м ({})",
            pos.accuracy_m,
            pos.accuracy_label(),
        )),
    );
    if let Some(alt) = pos.altitude_m {
        push_coord(&mut html, "Высота", &format_str(format_args!("{alt:.1} м")));
    }
    if let Some(h) = pos.heading_deg {
        push_coord(
            &mut html,
            "heading (GPS)",
            &format_str(format_args!("{h:.1}°")),
        );
    }
    if let Some(b) = bearing {
        let d = compass_dir(b);
        push_coord(
            &mut html,
            "bearing (расчёт)",
            &format_str(format_args!("{b:.1}° {d}")),
        );
    }
    if let Some(s) = pos.speed_mps {
        push_coord(
            &mut html,
            "speed (GPS)",
            &format_str(format_args!("{s:.2} м/с = {:.1} км/ч", s * 3.6)),
        );
    }
    if let Some(s) = sample_speed {
        push_coord(
            &mut html,
            "speed (расчёт)",
            &format_str(format_args!("{s:.2} м/с = {:.1} км/ч", s * 3.6)),
        );
    }
    if let Some(d) = distance_from_prev {
        push_coord(&mut html, "От пред. точки", &format_distance(d));
    } else if filtered {
        push_coord(&mut html, "От пред. точки", "отфильтровано (шум)");
    }
    if filtered_total > 0 {
        push_coord(
            &mut html,
            "Отфильтровано всего",
            &format_str(format_args!("{filtered_total} сэмплов")),
        );
    }
    push_coord(
        &mut html,
        "Всего пройдено",
        &if total_distance_m < 1.0 {
            "0 м (стою)".to_string()
        } else {
            format_distance(total_distance_m)
        },
    );
    el.set_inner_html(&html);

    let mut link = String::with_capacity(192);
    link.push_str("<a class=\"gl-map-link\" href=\"");
    link.push_str(&format_str(format_args!(
        "https://www.openstreetmap.org/?mlat={}&mlon={}&zoom=15",
        pos.latitude, pos.longitude,
    )));
    link.push_str("\" target=\"_blank\" rel=\"noopener\">");
    link.push_str("Открыть на OpenStreetMap →");
    link.push_str("</a>");
    map_link.set_inner_html(&link);
}

fn push_coord(html: &mut String, key: &str, value: &str) {
    html.push_str("<span class=\"k\">");
    html.push_str(&esc_html(key));
    html.push_str("</span><span class=\"v\">");
    html.push_str(&esc_html(value));
    html.push_str("</span>");
}

fn compass_dir(bearing_deg: f64) -> &'static str {
    let dirs = ["С", "СВ", "В", "ЮВ", "Ю", "ЮЗ", "З", "СЗ"];
    let idx = (((bearing_deg + 22.5) / 45.0) as i32).rem_euclid(8) as usize;
    return dirs[idx];
}

fn render_trajectory(el: &web_sys::Element, points: &[(f64, f64)]) {
    if points.is_empty() {
        el.set_inner_html("");
        return;
    }

    const W: f64 = 300.0;
    const H: f64 = 100.0;
    const PAD: f64 = 8.0;

    // Одиночная точка — рисуем её в центре и подписываем координаты,
    // чтобы было видно, что трек начался (а не «ничего не работает»).
    if points.len() == 1 {
        let (lat, lon) = points[0];
        let mut svg = String::with_capacity(256);
        svg.push_str("<circle cx=\"150\" cy=\"42\" r=\"6\" fill=\"#c0392b\"/>");
        svg.push_str("<circle cx=\"150\" cy=\"42\" r=\"10\" fill=\"none\" stroke=\"#c0392b\" stroke-width=\"1\" opacity=\"0.4\"/>");
        svg.push_str("<text x=\"150\" y=\"75\" text-anchor=\"middle\" font-size=\"10\" fill=\"#666\">");
        svg.push_str(&format_str(format_args!("{:.5}, {:.5}", lat, lon)));
        svg.push_str("</text>");
        el.set_inner_html(&svg);
        return;
    }

    let mut min_lat = f64::INFINITY;
    let mut max_lat = f64::NEG_INFINITY;
    let mut min_lon = f64::INFINITY;
    let mut max_lon = f64::NEG_INFINITY;
    for (lat, lon) in points {
        if *lat < min_lat { min_lat = *lat; }
        if *lat > max_lat { max_lat = *lat; }
        if *lon < min_lon { min_lon = *lon; }
        if *lon > max_lon { max_lon = *lon; }
    }
    let dlat = (max_lat - min_lat).max(1e-6);
    let dlon = (max_lon - min_lon).max(1e-6);

    let mut svg = String::with_capacity(1024);
    svg.push_str("<rect width=\"300\" height=\"100\" fill=\"transparent\"/>");

    svg.push_str("<polyline fill=\"none\" stroke=\"#3498db\" stroke-width=\"2\" points=\"");
    for (i, (lat, lon)) in points.iter().enumerate() {
        let x = PAD + ((lon - min_lon) / dlon) * (W - 2.0 * PAD);
        let y = H - PAD - ((lat - min_lat) / dlat) * (H - 2.0 * PAD);
        if i > 0 { svg.push(' '); }
        svg.push_str(&format_str(format_args!("{:.1},{:.1}", x, y)));
    }
    svg.push_str("\"/>");

    for (lat, lon) in points.iter().rev().take(30) {
        let x = PAD + ((lon - min_lon) / dlon) * (W - 2.0 * PAD);
        let y = H - PAD - ((lat - min_lat) / dlat) * (H - 2.0 * PAD);
        svg.push_str("<circle cx=\"");
        svg.push_str(&format_str(format_args!("{:.1}", x)));
        svg.push_str("\" cy=\"");
        svg.push_str(&format_str(format_args!("{:.1}", y)));
        svg.push_str("\" r=\"2\" fill=\"#3498db\"/>");
    }

    if let Some((lat, lon)) = points.last() {
        let x = PAD + ((lon - min_lon) / dlon) * (W - 2.0 * PAD);
        let y = H - PAD - ((lat - min_lat) / dlat) * (H - 2.0 * PAD);
        svg.push_str("<circle cx=\"");
        svg.push_str(&format_str(format_args!("{:.1}", x)));
        svg.push_str("\" cy=\"");
        svg.push_str(&format_str(format_args!("{:.1}", y)));
        svg.push_str("\" r=\"4\" fill=\"#c0392b\"/>");
    }

    el.set_inner_html(&svg);
}

fn render_accuracy(el: &web_sys::Element, values: &[f64]) {
    if values.is_empty() {
        el.set_inner_html("");
        return;
    }
    const W: f64 = 300.0;
    const H: f64 = 60.0;

    let max_v = values.iter().cloned().fold(0.0_f64, f64::max).max(50.0);

    let step = if values.len() > 1 {
        W / (values.len() - 1) as f64
    } else {
        0.0
    };
    let x_single = if values.len() == 1 { W / 2.0 } else { 0.0 };

    let mut svg = String::with_capacity(1024);

    svg.push_str("<line x1=\"0\" y1=\"");
    svg.push_str(&format_str(format_args!("{:.1}", H)));
    svg.push_str("\" x2=\"");
    svg.push_str(&format_str(format_args!("{:.1}", W)));
    svg.push_str("\" y2=\"");
    svg.push_str(&format_str(format_args!("{:.1}", H)));
    svg.push_str("\" stroke=\"#ccc\" stroke-width=\"1\"/>");

    if values.len() == 1 {
        let v = values[0];
        let y = H - (v / max_v * H);
        svg.push_str("<circle cx=\"");
        svg.push_str(&format_str(format_args!("{:.1}", x_single)));
        svg.push_str("\" cy=\"");
        svg.push_str(&format_str(format_args!("{:.1}", y)));
        svg.push_str("\" r=\"4\" fill=\"#e67e22\"/>");
    } else {
        svg.push_str("<polyline fill=\"none\" stroke=\"#e67e22\" stroke-width=\"2\" points=\"");
        for (i, v) in values.iter().enumerate() {
            let x = i as f64 * step;
            let y = H - (v / max_v * H);
            if i > 0 { svg.push(' '); }
            svg.push_str(&format_str(format_args!("{:.1},{:.1}", x, y)));
        }
        svg.push_str("\"/>");
    }

    el.set_inner_html(&svg);
}

fn format_distance(m: f64) -> String {
    return if m < 1000.0 {
        format_str(format_args!("{m:.0} м"))
    } else {
        format_str(format_args!("{:.2} км", m / 1000.0))
    };
}

fn read_format(app: &Rc<App>) -> GeoFormat {
    return if app.ui.dms.checked() {
        GeoFormat::Dms
    } else {
        GeoFormat::Decimal
    };
}

fn yes_no(b: bool) -> String {
    return if b { "yes".into() } else { "no".into() };
}

// ============================================================
//                    READ OPTIONS
// ============================================================

fn read_options(app: &Rc<App>) -> GeoOptions {
    let high_accuracy = app.ui.high_acc.checked();
    let timeout_ms = app
        .ui
        .timeout
        .value()
        .parse::<u32>()
        .unwrap_or(12000)
        .clamp(1000, 60_000);
    let max_age_ms = app.ui.max_age.value().parse::<u32>().unwrap_or(0);
    return GeoOptions {
        high_accuracy,
        timeout_ms,
        max_age_ms,
    };
}

// ============================================================
//                    BUTTON STATE
// ============================================================

fn update_toggle_button(app: &Rc<App>) {
    let (label, cls) = match app.tracking.get() {
        TrackingState::Stopped => ("▶ Начать трекинг", "gl-success"),
        TrackingState::Running => ("❚❚ Пауза", "gl-primary"),
        TrackingState::Paused => ("▶ Продолжить", "gl-success"),
    };
    app.ui.btn_watch_toggle.set_text_content(Some(label));
    app.ui.btn_watch_toggle.set_class_name(cls);
}

// ============================================================
//                    UI HELPERS
// ============================================================

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "gl-status ok",
        "err" => "gl-status err",
        "busy" => "gl-status busy",
        _ => "gl-status",
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
                "<div class=\"gl-log-empty\">здесь появятся события</div>",
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
            html.push_str("<div class=\"gl-log-entry ");
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

    fn update_badges(&self) {
        // Состояние трекинга.
        let state = self.tracking.get();
        let (state_text, state_cls) = match state {
            TrackingState::Stopped => ("остановлен", "gl-badge"),
            TrackingState::Running => ("работает", "gl-badge gl-badge--ok"),
            TrackingState::Paused => ("пауза", "gl-badge gl-badge--warn"),
        };
        self.ui.badge_state.set_text_content(Some(state_text));
        self.ui.badge_state.set_class_name(state_cls);

        // Индикатор движения. Показываем только при активном трекинге.
        if state == TrackingState::Running {
            let moving = self.moving.get();
            let (mt, mc) = if moving {
                ("двигаюсь", "gl-badge gl-badge--ok")
            } else {
                ("стою", "gl-badge gl-badge--warn")
            };
            self.ui.badge_move.set_text_content(Some(mt));
            self.ui.badge_move.set_class_name(mc);
        } else {
            self.ui.badge_move.set_text_content(Some("—"));
            self.ui.badge_move.set_class_name("gl-badge");
        }

        // Счётчики.
        let smp = self.watch_ok.get();
        self.ui
            .badge_count
            .set_text_content(Some(&format_str(format_args!("{smp} smp"))));
        self.ui.badge_count.set_class_name("gl-badge");

        let acc = self.watch_accepted.get();
        let acc_cls = if acc > 0 {
            "gl-badge gl-badge--ok"
        } else {
            "gl-badge"
        };
        self.ui
            .badge_accepted
            .set_text_content(Some(&format_str(format_args!("{acc} acc"))));
        self.ui.badge_accepted.set_class_name(acc_cls);

        let err = self.watch_err.get();
        let err_cls = if err > 0 {
            "gl-badge gl-badge--err"
        } else {
            "gl-badge"
        };
        self.ui
            .badge_errors
            .set_text_content(Some(&format_str(format_args!("{err} err"))));
        self.ui.badge_errors.set_class_name(err_cls);

        let flt = self.watch_filtered.get();
        let flt_cls = if flt > 0 {
            "gl-badge gl-badge--warn"
        } else {
            "gl-badge"
        };
        self.ui
            .badge_filtered
            .set_text_content(Some(&format_str(format_args!("{flt} flt"))));
        self.ui.badge_filtered.set_class_name(flt_cls);
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

fn get_input(id: &str) -> Result<web_sys::HtmlInputElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не input")));
}

fn get_button(id: &str) -> Result<web_sys::HtmlButtonElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")));
}

fn set_click<F>(el: &web_sys::HtmlButtonElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```