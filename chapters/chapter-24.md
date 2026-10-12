# Глава 24. IndexedDB

## 24.0. Что вы получите в этой главе

К концу главы у вас будет работающее приложение **«Каталог домашней библиотеки»**: авторы, книги, обложки. Два object store в одной базе, два индекса, транзакции на несколько store, каскадное удаление, хранение бинарных данных без base64. Демо в `app.rs` показывает всё это в UI, тесты покрывают каждый публичный вызов.

Отдельно разберём, **чего демо не показывает** — синергии WebAssembly и IndexedDB. Она раскрывается в отдельной главе и в упражнении в конце этой.

## 24.1. Почему `localStorage` недостаточно

`localStorage` хранит только строки. Чтобы положить туда картинку, её кодируют в base64: объём растёт на 33%, лимит в 5 МБ исчерпывается после двух-трёх фотографий. Хуже того, `localStorage` синхронный — запись на 4 МБ блокирует главный поток на десятки миллисекунд, а во время блокировки UI не перерисовывается.

IndexedDB решает обе проблемы:

| | `localStorage` | IndexedDB |
|---|---|---|
| Тип данных | только `string` | любой structured clone (объекты, массивы, `Blob`, `ArrayBuffer`, `Date`) |
| Бинарные данные | base64 (+33%) | нативно, `Blob`/`Uint8Array` |
| API | синхронный | асинхронный |
| Лимит | ~5 МБ | 10–60% свободного места на диске |
| Транзакции | нет | ACID, на несколько store |
| Индексы | нет | да, произвольные поля |

Для «каталога с обложками» выбор очевиден: `Blob` в IndexedDB vs base64-строка в `localStorage`.

## 24.2. `IdbFactory`

`IdbFactory` — точка входа, доступна в браузере как `window.indexedDB`. В `idb` crate обёрнута в `Factory::new()`:

```rust
let factory = Factory::new()?;
let mut request = factory.open("hello-wasm-library", Some(1))?;
```

Второй аргумент — **версия схемы**. При росте версии вызывается обработчик `on_upgrade_needed` — единственное место, где разрешено менять схему (создавать object store и индексы). Ронять существующие данные не обязательно: обработчик получает `old_version` и может мигрировать постепенно.

## 24.3. `IdbDatabase`

`IdbDatabase` — открытое соединение. Живёт всё время работы приложения, из него создаются транзакции. Одна БД привязана к одному origin: `https://example.com` и `http://localhost:5173` видят разные базы.

## 24.4. `IdbObjectStore`

`IdbObjectStore` — контейнер записей. Ключ задаётся либо явно в `key_path` (поле объекта), либо генерируется (`auto_increment`). В нашем проекте два store с явными ключами:

```rust
// authors: ключ — поле id
let mut p = ObjectStoreParams::new();
p.key_path(Some(KeyPath::new_single("id")));
db.create_object_store("authors", p)?;

// books: ключ — поле isbn
let mut p = ObjectStoreParams::new();
p.key_path(Some(KeyPath::new_single("isbn")));
db.create_object_store("books", p)?;
```

Операции: `put` (upsert), `add` (insert, ошибка при дубликате), `get`, `get_all`, `delete`, `count`, `open_cursor`, `index`.

## 24.5. `IdbTransaction`

Транзакция — атомарная единица работы. Создаётся на список store и режим:

```rust
let tx = db.transaction(&["authors", "books"], TransactionMode::ReadWrite)?;
```

Правило: **одна логическая операция = одна транзакция**. Удаление автора вместе с его книгами — одна `ReadWrite` транзакция на два store. Если делать по транзакции на книгу, при падении посередине каталог останется в рассогласованном состоянии: автор уже удалён, книги — нет.

Все изменения внутри транзакции видны друг другу. `tx.commit()` завершает явно.

## 24.6. `IdbRequest`

Каждая операция возвращает `IdbRequest`. В `idb` он реализует `Future`, поэтому `await` даёт результат:

```rust
let value: Option<JsValue> = store.get(key)?.await?;       // get
let values: Vec<JsValue> = store.get_all(None, None)?.await?; // get_all
```

`get_all(None, None)` читает всё — удобно для небольших store. Для крупных используйте курсор.

## 24.7. `IdbCursor`

Курсор итерирует записи без загрузки всего store в память. Порядок задаёт `CursorDirection::Next` или `Prev`. Курсор можно открыть и по индексу — тогда порядок задаётся индексом. В нашем проекте курсор не используется: `get_all` быстрее и проще, а объёмы каталога невелики. Курсор становится обязательным при миллионах записей или при потоковой обработке (чтение → трансформация → запись в тот же проход).

## 24.8. Индексы

Индекс — отсортированный «вид» на store по произвольному полю. Создаётся в `on_upgrade_needed`:

```rust
store.create_index("by_author", KeyPath::new_single("author_id"), None)?;
store.create_index("by_year",   KeyPath::new_single("year"),      None)?;
```

`unique(true)` запрещает дубликаты ключа. По индексу работают `get`, `get_all`, `open_cursor`:

```rust
let idx = store.index("by_author")?;
let query = Query::Key(JsValue::from_str(author_id));
let books: Vec<JsValue> = idx.get_all(Some(query), None)?.await?;
```

Это позволяет получить книги одного автора за время порядка O(log n + k) вместо полного сканирования store.

## 24.9. Транзакции на практике: каскадное удаление

Каскад — классический пример того, зачем нужны транзакции на несколько store:

```rust
let tx = write_tx(&db, &["authors", "books"])?;
let authors = tx.object_store("authors")?;
let books   = tx.object_store("books")?;
let idx     = books.index("by_author")?;

// 1) найти все книги автора по индексу
let q = Query::Key(JsValue::from_str(author_id));
let list: Vec<JsValue> = idx.get_all(Some(q), None)?.await?;

// 2) удалить каждую
for v in list {
    let b: Book = serde_wasm_bindgen::from_value(v)?;
    books.delete(JsValue::from_str(&b.isbn))?;
}

// 3) удалить автора
authors.delete(JsValue::from_str(author_id))?;

// 4) зафиксировать всё разом
tx.commit()?;
```

Если на шаге 2 упадёт (например, кончится квота) — откатится и шаг 1, и шаг 3. Каталог останется консистентным.

Функция идемпотентна: повторный вызов возвращает 0 удалённых книг и не ошибку. Это правило проекта: **«отменить / удалить» должно работать одинаково при первом и повторном вызове**. Пользователь мог дважды кликнуть, или запрос повторился после сбоя сети — оба варианта успех.

## 24.10. Хранение binary data

IndexedDB хранит `Blob`, `ArrayBuffer`, `Uint8Array` нативно. В Rust-структуре байты удобно держать как `Vec<u8>` — `serde_wasm_bindgen` превращает его в `Uint8Array`:

```rust
#[derive(Serialize, Deserialize)]
pub struct Book {
    pub isbn: String,
    pub title: String,
    pub author_id: String,
    pub year: u32,
    pub cover_mime: String,
    pub cover_bytes: Vec<u8>,
}
```

MIME храним отдельным полем, чтобы восстанавливать `Blob` с правильным типом при чтении. Обратное преобразование — через `Uint8Array::new(&array_buffer).to_vec()`.

Ключевое преимущество перед `localStorage`: обложка на 300 КБ занимает в БД ровно 300 КБ, а не 400 КБ после base64. На каталоге из 1000 книг это разница между 300 МБ и 400 МБ — и квота браузера позволяет первое, но не всегда второе.

## 24.11. Rust application state

Rust-структуры `Author` и `Book` — единственный источник правды. На границе с IndexedDB они сериализуются через `serde_wasm_bindgen::to_value` и обратно через `from_value`. Это даёт типовую безопасность на уровне Rust:

```rust
let value = serde_wasm_bindgen::to_value(a)?;   // Author → JsValue
store.put(&value, None)?.await?;

// обратно:
let a: Author = serde_wasm_bindgen::from_value(v)?;  // JsValue → Author
```

Если в БД лежит запись в формате прошлой версии схемы, `from_value` вернёт ошибку — её видно в UI, а не молчаливое `undefined`-поле.

Состояние демо (`Rc<RefCell<Vec<String>>>` с созданными ObjectURL) держим отдельно — это не данные БД, а UI-кэш. Он очищается при каждой перерисовке.

## 24.12. Практический проект: локальная база

**Каталог домашней библиотеки.**

- Два store: `authors` (`{ id, name }`) и `books` (`{ isbn, title, author_id, year, cover_mime, cover_bytes }`).
- Два индекса на `books`: `by_author` и `by_year`.
- CRUD авторов и книг.
- Каскадное удаление автора с его книгами в одной транзакции.
- Бинарные данные: обложки книг, `Vec<u8>` ↔ `Blob`.
- Демо: список авторов, под каждым — его книги (через `by_author`), поиск по году (через `by_year`), миниатюры обложек.

### Структура файлов

```
src/
  idb_core.rs      — открытие БД, upgrade-схема, константы
  idb_library.rs   — модель (Author, Book) и все операции
  app.rs           — демо только этой главы
tests/
  idb_core_test.rs     — host-тесты (сериализация, константы)
  idb_library_test.rs  — wasm-тесты (реальные операции с IDB)
```

`idb_core` содержит только инфраструктуру: открыть БД, объявить имена store и индексов, дать helper `write_tx`. Всё предметное — в `idb_library`. `app.rs` не знает про `idb` напрямую, только про `idb_library`.

**idb_core.rs**
```
//! IndexedDB: открытие БД, upgrade-схема, общие константы.
//! Автономный модуль — не зависит от других модулей проекта.

use idb::{
    Database, DatabaseEvent, Factory, KeyPath, ObjectStoreParams, TransactionMode,
};
use wasm_bindgen::prelude::*;

pub const DB_NAME: &str = "hello-wasm-library";
pub const DB_VERSION: u32 = 1;

pub const STORE_AUTHORS: &str = "authors";
pub const STORE_BOOKS: &str = "books";

pub const IDX_BOOKS_BY_AUTHOR: &str = "by_author";
pub const IDX_BOOKS_BY_YEAR: &str = "by_year";

/// Открывает БД `hello-wasm-library` v1.
/// При первом открытии создаёт stores `authors`, `books`
/// и индексы `by_author`, `by_year` на `books`.
pub async fn open() -> Result<Database, JsValue> {
    let factory = Factory::new()
        .map_err(|e| JsValue::from_str(&format!("Factory::new: {e:?}")))?;

    let mut request = factory
        .open(DB_NAME, Some(DB_VERSION))
        .map_err(|e| JsValue::from_str(&format!("open: {e:?}")))?;

    // `DatabaseEvent` — это трейт (в idb 0.6.5), а не структура.
    // Аннотацию типа для параметра замыкания писать нельзя —
    // тип выводится. Импорт трейта нужен, чтобы в scope был
    // метод `event.database()`.
    request.on_upgrade_needed(|event| {
        let db: Database = match event.database() {
            Ok(d) => d,
            Err(_) => return,
        };

        // store `authors` — ключ `id`
        let mut authors_params = ObjectStoreParams::new();
        authors_params.key_path(Some(KeyPath::new_single("id")));
        let _ = db.create_object_store(STORE_AUTHORS, authors_params);

        // store `books` — ключ `isbn`, два индекса
        let mut books_params = ObjectStoreParams::new();
        books_params.key_path(Some(KeyPath::new_single("isbn")));
        if let Ok(store) = db.create_object_store(STORE_BOOKS, books_params) {
            let _ = store.create_index(
                IDX_BOOKS_BY_AUTHOR,
                KeyPath::new_single("author_id"),
                None,
            );
            let _ = store.create_index(
                IDX_BOOKS_BY_YEAR,
                KeyPath::new_single("year"),
                None,
            );
        }
    });

    request
        .await
        .map_err(|e| JsValue::from_str(&format!("await open: {e:?}")))
}

/// ReadWrite-транзакция на произвольный набор stores.
///
/// В idb 0.6.5 `idb::Transaction` — без generic-параметров
/// времени жизни.
pub fn write_tx(
    db: &Database,
    stores: &[&str],
) -> Result<idb::Transaction, JsValue> {
    db.transaction(stores, TransactionMode::ReadWrite)
        .map_err(|e| JsValue::from_str(&format!("tx: {e:?}")))
}
```


### Ключевые функции `idb_library`

```rust
// Авторы
pub async fn put_author(a: &Author) -> Result<(), JsValue>;
pub async fn list_authors() -> Result<Vec<Author>, JsValue>;

// Книги
pub async fn put_book(b: &Book) -> Result<(), JsValue>;
pub async fn list_books() -> Result<Vec<Book>, JsValue>;

// Индексы
pub async fn books_by_author(author_id: &str) -> Result<Vec<Book>, JsValue>;
pub async fn books_by_year(year: u32) -> Result<Vec<Book>, JsValue>;

// Удаление
pub async fn remove_book(isbn: &str) -> Result<(), JsValue>;
pub async fn remove_author_cascade(author_id: &str) -> Result<u32, JsValue>;

// Бинарные данные
pub async fn book_with_cover(...) -> Result<Book, JsValue>;
pub fn cover_blob(b: &Book) -> Result<Blob, JsValue>;
```

```
//! Wasm-тесты практического проекта Главы 24 (IndexedDB).
//! Компилируются только под wasm32 — на host файл пуст.
//!
//! Тесты соответствуют публичному API `idb_library`:
//!   put_author / list_authors
//!   put_book   / list_books
//!   books_by_author (индекс by_author)
//!   books_by_year   (индекс by_year)
//!   remove_book / remove_author_cascade (идемпотентно)
//!   book_with_cover / cover_blob (binary data)

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;
use web_sys::Blob;

use hello_wasm02::idb_library::{
    book_with_cover, books_by_author, books_by_year, cover_blob, list_authors,
    list_books, put_author, put_book, remove_author_cascade, remove_book,
    Author, Book,
};

wasm_bindgen_test_configure!(run_in_browser);

/// Уникальный суффикс, чтобы тесты не сталкивались в одной и той же БД
/// при последовательных запусках.
fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64)
}

// ---------------- authors ----------------

#[wasm_bindgen_test]
async fn put_and_list_authors() {
    let id = format!("auth-{}", uid());
    let a = Author::new(&id, "Test Author");
    put_author(&a).await.expect("put_author");

    let list = list_authors().await.expect("list_authors");
    let found = list.iter().find(|x| x.id == id).expect("author present");
    assert_eq!(found.name, "Test Author");

    let _ = remove_author_cascade(&id).await;
}

#[wasm_bindgen_test]
async fn put_author_overwrites_by_id() {
    let id = format!("auth-{}", uid());
    put_author(&Author::new(&id, "First")).await.unwrap();
    put_author(&Author::new(&id, "Second")).await.unwrap();

    let list = list_authors().await.unwrap();
    let found = list.iter().find(|x| x.id == id).expect("author present");
    assert_eq!(found.name, "Second");

    let _ = remove_author_cascade(&id).await;
}

// ---------------- books ----------------

#[wasm_bindgen_test]
async fn put_and_list_books() {
    let isbn = format!("isbn-{}", uid());
    let b = Book::new(&isbn, "Some Book", "any-author", 2020);
    put_book(&b).await.expect("put_book");

    let books = list_books().await.expect("list_books");
    let found = books.iter().find(|x| x.isbn == isbn).expect("book present");
    assert_eq!(found.title, "Some Book");
    assert_eq!(found.year, 2020);

    let _ = remove_book(&isbn).await;
}

// ---------------- indexes ----------------

#[wasm_bindgen_test]
async fn index_books_by_author() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Index Author");
    put_author(&author).await.expect("put_author");

    let b1 = Book::new(&format!("isbn-a-{suffix}"), "Книга 1", &author.id, 2000);
    let b2 = Book::new(&format!("isbn-b-{suffix}"), "Книга 2", &author.id, 2001);
    put_book(&b1).await.expect("put b1");
    put_book(&b2).await.expect("put b2");

    let by_author = books_by_author(&author.id).await.expect("by_author");
    assert_eq!(by_author.len(), 2, "индекс by_author должен вернуть 2 книги");
    assert!(by_author.iter().any(|x| x.isbn == b1.isbn));
    assert!(by_author.iter().any(|x| x.isbn == b2.isbn));

    let _ = remove_author_cascade(&author.id).await;
}

#[wasm_bindgen_test]
async fn index_books_by_year() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Year Author");
    put_author(&author).await.unwrap();

    // Год 2000+, разный для каждого запуска — тест не сталкивается сам с собой.
    let year: u32 = 2000 + (suffix.parse::<u32>().unwrap_or(0) % 1000);
    let isbn = format!("isbn-year-{suffix}");
    put_book(&Book::new(&isbn, "Y", &author.id, year)).await.unwrap();

    let by_year = books_by_year(year).await.expect("by_year");
    assert!(by_year.iter().any(|x| x.isbn == isbn));

    let _ = remove_author_cascade(&author.id).await;
}

#[wasm_bindgen_test]
async fn books_by_author_empty_for_unknown() {
    let unknown = format!("no-author-{}", uid());
    let result = books_by_author(&unknown).await.expect("by_author");
    assert!(result.is_empty(), "индекс по неизвестному автору должен вернуть пусто");
}

// ---------------- cascade ----------------

#[wasm_bindgen_test]
async fn cascade_removes_books() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Cascade");
    put_author(&author).await.unwrap();
    put_book(&Book::new(&format!("c-1-{suffix}"), "C1", &author.id, 1990)).await.unwrap();
    put_book(&Book::new(&format!("c-2-{suffix}"), "C2", &author.id, 1991)).await.unwrap();

    let removed = remove_author_cascade(&author.id).await.expect("cascade");
    assert_eq!(removed, 2);

    let left = books_by_author(&author.id).await.unwrap();
    assert!(left.is_empty());
    let authors = list_authors().await.unwrap();
    assert!(!authors.iter().any(|a| a.id == author.id));
}

#[wasm_bindgen_test]
async fn cascade_is_idempotent() {
    let r1 = remove_author_cascade("no-such-author").await.expect("first");
    assert_eq!(r1, 0);
    let r2 = remove_author_cascade("no-such-author").await.expect("second");
    assert_eq!(r2, 0);
}

#[wasm_bindgen_test]
async fn remove_book_idempotent() {
    remove_book("no-such-isbn").await.expect("first");
    remove_book("no-such-isbn").await.expect("second");
}

// ---------------- binary data (cover) ----------------

#[wasm_bindgen_test]
async fn cover_blob_roundtrip() {
    let bytes: [u8; 4] = [0x89, 0x50, 0x4E, 0x47]; // PNG-магия
    let blob = make_blob(&bytes, "image/png").expect("make_blob");

    let book = book_with_cover("cover-isbn", "Cover", "auth-1", 2020, &blob)
        .await
        .expect("book_with_cover");
    assert_eq!(book.cover_mime, "image/png");
    assert_eq!(book.cover_bytes, bytes.to_vec());

    let back = cover_blob(&book).expect("cover_blob");
    assert_eq!(back.type_(), "image/png");
    let back_bytes = blob_bytes(&back).await;
    assert_eq!(back_bytes, bytes.to_vec());
}

#[wasm_bindgen_test]
async fn cover_bytes_survive_indexeddb() {
    let suffix = uid();
    let author = Author::new(&format!("cover-auth-{suffix}"), "Cover Author");
    put_author(&author).await.unwrap();

    let bytes: Vec<u8> = (0..32u8).map(|i| i.wrapping_mul(7)).collect();
    let blob = make_blob(&bytes, "image/jpeg").unwrap();

    let book = book_with_cover(
        &format!("cover-isbn-{suffix}"),
        "Book with cover",
        &author.id,
        2021,
        &blob,
    )
    .await
    .unwrap();
    put_book(&book).await.unwrap();

    let loaded = list_books().await.unwrap();
    let found = loaded
        .iter()
        .find(|b| b.isbn == book.isbn)
        .expect("book loaded from IDB");
    assert_eq!(found.cover_mime, "image/jpeg");
    assert_eq!(found.cover_bytes, bytes, "binary data должны сохраниться без искажений");

    let _ = remove_author_cascade(&author.id).await;
}

// ---------------- helpers ----------------

fn make_blob(bytes: &[u8], mime: &str) -> Result<Blob, wasm_bindgen::JsValue> {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let opts = web_sys::BlobPropertyBag::new();
    opts.set_type(mime);
    Blob::new_with_u8_array_sequence_and_options(&parts, &opts)
}

async fn blob_bytes(blob: &Blob) -> Vec<u8> {
    let buf = wasm_bindgen_futures::JsFuture::from(blob.array_buffer())
        .await
        .expect("array_buffer");
    js_sys::Uint8Array::new(&buf).to_vec()
}
```

Каждая функция открывает БД, делает одну транзакцию, коммитит. Это простой и надёжный паттерн для каталога из сотен записей.

### Что показывает демо

1. **Два store в одной БД.** Авторы и книги — отдельные сущности.
2. **Индекс `by_author`.** При отрисовке каталога книги группируются запросом `books_by_author(&a.id)`, а не фильтрацией массива в памяти.
3. **Индекс `by_year`.** Отдельная секция «Поиск по году».
4. **Бинарные данные.** Обложки хранятся как `Vec<u8>`, отображаются как `<img>` через `URL.createObjectURL(Blob)`.
5. **Каскадное удаление.** Кнопка «Удалить автора и книги» — одна транзакция.
6. **Идемпотентность.** Повторное удаление возвращает успех, а не 404.
7. **Автообновление.** После каждой операции список перерисовывается сам.
8. **Освобождение ресурсов.** ObjectURL для обложек освобождаются при перерисовке — иначе память вкладки течёт.

### Что демо намеренно не показывает

- **Синергия WASM + IndexedDB.** В нашем демо WASM — только мост между UI и IDB. Вся логика — это `serde_wasm_bindgen::to_value` → `store.put`. Тот же сценарий на TypeScript пишется за 20 минут с библиотекой `idb`.
- **Ускорение за счёт WASM.** Чтобы увидеть преимущество, нужно, чтобы WASM делал вычислительно тяжёлую работу с данными до записи в БД: ресайз изображений, хеширование, сжатие, индексацию для полнотекстового поиска.
- **Кэш «горячих» данных в памяти WASM.** В `idb_library` каждый вызов открывает БД заново. Для продакшена соединение держат в `OnceCell` или в состоянии приложения.

Это нормально: глава про IndexedDB, и демо показывает ровно IndexedDB. Синергия — тема главы про WASM-обработку данных, и там нужен свой сценарий.

### Упражнение со звёздочкой

Добавьте в `book_with_cover` обработку до записи в БД:

1. Ресайз обложки до 200×300 через crate `image` (WASM CPU).
2. SHA-256 от содержимого исходного файла (crate `sha2`).
3. Сохраняйте в БД уже уменьшенную версию.

Замерьте на 100 файлах по 2 МБ:
- время обработки через `canvas.toBlob` в JS vs через WASM;
- итоговый размер БД в DevTools → Application → IndexedDB.

На этих данных видно, **зачем** комбинировать WASM и IndexedDB: WASM выполняет CPU-работу быстро и не блокирует UI, IndexedDB хранит результат компактно и надёжно. По отдельности ни та, ни другая технология этого не даёт.

## 24.13. Тестирование

Две группы тестов:

**idb_core_test.rs**
```
//! Host-тесты модуля idb_core: константы схемы и сериализация моделей.
//! Никаких JsValue/IndexedDB здесь быть не должно — только чистая Rust-логика.
//! Реальные операции с IndexedDB — в tests/idb_library_test.rs (wasm-таргет).

use hello_wasm02::idb_core::{
    DB_NAME, DB_VERSION, IDX_BOOKS_BY_AUTHOR, IDX_BOOKS_BY_YEAR,
    STORE_AUTHORS, STORE_BOOKS,
};
use hello_wasm02::idb_library::{Author, Book};

#[test]
fn db_constants_are_stable() {
    assert_eq!(DB_NAME, "hello-wasm-library");
    assert_eq!(DB_VERSION, 1);
    assert_eq!(STORE_AUTHORS, "authors");
    assert_eq!(STORE_BOOKS, "books");
    assert_eq!(IDX_BOOKS_BY_AUTHOR, "by_author");
    assert_eq!(IDX_BOOKS_BY_YEAR, "by_year");
}

#[test]
fn author_json_roundtrip() {
    let a = Author::new("tolstoy", "Лев Толстой");
    let s = serde_json::to_string(&a).unwrap();
    let back: Author = serde_json::from_str(&s).unwrap();
    assert_eq!(a, back);
}

#[test]
fn book_json_roundtrip_with_cover() {
    let mut b = Book::new("978-5-04-118457-7", "Война и мир", "tolstoy", 1869);
    b.cover_mime = "image/png".into();
    b.cover_bytes = vec![0x89, 0x50, 0x4E, 0x47];
    let s = serde_json::to_string(&b).unwrap();
    let back: Book = serde_json::from_str(&s).unwrap();
    assert_eq!(b, back);
    assert_eq!(back.cover_bytes, vec![0x89, 0x50, 0x4E, 0x47]);
}
```

**Host-тесты** (`cargo test`) — сериализация моделей и стабильность констант схемы:

```rust
#[test]
fn book_json_roundtrip_with_cover() {
    let mut b = Book::new("isbn", "title", "author", 2020);
    b.cover_mime = "image/png".into();
    b.cover_bytes = vec![0x89, 0x50, 0x4E, 0x47];
    let s = serde_json::to_string(&b).unwrap();
    let back: Book = serde_json::from_str(&s).unwrap();
    assert_eq!(b, back);
}
```

**idb_library_test**
```
//! Wasm-тесты практического проекта Главы 24 (IndexedDB).
//! Компилируются только под wasm32 — на host файл пуст.
//!
//! Тесты соответствуют публичному API `idb_library`:
//!   put_author / list_authors
//!   put_book   / list_books
//!   books_by_author (индекс by_author)
//!   books_by_year   (индекс by_year)
//!   remove_book / remove_author_cascade (идемпотентно)
//!   book_with_cover / cover_blob (binary data)

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;
use web_sys::Blob;

use hello_wasm02::idb_library::{
    book_with_cover, books_by_author, books_by_year, cover_blob, list_authors,
    list_books, put_author, put_book, remove_author_cascade, remove_book,
    Author, Book,
};

wasm_bindgen_test_configure!(run_in_browser);

/// Уникальный суффикс, чтобы тесты не сталкивались в одной и той же БД
/// при последовательных запусках.
fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64)
}

// ---------------- authors ----------------

#[wasm_bindgen_test]
async fn put_and_list_authors() {
    let id = format!("auth-{}", uid());
    let a = Author::new(&id, "Test Author");
    put_author(&a).await.expect("put_author");

    let list = list_authors().await.expect("list_authors");
    let found = list.iter().find(|x| x.id == id).expect("author present");
    assert_eq!(found.name, "Test Author");

    let _ = remove_author_cascade(&id).await;
}

#[wasm_bindgen_test]
async fn put_author_overwrites_by_id() {
    let id = format!("auth-{}", uid());
    put_author(&Author::new(&id, "First")).await.unwrap();
    put_author(&Author::new(&id, "Second")).await.unwrap();

    let list = list_authors().await.unwrap();
    let found = list.iter().find(|x| x.id == id).expect("author present");
    assert_eq!(found.name, "Second");

    let _ = remove_author_cascade(&id).await;
}

// ---------------- books ----------------

#[wasm_bindgen_test]
async fn put_and_list_books() {
    let isbn = format!("isbn-{}", uid());
    let b = Book::new(&isbn, "Some Book", "any-author", 2020);
    put_book(&b).await.expect("put_book");

    let books = list_books().await.expect("list_books");
    let found = books.iter().find(|x| x.isbn == isbn).expect("book present");
    assert_eq!(found.title, "Some Book");
    assert_eq!(found.year, 2020);

    let _ = remove_book(&isbn).await;
}

// ---------------- indexes ----------------

#[wasm_bindgen_test]
async fn index_books_by_author() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Index Author");
    put_author(&author).await.expect("put_author");

    let b1 = Book::new(&format!("isbn-a-{suffix}"), "Книга 1", &author.id, 2000);
    let b2 = Book::new(&format!("isbn-b-{suffix}"), "Книга 2", &author.id, 2001);
    put_book(&b1).await.expect("put b1");
    put_book(&b2).await.expect("put b2");

    let by_author = books_by_author(&author.id).await.expect("by_author");
    assert_eq!(by_author.len(), 2, "индекс by_author должен вернуть 2 книги");
    assert!(by_author.iter().any(|x| x.isbn == b1.isbn));
    assert!(by_author.iter().any(|x| x.isbn == b2.isbn));

    let _ = remove_author_cascade(&author.id).await;
}

#[wasm_bindgen_test]
async fn index_books_by_year() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Year Author");
    put_author(&author).await.unwrap();

    // Год 2000+, разный для каждого запуска — тест не сталкивается сам с собой.
    let year: u32 = 2000 + (suffix.parse::<u32>().unwrap_or(0) % 1000);
    let isbn = format!("isbn-year-{suffix}");
    put_book(&Book::new(&isbn, "Y", &author.id, year)).await.unwrap();

    let by_year = books_by_year(year).await.expect("by_year");
    assert!(by_year.iter().any(|x| x.isbn == isbn));

    let _ = remove_author_cascade(&author.id).await;
}

#[wasm_bindgen_test]
async fn books_by_author_empty_for_unknown() {
    let unknown = format!("no-author-{}", uid());
    let result = books_by_author(&unknown).await.expect("by_author");
    assert!(result.is_empty(), "индекс по неизвестному автору должен вернуть пусто");
}

// ---------------- cascade ----------------

#[wasm_bindgen_test]
async fn cascade_removes_books() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Cascade");
    put_author(&author).await.unwrap();
    put_book(&Book::new(&format!("c-1-{suffix}"), "C1", &author.id, 1990)).await.unwrap();
    put_book(&Book::new(&format!("c-2-{suffix}"), "C2", &author.id, 1991)).await.unwrap();

    let removed = remove_author_cascade(&author.id).await.expect("cascade");
    assert_eq!(removed, 2);

    let left = books_by_author(&author.id).await.unwrap();
    assert!(left.is_empty());
    let authors = list_authors().await.unwrap();
    assert!(!authors.iter().any(|a| a.id == author.id));
}

#[wasm_bindgen_test]
async fn cascade_is_idempotent() {
    let r1 = remove_author_cascade("no-such-author").await.expect("first");
    assert_eq!(r1, 0);
    let r2 = remove_author_cascade("no-such-author").await.expect("second");
    assert_eq!(r2, 0);
}

#[wasm_bindgen_test]
async fn remove_book_idempotent() {
    remove_book("no-such-isbn").await.expect("first");
    remove_book("no-such-isbn").await.expect("second");
}

// ---------------- binary data (cover) ----------------

#[wasm_bindgen_test]
async fn cover_blob_roundtrip() {
    let bytes: [u8; 4] = [0x89, 0x50, 0x4E, 0x47]; // PNG-магия
    let blob = make_blob(&bytes, "image/png").expect("make_blob");

    let book = book_with_cover("cover-isbn", "Cover", "auth-1", 2020, &blob)
        .await
        .expect("book_with_cover");
    assert_eq!(book.cover_mime, "image/png");
    assert_eq!(book.cover_bytes, bytes.to_vec());

    let back = cover_blob(&book).expect("cover_blob");
    assert_eq!(back.type_(), "image/png");
    let back_bytes = blob_bytes(&back).await;
    assert_eq!(back_bytes, bytes.to_vec());
}

#[wasm_bindgen_test]
async fn cover_bytes_survive_indexeddb() {
    let suffix = uid();
    let author = Author::new(&format!("cover-auth-{suffix}"), "Cover Author");
    put_author(&author).await.unwrap();

    let bytes: Vec<u8> = (0..32u8).map(|i| i.wrapping_mul(7)).collect();
    let blob = make_blob(&bytes, "image/jpeg").unwrap();

    let book = book_with_cover(
        &format!("cover-isbn-{suffix}"),
        "Book with cover",
        &author.id,
        2021,
        &blob,
    )
    .await
    .unwrap();
    put_book(&book).await.unwrap();

    let loaded = list_books().await.unwrap();
    let found = loaded
        .iter()
        .find(|b| b.isbn == book.isbn)
        .expect("book loaded from IDB");
    assert_eq!(found.cover_mime, "image/jpeg");
    assert_eq!(found.cover_bytes, bytes, "binary data должны сохраниться без искажений");

    let _ = remove_author_cascade(&author.id).await;
}

// ---------------- helpers ----------------

fn make_blob(bytes: &[u8], mime: &str) -> Result<Blob, wasm_bindgen::JsValue> {
    let uint8 = js_sys::Uint8Array::from(bytes);
    let parts = js_sys::Array::new();
    parts.push(&uint8.buffer());
    let opts = web_sys::BlobPropertyBag::new();
    opts.set_type(mime);
    Blob::new_with_u8_array_sequence_and_options(&parts, &opts)
}

async fn blob_bytes(blob: &Blob) -> Vec<u8> {
    let buf = wasm_bindgen_futures::JsFuture::from(blob.array_buffer())
        .await
        .expect("array_buffer");
    js_sys::Uint8Array::new(&buf).to_vec()
}
```

**Wasm-тесты** (`wasm-pack test --headless --chrome idb_library_test`) — реальные операции с IndexedDB:

```rust
#![cfg(target_arch = "wasm32")]
use wasm_bindgen_test::*;
wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
async fn cascade_removes_books() {
    let suffix = uid();
    let author = Author::new(&format!("auth-{suffix}"), "Cascade");
    put_author(&author).await.unwrap();
    put_book(&Book::new(&format!("c1-{suffix}"), "C1", &author.id, 1990)).await.unwrap();
    put_book(&Book::new(&format!("c2-{suffix}"), "C2", &author.id, 1991)).await.unwrap();

    let removed = remove_author_cascade(&author.id).await.unwrap();
    assert_eq!(removed, 2);

    let left = books_by_author(&author.id).await.unwrap();
    assert!(left.is_empty());
}
```

Три обязательные строки в каждом wasm-тесте:

```rust
#![cfg(target_arch = "wasm32")]
use wasm_bindgen_test::*;
wasm_bindgen_test_configure!(run_in_browser);
```

Без первой — падает `cargo test` на host. Без второй — не резолвится атрибут. Без третьей — тест не запускается в браузере.

Уникальные id через `Date::now()` — чтобы тесты не пересекались в одной и той же БД при повторных запусках.

## 24.14. Итоги

- IndexedDB — асинхронное транзакционное хранилище структурированных и бинарных данных.
- Схема объявляется в `on_upgrade_needed`; версия БД — часть контракта.
- Транзакция — атомарная единица; одна логическая операция = одна транзакция, даже на несколько store.
- Индексы дают быстрый поиск по произвольному полю без полного сканирования.
- Бинарные данные хранятся нативно, без base64.
- Rust-модели сериализуются в `JsValue` через `serde_wasm_bindgen` — типовая безопасность на границе с БД.
- Каскадные операции должны быть идемпотентны: повторный вызов не ошибка.
- Объекты `Blob`-URL нужно освобождать при перерисовке, иначе память вкладки растёт.

### Демо-пример

```
//! Демо Главы 24: IndexedDB — каталог домашней библиотеки.
//!
//! Что показывает демо:
//!   • Два object store (`authors`, `books`) в одной БД.
//!   • Индекс `by_author` — книги группируются запросом books_by_author.
//!   • Индекс `by_year` — поиск «книг за год» через books_by_year.
//!   • Бинарные данные — обложки: Blob ↔ Uint8Array ↔ object URL.
//!   • Каскадное удаление автора вместе с его книгами в одной транзакции.
//!
//! Фокусный подход: только демо этой главы.

use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::spawn_local;
use web_sys::{HtmlElement, HtmlInputElement, Url};

use crate::format_str::format_str;
use crate::idb_library::{self, Author, Book};

const HTML: &str = r#"
<h1>Глава 24: IndexedDB</h1>
<p>Каталог домашней библиотеки: авторы, книги, обложки.</p>

<fieldset style="margin:8px 0">
  <legend>Автор</legend>
  <input id="a-id" placeholder="ID (tolstoy)">
  <input id="a-name" placeholder="Имя">
  <button id="add-author">Добавить</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Книга</legend>
  <input id="b-isbn" placeholder="ISBN">
  <input id="b-title" placeholder="Название">
  <input id="b-author" placeholder="ID автора">
  <input id="b-year" placeholder="Год" type="number" style="width:80px">
  <input id="b-cover" type="file" accept="image/*">
  <button id="add-book">Добавить</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Поиск (индекс by_year)</legend>
  <input id="s-year" placeholder="Год" type="number" style="width:80px">
  <button id="search-year">Найти</button>
  <button id="clear-search">×</button>
</fieldset>

<div id="status" style="color:#555;margin:8px 0">Загрузка…</div>
<div id="year-result"></div>
<div id="catalog"></div>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no document")?;

    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let status = document.get_element_by_id("status").ok_or("no status")?;
    let catalog = document.get_element_by_id("catalog").ok_or("no catalog")?;
    let year_result = document
        .get_element_by_id("year-result")
        .ok_or("no year-result")?;

    // Кэш созданных ObjectURL — чтобы освобождать при перерисовке.
    let urls: Rc<RefCell<Vec<String>>> = Rc::new(RefCell::new(Vec::new()));

    // Первичная отрисовка
    spawn_local({
        let status = status.clone();
        let catalog = catalog.clone();
        let urls = urls.clone();
        async move {
            set_status(&status, "БД открыта");
            render_catalog(&catalog, &urls).await;
        }
    });

    // Делегированный обработчик кликов по каталогу
    let catalog_el: HtmlElement = catalog.clone().dyn_into()?;
    {
        let status = status.clone();
        let catalog = catalog.clone();
        let urls = urls.clone();
        set_click(&catalog_el, move |e: web_sys::MouseEvent| {
            let target: web_sys::Element = match e.target().and_then(|t| t.dyn_into().ok()) {
                Some(t) => t,
                None => return,
            };
            let kind = target.get_attribute("data-kind").unwrap_or_default();
            let id = target.get_attribute("data-id").unwrap_or_default();
            if kind.is_empty() || id.is_empty() {
                return;
            }

            let status = status.clone();
            let catalog = catalog.clone();
            let urls = urls.clone();
            spawn_local(async move {
                match kind.as_str() {
                    "del-book" => {
                        if let Err(e) = idb_library::remove_book(&id).await {
                            set_status(&status, &format_str(format_args!("Ошибка: {e:?}")));
                            return;
                        }
                        set_status(&status, "Книга удалена");
                    }
                    "del-author" => match idb_library::remove_author_cascade(&id).await {
                        Ok(n) => set_status(
                            &status,
                            &format_str(format_args!("Автор и {n} книг(и) удалены")),
                        ),
                        Err(e) => {
                            set_status(&status, &format_str(format_args!("Ошибка: {e:?}")));
                            return;
                        }
                    },
                    _ => return,
                }
                render_catalog(&catalog, &urls).await;
            });
        });
    }

    // Кнопка «Добавить автора»
    let add_author: HtmlElement = document
        .get_element_by_id("add-author")
        .ok_or("no add-author")?
        .dyn_into()?;
    {
        let status = status.clone();
        let catalog = catalog.clone();
        let urls = urls.clone();
        set_click(&add_author, move |_| {
            let id = get_input("a-id");
            let name = get_input("a-name");
            if id.is_empty() || name.is_empty() {
                return;
            }
            let status = status.clone();
            let catalog = catalog.clone();
            let urls = urls.clone();
            spawn_local(async move {
                match idb_library::put_author(&Author::new(&id, &name)).await {
                    Ok(_) => set_status(&status, "Автор добавлен"),
                    Err(e) => {
                        set_status(&status, &format_str(format_args!("Ошибка: {e:?}")));
                        return;
                    }
                }
                render_catalog(&catalog, &urls).await;
            });
        });
    }

    // Кнопка «Добавить книгу»
    let add_book: HtmlElement = document
        .get_element_by_id("add-book")
        .ok_or("no add-book")?
        .dyn_into()?;
    {
        let status = status.clone();
        let catalog = catalog.clone();
        let urls = urls.clone();
        set_click(&add_book, move |_| {
            let isbn = get_input("b-isbn");
            let title = get_input("b-title");
            let author_id = get_input("b-author");
            let year: u32 = get_input("b-year").parse().unwrap_or(0);
            if isbn.is_empty() || title.is_empty() || author_id.is_empty() {
                return;
            }
            let cover = web_sys::window()
                .and_then(|w| w.document())
                .and_then(|d| d.get_element_by_id("b-cover"))
                .and_then(|e| e.dyn_into::<HtmlInputElement>().ok())
                .and_then(|i| i.files())
                .and_then(|f| f.get(0));

            let status = status.clone();
            let catalog = catalog.clone();
            let urls = urls.clone();
            spawn_local(async move {
                set_status(&status, "Проверка автора…");

                // Валидация: автор должен существовать.
                let authors = match idb_library::list_authors().await {
                    Ok(a) => a,
                    Err(e) => {
                        set_status(&status, &format_str(format_args!("Ошибка: {e:?}")));
                        return;
                    }
                };
                if !authors.iter().any(|a| a.id == author_id) {
                    set_status(
                        &status,
                        &format_str(format_args!("Нет автора с ID = {author_id}")),
                    );
                    return;
                }

                let book = match cover {
                    Some(blob) => {
                        match idb_library::book_with_cover(&isbn, &title, &author_id, year, &blob)
                            .await
                        {
                            Ok(b) => b,
                            Err(e) => {
                                set_status(
                                    &status,
                                    &format_str(format_args!("Ошибка чтения файла: {e:?}")),
                                );
                                return;
                            }
                        }
                    }
                    None => Book::new(&isbn, &title, &author_id, year),
                };
                match idb_library::put_book(&book).await {
                    Ok(_) => set_status(&status, "Книга добавлена"),
                    Err(e) => {
                        set_status(&status, &format_str(format_args!("Ошибка: {e:?}")));
                        return;
                    }
                }
                render_catalog(&catalog, &urls).await;
            });
        });
    }

    // Кнопка «Найти книги за год» (индекс by_year)
    let search_year: HtmlElement = document
        .get_element_by_id("search-year")
        .ok_or("no search-year")?
        .dyn_into()?;
    {
        let year_result = year_result.clone();
        set_click(&search_year, move |_| {
            let y: u32 = get_input("s-year").parse().unwrap_or(0);
            let year_result = year_result.clone();
            spawn_local(async move {
                render_year(&year_result, y).await;
            });
        });
    }

    // Кнопка «×» — сброс результата поиска
    let clear_search: HtmlElement = document
        .get_element_by_id("clear-search")
        .ok_or("no clear-search")?
        .dyn_into()?;
    {
        let year_result = year_result.clone();
        set_click(&clear_search, move |_| {
            year_result.set_inner_html("");
        });
    }

    Ok(())
}

// ---------------- helpers ----------------

/// Навешивает обработчик клика. `Closure::forget()` — осознанная плата
/// за отсутствие контейнера `Closure` в этом демо: обработчики живут до
/// конца страницы. Для динамически создаваемых кнопок нужен отдельный
/// трекер (см. главу про `callbacks`).
fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn get_input(id: &str) -> String {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlInputElement>().ok())
        .map(|i| i.value())
        .unwrap_or_default()
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

/// Перерисовка каталога: авторы + их книги (через индекс `by_author`).
async fn render_catalog(catalog: &web_sys::Element, urls: &Rc<RefCell<Vec<String>>>) {
    revoke_urls(urls);

    let authors = match idb_library::list_authors().await {
        Ok(a) => a,
        Err(e) => {
            catalog.set_inner_html(&format_str(format_args!(
                "<p style=\"color:#a00\">Ошибка чтения авторов: {e:?}</p>"
            )));
            return;
        }
    };
    let books = match idb_library::list_books().await {
        Ok(b) => b,
        Err(e) => {
            catalog.set_inner_html(&format_str(format_args!(
                "<p style=\"color:#a00\">Ошибка чтения книг: {e:?}</p>"
            )));
            return;
        }
    };

    if authors.is_empty() {
        catalog.set_inner_html("<p><em>Каталог пуст. Добавьте автора, затем книгу.</em></p>");
        return;
    }

    let mut html = String::with_capacity(1024);
    html.push_str(&format!(
        "<p><small>Авторов: {}, книг: {}</small></p>",
        authors.len(),
        books.len()
    ));

    for a in &authors {
        html.push_str("<div style=\"margin:12px 0;padding:8px;border:1px solid #ddd\">");
        html.push_str(&format!(
            "<strong>{}</strong> <small>(id: {})</small> \
             <button data-kind=\"del-author\" data-id=\"{}\">Удалить автора и книги</button>",
            esc(&a.name),
            esc(&a.id),
            esc(&a.id)
        ));

        // Явное использование индекса by_author.
        let my_books = match idb_library::books_by_author(&a.id).await {
            Ok(v) => v,
            Err(e) => {
                html.push_str(&format!(
                    "<p style=\"color:#a00\">Ошибка индекса: {e:?}</p>"
                ));
                html.push_str("</div>");
                continue;
            }
        };

        if my_books.is_empty() {
            html.push_str("<p><em>книг нет</em></p>");
        } else {
            html.push_str("<ul>");
            for b in &my_books {
                let img = cover_tag(b, urls);
                html.push_str(&format!(
                    "<li>{img}{} ({}) \
                     <button data-kind=\"del-book\" data-id=\"{}\">Удалить</button></li>",
                    esc(&b.title),
                    b.year,
                    esc(&b.isbn)
                ));
            }
            html.push_str("</ul>");
        }
        html.push_str("</div>");
    }

    catalog.set_inner_html(&html);
}

/// Перерисовка результата поиска по индексу `by_year`.
async fn render_year(year_result: &web_sys::Element, year: u32) {
    if year == 0 {
        year_result.set_inner_html("<p><em>Укажите год.</em></p>");
        return;
    }
    let books = match idb_library::books_by_year(year).await {
        Ok(b) => b,
        Err(e) => {
            year_result.set_inner_html(&format_str(format_args!(
                "<p style=\"color:#a00\">Ошибка: {e:?}</p>"
            )));
            return;
        }
    };
    if books.is_empty() {
        year_result.set_inner_html(&format_str(format_args!(
            "<p><em>За {year} год книг нет.</em></p>"
        )));
        return;
    }

    // Имена авторов — для читаемости результата.
    let authors = idb_library::list_authors().await.unwrap_or_default();
    let name_of = |id: &str| -> String {
        authors
            .iter()
            .find(|a| a.id == id)
            .map(|a| a.name.clone())
            .unwrap_or_else(|| format!("id:{id}"))
    };

    let mut html = String::with_capacity(256);
    html.push_str(&format!(
        "<p><strong>За {year} год — {} книг(и):</strong></p><ul>",
        books.len()
    ));
    for b in &books {
        html.push_str(&format!(
            "<li>{} — {}</li>",
            esc(&b.title),
            esc(&name_of(&b.author_id))
        ));
    }
    html.push_str("</ul>");
    year_result.set_inner_html(&html);
}

/// Создаёт `<img>` с object URL из Blob обложки (или пустую строку).
/// URL сохраняется в `urls`, чтобы его можно было освободить.
fn cover_tag(b: &Book, urls: &Rc<RefCell<Vec<String>>>) -> String {
    if b.cover_bytes.is_empty() {
        return String::new();
    }
    let blob = match idb_library::cover_blob(b) {
        Ok(bl) => bl,
        Err(_) => return String::new(),
    };
    let url = match Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(_) => return String::new(),
    };
    urls.borrow_mut().push(url.clone());
    format_str(format_args!(
        "<img src=\"{}\" alt=\"обложка\" \
         style=\"width:32px;height:48px;object-fit:cover;vertical-align:middle;margin-right:6px\">",
        esc(&url)
    ))
}

/// Освобождает ранее созданные ObjectURL.
fn revoke_urls(urls: &Rc<RefCell<Vec<String>>>) {
    let mut cache = urls.borrow_mut();
    for u in cache.drain(..) {
        let _ = Url::revoke_object_url(&u);
    }
}

fn esc(s: &str) -> String {
    s.replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;")
}
```

## Ссылки

- MDN, IndexedDB API: https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
- MDN, Using IndexedDB: https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB
- Документация `idb` crate: https://docs.rs/idb
- Best practices для IndexedDB (web.dev): https://web.dev/articles/indexeddb-best-practices
- Structured clone algorithm: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm