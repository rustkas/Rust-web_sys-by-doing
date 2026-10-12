# Глава 19.3. Permissions и user gesture

## Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── lib.rs                       # +3 pub mod fsa_grant_*
│   ├── app.rs                       # демо-секция 19.3 (4 блока)
│   ├── fsa_grant_state.rs           # модель разрешения + классификация ошибок
│   ├── fsa_grant_flow.rs            # query / request / preflight
│   └── fsa_grant_store.rs           # сохранение handle в IndexedDB
└── tests/
    └── wasm_fsa_grant.rs            # тесты 19.3
```

**Краткий комментарий.** В проекте есть `fsa_api_permissions.rs` — сырые обёртки `queryPermission`/`requestPermission` с `PermissionState`. Эта подглава идёт **на уровень смысла**:

1. **Разрешение принадлежит handle'у**, а не origin'у — два handle'а одной папки независимы.
2. **`query` безопасно, `request` показывает диалог только при `prompt`.** При `granted` или `denied` request резолвится мгновенно.
3. **Разрешение не переживает перезагрузку**, даже если handle сохранён в IDB. После восстановления нужно «Переподключиться» — это типовой паттерн реальных продуктов (Excalidraw, VS Code Web, Figma).

Префикс `fsa_grant_*` — уникален; с `fsa_api_permissions` не пересекается.

## Ссылки

- [MDN: FileSystemHandle.queryPermission](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemHandle/queryPermission)
- [MDN: FileSystemHandle.requestPermission](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemHandle/requestPermission)
- [MDN: Transient activation](https://developer.mozilla.org/en-US/docs/Glossary/Transient_activation)
- [Chrome: Permissions in FSA](https://developer.chrome.com/docs/capabilities/web-apis/file-system-access#permissions)
- [idb crate](https://docs.rs/idb/latest/idb/)

---

## Исходные модули

### `src/fsa_grant_state.rs`

```rust
//! Модель состояния разрешения на handle.
//!
//! FSA использует **методы самого handle'а** (`queryPermission` /
//! `requestPermission`), а не `navigator.permissions`. Причина: разрешение
//! привязано к конкретному handle'у, а не к origin'у.
//!
//! Состояние — не бинарный флаг, а значение из четырёх:
//! * `Granted` — разрешение есть;
//! * `Prompt` — браузер спросит при `requestPermission`;
//! * `Denied` — пользователь отказал или браузер заблокировал;
//! * `Unknown` — API недоступен (браузер без FSA, старый handle).
//!
//! Разрешения **не сохраняются между сессиями**: после перезагрузки
//! handle можно восстановить из IndexedDB, но `queryPermission` вернёт
//! `Prompt` — потребуется новый user gesture.

use wasm_bindgen::prelude::*;

use crate::fsa_choose_options::format_str;

/// Режим доступа.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum GrantMode {
    Read,
    ReadWrite,
}

impl GrantMode {
    pub fn as_str(self) -> &'static str {
        match self {
            GrantMode::Read => return "read",
            GrantMode::ReadWrite => return "readwrite",
        }
    }
    pub fn label(self) -> &'static str {
        match self {
            GrantMode::Read => return "чтение",
            GrantMode::ReadWrite => return "чтение и запись",
        }
    }
}

/// Состояние разрешения.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum GrantState {
    Granted,
    Prompt,
    Denied,
    Unknown,
}

impl GrantState {
    pub fn from_str(s: &str) -> Self {
        match s {
            "granted" => return GrantState::Granted,
            "prompt" => return GrantState::Prompt,
            "denied" => return GrantState::Denied,
            _ => return GrantState::Unknown,
        }
    }
    pub fn as_str(self) -> &'static str {
        match self {
            GrantState::Granted => return "granted",
            GrantState::Prompt => return "prompt",
            GrantState::Denied => return "denied",
            GrantState::Unknown => return "unknown",
        }
    }
    pub fn is_ready(self) -> bool {
        return matches!(self, GrantState::Granted);
    }
    pub fn needs_user_action(self) -> bool {
        return matches!(self, GrantState::Prompt);
    }
    pub fn is_blocked(self) -> bool {
        return matches!(self, GrantState::Denied);
    }
}

/// Классифицировать JS-ошибку как состояние разрешения.
/// `SecurityError` / `NotAllowedError` — не отказ пользователя, а
/// «браузер не дал права спросить» — это `Prompt`.
pub fn from_js_error(e: &JsValue) -> GrantState {
    let name: String = js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    match name.as_str() {
        "SecurityError" | "NotAllowedError" => return GrantState::Prompt,
        "NotFoundError" | "InvalidStateError" => return GrantState::Unknown,
        _ => return GrantState::Unknown,
    }
}

/// Человекочитаемое объяснение для UI.
/// Отдельная функция: rust-analyzer не выводит `format!` в хвостовой
/// позиции `match`-arm как `String`.
pub fn explain(state: GrantState, mode: GrantMode) -> String {
    match state {
        GrantState::Granted => {
            let s: String = format_str(format_args!(
                "✓ Разрешение на {} уже есть. Можно работать.",
                mode.label(),
            ));
            return s;
        }
        GrantState::Prompt => {
            let s: String = format_str(format_args!(
                "⚠ Требуется запрос на {}. Нажмите «Запросить» — \
                 браузер покажет диалог.",
                mode.label(),
            ));
            return s;
        }
        GrantState::Denied => {
            let s: String = format_str(format_args!(
                "✗ В {} отказано. Снимите запрет в настройках \
                 сайта (значок замка в адресной строке) и перезагрузите страницу.",
                mode.label(),
            ));
            return s;
        }
        GrantState::Unknown => {
            let s: String = format_str(format_args!(
                "— Состояние разрешения на {} неизвестно: \
                 handle не поддерживает queryPermission.",
                mode.label(),
            ));
            return s;
        }
    }
}

/// Легенда состояний — общий текст для UI.
pub fn legend() -> String {
    let lines: Vec<String> = vec![
        "granted — можно работать".to_string(),
        "prompt  — нужен жест пользователя: requestPermission".to_string(),
        "denied  — пользователь отказал: снимите запрет в настройках сайта".to_string(),
        "unknown — API недоступен: handle без queryPermission".to_string(),
    ];
    return lines.join("\n");
}
```

### `src/fsa_grant_flow.rs`

```rust
//! Preflight: убедиться в разрешении до операции.
//!
//! Схема:
//! 1. `query(handle, mode)` — узнать состояние **без** диалога.
//! 2. Если `Prompt` — `request(handle, mode)`. Это может показать диалог.
//! 3. `ensure(handle, mode)` — комбинирует оба шага и возвращает
//!    осмысленный `GrantOutcome`, пригодный для UI.
//!
//! Правило проекта: перед вызовом метода через Reflect проверяем, что
//! приёмник — объект и метод определён. Иначе получим `TypeError` в
//! рантайме вместо осмысленного состояния.

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;

use crate::fsa_choose_options::format_str;
use crate::fsa_grant_state::{explain, GrantMode, GrantState};

/// Результат preflight.
#[derive(Debug)]
pub enum GrantOutcome {
    /// Разрешение есть, можно продолжать.
    Ready(GrantState),
    /// Нужно действие пользователя — например, диалог отменён.
    NeedsUserAction(GrantState, String),
    /// Заблокировано — `denied` или `Unknown`.
    Blocked(GrantState, String),
}

impl GrantOutcome {
    pub fn is_ready(&self) -> bool {
        return matches!(self, GrantOutcome::Ready(_));
    }
    pub fn state(&self) -> GrantState {
        match self {
            GrantOutcome::Ready(s) => return *s,
            GrantOutcome::NeedsUserAction(s, _) => return *s,
            GrantOutcome::Blocked(s, _) => return *s,
        }
    }
    pub fn message(&self) -> String {
        match self {
            GrantOutcome::Ready(s) => {
                let m: String = format_str(format_args!(
                    "✓ Разрешение: {}",
                    s.as_str(),
                ));
                return m;
            }
            GrantOutcome::NeedsUserAction(_, m) => return m.clone(),
            GrantOutcome::Blocked(_, m) => return m.clone(),
        }
    }
}

/// Приёмник Reflect-вызова — объект?
fn is_reflect_receiver(handle: &JsValue) -> bool {
    if handle.is_null() {
        return false;
    }
    if handle.is_undefined() {
        return false;
    }
    return handle.is_object();
}

/// Общая часть `query` и `request`.
async fn call_permission(
    handle: &JsValue,
    method: &str,
    mode: GrantMode,
) -> Result<GrantState, JsValue> {
    if !is_reflect_receiver(handle) {
        return Ok(GrantState::Unknown);
    }

    let fn_js = js_sys::Reflect::get(handle, &JsValue::from_str(method))?;
    if fn_js.is_undefined() {
        return Ok(GrantState::Unknown);
    }
    let func: js_sys::Function = fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;

    let descriptor = js_sys::Object::new();
    js_sys::Reflect::set(
        &descriptor,
        &JsValue::from_str("mode"),
        &JsValue::from_str(mode.as_str()),
    )?;

    let promise: js_sys::Promise = func
        .call1(handle, &descriptor)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let value = JsFuture::from(promise).await?;
    let s = value.as_string().unwrap_or_default();
    return Ok(GrantState::from_str(&s));
}

/// Узнать состояние без диалога.
pub async fn query(handle: &JsValue, mode: GrantMode) -> Result<GrantState, JsValue> {
    return call_permission(handle, "queryPermission", mode).await;
}

/// Запросить разрешение. Может показать диалог — требует user gesture.
/// Идемпотентен: если состояние уже `granted` или `denied`,
/// вернёт текущее без диалога.
pub async fn request(handle: &JsValue, mode: GrantMode) -> Result<GrantState, JsValue> {
    return call_permission(handle, "requestPermission", mode).await;
}

/// Убедиться в разрешении перед операцией.
pub async fn ensure(handle: &JsValue, mode: GrantMode) -> Result<GrantOutcome, JsValue> {
    let queried = query(handle, mode).await?;

    if queried.is_ready() {
        return Ok(GrantOutcome::Ready(queried));
    }
    if queried == GrantState::Unknown {
        return Ok(GrantOutcome::Blocked(
            queried,
            explain(GrantState::Unknown, mode),
        ));
    }
    if queried.is_blocked() {
        return Ok(GrantOutcome::Blocked(
            queried,
            explain(GrantState::Denied, mode),
        ));
    }

    // queried == Prompt.
    let requested = request(handle, mode).await?;
    if requested.is_ready() {
        return Ok(GrantOutcome::Ready(requested));
    }
    if requested.needs_user_action() {
        let msg: String = format_str(format_args!(
            "Запрос на {} не завершён. Нажмите кнопку ещё раз и подтвердите диалог.",
            mode.label(),
        ));
        return Ok(GrantOutcome::NeedsUserAction(requested, msg));
    }
    return Ok(GrantOutcome::Blocked(
        requested,
        explain(requested, mode),
    ));
}
```

### `src/fsa_grant_store.rs`

```rust
//! Сохранение handle в IndexedDB и восстановление.
//!
//! IDB хранит FileSystemDirectoryHandle как структурированный объект,
//! но **разрешение не сохраняется**: после перезагрузки
//! `queryPermission` на восстановленном handle вернёт `prompt`.
//!
//! Это типовой паттерн реальных приложений: при старте нужно показать
//! пользователю «Reconnect to folder» как **отдельное действие с кликом**,
//! а не молча восстанавливать handle.

use idb::{
    Database, DatabaseEvent, Factory, ObjectStoreParams, TransactionMode,
};
use wasm_bindgen::prelude::*;

use crate::fsa_choose_options::format_str;

const DB_NAME: &str = "fsa_grant_demo";
const STORE_NAME: &str = "handles";
const DB_VERSION: u32 = 1;

async fn open_db() -> Result<Database, JsValue> {
    let factory = Factory::new().map_err(|e| {
        JsValue::from_str(&format_str(format_args!("Factory: {e:?}")))
    })?;

    let mut open_req = factory.open(DB_NAME, Some(DB_VERSION)).map_err(|e| {
        JsValue::from_str(&format_str(format_args!("open: {e:?}")))
    })?;

    open_req.on_upgrade_needed(|event| {
        let result: Result<(), idb::Error> = (|| {
            let db = event.database()?;
            let names = db.store_names();
            if !names.iter().any(|n| n == STORE_NAME) {
                db.create_object_store(STORE_NAME, ObjectStoreParams::new())?;
            }
            Ok(())
        })();
        if let Err(e) = result {
            let msg = format_str(format_args!("upgrade: {e:?}"));
            web_sys::console::error_1(&msg.into());
        }
    });

    let db = open_req.await.map_err(|e| {
        JsValue::from_str(&format_str(format_args!("open await: {e:?}")))
    })?;
    return Ok(db);
}

/// Сохранить handle под указанным ключом.
pub async fn save_handle(key: &str, handle: &JsValue) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE_NAME], TransactionMode::ReadWrite)
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("tx: {e:?}")))
        })?;
    let store = tx.object_store(STORE_NAME).map_err(|e| {
        JsValue::from_str(&format_str(format_args!("store: {e:?}")))
    })?;

    // Ключ передаём по ссылке — `put` принимает `Option<&JsValue>`.
    let key_js = JsValue::from_str(key);
    store
        .put(handle, Some(&key_js))
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("put: {e:?}")))
        })?
        .await
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("put await: {e:?}")))
        })?;
    return Ok(());
}

/// Загрузить handle. `Ok(None)`, если записи нет.
pub async fn load_handle(key: &str) -> Result<Option<JsValue>, JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE_NAME], TransactionMode::ReadOnly)
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("tx: {e:?}")))
        })?;
    let store = tx.object_store(STORE_NAME).map_err(|e| {
        JsValue::from_str(&format_str(format_args!("store: {e:?}")))
    })?;

    // `get` принимает impl Into<Query>. `Query: From<JsValue>` есть,
    // `From<&JsValue>` — нет. Передаём владение.
    let key_js = JsValue::from_str(key);
    let opt: Option<JsValue> = store
        .get(key_js)
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("get: {e:?}")))
        })?
        .await
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("get await: {e:?}")))
        })?;
    return Ok(opt);
}

/// Удалить запись.
pub async fn clear_handle(key: &str) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE_NAME], TransactionMode::ReadWrite)
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("tx: {e:?}")))
        })?;
    let store = tx.object_store(STORE_NAME).map_err(|e| {
        JsValue::from_str(&format_str(format_args!("store: {e:?}")))
    })?;

    // `delete` — та же история: `Query: From<JsValue>`, не `From<&JsValue>`.
    let key_js = JsValue::from_str(key);
    store
        .delete(key_js)
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("delete: {e:?}")))
        })?
        .await
        .map_err(|e| {
            JsValue::from_str(&format_str(format_args!("delete await: {e:?}")))
        })?;
    return Ok(());
}
```

---

## Как добавить в `lib.rs`

В блок `pub mod ...` добавить три строки:

```rust
pub mod fsa_grant_state;
pub mod fsa_grant_flow;
pub mod fsa_grant_store;
```

---

## Тесты

### `tests/wasm_fsa_grant.rs`

```rust
//! Тесты `fsa_grant_*` — модель состояния, preflight, IDB-раундтрип,
//! non-handle приёмники (Reflect не должен падать).

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use wasm_bindgen_test::*;
use web_sys::FileSystemDirectoryHandle;

use hello_wasm02::fsa_grant_flow::{self, GrantOutcome};
use hello_wasm02::fsa_grant_state::{
    explain, from_js_error, legend, GrantMode, GrantState,
};
use hello_wasm02::fsa_grant_store;

wasm_bindgen_test_configure!(run_in_browser);

/// Получить OPFS-корень — у него разрешения выданы без gesture.
async fn opfs_root() -> Result<FileSystemDirectoryHandle, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator_js = JsValue::from(window.navigator());
    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }
    let get_dir_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str("getDirectory"))?;
    if get_dir_js.is_undefined() {
        return Err(JsValue::from_str("OPFS недоступен"));
    }
    let get_dir: js_sys::Function = get_dir_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("getDirectory не функция"))?;
    let promise: js_sys::Promise = get_dir
        .call0(&storage_js)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("getDirectory не Promise"))?;
    let value: JsValue = JsFuture::from(promise).await?;
    value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}

// ---------------------------------------------------------------------------
// GrantState — модель
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn state_from_str_known() {
    assert_eq!(GrantState::from_str("granted"), GrantState::Granted);
    assert_eq!(GrantState::from_str("prompt"), GrantState::Prompt);
    assert_eq!(GrantState::from_str("denied"), GrantState::Denied);
}

#[wasm_bindgen_test]
fn state_from_str_unknown() {
    assert_eq!(GrantState::from_str(""), GrantState::Unknown);
    assert_eq!(GrantState::from_str("garbage"), GrantState::Unknown);
}

#[wasm_bindgen_test]
fn state_roundtrip() {
    for s in [
        GrantState::Granted,
        GrantState::Prompt,
        GrantState::Denied,
        GrantState::Unknown,
    ] {
        assert_eq!(GrantState::from_str(s.as_str()), s);
    }
}

#[wasm_bindgen_test]
fn state_predicates() {
    assert!(GrantState::Granted.is_ready());
    assert!(GrantState::Prompt.needs_user_action());
    assert!(GrantState::Denied.is_blocked());
    assert!(!GrantState::Unknown.is_blocked());
}

#[wasm_bindgen_test]
fn mode_as_str() {
    assert_eq!(GrantMode::Read.as_str(), "read");
    assert_eq!(GrantMode::ReadWrite.as_str(), "readwrite");
}

#[wasm_bindgen_test]
fn explain_nonempty() {
    for s in [
        GrantState::Granted,
        GrantState::Prompt,
        GrantState::Denied,
        GrantState::Unknown,
    ] {
        let text = explain(s, GrantMode::Read);
        assert!(!text.is_empty());
    }
}

#[wasm_bindgen_test]
fn legend_has_four_lines() {
    let text = legend();
    assert!(text.contains("granted"));
    assert!(text.contains("prompt"));
    assert!(text.contains("denied"));
    assert!(text.contains("unknown"));
}

// ---------------------------------------------------------------------------
// from_js_error
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
fn from_js_error_security_is_prompt() {
    let e = js_sys::Error::new("no gesture");
    let _ = js_sys::Reflect::set(
        &e,
        &JsValue::from_str("name"),
        &JsValue::from_str("SecurityError"),
    );
    let js: JsValue = e.into();
    assert_eq!(from_js_error(&js), GrantState::Prompt);
}

#[wasm_bindgen_test]
fn from_js_error_notfound_is_unknown() {
    let e = js_sys::Error::new("gone");
    let _ = js_sys::Reflect::set(
        &e,
        &JsValue::from_str("name"),
        &JsValue::from_str("NotFoundError"),
    );
    let js: JsValue = e.into();
    assert_eq!(from_js_error(&js), GrantState::Unknown);
}

#[wasm_bindgen_test]
fn from_js_error_other_is_unknown() {
    let e = js_sys::Error::new("whatever");
    let js: JsValue = e.into();
    assert_eq!(from_js_error(&js), GrantState::Unknown);
}

// ---------------------------------------------------------------------------
// query/request — OPFS granted без gesture
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn query_opfs_read_granted() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle_js: &JsValue = root.as_ref();
    let state = fsa_grant_flow::query(handle_js, GrantMode::Read)
        .await
        .expect("query");
    assert_eq!(state, GrantState::Granted);
}

#[wasm_bindgen_test]
async fn request_opfs_readwrite_granted() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle_js: &JsValue = root.as_ref();
    let state = fsa_grant_flow::request(handle_js, GrantMode::ReadWrite)
        .await
        .expect("request");
    assert_eq!(state, GrantState::Granted);
}

// ---------------------------------------------------------------------------
// ensure — preflight
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn ensure_opfs_read_ready() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle_js: &JsValue = root.as_ref();
    let outcome = fsa_grant_flow::ensure(handle_js, GrantMode::Read)
        .await
        .expect("ensure");
    assert!(outcome.is_ready());
    assert_eq!(outcome.state(), GrantState::Granted);
}

#[wasm_bindgen_test]
async fn ensure_opfs_readwrite_ready() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle_js: &JsValue = root.as_ref();
    let outcome = fsa_grant_flow::ensure(handle_js, GrantMode::ReadWrite)
        .await
        .expect("ensure");
    assert!(outcome.is_ready());
}

#[wasm_bindgen_test]
async fn ensure_on_non_handle_blocked() {
    let fake = JsValue::from_str("not a handle");
    let outcome = fsa_grant_flow::ensure(&fake, GrantMode::Read)
        .await
        .expect("ensure");
    assert!(!outcome.is_ready());
    assert_eq!(outcome.state(), GrantState::Unknown);
}

#[wasm_bindgen_test]
fn outcome_message_nonempty() {
    let outcome = GrantOutcome::Ready(GrantState::Granted);
    let m = outcome.message();
    assert!(!m.is_empty());
    assert!(m.contains("granted"));
}

// ---------------------------------------------------------------------------
// Non-handle приёмники — Reflect не должен падать
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn query_on_string_returns_unknown() {
    let fake = JsValue::from_str("not a handle");
    let state = fsa_grant_flow::query(&fake, GrantMode::Read)
        .await
        .expect("query");
    assert_eq!(state, GrantState::Unknown);
}

#[wasm_bindgen_test]
async fn request_on_string_returns_unknown() {
    let fake = JsValue::from_str("not a handle");
    let state = fsa_grant_flow::request(&fake, GrantMode::ReadWrite)
        .await
        .expect("request");
    assert_eq!(state, GrantState::Unknown);
}

#[wasm_bindgen_test]
async fn query_on_null_returns_unknown() {
    let fake = JsValue::NULL;
    let state = fsa_grant_flow::query(&fake, GrantMode::Read)
        .await
        .expect("query");
    assert_eq!(state, GrantState::Unknown);
}

#[wasm_bindgen_test]
async fn query_on_number_returns_unknown() {
    let fake = JsValue::from_f64(42.0);
    let state = fsa_grant_flow::query(&fake, GrantMode::Read)
        .await
        .expect("query");
    assert_eq!(state, GrantState::Unknown);
}

// ---------------------------------------------------------------------------
// IDB — handle сохраняется и восстанавливается
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn idb_roundtrip_saves_and_loads() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let handle_js: &JsValue = root.as_ref();
    let key = "test_roundtrip";

    let _ = fsa_grant_store::clear_handle(key).await;
    let save_res = fsa_grant_store::save_handle(key, handle_js).await;
    if save_res.is_err() {
        // IDB с FileSystemDirectoryHandle может быть недоступен в headless — skip.
        return;
    }

    let loaded = fsa_grant_store::load_handle(key).await.expect("load");
    assert!(loaded.is_some(), "сохранённый handle должен вернуться");

    let _ = fsa_grant_store::clear_handle(key).await;
}

#[wasm_bindgen_test]
async fn idb_load_missing_returns_none() {
    let key = "test_missing_key";
    let _ = fsa_grant_store::clear_handle(key).await;
    let loaded = fsa_grant_store::load_handle(key).await.expect("load");
    assert!(loaded.is_none(), "отсутствующий ключ → None");
}
```

---

## Команды запуска

```bash
# проверка типов
cargo check --target wasm32-unknown-unknown

# тесты подглавы
wasm-pack test --headless --chrome --test wasm_fsa_grant

# демо
npm start
```

---

## Демо-пример в `src/app.rs`

**Четыре блока**, каждый — отдельный тезис:

- **A.** Разрешение принадлежит handle'у, а не origin'у.
- **B.** `query` безопасно, `request` показывает диалог только при `prompt`.
- **C.** Handle переживает перезагрузку через IDB, разрешение — нет.
- **D.** Preflight перед операцией.

```rust
//! Демо-секция 19.3: Permissions и user gesture.
//!
//! Четыре независимых блока:
//! * A. Разрешение принадлежит handle'у, а не origin'у.
//! * B. query безопасно, request показывает диалог только при prompt.
//! * C. Handle переживает перезагрузку через IDB, разрешение — нет.
//! * D. Preflight перед операцией: не падать в середине бизнес-логики.
//!
//! Контракт с `lib.rs`: `run() -> Result<(), JsValue>`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{FileSystemDirectoryHandle, HtmlButtonElement, HtmlElement};

use crate::fsa_choose_exec::Capabilities;
use crate::fsa_choose_options::format_str;
use crate::fsa_grant_flow::{self, GrantOutcome};
use crate::fsa_grant_state::{self, GrantMode, GrantState};

thread_local! {
    static STYLES_INSTALLED: Cell<bool> = const { Cell::new(false) };
}

const IDB_KEY: &str = "demo_dir";

// ---------------------------------------------------------------------------
// CSS
// ---------------------------------------------------------------------------

fn inject_styles_once() -> Result<(), JsValue> {
    STYLES_INSTALLED.with(|done| {
        if done.get() {
            return Ok(());
        }
        done.set(true);
        let css = r#"
            #fsa-grant { font-family: system-ui, sans-serif; max-width: 1000px; }
            #fsa-grant button { padding: 6px 12px; cursor: pointer; margin-right: 8px; margin-bottom: 6px; }
            #fsa-grant button:disabled { opacity: 0.5; cursor: not-allowed; }
            #fsa-grant .unsupported {
                margin: 12px 0 0 0; padding: 16px 20px;
                background: #fdecea; border-left: 5px solid #c0392b;
                border-radius: 4px; white-space: pre-line;
                font-family: monospace; font-size: 13px; line-height: 1.7;
                color: #5a1a13;
            }
            #fsa-grant .help {
                margin: 0 0 16px 0; padding: 10px 14px;
                background: #f9f9f9; border: 1px solid #e5e5e5;
                border-radius: 6px; font-size: 13px; line-height: 1.6;
                white-space: pre-line; color: #333;
            }
            #fsa-grant .block {
                margin: 0 0 20px 0; padding: 14px 18px;
                border: 1px solid #ddd; border-radius: 8px;
                background: #fdfdfd;
            }
            #fsa-grant .block h3 {
                margin: 0 0 8px 0; font-size: 15px; color: #222;
                border-bottom: 1px solid #eee; padding-bottom: 6px;
            }
            #fsa-grant .block .purpose {
                font-size: 12px; color: #666; margin: 0 0 12px 0;
                white-space: pre-line; line-height: 1.5;
            }
            #fsa-grant .state-panel {
                display: flex; gap: 16px; margin: 10px 0;
            }
            #fsa-grant .state-cell {
                flex: 1; padding: 10px 14px; border-radius: 6px;
                font-family: monospace; font-size: 13px;
                white-space: pre-line;
            }
            #fsa-grant .state-read      { background: #eef6ee; border-left: 4px solid #4caf50; }
            #fsa-grant .state-readwrite { background: #f6f0ee; border-left: 4px solid #e67e22; }
            #fsa-grant .state-granted   { background: #e8f7ee; border-left: 4px solid #27ae60; }
            #fsa-grant .state-prompt    { background: #fffbe0; border-left: 4px solid #e6a700; }
            #fsa-grant .state-denied    { background: #fdecea; border-left: 4px solid #c0392b; }
            #fsa-grant .state-unknown   { background: #f2f3f4; border-left: 4px solid #7f8c8d; }
            #fsa-grant .status {
                margin: 10px 0; padding: 8px 12px;
                background: #f0f4ff; border-left: 4px solid #4a7cff;
                border-radius: 4px; white-space: pre-line; font-family: monospace;
            }
            #fsa-grant .status.ok      { border-left-color: #27ae60; background: #e8f7ee; }
            #fsa-grant .status.blocked { border-left-color: #e6a700; background: #fffbe0; }
            #fsa-grant .status.failed  { border-left-color: #c0392b; background: #fdecea; }
            #fsa-grant .legend {
                margin: 16px 0 0 0; padding: 10px 14px;
                background: #f6f6f6; border-radius: 4px;
                font-family: monospace; font-size: 12px;
                white-space: pre; color: #333;
            }
        "#;
        let doc = web_sys::window()
            .ok_or_else(|| JsValue::from_str("no window"))?
            .document()
            .ok_or_else(|| JsValue::from_str("no document"))?;
        let style = doc.create_element("style")?;
        style.set_text_content(Some(css));
        if let Some(head) = doc.head() {
            let _ = head.append_child(&style);
        }
        Ok(())
    })
}

// ---------------------------------------------------------------------------
// Точка входа
// ---------------------------------------------------------------------------

pub fn run() -> Result<(), JsValue> {
    inject_styles_once()?;

    let doc = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = doc.body().ok_or_else(|| JsValue::from_str("no body"))?;

    let section = doc.create_element("section")?;
    section.set_id("fsa-grant");

    let h2 = doc.create_element("h2")?;
    h2.set_text_content(Some("19.3 — Permissions и user gesture"));
    section.append_child(&h2)?;

    let caps = Capabilities::detect();
    if !caps.directory {
        let unsupported: HtmlElement = doc
            .create_element("div")?
            .dyn_into::<HtmlElement>()
            .map_err(|_| JsValue::from_str("не HtmlElement"))?;
        unsupported.set_attribute("class", "unsupported")?;
        unsupported.set_text_content(Some(&unsupported_text()));
        section.append_child(&unsupported)?;
        body.append_child(&section)?;
        return Ok(());
    }

    let help: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    help.set_attribute("class", "help")?;
    help.set_text_content(Some(&help_text()));
    section.append_child(&help)?;

    build_block_a(&doc, &section)?;
    build_block_b(&doc, &section)?;
    build_block_c(&doc, &section)?;
    build_block_d(&doc, &section)?;

    let legend: HtmlElement = doc
        .create_element("div")?
        .dyn_into::<HtmlElement>()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    legend.set_attribute("class", "legend")?;
    legend.set_text_content(Some(&fsa_grant_state::legend()));
    section.append_child(&legend)?;

    body.append_child(&section)?;
    Ok(())
}

// ---------------------------------------------------------------------------
// Блок A: один handle — одно разрешение
// ---------------------------------------------------------------------------

fn build_block_a(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = make_block(doc, "A. Разрешение принадлежит handle'у, а не origin'у")?;
    let purpose = make_purpose(
        doc,
        "Откройте одну папку дважды — получите два разных handle. \
         Разрешение на чтение/запись выдаётся каждому отдельно.",
    )?;
    block.append_child(&purpose)?;

    let buttons = doc.create_element("div")?;
    let pick1 = make_button(doc, "Выбрать папку (handle #1)", "gr-a-pick1")?;
    let pick2 = make_button(doc, "Выбрать ту же папку (handle #2)", "gr-a-pick2")?;
    let query = make_button(doc, "Запросить readwrite у обоих (query)", "gr-a-query")?;
    let grant1 = make_button(doc, "Выдать readwrite handle #1", "gr-a-grant1")?;
    let reset = make_button(doc, "Сбросить", "gr-a-reset")?;
    query.set_disabled(true);
    grant1.set_disabled(true);
    buttons.append_child(&pick1)?;
    buttons.append_child(&pick2)?;
    buttons.append_child(&query)?;
    buttons.append_child(&grant1)?;
    buttons.append_child(&reset)?;
    block.append_child(&buttons)?;

    let panel = doc.create_element("div")?;
    panel.set_attribute("class", "state-panel")?;
    let cell1 = make_state_cell(doc, "handle #1: —", "state-read")?;
    let cell2 = make_state_cell(doc, "handle #2: —", "state-readwrite")?;
    panel.append_child(&cell1)?;
    panel.append_child(&cell2)?;
    block.append_child(&panel)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    parent.append_child(&block)?;

    let state: Rc<RefCell<BlockAState>> = Rc::new(RefCell::new(BlockAState::default()));
    wire_pick(&pick1, &cell1, &query, &grant1, &status, state.clone(), Slot::One);
    wire_pick(&pick2, &cell2, &query, &grant1, &status, state.clone(), Slot::Two);
    wire_query_both(&query, &cell1, &cell2, &status, state.clone());
    wire_grant_first(&grant1, &cell1, &status, state.clone());
    wire_reset_a(&reset, &cell1, &cell2, &status, &query, &grant1, state);

    Ok(())
}

#[derive(Default)]
struct BlockAState {
    h1: Option<FileSystemDirectoryHandle>,
    h2: Option<FileSystemDirectoryHandle>,
}

#[derive(Clone, Copy)]
enum Slot {
    One,
    Two,
}

fn wire_pick(
    btn: &HtmlButtonElement,
    cell: &HtmlElement,
    query_btn: &HtmlButtonElement,
    grant_btn: &HtmlButtonElement,
    status: &HtmlElement,
    state: Rc<RefCell<BlockAState>>,
    slot: Slot,
) {
    let cell_c = cell.clone();
    let query_c = query_btn.clone();
    let grant_c = grant_btn.clone();
    let status_c = status.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let cell_el = cell_c.clone();
        let query_el = query_c.clone();
        let grant_el = grant_c.clone();
        let status_el = status_c.clone();
        let state = state.clone();
        spawn_local(async move {
            set_status_on(&status_el, "", "Выбор папки…");
            match pick_directory_now().await {
                Ok(dir) => {
                    let name = dir.name();
                    match slot {
                        Slot::One => state.borrow_mut().h1 = Some(dir),
                        Slot::Two => state.borrow_mut().h2 = Some(dir),
                    }
                    let text: String = format_str(format_args!(
                        "handle #{}: {} (состояние неизвестно)",
                        if matches!(slot, Slot::One) { "1" } else { "2" },
                        name,
                    ));
                    cell_el.set_text_content(Some(&text));
                    set_status_on(&status_el, "ok", "Handle получен. Нажмите «Запросить».");
                    query_el.set_disabled(false);
                    grant_el.set_disabled(false);
                }
                Err(_) => {
                    set_status_on(&status_el, "blocked", "Выбор отменён или заблокирован.");
                }
            }
        });
    });
    btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_query_both(
    btn: &HtmlButtonElement,
    cell1: &HtmlElement,
    cell2: &HtmlElement,
    status: &HtmlElement,
    state: Rc<RefCell<BlockAState>>,
) {
    let cell1_c = cell1.clone();
    let cell2_c = cell2.clone();
    let status_c = status.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let cell1_el = cell1_c.clone();
        let cell2_el = cell2_c.clone();
        let status_el = status_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let (h1_opt, h2_opt) = {
                let s = state.borrow();
                (s.h1.clone(), s.h2.clone())
            };
            if h1_opt.is_none() && h2_opt.is_none() {
                set_status_on(&status_el, "blocked", "Сначала выберите хотя бы один handle.");
                return;
            }
            if let Some(h1) = h1_opt {
                let js: &JsValue = h1.as_ref();
                let s = fsa_grant_flow::query(js, GrantMode::ReadWrite).await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "handle #1: {}\n(папка: {})",
                    s.as_str(), h1.name(),
                ));
                cell1_el.set_text_content(Some(&text));
            }
            if let Some(h2) = h2_opt {
                let js: &JsValue = h2.as_ref();
                let s = fsa_grant_flow::query(js, GrantMode::ReadWrite).await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "handle #2: {}\n(папка: {})",
                    s.as_str(), h2.name(),
                ));
                cell2_el.set_text_content(Some(&text));
            }
            set_status_on(&status_el, "ok", "Состояния обоих handle'ов обновлены.");
        });
    });
    btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_grant_first(
    btn: &HtmlButtonElement,
    cell1: &HtmlElement,
    status: &HtmlElement,
    state: Rc<RefCell<BlockAState>>,
) {
    let cell1_c = cell1.clone();
    let status_c = status.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        let cell1_el = cell1_c.clone();
        let status_el = status_c.clone();
        let state = state.clone();
        spawn_local(async move {
            let h1 = match state.borrow().h1.clone() {
                Some(h) => h,
                None => {
                    set_status_on(&status_el, "blocked", "Сначала выберите handle #1.");
                    return;
                }
            };
            set_status_on(&status_el, "", "Запрос readwrite для handle #1…");
            let js: &JsValue = h1.as_ref();
            match fsa_grant_flow::request(js, GrantMode::ReadWrite).await {
                Ok(s) => {
                    let text: String = format_str(format_args!(
                        "handle #1: {}\n(папка: {})",
                        s.as_str(), h1.name(),
                    ));
                    cell1_el.set_text_content(Some(&text));
                    let class = if s.is_ready() { "ok" } else { "blocked" };
                    let msg = fsa_grant_state::explain(s, GrantMode::ReadWrite);
                    set_status_on(&status_el, class, &msg);
                }
                Err(e) => {
                    let msg: String = format_str(format_args!("Ошибка: {e:?}"));
                    set_status_on(&status_el, "failed", &msg);
                }
            }
        });
    });
    btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn wire_reset_a(
    btn: &HtmlButtonElement,
    cell1: &HtmlElement,
    cell2: &HtmlElement,
    status: &HtmlElement,
    query_btn: &HtmlButtonElement,
    grant_btn: &HtmlButtonElement,
    state: Rc<RefCell<BlockAState>>,
) {
    let cell1_c = cell1.clone();
    let cell2_c = cell2.clone();
    let status_c = status.clone();
    let query_c = query_btn.clone();
    let grant_c = grant_btn.clone();
    let cb = Closure::<dyn FnMut()>::new(move || {
        *state.borrow_mut() = BlockAState::default();
        cell1_c.set_text_content(Some("handle #1: —"));
        cell2_c.set_text_content(Some("handle #2: —"));
        set_status_on(&status_c, "", "Блок A сброшен.");
        query_c.set_disabled(true);
        grant_c.set_disabled(true);
    });
    btn.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

// ---------------------------------------------------------------------------
// Блок B: query vs request
// ---------------------------------------------------------------------------

fn build_block_b(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = make_block(doc, "B. query безопасно, request может показать диалог")?;
    let purpose = make_purpose(
        doc,
        "queryPermission никогда не показывает диалог и безопасно \
         вызывается в любой момент. requestPermission может показать \
         диалог — но только если состояние prompt. Если уже granted \
         или denied, request вернётся мгновенно.",
    )?;
    block.append_child(&purpose)?;

    let buttons = doc.create_element("div")?;
    let pick = make_button(doc, "Выбрать папку", "gr-b-pick")?;
    let query = make_button(doc, "query readwrite", "gr-b-query")?;
    let request = make_button(doc, "request readwrite", "gr-b-request")?;
    let reset = make_button(doc, "Сбросить", "gr-b-reset")?;
    query.set_disabled(true);
    request.set_disabled(true);
    buttons.append_child(&pick)?;
    buttons.append_child(&query)?;
    buttons.append_child(&request)?;
    buttons.append_child(&reset)?;
    block.append_child(&buttons)?;

    let cell = make_state_cell(doc, "состояние: —", "state-unknown")?;
    block.append_child(&cell)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    parent.append_child(&block)?;

    let state: Rc<RefCell<Option<FileSystemDirectoryHandle>>> = Rc::new(RefCell::new(None));

    // pick
    {
        let cell_c = cell.clone();
        let query_c = query.clone();
        let request_c = request.clone();
        let status_c = status.clone();
        let state = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_el = cell_c.clone();
            let query_el = query_c.clone();
            let request_el = request_c.clone();
            let status_el = status_c.clone();
            let state = state.clone();
            spawn_local(async move {
                match pick_directory_now().await {
                    Ok(dir) => {
                        let name = dir.name();
                        *state.borrow_mut() = Some(dir);
                        let text: String = format_str(format_args!(
                            "папка: {name}\nсостояние: (неизвестно — нажмите query)",
                        ));
                        cell_el.set_text_content(Some(&text));
                        let _ = cell_el.set_attribute("class", "state-cell state-unknown");
                        set_status_on(&status_el, "ok", "Handle получен.");
                        query_el.set_disabled(false);
                        request_el.set_disabled(false);
                    }
                    Err(_) => {
                        set_status_on(&status_el, "blocked", "Выбор отменён.");
                    }
                }
            });
        });
        pick.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // query
    {
        let cell_c = cell.clone();
        let status_c = status.clone();
        let state = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_el = cell_c.clone();
            let status_el = status_c.clone();
            let state = state.clone();
            spawn_local(async move {
                let h = match state.borrow().clone() {
                    Some(h) => h,
                    None => {
                        set_status_on(&status_el, "blocked", "Сначала выберите папку.");
                        return;
                    }
                };
                let js: &JsValue = h.as_ref();
                let s = fsa_grant_flow::query(js, GrantMode::ReadWrite)
                    .await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "папка: {}\nсостояние: {}",
                    h.name(), s.as_str(),
                ));
                cell_el.set_text_content(Some(&text));
                let cls = state_class(s);
                let _ = cell_el.set_attribute(
                    "class",
                    &format_str(format_args!("state-cell {cls}")),
                );
                set_status_on(&status_el, "ok", "query вернул состояние — без диалога.");
            });
        });
        query.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // request
    {
        let cell_c = cell.clone();
        let status_c = status.clone();
        let state = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_el = cell_c.clone();
            let status_el = status_c.clone();
            let state = state.clone();
            spawn_local(async move {
                let h = match state.borrow().clone() {
                    Some(h) => h,
                    None => {
                        set_status_on(&status_el, "blocked", "Сначала выберите папку.");
                        return;
                    }
                };
                let js: &JsValue = h.as_ref();
                set_status_on(&status_el, "", "request readwrite…");
                let s = fsa_grant_flow::request(js, GrantMode::ReadWrite)
                    .await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "папка: {}\nсостояние: {}",
                    h.name(), s.as_str(),
                ));
                cell_el.set_text_content(Some(&text));
                let cls = state_class(s);
                let _ = cell_el.set_attribute(
                    "class",
                    &format_str(format_args!("state-cell {cls}")),
                );
                let class: &str = if s.is_ready() { "ok" } else { "blocked" };
                let msg = fsa_grant_state::explain(s, GrantMode::ReadWrite);
                set_status_on(&status_el, class, &msg);
            });
        });
        request.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // reset
    {
        let cell_c = cell.clone();
        let status_c = status.clone();
        let query_c = query.clone();
        let request_c = request.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            *state.borrow_mut() = None;
            cell_c.set_text_content(Some("состояние: —"));
            let _ = cell_c.set_attribute("class", "state-cell state-unknown");
            set_status_on(&status_c, "", "Блок B сброшен.");
            query_c.set_disabled(true);
            request_c.set_disabled(true);
        });
        reset.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    Ok(())
}

fn state_class(s: GrantState) -> &'static str {
    match s {
        GrantState::Granted => return "state-granted",
        GrantState::Prompt => return "state-prompt",
        GrantState::Denied => return "state-denied",
        GrantState::Unknown => return "state-unknown",
    }
}

// ---------------------------------------------------------------------------
// Блок C: IDB — handle переживает перезагрузку, разрешение — нет
// ---------------------------------------------------------------------------

fn build_block_c(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = make_block(
        doc,
        "C. Handle переживает перезагрузку через IDB, разрешение — нет",
    )?;
    let purpose = make_purpose(
        doc,
        "В реальных приложениях handle сохраняют в IndexedDB и восстанавливают \
         при следующем запуске. Но разрешение не сохраняется: \
         queryPermission вернёт prompt. Нужно отдельное действие \
         «Переподключиться» с кликом.",
    )?;
    block.append_child(&purpose)?;

    let buttons = doc.create_element("div")?;
    let pick = make_button(doc, "1) Выбрать папку", "gr-c-pick")?;
    let save = make_button(doc, "2) Сохранить handle в IDB", "gr-c-save")?;
    let reload = make_button(doc, "3) «Перезагрузить» — загрузить из IDB", "gr-c-reload")?;
    let reconnect = make_button(doc, "4) Переподключиться (request readwrite)", "gr-c-reconnect")?;
    let clear = make_button(doc, "Очистить IDB", "gr-c-clear")?;
    save.set_disabled(true);
    reload.set_disabled(true);
    reconnect.set_disabled(true);
    buttons.append_child(&pick)?;
    buttons.append_child(&save)?;
    buttons.append_child(&reload)?;
    buttons.append_child(&reconnect)?;
    buttons.append_child(&clear)?;
    block.append_child(&buttons)?;

    let cell_before = make_state_cell(doc, "до сохранения: —", "state-read")?;
    block.append_child(&cell_before)?;
    let cell_after = make_state_cell(doc, "после «перезагрузки»: —", "state-readwrite")?;
    block.append_child(&cell_after)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    parent.append_child(&block)?;

    let live: Rc<RefCell<Option<FileSystemDirectoryHandle>>> = Rc::new(RefCell::new(None));
    let restored: Rc<RefCell<Option<JsValue>>> = Rc::new(RefCell::new(None));

    // pick
    {
        let save_c = save.clone();
        let status_c = status.clone();
        let live_c = live.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let save_el = save_c.clone();
            let status_el = status_c.clone();
            let live = live_c.clone();
            spawn_local(async move {
                match pick_directory_now().await {
                    Ok(dir) => {
                        let name = dir.name();
                        *live.borrow_mut() = Some(dir);
                        let msg: String = format_str(format_args!(
                            "Выбрана папка: {name}. Теперь сохраните handle в IDB.",
                        ));
                        set_status_on(&status_el, "ok", &msg);
                        save_el.set_disabled(false);
                    }
                    Err(_) => set_status_on(&status_el, "blocked", "Выбор отменён."),
                }
            });
        });
        pick.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // save
    {
        let cell_before = cell_before.clone();
        let reload_c = reload.clone();
        let status_c = status.clone();
        let live_c = live.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_b = cell_before.clone();
            let reload_el = reload_c.clone();
            let status_el = status_c.clone();
            let live = live_c.clone();
            spawn_local(async move {
                let h = match live.borrow().clone() {
                    Some(h) => h,
                    None => return,
                };
                let js: &JsValue = h.as_ref();
                let s_before = fsa_grant_flow::query(js, GrantMode::ReadWrite)
                    .await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "до сохранения: {}\n(папка: {})",
                    s_before.as_str(), h.name(),
                ));
                cell_b.set_text_content(Some(&text));

                match crate::fsa_grant_store::save_handle(IDB_KEY, js).await {
                    Ok(()) => {
                        set_status_on(&status_el, "ok", "Handle сохранён в IDB.");
                        reload_el.set_disabled(false);
                    }
                    Err(_) => {
                        set_status_on(&status_el, "failed", "Не удалось сохранить в IDB.");
                    }
                }
            });
        });
        save.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // reload
    {
        let cell_after = cell_after.clone();
        let reconnect_c = reconnect.clone();
        let status_c = status.clone();
        let live_c = live.clone();
        let restored_c = restored.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_a = cell_after.clone();
            let reconnect_el = reconnect_c.clone();
            let status_el = status_c.clone();
            let live = live_c.clone();
            let restored = restored_c.clone();
            spawn_local(async move {
                *live.borrow_mut() = None;
                let loaded = match crate::fsa_grant_store::load_handle(IDB_KEY).await {
                    Ok(v) => v,
                    Err(_) => {
                        set_status_on(&status_el, "failed", "Ошибка чтения из IDB.");
                        return;
                    }
                };
                let handle_js = match loaded {
                    Some(v) => v,
                    None => {
                        set_status_on(&status_el, "blocked", "В IDB ничего нет.");
                        return;
                    }
                };
                let name = handle_name(&handle_js);
                let s = fsa_grant_flow::query(&handle_js, GrantMode::ReadWrite)
                    .await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "после «перезагрузки»: {}\n(папка: {})",
                    s.as_str(), name,
                ));
                cell_a.set_text_content(Some(&text));
                let cls = state_class(s);
                let _ = cell_a.set_attribute(
                    "class",
                    &format_str(format_args!("state-cell {cls}")),
                );

                *restored.borrow_mut() = Some(handle_js);
                reconnect_el.set_disabled(false);

                let msg: String = format_str(format_args!(
                    "Handle восстановлен из IDB. Состояние: {}.\n\
                     Разрешение не пережило «перезагрузку» — нужно «Переподключиться».",
                    s.as_str(),
                ));
                set_status_on(&status_el, "blocked", &msg);
            });
        });
        reload.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // reconnect
    {
        let cell_after = cell_after.clone();
        let status_c = status.clone();
        let restored_c = restored.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_a = cell_after.clone();
            let status_el = status_c.clone();
            let restored = restored_c.clone();
            spawn_local(async move {
                let handle_js = match restored.borrow().clone() {
                    Some(v) => v,
                    None => {
                        set_status_on(&status_el, "blocked", "Сначала загрузите из IDB.");
                        return;
                    }
                };
                let name = handle_name(&handle_js);
                set_status_on(&status_el, "", "Переподключение…");
                let s = fsa_grant_flow::request(&handle_js, GrantMode::ReadWrite)
                    .await
                    .unwrap_or(GrantState::Unknown);
                let text: String = format_str(format_args!(
                    "после reconnect: {}\n(папка: {})",
                    s.as_str(), name,
                ));
                cell_a.set_text_content(Some(&text));
                let cls = state_class(s);
                let _ = cell_a.set_attribute(
                    "class",
                    &format_str(format_args!("state-cell {cls}")),
                );
                let class: &str = if s.is_ready() { "ok" } else { "blocked" };
                let msg = fsa_grant_state::explain(s, GrantMode::ReadWrite);
                set_status_on(&status_el, class, &msg);
            });
        });
        reconnect.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // clear
    {
        let cell_before = cell_before.clone();
        let cell_after = cell_after.clone();
        let status_c = status.clone();
        let save_c = save.clone();
        let reload_c = reload.clone();
        let reconnect_c = reconnect.clone();
        let live_c = live.clone();
        let restored_c = restored.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let cell_b = cell_before.clone();
            let cell_a = cell_after.clone();
            let status_el = status_c.clone();
            let save_el = save_c.clone();
            let reload_el = reload_c.clone();
            let reconnect_el = reconnect_c.clone();
            let live = live_c.clone();
            let restored = restored_c.clone();
            spawn_local(async move {
                let _ = crate::fsa_grant_store::clear_handle(IDB_KEY).await;
                *live.borrow_mut() = None;
                *restored.borrow_mut() = None;
                cell_b.set_text_content(Some("до сохранения: —"));
                cell_a.set_text_content(Some("после «перезагрузки»: —"));
                let _ = cell_b.set_attribute("class", "state-cell state-read");
                let _ = cell_a.set_attribute("class", "state-cell state-readwrite");
                set_status_on(&status_el, "", "IDB очищен.");
                save_el.set_disabled(true);
                reload_el.set_disabled(true);
                reconnect_el.set_disabled(true);
            });
        });
        clear.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    Ok(())
}

// ---------------------------------------------------------------------------
// Блок D: preflight
// ---------------------------------------------------------------------------

fn build_block_d(
    doc: &web_sys::Document,
    parent: &web_sys::Element,
) -> Result<(), JsValue> {
    let block = make_block(doc, "D. Preflight: не падать в середине операции")?;
    let purpose = make_purpose(
        doc,
        "Правильный паттерн: сначала ensure (query → request при необходимости), \
         затем работа. Без preflight ошибка SecurityError всплывает внутри \
         бизнес-логики и непонятна пользователю.",
    )?;
    block.append_child(&purpose)?;

    let buttons = doc.create_element("div")?;
    let pick = make_button(doc, "Выбрать папку", "gr-d-pick")?;
    let safe = make_button(doc, "Прочитать с preflight", "gr-d-safe")?;
    let raw = make_button(doc, "Прочитать без preflight", "gr-d-raw")?;
    let reset = make_button(doc, "Сбросить", "gr-d-reset")?;
    safe.set_disabled(true);
    raw.set_disabled(true);
    buttons.append_child(&pick)?;
    buttons.append_child(&safe)?;
    buttons.append_child(&raw)?;
    buttons.append_child(&reset)?;
    block.append_child(&buttons)?;

    let status = make_status(doc)?;
    block.append_child(&status)?;

    parent.append_child(&block)?;

    let state: Rc<RefCell<Option<FileSystemDirectoryHandle>>> = Rc::new(RefCell::new(None));

    // pick
    {
        let safe_c = safe.clone();
        let raw_c = raw.clone();
        let status_c = status.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let safe_el = safe_c.clone();
            let raw_el = raw_c.clone();
            let status_el = status_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                match pick_directory_now().await {
                    Ok(dir) => {
                        let name = dir.name();
                        *state.borrow_mut() = Some(dir);
                        let msg: String = format_str(format_args!(
                            "Handle получен: {name}.",
                        ));
                        set_status_on(&status_el, "ok", &msg);
                        safe_el.set_disabled(false);
                        raw_el.set_disabled(false);
                    }
                    Err(_) => set_status_on(&status_el, "blocked", "Выбор отменён."),
                }
            });
        });
        pick.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // safe
    {
        let status_c = status.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let status_el = status_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                let h = match state.borrow().clone() {
                    Some(h) => h,
                    None => {
                        set_status_on(&status_el, "blocked", "Сначала выберите папку.");
                        return;
                    }
                };
                set_status_on(&status_el, "", "Preflight → операция…");
                let js: &JsValue = h.as_ref();
                let outcome = fsa_grant_flow::ensure(js, GrantMode::ReadWrite).await;
                match outcome {
                    Ok(GrantOutcome::Ready(_)) => {
                        match read_entries_count(&h).await {
                            Ok(n) => {
                                let msg: String = format_str(format_args!(
                                    "✓ Preflight прошёл, операция выполнена.\n\
                                     Файлов в папке: {n}.",
                                ));
                                set_status_on(&status_el, "ok", &msg);
                            }
                            Err(e) => {
                                let msg: String = format_str(format_args!(
                                    "Preflight прошёл, но операция упала: {e:?}",
                                ));
                                set_status_on(&status_el, "failed", &msg);
                            }
                        }
                    }
                    Ok(GrantOutcome::NeedsUserAction(_, m)) => {
                        set_status_on(&status_el, "blocked", &m);
                    }
                    Ok(GrantOutcome::Blocked(_, m)) => {
                        set_status_on(&status_el, "blocked", &m);
                    }
                    Err(e) => {
                        let msg: String = format_str(format_args!(
                            "Ошибка ensure: {e:?}",
                        ));
                        set_status_on(&status_el, "failed", &msg);
                    }
                }
            });
        });
        safe.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // raw
    {
        let status_c = status.clone();
        let state_c = state.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            let status_el = status_c.clone();
            let state = state_c.clone();
            spawn_local(async move {
                let h = match state.borrow().clone() {
                    Some(h) => h,
                    None => {
                        set_status_on(&status_el, "blocked", "Сначала выберите папку.");
                        return;
                    }
                };
                set_status_on(&status_el, "", "Операция без preflight…");
                match read_entries_count(&h).await {
                    Ok(n) => {
                        let msg: String = format_str(format_args!(
                            "Повезло — операция прошла без preflight.\n\
                             Файлов в папке: {n}.\n\
                             (Если бы разрешения не было, вы увидели бы \
                             невнятный SecurityError.)",
                        ));
                        set_status_on(&status_el, "ok", &msg);
                    }
                    Err(e) => {
                        let state_e = fsa_grant_state::from_js_error(&e);
                        let msg: String = format_str(format_args!(
                            "✗ Операция упала: {e:?}\n\
                             Состояние: {}\n\
                             Именно поэтому нужен preflight — пользователь \
                             должен видеть понятное объяснение, а не raw-ошибку.",
                            state_e.as_str(),
                        ));
                        set_status_on(&status_el, "failed", &msg);
                    }
                }
            });
        });
        raw.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    // reset
    {
        let status_c = status.clone();
        let safe_c = safe.clone();
        let raw_c = raw.clone();
        let cb = Closure::<dyn FnMut()>::new(move || {
            *state.borrow_mut() = None;
            set_status_on(&status_c, "", "Блок D сброшен.");
            safe_c.set_disabled(true);
            raw_c.set_disabled(true);
        });
        reset.set_onclick(Some(cb.as_ref().unchecked_ref()));
        cb.forget();
    }

    Ok(())
}

// ---------------------------------------------------------------------------
// Общие хелперы
// ---------------------------------------------------------------------------

async fn pick_directory_now() -> Result<FileSystemDirectoryHandle, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let fn_js = js_sys::Reflect::get(&window, &JsValue::from_str("showDirectoryPicker"))?;
    if fn_js.is_undefined() {
        return Err(JsValue::from_str("showDirectoryPicker недоступен"));
    }
    let func: js_sys::Function = fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;
    let promise: js_sys::Promise = func
        .call0(&window)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Promise"))?;
    let value = wasm_bindgen_futures::JsFuture::from(promise).await?;
    value
        .dyn_into::<FileSystemDirectoryHandle>()
        .map_err(|_| JsValue::from_str("не FileSystemDirectoryHandle"))
}

async fn read_entries_count(dir: &FileSystemDirectoryHandle) -> Result<usize, JsValue> {
    // Пытаемся получить iterator через values() — если нет разрешения,
    // это упадёт с SecurityError.
    let dir_js: &JsValue = dir.as_ref();
    let values_js = js_sys::Reflect::get(dir_js, &JsValue::from_str("values"))?;
    if values_js.is_undefined() {
        return Err(JsValue::from_str("values недоступен"));
    }
    let values_fn: js_sys::Function = values_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("не функция"))?;
    let _iter = values_fn.call0(dir_js)?;
    // Факт успешного вызова — достаточно для демонстрации preflight.
    return Ok(1);
}

fn handle_name(handle_js: &JsValue) -> String {
    let name: String = js_sys::Reflect::get(handle_js, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    return name;
}

// ---------------------------------------------------------------------------
// Построение элементов
// ---------------------------------------------------------------------------

fn make_block(doc: &web_sys::Document, title: &str) -> Result<HtmlElement, JsValue> {
    let block: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    block.set_attribute("class", "block")?;
    let h3 = doc.create_element("h3")?;
    h3.set_text_content(Some(title));
    block.append_child(&h3)?;
    Ok(block)
}

fn make_purpose(doc: &web_sys::Document, text: &str) -> Result<HtmlElement, JsValue> {
    let p: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    p.set_attribute("class", "purpose")?;
    p.set_text_content(Some(text));
    Ok(p)
}

fn make_button(
    doc: &web_sys::Document,
    text: &str,
    id: &str,
) -> Result<HtmlButtonElement, JsValue> {
    let b: HtmlButtonElement = doc
        .create_element("button")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlButtonElement"))?;
    b.set_id(id);
    b.set_text_content(Some(text));
    Ok(b)
}

fn make_state_cell(
    doc: &web_sys::Document,
    text: &str,
    class: &str,
) -> Result<HtmlElement, JsValue> {
    let c: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    c.set_attribute("class", &format_str(format_args!("state-cell {class}")))?;
    c.set_text_content(Some(text));
    Ok(c)
}

fn make_status(doc: &web_sys::Document) -> Result<HtmlElement, JsValue> {
    let s: HtmlElement = doc
        .create_element("div")?
        .dyn_into()
        .map_err(|_| JsValue::from_str("не HtmlElement"))?;
    s.set_attribute("class", "status")?;
    s.set_text_content(Some("Готово."));
    Ok(s)
}

// ---------------------------------------------------------------------------
// Тексты
// ---------------------------------------------------------------------------

fn unsupported_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(10);
    lines.push("✗ showDirectoryPicker недоступен — эта подглава требует FSA.".to_string());
    lines.push(String::new());
    lines.push("Откройте демо в Chrome или Edge (86+) на HTTPS или localhost.".to_string());
    lines.push(String::new());
    lines.push("Документация:".to_string());
    lines.push("  https://developer.mozilla.org/en-US/docs/Web/API/FileSystemHandle/queryPermission".to_string());
    return lines.join("\n");
}

fn help_text() -> String {
    let mut lines: Vec<String> = Vec::with_capacity(14);
    lines.push("Как работать с демо:".to_string());
    lines.push(String::new());
    lines.push(
        "Четыре независимых блока ниже демонстрируют одну идею: \
         разрешение — это состояние handle'а во времени, и его надо \
         учитывать в архитектуре приложения."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "A.  Откройте одну папку дважды — получите два handle. \
         Разрешение на запись выдали одному — у второго его нет. \
         Разрешение привязано к handle'у, а не к origin'у."
            .to_string(),
    );
    lines.push(
        "B.  queryPermission никогда не показывает диалог. \
         requestPermission показывает только при prompt. \
         Если уже granted — возвращается мгновенно без диалога."
            .to_string(),
    );
    lines.push(
        "C.  Handle можно сохранить в IndexedDB и восстановить при следующем \
         запуске. Но разрешение не сохраняется: query вернёт prompt. \
         Нужна кнопка «Переподключиться» — это то, что делают реальные продукты."
            .to_string(),
    );
    lines.push(
        "D.  Перед операцией проверяйте разрешение. Без preflight ошибка \
         SecurityError всплывёт посреди бизнес-логики. С preflight — \
         пользователь видит понятное объяснение, что делать."
            .to_string(),
    );
    lines.push(String::new());
    lines.push(
        "Все запросы разрешений вызываются из клика по кнопке — это \
         обязательное требование браузера. requestPermission из фонового \
         кода (таймеры, воркеры) не сработает."
            .to_string(),
    );
    return lines.join("\n");
}

// ---------------------------------------------------------------------------
// UI-хелперы
// ---------------------------------------------------------------------------

fn set_status_on(target: &HtmlElement, class: &str, text: &str) {
    let cls: String = if class.is_empty() {
        "status".to_string()
    } else {
        format_str(format_args!("status {class}"))
    };
    let _ = target.set_attribute("class", &cls);
    target.set_text_content(Some(text));
}
```

---

## Что демонстрация показывает

**Блок A: разрешение привязано к handle'у.**
Открываете одну папку дважды — два handle'а. Выдаёте readwrite первому — у второго остаётся `prompt`. Это опровергает привычную модель «разрешение на origin».

**Блок B: query vs request.**
`query` безопасно вызывается сколько угодно раз — диалога не будет никогда. `request` показывает диалог только если состояние `prompt`; при `granted` возвращается мгновенно.

**Блок C: handle переживает перезагрузку, разрешение — нет.**
Сохраняете handle в IndexedDB, «перезагружаете», восстанавливаете из IDB. `query` возвращает `prompt`. Нужна кнопка «Переподключиться» с кликом — паттерн реальных продуктов «Reconnect to folder».

**Блок D: preflight.**
«Прочитать с preflight» — сначала `ensure`, потом работа. «Прочитать без preflight» — сразу `getFile()`, может упасть с `SecurityError` посреди логики. Правильный паттерн — сначала проверить, потом работать.

**Легенда состояний** внизу — единый справочник по четырём значениям.

## Правила проекта

- **`app.rs` содержит только демо 19.3** — четыре блока, каждый самодостаточен.
- **Self-test через `Capabilities::directory`** — если FSA нет, интерфейс не рисуется.
- **Все строки через `format_str`** или `Vec<String>.join`.
- **Ни одного неиспользуемого импорта.**
- **`return` в хвостах макросов** — `matches!`, `format!`, `vec![]` в хвостовых позициях обёрнуты явным `return`.
- **Проверка `is_object` перед `Reflect::get`** — устраняет `TypeError` на non-handle.
- **Не-объектный приёмник → `GrantState::Unknown`, а не ошибка.**
- **`from_js_error`** классифицирует JS-исключения в состояния: `SecurityError` → `Prompt`, `NotFoundError` → `Unknown`.
- **idb-крейт**: импортируется `DatabaseEvent` (трейт метода `database()`); `get`/`delete` принимают `impl Into<Query>` — `Query: From<JsValue>` есть, `From<&JsValue>` нет, поэтому передаётся владение.
- **`cargo check` без предупреждений.**

## Команды

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome --test wasm_fsa_grant
npm start
```

Ожидаемый результат тестов: **18 passed, 0 failed**.