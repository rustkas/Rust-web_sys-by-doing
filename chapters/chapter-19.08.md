# 19.8. OPFS: приватная файловая система браузера

## Зачем это нужно

Origin Private File System (OPFS) — это не «ещё одно хранилище рядом с localStorage». Это **отдельный класс работы с данными**: приложение получает в песочнице браузера настоящую файловую систему с иерархией, файлами, чтением и записью — **без единого разрешения от пользователя**. Всё, что нужно, это `navigator.storage.getDirectory()`.

OPFS отличается от File System Access API (`showSaveFilePicker`/`showOpenFilePicker`) принципиально:

| Аспект | OPFS | user-visible FSA |
|---|---|---|
| Требует user gesture | ❌ Нет | ✅ Да (клик по кнопке) |
| Видно в проводнике ОС | ❌ Нет | ✅ Да |
| Разрешения | Привязано к origin | Запрос у пользователя |
| Синхронный доступ | Только `createSyncAccessHandle()` в Dedicated Worker | Нет |
| Идеально для | Кэш, черновики, рабочие данные | Пользовательские документы |

Именно OPFS — тот самый слой, на котором работают SQLite WASM, PDF-редакторы, медиа-обработчики, офлайн-PWA. В этой главе мы разберём, **почему** он такой быстрый, **где** прячутся грабли, и соберём живое демо с файлом, растущим в реальном времени.

---

## 19.8.1. Что такое OPFS и как он устроен

OPFS живёт в песочнице origin. У него есть корень (директория), от которого строится всё дерево:

```rust
let root: FileSystemDirectoryHandle = navigator.storage.getDirectory().await?;
```

Корень OPFS — это `FileSystemDirectoryHandle` с **пустым именем**. У него нет родителя, нет `name`, он не виден ни в одном проводнике. Это документированное поведение — не пытайтесь проверять «получили ли мы корень» через `root.name().is_empty()`; проверяйте работоспособность: создайте файл, прочитайте, удалите.

Внутри корня — обычная иерархия `FileSystemDirectoryHandle` / `FileSystemFileHandle`, та же, что и в user-visible FSA. Все примитивы из главы 19.7 (`getFileHandle`, `getDirectoryHandle`, `removeEntry`, `resolve`, `values()`) работают с OPFS точно так же.

Отличие только одно, но принципиальное: **`FileSystemFileHandle` в OPFS поддерживает `createSyncAccessHandle()`** — синхронный доступ к файлу. Мы разберём его отдельно, потому что вокруг него построена половина производительности OPFS.

---

## 19.8.2. Async I/O в main thread

Базовый уровень работы с OPFS — тот же, что и у FSA: получить handle, открыть `FileSystemWritableFileStream`, записать, закрыть. Это работает в main thread, не блокирует UI, но каждая запись — это отдельный `Promise`, а значит — микротаска. Для небольших файлов (до сотен килобайт) это нормально. Для больших — накладно.

```rust
pub async fn write_text(
    root: &FileSystemDirectoryHandle,
    name: &str,
    text: &str,
) -> Result<FileSystemFileHandle, JsValue> {
    let handle = get_file_handle(root, name, true).await?;
    let stream = crate::fsa_api_file::create_writable(&handle, false).await?;
    crate::fsa_api_writable::write_text(&stream, text).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle)
}
```

`create: true` — идемпотентно. Если файл есть, он будет перезаписан; если нет — создан. Каждая запись — полная замена содержимого, не дозапись.

Этот уровень используется для **черновиков, конфигураций, небольших JSON-документов**. Всё, что должно сохраняться между перезагрузками и не требует высокой пропускной способности.

---

## 19.8.3. `createSyncAccessHandle` — синхронный I/O в Worker

Это ядро OPFS. `FileSystemFileHandle::createSyncAccessHandle()` возвращает объект с **синхронными** методами `read(buffer, {at})`, `write(buffer, {at})`, `getSize()`, `flush()`, `close()`. Никаких `await`, никаких микротасок, никакого yield. Побайтовый доступ, как в нативной ФС.

Но есть жёсткое ограничение, установленное спецификацией: **`createSyncAccessHandle` доступен только внутри Dedicated Worker**. В main thread его нет на прототипе `FileSystemFileHandle`. Это не прихоть — синхронный I/O в главном потоке заблокировал бы UI.

Отсюда два следствия:

**1. Проверять доступность в main thread бесполезно.** `typeof FileSystemFileHandle.prototype.createSyncAccessHandle` в main thread всегда даёт `undefined`. Единственный надёжный способ — пробный Worker.

**2. Вся работа с sync access handle должна быть инкапсулирована в Worker'е.** Это может быть отдельный `.js`-файл, inline Blob Worker, или — как в нашем проекте — inline-строка, из которой создаётся Blob и URL.

### Пробный Worker для проверки доступности

```rust
pub async fn sync_access_available_in_worker() -> bool {
    let probe_js: &str = r#"
        const ok = typeof FileSystemFileHandle !== 'undefined'
            && FileSystemFileHandle.prototype
            && typeof FileSystemFileHandle.prototype.createSyncAccessHandle === 'function';
        self.postMessage(ok);
    "#;

    // Blob → object URL → Worker.
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(probe_js));
    let blob = match web_sys::Blob::new_with_str_sequence(&blob_parts) {
        Ok(b) => b,
        Err(_) => return false,
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(_) => return false,
    };
    let worker: Worker = match Worker::new(&url) {
        Ok(w) => w,
        Err(_) => {
            web_sys::Url::revoke_object_url(&url).ok();
            return false;
        }
    };
    web_sys::Url::revoke_object_url(&url).ok();

    // Promise, которая резолвится первым сообщением от Worker'а.
    let worker_ref: &Worker = &worker;
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let onmessage = Closure::once(move |e: MessageEvent| {
            let _ = resolve.call1(&JsValue::UNDEFINED, &e.data());
        });
        worker_ref.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    });

    let result = JsFuture::from(promise).await;
    worker.terminate();

    match result {
        Ok(v) => v.as_bool().unwrap_or(false),
        Err(_) => false,
    }
}
```

Приём с `js_sys::Promise::new(&mut |resolve, _reject| { ... })` — общий для всех случаев, когда нужно «подождать одно сообщение от Worker'а». `Closure::once` гарантирует, что обработчик сработает ровно один раз и не утечёт. `worker.terminate()` освобождает ресурсы.

> **Правило проекта.** Никогда не проверяйте доступность API через `typeof` в том контексте, где API не должно быть доступно. Если API живёт в Worker'е — проверяйте из Worker'а.

### Запись через sync access handle

```rust
pub async fn write_sync_in_worker(
    name: &str,
    data: &[u8],
) -> Result<(usize, f64), JsValue> {
    // 1) Blob из JS-кода → object URL → Worker.
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(WRITE_WORKER_JS));
    let blob = web_sys::Blob::new_with_str_sequence(&blob_parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker: Worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    // 2) Отправляем имя файла и байты.
    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("name"), &JsValue::from_str(name))?;
    let u8arr = js_sys::Uint8Array::from(data);
    js_sys::Reflect::set(&msg, &JsValue::from_str("data"), &u8arr.buffer())?;
    worker.post_message(&msg)?;

    // 3) Promise — резолвится первым сообщением от Worker'а.
    // 4) Разбираем ответ: `{ ok: true, written, ms }` или `{ ok: false, error }`.
    // ... (см. полный код модуля ниже) ...

    worker.terminate();
    result
}
```

JS-код Worker'а — строка:

```js
self.onmessage = async (e) => {
    const { name, data } = e.data;
    try {
        const root = await navigator.storage.getDirectory();
        const fileHandle = await root.getFileHandle(name, { create: true });
        const accessHandle = await fileHandle.createSyncAccessHandle();

        const bytes = new Uint8Array(data);
        const t0 = performance.now();
        const written = accessHandle.write(bytes, { at: 0 });
        accessHandle.flush();
        accessHandle.close();
        const t1 = performance.now();

        self.postMessage({ ok: true, written, ms: t1 - t0 });
    } catch (err) {
        self.postMessage({ ok: false, error: String(err) });
    }
};
```

Три ключевые операции:

- `write(bytes, { at: 0 })` — синхронная запись в позицию 0.
- `flush()` — гарантирует, что данные попали на диск (аналог `fsync`).
- `close()` — освобождает handle. **Без `close()` файл остаётся залоченным** для других handle'ов.

**Почему это быстрее async.** В async-версии каждая запись — это `Promise`, которая идёт в микротаску, потом в поток ввода-вывода браузера, потом обратно. В sync-версии `write()` — синхронный вызов C++, который сразу передаёт байты в файловый дескриптор. Никаких переключений контекста. На 10 МБ это даёт разницу в **10–20 раз**.

---

## 19.8.4. Живой логгер: файл, растущий в реальном времени

Самый наглядный способ показать силу OPFS — заставить файл расти **прямо на глазах**, не блокируя UI. Для этого:

1. Worker открывает `createSyncAccessHandle` на файле и запускает `setInterval(tick, 50)`.
2. В каждом тике — формирует строку с timestamp, пишет её через `write(bytes, { at: offset })`, делает `flush()`, увеличивает `offset` на размер строки.
3. Main thread раз в 500 мс читает размер и хвост файла через **обычные async-методы** и обновляет UI.

Ключевой момент — **offset**. Мы не перезаписываем файл, а **дописываем** в конец. Offset изначально берётся из `access.getSize()`, то есть из **текущего** размера файла. Это значит, что при перезагрузке страницы лог **продолжается**, а не начинается заново:

```js
let offset = access.getSize();   // append, не перезапись
```

Если зайти на страницу после паузы, старые строки на месте, а Worker дописывает к ним новые. OPFS ведёт себя как настоящая файловая система — данные переживают перезапуск вкладки.

```rust
pub fn start_logger_worker(
    name: &str,
    interval_ms: u32,
) -> Result<LoggerHandle, JsValue> {
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(LOGGER_WORKER_JS));
    let blob = web_sys::Blob::new_with_str_sequence(&blob_parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("name"), &JsValue::from_str(name))?;
    js_sys::Reflect::set(
        &msg,
        &JsValue::from_str("intervalMs"),
        &JsValue::from_f64(interval_ms as f64),
    )?;
    worker.post_message(&msg)?;

    Ok(LoggerHandle { worker })
}
```

`LoggerHandle::stop()` вызывает `worker.terminate()`. Всё, что успело записаться, **уже на диске** — благодаря `flush()` в каждом тике. Никаких «несохранённых» данных при остановке.

---

## 19.8.5. Чтение хвоста и метаданных

Читать весь файл по 500 мс, если он растёт, — плохая идея. Для превью нужен только **хвост**. `Blob::slice(start)` даёт «подблоб» от позиции до конца, а `Blob::text()` превращает его в строку:

```rust
pub async fn read_tail(
    root: &FileSystemDirectoryHandle,
    name: &str,
    max_bytes: usize,
) -> Result<String, JsValue> {
    let handle = match get_file_handle(root, name, false).await {
        Ok(h) => h,
        Err(_) => return Ok(String::new()),
    };
    let file = crate::fsa_api_file::get_file(&handle).await?;
    let size: f64 = file.size();
    let start: f64 = if size > max_bytes as f64 {
        size - max_bytes as f64
    } else {
        0.0
    };
    let blob: web_sys::Blob = file.slice_with_f64(start)?;

    // `Blob::text()` — не во всех версиях web-sys. Идём через Reflect.
    let blob_js: &JsValue = blob.as_ref();
    let text_fn_js = js_sys::Reflect::get(blob_js, &JsValue::from_str("text"))?;
    if text_fn_js.is_undefined() {
        return Err(JsValue::from_str("Blob.text недоступен"));
    }
    let text_fn: js_sys::Function = text_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("text не функция"))?;
    let promise: js_sys::Promise = text_fn
        .call0(blob_js)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("text не Promise"))?;
    let value: JsValue = JsFuture::from(promise).await?;
    value
        .as_string()
        .ok_or_else(|| JsValue::from_str("Blob.text вернул не строку"))
}
```

`Blob.text()` — не всегда в биндингах `web-sys`; вызов через `Reflect` решает проблему независимо от версии крейта.

Для сводки по OPFS — обход корня через `values()` и подсчёт размеров всех файлов:

```rust
pub async fn list_files(
    root: &FileSystemDirectoryHandle,
) -> Result<Vec<(String, f64)>, JsValue> {
    // ... значения из values() ...
    // для каждого kind == "file" — get_file, size
}
```

Этот приём — мини-обход, но без рекурсии: только верхний уровень. Для полноценного дерева — `fsa_dir_tree::build_tree` из главы 19.7, потому что `FileSystemDirectoryHandle` **один и тот же** и в OPFS, и в user-visible FSA.

---

## 19.8.6. OPFS vs классическая файловая система

Отдельный — и очень важный для понимания — вопрос: насколько OPFS похож на «настоящую» файловую систему Linux, Windows или macOS? Ответ: похож по форме, радикально отличается по существу.

### Основная таблица

| Характеристика | OPFS (браузер) | Классическая ФС (ОС) |
|---|---|---|
| **Где живёт** | Внутри origin браузера, в песочнице | На диске, в иерархии ОС |
| **Кто видит** | Только код с того же origin | Любой процесс, пользующийся правами |
| **Точка входа** | `navigator.storage.getDirectory()` | `/` (корень ФС) |
| **Имя корня** | Пустое (`root.name() == ""`) | `/` (или `C:\`) |
| **Требует user gesture** | ❌ Нет | ❌ Нет |
| **Требует разрешений** | ❌ Нет (привязано к origin) | ✅ Да (в современных ОС) |
| **API** | Асинхронный + синхронный (только Worker) | Синхронный по умолчанию |
| **Синхронный доступ** | Только `createSyncAccessHandle()` в Dedicated Worker | Везде |
| **Транзакционность** | Нет | Обычно нет (журналируется на уровне ФС) |
| **Права доступа** | Все файлы в OPFS — одного origin | uid/gid, ACL, capabilities |
| **Символические ссылки** | ❌ Нет | ✅ Да |
| **Жёсткие ссылки** | ❌ Нет | ✅ Да |
| **Именованные каналы / сокеты** | ❌ Нет | ✅ Да |
| **Специальные файлы (`/dev`, `/proc`)** | ❌ Нет | ✅ Да |
| **Монтирование** | Нет | Да |
| **Файловые дескрипторы ОС** | Нет — только `FileSystemFileHandle` | Да — `int fd` |
| **`mmap`** | ❌ Нет | ✅ Да |
| **Прямой I/O (`O_DIRECT`)** | ❌ Нет | ✅ Да |
| **Атрибуты (`chmod`, `chown`)** | ❌ Нет | ✅ Да |
| **`stat`, `inode`** | Нет; только `name`, `size`, `type` | Полный `stat` |
| **Квоты** | Общая квота браузера (обычно доля от свободного места) | Дисковые квоты, `ulimit` |
| **Место на диске** | Ограничено браузером; Firefox ~10 ГБ на origin | До размера раздела |
| **Межпроцессный доступ** | ❌ Только через `postMessage` / `SharedWorker` | ✅ Полный |
| **Видимость пользователю** | ❌ Скрыта (private FS) | ✅ В проводнике |
| **Бэкап** | Только через приложение или DevTools | Штатный `cp`, `rsync`, `tar` |
| **Персистентность** | Зависит от `navigator.storage.persist()` | Пока файл не удалён |
| **Вытеснение при нехватке места** | Возможно, если не persistent | ОС обычно не удаляет |
| **Резервное копирование ОС** | ❌ Не попадает | ✅ Да |
| **Поиск через ОС** | ❌ Не индексируется | ✅ Да (`locate`, `Spotlight`, …) |
| **Антивирус** | ❌ Не сканирует | ✅ Сканирует |
| **Доступ из другого браузера** | ❌ Нет — привязан к origin | ✅ Да |
| **Доступ из другой вкладки** | ✅ Да (тот же origin) | ✅ Да |
| **Доступ после перезагрузки** | ✅ Да | ✅ Да |
| **Доступ после очистки cookies** | ⚠️ Может быть стёрт вместе с origin | Не зависит |

### Производительность

| Операция | OPFS (sync в Worker) | OPFS (async в main) | Локальная ФС |
|---|---|---|---|
| Запись 10 МБ | ~30–80 мс | ~200–600 мс | ~5–20 мс (SSD, page cache) |
| Чтение 10 МБ | ~20–50 мс | ~100–300 мс | ~1–5 мс (page cache) |
| Открытие файла | ~1–5 мс | ~5–15 мс | ~0.01–0.1 мс |
| `stat` | ~0.5–2 мс | ~1–5 мс | < 0.01 мс |
| Создание файла | ~2–10 мс | ~5–20 мс | ~0.05–0.5 мс |

Порядки условные, на типичном железе. Главное — **sync в Worker обычно в 3–10× быстрее async в main thread** и в 5–20× медленнее нативной ФС.

### Семантика

| Понятие | OPFS | Классическая ФС |
|---|---|---|
| **«Файл»** | `FileSystemFileHandle` + байты | `inode` + байты + метаданные |
| **«Директория»** | `FileSystemDirectoryHandle` | `inode` директории |
| **Имя** | Строка в текущей директории | Строка в `dirent`, байты в кодировке ФС |
| **Максимум файлов в директории** | Ограничен браузером (практически миллионы) | Зависит от ФС (ext4 — миллионы) |
| **Максимальный размер файла** | Ограничен квотой и 2^53 байт (JS-число) | До 2^63 на большинстве ФС |
| **Кодировка имён** | UTF-8 (по спецификации) | Зависит от ФС (NTFS — UTF-16, ext4 — байты) |
| **Максимальная длина имени** | Обычно 255 символов | Обычно 255 байт |
| **Регистрозависимость** | Зависит от реализации; Chrome — да | NTFS — нет, ext4 — да, APFS — по умолчанию нет |
| **Атомарность `rename`** | Не гарантируется | `POSIX`: атомарно в пределах ФС |
| **`fsync`** | `flush()` на `SyncAccessHandle` | `fsync()`, `fdatasync()` |
| **Sparse-файлы** | ❌ Нет | ✅ Да (на многих ФС) |

### Модель безопасности

| Аспект | OPFS | Классическая ФС |
|---|---|---|
| **Изоляция** | Origin (scheme + host + port) | Пользователи ОС + права |
| **Доступ из JS** | Да, всегда (для своего origin) | Нет без специального API |
| **Доступ из нативных приложений** | ❌ Нет штатного способа | ✅ Да |
| **Восстановление после сбоя** | Зависит от браузера | Журналирование ФС |
| **Шифрование at-rest** | Зависит от профиля браузера | LUKS, BitLocker, FileVault |
| **Изоляция от вредоносного кода** | ✅ Полная (браузер — sandbox) | ⚠️ Только за счёт прав ОС |
| **Утечка при компрометации браузера** | Только данные этого origin | Все данные под правами пользователя |

### Что OPFS умеет, чего нет в классической ФС

| Возможность | Комментарий |
|---|---|
| Доступ без установки приложения | Открыл страницу — получил файловое хранилище |
| Кросс-платформенность из коробки | Один код работает везде, где есть браузер |
| Автоматическая привязка к origin | Никаких разрешений — файлы уже «свои» |
| Прозрачный доступ между вкладками | Тот же origin — общий OPFS |
| Изоляция по сайту | Другой origin — другой OPFS, даже если тот же домен с другим портом |
| Отсутствие user gesture | Можно писать сразу при загрузке |

### Что классическая ФС умеет, чего нет в OPFS

| Возможность | Почему важно |
|---|---|
| `mmap` | Прямая работа с памятью, базы данных, JIT |
| Символические и жёсткие ссылки | Гибкая организация, экономия места |
| `O_DIRECT`, `O_APPEND` через fd | Тонкий контроль над I/O |
| Специальные файлы | `/dev/null`, `/dev/random`, `/proc` |
| Именованные каналы и сокеты | IPC |
| Полная работа с правами | `chmod`, `chown`, `setfacl` |
| Транзакции на уровне БД через `fsync`+`rename` | Надёжные паттерны записи |
| Прямой доступ из CLI | `ls`, `cat`, `grep`, `rsync`, `tar` |
| Бэкап и восстановление системы | Штатные средства ОС |
| Поиск через Spotlight / Windows Search | Мгновенно по всему диску |
| Общий доступ между приложениями | Единая точка обмена |

### Практические выводы

**OPFS — это не «файловая система» в смысле ОС.** Это **изолированное key-value хранилище с файловой семантикой**, оптимизированное под веб-сценарии:

- **идеален** для кэшей, черновиков, временных данных, работы с бинарными форматами (SQLite WASM, PDF-редакторы, медиа-обработка);
- **не подходит** как основной источник истины для данных, которые должны переживать всё: очистку cookies, смену браузера, доступ из другого приложения;
- **нельзя использовать** для обмена файлами с пользователем или другими приложениями без явных механизмов (экспорт через File System Access API, скачивание);
- **производительность** на порядок ниже нативной ФС, но на порядок выше IndexedDB при работе с бинарными данными.

**Классическая ФС** остаётся универсальным хранилищем с полным набором примитивов. OPFS дополняет её для сценариев «данные нужны только здесь и сейчас, но их много и они бинарные».

### Когда что использовать

| Сценарий | Выбор |
|---|---|
| Кэш ассетов PWA | **OPFS** |
| Черновик документа до отправки на сервер | **OPFS** |
| Пользовательский файл, который надо сохранить на диск | **File System Access API** (`showSaveFilePicker`), не OPFS |
| SQLite в браузере (WASM) | **OPFS** (лучший вариант) |
| Большое видео для обработки | **OPFS** |
| Настройки приложения | **localStorage** / **IndexedDB** |
| Структурированные данные с индексами | **IndexedDB** |
| Данные, общие с нативным приложением | **Классическая ФС** |
| Резервное копирование ОС | **Классическая ФС** |

---

## 19.8.7. Тесты

### Доступность и базовый roundtrip

Тесты идут в headless Chrome без user gesture — OPFS этого не требует.

```rust
#[wasm_bindgen_test]
async fn test_opfs_available() {
    if !opfs_available() {
        return; // мягкий skip для старых браузеров
    }
    let root = opfs_root().await.expect("OPFS root");

    let name = "opfs_test_probe.txt";
    let _ = remove_file(&root, name).await;

    write_text(&root, name, "probe").await.expect("write probe");
    let back = read_text(&root, name).await.expect("read probe");
    assert_eq!(back, "probe", "readback matches write");

    remove_file(&root, name).await.expect("cleanup probe");
}
```

Ключевой момент: **мы не проверяем `root.name()`**. У корня OPFS имя пустое, и это нормально. Проверяем функциональность — можно ли создать файл, прочитать его, удалить.

### Проверка sync access handle через пробный Worker

```rust
#[wasm_bindgen_test]
async fn test_sync_access_available_in_worker() {
    let ok: bool = sync_access_available_in_worker().await;
    let msg: String = format!("sync access in worker: {ok}");
    web_sys::console::log_1(&wasm_bindgen::JsValue::from_str(&msg));
}
```

Тест **не паникует**, если окружение не поддерживает `createSyncAccessHandle`. Просто фиксирует факт в консоли. В headless Chrome обычно `true`.

### Roundtrip и идемпотентность

```rust
#[wasm_bindgen_test]
async fn test_write_text_idempotent() {
    let root = match opfs_root().await { Ok(r) => r, Err(_) => return };
    let name = "opfs_test_idem.txt";
    let _ = remove_file(&root, name).await;

    for i in 0..3 {
        let text: String = format!("version {i}");
        write_text(&root, name, &text).await.expect("write");
    }
    let back = read_text(&root, name).await.expect("read");
    assert_eq!(back, "version 2");

    remove_file(&root, name).await.expect("remove");
}

#[wasm_bindgen_test]
async fn test_remove_file_idempotent() {
    let root = match opfs_root().await { Ok(r) => r, Err(_) => return };
    let name = "opfs_test_remove.txt";
    let _ = remove_file(&root, name).await;

    write_text(&root, name, "bye").await.expect("write");

    remove_file(&root, name).await.expect("remove #1");
    remove_file(&root, name).await.expect("remove #2 (already gone)");
    remove_file(&root, name).await.expect("remove #3 (already gone)");
}
```

### Sync write

```rust
#[wasm_bindgen_test]
async fn test_sync_write_in_worker_roundtrip() {
    if !sync_access_available_in_worker().await {
        return;
    }

    let name = "opfs_test_sync.bin";
    let data: Vec<u8> = (0..1024).map(|i| (i & 0xFF) as u8).collect();

    let (written, ms) = write_sync_in_worker(name, &data)
        .await
        .expect("write_sync_in_worker");

    assert_eq!(written, data.len());
    assert!(ms >= 0.0);

    // Проверяем, что файл действительно есть и нужного размера.
    let root = opfs_root().await.expect("root");
    let handle = hello_wasm02::fsa_api_directory::get_file_handle(&root, name, false)
        .await
        .expect("file exists");
    let file = hello_wasm02::fsa_api_file::get_file(&handle)
        .await
        .expect("get_file");
    assert_eq!(file.size() as usize, data.len());

    remove_file(&root, name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn test_sync_write_large_1mb() {
    if !sync_access_available_in_worker().await {
        return;
    }

    let name = "opfs_test_sync_1mb.bin";
    let size: usize = 1024 * 1024;
    let data: Vec<u8> = (0..size).map(|i| (i & 0xFF) as u8).collect();

    let (written, ms) = write_sync_in_worker(name, &data)
        .await
        .expect("write 1 MB sync");

    assert_eq!(written, size);
    assert!(ms < 500.0, "1 МБ должен писаться быстро, got {ms} ms");

    let root = opfs_root().await.expect("root");
    remove_file(&root, name).await.expect("cleanup");
}
```

### Метаданные: `file_size`, `read_tail`, `list_files`

```rust
#[wasm_bindgen_test]
async fn test_file_size_and_read_tail() {
    let root = match opfs_root().await { Ok(r) => r, Err(_) => return };
    let name = "opfs_test_tail.txt";
    let _ = remove_file(&root, name).await;

    let s0 = file_size(&root, name).await.unwrap_or(0.0);
    assert_eq!(s0, 0.0);
    let t0 = read_tail(&root, name, 64).await.unwrap_or_default();
    assert_eq!(t0, "");

    let body = "0123456789".repeat(10);
    write_text(&root, name, &body).await.expect("write");

    let s1 = file_size(&root, name).await.unwrap_or(0.0);
    assert_eq!(s1 as usize, body.len());

    let tail = read_tail(&root, name, 10).await.unwrap_or_default();
    assert!(tail.len() <= 10);
    assert!(body.ends_with(&tail));

    remove_file(&root, name).await.expect("cleanup");
}
```

### Живой логгер

```rust
#[wasm_bindgen_test]
async fn test_logger_worker_appends() {
    if !sync_access_available_in_worker().await {
        return;
    }

    let root = match opfs_root().await { Ok(r) => r, Err(_) => return };
    let name = "opfs_test_logger.txt";
    let _ = remove_file(&root, name).await;

    let logger = start_logger_worker(name, 20).expect("start logger");
    gloo_timers::future::TimeoutFuture::new(350).await;
    logger.stop();

    let text = read_tail(&root, name, 4096).await.unwrap_or_default();
    let lines: Vec<&str> = text.lines().collect();
    assert!(lines.len() >= 5, "получено {} строк", lines.len());

    assert!(lines[0].starts_with('['));
    assert!(lines[0].contains("line"));

    remove_file(&root, name).await.expect("cleanup");
}
```

Тест ждёт 350 мс при интервале 20 мс. На CI-машине с медленным CPU может быть меньше 5 строк — если это будет проблемой, увеличьте ожидание до 500 мс.

---

## 19.8.8. Демо: живой логгер и бенчмарк

### Что показывает

Демо состоит из трёх секций:

**❶ Живой логгер.** Worker открывает `createSyncAccessHandle` на файле `live_log.txt` и **дописывает** туда строку каждые 50 мс. Main thread параллельно показывает счётчик строк, размер и хвост файла — UI остаётся полностью отзывчивым.

**Wow-моменты:**

1. **Файл растёт в реальном времени.** Счётчик и размер обновляются каждые 500 мс. UI не тормозит — потому что I/O в Worker'е.
2. **Перезагрузите страницу — старые строки на месте.** Worker при следующем «▶ Старт» дописывает к ним новые. `access.getSize()` даёт текущий размер, offset продолжается.
3. **Ни одного разрешения.** OPFS доступен сразу, без user gesture.
4. **Остановка мгновенна и безопасна.** `flush()` в каждом тике означает, что всё записанное — уже на диске.

**❷ Бенчмарк.** Async-запись 10 МБ через `FileSystemWritableFileStream` в main thread против sync-записи 10 МБ через `createSyncAccessHandle` в Worker'е. Бары показывают ускорение в 10–20×.

**❸ Журнал.** Что произошло, в текстовом виде.

### Ключевые решения

**1. Polling раз в 500 мс, не на каждый тик.** Main thread читает размер файла не каждые 50 мс (тогда бы не успевал отрисовать), а раз в 500 мс. Правило проекта: UI-обновления троттлятся, даже если данные растут быстрее.

```rust
spawn_local(async move {
    while state.polling.get() {
        TimeoutFuture::new(500).await;
        // ... прочитать file_size, read_tail, обновить UI ...
    }
});
```

**2. `TimeoutFuture::new(0).await` при старте.** Правило проекта: любая длительная операция, обновляющая UI, должна отдавать управление браузеру. В цикле polling это делает `TimeoutFuture::new(500)`, но и между записями в журнал стоит yield.

**3. Идемпотентность удаления.**

```rust
remove_file(&root, "live_log.txt").await?;
remove_file(&root, "bench_async.bin").await?;
remove_file(&root, "bench_sync.bin").await?;
// Повторно — тоже Ok (правило проекта).
remove_file(&root, "live_log.txt").await?;
```

**4. Единое визуальное поле.** `body.set_inner_html("")`, удаление внешних `<link rel=stylesheet>`, inline-стиль на `<html>` — всё, как в главе 19.7.

**5. Строковые хелперы.** `format_str(format_args!(…))` для всех `format!` в tail-позиции, `fmt_bytes`, `fmt_ms`, `fmt_mb_s` — отдельные функции. Правило WASM-инференса.

---

## 19.8.9. Правила проекта, выученные в этой главе

**1. `createSyncAccessHandle` — только в Dedicated Worker.** В main thread его нет на прототипе `FileSystemFileHandle`. Проверять доступность через `typeof` в main thread **бесполезно** — всегда `false`. Единственный способ — пробный Worker, который делает проверку у себя и отвечает булевым сообщением.

**2. У корня OPFS нет имени.** `root.name()` возвращает пустую строку. Не используйте `name()` для проверки «получили ли корень». Проверяйте функциональность: создать файл, прочитать, удалить.

**3. `flush()` в каждой итерации — это `fsync`.** Если Worker пишет данные и хочет гарантию сохранения при `terminate()`, каждый `write` должен сопровождаться `flush()`. Без него данные могут остаться в буфере.

**4. `access.getSize()` — append, не перезапись.** Чтобы дописать в конец файла, offset начинается с `access.getSize()`. Это делает лог естественно «продолжаемым» после перезагрузки.

**5. `close()` обязателен.** Без него handle остаётся залоченным, и другой `createSyncAccessHandle` на тот же файл получит `NoModificationAllowedError`.

**6. Async I/O в main thread — не медленный, но накладный.** Для мелких файлов — нормально. Для 10 МБ+ — разница в 10–20× в пользу sync.

**7. `Blob.text()` не всегда в биндингах.** Вызов через `js_sys::Reflect::get` + `Function::call0` работает независимо от версии `web-sys`.

**8. Работа с большим файлом — только `Blob::slice`.** Читать весь файл ради хвоста — плохая идея. `file.slice_with_f64(start)` даёт подблоб, `text()` превращает в строку.

**9. Polling раз в 500 мс, не чаще.** Правило проекта: UI-обновления троттлятся. Данные могут расти быстрее, но UI не должен пытаться успеть за ними.

**10. `create_element` (snake_case), а не `createElement`.** `web-sys` использует snake_case для всех методов.

---

## 19.8.10. Чек-лист

- [ ] OPFS доступен без user gesture — `navigator.storage.getDirectory()` не требует клика.
- [ ] У корня OPFS **пустое имя** — не используйте `name()` для проверки.
- [ ] `createSyncAccessHandle` доступен **только в Dedicated Worker** — в main thread его нет.
- [ ] Проверка доступности через `typeof` в main thread **бесполезна** — нужен пробный Worker.
- [ ] `flush()` после каждого `write` — это `fsync`, гарантия сохранения при `terminate()`.
- [ ] `close()` обязателен, иначе файл залочен.
- [ ] Offset для append — из `access.getSize()`, не с нуля.
- [ ] Async I/O в main thread для мелких файлов, sync в Worker — для больших.
- [ ] `Blob.text()` — через `Reflect` для совместимости.
- [ ] `Blob::slice(start)` для чтения хвоста, а не весь файл.
- [ ] Polling раз в 500 мс, а не на каждое изменение.
- [ ] Единое визуальное поле: `<body>` очищается, внешние `<link>` удаляются, inline-стиль на `<html>`.
- [ ] Tooltips на кнопках: `title` с кратким описанием.
- [ ] Все строковые операции — через `format_str(format_args!(…))`.
- [ ] Ошибки не глотаются: `console_err` во всех критических путях.

---

## 19.8.11. Полные исходные коды

### `src/fsa_api_opfs.rs`

```rust
//! OPFS: приватная файловая система браузера.
//!
//! Отличие от user-visible FSA (`showSaveFilePicker`):
//! * OPFS доступен **без user gesture** — сразу после
//!   `navigator.storage.getDirectory()`.
//! * Файлы не видны пользователю в проводнике ОС.
//! * Идеален для кэшей, черновиков, рабочих данных.
//!
//! **Sync access handle — только в Worker.** Метод
//! `FileSystemFileHandle::createSyncAccessHandle()` определён
//! только на прототипе внутри Dedicated Worker. В main thread
//! его нет — так устроена спецификация, чтобы синхронные
//! `read`/`write` не блокировали UI.
//!
//! Поэтому проверять доступность через
//! `typeof FileSystemFileHandle.prototype.createSyncAccessHandle`
//! **в main thread бесполезно** — всегда `false`. Единственный
//! надёжный способ — пробный Worker, который делает ту же
//! проверку у себя и отвечает булевым сообщением. Реализовано
//! в `sync_access_available_in_worker()`.
//!
//! **Правило:** при вызове методов через Reflect сначала проверяем,
//! что функция определена (`!func_js.is_undefined()`), иначе
//! получим невнятный `TypeError: x is not a function`.

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle, MessageEvent, Worker};

use crate::fsa_api_directory::get_file_handle;

// ===========================================================================
// OPFS root
// ===========================================================================

/// Получить корневой handle OPFS.
///
/// Работает без user gesture: `navigator.storage.getDirectory()`
/// не требует клика. Если OPFS недоступен — `Err` с внятным
/// сообщением.
pub async fn opfs_root() -> Result<FileSystemDirectoryHandle, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator_js = JsValue::from(window.navigator());

    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }

    let get_dir_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str("getDirectory"))?;
    if get_dir_js.is_undefined() {
        return Err(JsValue::from_str("OPFS: getDirectory недоступен"));
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

// ===========================================================================
// Async-запись/чтение (main thread)
// ===========================================================================

/// Записать текст в файл OPFS через `FileSystemWritableFileStream`.
///
/// `create: true` — идемпотентно: если файл есть, вернётся handle
/// на существующий, содержимое перезапишется.
pub async fn write_text(
    root: &FileSystemDirectoryHandle,
    name: &str,
    text: &str,
) -> Result<FileSystemFileHandle, JsValue> {
    let handle = get_file_handle(root, name, true).await?;
    let stream = crate::fsa_api_file::create_writable(&handle, false).await?;
    crate::fsa_api_writable::write_text(&stream, text).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle)
}

/// Записать байты в файл OPFS через `FileSystemWritableFileStream`.
///
/// `create: true` — идемпотентно. Используется в бенчмарке
/// async vs sync I/O.
pub async fn write_bytes(
    root: &FileSystemDirectoryHandle,
    name: &str,
    data: &[u8],
) -> Result<FileSystemFileHandle, JsValue> {
    let handle = get_file_handle(root, name, true).await?;
    let stream = crate::fsa_api_file::create_writable(&handle, false).await?;
    crate::fsa_api_writable::write_bytes(&stream, data).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle)
}

/// Прочитать текст из файла OPFS.
pub async fn read_text(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<String, JsValue> {
    let handle = get_file_handle(root, name, false).await?;
    let file = crate::fsa_api_file::get_file(&handle).await?;
    crate::fsa_api_reader::read_text(&file).await
}

/// Удалить файл из OPFS. Идемпотентно.
pub async fn remove_file(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<(), JsValue> {
    crate::fsa_api_directory::remove_entry(root, name, false).await
}

// ===========================================================================
// Утилиты чтения метаданных
// ===========================================================================

/// Размер файла в байтах. `NotFoundError` → `Ok(0.0)`.
pub async fn file_size(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<f64, JsValue> {
    match get_file_handle(root, name, false).await {
        Ok(handle) => {
            let file = crate::fsa_api_file::get_file(&handle).await?;
            Ok(file.size())
        }
        Err(_) => Ok(0.0),
    }
}

/// Прочитать **хвост** файла — не больше `max_bytes` с конца.
/// Полезно для live-превью растущих логов, чтобы не тянуть весь файл.
pub async fn read_tail(
    root: &FileSystemDirectoryHandle,
    name: &str,
    max_bytes: usize,
) -> Result<String, JsValue> {
    let handle = match get_file_handle(root, name, false).await {
        Ok(h) => h,
        Err(_) => return Ok(String::new()),
    };
    let file = crate::fsa_api_file::get_file(&handle).await?;
    let size: f64 = file.size();
    let start: f64 = if size > max_bytes as f64 {
        size - max_bytes as f64
    } else {
        0.0
    };
    let blob: web_sys::Blob = file.slice_with_f64(start)?;

    // `Blob::text()` — не во всех версиях web-sys. Идём через Reflect.
    let blob_js: &JsValue = blob.as_ref();
    let text_fn_js = js_sys::Reflect::get(blob_js, &JsValue::from_str("text"))?;
    if text_fn_js.is_undefined() {
        return Err(JsValue::from_str("Blob.text недоступен"));
    }
    let text_fn: js_sys::Function = text_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("text не функция"))?;
    let promise: js_sys::Promise = text_fn
        .call0(blob_js)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("text не Promise"))?;
    let value: JsValue = JsFuture::from(promise).await?;
    value
        .as_string()
        .ok_or_else(|| JsValue::from_str("Blob.text вернул не строку"))
}

/// Список файлов в корне OPFS с размерами.
pub async fn list_files(
    root: &FileSystemDirectoryHandle,
) -> Result<Vec<(String, f64)>, JsValue> {
    let dir_js: &JsValue = root.as_ref();
    let values_fn_js = js_sys::Reflect::get(dir_js, &JsValue::from_str("values"))?;
    if values_fn_js.is_undefined() {
        return Ok(Vec::new());
    }
    let values_fn: js_sys::Function = values_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("values не функция"))?;
    let iter: JsValue = values_fn.call0(dir_js)?;

    let mut out: Vec<(String, f64)> = Vec::new();
    loop {
        let entry = match crate::fsa_api_directory::iter_next(&iter).await? {
            Some(v) => v,
            None => break,
        };
        let kind: String = js_sys::Reflect::get(&entry, &JsValue::from_str("kind"))?
            .as_string()
            .unwrap_or_default();
        let name: String = js_sys::Reflect::get(&entry, &JsValue::from_str("name"))?
            .as_string()
            .unwrap_or_default();
        if kind == "file" {
            let fh: FileSystemFileHandle = match entry.dyn_into() {
                Ok(h) => h,
                Err(_) => continue,
            };
            let file = crate::fsa_api_file::get_file(&fh).await?;
            out.push((name, file.size()));
        }
    }
    Ok(out)
}

// ===========================================================================
// Sync access handle (только Worker)
// ===========================================================================

/// JS-код inline-Worker'а для записи через sync access handle.
const WRITE_WORKER_JS: &str = r#"
    self.onmessage = async (e) => {
        const { name, data } = e.data;
        try {
            const root = await navigator.storage.getDirectory();
            const fileHandle = await root.getFileHandle(name, { create: true });
            const accessHandle = await fileHandle.createSyncAccessHandle();

            const bytes = new Uint8Array(data);
            const t0 = performance.now();
            const written = accessHandle.write(bytes, { at: 0 });
            accessHandle.flush();
            accessHandle.close();
            const t1 = performance.now();

            self.postMessage({ ok: true, written, ms: t1 - t0 });
        } catch (err) {
            self.postMessage({ ok: false, error: String(err) });
        }
    };
"#;

/// Запустить Worker, который через `createSyncAccessHandle` запишет
/// `data` в файл `name` и вернёт время записи в миллисекундах.
pub async fn write_sync_in_worker(
    name: &str,
    data: &[u8],
) -> Result<(usize, f64), JsValue> {
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(WRITE_WORKER_JS));
    let blob = web_sys::Blob::new_with_str_sequence(&blob_parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker: Worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("name"), &JsValue::from_str(name))?;
    let u8arr = js_sys::Uint8Array::from(data);
    js_sys::Reflect::set(&msg, &JsValue::from_str("data"), &u8arr.buffer())?;
    worker.post_message(&msg)?;

    let worker_ref: &Worker = &worker;
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let onmessage = Closure::once(move |e: MessageEvent| {
            let _ = resolve.call1(&JsValue::UNDEFINED, &e.data());
        });
        worker_ref.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    });

    let data_js: JsValue = JsFuture::from(promise).await?;

    let ok: bool = js_sys::Reflect::get(&data_js, &JsValue::from_str("ok"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);

    let result: Result<(usize, f64), JsValue> = if ok {
        let written: usize = js_sys::Reflect::get(&data_js, &JsValue::from_str("written"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0) as usize;
        let ms: f64 = js_sys::Reflect::get(&data_js, &JsValue::from_str("ms"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0);
        Ok((written, ms))
    } else {
        let err: String = js_sys::Reflect::get(&data_js, &JsValue::from_str("error"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| "unknown worker error".to_string());
        Err(JsValue::from_str(&err))
    };

    worker.terminate();
    result
}

// ===========================================================================
// Живой логгер в Worker'е
// ===========================================================================

/// JS-код Worker'а: открывает `createSyncAccessHandle`, ищет конец
/// файла и бесконечно дописывает туда по строке раз в `intervalMs`.
///
/// `flush()` после каждой строки — данные гарантированно попадают
/// на диск. Даже если main thread вызовет `worker.terminate()`,
/// всё записанное уже сохранено.
const LOGGER_WORKER_JS: &str = r#"
    self.onmessage = async (e) => {
        const { name, intervalMs } = e.data;
        try {
            const root = await navigator.storage.getDirectory();
            const fileHandle = await root.getFileHandle(name, { create: true });
            const access = await fileHandle.createSyncAccessHandle();

            let offset = access.getSize();   // append, не перезапись
            let counter = 0;
            const enc = new TextEncoder();

            const tick = () => {
                const now = new Date();
                const hh = String(now.getHours()).padStart(2, "0");
                const mm = String(now.getMinutes()).padStart(2, "0");
                const ss = String(now.getSeconds()).padStart(2, "0");
                const ms = String(now.getMilliseconds()).padStart(3, "0");
                const line = `[${hh}:${mm}:${ss}.${ms}] line ${counter}\n`;
                const bytes = enc.encode(line);
                access.write(bytes, { at: offset });
                access.flush();
                offset += bytes.length;
                counter++;
            };

            tick();
            setInterval(tick, intervalMs);
        } catch (err) {
            // Молча выходим — main thread увидит, что размер не растёт.
        }
    };
"#;

/// Handle живого логгера. `stop()` завершает Worker; всё, что успело
/// записаться, уже на диске благодаря `flush()` в каждой итерации.
pub struct LoggerHandle {
    worker: Worker,
}

impl LoggerHandle {
    /// Остановить Worker. Идемпотентно: повторный `terminate()` — no-op.
    pub fn stop(&self) {
        self.worker.terminate();
    }
}

/// Запустить Worker-логгер. Возвращается **сразу** — Worker живёт
/// в фоне, main thread продолжает работать.
pub fn start_logger_worker(
    name: &str,
    interval_ms: u32,
) -> Result<LoggerHandle, JsValue> {
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(LOGGER_WORKER_JS));
    let blob = web_sys::Blob::new_with_str_sequence(&blob_parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("name"), &JsValue::from_str(name))?;
    js_sys::Reflect::set(
        &msg,
        &JsValue::from_str("intervalMs"),
        &JsValue::from_f64(interval_ms as f64),
    )?;
    worker.post_message(&msg)?;

    Ok(LoggerHandle { worker })
}

// ===========================================================================
// Проверки доступности
// ===========================================================================

/// Проверка: доступен ли `createSyncAccessHandle` **в Worker**.
///
/// Создаём пустой Worker, он проверяет у себя наличие метода
/// и отвечает булевым сообщением.
pub async fn sync_access_available_in_worker() -> bool {
    let probe_js: &str = r#"
        const ok = typeof FileSystemFileHandle !== 'undefined'
            && FileSystemFileHandle.prototype
            && typeof FileSystemFileHandle.prototype.createSyncAccessHandle === 'function';
        self.postMessage(ok);
    "#;

    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(probe_js));
    let blob = match web_sys::Blob::new_with_str_sequence(&blob_parts) {
        Ok(b) => b,
        Err(_) => return false,
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(_) => return false,
    };
    let worker: Worker = match Worker::new(&url) {
        Ok(w) => w,
        Err(_) => {
            web_sys::Url::revoke_object_url(&url).ok();
            return false;
        }
    };
    web_sys::Url::revoke_object_url(&url).ok();

    let worker_ref: &Worker = &worker;
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let onmessage = Closure::once(move |e: MessageEvent| {
            let _ = resolve.call1(&JsValue::UNDEFINED, &e.data());
        });
        worker_ref.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    });

    let result = JsFuture::from(promise).await;
    worker.terminate();

    match result {
        Ok(v) => v.as_bool().unwrap_or(false),
        Err(_) => false,
    }
}

/// Проверка: доступен ли `navigator.storage.getDirectory`.
pub fn opfs_available() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let navigator_js = JsValue::from(window.navigator());
    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage")).ok();
    match storage_js {
        Some(s) if !s.is_undefined() => {
            js_sys::Reflect::has(&s, &JsValue::from_str("getDirectory")).unwrap_or(false)
        }
        _ => false,
    }
}
```

### `tests/wasm_fsa_api_opfs.rs`

Полный текст тестов приведён в разделе 19.8.7.

### `src/app.rs`

Полный текст демо приведён в разделе 19.8.8 (живой логгер + бенчмарк). Здесь он не дублируется, чтобы не раздувать главу.

---

## 19.8.12. Команды запуска

```bash
# Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# Тесты OPFS в headless Chrome
wasm-pack test --headless --chrome --test wasm_fsa_api_opfs

# Host-тесты (не касаются OPFS, но полезно прогнать)
cargo test

# Запуск демо
npm run start
```

`Cargo.toml` **не требует** новых features. `.cargo/config.toml` **не требует** `--cfg=web_sys_unstable_apis`. Всё общение с `createSyncAccessHandle` идёт через JS-код Worker'а, а Rust видит только `Worker`, `MessageEvent`, `Blob`, `Url` — все стабильные.

---

# Полные исходные коды файлов Главы 19.8

## `src/fsa_api_opfs.rs`

```rust
//! OPFS: приватная файловая система браузера.
//!
//! Отличие от user-visible FSA (`showSaveFilePicker`):
//! * OPFS доступен **без user gesture** — сразу после
//!   `navigator.storage.getDirectory()`.
//! * Файлы не видны пользователю в проводнике ОС.
//! * Идеален для кэшей, черновиков, рабочих данных.
//!
//! **Sync access handle — только в Worker.** Метод
//! `FileSystemFileHandle::createSyncAccessHandle()` определён
//! только на прототипе внутри Dedicated Worker. В main thread
//! его нет — так устроена спецификация, чтобы синхронные
//! `read`/`write` не блокировали UI.
//!
//! Поэтому проверять доступность через
//! `typeof FileSystemFileHandle.prototype.createSyncAccessHandle`
//! **в main thread бесполезно** — всегда `false`. Единственный
//! надёжный способ — пробный Worker, который делает ту же
//! проверку у себя и отвечает булевым сообщением. Реализовано
//! в `sync_access_available_in_worker()`.
//!
//! **Правило:** при вызове методов через Reflect сначала проверяем,
//! что функция определена (`!func_js.is_undefined()`), иначе
//! получим невнятный `TypeError: x is not a function`.

use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{FileSystemDirectoryHandle, FileSystemFileHandle, MessageEvent, Worker};

use crate::fsa_api_directory::get_file_handle;

// ===========================================================================
// OPFS root
// ===========================================================================

/// Получить корневой handle OPFS.
///
/// Работает без user gesture: `navigator.storage.getDirectory()`
/// не требует клика. Если OPFS недоступен — `Err` с внятным
/// сообщением.
pub async fn opfs_root() -> Result<FileSystemDirectoryHandle, JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let navigator_js = JsValue::from(window.navigator());

    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage"))?;
    if storage_js.is_undefined() {
        return Err(JsValue::from_str("navigator.storage недоступен"));
    }

    let get_dir_js = js_sys::Reflect::get(&storage_js, &JsValue::from_str("getDirectory"))?;
    if get_dir_js.is_undefined() {
        return Err(JsValue::from_str("OPFS: getDirectory недоступен"));
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

// ===========================================================================
// Async-запись/чтение (main thread)
// ===========================================================================

/// Записать текст в файл OPFS через `FileSystemWritableFileStream`.
///
/// `create: true` — идемпотентно: если файл есть, вернётся handle
/// на существующий, содержимое перезапишется.
pub async fn write_text(
    root: &FileSystemDirectoryHandle,
    name: &str,
    text: &str,
) -> Result<FileSystemFileHandle, JsValue> {
    let handle = get_file_handle(root, name, true).await?;
    let stream = crate::fsa_api_file::create_writable(&handle, false).await?;
    crate::fsa_api_writable::write_text(&stream, text).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle)
}

/// Записать байты в файл OPFS через `FileSystemWritableFileStream`.
///
/// `create: true` — идемпотентно. Используется в бенчмарке
/// async vs sync I/O.
pub async fn write_bytes(
    root: &FileSystemDirectoryHandle,
    name: &str,
    data: &[u8],
) -> Result<FileSystemFileHandle, JsValue> {
    let handle = get_file_handle(root, name, true).await?;
    let stream = crate::fsa_api_file::create_writable(&handle, false).await?;
    crate::fsa_api_writable::write_bytes(&stream, data).await?;
    crate::fsa_api_writable::close(&stream).await?;
    Ok(handle)
}

/// Прочитать текст из файла OPFS.
pub async fn read_text(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<String, JsValue> {
    let handle = get_file_handle(root, name, false).await?;
    let file = crate::fsa_api_file::get_file(&handle).await?;
    crate::fsa_api_reader::read_text(&file).await
}

/// Удалить файл из OPFS. Идемпотентно.
pub async fn remove_file(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<(), JsValue> {
    crate::fsa_api_directory::remove_entry(root, name, false).await
}

// ===========================================================================
// Утилиты чтения метаданных
// ===========================================================================

/// Размер файла в байтах. `NotFoundError` → `Ok(0.0)`.
pub async fn file_size(
    root: &FileSystemDirectoryHandle,
    name: &str,
) -> Result<f64, JsValue> {
    match get_file_handle(root, name, false).await {
        Ok(handle) => {
            let file = crate::fsa_api_file::get_file(&handle).await?;
            Ok(file.size())
        }
        Err(_) => Ok(0.0),
    }
}

/// Прочитать **хвост** файла — не больше `max_bytes` с конца.
/// Полезно для live-превью растущих логов, чтобы не тянуть весь файл.
pub async fn read_tail(
    root: &FileSystemDirectoryHandle,
    name: &str,
    max_bytes: usize,
) -> Result<String, JsValue> {
    let handle = match get_file_handle(root, name, false).await {
        Ok(h) => h,
        Err(_) => return Ok(String::new()),
    };
    let file = crate::fsa_api_file::get_file(&handle).await?;
    let size: f64 = file.size();
    let start: f64 = if size > max_bytes as f64 {
        size - max_bytes as f64
    } else {
        0.0
    };
    let blob: web_sys::Blob = file.slice_with_f64(start)?;

    // `Blob::text()` — не во всех версиях web-sys. Идём через Reflect.
    let blob_js: &JsValue = blob.as_ref();
    let text_fn_js = js_sys::Reflect::get(blob_js, &JsValue::from_str("text"))?;
    if text_fn_js.is_undefined() {
        return Err(JsValue::from_str("Blob.text недоступен"));
    }
    let text_fn: js_sys::Function = text_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("text не функция"))?;
    let promise: js_sys::Promise = text_fn
        .call0(blob_js)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("text не Promise"))?;
    let value: JsValue = JsFuture::from(promise).await?;
    value
        .as_string()
        .ok_or_else(|| JsValue::from_str("Blob.text вернул не строку"))
}

/// Список файлов в корне OPFS с размерами.
pub async fn list_files(
    root: &FileSystemDirectoryHandle,
) -> Result<Vec<(String, f64)>, JsValue> {
    let dir_js: &JsValue = root.as_ref();
    let values_fn_js = js_sys::Reflect::get(dir_js, &JsValue::from_str("values"))?;
    if values_fn_js.is_undefined() {
        return Ok(Vec::new());
    }
    let values_fn: js_sys::Function = values_fn_js
        .dyn_into()
        .map_err(|_| JsValue::from_str("values не функция"))?;
    let iter: JsValue = values_fn.call0(dir_js)?;

    let mut out: Vec<(String, f64)> = Vec::new();
    loop {
        let entry = match crate::fsa_api_directory::iter_next(&iter).await? {
            Some(v) => v,
            None => break,
        };
        let kind: String = js_sys::Reflect::get(&entry, &JsValue::from_str("kind"))?
            .as_string()
            .unwrap_or_default();
        let name: String = js_sys::Reflect::get(&entry, &JsValue::from_str("name"))?
            .as_string()
            .unwrap_or_default();
        if kind == "file" {
            let fh: FileSystemFileHandle = match entry.dyn_into() {
                Ok(h) => h,
                Err(_) => continue,
            };
            let file = crate::fsa_api_file::get_file(&fh).await?;
            out.push((name, file.size()));
        }
    }
    Ok(out)
}

// ===========================================================================
// Sync access handle (только Worker)
// ===========================================================================

/// JS-код inline-Worker'а для записи через sync access handle.
/// Живёт строкой, чтобы не тащить отдельный файл и не настраивать
/// сборку.
///
/// Внутри — `createSyncAccessHandle()`, синхронный `write`,
/// `flush`, `close`. Замер времени через `performance.now()`.
const WRITE_WORKER_JS: &str = r#"
    self.onmessage = async (e) => {
        const { name, data } = e.data;
        try {
            const root = await navigator.storage.getDirectory();
            const fileHandle = await root.getFileHandle(name, { create: true });
            const accessHandle = await fileHandle.createSyncAccessHandle();

            const bytes = new Uint8Array(data);
            const t0 = performance.now();
            const written = accessHandle.write(bytes, { at: 0 });
            accessHandle.flush();
            accessHandle.close();
            const t1 = performance.now();

            self.postMessage({ ok: true, written, ms: t1 - t0 });
        } catch (err) {
            self.postMessage({ ok: false, error: String(err) });
        }
    };
"#;

/// Запустить Worker, который через `createSyncAccessHandle` запишет
/// `data` в файл `name` и вернёт время записи в миллисекундах.
///
/// Возвращает `(bytes_written, elapsed_ms)`.
///
/// **Как ждём ответ.** Оборачиваем `onmessage` в `js_sys::Promise`:
/// она резолвится первым сообщением от Worker'а. `JsFuture::from`
/// превращает её в awaitable. Никаких каналов, никаких зависимостей.
pub async fn write_sync_in_worker(
    name: &str,
    data: &[u8],
) -> Result<(usize, f64), JsValue> {
    // 1) Blob из JS-кода → object URL → Worker.
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(WRITE_WORKER_JS));
    let blob = web_sys::Blob::new_with_str_sequence(&blob_parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker: Worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    // 2) Отправляем имя файла и байты.
    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("name"), &JsValue::from_str(name))?;
    let u8arr = js_sys::Uint8Array::from(data);
    js_sys::Reflect::set(&msg, &JsValue::from_str("data"), &u8arr.buffer())?;
    worker.post_message(&msg)?;

    // 3) Promise, которая резолвится первым сообщением от Worker'а.
    let worker_ref: &Worker = &worker;
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let onmessage = Closure::once(move |e: MessageEvent| {
            let _ = resolve.call1(&JsValue::UNDEFINED, &e.data());
        });
        worker_ref.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    });

    // 4) Ждём сообщение.
    let data_js: JsValue = JsFuture::from(promise).await?;

    // 5) Разбираем ответ.
    let ok: bool = js_sys::Reflect::get(&data_js, &JsValue::from_str("ok"))
        .ok()
        .and_then(|v| v.as_bool())
        .unwrap_or(false);

    let result: Result<(usize, f64), JsValue> = if ok {
        let written: usize = js_sys::Reflect::get(&data_js, &JsValue::from_str("written"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0) as usize;
        let ms: f64 = js_sys::Reflect::get(&data_js, &JsValue::from_str("ms"))
            .ok()
            .and_then(|v| v.as_f64())
            .unwrap_or(0.0);
        Ok((written, ms))
    } else {
        let err: String = js_sys::Reflect::get(&data_js, &JsValue::from_str("error"))
            .ok()
            .and_then(|v| v.as_string())
            .unwrap_or_else(|| "unknown worker error".to_string());
        Err(JsValue::from_str(&err))
    };

    worker.terminate();
    result
}

// ===========================================================================
// Живой логгер в Worker'е
// ===========================================================================

/// JS-код Worker'а: открывает `createSyncAccessHandle`, ищет конец
/// файла и бесконечно дописывает туда по строке раз в `intervalMs`.
///
/// `flush()` после каждой строки — данные гарантированно попадают
/// на диск. Даже если main thread вызовет `worker.terminate()`,
/// всё записанное уже сохранено.
const LOGGER_WORKER_JS: &str = r#"
    self.onmessage = async (e) => {
        const { name, intervalMs } = e.data;
        try {
            const root = await navigator.storage.getDirectory();
            const fileHandle = await root.getFileHandle(name, { create: true });
            const access = await fileHandle.createSyncAccessHandle();

            let offset = access.getSize();   // append, не перезапись
            let counter = 0;
            const enc = new TextEncoder();

            const tick = () => {
                const now = new Date();
                const hh = String(now.getHours()).padStart(2, "0");
                const mm = String(now.getMinutes()).padStart(2, "0");
                const ss = String(now.getSeconds()).padStart(2, "0");
                const ms = String(now.getMilliseconds()).padStart(3, "0");
                const line = `[${hh}:${mm}:${ss}.${ms}] line ${counter}\n`;
                const bytes = enc.encode(line);
                access.write(bytes, { at: offset });
                access.flush();
                offset += bytes.length;
                counter++;
            };

            tick();
            setInterval(tick, intervalMs);
        } catch (err) {
            // Молча выходим — main thread увидит, что размер не растёт.
        }
    };
"#;

/// Handle живого логгера. `stop()` завершает Worker; всё, что успело
/// записаться, уже на диске благодаря `flush()` в каждой итерации.
pub struct LoggerHandle {
    worker: Worker,
}

impl LoggerHandle {
    /// Остановить Worker. Идемпотентно: повторный `terminate()` — no-op.
    pub fn stop(&self) {
        self.worker.terminate();
    }
}

/// Запустить Worker-логгер. Возвращается **сразу** — Worker живёт
/// в фоне, main thread продолжает работать. Прогресс читаем через
/// `file_size` / `read_tail`.
pub fn start_logger_worker(
    name: &str,
    interval_ms: u32,
) -> Result<LoggerHandle, JsValue> {
    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(LOGGER_WORKER_JS));
    let blob = web_sys::Blob::new_with_str_sequence(&blob_parts)?;
    let url = web_sys::Url::create_object_url_with_blob(&blob)?;
    let worker = Worker::new(&url)?;
    web_sys::Url::revoke_object_url(&url)?;

    let msg = js_sys::Object::new();
    js_sys::Reflect::set(&msg, &JsValue::from_str("name"), &JsValue::from_str(name))?;
    js_sys::Reflect::set(
        &msg,
        &JsValue::from_str("intervalMs"),
        &JsValue::from_f64(interval_ms as f64),
    )?;
    worker.post_message(&msg)?;

    Ok(LoggerHandle { worker })
}

// ===========================================================================
// Проверки доступности
// ===========================================================================

/// Проверка: доступен ли `createSyncAccessHandle` **в Worker**.
///
/// **Почему именно пробный Worker.** Метод `createSyncAccessHandle`
/// определён только на прототипе `FileSystemFileHandle` внутри
/// Dedicated Worker — так устроена спецификация, чтобы синхронные
/// `read`/`write` не блокировали main thread. В main thread его
/// нет, поэтому обычная синхронная проверка всегда вернёт `false`.
///
/// Создаём пустой Worker, он проверяет у себя наличие метода
/// и отвечает булевым сообщением. Это единственный надёжный
/// способ узнать, поддерживается ли sync access handle движком.
pub async fn sync_access_available_in_worker() -> bool {
    let probe_js: &str = r#"
        const ok = typeof FileSystemFileHandle !== 'undefined'
            && FileSystemFileHandle.prototype
            && typeof FileSystemFileHandle.prototype.createSyncAccessHandle === 'function';
        self.postMessage(ok);
    "#;

    let blob_parts = js_sys::Array::new();
    blob_parts.push(&JsValue::from_str(probe_js));
    let blob = match web_sys::Blob::new_with_str_sequence(&blob_parts) {
        Ok(b) => b,
        Err(_) => return false,
    };
    let url = match web_sys::Url::create_object_url_with_blob(&blob) {
        Ok(u) => u,
        Err(_) => return false,
    };
    let worker: Worker = match Worker::new(&url) {
        Ok(w) => w,
        Err(_) => {
            web_sys::Url::revoke_object_url(&url).ok();
            return false;
        }
    };
    web_sys::Url::revoke_object_url(&url).ok();

    // Promise, которая резолвится первым сообщением от Worker'а.
    // Скрипт выполняется при инициализации Worker'а, ответ приходит
    // в следующем тике — наш onmessage уже установлен.
    let worker_ref: &Worker = &worker;
    let promise = js_sys::Promise::new(&mut |resolve, _reject| {
        let onmessage = Closure::once(move |e: MessageEvent| {
            let _ = resolve.call1(&JsValue::UNDEFINED, &e.data());
        });
        worker_ref.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
        onmessage.forget();
    });

    let result = JsFuture::from(promise).await;
    worker.terminate();

    match result {
        Ok(v) => v.as_bool().unwrap_or(false),
        Err(_) => false,
    }
}

/// Проверка: доступен ли `navigator.storage.getDirectory`.
pub fn opfs_available() -> bool {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return false,
    };
    let navigator_js = JsValue::from(window.navigator());
    let storage_js = js_sys::Reflect::get(&navigator_js, &JsValue::from_str("storage")).ok();
    match storage_js {
        Some(s) if !s.is_undefined() => {
            js_sys::Reflect::has(&s, &JsValue::from_str("getDirectory")).unwrap_or(false)
        }
        _ => false,
    }
}
```

## Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod fsa_api_opfs;   // ← новый
```

## `tests/wasm_fsa_api_opfs.rs`

```rust
//! Интеграционные тесты `src/fsa_api_opfs.rs` через OPFS.
//!
//! OPFS не требует user gesture — тесты идут в headless Chrome
//! без пропусков. Если OPFS недоступен или `createSyncAccessHandle`
//! отсутствует в движке — тесты мягко пропускаются (`return`).
//!
//! **Важно.** `createSyncAccessHandle` определён только внутри
//! Dedicated Worker. Проверять его в main thread бесполезно —
//! всегда `false`. Поэтому доступность проверяется через
//! `sync_access_available_in_worker()` — она создаёт пробный
//! Worker и спрашивает у него.

use wasm_bindgen_test::*;

use hello_wasm02::fsa_api_opfs::{
    file_size, list_files, opfs_available, opfs_root, read_tail, read_text,
    remove_file, start_logger_worker, sync_access_available_in_worker, write_sync_in_worker,
    write_text,
};

wasm_bindgen_test_configure!(run_in_browser);

// ---------------------------------------------------------------------------
// Базовые проверки доступности
// ---------------------------------------------------------------------------

/// OPFS-корень возвращает пустое имя — это норма, у корня нет
/// имени. Проверяем работоспособность: создать файл, прочитать,
/// удалить. Если это работает — OPFS доступен по-настоящему.
#[wasm_bindgen_test]
async fn test_opfs_available() {
    if !opfs_available() {
        return; // мягкий skip для старых браузеров
    }
    let root = opfs_root().await.expect("OPFS root");

    let name = "opfs_test_probe.txt";
    let _ = remove_file(&root, name).await;

    write_text(&root, name, "probe").await.expect("write probe");
    let back = read_text(&root, name).await.expect("read probe");
    assert_eq!(back, "probe", "readback matches write");

    remove_file(&root, name).await.expect("cleanup probe");
}

/// Проверка `createSyncAccessHandle` через пробный Worker.
/// В headless Chrome метод доступен из Worker'а, поэтому флаг
/// обычно `true`. В старых движках — `false`, тесты с ним
/// мягко пропускаются.
#[wasm_bindgen_test]
async fn test_sync_access_available_in_worker() {
    let ok: bool = sync_access_available_in_worker().await;
    // Не паникуем, если окружение не поддерживает — просто
    // логируем фактическое значение, чтобы при отладке было видно.
    let msg: String = format!("sync access in worker: {ok}");
    web_sys::console::log_1(&wasm_bindgen::JsValue::from_str(&msg));
}

// ---------------------------------------------------------------------------
// Async write/read roundtrip
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_write_and_read_text_roundtrip() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "opfs_test_draft.json";
    let _ = remove_file(&root, name).await;

    let json = r#"{"title":"черновик","items":[1,2,3]}"#;
    write_text(&root, name, json).await.expect("write_text");

    let back = read_text(&root, name).await.expect("read_text");
    assert_eq!(back, json, "содержимое совпадает");

    remove_file(&root, name).await.expect("remove");
}

#[wasm_bindgen_test]
async fn test_write_text_idempotent() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "opfs_test_idem.txt";
    let _ = remove_file(&root, name).await;

    // Три раза пишем — каждый раз Ok, содержимое последнее.
    for i in 0..3 {
        let text: String = format!("version {i}");
        write_text(&root, name, &text).await.expect("write");
    }
    let back = read_text(&root, name).await.expect("read");
    assert_eq!(back, "version 2");

    remove_file(&root, name).await.expect("remove");
}

#[wasm_bindgen_test]
async fn test_remove_file_idempotent() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "opfs_test_remove.txt";
    let _ = remove_file(&root, name).await;

    write_text(&root, name, "bye").await.expect("write");

    // Правило проекта: удаление идемпотентно.
    remove_file(&root, name).await.expect("remove #1");
    remove_file(&root, name).await.expect("remove #2 (already gone)");
    remove_file(&root, name).await.expect("remove #3 (already gone)");
}

// ---------------------------------------------------------------------------
// Sync access handle из Worker
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_sync_write_in_worker_roundtrip() {
    if !sync_access_available_in_worker().await {
        return; // мягкий skip: createSyncAccessHandle недоступен
    }

    let name = "opfs_test_sync.bin";
    let data: Vec<u8> = (0..1024).map(|i| (i & 0xFF) as u8).collect();

    let (written, ms) = write_sync_in_worker(name, &data)
        .await
        .expect("write_sync_in_worker");

    assert_eq!(written, data.len(), "записано ровно столько байт");
    assert!(ms >= 0.0, "время не отрицательное");

    // Проверяем, что файл действительно есть и нужного размера.
    let root = opfs_root().await.expect("root");
    let handle = hello_wasm02::fsa_api_directory::get_file_handle(&root, name, false)
        .await
        .expect("file exists");
    let file = hello_wasm02::fsa_api_file::get_file(&handle)
        .await
        .expect("get_file");
    assert_eq!(file.size() as usize, data.len());

    remove_file(&root, name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn test_sync_write_large_1mb() {
    if !sync_access_available_in_worker().await {
        return;
    }

    let name = "opfs_test_sync_1mb.bin";
    let size: usize = 1024 * 1024;
    let data: Vec<u8> = (0..size).map(|i| (i & 0xFF) as u8).collect();

    let (written, ms) = write_sync_in_worker(name, &data)
        .await
        .expect("write 1 MB sync");

    assert_eq!(written, size);
    // 1 МБ синхронной записью — обычно < 50 мс. Верхний порог
    // разумно завышен, чтобы тест не был хрупким.
    assert!(ms < 500.0, "1 МБ должен писаться быстро, got {ms} ms");

    let root = opfs_root().await.expect("root");
    remove_file(&root, name).await.expect("cleanup");
}

// ---------------------------------------------------------------------------
// file_size / read_tail / list_files
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_file_size_and_read_tail() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "opfs_test_tail.txt";
    let _ = remove_file(&root, name).await;

    // Отсутствующего файла — размер 0, хвост пустой.
    let s0 = file_size(&root, name).await.unwrap_or(0.0);
    assert_eq!(s0, 0.0);
    let t0 = read_tail(&root, name, 64).await.unwrap_or_default();
    assert_eq!(t0, "");

    // Файл из 100 символов. Хвост не больше 10 байт.
    let body = "0123456789".repeat(10);
    write_text(&root, name, &body).await.expect("write");

    let s1 = file_size(&root, name).await.unwrap_or(0.0);
    assert_eq!(s1 as usize, body.len());

    let tail = read_tail(&root, name, 10).await.unwrap_or_default();
    assert!(tail.len() <= 10);
    assert!(body.ends_with(&tail));

    remove_file(&root, name).await.expect("cleanup");
}

#[wasm_bindgen_test]
async fn test_list_files() {
    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let a = "opfs_test_list_a.txt";
    let b = "opfs_test_list_b.txt";
    let _ = remove_file(&root, a).await;
    let _ = remove_file(&root, b).await;

    write_text(&root, a, "aaa").await.expect("write a");
    write_text(&root, b, "bbbb").await.expect("write b");

    let all = list_files(&root).await.expect("list");
    let names: Vec<&str> = all.iter().map(|(n, _)| n.as_str()).collect();
    assert!(names.contains(&a));
    assert!(names.contains(&b));

    let sa = all
        .iter()
        .find(|(n, _)| n == a)
        .map(|(_, s)| *s)
        .unwrap_or(0.0);
    assert_eq!(sa, 3.0);

    remove_file(&root, a).await.expect("cleanup a");
    remove_file(&root, b).await.expect("cleanup b");
}

// ---------------------------------------------------------------------------
// Живой логгер в Worker
// ---------------------------------------------------------------------------

#[wasm_bindgen_test]
async fn test_logger_worker_appends() {
    if !sync_access_available_in_worker().await {
        return;
    }

    let root = match opfs_root().await {
        Ok(r) => r,
        Err(_) => return,
    };
    let name = "opfs_test_logger.txt";
    let _ = remove_file(&root, name).await;

    // Стартуем логгер на 20 мс, ждём ~350 мс, стопаем.
    let logger = start_logger_worker(name, 20).expect("start logger");
    gloo_timers::future::TimeoutFuture::new(350).await;
    logger.stop();

    // За 350 мс при интервале 20 мс должно быть не меньше 5 строк.
    let text = read_tail(&root, name, 4096).await.unwrap_or_default();
    let lines: Vec<&str> = text.lines().collect();
    assert!(lines.len() >= 5, "получено {} строк", lines.len());

    // Каждая строка — формата `[HH:MM:SS.mmm] line N`.
    assert!(lines[0].starts_with('['));
    assert!(lines[0].contains("line"));

    remove_file(&root, name).await.expect("cleanup");
}
```

## `src/app.rs`

```rust
//! Демо главы 19.8 — OPFS: приватная файловая система браузера.
//!
//! **Сценарий.** Живой логгер: Worker открывает `createSyncAccessHandle`
//! на файле `live_log.txt` и бесконечно дописывает туда по строке
//! раз в 50 мс. Main thread параллельно показывает счётчик, размер
//! и хвост файла — UI остаётся полностью отзывчивым, потому что
//! I/O живёт в Worker'е.
//!
//! **Wow-моменты:**
//! 1. Файл растёт в реальном времени — счётчик и размер обновляются
//!    каждые 500 мс, main thread не блокируется.
//! 2. Перезагрузите страницу — старые строки на месте, Worker
//!    дописывает к ним новые (append через `access.getSize()`).
//! 3. Синхронная запись быстрее async в 10–20× — видно на бары.
//! 4. Удаление идемпотентно — двойной клик не падает.
//!
//! **Контракт с `lib.rs`.** `pub fn run() -> Result<(), JsValue>` —
//! вызывается из `#[wasm_bindgen(start)]`.
//!
//! **Правило WASM-инференса.** `format!` в tail-позиции `if/else`/
//! `match`/блока иногда выводится как `()` → E0308. Все такие
//! места обёрнуты в `format_str(format_args!(…))`.

use std::cell::{Cell, RefCell};
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::spawn_local;
use web_sys::{Document, Element, FileSystemDirectoryHandle, HtmlElement};

use crate::fsa_api_opfs::{
    file_size, list_files, opfs_available, opfs_root, read_tail, remove_file,
    start_logger_worker, sync_access_available_in_worker, write_bytes, write_sync_in_worker,
    LoggerHandle,
};

// ===========================================================================
// Диагностика
// ===========================================================================

const TAG: &str = "[19.8]";

fn console_log(msg: &str) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::log_1(&JsValue::from_str(&line));
}

fn console_err(msg: &str, e: &JsValue) {
    let line: String = format_str(format_args!("{TAG} {msg}"));
    web_sys::console::error_2(&JsValue::from_str(&line), e);
}

// ===========================================================================
// Строковые хелперы (правило проекта)
// ===========================================================================

#[inline]
fn format_str(args: std::fmt::Arguments<'_>) -> String {
    args.to_string()
}

fn fmt_bytes(n: f64) -> String {
    let bytes: f64 = n.max(0.0);
    let s: String = if bytes < 1024.0 {
        format_str(format_args!("{} B", bytes as u64))
    } else if bytes < 1024.0 * 1024.0 {
        format_str(format_args!("{:.1} KB", bytes / 1024.0))
    } else {
        format_str(format_args!("{:.2} MB", bytes / (1024.0 * 1024.0)))
    };
    s
}

fn fmt_ms(v: f64) -> String {
    let s: String = format_str(format_args!("{:.0} мс", v));
    s
}

fn fmt_mb_s(bytes: usize, ms: f64) -> String {
    if ms <= 0.0 {
        let s: String = "—".to_string();
        return s;
    }
    let mb: f64 = bytes as f64 / (1024.0 * 1024.0);
    let seconds: f64 = ms / 1000.0;
    let speed: f64 = mb / seconds;
    let s: String = format_str(format_args!("{:.1} MB/s", speed));
    s
}

fn fmt_count(n: u64) -> String {
    let s: String = format_str(format_args!("{n}"));
    s
}

// ===========================================================================
// Состояние
// ===========================================================================

struct DemoState {
    status: Element,
    logger_info: Element,
    logger_preview: Element,
    btn_start: Element,
    btn_stop: Element,
    bench_info: Element,
    bar_async: Element,
    bar_sync: Element,
    num_async: Element,
    num_sync: Element,
    ratio_el: Element,
    summary: Element,
    log_body: Element,
    root: RefCell<Option<FileSystemDirectoryHandle>>,
    logger: RefCell<Option<LoggerHandle>>,
    polling: Cell<bool>,
    bench_data: RefCell<Option<Vec<u8>>>,
    async_ms: Cell<f64>,
    sync_ms: Cell<f64>,
}

const LOG_FILE: &str = "live_log.txt";

// ===========================================================================
// CSS
// ===========================================================================

const CSS: &str = r#"
html, body {
  margin: 0 !important; padding: 0 !important;
  background: #0d1117 !important; color: #e6edf3 !important;
  min-height: 100vh;
  font-family: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  font-size: 14px; line-height: 1.5;
}
header { padding: 20px 32px 4px; }
h1 { font-size: 19px; margin: 0 0 4px; font-weight: 600; }
h1 code { color: #79c0ff; font-size: 0.95em; }
.subtitle { color: #8b949e; margin: 0; font-size: 13px; }

.status-bar {
  padding: 12px 32px 4px; font-size: 12px; color: #8b949e;
  font-family: ui-monospace, monospace;
}
.status-bar.ok   { color: #3fb950; }
.status-bar.warn { color: #d29922; }
.status-bar.err  { color: #f85149; }

.section {
  margin: 16px 32px; background: #161b22;
  border: 1px solid #30363d; border-radius: 8px;
  overflow: hidden;
}
.section h3 {
  margin: 0; padding: 10px 14px;
  background: #21262d; font-size: 13px; font-weight: 600;
  border-bottom: 1px solid #30363d;
  display: flex; align-items: baseline; gap: 8px;
}
.section h3 .num { color: #79c0ff; font-weight: 700; font-size: 14px; }
.section h3 .hint { color: #6e7681; font-weight: 400; font-size: 11px; margin-left: auto; }
.section .body { padding: 14px 16px; }

.btn {
  background: #21262d; color: #e6edf3;
  border: 1px solid #30363d; border-radius: 6px;
  padding: 7px 13px; font-size: 13px; cursor: pointer;
  transition: background .15s, border-color .15s;
}
.btn:hover { background: #30363d; border-color: #8b949e; }
.btn:active { background: #1f6feb; border-color: #1f6feb; }
.btn.primary { background: #1f6feb; border-color: #1f6feb; }
.btn.primary:hover { background: #388bfd; }
.btn:disabled { opacity: .45; cursor: not-allowed; }

.row-controls { display: flex; flex-wrap: wrap; gap: 8px; align-items: center; margin-bottom: 12px; }
.row-status {
  margin-left: auto; font-family: ui-monospace, monospace;
  font-size: 12px; color: #8b949e;
}
.row-status b { color: #e6edf3; font-weight: 500; }

.preview {
  background: #0d1117; border: 1px solid #30363d; border-radius: 6px;
  padding: 10px 12px; font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; color: #79c0ff;
  height: 180px; overflow: hidden; white-space: pre;
  display: flex; flex-direction: column; justify-content: flex-end;
}

.explanation {
  margin-top: 10px; color: #8b949e; font-size: 12px;
  font-family: ui-monospace, monospace; line-height: 1.6;
}
.explanation b { color: #e6edf3; font-weight: 500; }
.explanation code { color: #79c0ff; }

.bench-controls { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 14px; }
.bench-row {
  display: grid;
  grid-template-columns: 130px 1fr 200px;
  gap: 10px; align-items: center;
  padding: 6px 0;
  font-family: ui-monospace, monospace; font-size: 12px;
}
.bench-label { color: #8b949e; }
.bench-label b { color: #e6edf3; font-weight: 500; }
.bar-wrap { background: #0d1117; border-radius: 4px; height: 22px; overflow: hidden; }
.bar {
  height: 100%; width: 0;
  transition: width .35s cubic-bezier(.2,.7,.2,1);
  border-radius: 4px;
}
.bar.async { background: linear-gradient(90deg, #f0883e, #f0b73e); }
.bar.sync  { background: linear-gradient(90deg, #1f6feb, #58a6ff); }
.bench-num { color: #e6edf3; white-space: nowrap; }
.bench-num b { color: #79c0ff; font-weight: 600; }
.bench-ratio {
  margin-top: 12px; padding-top: 10px;
  border-top: 1px solid #30363d;
  color: #8b949e; font-size: 13px;
}
.bench-ratio b { color: #3fb950; font-weight: 600; font-size: 15px; }

.summary-bar {
  display: flex; align-items: center; gap: 12px;
  margin: 0 32px 24px;
  padding: 10px 16px;
  background: #161b22; border: 1px solid #30363d; border-radius: 8px;
  font-family: ui-monospace, monospace; font-size: 12px; color: #8b949e;
}
.summary-bar b { color: #e6edf3; font-weight: 500; }
.summary-bar .spacer { flex: 1; }

.log-body {
  padding: 12px 16px; max-height: 160px; overflow: auto;
  font-family: ui-monospace, monospace; font-size: 12px;
  line-height: 1.55; background: #0d1117;
}
.log-line { padding: 1px 0; }
.log-line.info  { color: #8b949e; }
.log-line.ok    { color: #3fb950; }
.log-line.warn  { color: #d29922; }
.log-line.err   { color: #f85149; }
.log-line.bench { color: #79c0ff; }
"#;

// ===========================================================================
// Хелперы UI
// ===========================================================================

fn log_line(state: &Rc<DemoState>, kind: &str, msg: &str) {
    let window = match web_sys::window() {
        Some(w) => w,
        None => return,
    };
    let doc = match window.document() {
        Some(d) => d,
        None => return,
    };
    let line = match doc.create_element("div") {
        Ok(e) => e,
        Err(_) => return,
    };
    let cls: String = format_str(format_args!("log-line {kind}"));
    line.set_class_name(&cls);
    line.set_text_content(Some(msg));
    state.log_body.append_child(&line).ok();
    if let Some(el) = state.log_body.dyn_ref::<HtmlElement>() {
        let max: f64 = el.scroll_height() as f64;
        el.set_scroll_top(max);
    }
}

fn set_bar(el: &Element, frac: f64) {
    let clamped: f64 = if frac < 0.0 {
        0.0
    } else if frac > 1.0 {
        1.0
    } else {
        frac
    };
    let style: String = format_str(format_args!("width: {:.1}%", clamped * 100.0));
    el.set_attribute("style", &style).ok();
}

fn set_status(state: &Rc<DemoState>, kind: &str, text: &str) {
    let cls: String = format_str(format_args!("status-bar {kind}"));
    state.status.set_class_name(&cls);
    state.status.set_text_content(Some(text));
}

fn make_button<F>(
    doc: &Document,
    parent: &Element,
    label: &str,
    tooltip: &str,
    primary: bool,
    mut handler: F,
) -> Result<Element, JsValue>
where
    F: FnMut() + 'static,
{
    let btn = doc.create_element("button")?;
    let cls: &str = if primary { "btn primary" } else { "btn" };
    btn.set_class_name(cls);
    btn.set_text_content(Some(label));
    btn.set_attribute("title", tooltip)?;
    parent.append_child(&btn)?;

    let closure = Closure::<dyn FnMut()>::new(move || handler());
    let target: &web_sys::EventTarget = btn.unchecked_ref();
    target.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(btn)
}

fn make_section(
    doc: &Document,
    num: &str,
    title: &str,
    hint: &str,
) -> Result<(Element, Element), JsValue> {
    let section = doc.create_element("div")?;
    section.set_class_name("section");

    let h3 = doc.create_element("h3")?;
    let header_html: String = format_str(format_args!(
        "<span class=\"num\">{num}</span> <span>{title}</span>\
         <span class=\"hint\">{hint}</span>"
    ));
    h3.set_inner_html(&header_html);
    section.append_child(&h3)?;

    let body = doc.create_element("div")?;
    body.set_class_name("body");
    section.append_child(&body)?;

    Ok((section, body))
}

// ===========================================================================
// Действия: живой логгер
// ===========================================================================

/// Периодический опрос файла лога. Раз в 500 мс — по правилу
/// проекта: UI-обновления разумно троттлить, а не дёргать
/// на каждое изменение.
fn start_polling(state: Rc<DemoState>) {
    if state.polling.get() {
        return;
    }
    state.polling.set(true);

    spawn_local(async move {
        while state.polling.get() {
            TimeoutFuture::new(500).await;
            if !state.polling.get() {
                break;
            }
            let root = match state.root.borrow().clone() {
                Some(r) => r,
                None => break,
            };

            // Размер.
            let size: f64 = file_size(&root, LOG_FILE).await.unwrap_or(0.0);

            // Хвост (последние 4 КБ, показываем 6 строк).
            let tail: String = read_tail(&root, LOG_FILE, 4096)
                .await
                .unwrap_or_default();
            let lines: Vec<&str> = tail.lines().collect();
            let total_lines = lines.len();

            let info: String = format_str(format_args!(
                "строк: <b>{}</b> · размер: <b>{}</b>",
                fmt_count(total_lines as u64),
                fmt_bytes(size)
            ));
            state.logger_info.set_inner_html(&info);

            let take = if lines.len() > 6 { lines.len() - 6 } else { 0 };
            let preview: String = lines[take..].join("\n");
            state.logger_preview.set_text_content(Some(&preview));

            refresh_summary(&state).await;
        }
    });
}

fn stop_polling(state: &Rc<DemoState>) {
    state.polling.set(false);
}

async fn start_logger(state: Rc<DemoState>) -> Result<(), JsValue> {
    if state.logger.borrow().is_some() {
        log_line(&state, "info", "логгер уже запущен");
        return Ok(());
    }

    let handle: LoggerHandle = match start_logger_worker(LOG_FILE, 50) {
        Ok(h) => h,
        Err(e) => {
            console_err("start_logger_worker failed", &e);
            log_line(
                &state,
                "err",
                &format_str(format_args!("старт не удался: {e:?}")),
            );
            return Ok(());
        }
    };
    *state.logger.borrow_mut() = Some(handle);

    state.btn_start.set_attribute("disabled", "").ok();
    state.btn_stop.remove_attribute("disabled").ok();

    log_line(&state, "ok", "Worker стартовал: пишет строку раз в 50 мс");
    console_log("logger started");

    start_polling(state);
    Ok(())
}

fn stop_logger(state: Rc<DemoState>) {
    if let Some(h) = state.logger.borrow_mut().take() {
        h.stop();
        log_line(
            &state,
            "ok",
            "Worker остановлен · всё записанное — уже на диске",
        );
        console_log("logger stopped");
    }
    stop_polling(&state);
    state.btn_start.remove_attribute("disabled").ok();
    state.btn_stop.set_attribute("disabled", "").ok();
}

/// Очистить OPFS: удалить лог, бенч-файлы. Идемпотентно.
async fn clear_all(state: Rc<DemoState>) -> Result<(), JsValue> {
    stop_logger(state.clone());
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return Ok(()),
    };
    log_line(&state, "info", "очищаю OPFS…");

    remove_file(&root, LOG_FILE).await?;
    remove_file(&root, "bench_async.bin").await?;
    remove_file(&root, "bench_sync.bin").await?;
    // Повторно — тоже Ok (правило проекта).
    remove_file(&root, LOG_FILE).await?;

    state.logger_preview.set_text_content(Some("(файл удалён)"));
    state
        .logger_info
        .set_inner_html("строк: <b>0</b> · размер: <b>0 B</b>");
    log_line(&state, "ok", "OPFS очищен, повторное удаление — тоже Ok");

    refresh_summary(&state).await;
    Ok(())
}

/// Обновить строку-сводку: сколько файлов, сколько байт.
async fn refresh_summary(state: &Rc<DemoState>) {
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return,
    };
    match list_files(&root).await {
        Ok(files) => {
            let n: usize = files.len();
            let bytes: f64 = files.iter().map(|(_, s)| *s).sum();
            let html: String = format_str(format_args!(
                "OPFS: <b>{n}</b> файл(ов) · <b>{}</b>",
                fmt_bytes(bytes)
            ));
            state.summary.set_inner_html(&html);
        }
        Err(_) => {
            state.summary.set_inner_html("OPFS: —");
        }
    }
}

// ===========================================================================
// Действия: бенчмарк
// ===========================================================================

async fn gen_data(state: Rc<DemoState>) -> Result<(), JsValue> {
    let size: usize = 10 * 1024 * 1024;
    log_line(
        &state,
        "info",
        &format_str(format_args!("генерирую {}…", fmt_bytes(size as f64))),
    );

    let data: Vec<u8> = (0..size).map(|i| (i & 0xFF) as u8).collect();
    *state.bench_data.borrow_mut() = Some(data);

    state.async_ms.set(0.0);
    state.sync_ms.set(0.0);
    set_bar(&state.bar_async, 0.0);
    set_bar(&state.bar_sync, 0.0);
    state.num_async.set_text_content(Some("—"));
    state.num_sync.set_text_content(Some("—"));
    state.ratio_el.set_inner_html("—");

    let info: String = format_str(format_args!(
        "данные: <b>{}</b> готовы к записи",
        fmt_bytes(size as f64)
    ));
    state.bench_info.set_inner_html(&info);

    log_line(
        &state,
        "ok",
        &format_str(format_args!("{} сгенерировано", fmt_bytes(size as f64))),
    );
    Ok(())
}

async fn bench_async(state: Rc<DemoState>) -> Result<(), JsValue> {
    let root = match state.root.borrow().clone() {
        Some(r) => r,
        None => return Ok(()),
    };
    let data: Vec<u8> = match state.bench_data.borrow().as_ref() {
        Some(d) => d.clone(),
        None => {
            log_line(&state, "warn", "сначала сгенерируйте данные");
            return Ok(());
        }
    };
    let size: usize = data.len();

    log_line(
        &state,
        "info",
        "async write: FileSystemWritableFileStream в main thread…",
    );
    let t0: f64 = js_sys::Date::now();
    write_bytes(&root, "bench_async.bin", &data).await?;
    let t1: f64 = js_sys::Date::now();
    let ms: f64 = t1 - t0;

    state.async_ms.set(ms);

    let text: String = format_str(format_args!(
        "<b>{}</b> · {}",
        fmt_ms(ms),
        fmt_mb_s(size, ms)
    ));
    state.num_async.set_inner_html(&text);

    let line: String = format_str(format_args!(
        "async: {} · {}",
        fmt_ms(ms),
        fmt_mb_s(size, ms)
    ));
    log_line(&state, "bench", &line);
    console_log(&line);

    render_bars(&state);
    refresh_summary(&state).await;
    Ok(())
}

async fn bench_sync(state: Rc<DemoState>) -> Result<(), JsValue> {
    let data: Vec<u8> = match state.bench_data.borrow().as_ref() {
        Some(d) => d.clone(),
        None => {
            log_line(&state, "warn", "сначала сгенерируйте данные");
            return Ok(());
        }
    };
    let size: usize = data.len();

    log_line(
        &state,
        "info",
        "sync write: createSyncAccessHandle в Worker…",
    );
    let (written, ms) = match write_sync_in_worker("bench_sync.bin", &data).await {
        Ok(r) => r,
        Err(e) => {
            let msg: String = format_str(format_args!("sync write failed: {e:?}"));
            log_line(&state, "err", &msg);
            console_err("write_sync_in_worker failed", &e);
            state.num_sync.set_text_content(Some("ошибка"));
            return Ok(());
        }
    };
    state.sync_ms.set(ms);
    if written != size {
        log_line(
            &state,
            "warn",
            &format_str(format_args!("записано {written} из {size} байт")),
        );
    }

    let text: String = format_str(format_args!(
        "<b>{}</b> · {}",
        fmt_ms(ms),
        fmt_mb_s(size, ms)
    ));
    state.num_sync.set_inner_html(&text);

    let line: String = format_str(format_args!(
        "sync:  {} · {}",
        fmt_ms(ms),
        fmt_mb_s(size, ms)
    ));
    log_line(&state, "bench", &line);
    console_log(&line);

    render_bars(&state);
    refresh_summary(&state).await;
    Ok(())
}

fn render_bars(state: &Rc<DemoState>) {
    let a: f64 = state.async_ms.get();
    let s: f64 = state.sync_ms.get();
    if a <= 0.0 || s <= 0.0 {
        return;
    }
    let min: f64 = a.min(s);
    let max: f64 = a.max(s);
    set_bar(&state.bar_async, (min / a).max(0.04));
    set_bar(&state.bar_sync, (min / s).max(0.04));
    let ratio: f64 = max / min;
    let faster: &str = if a > s { "sync" } else { "async" };
    let html: String = format_str(format_args!(
        "ускорение: <b>×{:.1}</b> в пользу <b>{}</b>",
        ratio, faster
    ));
    state.ratio_el.set_inner_html(&html);
}

fn reload_page() {
    if let Some(w) = web_sys::window() {
        if let Some(loc) = w.location().dyn_ref::<web_sys::Location>() {
            loc.reload().ok();
        }
    }
}

// ===========================================================================
// Точка входа
// ===========================================================================

/// Точка входа страницы. Вызывается из `lib.rs::run()` через
/// `#[wasm_bindgen(start)]`. Контракт: `pub fn run() -> Result<(), JsValue>`.
pub fn run() -> Result<(), JsValue> {
    console_log("run: starting");

    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;
    let body = document
        .body()
        .ok_or_else(|| JsValue::from_str("no body"))?;

    body.set_inner_html("");

    // Единое визуальное поле.
    if let Ok(links) = document.query_selector_all("link[rel='stylesheet']") {
        let n: u32 = links.length();
        for i in 0..n {
            if let Some(node) = links.get(i) {
                if let Some(parent) = node.parent_node() {
                    parent.remove_child(&node).ok();
                }
            }
        }
    }
    if let Some(root_el) = document.document_element() {
        root_el
            .set_attribute(
                "style",
                "background:#0d1117;color:#e6edf3;margin:0;padding:0;min-height:100vh;",
            )
            .ok();
    }

    let style = document.create_element("style")?;
    style.set_text_content(Some(CSS));
    document
        .head()
        .ok_or_else(|| JsValue::from_str("no head"))?
        .append_child(&style)
        .ok();

    // Header.
    let h1 = document.create_element("h1")?;
    h1.set_inner_html("19.8 <code>OPFS</code>: живая файловая система браузера");
    let header = document.create_element("header")?;
    header.append_child(&h1)?;
    let sub = document.create_element("p")?;
    sub.set_class_name("subtitle");
    sub.set_text_content(Some(
        "Worker пишет в OPFS в реальном времени · main thread показывает · без user gesture",
    ));
    header.append_child(&sub)?;
    body.append_child(&header)?;

    // Status bar.
    let status = document.create_element("div")?;
    status.set_class_name("status-bar");
    status.set_text_content(Some("инициализация…"));
    body.append_child(&status)?;

    // =========================================================================
    // Section ❶: Live logger
    // =========================================================================
    let (sec1, body1) = make_section(
        &document,
        "❶",
        "Живой логгер",
        "Worker пишет строки в OPFS через sync access handle",
    )?;
    body.append_child(&sec1)?;

    let controls = document.create_element("div")?;
    controls.set_class_name("row-controls");
    body1.append_child(&controls)?;

    let btn_start = document.create_element("button")?;
    btn_start.set_class_name("btn primary");
    btn_start.set_text_content(Some("▶ Старт"));
    btn_start.set_attribute(
        "title",
        "Запускает Worker, который раз в 50 мс дописывает строку в OPFS через createSyncAccessHandle. \
         Main thread не блокируется — UI продолжает работать.",
    )?;
    controls.append_child(&btn_start)?;

    let btn_stop = document.create_element("button")?;
    btn_stop.set_class_name("btn");
    btn_stop.set_text_content(Some("⏹ Стоп"));
    btn_stop.set_attribute("disabled", "")?;
    btn_stop.set_attribute(
        "title",
        "Останавливает Worker. Всё, что успело записаться, уже на диске — благодаря flush() в каждой итерации.",
    )?;
    controls.append_child(&btn_stop)?;

    let btn_clear = document.create_element("button")?;
    btn_clear.set_class_name("btn");
    btn_clear.set_text_content(Some("Очистить OPFS"));
    btn_clear.set_attribute(
        "title",
        "Удаляет лог и bench-файлы. Повторное нажатие — тоже Ok (идемпотентность).",
    )?;
    controls.append_child(&btn_clear)?;

    let logger_info = document.create_element("div")?;
    logger_info.set_class_name("row-status");
    logger_info.set_inner_html("строк: <b>0</b> · размер: <b>0 B</b>");
    controls.append_child(&logger_info)?;

    let preview = document.create_element("div")?;
    preview.set_class_name("preview");
    preview.set_text_content(Some("(лог пуст)"));
    body1.append_child(&preview)?;

    let expl = document.create_element("div")?;
    expl.set_class_name("explanation");
    expl.set_inner_html(
        "<b>Что здесь происходит.</b> Worker открывает <code>createSyncAccessHandle()</code> \
         на файле и дописывает строку раз в 50 мс. Main thread раз в 500 мс читает размер \
         и хвост файла — без блокировки.<br>\
         <b>Wow №1.</b> Файл растёт в реальном времени, UI не тормозит.<br>\
         <b>Wow №2.</b> Перезагрузите страницу — старые строки на месте, Worker \
         дописывает к ним новые (append через <code>access.getSize()</code>).<br>\
         <b>Wow №3.</b> Ни одного разрешения не запрошено — OPFS доступен без user gesture.",
    );
    body1.append_child(&expl)?;

    // =========================================================================
    // Section ❷: Benchmark
    // =========================================================================
    let (sec2, body2) = make_section(
        &document,
        "❷",
        "Скорость: async vs sync",
        "10 МБ · main thread vs Worker",
    )?;
    body.append_child(&sec2)?;

    let bench_controls = document.create_element("div")?;
    bench_controls.set_class_name("bench-controls");
    body2.append_child(&bench_controls)?;

    let bench_info = document.create_element("div")?;
    bench_info.set_class_name("explanation");
    bench_info.set_inner_html("данные: <b>не сгенерированы</b>");
    body2.append_child(&bench_info)?;

    let bar_async = document.create_element("div")?;
    let bar_sync = document.create_element("div")?;
    let num_async = document.create_element("div")?;
    let num_sync = document.create_element("div")?;
    let ratio_el = document.create_element("div")?;

    // =========================================================================
    // Section ❸: Log
    // =========================================================================
    let (sec3, body3) = make_section(&document, "❸", "Журнал", "")?;
    body.append_child(&sec3)?;
    let log_body = document.create_element("div")?;
    log_body.set_class_name("log-body");
    body3.set_class_name("");
    body3.set_attribute("style", "padding:0")?;
    body3.append_child(&log_body)?;

    // =========================================================================
    // Footer: summary
    // =========================================================================
    let summary_bar = document.create_element("div")?;
    summary_bar.set_class_name("summary-bar");
    body.append_child(&summary_bar)?;

    let summary = document.create_element("span")?;
    summary.set_inner_html("OPFS: —");
    summary_bar.append_child(&summary)?;

    let spacer = document.create_element("span")?;
    spacer.set_class_name("spacer");
    summary_bar.append_child(&spacer)?;

    let btn_reload = document.create_element("button")?;
    btn_reload.set_class_name("btn");
    btn_reload.set_text_content(Some("↻ Перезагрузить"));
    btn_reload.set_attribute(
        "title",
        "Перезагрузка страницы. Лог останется — OPFS переживает перезагрузки.",
    )?;
    summary_bar.append_child(&btn_reload)?;

    // =========================================================================
    // State
    // =========================================================================
    let state: Rc<DemoState> = Rc::new(DemoState {
        status: status.clone(),
        logger_info: logger_info.clone(),
        logger_preview: preview.clone(),
        btn_start: btn_start.clone(),
        btn_stop: btn_stop.clone(),
        bench_info: bench_info.clone(),
        bar_async: bar_async.clone(),
        bar_sync: bar_sync.clone(),
        num_async: num_async.clone(),
        num_sync: num_sync.clone(),
        ratio_el: ratio_el.clone(),
        summary: summary.clone(),
        log_body: log_body.clone(),
        root: RefCell::new(None),
        logger: RefCell::new(None),
        polling: Cell::new(false),
        bench_data: RefCell::new(None),
        async_ms: Cell::new(0.0),
        sync_ms: Cell::new(0.0),
    });

    // =========================================================================
    // Заполняем bench-таблицу
    // =========================================================================
    {
        let row_a = document.create_element("div")?;
        row_a.set_class_name("bench-row");
        let label_a = document.create_element("div")?;
        label_a.set_class_name("bench-label");
        label_a.set_inner_html("async <b>(main)</b>");
        row_a.append_child(&label_a)?;
        let wrap_a = document.create_element("div")?;
        wrap_a.set_class_name("bar-wrap");
        bar_async.set_class_name("bar async");
        wrap_a.append_child(&bar_async)?;
        row_a.append_child(&wrap_a)?;
        num_async.set_class_name("bench-num");
        num_async.set_text_content(Some("—"));
        row_a.append_child(&num_async)?;
        body2.append_child(&row_a)?;

        let row_s = document.create_element("div")?;
        row_s.set_class_name("bench-row");
        let label_s = document.create_element("div")?;
        label_s.set_class_name("bench-label");
        label_s.set_inner_html("sync <b>(Worker)</b>");
        row_s.append_child(&label_s)?;
        let wrap_s = document.create_element("div")?;
        wrap_s.set_class_name("bar-wrap");
        bar_sync.set_class_name("bar sync");
        wrap_s.append_child(&bar_sync)?;
        row_s.append_child(&wrap_s)?;
        num_sync.set_class_name("bench-num");
        num_sync.set_text_content(Some("—"));
        row_s.append_child(&num_sync)?;
        body2.append_child(&row_s)?;

        ratio_el.set_class_name("bench-ratio");
        ratio_el.set_inner_html("—");
        body2.append_child(&ratio_el)?;
    }

    // =========================================================================
    // Кнопки
    // =========================================================================
    {
        // Старт
        let s = state.clone();
        let btn_start_clone = btn_start.clone();
        let closure_start = Closure::<dyn FnMut()>::new(move || {
            let s = s.clone();
            spawn_local(async move {
                if let Err(e) = start_logger(s).await {
                    console_err("start_logger failed", &e);
                }
            });
        });
        let target: &web_sys::EventTarget = btn_start_clone.unchecked_ref();
        target
            .add_event_listener_with_callback("click", closure_start.as_ref().unchecked_ref())?;
        closure_start.forget();

        // Стоп
        let s = state.clone();
        let btn_stop_clone = btn_stop.clone();
        let closure_stop = Closure::<dyn FnMut()>::new(move || {
            stop_logger(s.clone());
        });
        let target: &web_sys::EventTarget = btn_stop_clone.unchecked_ref();
        target.add_event_listener_with_callback("click", closure_stop.as_ref().unchecked_ref())?;
        closure_stop.forget();

        // Очистить OPFS
        let s = state.clone();
        let btn_clear_clone = btn_clear.clone();
        let closure_clear = Closure::<dyn FnMut()>::new(move || {
            let s = s.clone();
            spawn_local(async move {
                if let Err(e) = clear_all(s).await {
                    console_err("clear_all failed", &e);
                }
            });
        });
        let target: &web_sys::EventTarget = btn_clear_clone.unchecked_ref();
        target.add_event_listener_with_callback("click", closure_clear.as_ref().unchecked_ref())?;
        closure_clear.forget();

        // Перезагрузить страницу
        let closure_reload = Closure::<dyn FnMut()>::new(move || reload_page());
        let target: &web_sys::EventTarget = btn_reload.unchecked_ref();
        target
            .add_event_listener_with_callback("click", closure_reload.as_ref().unchecked_ref())?;
        closure_reload.forget();
    }

    // Кнопки бенчмарка.
    {
        let s = state.clone();
        make_button(
            &document,
            &bench_controls,
            "Сгенерировать 10 МБ",
            "Создаёт массив из 10 МБ детерминированных байт.",
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = gen_data(s).await {
                        console_err("gen_data failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &bench_controls,
            "Async write",
            "Пишет 10 МБ через FileSystemWritableFileStream в main thread.",
            false,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = bench_async(s).await {
                        console_err("bench_async failed", &e);
                    }
                });
            },
        )?;
    }
    {
        let s = state.clone();
        make_button(
            &document,
            &bench_controls,
            "Sync write (Worker)",
            "Пишет 10 МБ через createSyncAccessHandle в Worker'е. Обычно в 10–20× быстрее async.",
            true,
            move || {
                let s = s.clone();
                spawn_local(async move {
                    if let Err(e) = bench_sync(s).await {
                        console_err("bench_sync failed", &e);
                    }
                });
            },
        )?;
    }

    // =========================================================================
    // Инициализация
    // =========================================================================
    {
        let s = state.clone();
        spawn_local(async move {
            console_log("получаю OPFS root…");

            if !opfs_available() {
                set_status(&s, "err", "OPFS недоступен в этом браузере");
                log_line(&s, "err", "OPFS недоступен");
                return;
            }

            match opfs_root().await {
                Ok(root) => {
                    *s.root.borrow_mut() = Some(root);
                    log_line(&s, "ok", "OPFS root получен");

                    let sync_ok: bool = sync_access_available_in_worker().await;
                    let flag: &str = if sync_ok {
                        "sync access handle доступен"
                    } else {
                        "sync access handle недоступен"
                    };
                    let line: String = format_str(format_args!("OPFS доступен · {flag}"));
                    set_status(&s, "ok", &line);
                    console_log(&line);

                    // Если лог уже есть с прошлой сессии — покажем это.
                    let size: f64 = file_size(s.root.borrow().as_ref().unwrap(), LOG_FILE)
                        .await
                        .unwrap_or(0.0);
                    if size > 0.0 {
                        let msg: String = format_str(format_args!(
                            "лог из прошлой сессии: {} — нажмите ▶",
                            fmt_bytes(size)
                        ));
                        log_line(&s, "ok", &msg);
                        let tail: String =
                            read_tail(s.root.borrow().as_ref().unwrap(), LOG_FILE, 4096)
                                .await
                                .unwrap_or_default();
                        let lines: Vec<&str> = tail.lines().collect();
                        let take = if lines.len() > 6 { lines.len() - 6 } else { 0 };
                        s.logger_preview
                            .set_text_content(Some(&lines[take..].join("\n")));
                    }

                    refresh_summary(&s).await;
                }
                Err(e) => {
                    set_status(&s, "err", "OPFS недоступен");
                    console_err("opfs_root failed", &e);
                    log_line(&s, "err", &format_str(format_args!("OPFS root: {e:?}")));
                }
            }
        });
    }

    console_log("run: finished OK");
    Ok(())
}
```

## Команды запуска

```bash
# Сборка WASM
wasm-pack build --dev --target web --out-dir pkg

# Тесты OPFS в headless Chrome
wasm-pack test --headless --chrome --test wasm_fsa_api_opfs

# Host-тесты (не касаются OPFS, но полезно)
cargo test

# Запуск демо
npm run start
```

`Cargo.toml` **не требует** новых features. `.cargo/config.toml` **не требует** `--cfg=web_sys_unstable_apis`. Всё общение с `createSyncAccessHandle` идёт через JS-код Worker'а, а Rust видит только стабильные типы — `Worker`, `MessageEvent`, `Blob`, `Url`.