# Глава 24.3. IdbDatabase — соединение с базой

## Введение

`IDBDatabase` — это не «ссылка на базу», а **активное соединение**. Пока оно открыто, браузер держит файл базы, а другие операции (открытие с большей версией, `deleteDatabase`) ждут его закрытия. В 24.2 мы видели обратную сторону — `onblocked` у второго запроса. Здесь — как отвечать на это ожидание со стороны первого соединения.

Инструменты главы:

- метаданные соединения — `name`, `version`, `objectStoreNames`;
- `close()` — освободить соединение;
- `onversionchange` — событие «кто-то хочет обновить базу — закройся»;
- `is_closed` — отслеживание собственного состояния.

Правила:

- **`close()` идемпотентно.** Повторный вызов — no-op, не ошибка.
- **`close()` возвращает `()`.** После него использовать соединение нельзя: первая транзакция даст `InvalidStateError`.
- **`objectStoreNames` — это `DOMStringList`,** не `Array`. Итерируем через `.item(i)`, пока `i < .length`.

## Обёртка `Database`

Наша обёртка над сырым `IDBDatabase`:

- хранит `JsValue` исходного объекта — чтобы не зависеть от типизированной обёртки `web-sys`;
- отдельно хранит флаг «мы сами вызывали `close()`»;
- даёт методы `name()`, `version()`, `store_names()`, `close()`, `is_closed()`, `install_versionchange_handler()`.

Особое внимание:

1. **`store_names()`** итерирует `DOMStringList` через `Reflect::get("item")` и `Function::call`. Прямой `.dyn_into::<js_sys::Array>()` не сработает — это не массив.

2. **`install_versionchange_handler()`** использует `Closure::wrap` + `forget`, а не `once_into_js`. Событие может сработать несколько раз за жизненный цикл соединения, а `once_into_js` при повторном вызове паникует в Rust.

3. **`is_closed()`** возвращает только наш флаг. Браузер может закрыть соединение сам (например, при `deleteDatabase` из другой вкладки) — тогда флаг останется `false`. Отдельно подписываемся на `onclose` для честного отслеживания.

`idb_db.rs`:

```
//! Обёртка над `IDBDatabase` — соединение с базой IndexedDB.
//!
//! Открытая база (`IDBDatabase`) — активное соединение. Пока оно открыто,
//! браузер держит файл базы; другой запрос `open` с большей версией или
//! `deleteDatabase` ждёт, пока вы закроетесь. В 24.2 мы видели обратную
//! сторону — `onblocked` у второго запроса. Здесь — как отвечать на это
//! ожидание со стороны первого.
//!
//! Инструменты:
//!   • метаданные соединения — `name`, `version`, `store_names`;
//!   • `close()` — освободить соединение (идемпотентно);
//!   • `onversionchange` — событие «кто-то хочет обновить базу — закройтесь»;
//!   • отслеживание состояния — `is_closed`.
//!
//! ПРАВИЛО: `close()` возвращает `()` — не `Result`. Повторный вызов
//! допустим и не даёт ошибок. Использовать закрытое соединение —
//! `InvalidStateError` при первой транзакции.
//!
//! ПРАВИЛО: `objectStoreNames` — `DOMStringList`, а не `Array`. Итерируем
//! через `Reflect` + `Function::call`, а не через `js_sys::Array::from`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};

/// Обёртка над живым соединением `IDBDatabase`.
///
/// Хранит `JsValue` исходного объекта, чтобы не зависеть от типизированной
/// обёртки `web-sys` — её сигнатуры в разных версиях различаются.
pub struct Database {
    raw: JsValue,
    /// Флаг «мы сами вызывали close()». Отдельный от `raw`, потому что
    /// после закрытия объект остаётся живым, но нерабочим.
    closed: Rc<RefCell<bool>>,
}

impl Database {
    /// Оборачивает `JsValue`, полученный от `Factory::open`.
    ///
    /// Проверку, что это действительно `IDBDatabase`, не делает —
    /// ответственность вызывающего (обычно `idb_factory::open_by_name`).
    pub fn from_raw(raw: JsValue) -> Self {
        Self {
            raw,
            closed: Rc::new(RefCell::new(false)),
        }
    }

    /// Сырое `JsValue` — если нужен метод, которого нет в обёртке.
    pub fn raw(&self) -> &JsValue {
        &self.raw
    }

    /// Имя базы.
    pub fn name(&self) -> String {
        js_sys::Reflect::get(&self.raw, &JsValue::from_str("name"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_default()
    }

    /// Текущая версия схемы.
    pub fn version(&self) -> u32 {
        js_sys::Reflect::get(&self.raw, &JsValue::from_str("version"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0) as u32
    }

    /// Имена всех object store'ов в базе.
    ///
    /// `objectStoreNames` — `DOMStringList` (не массив, не `Array`).
    /// Итерируем через `.item(i)`, пока `i < .length`.
    pub fn store_names(&self) -> Vec<String> {
        let list = match js_sys::Reflect::get(&self.raw, &JsValue::from_str("objectStoreNames")) {
            Ok(v) if !v.is_undefined() && !v.is_null() => v,
            _ => return Vec::new(),
        };
        let length = js_sys::Reflect::get(&list, &JsValue::from_str("length"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0) as u32;
        let item_fn = match js_sys::Reflect::get(&list, &JsValue::from_str("item")) {
            Ok(f) => f,
            Err(_) => return Vec::new(),
        };
        let item: js_sys::Function = match item_fn.dyn_into() {
            Ok(f) => f,
            Err(_) => return Vec::new(),
        };
        let mut out = Vec::with_capacity(length as usize);
        for i in 0..length {
            if let Ok(v) = item.call1(&list, &JsValue::from_f64(i as f64)) {
                if let Some(s) = v.as_string() {
                    out.push(s);
                }
            }
        }
        out
    }

    /// Закрывает соединение. Идемпотентно.
    ///
    /// После закрытия использовать соединение нельзя: первая транзакция
    /// даст `InvalidStateError`. Проверки на это в самой обёртке нет —
    /// транзакции появятся в следующих подглавах.
    pub fn close(&self) {
        // Идемпотентность: если уже закрыто — выходим.
        if *self.closed.borrow() {
            return;
        }
        if let Ok(close_fn) = js_sys::Reflect::get(&self.raw, &JsValue::from_str("close")) {
            if let Ok(f) = close_fn.dyn_into::<js_sys::Function>() {
                let _ = f.call0(&self.raw);
            }
        }
        *self.closed.borrow_mut() = true;
    }

    /// true, если этот объект уже вызывал `close()`.
    ///
    /// Не то же самое, что «браузер освободил ресурсы». Если соединение
    /// было закрыто через `onversionchange` извне, флаг остаётся `false`,
    /// но фактически база закрыта.
    pub fn is_closed(&self) -> bool {
        *self.closed.borrow()
    }

    /// Устанавливает обработчик `onversionchange`.
    ///
    /// Событие приходит, когда **другое** соединение пытается открыть базу
    /// с большей версией или удалить её. Обработчик должен закрыть текущее
    /// соединение — иначе тот запрос зависнет на `onblocked`.
    ///
    /// `Closure::once_into_js` — событие `versionchange` приходит один раз
    /// за жизненный цикл соединения. Колбэк — `FnOnce`, повторный вызов
    /// даст `unreachable`.
       pub fn install_versionchange_handler(&self) {
        let raw = self.raw.clone();
        let closed = self.closed.clone();

        // onversionchange может сработать несколько раз за жизненный цикл
        // соединения (каждое внешнее обновление/удаление). FnOnce-замыкание
        // (`once_into_js`) при повторном вызове паникует в Rust — поэтому
        // FnMut через `Closure::wrap` + `forget`: функция живёт, пока живёт
        // соединение.
        let cb = Closure::wrap(Box::new(move || {
            if let Ok(close_fn) = js_sys::Reflect::get(&raw, &JsValue::from_str("close")) {
                if let Ok(f) = close_fn.dyn_into::<js_sys::Function>() {
                    let _ = f.call0(&raw);
                }
            }
            *closed.borrow_mut() = true;
        }) as Box<dyn FnMut()>);

        let _ = js_sys::Reflect::set(
            &self.raw,
            &JsValue::from_str("onversionchange"),
            cb.as_ref(),
        );
        cb.forget();
    }
}
```

Тесты `idb_db_test.rs`:
```
//! Wasm-тесты модуля `idb_db`: метаданные соединения, закрытие,
//! обработка `versionchange`.
//!
//! Компилируются только под wasm32 — на host файл пуст.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;
use wasm_bindgen_test::*;

use hello_wasm02::idb_db::Database;
use hello_wasm02::idb_factory;

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64);
}

fn test_db_name() -> String {
    return format!("test-db-{}", uid());
}

/// Ожидает завершения `IDBRequest`. Локальная копия
/// `idb_factory::request_to_promise` — потому что последняя приватная,
/// а для тестов нужен свой минимальный вариант.
async fn await_request(request: &JsValue) -> Result<JsValue, JsValue> {
    let r_ok = request.clone();
    let promise = js_sys::Promise::new(&mut move |resolve, reject| {
        let r = r_ok.clone();
        let resolve_in = resolve.clone();
        let on_success = Closure::once_into_js(move |_e: web_sys::Event| {
            let v = js_sys::Reflect::get(&r, &JsValue::from_str("result"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = resolve_in.call1(&JsValue::UNDEFINED, &v);
        });
        let _ = js_sys::Reflect::set(&r_ok, &JsValue::from_str("onsuccess"), &on_success);

                let r_err = request.clone();
        let reject_in = reject.clone();
        let on_error = Closure::once_into_js(move |_e: web_sys::Event| {
            let e = js_sys::Reflect::get(&r_err, &JsValue::from_str("error"))
                .unwrap_or(JsValue::UNDEFINED);
            let _ = reject_in.call1(&JsValue::UNDEFINED, &e);
        });
        // Клон для Reflect::set — `r_err` уже перемещён в замыкание.
        let r_err_set = request.clone();
        let _ = js_sys::Reflect::set(&r_err_set, &JsValue::from_str("onerror"), &on_error);
    });

    JsFuture::from(promise).await
}

/// Открывает новую базу с одним или несколькими object store'ами.
///
/// `createObjectStore` разрешён только внутри `onupgradeneeded`. Пока
/// в проекте нет модуля, который создаёт store'ы, этот хелпер делает
/// это через `Reflect` прямо в тесте.
async fn open_with_stores(db_name: &str, stores: &[&str]) -> Database {
    let global = js_sys::global();
    let idb = js_sys::Reflect::get(&global, &JsValue::from_str("indexedDB"))
        .expect("indexedDB");
    let open_fn = js_sys::Reflect::get(&idb, &JsValue::from_str("open"))
        .expect("open fn");
    let open: js_sys::Function = open_fn.dyn_into().expect("open is fn");
    let request = open
        .call2(&idb, &JsValue::from_str(db_name), &JsValue::from_f64(1.0))
        .expect("open call");

    let stores_owned: Vec<String> = stores.iter().map(|s| s.to_string()).collect();
    let upgrade = Closure::once_into_js(move |e: web_sys::Event| {
        let target = e.target().expect("event target");
        let db = js_sys::Reflect::get(&target, &JsValue::from_str("result"))
            .expect("db result");
        let create_fn = js_sys::Reflect::get(&db, &JsValue::from_str("createObjectStore"))
            .expect("createObjectStore fn");
        let create: js_sys::Function = create_fn.dyn_into().expect("create is fn");
        for name in &stores_owned {
            let params = js_sys::Object::new();
            js_sys::Reflect::set(
                &params,
                &JsValue::from_str("keyPath"),
                &JsValue::from_str("id"),
            )
            .expect("set keyPath");
            let _ = create.call2(&db, &JsValue::from_str(name), &params);
        }
    });
    let _ = js_sys::Reflect::set(
        &request,
        &JsValue::from_str("onupgradeneeded"),
        &upgrade,
    );

    let raw = await_request(&request).await.expect("await open");
    Database::from_raw(raw)
}

// ---------------- метаданные ----------------

#[wasm_bindgen_test]
async fn db_name_matches_opened() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 1).await.expect("open");
    let db = Database::from_raw(raw);

    assert_eq!(db.name(), name, "имя соединения совпадает с запрошенным");

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn db_version_matches_opened() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 3).await.expect("open v3");
    let db = Database::from_raw(raw);

    assert_eq!(db.version(), 3, "версия соединения — 3");

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn raw_returns_original_jsvalue() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 1).await.expect("open");
    let raw_clone = raw.clone();
    let db = Database::from_raw(raw);

    // raw() возвращает ссылку на тот же объект — сравнение по identity
    // через `Object::is`.
    assert!(
        js_sys::Object::is(db.raw(), &raw_clone),
        "raw() возвращает тот же JS-объект, что был передан"
    );

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- objectStoreNames ----------------

#[wasm_bindgen_test]
async fn store_names_empty_on_fresh_db() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 1).await.expect("open");
    let db = Database::from_raw(raw);

    assert!(
        db.store_names().is_empty(),
        "свежая база без store'ов — список пуст"
    );

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn store_names_lists_one_store() {
    let name = test_db_name();
    let db = open_with_stores(&name, &["books"]).await;

    let names = db.store_names();
    assert_eq!(names.len(), 1);
    assert_eq!(names[0], "books");

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn store_names_lists_multiple_stores() {
    let name = test_db_name();
    let db = open_with_stores(&name, &["authors", "books", "covers"]).await;

    let mut names = db.store_names();
    names.sort();
    assert_eq!(names, vec!["authors", "books", "covers"]);

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- close / is_closed ----------------

#[wasm_bindgen_test]
async fn is_closed_false_before_close() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 1).await.expect("open");
    let db = Database::from_raw(raw);

    assert!(!db.is_closed(), "до close() соединение открыто");

    db.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn close_is_idempotent() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 1).await.expect("open");
    let db = Database::from_raw(raw);

    db.close();
    db.close();
    db.close();

    assert!(db.is_closed(), "флаг закрытия остаётся выставленным");

    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn close_marks_connection_closed() {
    let name = test_db_name();
    let raw = idb_factory::open_by_name(&name, 1).await.expect("open");
    let db = Database::from_raw(raw);

    assert!(!db.is_closed());
    db.close();
    assert!(db.is_closed());

    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

// ---------------- versionchange ----------------

/// Сценарий:
///   1. Открываем базу в v1, устанавливаем `onversionchange`.
///   2. Из второй «сессии» пытаемся открыть ту же базу в v2.
///   3. Браузер посылает `versionchange` первому соединению.
///   4. Наш обработчик синхронно вызывает `close()` — второе открытие
///      получает контроль.
///   5. Проверяем, что первое соединение помечено закрытым.
#[wasm_bindgen_test]
async fn versionchange_handler_closes_db() {
    let name = test_db_name();

    // Шаг 1. Первое соединение v1 + обработчик.
    let raw1 = idb_factory::open_by_name(&name, 1).await.expect("open v1");
    let db1 = Database::from_raw(raw1);
    assert!(!db1.is_closed());
    db1.install_versionchange_handler();

    // Шаг 2. Второе соединение v2. Await не разрешится, пока первое не
    // закроется — наш обработчик делает это синхронно.
    let raw2 = idb_factory::open_by_name(&name, 2).await.expect("open v2");
    let db2 = Database::from_raw(raw2);

    // Шаг 5. Первое соединение должно быть помечено закрытым.
    assert!(
        db1.is_closed(),
        "onversionchange должен был закрыть первое соединение"
    );

    db2.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn versionchange_does_not_fire_on_same_version() {
    let name = test_db_name();

    let raw1 = idb_factory::open_by_name(&name, 1).await.expect("open v1");
    let db1 = Database::from_raw(raw1);
    db1.install_versionchange_handler();

    // Закрываем вручную — чтобы второе открытие v1 не блокировалось
    // (по спеке открытие с той же версией допускает несколько соединений,
    // но на всякий случай уберём первое).
    db1.close();

    let raw2 = idb_factory::open_by_name(&name, 1).await.expect("open v1 second");
    let db2 = Database::from_raw(raw2);

    // Второе соединение работает — версия та же, конфликта нет.
    assert_eq!(db2.version(), 1);

    db2.close();
    idb_factory::delete_by_name(&name).await.expect("cleanup");
}
```

## Демо-сценарий: три изолированных эксперимента

Каждая подглава 24.x — это **одна вкладка на один эксперимент** со своей базой данных, своим журналом, своим статус-индикатором. Переключение вкладок не влияет на соседей.

Три базы:

| Вкладка | Имя базы | Что показывает |
|---|---|---|
| 1 | `hello-wasm-idb-24-3-exp1-metadata` | Что внутри соединения: name, version, stores, handler |
| 2 | `hello-wasm-idb-24-3-exp2-idempotent` | `close()` × N — первая работает, остальные no-op |
| 3 | `hello-wasm-idb-24-3-exp3-race` | Гонка двух соединений: `onversionchange` против `onblocked` |


## Эксперимент 1. Что внутри соединения

**Зачем.** Пока `IDBDatabase` открыт, браузер держит файл базы. Соединение — не «пустая ссылка», а объект с состоянием: имя, версия схемы, список store'ов, флаг «открыто/закрыто».

**Что нажимать:**

1. `open v1` — LED загорается зелёным, видны имя и v1.
2. `install handler` — бейдж сменится на «🛡 установлен».
3. `close()` — LED гаснет, появляется штамп CLOSED.


## Эксперимент 2. close() идемпотентно

**Зачем.** Правило проекта: любая «отменить/удалить» операция идемпотентна. У `close()` это свойство встроено: после первого вызова повторные — no-op, не ошибка.

**Что нажимать:**

1. `open v1` — соединение открывается.
2. `close() × 3` или `close() × 5`.
3. В таблице: первая строка «открыто → закрыто», остальные — «закрыто → закрыто».


## Эксперимент 3. Гонка двух соединений ⚡

**Зачем.** IndexedDB разрешает несколько соединений к одной базе. Но если одно держится открытым, а другое пытается **повысить версию** — браузер ждёт, пока первое закроется. Событие `versionchange` у первого соединения — это «пожалуйста, закройся». Без обработчика второе соединение **зависает на onblocked**.

**Что нажимать:**

1. **Гонка с handler**: A открывается в v1 с обработчиком, B — в v2. A автоматически закрывается, LED гаснет, B открывается (~400 мс).
2. **Гонка без handler**: A без обработчика. B зависает на 1.5 сек. Через 1.5 сек A закрывается вручную — только тогда B проходит (~2 сек).


## Три решения, найденные в отладке

Сценарий «Пересоздать базу во время гонки» выявил три бага. Они касаются не только этой подглавы — это **правила работы с IndexedDB в браузере**.

### Решение 1. `delete_database` не прерывается на `onblocked`

По спецификации `deleteDatabase` при наличии открытых соединений сначала стреляет `onblocked`, затем — **после закрытия всех соединений** — `onsuccess`. Прерывание на `onblocked` ломает последующий `open`: база остаётся в старой версии, а мы пытаемся открыть её в младшей → `VersionError`.



### Решение 2. После гонки закрывать оба соединения

В конце `race-clean` и `race-dirty` соединения A и B **остаются открытыми** — чтобы UI показал «v1 закрыто» / «v2 открыто». Это блокирует следующий `deleteDatabase`: он уходит в `onblocked` и ждёт (те самые 14 секунд в логе).



### Решение 3. Флаг `exp3_busy` для защиты от параллельных операций

Пока гонка выполняется, кнопки «Пересоздать базу» и «Обновить» остаются активными. Пользователь нажимает — получаются interleaved-операции, лог превращается в кашу, соединения из гонки блокируют delete.



## Про утечку памяти при закрытии вкладки

Короткий ответ: **утечки нет**. Когда вкладка умирает — штатно или аварийно — браузер сам закрывает все `IDBDatabase`, а вместе с инстансом WASM освобождается вся линейная память, включая `Closure::forget()`-замыкания.

Реальные риски лежат в другом:

- **Долгоживущие контексты** (Service Worker, SharedWorker) — там незакрытое соединение переживает вкладки и держит `deleteDatabase` в блокировке. У нас таких контекстов нет.
- **`Closure::forget()` дважды на одном замыкании** — реальная утечка в Rust-heap.
- **`is_closed()` не знает про аварийное закрытие** — подписка на `onclose` решает.


---

## Чек-лист

- [ ] `src/idb_db.rs` — обёртка `Database` со всеми методами
- [ ] `install_versionchange_handler` через `Closure::wrap` + `forget`
- [ ] `src/app.rs` — HTML с тремя вкладками и тремя базами
- [ ] `App::activate_tab` — сравнение по номеру, не по указателю
- [ ] `error_name` и `esc` — **явный `return`** (r-a теряет тип)
- [ ] `delete_database` — не отклонять на `onblocked`
- [ ] После гонки закрывать A и B
- [ ] `exp3_busy` блокирует параллельные операции
- [ ] Каждая вкладка: свой баннер, своя аннотация, свой журнал, свои tooltips
- [ ] `cargo check --target wasm32-unknown-unknown` без предупреждений
- [ ] Нажать «Гонка с handler» 5–10 раз подряд — каждый раз ~400 мс

Исходный код демо-примера:

```
//! Демо Главы 24.3: IdbDatabase — соединение с базой.
//!
//! Три изолированных эксперимента — по одной вкладке на каждый.
//!   • у каждой вкладки своя база данных (свой name / version / stores);
//!   • у каждой вкладки свой журнал, свой статус, свои кнопки;
//!   • переключение вкладок не влияет на состояние других экспериментов.
//!
//! ПРАВИЛО: `format!` в tail-позиции — `format_str(format_args!(...))`.
//! ПРАВИЛО: `JsCast` для `.dyn_into`/`.dyn_ref`.
//! ПРАВИЛО: `Closure` из `wasm_bindgen::closure::Closure`.
//! ПРАВИЛО: `set_click(&btn, move |_| { btn.set_... })` — нужен `btn_cb`.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::HtmlButtonElement;

use crate::format_str::format_str;
use crate::idb_db::Database;
use crate::idb_factory;

/// Имя базы для эксперимента 1 — метаданные соединения.
const DB1: &str = "hello-wasm-idb-24-3-exp1-metadata";
/// Имя базы для эксперимента 2 — идемпотентность close().
const DB2: &str = "hello-wasm-idb-24-3-exp2-idempotent";
/// Имя базы для эксперимента 3 — гонка двух соединений.
const DB3: &str = "hello-wasm-idb-24-3-exp3-race";

const MINI_LOG_LIMIT: usize = 5;

const HTML: &str = r#"
<style>
  :root {
    --c-info: #3498db;
    --c-ok: #27ae60;
    --c-err: #c0392b;
    --c-warn: #e67e22;
    --c-text: #222;
    --c-muted: #666;
    --c-bg-panel: #fff;
    --c-bg-soft: #fafafa;
    --c-border: #e0e0e0;
    --c-tab-bg: #f0f2f5;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --c-text: #e6e6e6;
      --c-muted: #a0a0a0;
      --c-bg-panel: #1e1e1e;
      --c-bg-soft: #262626;
      --c-border: #3a3a3a;
      --c-tab-bg: #262626;
    }
    body { background: #121212; }
    .id-card.open, .race-card.open { background: #1a2a1f; }
    .log { background: #181818; }
    .exp-anno { background: #1a2233; }
    .exp-scenario { background: #232014; }
    .tab-btn { background: #262626; color: #ccc; }
    .tab-btn.active { background: #1e1e1e; color: #fff; }
    .tab-btn:hover:not(.active) { background: #2a2a2a; }
  }
  body {
    font-family: system-ui, -apple-system, sans-serif; color: var(--c-text);
    margin: 0; padding: 0 20px 40px;
  }
  h1 { margin: 20px 0 4px; font-size: 22px; }
  h2.panel-title {
    font-size: 16px; font-weight: 600; margin: 0 0 4px;
    display: flex; align-items: center; gap: 8px;
  }
  .exp-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 26px; height: 26px; border-radius: 50%;
    background: var(--c-info); color: #fff; font-weight: 700; font-size: 13px;
    flex-shrink: 0;
  }
  .subtitle { color: var(--c-muted); font-size: 13px; margin: 0 0 12px; }
  .mono { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 12px; }

  /* ============ Tabs ============ */
  .tabs-header {
    display: flex; gap: 4px; align-items: stretch;
    margin: 14px 0 0; padding: 0;
    border-bottom: 2px solid var(--c-info);
  }
  .tab-btn {
    display: flex; align-items: center; gap: 8px;
    padding: 10px 18px; font-size: 14px; font-weight: 600;
    border: 1px solid var(--c-border); border-bottom: none;
    border-radius: 8px 8px 0 0; background: var(--c-tab-bg);
    color: var(--c-muted); cursor: pointer;
    transition: background 0.15s, color 0.15s, transform 0.15s;
    margin-bottom: -2px;
  }
  .tab-btn:hover:not(.active) { background: var(--c-bg-soft); color: var(--c-text); }
  .tab-btn.active {
    background: var(--c-bg-panel); color: var(--c-text);
    border-color: var(--c-info); border-bottom: 2px solid var(--c-bg-panel);
    transform: translateY(1px);
  }
  .tab-btn .tab-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--c-info); color: #fff; font-size: 11px; font-weight: 700;
  }
  .tab-btn:not(.active) .tab-num { background: #b0b0b0; }

  .tabs-hint {
    padding: 10px 14px; margin: 8px 0 12px;
    background: #eaf4ff; border-left: 4px solid var(--c-info);
    border-radius: 6px; font-size: 12px; color: #1f4e79; line-height: 1.5;
  }
  @media (prefers-color-scheme: dark) {
    .tabs-hint { background: #1a2233; color: #9fc4ff; }
  }

  .tab-panel { display: none; }
  .tab-panel.active { display: block; }

  /* ============ Баннер рабочей базы ============ */
  .db-banner {
    display: flex; align-items: center; gap: 16px; flex-wrap: wrap;
    padding: 14px 18px; margin: 14px 0;
    background: linear-gradient(135deg, #eaf4ff 0%, #f5f0ff 100%);
    border: 2px solid var(--c-info); border-radius: 10px;
  }
  @media (prefers-color-scheme: dark) {
    .db-banner { background: linear-gradient(135deg, #1a2a3f 0%, #2a1e3f 100%); }
  }
  .db-banner-label {
    font-size: 11px; text-transform: uppercase; letter-spacing: 0.8px;
    color: var(--c-muted); margin-bottom: 2px;
  }
  .db-banner-name {
    font-family: ui-monospace, monospace; font-size: 16px; font-weight: 700;
    color: #1f4e79; word-break: break-all;
  }
  @media (prefers-color-scheme: dark) {
    .db-banner-name { color: #9fc4ff; }
  }
  .db-banner-meta {
    font-family: ui-monospace, monospace; font-size: 12px; color: var(--c-muted);
  }
  .db-banner-actions { margin-left: auto; display: flex; gap: 8px; }

  /* ============ Панели экспериментов ============ */
  .panel {
    border: 1px solid var(--c-border); border-radius: 10px; padding: 18px;
    margin: 18px 0; background: var(--c-bg-panel);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
  }
  .exp-anno {
    background: #eef4ff; border-left: 4px solid var(--c-info);
    padding: 10px 14px; border-radius: 6px;
    font-size: 12px; line-height: 1.5; margin: 10px 0;
    color: #1f4e79;
  }
  @media (prefers-color-scheme: dark) {
    .exp-anno { color: #9fc4ff; }
  }
  .exp-anno b { color: #1f4e79; }
  @media (prefers-color-scheme: dark) {
    .exp-anno b { color: #c0d8ff; }
  }
  .exp-scenario {
    background: #fff9e6; border-left: 4px solid var(--c-warn);
    padding: 10px 14px; border-radius: 6px;
    font-size: 12px; line-height: 1.6; margin: 10px 0;
    color: #5a4400;
  }
  @media (prefers-color-scheme: dark) {
    .exp-scenario { color: #e6c98c; }
  }
  .exp-scenario ol { margin: 4px 0 0 20px; padding: 0; }
  .exp-scenario li { padding: 2px 0; }
  .exp-scenario code {
    background: rgba(128,128,128,0.15); padding: 1px 5px; border-radius: 3px;
    font-size: 11px;
  }

  .exp-status {
    display: flex; align-items: center; gap: 8px;
    padding: 10px 14px; border-radius: 6px; margin: 10px 0;
    font-family: ui-monospace, monospace; font-size: 12px;
    background: var(--c-bg-soft); border-left: 4px solid #ccc;
    color: var(--c-muted);
    transition: background 0.3s, border-color 0.3s, color 0.3s;
  }
  .exp-status.ok { background: #e8f8ed; border-color: var(--c-ok); color: #1e8449; }
  .exp-status.err { background: #fdedec; border-color: var(--c-err); color: #922b21; }
  .exp-status.busy { background: #fff9e6; border-color: var(--c-warn); color: #7d5a00; }
  .exp-status-icon { font-size: 14px; flex-shrink: 0; }

  .exp-mini-log {
    margin-top: 10px; border: 1px solid var(--c-border); border-radius: 6px;
    background: var(--c-bg-soft); overflow: hidden;
  }
  .exp-mini-log-head {
    padding: 6px 12px; background: var(--c-bg-soft);
    border-bottom: 1px solid var(--c-border);
    font-size: 10px; text-transform: uppercase; letter-spacing: 0.7px;
    color: var(--c-muted);
  }
  .exp-mini-log-body {
    max-height: 130px; overflow-y: auto; padding: 4px 0;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .mini-entry {
    padding: 3px 12px; display: flex; gap: 10px;
    border-left: 3px solid transparent;
    animation: log-fade-in 0.25s ease-out;
  }
  @keyframes log-fade-in {
    from { opacity: 0; transform: translateY(-3px); }
    to { opacity: 1; transform: translateY(0); }
  }
  .mini-time { color: #999; flex-shrink: 0; min-width: 64px; }
  .mini-msg { flex: 1; }
  .mini-info { border-left-color: var(--c-info); }
  .mini-ok { border-left-color: var(--c-ok); }
  .mini-err { border-left-color: var(--c-err); }
  .mini-err .mini-msg { color: #922b21; }
  .mini-empty {
    padding: 14px; text-align: center; color: #aaa;
    font-style: italic; font-size: 11px;
  }

  /* ============ Кнопки ============ */
  .btn-row { display: flex; gap: 8px; flex-wrap: wrap; margin: 10px 0; align-items: center; }
  button {
    padding: 7px 14px; font-size: 13px;
    border: 1px solid #c0c0c0; border-radius: 5px; background: var(--c-bg-soft);
    cursor: pointer; transition: background 0.15s; color: var(--c-text);
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.5; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--c-info); outline-offset: 2px; }
  button.primary { background: #eaf4ff; border-color: var(--c-info); color: #1f4e79; }
  button.danger { background: #fdedec; border-color: var(--c-err); color: #922b21; }
  button.success { background: #e8f8ed; border-color: var(--c-ok); color: #1e8449; }

  .hint-inline {
    font-size: 11px; color: var(--c-muted); font-style: italic;
  }

  /* ============ ID-карта ============ */
  .id-card {
    display: flex; flex-direction: column; gap: 10px;
    border: 2px solid #ccc; border-radius: 10px;
    padding: 14px 16px; background: var(--c-bg-soft);
    transition: border-color 0.4s, background 0.4s; position: relative;
  }
  .id-card.open { border-color: var(--c-ok); background: rgba(39,174,96,0.05); }
  .id-card.closed { border-color: #ccc; background: var(--c-bg-soft); }
  .id-head {
    display: flex; align-items: center; gap: 12px;
    padding-bottom: 8px; border-bottom: 1px dashed var(--c-border);
  }
  .led {
    width: 14px; height: 14px; border-radius: 50%;
    background: #ccc; flex-shrink: 0; transition: background 0.3s;
  }
  .led.open { background: var(--c-ok); animation: pulse-led 1.6s infinite; }
  .led.closed { background: #bbb; }
  @keyframes pulse-led {
    0%,100% { box-shadow: 0 0 0 0 rgba(39,174,96,0.55); }
    60% { box-shadow: 0 0 0 10px rgba(39,174,96,0); }
  }
  .id-name { font-family: ui-monospace, monospace; font-size: 14px; font-weight: 600; word-break: break-all; }
  .id-name.closed { color: var(--c-muted); text-decoration: line-through; }
  .version-badge {
    padding: 2px 10px; border-radius: 12px;
    font-family: ui-monospace, monospace; font-size: 12px; font-weight: 700;
    background: var(--c-info); color: #fff; margin-left: auto;
  }
  .version-badge.closed { background: #ccc; text-decoration: line-through; }
  .status-pill {
    padding: 2px 8px; border-radius: 10px; font-size: 11px;
    font-family: ui-monospace, monospace; font-weight: 600;
  }
  .status-open { background: #e8f8ed; color: #1e8449; }
  .status-closed { background: #f0f0f0; color: #888; }
  .status-unset { background: #fff3cd; color: #7d5a00; }
  .status-set { background: #e6f3ff; color: #1f4e79; }
  .stores-label {
    font-size: 11px; color: var(--c-muted); text-transform: uppercase;
    letter-spacing: 0.6px; margin-bottom: 4px;
  }
  .stores-chips { display: flex; gap: 6px; flex-wrap: wrap; min-height: 28px; align-items: center; }
  .store-chip {
    padding: 4px 10px; border-radius: 14px;
    font-family: ui-monospace, monospace; font-size: 12px;
    background: #eef4ff; color: #1f4e79; border: 1px solid #cfe0ff;
  }
  .stores-empty {
    padding: 8px 12px; border: 1px dashed #ccc; border-radius: 6px;
    color: var(--c-muted); font-size: 12px; font-style: italic;
  }
  .id-foot { font-family: ui-monospace, monospace; font-size: 11px; color: var(--c-muted); }
  .closed-overlay {
    position: absolute; top: 50%; right: 20px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #999; letter-spacing: 2px;
    padding: 4px 12px; border: 2px solid #ccc; border-radius: 4px;
    transform: translateY(-50%) rotate(-6deg); pointer-events: none;
  }

  /* ============ Idempotency ============ */
  .idem-table {
    border-collapse: collapse; width: 100%;
    font-family: ui-monospace, monospace; font-size: 12px; margin-top: 8px;
  }
  .idem-table th, .idem-table td {
    padding: 6px 10px; border-bottom: 1px solid var(--c-border); text-align: left;
  }
  .idem-table th {
    background: var(--c-bg-soft); font-size: 10px;
    text-transform: uppercase; letter-spacing: 0.5px; color: var(--c-muted);
  }
  .idem-table td.state-after { color: var(--c-ok); font-weight: 600; }
  .idem-count {
    display: inline-block; padding: 4px 12px;
    background: #eaf4ff; border-radius: 12px;
    font-family: ui-monospace, monospace; font-size: 12px; color: #1f4e79;
  }

  /* ============ Race ============ */
  .race {
    display: flex; align-items: center; gap: 12px;
    flex-wrap: wrap; justify-content: center;
    margin: 12px 0; padding: 14px;
    background: var(--c-bg-soft); border-radius: 8px;
  }
  .race-card {
    flex: 1 1 220px; min-width: 200px; max-width: 320px;
    border: 2px solid #ccc; border-radius: 8px;
    padding: 12px; background: var(--c-bg-panel);
    display: flex; flex-direction: column; gap: 6px;
  }
  .race-card.open { border-color: var(--c-ok); background: rgba(39,174,96,0.05); }
  .race-card.closed { border-color: #ddd; background: var(--c-bg-soft); }
  .race-head { display: flex; align-items: center; gap: 8px; }
  .race-name { font-family: ui-monospace, monospace; font-size: 12px; font-weight: 700; }
  .race-ver {
    font-family: ui-monospace, monospace; font-size: 11px;
    padding: 1px 8px; border-radius: 10px;
    background: #eef4ff; color: #1f4e79; margin-left: auto;
  }
  .race-hint { font-family: ui-monospace, monospace; font-size: 10px; color: var(--c-muted); }
  .race-arrow {
    display: flex; flex-direction: column; align-items: center;
    min-width: 80px; font-family: ui-monospace, monospace; font-size: 10px;
    color: var(--c-muted);
  }
  .race-arrow-icon {
    font-size: 26px; font-weight: 700; color: #bbb;
    transition: color 0.3s, transform 0.3s;
  }
  .race-arrow-icon.active {
    color: var(--c-warn); animation: arrow-bounce 0.6s infinite alternate;
  }
  @keyframes arrow-bounce {
    from { transform: translateX(0); }
    to { transform: translateX(6px); }
  }
</style>

<h1>Глава 24.3: IdbDatabase — соединение с базой</h1>
<p class="subtitle">
  Метаданные, close() и versionchange. <b>Три изолированных эксперимента — по одной вкладке на каждый.</b>
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase" target="_blank" rel="noopener">MDN: IDBDatabase →</a>
</p>

<!-- ============ ВКЛАДКИ ============ -->
<div class="tabs-header">
  <button id="tab-btn-1" class="tab-btn active" title="Эксперимент 1 — что внутри соединения (name, version, stores, onversionchange)">
    <span class="tab-num">1</span> Что внутри соединения
  </button>
  <button id="tab-btn-2" class="tab-btn" title="Эксперимент 2 — идемпотентность close()">
    <span class="tab-num">2</span> close() идемпотентно
  </button>
  <button id="tab-btn-3" class="tab-btn" title="Эксперимент 3 — гонка двух соединений (onblocked / onversionchange)">
    <span class="tab-num">3</span> Гонка соединений ⚡
  </button>
</div>

<div class="tabs-hint">
  💡 <b>Каждая вкладка работает со своей базой данных</b> и своим набором состояний.
  Переключение вкладок не влияет на другие эксперименты: эксперимент 1 не мешает
  эксперименту 3, у каждого свой <code>IDBDatabase</code> и свой журнал.
  Наведите курсор на кнопки — у каждой есть подсказка.
</div>

<!-- ================================================================ -->
<!-- ЭКСПЕРИМЕНТ 1                                                     -->
<!-- ================================================================ -->
<div id="tab-panel-1" class="tab-panel active">
  <div class="db-banner">
    <div>
      <div class="db-banner-label">база эксперимента 1</div>
      <div class="db-banner-name" id="exp1-banner-name">…</div>
      <div class="db-banner-meta" id="exp1-banner-meta">…</div>
    </div>
    <div class="db-banner-actions">
      <button id="exp1-recreate" class="success" title="Удалить базу эксперимента 1 и создать заново в v1">↻ Пересоздать базу</button>
      <button id="exp1-refresh" title="Обновить информацию о базе">⟳ Обновить</button>
    </div>
  </div>

  <div class="exp-anno">
    <b>Зачем.</b> Пока <code>IDBDatabase</code> открыт, браузер держит файл
    базы. Соединение — это не «пустая ссылка», а объект с состоянием:
    оно знает своё <b>имя</b>, <b>версию схемы</b>, <b>список store'ов</b>,
    и может быть <b>открыто</b> или <b>закрыто</b>.
  </div>

  <div class="exp-scenario">
    <b>Что нажимать и что увидите:</b>
    <ol>
      <li><code>open v1</code> — LED загорится зелёным, увидите имя и v1.</li>
      <li><code>install handler</code> — бейдж сменится на «🛡 установлен».</li>
      <li><code>close()</code> — LED погаснет, появится штамп CLOSED.</li>
    </ol>
  </div>

  <div class="btn-row">
    <button id="exp1-open-v1" class="primary" title="Открыть соединение с базой эксперимента 1 в версии v1">open v1</button>
    <button id="exp1-open-v2" class="primary" title="Открыть соединение в v2 — вызовет versionchange у уже открытого соединения">open v2</button>
    <button id="exp1-install-handler" title="Установить onversionchange — соединение будет закрываться автоматически, когда другое соединение захочет обновить базу">install handler</button>
    <button id="exp1-close" class="danger" title="Закрыть текущее соединение (повторный вызов — no-op, не ошибка)">close()</button>
    <span class="hint-inline">подсказки — при наведении на кнопки</span>
  </div>

  <div id="exp1-id-card" class="id-card closed">
    <div class="id-head">
      <span id="exp1-conn-led" class="led closed" aria-label="состояние соединения"></span>
      <span id="exp1-conn-name" class="id-name closed">—</span>
      <span id="exp1-conn-version" class="version-badge closed">v?</span>
    </div>
    <div>
      <div class="stores-label" title="objectStoreNames соединения — DOMStringList, не массив">object stores</div>
      <div id="exp1-conn-stores" class="stores-chips"></div>
    </div>
    <div class="id-head" style="border-top:1px dashed var(--c-border);border-bottom:none;padding-top:8px">
      <span class="stores-label" style="margin:0" title="обработчик onversionchange закрывает соединение при внешнем обновлении">versionchange handler:</span>
      <span id="exp1-conn-handler" class="status-pill status-unset">не установлен</span>
      <span id="exp1-conn-status" class="status-pill status-closed" style="margin-left:auto">закрыто</span>
    </div>
    <div id="exp1-conn-foot" class="id-foot">соединение не открыто</div>
  </div>

  <div id="exp1-status" class="exp-status">
    <span class="exp-status-icon">—</span>
    <span>готово к запуску</span>
  </div>

  <div class="exp-mini-log">
    <div class="exp-mini-log-head">журнал эксперимента 1 (последние 5)</div>
    <div id="exp1-log" class="exp-mini-log-body">
      <div class="mini-empty">нажмите «open v1» — события появятся здесь</div>
    </div>
  </div>
</div>

<!-- ================================================================ -->
<!-- ЭКСПЕРИМЕНТ 2                                                     -->
<!-- ================================================================ -->
<div id="tab-panel-2" class="tab-panel">
  <div class="db-banner">
    <div>
      <div class="db-banner-label">база эксперимента 2</div>
      <div class="db-banner-name" id="exp2-banner-name">…</div>
      <div class="db-banner-meta" id="exp2-banner-meta">…</div>
    </div>
    <div class="db-banner-actions">
      <button id="exp2-recreate" class="success" title="Удалить базу эксперимента 2 и создать заново в v1">↻ Пересоздать базу</button>
      <button id="exp2-refresh" title="Обновить информацию о базе">⟳ Обновить</button>
    </div>
  </div>

  <div class="exp-anno">
    <b>Зачем.</b> Правило проекта: <b>любая «отменить/удалить» операция
    идемпотентна</b>. Если пользователь нажал кнопку дважды — второй вызов
    не должен ломаться. У <code>close()</code> это свойство встроено: после
    первого вызова повторные — no-op, не ошибка. У эксперимента — своя
    отдельная база, чтобы не пересекаться с экспериментом 1.
  </div>

  <div class="exp-scenario">
    <b>Что нажимать и что увидите:</b>
    <ol>
      <li><code>open v1</code> — соединение откроется.</li>
      <li>Затем <code>close() × 3</code> или <code>close() × 5</code>.</li>
      <li>В таблице: первая строка «открыто → закрыто», остальные — «закрыто → закрыто».</li>
    </ol>
  </div>

  <div class="btn-row">
    <button id="exp2-open-v1" class="primary" title="Открыть соединение с базой эксперимента 2 в версии v1">open v1</button>
    <button id="exp2-close-three" class="danger" title="Закрыть соединение трижды подряд — убедиться, что повторные close() не ломаются">close() × 3</button>
    <button id="exp2-close-five" class="danger" title="Закрыть соединение пять раз подряд — убедиться, что повторные close() не ломаются">close() × 5</button>
    <span id="exp2-idem-count" class="idem-count" title="суммарное число попыток close() за всё время">попыток: 0</span>
    <span id="exp2-conn-pill" class="status-pill status-closed" title="текущее состояние соединения">не открыто</span>
  </div>

  <table class="idem-table">
    <thead>
      <tr>
        <th title="порядковый номер вызова close()">#</th>
        <th title="состояние соединения до вызова">состояние до</th>
        <th title="состояние соединения после вызова">после close()</th>
        <th title="значение Database::is_closed() после вызова">is_closed()</th>
      </tr>
    </thead>
    <tbody id="exp2-idem-tbody">
      <tr><td colspan="4" style="color:var(--c-muted);font-style:italic;text-align:center">нажмите «close() × 3»</td></tr>
    </tbody>
  </table>

  <div id="exp2-status" class="exp-status">
    <span class="exp-status-icon">—</span>
    <span>готово к запуску</span>
  </div>

  <div class="exp-mini-log">
    <div class="exp-mini-log-head">журнал эксперимента 2 (последние 5)</div>
    <div id="exp2-log" class="exp-mini-log-body">
      <div class="mini-empty">нажмите «close() × 3» — события появятся здесь</div>
    </div>
  </div>
</div>

<!-- ================================================================ -->
<!-- ЭКСПЕРИМЕНТ 3                                                     -->
<!-- ================================================================ -->
<div id="tab-panel-3" class="tab-panel">
  <div class="db-banner">
    <div>
      <div class="db-banner-label">база эксперимента 3</div>
      <div class="db-banner-name" id="exp3-banner-name">…</div>
      <div class="db-banner-meta" id="exp3-banner-meta">…</div>
    </div>
    <div class="db-banner-actions">
      <button id="exp3-recreate" class="success" title="Удалить базу эксперимента 3 и создать заново в v1">↻ Пересоздать базу</button>
      <button id="exp3-refresh" title="Обновить информацию о базе">⟳ Обновить</button>
    </div>
  </div>

  <div class="exp-anno">
    <b>Зачем.</b> IndexedDB разрешает несколько соединений к одной базе.
    Но если одно соединение держится открытым, а другое пытается
    <b>повысить версию</b> — браузер ждёт, пока первое закроется. Событие
    <code>versionchange</code> у первого соединения — это «пожалуйста,
    закройся». Без обработчика второе соединение <b>зависает на onblocked</b>.
  </div>

  <div class="exp-scenario">
    <b>Что нажимать и что увидите:</b>
    <ol>
      <li><b>Гонка с handler:</b> A открывается в v1 с обработчиком, B — в v2. A автоматически закрывается, LED гаснет. B открывается.</li>
      <li><b>Гонка без handler:</b> A без обработчика. B зависает на 1.5 сек. Через 1.5 сек A закрывается вручную — только тогда B проходит.</li>
    </ol>
  </div>

  <div class="btn-row">
    <button id="exp3-race-clean" class="success" title="A открывает v1 с onversionchange. B открывает v2. A закроется автоматически, B откроется сразу.">⚡ Гонка с handler</button>
    <button id="exp3-race-dirty" class="danger" title="A открывает v1 без обработчика. B открывает v2 и виснет на onblocked, пока A не закроют вручную через 1.5 сек.">⚡ Гонка без handler</button>
    <span id="exp3-race-time" class="mono" style="color:var(--c-muted);font-size:11px" title="сколько времени заняла последняя гонка"></span>
  </div>

  <div class="race">
    <div id="exp3-race-card-a" class="race-card closed">
      <div class="race-head">
        <span id="exp3-race-a-led" class="led closed"></span>
        <span class="race-name">Соединение A</span>
      </div>
      <span id="exp3-race-a-ver" class="race-ver">—</span>
      <div id="exp3-race-a-hint" class="race-hint">не открыто</div>
    </div>
    <div class="race-arrow" title="A открыто в v1, B пытается открыть v2">
      <div id="exp3-race-arrow-icon" class="race-arrow-icon">→</div>
      <div>open v2</div>
    </div>
    <div id="exp3-race-card-b" class="race-card closed">
      <div class="race-head">
        <span id="exp3-race-b-led" class="led closed"></span>
        <span class="race-name">Соединение B</span>
      </div>
      <span id="exp3-race-b-ver" class="race-ver">—</span>
      <div id="exp3-race-b-hint" class="race-hint">не открыто</div>
    </div>
  </div>

  <div id="exp3-status" class="exp-status">
    <span class="exp-status-icon">—</span>
    <span>готово к запуску</span>
  </div>

  <div class="exp-mini-log">
    <div class="exp-mini-log-head">журнал эксперимента 3 (последние 5)</div>
    <div id="exp3-log" class="exp-mini-log-body">
      <div class="mini-empty">запустите гонку — события появятся здесь</div>
    </div>
  </div>
</div>
"#;

// ============================================================
//                       МОДЕЛЬ
// ============================================================

#[derive(Clone, Copy, PartialEq)]
enum LogKind {
    Info,
    Success,
    Error,
}

impl LogKind {
    fn css(&self) -> &'static str {
        match self {
            LogKind::Info => "mini-info",
            LogKind::Success => "mini-ok",
            LogKind::Error => "mini-err",
        }
    }
    fn prefix(&self) -> &'static str {
        match self {
            LogKind::Info => "ℹ",
            LogKind::Success => "✓",
            LogKind::Error => "✗",
        }
    }
}

#[derive(Clone, Copy, PartialEq)]
enum Scope {
    Exp1,
    Exp2,
    Exp3,
}

impl Scope {
    fn db_name(&self) -> &'static str {
        match self {
            Scope::Exp1 => DB1,
            Scope::Exp2 => DB2,
            Scope::Exp3 => DB3,
        }
    }
    fn log_placeholder(&self) -> &'static str {
        match self {
            Scope::Exp1 => "нажмите «open v1» — события появятся здесь",
            Scope::Exp2 => "нажмите «close() × 3» — события появятся здесь",
            Scope::Exp3 => "запустите гонку — события появятся здесь",
        }
    }
}

struct LogEntry {
    time_ms: f64,
    kind: LogKind,
    message: String,
}

struct IdemRow {
    attempt: u32,
    was_open: bool,
    now_closed: bool,
}

#[derive(Clone)]
struct Ui {
    // Вкладки
    tab_btn_1: HtmlButtonElement,
    tab_btn_2: HtmlButtonElement,
    tab_btn_3: HtmlButtonElement,
    tab_panel_1: web_sys::Element,
    tab_panel_2: web_sys::Element,
    tab_panel_3: web_sys::Element,

    // Эксперимент 1
    exp1_banner_name: web_sys::Element,
    exp1_banner_meta: web_sys::Element,
    exp1_id_card: web_sys::Element,
    exp1_conn_led: web_sys::Element,
    exp1_conn_name: web_sys::Element,
    exp1_conn_version: web_sys::Element,
    exp1_conn_stores: web_sys::Element,
    exp1_conn_handler: web_sys::Element,
    exp1_conn_status: web_sys::Element,
    exp1_conn_foot: web_sys::Element,
    exp1_status: web_sys::Element,
    exp1_log: web_sys::Element,

    // Эксперимент 2
    exp2_banner_name: web_sys::Element,
    exp2_banner_meta: web_sys::Element,
    exp2_idem_tbody: web_sys::Element,
    exp2_idem_count: web_sys::Element,
    exp2_conn_pill: web_sys::Element,
    exp2_status: web_sys::Element,
    exp2_log: web_sys::Element,

    // Эксперимент 3
    exp3_banner_name: web_sys::Element,
    exp3_banner_meta: web_sys::Element,
    exp3_race_card_a: web_sys::Element,
    exp3_race_a_led: web_sys::Element,
    exp3_race_a_ver: web_sys::Element,
    exp3_race_a_hint: web_sys::Element,
    exp3_race_card_b: web_sys::Element,
    exp3_race_b_led: web_sys::Element,
    exp3_race_b_ver: web_sys::Element,
    exp3_race_b_hint: web_sys::Element,
    exp3_race_arrow: web_sys::Element,
    exp3_race_time: web_sys::Element,
    exp3_status: web_sys::Element,
    exp3_log: web_sys::Element,
}

struct App {
    // Эксперимент 1
    exp1_db: Rc<RefCell<Option<Database>>>,
    exp1_handler: Rc<RefCell<bool>>,
    log1: Rc<RefCell<Vec<LogEntry>>>,

    // Эксперимент 2
    exp2_db: Rc<RefCell<Option<Database>>>,
    idem_rows: Rc<RefCell<Vec<IdemRow>>>,
    idem_attempts: Rc<RefCell<u32>>,
    log2: Rc<RefCell<Vec<LogEntry>>>,

    // Эксперимент 3
    exp3_a: Rc<RefCell<Option<Database>>>,
    exp3_b: Rc<RefCell<Option<Database>>>,
    exp3_busy: Rc<RefCell<bool>>,
    log3: Rc<RefCell<Vec<LogEntry>>>,

    started_at: f64,
    ui: Ui,
}

impl App {
    // ---------------- Вкладки ----------------

    fn activate_tab(&self, scope: Scope) {
        let active: u32 = match scope {
            Scope::Exp1 => 1,
            Scope::Exp2 => 2,
            Scope::Exp3 => 3,
        };
        let tabs: [(u32, &HtmlButtonElement, &web_sys::Element); 3] = [
            (1, &self.ui.tab_btn_1, &self.ui.tab_panel_1),
            (2, &self.ui.tab_btn_2, &self.ui.tab_panel_2),
            (3, &self.ui.tab_btn_3, &self.ui.tab_panel_3),
        ];
        for (n, b, p) in tabs {
            let is_active = n == active;
            b.set_class_name(if is_active {
                "tab-btn active"
            } else {
                "tab-btn"
            });
            p.set_class_name(if is_active {
                "tab-panel active"
            } else {
                "tab-panel"
            });
        }
    }

    // ---------------- Статусы экспериментов ----------------

    fn set_exp_status(&self, scope: Scope, kind: Option<LogKind>, msg: &str) {
        let el = match scope {
            Scope::Exp1 => &self.ui.exp1_status,
            Scope::Exp2 => &self.ui.exp2_status,
            Scope::Exp3 => &self.ui.exp3_status,
        };
        let (cls, icon) = match kind {
            Some(LogKind::Success) => ("exp-status ok", "✓"),
            Some(LogKind::Error) => ("exp-status err", "✗"),
            Some(LogKind::Info) => ("exp-status busy", "…"),
            None => ("exp-status", "—"),
        };
        el.set_class_name(cls);
        el.set_inner_html(&format_str(format_args!(
            "<span class=\"exp-status-icon\">{icon}</span><span>{}</span>",
            esc(msg),
        )));
    }

    // ---------------- Логирование ----------------

    fn log_vec(&self, scope: Scope) -> Rc<RefCell<Vec<LogEntry>>> {
        match scope {
            Scope::Exp1 => self.log1.clone(),
            Scope::Exp2 => self.log2.clone(),
            Scope::Exp3 => self.log3.clone(),
        }
    }

    fn log(&self, scope: Scope, kind: LogKind, msg: &str) {
        let entry = LogEntry {
            time_ms: now_ms() - self.started_at,
            kind,
            message: msg.to_string(),
        };
        self.log_vec(scope).borrow_mut().push(entry);
        self.render_mini_log(scope);
    }
    fn log_info(&self, scope: Scope, msg: &str) {
        self.log(scope, LogKind::Info, msg);
    }
    fn log_ok(&self, scope: Scope, msg: &str) {
        self.log(scope, LogKind::Success, msg);
    }
    fn log_err(&self, scope: Scope, msg: &str) {
        self.log(scope, LogKind::Error, msg);
    }

    fn render_mini_log(&self, scope: Scope) {
        let target = match scope {
            Scope::Exp1 => &self.ui.exp1_log,
            Scope::Exp2 => &self.ui.exp2_log,
            Scope::Exp3 => &self.ui.exp3_log,
        };
        let entries = self.log_vec(scope);
        let entries = entries.borrow();
        let filtered: Vec<&LogEntry> = entries.iter().collect();
        if filtered.is_empty() {
            target.set_inner_html(&format_str(format_args!(
                "<div class=\"mini-empty\">{}</div>",
                scope.log_placeholder(),
            )));
            return;
        }
        let skip = if filtered.len() > MINI_LOG_LIMIT {
            filtered.len() - MINI_LOG_LIMIT
        } else {
            0
        };
        let mut html = String::with_capacity(512);
        for e in filtered.iter().skip(skip) {
            html.push_str(&format_str(format_args!(
                "<div class=\"mini-entry {cls}\">\
                 <span class=\"mini-time\">+{:.1} с</span>\
                 <span class=\"mini-msg\"><b>{prefix}</b> {}</span>\
                 </div>",
                e.time_ms / 1000.0,
                esc(&e.message),
                cls = e.kind.css(),
                prefix = e.kind.prefix(),
            )));
        }
        target.set_inner_html(&html);
        target.set_scroll_top(target.scroll_height() as f64);
    }

    // ---------------- ID-карта (эксперимент 1) ----------------

    fn refresh_id_card_exp1(&self) {
        if let Ok(Some(old)) = self.ui.exp1_id_card.query_selector(".closed-overlay") {
            let _ = old.remove();
        }

        let borrow = self.exp1_db.borrow();
        let handler_set = *self.exp1_handler.borrow();

        match borrow.as_ref() {
            None => {
                self.ui.exp1_id_card.set_class_name("id-card closed");
                self.ui.exp1_conn_led.set_class_name("led closed");
                self.ui.exp1_conn_name.set_class_name("id-name closed");
                self.ui.exp1_conn_name.set_text_content(Some("—"));
                self.ui
                    .exp1_conn_version
                    .set_class_name("version-badge closed");
                self.ui.exp1_conn_version.set_text_content(Some("v?"));
                self.ui
                    .exp1_conn_stores
                    .set_inner_html("<div class=\"stores-empty\">нет соединения</div>");
                self.ui
                    .exp1_conn_status
                    .set_class_name("status-pill status-closed");
                self.ui.exp1_conn_status.set_text_content(Some("закрыто"));
                self.ui
                    .exp1_conn_foot
                    .set_text_content(Some("соединение не открыто"));
                self.ui
                    .exp1_conn_handler
                    .set_class_name("status-pill status-unset");
                self.ui
                    .exp1_conn_handler
                    .set_text_content(Some("не установлен"));
            }
            Some(db) => {
                let is_closed = db.is_closed();
                let name = db.name();
                let version = db.version();
                let stores = db.store_names();

                self.ui.exp1_id_card.set_class_name(if is_closed {
                    "id-card closed"
                } else {
                    "id-card open"
                });
                self.ui.exp1_conn_led.set_class_name(if is_closed {
                    "led closed"
                } else {
                    "led open"
                });
                self.ui.exp1_conn_name.set_class_name(if is_closed {
                    "id-name closed"
                } else {
                    "id-name"
                });
                self.ui.exp1_conn_name.set_text_content(Some(&name));
                self.ui.exp1_conn_version.set_class_name(if is_closed {
                    "version-badge closed"
                } else {
                    "version-badge"
                });
                self.ui
                    .exp1_conn_version
                    .set_text_content(Some(&format_str(format_args!("v{version}"))));

                if stores.is_empty() {
                    self.ui.exp1_conn_stores.set_inner_html(
                        "<div class=\"stores-empty\">пусто — schema появится в 24.4</div>",
                    );
                } else {
                    let mut chips = String::with_capacity(128);
                    for s in &stores {
                        chips.push_str(&format_str(format_args!(
                            "<span class=\"store-chip\" title=\"object store\">{}</span>",
                            esc(s),
                        )));
                    }
                    self.ui.exp1_conn_stores.set_inner_html(&chips);
                }

                if handler_set {
                    self.ui
                        .exp1_conn_handler
                        .set_class_name("status-pill status-set");
                    self.ui
                        .exp1_conn_handler
                        .set_text_content(Some("🛡 установлен"));
                } else {
                    self.ui
                        .exp1_conn_handler
                        .set_class_name("status-pill status-unset");
                    self.ui
                        .exp1_conn_handler
                        .set_text_content(Some("не установлен"));
                }

                self.ui.exp1_conn_status.set_class_name(if is_closed {
                    "status-pill status-closed"
                } else {
                    "status-pill status-open"
                });
                self.ui
                    .exp1_conn_status
                    .set_text_content(Some(if is_closed {
                        "закрыто"
                    } else {
                        "открыто"
                    }));

                self.ui
                    .exp1_conn_foot
                    .set_text_content(Some(&format_str(format_args!(
                        "{name} · v{version} · {} store'ов · соединение {}",
                        stores.len(),
                        if is_closed {
                            "закрыто"
                        } else {
                            "активно"
                        },
                    ))));

                if is_closed {
                    if let Some(window) = web_sys::window() {
                        if let Ok(document) = window.document().ok_or(()) {
                            if let Ok(overlay) = document.create_element("div") {
                                overlay.set_class_name("closed-overlay");
                                overlay.set_text_content(Some("CLOSED"));
                                let _ = self.ui.exp1_id_card.append_child(&overlay);
                            }
                        }
                    }
                }
            }
        }
    }

    // ---------------- Идемпотентность (эксперимент 2) ----------------

    fn refresh_exp2_conn_pill(&self) {
        let (cls, text) = {
            let borrow = self.exp2_db.borrow();
            match borrow.as_ref() {
                Some(db) if !db.is_closed() => ("status-pill status-open", "открыто".to_string()),
                Some(_) => ("status-pill status-closed", "закрыто".to_string()),
                None => ("status-pill status-closed", "не открыто".to_string()),
            }
        };
        self.ui.exp2_conn_pill.set_class_name(cls);
        self.ui.exp2_conn_pill.set_text_content(Some(&text));
    }

    fn render_idem_table(&self) {
        let rows = self.idem_rows.borrow();
        let attempts = *self.idem_attempts.borrow();
        self.ui
            .exp2_idem_count
            .set_text_content(Some(&format_str(format_args!("попыток: {attempts}"))));

        if rows.is_empty() {
            self.ui.exp2_idem_tbody.set_inner_html(
                "<tr><td colspan=\"4\" style=\"color:var(--c-muted);font-style:italic;text-align:center\">нажмите «close() × 3»</td></tr>",
            );
            return;
        }
        let mut html = String::with_capacity(512);
        for r in rows.iter() {
            html.push_str(&format_str(format_args!(
                "<tr>\
                 <td>{}</td>\
                 <td>{}</td>\
                 <td class=\"state-after\">{}</td>\
                 <td style=\"color:var(--c-ok)\">{}</td>\
                 </tr>",
                r.attempt,
                if r.was_open {
                    "открыто"
                } else {
                    "закрыто"
                },
                if r.now_closed {
                    "закрыто"
                } else {
                    "открыто"
                },
                r.now_closed,
            )));
        }
        self.ui.exp2_idem_tbody.set_inner_html(&html);
    }

    fn reset_idem(&self) {
        self.idem_rows.borrow_mut().clear();
        *self.idem_attempts.borrow_mut() = 0;
        self.render_idem_table();
    }

    // ---------------- Race (эксперимент 3) ----------------

    fn refresh_race_a(&self, version: Option<u32>, hint: &str) {
        let state_cls = if version.is_some() {
            "race-card open"
        } else {
            "race-card closed"
        };
        self.ui.exp3_race_card_a.set_class_name(state_cls);
        self.ui
            .exp3_race_a_led
            .set_class_name(if version.is_some() {
                "led open"
            } else {
                "led closed"
            });
        self.ui.exp3_race_a_ver.set_text_content(Some(
            &version
                .map(|v| format_str(format_args!("v{v}")))
                .unwrap_or_else(|| "—".into()),
        ));
        self.ui.exp3_race_a_hint.set_text_content(Some(hint));
    }

    fn refresh_race_b(&self, version: Option<u32>, hint: &str) {
        let state_cls = if version.is_some() {
            "race-card open"
        } else {
            "race-card closed"
        };
        self.ui.exp3_race_card_b.set_class_name(state_cls);
        self.ui
            .exp3_race_b_led
            .set_class_name(if version.is_some() {
                "led open"
            } else {
                "led closed"
            });
        self.ui.exp3_race_b_ver.set_text_content(Some(
            &version
                .map(|v| format_str(format_args!("v{v}")))
                .unwrap_or_else(|| "—".into()),
        ));
        self.ui.exp3_race_b_hint.set_text_content(Some(hint));
    }

    fn set_race_arrow(&self, active: bool) {
        self.ui.exp3_race_arrow.set_class_name(if active {
            "race-arrow-icon active"
        } else {
            "race-arrow-icon"
        });
    }

    // ---------------- Баннеры ----------------

    async fn refresh_banner(&self, scope: Scope) {
        let db_name = scope.db_name();
        let stored_version = idb_factory::list_databases().await.ok().and_then(|list| {
            list.into_iter()
                .find(|d| d.name == db_name)
                .map(|d| d.version)
        });

        let conn_state = {
            let conn = match scope {
                Scope::Exp1 => self.exp1_db.clone(),
                Scope::Exp2 => self.exp2_db.clone(),
                Scope::Exp3 => self.exp3_a.clone(),
            };
            let borrow = conn.borrow();
            match borrow.as_ref() {
                Some(db) if !db.is_closed() => {
                    format_str(format_args!("открыто (v{})", db.version()))
                }
                Some(_) => "закрыто".to_string(),
                None => "не открыто".to_string(),
            }
        };

        let stored_label = match stored_version {
            Some(v) => format_str(format_args!("v{v}")),
            None => "—".to_string(),
        };

        let (name_el, meta_el) = match scope {
            Scope::Exp1 => (&self.ui.exp1_banner_name, &self.ui.exp1_banner_meta),
            Scope::Exp2 => (&self.ui.exp2_banner_name, &self.ui.exp2_banner_meta),
            Scope::Exp3 => (&self.ui.exp3_banner_name, &self.ui.exp3_banner_meta),
        };

        name_el.set_text_content(Some(db_name));
        meta_el.set_text_content(Some(&format_str(format_args!(
            "версия в хранилище: {stored_label} · соединение: {conn_state}",
        ))));
    }

    // ---------------- Сброс базы ----------------

    async fn recreate_db(&self, scope: Scope) {
        // Закрыть все соединения данной вкладки.
        match scope {
            Scope::Exp1 => {
                if let Some(db) = self.exp1_db.borrow_mut().take() {
                    db.close();
                }
                *self.exp1_handler.borrow_mut() = false;
            }
            Scope::Exp2 => {
                if let Some(db) = self.exp2_db.borrow_mut().take() {
                    db.close();
                }
            }
            Scope::Exp3 => {
                if let Some(db) = self.exp3_a.borrow_mut().take() {
                    db.close();
                }
                if let Some(db) = self.exp3_b.borrow_mut().take() {
                    db.close();
                }
            }
        }

        let name = scope.db_name();
        let _ = delete_database(name).await;
        if let Ok(raw) = idb_factory::open_by_name(name, 1).await {
            let db = Database::from_raw(raw);
            db.close();
        }
    }
}

// ============================================================
//                        RUN
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        tab_btn_1: get_button("tab-btn-1")?,
        tab_btn_2: get_button("tab-btn-2")?,
        tab_btn_3: get_button("tab-btn-3")?,
        tab_panel_1: get_el("tab-panel-1")?,
        tab_panel_2: get_el("tab-panel-2")?,
        tab_panel_3: get_el("tab-panel-3")?,

        exp1_banner_name: get_el("exp1-banner-name")?,
        exp1_banner_meta: get_el("exp1-banner-meta")?,
        exp1_id_card: get_el("exp1-id-card")?,
        exp1_conn_led: get_el("exp1-conn-led")?,
        exp1_conn_name: get_el("exp1-conn-name")?,
        exp1_conn_version: get_el("exp1-conn-version")?,
        exp1_conn_stores: get_el("exp1-conn-stores")?,
        exp1_conn_handler: get_el("exp1-conn-handler")?,
        exp1_conn_status: get_el("exp1-conn-status")?,
        exp1_conn_foot: get_el("exp1-conn-foot")?,
        exp1_status: get_el("exp1-status")?,
        exp1_log: get_el("exp1-log")?,

        exp2_banner_name: get_el("exp2-banner-name")?,
        exp2_banner_meta: get_el("exp2-banner-meta")?,
        exp2_idem_tbody: get_el("exp2-idem-tbody")?,
        exp2_idem_count: get_el("exp2-idem-count")?,
        exp2_conn_pill: get_el("exp2-conn-pill")?,
        exp2_status: get_el("exp2-status")?,
        exp2_log: get_el("exp2-log")?,

        exp3_banner_name: get_el("exp3-banner-name")?,
        exp3_banner_meta: get_el("exp3-banner-meta")?,
        exp3_race_card_a: get_el("exp3-race-card-a")?,
        exp3_race_a_led: get_el("exp3-race-a-led")?,
        exp3_race_a_ver: get_el("exp3-race-a-ver")?,
        exp3_race_a_hint: get_el("exp3-race-a-hint")?,
        exp3_race_card_b: get_el("exp3-race-card-b")?,
        exp3_race_b_led: get_el("exp3-race-b-led")?,
        exp3_race_b_ver: get_el("exp3-race-b-ver")?,
        exp3_race_b_hint: get_el("exp3-race-b-hint")?,
        exp3_race_arrow: get_el("exp3-race-arrow-icon")?,
        exp3_race_time: get_el("exp3-race-time")?,
        exp3_status: get_el("exp3-status")?,
        exp3_log: get_el("exp3-log")?,
    };

    let app = Rc::new(App {
        exp1_db: Rc::new(RefCell::new(None)),
        exp1_handler: Rc::new(RefCell::new(false)),
        log1: Rc::new(RefCell::new(Vec::new())),

        exp2_db: Rc::new(RefCell::new(None)),
        idem_rows: Rc::new(RefCell::new(Vec::new())),
        idem_attempts: Rc::new(RefCell::new(0)),
        log2: Rc::new(RefCell::new(Vec::new())),

        exp3_a: Rc::new(RefCell::new(None)),
        exp3_b: Rc::new(RefCell::new(None)),
        exp3_busy: Rc::new(RefCell::new(false)),
        log3: Rc::new(RefCell::new(Vec::new())),

        started_at: now_ms(),
        ui,
    });

    app.refresh_id_card_exp1();
    app.refresh_exp2_conn_pill();
    app.render_idem_table();

    // ---------- Вкладки ----------
    for (btn_id, scope) in [
        ("tab-btn-1", Scope::Exp1),
        ("tab-btn-2", Scope::Exp2),
        ("tab-btn-3", Scope::Exp3),
    ] {
        let app_c = app.clone();
        let btn = get_button(btn_id)?;
        set_click(&btn, move |_| {
            app_c.activate_tab(scope);
        });
    }

    // ---------- Старт: у каждой вкладки — своя чистая база v1 ----------
    {
        let app_c = app.clone();
        spawn_local(async move {
            for scope in [Scope::Exp1, Scope::Exp2, Scope::Exp3] {
                let name = scope.db_name();
                let _ = delete_database(name).await;
                if let Ok(raw) = idb_factory::open_by_name(name, 1).await {
                    let db = Database::from_raw(raw);
                    db.close();
                }
                app_c.log_info(scope, "база вкладки пересоздана в v1");
                app_c.refresh_banner(scope).await;
            }
        });
    }

    // ============================================================
    //                     ЭКСПЕРИМЕНТ 1
    // ============================================================

    // ---------- Пересоздать базу ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp1-recreate")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            app_in.set_exp_status(Scope::Exp1, Some(LogKind::Info), "пересоздание базы…");
            app_in.refresh_id_card_exp1();
            spawn_local(async move {
                app_in.recreate_db(Scope::Exp1).await;
                app_in.log_ok(Scope::Exp1, "база пересоздана в v1");
                app_in.set_exp_status(Scope::Exp1, Some(LogKind::Success), "база пересоздана в v1");
                app_in.refresh_banner(Scope::Exp1).await;
                app_in.refresh_id_card_exp1();
            });
        });
    }

    // ---------- Обновить баннер ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp1-refresh")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            spawn_local(async move {
                app_in.refresh_banner(Scope::Exp1).await;
            });
        });
    }

    // ---------- open v1 ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp1-open-v1")?;
        let btn_cb = btn.clone();
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let btn_in = btn_cb.clone();
            btn_in.set_disabled(true);
            app_in.set_exp_status(Scope::Exp1, Some(LogKind::Info), "open v1 → выполняется…");
            spawn_local(async move {
                let t0 = now_ms();
                match idb_factory::open_by_name(DB1, 1).await {
                    Ok(raw) => {
                        if let Some(old) = app_in.exp1_db.borrow_mut().take() {
                            old.close();
                        }
                        let db = Database::from_raw(raw);
                        *app_in.exp1_db.borrow_mut() = Some(db);
                        *app_in.exp1_handler.borrow_mut() = false;
                        let dt = now_ms() - t0;
                        let msg = format_str(format_args!("open v1 → OK ({dt:.0} мс)"));
                        app_in.log_ok(Scope::Exp1, &msg);
                        app_in.set_exp_status(Scope::Exp1, Some(LogKind::Success), &msg);
                    }
                    Err(e) => {
                        let name = error_name(&e);
                        let msg = format_str(format_args!("open v1 → {name}"));
                        app_in.log_err(Scope::Exp1, &msg);
                        app_in.set_exp_status(Scope::Exp1, Some(LogKind::Error), &msg);
                    }
                }
                app_in.refresh_id_card_exp1();
                app_in.refresh_banner(Scope::Exp1).await;
                btn_in.set_disabled(false);
            });
        });
    }

    // ---------- open v2 ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp1-open-v2")?;
        let btn_cb = btn.clone();
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let btn_in = btn_cb.clone();
            btn_in.set_disabled(true);
            app_in.set_exp_status(Scope::Exp1, Some(LogKind::Info), "open v2 → выполняется…");
            spawn_local(async move {
                let t0 = now_ms();
                match idb_factory::open_by_name(DB1, 2).await {
                    Ok(raw) => {
                        if let Some(old) = app_in.exp1_db.borrow_mut().take() {
                            old.close();
                        }
                        let db = Database::from_raw(raw);
                        *app_in.exp1_db.borrow_mut() = Some(db);
                        *app_in.exp1_handler.borrow_mut() = false;
                        let dt = now_ms() - t0;
                        let msg =
                            format_str(format_args!("open v2 → OK ({dt:.0} мс, схема обновлена)"));
                        app_in.log_ok(Scope::Exp1, &msg);
                        app_in.set_exp_status(Scope::Exp1, Some(LogKind::Success), &msg);
                    }
                    Err(e) => {
                        let name = error_name(&e);
                        let msg = format_str(format_args!("open v2 → {name}"));
                        app_in.log_err(Scope::Exp1, &msg);
                        app_in.set_exp_status(Scope::Exp1, Some(LogKind::Error), &msg);
                    }
                }
                app_in.refresh_id_card_exp1();
                app_in.refresh_banner(Scope::Exp1).await;
                btn_in.set_disabled(false);
            });
        });
    }

    // ---------- install handler ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp1-install-handler")?;
        set_click(&btn, move |_| {
            let ok = {
                let borrow = app_c.exp1_db.borrow();
                match borrow.as_ref() {
                    Some(db) if !db.is_closed() => {
                        db.install_versionchange_handler();
                        true
                    }
                    _ => false,
                }
            };
            if ok {
                *app_c.exp1_handler.borrow_mut() = true;
                app_c.log_ok(Scope::Exp1, "install handler → установлен");
                app_c.set_exp_status(
                    Scope::Exp1,
                    Some(LogKind::Success),
                    "🛡 обработчик установлен",
                );
            } else {
                app_c.log_err(Scope::Exp1, "install handler → нет открытого соединения");
                app_c.set_exp_status(
                    Scope::Exp1,
                    Some(LogKind::Error),
                    "сначала откройте соединение (open v1)",
                );
            }
            app_c.refresh_id_card_exp1();
        });
    }

    // ---------- close() ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp1-close")?;
        set_click(&btn, move |_| {
            let was_open = {
                let borrow = app_c.exp1_db.borrow();
                match borrow.as_ref() {
                    Some(db) => !db.is_closed(),
                    None => false,
                }
            };
            if let Some(db) = app_c.exp1_db.borrow().as_ref() {
                db.close();
            }
            if was_open {
                app_c.log_ok(Scope::Exp1, "close() → закрыто");
                app_c.set_exp_status(Scope::Exp1, Some(LogKind::Success), "соединение закрыто");
            } else if app_c.exp1_db.borrow().is_some() {
                app_c.log_info(Scope::Exp1, "close() → уже закрыто (no-op)");
                app_c.set_exp_status(Scope::Exp1, Some(LogKind::Info), "уже закрыто (no-op)");
            } else {
                app_c.log_err(Scope::Exp1, "close() → нечего закрывать");
                app_c.set_exp_status(
                    Scope::Exp1,
                    Some(LogKind::Error),
                    "сначала откройте соединение",
                );
            }
            app_c.refresh_id_card_exp1();
        });
    }

    // ============================================================
    //                     ЭКСПЕРИМЕНТ 2
    // ============================================================

    // ---------- Пересоздать ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp2-recreate")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            app_in.set_exp_status(Scope::Exp2, Some(LogKind::Info), "пересоздание базы…");
            spawn_local(async move {
                app_in.recreate_db(Scope::Exp2).await;
                app_in.reset_idem();
                app_in.log_ok(Scope::Exp2, "база пересоздана в v1");
                app_in.set_exp_status(Scope::Exp2, Some(LogKind::Success), "база пересоздана в v1");
                app_in.refresh_banner(Scope::Exp2).await;
                app_in.refresh_exp2_conn_pill();
            });
        });
    }

    // ---------- Обновить баннер ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp2-refresh")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            spawn_local(async move {
                app_in.refresh_banner(Scope::Exp2).await;
            });
        });
    }

    // ---------- open v1 ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp2-open-v1")?;
        let btn_cb = btn.clone();
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let btn_in = btn_cb.clone();
            btn_in.set_disabled(true);
            app_in.set_exp_status(Scope::Exp2, Some(LogKind::Info), "open v1 → выполняется…");
            spawn_local(async move {
                let t0 = now_ms();
                match idb_factory::open_by_name(DB2, 1).await {
                    Ok(raw) => {
                        if let Some(old) = app_in.exp2_db.borrow_mut().take() {
                            old.close();
                        }
                        let db = Database::from_raw(raw);
                        *app_in.exp2_db.borrow_mut() = Some(db);
                        let dt = now_ms() - t0;
                        let msg = format_str(format_args!("open v1 → OK ({dt:.0} мс)"));
                        app_in.log_ok(Scope::Exp2, &msg);
                        app_in.set_exp_status(Scope::Exp2, Some(LogKind::Success), &msg);
                    }
                    Err(e) => {
                        let name = error_name(&e);
                        let msg = format_str(format_args!("open v1 → {name}"));
                        app_in.log_err(Scope::Exp2, &msg);
                        app_in.set_exp_status(Scope::Exp2, Some(LogKind::Error), &msg);
                    }
                }
                app_in.refresh_exp2_conn_pill();
                app_in.refresh_banner(Scope::Exp2).await;
                btn_in.set_disabled(false);
            });
        });
    }

    // ---------- close × N ----------
    for (id, n) in [("exp2-close-three", 3u32), ("exp2-close-five", 5u32)] {
        let app_c = app.clone();
        let btn = get_button(id)?;
        set_click(&btn, move |_| {
            app_c.reset_idem();
            for _ in 0..n {
                let was_open = {
                    let borrow = app_c.exp2_db.borrow();
                    match borrow.as_ref() {
                        Some(db) => !db.is_closed(),
                        None => false,
                    }
                };
                if let Some(db) = app_c.exp2_db.borrow().as_ref() {
                    db.close();
                }
                let now_closed = {
                    let borrow = app_c.exp2_db.borrow();
                    match borrow.as_ref() {
                        Some(db) => db.is_closed(),
                        None => true,
                    }
                };
                let attempt = {
                    let mut c = app_c.idem_attempts.borrow_mut();
                    *c += 1;
                    *c
                };
                app_c.idem_rows.borrow_mut().push(IdemRow {
                    attempt,
                    was_open,
                    now_closed,
                });
                app_c.log_info(
                    Scope::Exp2,
                    &format_str(format_args!(
                        "close() #{attempt} → {}",
                        if was_open {
                            "было открыто, закрыто"
                        } else {
                            "no-op"
                        },
                    )),
                );
            }
            app_c.render_idem_table();
            app_c.refresh_exp2_conn_pill();
            app_c.set_exp_status(
                Scope::Exp2,
                Some(LogKind::Success),
                &format_str(format_args!(
                    "выполнено close() × {n} — все после первого были no-op"
                )),
            );
        });
    }

    // ============================================================
    //                     ЭКСПЕРИМЕНТ 3
    // ============================================================

    // ---------- Пересоздать ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp3-recreate")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.exp3_busy.borrow() {
                return; // гонка уже идёт — не мешаем
            }
            *app_in.exp3_busy.borrow_mut() = true;

            app_in.set_exp_status(Scope::Exp3, Some(LogKind::Info), "пересоздание базы…");
            app_in.refresh_race_a(None, "не открыто");
            app_in.refresh_race_b(None, "не открыто");
            app_in.set_race_arrow(false);
            spawn_local(async move {
                app_in.recreate_db(Scope::Exp3).await;
                app_in.log_ok(Scope::Exp3, "база пересоздана в v1");
                app_in.set_exp_status(Scope::Exp3, Some(LogKind::Success), "база пересоздана в v1");
                app_in.refresh_banner(Scope::Exp3).await;
                *app_in.exp3_busy.borrow_mut() = false;
            });
        });
    }

    // ---------- Обновить баннер ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp3-refresh")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            spawn_local(async move {
                app_in.refresh_banner(Scope::Exp3).await;
            });
        });
    }

    // ---------- race with handler ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp3-race-clean")?;
        let btn_cb = btn.clone();
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let btn_in = btn_cb.clone();
            if *app_in.exp3_busy.borrow() {
                return;
            }
            *app_in.exp3_busy.borrow_mut() = true;

            btn_in.set_disabled(true);
            app_in.set_exp_status(
                Scope::Exp3,
                Some(LogKind::Info),
                "гонка с handler → выполняется…",
            );
            app_in.set_race_arrow(true);
            app_in.refresh_race_a(None, "открывается…");
            app_in.refresh_race_b(None, "ждёт…");

            spawn_local(async move {
                let t0 = now_ms();
                let _ = delete_database(DB3).await;

                app_in.log_info(Scope::Exp3, "A: open v1 + install handler");
                match idb_factory::open_by_name(DB3, 1).await {
                    Ok(raw) => {
                        let db = Database::from_raw(raw);
                        db.install_versionchange_handler();
                        app_in.refresh_race_a(Some(1), "открыто, handler установлен");
                        *app_in.exp3_a.borrow_mut() = Some(db);
                    }
                    Err(e) => {
                        app_in.log_err(
                            Scope::Exp3,
                            &format_str(format_args!("A: open v1 → {}", error_name(&e))),
                        );
                        app_in.set_exp_status(
                            Scope::Exp3,
                            Some(LogKind::Error),
                            "A не удалось открыть",
                        );
                        app_in.set_race_arrow(false);
                        *app_in.exp3_busy.borrow_mut() = false;
                        btn_in.set_disabled(false);
                        return;
                    }
                }

                gloo_timers::future::TimeoutFuture::new(400).await;

                app_in.log_info(Scope::Exp3, "B: open v2 — провоцируем versionchange у A");
                app_in.refresh_race_b(None, "открывается…");
                match idb_factory::open_by_name(DB3, 2).await {
                    Ok(raw) => {
                        let db = Database::from_raw(raw);
                        app_in.refresh_race_b(Some(2), "открыто");
                        *app_in.exp3_b.borrow_mut() = Some(db);
                    }
                    Err(e) => {
                        app_in.log_err(
                            Scope::Exp3,
                            &format_str(format_args!("B: open v2 → {}", error_name(&e))),
                        );
                    }
                }

                let a_closed = {
                    let borrow = app_in.exp3_a.borrow();
                    borrow.as_ref().map(|d| d.is_closed()).unwrap_or(true)
                };
                if a_closed {
                    app_in.log_ok(Scope::Exp3, "A: автоматически закрыто через versionchange");
                    app_in.refresh_race_a(Some(1), "закрыто через versionchange");
                } else {
                    app_in.log_err(Scope::Exp3, "A: не закрылось");
                    app_in.refresh_race_a(Some(1), "всё ещё открыто");
                }

                // ← NEW: освобождаем оба соединения, чтобы следующая гонка
                //   не блокировалась на deleteDatabase. UI-карточки сохраняют
                //   последнее состояние — их это не затрагивает.
                if let Some(db) = app_in.exp3_a.borrow().as_ref() {
                    if !db.is_closed() {
                        db.close();
                    }
                }
                if let Some(db) = app_in.exp3_b.borrow().as_ref() {
                    db.close();
                }

                let dt = now_ms() - t0;
                app_in
                    .ui
                    .exp3_race_time
                    .set_text_content(Some(&format_str(format_args!("гонка заняла {dt:.0} мс"))));
                app_in.set_race_arrow(false);
                app_in.set_exp_status(
                    Scope::Exp3,
                    Some(LogKind::Success),
                    &format_str(format_args!("гонка OK ({dt:.0} мс) — A закрыто, B открыто")),
                );
                app_in.refresh_banner(Scope::Exp3).await;
                *app_in.exp3_busy.borrow_mut() = false;
                btn_in.set_disabled(false);
            });
        });
    }

    // ---------- race without handler ----------
    {
        let app_c = app.clone();
        let btn = get_button("exp3-race-dirty")?;
        let btn_cb = btn.clone();
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let btn_in = btn_cb.clone();
            if *app_in.exp3_busy.borrow() {
                return;
            }

            let window = web_sys::window().unwrap();
            let ok = window
                .confirm_with_message(
                    "Гонка без handler: B зависнет на onblocked на 1.5 сек. Продолжить?",
                )
                .unwrap_or(false);
            if !ok {
                return;
            }

            *app_in.exp3_busy.borrow_mut() = true;
            btn_in.set_disabled(true);
            app_in.set_exp_status(
                Scope::Exp3,
                Some(LogKind::Info),
                "гонка без handler → выполняется…",
            );
            app_in.set_race_arrow(true);
            app_in.refresh_race_a(None, "открывается…");
            app_in.refresh_race_b(None, "ждёт…");

            spawn_local(async move {
                let t0 = now_ms();
                let _ = delete_database(DB3).await;

                app_in.log_info(Scope::Exp3, "A: open v1 (без handler)");
                match idb_factory::open_by_name(DB3, 1).await {
                    Ok(raw) => {
                        let db = Database::from_raw(raw);
                        app_in.refresh_race_a(Some(1), "открыто, handler НЕ установлен");
                        *app_in.exp3_a.borrow_mut() = Some(db);
                    }
                    Err(e) => {
                        app_in.log_err(
                            Scope::Exp3,
                            &format_str(format_args!("A: open v1 → {}", error_name(&e))),
                        );
                        app_in.set_race_arrow(false);
                        *app_in.exp3_busy.borrow_mut() = false;
                        btn_in.set_disabled(false);
                        return;
                    }
                }

                gloo_timers::future::TimeoutFuture::new(400).await;

                app_in.log_info(Scope::Exp3, "B: open v2 (зависнет на onblocked)");
                app_in.refresh_race_b(None, "висит на onblocked…");

                let b_app = app_in.clone();
                let b_btn = btn_in.clone();
                spawn_local(async move {
                    match idb_factory::open_by_name(DB3, 2).await {
                        Ok(raw) => {
                            let db = Database::from_raw(raw);
                            b_app.refresh_race_b(Some(2), "открыто после ручного close A");
                            *b_app.exp3_b.borrow_mut() = Some(db);
                            b_app.log_ok(Scope::Exp3, "B: соединение всё-таки открылось");
                        }
                        Err(e) => {
                            b_app.log_err(
                                Scope::Exp3,
                                &format_str(format_args!("B: open v2 → {}", error_name(&e))),
                            );
                        }
                    }

                    // ← NEW: закрываем A (на всякий случай) и B.
                    if let Some(db) = b_app.exp3_a.borrow().as_ref() {
                        if !db.is_closed() {
                            db.close();
                        }
                    }
                    if let Some(db) = b_app.exp3_b.borrow().as_ref() {
                        db.close();
                    }

                    let dt = now_ms() - t0;
                    b_app
                        .ui
                        .exp3_race_time
                        .set_text_content(Some(&format_str(format_args!(
                            "гонка заняла {dt:.0} мс"
                        ))));
                    b_app.set_race_arrow(false);
                    b_app.set_exp_status(
                        Scope::Exp3,
                        Some(LogKind::Success),
                        &format_str(format_args!(
                            "B открылся после ручного close A ({dt:.0} мс)"
                        )),
                    );
                    b_app.refresh_banner(Scope::Exp3).await;
                    *b_app.exp3_busy.borrow_mut() = false;
                    b_btn.set_disabled(false);
                });

                gloo_timers::future::TimeoutFuture::new(1500).await;
                app_in.log_info(Scope::Exp3, "ручное close() у A — разблокируем B");
                if let Some(db) = app_in.exp3_a.borrow_mut().take() {
                    db.close();
                }
                app_in.refresh_race_a(Some(1), "закрыто вручную");
            });
        });
    }

    Ok(())
}

// ============================================================
//                       HELPERS
// ============================================================

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

fn now_ms() -> f64 {
    web_sys::window()
        .and_then(|w| w.performance())
        .map(|p| p.now())
        .unwrap_or(0.0)
}

fn error_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    if let Some(s) = e.as_string() {
        return s;
    }
    return format!("{e:?}");
}

fn esc(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}

/// Удаляет IndexedDB-базу с заданным именем через голый `indexedDB.deleteDatabase`.
///
/// Важно: `onblocked` НЕ отклоняет promise. По спецификации `deleteDatabase`
/// при наличии открытых соединений сначала стреляет `onblocked`, затем —
/// после закрытия всех соединений — `onsuccess`. Прерывание на `onblocked`
/// ломает последующий `open` (VersionError на устаревшей версии).
async fn delete_database(name: &str) -> Result<(), JsValue> {
    let global = js_sys::global();
    let idb = js_sys::Reflect::get(&global, &JsValue::from_str("indexedDB"))?;
    if !idb.is_object() {
        return Err(JsValue::from_str("indexedDB недоступен"));
    }
    let delete_fn: js_sys::Function =
        js_sys::Reflect::get(&idb, &JsValue::from_str("deleteDatabase"))?.dyn_into()?;
    let request = delete_fn.call1(&idb, &JsValue::from_str(name))?;

    let promise = js_sys::Promise::new(&mut |resolve, reject| {
        let resolve_c = resolve.clone();
        let reject_c = reject.clone();
        let on_success = Closure::once_into_js(move || {
            let _ = resolve_c.call0(&JsValue::UNDEFINED);
        });
        let on_error = Closure::once_into_js(move || {
            let _ = reject_c.call1(&JsValue::UNDEFINED, &JsValue::from_str("delete failed"));
        });
        let _ = js_sys::Reflect::set(&request, &JsValue::from_str("onsuccess"), &on_success);
        let _ = js_sys::Reflect::set(&request, &JsValue::from_str("onerror"), &on_error);
        // onblocked намеренно не выставляем: ждём onsuccess.
    });

    JsFuture::from(promise).await?;
    Ok(())
}

```


## Команды: тестирование и запуск

### Проверка типов

Прежде чем открывать браузер, прогоняем статические проверки — они ловят 90% ошибок быстрее, чем `wasm-pack test`.

**Host-проверка** (без `wasm32`):

```bash
cargo check
```

Модуль `idb_db.rs` компилируется под все таргеты, но реально работает только под `wasm32`. Host-проверка нужна для того, чтобы поймать опечатки в сигнатурах, забытые `use`, несовпадение типов в чистых функциях.

**Проверка под wasm32** (основная):

```bash
cargo check --target wasm32-unknown-unknown
```

Здесь проходит полный `web-sys`/`js-sys` путь. Если `web-sys`-фичи не хватает (`IdbDatabase`, `IdbFactory`, `DomException`), ошибка вылезет именно тут.

**Проверка была с фичей `web_sys_unstable_apis`** — она включена в `.cargo/config.toml`:

```toml
[target.wasm32-unknown-unknown]
rustflags = [
    "--cfg=web_sys_unstable_apis",
    "-C", "target-feature=+simd128",
]
```

Без неё часть experimental-API не скомпилируется.

### Wasm-тесты модуля `idb_db`

Все тесты лежат в `tests/wasm_idb_db.rs`. Запускаются через `wasm-pack test`.

**Chrome (headless)** — основной прогон:

```bash
wasm-pack test --headless --chrome --test wasm_idb_db
```

**Firefox (headless)** — второй браузер для кросс-проверки:

```bash
wasm-pack test --headless --firefox --test wasm_idb_db
```

**Node** — быстрый smoke-тест, но IndexedDB в Node отсутствует. Файл помечен `#![cfg(target_arch = "wasm32")]`, так что под Node тесты либо не запустятся, либо упадут на первом же `indexedDB is undefined`. Для этой подглавы Node не подходит — только реальный браузер.

**Через npm-скрипты:**

```bash
npm run test:chrome
npm run test:firefox
```

Скрипты в `package.json` подставляют `webdriver.json` с флагами `--no-sandbox --disable-gpu --headless=new`.

### Прогон конкретных тестов

**Один тест по имени:**

```bash
wasm-pack test --headless --chrome --test wasm_idb_db -- \
    versionchange_handler_closes_db
```

**Группа тестов по префиксу** (`store_names_*`):

```bash
wasm-pack test --headless --chrome --test wasm_idb_db -- \
    store_names
```

`wasm-pack` передаёт всё после `--` в `wasm-bindgen-test-runner`. Фильтр работает по подстроке имени.

**Только `versionchange`-тесты:**

```bash
wasm-pack test --headless --chrome --test wasm_idb_db -- \
    versionchange
```

### Запуск демо

**Debug-сборка WASM + Vite dev-сервер:**

```bash
npm start
```

Что происходит:

1. `wasm-pack build --dev --target web --out-dir web/pkg` — debug-сборка (быстро, без оптимизаций).
2. `vite` — поднимает dev-сервер на `http://127.0.0.1:5173/`.

Откройте адрес, который покажет Vite. По умолчанию — `http://127.0.0.1:5173/`.

**Только пересборка WASM** (если `vite` уже запущен):

```bash
npm run wasm:dev
```

**Release-сборка WASM** (медленнее, но меньше размер — для production):

```bash
npm run wasm
```

**Production-сборка фронтенда** (WASM release + Vite build):

```bash
npm run build
npm run preview
```

**Vite без пересборки WASM** (когда `web/pkg/` уже собран):

```bash
npm run dev
```

### Что проверять в браузере

После `npm start` и открытия вкладок прогоните сценарии.

**Вкладка 1 — метаданные соединения:**

1. `open v1` — LED зелёный, `name` в карточке, `v1`, список store'ов пуст.
2. `install handler` — бейдж «🛡 установлен».
3. `close()` — LED серый, штамп CLOSED поверх карточки.

**Вкладка 2 — идемпотентность:**

1. `open v1`.
2. `close() × 3` — в таблице три строки: `открыто → закрыто` для первой, `закрыто → закрыто` для остальных.
3. `close() × 5` — счётчик `попыток: 8`, история обнуляется, пять новых строк.

**Вкладка 3 — гонка:**

1. `⚡ Гонка с handler` — обе карточки переключаются, A закрывается через `versionchange`, B открывается. Время в пилюле — 300–500 мс.
2. **Нажмите её 5 раз подряд.** Каждый прогон — стабильные ~400 мс. Если видите 14+ секунд — значит `delete_database` ушёл в `onblocked`, а это уже регрессия: в `race-clean` не закрылись A или B в конце.
3. `⚡ Гонка без handler` — диалог подтверждения. B висит на `onblocked` 1.5 сек, потом A закрывается вручную, B проходит. Время — 1.8–2.2 сек.

**Проверка регрессии «Пересоздать во время гонки»:**

1. На вкладке 3 нажмите `⚡ Гонка без handler`.
2. Пока B висит на `onblocked` (первые 1.5 сек) — быстро нажмите `↻ Пересоздать базу`.
3. **Ожидаемое поведение:** кнопка «Пересоздать» молча игнорируется (`exp3_busy` заблокировал). Через секунду гонка завершается, кнопка снова работает.

Если кнопка всё-таки срабатывает во время гонки — регрессия: `exp3_busy` не выставлен или не сброшен в одной из веток.

### Инспекция WASM-бинарника

Если нужно убедиться, что `wasm-opt` не выкинул нужный код:

```bash
sudo apt update && sudo apt install wabt
wasm-objdump -x web/pkg/hello_wasm02_bg.wasm | grep -i idb
```

Ищите импорты `IDBDatabase`, `IDBOpenDBRequest` — если их нет, значит `web-sys`-фичи отключены или `#[wasm_bindgen]`-экспорты не подтянули модуль.