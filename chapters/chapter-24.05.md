# Глава 24.5. `IdbObjectStore` — записи, курсоры, диапазоны, связи

## 24.5.0. Что вы получите в этой главе

24.4 дал нам **схему**: store'ы и индексы. Это был разговор про контракт базы — что в ней есть. 24.5 — про то, **что внутри store'а**.

`IDBObjectStore` — keyed-коллекция структурированных записей. Каждая запись — произвольный JS-объект, и у каждой есть ключ. Ключ определяет идентичность: занять его один раз или перезаписать, найти или удалить.

Заметьте: модуль называется **`idb_records.rs`**, а не `idb_store.rs`. Первый уже существует в проекте — это File-хранилище через crate `idb` (Глава 18). Мы не пересекаемся ни по имени, ни по API, ни по смыслу.

К концу главы у вас будет интерактивная песочница над реальной моделью данных:

1. **Четыре store** с осмысленными связями: `authors` (1:N) → `books` (M:N) ↔ `tags` через `book_tags`.
2. **CRUD со связями** — добавление, каскадное удаление, удаление одной связи без потери книги и тега.
3. **Курсор с yield** — итерация по 500 книгам с плавным прогрессом.
4. **Диапазоны ключей** — `IDBKeyRange` фильтрует на стороне движка.
5. **Миграции** через `cursor.update()` — добавить поле всем книгам без `ALTER TABLE`.
6. **Реальная библиотека** — 100 русских писателей, ~370 книг, жанры как теги.

## 24.5.1. Структура файлов

```
hello-wasm02/
├── src/
│   ├── idb_records.rs       # НОВЫЙ: обёртка над IDBObjectStore
│   ├── idb_schema.rs        # из 24.4
│   ├── idb_db.rs            # из 24.3
│   ├── idb_factory.rs       # из 24.2
│   ├── idb_store.rs         # из 18.x — File-хранилище, НЕ трогаем
│   ├── lib.rs               # +1 модуль
│   └── app.rs               # демо — песочница над 4 store'ами
├── tests/
│   └── wasm_idb_records.rs  # НОВЫЙ: 11 wasm-тестов
└── Cargo.toml               # без изменений
```

**Ссылки для детального изучения:**

- MDN, `IDBObjectStore` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore
- MDN, `IDBObjectStore.openCursor()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/openCursor
- MDN, `IDBKeyRange` — https://developer.mozilla.org/en-US/docs/Web/API/IDBKeyRange
- MDN, `IDBCursor.update()` — https://developer.mozilla.org/en-US/docs/Web/API/IDBCursor/update
- Spec, «Object Store Storage Operation» — https://w3c.github.io/IndexedDB/#object-store-storage-operation
- Spec, «Key construct» (составные ключи) — https://w3c.github.io/IndexedDB/#key-construct

## 24.5.2. Модель: store как keyed-коллекция

Интуиция, которая помогает сразу: **`IDBObjectStore` — это `Map<Key, Value>`, где `Value` — структурированный объект.** Всё остальное вырастает из этой аналогии:

- `add(k, v)` → `Map.set` с проверкой «ключа нет».
- `put(k, v)` → `Map.set` без проверки.
- `get(k)` → `Map.get`.
- `getAll()` → массив значений.
- `delete(k)` → `Map.delete` (идемпотентно).
- `count()` → размер.
- `openCursor()` → итератор.

Два отличия от `Map`:

**1. Ключ может быть частью значения.** Если у store'а задан `keyPath: "isbn"`, вы не передаёте ключ — вы кладёте его в объект. При `add(record)` браузер возьмёт `record.isbn` как ключ.

**2. Ключ — типизированное значение.** Не просто строка. Порядок ключей определяется structured key algorithm: `number < date < string < binary < array`. Мы это видели в 24.2 через `compare_keys`. Здесь — используем.

## 24.5.3. Ключи: типы и источники

### Источники ключа

| Источник | Когда | Пример |
|---|---|---|
| `keyPath` | Store создан с полем-ключом | `StoreSpec::with_key("books", "isbn")` |
| `autoIncrement` | Store без `keyPath`, `autoIncrement: true` | `StoreSpec::with_auto_key("logs")` |
| Явный ключ | Второй аргумент `put(record, key)` | `put_with_key(store, &rec, &JsValue::from_f64(42))` |

При `keyPath` браузер сам берёт поле объекта. При `autoIncrement` генерирует число и возвращает его в `result` запроса — единственный способ узнать сгенерированный ключ. Явный ключ используется, только когда у store'а нет `keyPath`.

### Типы ключей

IndexedDB принимает пять типов: **number**, **date**, **string**, **binary**, **array**.

| Тип | JS | Rust |
|---|---|---|
| number | `42` | `JsValue::from_f64(42.0)` |
| date | `new Date()` | `JsValue::from(js_sys::Date::new(&...))` |
| string | `"hello"` | `JsValue::from_str("hello")` |
| binary | `ArrayBuffer` | `Uint8Array::from(bytes).buffer().into()` |
| array | `[1, "two", true]` | `js_sys::Array::new().into()` |

Массив — **составной ключ**. Именно он позволяет связывать две сущности M:N через промежуточный store: `book_tags` имеет `keyPath: "book_isbn,tag_id"`, и ключ каждой записи — `["978-1", "classic"]`.

### Типизация в Rust

`JsValue::as_f64()` даёт `Some` только для числа. Строка остаётся строкой, дата — `Date`, `ArrayBuffer` — бинарный. Structured clone сохраняет тип через границу: `put({year: 2020})` вернёт `get()` с `year: 2020`, а не `year: "2020"`.

## 24.5.4. CRUD: базовые операции

Все публичные методы `idb_records` возвращают `Result<..., JsValue>`. Ошибка приходит как `DOMException` — не как строка.

```rust
pub async fn add(store: &RecordStore, record: &JsValue) -> Result<JsValue, JsValue>;
pub async fn put(store: &RecordStore, record: &JsValue) -> Result<JsValue, JsValue>;
pub async fn get(store: &RecordStore, key: &JsValue) -> Result<Option<JsValue>, JsValue>;
pub async fn count(store: &RecordStore) -> Result<u32, JsValue>;
pub async fn delete(store: &RecordStore, key: &JsValue) -> Result<(), JsValue>;
pub async fn clear(store: &RecordStore) -> Result<(), JsValue>;
pub async fn get_all(store: &RecordStore) -> Result<Vec<JsValue>, JsValue>;
pub async fn get_all_keys(store: &RecordStore) -> Result<Vec<JsValue>, JsValue>;
```

### `add` vs `put`

Главная семантическая развилка. `add` — строгая вставка, падает с `ConstraintError`, если ключ уже занят. `put` — upsert: создаёт или перезаписывает.

Правило для UI: если пользователь ввёл данные в форму и нажал «добавить» — `add`, чтобы он увидел `ConstraintError` и понял, что ключ занят. Если он редактирует существующую запись — `put`.

### `get` возвращает `Option`

```rust
match idb_records::get(&store, &JsValue::from_str("no-such")).await? {
    Some(rec) => /* запись есть */,
    None => /* ключа нет */,
}
```

`None` — это **успешный** результат. Отсутствие записи не ошибка. `IDBRequest.result` равен `undefined`, когда ключ не найден.

### `getAllKeys` без значений

Когда нужны только ключи (например, для отображения списка) — `getAllKeys` дешевле `getAll`. Движок не сериализует объекты.

### `count`

`count()` без аргументов возвращает число записей в store. Быстрее, чем `getAll().len()`, потому что движок не читает сами записи.

## 24.5.5. Bulk-операции: `seed_bulk`

Если записать 1000 записей по одной, вы сделаете 1000 транзакций. Транзакция IndexedDB имеет накладные расходы: открытие, планирование, коммит. В нашем демо seed 1000 книг в одной транзакции — **286 мс**, а по одной — минуты.

```rust
pub async fn seed_bulk(store: &RecordStore, records: &[JsValue]) -> Result<(), JsValue> {
    // Подписываемся на tx.oncomplete ДО первого запроса.
    let complete = tx_complete_promise(&store.tx);
    let put_fn = method(&store.raw, "put")?;
    // Все put'ы летят синхронно — транзакция остаётся активной.
    for rec in records {
        put_fn.call1(&store.raw, rec)?;
    }
    // Ждём коммит.
    JsFuture::from(complete).await?;
    Ok(())
}
```

**Правило.** Одна async-операция = одна транзакция. `await` на промежуточных `put`'ах разорвёт транзакцию: IndexedDB закрывает её, когда microtask-очередь опустеет без pending-запросов. Поэтому все `put` синхронно, потом один `await`.

## 24.5.6. Курсоры: итерация по store

`getAll` возвращает всё одним массивом — удобно для небольших store'ов. Для больших нужен курсор: одна запись → callback → `continue()`.

```rust
pub async fn cursor_visit<F>(
    store: &RecordStore,
    range: Option<KeyRangeSpec>,
    direction: Direction,
    limit: Option<u32>,
    on_progress: F,
) -> Result<CursorResult, JsValue>
where F: FnMut(u32);
```

### Направление

```rust
pub enum Direction {
    Next,        // от меньшего ключа к большему
    Prev,        // обратный обход
    NextUnique,  // пропускать дубликаты
    PrevUnique,
}
```

**Критично:** `openCursor(null, "prev")` — обязательно передавать direction вторым аргументом. `openCursor()` без аргументов всегда использует `next`, и `prev` теряется.

### `cursor.update()` и `cursor.delete()`

Курсор в режиме `readwrite` даёт методы `update(record)` и `delete()` — они изменяют запись, на которой стоит курсор. Это единственный способ массового изменения без повторного `get` + `put`.

```js
cursor.update(newValue);   // перезаписать текущую запись
cursor.delete();           // удалить текущую запись
cursor.continue();         // перейти к следующей
```

### Yield: почему UI замирает

`onsuccess` курсора работает в microtask-очереди. Браузер обрабатывает всю очередь подряд, не перерисовывая страницу. Если итерация длится 100 мс — UI заморожен на 100 мс.

Решение — **периодически отдавать управление через `TimeoutFuture::new(0)`**. Это `setTimeout(0)` — macrotask. Между ним и следующим macrotask'ом браузер обрабатывает рендер-пайплайн.

Правило проекта: **yield раз в 16 мс** (≈ 60 fps). Проверка `now - last_yield >= 16.0` занимает доли микросекунды, а обновление прогресс-бара происходит ровно столько раз, сколько браузер успевает отрисовать.

### `limit`: проверка в `on_success`, не в yield-loop

Первая версия проверяла лимит между yield'ами:

```rust
loop {
    if state.borrow().done { break; }
    if let Some(lim) = limit {
        if state.borrow().keys.len() as u32 >= lim { break; }
    }
    TimeoutFuture::new(0).await;
    // ...
}
```

За один макротакт курсор продвигается десятки раз — 50 записей легко превращаются в 78. Правильно — проверять в `on_success` **до** вызова `cursor.continue()`:

```rust
let should_stop = {
    let mut st = state.borrow_mut();
    st.keys.push(key);
    st.values.push(value);
    match limit_cell.get() {
        Some(lim) => st.keys.len() as u32 >= lim,
        None => false,
    }
};
if should_stop {
    state_cell.borrow_mut().done = true;
    trunc_cell.set(true);   // отдельный флаг, что остановились по лимиту
    return;                 // НЕ вызываем continue()
}
```

И `truncated: bool` в `CursorResult` — отдельный флаг. Он не вычисляется как `scanned >= limit`, потому что `scanned == limit` бывает и при естественном конце.

## 24.5.7. `IDBKeyRange`: фильтр на движке

`IDBKeyRange` — не массив и не фильтр в коде. Это объект, который говорит движку: «дай только записи в этом диапазоне». Курсор не читает записи вне диапазона — они не попадают даже в память.

```rust
pub enum KeyRangeSpec {
    Only(JsValue),
    LowerBound { value: JsValue, open: bool },
    UpperBound { value: JsValue, open: bool },
    Bound { lower: JsValue, upper: JsValue, lower_open: bool, upper_open: bool },
}
```

Создаётся через **статические методы конструктора**, не через `new IDBKeyRange(...)`:

```rust
let kr = js_sys::Reflect::get(&global, &"IDBKeyRange".into())?;
let f: js_sys::Function = js_sys::Reflect::get(&kr, &"bound".into())?
    .dyn_into()?;
f.call4(&kr, &lo, &hi, &lower_open, &upper_open)?;
```

| Метод | Диапазон |
|---|---|
| `only(v)` | ровно `v` |
| `lowerBound(v, open)` | `≥ v` или `> v` |
| `upperBound(v, open)` | `≤ v` или `< v` |
| `bound(lo, hi, open_lo, open_hi)` | `[lo, hi]` или `(lo, hi]` и т.д. |

### Практический эффект

В нашем демо `books.isbn` — строка. `bound("978-000000", "978-000100")` отберёт только первые 100 книг из 500. Курсор не читает остальные 400.

Для числовых ключей (например, `autoIncrement`) `bound(100, 200)` пройдёт 101 запись. Для смешанных типов порядок — `number < date < string < binary < array`. Это уже разбирали в 24.2.

## 24.5.8. `cursor.update`: миграции без `ALTER TABLE`

В IndexedDB нет фиксированной схемы записей. `books` в v1 и `books` в v5 могут содержать разные поля. Добавить поле всем записям — это обход курсором с `cursor.update()`:

```rust
async fn cursor_update_all<F>(
    db: &JsValue,
    store_name: &str,
    transform: F,   // F: FnMut(&JsValue) -> Option<JsValue>
    ...
) -> Result<(u32, u32), JsValue>;
```

`transform` получает старую запись, возвращает новую (или `None`, если пропустить). `cursor_update_all` открывает транзакцию, итерирует курсором, вызывает `update()` для каждой непустой трансформации.

### Счётчик «без изменений»

Если поле уже такое, каким его хотят сделать, `update()` не нужен. Сравниваем через `stringify_js`:

```rust
fn field_value_changed(old: &JsValue, new_obj: &JsValue, field: &str) -> bool {
    let old_v = js_sys::Reflect::get(old, &field.into()).unwrap_or(JsValue::UNDEFINED);
    let new_v = js_sys::Reflect::get(new_obj, &field.into()).unwrap_or(JsValue::UNDEFINED);
    return stringify_js(&old_v) != stringify_js(&new_v);
}
```

Сравнение через `JSON.stringify` ловит `undefined`, `null`, числа, строки, вложенные объекты. `PartialEq` для `JsValue` в Rust не определён, а `Object::is` сравнивает по identity, а не по значению.

**Правило.** Пропускать уже-правильные записи — не оптимизация, а часть контракта. Пользователь видит «обновлено 30 / без изменений 0». На повторном прогоне — «обновлено 0 / без изменений 30». Это и есть идемпотентность миграции.

## 24.5.9. Связи 1:N и M:N

IndexedDB — не реляционная база. Здесь нет `FOREIGN KEY` и `ON DELETE CASCADE`. Связи моделируются вручную, но именно это даёт гибкость.

### 1:N — через `author_id` в дочернем store

```rust
// authors: keyPath = "id"
// books:   keyPath = "isbn", индекс by_author по "author_id"

let book = js_sys::Object::new();
Reflect::set(&book, &"isbn".into(), &"978-1".into())?;
Reflect::set(&book, &"author_id".into(), &"pushkin".into())?;
```

Один автор — много книг. Индекс `by_author` даёт быстрый поиск книг автора за `O(log n + k)`.

### M:N — через промежуточный store с составным ключом

```rust
// book_tags: keyPath = "book_isbn,tag_id"
// индекс by_book (book_isbn), индекс by_tag (tag_id)
```

Запись в `book_tags` — это **ребро** между книгой и тегом. Составной ключ `["978-1", "classic"]` гарантирует, что одна пара книга-тег не появится дважды. `put` идемпотентен: повторная запись того же ребра — no-op.

### Каскадное удаление — вручную, но в одной транзакции

Удалить автора — значит:

1. Найти все его книги по индексу `by_author`.
2. Удалить все их рёбра в `book_tags` (по `by_book`).
3. Удалить сами книги.
4. Удалить автора.

Всё в **одной транзакции на три store**: `["authors", "books", "book_tags"]`. Если что-то упадёт на шаге 3 — откатятся и шаги 1, 2. Целостность сохранится.

```rust
let tx = db.transaction(&["authors", "books", "book_tags"], "readwrite")?;
// ... операции ...
JsFuture::from(tx_complete_promise(&tx)).await?;
```

**Правило.** Одна логическая операция = одна транзакция, даже если она затрагивает три store. Если делать по транзакции на store, при сбое посередине каталог останется в рассогласованном состоянии: книги удалены, рёбра висят.

## 24.5.10. Составные ключи

Составной ключ в `keyPath` задаётся строкой с запятой: `"book_isbn,tag_id"`. Планировщик схемы парсит её в JS-массив:

```rust
fn parse_key_path(kp: &str) -> JsValue {
    if kp.contains(',') {
        let arr = js_sys::Array::new();
        for part in kp.split(',') {
            arr.push(&JsValue::from_str(part.trim()));
        }
        return arr.into();
    }
    return JsValue::from_str(kp);
}
```

При `put({book_isbn: "978-1", tag_id: "classic"})` браузер возьмёт ключ `["978-1", "classic"]`. При повторном `put` того же объекта — no-op. При `delete` — нужен тот же массив:

```rust
let key_arr = js_sys::Array::new();
key_arr.push(&JsValue::from_str(&isbn));
key_arr.push(&JsValue::from_str(&tag));
idb_records::delete(&store, &key_arr.into()).await?;
```

**Правило.** Для составного ключа `delete`/`get` принимают массив. Простая строка `"978-1,classic"` не сработает — это будет другой ключ.

### Порядок частей важен

`["a", "b"]` и `["b", "a"]` — разные ключи. Порядок в `keyPath` — часть контракта store'а, менять его после создания нельзя.

## 24.5.11. Транзакции: четыре правила

1. **Одна операция = одна транзакция.** Каждый публичный метод (`add`, `put`, `get`, …) открывает свою транзакцию, делает работу и ждёт `tx.oncomplete`.

2. **`RecordStore` — binding, не транзакция.** `RecordStore::open(db, name, mode)` запоминает `db`, `name`, `mode`, но не открывает транзакцию. Каждая операция сама вызывает `open_tx()`. Так один `RecordStore` переиспользуется в цикле без `TransactionInactiveError`.

3. **Подписка на `tx.oncomplete` — до первого запроса.** Между последним `onsuccess` и `oncomplete` проходит минимум одна task. Если подписаться после первого `put` — await может зависнуть, потому что событие уже улетело.

4. **`tx_complete_promise` резолвится и на `onabort`.** Если запрос падает (например, `ConstraintError`), транзакция уходит в abort. Резолвить на abort — не баг, а фича: об ошибке знает сам request, а `tx_complete` сигнализирует только «соединение освобождено». Иначе `deleteDatabase` в cleanup уйдёт в `onblocked`.

### `db.close()` не отменяет транзакции

Синхронный `close()` фактически ставит флаг «закрыть после завершения активных транзакций». Если транзакция в полёте — соединение остаётся активным до её конца. Именно это ловили в тестах: `cleanup: blocked: соединение держится другим табом`.

## 24.5.12. Практические находки

**1. `<details>` не должен быть flex-item.** Известный баг Chromium: если `<summary>` в flex-контейнере, содержимое `<details>` не отрисовывается. `.rc-tree` — `display: block`, `details` — `display: block; margin-bottom: 6px`, `<summary>` — `display: flex`, но **без** `<div>` внутри (только `<span>` и `<button>`).

**2. `format_args!` с длинными литералами — разбивайте на `push_str`.** Если HTML внутри литерала содержит `>` двух вложенных `"`, парсер легко путает `}` от HTML-атрибута с закрытием блока. Разбиение на 5 коротких строк избавляет от mismatched-delimiter.

**3. `Reflect::get` вместо типизированных вызовов web-sys.** `IDBFactory.databases()`, `IDBObjectStore.getAll()`, `IDBCursor.update()` — не всегда есть в `web-sys`. Работаем через `Reflect::get` + `Function::call*`. Единый стиль для всех версий библиотеки.

**4. Клоны для `Closure::wrap` и `Reflect::set`.** Если JS-функция (`resolve`, `reject`) используется в N замыканиях — каждому нужен свой `clone()`. `Function` не `Copy`, первое замыкание «забирает» значение.

**5. `Closure::once_into_js` vs `Closure::wrap`.** `onsuccess`/`onerror` запроса — `once_into_js` (события приходят один раз). `onversionchange`, `oncomplete` — `wrap` + `forget` (могут прийти повторно или жить всё время работы соединения).

**6. `format!` в tail-позиции** — только через `format_str(format_args!(...))` или явный `return`. rust-analyzer стабильно теряет тип в этой позиции.

**7. Field `kept` в `VersionRecord`** удалён — dead_code накапливается и маскирует реальные проблемы.

**8. Индекс vs полное сканирование.** В 24.4 мы завели `by_author`, `by_year`, `by_book`, `by_tag`. В 24.5 используем `by_author` для каскадного удаления. Полное сканирование `get_all` + фильтр в Rust — работает, но O(n). Индекс — O(log n + k).

**9. DSL для seed-данных.** Массив Rust-кортежей на 100 авторов и 370 книг — 4000+ строк нечитаемого кода. Компактный DSL (автор + отступ-книги) занимает 600 строк и проверяется глазами. Парсер — 30 строк Rust.

**10. Каскадное удаление — не оптимизация.** Делать его через `get_all` + `filter` в Rust проще, чем через индексы, но на 100 000 записей это секунды. В демо-данных разница незаметна, но правило: индексы — для связей, полное сканирование — только для отладки.

**idb_records.rs:**
```
//! Операции над `IDBObjectStore` через `js_sys::Reflect`.
//!
//! ПРАВИЛО: `RecordStore` — это binding на store (db + name + mode),
//! а НЕ на транзакцию. Каждая async-операция открывает свою транзакцию
//! и ждёт её `oncomplete`. Так один и тот же `RecordStore` переиспользуется
//! в цикле без `TransactionInactiveError`.
//!
//! ПРАВИЛО: подписываемся на `tx.oncomplete` ДО первого запроса. Иначе
//! событие улетает в предыдущий макротакт, и `await` висит навсегда.
//!
//! ПРАВИЛО: `openCursor(range, direction)` — direction передаём всегда.
//! Без второго аргумента браузер жёстко использует "next".
//!
//! ПРАВИЛО: лимит итерации проверяем внутри `on_success`, до `continue()`.
//! Проверка между yield'ами ненадёжна: за один макротакт курсор
//! продвигается десятки раз.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

// ============================================================
//                       RecordStore
// ============================================================

/// Binding на object store: db + name + mode.
///
/// Не хранит транзакцию: каждая операция (add/put/get/cursor)
/// открывает свою. Это даёт переиспользование handle'а без
/// `TransactionInactiveError`.
pub struct RecordStore {
    db: JsValue,
    name: String,
    mode: String,
}

impl RecordStore {
    /// Принимает `db.raw()` (сам `IDBDatabase`), имя store'а,
    /// строку `"readonly"` или `"readwrite"`.
    pub fn open(db: &JsValue, name: &str, mode: &str) -> Result<Self, JsValue> {
        let mode_s = match mode {
            "readonly" | "readwrite" => mode.to_string(),
            other => {
                return Err(JsValue::from_str(&format!(
                    "open: mode должен быть readonly/readwrite, получено {other:?}"
                )));
            }
        };
        Ok(Self {
            db: db.clone(),
            name: name.to_string(),
            mode: mode_s,
        })
    }

    pub fn name(&self) -> &str { &self.name }
    pub fn mode(&self) -> &str { &self.mode }

    /// Открывает транзакцию и берёт в ней object store.
    /// Возвращает `(objectStore, transaction)`.
    fn open_tx(&self) -> Result<(JsValue, JsValue), JsValue> {
        return open_tx_and_store(&self.db, &self.name, &self.mode);
    }
}

fn open_tx_and_store(
    db: &JsValue,
    name: &str,
    mode: &str,
) -> Result<(JsValue, JsValue), JsValue> {
    let tx_fn: js_sys::Function =
        js_sys::Reflect::get(db, &JsValue::from_str("transaction"))
            .map_err(|e| JsValue::from_str(&format!("Reflect transaction: {e:?}")))?
            .dyn_into()
            .map_err(|_| JsValue::from_str("transaction не функция"))?;
    let names = js_sys::Array::new();
    names.push(&JsValue::from_str(name));
    let tx = tx_fn
        .call2(db, &names, &JsValue::from_str(mode))
        .map_err(|e| JsValue::from_str(&format!("transaction: {e:?}")))?;
    let os_fn: js_sys::Function =
        js_sys::Reflect::get(&tx, &JsValue::from_str("objectStore"))
            .map_err(|e| JsValue::from_str(&format!("Reflect objectStore: {e:?}")))?
            .dyn_into()
            .map_err(|_| JsValue::from_str("objectStore не функция"))?;
    let os = os_fn
        .call1(&tx, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("objectStore: {e:?}")))?;
    Ok((os, tx))
}

// ============================================================
//                       Direction
// ============================================================

#[derive(Clone, Copy, Debug, PartialEq)]
pub enum Direction {
    Next,
    Prev,
    NextUnique,
    PrevUnique,
}

impl Direction {
    pub fn as_str(&self) -> &'static str {
        match self {
            Direction::Next => "next",
            Direction::Prev => "prev",
            Direction::NextUnique => "nextunique",
            Direction::PrevUnique => "prevunique",
        }
    }
}

// ============================================================
//                       KeyRangeSpec
// ============================================================

#[derive(Clone, Debug)]
pub enum KeyRangeSpec {
    Only(JsValue),
    LowerBound { value: JsValue, open: bool },
    UpperBound { value: JsValue, open: bool },
    Bound { lower: JsValue, upper: JsValue, lower_open: bool, upper_open: bool },
}

impl KeyRangeSpec {
    pub fn to_js(&self) -> Result<JsValue, JsValue> {
        let kr = key_range_ctor()?;
        match self {
            KeyRangeSpec::Only(v) => {
                let f: js_sys::Function = js_sys::Reflect::get(&kr, &"only".into())?
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("only не функция"))?;
                f.call1(&kr, v)
                    .map_err(|e| JsValue::from_str(&format!("IDBKeyRange.only: {e:?}")))
            }
            KeyRangeSpec::LowerBound { value, open } => {
                let f: js_sys::Function = js_sys::Reflect::get(&kr, &"lowerBound".into())?
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("lowerBound не функция"))?;
                f.call2(&kr, value, &JsValue::from_bool(*open))
                    .map_err(|e| JsValue::from_str(&format!("IDBKeyRange.lowerBound: {e:?}")))
            }
            KeyRangeSpec::UpperBound { value, open } => {
                let f: js_sys::Function = js_sys::Reflect::get(&kr, &"upperBound".into())?
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("upperBound не функция"))?;
                f.call2(&kr, value, &JsValue::from_bool(*open))
                    .map_err(|e| JsValue::from_str(&format!("IDBKeyRange.upperBound: {e:?}")))
            }
            KeyRangeSpec::Bound { lower, upper, lower_open, upper_open } => {
                let f: js_sys::Function = js_sys::Reflect::get(&kr, &"bound".into())?
                    .dyn_into()
                    .map_err(|_| JsValue::from_str("bound не функция"))?;
                f.call4(
                    &kr,
                    lower,
                    upper,
                    &JsValue::from_bool(*lower_open),
                    &JsValue::from_bool(*upper_open),
                )
                .map_err(|e| JsValue::from_str(&format!("IDBKeyRange.bound: {e:?}")))
            }
        }
    }
}

// ============================================================
//                       CursorResult
// ============================================================

#[derive(Clone, Debug)]
pub struct CursorResult {
    pub scanned: u32,
    pub keys: Vec<JsValue>,
    pub values: Vec<JsValue>,
    pub elapsed_ms: f64,
    pub truncated: bool,
}

// ============================================================
//                       CRUD
//
// Каждая операция открывает собственную транзакцию и ждёт её
// `oncomplete`. Один `RecordStore` можно переиспользовать в цикле.
// ============================================================

pub async fn add(store: &RecordStore, record: &JsValue) -> Result<JsValue, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "add")?;
    let req = f
        .call1(&os, record)
        .map_err(|e| JsValue::from_str(&format!("add: {e:?}")))?;
    let res = await_request(&req).await;
    // Дожидаемся завершения транзакции В ЛЮБОМ СЛУЧАЕ — даже при ошибке
    // request'а (ConstraintError → abort). Иначе соединение остаётся
    // активным и следующий `deleteDatabase` уходит в `onblocked`.
    JsFuture::from(complete).await.ok();
    res
}

pub async fn add_with_key(
    store: &RecordStore,
    record: &JsValue,
    key: &JsValue,
) -> Result<JsValue, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "add")?;
    let req = f
        .call2(&os, record, key)
        .map_err(|e| JsValue::from_str(&format!("add: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    res
}

pub async fn put(store: &RecordStore, record: &JsValue) -> Result<JsValue, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "put")?;
    let req = f
        .call1(&os, record)
        .map_err(|e| JsValue::from_str(&format!("put: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    res
}

pub async fn delete(store: &RecordStore, key: &JsValue) -> Result<(), JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "delete")?;
    let req = f
        .call1(&os, key)
        .map_err(|e| JsValue::from_str(&format!("delete: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    res.map(|_| ())
}

pub async fn clear(store: &RecordStore) -> Result<(), JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "clear")?;
    let req = f
        .call0(&os)
        .map_err(|e| JsValue::from_str(&format!("clear: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    res.map(|_| ())
}

pub async fn get(store: &RecordStore, key: &JsValue) -> Result<Option<JsValue>, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "get")?;
    let req = f
        .call1(&os, key)
        .map_err(|e| JsValue::from_str(&format!("get: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    match res {
        Ok(v) if v.is_null() || v.is_undefined() => Ok(None),
        Ok(v) => Ok(Some(v)),
        Err(e) => Err(e),
    }
}

pub async fn count(store: &RecordStore) -> Result<u32, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "count")?;
    let req = f
        .call0(&os)
        .map_err(|e| JsValue::from_str(&format!("count: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    res.map(|v| v.as_f64().unwrap_or(0.0) as u32)
}

pub async fn get_all(store: &RecordStore) -> Result<Vec<JsValue>, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "getAll")?;
    let req = f
        .call0(&os)
        .map_err(|e| JsValue::from_str(&format!("getAll: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    match res {
        Ok(v) => array_to_vec(v),
        Err(e) => Err(e),
    }
}

pub async fn get_all_keys(store: &RecordStore) -> Result<Vec<JsValue>, JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let f = method(&os, "getAllKeys")?;
    let req = f
        .call0(&os)
        .map_err(|e| JsValue::from_str(&format!("getAllKeys: {e:?}")))?;
    let res = await_request(&req).await;
    JsFuture::from(complete).await.ok();
    match res {
        Ok(v) => array_to_vec(v),
        Err(e) => Err(e),
    }
}

pub async fn seed_bulk(store: &RecordStore, records: &[JsValue]) -> Result<(), JsValue> {
    let (os, tx) = store.open_tx()?;
    let complete = tx_complete_promise(&tx);
    let put_fn = method(&os, "put")?;
    let mut first_err: Option<JsValue> = None;
    for rec in records {
        if let Err(e) = put_fn.call1(&os, rec) {
            if first_err.is_none() {
                first_err = Some(e);
            }
            break; // дальнейшие put'ы в abort-транзакцию бессмысленны
        }
    }
    JsFuture::from(complete).await.ok();
    match first_err {
        Some(e) => Err(e),
        None => Ok(()),
    }
}

// ============================================================
//                       CURSOR
// ============================================================

pub async fn cursor_visit<F>(
    store: &RecordStore,
    range: Option<KeyRangeSpec>,
    direction: Direction,
    limit: Option<u32>,
    mut on_progress: F,
) -> Result<CursorResult, JsValue>
where
    F: FnMut(u32),
{
    let t0 = now_ms();
    let (os, tx) = store.open_tx()?;

    // Подписываемся на tx.oncomplete ДО первого касания курсора.
    // Между последним `null`-результатом курсора и коммитом транзакции
    // проходит как минимум одна task; если подписаться позже, событие
    // `oncomplete` уже улетит, и await зависнет.
    let tx_complete = tx_complete_promise(&tx);

    struct State {
        keys: Vec<JsValue>,
        values: Vec<JsValue>,
        done: bool,
        error: Option<JsValue>,
    }

    let state = Rc::new(RefCell::new(State {
        keys: Vec::new(),
        values: Vec::new(),
        done: false,
        error: None,
    }));
    let limit_cell: Rc<Cell<Option<u32>>> = Rc::new(Cell::new(limit));
    // Отдельный флаг: остановились ли мы по лимиту (а не по концу).
    let trunc_cell: Rc<Cell<bool>> = Rc::new(Cell::new(false));

    let open_fn = method(&os, "openCursor")?;
    let request = match range {
        Some(spec) => {
            let range_js = spec.to_js()?;
            open_fn
                .call2(&os, &range_js, &JsValue::from_str(direction.as_str()))
                .map_err(|e| JsValue::from_str(&format!("openCursor: {e:?}")))?
        }
        None => {
            // direction передаём всегда — без второго аргумента
            // openCursor() жёстко использует "next".
            open_fn
                .call2(
                    &os,
                    &JsValue::NULL,
                    &JsValue::from_str(direction.as_str()),
                )
                .map_err(|e| JsValue::from_str(&format!("openCursor: {e:?}")))?
        }
    };

    // onsuccess: продвигает курсор, проверяет лимит ДО continue().
    {
        let state_cb = state.clone();
        let limit_cb = limit_cell.clone();
        let trunc_cb = trunc_cell.clone();
        let on_success =
            Closure::<dyn FnMut(web_sys::Event)>::new(move |ev: web_sys::Event| {
                if state_cb.borrow().done {
                    return;
                }
                let target = match ev.target() {
                    Some(t) => t,
                    None => return,
                };
                let cursor = js_sys::Reflect::get(&target, &"result".into())
                    .unwrap_or(JsValue::NULL);
                if cursor.is_null() || cursor.is_undefined() {
                    state_cb.borrow_mut().done = true;
                    return;
                }
                let key = js_sys::Reflect::get(&cursor, &"key".into())
                    .unwrap_or(JsValue::NULL);
                let value = js_sys::Reflect::get(&cursor, &"value".into())
                    .unwrap_or(JsValue::NULL);

                let should_stop = {
                    let mut st = state_cb.borrow_mut();
                    st.keys.push(key);
                    st.values.push(value);
                    match limit_cb.get() {
                        Some(lim) => st.keys.len() as u32 >= lim,
                        None => false,
                    }
                };

                if should_stop {
                    state_cb.borrow_mut().done = true;
                    trunc_cb.set(true); // ← отдельный флаг для truncated
                    return; // НЕ вызываем continue() — итерация остановлена
                }

                if let Ok(cont) = js_sys::Reflect::get(&cursor, &"continue".into())
                    .and_then(|f| f.dyn_into::<js_sys::Function>())
                {
                    let _ = cont.call0(&cursor);
                }
            });
        let _ = js_sys::Reflect::set(
            &request,
            &"onsuccess".into(),
            on_success.as_ref().unchecked_ref(),
        );
        on_success.forget();
    }

    // onerror: явное приведение target к JsValue для единого типа match.
    {
        let state_cb = state.clone();
        let on_error =
            Closure::<dyn FnMut(web_sys::Event)>::new(move |ev: web_sys::Event| {
                let target: JsValue = match ev.target() {
                    Some(t) => t.into(),
                    None => JsValue::NULL,
                };
                let err = js_sys::Reflect::get(&target, &"error".into())
                    .unwrap_or(JsValue::UNDEFINED);
                state_cb.borrow_mut().error = Some(err);
                state_cb.borrow_mut().done = true;
            });
        let _ = js_sys::Reflect::set(
            &request,
            &"onerror".into(),
            on_error.as_ref().unchecked_ref(),
        );
        on_error.forget();
    }

    // Ждём завершения, изредка отдаём управление UI.
    let mut last_yield = now_ms();
    loop {
        if state.borrow().done {
            break;
        }
        TimeoutFuture::new(0).await;
        let now = now_ms();
        if now - last_yield >= 16.0 {
            on_progress(state.borrow().keys.len() as u32);
            last_yield = now;
        }
    }

    let scanned = state.borrow().keys.len() as u32;
    on_progress(scanned);

    if let Some(err) = state.borrow().error.clone() {
        return Err(err);
    }

    // Транзакция завершается вслед за курсором. Подписка уже сделана.
    JsFuture::from(tx_complete).await.ok();

    let elapsed_ms = now_ms() - t0;
    let st = state.borrow();
    Ok(CursorResult {
        scanned,
        keys: st.keys.clone(),
        values: st.values.clone(),
        elapsed_ms,
        truncated: trunc_cell.get(),
    })
}

// ============================================================
//                       HELPERS
// ============================================================

fn method(obj: &JsValue, name: &str) -> Result<js_sys::Function, JsValue> {
    js_sys::Reflect::get(obj, &JsValue::from_str(name))
        .map_err(|e| JsValue::from_str(&format!("Reflect {name}: {e:?}")))?
        .dyn_into()
        .map_err(|_| JsValue::from_str(&format!("{name} не функция")))
}

fn array_to_vec(v: JsValue) -> Result<Vec<JsValue>, JsValue> {
    let arr: js_sys::Array = v
        .dyn_into()
        .map_err(|_| JsValue::from_str("ожидали Array"))?;
    let mut out = Vec::with_capacity(arr.length() as usize);
    for i in 0..arr.length() {
        out.push(arr.get(i));
    }
    Ok(out)
}

fn key_range_ctor() -> Result<JsValue, JsValue> {
    let global = js_sys::global();
    let kr = js_sys::Reflect::get(&global, &JsValue::from_str("IDBKeyRange"))
        .map_err(|e| JsValue::from_str(&format!("Reflect IDBKeyRange: {e:?}")))?;
    if kr.is_undefined() || kr.is_null() {
        return Err(JsValue::from_str("IDBKeyRange недоступен"));
    }
    Ok(kr)
}

async fn await_request(req: &JsValue) -> Result<JsValue, JsValue> {
    let r_ok_set = req.clone();
    let r_ok_in = req.clone();
    let r_err_set = req.clone();
    let r_err_in = req.clone();

    let promise = js_sys::Promise::new(&mut move |resolve, reject| {
        let r = r_ok_in.clone();
        let resolve_c = resolve.clone();
        let on_success = Closure::once_into_js(move |_e: web_sys::Event| {
            let result = js_sys::Reflect::get(&r, &"result".into())
                .unwrap_or(JsValue::UNDEFINED);
            let _ = resolve_c.call1(&JsValue::UNDEFINED, &result);
        });
        let _ = js_sys::Reflect::set(&r_ok_set, &"onsuccess".into(), &on_success);

        let r = r_err_in.clone();
        let reject_c = reject.clone();
        let on_error = Closure::once_into_js(move |_e: web_sys::Event| {
            let err = js_sys::Reflect::get(&r, &"error".into())
                .unwrap_or(JsValue::UNDEFINED);
            let _ = reject_c.call1(&JsValue::UNDEFINED, &err);
        });
        let _ = js_sys::Reflect::set(&r_err_set, &"onerror".into(), &on_error);
    });

    JsFuture::from(promise).await
}

/// Promise, который резолвится, когда транзакция завершена — успешно
/// (`oncomplete`) или через откат (`onabort`).
///
/// Никаких reject'ов: об ошибке конкретного запроса знает сам request.
/// Наша задача — дождаться, чтобы соединение было освобождено и следующий
/// `deleteDatabase` не ушёл в `onblocked`.
fn tx_complete_promise(tx: &JsValue) -> js_sys::Promise {
    let tx_set = tx.clone();
    js_sys::Promise::new(&mut move |resolve, _reject| {
        // Успешный коммит.
        let resolve_complete = resolve.clone();
        let on_complete = Closure::once_into_js(move || {
            let _ = resolve_complete.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&tx_set, &"oncomplete".into(), &on_complete);

        // Откат — тоже завершение. Резолвим, чтобы await не висел вечно.
        let resolve_abort = resolve.clone();
        let on_abort = Closure::once_into_js(move || {
            let _ = resolve_abort.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&tx_set, &"onabort".into(), &on_abort);
    })
}

fn now_ms() -> f64 {
    web_sys::window()
        .and_then(|w| w.performance())
        .map(|p| p.now())
        .unwrap_or(0.0)
}
```

## 24.5.13. Демо

Демо — интерактивная песочница из шести панелей:

1. **CRUD** — формы добавления автора, книги, тега, связи.
2. **Дерево авторов** — раскрывающиеся `<details>` со списком книг.
3. **Дерево книг** — чипы тегов у каждой книги.
4. **Курсор** — сравнение yield vs no-yield с прогресс-баром.
5. **Диапазон** — `IDBKeyRange` на префикс ISBN.
6. **Миграция** — `cursor.update()` со счётчиком обновлённых.

Данные — реальная русская библиотека: **100 авторов**, **~370 книг**, 25 жанровых тегов. Seed подгружает первые N книг (30 / 100 / 200 / 500). При первом открытии — demo-сид на 8 авторов.

Пользователь может:

- Раскрыть «Пушкин» → увидеть 5 книг.
- Раскрыть «Евгений Онегин» → увидеть чипы: **поэма, классика, русская, 19 век**.
- Кликнуть × на чипе → удалить одну связь, но книгу и тег оставить.
- Кликнуть ✗ у автора → каскадно удалить все его книги и их рёбра.
- Переключить `next`/`prev` в курсоре — увидеть обратный обход.
- Изменить `bound [978-000000, 978-000100]` — увидеть только первые 100 книг.
- Ввести `shelf=A1` и нажать `⊕ set in all` — добавить поле всем книгам.
- Повторить миграцию — увидеть «обновлено 0, без изменений N».

## 24.5.14. Тесты


**wasm_idb_records.rs**:
```
//! Wasm-тесты модуля `idb_records`: CRUD, autoIncrement, cursor,
//! диапазоны ключей, направления.
//!
//! Компилируются только под wasm32 — на host файл пуст.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_test::*;

use hello_wasm02::idb_db::Database;
use hello_wasm02::idb_factory;
use hello_wasm02::idb_records::{self, Direction, KeyRangeSpec, RecordStore};
use hello_wasm02::idb_schema::{self, SchemaPlan, StoreSpec};

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64);
}

fn test_db() -> String {
    return format!("test-records-{}", uid());
}

fn err_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    return e.as_string().unwrap_or_else(|| format!("{e:?}"));
}

async fn setup(name: &str) -> Database {
    let plan = SchemaPlan::new()
        .add(StoreSpec::with_key("books", "isbn"))
        .add(StoreSpec::with_auto_key("logs"))
        .add(StoreSpec::with_auto_key("metrics"));
    let outcome = idb_schema::open_with_plan(name, 1, &plan)
        .await
        .expect("open_with_plan");
    return outcome.db;
}

fn make_book(isbn: &str, title: &str, year: u32) -> JsValue {
    let obj = js_sys::Object::new();
    let _ = js_sys::Reflect::set(&obj, &"isbn".into(), &JsValue::from_str(isbn));
    let _ = js_sys::Reflect::set(&obj, &"title".into(), &JsValue::from_str(title));
    let _ = js_sys::Reflect::set(&obj, &"year".into(), &JsValue::from_f64(year as f64));
    return obj.into();
}

fn make_metric(id: u32, value: f64) -> JsValue {
    let obj = js_sys::Object::new();
    let _ = js_sys::Reflect::set(&obj, &"metric_id".into(), &JsValue::from_f64(id as f64));
    let _ = js_sys::Reflect::set(&obj, &"value".into(), &JsValue::from_f64(value));
    return obj.into();
}

async fn cleanup(name: &str) {
    idb_factory::delete_by_name(name).await.expect("cleanup");
}

// ---------------- add / put ----------------

#[wasm_bindgen_test]
async fn add_inserts_and_returns_key() {
    let name = test_db();
    let db = setup(&name).await;

    let s = RecordStore::open(db.raw(), "books", "readwrite").expect("open");
    let key = idb_records::add(&s, &make_book("978-1", "Book", 2020))
        .await
        .expect("add");
    assert_eq!(key.as_string().as_deref(), Some("978-1"));

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn add_duplicate_throws_constraint() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "books", "readwrite").expect("open 1");
    idb_records::add(&s1, &make_book("978-1", "A", 2000)).await.unwrap();

    let s2 = RecordStore::open(db.raw(), "books", "readwrite").expect("open 2");
    let r = idb_records::add(&s2, &make_book("978-1", "B", 2001)).await;
    assert!(r.is_err(), "дубликат должен упасть");
    assert_eq!(err_name(&r.unwrap_err()), "ConstraintError");

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn put_overwrites_existing() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "books", "readwrite").expect("open 1");
    idb_records::put(&s1, &make_book("978-1", "Old", 2000)).await.unwrap();

    let s2 = RecordStore::open(db.raw(), "books", "readwrite").expect("open 2");
    idb_records::put(&s2, &make_book("978-1", "New", 2000)).await.unwrap();

    let s3 = RecordStore::open(db.raw(), "books", "readonly").expect("open 3");
    let rec = idb_records::get(&s3, &JsValue::from_str("978-1"))
        .await
        .expect("get")
        .expect("present");
    let title = js_sys::Reflect::get(&rec, &"title".into())
        .ok()
        .and_then(|v| v.as_string())
        .unwrap_or_default();
    assert_eq!(title, "New");

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn auto_increment_generates_numeric_key() {
    let name = test_db();
    let db = setup(&name).await;

    let s = RecordStore::open(db.raw(), "logs", "readwrite").expect("open");
    let rec = js_sys::Object::new();
    let _ = js_sys::Reflect::set(&rec, &"msg".into(), &JsValue::from_str("hello"));

    let key = idb_records::put(&s, &rec.into()).await.expect("put");
    assert!(key.as_f64().is_some(), "ожидали number key");
    assert!(key.as_f64().unwrap() >= 1.0);

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn count_and_get_all() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "books", "readwrite").expect("open");
    for i in 0..5 {
        idb_records::add(&s1, &make_book(&format!("978-{i}"), "T", 2000 + i))
            .await
            .unwrap();
    }

    let s2 = RecordStore::open(db.raw(), "books", "readonly").expect("open 2");
    let n = idb_records::count(&s2).await.expect("count");
    assert_eq!(n, 5);

    let s3 = RecordStore::open(db.raw(), "books", "readonly").expect("open 3");
    let all = idb_records::get_all(&s3).await.expect("get_all");
    assert_eq!(all.len(), 5);

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn delete_idempotent() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "books", "readwrite").expect("open");
    idb_records::add(&s1, &make_book("978-1", "T", 2020)).await.unwrap();

    let s2 = RecordStore::open(db.raw(), "books", "readwrite").expect("open 2");
    idb_records::delete(&s2, &JsValue::from_str("978-1")).await.unwrap();

    let s3 = RecordStore::open(db.raw(), "books", "readwrite").expect("open 3");
    idb_records::delete(&s3, &JsValue::from_str("978-1")).await.unwrap();

    let s4 = RecordStore::open(db.raw(), "books", "readonly").expect("open 4");
    let n = idb_records::count(&s4).await.expect("count");
    assert_eq!(n, 0);

    db.close();
    cleanup(&name).await;
}

// ---------------- cursor ----------------

#[wasm_bindgen_test]
async fn cursor_next_matches_get_all() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "metrics", "readwrite").expect("open");
    let records: Vec<JsValue> = (1..=100).map(|i| make_metric(i, i as f64 * 1.5)).collect();
    idb_records::seed_bulk(&s1, &records).await.expect("seed");

    let s2 = RecordStore::open(db.raw(), "metrics", "readonly").expect("open 2");
    let all = idb_records::get_all(&s2).await.expect("get_all");

    let s3 = RecordStore::open(db.raw(), "metrics", "readonly").expect("open 3");
    let cur = idb_records::cursor_visit(&s3, None, Direction::Next, None, |_| {})
        .await
        .expect("cursor");

    assert_eq!(all.len(), 100);
    assert_eq!(cur.scanned, 100);
    assert!(!cur.truncated);

    // Порядок совпадает.
    for i in 0..cur.keys.len() {
        let k = cur.keys[i].as_f64().unwrap_or(0.0) as u32;
        assert_eq!(k, (i as u32) + 1);
    }

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn cursor_prev_reverses_order() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "metrics", "readwrite").expect("open");
    let records: Vec<JsValue> = (1..=10).map(|i| make_metric(i, i as f64)).collect();
    idb_records::seed_bulk(&s1, &records).await.expect("seed");

    let s2 = RecordStore::open(db.raw(), "metrics", "readonly").expect("open 2");
    let cur = idb_records::cursor_visit(&s2, None, Direction::Prev, None, |_| {})
        .await
        .expect("cursor");

    assert_eq!(cur.scanned, 10);
    let first = cur.keys[0].as_f64().unwrap_or(0.0) as u32;
    let last = cur.keys[9].as_f64().unwrap_or(0.0) as u32;
    assert_eq!(first, 10, "Prev начинается с последнего");
    assert_eq!(last, 1, "Prev заканчивается первым");

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn cursor_lower_bound() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "metrics", "readwrite").expect("open");
    let records: Vec<JsValue> = (1..=100).map(|i| make_metric(i, i as f64)).collect();
    idb_records::seed_bulk(&s1, &records).await.expect("seed");

    let s2 = RecordStore::open(db.raw(), "metrics", "readonly").expect("open 2");
    let range = KeyRangeSpec::LowerBound {
        value: JsValue::from_f64(95.0),
        open: false,
    };
    let cur = idb_records::cursor_visit(&s2, Some(range), Direction::Next, None, |_| {})
        .await
        .expect("cursor");

    assert_eq!(cur.scanned, 6, "записи 95..100 — всего 6");

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn cursor_bound_range() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "metrics", "readwrite").expect("open");
    let records: Vec<JsValue> = (1..=100).map(|i| make_metric(i, i as f64)).collect();
    idb_records::seed_bulk(&s1, &records).await.expect("seed");

    let s2 = RecordStore::open(db.raw(), "metrics", "readonly").expect("open 2");
    let range = KeyRangeSpec::Bound {
        lower: JsValue::from_f64(10.0),
        upper: JsValue::from_f64(19.0),
        lower_open: false,
        upper_open: false,
    };
    let cur = idb_records::cursor_visit(&s2, Some(range), Direction::Next, None, |_| {})
        .await
        .expect("cursor");

    assert_eq!(cur.scanned, 10, "10..19 — всего 10");

    db.close();
    cleanup(&name).await;
}

#[wasm_bindgen_test]
async fn cursor_limit_truncates() {
    let name = test_db();
    let db = setup(&name).await;

    let s1 = RecordStore::open(db.raw(), "metrics", "readwrite").expect("open");
    let records: Vec<JsValue> = (1..=1000).map(|i| make_metric(i, i as f64)).collect();
    idb_records::seed_bulk(&s1, &records).await.expect("seed");

    let s2 = RecordStore::open(db.raw(), "metrics", "readonly").expect("open 2");
    let cur = idb_records::cursor_visit(&s2, None, Direction::Next, Some(50), |_| {})
        .await
        .expect("cursor");

    assert_eq!(cur.scanned, 50, "limit 50");
    assert!(cur.truncated, "итерация прервана по limit");

    db.close();
    cleanup(&name).await;
}
```

11 wasm-тестов в `tests/wasm_idb_records.rs`:

1. `add_inserts_and_returns_key` — `add` возвращает ключ.
2. `add_duplicate_throws_constraint` — дубликат даёт `ConstraintError`.
3. `put_overwrites_existing` — `put` перезаписывает.
4. `get_missing_returns_none` — `get` для отсутствующего ключа → `None`.
5. `auto_increment_generates_numeric_key` — `autoIncrement` возвращает число.
6. `delete_idempotent` — повторный `delete` — не ошибка.
7. `count_and_get_all` — счётчик и полный список.
8. `cursor_next_matches_get_all` — курсор `next` совпадает с `getAll`.
9. `cursor_prev_reverses_order` — `prev` идёт с последнего ключа.
10. `cursor_lower_bound` — `lowerBound ≥ 95` даёт 6 записей из 100.
11. `cursor_bound_range` — `bound(10, 19)` даёт 10 записей.
12. `cursor_limit_truncates` — `limit 50` прерывает итерацию, `truncated = true`.

Три обязательные строки в каждом wasm-тесте:

```rust
#![cfg(target_arch = "wasm32")]
use wasm_bindgen_test::*;
wasm_bindgen_test_configure!(run_in_browser);
```

**Команды:**

```bash
cargo check --target wasm32-unknown-unknown
wasm-pack test --headless --chrome  --test wasm_idb_records
wasm-pack test --headless --firefox --test wasm_idb_records
```

## 24.5.15. Демо-пример
**app.rs**:

```
//! Демо Главы 24.5: `IdbObjectStore` — записи, курсоры, диапазоны, связи.
//!
//! ПРАВИЛО: `format!` в tail-позиции — `format_str(format_args!(...))` или `return`.
//! ПРАВИЛО: `JsCast` для `.dyn_into()` / `.dyn_ref()`.
//! ПРАВИЛО: `<details>` НЕ должен быть flex-item. `.rc-tree` — display:block,
//! а сам `<details>` — display:block с margin.
//! ПРАВИЛО: внутри `<summary>` — только phrasing content (span/button).

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{Document, Element, HtmlButtonElement, HtmlInputElement, HtmlSelectElement};

use crate::format_str::format_str;
use crate::idb_db::Database;
use crate::idb_factory;
use crate::idb_records::{self, Direction, KeyRangeSpec, RecordStore};
use crate::idb_schema::{self, IndexSpec, SchemaPlan, StoreSpec};

const DB_NAME: &str = "hello-wasm-idb-24-5-related";

const ST_AUTHORS:   &str = "authors";
const ST_BOOKS:     &str = "books";
const ST_TAGS:      &str = "tags";
const ST_BOOK_TAGS: &str = "book_tags";

const MAX_TABLE_ROWS: usize = 200;
const MAX_PREVIEW_ROWS: usize = 30;

// ============================================================
//                  SEED DATA (DSL)
// ============================================================

/// Компактная DSL-таблица: автор + его книги.
///
/// Формат:
///   author_slug|Имя Фамилия
///   \tНазвание|год|жанр
///
/// Строка без `\t` — новый автор. С `\t` — книга текущего автора.
/// `#...` — комментарий.
const SEED_DATA: &str = r#"
# 1-25. XIX век
pushkin|Александр Пушкин
	Евгений Онегин|1833|poem_long
	Капитанская дочка|1836|novel
	Дубровский|1841|novel
	Пиковая дама|1834|story
	Борис Годунов|1825|tragedy
lermontov|Михаил Лермонтов
	Герой нашего времени|1840|novel
	Мцыри|1839|poem_long
	Демон|1839|poem_long
	Бородино|1837|verse
	Маскарад|1842|drama
gogol|Николай Гоголь
	Мёртвые души|1842|novel
	Шинель|1842|story
	Ревизор|1836|comedy
	Тарас Бульба|1835|novella
	Нос|1836|story
turgenev|Иван Тургенев
	Отцы и дети|1862|novel
	Дворянское гнездо|1859|novel
	Рудин|1856|novel
	Записки охотника|1852|story
	Накануне|1860|novel
dostoevsky|Фёдор Достоевский
	Преступление и наказание|1866|novel
	Идиот|1869|novel
	Братья Карамазовы|1880|novel
	Бесы|1872|novel
	Записки из подполья|1864|novella
tolstoy_ln|Лев Толстой
	Война и мир|1869|novel
	Анна Каренина|1878|novel
	Воскресение|1899|novel
	Смерть Ивана Ильича|1886|novella
	Хаджи-Мурат|1912|novella
goncharov|Иван Гончаров
	Обломов|1859|novel
	Обыкновенная история|1847|novel
	Обрыв|1869|novel
	Фрегат «Паллада»|1858|memoir
gercen|Александр Герцен
	Кто виноват|1846|novel
	Доктор Крупов|1849|novella
	Былое и думы|1868|memoir
	Сорока-воровка|1848|novella
nekrasov|Николай Некрасов
	Кому на Руси жить хорошо|1877|poem_long
	Русские женщины|1872|poem_long
	Мороз Красный нос|1863|poem_long
	Мазай и зайцы|1870|verse
salt_ged|Михаил Салтыков-Щедрин
	История одного города|1870|satire
	Господа Головлёвы|1880|novel
	Пошехонская старина|1889|novel
	Сказки|1886|satire
leskov|Николай Лесков
	Левша|1881|story
	Очарованный странник|1873|novella
	Соборяне|1872|novel
	Леди Макбет Мценского уезда|1865|novella
ostrovsky|Александр Островский
	Гроза|1859|drama
	Бесприданница|1879|drama
	Свои люди сочтёмся|1850|comedy
	Лес|1871|comedy
	Доходное место|1856|comedy
chernyshevsky|Николай Чернышевский
	Что делать|1863|novel
	Пролог|1877|novel
	Эстетические отношения искусства к действительности|1855|critique
griboedov|Александр Грибоедов
	Горе от ума|1825|comedy
	Студент|1817|comedy
	Молодые супруги|1815|comedy
karamzin|Николай Карамзин
	Бедная Лиза|1792|novella
	История государства Российского|1829|nonfiction
	Наталья боярская дочь|1792|novella
	Марфа-посадница|1803|novella
radishchev|Александр Радищев
	Путешествие из Петербурга в Москву|1790|novel
	Житие Фёдора Ушакова|1789|bio
	Вольность|1790|verse
zhukovsky|Василий Жуковский
	Светлана|1813|verse
	Людмила|1808|verse
	Море|1822|verse
	Одиссея|1849|poem_long
kolcov|Алексей Кольцов
	Косарь|1836|verse
	Песня пахаря|1839|verse
	Раздумье селянина|1837|verse
tutchev|Фёдор Тютчев
	Silentium|1830|verse
	Весенняя гроза|1828|verse
	Умом Россию не понять|1866|verse
	Последняя любовь|1851|verse
fet|Афанасий Фет
	Шёпот робкое дыханье|1850|verse
	Это утро радость эта|1887|verse
	Я пришёл к тебе с приветом|1843|verse
dobrolyubov|Николай Добролюбов
	Что такое обломовщина|1859|critique
	Луч света в тёмном царстве|1860|critique
	Когда же придёт настоящий день|1860|critique
pisarev|Дмитрий Писарев
	Базаров|1862|critique
	Реалисты|1864|critique
	Исторические идеи Огюста Конта|1865|critique
apuhtin|Алексей Апухтин
	Ночи в саду|1885|verse
	Дневник Павлика Дольского|1892|novella
	Архив графини Д|1890|novella
chekhov|Антон Чехов
	Палата № 6|1892|story
	Дама с собачкой|1899|story
	Вишнёвый сад|1904|drama
	Три сестры|1901|drama
	Чайка|1896|drama

# 26-50. Конец XIX — первая половина XX в.
gorky|Максим Горький
	На дне|1902|drama
	Мать|1906|novel
	Детство|1913|memoir
	Фома Гордеев|1899|novel
kuprin|Александр Куприн
	Поединок|1905|novella
	Олеся|1898|novella
	Гранатовый браслет|1911|novella
	Яма|1915|novella
bunin|Иван Бунин
	Жизнь Арсеньева|1930|novel
	Тёмные аллеи|1943|story
	Деревня|1910|novella
	Окаянные дни|1926|memoir
andreev|Леонид Андреев
	Иуда Искариот|1907|novella
	Жизнь Василия Фивейского|1904|novella
	Красный смех|1905|novella
	Мысль|1902|novella
blok|Александр Блок
	Двенадцать|1918|poem_long
	Незнакомка|1906|verse
	Соловьиный сад|1915|poem_long
	Возмездие|1921|poem_long
bely|Андрей Белый
	Петербург|1913|novel
	Серебряный голубь|1909|novel
	Котик Летаев|1922|novel
	Москва|1926|novel
brusov|Валерий Брюсов
	Огненный ангел|1908|novel
	Urbi et Orbi|1903|verse
	Венок|1906|verse
	Алтарь победы|1918|novel
balmont|Константин Бальмонт
	Будем как солнце|1903|verse
	Горящие здания|1900|verse
	Только любовь|1903|verse
gumilev|Николай Гумилёв
	Костёр|1918|verse
	Колчан|1916|verse
	Огненный столп|1921|verse
	Чужое небо|1912|verse
ahmatova|Анна Ахматова
	Вечер|1912|verse
	Чётки|1914|verse
	Реквием|1963|poem_long
	Поэма без героя|1965|poem_long
cvetaeva|Марина Цветаева
	Версты|1921|verse
	Лебединый стан|1957|verse
	Поэма Горы|1939|poem_long
	Поэма Конца|1926|poem_long
mandelshtam|Осип Мандельштам
	Камень|1913|verse
	Tristia|1922|verse
	Шум времени|1925|memoir
	Египетская марка|1928|novella
mayakovsky|Владимир Маяковский
	Облако в штанах|1915|poem_long
	Флейта-позвоночник|1916|poem_long
	Клоп|1929|comedy
	Баня|1930|comedy
esenin|Сергей Есенин
	Москва кабацкая|1924|verse
	Анна Снегина|1925|poem_long
	Чёрный человек|1925|poem_long
	Персидские мотивы|1925|verse
nabokov|Владимир Набоков
	Машенька|1926|novel
	Защита Лужина|1930|novel
	Дар|1938|novel
	Лолита|1955|novel
	Приглашение на казнь|1938|novel
bulgakov|Михаил Булгаков
	Мастер и Маргарита|1967|novel
	Белая гвардия|1925|novel
	Собачье сердце|1925|novella
	Записки юного врача|1926|story
zamyatin|Евгений Замятин
	Мы|1920|novel
	Уездное|1913|novella
	Островитяне|1918|novella
	Дракон|1918|story
babel|Исаак Бабель
	Конармия|1926|story
	Одесские рассказы|1927|story
	Закат|1928|drama
ilf|Илья Ильф
	Двенадцать стульев|1928|novel
	Золотой телёнок|1931|novel
	Одноэтажная Америка|1937|memoir
petrov|Евгений Петров
	Двенадцать стульев|1928|novel
	Золотой телёнок|1931|novel
	Одноэтажная Америка|1937|memoir
zoshenko|Михаил Зощенко
	Перед восходом солнца|1943|memoir
	Голубая книга|1935|story
	Аристократка|1923|story
	Баня|1925|story
platonov|Андрей Платонов
	Чевенгур|1929|novel
	Котлован|1930|novella
	Ювенильное море|1934|novella
	Возвращение|1946|story
olesha|Юрий Олеша
	Зависть|1927|novel
	Три толстяка|1928|fairy
	Ни дня без строчки|1965|memoir
sholohov|Михаил Шолохов
	Тихий Дон|1940|novel
	Поднятая целина|1959|novel
	Судьба человека|1956|story
	Они сражались за Родину|1959|novel
tolstoy_an|Алексей Толстой
	Пётр Первый|1934|novel
	Аэлита|1923|fantasy
	Гиперболоид инженера Гарина|1927|fantasy
	Золотой ключик|1936|fairy

# 51-75. Советская литература
grin|Александр Грин
	Алые паруса|1923|novella
	Бегущая по волнам|1928|novel
	Золотая цепь|1925|novel
	Блистающий мир|1924|novel
paustovsky|Константин Паустовский
	Золотая роза|1955|essay
	Повесть о жизни|1963|memoir
	Телеграмма|1946|story
	Мещёрская сторона|1939|essay
prishvin|Михаил Пришвин
	Кладовая солнца|1945|fairy
	Жень-шень|1933|novella
	Корабельная чаща|1954|novella
kaverin|Вениамин Каверин
	Два капитана|1944|novel
	Открытая книга|1956|novel
	Исполнение желаний|1936|novel
tynyanov|Юрий Тынянов
	Кюхля|1925|novel
	Смерть Вазир-Мухтара|1928|novel
	Пушкин|1936|bio
erenburg|Илья Эренбург
	Хулио Хуренито|1922|novel
	День второй|1934|novel
	Буря|1947|novel
	Оттепель|1954|novella
nekrasov_v|Виктор Некрасов
	В окопах Сталинграда|1946|novella
	В родном городе|1954|novella
	Кира Георгиевна|1961|novella
grossman|Василий Гроссман
	Жизнь и судьба|1980|novel
	За правое дело|1952|novel
	Всё течёт|1970|novella
	Народ бессмертен|1945|novella
solzhenicyn|Александр Солженицын
	Один день Ивана Денисовича|1962|novella
	Раковый корпус|1968|novel
	В круге первом|1968|novel
	Архипелаг ГУЛАГ|1973|nonfiction
shalamov|Варлам Шаламов
	Колымские рассказы|1973|story
	Левый берег|1973|story
	Очерки преступного мира|1973|essay
astafev|Виктор Астафьев
	Царь-рыба|1976|novella
	Прокляты и убиты|1994|novel
	Последний поклон|1994|novella
	Пастух и пастушка|1967|novella
rasputin|Валентин Распутин
	Живи и помни|1974|novella
	Прощание с Матёрой|1976|novella
	Деньги для Марии|1967|novella
	Последний срок|1970|novella
shukshin|Василий Шукшин
	Калина красная|1973|novella
	Я пришёл дать вам волю|1971|novel
	Сельские жители|1963|story
	Чудик|1967|story
iskander|Фазиль Искандер
	Сандро из Чегема|1989|novel
	Созвездие Козлотура|1966|novella
	Кролики и удавы|1981|novella
	Стоянка человека|1991|novella
vasiliev|Борис Васильев
	А зори здесь тихие|1969|novella
	В списках не значился|1974|novel
	Завтра была война|1984|novella
	Не стреляйте в белых лебедей|1973|novel
bondarev|Юрий Бондарев
	Горячий снег|1969|novel
	Батальоны просят огня|1957|novella
	Берег|1975|novel
	Тишина|1962|novel
baklanov|Григорий Бакланов
	Пядь земли|1959|novella
	Навеки девятнадцатилетние|1979|novel
	Мёртвые сраму не имут|1961|novella
voznesenskaya_v|Владимир Войнович
	Жизнь и приключения солдата Ивана Чонкина|1975|novel
	Москва 2042|1986|fantasy
	Иванкиада|1976|satire
tendryakov|Владимир Тендряков
	Ночь после выпуска|1974|novella
	Хлеб для собаки|1970|novella
	Ухабы|1960|novella
	Люди или нелюди|1966|story
harms|Даниил Хармс
	Случаи|1939|absurd
	Старуха|1939|absurd
	Голубая тетрадь № 10|1937|absurd
shvarc|Евгений Шварц
	Обыкновенное чудо|1956|drama
	Дракон|1944|drama
	Тень|1940|drama
	Золушка|1947|drama
gaydar|Аркадий Гайдар
	Тимур и его команда|1940|novella
	Чук и Гек|1939|story
	Школа|1930|novella
	Военная тайна|1935|novella
chukovsky|Корней Чуковский
	Мойдодыр|1923|verse
	Айболит|1929|verse
	Тараканище|1923|verse
	Федорино горе|1926|verse
marshak|Самуил Маршак
	Двенадцать месяцев|1943|fairy
	Вот какой рассеянный|1930|verse
	Кошкин дом|1922|fairy
	Умные вещи|1945|fairy

# 76-100. Вторая половина XX — XXI век
krapivin|Владислав Крапивин
	Мальчик со шпагой|1975|novella
	Острова и капитаны|1983|novel
	Трое с площади Карронад|1972|novella
bulychev|Кир Булычёв
	Сто лет тому вперёд|1978|fantasy
	Девочка с Земли|1974|fantasy
	Посёлок|1984|fantasy
	Подземелье ведьм|1988|fantasy
strugackie|Братья Стругацкие
	Пикник на обочине|1972|fantasy
	Трудно быть богом|1964|fantasy
	Понедельник начинается в субботу|1965|fantasy
	Обитаемый остров|1969|fantasy
	Град обреченный|1989|fantasy
vysotsky|Владимир Высоцкий
	Песни о войне|1975|verse
	Робин Гуд|1975|verse
	Гамлет|1971|verse
	Моя цыганская|1975|verse
brodsky|Иосиф Бродский
	Часть речи|1972|verse
	Урания|1987|verse
	Конец прекрасной эпохи|1972|verse
	Набережная неисцелимых|1992|essay
erofeev|Виктор Ерофеев
	Русская красавица|1990|novel
	Энциклопедия русской души|1999|novel
	Хороший Сталин|2004|memoir
t Tolstaya|Татьяна Толстая
	Кысь|2000|novel
	Ночь|1983|story
	Река|1999|novella
	Любишь не любишь|1997|story
pelevin|Виктор Пелевин
	Омон Ра|1992|fantasy
	Чапаев и Пустота|1996|novel
	Generation П|1999|novel
	S.N.U.F.F.|2011|novel
	iPhuck 10|2017|novel
sorokin|Владимир Сорокин
	Роман|1994|novel
	Голубое сало|1999|novel
	День опричника|2006|novel
	Теллурия|2013|novel
	Норма|2002|novel
akunin|Борис Акунин
	Азазель|1998|detective
	Турецкий гамбит|1998|detective
	Левиафан|1998|detective
	Коронация|2000|detective
	Статский советник|1999|detective
lukyanenko|Сергей Лукьяненко
	Ночной дозор|1998|fantasy
	Дневной дозор|2000|fantasy
	Лабиринт отражений|1997|fantasy
	Черновик|2005|fantasy
gluhovsky|Дмитрий Глуховский
	Метро 2033|2005|fantasy
	Метро 2034|2009|fantasy
	Метро 2035|2015|fantasy
	Текст|2017|novel
	Будущее|2013|novel
ulickaya|Людмила Улицкая
	Сонечка|1992|novella
	Казус Кукоцкого|2001|novel
	Даниэль Штайн переводчик|2006|novel
	Лестница Якова|2015|novel
vodolazkin|Евгений Водолазкин
	Лавр|2012|novel
	Авиатор|2016|novel
	Брисбен|2018|novel
	Чагин|2022|novel
ivanov|Алексей Иванов
	Сердце Пармы|2003|novel
	Географ глобус пропил|2003|novel
	Пищеблок|2018|novel
	Тобол|2017|novel
yahina|Гузель Яхина
	Зулейха открывает глаза|2015|novel
	Дети мои|2018|novel
	Волга впадает в Каспийское море|2022|story
prilepin|Захар Прилепин
	Санькя|2006|novel
	Обитель|2014|novel
	Чёрная обезьяна|2011|novel
bykov|Дмитрий Быков
	Орфография|2003|novel
	ЖД|2006|novel
	Икс|2012|novel
veller|Михаил Веллер
	Приключения майора Звягина|1991|novel
	Легенды Невского проспекта|1993|story
	Самовар|1996|novel
ustinova|Татьяна Устинова
	Персональный ангел|1999|detective
	Мой личный враг|2002|detective
	Большое зло и мелкие пакости|2000|detective
dontsova|Дарья Донцова
	Крутые наследнички|1999|detective
	Любимые родственники|2000|detective
	Доллары царя Гороха|2000|detective
prohanov|Александр Проханов
	Господин Гексоген|2001|novel
	Идущие в ночи|2005|novel
	Дерево в центре Кабула|2010|novel
kabakov|Александр Кабаков
	Невозвращенец|1989|novel
	Все поправимо|2005|novel
	Последний герой|1990|novel
makanin|Владимир Маканин
	Старые книги|1971|story
	Андеграунд или Герой нашего времени|1998|novel
	Лаз|1991|novella
	Кавказский пленный|1995|novella
"#;

/// Жанры → метки тегов. Id — латиницей, label — на русском.
const SEED_GENRES: &[(&str, &str)] = &[
    ("novel",       "роман"),
    ("novella",     "повесть"),
    ("story",       "рассказ"),
    ("poem_long",   "поэма"),
    ("verse",       "стихи"),
    ("drama",       "драма"),
    ("comedy",      "комедия"),
    ("tragedy",     "трагедия"),
    ("fantasy",     "фантастика"),
    ("detective",   "детектив"),
    ("memoir",      "мемуары"),
    ("bio",         "биография"),
    ("fairy",       "сказка"),
    ("essay",       "эссе"),
    ("critique",    "критика"),
    ("nonfiction",  "нон-фикшн"),
    ("satire",      "сатира"),
    ("absurd",      "абсурд"),
    ("classic",     "классика"),
    ("russian",     "русская"),
    ("soviet",      "советская"),
    ("modern",      "современная"),
    ("19c",         "19 век"),
    ("20c",         "20 век"),
    ("21c",         "21 век"),
];

// ============================================================
//                       HTML
// ============================================================

const HTML: &str = r#"
<style>
  :root {
    --rc-info: #3498db; --rc-ok: #27ae60; --rc-err: #c0392b; --rc-warn: #e67e22;
    --rc-text: #222; --rc-muted: #666;
    --rc-bg-panel: #fff; --rc-bg-soft: #fafafa; --rc-border: #e0e0e0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --rc-text: #e6e6e6; --rc-muted: #a0a0a0;
      --rc-bg-panel: #1e1e1e; --rc-bg-soft: #262626; --rc-border: #3a3a3a;
    }
    body { background: #121212; }
    .rc-anno { background: #1a2233; color: #9fc4ff; }
    .rc-anno code { background: rgba(52,152,219,0.2); }
    .rc-how { background: #232014; color: #e6c98c; }
    input, select { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    input::placeholder { color: #666; }
    .rc-kv-row { border-color: #3a3a3a; }
    progress { background: #2a2a2a; }
    .rc-hist { background: #262626; }
    summary { background: #262626 !important; }
    .rc-chip { background: #2a3a4a; color: #cfe0ff; border-color: #3a506a; }
    .rc-rel-list { background: #1e2a1e; color: #b8e0b8; }
    .rc-leaf { background: #1e1e1e; }
    .rc-count { background: #2a3a4a !important; color: #cfe0ff !important; }
  }

  html, body {
    width: 100% !important; max-width: none !important;
    margin: 0 !important; padding: 0 !important; overflow-x: hidden;
  }
  body {
    display: block !important;
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--rc-text); line-height: 1.45; font-size: 13px;
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
  .rc-subtitle { color: var(--rc-muted); font-size: 12px; margin: 0 0 8px; }
  .rc-subtitle code {
    background: rgba(128,128,128,0.15); padding: 1px 5px; border-radius: 3px;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rc-subtitle a { color: var(--rc-info); }

  .rc-how {
    background: #fff9e6; border: 1px solid #f0e0b0; border-radius: 6px;
    padding: 8px 12px; margin: 0 0 10px;
    font-size: 12px; line-height: 1.5; color: #5a4400;
  }
  .rc-how summary {
    cursor: pointer; font-weight: 600; list-style: none;
    display: flex; align-items: center; gap: 6px;
  }
  .rc-how summary::-webkit-details-marker { display: none; }
  .rc-how summary::marker { content: ""; }
  .rc-how summary::before { content: "▸"; transition: transform 0.15s; }
  .rc-how[open] summary::before { transform: rotate(90deg); }
  .rc-how ol { margin: 6px 0 0 20px; padding: 0; }
  .rc-how li { padding: 1px 0; }
  .rc-how kbd {
    background: #f0f0f0; border: 1px solid #ccc; border-radius: 3px;
    padding: 0 4px; font-family: ui-monospace, monospace; font-size: 10px;
    color: #333;
  }

  .rc-banner {
    display: flex; align-items: center; gap: 14px; flex-wrap: wrap;
    padding: 8px 14px; margin: 0 0 12px;
    background: linear-gradient(135deg, #eaf4ff 0%, #f5f0ff 100%);
    border: 2px solid var(--rc-info); border-radius: 8px;
    position: sticky; top: 0; z-index: 10;
  }
  @media (prefers-color-scheme: dark) {
    .rc-banner { background: linear-gradient(135deg, #1a2a3f 0%, #2a1e3f 100%); }
  }
  .rc-banner-mid { display: flex; flex-direction: column; gap: 1px; min-width: 0; flex: 1; }
  .rc-banner-label {
    font-size: 10px; text-transform: uppercase; letter-spacing: 0.7px;
    color: var(--rc-muted);
  }
  .rc-banner-name {
    font-family: ui-monospace, monospace; font-size: 13px; font-weight: 700;
    color: #1f4e79; word-break: break-all;
  }
  @media (prefers-color-scheme: dark) { .rc-banner-name { color: #9fc4ff; } }
  .rc-banner-meta {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--rc-muted); margin-top: 2px;
  }
  .rc-banner-actions { margin-left: auto; display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
  .rc-badge {
    display: inline-block; padding: 2px 10px; border-radius: 10px;
    background: #eaf4ff; color: #1f4e79;
    font-family: ui-monospace, monospace; font-size: 12px; font-weight: 700;
  }
  @media (prefers-color-scheme: dark) { .rc-badge { background: #2a3a4a; color: #cfe0ff; } }
  .rc-history {
    display: flex; gap: 6px; flex-wrap: wrap; align-items: center;
    font-family: ui-monospace, monospace; font-size: 10px;
    color: var(--rc-muted); margin-top: 2px;
  }
  .rc-hist {
    padding: 1px 6px; border-radius: 8px; background: #f0f0f0;
    max-width: 220px; overflow: hidden; text-overflow: ellipsis;
    white-space: nowrap;
  }

  .rc-grid {
    display: grid; gap: 12px; width: 100%; align-items: start;
    grid-template-columns: minmax(320px, 380px) minmax(0, 1fr) minmax(0, 1fr);
  }
  @media (max-width: 1280px) {
    .rc-grid { grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); }
    .rc-col-left { grid-column: 1 / -1; }
  }
  @media (max-width: 820px) {
    .rc-grid { grid-template-columns: 1fr; }
    .rc-col-left { grid-column: auto; }
  }
  .rc-col { display: flex; flex-direction: column; gap: 12px; min-width: 0; }

  .rc-panel {
    border: 1px solid var(--rc-border); border-radius: 8px;
    padding: 10px 12px; background: var(--rc-bg-panel);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    min-width: 0; overflow: hidden;
  }
  .rc-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 6px;
    display: flex; align-items: center; gap: 6px; flex-wrap: wrap;
  }
  .rc-panel-title .rc-mdn {
    margin-left: auto; font-size: 10px; font-weight: 400;
    color: var(--rc-muted); text-decoration: none;
  }
  .rc-panel-title .rc-mdn:hover { color: var(--rc-info); }
  .rc-panel-title .rc-hint-badge {
    padding: 0 8px; border-radius: 10px;
    background: #eaf4ff; color: #1f4e79;
    font-family: ui-monospace, monospace; font-size: 10px; font-weight: 700;
  }
  @media (prefers-color-scheme: dark) {
    .rc-panel-title .rc-hint-badge { background: #2a3a4a; color: #cfe0ff; }
  }
  .rc-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--rc-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .rc-num.ok { background: var(--rc-ok); }
  .rc-num.warn { background: var(--rc-warn); }

  .rc-anno {
    background: #eef4ff; border-left: 3px solid var(--rc-info);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 6px 0;
    color: #1f4e79;
    overflow-wrap: anywhere; word-break: break-word;
  }
  .rc-anno code {
    background: rgba(52,152,219,0.12); padding: 0 4px;
    border-radius: 3px; font-family: ui-monospace, monospace; font-size: 10px;
  }
  .rc-anno b { color: #0d3b66; }
  @media (prefers-color-scheme: dark) { .rc-anno b { color: #c0d8ff; } }

  .rc-form-row {
    display: flex; gap: 5px; flex-wrap: wrap;
    margin: 6px 0; align-items: center; min-width: 0;
  }
  input[type="text"], select {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--rc-bg-panel); color: var(--rc-text);
    font-family: ui-monospace, monospace;
    min-width: 0; max-width: 100%;
    transition: border-color 0.15s, box-shadow 0.15s;
  }
  input[type="text"]:focus, select:focus {
    outline: none; border-color: var(--rc-info);
    box-shadow: 0 0 0 2px rgba(52,152,219,0.2);
  }
  input[type="text"].rc-invalid {
    border-color: var(--rc-err);
    box-shadow: 0 0 0 2px rgba(192,57,43,0.15);
  }
  input[type="text"] { flex: 1 1 110px; width: 130px; }
  input[type="text"].rc-key { flex: 0 1 130px; width: 130px; }
  select { flex: 1 1 130px; min-width: 0; }
  label.rc-check {
    display: inline-flex; align-items: center; gap: 4px;
    font-size: 11px; color: var(--rc-muted); cursor: pointer;
    user-select: none; white-space: nowrap; flex-shrink: 0;
  }
  input[type="checkbox"] { margin: 0; flex-shrink: 0; }

  button {
    padding: 5px 10px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px; background: var(--rc-bg-soft);
    cursor: pointer; transition: background 0.15s, opacity 0.15s;
    color: var(--rc-text);
    font-family: inherit; white-space: nowrap; flex-shrink: 0;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button:focus-visible { outline: 2px solid var(--rc-info); outline-offset: 2px; }
  button.rc-primary { background: #eaf4ff; border-color: var(--rc-info); color: #1f4e79; }
  button.rc-danger  { background: #fdedec; border-color: var(--rc-err); color: #922b21; }
  button.rc-success { background: #e8f8ed; border-color: var(--rc-ok); color: #1e8449; }
  button.rc-mini    { padding: 2px 6px; font-size: 11px; }
  button.rc-x {
    padding: 1px 6px; font-size: 11px; flex-shrink: 0;
    background: transparent; border-color: transparent; color: var(--rc-err);
  }
  button.rc-x:hover:not(:disabled) { background: #fdedec; }
  button.rc-fill {
    background: #fff9e6; border-color: #f0e0b0; color: #7d5a00;
    padding: 2px 6px; font-size: 11px;
  }

  .rc-hint { font-size: 10px; color: var(--rc-muted); margin-top: 4px; line-height: 1.4; }
  .rc-hint-inline { font-size: 10px; color: var(--rc-muted); font-style: italic; }
  .rc-mono { font-family: ui-monospace, monospace; }

  .rc-progress-wrap { display: flex; align-items: center; gap: 8px; margin: 6px 0; }
  progress {
    flex: 1 1 auto; height: 14px;
    appearance: none; border: none; border-radius: 4px; overflow: hidden;
    background: #eee;
  }
  progress::-webkit-progress-bar { background: #eee; border-radius: 4px; }
  progress::-webkit-progress-value {
    background: var(--rc-info); border-radius: 4px;
    transition: background 0.3s;
  }
  progress::-moz-progress-bar { background: var(--rc-info); border-radius: 4px; }
  progress.rc-progress-busy::-webkit-progress-value { background: var(--rc-warn); }
  progress.rc-progress-busy::-moz-progress-bar { background: var(--rc-warn); }
  progress.rc-progress-ok::-webkit-progress-value { background: var(--rc-ok); }
  progress.rc-progress-ok::-moz-progress-bar { background: var(--rc-ok); }
  .rc-progress-label {
    font-family: ui-monospace, monospace; font-size: 11px;
    color: var(--rc-muted); flex-shrink: 0; min-width: 110px; text-align: right;
  }

  .rc-kv {
    display: grid; grid-template-columns: auto 1fr;
    gap: 3px 10px; margin-top: 6px;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rc-kv .k { color: var(--rc-muted); }
  .rc-kv .v { color: var(--rc-text); font-weight: 700; word-break: break-word; }

  .rc-preview {
    margin-top: 6px;
    border: 1px solid var(--rc-border); border-radius: 5px;
    background: var(--rc-bg-soft);
    max-height: 180px; overflow-y: auto;
    font-family: ui-monospace, monospace; font-size: 11px;
  }
  .rc-kv-row {
    padding: 3px 8px;
    border-bottom: 1px solid var(--rc-border);
    display: flex; gap: 8px; align-items: center; min-width: 0;
  }
  .rc-kv-row:last-child { border-bottom: none; }
  .rc-kv-row .k { color: var(--rc-info); font-weight: 700; flex-shrink: 0; }
  .rc-kv-row .v { color: var(--rc-text); overflow-wrap: anywhere; }
  .rc-preview-empty {
    padding: 12px; text-align: center;
    color: #999; font-style: italic; font-size: 11px;
  }

  .rc-tree {
    display: block;
    max-height: 480px; overflow-y: auto;
    margin-top: 6px;
    min-height: 40px;
  }
  .rc-tree details.rc-node {
    display: block;
    border: 1px solid var(--rc-border); border-radius: 6px;
    background: var(--rc-bg-soft);
    margin-bottom: 6px;
    overflow: hidden;
  }
  .rc-tree details.rc-node:last-child { margin-bottom: 0; }

  .rc-tree details.rc-node > summary {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 6px 10px;
    cursor: pointer;
    user-select: none;
    list-style: none;
    font-family: ui-monospace, monospace; font-size: 12px;
    background: var(--rc-bg-panel);
    transition: background 0.15s;
  }
  .rc-tree details.rc-node > summary::-webkit-details-marker { display: none; }
  .rc-tree details.rc-node > summary::marker { content: ""; }
  .rc-tree details.rc-node > summary::before {
    content: "▸";
    color: var(--rc-muted);
    font-size: 10px;
    transition: transform 0.15s;
    flex-shrink: 0;
  }
  .rc-tree details.rc-node[open] > summary::before {
    transform: rotate(90deg);
  }
  .rc-tree details.rc-node > summary:hover {
    background: rgba(52,152,219,0.06);
  }

  .rc-tree .rc-name { font-weight: 700; color: #1f4e79; }
  @media (prefers-color-scheme: dark) {
    .rc-tree .rc-name { color: #9fc4ff; }
  }
  .rc-tree .rc-sub { color: var(--rc-muted); font-size: 11px; }
  .rc-tree .rc-count {
    padding: 0 8px; border-radius: 10px;
    background: #eaf4ff; color: #1f4e79; font-size: 10px; font-weight: 700;
    margin-left: auto;
  }
  .rc-tree .rc-count.empty { background: #f0f0f0; color: #999; }
  @media (prefers-color-scheme: dark) {
    .rc-tree .rc-count { background: #2a3a4a; color: #cfe0ff; }
    .rc-tree .rc-count.empty { background: #2a2a2a; color: #888; }
  }

  .rc-node-body {
    padding: 6px 10px 8px 28px;
    font-family: ui-monospace, monospace; font-size: 11px;
    border-top: 1px solid var(--rc-border);
  }

  .rc-leaf {
    display: flex; align-items: center; gap: 8px;
    padding: 3px 8px; border-radius: 4px;
    background: var(--rc-bg-panel);
    border: 1px solid var(--rc-border);
    margin-bottom: 4px;
  }
  .rc-leaf:last-child { margin-bottom: 0; }
  .rc-leaf .rc-l-name { font-weight: 600; }
  .rc-leaf .rc-l-sub { color: var(--rc-muted); flex: 1; }
  .rc-leaf button.rc-x { margin-left: auto; }

  .rc-empty-leaf {
    padding: 6px 10px; color: #999; font-style: italic; font-size: 11px;
  }

  .rc-chips { display: flex; flex-wrap: wrap; gap: 4px; }
  .rc-chip {
    display: inline-flex; align-items: center; gap: 2px;
    padding: 1px 2px 1px 8px; border-radius: 10px;
    background: #eef4ff; color: #1f4e79; border: 1px solid #cfe0ff;
    font-family: ui-monospace, monospace; font-size: 10px;
  }
  @media (prefers-color-scheme: dark) {
    .rc-chip { background: #2a3a4a; color: #cfe0ff; border-color: #3a506a; }
  }
  .rc-chip button.rc-x { padding: 0 4px; font-size: 11px; color: var(--rc-err); }

  .rc-rel-list {
    margin-top: 6px;
    background: #eefaf0; border-left: 3px solid var(--rc-ok);
    padding: 6px 10px; border-radius: 4px;
    font-family: ui-monospace, monospace; font-size: 10.5px;
    color: #1e6b32;
    display: flex; flex-direction: column; gap: 2px;
  }
  @media (prefers-color-scheme: dark) { .rc-rel-list { color: #b8e0b8; } }
  .rc-rel-list b { color: #155724; }
  @media (prefers-color-scheme: dark) { .rc-rel-list b { color: #cfe0cf; } }

  .rc-status {
    display: flex; align-items: center; gap: 6px;
    padding: 4px 8px; border-radius: 4px; margin: 6px 0 0;
    font-family: ui-monospace, monospace; font-size: 11px;
    background: var(--rc-bg-soft); border-left: 3px solid #ccc;
    color: var(--rc-muted);
    min-height: 22px;
    overflow-wrap: anywhere; word-break: break-word;
  }
  .rc-status.ok      { background: #e8f8ed; border-color: var(--rc-ok);  color: #1e8449; }
  .rc-status.err     { background: #fdedec; border-color: var(--rc-err); color: #922b21; }
  .rc-status.busy    { background: #fff9e6; border-color: var(--rc-warn); color: #7d5a00; }
  .rc-status.stale   { background: #f0f0f0; border-color: #bbb; color: #666; font-style: italic; }
  @media (prefers-color-scheme: dark) {
    .rc-status.stale { background: #2a2a2a; color: #888; }
  }
  .rc-status-icon { font-size: 11px; flex-shrink: 0; }
</style>

<h1>Глава 24.5: IdbObjectStore — записи, курсоры, диапазоны, связи</h1>
<p class="rc-subtitle">
  Четыре store: <code>authors</code> (1:N) → <code>books</code> (M:N) ↔ <code>tags</code>
  через <code>book_tags</code>. Seed загружает реальную русскую библиотеку — 100 авторов, ~370 книг.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore" target="_blank" rel="noopener">MDN →</a>
</p>

<details class="rc-how">
  <summary>Как пользоваться</summary>
  <ol>
    <li><b>Автосид.</b> При первом открытии — 8 авторов, 30 книг, 10 тегов. Нажмите <b>Seed</b> для загрузки реальной библиотеки (100 авторов).</li>
    <li><b>1:N.</b> Панель 4: каждый автор раскрывается списком его книг. Удалите автора → каскадно удалятся его книги и связи.</li>
    <li><b>M:N.</b> Панель 5: каждая книга раскрывается чипами жанров. Удаляйте чипом ×.</li>
    <li><b>CRUD.</b> Панель 1 — добавьте автора/книгу/тег/связь. Кнопки <kbd>пример</kbd> заполняют форму.</li>
    <li><b>Курсор.</b> Панель 6: «yield» обновляет UI, «без yield» — только финальный результат. Скорость в K/сек.</li>
    <li><b>Диапазон.</b> Панель 7: <code>bound(978-000000, 978-000100)</code> — только эти ISBN.</li>
    <li><b>Миграция.</b> Панель 8: добавить поле всем книгам через <code>cursor.update()</code>.</li>
  </ol>
  <p style="margin:6px 0 0"><b>Клавиатура:</b> <kbd>Enter</kbd> в поле ISBN — добавить книгу. <kbd>Esc</kbd> — сбросить форму.</p>
</details>

<div class="rc-banner">
  <div class="rc-banner-mid">
    <div class="rc-banner-label">рабочая база</div>
    <div class="rc-banner-name" id="rc-db-name">…</div>
    <div class="rc-banner-meta" id="rc-db-meta">загрузка…</div>
    <div class="rc-history" id="rc-history"></div>
  </div>
  <div class="rc-banner-actions">
    <span class="rc-badge" id="rc-total">0 записей</span>
    <button id="rc-refresh-all" title="Перечитать все store и перерисовать представления">↻ Обновить всё</button>
    <button id="rc-reset" class="rc-danger" title="Удалить всю базу">↻ Сбросить базу</button>
  </div>
</div>

<div class="rc-grid">

  <div class="rc-col rc-col-left">

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">1.1</span> Автор
        <a class="rc-mdn" href="https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/add" target="_blank" rel="noopener">? MDN add</a>
      </h2>
      <div class="rc-anno">
        Родительская сущность. <code>authors.id</code> — слаг, на который ссылаются книги.
      </div>
      <div class="rc-form-row">
        <input type="text" id="rc-a-id" class="rc-key" placeholder="id (slug)" title="Уникальный идентификатор автора, латиницей">
        <input type="text" id="rc-a-name" placeholder="имя" title="Отображаемое имя">
      </div>
      <div class="rc-form-row">
        <button id="rc-a-add" class="rc-primary" title="put — upsert по id">+ author (put)</button>
        <button id="rc-a-fill" class="rc-fill" title="Заполнить примером">📋 пример</button>
      </div>
      <div id="rc-status-a" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">1.2</span> Книга (1:N)
      </h2>
      <div class="rc-anno">
        <code>books.author_id</code> — ссылка на автора. Проверяем существование автора до <code>put</code>.
      </div>
      <div class="rc-form-row">
        <input type="text" id="rc-b-isbn" class="rc-key" placeholder="isbn" title="Ключ книги — ISBN">
        <input type="text" id="rc-b-title" placeholder="название">
      </div>
      <div class="rc-form-row">
        <select id="rc-b-author" title="Автор книги"></select>
        <input type="text" id="rc-b-year" class="rc-key" placeholder="год" title="Год издания — число">
      </div>
      <div class="rc-form-row">
        <button id="rc-b-add" class="rc-primary" title="put — upsert по ISBN">+ book (put)</button>
        <button id="rc-b-fill" class="rc-fill" title="Заполнить примером">📋 пример</button>
      </div>
      <div id="rc-status-b" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">1.3</span> Тег
      </h2>
      <div class="rc-anno">
        Независимая сущность. Связывается с книгами через <code>book_tags</code>.
      </div>
      <div class="rc-form-row">
        <input type="text" id="rc-t-id" class="rc-key" placeholder="id" title="Идентификатор тега">
        <input type="text" id="rc-t-label" placeholder="метка">
      </div>
      <div class="rc-form-row">
        <button id="rc-t-add" class="rc-primary" title="put — upsert по id">+ tag (put)</button>
        <button id="rc-t-fill" class="rc-fill" title="Заполнить примером">📋 пример</button>
      </div>
      <div id="rc-status-t" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">1.4</span> Связь книга ↔ тег (M:N)
      </h2>
      <div class="rc-anno">
        <code>book_tags</code> с составным ключом <code>book_isbn,tag_id</code>.
        Один <code>put</code> создаёт ребро M:N. Повторный — no-op.
      </div>
      <div class="rc-form-row">
        <select id="rc-lt-book" title="Книга"></select>
        <select id="rc-lt-tag" title="Тег"></select>
      </div>
      <div class="rc-form-row">
        <button id="rc-lt-add" class="rc-primary" title="Создать связь">🔗 link</button>
      </div>
      <div id="rc-status-lt" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

  </div>

  <div class="rc-col">

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">4</span> 📚 Авторы → их книги (1:N)
        <span class="rc-hint-badge" id="rc-badge-a">—</span>
        <a class="rc-mdn" href="https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/index" target="_blank" rel="noopener">? MDN index</a>
      </h2>

      <div class="rc-anno">
        <b>Один-ко-многим.</b> Книги группируются по <code>author_id</code> через индекс
        <code>by_author</code>. Удаление автора каскадно удаляет его книги и связи этих
        книг с тегами. Раскройте автора — увидите его книги с кнопкой удаления каждой.
      </div>

      <div class="rc-tree" id="rc-tree-authors">
        <div class="rc-empty-leaf">Авторов пока нет</div>
      </div>

      <div id="rc-status-tree-a" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">5</span> 🏷 Книги → их теги (M:N)
        <span class="rc-hint-badge" id="rc-badge-b">—</span>
        <a class="rc-mdn" href="https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/delete" target="_blank" rel="noopener">? MDN delete</a>
      </h2>

      <div class="rc-anno">
        <b>Многие-ко-многим.</b> Каждая книга раскрывается чипами тегов, каждый чип
        можно удалить кликом на <b>×</b>. Удаление книги каскадно удаляет все её
        связи в <code>book_tags</code>.
      </div>

      <div class="rc-tree" id="rc-tree-books">
        <div class="rc-empty-leaf">Книг пока нет</div>
      </div>

      <div id="rc-status-tree-b" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

  </div>

  <div class="rc-col">

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">6</span> ⏩ Итерация курсором
        <a class="rc-mdn" href="https://developer.mozilla.org/en-US/docs/Web/API/IDBObjectStore/openCursor" target="_blank" rel="noopener">? MDN openCursor</a>
      </h2>
      <div class="rc-anno">
        <b>yield vs без yield.</b> Между записями курсор должен отдавать управление
        браузеру — иначе UI замирает. Yield — <code>TimeoutFuture::new(0)</code>, macrotask.
        Итерируем по <code>books</code>.
      </div>

      <div class="rc-form-row">
        <select id="rc-direction" title="next / prev">
          <option value="next">next →</option>
          <option value="prev">← prev</option>
        </select>
        <label class="rc-check" title="Ограничить число читаемых записей">
          <input type="checkbox" id="rc-limit-on"> limit
        </label>
        <input type="text" id="rc-limit-val" value="1000" class="rc-key" title="Положительное целое">
        <button id="rc-limit-reset" class="rc-mini" title="Сброс к 1000">↺</button>
      </div>

      <div class="rc-form-row">
        <button id="rc-iter-yield" class="rc-success" title="openCursor + yield">▶ yield</button>
        <button id="rc-iter-noyield" title="getAll — одним запросом">▶ без yield</button>
      </div>

      <div class="rc-progress-wrap">
        <progress id="rc-progress" value="0" max="100"></progress>
        <span class="rc-progress-label" id="rc-progress-label">—</span>
      </div>

      <div class="rc-kv">
        <div class="k">прочитано</div> <div class="v" id="rc-scanned">—</div>
        <div class="k">время</div>     <div class="v" id="rc-elapsed">—</div>
        <div class="k">скорость</div>  <div class="v" id="rc-rate">—</div>
      </div>

      <div id="rc-status-iter" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">7</span> 🎯 Диапазон ключей
        <a class="rc-mdn" href="https://developer.mozilla.org/en-US/docs/Web/API/IDBKeyRange" target="_blank" rel="noopener">? MDN IDBKeyRange</a>
      </h2>
      <div class="rc-anno">
        <b>IDBKeyRange</b> фильтрует на стороне движка. Ключ книг — строка ISBN,
        поэтому <code>bound("978-000000", "978-000100")</code> отберёт только
        первые 100 книг. Курсор не читает остальные.
      </div>

      <div class="rc-form-row">
        <select id="rc-range-kind" title="Тип диапазона">
          <option value="none">без диапазона</option>
          <option value="lower">lowerBound ≥</option>
          <option value="upper">upperBound ≤</option>
          <option value="bound" selected>bound [a, b]</option>
        </select>
        <input type="text" id="rc-range-lo" value="978-000000" class="rc-key" title="Нижняя граница (строка)" placeholder="lo">
        <input type="text" id="rc-range-hi" value="978-000100" class="rc-key" title="Верхняя граница (строка)" placeholder="hi">
        <button id="rc-range-reset" class="rc-mini" title="Сброс">↺</button>
      </div>

      <div class="rc-form-row">
        <button id="rc-apply-range" class="rc-primary" title="Пройти курсором по диапазону">▶ Применить</button>
      </div>

      <div class="rc-kv">
        <div class="k">в диапазоне</div> <div class="v" id="rc-range-count">—</div>
        <div class="k">первый</div>      <div class="v" id="rc-range-first">—</div>
        <div class="k">последний</div>   <div class="v" id="rc-range-last">—</div>
      </div>

      <div class="rc-preview" id="rc-preview">
        <div class="rc-preview-empty">Нажмите «Применить»</div>
      </div>

      <div id="rc-status-range" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num ok">8</span> 🔄 Миграция через cursor.update
        <a class="rc-mdn" href="https://developer.mozilla.org/en-US/docs/Web/API/IDBCursor/update" target="_blank" rel="noopener">? MDN cursor.update</a>
      </h2>
      <div class="rc-anno">
        <b>Эволюция данных.</b> Добавить поле всем книгам — обход курсором с
        <code>cursor.update()</code>. Записи, у которых поле уже такое — пропускаются.
      </div>

      <div class="rc-form-row">
        <input type="text" id="rc-mig-field" placeholder="field name" value="shelf" title="Имя поля. isbn менять нельзя">
        <input type="text" id="rc-mig-value" placeholder="value" value="A1" title="Значение">
      </div>

      <div class="rc-form-row">
        <button id="rc-mig-set" class="rc-primary" title="Установить поле во всех записях books">⊕ set in all</button>
        <button id="rc-mig-remove" class="rc-danger" title="Удалить поле из всех записей books">⊖ remove from all</button>
      </div>

      <div class="rc-progress-wrap">
        <progress id="rc-mig-progress" value="0" max="100"></progress>
        <span class="rc-progress-label" id="rc-mig-progress-label">—</span>
      </div>

      <div class="rc-kv">
        <div class="k">обновлено</div>     <div class="v" id="rc-mig-updated">0</div>
        <div class="k">без изменений</div> <div class="v" id="rc-mig-skipped">0</div>
        <div class="k">время</div>         <div class="v" id="rc-mig-elapsed">—</div>
      </div>

      <div id="rc-status-mig" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">
        <span class="rc-num">9</span> 📊 Статистика и seed
      </h2>
      <div class="rc-anno">
        <b>Реальная библиотека.</b> 100 русских писателей, ~370 книг.
        Кнопка <b>Seed</b> <b>заменяет</b> текущие данные на выбранный диапазон.
        Кнопка <b>Reset</b> в баннере возвращает demo-данные.
      </div>

      <div class="rc-form-row">
        <select id="rc-seed-size" title="Сколько книг загрузить из библиотеки">
          <option value="30">30 книг</option>
          <option value="100" selected>100 книг</option>
          <option value="200">200 книг</option>
          <option value="500">500 книг</option>
        </select>
        <button id="rc-seed" class="rc-primary" title="Заменить данные на реальную библиотеку">▶ Seed books</button>
        <button id="rc-clear" class="rc-danger" title="Очистить все 4 store">Clear all</button>
      </div>

      <div class="rc-progress-wrap">
        <progress id="rc-seed-progress" value="0" max="100"></progress>
        <span class="rc-progress-label" id="rc-seed-progress-label">—</span>
      </div>

      <div class="rc-kv">
        <div class="k">authors</div>          <div class="v" id="rc-cnt-a">0</div>
        <div class="k">books</div>            <div class="v" id="rc-cnt-b">0</div>
        <div class="k">tags</div>             <div class="v" id="rc-cnt-t">0</div>
        <div class="k">book_tags (M:N)</div>  <div class="v" id="rc-cnt-lt">0</div>
        <div class="k">seed, время</div>      <div class="v" id="rc-stat-seed-time">—</div>
        <div class="k">последнее</div>        <div class="v" id="rc-stat-last">—</div>
      </div>

      <div id="rc-status-seed" class="rc-status"><span class="rc-status-icon">—</span><span>готово</span></div>
    </div>

    <div class="rc-panel">
      <h2 class="rc-panel-title">🔗 Модель связей</h2>
      <div class="rc-rel-list">
        <div><b>books.author_id</b> → authors.id <em>(1:N)</em></div>
        <div><b>book_tags.book_isbn</b> → books.isbn <em>(M:N, часть 1)</em></div>
        <div><b>book_tags.tag_id</b> → tags.id <em>(M:N, часть 2)</em></div>
      </div>
      <div class="rc-hint">
        Составной ключ <code>book_isbn,tag_id</code> гарантирует, что одна пара книга-тег
        не появится дважды. Удаление книги каскадно удаляет её рёбра.
      </div>
    </div>

  </div>

</div>
"#;

// ============================================================
//                       МОДЕЛЬ
// ============================================================

#[derive(Clone)]
struct Ui {
    db_name: Element,
    db_meta: Element,
    history: Element,
    total: Element,

    a_id: HtmlInputElement,
    a_name: HtmlInputElement,

    b_isbn: HtmlInputElement,
    b_title: HtmlInputElement,
    b_author: HtmlSelectElement,
    b_year: HtmlInputElement,

    t_id: HtmlInputElement,
    t_label: HtmlInputElement,

    lt_book: HtmlSelectElement,
    lt_tag: HtmlSelectElement,

    tree_authors: Element,
    tree_books: Element,
    badge_a: Element,
    badge_b: Element,

    direction: HtmlSelectElement,
    limit_on: HtmlInputElement,
    limit_val: HtmlInputElement,
    progress: Element,
    progress_label: Element,
    scanned: Element,
    elapsed: Element,
    rate: Element,

    range_kind: HtmlSelectElement,
    range_lo: HtmlInputElement,
    range_hi: HtmlInputElement,
    range_count: Element,
    range_first: Element,
    range_last: Element,
    preview: Element,

    mig_field: HtmlInputElement,
    mig_value: HtmlInputElement,
    mig_progress: Element,
    mig_progress_label: Element,
    mig_updated: Element,
    mig_skipped: Element,
    mig_elapsed: Element,

    seed_size: HtmlSelectElement,
    seed_progress: Element,
    seed_progress_label: Element,
    cnt_a: Element,
    cnt_b: Element,
    cnt_t: Element,
    cnt_lt: Element,
    stat_seed_time: Element,
    stat_last: Element,

    status_a: Element,
    status_b: Element,
    status_t: Element,
    status_lt: Element,
    status_tree_a: Element,
    status_tree_b: Element,
    status_iter: Element,
    status_range: Element,
    status_mig: Element,
    status_seed: Element,
}

struct App {
    db: Rc<RefCell<Option<Database>>>,
    ui: Ui,
    busy: Rc<RefCell<bool>>,
    all_buttons: Rc<Vec<HtmlButtonElement>>,
    recent: Rc<RefCell<Vec<String>>>,
}

impl App {
    fn set_status(el: &Element, kind: &str, icon: &str, msg: &str) {
        el.set_class_name(&format_str(format_args!("rc-status {kind}")));
        el.set_inner_html(&format_str(format_args!(
            "<span class=\"rc-status-icon\">{icon}</span><span>{}</span>",
            esc(msg),
        )));
    }
    fn status_info(el: &Element, msg: &str) { Self::set_status(el, "busy", "…", msg); }
    fn status_ok(el: &Element, msg: &str)   { Self::set_status(el, "ok", "✓", msg); }
    fn status_err(el: &Element, msg: &str)  { Self::set_status(el, "err", "✗", msg); }
    fn status_stale(el: &Element, msg: &str){ Self::set_status(el, "stale", "!", msg); }

    fn db_raw(&self) -> Option<JsValue> {
        let g = self.db.borrow();
        match g.as_ref() {
            Some(d) if !d.is_closed() => Some(d.raw().clone()),
            _ => None,
        }
    }

    fn set_busy(&self, v: bool) {
        *self.busy.borrow_mut() = v;
        for b in self.all_buttons.iter() {
            b.set_disabled(v);
        }
    }

    fn push_recent(&self, label: String) {
        let mut r = self.recent.borrow_mut();
        r.push(label);
        if r.len() > 3 { r.remove(0); }
        drop(r);
        self.render_history();
    }

    fn render_history(&self) {
        let r = self.recent.borrow();
        if r.is_empty() { self.ui.history.set_inner_html(""); return; }
        let mut html = String::new();
        html.push_str("<span style=\"opacity:0.7\">недавние:</span>");
        for s in r.iter() {
            html.push_str(&format_str(format_args!(
                "<span class=\"rc-hist\" title=\"{}\">{}</span>",
                esc(s), esc(s),
            )));
        }
        self.ui.history.set_inner_html(&html);
    }

    fn refresh_banner(&self) {
        self.ui.db_name.set_text_content(Some(DB_NAME));
        let (ver, stores) = {
            let g = self.db.borrow();
            match g.as_ref() {
                Some(d) if !d.is_closed() => (d.version(), d.store_names().len()),
                _ => (0, 0),
            }
        };
        let ver_text = if ver == 0 { "не открыто".to_string() }
                       else { format_str(format_args!("v{ver}")) };
        self.ui.db_meta.set_text_content(Some(&format_str(format_args!(
            "версия: {ver_text} · stores: {stores}",
        ))));
    }

    fn set_last(&self, text: &str) {
        self.ui.stat_last.set_text_content(Some(text));
    }
}

// ============================================================
//                        RUN
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        db_name: get_el("rc-db-name")?,
        db_meta: get_el("rc-db-meta")?,
        history: get_el("rc-history")?,
        total: get_el("rc-total")?,

        a_id: get_input("rc-a-id")?,
        a_name: get_input("rc-a-name")?,

        b_isbn: get_input("rc-b-isbn")?,
        b_title: get_input("rc-b-title")?,
        b_author: get_select("rc-b-author")?,
        b_year: get_input("rc-b-year")?,

        t_id: get_input("rc-t-id")?,
        t_label: get_input("rc-t-label")?,

        lt_book: get_select("rc-lt-book")?,
        lt_tag: get_select("rc-lt-tag")?,

        tree_authors: get_el("rc-tree-authors")?,
        tree_books: get_el("rc-tree-books")?,
        badge_a: get_el("rc-badge-a")?,
        badge_b: get_el("rc-badge-b")?,

        direction: get_select("rc-direction")?,
        limit_on: get_input("rc-limit-on")?,
        limit_val: get_input("rc-limit-val")?,
        progress: get_el("rc-progress")?,
        progress_label: get_el("rc-progress-label")?,
        scanned: get_el("rc-scanned")?,
        elapsed: get_el("rc-elapsed")?,
        rate: get_el("rc-rate")?,

        range_kind: get_select("rc-range-kind")?,
        range_lo: get_input("rc-range-lo")?,
        range_hi: get_input("rc-range-hi")?,
        range_count: get_el("rc-range-count")?,
        range_first: get_el("rc-range-first")?,
        range_last: get_el("rc-range-last")?,
        preview: get_el("rc-preview")?,

        mig_field: get_input("rc-mig-field")?,
        mig_value: get_input("rc-mig-value")?,
        mig_progress: get_el("rc-mig-progress")?,
        mig_progress_label: get_el("rc-mig-progress-label")?,
        mig_updated: get_el("rc-mig-updated")?,
        mig_skipped: get_el("rc-mig-skipped")?,
        mig_elapsed: get_el("rc-mig-elapsed")?,

        seed_size: get_select("rc-seed-size")?,
        seed_progress: get_el("rc-seed-progress")?,
        seed_progress_label: get_el("rc-seed-progress-label")?,
        cnt_a: get_el("rc-cnt-a")?,
        cnt_b: get_el("rc-cnt-b")?,
        cnt_t: get_el("rc-cnt-t")?,
        cnt_lt: get_el("rc-cnt-lt")?,
        stat_seed_time: get_el("rc-stat-seed-time")?,
        stat_last: get_el("rc-stat-last")?,

        status_a: get_el("rc-status-a")?,
        status_b: get_el("rc-status-b")?,
        status_t: get_el("rc-status-t")?,
        status_lt: get_el("rc-status-lt")?,
        status_tree_a: get_el("rc-status-tree-a")?,
        status_tree_b: get_el("rc-status-tree-b")?,
        status_iter: get_el("rc-status-iter")?,
        status_range: get_el("rc-status-range")?,
        status_mig: get_el("rc-status-mig")?,
        status_seed: get_el("rc-status-seed")?,
    };

    let btn_ids = [
        "rc-reset", "rc-refresh-all",
        "rc-a-add", "rc-a-fill",
        "rc-b-add", "rc-b-fill",
        "rc-t-add", "rc-t-fill",
        "rc-lt-add",
        "rc-limit-reset", "rc-iter-yield", "rc-iter-noyield",
        "rc-range-reset", "rc-apply-range",
        "rc-mig-set", "rc-mig-remove",
        "rc-seed", "rc-clear",
    ];
    let mut all_buttons_vec: Vec<HtmlButtonElement> = Vec::with_capacity(btn_ids.len());
    for id in btn_ids {
        all_buttons_vec.push(get_button(id)?);
    }

    let app = Rc::new(App {
        db: Rc::new(RefCell::new(None)),
        ui,
        busy: Rc::new(RefCell::new(false)),
        all_buttons: Rc::new(all_buttons_vec),
        recent: Rc::new(RefCell::new(Vec::new())),
    });

    app.refresh_banner();
    app.render_history();

    {
        let app_c = app.clone();
        spawn_local(async move { bootstrap(&app_c).await; });
    }

    for id in ["rc-b-year", "rc-limit-val"] {
        let el = get_input(id)?;
        let cb = Closure::<dyn FnMut(web_sys::Event)>::new(move |ev: web_sys::Event| {
            let target = match ev.target() { Some(t) => t, None => return };
            let input: HtmlInputElement = match target.dyn_into() {
                Ok(i) => i, Err(_) => return,
            };
            let v = input.value();
            let v = v.trim();
            if v.is_empty() || v.parse::<f64>().is_ok() {
                input.set_class_name("rc-key");
            } else {
                input.set_class_name("rc-key rc-invalid");
            }
        });
        el.add_event_listener_with_callback("input", cb.as_ref().unchecked_ref())?;
        cb.forget();
    }
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::KeyboardEvent)>::new(
            move |ev: web_sys::KeyboardEvent| {
                if ev.key() == "Enter" {
                    ev.prevent_default();
                    let app_in = app_c.clone();
                    add_book(app_in);
                }
            });
        app.ui.b_isbn.add_event_listener_with_callback("keydown", cb.as_ref().unchecked_ref())?;
        cb.forget();
    }
    {
        let app_c = app.clone();
        let cb = Closure::<dyn FnMut(web_sys::KeyboardEvent)>::new(
            move |ev: web_sys::KeyboardEvent| {
                if ev.key() == "Escape" {
                    app_c.ui.a_id.set_value("");
                    app_c.ui.a_name.set_value("");
                    app_c.ui.b_isbn.set_value("");
                    app_c.ui.b_title.set_value("");
                    app_c.ui.b_year.set_value("");
                    app_c.ui.t_id.set_value("");
                    app_c.ui.t_label.set_value("");
                    App::status_ok(&app_c.ui.status_a, "форма сброшена");
                }
            });
        if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
            doc.add_event_listener_with_callback("keydown", cb.as_ref().unchecked_ref())?;
        }
        cb.forget();
    }

    {
        let app_c = app.clone();
        let btn = get_button("rc-reset")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            let ok = web_sys::window()
                .and_then(|w| w.confirm_with_message(
                    "Удалить базу целиком и создать заново с demo-данными?"
                ).ok()).unwrap_or(false);
            if !ok { return; }

            app_in.set_busy(true);
            App::status_info(&app_in.ui.status_seed, "reset + re-seed…");
            spawn_local(async move {
                if let Some(d) = app_in.db.borrow_mut().take() { d.close(); }
                let _ = idb_factory::delete_by_name(DB_NAME).await;
                if let Ok(o) = idb_schema::open_with_plan(DB_NAME, 1, &make_plan()).await {
                    *app_in.db.borrow_mut() = Some(o.db);
                }
                app_in.refresh_banner();
                seed_demo(&app_in).await;
                reload_all(&app_in).await;
                app_in.recent.borrow_mut().clear();
                app_in.render_history();
                App::status_ok(&app_in.ui.status_seed, "база пересоздана с demo-данными");
                app_in.set_busy(false);
            });
        });
    }

    {
        let app_c = app.clone();
        let btn = get_button("rc-refresh-all")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            if *app_in.busy.borrow() { return; }
            spawn_local(async move {
                reload_all(&app_in).await;
                App::status_ok(&app_in.ui.status_tree_a, "обновлено");
            });
        });
    }

    {
        let app_c = app.clone();
        let btn = get_button("rc-a-add")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            add_author(app_in);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-a-fill")?;
        set_click(&btn, move |_| {
            let n = (js_sys::Date::now() as u64) % 10000;
            app_c.ui.a_id.set_value(&format_str(format_args!("author-{n}")));
            app_c.ui.a_name.set_value(&format_str(format_args!("Новый автор {n}")));
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-b-add")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            add_book(app_in);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-b-fill")?;
        set_click(&btn, move |_| {
            let n = (js_sys::Date::now() as u64) % 10000;
            app_c.ui.b_isbn.set_value(&format_str(format_args!("978-9{n}")));
            app_c.ui.b_title.set_value(&format_str(format_args!("Новая книга {n}")));
            app_c.ui.b_year.set_value("2024");
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-t-add")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            add_tag(app_in);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-t-fill")?;
        set_click(&btn, move |_| {
            let n = (js_sys::Date::now() as u64) % 10000;
            app_c.ui.t_id.set_value(&format_str(format_args!("tag-{n}")));
            app_c.ui.t_label.set_value(&format_str(format_args!("Тег {n}")));
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-lt-add")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            add_link(app_in);
        });
    }

    for (id, mode) in [
        ("rc-iter-yield",   IterMode::Yield),
        ("rc-iter-noyield", IterMode::GetAll),
    ] {
        let app_c = app.clone();
        let btn = get_button(id)?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            iterate(app_in, mode);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-limit-reset")?;
        set_click(&btn, move |_| {
            app_c.ui.limit_val.set_value("1000");
            app_c.ui.limit_val.set_class_name("rc-key");
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-range-reset")?;
        set_click(&btn, move |_| {
            app_c.ui.range_kind.set_value("bound");
            app_c.ui.range_lo.set_value("978-000000");
            app_c.ui.range_hi.set_value("978-000100");
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-apply-range")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            apply_range(app_in);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-mig-set")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            run_migration(app_in, true);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-mig-remove")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            run_migration(app_in, false);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-seed")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            run_seed(app_in);
        });
    }
    {
        let app_c = app.clone();
        let btn = get_button("rc-clear")?;
        set_click(&btn, move |_| {
            let app_in = app_c.clone();
            let ok = web_sys::window()
                .and_then(|w| w.confirm_with_message("Очистить все 4 store?").ok())
                .unwrap_or(false);
            if !ok { return; }
            run_clear(app_in);
        });
    }

    Ok(())
}

// ============================================================
//                  SCHEMA
// ============================================================

fn make_plan() -> SchemaPlan {
    SchemaPlan::new()
        .add(StoreSpec::with_key(ST_AUTHORS, "id"))
        .add(StoreSpec::with_key(ST_BOOKS, "isbn").indexes(vec![
            IndexSpec::new("by_author", "author_id"),
            IndexSpec::new("by_year", "year"),
        ]))
        .add(StoreSpec::with_key(ST_TAGS, "id"))
        .add(StoreSpec::with_key(ST_BOOK_TAGS, "book_isbn,tag_id").indexes(vec![
            IndexSpec::new("by_book", "book_isbn"),
            IndexSpec::new("by_tag", "tag_id"),
        ]))
}

// ============================================================
//                  BOOTSTRAP + DEMO SEED
// ============================================================

async fn bootstrap(app: &Rc<App>) {
    let existing = idb_factory::list_databases().await
        .ok()
        .and_then(|list| list.into_iter().find(|d| d.name == DB_NAME));

    let db = if let Some(info) = existing {
        match idb_factory::open_by_name(DB_NAME, info.version).await {
            Ok(raw) => Database::from_raw(raw),
            Err(e) => {
                App::status_err(&app.ui.status_a,
                    &format_str(format_args!("open: {}", short_error(&e))));
                return;
            }
        }
    } else {
        match idb_schema::open_with_plan(DB_NAME, 1, &make_plan()).await {
            Ok(o) => o.db,
            Err(e) => {
                App::status_err(&app.ui.status_a,
                    &format_str(format_args!("init: {}", short_error(&e))));
                return;
            }
        }
    };

    *app.db.borrow_mut() = Some(db);
    app.refresh_banner();

    let count_a = count_store(app, ST_AUTHORS).await;
    if count_a == 0 {
        App::status_info(&app.ui.status_seed, "первичный demo-сид…");
        seed_demo(app).await;
        App::status_ok(&app.ui.status_seed, "demo-сид выполнен");
    }

    reload_all(app).await;
}

/// Начальный demo-сид при первом открытии.
async fn seed_demo(app: &Rc<App>) {
    let (authors, books, tags, links) = demo_data();

    if let Some(db_raw) = app.db_raw() {
        if let Ok(s) = RecordStore::open(&db_raw, ST_AUTHORS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &authors).await;
        }
    }
    if let Some(db_raw) = app.db_raw() {
        if let Ok(s) = RecordStore::open(&db_raw, ST_BOOKS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &books).await;
        }
    }
    if let Some(db_raw) = app.db_raw() {
        if let Ok(s) = RecordStore::open(&db_raw, ST_TAGS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &tags).await;
        }
    }
    if let Some(db_raw) = app.db_raw() {
        if let Ok(s) = RecordStore::open(&db_raw, ST_BOOK_TAGS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &links).await;
        }
    }
}

fn demo_data() -> (Vec<JsValue>, Vec<JsValue>, Vec<JsValue>, Vec<JsValue>) {
    let authors_raw: [(&str, &str); 8] = [
        ("tolstoy_ln", "Лев Толстой"),
        ("dostoevsky", "Фёдор Достоевский"),
        ("bulgakov",   "Михаил Булгаков"),
        ("chekhov",    "Антон Чехов"),
        ("nabokov",    "Владимир Набоков"),
        ("pushkin",    "Александр Пушкин"),
        ("gogol",      "Николай Гоголь"),
        ("esenin",     "Сергей Есенин"),
    ];

    let books_raw: [(&str, &str, &str, u32, &str); 30] = [
        ("978-000001", "Война и мир",              "tolstoy_ln", 1869, "novel"),
        ("978-000002", "Анна Каренина",            "tolstoy_ln", 1878, "novel"),
        ("978-000003", "Воскресение",              "tolstoy_ln", 1899, "novel"),
        ("978-000004", "Преступление и наказание", "dostoevsky", 1866, "novel"),
        ("978-000005", "Идиот",                    "dostoevsky", 1869, "novel"),
        ("978-000006", "Бесы",                     "dostoevsky", 1872, "novel"),
        ("978-000007", "Братья Карамазовы",        "dostoevsky", 1880, "novel"),
        ("978-000008", "Мастер и Маргарита",       "bulgakov",   1967, "novel"),
        ("978-000009", "Белая гвардия",            "bulgakov",   1925, "novel"),
        ("978-000010", "Собачье сердце",           "bulgakov",   1925, "novella"),
        ("978-000011", "Театральный роман",        "bulgakov",   1965, "novel"),
        ("978-000012", "Вишнёвый сад",             "chekhov",    1904, "drama"),
        ("978-000013", "Чайка",                    "chekhov",    1896, "drama"),
        ("978-000014", "Три сестры",               "chekhov",    1901, "drama"),
        ("978-000015", "Дама с собачкой",          "chekhov",    1899, "story"),
        ("978-000016", "Лолита",                   "nabokov",    1955, "novel"),
        ("978-000017", "Дар",                      "nabokov",    1938, "novel"),
        ("978-000018", "Защита Лужина",            "nabokov",    1930, "novel"),
        ("978-000019", "Приглашение на казнь",     "nabokov",    1938, "novel"),
        ("978-000020", "Евгений Онегин",           "pushkin",    1833, "poem_long"),
        ("978-000021", "Капитанская дочка",        "pushkin",    1836, "novel"),
        ("978-000022", "Борис Годунов",            "pushkin",    1825, "tragedy"),
        ("978-000023", "Медный всадник",           "pushkin",    1833, "poem_long"),
        ("978-000024", "Мёртвые души",             "gogol",      1842, "novel"),
        ("978-000025", "Ревизор",                  "gogol",      1836, "comedy"),
        ("978-000026", "Тарас Бульба",             "gogol",      1835, "novella"),
        ("978-000027", "Шинель",                   "gogol",      1842, "story"),
        ("978-000028", "Москва кабацкая",          "esenin",     1924, "verse"),
        ("978-000029", "Анна Снегина",             "esenin",     1925, "poem_long"),
        ("978-000030", "Чёрный человек",           "esenin",     1925, "poem_long"),
    ];

    let authors: Vec<JsValue> = authors_raw.iter().map(|(id, name)| {
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(id));
        let _ = js_sys::Reflect::set(&o, &"name".into(), &JsValue::from_str(name));
        o.into()
    }).collect();

    let books: Vec<JsValue> = books_raw.iter().map(|(isbn, title, aid, year, _genre)| {
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"isbn".into(), &JsValue::from_str(isbn));
        let _ = js_sys::Reflect::set(&o, &"title".into(), &JsValue::from_str(title));
        let _ = js_sys::Reflect::set(&o, &"author_id".into(), &JsValue::from_str(aid));
        let _ = js_sys::Reflect::set(&o, &"year".into(), &JsValue::from_f64(*year as f64));
        o.into()
    }).collect();

    let tags: Vec<JsValue> = SEED_GENRES.iter().map(|(id, label)| {
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(id));
        let _ = js_sys::Reflect::set(&o, &"label".into(), &JsValue::from_str(label));
        o.into()
    }).collect();

    let mut links: Vec<JsValue> = Vec::new();
    for (isbn, _t, _a, year, genre) in books_raw.iter() {
        push_link(&mut links, isbn, genre);
        push_link(&mut links, isbn, "russian");
        let era = if *year < 1900 { "19c" } else if *year < 2000 { "20c" } else { "21c" };
        push_link(&mut links, isbn, era);
    }

    (authors, books, tags, links)
}

// ============================================================
//                  RELOAD ALL
// ============================================================

async fn reload_all(app: &Rc<App>) {
    let Some(db_raw) = app.db_raw() else { return; };

    let cnt_a  = count_store_by_raw(&db_raw, ST_AUTHORS).await;
    let cnt_b  = count_store_by_raw(&db_raw, ST_BOOKS).await;
    let cnt_t  = count_store_by_raw(&db_raw, ST_TAGS).await;
    let cnt_lt = count_store_by_raw(&db_raw, ST_BOOK_TAGS).await;

    app.ui.cnt_a.set_text_content(Some(&format_str(format_args!("{cnt_a}"))));
    app.ui.cnt_b.set_text_content(Some(&format_str(format_args!("{cnt_b}"))));
    app.ui.cnt_t.set_text_content(Some(&format_str(format_args!("{cnt_t}"))));
    app.ui.cnt_lt.set_text_content(Some(&format_str(format_args!("{cnt_lt}"))));
    app.ui.total.set_text_content(Some(&format_str(format_args!(
        "{} книг · {} авторов", cnt_b, cnt_a,
    ))));

    app.ui.badge_a.set_text_content(Some(&format_str(format_args!(
        "{cnt_a} авторов · {cnt_b} книг"
    ))));
    app.ui.badge_b.set_text_content(Some(&format_str(format_args!(
        "{cnt_b} книг · {cnt_lt} связей"
    ))));

    let authors = read_all(&db_raw, ST_AUTHORS).await;
    let books   = read_all(&db_raw, ST_BOOKS).await;
    let tags    = read_all(&db_raw, ST_TAGS).await;
    let links   = read_all(&db_raw, ST_BOOK_TAGS).await;

    refresh_select_authors(&app.ui.b_author, &authors);
    refresh_select_books(&app.ui.lt_book, &books);
    refresh_select_tags(&app.ui.lt_tag, &tags);

    render_tree_authors(app, &authors, &books, &links);
    render_tree_books(app, &books, &tags, &links);
}

async fn read_all(db: &JsValue, store_name: &str) -> Vec<JsValue> {
    let s = match RecordStore::open(db, store_name, "readonly") {
        Ok(s) => s, Err(_) => return Vec::new(),
    };
    return idb_records::get_all(&s).await.unwrap_or_default();
}

async fn count_store(app: &Rc<App>, store_name: &str) -> u32 {
    let Some(db) = app.db_raw() else { return 0; };
    return count_store_by_raw(&db, store_name).await;
}

async fn count_store_by_raw(db: &JsValue, store_name: &str) -> u32 {
    let s = match RecordStore::open(db, store_name, "readonly") {
        Ok(s) => s, Err(_) => return 0,
    };
    return idb_records::count(&s).await.unwrap_or(0);
}

// ============================================================
//                  RENDER TREES
// ============================================================

fn render_tree_authors(
    app: &Rc<App>,
    authors: &[JsValue],
    books: &[JsValue],
    links: &[JsValue],
) {
    if authors.is_empty() {
        app.ui.tree_authors.set_inner_html(
            "<div class=\"rc-empty-leaf\">Авторов пока нет</div>");
        return;
    }

    let mut link_count_by_book: std::collections::HashMap<String, usize> =
        std::collections::HashMap::new();
    for l in links {
        let isbn = js_sys::Reflect::get(l, &"book_isbn".into())
            .ok().and_then(|v| v.as_string());
        if let Some(s) = isbn {
            *link_count_by_book.entry(s).or_insert(0) += 1;
        }
    }

    let mut by_author: std::collections::HashMap<String, Vec<&JsValue>> =
        std::collections::HashMap::new();
    for b in books {
        let aid = js_sys::Reflect::get(b, &"author_id".into())
            .ok().and_then(|v| v.as_string());
        if let Some(s) = aid {
            by_author.entry(s).or_default().push(b);
        }
    }

    let mut html = String::with_capacity(4096);
    for a in authors {
        let aid = js_sys::Reflect::get(a, &"id".into())
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        let name = js_sys::Reflect::get(a, &"name".into())
            .ok().and_then(|v| v.as_string()).unwrap_or_default();
        let his_books = by_author.get(&aid).cloned().unwrap_or_default();
        let n = his_books.len();

        html.push_str("<details class=\"rc-node\"><summary>");
        html.push_str("<span class=\"rc-name\">");
        html.push_str(&esc(&name));
        html.push_str("</span>");
        if n > 0 {
            html.push_str("<span class=\"rc-count\">");
            html.push_str(&format_str(format_args!("{n} книг")));
            html.push_str("</span>");
        } else {
            html.push_str("<span class=\"rc-count empty\">нет книг</span>");
        }
        html.push_str("<button class=\"rc-x\" data-kind=\"del-author\" data-id=\"");
        html.push_str(&esc(&aid));
        html.push_str("\" title=\"Каскадно удалить автора и его книги\">✗</button>");
        html.push_str("</summary><div class=\"rc-node-body\">");

        if his_books.is_empty() {
            html.push_str("<div class=\"rc-empty-leaf\">книг нет</div>");
        } else {
            for b in &his_books {
                let isbn = js_sys::Reflect::get(b, &"isbn".into())
                    .ok().and_then(|v| v.as_string()).unwrap_or_default();
                let title = js_sys::Reflect::get(b, &"title".into())
                    .ok().and_then(|v| v.as_string()).unwrap_or_default();
                let year = js_sys::Reflect::get(b, &"year".into())
                    .ok().and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
                let n_tags = link_count_by_book.get(&isbn).copied().unwrap_or(0);

                html.push_str("<div class=\"rc-leaf\">");
                html.push_str("<span class=\"rc-l-name\">");
                html.push_str(&esc(&title));
                html.push_str("</span>");
                html.push_str("<span class=\"rc-l-sub\">");
                html.push_str(&format_str(format_args!("{year} · {n_tags} тегов")));
                html.push_str("</span>");
                html.push_str("<button class=\"rc-x\" data-kind=\"del-book\" data-id=\"");
                html.push_str(&esc(&isbn));
                html.push_str("\" title=\"Удалить книгу и её связи\">✗</button>");
                html.push_str("</div>");
            }
        }
        html.push_str("</div></details>");
    }
    app.ui.tree_authors.set_inner_html(&html);
    attach_tree_handlers(app.clone(), "rc-tree-authors");
}

fn render_tree_books(
    app: &Rc<App>,
    books: &[JsValue],
    tags: &[JsValue],
    links: &[JsValue],
) {
    if books.is_empty() {
        app.ui.tree_books.set_inner_html(
            "<div class=\"rc-empty-leaf\">Книг пока нет</div>");
        return;
    }

    let tag_label: std::collections::HashMap<String, String> = {
        let mut m: std::collections::HashMap<String, String> = Default::default();
        for t in tags {
            let id = js_sys::Reflect::get(t, &"id".into()).ok()
                .and_then(|v| v.as_string()).unwrap_or_default();
            let label = js_sys::Reflect::get(t, &"label".into()).ok()
                .and_then(|v| v.as_string()).unwrap_or_default();
            m.insert(id, label);
        }
        m
    };

    let mut links_by_book: std::collections::HashMap<String, Vec<String>> = Default::default();
    for l in links {
        let isbn = js_sys::Reflect::get(l, &"book_isbn".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        let tid = js_sys::Reflect::get(l, &"tag_id".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if !isbn.is_empty() && !tid.is_empty() {
            links_by_book.entry(isbn).or_default().push(tid);
        }
    }

    let total = books.len();
    let take = total.min(MAX_TABLE_ROWS);
    let mut html = String::with_capacity(take * 200);

    if total > take {
        html.push_str("<div class=\"rc-empty-leaf\">");
        html.push_str(&format_str(format_args!("Показаны первые {take} из {total}")));
        html.push_str("</div>");
    }

    for b in books.iter().take(take) {
        let isbn = js_sys::Reflect::get(b, &"isbn".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        let title = js_sys::Reflect::get(b, &"title".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        let year = js_sys::Reflect::get(b, &"year".into()).ok()
            .and_then(|v| v.as_f64()).unwrap_or(0.0) as u32;
        let tags_of_book = links_by_book.get(&isbn).cloned().unwrap_or_default();
        let n = tags_of_book.len();

        html.push_str("<details class=\"rc-node\"><summary>");
        html.push_str("<span class=\"rc-name\">");
        html.push_str(&esc(&title));
        html.push_str("</span>");
        html.push_str("<span class=\"rc-sub\">");
        html.push_str(&format_str(format_args!("{year}")));
        html.push_str("</span>");
        if n > 0 {
            html.push_str("<span class=\"rc-count\">");
            html.push_str(&format_str(format_args!("{n} тегов")));
            html.push_str("</span>");
        } else {
            html.push_str("<span class=\"rc-count empty\">нет тегов</span>");
        }
        html.push_str("<button class=\"rc-x\" data-kind=\"del-book\" data-id=\"");
        html.push_str(&esc(&isbn));
        html.push_str("\" title=\"Удалить книгу и её связи\">✗</button>");
        html.push_str("</summary><div class=\"rc-node-body\">");

        if tags_of_book.is_empty() {
            html.push_str("<div class=\"rc-empty-leaf\">тегов нет</div>");
        } else {
            html.push_str("<div class=\"rc-chips\">");
            for tid in &tags_of_book {
                let label = tag_label.get(tid).cloned().unwrap_or_else(|| tid.clone());

                html.push_str("<span class=\"rc-chip\">");
                html.push_str(&esc(&label));
                html.push_str("<button class=\"rc-x\" data-kind=\"unlink\" data-book=\"");
                html.push_str(&esc(&isbn));
                html.push_str("\" data-tag=\"");
                html.push_str(&esc(tid));
                html.push_str("\" title=\"Удалить связь\">×</button>");
                html.push_str("</span>");
            }
            html.push_str("</div>");
        }
        html.push_str("</div></details>");
    }
    app.ui.tree_books.set_inner_html(&html);
    attach_tree_handlers(app.clone(), "rc-tree-books");
}

fn attach_tree_handlers(app: Rc<App>, container_id: &str) {
    let container = match get_el(container_id) {
        Ok(el) => el,
        Err(_) => return,
    };
    let app_cb = app.clone();
    let cb = Closure::<dyn FnMut(web_sys::MouseEvent)>::new(
        move |ev: web_sys::MouseEvent| {
            let target = match ev.target() {
                Some(t) => t, None => return,
            };
            let el: Element = match target.dyn_into() {
                Ok(e) => e, Err(_) => return,
            };
            let kind = el.get_attribute("data-kind").unwrap_or_default();
            if kind.is_empty() { return; }
            ev.stop_propagation();
            ev.prevent_default();

            let app_in = app_cb.clone();
            if kind == "del-author" {
                let id = el.get_attribute("data-id").unwrap_or_default();
                spawn_local(async move { delete_author_cascade(app_in, id).await; });
            } else if kind == "del-book" {
                let id = el.get_attribute("data-id").unwrap_or_default();
                spawn_local(async move { delete_book_cascade(app_in, id).await; });
            } else if kind == "unlink" {
                let book = el.get_attribute("data-book").unwrap_or_default();
                let tag = el.get_attribute("data-tag").unwrap_or_default();
                spawn_local(async move { unlink(app_in, book, tag).await; });
            }
        },
    );
    let _ = container.add_event_listener_with_callback(
        "click", cb.as_ref().unchecked_ref());
    cb.forget();
}

// ============================================================
//                  SELECTS
// ============================================================

fn refresh_select_authors(select: &HtmlSelectElement, authors: &[JsValue]) {
    let prev = select.value();
    let document = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d, None => return,
    };
    select.set_inner_html("");
    if authors.is_empty() {
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some("— нет авторов —"));
            let _ = opt.set_attribute("value", "");
            let _ = select.append_child(&opt);
        }
        return;
    }
    for a in authors {
        let id = js_sys::Reflect::get(a, &"id".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        let name = js_sys::Reflect::get(a, &"name".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some(&format_str(format_args!("{name}"))));
            let _ = opt.set_attribute("value", &id);
            let _ = select.append_child(&opt);
        }
    }
    if !prev.is_empty() { select.set_value(&prev); }
}

fn refresh_select_books(select: &HtmlSelectElement, books: &[JsValue]) {
    let prev = select.value();
    let document = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d, None => return,
    };
    select.set_inner_html("");
    if books.is_empty() {
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some("— нет книг —"));
            let _ = opt.set_attribute("value", "");
            let _ = select.append_child(&opt);
        }
        return;
    }
    for b in books.iter().take(200) {
        let isbn = js_sys::Reflect::get(b, &"isbn".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        let title = js_sys::Reflect::get(b, &"title".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some(&format_str(format_args!("{title}"))));
            let _ = opt.set_attribute("value", &isbn);
            let _ = select.append_child(&opt);
        }
    }
    if !prev.is_empty() { select.set_value(&prev); }
}

fn refresh_select_tags(select: &HtmlSelectElement, tags: &[JsValue]) {
    let prev = select.value();
    let document = match web_sys::window().and_then(|w| w.document()) {
        Some(d) => d, None => return,
    };
    select.set_inner_html("");
    if tags.is_empty() {
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some("— нет тегов —"));
            let _ = opt.set_attribute("value", "");
            let _ = select.append_child(&opt);
        }
        return;
    }
    for t in tags {
        let id = js_sys::Reflect::get(t, &"id".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        let label = js_sys::Reflect::get(t, &"label".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if let Ok(opt) = document.create_element("option") {
            opt.set_text_content(Some(&format_str(format_args!("{label}"))));
            let _ = opt.set_attribute("value", &id);
            let _ = select.append_child(&opt);
        }
    }
    if !prev.is_empty() { select.set_value(&prev); }
}

// ============================================================
//                  CRUD OPERATIONS
// ============================================================

fn add_author(app: Rc<App>) {
    if *app.busy.borrow() { return; }
    let id = app.ui.a_id.value().trim().to_string();
    let name = app.ui.a_name.value().trim().to_string();
    if id.is_empty() || name.is_empty() {
        App::status_err(&app.ui.status_a, "нужны id и имя");
        return;
    }
    app.set_busy(true);
    App::status_info(&app.ui.status_a, &format_str(format_args!("put author {id}…")));

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        let store = match RecordStore::open(&db_raw, ST_AUTHORS, "readwrite") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_a,
                    &format_str(format_args!("open: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(&id));
        let _ = js_sys::Reflect::set(&o, &"name".into(), &JsValue::from_str(&name));
        let t0 = now_ms();
        let res = idb_records::put(&store, &o.into()).await;
        let dt = now_ms() - t0;
        match res {
            Ok(_) => {
                App::status_ok(&app.ui.status_a,
                    &format_str(format_args!("author {id} сохранён за {}", format_time(dt))));
                app.push_recent(format_str(format_args!("+ author {id}")));
                app.ui.a_id.set_value("");
                app.ui.a_name.set_value("");
                reload_all(&app).await;
            }
            Err(e) => App::status_err(&app.ui.status_a,
                &format_str(format_args!("put: {}", short_error(&e)))),
        }
        app.set_busy(false);
    });
}

fn add_book(app: Rc<App>) {
    if *app.busy.borrow() { return; }
    let isbn = app.ui.b_isbn.value().trim().to_string();
    let title = app.ui.b_title.value().trim().to_string();
    let author_id = app.ui.b_author.value();
    let year_raw = app.ui.b_year.value().trim().to_string();

    if isbn.is_empty() || title.is_empty() || author_id.is_empty() {
        App::status_err(&app.ui.status_b, "нужны isbn, title, author");
        return;
    }
    let year: u32 = match year_raw.parse() {
        Ok(y) => y,
        Err(_) if year_raw.is_empty() => 0,
        Err(_) => {
            App::status_err(&app.ui.status_b, "год должен быть числом");
            app.ui.b_year.set_class_name("rc-key rc-invalid");
            return;
        }
    };
    app.ui.b_year.set_class_name("rc-key");

    app.set_busy(true);
    App::status_info(&app.ui.status_b, &format_str(format_args!("put book {isbn}…")));

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };

        let authors_store = match RecordStore::open(&db_raw, ST_AUTHORS, "readonly") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_b,
                    &format_str(format_args!("open authors: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };
        let author_ok = idb_records::get(&authors_store, &JsValue::from_str(&author_id))
            .await.ok().flatten().is_some();
        if !author_ok {
            App::status_err(&app.ui.status_b,
                &format_str(format_args!(
                    "author_id={author_id} не найден — сначала создайте автора"
                )));
            app.set_busy(false); return;
        }

        let store = match RecordStore::open(&db_raw, ST_BOOKS, "readwrite") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_b,
                    &format_str(format_args!("open books: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"isbn".into(), &JsValue::from_str(&isbn));
        let _ = js_sys::Reflect::set(&o, &"title".into(), &JsValue::from_str(&title));
        let _ = js_sys::Reflect::set(&o, &"author_id".into(), &JsValue::from_str(&author_id));
        let _ = js_sys::Reflect::set(&o, &"year".into(), &JsValue::from_f64(year as f64));

        let t0 = now_ms();
        let res = idb_records::put(&store, &o.into()).await;
        let dt = now_ms() - t0;
        match res {
            Ok(_) => {
                App::status_ok(&app.ui.status_b,
                    &format_str(format_args!(
                        "book {isbn} сохранена за {} (author={author_id})",
                        format_time(dt),
                    )));
                app.push_recent(format_str(format_args!("+ book {isbn}")));
                app.ui.b_isbn.set_value("");
                app.ui.b_title.set_value("");
                app.ui.b_year.set_value("");
                reload_all(&app).await;
            }
            Err(e) => App::status_err(&app.ui.status_b,
                &format_str(format_args!("put: {}", short_error(&e)))),
        }
        app.set_busy(false);
    });
}

fn add_tag(app: Rc<App>) {
    if *app.busy.borrow() { return; }
    let id = app.ui.t_id.value().trim().to_string();
    let label = app.ui.t_label.value().trim().to_string();
    if id.is_empty() || label.is_empty() {
        App::status_err(&app.ui.status_t, "нужны id и метка");
        return;
    }
    app.set_busy(true);
    App::status_info(&app.ui.status_t, &format_str(format_args!("put tag {id}…")));

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        let store = match RecordStore::open(&db_raw, ST_TAGS, "readwrite") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_t,
                    &format_str(format_args!("open: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(&id));
        let _ = js_sys::Reflect::set(&o, &"label".into(), &JsValue::from_str(&label));
        match idb_records::put(&store, &o.into()).await {
            Ok(_) => {
                App::status_ok(&app.ui.status_t, &format_str(format_args!("tag {id} сохранён")));
                app.push_recent(format_str(format_args!("+ tag {id}")));
                app.ui.t_id.set_value("");
                app.ui.t_label.set_value("");
                reload_all(&app).await;
            }
            Err(e) => App::status_err(&app.ui.status_t,
                &format_str(format_args!("put: {}", short_error(&e)))),
        }
        app.set_busy(false);
    });
}

fn add_link(app: Rc<App>) {
    if *app.busy.borrow() { return; }
    let book = app.ui.lt_book.value();
    let tag = app.ui.lt_tag.value();
    if book.is_empty() || tag.is_empty() {
        App::status_err(&app.ui.status_lt, "выберите книгу и тег");
        return;
    }
    app.set_busy(true);
    App::status_info(&app.ui.status_lt,
        &format_str(format_args!("link…")));

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        let store = match RecordStore::open(&db_raw, ST_BOOK_TAGS, "readwrite") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_lt,
                    &format_str(format_args!("open: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };
        let o = js_sys::Object::new();
        let _ = js_sys::Reflect::set(&o, &"book_isbn".into(), &JsValue::from_str(&book));
        let _ = js_sys::Reflect::set(&o, &"tag_id".into(), &JsValue::from_str(&tag));
        match idb_records::put(&store, &o.into()).await {
            Ok(_) => {
                App::status_ok(&app.ui.status_lt, "связь создана");
                app.push_recent("+ link".to_string());
                reload_all(&app).await;
            }
            Err(e) => App::status_err(&app.ui.status_lt,
                &format_str(format_args!("put: {}", short_error(&e)))),
        }
        app.set_busy(false);
    });
}

// ============================================================
//                  CASCADE DELETE
// ============================================================

async fn delete_author_cascade(app: Rc<App>, author_id: String) {
    if *app.busy.borrow() || author_id.is_empty() { return; }
    app.set_busy(true);
    App::status_info(&app.ui.status_tree_a,
        &format_str(format_args!("каскадное удаление автора…")));

    let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
    let t0 = now_ms();

    let books_store = match RecordStore::open(&db_raw, ST_BOOKS, "readonly") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    let all_books = idb_records::get_all(&books_store).await.unwrap_or_default();
    let his_books: Vec<String> = all_books.iter().filter_map(|b| {
        let aid = js_sys::Reflect::get(b, &"author_id".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if aid == author_id {
            js_sys::Reflect::get(b, &"isbn".into()).ok().and_then(|v| v.as_string())
        } else { None }
    }).collect();

    let links_store = match RecordStore::open(&db_raw, ST_BOOK_TAGS, "readwrite") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    let all_links = idb_records::get_all(&links_store).await.unwrap_or_default();
    let mut removed_links = 0u32;
    for l in &all_links {
        let isbn = js_sys::Reflect::get(l, &"book_isbn".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if his_books.contains(&isbn) {
            let tid = js_sys::Reflect::get(l, &"tag_id".into()).ok()
                .and_then(|v| v.as_string()).unwrap_or_default();
            let key_arr = js_sys::Array::new();
            key_arr.push(&JsValue::from_str(&isbn));
            key_arr.push(&JsValue::from_str(&tid));
            let _ = idb_records::delete(&links_store, &key_arr.into()).await;
            removed_links += 1;
        }
    }

    let books_rw = match RecordStore::open(&db_raw, ST_BOOKS, "readwrite") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    for isbn in &his_books {
        let _ = idb_records::delete(&books_rw, &JsValue::from_str(isbn)).await;
    }

    let authors_rw = match RecordStore::open(&db_raw, ST_AUTHORS, "readwrite") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    let _ = idb_records::delete(&authors_rw, &JsValue::from_str(&author_id)).await;

    let dt = now_ms() - t0;
    App::status_ok(&app.ui.status_tree_a,
        &format_str(format_args!(
            "автор удалён: {} книг, {removed_links} связей за {}",
            his_books.len(), format_time(dt),
        )));
    app.push_recent(format_str(format_args!("del-author {author_id}")));
    reload_all(&app).await;
    app.set_busy(false);
}

async fn delete_book_cascade(app: Rc<App>, isbn: String) {
    if *app.busy.borrow() || isbn.is_empty() { return; }
    app.set_busy(true);
    App::status_info(&app.ui.status_tree_b,
        &format_str(format_args!("каскадное удаление книги…")));

    let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
    let t0 = now_ms();

    let links_store = match RecordStore::open(&db_raw, ST_BOOK_TAGS, "readwrite") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    let all_links = idb_records::get_all(&links_store).await.unwrap_or_default();
    let mut removed = 0u32;
    for l in &all_links {
        let b = js_sys::Reflect::get(l, &"book_isbn".into()).ok()
            .and_then(|v| v.as_string()).unwrap_or_default();
        if b == isbn {
            let tid = js_sys::Reflect::get(l, &"tag_id".into()).ok()
                .and_then(|v| v.as_string()).unwrap_or_default();
            let key_arr = js_sys::Array::new();
            key_arr.push(&JsValue::from_str(&isbn));
            key_arr.push(&JsValue::from_str(&tid));
            let _ = idb_records::delete(&links_store, &key_arr.into()).await;
            removed += 1;
        }
    }

    let books_rw = match RecordStore::open(&db_raw, ST_BOOKS, "readwrite") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    let _ = idb_records::delete(&books_rw, &JsValue::from_str(&isbn)).await;

    let dt = now_ms() - t0;
    App::status_ok(&app.ui.status_tree_b,
        &format_str(format_args!(
            "book удалена: {removed} связей за {}", format_time(dt),
        )));
    app.push_recent(format_str(format_args!("del-book")));
    reload_all(&app).await;
    app.set_busy(false);
}

async fn unlink(app: Rc<App>, book: String, tag: String) {
    if *app.busy.borrow() || book.is_empty() || tag.is_empty() { return; }
    app.set_busy(true);
    App::status_info(&app.ui.status_tree_b,
        &format_str(format_args!("unlink…")));

    let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
    let store = match RecordStore::open(&db_raw, ST_BOOK_TAGS, "readwrite") {
        Ok(s) => s, Err(_) => { app.set_busy(false); return; }
    };
    let key_arr = js_sys::Array::new();
    key_arr.push(&JsValue::from_str(&book));
    key_arr.push(&JsValue::from_str(&tag));
    let _ = idb_records::delete(&store, &key_arr.into()).await;

    App::status_ok(&app.ui.status_tree_b, "связь удалена");
    app.push_recent("unlink".to_string());
    reload_all(&app).await;
    app.set_busy(false);
}

// ============================================================
//                  ITERATION
// ============================================================

#[derive(Clone, Copy)]
enum IterMode { Yield, GetAll }

fn iterate(app: Rc<App>, mode: IterMode) {
    if *app.busy.borrow() { return; }

    let direction = if app.ui.direction.value() == "prev" {
        Direction::Prev
    } else {
        Direction::Next
    };
    let limit: Option<u32> = if app.ui.limit_on.checked() {
        let raw = app.ui.limit_val.value();
        match raw.trim().parse::<u32>() {
            Ok(v) if v > 0 => Some(v),
            _ => {
                app.ui.limit_val.set_class_name("rc-key rc-invalid");
                App::status_err(&app.ui.status_iter, "limit должен быть положительным числом");
                return;
            }
        }
    } else {
        app.ui.limit_val.set_class_name("rc-key");
        None
    };

    let mode_label: &'static str = match mode {
        IterMode::Yield => "cursor + yield",
        IterMode::GetAll => "getAll",
    };

    app.set_busy(true);
    app.ui.scanned.set_text_content(Some("0"));
    app.ui.elapsed.set_text_content(Some("—"));
    app.ui.rate.set_text_content(Some("—"));
    set_progress(&app.ui.progress, &app.ui.progress_label, 0.0, "…".into(), "busy");
    App::status_info(&app.ui.status_iter,
        &format_str(format_args!("{mode_label} по books…")));

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        let t0 = now_ms();
        let store = match RecordStore::open(&db_raw, ST_BOOKS, "readonly") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_iter,
                    &format_str(format_args!("open: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };

        match mode {
            IterMode::GetAll => {
                let res = idb_records::get_all(&store).await;
                let dt = now_ms() - t0;
                match res {
                    Ok(v) => {
                        let n = v.len() as u32;
                        set_progress(&app.ui.progress, &app.ui.progress_label,
                            n as f64, format!("{n}"), "ok");
                        app.ui.scanned.set_text_content(Some(&format_str(format_args!("{n}"))));
                        app.ui.elapsed.set_text_content(Some(&format_time(dt)));
                        app.ui.rate.set_text_content(Some(&format_rate(n, dt)));
                        App::status_ok(&app.ui.status_iter,
                            &format_str(format_args!(
                                "getAll: {n} книг за {} · {}",
                                format_time(dt), format_rate(n, dt),
                            )));
                        app.push_recent(format_str(format_args!("getAll → {n}")));
                    }
                    Err(e) => App::status_err(&app.ui.status_iter,
                        &format_str(format_args!("getAll: {}", short_error(&e)))),
                }
            }
            IterMode::Yield => {
                let total = idb_records::count(&store).await.unwrap_or(1).max(1);
                let prog_el = app.ui.progress.clone();
                let label_el = app.ui.progress_label.clone();
                let scanned_el = app.ui.scanned.clone();
                let res = idb_records::cursor_visit(
                    &store, None, direction, limit,
                    move |scanned| {
                        let pct = (scanned as f64 / total as f64 * 100.0).min(100.0);
                        set_progress_el(&prog_el, &label_el, pct,
                            format_str(format_args!("{scanned} / {total}")), "busy");
                        scanned_el.set_text_content(Some(
                            &format_str(format_args!("{scanned}"))));
                    },
                ).await;
                let dt = now_ms() - t0;
                match res {
                    Ok(r) => {
                        app.ui.elapsed.set_text_content(Some(&format_time(dt)));
                        app.ui.rate.set_text_content(Some(&format_rate(r.scanned, dt)));
                        App::status_ok(&app.ui.status_iter,
                            &format_str(format_args!(
                                "cursor+yield: {} книг за {} · {}",
                                r.scanned, format_time(dt), format_rate(r.scanned, dt),
                            )));
                        app.push_recent(format_str(format_args!("yield → {}", r.scanned)));
                        set_progress(&app.ui.progress, &app.ui.progress_label,
                            r.scanned as f64, format!("{}", r.scanned), "ok");
                    }
                    Err(e) => App::status_err(&app.ui.status_iter,
                        &format_str(format_args!("cursor: {}", short_error(&e)))),
                }
            }
        }
        app.set_busy(false);
    });
}

// ============================================================
//                  APPLY RANGE
// ============================================================

fn apply_range(app: Rc<App>) {
    if *app.busy.borrow() { return; }

    let kind = app.ui.range_kind.value();
    let lo = app.ui.range_lo.value().trim().to_string();
    let hi = app.ui.range_hi.value().trim().to_string();

    let range: Option<KeyRangeSpec> = match kind.as_str() {
        "lower" => Some(KeyRangeSpec::LowerBound {
            value: JsValue::from_str(&lo), open: false,
        }),
        "upper" => Some(KeyRangeSpec::UpperBound {
            value: JsValue::from_str(&hi), open: false,
        }),
        "bound" => Some(KeyRangeSpec::Bound {
            lower: JsValue::from_str(&lo),
            upper: JsValue::from_str(&hi),
            lower_open: false,
            upper_open: false,
        }),
        _ => None,
    };

    app.set_busy(true);
    App::status_info(&app.ui.status_range,
        &format_str(format_args!("курсор по диапазону {kind}…")));
    set_progress(&app.ui.progress, &app.ui.progress_label, 0.0, "…".into(), "busy");

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        let store = match RecordStore::open(&db_raw, ST_BOOKS, "readonly") {
            Ok(s) => s, Err(e) => {
                App::status_err(&app.ui.status_range,
                    &format_str(format_args!("open: {}", short_error(&e))));
                app.set_busy(false); return;
            }
        };
        let total = idb_records::count(&store).await.unwrap_or(1).max(1);
        let prog_el = app.ui.progress.clone();
        let label_el = app.ui.progress_label.clone();
        let res = idb_records::cursor_visit(
            &store, range, Direction::Next, None,
            move |scanned| {
                let pct = (scanned as f64 / total as f64 * 100.0).min(100.0);
                set_progress_el(&prog_el, &label_el, pct,
                    format_str(format_args!("{scanned} / {total}")), "busy");
            },
        ).await;

        match res {
            Ok(r) => {
                app.ui.range_count.set_text_content(
                    Some(&format_str(format_args!("{}", r.scanned))));
                let first = r.keys.first().map(fmt_key).unwrap_or_else(|| "—".into());
                let last  = r.keys.last().map(fmt_key).unwrap_or_else(|| "—".into());
                app.ui.range_first.set_text_content(Some(&first));
                app.ui.range_last.set_text_content(Some(&last));

                let mut html = String::with_capacity(1024);
                let take = r.keys.len().min(MAX_PREVIEW_ROWS);
                for i in 0..take {
                    let k = fmt_key(&r.keys[i]);
                    let v = title_of(&r.values[i]);
                    html.push_str("<div class=\"rc-kv-row\"><span class=\"k\">");
                    html.push_str(&esc(&k));
                    html.push_str("</span><span class=\"v\">");
                    html.push_str(&esc(&v));
                    html.push_str("</span></div>");
                }
                if r.keys.len() > take {
                    html.push_str("<div class=\"rc-kv-row\"><span class=\"v\">");
                    html.push_str(&format_str(format_args!(
                        "… ещё {} (показаны первые {take})",
                        r.keys.len() - take,
                    )));
                    html.push_str("</span></div>");
                }
                if r.keys.is_empty() {
                    html = "<div class=\"rc-preview-empty\">В диапазоне нет книг</div>".to_string();
                }
                app.ui.preview.set_inner_html(&html);

                App::status_ok(&app.ui.status_range,
                    &format_str(format_args!(
                        "прочитано {} за {} · {}",
                        r.scanned, format_time(r.elapsed_ms), format_rate(r.scanned, r.elapsed_ms),
                    )));
                app.push_recent(format_str(format_args!("range → {}", r.scanned)));
                set_progress(&app.ui.progress, &app.ui.progress_label,
                    r.scanned as f64, format!("{}", r.scanned), "ok");
            }
            Err(e) => App::status_err(&app.ui.status_range,
                &format_str(format_args!("cursor: {}", short_error(&e)))),
        }
        app.set_busy(false);
    });
}

// ============================================================
//                  MIGRATION
// ============================================================

fn run_migration(app: Rc<App>, is_set: bool) {
    if *app.busy.borrow() { return; }

    let field = app.ui.mig_field.value().trim().to_string();
    if field.is_empty() {
        App::status_err(&app.ui.status_mig, "введите имя поля");
        return;
    }
    if field == "isbn" {
        App::status_err(&app.ui.status_mig, "isbn — ключ, менять нельзя");
        return;
    }

    let action: &'static str = if is_set { "set" } else { "remove" };
    let value_js: JsValue = if is_set {
        let raw = app.ui.mig_value.value().trim().to_string();
        if let Ok(n) = raw.parse::<f64>() { JsValue::from_f64(n) }
        else if raw == "true" { JsValue::TRUE }
        else if raw == "false" { JsValue::FALSE }
        else { JsValue::from_str(&raw) }
    } else { JsValue::UNDEFINED };

    app.set_busy(true);
    App::status_info(&app.ui.status_mig,
        &format_str(format_args!("{action} «{field}» в books…")));
    set_progress(&app.ui.mig_progress, &app.ui.mig_progress_label, 0.0, "…".into(), "busy");

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        let t0 = now_ms();

        let total = match RecordStore::open(&db_raw, ST_BOOKS, "readonly") {
            Ok(s) => idb_records::count(&s).await.unwrap_or(0),
            Err(_) => 0,
        };
        if total == 0 {
            App::status_err(&app.ui.status_mig, "books пуст — нечего мигрировать");
            app.set_busy(false); return;
        }

        let prog_el = app.ui.mig_progress.clone();
        let label_el = app.ui.mig_progress_label.clone();
        let field_c = field.clone();
        let value_c = value_js.clone();

        let res = cursor_update_all(
            &db_raw,
            ST_BOOKS,
            move |old| {
                if is_set { Some(modify_object(old, &field_c, Some(&value_c))) }
                else { Some(modify_object(old, &field_c, None)) }
            },
            move |done| {
                let pct = if total == 0 { 0.0 } else {
                    (done as f64 / total as f64 * 100.0).min(100.0)
                };
                set_progress_el(&prog_el, &label_el, pct,
                    format_str(format_args!("{done} / {total}")), "busy");
            },
            &field,
            is_set,
        ).await;

        let dt = now_ms() - t0;
        match res {
            Ok((updated, skipped)) => {
                app.ui.mig_updated.set_text_content(
                    Some(&format_str(format_args!("{updated}"))));
                app.ui.mig_skipped.set_text_content(
                    Some(&format_str(format_args!("{skipped}"))));
                app.ui.mig_elapsed.set_text_content(Some(&format_time(dt)));
                App::status_ok(&app.ui.status_mig,
                    &format_str(format_args!(
                        "{action} «{field}»: +{updated} / ~{skipped} за {}",
                        format_time(dt),
                    )));
                app.push_recent(format_str(format_args!("mig {action} {field} +{updated}")));
                set_progress(&app.ui.mig_progress, &app.ui.mig_progress_label,
                    total as f64, format!("{total} / {total}"), "ok");
                mark_stale_after_data_change(&app);
                reload_all(&app).await;
            }
            Err(e) => App::status_err(&app.ui.status_mig,
                &format_str(format_args!("migrate: {}", short_error(&e)))),
        }
        app.set_busy(false);
    });
}

fn mark_stale_after_data_change(app: &App) {
    App::status_stale(&app.ui.status_iter,
        "данные изменились — перезапустите итерацию");
    App::status_stale(&app.ui.status_range,
        "данные изменились — перезапустите диапазон");
    app.ui.scanned.set_text_content(Some("—"));
    app.ui.elapsed.set_text_content(Some("—"));
    app.ui.rate.set_text_content(Some("—"));
    app.ui.range_count.set_text_content(Some("—"));
    app.ui.range_first.set_text_content(Some("—"));
    app.ui.range_last.set_text_content(Some("—"));
    app.ui.preview.set_inner_html(
        "<div class=\"rc-preview-empty\">Данные изменились — нажмите «Применить»</div>");
    set_progress(&app.ui.progress, &app.ui.progress_label, 0.0, "—".into(), "ok");
}

fn modify_object(old: &JsValue, field: &str, new_value: Option<&JsValue>) -> JsValue {
    let new_obj = js_sys::Object::new();
    if let Some(obj) = old.dyn_ref::<js_sys::Object>() {
        let keys = js_sys::Object::keys(obj);
        for i in 0..keys.length() {
            let key = keys.get(i);
            let val = js_sys::Reflect::get(old, &key).unwrap_or(JsValue::UNDEFINED);
            let _ = js_sys::Reflect::set(&new_obj, &key, &val);
        }
    }
    match new_value {
        Some(v) => { let _ = js_sys::Reflect::set(&new_obj, &JsValue::from_str(field), v); }
        None => { let _ = js_sys::Reflect::delete_property(&new_obj, &JsValue::from_str(field)); }
    }
    new_obj.into()
}

async fn cursor_update_all<F, P>(
    db: &JsValue,
    store_name: &str,
    transform: F,
    mut on_progress: P,
    target_field: &str,
    check_skip: bool,
) -> Result<(u32, u32), JsValue>
where
    F: FnMut(&JsValue) -> Option<JsValue> + 'static,
    P: FnMut(u32) + 'static,
{
    let tx_fn: js_sys::Function =
        js_sys::Reflect::get(db, &JsValue::from_str("transaction"))
            .map_err(|e| JsValue::from_str(&format!("Reflect transaction: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("transaction не функция"))?;
    let names = js_sys::Array::new();
    names.push(&JsValue::from_str(store_name));
    let tx = tx_fn.call2(db, &names, &JsValue::from_str("readwrite"))
        .map_err(|e| JsValue::from_str(&format!("transaction: {e:?}")))?;
    let os_fn: js_sys::Function =
        js_sys::Reflect::get(&tx, &JsValue::from_str("objectStore"))
            .map_err(|e| JsValue::from_str(&format!("Reflect objectStore: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("objectStore не функция"))?;
    let os = os_fn.call1(&tx, &JsValue::from_str(store_name))
        .map_err(|e| JsValue::from_str(&format!("objectStore: {e:?}")))?;

    let complete = tx_complete_promise(&tx);

    struct State { updated: u32, skipped: u32, done: bool, error: Option<JsValue> }
    let state = Rc::new(RefCell::new(State {
        updated: 0, skipped: 0, done: false, error: None,
    }));

    let open_fn: js_sys::Function =
        js_sys::Reflect::get(&os, &JsValue::from_str("openCursor"))
            .map_err(|e| JsValue::from_str(&format!("Reflect openCursor: {e:?}")))?
            .dyn_into().map_err(|_| JsValue::from_str("openCursor не функция"))?;
    let request = open_fn.call0(&os)
        .map_err(|e| JsValue::from_str(&format!("openCursor: {e:?}")))?;

    let transform_cell: Rc<RefCell<F>> = Rc::new(RefCell::new(transform));
    let field_rc: Rc<RefCell<String>> = Rc::new(RefCell::new(target_field.to_string()));

    {
        let state_cb = state.clone();
        let transform_cb = transform_cell.clone();
        let field_cb = field_rc.clone();
        let on_success = Closure::<dyn FnMut(web_sys::Event)>::new(
            move |ev: web_sys::Event| {
                if state_cb.borrow().done { return; }
                let target = match ev.target() { Some(t) => t, None => return };
                let cursor = js_sys::Reflect::get(&target, &"result".into())
                    .unwrap_or(JsValue::NULL);
                if cursor.is_null() || cursor.is_undefined() {
                    state_cb.borrow_mut().done = true;
                    return;
                }
                let value = js_sys::Reflect::get(&cursor, &"value".into())
                    .unwrap_or(JsValue::NULL);
                let new_value = { let mut f = transform_cb.borrow_mut(); f(&value) };
                if let Some(nv) = new_value {
                    let changed = !check_skip || {
                        let field = field_cb.borrow();
                        field_value_changed(&value, &nv, &field)
                    };
                    if changed {
                        if let Ok(upd) = js_sys::Reflect::get(&cursor, &"update".into())
                            .and_then(|f| f.dyn_into::<js_sys::Function>())
                        {
                            let _ = upd.call1(&cursor, &nv);
                        }
                        state_cb.borrow_mut().updated += 1;
                    } else {
                        state_cb.borrow_mut().skipped += 1;
                    }
                }
                if let Ok(cont) = js_sys::Reflect::get(&cursor, &"continue".into())
                    .and_then(|f| f.dyn_into::<js_sys::Function>())
                {
                    let _ = cont.call0(&cursor);
                }
            });
        let _ = js_sys::Reflect::set(&request, &"onsuccess".into(),
            on_success.as_ref().unchecked_ref());
        on_success.forget();
    }

    {
        let state_cb = state.clone();
        let on_error = Closure::<dyn FnMut(web_sys::Event)>::new(
            move |ev: web_sys::Event| {
                let target: JsValue = match ev.target() {
                    Some(t) => t.unchecked_into(), None => JsValue::NULL,
                };
                let err = js_sys::Reflect::get(&target, &"error".into())
                    .unwrap_or(JsValue::UNDEFINED);
                state_cb.borrow_mut().error = Some(err);
                state_cb.borrow_mut().done = true;
            });
        let _ = js_sys::Reflect::set(&request, &"onerror".into(),
            on_error.as_ref().unchecked_ref());
        on_error.forget();
    }

    let mut last_yield = now_ms();
    loop {
        if state.borrow().done { break; }
        gloo_timers::future::TimeoutFuture::new(0).await;
        let now = now_ms();
        if now - last_yield >= 16.0 {
            on_progress(state.borrow().updated + state.borrow().skipped);
            last_yield = now;
        }
    }
    on_progress(state.borrow().updated + state.borrow().skipped);

    if let Some(err) = state.borrow().error.clone() { return Err(err); }
    JsFuture::from(complete).await.ok();
    let (u, s) = { let st = state.borrow(); (st.updated, st.skipped) };
    Ok((u, s))
}

fn field_value_changed(old: &JsValue, new_obj: &JsValue, field: &str) -> bool {
    let old_v = js_sys::Reflect::get(old, &JsValue::from_str(field))
        .unwrap_or(JsValue::UNDEFINED);
    let new_v = js_sys::Reflect::get(new_obj, &JsValue::from_str(field))
        .unwrap_or(JsValue::UNDEFINED);
    return stringify_js(&old_v) != stringify_js(&new_v);
}

fn stringify_js(v: &JsValue) -> String {
    if v.is_undefined() { return "<u>".into(); }
    if v.is_null() { return "<n>".into(); }
    if let Some(n) = v.as_f64() { return format_str(format_args!("n:{n}")); }
    if let Some(b) = v.as_bool() { return format_str(format_args!("b:{b}")); }
    if let Some(s) = v.as_string() { return format_str(format_args!("s:{s}")); }
    return js_sys::JSON::stringify(v).ok()
        .and_then(|s| s.as_string()).unwrap_or_default();
}

fn tx_complete_promise(tx: &JsValue) -> js_sys::Promise {
    let tx_set = tx.clone();
    js_sys::Promise::new(&mut move |resolve, _reject| {
        let resolve_c = resolve.clone();
        let on_complete = Closure::once_into_js(move || {
            let _ = resolve_c.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&tx_set, &"oncomplete".into(), &on_complete);
        let resolve_a = resolve.clone();
        let on_abort = Closure::once_into_js(move || {
            let _ = resolve_a.call0(&JsValue::UNDEFINED);
        });
        let _ = js_sys::Reflect::set(&tx_set, &"onabort".into(), &on_abort);
    })
}

// ============================================================
//                  SEED N (real library) / CLEAR
// ============================================================

struct SeedBook {
    isbn: String,
    title: String,
    author_id: String,
    year: u32,
    genre: String,
}

/// Разбирает DSL: автор + его книги.
fn parse_seed(data: &str) -> (Vec<(String, String)>, Vec<SeedBook>) {
    let mut authors: Vec<(String, String)> = Vec::new();
    let mut books: Vec<SeedBook> = Vec::new();
    let mut current_author: Option<String> = None;
    let mut book_idx: u32 = 0;

    for raw in data.lines() {
        if raw.is_empty() || raw.starts_with('#') { continue; }

        if let Some(rest) = raw.strip_prefix('\t') {
            let Some(aid) = current_author.clone() else { continue };
            let parts: Vec<&str> = rest.split('|').collect();
            if parts.len() < 3 { continue; }
            let title = parts[0].trim().to_string();
            let year: u32 = parts[1].trim().parse().unwrap_or(0);
            let genre = parts[2].trim().to_string();
            let isbn = format_str(format_args!("978-{book_idx:06}"));
            book_idx += 1;
            books.push(SeedBook { isbn, title, author_id: aid, year, genre });
        } else {
            let parts: Vec<&str> = raw.split('|').collect();
            if parts.len() < 2 { continue; }
            let id = parts[0].trim().to_string();
            let name = parts[1].trim().to_string();
            current_author = Some(id.clone());
            authors.push((id, name));
        }
    }

    (authors, books)
}

fn push_link(links: &mut Vec<JsValue>, isbn: &str, tag_id: &str) {
    let o = js_sys::Object::new();
    let _ = js_sys::Reflect::set(&o, &"book_isbn".into(), &JsValue::from_str(isbn));
    let _ = js_sys::Reflect::set(&o, &"tag_id".into(), &JsValue::from_str(tag_id));
    links.push(o.into());
}

fn run_seed(app: Rc<App>) {
    if *app.busy.borrow() { return; }
    let target: u32 = app.ui.seed_size.value().parse().unwrap_or(100);

    app.set_busy(true);
    App::status_info(&app.ui.status_seed,
        &format_str(format_args!("seed: загружаем библиотеку…")));
    set_progress(&app.ui.seed_progress, &app.ui.seed_progress_label,
        0.0, "…".into(), "busy");

    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        gloo_timers::future::TimeoutFuture::new(0).await;
        let t0 = now_ms();

        for name in [ST_BOOK_TAGS, ST_BOOKS, ST_AUTHORS, ST_TAGS] {
            if let Ok(s) = RecordStore::open(&db_raw, name, "readwrite") {
                let _ = idb_records::clear(&s).await;
            }
        }

        let (authors_data, books_data) = parse_seed(SEED_DATA);

        let take = (target as usize).min(books_data.len());
        let books_subset: Vec<&SeedBook> = books_data.iter().take(take).collect();

        let mut used_authors: Vec<String> = Vec::new();
        for b in &books_subset {
            if !used_authors.contains(&b.author_id) {
                used_authors.push(b.author_id.clone());
            }
        }

        let authors: Vec<JsValue> = authors_data.iter()
            .filter(|(id, _)| used_authors.contains(id))
            .map(|(id, name)| {
                let o = js_sys::Object::new();
                let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(id));
                let _ = js_sys::Reflect::set(&o, &"name".into(), &JsValue::from_str(name));
                o.into()
            })
            .collect();
        if let Ok(s) = RecordStore::open(&db_raw, ST_AUTHORS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &authors).await;
        }

        let tags: Vec<JsValue> = SEED_GENRES.iter().map(|(id, label)| {
            let o = js_sys::Object::new();
            let _ = js_sys::Reflect::set(&o, &"id".into(), &JsValue::from_str(id));
            let _ = js_sys::Reflect::set(&o, &"label".into(), &JsValue::from_str(label));
            o.into()
        }).collect();
        if let Ok(s) = RecordStore::open(&db_raw, ST_TAGS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &tags).await;
        }

        let books: Vec<JsValue> = books_subset.iter().map(|b| {
            let o = js_sys::Object::new();
            let _ = js_sys::Reflect::set(&o, &"isbn".into(), &JsValue::from_str(&b.isbn));
            let _ = js_sys::Reflect::set(&o, &"title".into(), &JsValue::from_str(&b.title));
            let _ = js_sys::Reflect::set(&o, &"author_id".into(),
                &JsValue::from_str(&b.author_id));
            let _ = js_sys::Reflect::set(&o, &"year".into(),
                &JsValue::from_f64(b.year as f64));
            o.into()
        }).collect();

        set_progress_el(&app.ui.seed_progress, &app.ui.seed_progress_label,
            (take as f64) * 0.5, format!("{take} книг"), "busy");
        if let Ok(s) = RecordStore::open(&db_raw, ST_BOOKS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &books).await;
        }

        let mut links: Vec<JsValue> = Vec::with_capacity(take * 4);
        for b in &books_subset {
            push_link(&mut links, &b.isbn, &b.genre);
            push_link(&mut links, &b.isbn, "russian");
            let era = match b.year {
                y if y < 1900 => "19c",
                y if y < 2000 => "20c",
                _ => "21c",
            };
            push_link(&mut links, &b.isbn, era);
            let theme = match b.year {
                y if y < 1917 => "classic",
                y if y < 1991 => "soviet",
                _ => "modern",
            };
            push_link(&mut links, &b.isbn, theme);
        }
        if let Ok(s) = RecordStore::open(&db_raw, ST_BOOK_TAGS, "readwrite") {
            let _ = idb_records::seed_bulk(&s, &links).await;
        }

        let dt = now_ms() - t0;
        set_progress(&app.ui.seed_progress, &app.ui.seed_progress_label,
            take as f64, format!("{take}"), "ok");
        app.ui.stat_seed_time.set_text_content(Some(&format_time(dt)));
        App::status_ok(&app.ui.status_seed,
            &format_str(format_args!(
                "seed: {} авторов, {take} книг, {} тегов, {} связей за {}",
                authors.len(), tags.len(), links.len(), format_time(dt),
            )));
        app.push_recent(format_str(format_args!("seed {take}")));
        app.set_last(&format_str(format_args!("seed {take}")));
        mark_stale_after_data_change(&app);
        reload_all(&app).await;
        app.set_busy(false);
    });
}

fn run_clear(app: Rc<App>) {
    if *app.busy.borrow() { return; }
    app.set_busy(true);
    App::status_info(&app.ui.status_seed, "очистка всех store…");
    spawn_local(async move {
        let Some(db_raw) = app.db_raw() else { app.set_busy(false); return; };
        for name in [ST_BOOK_TAGS, ST_BOOKS, ST_AUTHORS, ST_TAGS] {
            if let Ok(s) = RecordStore::open(&db_raw, name, "readwrite") {
                let _ = idb_records::clear(&s).await;
            }
        }
        set_progress(&app.ui.seed_progress, &app.ui.seed_progress_label, 0.0, "0".into(), "ok");
        app.ui.stat_seed_time.set_text_content(Some("—"));
        App::status_ok(&app.ui.status_seed, "все store очищены");
        app.push_recent("clear-all".to_string());
        app.set_last("clear");
        mark_stale_after_data_change(&app);
        reload_all(&app).await;
        app.set_busy(false);
    });
}

// ============================================================
//                  HELPERS
// ============================================================

fn set_progress(el: &Element, label: &Element, value: f64, text: String, state: &str) {
    set_progress_el(el, label, value, text, state);
}

fn set_progress_el(el: &Element, label: &Element, value: f64, text: String, state: &str) {
    if let Some(p) = el.dyn_ref::<web_sys::HtmlProgressElement>() {
        p.set_value(value);
    }
    el.set_class_name(&format_str(format_args!("rc-progress-{state}")));
    label.set_text_content(Some(&text));
}

fn fmt_key(k: &JsValue) -> String {
    if let Some(n) = k.as_f64() { return format_str(format_args!("{n}")); }
    if let Some(s) = k.as_string() { return format_str(format_args!("{s}")); }
    if js_sys::Array::is_array(k) {
        let arr = js_sys::Array::from(k);
        let mut parts: Vec<String> = Vec::new();
        for i in 0..arr.length() {
            parts.push(fmt_key(&arr.get(i)));
        }
        return format_str(format_args!("[{}]", parts.join(", ")));
    }
    return format_str(format_args!("{k:?}"));
}

fn title_of(v: &JsValue) -> String {
    return js_sys::Reflect::get(v, &"title".into()).ok()
        .and_then(|x| x.as_string())
        .unwrap_or_else(|| format_str(format_args!("{v:?}")));
}

fn format_time(ms: f64) -> String {
    if ms < 0.001 { return "0 µs".to_string(); }
    if ms < 1.0 { return format_str(format_args!("{:.0} µs", ms * 1000.0)); }
    if ms < 1000.0 { return format_str(format_args!("{ms:.1} ms")); }
    return format_str(format_args!("{:.2} s", ms / 1000.0));
}

fn format_rate(n: u32, ms: f64) -> String {
    if ms <= 0.0 || n == 0 { return "—".to_string(); }
    let rate = (n as f64) * 1000.0 / ms;
    if rate >= 1_000_000.0 { return format_str(format_args!("{:.2} M/сек", rate / 1_000_000.0)); }
    if rate >= 1_000.0 { return format_str(format_args!("{:.1} K/сек", rate / 1_000.0)); }
    return format_str(format_args!("{rate:.0} /сек"));
}

fn now_ms() -> f64 {
    web_sys::window().and_then(|w| w.performance())
        .map(|p| p.now()).unwrap_or(0.0)
}

fn error_name(e: &JsValue) -> String {
    if let Some(dx) = e.dyn_ref::<web_sys::DomException>() {
        return dx.name();
    }
    if let Some(s) = e.as_string() { return s; }
    return format!("{e:?}");
}

fn short_error(e: &JsValue) -> String {
    let full = error_name(e);
    let line = full.lines().next().unwrap_or("").trim();
    return line.chars().take(120).collect();
}

fn get_el(id: &str) -> Result<Element, JsValue> {
    let document: Document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    document.get_element_by_id(id)
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

fn esc(s: &str) -> String {
    return s
        .replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;");
}
```


## 24.5.16. Итоги

- `IDBObjectStore` — keyed-коллекция структурированных объектов.
- `add` — вставка, падает с `ConstraintError` при дубликате. `put` — upsert.
- Ключ: из `keyPath`, от `autoIncrement`, или передан вторым аргументом.
- Типы ключей: number, string, date, binary, array. Порядок — structured key algorithm.
- `getAll` — массив одним запросом; `openCursor` — поток.
- `IDBKeyRange` — фильтр на стороне движка: `lowerBound`, `upperBound`, `bound`, `only`.
- Составной ключ — массив `["a", "b"]`, задаётся строкой `"a,b"` в `keyPath`.
- `cursor.update()` — массовое изменение без `get`+`put`.
- Длительная итерация должна yield'ить через `TimeoutFuture::new(0)` — иначе UI замирает.
- Bulk-вставка: все `put` синхронно в одной транзакции, `await tx.oncomplete` в конце.
- Одна операция = одна транзакция. `RecordStore` — binding, не транзакция.
- Подписка на `tx.oncomplete` — до первого запроса. Резолвится и на `onabort`.
- Связи 1:N — через `author_id` + индекс. M:N — через промежуточный store с составным ключом.
- Каскадное удаление — вручную, в одной транзакции на несколько store.

**Мостик к 24.6.** Мы умеем писать, читать, итерировать и связывать сущности. Дальше — **транзакции**: как объединять произвольные операции над несколькими store'ами в атомарные блоки, как работать с курсором в readwrite, как проектировать каскадные миграции данных.