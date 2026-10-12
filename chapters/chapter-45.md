# Глава 45. Web Bluetooth — переработанная версия

Ниже — обновлённый текст главы, учитывающий всё, что выяснилось в ходе отладки (Linux, nRF Connect Advertiser vs GATT Server, диагностика «пустого» GATT-дерева), и полные исходные коды всех файлов.

---

## 45.0. Обновлённая структура

```
src/
  bluetooth_utils.rs              ← НОВЫЙ модуль 45
  bluetooth_gatt.rs               ← НОВЫЙ модуль 45
  video_frame.rs                  ← из 23.1 (format_str, set_prop)
  app.rs                          ← демо 45

tests/
  wasm_bluetooth.rs               ← тесты 45 (Этап 2)

src/lib.rs                        ← +2 строки: bluetooth_utils, bluetooth_gatt
Cargo.toml                        ← без изменений (см. ниже)
```

**Разделение слоёв.**

- `bluetooth_utils.rs` — **уровень устройства**: feature detection, secure context, `requestDevice`, разрешения, `connect`/`disconnect`, чтение полей `BluetoothDevice`, подписка на `gattserverdisconnected`.
- `bluetooth_gatt.rs` — **GATT-дерево**: сервисы, характеристики, чтение/запись, подписки на нотификации, разбор специфичных payload-ов (heart rate, battery level).

Тот же профит, что `webgpu_utils` + `webgpu_compute` в главе 44: инкапсуляция, читаемость, лёгкое тестирование.

**Префикс CSS** — `bt-` (bluetooth). Не пересекается с `vf-`, `vd-`, `ve-`, `vt-`, `ac-`, `id-`, `px-`, `ms-`, `cp-`, `dm-`, `mx-`, `pp-`, `wg-`.

**Ссылки:**

- W3C: [Web Bluetooth Specification](https://webbluetoothcg.github.io/web-bluetooth/)
- MDN: [`Bluetooth`](https://developer.mozilla.org/en-US/docs/Web/API/Bluetooth), [`BluetoothDevice`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothDevice), [`BluetoothRemoteGATTServer`](https://developer.mozilla.org/en-US/docs/Web/API/BluetoothRemoteGATTServer)
- Chrome Developers: [Communicating with Bluetooth devices over JavaScript](https://developer.chrome.com/docs/capabilities/bluetooth)
- Bluetooth SIG: [Assigned Numbers — GATT Services](https://www.bluetooth.com/specifications/assigned-numbers/) — `heart_rate` = 0x180D, `battery_service` = 0x180F
- Тестовое BLE-периферийное ПО: **nRF Connect for Mobile** (Nordic Semiconductor, Android/iOS) в режиме **Configure GATT Server**, **LightBlue**, **Web Bluetooth emulator** в Chrome DevTools (Windows/macOS/ChromeOS/Android — **не Linux**).

---

## 45.1. О `Cargo.toml` — важное замечание

**Никаких новых фич web-sys добавлять не нужно.** В `Cargo.toml` из главы 44 уже есть `wasm-bindgen`, `js-sys`, `web-sys` (с сотней фич), `wasm-bindgen-futures`, `gloo-timers`. Все вызовы Web Bluetooth API делаются через `js_sys::Reflect::get` + `Function::call*`.

**Почему не через типизированные bindings.** Web Bluetooth в web-sys 0.3.105 экспортируется **только под `#[cfg(web_sys_unstable_apis)]`**. Это требует переменной окружения `RUSTFLAGS='--cfg=web_sys_unstable_apis'` — загрязняет сборку и ломает воспроизводимость. Reflect-путь **стабилен для любой версии** web-sys и **явно проверяет** наличие каждого метода.

---

## 45.2. Модуль `src/bluetooth_utils.rs`

**Ключевые изменения после отладки:**

- `request_device` теперь принимает `RequestDeviceOptions` — структуру с двумя режимами: `for_service(uuid)` (фильтр) и `all_devices(optional_services)` (`acceptAllDevices: true` + `optionalServices`). Это позволяет одной функцией покрыть оба сценария UI-демо и корректно обрабатывать требование Chrome: при `acceptAllDevices: true` обязательно непустой `optionalServices`, иначе `SecurityError`.
- `get_known_devices` корректно обрабатывает владение `JsValue` — `result.clone().dyn_into::<Array>()`, потому что `dyn_into` забирает `self`.
- `subscribe_disconnect` **не использует `closure.forget()` вслепую** — оно остаётся для случая, когда handle не нужен вызывающему, но в продакшне лучше возвращать handle (см. `NotificationHandle` в `bluetooth_gatt.rs` для аналогичного паттерна).

```rust
//! 45. Web Bluetooth: обнаружение, разрешения, подключение.
//!
//! ## Публичный API
//!
//! - `has_web_bluetooth()` — есть ли `navigator.bluetooth`.
//! - `is_secure_context()` — HTTPS или localhost.
//! - `request_device(options)` — `navigator.bluetooth.requestDevice`
//!   с фильтрами или `acceptAllDevices`.
//! - `get_known_devices()` — `navigator.bluetooth.getDevices`.
//! - `read_device_info(device) -> DeviceInfo` — id, name, connected.
//! - `connect(device) -> GATT server`, `disconnect(device)`.
//!
//! ## Почему всё через `Reflect`
//!
//! Web Bluetooth в web-sys 0.3.105 доступен только под
//! `#[cfg(web_sys_unstable_apis)]`. Reflect-путь работает без
//! нестабильных фич и явно проверяет наличие каждого метода.

use js_sys::{Array, Object, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::{format_str, set_prop};

// =============================================================================
// Feature detection
// =============================================================================

/// Проверяет наличие `navigator.bluetooth`.
pub fn has_web_bluetooth() -> bool {
    let Some(window) = web_sys::window() else {
        return false;
    };
    let Ok(nav) = Reflect::get(window.as_ref(), &JsValue::from_str("navigator")) else {
        return false;
    };
    let Ok(bt) = Reflect::get(&nav, &JsValue::from_str("bluetooth")) else {
        return false;
    };
    return !bt.is_undefined() && !bt.is_null();
}

/// HTTPS или localhost.
pub fn is_secure_context() -> bool {
    let Some(window) = web_sys::window() else {
        return false;
    };
    return Reflect::get(window.as_ref(), &JsValue::from_str("isSecureContext"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
}

/// `navigator.bluetooth.getAvailability()` — deprecated, но всё ещё
/// работает в Chrome. Возвращает `None`, если метод отсутствует.
pub async fn get_availability() -> Option<bool> {
    let window = web_sys::window()?;
    let nav = Reflect::get(window.as_ref(), &JsValue::from_str("navigator")).ok()?;
    let bt = Reflect::get(&nav, &JsValue::from_str("bluetooth")).ok()?;
    let get = Reflect::get(&bt, &JsValue::from_str("getAvailability")).ok()?;
    let get = get.dyn_into::<js_sys::Function>().ok()?;
    let promise = get.call0(&bt).ok()?.dyn_into::<Promise>().ok()?;
    let result = JsFuture::from(promise).await.ok()?;
    return result.as_bool();
}

// =============================================================================
// Ошибки
// =============================================================================

/// Разбирает `DOMException` в пару (name, message).
fn read_exception(e: &JsValue) -> (String, String) {
    let name = Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| String::from("Error"));
    let message = Reflect::get(e, &JsValue::from_str("message"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    return (name, message);
}

/// Формирует человекочитаемое сообщение об ошибке `requestDevice`.
pub fn format_bluetooth_error(name: &str, message: &str) -> String {
    match name {
        "NotFoundError" | "AbortError" => {
            "Пользователь отменил выбор устройства.".to_string()
        }
        "SecurityError" => {
            "Web Bluetooth требует secure context (HTTPS или localhost) \
             и user gesture (клик по кнопке)."
                .to_string()
        }
        "NetworkError" => {
            "Bluetooth-адаптер недоступен (выключен или занят другим приложением)."
                .to_string()
        }
        "NotSupportedError" => {
            "Браузер не поддерживает Web Bluetooth.".to_string()
        }
        "InvalidStateError" => {
            "Состояние устройства не позволяет выполнить операцию.".to_string()
        }
        "NotAllowedError" => {
            "Пользователь запретил доступ к Bluetooth.".to_string()
        }
        _ => {
            format_str(format_args!("{name} — {message}"))
        }
    }
}

// =============================================================================
// requestDevice
// =============================================================================

/// Параметры для `navigator.bluetooth.requestDevice`.
pub struct RequestDeviceOptions {
    /// Если `true`, будут показаны все устройства.
    /// `filters` при этом игнорируется.
    pub accept_all: bool,
    /// UUID сервисов, к которым сайт хочет иметь доступ.
    /// Обязательно для `accept_all = true`, иначе будет `SecurityError`.
    pub optional_services: Vec<String>,
    /// Если `accept_all = false`, это фильтры по сервисам.
    pub filters_services: Vec<String>,
}

impl RequestDeviceOptions {
    /// Поиск конкретного типа устройства по сервису.
    pub fn for_service(service: &str) -> Self {
        Self {
            accept_all: false,
            optional_services: vec![],
            filters_services: vec![service.to_string()],
        }
    }

    /// Показать все устройства, но с доступом к перечисленным сервисам.
    pub fn all_devices(optional_services: Vec<String>) -> Self {
        Self {
            accept_all: true,
            optional_services,
            filters_services: vec![],
        }
    }
}

/// `navigator.bluetooth.requestDevice(options)`.
pub async fn request_device(options: RequestDeviceOptions) -> Result<JsValue, String> {
    if !has_web_bluetooth() {
        return Err(String::from("navigator.bluetooth недоступен"));
    }

    let window = web_sys::window().ok_or_else(|| String::from("no window"))?;
    let nav = Reflect::get(window.as_ref(), &JsValue::from_str("navigator"))
        .map_err(|_| String::from("no navigator"))?;
    let bt = Reflect::get(&nav, &JsValue::from_str("bluetooth"))
        .map_err(|_| String::from("no navigator.bluetooth"))?;
    let request = Reflect::get(&bt, &JsValue::from_str("requestDevice"))
        .map_err(|_| String::from("no requestDevice"))?;
    let request = request
        .dyn_into::<js_sys::Function>()
        .map_err(|_| String::from("requestDevice is not a function"))?;

    // Собираем JS-объект options
    let js_options = Object::new();

    if options.accept_all {
        // Для acceptAllDevices обязателен непустой optionalServices
        if options.optional_services.is_empty() {
            return Err(String::from(
                "optionalServices не может быть пустым при acceptAllDevices = true",
            ));
        }
        set_prop(&js_options, "acceptAllDevices", &JsValue::TRUE)
            .map_err(|e| format_str(format_args!("set acceptAllDevices: {e:?}")))?;

        let opt_services = Array::new();
        for s in &options.optional_services {
            opt_services.push(&JsValue::from_str(s));
        }
        set_prop(&js_options, "optionalServices", &opt_services)
            .map_err(|e| format_str(format_args!("set optionalServices: {e:?}")))?;
    } else {
        // Используем filters
        let services_arr = Array::new();
        for s in &options.filters_services {
            services_arr.push(&JsValue::from_str(s));
        }
        let filter = Object::new();
        set_prop(&filter, "services", &services_arr)
            .map_err(|e| format_str(format_args!("set filter.services: {e:?}")))?;

        let filters_arr = Array::new();
        filters_arr.push(&filter);
        set_prop(&js_options, "filters", &filters_arr)
            .map_err(|e| format_str(format_args!("set filters: {e:?}")))?;
    }

    let promise = request
        .call1(&bt, js_options.as_ref())
        .map_err(|e| format_str(format_args!("call requestDevice: {e:?}")))?
        .dyn_into::<Promise>()
        .map_err(|_| String::from("requestDevice did not return a Promise"))?;

    match JsFuture::from(promise).await {
        Ok(v) => Ok(v),
        Err(e) => {
            let (name, message) = read_exception(&e);
            Err(format_bluetooth_error(&name, &message))
        }
    }
}

/// `navigator.bluetooth.getDevices()` — уже сопряжённые устройства,
/// без нового запроса к пользователю. Доступно в Chrome 108+.
pub async fn get_known_devices() -> Result<Vec<JsValue>, String> {
    if !has_web_bluetooth() {
        return Err(String::from("navigator.bluetooth недоступен"));
    }
    let Some(window) = web_sys::window() else {
        return Err(String::from("no window"));
    };
    let Ok(nav) = Reflect::get(window.as_ref(), &JsValue::from_str("navigator")) else {
        return Err(String::from("no navigator"));
    };
    let Ok(bt) = Reflect::get(&nav, &JsValue::from_str("bluetooth")) else {
        return Err(String::from("no navigator.bluetooth"));
    };
    let get = match Reflect::get(&bt, &JsValue::from_str("getDevices")) {
        Ok(f) => f,
        Err(_) => return Ok(Vec::new()),
    };
    let Ok(get) = get.dyn_into::<js_sys::Function>() else {
        return Ok(Vec::new());
    };
    let Ok(promise) = get.call0(&bt) else {
        return Ok(Vec::new());
    };
    let Ok(promise) = promise.dyn_into::<Promise>() else {
        return Ok(Vec::new());
    };
    let Ok(result) = JsFuture::from(promise).await else {
        return Ok(Vec::new());
    };
    // `dyn_into` забирает self — клонируем JsValue (refcount++ на JS-объекте).
    let Ok(arr) = result.clone().dyn_into::<Array>() else {
        return Ok(Vec::new());
    };
    let mut out = Vec::with_capacity(arr.length() as usize);
    for i in 0..arr.length() {
        out.push(arr.get(i));
    }
    return Ok(out);
}

// =============================================================================
// Device info
// =============================================================================

pub struct DeviceInfo {
    pub id: String,
    pub name: String,
    pub connected: bool,
}

/// Читает поля `BluetoothDevice`. `name` может быть пустым —
/// некоторые устройства не объявляют его.
pub fn read_device_info(device: &JsValue) -> DeviceInfo {
    let read_str = |key: &str| -> String {
        Reflect::get(device, &JsValue::from_str(key))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default()
    };
    let connected = Reflect::get(device, &JsValue::from_str("gatt"))
        .ok()
        .and_then(|g| {
            Reflect::get(&g, &JsValue::from_str("connected"))
                .ok()
                .and_then(|v| v.as_bool())
        })
        .unwrap_or(false);
    return DeviceInfo {
        id: read_str("id"),
        name: read_str("name"),
        connected,
    };
}

/// Подписка на `gattserverdisconnected`.
///
/// Возвращает `Closure`, который нужно сохранить живым — иначе
/// callback будет собран GC.
pub fn subscribe_disconnect<F>(device: &JsValue, mut callback: F) -> Result<(), String>
where
    F: FnMut() + 'static,
{
    let add = Reflect::get(device, &JsValue::from_str("addEventListener"))
        .map_err(|_| String::from("no addEventListener"))?;
    let add = add
        .dyn_into::<js_sys::Function>()
        .map_err(|_| String::from("addEventListener is not a function"))?;

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(
        move |_event: JsValue| {
            callback();
        },
    );

    add.call2(
        device,
        &JsValue::from_str("gattserverdisconnected"),
        closure.as_ref().unchecked_ref(),
    )
    .map_err(|e| format_str(format_args!("addEventListener: {e:?}")))?;

    // Оставляем closure живым — JS держит ссылку на функцию.
    closure.forget();
    return Ok(());
}

// =============================================================================
// Connect / disconnect
// =============================================================================

/// `device.gatt.connect()` → `BluetoothRemoteGATTServer`.
pub async fn connect(device: &JsValue) -> Result<JsValue, String> {
    let gatt = Reflect::get(device, &JsValue::from_str("gatt"))
        .map_err(|_| String::from("device has no .gatt"))?;
    if gatt.is_undefined() || gatt.is_null() {
        return Err(String::from("device.gatt недоступен"));
    }

    let connect = Reflect::get(&gatt, &JsValue::from_str("connect"))
        .map_err(|_| String::from("no gatt.connect"))?;
    let connect = connect
        .dyn_into::<js_sys::Function>()
        .map_err(|_| String::from("gatt.connect is not a function"))?;

    let promise = connect
        .call0(&gatt)
        .map_err(|e| format_str(format_args!("call connect: {e:?}")))?
        .dyn_into::<Promise>()
        .map_err(|_| String::from("connect did not return a Promise"))?;

    match JsFuture::from(promise).await {
        Ok(v) => Ok(v),
        Err(e) => {
            let (name, message) = read_exception(&e);
            Err(format_str(format_args!("connect: {name} — {message}")))
        }
    }
}

/// `device.gatt.disconnect()`. Идемпотентно.
pub fn disconnect(device: &JsValue) -> Result<(), String> {
    let Ok(gatt) = Reflect::get(device, &JsValue::from_str("gatt")) else {
        return Ok(());
    };
    if gatt.is_undefined() || gatt.is_null() {
        return Ok(());
    }
    let Ok(disconnect) = Reflect::get(&gatt, &JsValue::from_str("disconnect")) else {
        return Ok(());
    };
    let Ok(disconnect) = disconnect.dyn_into::<js_sys::Function>() else {
        return Ok(());
    };
    let _ = disconnect.call0(&gatt);
    return Ok(());
}
```

---

## 45.3. Модуль `src/bluetooth_gatt.rs`

**Ключевые изменения:**

- `array_to_vec` получил `value.clone()` перед `dyn_into::<Array>()` — иначе borrow-checker отказывает, потому что `dyn_into` забирает `self`.
- `NotificationHandle` инкапсулирует `Closure` и даёт метод `as_function()` для `removeEventListener` — то же, что делает `subscribe_disconnect`, но с явным возвратом handle вместо `forget()`.
- Константы и человекочитаемые имена UUID вынесены отдельно — легко расширять.

```rust
//! 45. Web Bluetooth: GATT — сервисы, характеристики, нотификации.
//!
//! ## Публичный API
//!
//! - `get_primary_services(gatt) -> Vec<service>` — все сервисы.
//! - `get_primary_service(gatt, uuid)` — конкретный сервис.
//! - `get_characteristics(service) -> Vec<char>` — все характеристики.
//! - `get_characteristic(service, uuid)` — конкретная характеристика.
//! - `read_value(char) -> Vec<u8>` — разовое чтение.
//! - `write_value(char, data)` — запись.
//! - `start_notifications(char)` / `stop_notifications(char)`.
//! - `subscribe_value_changed(char, callback)` — подписка на события.
//! - `parse_heart_rate(data) -> Option<u16>` — разбор HR-измерения.
//! - `format_service_uuid(uuid) -> String`,
//!   `format_characteristic_uuid(uuid) -> String`
//!   — человекочитаемые имена известных UUID.

use js_sys::{Array, Promise, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::video_frame::format_str;

// =============================================================================
// Константы — стандартные UUID Bluetooth SIG
// =============================================================================

pub const SERVICE_HEART_RATE: &str = "heart_rate";
pub const SERVICE_BATTERY: &str = "battery_service";
pub const SERVICE_ENVIRONMENTAL: &str = "environmental_sensing";
pub const SERVICE_DEVICE_INFORMATION: &str = "device_information";

pub const CHAR_HEART_RATE_MEASUREMENT: &str = "heart_rate_measurement";
pub const CHAR_BATTERY_LEVEL: &str = "battery_level";
pub const CHAR_MANUFACTURER_NAME: &str = "manufacturer_name_string";
pub const CHAR_MODEL_NUMBER: &str = "model_number_string";

// =============================================================================
// Ошибки
// =============================================================================

fn read_exception(e: &JsValue) -> (String, String) {
    let name = Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| String::from("Error"));
    let message = Reflect::get(e, &JsValue::from_str("message"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    return (name, message);
}

fn format_gatt_error(op: &str, name: &str, message: &str) -> String {
    match name {
        "NotFoundError" => format_str(format_args!(
            "{op}: сервис или характеристика не найдены"
        )),
        "NetworkError" => format_str(format_args!(
            "{op}: устройство отключено"
        )),
        "InvalidStateError" => format_str(format_args!(
            "{op}: устройство не в состоянии выполнить операцию"
        )),
        "NotSupportedError" => format_str(format_args!(
            "{op}: операция не поддерживается этим устройством"
        )),
        "SecurityError" => format_str(format_args!(
            "{op}: сервис не был указан в optionalServices при requestDevice"
        )),
        _ => format_str(format_args!("{op}: {name} — {message}")),
    }
}

// =============================================================================
// Утилиты вызова
// =============================================================================

/// Вызывает метод объекта, возвращающий `Promise`.
async fn call_method_promise(
    obj: &JsValue,
    method: &str,
    args: &[JsValue],
    op_label: &str,
) -> Result<JsValue, String> {
    let f = Reflect::get(obj, &JsValue::from_str(method))
        .map_err(|_| format_str(format_args!("{op_label}: нет метода {method}")))?;
    let f = f
        .dyn_into::<js_sys::Function>()
        .map_err(|_| format_str(format_args!("{op_label}: {method} — не функция")))?;

    let call_result = match args.len() {
        0 => f.call0(obj),
        1 => f.call1(obj, &args[0]),
        2 => f.call2(obj, &args[0], &args[1]),
        _ => return Err(format_str(format_args!("{op_label}: слишком много аргументов"))),
    };

    let promise = call_result
        .map_err(|e| {
            let (name, message) = read_exception(&e);
            format_gatt_error(op_label, &name, &message)
        })?
        .dyn_into::<Promise>()
        .map_err(|_| format_str(format_args!("{op_label}: результат — не Promise")))?;

    JsFuture::from(promise).await.map_err(|e| {
        let (name, message) = read_exception(&e);
        format_gatt_error(op_label, &name, &message)
    })
}

fn array_to_vec(value: &JsValue) -> Result<Vec<JsValue>, String> {
    // `dyn_into` забирает `self` — клонируем `JsValue` (дешёвая
    // операция: увеличивает refcount на JS-объект).
    let arr = value
        .clone()
        .dyn_into::<Array>()
        .map_err(|_| String::from("ожидался Array"))?;
    let mut out = Vec::with_capacity(arr.length() as usize);
    for i in 0..arr.length() {
        out.push(arr.get(i));
    }
    return Ok(out);
}

/// `DataView` → `Vec<u8>`.
fn dataview_to_vec(value: &JsValue) -> Result<Vec<u8>, String> {
    let len = Reflect::get(value, &JsValue::from_str("byteLength"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as usize;
    let getter = Reflect::get(value, &JsValue::from_str("getUint8"))
        .map_err(|_| String::from("DataView без getUint8"))?;
    let getter = getter
        .dyn_into::<js_sys::Function>()
        .map_err(|_| String::from("getUint8 — не функция"))?;
    let mut out = Vec::with_capacity(len);
    for i in 0..len {
        let byte = getter
            .call1(value, &JsValue::from_f64(i as f64))
            .map_err(|e| format_str(format_args!("getUint8: {e:?}")))?
            .as_f64()
            .unwrap_or(0.0) as u8;
        out.push(byte);
    }
    return Ok(out);
}

// =============================================================================
// Сервисы
// =============================================================================

/// `gatt.getPrimaryServices()` → все primary-сервисы.
pub async fn get_primary_services(gatt: &JsValue) -> Result<Vec<JsValue>, String> {
    let result = call_method_promise(gatt, "getPrimaryServices", &[], "getPrimaryServices").await?;
    return array_to_vec(&result);
}

/// `gatt.getPrimaryService(uuid)` → один сервис.
pub async fn get_primary_service(gatt: &JsValue, uuid: &str) -> Result<JsValue, String> {
    let arg = JsValue::from_str(uuid);
    return call_method_promise(
        gatt,
        "getPrimaryService",
        &[arg],
        "getPrimaryService",
    )
    .await;
}

/// UUID сервиса — `BluetoothRemoteGATTService.uuid`.
pub fn read_service_uuid(service: &JsValue) -> String {
    return Reflect::get(service, &JsValue::from_str("uuid"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
}

// =============================================================================
// Характеристики
// =============================================================================

/// `service.getCharacteristics()` → все характеристики.
pub async fn get_characteristics(service: &JsValue) -> Result<Vec<JsValue>, String> {
    let result = call_method_promise(
        service,
        "getCharacteristics",
        &[],
        "getCharacteristics",
    )
    .await?;
    return array_to_vec(&result);
}

/// `service.getCharacteristic(uuid)` → одна характеристика.
pub async fn get_characteristic(service: &JsValue, uuid: &str) -> Result<JsValue, String> {
    let arg = JsValue::from_str(uuid);
    return call_method_promise(
        service,
        "getCharacteristic",
        &[arg],
        "getCharacteristic",
    )
    .await;
}

/// UUID характеристики — `BluetoothRemoteGATTCharacteristic.uuid`.
pub fn read_characteristic_uuid(characteristic: &JsValue) -> String {
    return Reflect::get(characteristic, &JsValue::from_str("uuid"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
}

/// `characteristic.properties` — объект с полями `read`, `write`,
/// `notify`, `indicate`. Возвращает массив строк-флагов.
pub fn read_characteristic_properties(characteristic: &JsValue) -> Vec<&'static str> {
    let props = Reflect::get(characteristic, &JsValue::from_str("properties"))
        .unwrap_or(JsValue::UNDEFINED);
    let mut out = Vec::new();
    for key in ["read", "write", "writeWithoutResponse", "notify", "indicate"] {
        let has = Reflect::get(&props, &JsValue::from_str(key))
            .ok()
            .and_then(|v| v.as_bool())
            .unwrap_or(false);
        if has {
            match key {
                "read" => out.push("read"),
                "write" => out.push("write"),
                "writeWithoutResponse" => out.push("write-no-resp"),
                "notify" => out.push("notify"),
                "indicate" => out.push("indicate"),
                _ => {}
            }
        }
    }
    return out;
}

// =============================================================================
// Read / write
// =============================================================================

/// `characteristic.readValue()` → `Vec<u8>`.
pub async fn read_value(characteristic: &JsValue) -> Result<Vec<u8>, String> {
    let value = call_method_promise(characteristic, "readValue", &[], "readValue").await?;
    return dataview_to_vec(&value);
}

/// `characteristic.writeValue(data)`. `data` копируется в `Uint8Array`.
pub async fn write_value(characteristic: &JsValue, data: &[u8]) -> Result<(), String> {
    let u8 = js_sys::Uint8Array::from(data);
    let u8_js: JsValue = u8.into();
    let _ = call_method_promise(
        characteristic,
        "writeValue",
        &[u8_js],
        "writeValue",
    )
    .await?;
    return Ok(());
}

// =============================================================================
// Notifications
// =============================================================================

/// `characteristic.startNotifications()`. После вызова устройство
/// начнёт слать события `characteristicvaluechanged`.
pub async fn start_notifications(characteristic: &JsValue) -> Result<(), String> {
    let _ = call_method_promise(
        characteristic,
        "startNotifications",
        &[],
        "startNotifications",
    )
    .await?;
    return Ok(());
}

/// `characteristic.stopNotifications()`. Идемпотентно.
pub async fn stop_notifications(characteristic: &JsValue) -> Result<(), String> {
    let _ = call_method_promise(
        characteristic,
        "stopNotifications",
        &[],
        "stopNotifications",
    )
    .await?;
    return Ok(());
}

/// Подписка на `characteristicvaluechanged`. Возвращает handle —
/// держите его живым столько, сколько нужна подписка.
pub struct NotificationHandle {
    closure: wasm_bindgen::closure::Closure<dyn FnMut(JsValue)>,
}

impl NotificationHandle {
    /// Возвращает raw `JsValue`-функцию для `removeEventListener`,
    /// если понадобится отписка.
    pub fn as_function(&self) -> &js_sys::Function {
        return self.closure.as_ref().unchecked_ref();
    }
}

/// `characteristic.addEventListener("characteristicvaluechanged", cb)`.
///
/// Callback получает `Vec<u8>` — уже разобранный `DataView` из
/// `event.target.value`.
pub fn subscribe_value_changed<F>(
    characteristic: &JsValue,
    mut callback: F,
) -> Result<NotificationHandle, String>
where
    F: FnMut(Vec<u8>) + 'static,
{
    let add = Reflect::get(characteristic, &JsValue::from_str("addEventListener"))
        .map_err(|_| String::from("no addEventListener"))?;
    let add = add
        .dyn_into::<js_sys::Function>()
        .map_err(|_| String::from("addEventListener — не функция"))?;

    let closure =
        wasm_bindgen::closure::Closure::<dyn FnMut(JsValue)>::new(move |event: JsValue| {
            let Ok(target) = Reflect::get(&event, &JsValue::from_str("target")) else {
                return;
            };
            let Ok(value) = Reflect::get(&target, &JsValue::from_str("value")) else {
                return;
            };
            if let Ok(bytes) = dataview_to_vec(&value) {
                callback(bytes);
            }
        });

    add.call2(
        characteristic,
        &JsValue::from_str("characteristicvaluechanged"),
        closure.as_ref().unchecked_ref(),
    )
    .map_err(|e| format_str(format_args!("addEventListener: {e:?}")))?;

    return Ok(NotificationHandle { closure });
}

// =============================================================================
// Парсинг специфичных payload-ов
// =============================================================================

/// `Heart Rate Measurement` (0x2A37):
///
/// ```text
/// byte 0: flags
///   bit 0: 0 = uint8 bpm, 1 = uint16 bpm
/// bytes 1..: heart rate value (LE)
/// ```
pub fn parse_heart_rate(data: &[u8]) -> Option<u16> {
    if data.len() < 2 {
        return None;
    }
    let flags = data[0];
    let is_uint16 = (flags & 0x01) != 0;
    if is_uint16 {
        if data.len() < 3 {
            return None;
        }
        return Some(u16::from_le_bytes([data[1], data[2]]));
    }
    return Some(data[1] as u16);
}

/// `Battery Level` (0x2A19): один байт 0–100.
pub fn parse_battery_level(data: &[u8]) -> Option<u8> {
    if data.is_empty() {
        return None;
    }
    return Some(data[0].min(100));
}

// =============================================================================
// Человекочитаемые имена UUID
// =============================================================================

/// Известные UUID сервисов → читаемые имена.
pub fn format_service_uuid(uuid: &str) -> String {
    let lower = uuid.to_lowercase();
    let name = match lower.as_str() {
        "180d" | "0000180d-0000-1000-8000-00805f9b34fb" => "Heart Rate",
        "180f" | "0000180f-0000-1000-8000-00805f9b34fb" => "Battery Service",
        "181a" | "0000181a-0000-1000-8000-00805f9b34fb" => "Environmental Sensing",
        "180a" | "0000180a-0000-1000-8000-00805f9b34fb" => "Device Information",
        "1800" | "00001800-0000-1000-8000-00805f9b34fb" => "Generic Access",
        "1801" | "00001801-0000-1000-8000-00805f9b34fb" => "Generic Attribute",
        _ => return uuid.to_string(),
    };
    return format_str(format_args!("{name} ({lower})"));
}

/// Известные UUID характеристик → читаемые имена.
pub fn format_characteristic_uuid(uuid: &str) -> String {
    let lower = uuid.to_lowercase();
    let name = match lower.as_str() {
        "2a37" | "00002a37-0000-1000-8000-00805f9b34fb" => "Heart Rate Measurement",
        "2a38" | "00002a38-0000-1000-8000-00805f9b34fb" => "Body Sensor Location",
        "2a19" | "00002a19-0000-1000-8000-00805f9b34fb" => "Battery Level",
        "2a29" | "00002a29-0000-1000-8000-00805f9b34fb" => "Manufacturer Name",
        "2a24" | "00002a24-0000-1000-8000-00805f9b34fb" => "Model Number",
        _ => return uuid.to_string(),
    };
    return format_str(format_args!("{name} ({lower})"));
}

// =============================================================================
// Форматирование для UI
// =============================================================================

/// Преобразует массив байт в hex-строку с пробелами.
pub fn hex_bytes(data: &[u8]) -> String {
    if data.is_empty() {
        return String::from("—");
    }
    let mut out = String::with_capacity(data.len() * 3);
    for (i, b) in data.iter().enumerate() {
        if i > 0 {
            out.push(' ');
        }
        out.push_str(&format_str(format_args!("{b:02X}")));
    }
    return out;
}
```

---

## 45.4. Модуль `src/app.rs` — с диагностикой пустого GATT

**Ключевое улучшение после отладки:** в `discover_gatt` добавлена **диагностика типичных ошибок**. Если `getPrimaryServices` вернул `NotFoundError` или пустой список — это почти всегда означает, что устройство **только вещает** (Advertiser), но не имеет GATT-сервера. В логе появится подсказка, которую вы видели во время отладки.

Также добавлена проверка: если после обхода остались только `0x1800` (Generic Access) и `0x1801` (Generic Attribute) — тоже выдаётся предупреждение, потому что такие «сервисы» есть у любого BLE-устройства, включая наушники в их служебном BLE-режиме.

```rust
//! Глава 45 — Web Bluetooth: обнаружение, GATT, нотификации.
//!
//! Демо: подключение BLE-устройства, обход GATT-дерева, чтение
//! характеристик, подписка на heart rate notifications.
//!
//! ## Два режима подключения
//!
//! - **«Подключить пульсометр»** — `requestDevice` с фильтром
//!   `services: ['heart_rate']`. Пользователь видит только
//!   устройства, объявляющие Heart Rate Service.
//! - **«Показать все устройства»** — `requestDevice` с
//!   `acceptAllDevices: true` и `optionalServices: [...]`.
//!   Пользователь видит все BLE-устройства поблизости, но
//!   доступ к их сервисам — только к указанным в optional.
//!
//! ## User gesture
//!
//! `requestDevice` работает **только из обработчика клика**. Мы
//! **не** делаем `await` до вызова — иначе user activation
//! тратится и браузер возвращает `SecurityError`.
//!
//! ## Диагностика пустого GATT
//!
//! Частая ошибка новичков: подключиться к устройству, которое
//! **только вещает** (Advertiser) без GATT-сервера. Тогда
//! `gatt.connect()` проходит, а `getPrimaryServices()` возвращает
//! `NotFoundError`. В логе выводится подсказка.

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use js_sys::Reflect;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, HtmlElement};

use crate::bluetooth_gatt::{
    format_characteristic_uuid, format_service_uuid, get_characteristics,
    get_primary_services, hex_bytes, parse_battery_level, parse_heart_rate,
    read_characteristic_properties, read_characteristic_uuid,
    read_service_uuid, read_value, start_notifications,
    subscribe_value_changed, NotificationHandle, SERVICE_BATTERY,
};
use crate::bluetooth_utils::{
    connect, disconnect, get_known_devices, has_web_bluetooth,
    is_secure_context, read_device_info, request_device,
    subscribe_disconnect, DeviceInfo, RequestDeviceOptions,
};
use crate::video_frame::format_str;

// =============================================================================
// CSS
// =============================================================================

const STYLE_CSS: &str = r#"
.bt-desc { max-width:1000px;margin:0 0 20px 0;padding:12px 16px;background:#161b22;border-left:3px solid #58a6ff;border-radius:4px;color:#c9d1d9;font-size:14px;line-height:1.55;}
.bt-desc-line { margin:6px 0; }
.bt-note { max-width:1000px;margin-bottom:16px;padding:10px 16px;background:#2a1d10;border-left:3px solid #f0883e;border-radius:4px;color:#e6b780;font-size:13px;line-height:1.55;}
.bt-note-err { background:#2a1010; border-left-color:#f85149; color:#ffa198; }
.bt-block { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#161b22;border:1px solid #21262d;border-radius:6px;display:flex;flex-wrap:wrap;gap:12px;align-items:center;}
.bt-btn { padding:8px 14px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;cursor:pointer;font-size:12px;font-weight:500;}
.bt-btn:hover:not([disabled]) { background:#30363d; }
.bt-btn:disabled { opacity:0.5;cursor:not-allowed; }
.bt-btn-primary { background:#238636;border-color:#2ea043;color:#fff;}
.bt-btn-primary:hover:not([disabled]) { background:#2ea043; }
.bt-btn-secondary { background:#8957e5;border-color:#a371f7;color:#fff;}
.bt-btn-secondary:hover:not([disabled]) { background:#a371f7; }
.bt-btn-danger { background:#a4302e;border-color:#d1242f;color:#fff;}
.bt-btn-danger:hover:not([disabled]) { background:#d1242f; }
.bt-label { font-family:ui-monospace,monospace;font-size:12px;color:#8b949e;}
.bt-select { padding:6px 10px;border-radius:6px;border:1px solid #30363d;background:#21262d;color:#c9d1d9;font-size:12px;font-family:ui-monospace,monospace;}

.bt-stats { max-width:1000px;margin-bottom:12px;padding:10px 14px;background:#0d1117;border-left:3px solid #3fb950;border-radius:4px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;overflow-wrap:anywhere;word-break:break-word;}
.bt-stats .lbl { color:#8b949e; }
.bt-stats .val { color:#79c0ff; }
.bt-stats .ok { color:#3fb950; }
.bt-stats .warn { color:#f0883e; }
.bt-stats .err { color:#f85149; }
.bt-stats .muted { color:#6e7681; }

.bt-device { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;font-size:12px;line-height:1.8;}
.bt-device .row { display:flex; gap:12px; }
.bt-device .lbl { color:#8b949e; min-width:130px; flex-shrink:0; }
.bt-device .val { color:#79c0ff; }
.bt-device .ok { color:#3fb950; }
.bt-device .muted { color:#6e7681; }

.bt-tree { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:12px;line-height:1.7;color:#c9d1d9;overflow-wrap:anywhere;}
.bt-tree .svc { color:#79c0ff; font-weight:600; margin-top:8px; }
.bt-tree .chr { color:#a371f7; margin-left:20px; }
.bt-tree .uuid { color:#6e7681; }
.bt-tree .prop { color:#3fb950; font-size:11px; }
.bt-tree .val { color:#f0883e; }
.bt-tree .btn-mini { display:inline-block;margin-left:8px;padding:1px 6px;border-radius:3px;background:#21262d;border:1px solid #30363d;color:#c9d1d9;cursor:pointer;font-size:10px;}
.bt-tree .btn-mini:hover { background:#30363d; }

.bt-live { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;color:#c9d1d9;font-family:ui-monospace,monospace;}
.bt-live .cards { display:flex;gap:20px;flex-wrap:wrap;margin-bottom:12px; }
.bt-live .card { flex:1 1 200px;background:#161b22;border:1px solid #21262d;border-radius:6px;padding:12px 16px; }
.bt-live .card-title { color:#8b949e;font-size:11px;margin-bottom:4px;text-transform:uppercase;letter-spacing:1px; }
.bt-live .card-value { color:#3fb950;font-size:32px;font-weight:700;line-height:1.1; }
.bt-live .card-unit { color:#6e7681;font-size:14px;margin-left:4px;font-weight:400; }
.bt-live .card-muted { color:#6e7681;font-size:24px; }
.bt-live .spark { display:flex; align-items:flex-end; height:60px; gap:2px; padding:6px 0; }
.bt-live .spark-bar { flex:1; min-width:2px; background:#3fb950; border-radius:1px; }
.bt-live .spark-empty { color:#6e7681; font-size:11px; padding:20px 0; text-align:center; }

.bt-log { max-width:1000px;margin-bottom:16px;padding:12px 16px;background:#0d1117;border:1px solid #21262d;border-radius:6px;font-family:ui-monospace,monospace;font-size:11.5px;line-height:1.6;color:#c9d1d9;max-height:280px;overflow-y:auto;}
.bt-log-entry { display:flex; gap:8px; padding:1px 0; }
.bt-log-entry .time { color:#6e7681; flex-shrink:0; }
.bt-log-info .msg { color:#c9d1d9; }
.bt-log-ok .msg { color:#3fb950; }
.bt-log-warn .msg { color:#f0883e; }
.bt-log-err .msg { color:#f85149; }

.bt-footnote { max-width:1000px;padding:10px 14px;background:#161b22;border-left:3px solid #f0883e;border-radius:4px;color:#8b949e;font-size:12px;line-height:1.55;}
"#;

// =============================================================================
// Фазы
// =============================================================================

#[derive(Clone, Copy, PartialEq, Eq)]
enum Phase {
    Idle,
    Connecting,
    Discovering,
    Ready,
    Disconnecting,
    Disconnected,
}

impl Phase {
    fn label(&self) -> &'static str {
        match self {
            Phase::Idle => "простой",
            Phase::Connecting => "подключение",
            Phase::Discovering => "обход GATT-дерева",
            Phase::Ready => "готово",
            Phase::Disconnecting => "отключение",
            Phase::Disconnected => "отключено",
        }
    }
}

// =============================================================================
// Модели
// =============================================================================

struct CharacteristicInfo {
    js: JsValue,
    uuid: String,
    display_name: String,
    properties: Vec<&'static str>,
    value_hex: Option<String>,
}

struct ServiceInfo {
    uuid: String,
    display_name: String,
    characteristics: Vec<CharacteristicInfo>,
}

struct LogEntry {
    time: String,
    level: &'static str,
    text: String,
}

// =============================================================================
// Состояние
// =============================================================================

struct DemoState {
    session: u64,
    phase: Phase,
    error: Option<String>,
    device: Option<JsValue>,
    device_info: Option<DeviceInfo>,
    disconnect_listener_attached: bool,
    services: Vec<ServiceInfo>,
    known_devices: Vec<JsValue>,
    logs: Vec<LogEntry>,
    heart_rate: Option<u16>,
    battery: Option<u8>,
    hr_history: Vec<u16>,
    hr_handle: Option<NotificationHandle>,
}

impl DemoState {
    fn new() -> Self {
        return Self {
            session: 0,
            phase: Phase::Idle,
            error: None,
            device: None,
            device_info: None,
            disconnect_listener_attached: false,
            services: Vec::new(),
            known_devices: Vec::new(),
            logs: Vec::new(),
            heart_rate: None,
            battery: None,
            hr_history: Vec::new(),
            hr_handle: None,
        };
    }

    fn is_active(&self) -> bool {
        return matches!(
            self.phase,
            Phase::Connecting | Phase::Discovering | Phase::Disconnecting
        );
    }

    fn is_connected(&self) -> bool {
        return matches!(self.phase, Phase::Ready | Phase::Discovering);
    }
}

// =============================================================================
// Утилиты UI
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "bt-injected-style";
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

fn now_timestamp() -> String {
    let d = js_sys::Date::new_0();
    let h = d.get_hours();
    let m = d.get_minutes();
    let s = d.get_seconds();
    let ms = d.get_milliseconds();
    return format_str(format_args!("{h:02}:{m:02}:{s:02}.{ms:03}"));
}

fn push_log(st: &Rc<RefCell<DemoState>>, level: &'static str, text: String) {
    let mut s = st.borrow_mut();
    s.logs.push(LogEntry {
        time: now_timestamp(),
        level,
        text,
    });
    if s.logs.len() > 200 {
        let drop_count = s.logs.len() - 200;
        s.logs.drain(0..drop_count);
    }
}

fn render_log_html(logs: &[LogEntry]) -> String {
    let mut html = String::with_capacity(logs.len() * 100);
    for e in logs {
        html.push_str(r#"<div class='bt-log-entry bt-log-"#);
        html.push_str(e.level);
        html.push_str(r#"'>"#);
        html.push_str(r#"<span class='time'>"#);
        html.push_str(&e.time);
        html.push_str(r#"</span>"#);
        html.push_str(r#"<span class='msg'>"#);
        let escaped = e
            .text
            .replace('&', "&amp;")
            .replace('<', "&lt;")
            .replace('>', "&gt;");
        html.push_str(&escaped);
        html.push_str("</span></div>");
    }
    return html;
}

fn spark_html(history: &[u16]) -> String {
    if history.is_empty() {
        return r#"<div class='spark-empty'>нет данных — начните подписку</div>"#.to_string();
    }
    let max = history.iter().copied().max().unwrap_or(120).max(120) as f32;
    let mut html = String::from(r#"<div class='spark'>"#);
    for &v in history {
        let h = ((v as f32 / max) * 100.0).max(5.0).min(100.0);
        html.push_str(&format_str(format_args!(
            r#"<div class='spark-bar' style='height:{h:.0}%'></div>"#,
            h = h,
        )));
    }
    html.push_str("</div>");
    return html;
}

// =============================================================================
// Рендер дерева GATT
// =============================================================================

fn render_tree_html(services: &[ServiceInfo]) -> String {
    if services.is_empty() {
        return r#"<div class='muted'>Нет данных обхода. Подключитесь к устройству.</div>"#.to_string();
    }

    let mut html = String::new();
    for (sid, svc) in services.iter().enumerate() {
        html.push_str(&format_str(format_args!(
            r#"<div class='svc'>📂 {name}  <span class='uuid'>{uuid}</span></div>"#,
            name = svc.display_name,
            uuid = svc.uuid,
        )));
        if svc.characteristics.is_empty() {
            html.push_str(r#"<div class='chr muted'>  (нет характеристик)</div>"#);
            continue;
        }
        for (cid, chr) in svc.characteristics.iter().enumerate() {
            html.push_str(&format_str(format_args!(
                r#"<div class='chr'>└─ {name}  <span class='uuid'>{uuid}</span>"#,
                name = chr.display_name,
                uuid = chr.uuid,
            )));
            for p in &chr.properties {
                html.push_str(&format_str(format_args!(
                    r#" <span class='prop'>[{p}]</span>"#,
                )));
            }
            if chr.properties.contains(&"read") {
                html.push_str(&format_str(format_args!(
                    r#" <span class='btn-mini' data-read='{sid}:{cid}'>read</span>"#,
                    sid = sid,
                    cid = cid,
                )));
            }
            if chr.properties.contains(&"notify") || chr.properties.contains(&"indicate") {
                html.push_str(&format_str(format_args!(
                    r#" <span class='btn-mini' data-notify='{sid}:{cid}'>notify</span>"#,
                    sid = sid,
                    cid = cid,
                )));
            }
            if let Some(v) = &chr.value_hex {
                html.push_str(&format_str(format_args!(
                    r#" = <span class='val'>{v}</span>"#,
                    v = v,
                )));
            }
            html.push_str("</div>");
        }
    }
    return html;
}

// =============================================================================
// Полный рендер
// =============================================================================

fn render_state(doc: &Document, st: &DemoState) {
    if let Some(el) = doc.get_element_by_id("bt-stats") {
        let mut html = String::new();
        html.push_str(r#"<div><span class='lbl'>Фаза: </span><span class='val'>"#);
        html.push_str(st.phase.label());
        html.push_str("</span>");

        if let Some(info) = &st.device_info {
            html.push_str(r#" · <span class='lbl'>Устройство: </span><span class='val'>"#);
            html.push_str(if info.name.is_empty() { "(без имени)" } else { &info.name });
            html.push_str("</span>");
            html.push_str(r#" · <span class='lbl'>id: </span><span class='muted'>"#);
            html.push_str(&info.id);
            html.push_str("</span>");
            let cls = if info.connected { "ok" } else { "warn" };
            let status = if info.connected { "connected" } else { "disconnected" };
            html.push_str(&format_str(format_args!(
                r#" · <span class='{cls}'>{status}</span>"#,
                cls = cls,
                status = status,
            )));
        }
        html.push_str("</div>");

        if let Some(err) = &st.error {
            html.push_str(&format_str(format_args!(
                r#"<div class='err'>{err}</div>"#,
                err = err,
            )));
        }
        el.set_inner_html(&html);
    }

    if let Some(el) = doc.get_element_by_id("bt-device") {
        let html = match &st.device_info {
            None => String::new(),
            Some(info) => {
                let mut h = String::new();
                h.push_str(&format_str(format_args!(
                    r#"<div class='row'><span class='lbl'>id:</span><span class='val'>{id}</span></div>"#,
                    id = info.id,
                )));
                h.push_str(&format_str(format_args!(
                    r#"<div class='row'><span class='lbl'>name:</span><span class='val'>{name}</span></div>"#,
                    name = if info.name.is_empty() { "—" } else { &info.name },
                )));
                h.push_str(&format_str(format_args!(
                    r#"<div class='row'><span class='lbl'>services:</span><span class='val'>{n}</span></div>"#,
                    n = st.services.len(),
                )));
                let total_chr: usize = st.services.iter().map(|s| s.characteristics.len()).sum();
                h.push_str(&format_str(format_args!(
                    r#"<div class='row'><span class='lbl'>characteristics:</span><span class='val'>{n}</span></div>"#,
                    n = total_chr,
                )));
                h
            }
        };
        el.set_inner_html(&html);
    }

    if let Some(el) = doc.get_element_by_id("bt-tree") {
        el.set_inner_html(&render_tree_html(&st.services));
    }

    if let Some(el) = doc.get_element_by_id("bt-live") {
        let hr_inner = match st.heart_rate {
            Some(v) => format_str(format_args!(
                r#"{v}<span class='card-unit'>BPM</span>"#,
                v = v,
            )),
            None => r#"<span class='card-muted'>—</span>"#.to_string(),
        };
        let battery_inner = match st.battery {
            Some(v) => format_str(format_args!(
                r#"{v}<span class='card-unit'>%</span>"#,
                v = v,
            )),
            None => r#"<span class='card-muted'>—</span>"#.to_string(),
        };
        let spark = spark_html(&st.hr_history);

        let mut html = String::new();
        html.push_str(r#"<div class='cards'>"#);
        html.push_str(r#"<div class='card'>"#);
        html.push_str(r#"<div class='card-title'>Heart rate</div>"#);
        html.push_str(&format_str(format_args!(
            r#"<div class='card-value'>{hr}</div>"#,
            hr = hr_inner,
        )));
        html.push_str("</div>");
        html.push_str(r#"<div class='card'>"#);
        html.push_str(r#"<div class='card-title'>Battery</div>"#);
        html.push_str(&format_str(format_args!(
            r#"<div class='card-value'>{b}</div>"#,
            b = battery_inner,
        )));
        html.push_str("</div>");
        html.push_str("</div>");
        html.push_str(&spark);
        el.set_inner_html(&html);
    }

    if let Some(el) = doc.get_element_by_id("bt-log") {
        el.set_inner_html(&render_log_html(&st.logs));
        if let Ok(el) = el.dyn_into::<HtmlElement>() {
            let _ = el.set_scroll_top(el.scroll_height() as f64);
        }
    }

    let active = st.is_active();
    let connected = st.is_connected();
    let has_known = !st.known_devices.is_empty();

    set_btn(doc, "bt-connect", !active && !connected);
    set_btn(doc, "bt-connect-all", !active && !connected);
    set_btn(doc, "bt-reconnect", !active && !connected && has_known);
    set_btn(doc, "bt-disconnect", !active && connected);
    set_btn(doc, "bt-forget", !active && connected);
    set_btn(doc, "bt-refresh", !active && connected);
}

fn set_btn(doc: &Document, id: &str, enabled: bool) {
    let Some(el) = doc.get_element_by_id(id) else {
        return;
    };
    if enabled {
        let _ = el.remove_attribute("disabled");
    } else {
        let _ = el.set_attribute("disabled", "");
    }
}

// =============================================================================
// Обработчики кликов по дереву (read / notify)
// =============================================================================

fn attach_tree_handler(
    doc: &Document,
    st: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let Some(tree_el) = doc.get_element_by_id("bt-tree") else {
        return Ok(());
    };
    let tree_el: HtmlElement = tree_el.dyn_into()?;

    let doc_clone = doc.clone();
    let st_clone = st.clone();
    let closure: Closure<dyn FnMut(web_sys::Event)> =
        Closure::new(move |event: web_sys::Event| {
            let Some(target) = event.target() else { return };
            let target: HtmlElement = match target.dyn_into() {
                Ok(t) => t,
                Err(_) => return,
            };

            if let Some(read_id) = target.get_attribute("data-read") {
                let (sid, cid) = parse_pair(&read_id);
                let doc_async = doc_clone.clone();
                let st_async = st_clone.clone();
                spawn_local(async move {
                    on_read_characteristic(doc_async, st_async, sid, cid).await;
                });
            } else if let Some(notify_id) = target.get_attribute("data-notify") {
                let (sid, cid) = parse_pair(&notify_id);
                let doc_async = doc_clone.clone();
                let st_async = st_clone.clone();
                spawn_local(async move {
                    on_notify_characteristic(doc_async, st_async, sid, cid).await;
                });
            }
        });
    tree_el.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    return Ok(());
}

fn parse_pair(s: &str) -> (usize, usize) {
    let mut parts = s.split(':');
    let sid = parts.next().and_then(|x| x.parse::<usize>().ok()).unwrap_or(0);
    let cid = parts.next().and_then(|x| x.parse::<usize>().ok()).unwrap_or(0);
    return (sid, cid);
}

// =============================================================================
// Действия
// =============================================================================

fn get_char_js(st: &Rc<RefCell<DemoState>>, sid: usize, cid: usize) -> Option<JsValue> {
    let s = st.borrow();
    let svc = s.services.get(sid)?;
    let chr = svc.characteristics.get(cid)?;
    return Some(chr.js.clone());
}

async fn on_read_characteristic(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    sid: usize,
    cid: usize,
) {
    let Some(char_js) = get_char_js(&st, sid, cid) else {
        return;
    };
    let uuid = read_characteristic_uuid(&char_js);
    let name = format_characteristic_uuid(&uuid);

    push_log(&st, "info", format_str(format_args!("readValue: {name}")));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    match read_value(&char_js).await {
        Ok(bytes) => {
            let hex = hex_bytes(&bytes);
            push_log(
                &st,
                "ok",
                format_str(format_args!("{name} = {hex}")),
            );
            if uuid.to_lowercase().contains("2a19") {
                if let Some(b) = parse_battery_level(&bytes) {
                    let mut s = st.borrow_mut();
                    s.battery = Some(b);
                }
            }
            {
                let mut s = st.borrow_mut();
                if let Some(svc) = s.services.get_mut(sid) {
                    if let Some(chr) = svc.characteristics.get_mut(cid) {
                        chr.value_hex = Some(hex);
                    }
                }
            }
        }
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("readValue: {e}")));
        }
    }
    render_state(&doc, &st.borrow());
}

async fn on_notify_characteristic(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    sid: usize,
    cid: usize,
) {
    let Some(char_js) = get_char_js(&st, sid, cid) else {
        return;
    };
    let uuid = read_characteristic_uuid(&char_js);
    let name = format_characteristic_uuid(&uuid);

    push_log(&st, "info", format_str(format_args!(
        "startNotifications: {name}"
    )));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let is_hr = uuid.to_lowercase().contains("2a37");

    if is_hr {
        let st_cb = st.clone();
        let doc_cb = doc.clone();
        let handle = subscribe_value_changed(&char_js, move |bytes| {
            let hr = parse_heart_rate(&bytes);
            {
                let mut s = st_cb.borrow_mut();
                s.heart_rate = hr;
                if let Some(v) = hr {
                    s.hr_history.push(v);
                    if s.hr_history.len() > 60 {
                        let drop_count = s.hr_history.len() - 60;
                        s.hr_history.drain(0..drop_count);
                    }
                }
            }
            {
                let mut s = st_cb.borrow_mut();
                if s.hr_history.len() % 5 == 0 {
                    let bytes_hex = hex_bytes(&bytes);
                    s.logs.push(LogEntry {
                        time: now_timestamp(),
                        level: "ok",
                        text: format_str(format_args!(
                            "HR notify: {} = {bytes_hex}",
                            hr.map(|v| v.to_string())
                                .unwrap_or_else(|| String::from("?")),
                        )),
                    });
                    if s.logs.len() > 200 {
                        let drop_count = s.logs.len() - 200;
                        s.logs.drain(0..drop_count);
                    }
                }
            }
            render_state(&doc_cb, &st_cb.borrow());
        });

        match handle {
            Ok(h) => {
                let mut s = st.borrow_mut();
                s.hr_handle = Some(h);
            }
            Err(e) => {
                push_log(&st, "err", format_str(format_args!("subscribe: {e}")));
                render_state(&doc, &st.borrow());
                return;
            }
        }
    }

    match start_notifications(&char_js).await {
        Ok(()) => {
            push_log(
                &st,
                "ok",
                format_str(format_args!("{name} — подписка активна")),
            );
        }
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("startNotifications: {e}")));
        }
    }
    render_state(&doc, &st.borrow());
}

// =============================================================================
// Диагностика
// =============================================================================

/// Выводит в лог подсказку, если обход GATT дал пустой или
/// бесполезный результат. Это типичный случай, когда устройство —
/// только Advertiser (например, nRF Connect без GATT Server),
/// либо это служебное BLE-устройство (наушники).
fn log_gatt_diagnostic(st: &Rc<RefCell<DemoState>>, services: &[ServiceInfo]) {
    if services.is_empty() {
        push_log(
            st,
            "warn",
            String::from(
                "Устройство не предоставило ни одного primary-сервиса. \
                 Скорее всего, это Advertiser без GATT-сервера. \
                 В nRF Connect запустите Configure GATT Server, а не только Advertiser.",
            ),
        );
        return;
    }

    // Проверяем, остались ли только служебные сервисы 0x1800/0x1801.
    let all_utility = services.iter().all(|s| {
        let u = s.uuid.to_lowercase();
        u.contains("1800") || u.contains("1801")
    });

    if all_utility {
        push_log(
            st,
            "warn",
            String::from(
                "В GATT-дереве только служебные сервисы (Generic Access / \
                 Generic Attribute). Полезных данных нет. Это типично для \
                 Bluetooth Classic-устройств (наушники, клавиатуры) в их \
                 BLE-режиме. Попробуйте фитнес-трекер, пульсометр или \
                 настроенный GATT Server в nRF Connect.",
            ),
        );
    }
}

// =============================================================================
// Обход GATT-дерева
// =============================================================================

async fn discover_gatt(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    session: u64,
    gatt: JsValue,
) {
    push_log(&st, "info", String::from("getPrimaryServices()"));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let services = match get_primary_services(&gatt).await {
        Ok(s) => s,
        Err(e) => {
            push_log(&st, "err", format_str(format_args!("getPrimaryServices: {e}")));
            push_log(
                &st,
                "warn",
                String::from(
                    "Подсказка: устройство подключилось, но не имеет GATT-сервера. \
                     В nRF Connect запустите Configure GATT Server (не только Advertiser), \
                     либо используйте реальное BLE-устройство с сервисами (пульсометр, \
                     фитнес-трекер, BLE-термометр).",
                ),
            );
            render_state(&doc, &st.borrow());
            return;
        }
    };

    push_log(
        &st,
        "ok",
        format_str(format_args!("Найдено сервисов: {}", services.len())),
    );
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    for (i, svc_js) in services.iter().enumerate() {
        if st.borrow().session != session {
            return;
        }

        let uuid = read_service_uuid(svc_js);
        let display_name = format_service_uuid(&uuid);

        push_log(
            &st,
            "info",
            format_str(format_args!("[{}] {display_name}", i + 1)),
        );
        render_state(&doc, &st.borrow());
        TimeoutFuture::new(0).await;

        let chars = match get_characteristics(svc_js).await {
            Ok(c) => c,
            Err(e) => {
                push_log(
                    &st,
                    "warn",
                    format_str(format_args!(
                        "getCharacteristics({display_name}): {e}"
                    )),
                );
                let mut s = st.borrow_mut();
                s.services.push(ServiceInfo {
                    uuid: uuid.clone(),
                    display_name: display_name.clone(),
                    characteristics: Vec::new(),
                });
                drop(s);
                render_state(&doc, &st.borrow());
                continue;
            }
        };

        let mut chr_infos = Vec::with_capacity(chars.len());
        for chr_js in chars.iter() {
            if st.borrow().session != session {
                return;
            }
            let chr_uuid = read_characteristic_uuid(chr_js);
            let chr_name = format_characteristic_uuid(&chr_uuid);
            let properties = read_characteristic_properties(chr_js);
            chr_infos.push(CharacteristicInfo {
                js: chr_js.clone(),
                uuid: chr_uuid,
                display_name: chr_name,
                properties,
                value_hex: None,
            });
        }

        let mut s = st.borrow_mut();
        s.services.push(ServiceInfo {
            uuid: uuid.clone(),
            display_name: display_name.clone(),
            characteristics: chr_infos,
        });
        drop(s);

        push_log(
            &st,
            "ok",
            format_str(format_args!(
                "{display_name}: {} характеристик",
                chars.len()
            )),
        );
        render_state(&doc, &st.borrow());
        TimeoutFuture::new(0).await;
    }

    // Диагностика: пусто или только служебные сервисы.
    {
        let s = st.borrow();
        log_gatt_diagnostic(&st, &s.services);
    }

    try_read_battery(&doc, &st, session).await;

    {
        let mut s = st.borrow_mut();
        if s.session == session {
            s.phase = Phase::Ready;
        }
    }
    push_log(&st, "ok", String::from("Обход завершён"));
    render_state(&doc, &st.borrow());
}

async fn try_read_battery(doc: &Document, st: &Rc<RefCell<DemoState>>, session: u64) {
    let char_js: Option<JsValue> = {
        let s = st.borrow();
        let mut found = None;
        for svc in &s.services {
            let svc_low = svc.uuid.to_lowercase();
            if svc_low.contains("180f") || svc.uuid.eq_ignore_ascii_case(SERVICE_BATTERY) {
                for chr in &svc.characteristics {
                    let chr_low = chr.uuid.to_lowercase();
                    if chr_low.contains("2a19") {
                        found = Some(chr.js.clone());
                        break;
                    }
                }
                if found.is_some() {
                    break;
                }
            }
        }
        found
    };

    let Some(bat_js) = char_js else {
        return;
    };
    if st.borrow().session != session {
        return;
    }

    push_log(st, "info", String::from("Первичное чтение battery_level"));
    if let Ok(bytes) = read_value(&bat_js).await {
        if let Some(b) = parse_battery_level(&bytes) {
            let mut s = st.borrow_mut();
            s.battery = Some(b);
            s.logs.push(LogEntry {
                time: now_timestamp(),
                level: "ok",
                text: format_str(format_args!("battery_level = {b}%")),
            });
        }
    }
    render_state(doc, &st.borrow());
}

// =============================================================================
// Действия: подключение, отключение, reconnect, forget
// =============================================================================

/// Универсальный запуск подключения с любыми `RequestDeviceOptions`.
///
/// ## User gesture
///
/// UI обновляется **до** `request_device` — иначе user activation
/// потратится на `TimeoutFuture::new(0).await`, и браузер вернёт
/// `SecurityError`.
async fn on_connect_new(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    options: RequestDeviceOptions,
) {
    if st.borrow().is_active() {
        return;
    }

    // Запоминаем, какой режим — для логирования.
    let log_text = if options.accept_all {
        format_str(format_args!(
            "requestDevice(acceptAllDevices: true, optionalServices: {:?})",
            options.optional_services,
        ))
    } else {
        format_str(format_args!(
            "requestDevice(filters: {:?})",
            options.filters_services,
        ))
    };

    {
        let mut s = st.borrow_mut();
        s.session = s.session.wrapping_add(1);
        s.phase = Phase::Connecting;
        s.error = None;
        s.services.clear();
        s.heart_rate = None;
        s.battery = None;
        s.hr_history.clear();
        s.hr_handle = None;
    }
    push_log(&st, "info", log_text);
    render_state(&doc, &st.borrow());

    // ВАЖНО: никаких await до requestDevice.
    let device = match request_device(options).await {
        Ok(d) => d,
        Err(e) => {
            {
                let mut s = st.borrow_mut();
                s.phase = Phase::Idle;
                s.error = Some(e.clone());
            }
            push_log(&st, "err", e);
            render_state(&doc, &st.borrow());
            refresh_known_devices(&st).await;
            return;
        }
    };

    let info = read_device_info(&device);
    push_log(
        &st,
        "ok",
        format_str(format_args!(
            "Устройство выбрано: {}",
            if info.name.is_empty() { "(без имени)" } else { &info.name }
        )),
    );
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    connect_and_discover(doc, st, device).await;
}

async fn on_reconnect_known(doc: Document, st: Rc<RefCell<DemoState>>, idx: usize) {
    let device = {
        let s = st.borrow();
        s.known_devices.get(idx).cloned()
    };
    let Some(device) = device else {
        return;
    };

    {
        let mut s = st.borrow_mut();
        s.session = s.session.wrapping_add(1);
        s.phase = Phase::Connecting;
        s.error = None;
        s.services.clear();
        s.heart_rate = None;
        s.battery = None;
        s.hr_history.clear();
        s.hr_handle = None;
    }
    push_log(&st, "info", String::from("Reconnect из getDevices()"));
    render_state(&doc, &st.borrow());

    connect_and_discover(doc, st, device).await;
}

async fn connect_and_discover(
    doc: Document,
    st: Rc<RefCell<DemoState>>,
    device: JsValue,
) {
    let session = st.borrow().session;

    let need_listener = {
        let s = st.borrow();
        !s.disconnect_listener_attached
    };
    if need_listener {
        let st_cb = st.clone();
        let doc_cb = doc.clone();
        let result = subscribe_disconnect(&device, move || {
            {
                let mut s = st_cb.borrow_mut();
                s.phase = Phase::Disconnected;
                s.device = None;
                s.device_info = None;
                s.services.clear();
                s.heart_rate = None;
                s.battery = None;
                s.hr_history.clear();
                s.hr_handle = None;
                s.disconnect_listener_attached = false;
            }
            push_log(&st_cb, "warn", String::from("Устройство отключено"));
            render_state(&doc_cb, &st_cb.borrow());
        });
        if result.is_ok() {
            let mut s = st.borrow_mut();
            s.disconnect_listener_attached = true;
        }
    }

    push_log(&st, "info", String::from("gatt.connect()"));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let gatt = match connect(&device).await {
        Ok(g) => g,
        Err(e) => {
            {
                let mut s = st.borrow_mut();
                if s.session == session {
                    s.phase = Phase::Idle;
                    s.error = Some(e.clone());
                }
            }
            push_log(&st, "err", format_str(format_args!("connect: {e}")));
            render_state(&doc, &st.borrow());
            return;
        }
    };

    let info = read_device_info(&device);
    push_log(&st, "ok", String::from("Подключено"));
    {
        let mut s = st.borrow_mut();
        s.device = Some(device.clone());
        s.device_info = Some(info);
        s.phase = Phase::Discovering;
        s.disconnect_listener_attached = true;
    }
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    discover_gatt(doc, st, session, gatt).await;
}

async fn on_disconnect(doc: Document, st: Rc<RefCell<DemoState>>) {
    let device = {
        let s = st.borrow();
        s.device.clone()
    };
    let Some(device) = device else { return };

    {
        let mut s = st.borrow_mut();
        s.phase = Phase::Disconnecting;
        s.disconnect_listener_attached = false;
    }
    push_log(&st, "info", String::from("gatt.disconnect()"));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let _ = disconnect(&device);

    {
        let mut s = st.borrow_mut();
        s.device = None;
        s.device_info = None;
        s.services.clear();
        s.heart_rate = None;
        s.battery = None;
        s.hr_history.clear();
        s.hr_handle = None;
        s.phase = Phase::Disconnected;
    }
    push_log(&st, "ok", String::from("Отключено"));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    refresh_known_devices(&st).await;
    render_state(&doc, &st.borrow());
}

async fn on_forget(doc: Document, st: Rc<RefCell<DemoState>>) {
    let device = {
        let s = st.borrow();
        s.device.clone()
    };
    let Some(device) = device else { return };

    let forget_fn = Reflect::get(&device, &JsValue::from_str("forget"))
        .ok()
        .and_then(|v| v.dyn_into::<js_sys::Function>().ok());

    let Some(forget_fn) = forget_fn else {
        push_log(&st, "warn", String::from("device.forget() недоступен"));
        render_state(&doc, &st.borrow());
        return;
    };

    push_log(&st, "info", String::from("device.forget()"));
    render_state(&doc, &st.borrow());
    TimeoutFuture::new(0).await;

    let promise = forget_fn
        .call0(&device)
        .ok()
        .and_then(|v| v.dyn_into::<js_sys::Promise>().ok());
    let Some(promise) = promise else {
        return;
    };
    let _ = wasm_bindgen_futures::JsFuture::from(promise).await;

    push_log(&st, "ok", String::from("Разрешение отозвано"));
    on_disconnect(doc, st).await;
}

async fn refresh_known_devices(st: &Rc<RefCell<DemoState>>) {
    let devices = get_known_devices().await.unwrap_or_default();
    let mut names = Vec::new();
    for d in &devices {
        let info = read_device_info(d);
        names.push(if info.name.is_empty() { info.id.clone() } else { info.name });
    }
    {
        let mut s = st.borrow_mut();
        s.known_devices = devices;
    }
    push_log(
        st,
        "info",
        format_str(format_args!("known devices: {}", names.len())),
    );
}

async fn on_refresh_values(doc: Document, st: Rc<RefCell<DemoState>>) {
    let chars: Vec<(usize, usize, JsValue, String)> = {
        let s = st.borrow();
        let mut out = Vec::new();
        for (sid, svc) in s.services.iter().enumerate() {
            for (cid, chr) in svc.characteristics.iter().enumerate() {
                if chr.properties.contains(&"read") {
                    out.push((sid, cid, chr.js.clone(), chr.display_name.clone()));
                }
            }
        }
        out
    };

    for (sid, cid, js, name) in chars {
        match read_value(&js).await {
            Ok(bytes) => {
                let hex = hex_bytes(&bytes);
                {
                    let mut s = st.borrow_mut();
                    if let Some(svc) = s.services.get_mut(sid) {
                        if let Some(chr) = svc.characteristics.get_mut(cid) {
                            chr.value_hex = Some(hex.clone());
                        }
                    }
                    let uuid_low = s.services[sid].characteristics[cid].uuid.to_lowercase();
                    if uuid_low.contains("2a19") {
                        if let Some(b) = parse_battery_level(&bytes) {
                            s.battery = Some(b);
                        }
                    } else if uuid_low.contains("2a37") {
                        if let Some(v) = parse_heart_rate(&bytes) {
                            s.heart_rate = Some(v);
                        }
                    }
                }
                push_log(
                    &st,
                    "ok",
                    format_str(format_args!("{name} = {hex}")),
                );
            }
            Err(e) => {
                push_log(
                    &st,
                    "warn",
                    format_str(format_args!("{name}: {e}")),
                );
            }
        }
        render_state(&doc, &st.borrow());
        TimeoutFuture::new(0).await;
    }
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

    // ---- Заголовок ----
    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("Глава 45 — Web Bluetooth"));
    root.append_child(&h2)?;

    // ---- Описание ----
    let desc = document.create_element("div")?;
    desc.set_class_name("bt-desc");
    desc.set_inner_html(
        "<p class='bt-desc-line'><strong>Web Bluetooth</strong> — прямое подключение к BLE-устройствам из браузера. \
         Работает только в <strong>secure context</strong> (HTTPS или localhost) и требует <strong>user gesture</strong> — \
         вызов <code>requestDevice</code> должен быть из клика.</p>\
         <p class='bt-desc-line'><strong>GATT-дерево.</strong> После подключения: <code>gatt → services → characteristics</code>. \
         У характеристики — свойства (<code>read</code>, <code>write</code>, <code>notify</code>) и значение (DataView с байтами).</p>\
         <p class='bt-desc-line'><strong>Два режима поиска.</strong> \
         «Подключить пульсометр» фильтрует по <code>heart_rate</code>. \
         «Показать все устройства» использует <code>acceptAllDevices: true</code> и показывает всё BLE-окружение, \
         но доступ к данным — только к указанным в <code>optionalServices</code>.</p>",
    );
    root.append_child(&desc)?;

    // ---- Проверка поддержки ----
    if !has_web_bluetooth() {
        let note = document.create_element("div")?;
        note.set_class_name("bt-note bt-note-err");
        note.set_inner_html(
            "<strong>Web Bluetooth недоступен.</strong> Требуется Chrome 56+, Edge 79+ или Opera 43+. \
             <strong>Linux официально не поддерживается</strong> (только Windows, macOS, ChromeOS, Android). \
             В Firefox включите <code>dom.bluetooth.enabled</code> в about:config. Safari не поддерживает.",
        );
        root.append_child(&note)?;
        return Ok(());
    }
    if !is_secure_context() {
        let note = document.create_element("div")?;
        note.set_class_name("bt-note bt-note-err");
        note.set_inner_html(
            "<strong>Не secure context.</strong> Web Bluetooth работает только на HTTPS или localhost. \
             Откройте страницу через <code>http://localhost</code> или настройте HTTPS.",
        );
        root.append_child(&note)?;
        return Ok(());
    }

    // ---- Controls ----
    let controls = document.create_element("div")?;
    controls.set_class_name("bt-block");
    controls.set_inner_html(
        "<button class='bt-btn bt-btn-primary' id='bt-connect'>Подключить пульсометр</button>\
         <button class='bt-btn bt-btn-secondary' id='bt-connect-all' \
           title='Показать все BLE-устройства. Доступ к данным — только к heart_rate и battery_service.'>\
           Показать все устройства\
         </button>\
         <select class='bt-select' id='bt-known' disabled></select>\
         <button class='bt-btn bt-btn-secondary' id='bt-reconnect' disabled>Переподключить</button>\
         <button class='bt-btn' id='bt-refresh' disabled>Обновить значения</button>\
         <button class='bt-btn bt-btn-danger' id='bt-disconnect' disabled>Отключить</button>\
         <button class='bt-btn' id='bt-forget' disabled>Забыть</button>",
    );
    root.append_child(&controls)?;

    // ---- Stats ----
    let stats = document.create_element("div")?;
    stats.set_class_name("bt-stats");
    stats.set_id("bt-stats");
    stats.set_inner_html("<div><span class='lbl'>Фаза: </span><span class='val'>простой</span></div>");
    root.append_child(&stats)?;

    // ---- Device info ----
    let device_el = document.create_element("div")?;
    device_el.set_class_name("bt-device");
    device_el.set_id("bt-device");
    root.append_child(&device_el)?;

    // ---- GATT tree ----
    let tree = document.create_element("div")?;
    tree.set_class_name("bt-tree");
    tree.set_id("bt-tree");
    root.append_child(&tree)?;

    // ---- Live values ----
    let live = document.create_element("div")?;
    live.set_class_name("bt-live");
    live.set_id("bt-live");
    root.append_child(&live)?;

    // ---- Log ----
    let log = document.create_element("div")?;
    log.set_class_name("bt-log");
    log.set_id("bt-log");
    root.append_child(&log)?;

    // ---- Footnote ----
    let footnote = document.create_element("p")?;
    footnote.set_class_name("bt-footnote");
    footnote.set_text_content(Some(
        "Правило: requestDevice работает только из обработчика клика. \
         Chrome показывает все BLE-устройства только через acceptAllDevices: true, \
         но при этом обязательно нужно указать optionalServices — иначе SecurityError. \
         Все найденные сервисы сохраняются живыми — можно читать и подписываться в любой момент. \
         ВАЖНО: устройство-«Advertiser» (только вещающее) не даёт GATT-сервисов. \
         Для теста через nRF Connect запускайте Configure GATT Server, а не только Advertiser.",
    ));
    root.append_child(&footnote)?;

    // ---- Состояние ----
    let state = Rc::new(RefCell::new(DemoState::new()));

    // Загружаем known devices сразу.
    {
        let st = state.clone();
        spawn_local(async move {
            refresh_known_devices(&st).await;
        });
    }

    // ---- Клик по дереву (read / notify) ----
    attach_tree_handler(&document, state.clone())?;

    // ---- Кнопка «Подключить пульсометр» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("bt-connect")
            .ok_or_else(|| JsValue::from_str("no #bt-connect"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            // Ищем только устройства с heart_rate.
            let opts = RequestDeviceOptions::for_service("heart_rate");
            spawn_local(async move {
                on_connect_new(doc_async, st_async, opts).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопка «Показать все устройства» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("bt-connect-all")
            .ok_or_else(|| JsValue::from_str("no #bt-connect-all"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            // acceptAllDevices: true с явным списком optionalServices.
            // Chrome требует непустой optionalServices, иначе SecurityError.
            let opts = RequestDeviceOptions::all_devices(vec![
                "heart_rate".to_string(),
                "battery_service".to_string(),
                "device_information".to_string(),
                "generic_access".to_string(),
                "generic_attribute".to_string(),
            ]);
            spawn_local(async move {
                on_connect_new(doc_async, st_async, opts).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопка «Переподключить» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("bt-reconnect")
            .ok_or_else(|| JsValue::from_str("no #bt-reconnect"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let select_el = document
            .get_element_by_id("bt-known")
            .ok_or_else(|| JsValue::from_str("no #bt-known"))?;
        let select: web_sys::HtmlSelectElement = select_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let idx = select.selected_index();
            if idx < 0 {
                return;
            }
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                on_reconnect_known(doc_async, st_async, idx as usize).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопка «Отключить» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("bt-disconnect")
            .ok_or_else(|| JsValue::from_str("no #bt-disconnect"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                on_disconnect(doc_async, st_async).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопка «Забыть» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("bt-forget")
            .ok_or_else(|| JsValue::from_str("no #bt-forget"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                on_forget(doc_async, st_async).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Кнопка «Обновить значения» ----
    {
        let doc = document.clone();
        let st = state.clone();
        let btn_el = document
            .get_element_by_id("bt-refresh")
            .ok_or_else(|| JsValue::from_str("no #bt-refresh"))?;
        let btn: HtmlElement = btn_el.dyn_into()?;
        let closure: Closure<dyn FnMut()> = Closure::new(move || {
            let doc_async = doc.clone();
            let st_async = st.clone();
            spawn_local(async move {
                on_refresh_values(doc_async, st_async).await;
            });
        });
        btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    // ---- Первичная отрисовка ----
    render_state(&document, &state.borrow());

    // ---- Синхронизация `<select>` с known devices ----
    {
        let doc = document.clone();
        let st = state.clone();
        spawn_local(async move {
            loop {
                TimeoutFuture::new(2000).await;
                let (devices, names): (Vec<JsValue>, Vec<String>) = {
                    let s = st.borrow();
                    let mut names = Vec::new();
                    for d in &s.known_devices {
                        let info = read_device_info(d);
                        names.push(if info.name.is_empty() {
                            info.id.clone()
                        } else {
                            info.name
                        });
                    }
                    (s.known_devices.clone(), names)
                };

                let Some(sel_el) = doc.get_element_by_id("bt-known") else {
                    continue;
                };
                let Ok(sel) = sel_el.dyn_into::<web_sys::HtmlSelectElement>() else {
                    continue;
                };

                if sel.length() != devices.len() as u32 {
                    sel.set_inner_html("");
                    for (i, name) in names.iter().enumerate() {
                        let opt = doc.create_element("option").ok();
                        if let Some(opt) = opt {
                            opt.set_attribute("value", &format_str(format_args!("{i}"))).ok();
                            opt.set_text_content(Some(name));
                            let _ = sel.append_child(&opt);
                        }
                    }
                }
                if devices.is_empty() {
                    sel.set_disabled(true);
                } else {
                    sel.set_disabled(false);
                }
            }
        });
    }

    return Ok(());
}
```

---

## 45.5. Добавление в `src/lib.rs`

```rust
pub mod app;
pub mod bluetooth_gatt;      // ← 45
pub mod bluetooth_utils;     // ← 45
pub mod container_parser;
pub mod mp4_demuxer;
pub mod mp4_muxer;
pub mod video_filters;
pub mod video_frame;
pub mod webgpu_compute;
pub mod webgpu_utils;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> Result<(), JsValue> {
    app::run()
}
```

---

## 45.6. Что НЕ добавлено и почему

| Что | Почему не добавлено |
|-----|---------------------|
| `web-sys` фичи `Bluetooth`, `BluetoothDevice`, `BluetoothRemoteGATT*` | Требуют `--cfg=web_sys_unstable_apis`. Reflect-путь стабилен и не ломает сборку |
| `serde` для девайсов | Избыточно: 3 поля у DeviceInfo, парсим через `Reflect::get` |
| `web-bluetooth` crate | Сторонняя обёртка. Мы даём свой минимальный слой, чтобы студент видел каждый вызов API |
| `futures` / `async-stream` | `JsFuture::from(promise).await` достаточно |
| Permissions API | `navigator.permissions.query({name:'bluetooth'})` deprecated; реальная проверка — сам `requestDevice` |

---

## 45.7. Частые ошибки и их диагностика

Раздел основан на реальных граблях, выявленных в ходе отладки этой главы.

### 45.7.1. Linux: Web Bluetooth не работает официально

**Симптом:** `navigator.bluetooth` есть, но `requestDevice` не находит ничего, либо панели «Web Bluetooth» нет в DevTools.

**Причина:** Chrome официально не поддерживает Web Bluetooth на Linux. Флаг `#enable-web-bluetooth` находится в разделе *Unavailable* с пометкой «Not available on your platform». Панели эмуляции в DevTools тоже нет.

**Что делать:** тестировать на Windows/macOS/ChromeOS/Android, либо использовать реальное BLE-устройство через внешний USB-Bluetooth-донгл на Windows.

### 45.7.2. Наушники и телефон не подключаются

**Симптом:** `requestDevice` возвращает `NotFoundError`, либо находит устройство, но `getPrimaryServices` возвращает `NotFoundError`.

**Причина:** Web Bluetooth работает только с **BLE + GATT**. Наушники — это Bluetooth Classic (A2DP/HFP), их BLE-режим (если есть) содержит только служебные сервисы `0x1800` (Generic Access) и `0x1801` (Generic Attribute). Телефон как периферия тоже не подходит — на нём нет GATT-сервера, если не запущено специальное приложение.

**Что подходит:** пульсометры (Polar H10, CooSpo H6), фитнес-трекеры (Mi Band), BLE-термометры (Xiaomi Mi Temperature), умные весы, BLE-лампочки.

### 45.7.3. nRF Connect Advertiser ≠ GATT Server

**Симптом:** `requestDevice` находит устройство `TestHR`, `connect()` проходит, а `getPrimaryServices()` возвращает `NotFoundError`.

**Причина:** Advertiser **только вещает** пакет с UUID `180D`. Внутри телефона нет GATT-сервера с этим сервисом. Chrome подключается — и обнаруживает пустоту.

**Что делать:** в nRF Connect запустить **Configure GATT Server**, добавить сервис `Heart Rate` (`0x180D`) с характеристикой `Heart Rate Measurement` (`0x2A37`), и только потом включить Advertiser с тем же UUID.

### 45.7.4. `NetworkError` при `gatt.connect()`

**Симптом:** устройство появилось в диалоге, но `connect()` падает с `NetworkError: Connection Error: Connection attempt failed`.

**Причины:**
- Устройство использует **Extended Advertising** (BLE 5.0), а ваш BT-адаптер его не переваривает. В nRF Connect → Advertiser → `Advertising Type` → переключите на **`Legacy Connectable`**.
- Устройство не помечено как **Connectable**. В Advertiser обязательно поставьте галочку `Connectable`.
- Слишком большое расстояние или помехи. Поднесите телефон на 20-30 см.
- Драйвер Bluetooth устаревший. Обновите с сайта производителя ноутбука.

### 45.7.5. `acceptAllDevices: true` без `optionalServices` → `SecurityError`

**Симптом:** `requestDevice({ acceptAllDevices: true })` падает с `SecurityError`.

**Причина:** Chrome требует явного `optionalServices`, даже если вы не собираетесь читать конкретные сервисы. Это защита от неявного доступа.

**Что делать:** наш `RequestDeviceOptions::all_devices(vec![...])` в `bluetooth_utils.rs` решает это автоматически — но если пишете JS вручную, не забудьте `optionalServices: ['heart_rate', 'battery_service']`.

### 45.7.6. Диагностика в логе демо

В `discover_gatt` встроены автоматические подсказки:

- `NotFoundError` от `getPrimaryServices` → в логе появляется: *«Устройство подключилось, но не имеет GATT-сервера. В nRF Connect запустите Configure GATT Server»*.
- Пустой список сервисов → то же сообщение.
- Только сервисы `0x1800` / `0x1801` → в логе: *«В GATT-дереве только служебные сервисы. Это типично для Bluetooth Classic-устройств (наушники) в их BLE-режиме»*.

Это снимает большую часть вопросов у студентов, впервые сталкивающихся с Web Bluetooth.

---

## Итог

Все исходные модули компилируются без единой внешней фичи, без `#[cfg(web_sys_unstable_apis)]`. Все ошибки `DOMException` разбираются в человекочитаемые сообщения. Демо-UI содержит встроенные подсказки для типичных ошибочных сценариев (Advertiser без GATT Server, наушники с пустым BLE-режимом, отсутствие BLE-адаптера).

**Для тестирования без железа:**
- Windows/macOS/ChromeOS/Android → DevTools → More tools → **Web Bluetooth** (эмулятор).
- Android → **nRF Connect** → **Configure GATT Server** + **Advertiser** (Legacy Connectable).
- Любая платформа → реальное BLE-устройство (пульсометр, фитнес-трекер, термометр).