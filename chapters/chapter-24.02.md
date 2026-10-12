# Глава 24.2. `IdbFactory` — точка входа

## 24.2.0. Что вы получите в этой главе

Глава 24.1 показала пределы `localStorage`: мал, дорог, спроектирован синхронным. Глава 24.2 — первый шаг к IndexedDB. Мы работаем с её **точкой входа** — `IDBFactory`, объектом `window.indexedDB`.

К концу главы у вас будет работающее демо с тремя экспериментами:

1. **Lifecycle.** Версия — часть контракта. Понизить нельзя, браузер вернёт `VersionError`.
2. **cmp.** Детерминированный порядок ключей: `number < date < string < binary < array`.
3. **Blocked.** Открытое соединение блокирует удаление базы — без обработки `onblocked` промис зависает навсегда.

Плюс инспектор баз — живой список всех IndexedDB-баз текущего origin.

## 24.2.1. Почему фабрика — правильная отправная точка

`IDBFactory` — единственный объект IndexedDB, доступный **напрямую из window** (`window.indexedDB`). Всё остальное — базы, store'ы, транзакции, запросы — открывается через неё.

Фабрика отвечает на четыре вопроса, которых у `localStorage` не было:

| Вопрос | Метод |
|---|---|
| Как открыть базу с определённой схемой? | `open(name, version)` |
| Как узнать, что вообще есть в origin? | `databases()` |
| Как удалить всё целиком? | `deleteDatabase(name)` |
| Как сравниваются ключи? | `cmp(a, b)` |

## 24.2.2. Обновлённая структура файлов

```
hello-wasm02/
├── src/
│   ├── idb_factory.rs       # НОВЫЙ: обёртка над IDBFactory — 4 инструмента
│   ├── lib.rs               # +1 модуль
│   └── app.rs               # демо — три эксперимента
├── tests/
│   └── idb_factory_test.rs  # НОВЫЙ: wasm-тесты (18 штук)
└── Cargo.toml               # без изменений
```

Никаких новых features в `Cargo.toml` не требуется — `Event`, `EventTarget`, `DomException`, `HtmlProgressElement` уже включены.

**Ссылки для детального изучения:**
- MDN, `IDBFactory`: https://developer.mozilla.org/en-US/docs/Web/API/IDBFactory
- MDN, `IDBFactory.open()`: https://developer.mozilla.org/en-US/docs/Web/API/IDBFactory/open
- MDN, `IDBFactory.deleteDatabase()`: https://developer.mozilla.org/en-US/docs/Web/API/IDBFactory/deleteDatabase
- Spec, «Keys»: https://w3c.github.io/IndexedDB/#key-construct

## 24.2.3. Исходный код `src/idb_factory.rs`

```rust
//! Обёртка над `IDBFactory` — точкой входа в IndexedDB.
//!
//! Работаем с `window.indexedDB` напрямую через `js_sys::Reflect`, не
//! используя обёртку `idb` crate: не все методы `IDBFactory` обёрнуты
//! в `idb 0.6.5`, и единый стиль через Reflect даёт простой код,
//! устойчивый к версии `web-sys`.
//!
//! Четыре инструмента фабрики:
//!   • `open_demo()` — открыть demo-базу (создаёт при первом открытии);
//!   • `delete_by_name()` — удалить базу целиком. Идемпотентно;
//!   • `list_databases()` — перечислить все базы origin;
//!   • `compare_keys()` — сравнить два ключа по правилам IndexedDB.
//!
//! ПРАВИЛО: `IDBRequest` — не `Promise`. У него события `success`/`error`.
//! Оборачиваем в `Promise` вручную через `request_to_promise`.
//!
//! ПРАВИЛО: ошибки `IDBRequest` пробрасываются как `JsValue` **без
//! обёртки в строку**. `JsValue::from_str(&format!("{e:?}"))` уничтожает
//! свойства `name`/`message` исходного `DOMException`, и `describe_error`
//! перестаёт распознавать `VersionError`, `QuotaExceededError` и т. д.

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

/// Имя demo-базы этой подглавы.
pub const DEMO_DB: &str = "hello-wasm-idb-factory-demo";
/// Версия схемы demo-базы.
pub const DEMO_VERSION: u32 = 1;

/// Одна запись в списке баз: имя и текущая версия.
#[derive(Debug, Clone, PartialEq)]
pub struct DbInfo {
    pub name: String,
    pub version: u32,
}

/// Возвращает `window.indexedDB`.
fn indexed_db() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let idb = js_sys::Reflect::get(&global, &JsValue::from_str("indexedDB"))
        .map_err(|e| JsValue::from_str(&format!("Reflect indexedDB: {e:?}")))?;
    if idb.is_undefined() || idb.is_null() {
        return Err(JsValue::from_str("indexedDB недоступен (нужен secure context)"));
    }
    Ok(idb)
}

/// Оборачивает `IDBRequest` в `Promise`.
///
/// `IDBRequest` не является `Promise`: у него события `success`/`error`.
/// Слушаем и `blocked`: он срабатывает, если другой таб держит соединение
/// и мешает удалить/обновить базу. Без него промис висит навсегда —
/// на success/error в этом случае никто не сработает.
fn request_to_promise(request: &JsValue) -> Result<js_sys::Promise, JsValue> {
    let req_ok = request.clone();
    let req_err = request.clone();
    let req_blocked = request.clone();

    let promise = js_sys::Promise::new(&mut move |resolve, reject| {
        // onsuccess: resolve(request.result)
        let r = req_ok.clone();
        let resolve_in = resolve.clone();
        let on_success = Closure::once_into_js(move |_e: web_sys::Event| {
            let result = js_sys::Reflect::get(&r, &JsValue::from_str("result"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = resolve_in.call1(&JsValue::UNDEFINED, &result);
        });
        let _ = js_sys::Reflect::set(
            &req_ok,
            &JsValue::from_str("onsuccess"),
            &on_success,
        );

        // onerror: reject(request.error)
        let r = req_err.clone();
        let reject_in = reject.clone();
        let on_error = Closure::once_into_js(move |_e: web_sys::Event| {
            let error = js_sys::Reflect::get(&r, &JsValue::from_str("error"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = reject_in.call1(&JsValue::UNDEFINED, &error);
        });
        let _ = js_sys::Reflect::set(
            &req_err,
            &JsValue::from_str("onerror"),
            &on_error,
        );

        // onblocked: reject сразу. Иначе — бесконечное ожидание, если
        // другой таб держит открытое соединение с этой базой.
        let reject_blocked = reject.clone();
        let on_blocked = Closure::once_into_js(move |_e: web_sys::Event| {
            let msg = JsValue::from_str("blocked: соединение держится другим табом");
            let _ = reject_blocked.call1(&JsValue::UNDEFINED, &msg);
        });
        let _ = js_sys::Reflect::set(
            &req_blocked,
            &JsValue::from_str("onblocked"),
            &on_blocked,
        );
    });

    Ok(promise)
}

/// Открывает demo-базу. При первом открытии — создаёт пустую.
pub async fn open_demo() -> Result<JsValue, JsValue> {
    open_by_name(DEMO_DB, DEMO_VERSION).await
}

/// Открывает базу с указанным именем и версией.
///
/// `onupgradeneeded` не устанавливаем — база создаётся пустой, без
/// object stores. Схему (stores, indexes) добавляем в следующих подглавах.
pub async fn open_by_name(name: &str, version: u32) -> Result<JsValue, JsValue> {
    let factory = indexed_db()?;
    let open_fn = js_sys::Reflect::get(&factory, &JsValue::from_str("open"))
        .map_err(|e| JsValue::from_str(&format!("Reflect open: {e:?}")))?;
    let open: js_sys::Function = open_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("open не функция"))?;

    let request = open
        .call2(
            &factory,
            &JsValue::from_str(name),
            &JsValue::from_f64(version as f64),
        )
        .map_err(|e| JsValue::from_str(&format!("open: {e:?}")))?;

    let promise = request_to_promise(&request)?;
    JsFuture::from(promise).await
}

/// Удаляет demo-базу. Идемпотентно.
pub async fn delete_demo() -> Result<(), JsValue> {
    delete_by_name(DEMO_DB).await
}

/// Удаляет базу с указанным именем.
///
/// Идемпотентно: повторный вызов — `Ok`. Если базы нет, `deleteDatabase`
/// резолвится с `success`, `request.error` не выставляется.
pub async fn delete_by_name(name: &str) -> Result<(), JsValue> {
    let factory = indexed_db()?;
    let del_fn = js_sys::Reflect::get(&factory, &JsValue::from_str("deleteDatabase"))
        .map_err(|e| JsValue::from_str(&format!("Reflect deleteDatabase: {e:?}")))?;
    let del: js_sys::Function = del_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("deleteDatabase не функция"))?;

    let request = del
        .call1(&factory, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("deleteDatabase: {e:?}")))?;

    let promise = request_to_promise(&request)?;
    JsFuture::from(promise).await?;
    Ok(())
}

/// Возвращает список баз в текущем origin.
///
/// `databases()` возвращает **настоящий** `Promise` (не `IDBRequest`).
/// Каждый элемент — `{ name: string, version: number }`.
pub async fn list_databases() -> Result<Vec<DbInfo>, JsValue> {
    let factory = indexed_db()?;
    let fn_val = js_sys::Reflect::get(&factory, &JsValue::from_str("databases"))
        .map_err(|e| JsValue::from_str(&format!("Reflect databases: {e:?}")))?;
    if fn_val.is_undefined() {
        return Err(JsValue::from_str(
            "IDBFactory.databases() не поддерживается этим браузером",
        ));
    }
    let func: js_sys::Function = fn_val
        .dyn_into()
        .map_err(|_| JsValue::from_str("databases не функция"))?;

    let promise_val = func
        .call0(&factory)
        .map_err(|e| JsValue::from_str(&format!("databases: {e:?}")))?;
    let promise: js_sys::Promise = promise_val
        .dyn_into()
        .map_err(|_| JsValue::from_str("databases не вернул Promise"))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await databases: {e:?}")))?;
    let array: js_sys::Array = result
        .dyn_into()
        .map_err(|_| JsValue::from_str("databases вернул не Array"))?;

    let mut out = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let info = array.get(i);
        let name = js_sys::Reflect::get(&info, &JsValue::from_str("name"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default();
        let version = js_sys::Reflect::get(&info, &JsValue::from_str("version"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0) as u32;
        out.push(DbInfo { name, version });
    }
    Ok(out)
}

/// Сравнивает два ключа по правилам IndexedDB.
///
/// Возвращает -1, 0 или 1. Правило сортировки:
///   number < date < string < binary < array.
/// Внутри одного типа — по естественному порядку.
pub fn compare_keys(a: &JsValue, b: &JsValue) -> Result<i32, JsValue> {
    let factory = indexed_db()?;
    let cmp_fn = js_sys::Reflect::get(&factory, &JsValue::from_str("cmp"))
        .map_err(|e| JsValue::from_str(&format!("Reflect cmp: {e:?}")))?;
    let cmp: js_sys::Function = cmp_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("cmp не функция"))?;

    let result = cmp
        .call2(&factory, a, b)
        .map_err(|e| JsValue::from_str(&format!("cmp: {e:?}")))?;
    Ok(result.as_f64().unwrap_or(0.0) as i32)
}

/// Закрывает `IDBDatabase`. Без этого `deleteDatabase` заблокируется.
///
/// `IDBDatabase.close()` — единственный способ освободить соединение.
/// `web-sys` не имеет типизированной обёртки для `IDBDatabase`, поэтому
/// вызываем через Reflect.
pub fn close_db(db: &JsValue) {
    if let Ok(close_fn) = js_sys::Reflect::get(db, &JsValue::from_str("close")) {
        if let Ok(f) = close_fn.dyn_into::<js_sys::Function>() {
            let _ = f.call0(db);
        }
    }
}
```

## 24.2.4. Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod idb_factory;
```

## 24.2.5. Тесты — `tests/idb_factory_test.rs`

```rust
//! Wasm-тесты модуля `idb_factory`: открытие, удаление, список баз,
//! сравнение ключей.
//!
//! Компилируются только под wasm32 — на host файл пуст.
//!
//! ПРАВИЛО: тесты не пересекаются между запусками — каждое имя базы
//! содержит уникальный суффикс `uid()`. После каждого теста база
//! удаляется, соединение закрывается.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::idb_factory;

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64);
}

fn test_db_name() -> String {
    return format!("test-idb-{}", uid());
}

fn close_db(db: &JsValue) {
    idb_factory::close_db(db);
}

fn db_name(db: &JsValue) -> String {
    js_sys::Reflect::get(db, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default()
}

fn db_version(db: &JsValue) -> u32 {
    js_sys::Reflect::get(db, &JsValue::from_str("version"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u32
}

// ---------------- open_by_name / open_demo ----------------

#[wasm_bindgen_test]
async fn open_returns_object_with_name_and_version() {
    let name = test_db_name();
    let db = idb_factory::open_by_name(&name, 1).await.expect("open_by_name");

    assert_eq!(db_name(&db), name);
    assert_eq!(db_version(&db), 1);

    close_db(&db);
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn open_with_higher_version_bumps_schema() {
    let name = test_db_name();

    let db = idb_factory::open_by_name(&name, 1).await.expect("open v1");
    assert_eq!(db_version(&db), 1);
    close_db(&db);

    let db = idb_factory::open_by_name(&name, 2).await.expect("open v2");
    assert_eq!(db_version(&db), 2);
    close_db(&db);

    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn open_demo_creates_demo_database() {
    let _ = idb_factory::delete_demo().await;
    let db = idb_factory::open_demo().await.expect("open_demo");
    assert_eq!(db_name(&db), idb_factory::DEMO_DB);
    assert_eq!(db_version(&db), idb_factory::DEMO_VERSION);
    close_db(&db);
    idb_factory::delete_demo().await.expect("cleanup");
}

// ---------------- delete_by_name / delete_demo ----------------

#[wasm_bindgen_test]
async fn delete_by_name_removes_database() {
    let name = test_db_name();
    let db = idb_factory::open_by_name(&name, 1).await.expect("open");
    close_db(&db);

    let before = idb_factory::list_databases().await.expect("list");
    assert!(before.iter().any(|d| d.name == name));

    idb_factory::delete_by_name(&name).await.expect("delete");

    let after = idb_factory::list_databases().await.expect("list after");
    assert!(!after.iter().any(|d| d.name == name));
}

#[wasm_bindgen_test]
async fn delete_by_name_is_idempotent() {
    let name = test_db_name();
    idb_factory::delete_by_name(&name).await.expect("first delete");
    idb_factory::delete_by_name(&name).await.expect("second delete");
    idb_factory::delete_by_name(&name).await.expect("third delete");
}

#[wasm_bindgen_test]
async fn delete_demo_is_idempotent() {
    let _ = idb_factory::delete_demo().await;
    let _ = idb_factory::delete_demo().await;
    let _ = idb_factory::delete_demo().await;
}

// ---------------- list_databases ----------------

#[wasm_bindgen_test]
async fn list_databases_returns_array_with_opened() {
    let name = test_db_name();
    let db = idb_factory::open_by_name(&name, 1).await.expect("open");
    close_db(&db);

    let list = idb_factory::list_databases().await.expect("list");
    let found = list.iter().find(|d| d.name == name).expect("in list");
    assert_eq!(found.version, 1);

    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn list_databases_reports_version() {
    let name = test_db_name();
    let db = idb_factory::open_by_name(&name, 3).await.expect("open v3");
    close_db(&db);

    let list = idb_factory::list_databases().await.expect("list");
    let found = list.iter().find(|d| d.name == name).expect("in list");
    assert_eq!(found.version, 3);

    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn list_databases_does_not_contain_deleted() {
    let name = test_db_name();
    let db = idb_factory::open_by_name(&name, 1).await.expect("open");
    close_db(&db);
    idb_factory::delete_by_name(&name).await.expect("delete");

    let list = idb_factory::list_databases().await.expect("list");
    assert!(!list.iter().any(|d| d.name == name));
}

// ---------------- compare_keys: правило типов ----------------
//
// IndexedDB сортирует ключи по типу: number < date < string < binary < array.
// Внутри типа — естественный порядок.

#[wasm_bindgen_test]
fn compare_keys_numbers() {
    let a = JsValue::from_f64(1.0);
    let b = JsValue::from_f64(2.0);
    assert_eq!(idb_factory::compare_keys(&a, &b).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&b, &a).unwrap(), 1);
    assert_eq!(idb_factory::compare_keys(&a, &a).unwrap(), 0);
}

#[wasm_bindgen_test]
fn compare_keys_number_less_than_date() {
    // По правилам IndexedDB: любой number < любой date.
    let num = JsValue::from_f64(1e9);
    let date = js_sys::Date::new(&JsValue::from_f64(0.0));
    let date_val: JsValue = date.into();

    assert_eq!(idb_factory::compare_keys(&num, &date_val).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&date_val, &num).unwrap(), 1);
}

#[wasm_bindgen_test]
fn compare_keys_date_less_than_string() {
    let date = js_sys::Date::new(&JsValue::from_f64(0.0));
    let date_val: JsValue = date.into();
    let s = JsValue::from_str("");

    assert_eq!(idb_factory::compare_keys(&date_val, &s).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&s, &date_val).unwrap(), 1);
}

#[wasm_bindgen_test]
fn compare_keys_string_less_than_binary() {
    let s = JsValue::from_str("zzz");
    let buf = js_sys::ArrayBuffer::new(0);
    let buf_val: JsValue = buf.into();

    assert_eq!(idb_factory::compare_keys(&s, &buf_val).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&buf_val, &s).unwrap(), 1);
}

#[wasm_bindgen_test]
fn compare_keys_binary_less_than_array() {
    let buf = js_sys::ArrayBuffer::new(100);
    let buf_val: JsValue = buf.into();
    let arr = js_sys::Array::new();
    let arr_val: JsValue = arr.into();

    assert_eq!(idb_factory::compare_keys(&buf_val, &arr_val).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&arr_val, &buf_val).unwrap(), 1);
}

// ---------------- compare_keys: внутри одного типа ----------------

#[wasm_bindgen_test]
fn compare_keys_strings_lexicographic() {
    let a = JsValue::from_str("apple");
    let b = JsValue::from_str("banana");
    assert_eq!(idb_factory::compare_keys(&a, &b).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&b, &a).unwrap(), 1);

    let a2 = JsValue::from_str("apple");
    assert_eq!(idb_factory::compare_keys(&a, &a2).unwrap(), 0);
}

#[wasm_bindgen_test]
fn compare_keys_dates_chronological() {
    let early = js_sys::Date::new(&JsValue::from_f64(0.0));
    let late = js_sys::Date::new(&JsValue::from_f64(1_000_000.0));
    let early_val: JsValue = early.into();
    let late_val: JsValue = late.into();

    assert_eq!(idb_factory::compare_keys(&early_val, &late_val).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&late_val, &early_val).unwrap(), 1);
}

#[wasm_bindgen_test]
fn compare_keys_negative_numbers() {
    let a = JsValue::from_f64(-10.0);
    let b = JsValue::from_f64(5.0);
    assert_eq!(idb_factory::compare_keys(&a, &b).unwrap(), -1);
    assert_eq!(idb_factory::compare_keys(&b, &a).unwrap(), 1);
}

// ---------------- compare_keys: full chain ----------------

#[wasm_bindgen_test]
fn compare_keys_full_type_order() {
    let number = JsValue::from_f64(1.0);
    let date: JsValue = js_sys::Date::new(&JsValue::from_f64(0.0)).into();
    let string = JsValue::from_str("");
    let binary: JsValue = js_sys::ArrayBuffer::new(0).into();
    let array: JsValue = js_sys::Array::new().into();

    let chain = [&number, &date, &string, &binary, &array];

    for i in 0..chain.len() {
        for j in (i + 1)..chain.len() {
            assert_eq!(
                idb_factory::compare_keys(chain[i], chain[j]).unwrap(),
                -1,
                "элемент {i} должен быть меньше {j} по типу"
            );
            assert_eq!(
                idb_factory::compare_keys(chain[j], chain[i]).unwrap(),
                1,
                "элемент {j} должен быть больше {i} по типу"
            );
        }
    }
}
```

Итого — **18 тестов**, все wasm-only.

## 24.2.6. Команды запуска

```bash
# Проверка компиляции
cargo check

# Wasm-тесты модуля idb_factory
wasm-pack test --headless --chrome --test idb_factory_test

# Сборка wasm + запуск демо
npm run wasm:dev && npm run dev
```

`cargo test` для этого файла — no-op: он помечен `#![cfg(target_arch = "wasm32")]`.

## 24.2.7. Демо — `src/app.rs`

```
//! Демо Главы 24.2: IdbFactory — точка входа в IndexedDB.
//!
//! Три эксперимента — один вывод:
//!   1. Lifecycle: open v1 → v2 → v1 (VersionError) → delete → verify → delete.
//!   2. cmp(a, b): правило сортировки ключей — number < date < string < binary < array.
//!   3. Blocked: два соединения с одной базой → delete резолвится onblocked.
//!
//! ПРАВИЛО: `format!` в tail-позиции — через `format_str(format_args!(...))`.
//! ПРАВИЛО: `JsCast` нужен для `.dyn_into` / `.dyn_ref` — импортируется явно.
//! ПРАВИЛО: `Closure` живёт в `wasm_bindgen::closure::Closure` — не в prelude.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;
use web_sys::{HtmlButtonElement, HtmlProgressElement};

use crate::format_str::format_str;
use crate::idb_factory;

const HTML: &str = r#"
<style>
  body { font-family: system-ui, -apple-system, sans-serif; }
  .panel {
    border: 1px solid #e0e0e0; border-radius: 8px; padding: 16px;
    margin: 14px 0; background: #ffffff;
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .panel-title { font-size: 15px; font-weight: 600; margin: 0 0 6px; color: #222; }
  .hint { font-size: 12px; color: #666; line-height: 1.5; margin: 0 0 10px; }
  .hint code { background: #f2f2f2; padding: 1px 5px; border-radius: 3px; font-size: 11px; }
  .mono { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 12px; }
  .btn-row { display: flex; gap: 8px; flex-wrap: wrap; margin: 8px 0; }
  button {
    padding: 7px 14px; font-size: 13px;
    border: 1px solid #c0c0c0; border-radius: 5px; background: #fafafa;
    cursor: pointer;
  }
  button:hover:not(:disabled) { background: #f0f0f0; }
  button:disabled { opacity: 0.5; cursor: not-allowed; }
  .status {
    padding: 8px 12px; background: #f5f5f5; border-left: 3px solid #888;
    font-family: ui-monospace, monospace; font-size: 12px;
    margin-top: 12px; border-radius: 4px;
  }
  .progress-row { display: flex; align-items: center; gap: 10px; margin: 8px 0; }
  progress { width: 260px; height: 14px; }

  /* ===== Timeline ===== */
  .timeline {
    display: flex; align-items: flex-start; flex-wrap: wrap;
    gap: 4px; padding: 14px 8px;
    background: #fafafa; border-radius: 6px; margin: 10px 0;
  }
  .timeline-empty {
    padding: 20px; text-align: center; color: #999; font-size: 12px;
    background: #fafafa; border-radius: 6px; margin: 10px 0;
  }
  .tl-step { display: flex; flex-direction: column; align-items: center; min-width: 86px; }
  .tl-circle {
    width: 34px; height: 34px; border-radius: 50%;
    display: flex; align-items: center; justify-content: center;
    font-weight: 700; font-size: 14px; color: #fff;
    background: #ccc; transition: background 0.2s;
  }
  .tl-step[data-state="running"] .tl-circle { background: #3498db; animation: pulse 1s infinite; }
  .tl-step[data-state="ok"] .tl-circle { background: #27ae60; }
  .tl-step[data-state="err"] .tl-circle { background: #c0392b; }
  .tl-step[data-state="blocked"] .tl-circle { background: #e67e22; }
  @keyframes pulse { 0%,100% { opacity: 1; } 50% { opacity: 0.6; } }
  .tl-label {
    font-family: ui-monospace, monospace; font-size: 10px;
    color: #333; margin-top: 6px; text-align: center; line-height: 1.3;
    max-width: 100px;
  }
  .tl-time {
    font-family: ui-monospace, monospace; font-size: 10px;
    color: #888; margin-top: 3px;
  }
  .tl-arrow {
    align-self: flex-start; padding-top: 8px; color: #bbb; font-size: 18px;
  }

  /* ===== Summary ===== */
  .summary {
    display: flex; gap: 20px; padding: 10px 14px;
    background: #eef7ff; border-left: 4px solid #3498db;
    border-radius: 5px; font-family: ui-monospace, monospace; font-size: 12px;
    margin: 10px 0;
  }
  .summary b { font-size: 14px; }
  .summary-ok { color: #27ae60; }
  .summary-err { color: #c0392b; }
  .summary-blocked { color: #e67e22; }

  /* ===== steps table ===== */
  .steps-table {
    border-collapse: collapse; font-family: ui-monospace, monospace;
    font-size: 11px; margin-top: 12px; width: 100%;
  }
  .steps-table th {
    text-align: left; padding: 6px 10px;
    background: #f0f0f0; color: #555; font-weight: 600;
    border-bottom: 2px solid #d0d0d0; font-size: 10px;
    letter-spacing: 0.5px;
  }
  .steps-table th.r { text-align: right; }
  .steps-table td {
    padding: 6px 10px; border-bottom: 1px solid #eee;
    vertical-align: top;
  }
  .steps-table td.r { text-align: right; color: #888; white-space: nowrap; }
  .steps-table td.status { font-weight: 700; padding-right: 4px; width: 70px; }
  .steps-table td.action { min-width: 180px; }

  /* ===== cmp axis ===== */
  .cmp-axis {
    display: flex; align-items: stretch;
    border-radius: 8px; overflow: hidden;
    box-shadow: 0 2px 8px rgba(0,0,0,0.08); margin: 14px 0;
    min-height: 72px;
  }
  .axis-segment {
    flex: 1; padding: 10px 6px;
    display: flex; flex-direction: column; align-items: center;
    justify-content: center; color: #fff;
    font-family: ui-monospace, monospace; font-size: 11px;
    position: relative;
  }
  .axis-segment .pos {
    position: absolute; top: 4px; left: 6px;
    background: rgba(0,0,0,0.25); width: 18px; height: 18px;
    border-radius: 50%; display: flex; align-items: center;
    justify-content: center; font-size: 10px; font-weight: 700;
  }
  .axis-type { font-weight: 700; font-size: 12px; text-transform: uppercase; letter-spacing: 0.5px; }
  .axis-value { opacity: 0.9; margin-top: 2px; font-size: 10px; }
  .axis-num    { background: linear-gradient(135deg,#3498db,#2471a3); }
  .axis-date   { background: linear-gradient(135deg,#9b59b6,#7d3c98); }
  .axis-string { background: linear-gradient(135deg,#e67e22,#ca6f1e); }
  .axis-binary { background: linear-gradient(135deg,#16a085,#117a65); }
  .axis-array  { background: linear-gradient(135deg,#c0392b,#922b21); }
  .axis-lt {
    display: flex; align-items: center; justify-content: center;
    width: 22px; background: #2c3e50; color: #fff;
    font-size: 16px; font-weight: 700;
  }

  /* ===== surprising pairs ===== */
  .surprising {
    background: #fff9e6; border-left: 4px solid #f39c12;
    padding: 12px 16px; border-radius: 6px; margin: 14px 0;
  }
  .surprising h4 { margin: 0 0 8px; font-size: 13px; color: #7d5a00; }
  .s-pair {
    display: flex; align-items: center; gap: 8px;
    font-family: ui-monospace, monospace; font-size: 12px;
    padding: 5px 0; color: #444;
  }
  .s-lt { color: #c0392b; font-weight: 700; font-size: 16px; }
  .s-badge {
    display: inline-block; padding: 1px 6px; border-radius: 3px;
    font-size: 10px; background: #c0392b; color: #fff;
    margin-left: 6px; font-weight: 600;
  }
  .s-note { color: #888; font-size: 11px; margin-left: auto; text-align: right; }

  /* ===== cmp matrix ===== */
  details { margin-top: 12px; }
  details summary { cursor: pointer; font-size: 12px; color: #666; padding: 4px 0; }
  .matrix {
    border-collapse: collapse; font-family: ui-monospace, monospace;
    font-size: 10px; margin: 10px 0;
  }
  .matrix th, .matrix td {
    padding: 5px 8px; border: 1px solid #e0e0e0; text-align: center;
  }
  .matrix th { background: #f0f0f0; font-weight: 700; }
  .matrix td.diag { background: #f6f6f6; color: #ccc; }
  .matrix td.m1 { background: #e8f8ed; color: #1e8449; font-weight: 700; }
  .matrix td.p1 { background: #fdedec; color: #b03a2e; font-weight: 700; }
  .legend { font-size: 11px; color: #666; margin: 6px 0; }
  .legend span { display: inline-block; padding: 1px 6px; border-radius: 3px; margin-right: 8px; }
  .legend .lg-m1 { background: #e8f8ed; color: #1e8449; }
  .legend .lg-p1 { background: #fdedec; color: #b03a2e; }

  /* ===== blocked snapshots ===== */
  .snapshots {
    display: flex; gap: 12px; align-items: stretch;
    flex-wrap: wrap;
    padding: 16px; background: #fafafa; border-radius: 8px;
    margin: 12px 0;
  }
  .snapshot {
    flex: 1 1 200px; min-width: 200px; max-width: 100%;
    border: 1px solid #e0e0e0; border-radius: 6px;
    padding: 12px; background: #fff;
    display: flex; flex-direction: column; gap: 8px;
    box-sizing: border-box;
  }
  .snapshot-title {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #555; font-weight: 700;
    letter-spacing: 0.3px; padding-bottom: 6px;
    border-bottom: 1px dashed #ddd;
    white-space: nowrap;
  }
  .snap-row {
    display: flex; align-items: center; gap: 8px;
    flex-wrap: wrap;
  }
  .conn-mini {
    padding: 6px 10px; border-radius: 6px;
    font-family: ui-monospace, monospace; font-size: 11px;
    text-align: center; flex-shrink: 0;
    white-space: nowrap;
  }
  .conn-mini.open { background: #e8f8ed; border: 2px solid #27ae60; color: #1e8449; }
  .conn-mini.closed { background: #f5f5f5; border: 2px dashed #aaa; color: #777; }
  .db-mini {
    padding: 6px 10px; border-radius: 6px;
    font-family: ui-monospace, monospace; font-size: 11px;
    text-align: center; flex-shrink: 0;
    white-space: nowrap;
  }
  .db-mini.present { background: #eef4ff; border: 2px solid #3498db; color: #1f4e79; }
  .db-mini.deleted { background: #f5f5f5; border: 2px dashed #aaa; color: #777; text-decoration: line-through; }
  .snap-arrow { font-size: 16px; font-weight: 700; flex-shrink: 0; }
  .snap-arrow.ok { color: #27ae60; }
  .snap-arrow.blocked { color: #e67e22; }
  .snap-arrow.none { color: #bbb; }
  .snap-caption {
    font-family: ui-monospace, monospace; font-size: 10px;
    color: #666; line-height: 1.4;
  }
</style>

<h1>Глава 24.2: IdbFactory — точка входа</h1>
<p style="color:#555;margin-bottom:14px">Версии, инспектор баз, сортировка ключей. Три эксперимента.</p>

<div class="panel">
  <div class="panel-title">Базы в этом origin</div>
  <div id="db-info" class="mono" style="padding:8px;background:#fafafa;border-radius:5px">загрузка…</div>
  <p class="hint" style="margin:8px 0 0">
    <code>hello-wasm-idb-factory-demo</code> — demo-база этой главы.
    Другие базы (<code>offline-notes-db</code>, <code>hello-wasm-library</code> и т. п.)
    принадлежат другим главам книги — они здесь же, потому что IndexedDB привязана
    к origin, а не к странице.
  </p>
  <div class="btn-row">
    <button id="refresh-list">↻ Обновить</button>
    <button id="cleanup-all">Удалить demo-базы этой книги</button>
  </div>
</div>

<div class="panel">
  <div class="panel-title">Эксперимент 1. Жизненный цикл базы</div>
  <p class="hint">
    Шесть шагов с одной кнопкой: создать v1 → обновить до v2 →
    попытаться открыть v1 (<code>VersionError</code>) → удалить → проверить
    отсутствие → удалить второй раз (идемпотентно). Таймлайн показывает
    статус каждого шага. В конце demo-база создаётся заново в v1 —
    чтобы её было видно в списке.
  </p>
  <p class="hint" style="margin-top:-4px">
    Колонки <b>«от старта»</b> и <b>«длительность»</b> не смыкаются между
    шагами: пауза — это перерисовка списка баз в DOM. В сумму Σ входят
    только длительности шагов, поэтому Σ меньше времени последнего шага.
  </p>
  <div class="btn-row">
    <button id="run-lifecycle">▶ Прогнать lifecycle</button>
    <button id="copy-result" disabled>⎘ Скопировать lifecycle</button>
  </div>
  <div class="progress-row">
    <progress id="lifecycle-progress" value="0" max="6"></progress>
    <span id="lifecycle-progress-text" class="mono">0 / 6</span>
  </div>
  <div id="lifecycle-timeline" class="timeline-empty">Таймлайн появится после запуска</div>
  <div id="lifecycle-summary" class="summary" style="display:none"></div>
  <div id="lifecycle-result"></div>
</div>

<div class="panel">
  <div class="panel-title">Эксперимент 2. cmp — правило сортировки ключей</div>
  <p class="hint">
    IndexedDB сортирует ключи по <b>типу</b>, затем по значению:
    <code>number &lt; date &lt; string &lt; binary &lt; array</code>.
    Тип доминирует над значением: даже очень большое число меньше самой
    старой даты.
  </p>
  <button id="run-cmp">Показать порядок</button>
  <div id="cmp-result"></div>
</div>

<div class="panel">
  <div class="panel-title">Эксперимент 3. Блокировка delete открытым соединением</div>
  <p class="hint">
    Если соединение с базой остаётся открытым, а второй запрос пытается
    удалить базу — срабатывает событие <code>blocked</code>. Без его
    обработки <code>deleteDatabase</code> никогда не разрешится.
    Снимки показывают состояние соединения и базы на каждом шаге.
  </p>
  <button id="run-blocked">Проверить blocked</button>
  <div id="blocked-viz" class="blocked-viz" style="display:none"></div>
  <div id="blocked-summary" class="summary" style="display:none"></div>
  <div id="blocked-result"></div>
</div>

<div id="status" class="status">Готово</div>
"#;

/// Один шаг сценария.
enum LifecycleStep {
    Open { label: String, name: String, version: u32 },
    Delete { label: String, name: String },
    VerifyAbsent { label: String, name: String },
}

/// Итог одного шага.
#[derive(Clone, Copy, PartialEq)]
enum StepOutcome {
    Ok,
    Err,
    /// Шаг «ожидаемо провалился» — например, blocked. Отличается от Err.
    Blocked,
}

impl StepOutcome {
    fn symbol(&self) -> &'static str {
        match self {
            StepOutcome::Ok => "✓",
            StepOutcome::Err => "✗",
            StepOutcome::Blocked => "⊘",
        }
    }
    fn color(&self) -> &'static str {
        match self {
            StepOutcome::Ok => "#1e8449",
            StepOutcome::Err => "#b03a2e",
            StepOutcome::Blocked => "#e67e22",
        }
    }
    fn label(&self) -> &'static str {
        match self {
            StepOutcome::Ok => "OK",
            StepOutcome::Err => "ERR",
            StepOutcome::Blocked => "BLOCKED",
        }
    }
}

/// Один выполненный шаг.
struct Step {
    action: String,
    result: String,
    outcome: StepOutcome,
    duration_ms: f64,
    started_at_ms: f64,
}

/// Сводка по сценарию.
///
/// `total_ms` — сумма длительностей шагов, а не wall-clock сценария.
/// Так Σ сходится с колонкой «длительность» в таблице.
struct LifecycleSummary {
    ok_count: u32,
    err_count: u32,
    blocked_count: u32,
    total_ms: f64,
}

impl LifecycleSummary {
    fn from_steps(
        steps: &[Step],
        ok_count: u32,
        err_count: u32,
        blocked_count: u32,
    ) -> Self {
        let total_ms: f64 = steps.iter().map(|s| s.duration_ms).sum();
        Self {
            ok_count,
            err_count,
            blocked_count,
            total_ms,
        }
    }
}

/// UI-хендлы для сценария lifecycle.
#[derive(Clone)]
struct LifecycleUi {
    timeline: web_sys::Element,
    summary: web_sys::Element,
    result: web_sys::Element,
    progress: HtmlProgressElement,
    progress_text: web_sys::Element,
    status: web_sys::Element,
    db_info: web_sys::Element,
    last_touched: Rc<RefCell<Option<String>>>,
    /// Готовый plain-text отчёт — для кнопки «Скопировать».
    /// Заполняется в конце сценария, потому что `text_content()` теряет
    /// разделители между ячейками таблицы.
    last_text: Rc<RefCell<String>>,
}

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let db_info = get_el("db-info")?;
    let status = get_el("status")?;
    let cmp_result = get_el("cmp-result")?;
    let blocked_viz = get_el("blocked-viz")?;
    let blocked_summary = get_el("blocked-summary")?;
    let blocked_result = get_el("blocked-result")?;

    let progress: HtmlProgressElement = document
        .get_element_by_id("lifecycle-progress")
        .ok_or("no progress")?
        .dyn_into()?;

    let lifecycle_btn = get_button("run-lifecycle")?;
    let cmp_btn = get_button("run-cmp")?;
    let blocked_btn = get_button("run-blocked")?;
    let refresh_btn = get_button("refresh-list")?;
    let cleanup_btn = get_button("cleanup-all")?;
    let copy_btn = get_button("copy-result")?;

    let all_buttons: Rc<Vec<HtmlButtonElement>> = Rc::new(vec![
        lifecycle_btn.clone(),
        cmp_btn.clone(),
        blocked_btn.clone(),
        refresh_btn.clone(),
        cleanup_btn.clone(),
    ]);

    let last_touched: Rc<RefCell<Option<String>>> = Rc::new(RefCell::new(None));
    let held_connection: Rc<RefCell<Option<JsValue>>> = Rc::new(RefCell::new(None));
    let last_lifecycle_text: Rc<RefCell<String>> = Rc::new(RefCell::new(String::new()));

    // Старт: чистое состояние + создать demo v1, чтобы она была видна
    // в списке сразу при загрузке.
    spawn_local({
        let di = db_info.clone();
        let lt = last_touched.clone();
        async move {
            let _ = idb_factory::delete_demo().await;
            if let Ok(db) = idb_factory::open_by_name(idb_factory::DEMO_DB, 1).await {
                idb_factory::close_db(&db);
            }
            refresh_db_list(&di, &lt).await;
        }
    });

    // ---------------- Обновить список ----------------
    {
        let di = db_info.clone();
        let st = status.clone();
        let lt = last_touched.clone();
        set_click(&refresh_btn, move |_| {
            let di_in = di.clone();
            let st_in = st.clone();
            let lt_in = lt.clone();
            spawn_local(async move {
                refresh_db_list(&di_in, &lt_in).await;
                set_status(&st_in, "Список обновлён");
            });
        });
    }

    // ---------------- Удалить demo-базы ----------------
    {
        let di = db_info.clone();
        let st = status.clone();
        let lt = last_touched.clone();
        let held = held_connection.clone();
        set_click(&cleanup_btn, move |_| {
            let di_in = di.clone();
            let st_in = st.clone();
            let lt_in = lt.clone();
            let held_in = held.clone();
            spawn_local(async move {
                // Сначала освобождаем удержанное соединение — иначе
                // delete может зависнуть на onblocked.
                close_held(&held_in);
                match idb_factory::list_databases().await {
                    Ok(list) => {
                        let targets: Vec<String> = list
                            .iter()
                            .filter(|d| d.name.starts_with("hello-wasm-"))
                            .map(|d| d.name.clone())
                            .collect();
                        if targets.is_empty() {
                            set_status(&st_in, "Demo-баз нет — нечего удалять");
                            return;
                        }
                        let mut removed: Vec<String> = Vec::new();
                        let mut failed: Vec<String> = Vec::new();
                        for name in &targets {
                            match idb_factory::delete_by_name(name).await {
                                Ok(_) => removed.push(name.clone()),
                                Err(_) => failed.push(name.clone()),
                            }
                        }
                        *lt_in.borrow_mut() = None;
                        refresh_db_list(&di_in, &lt_in).await;
                        let msg: String = if failed.is_empty() {
                            format_str(format_args!(
                                "Удалено demo-баз: {} — {}",
                                removed.len(),
                                removed.join(", "),
                            ))
                        } else {
                            format_str(format_args!(
                                "Удалено: {} | Не удалось: {}",
                                removed.join(", "),
                                failed.join(", "),
                            ))
                        };
                        set_status(&st_in, &msg);
                    }
                    Err(e) => set_status(
                        &st_in,
                        &format_str(format_args!("Ошибка списка: {}", error_name(&e))),
                    ),
                }
            });
        });
    }

    // ---------------- Lifecycle ----------------
    {
        let lr = get_el("lifecycle-result")?;
        let ls = get_el("lifecycle-summary")?;
        let tl = get_el("lifecycle-timeline")?;
        let pt = get_el("lifecycle-progress-text")?;
        let di = db_info.clone();
        let st = status.clone();
        let lt = last_touched.clone();
        let pr = progress.clone();
        let copy = copy_btn.clone();
        let buttons = all_buttons.clone();
        let last_text = last_lifecycle_text.clone();

        set_click(&lifecycle_btn, move |_| {
            disable_all(&buttons, true);
            copy.set_disabled(true);
            set_progress(&pr, &pt, 0, 6);
            lr.set_inner_html("");
            ls.set_inner_html("");
            let _ = ls.set_attribute("style", "display:none");
            *last_text.borrow_mut() = String::new();

            let ui = LifecycleUi {
                timeline: tl.clone(),
                summary: ls.clone(),
                result: lr.clone(),
                progress: pr.clone(),
                progress_text: pt.clone(),
                status: st.clone(),
                db_info: di.clone(),
                last_touched: lt.clone(),
                last_text: last_text.clone(),
            };
            let copy_in = copy.clone();
            let buttons_in = buttons.clone();

            spawn_local(async move {
                let st_final = ui.status.clone();
                let summary = lifecycle_scenario(ui).await;
                set_status(
                    &st_final,
                    &format_str(format_args!(
                        "Lifecycle завершён: {} OK, {} ERR, Σ {:.0} мс",
                        summary.ok_count, summary.err_count, summary.total_ms,
                    )),
                );
                disable_all(&buttons_in, false);
                copy_in.set_disabled(false);
            });
        });
    }

    // ---------------- Копировать результат ----------------
    {
        let st = status.clone();
        let last_text = last_lifecycle_text.clone();
        set_click(&copy_btn, move |_| {
            let text = last_text.borrow().clone();
            if text.is_empty() {
                set_status(&st, "Сначала прогоните lifecycle");
                return;
            }
            let st_in = st.clone();
            spawn_local(async move {
                match copy_to_clipboard(&text).await {
                    Ok(()) => set_status(&st_in, "Результат скопирован в буфер обмена"),
                    Err(e) => set_status(
                        &st_in,
                        &format_str(format_args!("Копирование: {}", error_name(&e))),
                    ),
                }
            });
        });
    }

    // ---------------- cmp ----------------
    {
        let cr = cmp_result.clone();
        let st = status.clone();
        set_click(&cmp_btn, move |_| {
            render_cmp(&cr);
            set_status(&st, "Правило сортировки показано");
        });
    }

    // ---------------- blocked ----------------
    {
        let br = blocked_result.clone();
        let bv = blocked_viz.clone();
        let bs = blocked_summary.clone();
        let st = status.clone();
        let held = held_connection.clone();
        let di = db_info.clone();
        let lt = last_touched.clone();
        let buttons = all_buttons.clone();
        set_click(&blocked_btn, move |_| {
            disable_all(&buttons, true);
            let br_in = br.clone();
            let bv_in = bv.clone();
            let bs_in = bs.clone();
            let st_in = st.clone();
            let held_in = held.clone();
            let di_in = di.clone();
            let lt_in = lt.clone();
            let buttons_in = buttons.clone();
            spawn_local(async move {
                let summary =
                    blocked_scenario(&st_in, &bv_in, &br_in, &held_in, &di_in, &lt_in).await;
                render_summary(&bs_in, &summary);
                refresh_db_list(&di_in, &lt_in).await;
                set_status(&st_in, "Эксперимент с blocked завершён");
                disable_all(&buttons_in, false);
            });
        });
    }

    Ok(())
}

// ---------------- сценарий lifecycle ----------------

async fn lifecycle_scenario(ui: LifecycleUi) -> LifecycleSummary {
    let name = idb_factory::DEMO_DB;
    let specs = vec![
        LifecycleStep::Open { label: "open v1".into(), name: name.into(), version: 1 },
        LifecycleStep::Open { label: "open v2".into(), name: name.into(), version: 2 },
        LifecycleStep::Open { label: "open v1".into(), name: name.into(), version: 1 },
        LifecycleStep::Delete { label: "delete".into(), name: name.into() },
        LifecycleStep::VerifyAbsent { label: "verify".into(), name: name.into() },
        LifecycleStep::Delete { label: "delete ×2".into(), name: name.into() },
    ];

    let total = specs.len() as u32;
    let scenario_start = now_ms();
    let mut steps: Vec<Step> = Vec::new();
    let mut ok_count = 0u32;
    let mut err_count = 0u32;
    let mut blocked_count = 0u32;

    render_timeline(&ui.timeline, &specs, &steps, None);

    for (i, spec) in specs.iter().enumerate() {
        let step_num = (i + 1) as u32;
        let label = step_label(spec);
        set_status(
            &ui.status,
            &format_str(format_args!("Lifecycle: шаг {step_num}/{total} — {label}")),
        );
        set_progress(&ui.progress, &ui.progress_text, i as u32, total);
        render_timeline(&ui.timeline, &specs, &steps, Some(i));

        let started_at = now_ms() - scenario_start;
        let t0 = now_ms();
        let (outcome, result) = run_step(spec).await;
        let duration_ms = now_ms() - t0;

        match outcome {
            StepOutcome::Ok => ok_count += 1,
            StepOutcome::Err => err_count += 1,
            StepOutcome::Blocked => blocked_count += 1,
        }

        steps.push(Step {
            action: label,
            result,
            outcome,
            duration_ms,
            started_at_ms: started_at,
        });

        render_timeline(&ui.timeline, &specs, &steps, None);
        refresh_db_list(&ui.db_info, &ui.last_touched).await;
    }

    // После всех шагов — recreate demo v1, чтобы пользователь видел
    // demo-базу в списке рядом с базами других глав.
    if let Ok(db) = idb_factory::open_by_name(name, 1).await {
        idb_factory::close_db(&db);
    }
    refresh_db_list(&ui.db_info, &ui.last_touched).await;

    set_progress(&ui.progress, &ui.progress_text, total, total);
    render_steps_table(&ui.result, &steps);

    // Готовим plain-text отчёт для кнопки «Скопировать».
    *ui.last_text.borrow_mut() = build_report(&steps);

    // Σ = сумма длительностей шагов, а не wall-clock сценария.
    let summary = LifecycleSummary::from_steps(&steps, ok_count, err_count, blocked_count);
    render_summary(&ui.summary, &summary);
    summary
}

/// Строит plain-text отчёт по шагам — для буфера обмена.
fn build_report(steps: &[Step]) -> String {
    let mut s = String::with_capacity(512);
    s.push_str("# Глава 24.2 · IdbFactory lifecycle\n\n");
    s.push_str("статус\tшаг\tрезультат\tот старта\tдлительность\n");
    for st in steps {
        let started_label: String = if st.started_at_ms < 0.5 {
            "—".to_string()
        } else {
            format_str(format_args!("+{:.0} мс", st.started_at_ms))
        };
        s.push_str(&format_str(format_args!(
            "{}\t{}\t{}\t{}\t{:.0} мс\n",
            st.outcome.label(),
            st.action,
            st.result,
            started_label,
            st.duration_ms,
        )));
    }
    return s;
}

fn step_label(spec: &LifecycleStep) -> String {
    match spec {
        LifecycleStep::Open { label, .. } => label.clone(),
        LifecycleStep::Delete { label, .. } => label.clone(),
        LifecycleStep::VerifyAbsent { label, .. } => label.clone(),
    }
}

async fn run_step(spec: &LifecycleStep) -> (StepOutcome, String) {
    match spec {
        LifecycleStep::Open { name, version, .. } => {
            match idb_factory::open_by_name(name, *version).await {
                Ok(db) => {
                    let v = db_version(&db);
                    idb_factory::close_db(&db);
                    (StepOutcome::Ok, format_str(format_args!("OK, version={v}")))
                }
                Err(e) => (StepOutcome::Err, describe_idb_error(&e)),
            }
        }
        LifecycleStep::Delete { name, .. } => match idb_factory::delete_by_name(name).await {
            Ok(_) => (StepOutcome::Ok, "OK".into()),
            Err(e) => (StepOutcome::Err, describe_idb_error(&e)),
        },
        LifecycleStep::VerifyAbsent { name, .. } => match idb_factory::list_databases().await {
            Ok(list) => {
                if list.iter().any(|d| d.name == *name) {
                    (StepOutcome::Err, format_str(format_args!("FAIL: {name} в списке")))
                } else {
                    (StepOutcome::Ok, "OK: отсутствует".into())
                }
            }
            Err(e) => (StepOutcome::Err, describe_idb_error(&e)),
        },
    }
}

// ---------------- timeline ----------------

fn render_timeline(
    el: &web_sys::Element,
    specs: &[LifecycleStep],
    completed: &[Step],
    running_idx: Option<usize>,
) {
    let mut html = String::with_capacity(2048);
    html.push_str("<div class=\"timeline\">");
    for (i, spec) in specs.iter().enumerate() {
        let (state, dur_text) = if i < completed.len() {
            let s = &completed[i];
            let st = match s.outcome {
                StepOutcome::Ok => "ok",
                StepOutcome::Err => "err",
                StepOutcome::Blocked => "blocked",
            };
            (st, format_str(format_args!("{:.0} мс", s.duration_ms)))
        } else if Some(i) == running_idx {
            ("running", "…".to_string())
        } else {
            ("pending", String::new())
        };

        let icon: String = match state {
            "ok" => "✓".to_string(),
            "err" => "✗".to_string(),
            "blocked" => "⊘".to_string(),
            "running" => "…".to_string(),
            _ => (i + 1).to_string(),
        };

        html.push_str(&format_str(format_args!(
            "<div class=\"tl-step\" data-state=\"{state}\">\
             <div class=\"tl-circle\">{icon}</div>\
             <div class=\"tl-label\">{}</div>\
             <div class=\"tl-time\">{dur_text}</div>\
             </div>",
            esc(&step_label(spec)),
        )));

        if i + 1 < specs.len() {
            html.push_str("<div class=\"tl-arrow\">→</div>");
        }
    }
    html.push_str("</div>");
    el.set_inner_html(&html);
}

/// Рендерит таблицу шагов с заголовками колонок.
fn render_steps_table(el: &web_sys::Element, steps: &[Step]) {
    if steps.is_empty() {
        el.set_inner_html("");
        return;
    }
    let mut html = String::with_capacity(1024);
    html.push_str("<table class=\"steps-table\">");
    html.push_str(
        "<thead><tr>\
         <th>статус</th>\
         <th>шаг</th>\
         <th>результат</th>\
         <th class=\"r\">от старта</th>\
         <th class=\"r\">длительность</th>\
         </tr></thead><tbody>",
    );
    for s in steps {
        // Первый шаг стартует на t=0 — «+0 мс» выглядит странно.
        let started_label: String = if s.started_at_ms < 0.5 {
            "—".to_string()
        } else {
            format_str(format_args!("+{:.0} мс", s.started_at_ms))
        };
        html.push_str(&format_str(format_args!(
            "<tr>\
             <td class=\"status\" style=\"color:{}\">{}</td>\
             <td class=\"action\">{}</td>\
             <td style=\"color:{}\">{}</td>\
             <td class=\"r\">{started_label}</td>\
             <td class=\"r\">{:.0} мс</td>\
             </tr>",
            s.outcome.color(),
            s.outcome.symbol(),
            esc(&s.action),
            s.outcome.color(),
            esc(&s.result),
            s.duration_ms,
        )));
    }
    html.push_str("</tbody></table>");
    el.set_inner_html(&html);
}

fn render_summary(el: &web_sys::Element, s: &LifecycleSummary) {
    let mut html = String::with_capacity(256);
    if s.ok_count > 0 {
        html.push_str(&format_str(format_args!(
            "<span class=\"summary-ok\">✓ <b>{}</b> OK</span>",
            s.ok_count
        )));
    }
    if s.err_count > 0 {
        html.push_str(&format_str(format_args!(
            "<span class=\"summary-err\">✗ <b>{}</b> ERR</span>",
            s.err_count
        )));
    }
    if s.blocked_count > 0 {
        html.push_str(&format_str(format_args!(
            "<span class=\"summary-blocked\">⊘ <b>{}</b> BLOCKED</span>",
            s.blocked_count
        )));
    }
    html.push_str(&format_str(format_args!(
        "<span>Σ <b>{:.0}</b> мс</span>",
        s.total_ms
    )));
    el.set_inner_html(&html);
    let _ = el.set_attribute("style", "display:flex");
}

// ---------------- cmp visual ----------------

fn render_cmp(el: &web_sys::Element) {
    let number = JsValue::from_f64(42.0);
    let big_number = JsValue::from_f64(999_999_999.0);
    let date: JsValue = js_sys::Date::new(&JsValue::from_f64(1_700_000_000_000.0)).into();
    let old_date: JsValue = js_sys::Date::new(&JsValue::from_f64(0.0)).into();
    let string = JsValue::from_str("hello");
    let string_z = JsValue::from_str("zzz");
    let binary: JsValue = js_sys::ArrayBuffer::new(8).into();
    let empty_binary: JsValue = js_sys::ArrayBuffer::new(0).into();
    let array: JsValue = {
        let a = js_sys::Array::new();
        a.push(&JsValue::from_f64(1.0));
        a.into()
    };

    let mut html = String::with_capacity(4096);

    html.push_str("<p style=\"font-size:13px;font-weight:600;margin:14px 0 4px\">Порядок возрастания: тип важнее значения</p>");
    html.push_str("<div class=\"cmp-axis\">");

    let segments: [(&str, &str, &str); 5] = [
        ("axis-num", "number", "42"),
        ("axis-date", "date", "2023-11-14"),
        ("axis-string", "string", "\"hello\""),
        ("axis-binary", "binary", "ArrayBuffer(8)"),
        ("axis-array", "array", "[1]"),
    ];
    for (i, (cls, ty, val)) in segments.iter().enumerate() {
        html.push_str(&format_str(format_args!(
            "<div class=\"axis-segment {cls}\">\
             <div class=\"pos\">{}</div>\
             <div class=\"axis-type\">{ty}</div>\
             <div class=\"axis-value\">{val}</div>\
             </div>",
            i + 1,
        )));
        if i + 1 < segments.len() {
            html.push_str("<div class=\"axis-lt\">&lt;</div>");
        }
    }
    html.push_str("</div>");

    html.push_str("<div class=\"surprising\">");
    html.push_str("<h4>⚡ Контринтуитивные пары — тип доминирует над значением</h4>");

    let pairs: [(&JsValue, &JsValue, &str, &str, &str); 3] = [
        (
            &big_number,
            &old_date,
            "999 999 999",
            "Date 1970-01-01",
            "число &lt; дата, даже если число огромное",
        ),
        (
            &string_z,
            &empty_binary,
            "\"zzz\"",
            "ArrayBuffer(0)",
            "строка &lt; binary, даже если строка длинная",
        ),
        (
            &empty_binary,
            &array,
            "ArrayBuffer(0)",
            "[]",
            "binary &lt; array — оба пустых, но тип решает",
        ),
    ];
    for (a, b, la, lb, note) in pairs {
        let cmp = idb_factory::compare_keys(a, b).unwrap_or(0);
        let sign = if cmp < 0 { "&lt;" } else if cmp > 0 { "&gt;" } else { "=" };
        let badge = if cmp < 0 {
            "<span class=\"s-badge\">тип важнее</span>"
        } else {
            ""
        };
        html.push_str(&format_str(format_args!(
            "<div class=\"s-pair\">\
             <code>{la}</code>\
             <span class=\"s-lt\">{sign}</span>\
             <code>{lb}</code>{badge}\
             <span class=\"s-note\">{note}</span>\
             </div>"
        )));
    }
    html.push_str("</div>");

    html.push_str("<details><summary>▸ Все 10 пар — матрица сравнений</summary>");
    html.push_str("<p class=\"legend\">\
        <span class=\"lg-m1\">-1</span> строка &lt; столбец · \
        <span class=\"lg-p1\">+1</span> строка &gt; столбец</p>");
    html.push_str("<table class=\"matrix\">");
    html.push_str("<thead><tr><th></th>");
    for (_, ty, _) in &segments {
        html.push_str(&format_str(format_args!("<th>{ty}</th>")));
    }
    html.push_str("</tr></thead><tbody>");

    let keys: [&JsValue; 5] = [&number, &date, &string, &binary, &array];
    let types: [&str; 5] = ["number", "date", "string", "binary", "array"];
    for i in 0..5 {
        html.push_str(&format_str(format_args!("<tr><th>{}</th>", types[i])));
        for j in 0..5 {
            if i == j {
                html.push_str("<td class=\"diag\">—</td>");
                continue;
            }
            let v = idb_factory::compare_keys(keys[i], keys[j]).unwrap_or(0);
            let cls = if v < 0 { "m1" } else if v > 0 { "p1" } else { "diag" };
            let text = if v < 0 { "-1" } else if v > 0 { "+1" } else { "0" };
            html.push_str(&format_str(format_args!("<td class=\"{cls}\">{text}</td>")));
        }
        html.push_str("</tr>");
    }
    html.push_str("</tbody></table>");
    html.push_str(
        "<p style=\"font-size:11px;color:#888\">Строки упорядочены по возрастанию — \
         значит над диагональю все -1, под диагональю все +1.</p>",
    );
    html.push_str("</details>");

    el.set_inner_html(&html);
}

// ---------------- blocked snapshots ----------------

async fn blocked_scenario(
    status: &web_sys::Element,
    viz: &web_sys::Element,
    result_el: &web_sys::Element,
    held: &Rc<RefCell<Option<JsValue>>>,
    db_info: &web_sys::Element,
    last_touched: &Rc<RefCell<Option<String>>>,
) -> LifecycleSummary {
    close_held(held);
    let _ = idb_factory::delete_demo().await;

    let name = idb_factory::DEMO_DB;
    let mut steps: Vec<Step> = Vec::new();
    let start = now_ms();
    let mut ok_count = 0u32;
    let mut blocked_count = 0u32;
    let mut err_count = 0u32;

    let mut snaps: Vec<BlockedSnapshot> = Vec::new();

    // Шаг 1
    set_status(status, "Blocked: шаг 1/3 — открываем и держим соединение");
    let t0 = now_ms();
    let (outcome, res, snap) = match idb_factory::open_by_name(name, 1).await {
        Ok(db) => {
            *held.borrow_mut() = Some(db);
            (
                StepOutcome::Ok,
                "OK: соединение открыто и удерживается".to_string(),
                BlockedSnapshot {
                    title: "шаг 1/3 · open v1",
                    conn_open: true,
                    db_deleted: false,
                    arrow: Arrow::Open,
                    caption: "Соединение открыто и удерживается. База demo-db существует в v1.",
                },
            )
        }
        Err(e) => (
            StepOutcome::Err,
            describe_idb_error(&e),
            BlockedSnapshot {
                title: "шаг 1/3 · open v1",
                conn_open: false,
                db_deleted: false,
                arrow: Arrow::None,
                caption: "Ошибка при открытии.",
            },
        ),
    };
    snaps.push(snap);
    if outcome == StepOutcome::Ok { ok_count += 1 } else { err_count += 1 }
    steps.push(Step {
        action: "open v1 (удерживаем)".into(),
        result: res,
        outcome,
        duration_ms: now_ms() - t0,
        started_at_ms: now_ms() - start,
    });
    render_snapshots(viz, &snaps);
    refresh_db_list(db_info, last_touched).await;

    // Шаг 2
    set_status(status, "Blocked: шаг 2/3 — delete при активном соединении");
    let t0 = now_ms();
    let (outcome, res) = match idb_factory::delete_by_name(name).await {
        Ok(_) => (
            StepOutcome::Ok,
            "OK (неожиданно — базы нет)".to_string(),
        ),
        Err(e) => {
            let err_name = error_name(&e);
            let text: String = if err_name == "blocked" {
                "BLOCKED — onblocked сработал, промис разрешён".into()
            } else {
                describe_idb_error(&e)
            };
            (StepOutcome::Blocked, text)
        }
    };
    let snap2 = BlockedSnapshot {
        title: "шаг 2/3 · delete",
        conn_open: true,
        db_deleted: false,
        arrow: Arrow::Blocked,
        caption: "Соединение удерживается → сработал onblocked. Мы перехватили событие и разрешили промис.",
    };
    snaps.push(snap2);
    match outcome {
        StepOutcome::Ok => ok_count += 1,
        StepOutcome::Blocked => blocked_count += 1,
        StepOutcome::Err => err_count += 1,
    }
    steps.push(Step {
        action: "delete при активном соединении".into(),
        result: res,
        outcome,
        duration_ms: now_ms() - t0,
        started_at_ms: now_ms() - start,
    });
    render_snapshots(viz, &snaps);
    refresh_db_list(db_info, last_touched).await;

    // Шаг 3
    set_status(status, "Blocked: шаг 3/3 — закрываем и удаляем");
    close_held(held);
    let t0 = now_ms();
    let (outcome, res) = match idb_factory::delete_by_name(name).await {
        Ok(_) => (
            StepOutcome::Ok,
            "OK — после close() удаление прошло".to_string(),
        ),
        Err(e) => (StepOutcome::Err, describe_idb_error(&e)),
    };
    let snap3 = BlockedSnapshot {
        title: "шаг 3/3 · close + delete",
        conn_open: false,
        db_deleted: true,
        arrow: Arrow::DeleteOk,
        caption: "После close() соединение освобождено. Удаление прошло успешно.",
    };
    snaps.push(snap3);
    match outcome {
        StepOutcome::Ok => ok_count += 1,
        StepOutcome::Err => err_count += 1,
        StepOutcome::Blocked => blocked_count += 1,
    }
    steps.push(Step {
        action: "close() + delete".into(),
        result: res,
        outcome,
        duration_ms: now_ms() - t0,
        started_at_ms: now_ms() - start,
    });
    render_snapshots(viz, &snaps);
    refresh_db_list(db_info, last_touched).await;

    // После blocked-сценария восстанавливаем demo v1 — чтобы demo-база
    // всегда была видна в списке.
    if let Ok(db) = idb_factory::open_by_name(name, 1).await {
        idb_factory::close_db(&db);
    }
    refresh_db_list(db_info, last_touched).await;

    render_steps_table(result_el, &steps);
    LifecycleSummary::from_steps(&steps, ok_count, err_count, blocked_count)
}

/// Одно состояние в снапшоте blocked-сценария.
struct BlockedSnapshot {
    title: &'static str,
    conn_open: bool,
    db_deleted: bool,
    arrow: Arrow,
    caption: &'static str,
}

#[derive(Clone, Copy)]
enum Arrow {
    Open,
    Blocked,
    DeleteOk,
    None,
}

/// Рендерит три снапшота подряд — история эксперимента.
fn render_snapshots(el: &web_sys::Element, snaps: &[BlockedSnapshot]) {
    let mut html = String::with_capacity(2048);
    html.push_str("<div class=\"snapshots\">");
    for s in snaps {
        let conn_cls = if s.conn_open { "open" } else { "closed" };
        let conn_label = if s.conn_open { "🔒 открыто" } else { "🔓 закрыто" };
        let db_cls = if s.db_deleted { "deleted" } else { "present" };
        // Единый русский ярлык вместо «present» / «удалена».
        let db_label = if s.db_deleted { "удалена" } else { "присутствует" };

        let (arrow_cls, arrow_icon) = match s.arrow {
            Arrow::Open => ("ok", "✓"),
            Arrow::Blocked => ("blocked", "⊘"),
            Arrow::DeleteOk => ("ok", "✓"),
            Arrow::None => ("none", "—"),
        };

        html.push_str(&format_str(format_args!(
            "<div class=\"snapshot\">\
             <div class=\"snapshot-title\">{}</div>\
             <div class=\"snap-row\">\
             <div class=\"conn-mini {conn_cls}\">{conn_label}</div>\
             <div class=\"snap-arrow {arrow_cls}\">{arrow_icon}</div>\
             <div class=\"db-mini {db_cls}\">{db_label}</div>\
             </div>\
             <div class=\"snap-caption\">{}</div>\
             </div>",
            s.title, s.caption,
        )));
    }
    html.push_str("</div>");
    el.set_inner_html(&html);
    let _ = el.set_attribute("style", "display:block");
}

fn close_held(held: &Rc<RefCell<Option<JsValue>>>) {
    if let Some(db) = held.borrow_mut().take() {
        idb_factory::close_db(&db);
    }
}

// ---------------- helpers ----------------

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}

fn get_button(id: &str) -> Result<HtmlButtonElement, JsValue> {
    get_el(id)?
        .dyn_into::<HtmlButtonElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}

fn set_click<F>(el: &HtmlButtonElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

fn disable_all(buttons: &Rc<Vec<HtmlButtonElement>>, disabled: bool) {
    for b in buttons.iter() {
        b.set_disabled(disabled);
    }
}

fn set_progress(
    el: &HtmlProgressElement,
    text_el: &web_sys::Element,
    value: u32,
    max: u32,
) {
    el.set_value(value as f64);
    el.set_max(max as f64);
    text_el.set_text_content(Some(&format_str(format_args!("{value} / {max}"))));
}

fn now_ms() -> f64 {
    web_sys::window()
        .and_then(|w| w.performance())
        .map(|p| p.now())
        .unwrap_or(0.0)
}

fn db_version(db: &JsValue) -> u32 {
    js_sys::Reflect::get(db, &JsValue::from_str("version"))
        .ok()
        .and_then(|v| v.as_f64())
        .unwrap_or(0.0) as u32
}

fn error_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    if let Some(s) = e.as_string() {
        if s.starts_with("blocked") {
            return "blocked".to_string();
        }
    }
    js_sys::Reflect::get(e, &JsValue::from_str("name"))
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_else(|| "unknown".to_string())
}

fn describe_idb_error(e: &JsValue) -> String {
    let name = error_name(e);
    let meaning: &str = match name.as_str() {
        "VersionError" => "VersionError: понижение версии запрещено",
        "SecurityError" => "SecurityError: доступ к IDB заблокирован",
        "InvalidStateError" => "InvalidStateError: некорректное состояние",
        "NotFoundError" => "NotFoundError: объект не найден",
        "QuotaExceededError" => "QuotaExceededError: переполнена квота",
        "InvalidAccessError" => "InvalidAccessError: неверный доступ",
        "AbortError" => "AbortError: транзакция прервана",
        "ConstraintError" => "ConstraintError: ограничение ключа",
        "DataError" => "DataError: некорректные данные",
        "TransactionInactiveError" => "TransactionInactiveError: транзакция неактивна",
        "ReadOnlyError" => "ReadOnlyError: попытка записи в read-only",
        "blocked" => "blocked: соединение держится другим табом",
        _ => return format_str(format_args!("ERR: {name}")),
    };
    return meaning.to_string();
}

async fn refresh_db_list(
    el: &web_sys::Element,
    last_touched: &Rc<RefCell<Option<String>>>,
) {
    match idb_factory::list_databases().await {
        Ok(list) => {
            if list.is_empty() {
                el.set_inner_html("<em style=\"color:#999\">нет баз в этом origin</em>");
                return;
            }
            let last = last_touched.borrow().clone();
            let mut html = String::with_capacity(256);
            html.push_str("<ul style=\"margin:0;padding-left:20px\">");
            for d in list {
                let is_demo = d.name == idb_factory::DEMO_DB;
                let is_last = last.as_deref() == Some(d.name.as_str());
                let style: String = if is_last {
                    "background:#fff9e6;border-left:3px solid #f39c12;padding-left:6px".into()
                } else if is_demo {
                    "background:#f0f8ff;padding-left:6px".into()
                } else {
                    String::new()
                };
                let marker: &str = if is_demo { " · demo этой главы" } else { "" };
                let star: &str = if is_last { " ★" } else { "" };
                html.push_str(&format_str(format_args!(
                    "<li style=\"{style}\"><code>{}</code> — v<b>{}</b>{marker}{star}</li>",
                    esc(&d.name),
                    d.version,
                )));
            }
            html.push_str("</ul>");
            el.set_inner_html(&html);
        }
        Err(e) => {
            el.set_inner_html(&format_str(format_args!(
                "<em style=\"color:#b03a2e\">ошибка: {}</em>",
                esc(&describe_idb_error(&e)),
            )));
        }
    }
}

async fn copy_to_clipboard(text: &str) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or("no window")?;
    let nav = window.navigator();
    let clipboard = js_sys::Reflect::get(&nav, &JsValue::from_str("clipboard"))
        .map_err(|e| JsValue::from_str(&format!("Reflect clipboard: {e:?}")))?;
    if clipboard.is_undefined() {
        return Err(JsValue::from_str("Clipboard API недоступен"));
    }
    let write_fn = js_sys::Reflect::get(&clipboard, &JsValue::from_str("writeText"))
        .map_err(|e| JsValue::from_str(&format!("Reflect writeText: {e:?}")))?;
    let write: js_sys::Function = write_fn
        .dyn_into()
        .map_err(|_| JsValue::from_str("writeText не функция"))?;
    let promise_val = write
        .call1(&clipboard, &JsValue::from_str(text))
        .map_err(|e| JsValue::from_str(&format!("writeText: {e:?}")))?;
    let promise: js_sys::Promise = promise_val
        .dyn_into()
        .map_err(|_| JsValue::from_str("writeText не Promise"))?;
    wasm_bindgen_futures::JsFuture::from(promise)
        .await
        .map(|_| ())
        .map_err(|e| JsValue::from_str(&format!("await writeText: {e:?}")))
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

## 24.2.8. Практические находки, обнаруженные в процессе работы

Это самое ценное в главе. Каждая находка стоила нам одной-двух итераций отладки — и теперь зафиксирована как правило.

### 1. `IDBRequest` — не `Promise`

Первое, что удивляет при знакомстве с IndexedDB: `open()` возвращает не `Promise`, а `IDBRequest`. У него **нет** `.then()`, `.catch()`, `await`. Есть события `success` и `error`.

В `web-sys` метод `Factory::open()` типизирован как `Result<IdbOpenDbRequest, JsValue>`, но работать с ним напрямую через `wasm-bindgen` неудобно. Мы построили `request_to_promise` — обёртку, которая создаёт `Promise`, подписывается на `onsuccess` и `onerror`, резолвит или реджектит.

**Правило.** Не полагайтесь на `await request` — он не сработает. Оборачивайте в `Promise` вручную.

### 2. `onblocked` — обязательное событие

Если другое соединение (в том же табе или в другой вкладке) держит базу открытой, `deleteDatabase` **не завершается ни успехом, ни ошибкой**. Он зависает. Единственное событие, которое приходит — `onblocked`.

**Что было:** без обработки `onblocked` промис в нашем демо висел бы навсегда. Пользователь нажимал «Проверить blocked» — и ничего не происходило.

**Правило.** В `request_to_promise` подписывайтесь на `onsuccess`, `onerror` **и** `onblocked`. Третий — не опциональный, а обязательный для `deleteDatabase` и `open` с повышением версии.

### 3. `IDBFactory.databases()` возвращает `Promise`, а не `IDBRequest`

Это исключение из правила #1. `databases()` — единственный метод фабрики, возвращающий **настоящий** `Promise<IDBDatabaseInfo[]>`. Мы не оборачиваем его в `request_to_promise`, а ждём через `JsFuture::from(promise)`.

**Правило.** Перед работой с IndexedDB-методом — проверьте в MDN, возвращает он `IDBRequest` или `Promise`. Они не взаимозаменяемы.

### 4. `cmp` — тип доминирует над значением

Это самое контринтуитивное правило IndexedDB. Сравнение ключей идёт **сначала по типу**, потом по значению:

```
number < date < string < binary < array
```

**Что это значит:** `1e9 < Date(0)`. Число 999 999 999 **меньше** даты 1 января 1970 года. Потому что сначала сравнивается тип — number всегда «младше» date, независимо от значения.

Аналогично: пустая строка `""` **меньше** пустого `ArrayBuffer(0)`. Пустой `ArrayBuffer(0)` **меньше** пустого массива `[]`.

**Что было:** первая версия демо показывала только 4 соседние пары. Студент мог решить, что порядок «по величине». Правильная визуализация — цветная ось с `<` между типами и отдельный блок контринтуитивных пар.

**Правило.** При смешивании типов в одном индексе порядок будет странным. Либо не смешивайте, либо объясняйте в UI.

### 5. `VersionError` — защита от отката схемы

`open(name, lower_version)` на существующей базе с большей версией → `VersionError`. IndexedDB **отказывается** открыть базу с меньшей версией, потому что не знает, как откатить уже выполненные миграции.

**Что было:** без правильного проброса `JsValue` ошибка показывалась как `ERR: unknown` — потому что `idb_factory::open_by_name` оборачивала её в строку, и `dyn_ref::<DomException>()` не срабатывал.

**Правило.** Никогда не переупаковывайте `JsValue` в строку. `JsValue::from_str(&format!("{e:?}"))` уничтожает свойства `name`/`message` — а без `name` невозможно классифицировать ошибку.

### 6. `close()` — единственный способ разблокировать `deleteDatabase`

`IDBDatabase` удерживает соединение до тех пор, пока вы явно не вызовете `close()` или пока страница не закроется. Без `close()` даже в том же табе `deleteDatabase` на этой базе получит `onblocked`.

**Правило.** Открыли базу — закройте. Идеально — через RAII-обёртку. В нашем демо — явный `idb_factory::close_db(&db)` после каждого `open`.

### 7. `HtmlButtonElement` vs `HtmlElement`

`set_disabled` — метод `HtmlButtonElement`, **не** `HtmlElement`. В `app.rs` кнопки получены через `get_element_by_id() -> Element`. Нужен явный `.dyn_into::<HtmlButtonElement>()` перед вызовом.

**Правило.** Не ленитесь типизировать элементы при получении. Один раз `get_button(id) -> HtmlButtonElement` — и все дальнейшие вызовы `set_disabled` компилируются.

### 8. `Closure` не экспортируется из prelude

`use wasm_bindgen::prelude::*;` **не** включает `Closure`. Он живёт в `wasm_bindgen::closure::Closure`. Это часто пропускают при переносе кода из примеров в проект, где нет `use wasm_bindgen::closure::Closure;`.

**Правило.** В каждом файле, использующем `Closure::wrap`, — отдельный импорт.

### 9. `JsCast` нужен для `.dyn_into()` и `.dyn_ref()`

Без `use wasm_bindgen::JsCast;` методы `.dyn_into::<T>()` и `.dyn_ref::<T>()` не резолвятся. Rust-analyzer подсказывает — но при переносе файла забывается.

### 10. `text_content()` теряет разделители таблицы

Если хотите скопировать таблицу в буфер обмена — `element.text_content()` вернёт **склеенный** текст без пробелов между ячейками: `статусшагрезультат…`. Потому что между `<td>` в DOM нет текстовых узлов.

**Решение.** Строить plain-text вручную: при сборке таблицы параллельно собирать TSV-отчёт, хранить в `Rc<RefCell<String>>` и по кнопке копировать его.

**Правило.** Буфер обмена — не DOM. Не полагайтесь на текст из HTML — форматируйте сами.

### 11. Wall-clock ≠ сумма длительностей

В первом варианте демо `Σ` считалось как `now_ms() - scenario_start` — wall-clock сценария. Оно не сходилось с суммой длительностей шагов, потому что между шагами было обновление списка баз, перерисовка DOM.

**Правило.** Σ должна быть суммой `step.duration_ms`, а не `now() - start`. Иначе пользователь видит `Σ 36 мс` при сумме шагов `12 мс` и не понимает.

### 12. `—` вместо `+0 мс` в первой строке

Первый шаг стартует на `t = 0`. Показывать `+0 мс` в колонке «от старта» бессмысленно. Правильно — `—`.

### 13. `flex-wrap: wrap` на контейнере снапшотов

Три снапшота по `240px min-width` не влезают в типичную ширину panel. Без `flex-wrap` третий обрезается справа. С `flex-wrap` — переносится. С `min-width: 200px` — все три влезают.

**Правило.** Для горизонтальных серий элементов всегда `flex-wrap: wrap` + `min-width`, чтобы не обрезалось.

### 14. UI требует пересоздания demo-базы после сценария

После lifecycle demo-база удалена (шаг 4 и 6 — delete). Пользователь нажимает «Обновить» и видит только foreign-базы. Список становится пустым — демо «сломалось».

**Решение.** В конце lifecycle и blocked — recreate demo v1. Тогда в списке всегда две базы: demo + foreign.

### 15. Cleanup должен закрывать соединение перед удалением

`cleanup_by_prefix` удалял базы одну за другой. Если в `held_connection` (Эксперимент 3) осталось открытое соединение с demo-базой — `delete_by_name` для неё висел бы на `onblocked`. Мы делаем `close_held()` перед циклом удаления.

**Правило.** Перед массовым удалением закройте все удерживаемые соединения.

### 16. Смешение языков в одной ячейке

В snapshot-карточках было `present` / `удалена` — английский и русский рядом. Унифицировать: `присутствует` / `удалена`.

### 17. `set_attribute("style", ...)` перетирает класс

Если у элемента уже есть класс `.summary` со всем оформлением, `set_attribute("style", "display:flex")` **дополняет** inline-стили к классовым. Это работает, но если первоначально в HTML было `style="display:none"` — его нужно убрать. Мы использовали `set_attribute("style", "display:flex")` — оставшийся `display:none` был перекрыт.

### 18. `<progress>` рендерится браузером по-разному

Зелёный цвет прогресса — это браузерный дефолт, не наш. Не пытайтесь стилизовать `<progress>` глобально — на разных платформах рендер разный. Приемлемо.

### 19. Timing шагов включает ожидание промиса, а не только работу

`open v2` в нашем эксперименте занимает ~17 мс, остальные — 0–4 мс. Это не «медленный open», а **стоимость обновления схемы**: IndexedDB запускает `onupgradeneeded`, пересобирает структуру базы. Даже без нашего обработчика движок выполняет свою работу.

**Правило.** Не удивляйтесь разнице во времени между `open v1` (первое открытие), `open v2` (upgrade), `open v1` (обычное). Это три разных сценария.

### 20. Ошибки в UI — не `{e:?}`, а `error_name(&e)`

`JsValue` при форматировании через `{:?}` даёт `JsValue(DOMException { name: "VersionError", ... })` — шумно и нечитаемо. Правильно — извлечь `name` через `dyn_ref::<DomException>()` или `Reflect::get("name")`.

**Правило.** Всегда классифицируйте ошибку до отображения.

## 24.2.9. Итоги главы

- `IDBFactory` — точка входа. Четыре инструмента: `open`, `deleteDatabase`, `databases`, `cmp`.
- `open(name, version)` — версия часть контракта. Понижение запрещено (`VersionError`).
- `deleteDatabase` идемпотентно: отсутствующая база — не ошибка.
- `databases()` возвращает настоящий `Promise`, остальные методы — `IDBRequest`.
- `cmp(a, b)` — тип доминирует над значением: `number < date < string < binary < array`.
- `IDBRequest` — не `Promise`. Оборачивайте в `Promise` вручную, не забывая про `onblocked`.
- Ошибки `IDBRequest` пробрасывайте как `JsValue` без обёртки в строку.
- `close()` — обязателен для разблокировки `deleteDatabase`.
- Между шагами сценария возможны паузы (перерисовка списка) — они не входят в Σ длительностей шагов.

### Мостик к 24.3

Фабрика открыла базу — мы получили `IDBDatabase`. Это соединение: его можно закрывать, слушать `versionchange`, использовать для транзакций. Следующая подглава — про то, что делать с соединением и как его правильно закрывать в разных сценариях.