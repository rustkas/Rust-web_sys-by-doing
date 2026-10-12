# 15.8. Ограничения Storage и синхронный API

## Обновлённая структура проекта

```text
hello-wasm02/
├── Cargo.toml
├── Cargo.lock
├── package.json
├── package-lock.json
├── vite.config.js
├── webdriver.json
├── .gitignore
├── LICENSE
├── README.md
├── build.rs
├── .cargo/
│   └── config.toml
├── src/
│   ├── lib.rs
│   ├── app.rs                      # +1 секция: «Ограничения» с 4 подсекциями
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # +4 функции: fill_to_quota, clear_prefix, measure_sync_block, current_origin
│   ├── value.rs
│   └── logging.rs
├── tests/
│   ├── common/
│   │   └── mod.rs
│   ├── pure_logic.rs
│   ├── wasm_logic.rs
│   ├── wasm_dom.rs
│   ├── wasm_events.rs
│   ├── wasm_js_objects.rs
│   ├── wasm_jsvalue.rs
│   ├── wasm_callbacks.rs
│   ├── wasm_async.rs
│   ├── wasm_async_utils.rs
│   ├── wasm_navigation.rs
│   └── wasm_storage.rs             # +12 тестов → 62 теста
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +2 правила для limits-demo и sync-demo
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/storage.rs` — **+4 функции**: `fill_to_quota` (заполнение до `QuotaExceededError`), `clear_prefix` (удаление ключей с префиксом), `measure_sync_block` (замер времени синхронной записи), `current_origin` (текущий origin). Плюс две структуры-результата: `FillResult`, `SyncBlockResult`.
- `src/app.rs` — **+1 секция «Ограничения»** с четырьмя подсекциями: типы, бинарь, квота, синхронная блокировка.
- `tests/wasm_storage.rs` — **+12 тестов** на квоту, префикс, синхронную запись, origin. Всего 62 теста.
- `web/styles/main.css` — **+2 правила** для стилизации подсекций с демонстрациями.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| Storage quotas and eviction (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria |
| `QuotaExceededError` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria#when_is_there_a_quota_exceeded_error |
| Origin (MDN) | https://developer.mozilla.org/en-US/docs/Glossary/Origin |
| Content Security Policy (MDN) | https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP |

---

## Ограничения Storage — почему это важно

Web Storage — простой API. Именно поэтому у него есть **жёсткие ограничения**. Знание этих ограничений определяет, **когда Storage подходит**, а когда нужно другое решение.

**Пять ограничений:**

**1. Только строки.** Разобрали в 15.5 и 15.6. Числа, булевы, структуры — через сериализацию. Бинарные данные — через JSON-массив или base64.

**2. ~5 МБ на origin.** Попытка записать больше — `QuotaExceededError`. Точный лимит зависит от браузера: Chrome — 10 МБ на origin, Firefox — 10 МБ, Safari — 5 МБ. Реальный лимит часто ниже из-за внутренних накладных расходов.

**3. Синхронный API.** Запись в Storage **блокирует главный поток**. Пока идёт запись 5 МБ — UI не отвечает. Для больших данных это критично. Решение — IndexedDB (глава 24), у него асинхронный API.

**4. Привязка к origin.** `protocol + host + port`. `http://localhost:5173` и `https://localhost:5173` — **разные origin**, разные хранилища. Даже смена порта с 5173 на 5174 даёт изолированный Storage.

**5. Уязвимость к XSS.** Любой JavaScript на том же origin читает все данные. Если атакующий внедрит свой скрипт — он получит доступ к токенам, паролям, персональным данным. **Никогда не храните секреты в Storage.**

**Плюс одно следствие:**

**6. Нет транзакций.** Две записи подряд — не атомарны. Между ними можно прочитать промежуточное состояние. В многопользовательских сценариях это может привести к race condition. В IndexedDB транзакции есть.

---

## `src/storage.rs`

Показаны **только новые части**. Остальное — из 15.7.

Добавьте в конец файла:

```rust
// ============================================================
// Оценка origin
// ============================================================

/// Возвращает текущий origin (protocol + host + port).
///
/// Origin — это тройка из протокола, хоста и порта. Storage
/// привязан к origin: данные, сохранённые на одном origin,
/// не видны на другом.
///
/// Возвращает строку вида `http://127.0.0.1:5173`. Если получить
/// origin не удалось — `"unknown"`.
pub fn current_origin() -> String {
    return web_sys::window()
        .and_then(|w| w.location().origin().ok())
        .unwrap_or_else(|| "unknown".to_string());
}

// ============================================================
// Демонстрация ограничений Storage
// ============================================================

/// Результат заполнения хранилища до квоты.
pub struct FillResult {
    /// Сколько ключей успели записать.
    pub keys_written: u32,
    /// Суммарный размер записанных данных в символах.
    pub total_chars: usize,
    /// Сообщение об ошибке, если она возникла.
    pub error_message: Option<String>,
    /// Достигли ли квоты. `true` — да, `false` — остановились по лимиту.
    pub hit_quota: bool,
}

/// Заполняет хранилище до срабатывания `QuotaExceededError`.
///
/// Пишет ключи `{prefix}:0`, `{prefix}:1`, ... со значением из
/// `chunk_size` символов. Останавливается, когда браузер возвращает
/// ошибку квоты, или когда записано `max_keys` ключей.
///
/// `max_keys` — защита от бесконечного цикла. Если браузер не
/// возвращает ошибку (например, лимит больше ожидаемого), мы всё
/// равно остановимся.
pub fn fill_to_quota(
    storage: &Storage,
    prefix: &str,
    chunk_size: usize,
    max_keys: u32,
) -> FillResult {
    let chunk = "x".repeat(chunk_size);
    let mut keys_written = 0u32;
    let mut total_chars = 0usize;

    loop {
        if keys_written >= max_keys {
            return FillResult {
                keys_written,
                total_chars,
                error_message: Some(format!(
                    "Остановлено после {} ключей (защита от бесконечного цикла)",
                    max_keys
                )),
                hit_quota: false,
            };
        }

        let key = format!("{}:{}", prefix, keys_written);
        match storage.set_item(&key, &chunk) {
            Ok(_) => {
                keys_written += 1;
                total_chars += key.len() + chunk.len();
            }
            Err(e) => {
                let message = e
                    .as_string()
                    .unwrap_or_else(|| format!("{:?}", e));
                return FillResult {
                    keys_written,
                    total_chars,
                    error_message: Some(message),
                    hit_quota: true,
                };
            }
        }
    }
}

/// Удаляет все ключи с заданным префиксом.
///
/// Возвращает количество удалённых ключей. Полезно для очистки
/// после тестов квоты: `clear_prefix(&ls, "quota-test")`.
pub fn clear_prefix(storage: &Storage, prefix: &str) -> u32 {
    let to_remove: Vec<String> = keys(storage)
        .into_iter()
        .filter(|k| k.starts_with(prefix))
        .collect();
    let count = to_remove.len() as u32;
    for k in to_remove {
        storage.remove_item(&k).ok();
    }
    return count;
}

/// Результат замера синхронной блокировки.
pub struct SyncBlockResult {
    /// Сколько миллисекунд заняла запись.
    pub duration_ms: f64,
    /// Сколько байт записано.
    pub bytes_written: usize,
    /// Сколько ключей записано.
    pub keys_written: u32,
}

/// Записывает `size_kb` КБ данных в storage, замеряя время.
///
/// Демонстрирует, что Web Storage — синхронный API: запись блокирует
/// главный поток на всё время операции. Чем больше данных, тем
/// заметнее пауза в UI.
///
/// Использует `js_sys::Date::now()` — миллисекунды с высокой точностью.
/// Останавливается, если запись провалилась.
pub fn measure_sync_block(
    storage: &Storage,
    size_kb: usize,
    prefix: &str,
) -> SyncBlockResult {
    let chunk_size = 1024; // ~1 КБ
    let chunk = "x".repeat(chunk_size);

    let start = js_sys::Date::now();

    let mut keys_written = 0u32;
    let mut bytes_written = 0usize;

    for i in 0..size_kb {
        let key = format!("{}:{}", prefix, i);
        match storage.set_item(&key, &chunk) {
            Ok(_) => {
                keys_written += 1;
                bytes_written += key.len() + chunk.len();
            }
            Err(_) => break,
        }
    }

    let end = js_sys::Date::now();

    return SyncBlockResult {
        duration_ms: end - start,
        bytes_written,
        keys_written,
    };
}
```

**Ключевые моменты:**

- **`current_origin`** использует `window.location.origin()` — встроенный метод браузера. Возвращает `protocol://host:port`.
- **`fill_to_quota`** возвращает `FillResult` со всей информацией: сколько записали, сколько символов, что за ошибка, достигли ли квоты.
- **Защита `max_keys`** предотвращает бесконечный цикл. Если браузер по какой-то причине не вернёт `QuotaExceededError`, мы всё равно остановимся.
- **`clear_prefix`** — точечная очистка. Не трогает ключи без префикса. Полезно для тестов: не сбрасывать всё хранилище, а удалить только свои ключи.
- **`measure_sync_block`** замеряет время через `js_sys::Date::now()`. Разница между началом и концом — миллисекунды синхронной блокировки.

**Что важно понять:**

- **`fill_to_quota` пишет блоками.** Каждый блок — отдельный ключ с `chunk_size` символов. Это моделирует реальную запись: данные почти всегда пишутся несколькими ключами, а не одним большим.
- **`clear_prefix` фильтрует по `starts_with`.** Использует нашу функцию `keys()` из 15.2.
- **`measure_sync_block` останавливается при первой ошибке.** Если запись не удалась — выходим из цикла, возвращаем то, что успели.

**Сводка `storage.rs` после 15.8:**

| Функция | Появилась в |
|---|---|
| `local_storage`, `session_storage` | 15.1 |
| `get`, `set`, `remove` | 15.1 |
| `length`, `key`, `keys`, `entries` | 15.2 |
| `has_key`, `clear` | 15.2 |
| `is_storage_writable` | 15.4 |
| `set_json`, `get_json` | 15.6 |
| `parse_storage_event`, `format_storage_change` | 15.7 |
| **`current_origin`** | **15.8** |
| **`fill_to_quota`, `clear_prefix`** | **15.8** |
| **`measure_sync_block`** | **15.8** |

**20 функций. Финальный размер модуля.**

---

## `src/app.rs`

Показаны **только новые части**. Остальное — из 15.7.

### Импорты

Добавьте в раздел `use crate::...`:

```rust
use crate::storage;
```

(если ещё не добавлен — уже должен быть из предыдущих секций).

### Новая секция 7: «Ограничения»

Добавьте **после секции 6** (События) и **перед** `std::mem::forget(closures)`:

```rust
    // ============================================================
    // СЕКЦИЯ 7: Ограничения Storage
    // ============================================================

    let section7 = build_section(
        &container,
        "7. Ограничения Storage и синхронный API",
    )?;

    // ---- Вводная информация ----
    let limits_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&limits_info), "limits-info")?;
    limits_info.set_text_content(Some(
        "• Только строки. Числа, bool, структуры — через сериализацию.\n\
         • ~5 МБ на origin. При превышении — QuotaExceededError.\n\
         • Синхронный API. Запись блокирует главный поток.\n\
         • Привязан к origin: protocol + host + port.\n\
         • Уязвим к XSS — любой JS на origin читает данные.\n\
         • Нет транзакций. Две записи подряд не атомарны.",
    ));
    dom::append(&section7, &limits_info)?;

    // ============================================================
    // 7a. Бинарные данные
    // ============================================================

    let binary_label = dom::create_element("h3")?;
    binary_label.set_text_content(Some("7a. Бинарные данные — через JSON-массив"));
    dom::append(&section7, &binary_label)?;

    let binary_hint = dom::create_paragraph(
        "Web Storage не принимает Uint8Array напрямую. Бинарные данные \
         нужно кодировать: либо как JSON-массив чисел, либо как base64-строку. \
         JSON-массив проще, но раздувает размер в 4–6 раз. Base64 компактнее \
         (на 33% больше исходного), но требует библиотеку.",
    )?;
    dom::add_class(&dom::as_html(&binary_hint), "hint")?;
    dom::append(&section7, &binary_hint)?;

    let binary_run_btn = dom::create_element("button")?;
    binary_run_btn.set_text_content(Some("Показать"));
    dom::append(&section7, &binary_run_btn)?;

    let binary_demo = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&binary_demo), "types-demo")?;
    dom::append(&section7, &binary_demo)?;

    {
        let binary_demo = binary_demo.clone();
        callbacks::listen(
            &binary_run_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        binary_demo.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                let mut lines = Vec::new();

                let bytes: Vec<u8> = vec![0xDE, 0xAD, 0xBE, 0xEF];
                lines.push(format!("Исходные байты: {:?}", bytes));
                lines.push(String::new());

                let json_array = serde_json::to_string(&bytes).unwrap();
                lines.push("Способ 1 — JSON-массив чисел:".to_string());
                lines.push(format!("  Строка: {}", json_array));
                lines.push(format!(
                    "  Размер: {} символов (вместо {} байт)",
                    json_array.len(),
                    bytes.len()
                ));
                lines.push(String::new());
                lines.push("Способ 2 — base64:".to_string());
                lines.push("  Base64 требует крейт base64 или btoa в JS".to_string());
                lines.push("  Размер: ~33% больше исходных байт".to_string());
                lines.push(String::new());

                let _ = storage::set(&ls, "demo:bytes", &json_array);
                let back = storage::get(&ls, "demo:bytes").unwrap_or_default();
                lines.push(format!("Из Storage прочитано: {}", back));

                let decoded: Vec<u8> =
                    serde_json::from_str(&back).unwrap_or_default();
                lines.push(format!("Десериализовано обратно: {:?}", decoded));
                lines.push(format!(
                    "Совпадает с исходным: {}",
                    decoded == bytes
                ));

                let text = lines.join("\n");
                binary_demo.set_inner_html("");
                if let Ok(pre) = dom::create_element("pre") {
                    pre.set_text_content(Some(text.as_str()));
                    let _ = dom::append(&binary_demo, &pre);
                }

                log!("Binary demo: {} bytes → {} chars", bytes.len(), json_array.len());
            },
            &closures,
        )?;
    }

    // ============================================================
    // 7b. QuotaExceededError
    // ============================================================

    let quota_label = dom::create_element("h3")?;
    quota_label.set_text_content(Some("7b. QuotaExceededError"));
    dom::append(&section7, &quota_label)?;

    let quota_hint = dom::create_paragraph(
        "Нажмите «Заполнить до квоты», чтобы записать большие строки \
         и получить реальный QuotaExceededError от браузера. Затем \
         «Очистить тестовые ключи» удалит только ключи с префиксом \
         quota-test, не трогая остальные.",
    )?;
    dom::add_class(&dom::as_html(&quota_hint), "hint")?;
    dom::append(&section7, &quota_hint)?;

    let fill_btn = dom::create_element("button")?;
    fill_btn.set_text_content(Some("Заполнить до квоты"));
    let cleanup_btn = dom::create_element("button")?;
    cleanup_btn.set_text_content(Some("Очистить тестовые ключи"));
    dom::append(&section7, &fill_btn)?;
    dom::append(&section7, &cleanup_btn)?;

    let quota_status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&quota_status), "usage-info")?;
    quota_status.set_text_content(Some("Ещё не запускалось."));
    dom::append(&section7, &quota_status)?;

    {
        let quota_status = quota_status.clone();
        callbacks::listen(
            &fill_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        quota_status.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                quota_status.set_text_content(Some(
                    "Заполняем localStorage (это может занять секунду)...",
                ));

                let result = storage::fill_to_quota(
                    &ls,
                    QUOTA_TEST_PREFIX,
                    100_000,
                    100,
                );

                let status: String = format_quota_result(&result);
                quota_status.set_text_content(Some(status.as_str()));

                log!(
                    "Quota test: hit_quota={}, keys={}, chars={}",
                    result.hit_quota,
                    result.keys_written,
                    result.total_chars
                );
            },
            &closures,
        )?;
    }

    {
        let quota_status = quota_status.clone();
        callbacks::listen(
            &cleanup_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let removed = storage::clear_prefix(&ls, QUOTA_TEST_PREFIX);
                let text: String = format!("Удалено {} тестовых ключей.", removed);
                quota_status.set_text_content(Some(text.as_str()));
                log!("Quota cleanup: removed {} keys", removed);
            },
            &closures,
        )?;
    }

    // ============================================================
    // 7c. Синхронная блокировка UI
    // ============================================================

    let sync_label = dom::create_element("h3")?;
    sync_label.set_text_content(Some("7c. Синхронный API — блокировка UI"));
    dom::append(&section7, &sync_label)?;

    let sync_hint = dom::create_paragraph(
        "Запись в Storage синхронна. Пока она идёт, главный поток \
         заблокирован. Нажмите кнопку — UI замрёт на несколько сотен \
         миллисекунд. Для больших данных это неприемлемо, используйте \
         IndexedDB с асинхронным API.",
    )?;
    dom::add_class(&dom::as_html(&sync_hint), "hint")?;
    dom::append(&section7, &sync_hint)?;

    let sync_run_btn = dom::create_element("button")?;
    sync_run_btn.set_text_content(Some("Записать 3 МБ (блокировка)"));
    let sync_cleanup_btn = dom::create_element("button")?;
    sync_cleanup_btn.set_text_content(Some("Очистить sync-ключи"));
    dom::append(&section7, &sync_run_btn)?;
    dom::append(&section7, &sync_cleanup_btn)?;

    let sync_status = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&sync_status), "usage-info")?;
    sync_status.set_text_content(Some("Ещё не запускалось."));
    dom::append(&section7, &sync_status)?;

    {
        let sync_status = sync_status.clone();
        callbacks::listen(
            &sync_run_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        sync_status.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                sync_status.set_text_content(Some(
                    "Записываем 3 МБ синхронно — UI заблокирован...",
                ));

                storage::clear_prefix(&ls, SYNC_TEST_PREFIX);

                let result = storage::measure_sync_block(
                    &ls,
                    3 * 1024,
                    SYNC_TEST_PREFIX,
                );

                let text: String = format!(
                    "Синхронная запись завершена.\n\
                     Записано ключей: {}\n\
                     Суммарный размер: ~{} KB\n\
                     Время: {:.1} мс\n\
                     \n\
                     Всё это время UI был заблокирован. \
                     Для больших данных используйте IndexedDB — у него \
                     асинхронный API и лимит в сотни МБ.",
                    result.keys_written,
                    result.bytes_written / 1024,
                    result.duration_ms
                );
                sync_status.set_text_content(Some(text.as_str()));

                log!(
                    "Sync block: {} ms, {} keys, {} bytes",
                    result.duration_ms,
                    result.keys_written,
                    result.bytes_written
                );
            },
            &closures,
        )?;
    }

    {
        let sync_status = sync_status.clone();
        callbacks::listen(
            &sync_cleanup_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let removed = storage::clear_prefix(&ls, SYNC_TEST_PREFIX);
                let text: String = format!("Удалено {} sync-ключей.", removed);
                sync_status.set_text_content(Some(text.as_str()));
            },
            &closures,
        )?;
    }

    // ============================================================
    // 7d. Привязка к origin
    // ============================================================

    let origin_label = dom::create_element("h3")?;
    origin_label.set_text_content(Some("7d. Привязка к origin"));
    dom::append(&section7, &origin_label)?;

    let origin_hint = dom::create_paragraph(
        "Storage привязан к origin: protocol + host + port. \
         Данные, сохранённые на одном origin, не видны на другом. \
         Даже смена порта (5173 → 5174) даёт отдельное хранилище. \
         Попробуйте запустить второй экземпляр приложения на другом порту.",
    )?;
    dom::add_class(&dom::as_html(&origin_hint), "hint")?;
    dom::append(&section7, &origin_hint)?;

    let origin_info = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&origin_info), "usage-info")?;

    let current_origin = storage::current_origin();
    let origin_text: String = format!(
        "Текущий origin: {}\n\
         Ключи в этом Storage привязаны к этому origin.\n\
         Откройте приложение на другом порту — данные будут отдельные.",
        current_origin
    );
    origin_info.set_text_content(Some(origin_text.as_str()));
    dom::append(&section7, &origin_info)?;
```

### Константы для префиксов

Добавьте **рядом с другими константами** в начале `app.rs`:

```rust
const QUOTA_TEST_PREFIX: &str = "quota-test";
const SYNC_TEST_PREFIX: &str = "sync-test";
```

### Функция `format_quota_result`

Добавьте **в конец файла**, рядом с другими helper-функциями:

```rust
/// Форматирует результат заполнения хранилища до квоты.
///
/// Вынесено в отдельную функцию, чтобы избежать проблемы
/// `rust-analyzer` с tail-выражениями `if/else` внутри замыканий.
fn format_quota_result(result: &storage::FillResult) -> String {
    if result.hit_quota {
        return format!(
            "QuotaExceededError!\n\
             Записано ключей: {}\n\
             Суммарный размер: ~{} символов (~{} MB)\n\
             Сообщение браузера: {}",
            result.keys_written,
            result.total_chars,
            result.total_chars / (1024 * 1024),
            result.error_message.as_deref().unwrap_or("")
        );
    }
    return format!(
        "Достигнут лимит безопасности.\n\
         Записано ключей: {}\n\
         Суммарный размер: ~{} символов\n\
         {}",
        result.keys_written,
        result.total_chars,
        result.error_message.as_deref().unwrap_or("")
    );
}
```

**Что демонстрирует секция 7:**

- **7a. Бинарные данные.** `Vec<u8>` кодируется в JSON-массив чисел. Показываем реальный размер: 4 байта → ~15 символов JSON. Плюс сравнение с base64.
- **7b. `QuotaExceededError`.** Реальный вызов `fill_to_quota`. Пользователь видит, как браузер упирается в лимит и возвращает ошибку.
- **7c. Синхронная блокировка.** Записываем 3 МБ с замером времени. UI замирает на несколько сотен миллисекунд.
- **7d. Origin.** Показываем текущий origin. Пользователь может открыть второй экземпляр на другом порту и убедиться, что данные разные.

**Почему `format_quota_result` вынесена.** Это тот же паттерн, что `make_render_inspector` и `format_storage_change`: длинный `if/else` с `format!` в каждой ветке. `rust-analyzer` спотыкается на tail-выражениях в замыкании. Вынесение в функцию решает проблему.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   Секция ограничений
   ============================================================ */

.limits-info {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.6;
    white-space: pre-wrap;
    background: #fff8f0;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #d97706;
    margin: 0.5rem 0 1rem 0;
}

.storage-section h3 {
    margin: 1rem 0 0.5rem 0;
    font-size: 0.95rem;
    color: #b7410e;
}

.usage-info {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    margin: 0.5rem 0;
    min-height: 2rem;
    max-height: 200px;
    overflow-y: auto;
}
```

**Что делает:**

- **`.limits-info`** — оранжевый блок с перечислением ограничений.
- **`.storage-section h3`** — стилизация подзаголовков внутри секции (7a, 7b, 7c, 7d).
- **`.usage-info`** — уже был, но напоминаем: это блок для вывода статуса.

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.7):

```rust
// ============================================================
// current_origin
// ============================================================

#[wasm_bindgen_test]
fn test_current_origin_is_valid() {
    let origin = storage::current_origin();

    assert!(!origin.is_empty(), "origin should not be empty");
    assert!(
        origin.starts_with("http"),
        "origin should start with http: {}",
        origin
    );
    assert_ne!(origin, "unknown");
}

#[wasm_bindgen_test]
fn test_current_origin_contains_port() {
    let origin = storage::current_origin();
    // В тестовом окружении (127.0.0.1:порт) origin содержит порт
    assert!(
        origin.contains(':'),
        "origin should contain port separator: {}",
        origin
    );
}

// ============================================================
// clear_prefix
// ============================================================

#[wasm_bindgen_test]
fn test_clear_prefix_removes_matching() {
    let ls = storage::local_storage().unwrap();

    storage::set(&ls, "test_prefix:a", "1").unwrap();
    storage::set(&ls, "test_prefix:b", "2").unwrap();
    storage::set(&ls, "other_key", "3").unwrap();

    let removed = storage::clear_prefix(&ls, "test_prefix:");
    assert_eq!(removed, 2);

    assert!(!storage::has_key(&ls, "test_prefix:a"));
    assert!(!storage::has_key(&ls, "test_prefix:b"));
    assert!(storage::has_key(&ls, "other_key"));

    storage::remove(&ls, "other_key").ok();
}

#[wasm_bindgen_test]
fn test_clear_prefix_no_matches() {
    let ls = storage::local_storage().unwrap();

    let removed = storage::clear_prefix(&ls, "definitely_no_such_prefix_xyz:");
    assert_eq!(removed, 0);
}

#[wasm_bindgen_test]
fn test_clear_prefix_empty_storage() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    let removed = storage::clear_prefix(&ls, "anything:");
    assert_eq!(removed, 0);

    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}

// ============================================================
// fill_to_quota
// ============================================================

#[wasm_bindgen_test]
fn test_fill_to_quota_respects_max_keys() {
    let ls = storage::local_storage().unwrap();

    // Маленький лимит: 5 ключей по 1000 символов
    let result = storage::fill_to_quota(&ls, "test_quota_small", 1000, 5);

    assert_eq!(result.keys_written, 5);
    assert!(!result.hit_quota);
    assert!(result.total_chars >= 5 * 1000);
    assert!(result.error_message.is_some());

    storage::clear_prefix(&ls, "test_quota_small");
}

#[wasm_bindgen_test]
fn test_fill_to_quota_max_zero() {
    let ls = storage::local_storage().unwrap();

    // max_keys = 0 — цикл не должен выполниться ни разу
    let result = storage::fill_to_quota(&ls, "test_quota_zero", 1000, 0);

    assert_eq!(result.keys_written, 0);
    assert!(!result.hit_quota);
    assert_eq!(result.total_chars, 0);
}

#[wasm_bindgen_test]
fn test_fill_to_quota_hits_real_quota() {
    let ls = storage::local_storage().unwrap();

    // Записываем блоками по 1 МБ. Реальная квота сработает
    // задолго до 200 ключей (200 МБ — намного больше 5–10 МБ лимита).
    let result = storage::fill_to_quota(&ls, "test_quota_real", 1_000_000, 200);

    // Либо упёрлись в реальную квоту, либо прошли все 200 ключей
    // (маловероятно, но возможно в браузере с большим лимитом)
    assert!(
        result.hit_quota || result.keys_written == 200,
        "should hit quota or reach max, got keys={} hit_quota={}",
        result.keys_written,
        result.hit_quota
    );

    assert!(result.keys_written > 0);

    storage::clear_prefix(&ls, "test_quota_real");
}

// ============================================================
// measure_sync_block
// ============================================================

#[wasm_bindgen_test]
fn test_measure_sync_block_writes_data() {
    let ls = storage::local_storage().unwrap();

    storage::clear_prefix(&ls, "test_sync");

    // Записываем 100 КБ — маленький объём, чтобы тест был быстрым
    let result = storage::measure_sync_block(&ls, 100, "test_sync");

    assert_eq!(result.keys_written, 100);
    assert!(result.bytes_written >= 100 * 1024);
    assert!(result.duration_ms >= 0.0);

    let removed = storage::clear_prefix(&ls, "test_sync");
    assert_eq!(removed, 100);
}

#[wasm_bindgen_test]
fn test_measure_sync_block_zero_size() {
    let ls = storage::local_storage().unwrap();

    let result = storage::measure_sync_block(&ls, 0, "test_sync_zero");

    assert_eq!(result.keys_written, 0);
    assert_eq!(result.bytes_written, 0);
    // Даже нулевая операция занимает какое-то время (0 или больше)
    assert!(result.duration_ms >= 0.0);
}

// ============================================================
// Демонстрация не-атомарности
// ============================================================

#[wasm_bindgen_test]
fn test_operations_are_not_atomic() {
    let ls = storage::local_storage().unwrap();

    let original: Vec<_> = storage::entries(&ls);
    storage::clear(&ls).ok();

    // Имитируем "транзакцию" из двух записей
    storage::set(&ls, "tx:a", "1").unwrap();

    // Между двумя set можно прочитать промежуточное состояние
    assert!(storage::has_key(&ls, "tx:a"));
    assert!(!storage::has_key(&ls, "tx:b"));

    storage::set(&ls, "tx:b", "2").unwrap();

    // Теперь видны обе
    assert!(storage::has_key(&ls, "tx:a"));
    assert!(storage::has_key(&ls, "tx:b"));

    storage::clear(&ls).ok();
    for (k, v) in original {
        storage::set(&ls, &k, &v).ok();
    }
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_current_origin_is_valid` | `current_origin` возвращает валидный origin |
| `test_current_origin_contains_port` | В тестовом окружении есть порт |
| `test_clear_prefix_removes_matching` | Удаляются только ключи с префиксом |
| `test_clear_prefix_no_matches` | Если совпадений нет — 0 |
| `test_clear_prefix_empty_storage` | На пустом хранилище — 0 |
| `test_fill_to_quota_respects_max_keys` | Достигает `max_keys` без квоты |
| `test_fill_to_quota_max_zero` | `max_keys = 0` — ничего не пишется |
| `test_fill_to_quota_hits_real_quota` | Реальное `QuotaExceededError` или 200 ключей |
| `test_measure_sync_block_writes_data` | 100 КБ записаны, время замерено |
| `test_measure_sync_block_zero_size` | 0 КБ — 0 ключей, время 0 |
| `test_operations_are_not_atomic` | Между двумя `set` видно промежуточное состояние |

**Почему `test_fill_to_quota_hits_real_quota` терпим к обоим исходам.** Chrome и Firefox имеют разные лимиты. Chrome ~10 МБ, Firefox ~10 МБ, Safari ~5 МБ. При записи блоками по 1 МБ ошибка квоты срабатывает на 5–10 ключах. Но если браузер имеет большой лимит — мы остановимся на `max_keys = 200`. Тест принимает оба исхода.

**Всего тестов:** 50 из 15.1–15.7 + 12 новых = **62 теста**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Прокрутите до секции 7.

**7a. Бинарные данные.** Нажмите «Показать». Увидите:

```
Исходные байты: [222, 173, 190, 239]

Способ 1 — JSON-массив чисел:
  Строка: [222,173,190,239]
  Размер: 17 символов (вместо 4 байт)

Способ 2 — base64:
  Base64 требует крейт base64 или btoa в JS
  Размер: ~33% больше исходных байт

Из Storage прочитано: [222,173,190,239]
Десериализовано обратно: [222, 173, 190, 239]
Совпадает с исходным: true
```

**7b. QuotaExceededError.** Нажмите «Заполнить до квоты». Через секунду:

```
QuotaExceededError!
Записано ключей: N
Суммарный размер: ~M символов (~K MB)
Сообщение браузера: ...
```

Затем нажмите «Очистить тестовые ключи». Увидите:

```
Удалено N тестовых ключей.
```

**7c. Синхронная блокировка.** Нажмите «Записать 3 МБ». UI замрёт на ~200–500 мс (в зависимости от скорости диска). После завершения:

```
Синхронная запись завершена.
Записано ключей: 3072
Суммарный размер: ~3072 KB
Время: 243.5 мс
```

**7d. Origin.** Показывает текущий origin — `http://127.0.0.1:5173`. Откройте приложение на другом порту (например, `vite --port 5174`) — увидите, что Storage там пуст.

### Тесты

```bash
# Только тесты Storage в Chrome
wasm-pack test --headless --chrome --test wasm_storage
```

```bash
# Только тесты Storage в Firefox
xvfb-run -a wasm-pack test --headless --firefox \
  --geckodriver ~/.cache/.wasm-pack/geckodriver-*/geckodriver \
  --test wasm_storage
```

```bash
# Все тесты проекта
npm run test:all
```

**Ожидаемый результат:** 62 теста `ok` в обоих браузерах.

---

## Что важно запомнить

**1. Только строки.** Уже знаем. Повторение — потому что это ограничение определяет всё остальное.

**2. ~5 МБ на origin.** Точный лимит зависит от браузера. `QuotaExceededError` — реальный сценарий, обрабатывайте его.

**3. Синхронный API.** Запись блокирует главный поток. Для больших данных — IndexedDB (глава 24).

**4. Привязка к origin.** `protocol + host + port`. Смена любого компонента — новое хранилище.

**5. Уязвимость к XSS.** Любой JS читает данные. **Не храните токены, пароли, персональные данные.**

**6. Нет транзакций.** Две записи не атомарны. Между ними можно прочитать промежуточное состояние.

**7. `fill_to_quota` с `max_keys`.** Защита от бесконечного цикла. Если браузер не возвращает ошибку — остановимся по лимиту.

**8. `clear_prefix` — точечная очистка.** Удаляет только ключи с префиксом. Полезно для тестов: не сбрасывает всё хранилище, а удаляет только свои ключи.

**9. `measure_sync_block` — замер блокировки.** Разница `Date::now()` до и после. Показывает реальную стоимость синхронной записи.

**10. `current_origin` — привязка к origin.** Возвращает `protocol://host:port`. Используется для демонстрации и для отладки.

**11. Не-атомарность — реальная проблема.** Если приложение пишет два связанных ключа (например, счётчик и метку времени), между записями может произойти чтение. В IndexedDB это решается транзакциями.

**12. Что дальше.** Web Storage не подходит для: больших данных, сложных запросов, бинарных данных, транзакций. Для всего этого есть IndexedDB (часть VII, глава 24). Cache API (глава 25) — для HTTP-ответов. Service Worker (глава 26) — для офлайн-работы.

---

## Что дальше

**15.9. Практический проект: панель управления Storage.** Финальная подглава. Мы соберём **все** секции из 15.1–15.8 в одно приложение:

- **9 секций:** Тема, Счётчики LS/SS, Инспектор, Типы, Заметки, События, Ограничения, Origin.
- **Warning-блок** при недоступности Storage.
- **Обработчик `storage`** на `window` для синхронизации вкладок.
- **Все 20 функций `storage.rs`** в действии.
- **Финальные стили** и структура `app.rs`.

Проект станет полноценным интерактивным стендом, где читатель может посмотреть каждое ограничение и каждую возможность Storage в работе. После 15.9 часть про Storage будет завершена — и мы перейдём к части V, где разберём Fetch, Streams и другие API, которые нужны почти каждому приложению.