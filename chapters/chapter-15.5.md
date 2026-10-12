# 15.5. Сериализация: строки, числа, булевы

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
│   ├── app.rs                      # +1 секция: «Типы»
│   ├── async_utils.rs
│   ├── callbacks.rs
│   ├── dom.rs
│   ├── events.rs
│   ├── form.rs
│   ├── js_objects.rs
│   ├── navigation.rs
│   ├── notifier.rs
│   ├── storage.rs                  # не меняется
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
│   └── wasm_storage.rs             # +6 тестов → 31 тест
├── web/
│   ├── index.html
│   ├── main.js
│   ├── styles/
│   │   └── main.css
│   └── assets/
│       └── favicon.ico
├── pkg/
├── target/
└── node_modules/
```

**Комментарии к структуре:**

- `src/storage.rs` — **не меняется**. Полного API из 15.1–15.4 достаточно. Сериализация — работа на уровне вызывающего кода, а не обёрток над `Storage`.
- `src/app.rs` — **дополняется секцией 4 «Типы»**. Демонстрация того, что Storage хранит только строки. Данные **остаются в localStorage** — пользователь открывает DevTools и видит шесть ключей.
- `tests/wasm_storage.rs` — **+6 тестов** на числа, булевы, edge cases. Всего 31 тест.

---

## Ссылки для детального изучения

| Тема | Ссылка |
|---|---|
| `str::parse` (Rust std) | https://doc.rust-lang.org/std/primitive.str.html#method.parse |
| `ToString` (Rust std) | https://doc.rust-lang.org/std/string/trait.ToString.html |
| `FromStr` (Rust std) | https://doc.rust-lang.org/std/str/trait.FromStr.html |
| Rust type conversions | https://doc.rust-lang.org/rust-by-example/types/cast.html |

---

## Почему это важная тема

**Web Storage хранит только строки.** Это ограничение заложено в спецификации: `get_item` возвращает `Option<String>`, `set_item` принимает `&str`. Никаких чисел, булевых значений, объектов, массивов.

Это фундаментальное ограничение, и оно определяет, как приложения работают с Storage. Когда вы сохраняете счётчик (`42`), в браузере лежит **строка** `"42"`. Когда вы её читаете, вы получаете **строку** `"42"`. Никакой автоматической конвертации не происходит.

**Что это значит на практике:**

- **Числа** нужно преобразовывать вручную: `to_string()` при записи, `parse()` при чтении.
- **Булевы значения** — то же самое: `to_string()` даёт `"true"`/`"false"`, `parse::<bool>()` возвращает их обратно.
- **Объекты и массивы** требуют сериализации в JSON. Это тема следующей подглавы (15.6).

**Почему Storage так устроен.** Исторически Web Storage появился как простой key-value store для строк. Дизайнеры спецификации выбрали минимализм: строки — универсальное представление, из которого можно собрать что угодно. Если нужны типы — сериализуйте. Если нужны индексы — используйте IndexedDB.

---

## Как работать с числами и булевыми

### Числа

```rust
// Сохранить
let n: i32 = 42;
storage::set(&ls, "counter", &n.to_string())?;
// В хранилище лежит строка "42"

// Прочитать
let raw = storage::get(&ls, "counter");  // Some("42")
let n: i32 = raw.unwrap().parse()?;      // 42
```

**Что может пойти не так:**

- **Значение не число.** Если в хранилище оказалось `"hello"`, `parse::<i32>()` вернёт `Err`. Обрабатывайте `Result`.
- **Разные числовые типы.** `parse::<i32>()` не работает для `"3.14"`. Для float — `parse::<f64>()`.
- **Округление float.** `f64::to_string()` даёт 15–17 значащих цифр. Для большинства задач этого достаточно, но для точных финансовых вычислений используйте целые числа (копейки как `i64`).
- **Переполнение.** `i8::parse("300")` вернёт ошибку.

### Булевы значения

```rust
// Сохранить
let flag: bool = true;
storage::set(&ls, "enabled", &flag.to_string())?;
// В хранилище лежит строка "true"

// Прочитать
let raw = storage::get(&ls, "enabled");  // Some("true")
let flag: bool = raw.unwrap().parse()?;  // true
```

**Что важно знать:**

- `bool::to_string()` даёт ровно `"true"` или `"false"`.
- `bool::parse()` принимает **только** эти две строки. `"TRUE"`, `"1"`, `"yes"` — ошибка.
- Если нужен другой формат (например, `"1"`/`"0"`), конвертируйте вручную.

### Сравнение типов

| Rust | `to_string()` | `parse::<T>()` |
|---|---|---|
| `i32` | `"42"`, `"-7"` | принимает цифры, знак |
| `u32` | `"42"` | принимает только неотрицательные |
| `f64` | `"3.14"`, `"1e10"` | принимает точку, экспоненту |
| `bool` | `"true"`, `"false"` | принимает только эти две строки |
| `String` | сама строка | не нужен — уже строка |

---

## Демо: что будет в DevTools

Секция 4 работает по трём кнопкам:

| Кнопка | Что делает |
|---|---|
| **«Сохранить примеры»** | Записывает 6 ключей в `localStorage`, **оставляет их** |
| **«Загрузить и показать»** | Читает ключи, парсит в Rust-типы, выводит результат |
| **«Очистить примеры»** | Удаляет только эти 6 ключей |

**Что появится в DevTools** после «Сохранить примеры»:

| Ключ | Значение в хранилище | Rust-тип при парсинге |
|---|---|---|
| `demo:i32` | `"42"` | `i32` → `42` |
| `demo:i32_negative` | `"-7"` | `i32` → `-7` |
| `demo:f64` | `"3.14159"` | `f64` → `3.14159` |
| `demo:bool_true` | `"true"` | `bool` → `true` |
| `demo:bool_false` | `"false"` | `bool` → `false` |
| `demo:string` | `"Привет, мир!"` | `String` |

Все значения — **строки**. В DevTools это видно явно: число `42` отображается точно так же, как `"Привет, мир!"`. Это и есть демонстрация ограничения Storage.

**Что может сделать пользователь:**

- **Увидеть** все шесть ключей.
- **Изменить** любое значение вручную — например, сделать `demo:i32` равным `hello`.
- **Удалить** ключ вручную — и увидеть, что «Очистить примеры» посчитает меньше удалённых.
- **Нажать «Загрузить и показать»** — увидеть, как парсинг обрабатывает корректные и некорректные данные.

---

## `src/app.rs`

Показана только **новая секция 4**. Она добавляется **после секции 3** (два счётчика) и **перед** `std::mem::forget(closures)`.

```rust
    // ============================================================
    // СЕКЦИЯ 4: Типы данных
    // ============================================================

    let section4 = build_section(
        &container,
        "4. Типы: только строки",
    )?;

    let types_hint = dom::create_paragraph(
        "Web Storage хранит только строки. Числа, отрицательные числа, \
         float, булевы значения — всё сохраняется как строки. \
         \n\
         Нажмите «Сохранить примеры», затем откройте DevTools \
         → Application → Storage → Local Storage → http://127.0.0.1:5173 \
         и убедитесь: под каждым ключом лежит строка. \
         Можете изменить любое значение вручную — например, сделать \
         demo:i32 равным \"hello\" — и нажать «Загрузить и показать», \
         чтобы увидеть, как парсинг обрабатывает некорректные данные.",
    )?;
    dom::add_class(&dom::as_html(&types_hint), "hint")?;
    dom::append(&section4, &types_hint)?;

    let save_types_btn = dom::create_element("button")?;
    save_types_btn.set_text_content(Some("Сохранить примеры"));
    let load_types_btn = dom::create_element("button")?;
    load_types_btn.set_text_content(Some("Загрузить и показать"));
    let clear_types_btn = dom::create_element("button")?;
    clear_types_btn.set_text_content(Some("Очистить примеры"));

    dom::append(&section4, &save_types_btn)?;
    dom::append(&section4, &load_types_btn)?;
    dom::append(&section4, &clear_types_btn)?;

    let types_demo = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&types_demo), "types-demo")?;
    types_demo.set_text_content(Some(
        "Нажмите «Сохранить примеры», чтобы заполнить localStorage.",
    ));
    dom::append(&section4, &types_demo)?;

    // ---- Обработчик «Сохранить примеры» ----
    {
        let types_demo = types_demo.clone();
        callbacks::listen(
            &save_types_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        types_demo.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                // Пары (ключ, значение). Все значения — строки,
                // потому что Storage принимает только строки.
                let examples: [(&str, String); 6] = [
                    ("demo:i32", 42_i32.to_string()),
                    ("demo:i32_negative", (-7_i32).to_string()),
                    ("demo:f64", 3.14159_f64.to_string()),
                    ("demo:bool_true", true.to_string()),
                    ("demo:bool_false", false.to_string()),
                    ("demo:string", "Привет, мир!".to_string()),
                ];

                let mut saved_count: u32 = 0;
                for (key, value) in &examples {
                    match storage::set(&ls, key, value) {
                        Ok(_) => saved_count += 1,
                        Err(e) => log!("Ошибка сохранения {}: {:?}", key, e),
                    }
                }

                let text: String = format!(
                    "Сохранено {} ключей в localStorage.\n\
                     \n\
                     Откройте DevTools → Application → Storage → Local Storage. \
                     Под каждым ключом лежит строка — независимо от того, \
                     что там число или булево значение.\n\
                     \n\
                     Попробуйте изменить любое значение вручную и нажать \
                     «Загрузить и показать».",
                    saved_count
                );
                types_demo.set_text_content(Some(text.as_str()));

                log!("Types: сохранено {} ключей", saved_count);
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Загрузить и показать» ----
    {
        let types_demo = types_demo.clone();
        callbacks::listen(
            &load_types_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => {
                        types_demo.set_text_content(Some("localStorage недоступен"));
                        return;
                    }
                };

                types_demo.set_inner_html("");

                // Список (ключ, тип для парсинга)
                let entries_to_check: [(&str, &str); 6] = [
                    ("demo:i32", "i32"),
                    ("demo:i32_negative", "i32"),
                    ("demo:f64", "f64"),
                    ("demo:bool_true", "bool"),
                    ("demo:bool_false", "bool"),
                    ("demo:string", "String"),
                ];

                for (key, type_name) in &entries_to_check {
                    let raw: Option<String> = storage::get(&ls, key);

                    let block = dom::create_element("pre").unwrap();

                    match raw {
                        Some(s) => {
                            let parsed_info: String = match *type_name {
                                "i32" => match s.parse::<i32>() {
                                    Ok(n) => format!("{}", n),
                                    Err(_) => "⚠ ошибка парсинга".into(),
                                },
                                "f64" => match s.parse::<f64>() {
                                    Ok(n) => format!("{}", n),
                                    Err(_) => "⚠ ошибка парсинга".into(),
                                },
                                "bool" => match s.parse::<bool>() {
                                    Ok(b) => format!("{}", b),
                                    Err(_) => "⚠ ошибка парсинга".into(),
                                },
                                _ => s.clone(),
                            };

                            let text: String = format!(
                                "{} → \"{}\" (в хранилище)\n\
                                 Парсим как {} → {}",
                                key, s, type_name, parsed_info
                            );
                            block.set_text_content(Some(text.as_str()));
                        }
                        None => {
                            let text: String = format!(
                                "{} → (не найден — нажмите «Сохранить примеры»)",
                                key
                            );
                            block.set_text_content(Some(text.as_str()));
                        }
                    }

                    let _ = dom::append(&types_demo, &block);
                }

                log!("Types: загружено {} ключей", entries_to_check.len());
            },
            &closures,
        )?;
    }

    // ---- Обработчик «Очистить примеры» ----
    {
        let types_demo = types_demo.clone();
        callbacks::listen(
            &clear_types_btn,
            "click",
            move |_event| {
                let ls = match storage::local_storage() {
                    Ok(s) => s,
                    Err(_) => return,
                };

                let keys_to_remove: [&str; 6] = [
                    "demo:i32",
                    "demo:i32_negative",
                    "demo:f64",
                    "demo:bool_true",
                    "demo:bool_false",
                    "demo:string",
                ];

                let mut removed: u32 = 0;
                for key in &keys_to_remove {
                    if storage::has_key(&ls, key) {
                        let _ = storage::remove(&ls, key);
                        removed += 1;
                    }
                }

                let text: String = format!(
                    "Удалено {} ключей из localStorage.\n\
                     Проверьте DevTools — примеры исчезли.",
                    removed
                );
                types_demo.set_text_content(Some(text.as_str()));

                log!("Types: удалено {} ключей", removed);
            },
            &closures,
        )?;
    }
```

**Ключевые моменты:**

- **Данные остаются в localStorage.** Кнопка «Сохранить примеры» ничего не удаляет. Пользователь открывает DevTools — видит все шесть ключей.
- **Ручное изменение поощряется.** В подсказке прямо сказано: измените `demo:i32` на `hello` и нажмите «Загрузить и показать».
- **Обработка ошибок парсинга.** Если значение повреждено — вместо числа выводится `⚠ ошибка парсинга`. Это демонстрирует `Result` от `parse()`.
- **`storage::has_key` в «Очистить».** Считаем только реально удалённые ключи. Если пользователь уже удалил что-то вручную, счётчик покажет меньше.
- **Кириллица в `demo:string`.** Значение `"Привет, мир!"` показывает, что Storage работает с Unicode.
- **Отрицательное число в `demo:i32_negative`.** Показывает, что знак `-` сохраняется в строке.

---

## `web/styles/main.css`

Правила для `.types-demo` из 15.5 остаются, но `gap` уменьшается для плотности блоков:

```css
/* ============================================================
   Секция типов
   ============================================================ */

.types-demo {
    margin-top: 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
}

.types-demo pre {
    font-family: ui-monospace, "SF Mono", Menlo, Consolas, monospace;
    font-size: 0.85rem;
    line-height: 1.5;
    white-space: pre-wrap;
    background: #f5f5f5;
    padding: 0.75rem;
    border-radius: 4px;
    border-left: 3px solid #b7410e;
    margin: 0;
    overflow-x: auto;
}
```

**Что изменилось:** `gap: 0.5rem` вместо `1rem`. Блоки выводов стоят плотнее, потому что их теперь шесть.

---

## `tests/wasm_storage.rs`

Добавьте в конец файла (после тестов из 15.4):

```rust
// ============================================================
// Сериализация: числа
// ============================================================

#[wasm_bindgen_test]
fn test_i32_roundtrip() {
    let ls = storage::local_storage().unwrap();

    let original: i32 = 42;
    let serialized: String = original.to_string();
    storage::set(&ls, "test:i32", &serialized).unwrap();

    let retrieved = storage::get(&ls, "test:i32").unwrap();
    let parsed: i32 = retrieved.parse().unwrap();

    assert_eq!(parsed, original);
    assert_eq!(retrieved, "42");

    storage::remove(&ls, "test:i32").ok();
}

#[wasm_bindgen_test]
fn test_negative_i32_roundtrip() {
    let ls = storage::local_storage().unwrap();

    let original: i32 = -123;
    storage::set(&ls, "test:neg_i32", &original.to_string()).unwrap();

    let retrieved = storage::get(&ls, "test:neg_i32").unwrap();
    let parsed: i32 = retrieved.parse().unwrap();

    assert_eq!(parsed, original);
    assert_eq!(retrieved, "-123");

    storage::remove(&ls, "test:neg_i32").ok();
}

#[wasm_bindgen_test]
fn test_f64_roundtrip() {
    let ls = storage::local_storage().unwrap();

    let original: f64 = 3.14159;
    storage::set(&ls, "test:f64", &original.to_string()).unwrap();

    let retrieved = storage::get(&ls, "test:f64").unwrap();
    let parsed: f64 = retrieved.parse().unwrap();

    assert!((parsed - original).abs() < 1e-10);
    assert!(retrieved.starts_with("3.14"));

    storage::remove(&ls, "test:f64").ok();
}

// ============================================================
// Сериализация: булевы значения
// ============================================================

#[wasm_bindgen_test]
fn test_bool_true_roundtrip() {
    let ls = storage::local_storage().unwrap();

    let original: bool = true;
    storage::set(&ls, "test:bool_true", &original.to_string()).unwrap();

    let retrieved = storage::get(&ls, "test:bool_true").unwrap();
    let parsed: bool = retrieved.parse().unwrap();

    assert_eq!(parsed, original);
    assert_eq!(retrieved, "true");

    storage::remove(&ls, "test:bool_true").ok();
}

#[wasm_bindgen_test]
fn test_bool_false_roundtrip() {
    let ls = storage::local_storage().unwrap();

    let original: bool = false;
    storage::set(&ls, "test:bool_false", &original.to_string()).unwrap();

    let retrieved = storage::get(&ls, "test:bool_false").unwrap();
    let parsed: bool = retrieved.parse().unwrap();

    assert_eq!(parsed, original);
    assert_eq!(retrieved, "false");

    storage::remove(&ls, "test:bool_false").ok();
}

// ============================================================
// Ошибки парсинга
// ============================================================

#[wasm_bindgen_test]
fn test_parse_fails_on_invalid_input() {
    let ls = storage::local_storage().unwrap();

    // Кладём в хранилище строку, которая не является числом
    storage::set(&ls, "test:invalid_int", "hello").unwrap();

    let retrieved = storage::get(&ls, "test:invalid_int").unwrap();
    let parsed: Result<i32, _> = retrieved.parse::<i32>();

    assert!(parsed.is_err(), "parsing 'hello' as i32 should fail");

    storage::remove(&ls, "test:invalid_int").ok();
}
```

**Что проверяют новые тесты:**

| Тест | Что проверяет |
|---|---|
| `test_i32_roundtrip` | `42` → `"42"` → `42` |
| `test_negative_i32_roundtrip` | `-123` → `"-123"` → `-123` |
| `test_f64_roundtrip` | `3.14159` → строка → то же число |
| `test_bool_true_roundtrip` | `true` → `"true"` → `true` |
| `test_bool_false_roundtrip` | `false` → `"false"` → `false` |
| `test_parse_fails_on_invalid_input` | `parse::<i32>()` на `"hello"` возвращает `Err` |

**Почему в тестах `parse::<i32>()` на хосте работает.** `str::parse` — метод стандартной библиотеки Rust. Он не зависит от Wasm. Тесты работают одинаково в обоих браузерах.

**Всего тестов:** 25 из 15.1–15.4 + 6 новых = **31 тест**.

---

## Команды запуска

### Разработка

```bash
npm start
```

Откройте `http://127.0.0.1:5173`. Прокрутите до секции 4.

**Сценарий 1: базовый.**

1. Нажмите **«Сохранить примеры»**. Появится: «Сохранено 6 ключей в localStorage».
2. Откройте DevTools → **Application** → **Storage** → **Local Storage** → `http://127.0.0.1:5173`.
3. Увидите шесть ключей:

   ```
   demo:i32            42
   demo:i32_negative   -7
   demo:f64            3.14159
   demo:bool_true      true
   demo:bool_false     false
   demo:string         Привет, мир!
   ```

   Все значения — строки. DevTools показывает их одинаково.

4. Вернитесь в UI, нажмите **«Загрузить и показать»**. Увидите:

   ```
   demo:i32 → "42" (в хранилище)
   Парсим как i32 → 42

   demo:i32_negative → "-7" (в хранилище)
   Парсим как i32 → -7

   demo:f64 → "3.14159" (в хранилище)
   Парсим как f64 → 3.14159

   demo:bool_true → "true" (в хранилище)
   Парсим как bool → true

   demo:bool_false → "false" (в хранилище)
   Парсим как bool → false

   demo:string → "Привет, мир!" (в хранилище)
   Парсим как String → Привет, мир!
   ```

**Сценарий 2: пользователь ломает данные.**

1. Нажмите «Сохранить примеры».
2. В DevTools **измените** `demo:i32` на `hello`.
3. Нажмите «Загрузить и показать».
4. Увидите:

   ```
   demo:i32 → "hello" (в хранилище)
   Парсим как i32 → ⚠ ошибка парсинга
   ```

   Остальные пять ключей отобразятся нормально — ошибка локальна для одного ключа.

**Сценарий 3: пользователь удаляет вручную.**

1. В DevTools **удалите** `demo:bool_false`.
2. Нажмите «Очистить примеры».
3. Увидите: «Удалено 5 ключей» — потому что один уже удалён вручную.

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

**Ожидаемый результат:** 31 тест `ok` в обоих браузерах.

---

## Что важно запомнить

**1. Web Storage хранит только строки.** `get_item` возвращает `Option<String>`, `set_item` принимает `&str`. Никаких чисел, булевых, объектов.

**2. Числа — через `to_string()` и `parse()`.** `42.to_string()` даёт `"42"`. `"42".parse::<i32>()` даёт `42`.

**3. Булевы — то же самое.** `true.to_string()` даёт `"true"`. `"true".parse::<bool>()` даёт `true`.

**4. `parse()` возвращает `Result`.** Может провалиться, если строка не соответствует формату. Обрабатывайте через `?`, `match`, `unwrap_or`, `.ok()`.

**5. `parse::<bool>()` строг.** Принимает только `"true"` и `"false"`. `"TRUE"`, `"1"`, `"yes"` — ошибка.

**6. Float-числа округляются.** `f64::to_string()` даёт 15–17 значащих цифр. Обычно хватает, но для точных финансовых вычислений используйте целые числа (копейки как `i64`).

**7. `parse::<u32>()` на отрицательном — ошибка.** `"-5".parse::<u32>()` вернёт `Err`. Выбирайте тип по диапазону значений.

**8. Тестируйте round-trip.** Сохранили — прочитали — сравнили. Это лучший способ убедиться, что сериализация работает.

**9. Данные в DevTools наглядны.** Оставив ключи в хранилище, мы даём пользователю увидеть, что Storage хранит строки. Пользователь может изменить значения вручную и увидеть, как приложение реагирует.

**10. Разделение уровней.** `storage.rs` не знает о числах и булевых. Он работает со строками. Сериализация — ответственность `app.rs` или выше.

---

## Что дальше

**15.6. JSON: структуры в Storage.** Мы перейдём от примитивов к **сложным структурам**: `Vec<Note>`, `Settings`, вложенные объекты. Понадобятся crate'ы `serde` и `serde_json`. В `storage.rs` появятся две новые функции: `set_json` и `get_json` — обёртки над `serde_json::to_string` и `from_str`. В `app.rs` появится секция «Заметки»: список заметок как `Vec<Note>`, сохранение в `localStorage`, отображение и preview сырого JSON. В тестах — 10 новых проверок на структуры, `Option`, `Vec`, невалидный JSON.