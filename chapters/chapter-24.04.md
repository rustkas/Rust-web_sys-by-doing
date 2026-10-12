# Глава 24.4. `IdbDatabase` — схема как контракт

## 24.4.0. Что вы получите в этой главе

24.3 показал, что `IDBDatabase` — это **соединение**, у него есть `name`, `version`, `store_names`, и его можно закрывать. Но мы ни разу не создали ни одного store: база была пустой «схемой из воздуха».

24.4 закрывает этот пробел. Речь пойдёт о **мутациях схемы** — единственных операциях, которые меняют структуру базы:

- `createObjectStore(name, options)` — создать store.
- `deleteObjectStore(name)` — удалить store.
- `createIndex(name, keyPath, options)` — создать индекс.
- `deleteIndex(name)` — удалить индекс.
- Правило: все четыре доступны **только в versionchange-транзакции** (`onupgradeneeded`). Вне её — `InvalidStateError`.
- Следствие: чтобы изменить схему, нужно поднять версию. Менять её «на лету» нельзя.

Демо — **интерактивная песочница**. Пользователь:

1. Загружает **реальную схему домашней библиотеки** — 12 stores, M:N-связи, составные ключи, unique-индексы.
2. Добавляет собственные store'ы и индексы через формы.
3. Разворачивает каждый store в дереве, чтобы увидеть его метаданные.
4. Видит список **документированных FK-связей** между store'ами.

Три «замка» IndexedDB показаны явно:

1. **Растущая схема.** v1 → v2 → v3 — добавление store'ов. Тот же план на новой версии — идемпотентен.
2. **Замок схемы.** `createObjectStore` вне `onupgradeneeded` → `InvalidStateError`.
3. **Монотонность версий.** `open v(N−1)` при текущей `vN` → `VersionError`.

## 24.4.1. Структура файлов

```
hello-wasm02/
├── src/
│   ├── idb_schema.rs        # НОВЫЙ: планировщик схемы (stores + indexes)
│   ├── idb_db.rs            # без изменений (из 24.3)
│   ├── lib.rs               # +1 модуль
│   └── app.rs               # демо — интерактивная песочница
├── tests/
│   └── idb_schema_test.rs   # НОВЫЙ: wasm-тесты (7 штук)
└── Cargo.toml               # без изменений
```

Новых features в `Cargo.toml` не требуется.

**Ссылки для детального изучения:**

- MDN, `IDBDatabase.createObjectStore()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase/createObjectStore
- MDN, `IDBDatabase.deleteObjectStore()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase/deleteObjectStore
- MDN, `IDBObjectStore.createIndex()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/createIndex
- MDN, `IDBObjectStore.deleteIndex()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/deleteIndex
- Spec, «Upgrade a database» — https://w3c.github.io/IndexedDB/#upgrade-transaction-construct
- Spec, «Key construct» (составные ключи) — https://w3c.github.io/IndexedDB/#key-construct

## 24.4.2. Схема — это не только stores

Первое уточнение к 24.3: **схема базы** — это не список store'ов. Это список store'ов **плюс индексы на них**. Индексы — тоже часть контракта: они определяют, какие запросы быстрые, какие ограничения (unique) действуют, и как данные отсортированы.

Поэтому наш планировщик описывает обе сущности:

```rust
#[derive(Clone, Debug, PartialEq)]
pub struct IndexSpec {
    pub name: String,
    /// Имя поля или список через запятую для составного индекса:
    /// «author_id» или «author_id,year».
    pub key_path: String,
    pub unique: bool,
    pub multi_entry: bool,
}

#[derive(Clone, Debug, PartialEq)]
pub struct StoreSpec {
    pub name: String,
    pub key_path: Option<String>,
    pub auto_increment: bool,
    pub indexes: Vec<IndexSpec>,
}
```

`SchemaPlan` — это `Vec<StoreSpec>`. Планировщик применяет его идемпотентно: создаёт недостающие store'ы, синхронизирует индексы, удаляет лишние store'ы. Тот же план на новой версии — no-op.

## 24.4.3. Исходный код `src/idb_schema.rs`

```rust
//! Управление схемой IndexedDB: stores, индексы, версии.
//!
//! ПРАВИЛО: `createObjectStore`, `deleteObjectStore`, `createIndex`,
//! `deleteIndex` работают только внутри versionchange-транзакции
//! (`onupgradeneeded`). Вне её — синхронно `InvalidStateError`.
//!
//! ПРАВИЛО: `keyPath` может быть составным — массив полей. Парсим
//! строку «author_id,year» в массив `["author_id", "year"]`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use crate::idb_db::Database;

/// Описание индекса на object store.
#[derive(Clone, Debug, PartialEq)]
pub struct IndexSpec {
    pub name: String,
    /// Имя поля или список через запятую для составного индекса.
    pub key_path: String,
    pub unique: bool,
    pub multi_entry: bool,
}

impl IndexSpec {
    pub fn new(name: &str, key_path: &str) -> Self {
        Self {
            name: name.to_string(),
            key_path: key_path.to_string(),
            unique: false,
            multi_entry: false,
        }
    }
    pub fn unique(mut self, v: bool) -> Self { self.unique = v; self }
    pub fn multi(mut self, v: bool) -> Self { self.multi_entry = v; self }
}

/// Описание object store.
#[derive(Clone, Debug, PartialEq)]
pub struct StoreSpec {
    pub name: String,
    /// `None` — autoIncrement без явного keyPath.
    pub key_path: Option<String>,
    pub auto_increment: bool,
    pub indexes: Vec<IndexSpec>,
}

impl StoreSpec {
    pub fn with_key(name: &str, key_path: &str) -> Self {
        Self {
            name: name.to_string(),
            key_path: Some(key_path.to_string()),
            auto_increment: false,
            indexes: vec![],
        }
    }
    pub fn with_auto_key(name: &str) -> Self {
        Self {
            name: name.to_string(),
            key_path: None,
            auto_increment: true,
            indexes: vec![],
        }
    }
    pub fn indexes(mut self, idx: Vec<IndexSpec>) -> Self {
        self.indexes = idx;
        self
    }
}

/// Декларативное описание схемы. План — источник правды.
#[derive(Clone, Debug, Default)]
pub struct SchemaPlan {
    pub stores: Vec<StoreSpec>,
}

impl SchemaPlan {
    pub fn new() -> Self { Self { stores: Vec::new() } }
    pub fn add(mut self, spec: StoreSpec) -> Self {
        self.stores.push(spec);
        self
    }
    pub fn names(&self) -> Vec<String> {
        self.stores.iter().map(|s| s.name.clone()).collect()
    }
}

/// Что изменилось при применении плана.
#[derive(Clone, Debug, Default)]
pub struct Diff {
    pub upgraded: bool,
    pub old_version: u32,
    pub new_version: u32,
    pub created: Vec<String>,
    pub deleted: Vec<String>,
    pub kept: Vec<String>,
    pub indexes_created: Vec<String>,
    pub indexes_deleted: Vec<String>,
    pub errors: Vec<String>,
}

pub struct OpenOutcome {
    pub db: Database,
    pub diff: Diff,
}

pub async fn open_with_plan(
    name: &str,
    version: u32,
    plan: &SchemaPlan,
) -> Result<OpenOutcome, JsValue> {
    let diff = Rc::new(RefCell::new(Diff::default()));
    let diff_c = diff.clone();
    let plan_c = plan.clone();

    let raw_db = open_with_upgrade(
        name, version,
        move |db, tx, old_v, new_v| {
            let mut d = diff_c.borrow_mut();
            d.upgraded = true;
            d.old_version = old_v;
            d.new_version = new_v;
            apply_plan(db, tx, &plan_c, &mut d);
        },
    ).await?;

    let final_diff = diff.borrow().clone();
    Ok(OpenOutcome {
        db: Database::from_raw(raw_db),
        diff: final_diff,
    })
}

pub fn try_create_on_open(db: &Database, spec: &StoreSpec) -> Result<(), JsValue> {
    create_store_raw(db.raw(), spec)
}

pub fn try_delete_on_open(db: &Database, store_name: &str) -> Result<(), JsValue> {
    delete_store_raw(db.raw(), store_name)
}

// ============================================================
//                     ВНУТРЕННЕЕ
// ============================================================

async fn open_with_upgrade<F>(
    name: &str,
    version: u32,
    on_upgrade: F,
) -> Result<JsValue, JsValue>
where
    F: FnOnce(&JsValue, &JsValue, u32, u32) + 'static,
{
    let global = js_sys::global();
    let idb = js_sys::Reflect::get(&global, &JsValue::from_str("indexedDB"))
        .map_err(|e| JsValue::from_str(&format!("Reflect indexedDB: {e:?}")))?;
    if !idb.is_object() {
        return Err(JsValue::from_str("indexedDB недоступен"));
    }
    let open_fn: js_sys::Function = js_sys::Reflect::get(&idb, &JsValue::from_str("open"))
        .map_err(|e| JsValue::from_str(&format!("Reflect open: {e:?}")))?
        .dyn_into().map_err(|_| JsValue::from_str("open не функция"))?;

    let request = open_fn
        .call2(&idb, &JsValue::from_str(name), &JsValue::from_f64(version as f64))
        .map_err(|e| JsValue::from_str(&format!("open: {e:?}")))?;

    // onupgradeneeded — единственное окно, где разрешены мутации схемы.
    let on_upgrade_js = Closure::once_into_js(move |e: web_sys::Event| {
        let target = match e.target() { Some(t) => t, None => return };
        let db = match js_sys::Reflect::get(&target, &JsValue::from_str("result")) {
            Ok(v) => v, Err(_) => return,
        };
        // request.transaction — versionchange-транзакция, активна внутри
        // этого события. Через неё берём store-объекты для sync индекс.
        let tx = match js_sys::Reflect::get(&target, &JsValue::from_str("transaction")) {
            Ok(v) => v, Err(_) => return,
        };
        let old_v = js_sys::Reflect::get(e.as_ref(), &JsValue::from_str("oldVersion"))
            .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
        let new_v = js_sys::Reflect::get(e.as_ref(), &JsValue::from_str("newVersion"))
            .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
        on_upgrade(&db, &tx, old_v, new_v);
    });
    js_sys::Reflect::set(&request, &JsValue::from_str("onupgradeneeded"), &on_upgrade_js)?;

    let r_ok = request.clone();
    let r_err = request.clone();
    let r_block = request.clone();
    let promise = js_sys::Promise::new(&mut move |resolve, reject| {
        let r = r_ok.clone();
        let resolve_in = resolve.clone();
        let on_success = Closure::once_into_js(move |_e: web_sys::Event| {
            let result = js_sys::Reflect::get(&r, &JsValue::from_str("result"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = resolve_in.call1(&JsValue::UNDEFINED, &result);
        });
        let _ = js_sys::Reflect::set(&r_ok, &JsValue::from_str("onsuccess"), &on_success);

        let r = r_err.clone();
        let reject_in = reject.clone();
        let on_error = Closure::once_into_js(move |_e: web_sys::Event| {
            let err = js_sys::Reflect::get(&r, &JsValue::from_str("error"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = reject_in.call1(&JsValue::UNDEFINED, &err);
        });
        let _ = js_sys::Reflect::set(&r_err, &JsValue::from_str("onerror"), &on_error);

        let reject_b = reject.clone();
        let on_blocked = Closure::once_into_js(move |_e: web_sys::Event| {
            let _ = reject_b.call1(&JsValue::UNDEFINED,
                &JsValue::from_str("blocked: соединение держится другим табом"));
        });
        let _ = js_sys::Reflect::set(&r_block, &JsValue::from_str("onblocked"), &on_blocked);
    });

    JsFuture::from(promise).await
}

fn apply_plan(db: &JsValue, tx: &JsValue, plan: &SchemaPlan, diff: &mut Diff) {
    let existing = list_store_names(db);
    let plan_names: Vec<&str> = plan.stores.iter().map(|s| s.name.as_str()).collect();

    // 1. Создать недостающие store'ы + синхронизировать индексы.
    for spec in &plan.stores {
        let is_new = !existing.iter().any(|n| n == &spec.name);
        if is_new {
            match create_store_raw(db, spec) {
                Ok(()) => diff.created.push(spec.name.clone()),
                Err(e) => {
                    diff.errors.push(format!("create {}: {e:?}", spec.name));
                    continue;
                }
            }
        } else {
            diff.kept.push(spec.name.clone());
        }

        // Синхронизация индексов на этом store.
        if let Ok(store) = get_store_from_tx(tx, &spec.name) {
            sync_indexes(&store, spec, diff);
        }
    }

    // 2. Удалить store'ы, которых нет в плане.
    for name in existing {
        if !plan_names.contains(&name.as_str()) {
            match delete_store_raw(db, &name) {
                Ok(()) => diff.deleted.push(name),
                Err(e) => diff.errors.push(format!("delete {name}: {e:?}")),
            }
        }
    }
}

fn sync_indexes(store: &JsValue, spec: &StoreSpec, diff: &mut Diff) {
    let existing = list_index_names(store);
    let planned: Vec<&str> = spec.indexes.iter().map(|i| i.name.as_str()).collect();

    // Создать недостающие.
    for idx in &spec.indexes {
        if existing.iter().any(|n| n == &idx.name) { continue; }
        match create_index_raw(store, idx) {
            Ok(()) => diff.indexes_created.push(format!("{}.{}", spec.name, idx.name)),
            Err(e) => diff.errors.push(format!("index {}.{}: {e:?}", spec.name, idx.name)),
        }
    }

    // Удалить лишние.
    for name in existing {
        if !planned.contains(&name.as_str()) {
            match delete_index_raw(store, &name) {
                Ok(()) => diff.indexes_deleted.push(format!("{}.{}", spec.name, name)),
                Err(e) => diff.errors.push(format!("delIndex {}.{}: {e:?}", spec.name, name)),
            }
        }
    }
}

fn get_store_from_tx(tx: &JsValue, name: &str) -> Result<JsValue, JsValue> {
    let f: js_sys::Function = js_sys::Reflect::get(tx, &JsValue::from_str("objectStore"))
        .map_err(|e| JsValue::from_str(&format!("Reflect objectStore: {e:?}")))?
        .dyn_into().map_err(|_| JsValue::from_str("objectStore не функция"))?;
    f.call1(tx, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("objectStore: {e:?}")))
}

fn list_store_names(db: &JsValue) -> Vec<String> {
    let list = match js_sys::Reflect::get(db, &JsValue::from_str("objectStoreNames")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return Vec::new(),
    };
    return list_domstringlist(&list);
}

fn list_index_names(store: &JsValue) -> Vec<String> {
    let list = match js_sys::Reflect::get(store, &JsValue::from_str("indexNames")) {
        Ok(v) if !v.is_undefined() && !v.is_null() => v,
        _ => return Vec::new(),
    };
    return list_domstringlist(&list);
}

fn list_domstringlist(list: &JsValue) -> Vec<String> {
    let length = js_sys::Reflect::get(list, &JsValue::from_str("length"))
        .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
    let item_fn = match js_sys::Reflect::get(list, &JsValue::from_str("item")) {
        Ok(v) => v, Err(_) => return Vec::new(),
    };
    let item: js_sys::Function = match item_fn.dyn_into() { Ok(f) => f, Err(_) => return Vec::new() };
    let mut out = Vec::with_capacity(length as usize);
    for i in 0..length {
        if let Ok(v) = item.call1(list, &JsValue::from_f64(i as f64)) {
            if let Some(s) = v.as_string() { out.push(s); }
        }
    }
    out
}

/// Разбирает `keyPath`: «author_id,year» → `["author_id", "year"]`.
fn parse_key_path(kp: &str) -> JsValue {
    let t = kp.trim();
    if t.contains(',') {
        let arr = js_sys::Array::new();
        for p in t.split(',') {
            arr.push(&JsValue::from_str(p.trim()));
        }
        arr.into()
    } else {
        JsValue::from_str(t)
    }
}

fn create_store_raw(db: &JsValue, spec: &StoreSpec) -> Result<(), JsValue> {
    let create_fn: js_sys::Function =
        js_sys::Reflect::get(db, &JsValue::from_str("createObjectStore"))
            .map_err(|e| JsValue::from_str(&format!("Reflect createObjectStore: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("createObjectStore не функция"))?;

    let options = js_sys::Object::new();
    if let Some(kp) = &spec.key_path {
        js_sys::Reflect::set(&options, &JsValue::from_str("keyPath"), &parse_key_path(kp))?;
    }
    if spec.auto_increment {
        js_sys::Reflect::set(&options, &JsValue::from_str("autoIncrement"), &JsValue::TRUE)?;
    }
    create_fn.call2(db, &JsValue::from_str(&spec.name), &options)
        .map_err(|e| e)?;
    Ok(())
}

fn create_index_raw(store: &JsValue, idx: &IndexSpec) -> Result<(), JsValue> {
    let create_fn: js_sys::Function =
        js_sys::Reflect::get(store, &JsValue::from_str("createIndex"))
            .map_err(|e| JsValue::from_str(&format!("Reflect createIndex: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("createIndex не функция"))?;

    let kp = parse_key_path(&idx.key_path);
    let opts = js_sys::Object::new();
    if idx.unique {
        js_sys::Reflect::set(&opts, &JsValue::from_str("unique"), &JsValue::TRUE)?;
    }
    if idx.multi_entry {
        js_sys::Reflect::set(&opts, &JsValue::from_str("multiEntry"), &JsValue::TRUE)?;
    }
    create_fn.call3(store, &JsValue::from_str(&idx.name), &kp, &opts)
        .map_err(|e| e)?;
    Ok(())
}

fn delete_index_raw(store: &JsValue, name: &str) -> Result<(), JsValue> {
    let del_fn: js_sys::Function =
        js_sys::Reflect::get(store, &JsValue::from_str("deleteIndex"))
            .map_err(|e| JsValue::from_str(&format!("Reflect deleteIndex: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("deleteIndex не функция"))?;
    del_fn.call1(store, &JsValue::from_str(name)).map_err(|e| e)?;
    Ok(())
}

fn delete_store_raw(db: &JsValue, name: &str) -> Result<(), JsValue> {
    let del_fn: js_sys::Function =
        js_sys::Reflect::get(db, &JsValue::from_str("deleteObjectStore"))
            .map_err(|e| JsValue::from_str(&format!("Reflect deleteObjectStore: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("deleteObjectStore не функция"))?;
    del_fn.call1(db, &JsValue::from_str(name)).map_err(|e| e)?;
    Ok(())
}
```

## 24.4.4. Индексы — что реально даёт каждое поле

**`keyPath`.** Одно поле (`"id"`) или список через запятую (`"author_id,year"`). Составной индекс — это путь к полю внутри объекта, где сам ключ является массивом: `{author_id: "a", year: 2020}` → ключ `["a", 2020]`.

**`unique`.** `true` — движок бросает `ConstraintError` при попытке вставить вторую запись с тем же значением. Это аналог `UNIQUE` в SQL.

**`multiEntry`.** `true` — если поле содержит массив, каждый элемент становится отдельным ключом индекса. Так моделируется M:N без промежуточного store'а: `{isbn: "x", tags: ["fiction", "classic"]}` — индекс по `tags` вернёт эту книгу и для `"fiction"`, и для `"classic"`.

## 24.4.5. Тесты — `tests/idb_schema_test.rs`

Семь wasm-тестов покрывают:

1. **`open_creates_planned_stores`** — создание базы с двумя store'ами, проверка `diff.created`, `store_names()`, `version()`.
2. **`auto_increment_store_creation`** — store с `autoIncrement: true`.
3. **`open_is_idempotent_with_same_plan`** — тот же план на новой версии: `diff.created` и `diff.deleted` пусты, `diff.kept` полон.
4. **`plan_delete_removes_store`** — план без store'а → `diff.deleted`.
5. **`create_outside_upgrade_fails`** — `try_create_on_open` бросает `InvalidStateError`.
6. **`delete_outside_upgrade_fails`** — то же для `deleteObjectStore`.
7. **`version_cannot_go_back`** — `open v1` при `v2` → `VersionError`.

```rust
//! Wasm-тесты модуля `idb_schema`: планировщик схемы, createObjectStore,
//! deleteObjectStore, version-монотонность.
//!
//! Компилируются только под wasm32 — на host файл пуст.
//!
//! ПРАВИЛО: `format!` в tail-позиции функции оборачивается в явный `return`.
//! rust-analyzer стабильно теряет тип `String` в этой позиции и рапортует
//! `expected String, found ()` — ошибка потом «проливается» на вызывающих.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::idb_factory;
use hello_wasm02::idb_schema::{self, SchemaPlan, StoreSpec};

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64);
}

fn test_db() -> String {
    return format!("test-schema-{}", uid());
}

fn err_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    return e.as_string().unwrap_or_else(|| format!("{e:?}"));
}

// ---------------- создание схемы ----------------

#[wasm_bindgen_test]
async fn open_creates_planned_stores() {
    let name = test_db();
    let plan = SchemaPlan::new()
        .add(StoreSpec::with_key("authors", "id"))
        .add(StoreSpec::with_key("books", "isbn"));

    let outcome = idb_schema::open_with_plan(&name, 1, &plan)
        .await
        .expect("open v1");

    assert!(outcome.diff.upgraded, "должен был быть upgrade");
    assert_eq!(outcome.diff.old_version, 0);
    assert_eq!(outcome.diff.new_version, 1);
    assert_eq!(outcome.diff.created.len(), 2);
    assert!(outcome.diff.created.contains(&"authors".to_string()));
    assert!(outcome.diff.created.contains(&"books".to_string()));
    assert!(outcome.diff.deleted.is_empty());
    assert!(outcome.diff.errors.is_empty());

    let mut names = outcome.db.store_names();
    names.sort();
    assert_eq!(names, vec!["authors", "books"]);
    assert_eq!(outcome.db.version(), 1);

    outcome.db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn auto_increment_store_creation() {
    let name = test_db();
    let plan = SchemaPlan::new().add(StoreSpec::with_auto_key("logs"));

    let outcome = idb_schema::open_with_plan(&name, 1, &plan)
        .await
        .expect("open");
    assert_eq!(outcome.diff.created, vec!["logs"]);
    assert_eq!(outcome.db.store_names(), vec!["logs"]);

    outcome.db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- идемпотентность плана ----------------

#[wasm_bindgen_test]
async fn open_is_idempotent_with_same_plan() {
    let name = test_db();
    let plan = SchemaPlan::new().add(StoreSpec::with_key("authors", "id"));

    let o1 = idb_schema::open_with_plan(&name, 1, &plan)
        .await
        .expect("open v1");
    assert_eq!(o1.diff.created, vec!["authors"]);
    o1.db.close();

    // Тот же план, но версия выше — upgrade запускается, но менять нечего.
    let o2 = idb_schema::open_with_plan(&name, 2, &plan)
        .await
        .expect("open v2");
    assert!(o2.diff.upgraded);
    assert!(o2.diff.created.is_empty(), "нечего создавать");
    assert!(o2.diff.deleted.is_empty(), "нечего удалять");
    assert_eq!(o2.diff.kept, vec!["authors"]);
    assert_eq!(o2.db.version(), 2);

    o2.db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- удаление store через план ----------------

#[wasm_bindgen_test]
async fn plan_delete_removes_store() {
    let name = test_db();

    // v1: authors + books.
    let plan_v1 = SchemaPlan::new()
        .add(StoreSpec::with_key("authors", "id"))
        .add(StoreSpec::with_key("books", "isbn"));
    let o1 = idb_schema::open_with_plan(&name, 1, &plan_v1)
        .await
        .expect("v1");
    o1.db.close();

    // v2: только authors — books удаляется планом.
    let plan_v2 = SchemaPlan::new().add(StoreSpec::with_key("authors", "id"));
    let o2 = idb_schema::open_with_plan(&name, 2, &plan_v2)
        .await
        .expect("v2");

    assert_eq!(o2.diff.deleted, vec!["books"]);
    assert_eq!(o2.diff.kept, vec!["authors"]);
    assert_eq!(o2.db.store_names(), vec!["authors"]);

    o2.db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- замок схемы: только в upgrade ----------------

#[wasm_bindgen_test]
async fn create_outside_upgrade_fails() {
    let name = test_db();
    let plan = SchemaPlan::new().add(StoreSpec::with_key("authors", "id"));
    let o = idb_schema::open_with_plan(&name, 1, &plan)
        .await
        .expect("open");

    let spec = StoreSpec::with_key("late_store", "id");
    let result = idb_schema::try_create_on_open(&o.db, &spec);

    assert!(
        result.is_err(),
        "createObjectStore вне upgrade должен упасть"
    );
    let name_str = err_name(&result.unwrap_err());
    assert_eq!(
        name_str, "InvalidStateError",
        "ожидали InvalidStateError, получили {name_str}"
    );

    o.db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn delete_outside_upgrade_fails() {
    let name = test_db();
    let plan = SchemaPlan::new().add(StoreSpec::with_key("authors", "id"));
    let o = idb_schema::open_with_plan(&name, 1, &plan)
        .await
        .expect("open");

    let result = idb_schema::try_delete_on_open(&o.db, "authors");
    assert!(
        result.is_err(),
        "deleteObjectStore вне upgrade должен упасть"
    );
    let name_str = err_name(&result.unwrap_err());
    assert_eq!(name_str, "InvalidStateError");

    o.db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- монотонность версий ----------------

#[wasm_bindgen_test]
async fn version_cannot_go_back() {
    let name = test_db();
    let plan = SchemaPlan::new().add(StoreSpec::with_key("authors", "id"));

    let o = idb_schema::open_with_plan(&name, 2, &plan)
        .await
        .expect("v2");
    o.db.close();

    // Попытка открыть базу в версии ниже существующей.
    let result = idb_schema::open_with_plan(&name, 1, &plan).await;
    assert!(result.is_err(), "понижение версии запрещено");

    let err = result.err().unwrap();
    assert_eq!(err_name(&err), "VersionError");

    idb_factory::delete_by_name(&name).await.expect("cleanup");
}
```

## 24.4.6. Демо-пример — `src/app.rs`

```

```


Демо — **интерактивная песочница**. Что важно:

**1. Схема библиотеки — пресет.** 12 store'ов домашней библиотеки: `authors`, `publishers`, `books`, `genres`, `book_genres` (M:N), `tags`, `book_tags` (M:N), `readers`, `loans`, `reviews`, `covers` (1:1), `audit_log`. Каждый с реалистичными индексами:

```rust
fn library_schema() -> SchemaPlan {
    use IndexSpec as I;
    SchemaPlan::new()
        .add(StoreSpec::with_key("authors", "id")
            .indexes(vec![I::new("by_name", "name")]))
        .add(StoreSpec::with_key("books", "isbn")
            .indexes(vec![
                I::new("by_author", "author_id"),
                I::new("by_publisher", "publisher_id"),
                I::new("by_year", "year"),
                I::new("by_title", "title"),
            ]))
        .add(StoreSpec::with_key("book_genres", "book_isbn,genre_id")
            .indexes(vec![
                I::new("by_book", "book_isbn"),
                I::new("by_genre", "genre_id"),
            ]))
        .add(StoreSpec::with_key("readers", "id")
            .indexes(vec![I::new("by_email", "email").unique(true)]))
        // ...
        .add(StoreSpec::with_auto_key("audit_log")
            .indexes(vec![I::new("by_ts", "ts")]))
}
```

**2. Инспектор схемы.** Каждый store раскрывается через `<details>`:

```
▸ authors        key: id     auto: —    1 idx
  keyPath: id
  autoIncrement: false
  indexes:
    by_name (name)
```

Уникальные индексы помечены бейджем `UNIQUE`, multiEntry — `MULTI`.

**3. Панель «Связи»** — документированные FK. В IndexedDB нет `FOREIGN KEY`, но связи моделируются ссылками между полями. Панель показывает 11 связей:

```
books.author_id       → authors.id      (1:N)
book_genres.book_isbn → books.isbn      (M:N)
book_genres.genre_id  → genres.id       (M:N)
covers.book_isbn      → books.isbn      (1:1)
```

**4. Форма «Add index».** Пользователь выбирает store, вводит имя и keyPath, ставит галочки unique/multi — планировщик создаёт индекс в новой версии.

**5. Восстановление после перезагрузки.** При старте `restore_from_existing` читает `indexedDB.databases()`, если наша база есть — открывает её и через readonly-транзакцию читает `keyPath`, `autoIncrement`, `indexNames`, `index.keyPath`, `index.unique`, `index.multiEntry` каждого store'а.

**6. Наблюдатель.** Каждые 2 секунды `watch_db_state` опрашивает `databases()`. Если пользователь удалил базу из DevTools — UI сбрасывается и показывает предупреждение.

**7. Rename store.** Трёхшаговый workflow, при котором индексы source'а переносятся на target (шаг 1), данные копируются (шаг 2 — пропускается при пустом source), source удаляется (шаг 3).

## 24.4.7. Найденные практики

**1. `keyPath` — не строка, а JS-значение.** `createObjectStore("x", {keyPath: "id"})` принимает строку. `createObjectStore("x", {keyPath: ["a", "b"]})` — массив. При сериализации плана в Rust удобно хранить строку `"a,b"` и парсить в массив на границе.

**2. Индексы внутри versionchange-транзакции.** `createIndex` доступен только внутри `onupgradeneeded` — так же, как `createObjectStore`. Но обращаться к store-объекту можно двумя путями:
- Через `db.createObjectStore(...)` — если store только что создан.
- Через `request.transaction.objectStore(name)` — если store уже существовал до этого upgrade.

Второй путь нужен, чтобы досоздать индекс на существующем store'е. Без него получаем `NotFoundError`.

**3. Наблюдение за удалением базы из DevTools.** `indexedDB.databases()` возвращает snapshot. Опрос раз в 2 секунды — приемлемая цена за то, что пользователь видит консистентное состояние. Без опроса UI показывает устаревшие данные, а следующий `put` даёт `InvalidStateError`.

**4. CSS `#app { width: 100vw !important; margin-left: calc(50% - 50vw); }`.** В стандартном Vite-шаблоне `main.css` ограничивает `body` до 760px. Full-bleed обходит это без правки глобальных стилей.

**5. `format!` в tail-позиции** — только через `format_str(format_args!(...))` или явный `return`. Эта глава — не исключение.

**6. Поле структуры, которое не читается — `dead_code`.** `kept` в `VersionRecord` был нужен только для diff-строки в таймлайне, но его можно вывести из `plan.stores` минус `created` минус `deleted`. Убрали.

## 24.4.8. Связи между store'ами — моделирование без FOREIGN KEY

Короткий ответ на частый вопрос: **внешних ключей в IndexedDB нет**. Это не реляционная база. Но связи моделируются вручную тремя способами.

**Reference by key.** То, что уже используется в `books.author_id`. Плюс: быстрый поиск через индекс `by_author`. Минус: движок не проверяет существование автора при `put(book)`.

**Denormalization.** Хранить копию автора внутри книги. Плюс: один запрос читает всё. Минус: обновление имени автора требует обхода всех книг.

**Explicit JOIN.** Два store'а в одной транзакции, ручное соединение в Rust. `O(n)` по числу книг, но все запросы в памяти движка.

Каскадное удаление — **ваша функция**, а не свойство движка:

```rust
pub async fn remove_author_cascade(author_id: &str) -> Result<u32, JsValue> {
    let db = open().await?;
    let tx = db.transaction(&["authors", "books"], TransactionMode::ReadWrite)?;

    // 1) найти книги автора через индекс by_author
    let books = tx.object_store("books")?;
    let idx = books.index("by_author")?;
    let list = idx.get_all(...)?.await?;

    // 2) удалить каждую
    for v in list {
        let b: Book = serde_wasm_bindgen::from_value(v)?;
        books.delete(JsValue::from_str(&b.isbn))?;
    }

    // 3) удалить автора
    tx.object_store("authors")?.delete(JsValue::from_str(author_id))?;

    // 4) одна транзакция = атомарность
    tx.commit()?;
    Ok(removed)
}
```

Атомарность между store'ами есть — а правила целостности пишете вы.

## 24.4.9. Итоги главы

- Схема базы — это **stores + indexes**, обе сущности меняются только в `onupgradeneeded`.
- Планировщик (`SchemaPlan`) описывает желаемое состояние декларативно, применение идемпотентно.
- `keyPath` может быть составным — массив полей, парсится из строки `"a,b"`.
- `unique` и `multiEntry` — часть контракта индекса.
- Вне versionchange-транзакции `createObjectStore` и `deleteObjectStore` **синхронно** бросают `InvalidStateError`.
- Версии базы **монотонны**. Понижение — `VersionError`.
- Удаление базы из DevTools отслеживается опросом `indexedDB.databases()`.
- Восстановление схемы после перезагрузки — чтение `keyPath`, `autoIncrement`, `indexNames` через readonly-транзакцию.
- Связи между store'ами моделируются вручную: reference by key, denormalization, explicit JOIN. Каскады — ваша ответственность.

**Мостик к 24.5.** Схема готова. Дальше — данные: `IdbObjectStore` и его операции `put`, `get`, `getAll`, `delete`, `openCursor`. Индексы, которые мы только что построили, начнут работать в полную силу.

## 24.4.10. Демо-пример — `src/app.rs` (исходный код)

```
//! Демо Главы 24.4: IdbDatabase — схема как контракт.
//!
//! ПРАВИЛО: `format!` в tail-позиции — `format_str(format_args!(...))` или явный `return`.
//! ПРАВИЛО: `JsCast` для `.dyn_into`/`.dyn_ref`.
//! ПРАВИЛО: `Closure` — из `wasm_bindgen::closure::Closure`.
//! ПРАВИЛО: длительные операции с UI — yield через TimeoutFuture::new(0).

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{HtmlButtonElement, HtmlInputElement, HtmlSelectElement};

use crate::format_str::format_str;
use crate::idb_db::Database;
use crate::idb_factory;
use crate::idb_schema::{self, IndexSpec, SchemaPlan, StoreSpec};

const DEMO_DB: &str = "hello-wasm-idb-24-4-schema";
const TIMELINE_LIMIT: usize = 30;
const WATCH_INTERVAL_MS: u32 = 2000;

const HTML: &str = r#"
<style>
  :root {
    --c-info: #3498db; --c-ok: #27ae60; --c-err: #c0392b; --c-warn: #e67e22;
    --c-text: #222; --c-muted: #666;
    --c-bg-panel: #fff; --c-bg-soft: #fafafa; --c-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --c-text: #e6e6e6; --c-muted: #a0a0a0;
      --c-bg-panel: #1e1e1e; --c-bg-soft: #262626; --c-border: #3a3a3a;
    }
    body { background: #121212; }
    .anno { background: #1a2233; color: #9fc4ff; }
    .scenario { background: #232014; color: #e6c98c; }
    .lc-node { background: #262626; }
    .tl-entry { background: #262626; }
    .chip { background: #2a3a4a; color: #cfe0ff; border-color: #3a506a; }
    .chip.created { background: #1e3a1e; color: #b8e0b8; border-color: #2a4a2a; }
    .chip.deleted { background: #3a1e1e; color: #e0b8b8; border-color: #4a2a2a; }
    .tl-ver { background: #2a3a4a; color: #cfe0ff; }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    input::placeholder { color: #666; }
    input:disabled { background: #181818; color: #555; }
    .records-badge { background: #2a3a4a; color: #cfe0ff; }
    details.store-details summary { background: #262626; }
    details.store-details[open] summary { background: #2a3a4a; }
    .idx-row { background: #1e1e1e; }
    .rel-item { background: #1e2a1e; }
  }

  /* Обход main.css (body max-width) */
  html { width: 100% !important; max-width: none !important; margin: 0 !important; padding: 0 !important; overflow-x: hidden; }
  body {
    display: block !important;
    width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--c-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa; overflow-x: hidden;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app {
    display: block !important; width: 100% !important; max-width: 100% !important;
    margin: 0 !important; padding: 12px 16px 20px !important;
    box-sizing: border-box !important;
  }
  #app, #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .subtitle { color: var(--c-muted); font-size: 12px; margin: 0 0 10px; }
  .subtitle code { background: rgba(128,128,128,0.15); padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px; }
  .subtitle a { color: var(--c-info); }

  .banner {
    display: flex; align-items: center; gap: 14px; flex-wrap: wrap;
    padding: 9px 14px; margin: 0 0 12px;
    background: linear-gradient(135deg, #eaf4ff 0%, #f5f0ff 100%);
    border: 2px solid var(--c-info); border-radius: 8px;
  }
  @media (prefers-color-scheme: dark) {
    .banner { background: linear-gradient(135deg, #1a2a3f 0%, #2a1e3f 100%); }
  }
  .banner-label { font-size: 10px; text-transform: uppercase; letter-spacing: 0.7px; color: var(--c-muted); }
  .banner-name { font-family: ui-monospace, monospace; font-size: 13px; font-weight: 700;
    color: #1f4e79; word-break: break-all; }
  @media (prefers-color-scheme: dark) { .banner-name { color: #9fc4ff; } }
  .banner-meta { font-family: ui-monospace, monospace; font-size: 11px; color: var(--c-muted); margin-top: 2px; }
  .banner-actions { margin-left: auto; display: flex; gap: 8px; }

  .app-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(0, 1.1fr) minmax(0, 1.1fr) minmax(0, 0.9fr);
    grid-template-areas:
      "lifecycle lifecycle lifecycle"
      "state     state     state"
      "mutations constraints history";
  }
  .lifecycle-panel { grid-area: lifecycle; }
  .state-row {
    grid-area: state;
    display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1fr) minmax(0, 1fr);
    gap: 12px;
  }
  .col-mutations   { grid-area: mutations; }
  .col-constraints { grid-area: constraints; }
  .col-history     { grid-area: history; }

  @media (max-width: 1180px) {
    .app-grid {
      grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
      grid-template-areas:
        "lifecycle lifecycle"
        "state     state"
        "mutations constraints"
        "history   history";
    }
    .state-row { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); }
  }
  @media (max-width: 760px) {
    .app-grid {
      grid-template-columns: 1fr;
      grid-template-areas: "lifecycle" "state" "mutations" "constraints" "history";
    }
    .state-row { grid-template-columns: 1fr; }
  }

  .col { display: flex; flex-direction: column; gap: 10px; min-width: 0; }

  .panel {
    border: 1px solid var(--c-border); border-radius: 8px;
    padding: 10px 12px; background: var(--c-bg-panel);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0; overflow: hidden;
  }
  .panel-title { font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap; }
  .exp-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--c-info); color: #fff; font-weight: 700; font-size: 11px; flex-shrink: 0;
  }

  .anno { background: #eef4ff; border-left: 3px solid var(--c-info);
    padding: 6px 10px; border-radius: 4px; font-size: 11px; line-height: 1.45;
    margin: 6px 0; color: #1f4e79; overflow-wrap: anywhere; word-break: break-word; }
  .anno code { background: rgba(52,152,219,0.12); padding: 0 4px;
    border-radius: 3px; font-family: ui-monospace, monospace; font-size: 10px; }
  .scenario { background: #fff9e6; border-left: 3px solid var(--c-warn);
    padding: 6px 10px; border-radius: 4px; font-size: 11px; line-height: 1.5;
    margin: 6px 0; color: #5a4400; overflow-wrap: anywhere; word-break: break-word; }
  .scenario ol { margin: 2px 0 0 16px; padding: 0; }
  .scenario li { padding: 0; }

  .form-row { display: flex; gap: 5px; flex-wrap: wrap; margin: 6px 0;
    align-items: center; min-width: 0; }
  input[type="text"], select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--c-bg-panel); color: var(--c-text);
    font-family: ui-monospace, monospace; min-width: 0; max-width: 100%;
  }
  input[type="text"] { width: 120px; flex: 1 1 100px; }
  input[type="text"].key-input { width: 80px; flex: 0 1 80px; }
  input[type="text"]:disabled { opacity: 0.55; cursor: not-allowed; background: var(--c-bg-soft); }
  select { min-width: 0; flex: 1 1 100px; }
  label.check { display: inline-flex; align-items: center; gap: 4px;
    font-size: 11px; color: var(--c-muted); cursor: pointer;
    user-select: none; white-space: nowrap; flex-shrink: 0; }
  input[type="checkbox"] { margin: 0; flex-shrink: 0; }
  input[type="range"] { flex: 1 1 100px; min-width: 80px; max-width: 100%; accent-color: var(--c-info); }

  button {
    padding: 5px 10px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px; background: var(--c-bg-soft);
    cursor: pointer; transition: background 0.15s; color: var(--c-text);
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--c-info); outline-offset: 2px; }
  button.primary { background: #eaf4ff; border-color: var(--c-info); color: #1f4e79; }
  button.danger  { background: #fdedec; border-color: var(--c-err); color: #922b21; }
  button.success { background: #e8f8ed; border-color: var(--c-ok);  color: #1e8449; }
  button.preset  { font-size: 11px; padding: 3px 8px; }

  .hint-inline { font-size: 10px; color: var(--c-muted); font-style: italic; }
  .hint { font-size: 10px; color: var(--c-muted); margin-top: 4px;
    overflow-wrap: anywhere; word-break: break-word; }
  .records-badge {
    display: inline-block; padding: 2px 8px; border-radius: 10px;
    font-family: ui-monospace, monospace; font-size: 11px; font-weight: 700;
    background: #eaf4ff; color: #1f4e79;
  }

  .lc-row { display: flex; align-items: stretch; gap: 4px;
    padding: 10px 6px; background: var(--c-bg-soft); border-radius: 6px; margin: 4px 0; }
  .lc-node { flex: 1 1 0; min-width: 0; padding: 8px 6px;
    border: 2px solid var(--c-border); border-radius: 6px;
    background: #fff; text-align: center;
    display: flex; flex-direction: column; gap: 2px;
    transition: border-color 0.3s, box-shadow 0.3s; }
  .lc-node[data-state="closed"]  { border-color: #bbb; }
  .lc-node[data-state="upgrade"] { border-color: var(--c-warn); }
  .lc-node[data-state="open"]    { border-color: var(--c-ok); }
  .lc-node[data-active="true"]   { animation: lc-pulse 1.8s infinite; }
  @keyframes lc-pulse {
    0%,100% { box-shadow: 0 0 0 0 rgba(52,152,219,0.35); }
    60%     { box-shadow: 0 0 0 8px rgba(52,152,219,0); }
  }
  .lc-icon { font-size: 18px; line-height: 1; }
  .lc-title { font-family: ui-monospace, monospace; font-size: 12px; font-weight: 700; }
  .lc-desc { font-size: 10px; color: var(--c-muted); overflow-wrap: anywhere; }
  .lc-arrow { display: flex; flex-direction: column; align-items: center;
    justify-content: center; min-width: 60px;
    font-family: ui-monospace, monospace; font-size: 10px;
    color: var(--c-muted); text-align: center; }
  .lc-arrow-icon { font-size: 16px; color: #bbb; }

  /* Schema details */
  .schema-wrap { margin-top: 4px; border: 1px solid var(--c-border);
    border-radius: 5px; overflow: hidden; max-height: 340px; overflow-y: auto; }
  .schema-empty { padding: 12px; text-align: center; color: #aaa;
    font-style: italic; font-size: 11px; background: var(--c-bg-soft); }
  details.store-details { border-bottom: 1px solid var(--c-border); }
  details.store-details:last-child { border-bottom: none; }
  details.store-details summary {
    padding: 6px 10px; cursor: pointer; user-select: none;
    font-family: ui-monospace, monospace; font-size: 11px;
    display: flex; flex-wrap: wrap; gap: 8px; align-items: center;
    background: var(--c-bg-soft); list-style: none;
  }
  details.store-details summary::-webkit-details-marker { display: none; }
  details.store-details summary::before {
    content: "▸"; display: inline-block; transition: transform 0.15s;
    color: var(--c-muted); flex-shrink: 0; width: 12px;
  }
  details.store-details[open] summary::before { transform: rotate(90deg); }
  details.store-details summary .st-name { font-weight: 700; color: #1f4e79; }
  @media (prefers-color-scheme: dark) { details.store-details summary .st-name { color: #9fc4ff; } }
  details.store-details summary .st-key { color: var(--c-muted); }
  details.store-details summary .st-auto { color: var(--c-ok); }
  details.store-details summary .st-idx {
    padding: 1px 6px; border-radius: 8px;
    background: #eaf4ff; color: #1f4e79; font-size: 10px; font-weight: 700;
    margin-left: auto;
  }
  .store-body { padding: 6px 12px 10px 22px; font-family: ui-monospace, monospace;
    font-size: 11px; color: var(--c-text); }
  .store-meta { display: flex; flex-wrap: wrap; gap: 6px 14px; margin-bottom: 6px; }
  .store-meta b { color: var(--c-muted); font-weight: 600; }
  .idx-list { display: flex; flex-direction: column; gap: 3px; margin-top: 4px; }
  .idx-row {
    display: flex; flex-wrap: wrap; gap: 8px; align-items: center;
    padding: 3px 8px; background: var(--c-bg-soft);
    border-left: 3px solid var(--c-info); border-radius: 3px;
  }
  .idx-row .idx-name { font-weight: 700; color: #1f4e79; }
  @media (prefers-color-scheme: dark) { .idx-row .idx-name { color: #9fc4ff; } }
  .idx-row .idx-kp { color: var(--c-muted); }
  .badge-unique { background: #e8f8ed; color: #1e8449;
    padding: 0 5px; border-radius: 8px; font-size: 9px; font-weight: 700; }
  .badge-multi  { background: #fff3cd; color: #7d5a00;
    padding: 0 5px; border-radius: 8px; font-size: 9px; font-weight: 700; }
  .idx-empty { color: var(--c-muted); font-style: italic; font-size: 10px; }

  /* Real DBs */
  .db-wrap { max-height: 160px; overflow-y: auto;
    border: 1px solid var(--c-border); border-radius: 5px;
    background: var(--c-bg-soft);
    font-family: ui-monospace, monospace; font-size: 11px; }
  .db-empty { padding: 12px; text-align: center; color: #aaa; font-style: italic; }
  .db-item { display: flex; align-items: center; gap: 6px;
    padding: 4px 8px; border-bottom: 1px solid var(--c-border); min-width: 0; }
  .db-item:last-child { border-bottom: none; }
  .db-item code { color: #1f4e79; font-weight: 600;
    word-break: break-all; min-width: 0; flex: 1; }
  @media (prefers-color-scheme: dark) { .db-item code { color: #9fc4ff; } }
  .db-item.demo { background: #f0f8ff; }
  @media (prefers-color-scheme: dark) { .db-item.demo { background: #1a2a3f; } }
  .db-ver { padding: 0 6px; border-radius: 8px;
    background: #eaf4ff; color: #1f4e79; font-size: 10px;
    font-weight: 700; flex-shrink: 0; }
  .db-star { color: var(--c-warn); font-weight: 700; flex-shrink: 0; }

  /* Relationships */
  .rel-list { display: flex; flex-direction: column; gap: 3px;
    font-family: ui-monospace, monospace; font-size: 10.5px; }
  .rel-item {
    padding: 3px 8px; background: #eefaf0;
    border-left: 3px solid var(--c-ok); border-radius: 3px;
    color: #1e6b32;
  }
  @media (prefers-color-scheme: dark) { .rel-item { color: #b8e0b8; } }
  .rel-item b { color: #155724; }
  @media (prefers-color-scheme: dark) { .rel-item b { color: #cfe0cf; } }

  /* Timeline */
  .tl-flow { display: flex; flex-direction: column; gap: 4px;
    max-height: 500px; overflow-y: auto; padding: 2px 0; }
  .tl-empty { padding: 14px; text-align: center;
    color: #999; font-size: 11px; font-style: italic; }
  .tl-entry { padding: 5px 8px; background: var(--c-bg-panel);
    border: 1px solid var(--c-border);
    border-left: 3px solid var(--c-info); border-radius: 4px;
    font-family: ui-monospace, monospace; font-size: 11px;
    display: flex; flex-wrap: wrap; align-items: center; gap: 4px;
    animation: tl-appear 0.25s ease-out; min-width: 0; }
  @keyframes tl-appear {
    from { opacity: 0; transform: translateY(-3px); }
    to   { opacity: 1; transform: translateY(0); }
  }
  .tl-entry.upgraded { border-left-color: var(--c-ok); }
  .tl-entry.noop     { border-left-color: #ccc; }
  .tl-entry.err      { border-left-color: var(--c-err); }
  .tl-ver { background: #eaf4ff; color: #1f4e79;
    padding: 1px 7px; border-radius: 8px; font-weight: 700; flex-shrink: 0; }
  .tl-badge { font-size: 9px; padding: 1px 5px; border-radius: 6px;
    font-weight: 700; letter-spacing: 0.3px; flex-shrink: 0; }
  .tl-badge.up   { background: #e8f8ed; color: #1e8449; }
  .tl-badge.noop { background: #f0f0f0; color: #666; }
  .tl-badge.err  { background: #fdedec; color: #922b21; }
  .tl-badge.restore { background: #fff3cd; color: #7d5a00; }
  .chip { padding: 0 6px; border-radius: 8px;
    background: #eef4ff; color: #1f4e79; border: 1px solid #cfe0ff;
    font-size: 10px; word-break: break-all; }
  .chip.created { background: #e8f8ed; color: #1e8449; border-color: #b8e0b8; }
  .chip.deleted { background: #fdedec; color: #922b21; border-color: #e8b8b8;
    text-decoration: line-through; }
  .chip.idx { background: #e6f3ff; color: #1f4e79; border-color: #cfe0ff; }
  .tl-note { color: var(--c-muted); font-style: italic; font-size: 10px;
    flex-basis: 100%; overflow-wrap: anywhere; word-break: break-word; }

  .status { display: flex; align-items: center; gap: 6px;
    padding: 5px 9px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--c-bg-soft); border-left: 3px solid #ccc;
    color: var(--c-muted);
    transition: background 0.2s, border-color 0.2s, color 0.2s;
    min-height: 24px; overflow-wrap: anywhere; word-break: break-word; }
  .status.ok   { background: #e8f8ed; border-color: var(--c-ok);  color: #1e8449; }
  .status.err  { background: #fdedec; border-color: var(--c-err); color: #922b21; }
  .status.busy { background: #fff9e6; border-color: var(--c-warn); color: #7d5a00; }
  .status-icon { font-size: 11px; flex-shrink: 0; }

  .rename-steps { display: flex; gap: 4px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; }
  .rename-step { display: inline-flex; align-items: center; gap: 4px;
    padding: 3px 8px; border: 1px solid var(--c-border); border-radius: 4px;
    background: var(--c-bg-soft);
    font-family: ui-monospace, monospace; font-size: 10px;
    color: var(--c-muted); transition: all 0.2s;
    white-space: nowrap; flex-shrink: 0; }
  .rename-step[data-state="done"] { background: #e8f8ed; border-color: var(--c-ok);
    color: #1e8449; font-weight: 700; }
  .rename-step[data-state="active"] { background: #fff9e6; border-color: var(--c-warn);
    color: #7d5a00; animation: lc-pulse 1.8s infinite; }
  .rename-step[data-state="skip"] { background: #f0f0f0; color: #999; text-decoration: line-through; }
  .rename-step-num { display: inline-flex; align-items: center; justify-content: center;
    width: 14px; height: 14px; border-radius: 50%;
    background: #ccc; color: #fff; font-size: 9px; font-weight: 700; flex-shrink: 0; }
  .rename-step[data-state="done"] .rename-step-num { background: var(--c-ok); }
  .rename-step[data-state="active"] .rename-step-num { background: var(--c-warn); }
  .rename-arrow { color: #bbb; font-size: 12px; flex-shrink: 0; }

  .slider-row { display: flex; align-items: center; gap: 8px;
    margin: 6px 0; min-width: 0; }
  .slider-row input[type="range"] { flex: 1; }
  .slider-max-label { font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--c-muted); flex-shrink: 0; }
</style>

<h1>Глава 24.4: IdbDatabase — схема как контракт</h1>
<p class="subtitle">
  Метаданные stores и индексов. Загрузите схему библиотеки — 12 stores с индексами и связями.
  DevTools → Application → IndexedDB → ↻.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase" target="_blank" rel="noopener">MDN →</a>
</p>

<div class="banner">
  <div>
    <div class="banner-label">рабочая база</div>
    <div class="banner-name" id="sch-db-name">…</div>
    <div class="banner-meta" id="sch-db-meta">загрузка…</div>
  </div>
  <div class="banner-actions">
    <button id="sch-reset" class="danger" title="Закрыть соединение, удалить базу">↻ Сбросить базу</button>
  </div>
</div>

<div class="app-grid">

  <!-- Row 1: Lifecycle -->
  <div class="panel lifecycle-panel">
    <div class="panel-title">Жизненный цикл</div>
    <div class="lc-row">
      <div class="lc-node" id="lc-node-closed" data-state="closed" data-active="true">
        <div class="lc-icon">🔒</div><div class="lc-title">Закрыто</div>
        <div class="lc-desc">схема недоступна</div>
      </div>
      <div class="lc-arrow"><span class="lc-arrow-icon">→</span>open()</div>
      <div class="lc-node" id="lc-node-upgrade" data-state="upgrade" data-active="false">
        <div class="lc-icon">⚙️</div><div class="lc-title">Upgrade</div>
        <div class="lc-desc">мутации разрешены</div>
      </div>
      <div class="lc-arrow"><span class="lc-arrow-icon">→</span>commit</div>
      <div class="lc-node" id="lc-node-open" data-state="open" data-active="false">
        <div class="lc-icon">🔓</div><div class="lc-title">Открыто</div>
        <div class="lc-desc">схема заморожена</div>
      </div>
    </div>
    <div class="hint-inline">
      Схему — store'ы и индексы — можно менять только в фазе <b>Upgrade</b>.
    </div>
  </div>

  <!-- Row 2: State (3 panels) -->
  <div class="state-row">
    <div class="panel">
      <div class="panel-title">Схема сейчас <span class="hint-inline">кликните store — раскроются индексы</span></div>
      <div class="schema-wrap" id="sch-schema-wrap">
        <div class="schema-empty">Соединение не открыто.</div>
      </div>
    </div>
    <div class="panel">
      <div class="panel-title">Связи <span class="hint-inline">документированные FK</span></div>
      <div class="rel-list" id="sch-relations">
        <div class="idx-empty">загрузите Library schema — появятся связи</div>
      </div>
    </div>
    <div class="panel">
      <div class="panel-title">Базы в origin</div>
      <div class="db-wrap" id="sch-real-dbs">
        <div class="db-empty">загрузка…</div>
      </div>
      <div class="hint">★ — наша рабочая база. DevTools → Application → IndexedDB → ↻.</div>
    </div>
  </div>

  <!-- Row 3: Mutations -->
  <div class="panel col-mutations">
    <h2 class="panel-title"><span class="exp-num">1</span>Управление схемой</h2>

    <div class="anno">
      <b>Зачем.</b> В IndexedDB нет <code>ALTER TABLE</code> и <code>CREATE INDEX</code> вне upgrade.
      Изменения — только в <code>onupgradeneeded</code>, только вперёд.
    </div>

    <div class="form-row" style="margin-bottom:2px">
      <span class="hint-inline">preset:</span>
      <button id="sch-load-library" class="success" title="12 stores + индексы + M:N связи">📚 Library schema</button>
      <button id="sch-preset-minimal" class="preset" title="authors + books">+ minimal</button>
    </div>

    <div class="panel-title" style="font-size:12px;margin-top:8px">Добавить store</div>
    <div class="form-row">
      <input type="text" id="sch-new-name" placeholder="store name">
      <input type="text" id="sch-new-key" class="key-input" list="keypath-examples" placeholder="keyPath">
      <label class="check" title="autoIncrement — ключ генерируется автоматически">
        <input type="checkbox" id="sch-new-auto"> auto
      </label>
      <button id="sch-add-store" class="primary">+ Add</button>
    </div>

    <datalist id="keypath-examples">
      <option value="id"><option value="isbn"><option value="name">
      <option value="slug"><option value="uuid"><option value="email">
    </datalist>

    <div class="hint">
      keyPath: одно поле (<code>id</code>) или составной (<code>book_isbn,genre_id</code>).
      Тип ключа: number, string, date, binary, array.
    </div>

    <div class="panel-title" style="font-size:12px;margin-top:10px">Добавить индекс</div>
    <div class="form-row">
      <select id="sch-idx-store" title="Store, к которому добавляем индекс">
        <option value="">— выберите store —</option>
      </select>
      <input type="text" id="sch-idx-name" placeholder="index name">
    </div>
    <div class="form-row">
      <input type="text" id="sch-idx-key" class="key-input" placeholder="keyPath">
      <label class="check"><input type="checkbox" id="sch-idx-unique"> unique</label>
      <label class="check"><input type="checkbox" id="sch-idx-multi"> multi</label>
      <button id="sch-add-index" class="primary">+ Add index</button>
    </div>
    <div class="hint">
      unique — запрещает дубликаты значений. multi — индекс по массиву (каждый элемент = отдельный ключ).
    </div>

    <div class="panel-title" style="font-size:12px;margin-top:10px">Удалить store</div>
    <div class="form-row">
      <select id="sch-del-select"><option value="">— нет store'ов —</option></select>
      <button id="sch-del-store" class="danger">− Delete</button>
    </div>

    <div id="sch-status1" class="status"><span class="status-icon">—</span><span>готово</span></div>
  </div>

  <!-- Row 3: Constraints -->
  <div class="col col-constraints">
    <div class="panel">
      <h2 class="panel-title"><span class="exp-num">2</span>Замок схемы</h2>
      <div class="anno">
        <b>Зачем.</b> <code>createObjectStore</code> и <code>deleteObjectStore</code> работают
        <b>только</b> внутри <code>onupgradeneeded</code>. Вне его — <code>InvalidStateError</code>.
      </div>
      <div class="form-row">
        <button id="sch-try-create">try createObjectStore</button>
        <button id="sch-try-delete">try deleteObjectStore</button>
      </div>
      <div id="sch-status2" class="status"><span class="status-icon">—</span><span>ожидание</span></div>
    </div>

    <div class="panel">
      <h2 class="panel-title"><span class="exp-num">3</span>Монотонность версий</h2>
      <div class="anno">
        <b>Зачем.</b> Версии <b>монотонны</b>. Открыть базу как <code>v(N−1)</code> нельзя —
        <code>VersionError</code>.
      </div>
      <div class="slider-row">
        <input type="range" id="sch-try-v-slider" min="1" max="5" value="1">
        <span class="slider-max-label" id="sch-try-v-max">max: 5</span>
      </div>
      <div class="form-row">
        <input type="text" id="sch-try-v" value="1" class="key-input">
        <button id="sch-try-v-btn" class="danger">open v?</button>
      </div>
      <div id="sch-status3" class="status"><span class="status-icon">—</span><span>ожидание</span></div>
    </div>

    <div class="panel">
      <h2 class="panel-title" id="rename-title"><span class="exp-num">4</span>Rename store — 3 шага</h2>
      <div class="anno">
        <b>Зачем.</b> Миграция — три версии: <b>v+1</b> создать target, <b>v+2</b> скопировать данные,
        <b>v+3</b> удалить source. Если в source нет записей — шаг 2 пропускается.
      </div>
      <div class="form-row">
        <select id="sch-rename-source"><option value="">— нет store'ов —</option></select>
        <span class="rename-arrow">→</span>
        <input type="text" id="sch-rename-target" placeholder="new name">
      </div>
      <div class="form-row">
        <button id="sch-rename-add-record">+ sample</button>
        <span class="records-badge" id="sch-rename-count">records: 0</span>
        <button id="sch-rename-refresh-count">↻</button>
      </div>
      <div class="rename-steps">
        <div class="rename-step" id="rn-step1" data-state="pending">
          <span class="rename-step-num">1</span><span>Создать</span>
        </div>
        <span class="rename-arrow">→</span>
        <div class="rename-step" id="rn-step2" data-state="pending">
          <span class="rename-step-num">2</span><span>Копировать</span>
        </div>
        <span class="rename-arrow">→</span>
        <div class="rename-step" id="rn-step3" data-state="pending">
          <span class="rename-step-num">3</span><span>Удалить</span>
        </div>
      </div>
      <div class="form-row">
        <button id="sch-rename-run" class="primary">▶ Выполнить шаг</button>
        <button id="sch-rename-cancel">× Отмена</button>
      </div>
      <div class="hint-inline" id="sch-rename-hint">Введите имя target и нажмите «Выполнить шаг».</div>
      <div id="sch-status4" class="status"><span class="status-icon">—</span><span>ожидание</span></div>
    </div>
  </div>

  <!-- Row 3: History -->
  <div class="col col-history">
    <div class="panel">
      <div class="panel-title">История версий <span class="hint-inline">последние 30</span></div>
      <div class="tl-flow" id="sch-timeline">
        <div class="tl-empty">История пуста</div>
      </div>
    </div>
  </div>

</div>
"#;

// ============================================================
//                       МОДЕЛЬ
// ============================================================

struct VersionRecord {
    version: u32,
    plan: SchemaPlan,
    created: Vec<String>,
    deleted: Vec<String>,
    
    idx_created: Vec<String>,
    idx_deleted: Vec<String>,
    upgraded: bool,
    restored: bool,
    error: Option<String>,
    note: Option<String>,
}

#[derive(Clone, Debug)]
struct RenameWorkflow {
    source: String,
    target: String,
    step: u8,
    records_copied: u32,
    step2_skipped: bool,
}

#[derive(Clone)]
struct Ui {
    db_name: web_sys::Element,
    db_meta: web_sys::Element,
    schema_wrap: web_sys::Element,
    relations: web_sys::Element,
    timeline: web_sys::Element,
    real_dbs: web_sys::Element,
    status1: web_sys::Element,
    status2: web_sys::Element,
    status3: web_sys::Element,
    status4: web_sys::Element,
    lc_closed: web_sys::Element,
    lc_upgrade: web_sys::Element,
    lc_open: web_sys::Element,
    rename_title: web_sys::Element,
    del_select: HtmlSelectElement,
    idx_store_select: HtmlSelectElement,
    rename_source: HtmlSelectElement,
    new_name: HtmlInputElement,
    new_key: HtmlInputElement,
    new_auto: HtmlInputElement,
    idx_name: HtmlInputElement,
    idx_key: HtmlInputElement,
    idx_unique: HtmlInputElement,
    idx_multi: HtmlInputElement,
    try_v: HtmlInputElement,
    try_v_slider: HtmlInputElement,
    try_v_max: web_sys::Element,
    rename_target: HtmlInputElement,
    rename_count: web_sys::Element,
    rn_step1: web_sys::Element,
    rn_step2: web_sys::Element,
    rn_step3: web_sys::Element,
    rename_hint: web_sys::Element,
    rename_run: HtmlButtonElement,
    rename_cancel: HtmlButtonElement,
}

struct App {
    current_db: Rc<RefCell<Option<Database>>>,
    current_plan: Rc<RefCell<SchemaPlan>>,
    history: Rc<RefCell<Vec<VersionRecord>>>,
    rename: Rc<RefCell<Option<RenameWorkflow>>>,
    busy: Rc<RefCell<bool>>,
    all_buttons: Rc<Vec<HtmlButtonElement>>,
    ui: Ui,
}

impl App {
    fn set_status(el: &web_sys::Element, kind: &str, icon: &str, msg: &str) {
        el.set_class_name(&format_str(format_args!("status {kind}")));
        el.set_inner_html(&format_str(format_args!(
            "<span class=\"status-icon\">{icon}</span><span>{}</span>", esc(msg))));
    }
    fn status_info(el: &web_sys::Element, msg: &str) { Self::set_status(el, "busy", "…", msg); }
    fn status_ok(el: &web_sys::Element, msg: &str)   { Self::set_status(el, "ok", "✓", msg); }
    fn status_err(el: &web_sys::Element, msg: &str)  { Self::set_status(el, "err", "✗", msg); }
    fn status_idle(el: &web_sys::Element, msg: &str) { Self::set_status(el, "", "—", msg); }

    fn set_lifecycle(&self, state: &str) {
        let nodes: [(&str, &web_sys::Element); 3] = [
            ("closed",  &self.ui.lc_closed),
            ("upgrade", &self.ui.lc_upgrade),
            ("open",    &self.ui.lc_open),
        ];
        for (name, el) in nodes {
            let active = if name == state { "true" } else { "false" };
            let _ = el.set_attribute("data-active", active);
        }
    }

    fn refresh_lifecycle_from_state(&self) {
        let db = self.current_db.borrow();
        match db.as_ref() {
            Some(d) if !d.is_closed() => self.set_lifecycle("open"),
            _ => self.set_lifecycle("closed"),
        }
    }

    fn refresh_banner(&self) {
        self.ui.db_name.set_text_content(Some(DEMO_DB));
        let (ver_text, stores_count, idx_count) = {
            let db = self.current_db.borrow();
            match db.as_ref() {
                Some(d) if !d.is_closed() => {
                    let plan = self.current_plan.borrow();
                    let idx: usize = plan.stores.iter().map(|s| s.indexes.len()).sum();
                    (format_str(format_args!("v{}", d.version())), d.store_names().len(), idx)
                }
                Some(_) => ("закрыто".to_string(), 0, 0),
                None => ("не открыто".to_string(), 0, 0),
            }
        };
        self.ui.db_meta.set_text_content(Some(&format_str(format_args!(
            "версия: {ver_text} · stores: {stores_count} · indexes: {idx_count}",
        ))));
    }

    fn render_schema_table(&self) {
        let db = self.current_db.borrow();
        let Some(d) = db.as_ref().filter(|d| !d.is_closed()) else {
            self.ui.schema_wrap.set_inner_html(
                "<div class=\"schema-empty\">Соединение не открыто.</div>");
            return;
        };
        let mut names = d.store_names();
        names.sort();
        if names.is_empty() {
            self.ui.schema_wrap.set_inner_html(
                "<div class=\"schema-empty\">База пуста — нет ни одного store.</div>");
            return;
        }
        let plan = self.current_plan.borrow();
        let mut html = String::with_capacity(1024);
        for name in &names {
            let spec = plan.stores.iter().find(|s| &s.name == name);
            let (key_str, auto_str, idx_count) = match spec {
                Some(s) => {
                    let k = s.key_path.clone().unwrap_or_else(|| "—".to_string());
                    let a = if s.auto_increment { "✓" } else { "—" };
                    (k, a, s.indexes.len())
                }
                None => ("?".to_string(), "?", 0),
            };
            html.push_str("<details class=\"store-details\"><summary>");
            html.push_str(&format_str(format_args!(
                "<span class=\"st-name\">{}</span>\
                 <span class=\"st-key\">key: {}</span>\
                 <span class=\"st-auto\">auto: {}</span>",
                esc(name), esc(&key_str), auto_str)));
            html.push_str(&format_str(format_args!(
                "<span class=\"st-idx\">{} idx</span></summary>", idx_count)));

            // Body.
            html.push_str("<div class=\"store-body\">");
            html.push_str(&format_str(format_args!(
                "<div class=\"store-meta\"><b>keyPath:</b> <code>{}</code>\
                 <b>autoIncrement:</b> <code>{}</code></div>",
                esc(&key_str),
                if auto_str == "✓" { "true" } else { "false" },
            )));
            html.push_str("<div><b>indexes:</b>");
            if let Some(s) = spec {
                if s.indexes.is_empty() {
                    html.push_str("<div class=\"idx-empty\">нет индексов</div>");
                } else {
                    html.push_str("<div class=\"idx-list\">");
                    for i in &s.indexes {
                        html.push_str("<div class=\"idx-row\">");
                        html.push_str(&format_str(format_args!(
                            "<span class=\"idx-name\">{}</span>", esc(&i.name))));
                        html.push_str(&format_str(format_args!(
                            "<span class=\"idx-kp\">({})</span>", esc(&i.key_path))));
                        if i.unique {
                            html.push_str("<span class=\"badge-unique\">UNIQUE</span>");
                        }
                        if i.multi_entry {
                            html.push_str("<span class=\"badge-multi\">MULTI</span>");
                        }
                        html.push_str("</div>");
                    }
                    html.push_str("</div>");
                }
            } else {
                html.push_str("<div class=\"idx-empty\">метаданные неизвестны</div>");
            }
            html.push_str("</div>");
            html.push_str("</div>");
            html.push_str("</details>");
        }
        self.ui.schema_wrap.set_inner_html(&html);
    }

    fn render_relations(&self) {
        // Статичный список связей — если загружена библиотека.
        let has_library = {
            let plan = self.current_plan.borrow();
            plan.stores.iter().any(|s| s.name == "book_genres")
        };
        if !has_library {
            self.ui.relations.set_inner_html(
                "<div class=\"idx-empty\">загрузите Library schema — появятся связи</div>");
            return;
        }
        let rels: [(&str, &str, &str); 11] = [
            ("books.author_id",     "authors.id",       "1:N"),
            ("books.publisher_id",  "publishers.id",    "1:N"),
            ("book_genres.book_isbn","books.isbn",      "M:N"),
            ("book_genres.genre_id","genres.id",        "M:N"),
            ("book_tags.book_isbn", "books.isbn",       "M:N"),
            ("book_tags.tag_id",    "tags.id",          "M:N"),
            ("loans.book_isbn",     "books.isbn",       "1:N"),
            ("loans.reader_id",     "readers.id",       "1:N"),
            ("reviews.book_isbn",   "books.isbn",       "1:N"),
            ("reviews.reader_id",   "readers.id",       "1:N"),
            ("covers.book_isbn",    "books.isbn",       "1:1"),
        ];
        let mut html = String::with_capacity(1024);
        for (a, b, k) in rels {
            html.push_str(&format_str(format_args!(
                "<div class=\"rel-item\"><b>{a}</b> → {b} <em>({k})</em></div>"
            )));
        }
        self.ui.relations.set_inner_html(&html);
    }

    fn render_timeline(&self) {
        let history = self.history.borrow();
        if history.is_empty() {
            self.ui.timeline.set_inner_html("<div class=\"tl-empty\">История пуста</div>");
            return;
        }
        let start = history.len().saturating_sub(TIMELINE_LIMIT);
        let mut html = String::with_capacity(2048);
        for r in history.iter().skip(start) {
            let (nc, bc, bt) = if r.error.is_some() { ("tl-entry err", "tl-badge err", "ERR") }
                else if r.restored { ("tl-entry upgraded", "tl-badge restore", "RESTORED") }
                else if !r.upgraded { ("tl-entry noop", "tl-badge noop", "NOOP") }
                else if r.created.is_empty() && r.deleted.is_empty()
                     && r.idx_created.is_empty() && r.idx_deleted.is_empty() {
                    ("tl-entry noop", "tl-badge noop", "IDEMPOTENT")
                } else { ("tl-entry upgraded", "tl-badge up", "UPGRADED") };
            html.push_str(&format_str(format_args!(
                "<div class=\"{nc}\"><span class=\"tl-ver\">v{}</span>\
                 <span class=\"{bc}\">{bt}</span>", r.version)));
            if let Some(err) = &r.error {
                html.push_str(&format_str(format_args!(
                    "<span class=\"chip deleted\">{}</span>", esc(err))));
            } else {
                for spec in &r.plan.stores {
                    let cls: &str = if r.created.contains(&spec.name) { "chip created" }
                                    else if r.deleted.contains(&spec.name) { "chip deleted" }
                                    else { "chip" };
                    html.push_str(&format_str(format_args!(
                        "<span class=\"{cls}\">{}</span>", esc(&spec.name))));
                }
            }
            if !r.idx_created.is_empty() {
                for i in &r.idx_created {
                    html.push_str(&format_str(format_args!(
                        "<span class=\"chip idx\">+{}</span>", esc(i))));
                }
            }
            if !r.idx_deleted.is_empty() {
                for i in &r.idx_deleted {
                    html.push_str(&format_str(format_args!(
                        "<span class=\"chip deleted\">−{}</span>", esc(i))));
                }
            }
            if let Some(note) = &r.note {
                html.push_str(&format_str(format_args!(
                    "<span class=\"tl-note\">{}</span>", esc(note))));
            }
            html.push_str("</div>");
        }
        self.ui.timeline.set_inner_html(&html);
    }

    fn refresh_selects(&self) {
        let stores = {
            let db = self.current_db.borrow();
            match db.as_ref() {
                Some(d) if !d.is_closed() => {
                    let mut n = d.store_names(); n.sort(); n
                }
                _ => Vec::new(),
            }
        };
        populate_select(&self.ui.del_select, &stores, "— нет store'ов —");
        populate_select(&self.ui.idx_store_select, &stores, "— выберите store —");
        populate_select(&self.ui.rename_source, &stores, "— нет store'ов —");
        let has = !stores.is_empty();
        if let Ok(b) = get_button("sch-del-store") { b.set_disabled(!has); }
        if let Ok(b) = get_button("sch-rename-add-record") { b.set_disabled(!has); }
    }

    fn refresh_rename_ui(&self) {
        let wf = self.rename.borrow();
        let step = wf.as_ref().map(|w| w.step).unwrap_or(0);
        let active = wf.is_some();
        let s2s = wf.as_ref().map(|w| w.step2_skipped).unwrap_or(false);
        let copied = wf.as_ref().map(|w| w.records_copied).unwrap_or(0);

        set_step_state(&self.ui.rn_step1, if !active { "pending" }
            else if step >= 1 { "done" } else { "active" });
        set_step_state(&self.ui.rn_step2, if !active { "pending" }
            else if s2s { "skip" } else if step >= 2 { "done" }
            else if step == 1 { "active" } else { "pending" });
        set_step_state(&self.ui.rn_step3, if !active { "pending" }
            else if step >= 3 { "done" } else if step == 2 { "active" } else { "pending" });

        let (sfx, lbl, hint, can) : (String, &str, String, bool) = match wf.as_ref() {
            None => (String::new(), "▶ Выполнить шаг",
                "Введите имя target и нажмите «Выполнить шаг».".into(), true),
            Some(w) if w.step == 0 => (" · шаг 1/3".into(), "▶ Шаг 1: создать target",
                format_str(format_args!("Создадим store «{}»", w.target)), true),
            Some(w) if w.step == 1 => (" · шаг 2/3".into(), "▶ Шаг 2: копировать данные",
                format_str(format_args!("Копирование «{}» → «{}»", w.source, w.target)), true),
            Some(w) if w.step == 2 => (" · шаг 3/3".into(), "▶ Шаг 3: удалить source",
                format_str(format_args!("Удалим «{}»", w.source)), true),
            Some(_) => (" · готово".into(), "✓ Готово",
                format!("Скопировано записей: {copied}. Workflow завершён."), false),
        };
        let title_html = format_str(format_args!(
            "<span class=\"exp-num\">4</span>Rename store — 3 шага{}", esc(&sfx)));
        self.ui.rename_title.set_inner_html(&title_html);
        self.ui.rename_run.set_text_content(Some(lbl));
        self.ui.rename_hint.set_text_content(Some(&hint));
        self.ui.rename_run.set_disabled(!can);
        self.ui.rename_cancel.set_disabled(wf.is_none());
    }

    fn refresh_slider(&self) {
        let current = {
            let db = self.current_db.borrow();
            match db.as_ref() {
                Some(d) if !d.is_closed() => d.version(),
                _ => {
                    let h = self.history.borrow();
                    h.iter().map(|r| r.version).max().unwrap_or(1)
                }
            }
        };
        let max_v = current.max(5);
        self.ui.try_v_slider.set_min("1");
        self.ui.try_v_slider.set_max(&max_v.to_string());
        self.ui.try_v_max.set_text_content(Some(&format_str(format_args!("max: {max_v}"))));
    }

    fn refresh_all(&self) {
        self.refresh_banner();
        self.render_schema_table();
        self.render_relations();
        self.render_timeline();
        self.refresh_lifecycle_from_state();
        self.refresh_selects();
        self.refresh_rename_ui();
        self.refresh_slider();
    }

    fn set_buttons_disabled(&self, disabled: bool) {
        for b in self.all_buttons.iter() { b.set_disabled(disabled); }
    }
}

// ============================================================
//                  LIBRARY SCHEMA
// ============================================================

/// Реальная схема домашней библиотеки: 12 stores, M:N, 1:N, 1:1.
fn library_schema() -> SchemaPlan {
    use IndexSpec as I;
    SchemaPlan::new()
        .add(StoreSpec::with_key("authors", "id")
            .indexes(vec![I::new("by_name", "name")]))
        .add(StoreSpec::with_key("publishers", "id")
            .indexes(vec![I::new("by_name", "name")]))
        .add(StoreSpec::with_key("books", "isbn")
            .indexes(vec![
                I::new("by_author", "author_id"),
                I::new("by_publisher", "publisher_id"),
                I::new("by_year", "year"),
                I::new("by_title", "title"),
            ]))
        .add(StoreSpec::with_key("genres", "id")
            .indexes(vec![I::new("by_name", "name")]))
        .add(StoreSpec::with_key("book_genres", "book_isbn,genre_id")
            .indexes(vec![
                I::new("by_book", "book_isbn"),
                I::new("by_genre", "genre_id"),
            ]))
        .add(StoreSpec::with_key("tags", "id")
            .indexes(vec![I::new("by_label", "label")]))
        .add(StoreSpec::with_key("book_tags", "book_isbn,tag_id")
            .indexes(vec![
                I::new("by_book", "book_isbn"),
                I::new("by_tag", "tag_id"),
            ]))
        .add(StoreSpec::with_key("readers", "id")
            .indexes(vec![I::new("by_email", "email").unique(true)]))
        .add(StoreSpec::with_key("loans", "id")
            .indexes(vec![
                I::new("by_book", "book_isbn"),
                I::new("by_reader", "reader_id"),
                I::new("by_due", "due_at"),
            ]))
        .add(StoreSpec::with_key("reviews", "id")
            .indexes(vec![
                I::new("by_book", "book_isbn"),
                I::new("by_reader", "reader_id"),
            ]))
        .add(StoreSpec::with_key("covers", "id")
            .indexes(vec![I::new("by_book", "book_isbn").unique(true)]))
        .add(StoreSpec::with_auto_key("audit_log")
            .indexes(vec![I::new("by_ts", "ts")]))
}

// ============================================================
//                        RUN
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        db_name: get_el("sch-db-name")?,
        db_meta: get_el("sch-db-meta")?,
        schema_wrap: get_el("sch-schema-wrap")?,
        relations: get_el("sch-relations")?,
        timeline: get_el("sch-timeline")?,
        real_dbs: get_el("sch-real-dbs")?,
        status1: get_el("sch-status1")?,
        status2: get_el("sch-status2")?,
        status3: get_el("sch-status3")?,
        status4: get_el("sch-status4")?,
        lc_closed: get_el("lc-node-closed")?,
        lc_upgrade: get_el("lc-node-upgrade")?,
        lc_open: get_el("lc-node-open")?,
        rename_title: get_el("rename-title")?,
        del_select: get_select("sch-del-select")?,
        idx_store_select: get_select("sch-idx-store")?,
        rename_source: get_select("sch-rename-source")?,
        new_name: get_input("sch-new-name")?,
        new_key: get_input("sch-new-key")?,
        new_auto: get_input("sch-new-auto")?,
        idx_name: get_input("sch-idx-name")?,
        idx_key: get_input("sch-idx-key")?,
        idx_unique: get_input("sch-idx-unique")?,
        idx_multi: get_input("sch-idx-multi")?,
        try_v: get_input("sch-try-v")?,
        try_v_slider: get_input("sch-try-v-slider")?,
        try_v_max: get_el("sch-try-v-max")?,
        rename_target: get_input("sch-rename-target")?,
        rename_count: get_el("sch-rename-count")?,
        rn_step1: get_el("rn-step1")?,
        rn_step2: get_el("rn-step2")?,
        rn_step3: get_el("rn-step3")?,
        rename_hint: get_el("sch-rename-hint")?,
        rename_run: get_button("sch-rename-run")?,
        rename_cancel: get_button("sch-rename-cancel")?,
    };

    let btn_ids = [
        "sch-reset", "sch-add-store", "sch-add-index",
        "sch-load-library", "sch-preset-minimal",
        "sch-del-store", "sch-try-create", "sch-try-delete",
        "sch-try-v-btn",
        "sch-rename-run", "sch-rename-cancel",
        "sch-rename-add-record", "sch-rename-refresh-count",
    ];
    let mut all_buttons_vec: Vec<HtmlButtonElement> = Vec::with_capacity(btn_ids.len());
    for id in btn_ids { all_buttons_vec.push(get_button(id)?); }

    let app = Rc::new(App {
        current_db: Rc::new(RefCell::new(None)),
        current_plan: Rc::new(RefCell::new(SchemaPlan::new())),
        history: Rc::new(RefCell::new(Vec::new())),
        rename: Rc::new(RefCell::new(None)),
        busy: Rc::new(RefCell::new(false)),
        all_buttons: Rc::new(all_buttons_vec),
        ui,
    });

    app.refresh_all();

    {
        let app_c = app.clone();
        spawn_local(async move {
            restore_from_existing(&app_c).await;
            refresh_real_dbs(&app_c).await;
        });
    }

    setup_auto_toggle(&app)?;
    setup_slider_sync(&app)?;
    setup_source_change(&app)?;
    setup_db_watcher(&app)?;

    // ---------- RESET ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-reset")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            *app_in.busy.borrow_mut() = true;
            app_in.set_buttons_disabled(true);
            App::status_info(&app_in.ui.status1, "сброс…");
            app_in.set_lifecycle("closed");
            spawn_local(async move {
                if let Some(old) = app_in.current_db.borrow_mut().take() { old.close(); }
                let _ = idb_factory::delete_by_name(DEMO_DB).await;
                *app_in.current_plan.borrow_mut() = SchemaPlan::new();
                app_in.history.borrow_mut().clear();
                *app_in.rename.borrow_mut() = None;
                app_in.refresh_all();
                refresh_real_dbs(&app_in).await;
                App::status_ok(&app_in.ui.status1, "база удалена");
                App::status_idle(&app_in.ui.status2, "ожидание");
                App::status_idle(&app_in.ui.status3, "ожидание");
                App::status_idle(&app_in.ui.status4, "ожидание");
                *app_in.busy.borrow_mut() = false;
                app_in.set_buttons_disabled(false);
            });
        });
    }

    // ---------- PRESET: Library ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-load-library")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let new_plan = library_schema();
            let stores = new_plan.stores.len();
            App::status_info(&app_in.ui.status1,
                &format_str(format_args!("загрузка Library schema ({stores} stores)…")));
            app_in.set_lifecycle("upgrade");
            spawn_local(async move {
                gloo_timers::future::TimeoutFuture::new(0).await;
                let next_v = next_version(&app_in).await;
                let success = run_open(app_in.clone(), next_v, new_plan, None).await;
                if success {
                    App::status_ok(&app_in.ui.status1,
                        &format_str(format_args!(
                            "Library schema загружена — v{next_v}, {stores} stores. DevTools → ↻"
                        )));
                }
            });
        });
    }
    // ---------- PRESET: minimal ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-preset-minimal")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let new_plan = SchemaPlan::new()
                .add(StoreSpec::with_key("authors", "id"))
                .add(StoreSpec::with_key("books", "isbn")
                    .indexes(vec![IndexSpec::new("by_author", "author_id")]));
            App::status_info(&app_in.ui.status1, "загрузка minimal schema…");
            app_in.set_lifecycle("upgrade");
            spawn_local(async move {
                gloo_timers::future::TimeoutFuture::new(0).await;
                let next_v = next_version(&app_in).await;
                let success = run_open(app_in.clone(), next_v, new_plan, None).await;
                if success {
                    App::status_ok(&app_in.ui.status1,
                        &format_str(format_args!("minimal schema — v{next_v}")));
                }
            });
        });
    }

    // ---------- ADD STORE ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-add-store")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let name = app_in.ui.new_name.value().trim().to_string();
            if name.is_empty() {
                App::status_err(&app_in.ui.status1, "введите имя store");
                let _ = app_in.ui.new_name.focus(); return;
            }
            let auto = app_in.ui.new_auto.checked();
            let key_raw = app_in.ui.new_key.value().trim().to_string();
            let spec = if auto {
                StoreSpec::with_auto_key(&name)
            } else {
                if key_raw.is_empty() {
                    App::status_err(&app_in.ui.status1, "укажите keyPath или отметьте auto");
                    let _ = app_in.ui.new_key.focus(); return;
                }
                StoreSpec::with_key(&name, &key_raw)
            };
            let new_plan = plan_with_store(&app_in.current_plan.borrow().clone(), spec.clone());
            App::status_info(&app_in.ui.status1,
                &format_str(format_args!("добавление «{name}»…")));
            app_in.set_lifecycle("upgrade");
            spawn_local(async move {
                gloo_timers::future::TimeoutFuture::new(0).await;
                let next_v = next_version(&app_in).await;
                let ok = run_open(app_in.clone(), next_v, new_plan, None).await;
                if ok {
                    App::status_ok(&app_in.ui.status1,
                        &format_str(format_args!("«{name}» — v{next_v}")));
                    app_in.ui.new_name.set_value("");
                    app_in.ui.new_key.set_value("");
                    app_in.ui.new_auto.set_checked(false);
                    let _ = app_in.ui.new_name.focus();
                }
            });
        });
    }

    // ---------- ADD INDEX ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-add-index")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let store = app_in.ui.idx_store_select.value();
            if store.is_empty() {
                App::status_err(&app_in.ui.status1, "выберите store для индекса"); return;
            }
            let name = app_in.ui.idx_name.value().trim().to_string();
            if name.is_empty() {
                App::status_err(&app_in.ui.status1, "введите имя индекса"); return;
            }
            let kp = app_in.ui.idx_key.value().trim().to_string();
            if kp.is_empty() {
                App::status_err(&app_in.ui.status1, "введите keyPath индекса"); return;
            }
            let uniq = app_in.ui.idx_unique.checked();
            let multi = app_in.ui.idx_multi.checked();
            let mut idx = IndexSpec::new(&name, &kp);
            idx.unique = uniq;
            idx.multi_entry = multi;

            // Клонируем план, добавляем индекс к store.
            let mut new_plan = app_in.current_plan.borrow().clone();
            let mut found = false;
            for s in new_plan.stores.iter_mut() {
                if s.name == store {
                    // Если индекс с таким именем уже есть — заменим.
                    if let Some(pos) = s.indexes.iter().position(|i| i.name == name) {
                        s.indexes[pos] = idx.clone();
                    } else {
                        s.indexes.push(idx.clone());
                    }
                    found = true;
                    break;
                }
            }
            if !found {
                App::status_err(&app_in.ui.status1,
                    &format_str(format_args!("store «{store}» не найден в плане")));
                return;
            }

            App::status_info(&app_in.ui.status1,
                &format_str(format_args!("добавление индекса {store}.{name}…")));
            app_in.set_lifecycle("upgrade");
            spawn_local(async move {
                gloo_timers::future::TimeoutFuture::new(0).await;
                let next_v = next_version(&app_in).await;
                let ok = run_open(app_in.clone(), next_v, new_plan, None).await;
                if ok {
                    App::status_ok(&app_in.ui.status1,
                        &format_str(format_args!(
                            "{store}.{name} добавлен — v{next_v}. DevTools → ↻"
                        )));
                    app_in.ui.idx_name.set_value("");
                    app_in.ui.idx_key.set_value("");
                    app_in.ui.idx_unique.set_checked(false);
                    app_in.ui.idx_multi.set_checked(false);
                }
            });
        });
    }

    // ---------- DELETE STORE ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-del-store")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let name = app_in.ui.del_select.value();
            if name.is_empty() {
                App::status_err(&app_in.ui.status1, "выберите store"); return;
            }
            let new_plan = plan_without_store(&app_in.current_plan.borrow().clone(), &name);
            App::status_info(&app_in.ui.status1,
                &format_str(format_args!("удаление «{name}»…")));
            app_in.set_lifecycle("upgrade");
            spawn_local(async move {
                gloo_timers::future::TimeoutFuture::new(0).await;
                let next_v = next_version(&app_in).await;
                let ok = run_open(app_in.clone(), next_v, new_plan, None).await;
                if ok {
                    App::status_ok(&app_in.ui.status1,
                        &format_str(format_args!("«{name}» удалён — v{next_v}")));
                }
            });
        });
    }

    // ---------- SCHEMA LOCK ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-try-create")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let db_guard = app_in.current_db.borrow();
            let Some(db) = db_guard.as_ref() else {
                App::status_err(&app_in.ui.status2, "сначала создайте store"); return;
            };
            let spec = StoreSpec::with_key("late_store", "id");
            match idb_schema::try_create_on_open(db, &spec) {
                Ok(()) => App::status_err(&app_in.ui.status2, "неожиданно: сработал вне upgrade"),
                Err(e) => {
                    let n = error_name(&e);
                    let msg = if n == "InvalidStateError" {
                        "InvalidStateError ✓ — createObjectStore запрещён вне onupgradeneeded".to_string()
                    } else { format_str(format_args!("Ошибка: {n}")) };
                    App::status_ok(&app_in.ui.status2, &msg);
                }
            }
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("sch-try-delete")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let db_guard = app_in.current_db.borrow();
            let Some(db) = db_guard.as_ref() else {
                App::status_err(&app_in.ui.status2, "сначала создайте store"); return;
            };
            let names = db.store_names();
            let Some(target) = names.first() else {
                App::status_err(&app_in.ui.status2, "в базе нет store'ов"); return;
            };
            let target = target.clone();
            match idb_schema::try_delete_on_open(db, &target) {
                Ok(()) => App::status_err(&app_in.ui.status2, "неожиданно: сработал вне upgrade"),
                Err(e) => {
                    let n = error_name(&e);
                    let msg = if n == "InvalidStateError" {
                        "InvalidStateError ✓ — deleteObjectStore запрещён вне onupgradeneeded".to_string()
                    } else { format_str(format_args!("Ошибка: {n}")) };
                    App::status_ok(&app_in.ui.status2, &msg);
                }
            }
        });
    }

    // ---------- MONOTONICITY ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-try-v-btn")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let raw = app_in.ui.try_v.value();
            let target_v: u32 = match raw.trim().parse() {
                Ok(v) if v >= 1 && v <= 1_000_000 => v,
                Ok(_) => {
                    App::status_err(&app_in.ui.status3, "слишком большая версия"); return;
                }
                _ => {
                    App::status_err(&app_in.ui.status3, "введите целое ≥ 1"); return;
                }
            };
            let plan = app_in.current_plan.borrow().clone();
            App::status_info(&app_in.ui.status3,
                &format_str(format_args!("попытка открыть v{target_v}…")));
            app_in.set_lifecycle("upgrade");
            spawn_local(async move {
                gloo_timers::future::TimeoutFuture::new(0).await;
                let success = run_open(app_in.clone(), target_v, plan, None).await;
                if success {
                    App::status_ok(&app_in.ui.status3,
                        &format_str(format_args!("v{target_v} открыта успешно")));
                } else {
                    let last = app_in.history.borrow().last()
                        .and_then(|r| r.error.clone()).unwrap_or_default();
                    if last == "VersionError" {
                        App::status_ok(&app_in.ui.status3,
                            &format_str(format_args!(
                                "VersionError ✓ — v{target_v} ниже текущей, откат запрещён"
                            )));
                    }
                }
            });
        });
    }

    // ---------- SAMPLE RECORD ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-rename-add-record")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            add_samples(app_in, 1);
        });
    }
    // ---------- REFRESH COUNT ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-rename-refresh-count")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let source = app_in.ui.rename_source.value();
            if source.is_empty() {
                App::status_idle(&app_in.ui.status4, "выберите source store"); return;
            }
            spawn_local(async move {
                let db_guard = app_in.current_db.borrow();
                let Some(db) = db_guard.as_ref() else { return; };
                match count_records(db.raw(), &source).await {
                    Ok(n) => {
                        app_in.ui.rename_count.set_text_content(
                            Some(&format_str(format_args!("records: {n}"))));
                        App::status_ok(&app_in.ui.status4,
                            &format_str(format_args!("«{source}»: {n} записей")));
                    }
                    Err(e) => App::status_err(&app_in.ui.status4,
                        &format_str(format_args!("ошибка: {}", short_error(&e)))),
                }
            });
        });
    }

    // ---------- RENAME run ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-rename-run")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let (step, source, target) = {
                let wf = app_in.rename.borrow();
                match wf.as_ref() {
                    None => (0u8, String::new(), String::new()),
                    Some(w) => (w.step, w.source.clone(), w.target.clone()),
                }
            };

            if step == 0 {
                let source = app_in.ui.rename_source.value();
                let target = app_in.ui.rename_target.value().trim().to_string();
                if source.is_empty() {
                    App::status_err(&app_in.ui.status4, "выберите source"); return;
                }
                if target.is_empty() {
                    App::status_err(&app_in.ui.status4, "введите target"); return;
                }
                if target == source {
                    App::status_err(&app_in.ui.status4, "target = source"); return;
                }
                let plan = app_in.current_plan.borrow();
                if plan.stores.iter().any(|s| s.name == target) {
                    drop(plan);
                    App::status_err(&app_in.ui.status4, "target уже существует"); return;
                }
                if !plan.stores.iter().any(|s| s.name == source) {
                    drop(plan);
                    App::status_err(&app_in.ui.status4, "source не найден"); return;
                }
                drop(plan);
                *app_in.rename.borrow_mut() = Some(RenameWorkflow {
                    source, target: target.clone(), step: 0,
                    records_copied: 0, step2_skipped: false,
                });
                app_in.refresh_rename_ui();
                run_rename_step1(app_in.clone(), target);
                return;
            }

            if step == 1 {
                spawn_local(async move {
                    let count = {
                        let db_guard = app_in.current_db.borrow();
                        match db_guard.as_ref() {
                            Some(db) => count_records(db.raw(), &source).await.unwrap_or(0),
                            None => 0,
                        }
                    };
                    if count == 0 {
                        if let Some(w) = app_in.rename.borrow_mut().as_mut() {
                            w.step = 2; w.step2_skipped = true;
                        }
                        let plan_now = app_in.current_plan.borrow().clone();
                        let v = app_in.current_db.borrow().as_ref()
                            .map(|d| d.version()).unwrap_or(0);
                        app_in.history.borrow_mut().push(VersionRecord {
                            version: v, plan: plan_now,
                            created: vec![], deleted: vec![], 
                            idx_created: vec![], idx_deleted: vec![],
                            upgraded: false, restored: false, error: None,
                            note: Some("step 2: пропущен — нет записей".to_string()),
                        });
                        app_in.render_timeline();
                        app_in.refresh_rename_ui();
                        App::status_ok(&app_in.ui.status4,
                            "шаг 2 пропущен — в source нет данных");
                        return;
                    }
                    App::status_info(&app_in.ui.status4,
                        &format_str(format_args!("шаг 2: копирование {count} записей…")));
                    let res = {
                        let db_guard = app_in.current_db.borrow();
                        match db_guard.as_ref() {
                            Some(db) => copy_records(db.raw(), &source, &target).await,
                            None => Err(JsValue::from_str("нет соединения")),
                        }
                    };
                    match res {
                        Ok(copied) => {
                            if let Some(w) = app_in.rename.borrow_mut().as_mut() {
                                w.step = 2; w.records_copied = copied;
                            }
                            let plan_now = app_in.current_plan.borrow().clone();
                            let v = app_in.current_db.borrow().as_ref()
                                .map(|d| d.version()).unwrap_or(0);
                            app_in.history.borrow_mut().push(VersionRecord {
                                version: v, plan: plan_now,
                                created: vec![], deleted: vec![], 
                                idx_created: vec![], idx_deleted: vec![],
                                upgraded: false, restored: false, error: None,
                                note: Some(format!(
                                    "step 2: скопировано {copied} записей «{source}» → «{target}»"
                                )),
                            });
                            app_in.render_timeline();
                            app_in.refresh_rename_ui();
                            App::status_ok(&app_in.ui.status4,
                                &format_str(format_args!(
                                    "шаг 2 готов: скопировано {copied} записей"
                                )));
                        }
                        Err(e) => App::status_err(&app_in.ui.status4,
                            &format_str(format_args!("ошибка: {}", short_error(&e)))),
                    }
                });
                return;
            }

            if step == 2 {
                run_rename_step3(app_in.clone(), source, target);
            }
        });
    }

    // ---------- RENAME cancel ----------
    {
        let app_c = app.clone();
        let btn = get_button("sch-rename-cancel")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            *app_in.rename.borrow_mut() = None;
            app_in.ui.rename_target.set_value("");
            app_in.refresh_rename_ui();
            App::status_idle(&app_in.ui.status4, "workflow сброшен");
        });
    }

    Ok(())
}

// ============================================================
//                  BOOTSTRAP
// ============================================================

async fn restore_from_existing(app: &Rc<App>) {
    let list = match idb_factory::list_databases().await {
        Ok(l) => l,
        Err(_) => return,
    };
    let Some(info) = list.into_iter().find(|d| d.name == DEMO_DB) else { return; };

    let raw = match idb_factory::open_by_name(DEMO_DB, info.version).await {
        Ok(v) => v,
        Err(_) => return,
    };
    let db = Database::from_raw(raw);

    let names = db.store_names();
    let specs = read_store_specs(db.raw(), &names);

    *app.current_plan.borrow_mut() = SchemaPlan { stores: specs.clone() };
    *app.current_db.borrow_mut() = Some(db);

    app.history.borrow_mut().push(VersionRecord {
        version: info.version,
        plan: SchemaPlan { stores: specs.clone() },
        created: vec![], deleted: vec![],
        idx_created: vec![], idx_deleted: vec![],
        upgraded: false, restored: true, error: None,
        note: Some("восстановлено из indexedDB".to_string()),
    });

    app.refresh_all();

    let source = app.ui.rename_source.value();
    if !source.is_empty() {
        if let Some(db) = app.current_db.borrow().as_ref() {
            if let Ok(n) = count_records(db.raw(), &source).await {
                app.ui.rename_count.set_text_content(
                    Some(&format_str(format_args!("records: {n}"))));
            }
        }
    }

    App::status_ok(&app.ui.status1,
        &format_str(format_args!(
            "восстановлена база v{}: {} store'ов", info.version, names.len(),
        )));
}

/// Читает keyPath, autoIncrement и индексы каждого store.
fn read_store_specs(db: &JsValue, names: &[String]) -> Vec<StoreSpec> {
    if names.is_empty() { return Vec::new(); }
    let refs: Vec<&str> = names.iter().map(|s| s.as_str()).collect();
    let tx = match open_tx(db, &refs, "readonly") {
        Ok(t) => t,
        Err(_) => return names.iter().map(|n| StoreSpec {
            name: n.clone(), key_path: None, auto_increment: false, indexes: vec![],
        }).collect(),
    };

    let mut out = Vec::with_capacity(names.len());
    for name in names {
        let store = match get_object_store(&tx, name) {
            Ok(s) => s,
            Err(_) => {
                out.push(StoreSpec { name: name.clone(), key_path: None, auto_increment: false, indexes: vec![] });
                continue;
            }
        };

        let kp_val = js_sys::Reflect::get(&store, &JsValue::from_str("keyPath"))
            .unwrap_or(JsValue::UNDEFINED);
        let kp: Option<String> = if let Some(s) = kp_val.as_string() {
            Some(s)
        } else if js_sys::Array::is_array(&kp_val) {
            let arr = js_sys::Array::from(&kp_val);
            let mut parts = Vec::new();
            for i in 0..arr.length() {
                if let Some(s) = arr.get(i).as_string() { parts.push(s); }
            }
            Some(parts.join(","))
        } else { None };

        let auto = js_sys::Reflect::get(&store, &JsValue::from_str("autoIncrement"))
            .ok().and_then(|v| v.as_bool()).unwrap_or(false);

        // Читаем индексы.
        let idx_names = read_index_names(&store);
        let mut indexes = Vec::with_capacity(idx_names.len());
        for iname in idx_names {
            if let Some(spec) = read_index_spec(&store, &iname) {
                indexes.push(spec);
            }
        }

        out.push(StoreSpec { name: name.clone(), key_path: kp, auto_increment: auto, indexes });
    }
    return out;
}

fn read_index_names(store: &JsValue) -> Vec<String> {
    let list = match js_sys::Reflect::get(store, &JsValue::from_str("indexNames")) {
        Ok(v) if !v.is_undefined() => v,
        _ => return Vec::new(),
    };
    let length = js_sys::Reflect::get(&list, &JsValue::from_str("length"))
        .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
    let item_fn = match js_sys::Reflect::get(&list, &JsValue::from_str("item")) {
        Ok(v) => v, Err(_) => return Vec::new(),
    };
    let item: js_sys::Function = match item_fn.dyn_into() { Ok(f) => f, Err(_) => return Vec::new() };
    let mut out = Vec::new();
    for i in 0..length {
        if let Ok(v) = item.call1(&list, &JsValue::from_f64(i as f64)) {
            if let Some(s) = v.as_string() { out.push(s); }
        }
    }
    out
}

fn read_index_spec(store: &JsValue, idx_name: &str) -> Option<IndexSpec> {
    let fn_val = js_sys::Reflect::get(store, &JsValue::from_str("index")).ok()?;
    let idx_fn: js_sys::Function = fn_val.dyn_into().ok()?;
    let idx = idx_fn.call1(store, &JsValue::from_str(idx_name)).ok()?;

    let kp_val = js_sys::Reflect::get(&idx, &JsValue::from_str("keyPath")).unwrap_or(JsValue::UNDEFINED);
    let kp = if let Some(s) = kp_val.as_string() { s }
        else if js_sys::Array::is_array(&kp_val) {
            let arr = js_sys::Array::from(&kp_val);
            let mut parts = Vec::new();
            for i in 0..arr.length() {
                if let Some(s) = arr.get(i).as_string() { parts.push(s); }
            }
            parts.join(",")
        } else { String::new() };

    let unique = js_sys::Reflect::get(&idx, &JsValue::from_str("unique"))
        .ok().and_then(|v| v.as_bool()).unwrap_or(false);
    let multi = js_sys::Reflect::get(&idx, &JsValue::from_str("multiEntry"))
        .ok().and_then(|v| v.as_bool()).unwrap_or(false);

    Some(IndexSpec {
        name: idx_name.to_string(),
        key_path: kp,
        unique,
        multi_entry: multi,
    })
}

// ============================================================
//                  WATCHER
// ============================================================

fn setup_db_watcher(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let interval = gloo_timers::callback::Interval::new(WATCH_INTERVAL_MS, move || {
        let app_in = app_c.clone();
        spawn_local(async move { watch_db_state(&app_in).await; });
    });
    interval.forget();
    Ok(())
}

async fn watch_db_state(app: &Rc<App>) {
    if *app.busy.borrow() { return; }
    let list = match idb_factory::list_databases().await { Ok(l) => l, Err(_) => return };
    let in_origin = list.iter().any(|d| d.name == DEMO_DB);
    let has_conn = {
        let db = app.current_db.borrow();
        db.as_ref().map(|d| !d.is_closed()).unwrap_or(false)
    };
    if has_conn && !in_origin {
        if let Some(db) = app.current_db.borrow_mut().take() { db.close(); }
        *app.current_plan.borrow_mut() = SchemaPlan::new();
        app.history.borrow_mut().clear();
        *app.rename.borrow_mut() = None;
        app.refresh_all();
        refresh_real_dbs(app).await;
        App::status_err(&app.ui.status1,
            "⚠ база удалена извне (DevTools) — состояние сброшено");
    }
}

// ============================================================
//                  ADD SAMPLES
// ============================================================

fn add_samples(app: Rc<App>, count: u32) {
    if *app.busy.borrow() { return; }
    let source = app.ui.rename_source.value();
    if source.is_empty() {
        App::status_err(&app.ui.status4, "выберите source"); return;
    }
    App::status_info(&app.ui.status4,
        &format_str(format_args!("добавляю {count} записей…")));

    spawn_local(async move {
        let db_guard = app.current_db.borrow();
        let Some(db) = db_guard.as_ref() else {
            App::status_err(&app.ui.status4, "нет соединения"); return;
        };
        let plan = app.current_plan.borrow();
        let spec = plan.stores.iter().find(|s| s.name == source).cloned();
        drop(plan);

        let mut added = 0u32;
        let mut last_err: Option<String> = None;
        for i in 0..count {
            let rec = build_sample_record(spec.as_ref(), i);
            match put_record(db.raw(), &source, &rec).await {
                Ok(()) => added += 1,
                Err(e) => { last_err = Some(short_error(&e)); break; }
            }
        }
        let cnt = count_records(db.raw(), &source).await.unwrap_or(0);
        app.ui.rename_count.set_text_content(
            Some(&format_str(format_args!("records: {cnt}"))));
        if let Some(err) = last_err {
            App::status_err(&app.ui.status4,
                &format_str(format_args!("добавлено {added}/{count}: {err}")));
        } else {
            App::status_ok(&app.ui.status4,
                &format_str(format_args!("добавлено {added}; всего {cnt}. DevTools → ↻")));
        }
    });
}

// ============================================================
//                  RENAME STEPS
// ============================================================

fn run_rename_step1(app: Rc<App>, target: String) {
    let source = match app.rename.borrow().as_ref() { Some(w) => w.source.clone(), None => return };
    let source_spec = app.current_plan.borrow()
        .stores.iter().find(|s| s.name == source).cloned();
    let Some(s) = source_spec else {
        App::status_err(&app.ui.status4, "не нашёл source в плане"); return;
    };
    let target_spec = StoreSpec {
        name: target.clone(),
        key_path: s.key_path,
        auto_increment: s.auto_increment,
        indexes: s.indexes,  // переносим индексы!
    };
    let new_plan = plan_with_store(&app.current_plan.borrow().clone(), target_spec);
    App::status_info(&app.ui.status4, &format_str(format_args!("шаг 1: «{target}»…")));
    app.set_lifecycle("upgrade");

    spawn_local(async move {
        gloo_timers::future::TimeoutFuture::new(0).await;
        let next_v = next_version(&app).await;
        let ok = run_open(app.clone(), next_v, new_plan, None).await;
        if ok {
            if let Some(w) = app.rename.borrow_mut().as_mut() { w.step = 1; }
            app.refresh_rename_ui();
            App::status_ok(&app.ui.status4,
                &format_str(format_args!("шаг 1 готов: «{target}» создан (v{next_v})")));
        }
    });
}

fn run_rename_step3(app: Rc<App>, source: String, target: String) {
    let new_plan = plan_without_store(&app.current_plan.borrow().clone(), &source);
    App::status_info(&app.ui.status4, &format_str(format_args!("шаг 3: «{source}»…")));
    app.set_lifecycle("upgrade");

    spawn_local(async move {
        gloo_timers::future::TimeoutFuture::new(0).await;
        let next_v = next_version(&app).await;
        let copied = app.rename.borrow().as_ref().map(|w| w.records_copied).unwrap_or(0);
        let ok = run_open(app.clone(), next_v, new_plan, None).await;
        if ok {
            *app.rename.borrow_mut() = None;
            app.ui.rename_target.set_value("");
            app.refresh_all();
            App::status_ok(&app.ui.status4,
                &format_str(format_args!(
                    "rename завершён: «{source}» → «{target}» (v{next_v}, скопировано {copied})"
                )));
        }
    });
}

// ============================================================
//                  OPEN
// ============================================================

async fn run_open(
    app: Rc<App>,
    version: u32,
    plan: SchemaPlan,
    note: Option<String>,
) -> bool {
    if *app.busy.borrow() { return false; }
    *app.busy.borrow_mut() = true;
    app.set_buttons_disabled(true);
    app.set_lifecycle("upgrade");
    gloo_timers::future::TimeoutFuture::new(0).await;

    if let Some(old) = app.current_db.borrow_mut().take() { old.close(); }

    let result = idb_schema::open_with_plan(DEMO_DB, version, &plan).await;
    let success = match result {
        Ok(outcome) => {
            app.history.borrow_mut().push(VersionRecord {
                version: outcome.db.version(),
                plan: plan.clone(),
                created: outcome.diff.created.clone(),
                deleted: outcome.diff.deleted.clone(),
                idx_created: outcome.diff.indexes_created.clone(),
                idx_deleted: outcome.diff.indexes_deleted.clone(),
                upgraded: outcome.diff.upgraded,
                restored: false,
                error: None,
                note,
            });
            *app.current_db.borrow_mut() = Some(outcome.db);
            *app.current_plan.borrow_mut() = plan.clone();
            true
        }
        Err(e) => {
            let name = error_name(&e);
            let desc = describe_error(&name);
            app.history.borrow_mut().push(VersionRecord {
                version, plan: plan.clone(),
                created: vec![], deleted: vec![], 
                idx_created: vec![], idx_deleted: vec![],
                upgraded: false, restored: false,
                error: Some(name.clone()), note: None,
            });
            App::status_err(&app.ui.status1, &desc);
            false
        }
    };
    app.refresh_all();
    refresh_real_dbs(&app).await;
    *app.busy.borrow_mut() = false;
    app.set_buttons_disabled(false);
    success
}

// ============================================================
//                  REAL DBS
// ============================================================

async fn refresh_real_dbs(app: &Rc<App>) {
    match idb_factory::list_databases().await {
        Ok(mut list) => {
            list.sort_by(|a, b| a.name.cmp(&b.name));
            if list.is_empty() {
                app.ui.real_dbs.set_inner_html(
                    "<div class=\"db-empty\">В origin пока нет ни одной базы</div>");
                return;
            }
            let mut html = String::with_capacity(256);
            for d in &list {
                let is_demo = d.name == DEMO_DB;
                let cls = if is_demo { "db-item demo" } else { "db-item" };
                let star = if is_demo {
                    "<span class=\"db-star\" title=\"наша рабочая база\">★</span>"
                } else { "" };
                html.push_str(&format_str(format_args!(
                    "<div class=\"{cls}\">{star}<code>{}</code><span class=\"db-ver\">v{}</span></div>",
                    esc(&d.name), d.version,
                )));
            }
            app.ui.real_dbs.set_inner_html(&html);
        }
        Err(e) => {
            app.ui.real_dbs.set_inner_html(&format_str(format_args!(
                "<div class=\"db-empty\" style=\"color:var(--c-err)\">Ошибка: {}</div>",
                esc(&error_name(&e)))));
        }
    }
}

// ============================================================
//                  LISTENERS
// ============================================================

fn setup_auto_toggle(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
        let checked = app_c.ui.new_auto.checked();
        app_c.ui.new_key.set_disabled(checked);
        if checked { app_c.ui.new_key.set_value(""); app_c.ui.new_key.set_placeholder("— auto —"); }
        else { app_c.ui.new_key.set_placeholder("keyPath"); }
    });
    app.ui.new_auto.add_event_listener_with_callback("change", cb.as_ref().unchecked_ref())?;
    cb.forget(); Ok(())
}

fn setup_slider_sync(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let cb_s = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
        let v = app_c.ui.try_v_slider.value();
        app_c.ui.try_v.set_value(&v);
    });
    app.ui.try_v_slider.add_event_listener_with_callback("input", cb_s.as_ref().unchecked_ref())?;
    cb_s.forget();

    let app_c = app.clone();
    let cb_i = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
        let v = app_c.ui.try_v.value();
        if v.parse::<u32>().is_ok() { app_c.ui.try_v_slider.set_value(&v); }
    });
    app.ui.try_v.add_event_listener_with_callback("input", cb_i.as_ref().unchecked_ref())?;
    cb_i.forget(); Ok(())
}

fn setup_source_change(app: &Rc<App>) -> Result<(), JsValue> {
    let app_c = app.clone();
    let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |_| {
        let source = app_c.ui.rename_source.value();
        if source.is_empty() { return; }
        let app_in = app_c.clone();
        spawn_local(async move {
            let db_guard = app_in.current_db.borrow();
            let Some(db) = db_guard.as_ref() else { return; };
            if let Ok(n) = count_records(db.raw(), &source).await {
                app_in.ui.rename_count.set_text_content(
                    Some(&format_str(format_args!("records: {n}"))));
            }
        });
    });
    app.ui.rename_source.add_event_listener_with_callback("change", cb.as_ref().unchecked_ref())?;
    cb.forget(); Ok(())
}

// ============================================================
//                  IDB OPERATIONS
// ============================================================

fn build_sample_record(spec: Option<&StoreSpec>, idx: u32) -> JsValue {
    let obj = js_sys::Object::new();
    let n = ((js_sys::Date::now() as u64) + idx as u64) % 1_000_000;

    let _ = js_sys::Reflect::set(&obj, &JsValue::from_str("name"),
        &JsValue::from_str(&format_str(format_args!("Sample #{n}"))));
    let _ = js_sys::Reflect::set(&obj, &JsValue::from_str("created_at"),
        &JsValue::from_f64(js_sys::Date::now()));

    let auto = spec.map(|s| s.auto_increment).unwrap_or(false);
    if !auto {
        let kp = spec.and_then(|s| s.key_path.clone()).unwrap_or_else(|| "id".into());
        // Составной keyPath — пишем каждое поле.
        let parts: Vec<&str> = kp.split(',').map(|s| s.trim()).collect();
        for (i, p) in parts.iter().enumerate() {
            let val = format_str(format_args!("sample-{n}-{i}"));
            let _ = js_sys::Reflect::set(&obj, &JsValue::from_str(p), &JsValue::from_str(&val));
        }
    }
    return obj.into();
}

async fn count_records(db: &JsValue, store_name: &str) -> Result<u32, JsValue> {
    let tx = open_tx(db, &[store_name], "readonly")?;
    let store = get_object_store(&tx, store_name)?;
    let count_fn: js_sys::Function = js_sys::Reflect::get(&store, &JsValue::from_str("count"))?
        .dyn_into().map_err(|_| JsValue::from_str("count не функция"))?;
    let req = count_fn.call0(&store)
        .map_err(|e| JsValue::from_str(&format!("count: {e:?}")))?;
    let result = await_request(&req).await?;
    Ok(result.as_f64().unwrap_or(0.0) as u32)
}

async fn put_record(db: &JsValue, store_name: &str, record: &JsValue) -> Result<(), JsValue> {
    let tx = open_tx(db, &[store_name], "readwrite")?;
    let store = get_object_store(&tx, store_name)?;
    let put_fn: js_sys::Function = js_sys::Reflect::get(&store, &JsValue::from_str("put"))?
        .dyn_into().map_err(|_| JsValue::from_str("put не функция"))?;
    let req = put_fn.call1(&store, record)
        .map_err(|e| JsValue::from_str(&format!("put: {e:?}")))?;
    let _ = await_request(&req).await?;
    Ok(())
}

async fn copy_records(db: &JsValue, source: &str, target: &str) -> Result<u32, JsValue> {
    let records = get_all_records(db, source).await?;
    let n = records.length();
    if n == 0 { return Ok(0); }

    let tx = open_tx(db, &[target], "readwrite")?;
    let complete_promise = {
        let tx_c = tx.clone();
        js_sys::Promise::new(&mut move |resolve, reject| {
            let resolve_c = resolve.clone();
            let on_complete = Closure::once_into_js(move || {
                let _ = resolve_c.call0(&JsValue::UNDEFINED);
            });
            let _ = js_sys::Reflect::set(&tx_c, &JsValue::from_str("oncomplete"), &on_complete);
            let tx_e = tx_c.clone();
            let reject_c = reject.clone();
            let on_error = Closure::once_into_js(move || {
                let _ = reject_c.call1(&JsValue::UNDEFINED, &JsValue::from_str("tx error"));
                let _ = tx_e;
            });
            let _ = js_sys::Reflect::set(&tx_c, &JsValue::from_str("onerror"), &on_error);
        })
    };
    let store = get_object_store(&tx, target)?;
    let put_fn: js_sys::Function = js_sys::Reflect::get(&store, &JsValue::from_str("put"))?
        .dyn_into().map_err(|_| JsValue::from_str("put не функция"))?;
    for i in 0..n {
        let rec = records.get(i);
        put_fn.call1(&store, &rec)
            .map_err(|e| JsValue::from_str(&format!("put: {e:?}")))?;
    }
    JsFuture::from(complete_promise).await?;
    Ok(n)
}

async fn get_all_records(db: &JsValue, store_name: &str) -> Result<js_sys::Array, JsValue> {
    let tx = open_tx(db, &[store_name], "readonly")?;
    let store = get_object_store(&tx, store_name)?;
    let getall_fn: js_sys::Function = js_sys::Reflect::get(&store, &JsValue::from_str("getAll"))?
        .dyn_into().map_err(|_| JsValue::from_str("getAll не функция"))?;
    let req = getall_fn.call0(&store)
        .map_err(|e| JsValue::from_str(&format!("getAll: {e:?}")))?;
    let result = await_request(&req).await?;
    result.dyn_into().map_err(|_| JsValue::from_str("getAll вернул не Array"))
}

fn open_tx(db: &JsValue, stores: &[&str], mode: &str) -> Result<JsValue, JsValue> {
    let tx_fn: js_sys::Function = js_sys::Reflect::get(db, &JsValue::from_str("transaction"))?
        .dyn_into().map_err(|_| JsValue::from_str("transaction не функция"))?;
    let stores_arr = js_sys::Array::new();
    for s in stores { stores_arr.push(&JsValue::from_str(s)); }
    tx_fn.call2(db, &stores_arr, &JsValue::from_str(mode))
        .map_err(|e| JsValue::from_str(&format!("transaction: {e:?}")))
}

fn get_object_store(tx: &JsValue, name: &str) -> Result<JsValue, JsValue> {
    let fn_v: js_sys::Function = js_sys::Reflect::get(tx, &JsValue::from_str("objectStore"))?
        .dyn_into().map_err(|_| JsValue::from_str("objectStore не функция"))?;
    fn_v.call1(tx, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("objectStore: {e:?}")))
}

async fn await_request(req: &JsValue) -> Result<JsValue, JsValue> {
    let r_ok_set = req.clone(); let r_ok_in = req.clone();
    let r_err_set = req.clone(); let r_err_in = req.clone();
    let promise = js_sys::Promise::new(&mut move |resolve, reject| {
        let r = r_ok_in.clone();
        let resolve_c = resolve.clone();
        let on_success = Closure::once_into_js(move |_e: web_sys::Event| {
            let result = js_sys::Reflect::get(&r, &JsValue::from_str("result"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = resolve_c.call1(&JsValue::UNDEFINED, &result);
        });
        let _ = js_sys::Reflect::set(&r_ok_set, &JsValue::from_str("onsuccess"), &on_success);

        let r = r_err_in.clone();
        let reject_c = reject.clone();
        let on_error = Closure::once_into_js(move |_e: web_sys::Event| {
            let err = js_sys::Reflect::get(&r, &JsValue::from_str("error"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = reject_c.call1(&JsValue::UNDEFINED, &err);
        });
        let _ = js_sys::Reflect::set(&r_err_set, &JsValue::from_str("onerror"), &on_error);
    });
    JsFuture::from(promise).await
}

// ============================================================
//                  ПЛАНЫ
// ============================================================

fn plan_with_store(base: &SchemaPlan, spec: StoreSpec) -> SchemaPlan {
    let mut p = base.clone();
    if let Some(i) = p.stores.iter().position(|s| s.name == spec.name) {
        p.stores[i] = spec;
    } else { p.stores.push(spec); }
    p
}

fn plan_without_store(base: &SchemaPlan, name: &str) -> SchemaPlan {
    let mut p = base.clone();
    p.stores.retain(|s| s.name != name);
    p
}

async fn next_version(app: &Rc<App>) -> u32 {
    {
        let db = app.current_db.borrow();
        if let Some(d) = db.as_ref() {
            if !d.is_closed() { return d.version() + 1; }
        }
    }
    if let Ok(list) = idb_factory::list_databases().await {
        if let Some(info) = list.iter().find(|d| d.name == DEMO_DB) {
            return info.version + 1;
        }
    }
    1
}

// ============================================================
//                  HELPERS
// ============================================================

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    web_sys::window().ok_or("no window")?.document().ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no #{id}")))
}
fn get_button(id: &str) -> Result<HtmlButtonElement, JsValue> {
    get_el(id)?.dyn_into::<HtmlButtonElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} не кнопка")))
}
fn get_input(id: &str) -> Result<HtmlInputElement, JsValue> {
    get_el(id)?.dyn_into::<HtmlInputElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} не input")))
}
fn get_select(id: &str) -> Result<HtmlSelectElement, JsValue> {
    get_el(id)?.dyn_into::<HtmlSelectElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} не select")))
}

fn set_click<F>(el: &HtmlButtonElement, f: F)
where F: FnMut(web_sys::MouseEvent) + 'static {
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn error_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() { return dx.name(); }
    if let Some(s) = e.as_string() { return s; }
    return format!("{e:?}")
}

fn short_error(e: &JsValue) -> String {
    let full = error_name(e);
    let line = full.lines().next().unwrap_or("").trim();
    line.chars().take(120).collect()
}

fn describe_error(name: &str) -> String {
    match name {
        "VersionError" => "VersionError — понижение версии запрещено".to_string(),
        "InvalidStateError" => "InvalidStateError — схему можно менять только в onupgradeneeded".to_string(),
        "QuotaExceededError" => "QuotaExceededError — квота исчерпана".to_string(),
        "SecurityError" => "SecurityError — доступ заблокирован".to_string(),
        "NotFoundError" => "NotFoundError — объект не найден".to_string(),
        "DataError" => "DataError — несоответствие ключа и keyPath".to_string(),
        "ConstraintError" => "ConstraintError — нарушено unique-ограничение".to_string(),
        _ => return format_str(format_args!("{name}")),
    }
}

fn populate_select(select: &HtmlSelectElement, items: &[String], empty: &str) {
    let prev = select.value();
    let document = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d, None => return,
    };
    select.set_inner_html("");
    if items.is_empty() {
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some(empty));
            let _ = opt.set_attribute("value", "");
            let _ = select.append_child(&opt);
        }
        return;
    }
    for name in items {
        let Ok(el) = document.create_element("option") else { continue; };
        el.set_text_content(Some(name));
        let _ = el.set_attribute("value", name);
        let _ = select.append_child(&el);
    }
    if !prev.is_empty() && items.iter().any(|n| n == &prev) {
        select.set_value(&prev);
    }
}

fn set_step_state(el: &web_sys::Element, state: &str) {
    let _ = el.set_attribute("data-state", state);
}

fn esc(s: &str) -> String {
    s.replace('&', "&amp;").replace('<', "&lt;").replace('>', "&gt;")
        .replace('"', "&quot;").replace('\'', "&#39;")
}
```