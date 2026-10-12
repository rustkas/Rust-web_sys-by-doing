# Глава 24.6. `add` vs `put` — семантика вставки

## 24.6.0. Что вы получите в этой главе

В 24.5 мы работали с `IDBObjectStore` и использовали `put` для всех записей. Это скрывало целый пласт семантики: `put` — только одна из двух операций записи, и выбор между `add` и `put` определяет поведение приложения при конфликте ключей.

К концу главы у вас будет демо с тремя экспериментами:

1. **Одна запись — три режима.** `add`, `put`, `save_with_diagnosis` на одном и том же ключе.
2. **Пачка — атомарность.** Один конфликтный `add` откатывает всю транзакцию, `put` — идемпотентен.
3. **Таблица решений.** Когда `add`, когда `put`, когда гибрид.

Плюс — рабочий модуль `idb_upsert`, который можно использовать напрямую как высокоуровневую обёртку.

## 24.6.1. Обновлённая структура файлов

```
hello-wasm02/
├── src/
│   ├── idb_upsert.rs       # НОВЫЙ: семантика записи, диагностика, атомарные пачки
│   ├── lib.rs              # +1 модуль
│   └── app.rs              # демо — три режима записи и сравнение add/put
├── tests/
│   └── wasm_idb_upsert.rs  # НОВЫЙ: 14 wasm-тестов
└── Cargo.toml              # без изменений
```

Никаких новых features в `Cargo.toml` не нужно: `DomException`, `Event`, `JsCast`, `wasm-bindgen-futures`, `js-sys` уже включены.

**Ссылки для детального изучения:**

- MDN, `IDBObjectStore.add()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/add
- MDN, `IDBObjectStore.put()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/put
- MDN, `ConstraintError` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/add#exceptions
- Spec, «Store a record into an object store» — https://w3c.github.io/IndexedDB/#store-record-into-an-object-store
- Spec, «Abort a transaction» — https://w3c.github.io/IndexedDB/#abort-a-transaction

## 24.6.2. Почему разница между `add` и `put` — это архитектурное решение

Первое, что бросается в глаза при знакомстве с IndexedDB: `put` — это «upsert», он перезаписывает запись, если ключ занят. `add` — это «insert», он падает с `ConstraintError`, если ключ занят. Кажется, что это косметическая разница: оба записывают запись.

Разница принципиальная. Она определяет **три уровня поведения**:

**Уровень 1 — UX одной операции.** Пользователь нажимает «Сохранить». Что он ожидает увидеть? Если это регистрация нового пользователя — «такой id занят» (значит, `add`). Если это автосохранение черновика — «сохранено», без уточнений (значит, `put`).

**Уровень 2 — поведение пачки.** Пачка из 100 записей в одной транзакции. Одна из них конфликтует. С `add` падает вся транзакция — включая 99 уже прошедших `onsuccess`. В store не останется ничего. С `put` пачка проходит целиком независимо от того, что уже было в store.

**Уровень 3 — идемпотентность повторов.** Сеть упала, клиент перезапустил операцию. Тот же `add` теперь упадёт с `ConstraintError` — потому что предыдущая попытка успела записать. Тот же `put` пройдёт как ни в чём не бывало.

**Правило.** Если операция должна быть идемпотентной — используйте `put`. Если она должна сигнализировать о конфликте — используйте `add`.

**Третий путь:** `add` + fallback на `put` с диагностикой. Это то, что в UI называют «умное сохранение»: пользователь не думает о том, новая это запись или существующая, а система сама решает и сообщает ему результат.

## 24.6.3. `ConstraintError` — это `DOMException`, а не строка

До того как написать код, важное замечание о том, как выглядит ошибка.

Когда `add` конфликтует, движок возвращает в `request.error` объект `DOMException`. У него есть поля `name = "ConstraintError"` и `message`. Это **не** строка. Если сделать `format!("{e:?}")` и завернуть результат в `JsValue::from_str`, исходный `name` потеряется — останется только отладочное представление.

Отсюда правило проекта: **`JsValue`-ошибки не переупаковываются в строки**. В модуле `idb_upsert` классификация происходит в `classify_write_error`, которая сначала пытается `dyn_ref::<DomException>()`, потом — `as_string()`, и только в крайнем случае — `Reflect::get("name")`. Никогда не паникует.

## 24.6.4. Исходный код `src/idb_upsert.rs`

```rust
//! Семантика вставки: `add` vs `put`, `ConstraintError`, диагностика upsert.
//!
//! `IDBObjectStore` даёт два метода записи одной записи с разным контрактом:
//!
//!   • `add(value)` — insert. Если ключ занят — синхронно идёт `error`-событие
//!     с `ConstraintError`, транзакция уходит в abort.
//!   • `put(value)` — upsert. Тот же ключ молча перезаписывается.
//!
//! Разница не косметическая: она определяет, что видит пользователь —
//! «такой id уже занят» или «сохранено». И определяет поведение пачки:
//! один конфликтный `add` откатывает **всю** транзакцию, включая уже
//! успешные записи. `put` в этом смысле идемпотентен.
//!
//! Модуль даёт три уровня API:
//!
//!   1. `add_one` / `put_one` — одна операция, своя транзакция.
//!   2. `save_with_diagnosis` — «умное сохранение»: сначала `add`,
//!      при `ConstraintError` читаем предыдущее значение и делаем `put`.
//!      Возвращает `SaveReport { outcome, previous }` — UI показывает
//!      «создано» или «обновлено (было → стало)».
//!   3. `batch_add_atomic` / `batch_put_idempotent` — пачки в одной
//!      транзакции. Наглядно показывают разницу семантик.
//!
//! ПРАВИЛО: ошибки пробрасываем как `JsValue` без обёртки в строку.
//! `classify_write_error` извлекает `name` из `DOMException`.
//!
//! ПРАВИЛО: после любого запроса ждём `tx.oncomplete`/`tx.onabort`, чтобы
//! соединение освободилось до следующего вызова. Иначе следующий `open`
//! на тот же store даст `TransactionInactiveError`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

// ============================================================
//                    ПУБЛИЧНЫЕ ТИПЫ
// ============================================================

/// Исход «умного сохранения».
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum SaveOutcome {
    Created,
    Updated,
}

/// Отчёт о сохранении с диагностикой.
#[derive(Debug)]
pub struct SaveReport {
    pub outcome: SaveOutcome,
    /// Предыдущее значение — `Some` только при `SaveOutcome::Updated`.
    pub previous: Option<JsValue>,
}

/// Классифицированный `DOMException` из операции записи.
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum WriteError {
    Constraint,
    Data,
    Quota,
    ReadOnly,
    InvalidState,
    TransactionInactive,
    Abort,
    Other(String),
}

impl WriteError {
    /// Короткая интерпретация для UI.
    pub fn short(&self) -> &'static str {
        match self {
            WriteError::Constraint => "ключ уже занят",
            WriteError::Data => "ключ не соответствует keyPath",
            WriteError::Quota => "квота исчерпана",
            WriteError::ReadOnly => "запись в readonly-транзакции",
            WriteError::InvalidState => "некорректное состояние",
            WriteError::TransactionInactive => "транзакция неактивна",
            WriteError::Abort => "транзакция отменена",
            WriteError::Other(_) => "ошибка записи",
        }
    }
    /// Строковое имя, как в `DOMException.name`.
    pub fn name(&self) -> &str {
        match self {
            WriteError::Constraint => "ConstraintError",
            WriteError::Data => "DataError",
            WriteError::Quota => "QuotaExceededError",
            WriteError::ReadOnly => "ReadOnlyError",
            WriteError::InvalidState => "InvalidStateError",
            WriteError::TransactionInactive => "TransactionInactiveError",
            WriteError::Abort => "AbortError",
            WriteError::Other(s) => s.as_str(),
        }
    }
}

/// Результат пачечной операции.
#[derive(Debug)]
pub struct BatchOutcome {
    pub attempted: u32,
    /// Сколько запросов успели завершиться до отката. Если транзакция
    /// прошла целиком — `written == attempted`.
    pub written: u32,
    /// Индекс первой упавшей записи, `None` — пачка прошла целиком.
    pub aborted_at: Option<u32>,
    /// Причина отката. `None` — пачка прошла целиком.
    pub abort_reason: Option<WriteError>,
}

// ============================================================
//                    КЛАССИФИКАЦИЯ
// ============================================================

pub fn classify_write_error(e: &JsValue) -> WriteError {
    let name = error_name(e);
    match name.as_str() {
        "ConstraintError" => WriteError::Constraint,
        "DataError" => WriteError::Data,
        "QuotaExceededError" => WriteError::Quota,
        "ReadOnlyError" => WriteError::ReadOnly,
        "InvalidStateError" => WriteError::InvalidState,
        "TransactionInactiveError" => WriteError::TransactionInactive,
        "AbortError" => WriteError::Abort,
        other => WriteError::Other(other.to_string()),
    }
}

pub fn is_constraint_error(e: &JsValue) -> bool {
    matches!(classify_write_error(e), WriteError::Constraint)
}

/// Достаёт `name` из `JsValue`. Порядок: `DomException.name` → строка →
/// `Reflect::get("name")`. Никогда не паникует.
fn error_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    if let Some(s) = e.as_string() {
        return s;
    }
    js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| "unknown".to_string())
}

// ============================================================
//                    ОДНА ОПЕРАЦИЯ
// ============================================================

/// `put` одной записи. Идемпотентен: тот же ключ перезаписывается.
pub async fn put_one(db: &JsValue, store_name: &str, value: &JsValue) -> Result<(), JsValue> {
    single_write(db, store_name, value, "put").await.map(|_| ())
}

/// `add` одной записи. `ConstraintError`, если ключ занят.
pub async fn add_one(db: &JsValue, store_name: &str, value: &JsValue) -> Result<(), JsValue> {
    single_write(db, store_name, value, "add").await.map(|_| ())
}

async fn single_write(
    db: &JsValue,
    store_name: &str,
    value: &JsValue,
    method_name: &str,
) -> Result<JsValue, JsValue> {
    let (os, tx) = open_tx(db, store_name, "readwrite")?;
    let tx_done = tx_done_promise(&tx);
    let f = method(&os, method_name)?;
    let req = f
        .call1(&os, value)
        .map_err(|e| JsValue::from_str(&format!("{method_name}: {e:?}")))?;
    let res = await_request(&req).await;
    // Ждём завершения транзакции в любом случае — иначе следующий вызов
    // получит TransactionInactiveError, а deleteDatabase — onblocked.
    let _ = JsFuture::from(tx_done).await;
    res
}

/// `get` одной записи по ключу. `None` — ключа нет.
pub async fn get_one(
    db: &JsValue,
    store_name: &str,
    key: &JsValue,
) -> Result<Option<JsValue>, JsValue> {
    let (os, tx) = open_tx(db, store_name, "readonly")?;
    let tx_done = tx_done_promise(&tx);
    let f = method(&os, "get")?;
    let req = f
        .call1(&os, key)
        .map_err(|e| JsValue::from_str(&format!("get: {e:?}")))?;
    let res = await_request(&req).await;
    let _ = JsFuture::from(tx_done).await;
    match res? {
        v if v.is_null() || v.is_undefined() => Ok(None),
        v => Ok(Some(v)),
    }
}

// ============================================================
//                    УМНОЕ СОХРАНЕНИЕ
// ============================================================

/// «Создать или обновить» с диагностикой.
///
/// Пробуем `add` в свежей транзакции. Создалось — `Created`. Упало с
/// `ConstraintError` — читаем предыдущее значение (`previous`) и делаем
/// `put` в новой транзакции — `Updated`. Любая другая ошибка — наверх.
///
/// Диагностика race-free: между попыткой `add` и выводом «запись
/// существовала» не остаётся окна, потому что `ConstraintError` —
/// это вердикт самого движка.
pub async fn save_with_diagnosis(
    db: &JsValue,
    store_name: &str,
    value: &JsValue,
    key: &JsValue,
) -> Result<SaveReport, JsValue> {
    match add_one(db, store_name, value).await {
        Ok(()) => Ok(SaveReport {
            outcome: SaveOutcome::Created,
            previous: None,
        }),
        Err(e) => {
            if !is_constraint_error(&e) {
                return Err(e);
            }
            let previous = get_one(db, store_name, key).await?;
            put_one(db, store_name, value).await?;
            Ok(SaveReport {
                outcome: SaveOutcome::Updated,
                previous,
            })
        }
    }
}

// ============================================================
//                    ПАЧКИ
// ============================================================

/// Все `add` в одной транзакции. Первый конфликт откатывает всю пачку.
///
/// `written` показывает, сколько запросов успели получить `onsuccess`
/// до отката. Но в store после отката они не останутся — транзакция
/// атомарна. Это и есть ключевое свойство IndexedDB.
pub async fn batch_add_atomic(
    db: &JsValue,
    store_name: &str,
    values: &[JsValue],
) -> Result<BatchOutcome, JsValue> {
    batch_write(db, store_name, values, "add").await
}

/// Все `put` в одной транзакции. Идемпотентно: пачка всегда проходит
/// целиком, независимо от того, какие ключи уже есть в store.
pub async fn batch_put_idempotent(
    db: &JsValue,
    store_name: &str,
    values: &[JsValue],
) -> Result<BatchOutcome, JsValue> {
    batch_write(db, store_name, values, "put").await
}

struct BatchState {
    written: u32,
    aborted_at: Option<u32>,
    abort_reason: Option<WriteError>,
}

async fn batch_write(
    db: &JsValue,
    store_name: &str,
    values: &[JsValue],
    method_name: &str,
) -> Result<BatchOutcome, JsValue> {
    let (os, tx) = open_tx(db, store_name, "readwrite")?;
    let tx_done = tx_done_promise(&tx);

    let state = Rc::new(RefCell::new(BatchState {
        written: 0,
        aborted_at: None,
        abort_reason: None,
    }));

    let f = method(&os, method_name)?;
    let attempted = values.len() as u32;

    for (i, v) in values.iter().enumerate() {
        let req = match f.call1(&os, v) {
            Ok(r) => r,
            Err(e) => {
                // Синхронная ошибка — транзакция уже неактивна.
                let mut s = state.borrow_mut();
                if s.aborted_at.is_none() {
                    s.aborted_at = Some(i as u32);
                    s.abort_reason = Some(classify_write_error(&e));
                }
                break;
            }
        };

        let state_ok = state.clone();
        let on_ok = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
            state_ok.borrow_mut().written += 1;
        });
        let _ = js_sys::Reflect::set(
            &req,
            &JsValue::from_str("onsuccess"),
            on_ok.as_ref().unchecked_ref(),
        );
        on_ok.forget();

        let state_err = state.clone();
        let idx = i as u32;
        let on_err = Closure::<dyn FnMut(web_sys::Event)>::new(move |ev: web_sys::Event| {
            let mut s = state_err.borrow_mut();
            if s.aborted_at.is_some() {
                return; // первая причина уже записана
            }
            s.aborted_at = Some(idx);
            let err = ev
                .target()
                .and_then(|t| js_sys::Reflect::get(&t, &JsValue::from_str("error")).ok())
                .unwrap_or(JsValue::UNDEFINED);
            s.abort_reason = Some(classify_write_error(&err));
        });
        let _ = js_sys::Reflect::set(
            &req,
            &JsValue::from_str("onerror"),
            on_err.as_ref().unchecked_ref(),
        );
        on_err.forget();
    }

    // Ждём финала транзакции — complete или abort, неважно.
    let _ = JsFuture::from(tx_done).await;

    let s = state.borrow();
    Ok(BatchOutcome {
        attempted,
        written: s.written,
        aborted_at: s.aborted_at,
        abort_reason: s.abort_reason.clone(),
    })
}

/// `count()` — сколько записей в store.
pub async fn count_all(db: &JsValue, store_name: &str) -> Result<u32, JsValue> {
    let (os, tx) = open_tx(db, store_name, "readonly")?;
    let tx_done = tx_done_promise(&tx);
    let f = method(&os, "count")?;
    let req = f
        .call0(&os)
        .map_err(|e| JsValue::from_str(&format!("count: {e:?}")))?;
    let res = await_request(&req).await;
    let _ = JsFuture::from(tx_done).await;
    res.map(|v| v.as_f64().unwrap_or(0.0) as u32)
}

/// `getAll()` — все записи.
pub async fn list_all(db: &JsValue, store_name: &str) -> Result<Vec<JsValue>, JsValue> {
    let (os, tx) = open_tx(db, store_name, "readonly")?;
    let tx_done = tx_done_promise(&tx);
    let f = method(&os, "getAll")?;
    let req = f
        .call0(&os)
        .map_err(|e| JsValue::from_str(&format!("getAll: {e:?}")))?;
    let res = await_request(&req).await;
    let _ = JsFuture::from(tx_done).await;
    let arr: js_sys::Array = res?
        .dyn_into()
        .map_err(|_| JsValue::from_str("getAll вернул не Array"))?;
    let mut out = Vec::with_capacity(arr.length() as usize);
    for i in 0..arr.length() {
        out.push(arr.get(i));
    }
    Ok(out)
}

// ============================================================
//                    ХЕЛПЕРЫ ТРАНЗАКЦИЙ
// ============================================================

fn open_tx(
    db: &JsValue,
    store_name: &str,
    mode: &str,
) -> Result<(JsValue, JsValue), JsValue> {
    let tx_fn: js_sys::Function = js_sys::Reflect::get(db, &JsValue::from_str("transaction"))
        .map_err(|e| JsValue::from_str(&format!("Reflect transaction: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str("transaction не функция"))?;
    let names = js_sys::Array::new();
    names.push(&JsValue::from_str(store_name));
    let tx = tx_fn
        .call2(db, &names, &JsValue::from_str(mode))
        .map_err(|e| JsValue::from_str(&format!("transaction: {e:?}")))?;
    let os_fn: js_sys::Function = js_sys::Reflect::get(&tx, &JsValue::from_str("objectStore"))
        .map_err(|e| JsValue::from_str(&format!("Reflect objectStore: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str("objectStore не функция"))?;
    let os = os_fn
        .call1(&tx, &JsValue::from_str(store_name))
        .map_err(|e| JsValue::from_str(&format!("objectStore: {e:?}")))?;
    Ok((os, tx))
}

fn method(obj: &JsValue, name: &str) -> Result<js_sys::Function, JsValue> {
    js_sys::Reflect::get(obj, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("Reflect {name}: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{name} не функция")))
}

async fn await_request(req: &JsValue) -> Result<JsValue, JsValue> {
    let r_ok = req.clone();
    let r_err = req.clone();
    let promise = js_sys::Promise::new(&mut move |resolve, reject| {
        let r = r_ok.clone();
        let resolve_c = resolve.clone();
        let on_ok = Closure::once_into_js(move |_e: web_sys::Event| {
            let v = js_sys::Reflect::get(&r, &JsValue::from_str("result"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = resolve_c.call1(&JsValue::UNDEFINED, &v);
        });
        let _ = js_sys::Reflect::set(&r_ok, &JsValue::from_str("onsuccess"), &on_ok);

        let r = r_err.clone();
        let reject_c = reject.clone();
        let on_err = Closure::once_into_js(move |_e: web_sys::Event| {
            let err = js_sys::Reflect::get(&r, &JsValue::from_str("error"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = reject_c.call1(&JsValue::UNDEFINED, &err);
        });
        let _ = js_sys::Reflect::set(&r_err, &JsValue::from_str("onerror"), &on_err);
    });
    JsFuture::from(promise).await
}

/// Резолвится, когда транзакция завершилась — успешно или через abort.
/// Не отклоняется: об ошибке конкретного запроса знает сам запрос.
fn tx_done_promise(tx: &JsValue) -> js_sys::Promise {
    let tx_c = tx.clone();
    js_sys::Promise::new(&mut move |resolve, _reject| {
        let r1 = resolve.clone();
        let on_complete = Closure::once_into_js(move || {
            let _ = r1.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&tx_c, &JsValue::from_str("oncomplete"), &on_complete);

        let r2 = resolve.clone();
        let on_abort = Closure::once_into_js(move || {
            let _ = r2.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&tx_c, &JsValue::from_str("onabort"), &on_abort);
    })
}

// ============================================================
//                    DEMO-БАЗА
// ============================================================

/// Открывает demo-базу со store `contacts` (`keyPath = "id"`).
/// Создаёт store при первом открытии. Возвращает `IDBDatabase`.
pub async fn open_demo_db(name: &str) -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let idb = js_sys::Reflect::get(&global, &JsValue::from_str("indexedDB"))
        .map_err(|e| JsValue::from_str(&format!("Reflect indexedDB: {e:?}")))?;
    if !idb.is_object() {
        return Err(JsValue::from_str("indexedDB недоступен"));
    }
    let open_fn: js_sys::Function = js_sys::Reflect::get(&idb, &JsValue::from_str("open"))
        .map_err(|e| JsValue::from_str(&format!("Reflect open: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str("open не функция"))?;

    let request = open_fn
        .call2(&idb, &JsValue::from_str(name), &JsValue::from_f64(1.0))
        .map_err(|e| JsValue::from_str(&format!("open: {e:?}")))?;

    let on_upgrade = Closure::once_into_js(move |e: web_sys::Event| {
        let Some(target) = e.target() else { return };
        let Ok(db) = js_sys::Reflect::get(&target, &JsValue::from_str("result")) else {
            return;
        };
        let Ok(create_fn_js) =
            js_sys::Reflect::get(&db, &JsValue::from_str("createObjectStore"))
        else {
            return;
        };
        let Ok(create_fn) = create_fn_js.dyn_into::<js_sys::Function>() else {
            return;
        };
        let opts = js_sys::Object::new();
        let _ = js_sys::Reflect::set(
            &opts,
            &JsValue::from_str("keyPath"),
            &JsValue::from_str("id"),
        );
        let _ = create_fn.call2(&db, &JsValue::from_str("contacts"), &opts);
    });
    let _ = js_sys::Reflect::set(
        &request,
        &JsValue::from_str("onupgradeneeded"),
        &on_upgrade,
    );

    await_request(&request).await
}

/// Удаляет базу целиком. Идемпотентно.
pub async fn delete_demo_db(name: &str) -> Result<(), JsValue> {
    let global = js_sys::global();
    let idb = js_sys::Reflect::get(&global, &JsValue::from_str("indexedDB"))
        .map_err(|e| JsValue::from_str(&format!("Reflect indexedDB: {e:?}")))?;
    let del_fn: js_sys::Function = js_sys::Reflect::get(&idb, &JsValue::from_str("deleteDatabase"))
        .map_err(|e| JsValue::from_str(&format!("Reflect deleteDatabase: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str("deleteDatabase не функция"))?;
    let request = del_fn
        .call1(&idb, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("deleteDatabase: {e:?}")))?;

    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let rc = resolve.clone();
        let on_ok = Closure::once_into_js(move || {
            let _ = rc.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&request, &JsValue::from_str("onsuccess"), &on_ok);
    });
    JsFuture::from(promise).await?;
    Ok(())
}
```

## 24.6.5. Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod idb_upsert;
```

## 24.6.6. Тесты — `tests/wasm_idb_upsert.rs`

Тесты делятся на три группы: классификация ошибок (3), одна операция (3), пачки (6). Плюс один тест на пустую пачку. Всего — 13 тестов.

```rust
//! Wasm-тесты модуля `idb_upsert`: семантика add/put, диагностика,
//! атомарность пачки, классификация ошибок.
//!
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`
//! или явный `return`. rust-analyzer теряет тип в этой позиции.
//!
//! ПРАВИЛО: каждое имя базы содержит уникальный суффикс `uid()` —
//! тесты не пересекаются между запусками.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::idb_upsert::{
    self, SaveOutcome, WriteError,
};

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64);
}

fn test_db_name() -> String {
    return format!("test-upsert-{}", uid());
}

fn make_record(id: &str, name: &str) -> JsValue {
    let o = js_sys::Object::new();
    let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(id));
    let _ = js_sys::Reflect::set(&o, &"name".into(), &JsValue::from_str(name));
    let _ = js_sys::Reflect::set(&o, &"updated_at".into(), &JsValue::from_f64(0.0));
    return o.into();
}

async fn fresh_db() -> (String, JsValue) {
    let name = test_db_name();
    let db = idb_upsert::open_demo_db(&name).await.expect("open_demo_db");
    return (name, db);
}

async fn cleanup(name: &str, db: &JsValue) {
    if let Ok(close_fn) = js_sys::Reflect::get(db, &"close".into()) {
        if let Ok(f) = close_fn.dyn_into::<js_sys::Function>() {
            let _ = f.call0(db);
        }
    }
    idb_upsert::delete_demo_db(name).await.expect("cleanup");
}

// ---------------- classify_write_error ----------------

#[wasm_bindgen_test]
fn classify_constraint_error() {
    let e = web_sys::DomException::new_with_message_and_name(
        "duplicate",
        "ConstraintError",
    )
    .expect("make DomException");
    let js: JsValue = e.into();
    assert_eq!(
        idb_upsert::classify_write_error(&js),
        WriteError::Constraint,
    );
    assert!(idb_upsert::is_constraint_error(&js));
}

#[wasm_bindgen_test]
fn classify_unknown_error() {
    let e = JsValue::from_str("some strange error");
    let kind = idb_upsert::classify_write_error(&e);
    assert_eq!(kind, WriteError::Other("some strange error".to_string()));
}

#[wasm_bindgen_test]
fn write_error_names() {
    assert_eq!(WriteError::Constraint.name(), "ConstraintError");
    assert_eq!(WriteError::Data.name(), "DataError");
    assert_eq!(WriteError::Quota.name(), "QuotaExceededError");
    assert_eq!(WriteError::Abort.name(), "AbortError");
}

// ---------------- add_one / put_one ----------------

#[wasm_bindgen_test]
async fn add_creates_then_conflicts() {
    let (name, db) = fresh_db().await;

    idb_upsert::add_one(&db, "contacts", &make_record("a", "Alice"))
        .await
        .expect("first add");

    let r = idb_upsert::add_one(&db, "contacts", &make_record("a", "Alice2")).await;
    assert!(r.is_err(), "повторный add с тем же ключом должен упасть");
    let kind = idb_upsert::classify_write_error(&r.unwrap_err());
    assert_eq!(kind, WriteError::Constraint);

    let count = idb_upsert::count_all(&db, "contacts").await.unwrap();
    assert_eq!(count, 1, "вторая запись не должна была записаться");

    cleanup(&name, &db).await;
}

#[wasm_bindgen_test]
async fn put_is_idempotent() {
    let (name, db) = fresh_db().await;

    idb_upsert::put_one(&db, "contacts", &make_record("a", "Alice"))
        .await
        .expect("first put");
    idb_upsert::put_one(&db, "contacts", &make_record("a", "Alice2"))
        .await
        .expect("second put");

    let count = idb_upsert::count_all(&db, "contacts").await.unwrap();
    assert_eq!(count, 1);

    let rec = idb_upsert::get_one(&db, "contacts", &JsValue::from_str("a"))
        .await
        .unwrap()
        .expect("present");
    let new_name = js_sys::Reflect::get(&rec, &"name".into())
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    assert_eq!(new_name, "Alice2", "put перезаписывает значение");

    cleanup(&name, &db).await;
}

// ---------------- save_with_diagnosis ----------------

#[wasm_bindgen_test]
async fn diagnosis_created_on_fresh_key() {
    let (name, db) = fresh_db().await;

    let report = idb_upsert::save_with_diagnosis(
        &db,
        "contacts",
        &make_record("a", "Alice"),
        &JsValue::from_str("a"),
    )
    .await
    .expect("save");

    assert_eq!(report.outcome, SaveOutcome::Created);
    assert!(report.previous.is_none());

    cleanup(&name, &db).await;
}

#[wasm_bindgen_test]
async fn diagnosis_updated_with_previous() {
    let (name, db) = fresh_db().await;

    idb_upsert::put_one(&db, "contacts", &make_record("a", "Alice"))
        .await
        .unwrap();

    let report = idb_upsert::save_with_diagnosis(
        &db,
        "contacts",
        &make_record("a", "Alice2"),
        &JsValue::from_str("a"),
    )
    .await
    .expect("save");

    assert_eq!(report.outcome, SaveOutcome::Updated);
    let prev = report.previous.expect("previous present");
    let old_name = js_sys::Reflect::get(&prev, &"name".into())
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    assert_eq!(old_name, "Alice", "diagnosis вернула старое значение");

    cleanup(&name, &db).await;
}

// ---------------- batch_add_atomic ----------------

#[wasm_bindgen_test]
async fn batch_add_succeeds_when_all_new() {
    let (name, db) = fresh_db().await;

    let values = vec![
        make_record("a", "A"),
        make_record("b", "B"),
        make_record("c", "C"),
    ];
    let out = idb_upsert::batch_add_atomic(&db, "contacts", &values)
        .await
        .expect("batch");

    assert_eq!(out.attempted, 3);
    assert_eq!(out.written, 3);
    assert!(out.aborted_at.is_none());
    assert!(out.abort_reason.is_none());

    let count = idb_upsert::count_all(&db, "contacts").await.unwrap();
    assert_eq!(count, 3);

    cleanup(&name, &db).await;
}

#[wasm_bindgen_test]
async fn batch_add_aborts_on_conflict_and_rolls_back() {
    let (name, db) = fresh_db().await;

    // Одна запись уже есть.
    idb_upsert::put_one(&db, "contacts", &make_record("a", "A"))
        .await
        .unwrap();

    // Пачка: d, e — новые, a — конфликт на 3-й позиции.
    let values = vec![
        make_record("d", "D"),
        make_record("e", "E"),
        make_record("a", "A2"),
    ];
    let out = idb_upsert::batch_add_atomic(&db, "contacts", &values)
        .await
        .expect("batch");

    assert_eq!(out.attempted, 3);
    assert_eq!(out.written, 2, "два запроса успели до конфликта");
    assert_eq!(out.aborted_at, Some(2), "пачка упала на 3-й записи");
    assert_eq!(out.abort_reason, Some(WriteError::Constraint));

    // Ключевое: несмотря на written=2, в store осталась только исходная
    // запись `a`. Транзакция откатила d и e.
    let count = idb_upsert::count_all(&db, "contacts").await.unwrap();
    assert_eq!(count, 1, "транзакция откатила всю пачку");

    cleanup(&name, &db).await;
}

#[wasm_bindgen_test]
async fn batch_add_aborts_on_first_conflict() {
    let (name, db) = fresh_db().await;

    idb_upsert::put_one(&db, "contacts", &make_record("a", "A"))
        .await
        .unwrap();

    let values = vec![
        make_record("a", "A2"), // конфликт сразу
        make_record("b", "B"),
    ];
    let out = idb_upsert::batch_add_atomic(&db, "contacts", &values)
        .await
        .expect("batch");

    assert_eq!(out.written, 0, "первый же запрос упал");
    assert_eq!(out.aborted_at, Some(0));
    assert_eq!(out.abort_reason, Some(WriteError::Constraint));

    cleanup(&name, &db).await;
}

// ---------------- batch_put_idempotent ----------------

#[wasm_bindgen_test]
async fn batch_put_never_aborts_on_existing_keys() {
    let (name, db) = fresh_db().await;

    // Три существующие записи.
    idb_upsert::put_one(&db, "contacts", &make_record("a", "A"))
        .await
        .unwrap();
    idb_upsert::put_one(&db, "contacts", &make_record("b", "B"))
        .await
        .unwrap();
    idb_upsert::put_one(&db, "contacts", &make_record("c", "C"))
        .await
        .unwrap();

    // put поверх существующих + один новый — всё должно пройти.
    let values = vec![
        make_record("a", "A2"),
        make_record("b", "B2"),
        make_record("d", "D"),
    ];
    let out = idb_upsert::batch_put_idempotent(&db, "contacts", &values)
        .await
        .expect("batch put");

    assert_eq!(out.attempted, 3);
    assert_eq!(out.written, 3);
    assert!(out.aborted_at.is_none());

    let count = idb_upsert::count_all(&db, "contacts").await.unwrap();
    assert_eq!(count, 4, "a,b,c,d — четыре записи");

    cleanup(&name, &db).await;
}

#[wasm_bindgen_test]
async fn batch_put_idempotent_on_repeat() {
    let (name, db) = fresh_db().await;

    let values = vec![
        make_record("a", "A"),
        make_record("b", "B"),
    ];

    let out1 = idb_upsert::batch_put_idempotent(&db, "contacts", &values)
        .await
        .unwrap();
    assert_eq!(out1.written, 2);

    // Повторный прогон — тот же результат.
    let out2 = idb_upsert::batch_put_idempotent(&db, "contacts", &values)
        .await
        .unwrap();
    assert_eq!(out2.written, 2);
    assert!(out2.aborted_at.is_none());

    let count = idb_upsert::count_all(&db, "contacts").await.unwrap();
    assert_eq!(count, 2);

    cleanup(&name, &db).await;
}

// ---------------- empty batch ----------------

#[wasm_bindgen_test]
async fn batch_empty_is_ok() {
    let (name, db) = fresh_db().await;

    let out = idb_upsert::batch_add_atomic(&db, "contacts", &[])
        .await
        .expect("empty batch");
    assert_eq!(out.attempted, 0);
    assert_eq!(out.written, 0);
    assert!(out.aborted_at.is_none());

    cleanup(&name, &db).await;
}
```

**Команды запуска:**

```bash
# Проверка компиляции
cargo check
cargo check --target wasm32-unknown-unknown

# Все тесты модуля
wasm-pack test --headless --chrome --test wasm_idb_upsert
wasm-pack test --headless --firefox --test wasm_idb_upsert

# Один тест по имени
wasm-pack test --headless --chrome --test wasm_idb_upsert -- \
    batch_add_aborts_on_conflict_and_rolls_back

# Группа по подстроке
wasm-pack test --headless --chrome --test wasm_idb_upsert -- batch_
wasm-pack test --headless --chrome --test wasm_idb_upsert -- diagnosis_
```

## 24.6.7. Демо — `src/app.rs`

Демо — **три колонки на одном экране**. Левая — про одну запись, центральная — про пачку, правая — про текущее состояние store.

```
┌─────────────────────┬─────────────────────────┬───────────────────┐
│ 1. Одна запись      │ 2. Пачка                │ Записи в store    │
│    три режима       │    атомарность          │ (live-таблица)    │
│                     │                         │                   │
│ • add               │ • batch add (atomic)    │ id | имя | время  │
│ • put               │ • batch put (idempotent)│ a  | ... | ...    │
│ • save_with_...     │                         │ d  | ... | ...    │
│                     │ attempted / written     │ e  | ... | ...    │
│ 3. Журнал           │ aborted_at / reason     │                   │
│ операций            │ в store: до / после     │                   │
│                     │                         │                   │
│                     │ Таблица «когда что»     │                   │
└─────────────────────┴─────────────────────────┴───────────────────┘
```

**Три сценария для проверки:**

1. **Одиночная операция.**
   - Выбрать `add`, id=`a`, имя=`Alice`. Нажать «Выполнить». Журнал: `add(a) → создано`.
   - Нажать ещё раз. Журнал: `add(a) → ConstraintError: ключ уже занят`.
   - Переключить на `put`, нажать. Журнал: `put(a) → OK`.
   - Переключить на `smart`, id=`a`. Журнал: `save(a) → обновлено: «Alice» → «Alice (updated)»`.

2. **Пачка — атомарность.**
   - Пресет `d, e, a` (id `a` уже занят).
   - Нажать «Batch add (atomic)». Результат: `written=2`, `aborted at=2`, `reason=ConstraintError`. В store до = 1, после = 1. `— вся пачка откатилась`.
   - Нажать «Batch put (idempotent)» с тем же пресетом. Результат: `written=3`, без отката. В store до = 1, после = 3.

3. **Идемпотентность повтора.**
   - Нажать «Batch put» дважды подряд. Оба раза — успех, счётчик в store не меняется.

**Структурные решения:**

- **Радио-кнопки, не select.** Три режима — визуально равнозначны, пользователь переключается мышью, а не открывает меню.
- **`в store до` / `в store после`** — два отдельных поля. Первое серым (это фон), второе жирным (это результат). Разница между ними — доказательство атомарности.
- **Журнал операций** — последние 8 записей, `ok` / `err` / `info` по цвету левой полосы. Автоскролл вниз.
- **Таблица решений** — статичная, статичный HTML. Это «шпаргалка», к которой возвращаются после экспериментов.

**Ключевой момент демо:** когда пользователь нажимает «Batch add» с пресетом `d, e, a` и видит `written=2`, но `в store: 1 → 1`, он впервые встречает разницу между «запрос успел выполниться» и «данные записались». Это самое ценное наблюдение главы.


**app.rs**
```
//! Демо Главы 24.6: `add` vs `put` — семантика вставки.
//!
//! Три эксперимента:
//!   1. Одна запись — три режима (add, put, save_with_diagnosis).
//!   2. Пачка — atomic add vs idempotent put, с показом отката.
//!   3. Статичная таблица «когда что использовать».
//!
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`.
//! ПРАВИЛО: HTML внутри `format_args!` с двумя и более `"` — через `push_str`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;

use crate::format_str::format_str;
use crate::idb_upsert::{self, SaveOutcome};

const DB_NAME: &str = "hello-wasm-idb-24-6-upsert";
const LOG_LIMIT: usize = 8;

const HTML: &str = r#"
<style>
  :root {
    --up-info: #3498db; --up-ok: #27ae60; --up-err: #c0392b; --up-warn: #e67e22;
    --up-text: #222; --up-muted: #666;
    --up-bg: #fff; --up-soft: #fafafa; --up-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --up-text: #e6e6e6; --up-muted: #a0a0a0;
      --up-bg: #1e1e1e; --up-soft: #262626; --up-border: #3a3a3a;
    }
    body { background: #121212; }
    .up-anno { background: #1a2233; color: #9fc4ff; }
    .up-anno code { background: rgba(52,152,219,0.2); }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    input::placeholder { color: #666; }
    .up-table tbody tr:nth-child(even) { background: rgba(128,128,128,0.08); }
    .up-log-entry { background: #1e1e1e; }
  }
  html, body { width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important; overflow-x: hidden; }
  body {
    display: block !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--up-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app {
    display: block !important; width: 100% !important; max-width: 100% !important;
    margin: 0 !important; padding: 12px 16px 24px !important;
    box-sizing: border-box !important;
  }
  #app, #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .up-subtitle { color: var(--up-muted); font-size: 12px; margin: 0 0 12px; }
  .up-subtitle code { background: rgba(128,128,128,0.15);
    padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .up-subtitle a { color: var(--up-info); }

  .up-banner {
    display: flex; align-items: center; gap: 14px; flex-wrap: wrap;
    padding: 8px 14px; margin: 0 0 12px;
    background: linear-gradient(135deg, #eaf4ff 0%, #f5f0ff 100%);
    border: 2px solid var(--up-info); border-radius: 8px;
  }
  @media (prefers-color-scheme: dark) {
    .up-banner { background: linear-gradient(135deg, #1a2a3f 0%, #2a1e3f 100%); }
  }
  .up-banner-label { font-size: 10px; text-transform: uppercase;
    letter-spacing: 0.7px; color: var(--up-muted); }
  .up-banner-name { font-family: ui-monospace, monospace; font-size: 13px;
    font-weight: 700; color: #1f4e79; word-break: break-all; }
  @media (prefers-color-scheme: dark) { .up-banner-name { color: #9fc4ff; } }
  .up-banner-actions { margin-left: auto; display: flex; gap: 8px; }
  .up-badge {
    display: inline-block; padding: 2px 10px; border-radius: 10px;
    background: #eaf4ff; color: #1f4e79;
    font-family: ui-monospace, monospace; font-size: 12px; font-weight: 700;
  }
  @media (prefers-color-scheme: dark) { .up-badge { background: #2a3a4a; color: #cfe0ff; } }

  .up-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(300px, 1fr) minmax(340px, 1.15fr) minmax(0, 1fr);
  }
  @media (max-width: 1100px) { .up-grid { grid-template-columns: 1fr 1fr; } }
  @media (max-width: 720px) { .up-grid { grid-template-columns: 1fr; } }
  .up-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .up-panel {
    border: 1px solid var(--up-border); border-radius: 8px;
    padding: 10px 12px; background: var(--up-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0;
  }
  .up-panel-title { font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .up-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--up-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }

  .up-anno { background: #eef4ff; border-left: 3px solid var(--up-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79; overflow-wrap: anywhere; word-break: break-word; }
  .up-anno code { background: rgba(52,152,219,0.12);
    padding: 0 4px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 10px; }
  .up-anno b { color: #0d3b66; }
  @media (prefers-color-scheme: dark) { .up-anno b { color: #c0d8ff; } }

  .up-form-row { display: flex; gap: 6px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; min-width: 0; }
  input[type="text"] {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--up-bg); color: var(--up-text);
    font-family: ui-monospace, monospace;
    flex: 1 1 100px; min-width: 0; max-width: 100%;
  }
  input[type="text"]:focus {
    outline: none; border-color: var(--up-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }
  .up-form-row label.up-radio {
    display: inline-flex; align-items: center; gap: 4px;
    font-size: 11px; color: var(--up-text); cursor: pointer;
    padding: 3px 8px; border: 1px solid var(--up-border); border-radius: 4px;
    background: var(--up-soft); user-select: none; white-space: nowrap;
  }
  .up-form-row label.up-radio:has(input:checked) {
    background: #eaf4ff; border-color: var(--up-info); color: #1f4e79;
  }
  @media (prefers-color-scheme: dark) {
    .up-form-row label.up-radio:has(input:checked) {
      background: #1a3a5a; color: #cfe0ff;
    }
  }
  .up-form-row label.up-radio input { margin: 0; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--up-soft); color: var(--up-text);
    cursor: pointer; transition: background 0.15s;
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--up-info); outline-offset: 2px; }
  button.up-primary { background: #eaf4ff; border-color: var(--up-info); color: #1f4e79; }
  button.up-danger { background: #fdedec; border-color: var(--up-err); color: #922b21; }
  button.up-success { background: #e8f8ed; border-color: var(--up-ok); color: #1e8449; }

  .up-status {
    display: flex; align-items: center; gap: 6px;
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--up-soft); border-left: 3px solid #ccc;
    color: var(--up-muted);
    min-height: 22px; overflow-wrap: anywhere; word-break: break-word;
  }
  .up-status.ok { background: #e8f8ed; border-color: var(--up-ok); color: #1e8449; }
  .up-status.err { background: #fdedec; border-color: var(--up-err); color: #922b21; }
  .up-status.busy { background: #fff9e6; border-color: var(--up-warn); color: #7d5a00; }

  .up-kv { display: grid; grid-template-columns: max-content 1fr;
    gap: 3px 12px; margin-top: 6px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .up-kv .k { color: var(--up-muted); white-space: nowrap; }
  .up-kv .v { color: var(--up-text); font-weight: 700; word-break: break-word; }
  .up-kv .v.dim { color: var(--up-muted); font-weight: 400; }
  .up-kv .v.warn { color: var(--up-warn); }

  .up-table-wrap {
    border: 1px solid var(--up-border); border-radius: 5px;
    background: var(--up-soft);
    max-height: 300px; overflow-y: auto;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .up-table-empty { padding: 14px; text-align: center;
    color: #999; font-style: italic; }
  table.up-table {
    width: 100%; border-collapse: collapse;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .up-table th, .up-table td {
    padding: 5px 9px; border-bottom: 1px solid var(--up-border);
    text-align: left; vertical-align: top;
  }
  .up-table th { background: var(--up-soft); color: var(--up-muted);
    font-weight: 600; font-size: 10px; text-transform: uppercase;
    letter-spacing: 0.5px; }
  .up-table tbody tr:nth-child(even) { background: rgba(128,128,128,0.04); }
  .up-table code { background: rgba(52,152,219,0.12);
    padding: 0 4px; border-radius: 3px; font-size: 10px; }

  .up-log {
    border: 1px solid var(--up-border); border-radius: 5px;
    background: var(--up-soft); max-height: 180px; overflow-y: auto;
    padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .up-log-empty { padding: 12px; text-align: center;
    color: #999; font-style: italic; }
  .up-log-entry { display: flex; gap: 8px; padding: 3px 10px;
    border-left: 3px solid transparent; }
  .up-log-entry.ok { border-left-color: var(--up-ok); }
  .up-log-entry.err { border-left-color: var(--up-err); }
  .up-log-entry.info { border-left-color: var(--up-info); }
  .up-log-entry .t { color: var(--up-muted); flex-shrink: 0; }
  .up-log-entry .m { color: var(--up-text); flex: 1; overflow-wrap: anywhere; }
  .up-log-entry.err .m { color: #922b21; }

  .up-count-badge {
    margin-left: auto; padding: 1px 10px; border-radius: 10px;
    background: #eaf4ff; color: #1f4e79;
    font-family: ui-monospace, monospace; font-size: 11px; font-weight: 700;
  }
  @media (prefers-color-scheme: dark) { .up-count-badge { background: #2a3a4a; color: #cfe0ff; } }
</style>

<h1>Глава 24.6: add vs put — семантика вставки</h1>
<p class="up-subtitle">
  <code>add</code> — insert, падает с <code>ConstraintError</code> на дубликате.
  <code>put</code> — upsert, тихо перезаписывает.
  Один конфликтный <code>add</code> в пачке откатывает всю транзакцию.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/add" target="_blank" rel="noopener">MDN →</a>
</p>

<div class="up-banner">
  <div>
    <div class="up-banner-label">рабочая база</div>
    <div class="up-banner-name" id="up-db-name">…</div>
  </div>
  <div class="up-banner-actions">
    <span class="up-badge" id="up-count">0 записей</span>
    <button id="up-reset" class="up-danger" title="Удалить базу и создать заново">
      ↻ Сбросить базу
    </button>
  </div>
</div>

<div class="up-grid">

  <!-- ============ Колонка 1: одна запись ============ -->
  <div class="up-col">

    <div class="up-panel">
      <h2 class="up-panel-title">
        <span class="up-num">1</span>Одна запись — три режима
      </h2>

      <div class="up-anno">
        <b>add</b> — строгая вставка. Ключ занят? <code>ConstraintError</code>.<br>
        <b>put</b> — upsert. Тот же ключ молча перезаписывается.<br>
        <b>save_with_diagnosis</b> — «умное сохранение»: сначала <code>add</code>,
        при конфликте читаем старое значение и делаем <code>put</code>.
      </div>

      <div class="up-form-row">
        <input type="text" id="up-id" placeholder="id" value="a" title="Первичный ключ (slug)">
        <input type="text" id="up-name" placeholder="имя" value="Alice">
      </div>

      <div class="up-form-row">
        <label class="up-radio" title="Строгая вставка">
          <input type="radio" name="up-mode" value="add" checked> add
        </label>
        <label class="up-radio" title="Upsert">
          <input type="radio" name="up-mode" value="put"> put
        </label>
        <label class="up-radio" title="add + fallback на put с диагностикой">
          <input type="radio" name="up-mode" value="smart"> save_with_diagnosis
        </label>
      </div>

      <div class="up-form-row">
        <button id="up-exec" class="up-primary">▶ Выполнить</button>
        <button id="up-fill-new">Пример: новый id</button>
        <button id="up-fill-existing">Пример: занятый id</button>
      </div>

      <div id="up-status" class="up-status">готово</div>
    </div>

    <div class="up-panel">
      <h2 class="up-panel-title">
        <span class="up-num">3</span>Журнал операций
      </h2>
      <div id="up-log" class="up-log">
        <div class="up-log-empty">здесь появятся выполненные операции</div>
      </div>
    </div>

  </div>

  <!-- ============ Колонка 2: пачка ============ -->
  <div class="up-col">

    <div class="up-panel">
      <h2 class="up-panel-title">
        <span class="up-num">2</span>Пачка — атомарность и идемпотентность
      </h2>

      <div class="up-anno">
        <b>batch_add_atomic</b> — все <code>add</code> в одной транзакции.
        Первый конфликт откатывает <b>всю</b> пачку — включая записи,
        которые уже прошли до падения.<br>
        <b>batch_put_idempotent</b> — те же данные через <code>put</code>.
        Проходит всегда целиком, безопасно повторять.
      </div>

      <div class="up-form-row">
        <input type="text" id="up-batch-ids" placeholder="id1, id2, id3" value="d, e, a">
        <button id="up-batch-preset" title="d, e, a — если id `a` уже занят, batch add откатится">preset: d, e, a</button>
      </div>

      <div class="up-form-row">
        <button id="up-batch-add" class="up-danger">▶ Batch add (atomic)</button>
        <button id="up-batch-put" class="up-success">▶ Batch put (idempotent)</button>
      </div>

      <div class="up-kv">
        <div class="k">attempted</div>          <div class="v" id="up-b-att">—</div>
        <div class="k">written (до отката)</div><div class="v" id="up-b-wr">—</div>
        <div class="k">aborted at</div>          <div class="v" id="up-b-ab">—</div>
        <div class="k">reason</div>              <div class="v" id="up-b-re">—</div>
        <div class="k">в store до</div>          <div class="v dim" id="up-b-before">—</div>
        <div class="k">в store после</div>       <div class="v" id="up-b-after">—</div>
      </div>

      <div id="up-batch-status" class="up-status">готово</div>
    </div>

    <div class="up-panel">
      <h2 class="up-panel-title">Когда что использовать</h2>
      <table class="up-table">
        <thead>
          <tr><th>Ситуация</th><th>Метод</th></tr>
        </thead>
        <tbody>
          <tr>
            <td>Регистрация нового пользователя</td>
            <td><code>add</code> — пусть форма покажет «такой id занят»</td>
          </tr>
          <tr>
            <td>Сохранение черновика</td>
            <td><code>put</code> — пользователь ждёт «сохранено», не «конфликт»</td>
          </tr>
          <tr>
            <td>Кнопка «Сохранить» с неясной семантикой</td>
            <td><code>add</code> + fallback на <code>put</code>, показать что именно произошло</td>
          </tr>
          <tr>
            <td>Импорт данных из файла</td>
            <td><code>put</code> — идемпотентен, безопасно перезапускать</td>
          </tr>
          <tr>
            <td>Пакетная проверка «все уникальны»</td>
            <td><code>add</code> в одной транзакции — откат докажет, что были конфликты</td>
          </tr>
        </tbody>
      </table>
    </div>

  </div>

  <!-- ============ Колонка 3: записи ============ -->
  <div class="up-col">

    <div class="up-panel">
      <h2 class="up-panel-title">
        Записи в store
        <span class="up-count-badge" id="up-count-2">0</span>
      </h2>
      <div id="up-table" class="up-table-wrap">
        <div class="up-table-empty">загрузка…</div>
      </div>
    </div>

  </div>

</div>
"#;

#[derive(Clone, Copy, PartialEq)]
enum LogKind { Ok, Err, Info }

struct LogEntry {
    time: String,
    text: String,
    kind: LogKind,
}

struct App {
    db: Rc<RefCell<Option<JsValue>>>,
    log: Rc<RefCell<Vec<LogEntry>>>,
    ui: Ui,
}

#[derive(Clone)]
struct Ui {
    db_name: web_sys::Element,
    count: web_sys::Element,
    count_2: web_sys::Element,
    status: web_sys::Element,
    log: web_sys::Element,
    table: web_sys::Element,
    input_id: web_sys::HtmlInputElement,
    input_name: web_sys::HtmlInputElement,
    batch_ids: web_sys::HtmlInputElement,
    b_att: web_sys::Element,
    b_wr: web_sys::Element,
    b_ab: web_sys::Element,
    b_re: web_sys::Element,
    b_before: web_sys::Element,
    b_after: web_sys::Element,
    batch_status: web_sys::Element,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        db_name: get_el("up-db-name")?,
        count: get_el("up-count")?,
        count_2: get_el("up-count-2")?,
        status: get_el("up-status")?,
        log: get_el("up-log")?,
        table: get_el("up-table")?,
        input_id: get_input("up-id")?,
        input_name: get_input("up-name")?,
        batch_ids: get_input("up-batch-ids")?,
        b_att: get_el("up-b-att")?,
        b_wr: get_el("up-b-wr")?,
        b_ab: get_el("up-b-ab")?,
        b_re: get_el("up-b-re")?,
        b_before: get_el("up-b-before")?,
        b_after: get_el("up-b-after")?,
        batch_status: get_el("up-batch-status")?,
    };

    let app = Rc::new(App {
        db: Rc::new(RefCell::new(None)),
        log: Rc::new(RefCell::new(Vec::new())),
        ui,
    });

    // Старт: открыть базу.
    {
        let app_c = app.clone();
        spawn_local(async move {
            match idb_upsert::open_demo_db(DB_NAME).await {
                Ok(db) => {
                    *app_c.db.borrow_mut() = Some(db);
                    app_c.ui.db_name.set_text_content(Some(DB_NAME));
                    app_c.push_log(LogKind::Info, "база открыта".to_string());
                    refresh_view(&app_c).await;
                }
                Err(e) => {
                    set_status(&app_c.ui.status, "err", &format_str(format_args!(
                        "open: {}", short_error(&e),
                    )));
                }
            }
        });
    }

    // Reset
    {
        let app_c = app.clone();
        let btn = get_button("up-reset")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let ok = web_sys::window()
                .and_then(|w| w.confirm_with_message(
                    "Удалить базу целиком и создать заново?"
                ).ok())
                .unwrap_or(false);
            if !ok { return; }

            set_status(&app_in.ui.status, "busy", "сброс…");
            spawn_local(async move {
                if let Some(db) = app_in.db.borrow_mut().take() {
                    if let Ok(cf) = js_sys::Reflect::get(&db, &"close".into()) {
                        if let Ok(f) = cf.dyn_into::<js_sys::Function>() {
                            let _ = f.call0(&db);
                        }
                    }
                }
                let _ = idb_upsert::delete_demo_db(DB_NAME).await;
                match idb_upsert::open_demo_db(DB_NAME).await {
                    Ok(db) => {
                        *app_in.db.borrow_mut() = Some(db);
                        app_in.push_log(LogKind::Info, "база пересоздана".to_string());
                        set_status(&app_in.ui.status, "ok", "база пересоздана");
                        refresh_view(&app_in).await;
                    }
                    Err(e) => set_status(&app_in.ui.status, "err", &format_str(format_args!(
                        "open: {}", short_error(&e),
                    ))),
                }
            });
        });
    }

    // Заполнить: новый
    {
        let app_c = app.clone();
        let btn = get_button("up-fill-new")?;
        set_click(&btn, move |_| {
            let n = (js_sys::Date::now() as u64) % 1000;
            app_c.ui.input_id.set_value(&format_str(format_args!("u{n}")));
            app_c.ui.input_name.set_value(&format_str(format_args!("User {n}")));
        });
    }

    // Заполнить: занятый
    {
        let app_c = app.clone();
        let btn = get_button("up-fill-existing")?;
        set_click(&btn, move |_| {
            app_c.ui.input_id.set_value("a");
            app_c.ui.input_name.set_value("Alice (updated)");
        });
    }

    // Выполнить одну операцию
    {
        let app_c = app.clone();
        let btn = get_button("up-exec")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            exec_single(app_in);
        });
    }

    // Preset пачки
    {
        let app_c = app.clone();
        let btn = get_button("up-batch-preset")?;
        set_click(&btn, move |_| {
            app_c.ui.batch_ids.set_value("d, e, a");
        });
    }

    // Batch add
    {
        let app_c = app.clone();
        let btn = get_button("up-batch-add")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            exec_batch(app_in, true);
        });
    }

    // Batch put
    {
        let app_c = app.clone();
        let btn = get_button("up-batch-put")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            exec_batch(app_in, false);
        });
    }

    Ok(())
}

// ============================================================
//                  Операции
// ============================================================

fn exec_single(app: Rc<App>) {
    let id = app.ui.input_id.value().trim().to_string();
    let name = app.ui.input_name.value().trim().to_string();
    if id.is_empty() || name.is_empty() {
        set_status(&app.ui.status, "err", "нужны id и имя");
        return;
    }
    let mode = current_mode();

    set_status(&app.ui.status, "busy", &format_str(format_args!(
        "{mode}: {id} = «{name}»…"
    )));

    spawn_local(async move {
        let Some(db) = app.db.borrow().clone() else {
            set_status(&app.ui.status, "err", "база не открыта");
            return;
        };
        let rec = make_record(&id, &name);
        let key = JsValue::from_str(&id);

        match mode.as_str() {
            "add" => {
                match idb_upsert::add_one(&db, "contacts", &rec).await {
                    Ok(()) => {
                        let msg = format_str(format_args!("add({id}) → создано"));
                        app.push_log(LogKind::Ok, msg.clone());
                        set_status(&app.ui.status, "ok", &msg);
                    }
                    Err(e) => {
                        let kind = idb_upsert::classify_write_error(&e);
                        let msg = format_str(format_args!(
                            "add({id}) → {}: {}",
                            kind.name(), kind.short(),
                        ));
                        app.push_log(LogKind::Err, msg.clone());
                        set_status(&app.ui.status, "err", &msg);
                    }
                }
            }
            "put" => {
                match idb_upsert::put_one(&db, "contacts", &rec).await {
                    Ok(()) => {
                        let msg = format_str(format_args!("put({id}) → OK"));
                        app.push_log(LogKind::Ok, msg.clone());
                        set_status(&app.ui.status, "ok", &msg);
                    }
                    Err(e) => {
                        let msg = format_str(format_args!(
                            "put({id}) → {}", short_error(&e),
                        ));
                        app.push_log(LogKind::Err, msg.clone());
                        set_status(&app.ui.status, "err", &msg);
                    }
                }
            }
            _ => {
                // smart
                match idb_upsert::save_with_diagnosis(&db, "contacts", &rec, &key).await {
                    Ok(r) => match r.outcome {
                        SaveOutcome::Created => {
                            let msg = format_str(format_args!(
                                "save({id}) → создано"
                            ));
                            app.push_log(LogKind::Ok, msg.clone());
                            set_status(&app.ui.status, "ok", &msg);
                        }
                        SaveOutcome::Updated => {
                            let prev_name = r.previous
                                .as_ref()
                                .and_then(|p| js_sys::Reflect::get(p, &"name".into()).ok())
                                .and_then(|v| v.as_string())
                                .unwrap_or_else(|| "—".to_string());
                            let msg = format_str(format_args!(
                                "save({id}) → обновлено: «{prev_name}» → «{name}»"
                            ));
                            app.push_log(LogKind::Ok, msg.clone());
                            set_status(&app.ui.status, "ok", &msg);
                        }
                    },
                    Err(e) => {
                        let kind = idb_upsert::classify_write_error(&e);
                        let msg = format_str(format_args!(
                            "save({id}) → {}: {}", kind.name(), kind.short(),
                        ));
                        app.push_log(LogKind::Err, msg.clone());
                        set_status(&app.ui.status, "err", &msg);
                    }
                }
            }
        }

        refresh_view(&app).await;
    });
}

fn current_mode() -> String {
    let document = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d,
        None => return "add".into(),
    };
    let radios = match document.query_selector_all("input[name=\"up-mode\"]") {
        Ok(r) => r,
        Err(_) => return "add".into(),
    };
    for i in 0..radios.length() {
        if let Some(node) = radios.item(i) {
            if let Ok(input) = node.dyn_into::<web_sys::HtmlInputElement>() {
                if input.checked() {
                    return input.value();
                }
            }
        }
    }
    return "add".into();
}

fn exec_batch(app: Rc<App>, is_add: bool) {
    let raw = app.ui.batch_ids.value();
    let ids: Vec<String> = raw
        .split(',')
        .map(|s| s.trim().to_string())
        .filter(|s| !s.is_empty())
        .collect();
    if ids.is_empty() {
        set_status(&app.ui.batch_status, "err", "введите хотя бы один id");
        return;
    }

    let label = if is_add { "batch add" } else { "batch put" };
    set_status(&app.ui.batch_status, "busy", &format_str(format_args!(
        "{label}: {} записей…", ids.len(),
    )));

    spawn_local(async move {
        let Some(db) = app.db.borrow().clone() else {
            set_status(&app.ui.batch_status, "err", "база не открыта");
            return;
        };

        let before = idb_upsert::count_all(&db, "contacts").await.unwrap_or(0);
        app.ui.b_before.set_text_content(Some(&before.to_string()));

        let values: Vec<JsValue> = ids.iter()
            .map(|id| make_record(id, &format_str(format_args!("name-{id}"))))
            .collect();

        let result = if is_add {
            idb_upsert::batch_add_atomic(&db, "contacts", &values).await
        } else {
            idb_upsert::batch_put_idempotent(&db, "contacts", &values).await
        };

        match result {
            Ok(out) => {
                app.ui.b_att.set_text_content(Some(&out.attempted.to_string()));
                app.ui.b_wr.set_text_content(Some(&out.written.to_string()));
                app.ui.b_ab.set_text_content(Some(
                    &out.aborted_at.map(|i| i.to_string()).unwrap_or_else(|| "—".into()),
                ));
                app.ui.b_re.set_text_content(Some(
                    &out.abort_reason.as_ref()
                        .map(|r| format_str(format_args!("{} ({})", r.name(), r.short())))
                        .unwrap_or_else(|| "—".into()),
                ));

                let after = idb_upsert::count_all(&db, "contacts").await.unwrap_or(0);
                app.ui.b_after.set_text_content(Some(&after.to_string()));

                let rolled_back = out.aborted_at.is_some() && after == before;

                let summary = if let Some(idx) = out.aborted_at {
                    let reason = out.abort_reason.as_ref()
                        .map(|r| r.name())
                        .unwrap_or("?");
                    format_str(format_args!(
                        "{label}: откат на записи #{idx} ({reason}). \
                         В store: было {before}, стало {after}{}",
                        if rolled_back { " — вся пачка откатилась" } else { "" },
                    ))
                } else {
                    format_str(format_args!(
                        "{label}: все {attempted} записей прошли. \
                         В store: было {before}, стало {after}",
                        attempted = out.attempted,
                    ))
                };

                let kind = if out.aborted_at.is_some() { LogKind::Err } else { LogKind::Ok };
                app.push_log(kind, summary.clone());
                set_status(&app.ui.batch_status,
                    if out.aborted_at.is_some() { "err" } else { "ok" },
                    &summary);
            }
            Err(e) => {
                let msg = format_str(format_args!("{label}: {}", short_error(&e)));
                app.push_log(LogKind::Err, msg.clone());
                set_status(&app.ui.batch_status, "err", &msg);
            }
        }

        refresh_view(&app).await;
    });
}

// ============================================================
//                  View
// ============================================================

async fn refresh_view(app: &Rc<App>) {
    let Some(db) = app.db.borrow().clone() else { return; };

    let count = idb_upsert::count_all(&db, "contacts").await.unwrap_or(0);
    app.ui.count.set_text_content(Some(&format_str(format_args!(
        "{count} записей"
    ))));
    app.ui.count_2.set_text_content(Some(&count.to_string()));

    let records = match idb_upsert::list_all(&db, "contacts").await {
        Ok(r) => r,
        Err(e) => {
            app.ui.table.set_inner_html(&format_str(format_args!(
                "<div class=\"up-table-empty\">ошибка чтения: {}</div>",
                esc(&short_error(&e)),
            )));
            return;
        }
    };

    if records.is_empty() {
        app.ui.table.set_inner_html(
            "<div class=\"up-table-empty\">пусто — добавьте запись</div>");
        return;
    }

    // Сортируем по id для стабильного порядка.
    let mut pairs: Vec<(String, JsValue)> = records.into_iter().map(|r| {
        let id = js_sys::Reflect::get(&r, &"id".into())
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        (id, r)
    }).collect();
    pairs.sort_by(|a, b| a.0.cmp(&b.0));

    let mut html = String::with_capacity(512);
    html.push_str("<table class=\"up-table\"><thead><tr>");
    html.push_str("<th>id</th><th>имя</th><th>updated_at</th>");
    html.push_str("</tr></thead><tbody>");

    for (_id, r) in &pairs {
        let id = js_sys::Reflect::get(r, &"id".into())
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        let name = js_sys::Reflect::get(r, &"name".into())
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        let ts = js_sys::Reflect::get(r, &"updated_at".into())
            .ok().and_then(|v| v.as_f64()).unwrap_or(0.0);

        html.push_str("<tr>");
        html.push_str(&format_str(format_args!("<td><code>{}</code></td>", esc(&id))));
        html.push_str(&format_str(format_args!("<td>{}</td>", esc(&name))));
        html.push_str(&format_str(format_args!("<td>{}</td>", esc(&fmt_time(ts)))));
        html.push_str("</tr>");
    }

    html.push_str("</tbody></table>");
    app.ui.table.set_inner_html(&html);
}

impl App {
    fn push_log(&self, kind: LogKind, text: String) {
        let t = fmt_now_hms();
        self.log.borrow_mut().push(LogEntry { time: t, text, kind });
        let mut v = self.log.borrow_mut();
        if v.len() > LOG_LIMIT {
            let drop_n = v.len() - LOG_LIMIT;
            v.drain(0..drop_n);
        }
        drop(v);
        self.render_log();
    }

    fn render_log(&self) {
        let entries = self.log.borrow();
        if entries.is_empty() {
            self.ui.log.set_inner_html(
                "<div class=\"up-log-empty\">здесь появятся выполненные операции</div>");
            return;
        }
        let mut html = String::with_capacity(256);
        for e in entries.iter() {
            let cls = match e.kind {
                LogKind::Ok => "ok",
                LogKind::Err => "err",
                LogKind::Info => "info",
            };
            html.push_str(&format_str(format_args!(
                "<div class=\"up-log-entry {cls}\">\
                 <span class=\"t\">{}</span>\
                 <span class=\"m\">{}</span>\
                 </div>",
                esc(&e.time), esc(&e.text),
            )));
        }
        self.ui.log.set_inner_html(&html);
        self.ui.log.set_scroll_top(self.ui.log.scroll_height() as f64);
    }
}

// ============================================================
//                  Хелперы
// ============================================================

fn make_record(id: &str, name: &str) -> JsValue {
    let o = js_sys::Object::new();
    let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(id));
    let _ = js_sys::Reflect::set(&o, &"name".into(), &JsValue::from_str(name));
    let _ = js_sys::Reflect::set(
        &o, &"updated_at".into(), &JsValue::from_f64(js_sys::Date::now()),
    );
    o.into()
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "up-status ok",
        "err" => "up-status err",
        "busy" => "up-status busy",
        _ => "up-status",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn fmt_time(ms: f64) -> String {
    if ms <= 0.0 { return "—".to_string(); }
    let d = js_sys::Date::new(&JsValue::from_f64(ms));
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(), d.get_minutes(), d.get_seconds(),
    ));
}

fn fmt_now_hms() -> String {
    let d = js_sys::Date::new_0();
    return format_str(format_args!(
        "{:02}:{:02}:{:02}",
        d.get_hours(), d.get_minutes(), d.get_seconds(),
    ));
}

fn short_error(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    if let Some(s) = e.as_string() {
        return s.lines().next().unwrap_or("").trim().to_string();
    }
    format_str(format_args!("{e:?}"))
}

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window().ok_or("no window")?
        .document().ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}

fn get_input(id: &str) -> Result<web_sys::HtmlInputElement, JsValue> {
    get_el(id)?.dyn_into().map_err(|_| JsValue::from_str(&format!("#{id} не input")))
}

fn get_button(id: &str) -> Result<web_sys::HtmlButtonElement, JsValue> {
    get_el(id)?.dyn_into().map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}

fn set_click<F>(el: &web_sys::HtmlButtonElement, f: F)
where F: FnMut(web_sys::MouseEvent) + 'static {
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn esc(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}
```


## 24.6.8. Анализ решения

**Что демонстрирует каждый эксперимент:**

| Эксперимент | Что доказывает | Ключевая цифра |
|---|---|---|
| 1. Одна запись | `add` падает, `put` — нет, `smart` — диагностирует | `ConstraintError` vs «обновлено» |
| 2. Пачка | Один конфликт откатывает всю транзакцию | `written=2`, но `в store после = до` |
| 3. Таблица | Практическое правило выбора | 5 сценариев → 3 метода |

**Что было в первой версии и почему переделано:**

- **Первая версия не показывала откат явно.** Она выводила `written` без сравнения с состоянием store до и после. Студент делал вывод «две записи записались». Добавлены поля `в store до` / `в store после` — они доказывают атомарность.
- **Первая версия использовала одну кнопку «Сохранить».** Разница семантик размывалась. Заменено на радио-переключатель с явными названиями методов.
- **Первая версия не показывала `previous` в диагностике.** «Обновлено» — половина истории. Правильный UX: «обновлено: было «Alice» → стало «Alice2»». Добавлено извлечение `previous.name`.
- **Первая версия не проверяла идемпотентность `put`.** Теперь это отдельный сценарий: нажать `Batch put` дважды, увидеть тот же результат.

**Ключевая находка итераций:** `written` в `BatchOutcome` — это **не** «сколько записалось». Это «сколько запросов успели получить `onsuccess` до abort'а». Студент, впервые встретившийся с этим, думает: «2 запроса прошли, значит 2 записи в store». На самом деле 0 записей в store — транзакция откатила всё. Именно поэтому колонка `в store после` обязательна в UI: она разрушает ложную интуицию.

**Почему диагностика `save_with_diagnosis` race-free.** Первая версия делала `get` → если `None` → `add`, иначе `put`. Это классический TOCTOU (time-of-check vs time-of-use): между `get` и `add` другая вкладка может вставить ту же запись. `ConstraintError` от `add` — это **вердикт движка**, а не результат предварительной проверки. Он не может «протухнуть» между проверкой и действием.

**Производительность.** `save_with_diagnosis` делает одну транзакцию `add` в happy path и две (add + get + put) в fallback. `batch_add_atomic` — одну транзакцию на N записей, `batch_put_idempotent` — тоже одну. Это соответствует правилу «одна логическая операция = одна транзакция».

## 24.6.9. Правила проекта, закреплённые в главе

1. **`JsValue`-ошибки не переупаковываются в строки.** `format!("{e:?}")` уничтожает `name`/`message`. `classify_write_error` извлекает `name` явно через `dyn_ref::<DomException>()` с fallback на `Reflect::get("name")`.
2. **`written` ≠ «записано».** В пачечной операции `written` считает `onsuccess`-события до abort'а. После отката записей в store не остаётся. UI обязан показывать `в store до` / `после` — иначе пользователь делает ложный вывод.
3. **`ConstraintError` — это вердикт движка.** Проверка «ключ занят» через `get` → `add` имеет race. Единственный race-free способ — попробовать `add` и обработать `ConstraintError`.
4. **Одна логическая операция = одна транзакция.** Все `put`/`add` пачки летят синхронно в одной транзакции, `await tx.oncomplete` — один раз в конце. Промежуточный `await` разрывает транзакцию.
5. **Подписка на `tx.oncomplete`/`tx.onabort` — до первого запроса.** Иначе событие улетает до того, как `await` начнёт слушать, и промис висит вечно.
6. **`put` — идемпотентная запись.** Если операция должна быть безопасной при повторе (retry после сбоя сети), используйте `put`. `add` — только когда нужно сигнализировать о конфликте.
7. **`format!` в tail-позиции — через `format_str(format_args!(...))`.** rust-analyzer теряет тип `String` в этой позиции.
8. **HTML внутри `format_args!` с ≥2 `"` — через `push_str`.** Иначе парсер путает `}` от атрибута с закрытием блока.
9. **`scroll_height()` возвращает `i32`, `set_scroll_top` принимает `f64`.** Явный каст `. as f64` — обязателен.

## 24.6.10. Итоги главы

- `add` — insert, `put` — upsert. Это не косметика, а семантика.
- `ConstraintError` — единственный race-free способ узнать, что ключ занят.
- Один конфликтный `add` в пачке откатывает всю транзакцию.
- `written` в отчёте пачки — не «сколько записалось», а «сколько успело до abort'а».
- `save_with_diagnosis` — паттерн для UI, где семантика «создать или обновить» зависит от состояния.
- `batch_put_idempotent` — единственный способ гарантировать, что повторный импорт не упадёт.
- Классификация `DOMException` — обязательна для корректного UX.

**Мостик к 24.7.** Мы умеем записывать данные с нужной семантикой. Дальше — **события транзакций**: `oncomplete`, `onerror`, `onabort`, `event.prevent_default()`. Без них невозможно понять, что произошло с транзакцией после того, как все запросы завершились — успех, откат, или откат, который можно остановить.