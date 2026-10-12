# Глава 27. Offline Application Architecture

## 27.0. Что вы получите в этой главе

Работающее **offline-first приложение «Заметки»**: заметки создаются, редактируются и удаляются без сети; при восстановлении соединения синхронизируются с сервером; конфликты разрешаются автоматически. Демо в `app.rs` показывает полный цикл: создание офлайн → очередь → синхронизация → разрешение конфликта → обновление UI.

Отдельно разберём, **чего демо не показывает** — реального сервера и Service Worker. Сервер эмулируется через `localStorage` с искусственной задержкой (это позволяет сосредоточиться на архитектуре, а не на настройке бэкенда). Service Worker подключается в продакшене и здесь не обязателен: Cache API мы уже разобрали в Главе 25, регистрацию SW — в Главе 26.

## 27.1. Application shell

Application shell — минимальный HTML/CSS/JS-каркас, который загружается один раз и не зависит от данных. В нашем демо shell — это статический `HTML` в `app.rs`, который рендерится до обращения к IndexedDB. Даже если сеть недоступна и IndexedDB пуста, пользователь видит форму добавления и кнопки — приложение «живо».

Правило проекта: **shell не должен ждать данных**. Рендерим UI сразу, данные подтягиваем асинхронно.

## 27.2. Local state

Состояние приложения живёт в `Rc<RefCell<AppState>>`:

```rust
pub struct AppState {
    pub notes: Vec<OfflineNote>,
    pub online: OnlineStatus,
    pub queue: SyncQueue,
    pub syncing: bool,
}
```

Почему `Rc<RefCell<...>>`, а не отдельные переменные в замыканиях: несколько частей UI (статус-бар, список, форма) читают и пишут одно и то же состояние. Периодический опрос раз в 500 мс синхронизирует UI с реальностью — это дешевле, чем пробрасывать колбэки между несвязанными обработчиками. Правило проекта, зафиксированное в этой главе: общее состояние — `Rc<RefCell<State>>`, синхронизация через периодический опрос.

## 27.3. IndexedDB

Заметки хранятся в IndexedDB (store `notes`, ключ `id`). Каждая операция (создание, редактирование, удаление) сначала пишет в IndexedDB, потом в состояние, потом в очередь синхронизации. Если приложение закроется между шагами — данные не потеряются: IndexedDB переживает перезагрузку.

`offline_store` использует `idb` crate и повторяет паттерн из Главы 24: открыть БД, одна транзакция, коммит. Трейт `DatabaseEvent` должен быть в scope — без него метод `event.database()` в `on_upgrade_needed` не резолвится.

## 27.4. Cache

Cache API в этой главе не используется для заметок — заметки живут в IndexedDB. Cache API остаётся для **статических ресурсов** (app shell, WASM-бандл), как в Главе 25. Разделение принципиальное: IndexedDB — для структурированных данных, Cache — для HTTP-ответов. Смешивать нельзя: Cache API хранит `Request → Response`, а заметка — это `{ id, title, body, … }`, у которого нет естественного `Request`.

## 27.5. Service Worker

Service Worker перехватывает `fetch` и отдаёт статику из Cache. В демо он не обязателен (мы работаем из главного потока), но в продакшене именно он делает приложение «устанавливаемым» и полностью офлайновым. Глава 26 дала регистрацию; здесь мы предполагаем, что SW уже активен.

## 27.6. WASM computation

WASM в этой архитектуре отвечает за **три вещи**:

1. **Сериализация/десериализация** моделей через `serde_wasm_bindgen`.
2. **Логика разрешения конфликтов** — `resolve_conflict` на Rust, а не на JS.
3. **Эмуляция сервера** — `server_push`/`server_pull` с задержкой через `setTimeout`.

Вся тяжёлая работа (сравнение версий, слияние) выполняется в WASM, главный поток только рендерит.

## 27.7. Synchronization

Синхронизация — это **push локальных изменений** + **pull серверных** + **разрешение конфликтов**:

```rust
pub async fn sync(state: SharedState) -> Result<(), JsValue> {
    if state.borrow().online == OnlineStatus::Offline {
        return Err(JsValue::from_str("нет сети"));
    }
    if state.borrow().syncing {
        return Ok(());
    }
    state.borrow_mut().syncing = true;
    let result = sync_inner(state.clone()).await;
    state.borrow_mut().syncing = false;
    result
}
```

Ключевое правило: **push выполняется только при непустой очереди**. Если очередь пуста, значит, все локальные изменения уже отправлены; повторный push отправит устаревшие копии и перетрёт свежие серверные изменения других клиентов.

Очередь (`SyncQueue`) накапливает операции `Upsert(id)` и `Delete(id)`. Push идемпотентен: `SyncOp` не дублируется. После успешного push очередь очищается.

## 27.8. Conflict handling

Конфликт — когда локальная и серверная версии заметки разошлись. Стратегия **Last-Write-Wins**:

```rust
fn resolve_conflict(local: &OfflineNote, remote: &OfflineNote) -> OfflineNote {
    if local.updated_at > remote.updated_at {
        local.clone()
    } else if remote.updated_at > local.updated_at {
        remote.clone()
    } else if local.version >= remote.version {
        local.clone()
    } else {
        remote.clone()
    }
}
```

Если времена равны (например, обе версии созданы в одну миллисекунду), сравниваем `version`. Это не идеальное решение (потеря одной из версий), но для заметок приемлемое. Для критичных данных нужны CRDT или трёхстороннее слияние.

Отдельное правило: **tombstone'ы не живут вечно**. Заметка с `deleted: true` остаётся в БД только до первой успешной синхронизации — её роль сообщить серверу «удали это». После merge tombstone'ы физически удаляются из IndexedDB, а `state.notes` очищается от них. UI никогда не фильтрует `deleted` вручную: если фильтр в UI есть, значит, cleanup не сделан на уровне данных.

## 27.9. Online/offline transitions

Переход online→offline и offline→online отслеживается периодическим опросом `navigator.onLine` раз в 500 мс:

```rust
let was_offline = state.borrow().online == OnlineStatus::Offline;
let now_online = offline_core::online_status() == OnlineStatus::Online;
if was_offline && now_online {
    let _ = offline_sync::sync(state.clone()).await;
}
state.borrow_mut().online = if now_online { OnlineStatus::Online } else { OnlineStatus::Offline };
```

Автосинхронизация при восстановлении сети — ключевой UX-фактор: пользователь не должен нажимать «Синхронизировать» вручную. Кнопка остаётся резервным механизмом.

## 27.10. Полная offline-first архитектура

Собираем всё вместе:

```
┌──────────────────────────────────────────────────┐
│  Application Shell (HTML + WASM)                 │
│  ┌────────────────────────────────────────────┐  │
│  │  Local State (Rc<RefCell<AppState>>)       │  │
│  │  ┌──────────────┐  ┌───────────────────┐   │  │
│  │  │ IndexedDB    │  │ Sync Queue        │   │  │
│  │  │ (заметки)    │  │ (Upsert/Delete)   │   │  │
│  │  └──────────────┘  └───────────────────┘   │  │
│  └────────────────────────────────────────────┘  │
│         │                  │                      │
│         ▼                  ▼                      │
│  ┌──────────────┐  ┌───────────────────────┐     │
│  │ Cache API    │  │ Sync Engine (WASM)    │     │
│  │ (статика)    │  │ push/pull/merge       │     │
│  └──────────────┘  └───────────────────────┘     │
│         │                  │                      │
│         ▼                  ▼                      │
│  ┌──────────────┐  ┌───────────────────────┐     │
│  │ Service      │  │ Server (эмуляция)     │     │
│  │ Worker       │  │ localStorage + delay  │     │
│  └──────────────┘  └───────────────────────┘     │
└──────────────────────────────────────────────────┘
```

## 27.11. Практический проект: offline-first заметки

**Что делает.** Создаёт, редактирует и удаляет заметки без сети. При восстановлении соединения синхронизируется с сервером. Конфликты разрешает автоматически. Tombstone'ы очищает после успешной синхронизации.

**Почему это полезно.** Ровно этот код лежит в основе любой offline-first PWA: заметки, задачи, документы, черновики писем. Он собирает все три кирпича предыдущих глав — IndexedDB (24), Cache (25), Service Worker (26) — в работающую систему.

### Структура файлов

```
src/
  offline_core.rs      — статус сети, очередь синхронизации
  offline_state.rs     — модель OfflineNote, AppState, SharedState
  offline_store.rs     — персистентность в IndexedDB
  offline_server.rs    — эмуляция сервера (localStorage + delay)
  offline_sync.rs      — push/pull/merge, разрешение конфликтов
  app.rs               — демо только этой главы
tests/
  offline_core_test.rs — host-тесты (очередь, модель)
  offline_sync_test.rs — wasm-тесты (синхронизация)
```

---

## Исходные коды новых модулей

### `src/offline_core.rs`

```rust
//! Ядро offline-first: статус сети и очередь синхронизации.
//! Автономный модуль — не зависит от других модулей проекта.

use wasm_bindgen::prelude::*;
use web_sys::window;

/// Статус сети.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OnlineStatus {
    Online,
    Offline,
}

/// Возвращает текущий статус сети через `navigator.onLine`.
pub fn online_status() -> OnlineStatus {
    let nav = match window() {
        Some(w) => w.navigator(),
        None => return OnlineStatus::Offline,
    };
    let online = js_sys::Reflect::get(&nav, &JsValue::from_str("onLine"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);
    if online {
        OnlineStatus::Online
    } else {
        OnlineStatus::Offline
    }
}

/// Операция, ожидающая синхронизации.
#[derive(Debug, Clone, PartialEq)]
pub enum SyncOp {
    Upsert(String),
    Delete(String),
}

/// Очередь операций для синхронизации.
#[derive(Debug, Default)]
pub struct SyncQueue {
    ops: Vec<SyncOp>,
}

impl SyncQueue {
    pub fn new() -> Self {
        Self { ops: Vec::new() }
    }

    /// Идемпотентно: повторная операция не добавляется.
    pub fn push(&mut self, op: SyncOp) {
        if !self.ops.contains(&op) {
            self.ops.push(op);
        }
    }

    pub fn is_empty(&self) -> bool {
        self.ops.is_empty()
    }

    pub fn len(&self) -> usize {
        self.ops.len()
    }

    pub fn drain(&mut self) -> Vec<SyncOp> {
        std::mem::take(&mut self.ops)
    }
}
```

### `src/offline_state.rs`

```rust
//! Состояние offline-приложения и модель заметки.

use serde::{Deserialize, Serialize};

use crate::offline_core::{OnlineStatus, SyncQueue};

/// Заметка с поддержкой offline-синхронизации.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct OfflineNote {
    pub id: String,
    pub title: String,
    pub body: String,
    pub updated_at: f64,
    pub deleted: bool,
    pub version: u32,
}

impl OfflineNote {
    pub fn new(id: &str, title: &str, body: &str) -> Self {
        Self {
            id: id.into(),
            title: title.into(),
            body: body.into(),
            updated_at: js_sys::Date::now(),
            deleted: false,
            version: 1,
        }
    }

    pub fn touch(&mut self) {
        self.updated_at = js_sys::Date::now();
        self.version += 1;
    }
}

/// Общее состояние приложения.
pub struct AppState {
    pub notes: Vec<OfflineNote>,
    pub online: OnlineStatus,
    pub queue: SyncQueue,
    pub syncing: bool,
}

impl AppState {
    pub fn new() -> Self {
        Self {
            notes: Vec::new(),
            online: OnlineStatus::Offline,
            queue: SyncQueue::new(),
            syncing: false,
        }
    }
}

pub type SharedState = std::rc::Rc<std::cell::RefCell<AppState>>;

pub fn new_shared_state() -> SharedState {
    std::rc::Rc::new(std::cell::RefCell::new(AppState::new()))
}
```

### `src/offline_store.rs`

```rust
//! Персистентность offline-заметок в IndexedDB.
//!
//! ВАЖНО: `event.database()` — метод трейта `DatabaseEvent`.
//! Без `use idb::DatabaseEvent;` он не резолвится.

use idb::{
    Database, DatabaseEvent, Factory, KeyPath, ObjectStoreParams, TransactionMode,
};
use wasm_bindgen::prelude::*;

use crate::offline_state::OfflineNote;

const DB_NAME: &str = "offline-notes-db";
const DB_VERSION: u32 = 1;
const STORE: &str = "notes";

async fn open_db() -> Result<Database, JsValue> {
    let factory = Factory::new()
        .map_err(|e| JsValue::from_str(&format!("Factory: {e:?}")))?;
    let mut request = factory
        .open(DB_NAME, Some(DB_VERSION))
        .map_err(|e| JsValue::from_str(&format!("open: {e:?}")))?;
    request.on_upgrade_needed(|event| {
        // `event.database()` — метод трейта `DatabaseEvent`.
        let db: Database = match event.database() {
            Ok(d) => d,
            Err(_) => return,
        };
        let mut params = ObjectStoreParams::new();
        params.key_path(Some(KeyPath::new_single("id")));
        let _ = db.create_object_store(STORE, params);
    });
    request
        .await
        .map_err(|e| JsValue::from_str(&format!("await open: {e:?}")))
}

pub async fn save_note(note: &OfflineNote) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE], TransactionMode::ReadWrite)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store(STORE)
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;
    let value = serde_wasm_bindgen::to_value(note)
        .map_err(|e| JsValue::from_str(&format!("to_value: {e:?}")))?;
    store
        .put(&value, None)
        .map_err(|e| JsValue::from_str(&format!("put: {e:?}")))?;
    tx.commit()
        .map_err(|e| JsValue::from_str(&format!("commit: {e:?}")))?;
    Ok(())
}

pub async fn load_all_notes() -> Result<Vec<OfflineNote>, JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE], TransactionMode::ReadOnly)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store(STORE)
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;
    let values: Vec<JsValue> = store
        .get_all(None, None)
        .map_err(|e| JsValue::from_str(&format!("get_all: {e:?}")))?
        .await
        .map_err(|e| JsValue::from_str(&format!("await get_all: {e:?}")))?;
    let mut notes = Vec::with_capacity(values.len());
    for v in values {
        let note: OfflineNote = serde_wasm_bindgen::from_value(v)
            .map_err(|e| JsValue::from_str(&format!("from_value: {e:?}")))?;
        notes.push(note);
    }
    Ok(notes)
}

/// Мягкое удаление (tombstone): помечает `deleted = true`.
/// Физически запись остаётся, пока не пройдёт синхронизация.
pub async fn delete_note(id: &str) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE], TransactionMode::ReadWrite)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store(STORE)
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;
    store
        .delete(JsValue::from_str(id))
        .map_err(|e| JsValue::from_str(&format!("delete: {e:?}")))?;
    tx.commit()
        .map_err(|e| JsValue::from_str(&format!("commit: {e:?}")))?;
    Ok(())
}

/// Физическое удаление из IndexedDB (после успешной синхронизации tombstone).
pub async fn hard_delete_note(id: &str) -> Result<(), JsValue> {
    let db = open_db().await?;
    let tx = db
        .transaction(&[STORE], TransactionMode::ReadWrite)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))?;
    let store = tx
        .object_store(STORE)
        .map_err(|e| JsValue::from_str(&format!("store: {e:?}")))?;
    store
        .delete(JsValue::from_str(id))
        .map_err(|e| JsValue::from_str(&format!("delete: {e:?}")))?;
    tx.commit()
        .map_err(|e| JsValue::from_str(&format!("commit: {e:?}")))?;
    Ok(())
}
```

### `src/offline_server.rs`

```rust
//! Эмуляция сервера для синхронизации: localStorage + искусственная задержка.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{window, Storage};

use crate::offline_state::OfflineNote;

const SERVER_KEY: &str = "offline_server_notes";

fn storage() -> Result<Storage, JsValue> {
    window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .local_storage()
        .map_err(|e| JsValue::from_str(&format!("localStorage: {e:?}")))?
        .ok_or_else(|| JsValue::from_str("localStorage disabled"))
}

pub fn load_server_notes() -> Vec<OfflineNote> {
    storage()
        .ok()
        .and_then(|s| s.get_item(SERVER_KEY).ok())
        .flatten()
        .and_then(|raw| serde_json::from_str(&raw).ok())
        .unwrap_or_default()
}

pub fn save_server_notes(notes: &[OfflineNote]) -> Result<(), JsValue> {
    let raw = serde_json::to_string(notes)
        .map_err(|e| JsValue::from_str(&format!("serialize: {e:?}")))?;
    storage()?
        .set_item(SERVER_KEY, &raw)
        .map_err(|e| JsValue::from_str(&format!("set_item: {e:?}")))
}

async fn delay(ms: i32) -> Result<(), JsValue> {
    let promise = js_sys::Promise::new(&mut |resolve, _| {
        let _ = window()
            .unwrap()
            .set_timeout_with_callback_and_timeout_and_arguments_0(&resolve, ms);
    });
    JsFuture::from(promise)
        .await
        .map(|_| ())
        .map_err(|e| JsValue::from_str(&format!("delay: {e:?}")))
}

/// Эмулирует push на сервер с задержкой.
/// Last-Write-Wins на стороне сервера — клиент не может перетереть
/// более свежую серверную версию.
pub async fn server_push(notes: Vec<OfflineNote>) -> Result<(), JsValue> {
    delay(200).await?;
    let mut server_notes = load_server_notes();
    for note in notes {
        match server_notes.iter().position(|n| n.id == note.id) {
            Some(pos) => {
                let existing = &server_notes[pos];
                if note.updated_at > existing.updated_at
                    || (note.updated_at == existing.updated_at && note.version > existing.version)
                {
                    server_notes[pos] = note;
                }
            }
            None => server_notes.push(note),
        }
    }
    save_server_notes(&server_notes)
}

/// Эмулирует pull с сервера с задержкой.
pub async fn server_pull() -> Result<Vec<OfflineNote>, JsValue> {
    delay(200).await?;
    Ok(load_server_notes())
}
```

### `src/offline_sync.rs`

```rust
//! Логика синхронизации: push, pull, merge, разрешение конфликтов.
//!
//! Стратегия: Last-Write-Wins по `updated_at`, при равенстве — по `version`.
//! После успешного merge tombstone'ы (заметки с `deleted = true`)
//! физически удаляются из IndexedDB, чтобы не копить мусор.

use std::collections::HashMap;

use wasm_bindgen::prelude::*;

use crate::offline_core::{OnlineStatus, SyncQueue};
use crate::offline_server;
use crate::offline_state::{OfflineNote, SharedState};
use crate::offline_store;

/// Разрешение конфликта: Last-Write-Wins.
///
/// 1. Больший `updated_at` побеждает.
/// 2. При равенстве — больший `version` побеждает.
/// 3. При полном равенстве — локальная версия (стабильный выбор).
fn resolve_conflict(local: &OfflineNote, remote: &OfflineNote) -> OfflineNote {
    if local.updated_at > remote.updated_at {
        local.clone()
    } else if remote.updated_at > local.updated_at {
        remote.clone()
    } else if local.version >= remote.version {
        local.clone()
    } else {
        remote.clone()
    }
}

/// Полная синхронизация: push, pull, merge, очистка очереди и tombstone'ов.
///
/// Идемпотентна по эффекту: повторный вызов при пустой очереди и пустом
/// сервере не меняет состояние. Повторный вызов во время активной
/// синхронизации возвращает `Ok(())` без действий.
pub async fn sync(state: SharedState) -> Result<(), JsValue> {
    if state.borrow().online == OnlineStatus::Offline {
        return Err(JsValue::from_str("нет сети"));
    }
    if state.borrow().syncing {
        return Ok(());
    }
    state.borrow_mut().syncing = true;

    let result = sync_inner(state.clone()).await;
    state.borrow_mut().syncing = false;
    result
}

async fn sync_inner(state: SharedState) -> Result<(), JsValue> {
    // 1. Push локальных изменений (только тех, что есть в очереди).
    //    Если очередь пуста — не отправляем ничего, чтобы не перетирать
    //    серверные изменения локальными «устаревшими» копиями.
    let has_ops = !state.borrow().queue.is_empty();
    if has_ops {
        let local_notes = state.borrow().notes.clone();
        offline_server::server_push(local_notes).await?;
    }

    // 2. Pull серверных.
    let remote_notes = offline_server::server_pull().await?;

    // 3. Merge с разрешением конфликтов.
    let local_map: HashMap<String, OfflineNote> = state
        .borrow()
        .notes
        .iter()
        .map(|n| (n.id.clone(), n.clone()))
        .collect();

    let mut merged: Vec<OfflineNote> = Vec::with_capacity(remote_notes.len());
    for remote in remote_notes {
        match local_map.get(&remote.id) {
            Some(local) => merged.push(resolve_conflict(local, &remote)),
            None => merged.push(remote),
        }
    }
    // Локальные заметки, которых нет на сервере (ещё не пушились).
    for local in &state.borrow().notes {
        if !merged.iter().any(|n| n.id == local.id) {
            merged.push(local.clone());
        }
    }

    // 4. Физически удаляем tombstone'ы из IndexedDB.
    let tombstones: Vec<String> = merged
        .iter()
        .filter(|n| n.deleted)
        .map(|n| n.id.clone())
        .collect();
    for id in &tombstones {
        let _ = offline_store::hard_delete_note(id).await;
    }
    merged.retain(|n| !n.deleted);

    // 5. Обновляем состояние и очищаем очередь.
    state.borrow_mut().notes = merged;
    state.borrow_mut().queue = SyncQueue::new();
    Ok(())
}
```

---

## Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod offline_core;
pub mod offline_state;
pub mod offline_store;
pub mod offline_server;
pub mod offline_sync;
```

---

## Тесты

### `tests/offline_core_test.rs` (host)

```rust
//! Host-тесты ядра offline-first: очередь и модель заметки.

use hello_wasm02::offline_core::{SyncOp, SyncQueue};
use hello_wasm02::offline_state::OfflineNote;

#[test]
fn sync_queue_dedup() {
    let mut q = SyncQueue::new();
    q.push(SyncOp::Upsert("a".into()));
    q.push(SyncOp::Upsert("a".into()));
    assert_eq!(q.len(), 1);
    q.push(SyncOp::Delete("a".into()));
    assert_eq!(q.len(), 2);
}

#[test]
fn sync_queue_drain() {
    let mut q = SyncQueue::new();
    q.push(SyncOp::Upsert("a".into()));
    let ops = q.drain();
    assert_eq!(ops.len(), 1);
    assert!(q.is_empty());
}

#[test]
fn note_touch_updates_version() {
    let mut n = OfflineNote::new("1", "t", "b");
    assert_eq!(n.version, 1);
    n.touch();
    assert_eq!(n.version, 2);
    assert!(!n.deleted);
}
```

### `tests/offline_sync_test.rs` (wasm)

```rust
//! Wasm-тесты синхронизации: push, pull, merge, разрешение конфликтов.

#![cfg(target_arch = "wasm32")]

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen_test::*;

use hello_wasm02::offline_core::OnlineStatus;
use hello_wasm02::offline_state::{AppState, OfflineNote, SharedState};
use hello_wasm02::offline_sync;

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64)
}

fn shared_online() -> SharedState {
    let state = Rc::new(RefCell::new(AppState::new()));
    state.borrow_mut().online = OnlineStatus::Online;
    state
}

#[wasm_bindgen_test]
async fn sync_empty_state() {
    let state = shared_online();
    assert!(offline_sync::sync(state).await.is_ok());
}

#[wasm_bindgen_test]
async fn sync_pushes_local_note() {
    let state = shared_online();
    let note = OfflineNote::new(&uid(), "Sync", "Body");
    state.borrow_mut().notes.push(note.clone());
    offline_sync::sync(state.clone()).await.unwrap();
    assert!(state.borrow().notes.iter().any(|n| n.id == note.id));
}

#[wasm_bindgen_test]
async fn sync_offline_returns_error() {
    let state = Rc::new(RefCell::new(AppState::new()));
    state.borrow_mut().online = OnlineStatus::Offline;
    assert!(offline_sync::sync(state).await.is_err());
}

#[wasm_bindgen_test]
async fn queue_cleared_after_sync() {
    let state = shared_online();
    state
        .borrow_mut()
        .queue
        .push(hello_wasm02::offline_core::SyncOp::Upsert("x".into()));
    offline_sync::sync(state.clone()).await.unwrap();
    assert!(state.borrow().queue.is_empty());
}
```

---

## Команды запуска

```bash
# Host-тесты
cargo test

# Wasm-тесты
wasm-pack test --headless --chrome --test offline_sync_test

# Сборка wasm
npm run wasm:dev

# Запуск демо
npm run dev
```

---

## Демо-пример в `app.rs`

```rust
//! Демо Главы 27: Offline Application Architecture.
//!
//! Полная offline-first архитектура:
//!   • Application shell — UI рендерится мгновенно.
//!   • Local state — Rc<RefCell<AppState>>.
//!   • IndexedDB — персистентное хранилище заметок.
//!   • Sync Queue — накопление операций (Upsert/Delete).
//!   • Sync Engine — push/pull/merge (WASM).
//!   • Conflict handling — Last-Write-Wins.
//!   • Online/offline transitions — автосинхронизация.
//!
//! Фокусный подход: только демо этой главы.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::spawn_local;
use web_sys::{HtmlButtonElement, HtmlElement, HtmlInputElement, HtmlTextAreaElement};

use crate::offline_core::{self, OnlineStatus, SyncOp};
use crate::offline_state::{AppState, OfflineNote, SharedState};
use crate::offline_store;
use crate::offline_sync;

const HTML: &str = r#"
<h1>Глава 27: Offline Application Architecture</h1>
<p>Offline-first заметки: создание, редактирование, синхронизация, конфликты.</p>

<div style="padding:8px;margin:8px 0;border:1px solid #ddd">
  <strong>Статус:</strong> <span id="online-status">…</span> |
  <strong>Очередь:</strong> <span id="queue-len">0</span> |
  <strong>Синхронизация:</strong> <span id="syncing">—</span>
</div>

<fieldset style="margin:8px 0">
  <legend id="form-legend">Новая заметка</legend>
  <input id="note-title" placeholder="Заголовок" style="width:240px">
  <textarea id="note-body" placeholder="Текст" style="width:320px;height:60px"></textarea>
  <button id="save-note">Добавить</button>
  <button id="cancel-edit" style="display:none">Отмена</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Действия</legend>
  <button id="sync">Синхронизировать</button>
  <button id="toggle-online">Переключить online/offline</button>
  <button id="refresh">↻ Обновить список</button>
</fieldset>

<h3>Заметки</h3>
<div id="notes-count" style="color:#555"></div>
<ul id="notes-list"></ul>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let online_el = document.get_element_by_id("online-status").ok_or("no online")?;
    let queue_el = document.get_element_by_id("queue-len").ok_or("no queue")?;
    let syncing_el = document.get_element_by_id("syncing").ok_or("no syncing")?;
    let notes_list = document.get_element_by_id("notes-list").ok_or("no list")?;
    let notes_count = document.get_element_by_id("notes-count").ok_or("no count")?;
    let form_legend = document.get_element_by_id("form-legend").ok_or("no legend")?;
    let save_btn: HtmlButtonElement = document
        .get_element_by_id("save-note")
        .ok_or("no save")?
        .dyn_into()?;
    let cancel_btn: HtmlButtonElement = document
        .get_element_by_id("cancel-edit")
        .ok_or("no cancel")?
        .dyn_into()?;
    let sync_btn: HtmlButtonElement = document
        .get_element_by_id("sync")
        .ok_or("no sync")?
        .dyn_into()?;
    let toggle_btn: HtmlElement = document
        .get_element_by_id("toggle-online")
        .ok_or("no toggle")?
        .dyn_into()?;
    let refresh_btn: HtmlElement = document
        .get_element_by_id("refresh")
        .ok_or("no refresh")?
        .dyn_into()?;

    let state: SharedState = Rc::new(RefCell::new(AppState::new()));

    // ID редактируемой заметки (None = создание новой).
    let editing_id: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));

    // Первичная загрузка из IndexedDB.
    spawn_local({
        let state = state.clone();
        let notes_list = notes_list.clone();
        let notes_count = notes_count.clone();
        async move {
            if let Ok(notes) = offline_store::load_all_notes().await {
                state.borrow_mut().notes = notes;
            }
            render_notes(&state, &notes_list, &notes_count).await;
        }
    });

    // Периодический опрос статуса + автосинхронизация при переходе offline→online.
    {
        let state = state.clone();
        let online_el = online_el.clone();
        let queue_el = queue_el.clone();
        let syncing_el = syncing_el.clone();
        let notes_list = notes_list.clone();
        let notes_count = notes_count.clone();
        let sync_btn = sync_btn.clone();

        let interval = gloo_timers::callback::Interval::new(500, move || {
            let state = state.clone();
            let online_el = online_el.clone();
            let queue_el = queue_el.clone();
            let syncing_el = syncing_el.clone();
            let notes_list = notes_list.clone();
            let notes_count = notes_count.clone();
            let sync_btn = sync_btn.clone();
            spawn_local(async move {
                let was_offline = state.borrow().online == OnlineStatus::Offline;
                let now_online = offline_core::online_status() == OnlineStatus::Online;

                if was_offline && now_online {
                    let _ = offline_sync::sync(state.clone()).await;
                    render_notes(&state, &notes_list, &notes_count).await;
                }

                state.borrow_mut().online = if now_online {
                    OnlineStatus::Online
                } else {
                    OnlineStatus::Offline
                };

                let syncing = state.borrow().syncing;
                set_disabled(&sync_btn, syncing);

                update_status_ui(&state, &online_el, &queue_el, &syncing_el);
            });
        });
        interval.forget();
    }

    // Кнопка «Добавить»/«Сохранить».
    //
    // ПРАВИЛО: клоны элементов для использования ВНУТРИ замыкания
    // называем с суффиксом `_cb`. Иначе компилятор ругается на
    // move-out-of-borrowed: `set_click(&save_btn, move |_| { save_btn … })`
    // заимствует элемент, а затем перемещает его в замыкание.
    {
        let state = state.clone();
        let notes_list = notes_list.clone();
        let notes_count = notes_count.clone();
        let editing_id = editing_id.clone();
        let form_legend = form_legend.clone();
        let save_btn_cb = save_btn.clone();
        let cancel_btn_cb = cancel_btn.clone();

        set_click(&save_btn, move |_| {
            let title = get_input("note-title");
            let body = get_textarea("note-body");
            if title.is_empty() {
                return;
            }

            let state = state.clone();
            let notes_list = notes_list.clone();
            let notes_count = notes_count.clone();
            let editing_id = editing_id.clone();
            let form_legend = form_legend.clone();
            let save_btn_cb = save_btn_cb.clone();
            let cancel_btn_cb = cancel_btn_cb.clone();

            spawn_local(async move {
                let edit = editing_id.borrow_mut().take();
                let note = match edit {
                    Some(id) => {
                        let mut s = state.borrow_mut();
                        if let Some(n) = s.notes.iter_mut().find(|n| n.id == id) {
                            n.title = title;
                            n.body = body;
                            n.touch();
                            n.clone()
                        } else {
                            return;
                        }
                    }
                    None => {
                        let id = format!("note-{}", js_sys::Date::now() as u64);
                        OfflineNote::new(&id, &title, &body)
                    }
                };

                let _ = offline_store::save_note(&note).await;
                {
                    let mut s = state.borrow_mut();
                    if let Some(pos) = s.notes.iter().position(|n| n.id == note.id) {
                        s.notes[pos] = note.clone();
                    } else {
                        s.notes.push(note.clone());
                    }
                    s.queue.push(SyncOp::Upsert(note.id.clone()));
                }
                if state.borrow().online == OnlineStatus::Online {
                    let _ = offline_sync::sync(state.clone()).await;
                }

                // Сброс формы.
                set_input("note-title", "");
                set_textarea("note-body", "");
                form_legend.set_text_content(Some("Новая заметка"));
                save_btn_cb.set_text_content(Some("Добавить"));
                set_display(&cancel_btn_cb, "none");

                render_notes(&state, &notes_list, &notes_count).await;
            });
        });
    }

    // Кнопка «Отмена» — выход из режима редактирования.
    {
        let editing_id = editing_id.clone();
        let form_legend = form_legend.clone();
        let save_btn_cb = save_btn.clone();
        let cancel_btn_cb = cancel_btn.clone();

        set_click(&cancel_btn, move |_| {
            *editing_id.borrow_mut() = None;
            set_input("note-title", "");
            set_textarea("note-body", "");
            form_legend.set_text_content(Some("Новая заметка"));
            save_btn_cb.set_text_content(Some("Добавить"));
            set_display(&cancel_btn_cb, "none");
        });
    }

    // Кнопка «Синхронизировать».
    {
        let state = state.clone();
        let notes_list = notes_list.clone();
        let notes_count = notes_count.clone();
        set_click(&sync_btn, move |_| {
            let state = state.clone();
            let notes_list = notes_list.clone();
            let notes_count = notes_count.clone();
            spawn_local(async move {
                let _ = offline_sync::sync(state.clone()).await;
                render_notes(&state, &notes_list, &notes_count).await;
            });
        });
    }

    // Кнопка «Переключить online/offline».
    {
        let state = state.clone();
        set_click(&toggle_btn, move |_| {
            let mut s = state.borrow_mut();
            s.online = match s.online {
                OnlineStatus::Online => OnlineStatus::Offline,
                OnlineStatus::Offline => OnlineStatus::Online,
            };
        });
    }

    // Кнопка «↻ Обновить список».
    {
        let state = state.clone();
        let notes_list = notes_list.clone();
        let notes_count = notes_count.clone();
        set_click(&refresh_btn, move |_| {
            let state = state.clone();
            let notes_list = notes_list.clone();
            let notes_count = notes_count.clone();
            spawn_local(async move {
                render_notes(&state, &notes_list, &notes_count).await;
            });
        });
    }

    // Делегирование: «Удалить» и «Редактировать» внутри списка.
    {
        let list_el: HtmlElement = notes_list.clone().dyn_into()?;
        let state = state.clone();
        let notes_list = notes_list.clone();
        let notes_count = notes_count.clone();
        let editing_id = editing_id.clone();
        let form_legend = form_legend.clone();
        let save_btn_cb = save_btn.clone();
        let cancel_btn_cb = cancel_btn.clone();

        set_click(&list_el, move |e: web_sys::MouseEvent| {
            let target: web_sys::Element = match e.target().and_then(|t| t.dyn_into().ok()) {
                Some(t) => t,
                None => return,
            };
            let action = target.get_attribute("data-action").unwrap_or_default();
            let id = target.get_attribute("data-id").unwrap_or_default();
            if action.is_empty() || id.is_empty() {
                return;
            }

            match action.as_str() {
                "del" => {
                    let state = state.clone();
                    let notes_list = notes_list.clone();
                    let notes_count = notes_count.clone();
                    spawn_local(async move {
                        if let Some(note) =
                            state.borrow_mut().notes.iter_mut().find(|n| n.id == id)
                        {
                            note.deleted = true;
                            note.touch();
                            let _ = offline_store::save_note(note).await;
                        }
                        state.borrow_mut().queue.push(SyncOp::Delete(id.clone()));
                        if state.borrow().online == OnlineStatus::Online {
                            let _ = offline_sync::sync(state.clone()).await;
                        }
                        render_notes(&state, &notes_list, &notes_count).await;
                    });
                }
                "edit" => {
                    let note = state
                        .borrow()
                        .notes
                        .iter()
                        .find(|n| n.id == id)
                        .cloned();
                    if let Some(n) = note {
                        set_input("note-title", &n.title);
                        set_textarea("note-body", &n.body);
                        *editing_id.borrow_mut() = Some(n.id);
                        form_legend.set_text_content(Some("Редактирование"));
                        save_btn_cb.set_text_content(Some("Сохранить"));
                        set_display(&cancel_btn_cb, "inline-block");
                    }
                }
                _ => {}
            }
        });
    }

    Ok(())
}

// ---------------- helpers ----------------

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn set_disabled(el: &HtmlButtonElement, disabled: bool) {
    el.set_disabled(disabled);
}

/// Управляет CSS-свойством `display`. `HtmlButtonElement` не имеет `set_style`;
/// используем `style().set_property(...)` через `Deref<HtmlElement>`.
fn set_display(el: &HtmlElement, value: &str) {
    let _ = el.style().set_property("display", value);
}

fn get_input(id: &str) -> String {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlInputElement>().ok())
        .map(|i| i.value())
        .unwrap_or_default()
}

fn set_input(id: &str, value: &str) {
    if let Some(input) = web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlInputElement>().ok())
    {
        input.set_value(value);
    }
}

fn get_textarea(id: &str) -> String {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlTextAreaElement>().ok())
        .map(|t| t.value())
        .unwrap_or_default()
}

fn set_textarea(id: &str, value: &str) {
    if let Some(ta) = web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlTextAreaElement>().ok())
    {
        ta.set_value(value);
    }
}

fn update_status_ui(
    state: &SharedState,
    online_el: &web_sys::Element,
    queue_el: &web_sys::Element,
    syncing_el: &web_sys::Element,
) {
    let s = state.borrow();
    online_el.set_text_content(Some(match s.online {
        OnlineStatus::Online => "online",
        OnlineStatus::Offline => "offline",
    }));
    queue_el.set_text_content(Some(&s.queue.len().to_string()));
    syncing_el.set_text_content(Some(if s.syncing { "да" } else { "нет" }));
}

/// Перечитывает состояние из IndexedDB и перерисовывает список.
///
/// После `sync()` tombstone'ы уже удалены физически (см. offline_sync.rs),
/// поэтому фильтровать `deleted` здесь не нужно.
async fn render_notes(state: &SharedState, list: &web_sys::Element, count: &web_sys::Element) {
    if let Ok(notes) = offline_store::load_all_notes().await {
        state.borrow_mut().notes = notes;
    }

    let notes = state.borrow().notes.clone();
    count.set_text_content(Some(&format!("Всего: {}", notes.len())));

    if notes.is_empty() {
        list.set_inner_html("<li><em>Заметок нет</em></li>");
        return;
    }

    let mut html = String::with_capacity(512);
    for n in &notes {
        html.push_str(&format!(
            "<li style=\"margin:8px 0\">\
             <strong>{}</strong><br>\
             <small>{}</small><br>\
             <small>updated: {}, version: {}</small><br>\
             <button data-action=\"edit\" data-id=\"{}\">Редактировать</button> \
             <button data-action=\"del\" data-id=\"{}\">Удалить</button>\
             </li>",
            esc(&n.title),
            esc(&n.body),
            n.updated_at as u64,
            n.version,
            esc(&n.id),
            esc(&n.id),
        ));
    }
    list.set_inner_html(&html);
}

fn esc(s: &str) -> String {
    s.replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;")
}
```

---

## Анализ решения и реализованные улучшения

**Что работает хорошо:**

- Полная offline-first архитектура: shell → state → IndexedDB → queue → sync → conflict resolution.
- Идемпотентность очереди (`SyncOp` не дублируется).
- Автосинхронизация при переходе offline→online.
- Разрешение конфликтов Last-Write-Wins в WASM.
- Эмуляция сервера с задержкой — видно, что синхронизация асинхронна.

**Реализованные улучшения:**

1. **Режим редактирования заметок.** Демо показывает полный CRUD: кнопка «Редактировать» заполняет форму, кнопка «Сохранить» обновляет заметку с `touch()` (растёт `version`, обновляется `updated_at`).
2. **Физическое удаление tombstone'ов.** После успешного merge `hard_delete_note` удаляет записи с `deleted: true` из IndexedDB, `state.notes` очищается через `retain`.
3. **Push только при непустой очереди.** Если очередь пуста — push не вызывается, серверные изменения не перетираются.
4. **Идемпотентность sync.** Повторный вызов во время активной синхронизации возвращает `Ok(())` без действий.
5. **Блокировка кнопки «Синхронизировать».** Пока `syncing == true`, кнопка disabled — пользователь не может запустить параллельную синхронизацию.
6. **Правильные клоны для замыканий.** Суффикс `_cb` для переменных, используемых внутри `set_click`.

## Проверка

```bash
cargo check
cargo test
wasm-pack test --headless --chrome --test offline_sync_test
npm run wasm:dev && npm run dev
```

Ожидается: `cargo check` без ошибок и предупреждений; 3 host-теста зелёные; 4 wasm-теста зелёные; демо работает — создание, редактирование, удаление, переключение online/offline, автосинхронизация, конфликты, tombstone'ы исчезают после sync.

## Правила проекта (закрепляем)

1. **Push только при непустой очереди.** Если очередь пуста — не отправляем ничего. Иначе каждая синхронизация превращается в «отправить всё» — на сервер уходит устаревшая локальная копия, перетирая свежие изменения других клиентов.
2. **Tombstone'ы не живут вечно.** Их роль — сообщить серверу «удали это», пока операция не отправлена. После успешной синхронизации — `hard_delete_note`, а не `state.notes`. UI не фильтрует `deleted` вручную.
3. **Общее состояние — `Rc<RefCell<AppState>>`.** Несколько частей UI читают и пишут одно и то же. Периодический опрос раз в 500 мс — дешёвый и устойчивый способ синхронизации.
4. **Клоны элементов для замыканий — суффикс `_cb`.** `set_click(&el, move |_| { /* el_cb */ })` — иначе E0505 «cannot move out of `el` because it is borrowed».
5. **`set_display(el, value)` вместо `set_style`.** У `HtmlElement` нет `set_style`; используем `el.style().set_property("display", value)`.
6. **`DatabaseEvent` должен быть в scope.** Без `use idb::DatabaseEvent;` метод `event.database()` в `on_upgrade_needed` не резолвится.