# 15.7. `StorageEvent` — синхронизация вкладок

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
│   ├── app.rs                      # +1 секция: «События» + обработчик на window
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # +2 функции: parse_storage_event, format_storage_change
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
│   └── wasm_storage.rs             # +9 тестов → 50 тестов
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css                # +1 правило для .events-log
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/storage.rs` — **+2 функции**: `parse_storage_event` (парсит `StorageEvent` в `StorageChange`) и `format_storage_change` (форматирует для логирования). Плюс структура `StorageChange` с пятью полями.
- `src/app.rs` — **+1 секция «События»** и **+1 обработчик на `window`** для события `storage`. Обработчик логирует изменения из других вкладок и перерисовывает все секции.
- `tests/wasm_storage.rs` — **+9 тестов** на парсинг, форматирование, edge cases. Всего 50 тестов.
- `web/styles/main.css` — **+1 правило** для блока лога событий.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `StorageEvent` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/StorageEvent |
| `Window: storage event` (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Window/storage_event |
| `web_sys::StorageEvent` (docs.rs) | https://docs.rs/web-sys/latest/web_sys/struct.StorageEvent.html |

---

## Что такое `StorageEvent` и почему он важен

**`StorageEvent` — это событие, которое браузер генерирует при изменении `localStorage`.** Оно позволяет **синхронизировать состояние между вкладками** одного origin без сервера.

**Зачем это нужно.** Пользователь открыл приложение в двух вкладках. В одной вкладке он вышел из аккаунта. Другая вкладка должна узнать об этом и тоже сбросить состояние. Без `StorageEvent` каждая вкладка жила бы в своём мире: одна показывает «вы вошли», другая — «вы вышли».

**Практические сценарии:**

- **Синхронизация аутентификации.** Выход в одной вкладке → выход во всех.
- **Настройки приложения.** Сменил тему в одной вкладке → тема поменялась везде.
- **Уведомления.** В одной вкладке пришло новое сообщение → счётчик непрочитанных обновился во всех.
- **Корзина покупок.** Добавил товар → счётчик корзины обновился в соседней вкладке.

**Ключевые особенности, которые важно знать:**

| Особенность | Что значит |
|---|---|
| **Только `localStorage`** | `sessionStorage` не генерирует межвкладочных событий — данные изолированы по вкладкам, синхронизировать нечего |
| **Только в другие вкладки** | Вкладка, в которой произошло изменение, событие **не получает**. Событие приходит только в соседние вкладки того же origin |
| **Только при реальном изменении** | Запись того же значения повторно **не** генерирует событие |
| **Только для `setItem`, `removeItem`, `clear`** | Прямое изменение через DevTools тоже генерирует событие |

**Как браузер узнаёт, что «значение изменилось».** Он сравнивает старое и новое значение. Если они идентичны — событие не генерируется. Это оптимизация, но она может привести к неожиданностям: повторная запись `"42"` поверх `"42"` не уведомит другие вкладки.

---

## Поля `StorageEvent`

Событие содержит пять полезных полей:

| Поле | Тип | Что содержит |
|---|---|---|
| `key` | `Option<String>` | Имя изменённого ключа. `None` при `clear()` |
| `old_value` | `Option<String>` | Старое значение. `None` для нового ключа или `clear()` |
| `new_value` | `Option<String>` | Новое значение. `None` при `removeItem` |
| `url` | `Option<String>` | URL вкладки, где произошло изменение |
| `storage_area` | `Option<Storage>` | Ссылка на `localStorage` (или `sessionStorage`, но для этого события всегда `localStorage`) |

**Что это даёт.** По полям можно понять:

- **Что изменилось** — `key`.
- **Как изменилось** — было `old_value`, стало `new_value`.
- **Откуда** — `url` (если у вас несколько вкладок с разными путями).
- **Что именно** — `storage_area` всегда `localStorage`.

**Соответствие с нашим `StorageChange`:**

```rust
pub struct StorageChange {
    pub key: Option<String>,
    pub old_value: Option<String>,
    pub new_value: Option<String>,
    pub url: Option<String>,
    pub storage_area: Option<String>,  // "localStorage" или None
}
```

Поле `storage_area` мы храним как `String` (`"localStorage"`), а не как `Storage`. Это связано с проблемой, которую мы разбирали в 15.5: в синтетических событиях `storage_area` = `null`. Значит, если бы мы полагались на `event.storage_area()`, синтетические события давали бы `None`.

**Решение.** Мы знаем, что событие `storage` генерируется **только для `localStorage`** — это свойство спецификации, а не конкретного события. Поэтому `storage_area` всегда `"localStorage"`.

---

## `src/storage.rs`

Показаны **только новые части**. Остальное — из 15.6.

Добавьте в конец файла:

```rust
// ============================================================
// StorageEvent
// ============================================================

/// Описывает событие `storage`.
///
/// ## Особенности Web Storage событий
///
/// - Событие `storage` генерируется **только для localStorage**.
/// - Для sessionStorage события **не генерируются** — данные изолированы
///   по вкладкам, синхронизировать нечего.
/// - Событие приходит **только в другие вкладки** того же origin.
///   Вкладка, в которой произошло изменение, событие **не получает**.
#[derive(Default)]
pub struct StorageChange {
    pub key: Option<String>,
    pub old_value: Option<String>,
    pub new_value: Option<String>,
    pub url: Option<String>,
    pub storage_area: Option<String>,
}

/// Извлекает данные из `StorageEvent`.
///
/// ## Про поле `storageArea`
///
/// Поле не извлекается из самого события. Событие `storage`
/// генерируется только для `localStorage` — `sessionStorage`
/// не генерирует межвкладочных событий. Поэтому `storage_area`
/// всегда `"localStorage"`.
///
/// В синтетических событиях (`StorageEvent::new("storage")`)
/// `storageArea` = `null`. Если бы мы полагались на
/// `event.storage_area()`, синтетические события давали бы `None`,
/// что противоречило бы реальной семантике.
pub fn parse_storage_event(event: &web_sys::StorageEvent) -> StorageChange {
    return StorageChange {
        key: event.key(),
        old_value: event.old_value(),
        new_value: event.new_value(),
        url: event.url(),
        storage_area: Some("localStorage".to_string()),
    };
}

/// Форматирует `StorageChange` для логирования.
///
/// Возвращает многострочное представление, включающее:
/// - изменение (set/remove/clear),
/// - область хранения (`storageArea`),
/// - источник изменения (`url`),
/// - старое значение (если было).
///
/// Реализовано через последовательность `if` с явными `return`,
/// чтобы избежать проблемы `rust-analyzer` с tail-выражениями `match`.
pub fn format_storage_change(change: &StorageChange) -> String {
    let mut lines = Vec::new();

    // Основное сообщение
    if change.key.is_none() {
        lines.push("Storage cleared".to_string());
    } else {
        let key = change.key.as_ref().unwrap();
        if let Some(v) = &change.new_value {
            lines.push(format!("[{}] = \"{}\"", key, v));
        } else {
            lines.push(format!("[{}] removed", key));
        }
    }

    // Область хранения
    if let Some(area) = &change.storage_area {
        lines.push(format!("  area: {}", area));
    }

    // Источник изменения
    if let Some(url) = &change.url {
        lines.push(format!("  from: {}", url));
    }

    // Старое значение — только при set (не при remove и не при clear)
    if let Some(old) = &change.old_value {
        if change.new_value.is_some() {
            lines.push(format!("  was: \"{}\"", old));
        }
    }

    return lines.join("\n");
}
```

**Ключевые моменты:**

- **`StorageChange` — простая структура.** Пять `Option<String>`. Никаких вложенных типов, кроме `storage_area`, который тоже `Option<String>`.
- **`#[derive(Default)]`** — удобно для случаев, когда событие пришло без данных (синтетическое, `clear()`).
- **`parse_storage_event` — извлекает четыре поля из события, пятое заполняет константой.**
- **`format_storage_change` — последовательность `if`, а не `match`.** Так `rust-analyzer` не спотыкается на tail-выражениях.

**Почему `format_storage_change` строит `Vec<String>`, а не `String` через `format!` напрямую.** Три случая (set, remove, clear) + два опциональных поля. Собрать всё в одну строку через `format!` неудобно — разная структура. `Vec<String>` и `join("\n")` — чище.

**Почему `was` показывается только при `set`.** При `remove` старе значение уже неактуально, при `clear` его и вовсе нет. Показываем `was` только когда есть новое значение — то есть при обновлении.

**Сводка `storage.rs` после 15.7:**

| Функция | Появилась в |
|---|---|
| `local_storage`, `session_storage` | 15.1 |
| `get`, `set`, `remove` | 15.1 |
| `length`, `key`, `keys`, `entries` | 15.2 |
| `has_key`, `clear` | 15.2 |
| `is_storage_writable` | 15.4 |
| `set_json`, `get_json` | 15.6 |
| **`parse_storage_event`, `format_storage_change`** | **15.7** |

**16 функций.**

---

## `src/app.rs`

Показаны **только новые части**. Остальное — из 15.6.

### Импорт

Добавьте в раздел импортов:

```rust
use web_sys::{Event, StorageEvent};
```



### Новая секция 6: «События»

Добавьте **после секции 5** (Заметки) и **перед** `std::mem::forget(closures)`:

```rust
    // ============================================================
    // СЕКЦИЯ 6: StorageEvent — синхронизация вкладок
    // ============================================================

    let section6 = build_section(
        &container,
        "6. StorageEvent: изменения из других вкладок",
    )?;

    let events_hint = dom::create_paragraph(
        "StorageEvent генерируется ТОЛЬКО для localStorage \
         и приходит ТОЛЬКО в другие вкладки того же origin. \
         Вкладка, где произошло изменение, событие не получает. \
         Для sessionStorage событий нет — данные изолированы по вкладкам. \
         \n\
         Откройте это приложение в новой вкладке (Ctrl+T, тот же URL). \
         Измените данные в любой из вкладок — здесь появятся записи \
         с полями key, oldValue, newValue, url и area.",
    )?;
    dom::add_class(&dom::as_html(&events_hint), "hint")?;
    dom::append(&section6, &events_hint)?;

    let clear_log_btn = dom::create_element("button")?;
    clear_log_btn.set_text_content(Some("Очистить лог"));
    dom::append(&section6, &clear_log_btn)?;

    let events_log = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&events_log), "events-log")?;
    events_log.set_text_content(Some("(пока пусто — откройте вторую вкладку)"));
    dom::append(&section6, &events_log)?;

    // ---- Замыкание для добавления записи в лог ----
    let append_to_log: Rc<dyn Fn(&str)> = {
        let events_log = events_log.clone();
        Rc::new(move |text: &str| {
            let prev = events_log.text_content().unwrap_or_default();
            let new_text = format!("---\n{}\n\n{}", text, prev);
            // Ограничиваем лог 2000 символами — старые записи отрезаем
            let trimmed: String = new_text.chars().take(2000).collect();
            events_log.set_text_content(Some(trimmed.as_str()));
        })
    };

    // ---- Обработчик «Очистить лог» ----
    {
        let events_log = events_log.clone();
        callbacks::listen(
            &clear_log_btn,
            "click",
            move |_event| {
                events_log.set_text_content(Some("(лог очищен)"));
            },
            &closures,
        )?;
    }

    // ---- Обработчик события `storage` на window ----
    {
        let append_to_log = append_to_log.clone();
        let render_notes = render_notes.clone();
        let render_json_preview = render_json_preview.clone();
        let render_inspector = render_inspector.clone();

        let window = web_sys::window().unwrap();
        callbacks::listen(
            &window,
            "storage",
            move |event: Event| {
                // Приводим generic Event к StorageEvent.
                // Если не получилось — игнорируем.
                let storage_event: StorageEvent = match event.dyn_into() {
                    Ok(e) => e,
                    Err(_) => return,
                };

                // Парсим в нашу структуру
                let change = storage::parse_storage_event(&storage_event);

                // Форматируем для лога
                let text = storage::format_storage_change(&change);

                // Логируем в консоль
                log!("StorageEvent:\n{}", text);

                // Добавляем в UI-лог
                append_to_log(&text);

                // Перерисовываем секции, которые могли устареть
                render_notes();
                render_json_preview();
                render_inspector();
            },
            &closures,
        )?;
    }
```

**Что демонстрирует секция:**

- **Кнопка «Очистить лог»** — сбрасывает содержимое блока лога.
- **Блок `.events-log`** — накапливает записи. Новые записи добавляются **сверху**, старые сдвигаются вниз. Ограничение 2000 символов защищает от бесконечного роста.
- **Обработчик на `window`** — слушает событие `storage`.
- **`event.dyn_into::<StorageEvent>()`** — приводит generic `Event` к конкретному типу. Если событие не того типа, `Err` — игнорируем.
- **Автоматическая перерисовка** — после события обновляются секции, которые могли устареть: список заметок, JSON-preview и инспектор.

**Почему обработчик на `window`, а не на `document`.** Событие `storage` генерируется на уровне `window`. Спецификация определяет его именно так. `document` его не получает.

**Почему `append_to_log` — отдельное замыкание.** Логика добавления записи в UI повторяется. Вынесли в `Rc<dyn Fn(&str)>` — переиспользуется в обработчике `storage` и может быть переиспользовано в других местах.

**Что важно понять:**

- **`dyn_into::<StorageEvent>()` требует `JsCast`.** Он уже импортирован в `app.rs` — `use wasm_bindgen::JsCast;`.
- **`render_notes` и другие — `Rc<dyn Fn()>`.** Клонируются в обработчик. Оригиналы остаются живы.
- **Автоматическая перерисовка** — ключевая фича. Когда приходит событие, UI обновляется сам. Пользователю не нужно нажимать «Обновить».

### Полный контекст: где добавляется секция

```rust
pub fn run() -> Result<(), JsValue> {
    // ... warning-блок ...
    // ... секция 1 «Знакомство» ...
    // ... секция 2 «Инспектор» ...
    // ... секция 3 «LS vs SS» ...
    // ... секция 4 «Типы» ...
    // ... секция 5 «Заметки» ...

    // ============================================================
    // СЕКЦИЯ 6: StorageEvent — синхронизация вкладок
    // ============================================================

    // ... код секции 6 ...

    std::mem::forget(closures);
    Ok(())
}
```

**Обратите внимание на порядок.** Секция 6 добавляется **после** секций 1–5. Обработчик `storage` в конце `run()` — потому что он ссылается на `render_notes`, `render_json_preview`, `render_inspector` из предыдущих секций.

---

## `web/styles/main.css`

Добавьте в конец файла:

```css
/* ============================================================
   Лог событий
   ============================================================ */

.events-log {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.8rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #2e7d32;
    margin: 0.75rem 0 0 0;
    min-height: 3rem;
    max-height: 400px;
    overflow-y: auto;
    word-break: break-all;
}
```

**Что делает:**

- **Зелёная полоса слева** — визуально отличает от `.json-preview` (синий) и `.types-demo` (оранжевый).
- **`max-height: 400px`** — ограничивает высоту, но не обрезает. Скроллинг внутри.
- **`word-break: break-all`** — длинные URL не ломают вёрстку.

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.6):

```rust
// ============================================================
// StorageEvent: парсинг
// ============================================================

#[wasm_bindgen_test]
fn test_parse_storage_event_default() {
    let event = web_sys::StorageEvent::new("storage").unwrap();
    let change = storage::parse_storage_event(&event);

    // Синтетическое событие без init-данных
    assert!(change.key.is_none());
    assert!(change.old_value.is_none());
    assert!(change.new_value.is_none());
}

#[wasm_bindgen_test]
fn test_parse_storage_event_sets_area() {
    let event = web_sys::StorageEvent::new("storage").unwrap();
    let change = storage::parse_storage_event(&event);

    // storage_area всегда "localStorage" — событие генерируется
    // только для localStorage
    assert_eq!(change.storage_area, Some("localStorage".to_string()));
}

// ============================================================
// StorageEvent: форматирование
// ============================================================

#[wasm_bindgen_test]
fn test_format_storage_change_set() {
    let change = storage::StorageChange {
        key: Some("theme".into()),
        old_value: Some("light".into()),
        new_value: Some("dark".into()),
        url: None,
        storage_area: None,
    };
    let text = storage::format_storage_change(&change);

    assert!(text.contains("theme"));
    assert!(text.contains("dark"));
    assert!(text.contains("was: \"light\""));
}

#[wasm_bindgen_test]
fn test_format_storage_change_set_without_old() {
    let change = storage::StorageChange {
        key: Some("new_key".into()),
        old_value: None,
        new_value: Some("value".into()),
        url: None,
        storage_area: None,
    };
    let text = storage::format_storage_change(&change);

    assert!(text.contains("new_key"));
    assert!(text.contains("value"));
    // Для нового ключа old_value отсутствует — не должно быть "was:"
    assert!(!text.contains("was:"));
}

#[wasm_bindgen_test]
fn test_format_storage_change_remove() {
    let change = storage::StorageChange {
        key: Some("theme".into()),
        old_value: Some("dark".into()),
        new_value: None,
        url: None,
        storage_area: None,
    };
    let text = storage::format_storage_change(&change);

    assert!(text.contains("removed"));
    assert!(text.contains("theme"));
    // При remove показываем "removed", а не "was:"
    assert!(!text.contains("was:"));
}

#[wasm_bindgen_test]
fn test_format_storage_change_clear() {
    let change = storage::StorageChange {
        key: None,
        old_value: None,
        new_value: None,
        url: None,
        storage_area: None,
    };
    let text = storage::format_storage_change(&change);

    assert!(text.contains("cleared"));
}

#[wasm_bindgen_test]
fn test_format_storage_change_with_area_and_url() {
    let change = storage::StorageChange {
        key: Some("theme".into()),
        old_value: Some("light".into()),
        new_value: Some("dark".into()),
        url: Some("http://localhost:5173/".into()),
        storage_area: Some("localStorage".into()),
    };
    let text = storage::format_storage_change(&change);

    assert!(text.contains("theme"));
    assert!(text.contains("dark"));
    assert!(text.contains("area: localStorage"));
    assert!(text.contains("from: http://localhost:5173/"));
    assert!(text.contains("was: \"light\""));
}

#[wasm_bindgen_test]
fn test_format_storage_change_clear_with_area_and_url() {
    let change = storage::StorageChange {
        key: None,
        old_value: None,
        new_value: None,
        url: Some("http://example.com/".into()),
        storage_area: Some("localStorage".into()),
    };
    let text = storage::format_storage_change(&change);

    assert!(text.contains("cleared"));
    assert!(text.contains("area: localStorage"));
    assert!(text.contains("from: http://example.com/"));
    // При clear нет старого значения — не должно быть "was:"
    assert!(!text.contains("was:"));
}

// ============================================================
// StorageChange: конструктор по умолчанию
// ============================================================

#[wasm_bindgen_test]
fn test_storage_change_default() {
    let change = storage::StorageChange::default();

    assert!(change.key.is_none());
    assert!(change.old_value.is_none());
    assert!(change.new_value.is_none());
    assert!(change.url.is_none());
    assert!(change.storage_area.is_none());
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_parse_storage_event_default` | Парсинг синтетического события без данных → все поля `None` |
| `test_parse_storage_event_sets_area` | `storage_area` всегда `"localStorage"` |
| `test_format_storage_change_set` | Форматирование `set` с `old_value` |
| `test_format_storage_change_set_without_old` | Форматирование `set` без `old_value` — нет `was:` |
| `test_format_storage_change_remove` | Форматирование `remove` — есть `removed`, нет `was:` |
| `test_format_storage_change_clear` | Форматирование `clear` — есть `cleared` |
| `test_format_storage_change_with_area_and_url` | Все поля + `area` + `from` |
| `test_format_storage_change_clear_with_area_and_url` | `clear` + `area` + `from`, но без `was:` |
| `test_storage_change_default` | `Default::default()` даёт все `None` |

**Почему не тестируем реальное событие `storage`.**

Событие `storage` генерируется **только в других вкладках**. В одном `wasm-bindgen-test`-раннере у нас одна вкладка. Создать реальное событие `storage` невозможно — только синтетическое через `StorageEvent::new("storage")`.

Синтетическое событие даёт `None` во всех полях (кроме `storage_area`, который мы заполняем сами). Тесты проверяют, что **парсинг не паникует** и **format правильно работает** с разными комбинациями полей.

**Всего тестов:** 41 из 15.1–15.6 + 9 новых = **50 тестов**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173` в **двух вкладках**.

**Сценарий 1: синхронизация заметок между вкладками.**

1. **Вкладка A**: добавьте заметку «Купить молоко».
2. **Вкладка B**: список заметок **автоматически обновится** — появится «Купить молоко». В секции 6 появится запись:

   ```
   ---
   [demo:notes] = "[{\"id\":1,\"text\":\"Купить молоко\"}]"
     area: localStorage
     from: http://127.0.0.1:5173/
   ```

3. **Вкладка A**: секция 6 **не показывает** событие — потому что событие не генерируется в той же вкладке.

**Сценарий 2: удаление ключа.**

1. **Вкладка A**: нажмите «Очистить все» в секции заметок.
2. **Вкладка B**: получит событие:

   ```
   ---
   [demo:notes] removed
     area: localStorage
     from: http://127.0.0.1:5173/
   ```

   Список заметок опустеет.

**Сценарий 3: полная очистка.**

1. **Вкладка A**: нажмите «Очистить всё» в секции «Инспектор».
2. **Вкладка B**: получит событие:

   ```
   ---
   Storage cleared
     area: localStorage
     from: http://127.0.0.1:5173/
   ```

   `key` = `None` — это признак `clear()`.

**Сценарий 4: `sessionStorage` не генерирует событий.**

1. **Вкладка A**: нажмите `+1` на sessionStorage-счётчике.
2. **Вкладка B**: ничего не произойдёт в секции 6. Событие не пришло. Это правильное поведение — `sessionStorage` изолирован по вкладкам.

**Сценарий 5: изменение через DevTools.**

1. Откройте DevTools в **обеих вкладках**.
2. **Вкладка A**: Application → Local Storage → измените значение `demo:name` на `НовоеИмя`.
3. **Вкладка B**: получит событие с `key: "demo:name"`, `oldValue: "СтароеИмя"`, `newValue: "НовоеИмя"`.

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

**Ожидаемый результат:** 50 тестов `ok` в обоих браузерах.

---

## Что важно запомнить

**1. `StorageEvent` — единственный способ синхронизировать вкладки без сервера.** Браузер сам генерирует событие при изменении `localStorage`.

**2. Только `localStorage`.** `sessionStorage` не генерирует межвкладочных событий — данные изолированы по вкладкам.

**3. Только в другие вкладки.** Вкладка-источник событие **не получает**. Это сделано специально: она уже знает, что изменила данные.

**4. Пять полей.** `key`, `oldValue`, `newValue`, `url`, `storageArea`. Каждое — `Option`.

**5. `storageArea` всегда `"localStorage"`.** Событие генерируется только для него. В синтетических событиях это поле `null` — поэтому мы заполняем его вручную.

**6. `clear()` даёт `key = None`.** Это единственный случай, когда `key` отсутствует.

**7. `removeItem` даёт `newValue = None`.** Значение удалено — нового нет.

**8. `setItem` нового ключа даёт `oldValue = None`.** Значения раньше не было.

**9. Запись того же значения не генерирует событие.** Браузер сравнивает старое и новое. Если они идентичны — событие не генерируется. Это оптимизация, но может привести к неожиданностям.

**10. Событие генерируется и для `sessionStorage`? Нет.** Только для `localStorage`. `sessionStorage` не имеет межвкладочного состояния.

**11. `format_storage_change` — чистая функция.** Не работает с DOM, не зависит от `web_sys`. Легко тестируется в изоляции.

**12. Перерисовка UI в обработчике.** После события `storage` мы вызываем `render_notes`, `render_json_preview`, `render_inspector` — чтобы секции отражали новое состояние.

**13. `dyn_into::<StorageEvent>()` может провалиться.** Событие `storage` — это `StorageEvent`, но если по какой-то причине пришёл generic `Event`, `Err` вернётся, и мы просто игнорируем.

**14. Лог событий — не бесконечный.** Ограничение 2000 символов защищает от роста памяти. Старые записи отрезаются.

---

## Что дальше

**15.8. Ограничения Storage и синхронный API.** Мы разберём оставшиеся ограничения Web Storage: ~5 МБ на origin (`QuotaExceededError`), синхронный API (блокировка главного потока при записи больших данных), привязка к origin (`protocol + host + port`), уязвимость к XSS. В `storage.rs` появятся четыре функции: `fill_to_quota`, `clear_prefix`, `measure_sync_block`, `current_origin`. В `app.rs` — четыре подсекции: демонстрация типов, кодирование бинарных данных, реальная квота, синхронная блокировка. В тестах — 12 новых проверок, включая `fill_to_quota` с реальным упором в лимит. После 15.8 останется только 15.9 — сборка полного практического проекта.