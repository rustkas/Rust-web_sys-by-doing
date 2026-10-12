# Глава 3. Первый Rust + `web_sys` проект

## 1. Создание проекта

Начнём с создания структуры проекта. Мы будем использовать `cargo new` с флагом `--lib`, потому что наш Rust-код будет компилироваться в динамическую библиотеку (`.wasm`), а не в исполняемый файл.

```bash
cargo new --lib wasm-hello
cd wasm-hello
```

Структура проекта после создания:

```
wasm-hello/
├── Cargo.toml
└── src/
    └── lib.rs
```

Откройте `Cargo.toml` — Cargo создал заготовку. Нам нужно добавить несколько секций. Во-первых, укажем, что crate имеет тип `cdylib` (C-compatible dynamic library). Во-вторых, добавим зависимости.

```toml
[package]
name = "wasm-hello"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"

[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "Element",
    "HtmlElement",
    "Node",
]
```

Секция `[lib]` с `crate-type = ["cdylib"]` критически важна. Без неё `wasm-pack` не сможет собрать `.wasm`-модуль, потому что Rust по умолчанию компилирует библиотеки в формат `rlib`, предназначенный для линковки с другими Rust-crate'ами, а не для загрузки в браузер.

## 2. `wasm-bindgen`

`wasm-bindgen` — это ядро всей системы. Без него Rust-код, скомпилированный в Wasm, был бы полностью изолирован от JavaScript: он мог бы экспортировать только функции с примитивными типами (`i32`, `f64`), и JavaScript-код должен был бы вручную управлять памятью и преобразованиями.

Добавим `wasm-bindgen` в `Cargo.toml`, как показано выше. Это одновременно и библиотека (crate), и CLI-инструмент. Библиотека нужна для атрибутов вроде `#[wasm_bindgen]`, а CLI — для генерации JavaScript-клея.

Установим CLI-инструмент:

```bash
cargo install wasm-bindgen-cli
```

Убедитесь, что версия CLI совпадает с версией crate в `Cargo.toml`. Если они разойдутся, `wasm-pack` выдаст ошибку.

## 3. `web-sys`

`web-sys` — это crate, предоставляющий привязки к браузерным API. В `Cargo.toml` мы уже добавили его с features: `Window`, `Document`, `Element`, `HtmlElement`, `Node`.

Каждый feature включает генерацию соответствующего типа. `Window` нужен для получения глобального объекта `window`, `Document` — для работы с DOM-деревом, `Element` и `HtmlElement` — для манипуляции элементами, `Node` — для базовых операций с узлами.

Обратите внимание: мы **не** включаем `HtmlCanvasElement` или `Storage` — они не нужны для этого проекта. Принцип «включать только необходимое» особенно важен для `web_sys`, потому что лишние features увеличивают время компиляции и размер `.wasm`-файла.

## 4. `wasm-pack`

`wasm-pack` — это инструмент, который оркестрирует весь процесс сборки. Он делает следующее:

1. Компилирует Rust-код в `.wasm` с target `wasm32-unknown-unknown`.
2. Запускает `wasm-bindgen` для генерации JavaScript-клея и `.d.ts`-файлов.
3. Оптимизирует `.wasm` в release-режиме через `wasm-opt`.
4. Создаёт npm-совместимую структуру пакета в директории `pkg/`.

Установим `wasm-pack`:

```bash
cargo install wasm-pack
```

Или через официальный установщик:

```bash
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
```

Проверьте установку:

```bash
wasm-pack --version
```

## 5. `wasm32-unknown-unknown`

Rust компилируется в машинный код для конкретной платформы. Обычные цели — `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin` и т.д. Для WebAssembly существует отдельная цель — `wasm32-unknown-unknown`.

`wasm32` означает 32-битную архитектуру Wasm. `unknown-unknown` означает, что операционная система и ABI не определены — Wasm-модуль работает в песочнице и не зависит от ОС.

Установим цель:

```bash
rustup target add wasm32-unknown-unknown
```

Проверьте, что цель добавлена:

```bash
rustup target list --installed
```

В списке должен появиться `wasm32-unknown-unknown`.

## 6. HTML + JavaScript + WASM

Теперь создадим минимальный HTML-файл и JavaScript-код, который загрузит наш Wasm-модуль.

Создайте `index.html` в корне проекта:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rust + web_sys</title>
</head>
<body>
    <h1>Rust + web_sys</h1>
    <p id="output"></p>
    <script type="module" src="/src/main.js"></script>
</body>
</html>
```

Создайте директорию `src/` (если её нет — она уже существует для Rust-кода, но теперь мы добавим туда JavaScript-файл). Создайте `src/main.js`:

```javascript
import init from '../pkg/wasm_hello.js';

async function run() {
    // Инициализируем Wasm-модуль
    const wasm = await init();
    
    // Вызываем экспортированную Rust-функцию
    wasm.greet("Rust");
}

run();
```

Обратите внимание: мы используем `import init from '../pkg/wasm_hello.js'`. Директория `pkg/` появится после первой сборки `wasm-pack`. Имя файла — `wasm_hello.js` — соответствует имени пакета, где дефисы заменены на подчёркивания.

## 7. Первый вызов браузерного API

Напишем Rust-код, который создаёт DOM-элемент и добавляет его на страницу. Откройте `src/lib.rs`:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) {
    // Получаем глобальный объект window
    let window = web_sys::window().expect("no global `window` exists");
    
    // Получаем document
    let document = window.document().expect("should have a document on window");
    
    // Получаем body
    let body = document.body().expect("document should have a body");
    
    // Создаём элемент <p>
    let val = document.create_element("p").expect("failed to create element");
    
    // Устанавливаем текстовое содержимое
    val.set_text_content(Some(&format!("Привет, {}!", name)));
    
    // Добавляем элемент в body
    body.append_child(&val).expect("failed to append child");
}
```

Разберём код по шагам:

**`web_sys::window()`** — возвращает `Option<Window>`. В браузере `window` всегда существует, поэтому мы используем `expect`. Если бы это был Worker или Service Worker, `window` мог бы отсутствовать.

**`window.document()`** — возвращает `Option<Document>`. Документ существует только в контексте окна.

**`document.body()`** — возвращает `Option<HtmlElement>`. Это тело документа.

**`document.create_element("p")`** — возвращает `Result<Element, JsValue>`. Создаёт новый DOM-элемент указанного типа.

**`val.set_text_content(Some(...))`** — устанавливает текстовое содержимое. Принимает `Option<&str>`, потому что `null` допустим (очистка содержимого).

**`body.append_child(&val)`** — возвращает `Result<Node, JsValue>`. Добавляет элемент в конец body.

Все DOM-операции, которые могут завершиться ошибкой, возвращают `Result<T, JsValue>`. `JsValue` — это универсальный тип для любого JavaScript-значения. Обработка ошибок через `expect` подходит для примеров, но в production-коде лучше использовать `?` и передавать ошибку дальше.

## 8. Проверка результата в браузере

Теперь настроим Vite для запуска JS-части. Установим Vite локально в проект:

```bash
npm init -y
npm install --save-dev vite
```

Добавим npm-скрипт в `package.json`:

```json
{
  "scripts": {
    "dev": "vite --host 127.0.0.1"
  }
}
```

`--host 127.0.0.1` гарантирует, что сервер будет доступен по локальному адресу, а не только на `localhost` (иногда это важно в контейнерах и WSL).

Теперь соберём Wasm и запустим dev-сервер. Итоговая команда:

```bash
wasm-pack build --target web --out-dir pkg && npm run dev
```

Разберём флаги `wasm-pack build`:

- **`--target web`** — генерирует JavaScript-клей в формате ES-модулей. Это то, что нужно для Vite и современных браузеров.
- **`--out-dir pkg`** — указывает, куда положить сгенерированные файлы. По умолчанию `wasm-pack` использует `pkg/`, но явное указание полезно для ясности.

После сборки в директории `pkg/` появятся:

```
pkg/
├── wasm_hello.js       # JavaScript-клей
├── wasm_hello_bg.wasm  # WebAssembly-модуль
├── wasm_hello.d.ts     # TypeScript-определения
└── package.json        # npm-манифест (опционально)
```

Vite автоматически подхватит `index.html` как точку входа. Откройте браузер по адресу, который покажет Vite (обычно `http://127.0.0.1:5173`). На странице должен появиться заголовок «Rust + web_sys» и абзац «Привет, Rust!», созданный Rust-кодом.

Если абзац не появился, откройте DevTools (F12) и проверьте консоль. Типичные ошибки:

- **`no global window exists`** — код выполняется не в контексте окна (например, в Worker).
- **`should have a document on window`** — то же самое.
- **`failed to create element`** — имя тега невалидно.
- **`failed to append child`** — элемент уже является частью другого узла (маловероятно для нового элемента).

## 9. Debug-сборка

По умолчанию `wasm-pack build` создаёт **debug-сборку**. Она быстрее компилируется, но `.wasm`-файл больше и содержит отладочную информацию.

Характеристики debug-сборки:

- **Время компиляции:** секунды.
- **Размер `.wasm`:** может быть несколько мегабайт даже для простого проекта.
- **Оптимизации:** отключены.
- **Отладочные символы:** присутствуют.

Debug-сборка полезна на этапе разработки, когда вы часто меняете код и хотите быстро видеть результат.

Чтобы явно указать debug-режим:

```bash
wasm-pack build --target web --out-dir pkg --dev
```

Флаг `--dev` (или `--debug`) гарантирует, что сборка будет в debug-режиме, даже если в `Cargo.toml` есть настройки профиля.

## 10. Release-сборка

Для production нужна **release-сборка**. Она компилируется дольше, но `.wasm`-файл значительно меньше и работает быстрее.

```bash
wasm-pack build --target web --out-dir pkg --release
```

Что происходит в release-режиме:

1. Rust компилирует с оптимизациями (`opt-level = 3` по умолчанию).
2. `wasm-pack` запускает `wasm-opt` (из Binaryen) для дополнительной оптимизации размера.
3. Отладочные символы удаляются.
4. Мёртвый код устраняется (dead code elimination).

Разница в размере может быть десятикратной. Например, debug-сборка простого проекта может весить 2 МБ, а release-сборка — 150–200 КБ.

Для дополнительной оптимизации размера можно добавить в `Cargo.toml`:

```toml
[profile.release]
opt-level = "s"    # оптимизация по размеру
lto = true         # link-time optimization
codegen-units = 1  # меньше единиц компиляции — лучше оптимизация
strip = true       # удаление символов
```

`opt-level = "s"` оптимизирует по размеру, а не по скорости. Для большинства веб-приложений это правильный выбор. `lto = true` включает межмодульную оптимизацию — она может существенно уменьшить размер, но увеличивает время компиляции.

После release-сборки можно проверить размер `.wasm`:

```bash
ls -lh pkg/*.wasm
```

Для нашего проекта release-сборка должна весить менее 200 КБ.

---

**Итог главы:**

- Проект создаётся через `cargo new --lib` с `crate-type = ["cdylib"]`.
- `wasm-bindgen` — ядро связи Rust и JavaScript; устанавливается и как crate, и как CLI.
- `web-sys` требует явного указания features для каждого используемого типа.
- `wasm-pack` оркестрирует сборку: компиляция, генерация клея, оптимизация.
- Цель `wasm32-unknown-unknown` устанавливается через `rustup target add`.
- HTML + JavaScript загружают Wasm-модуль через `import init from '../pkg/...'`.
- Первый вызов браузерного API — получение `window`, `document`, создание элемента и добавление в DOM.
- Debug-сборка быстрая, но большая; release-сборка медленнее, но значительно меньше.
- Команда для полного цикла: `wasm-pack build --target web --out-dir pkg && npm run dev`.

---

**Что дальше.** В следующей главе мы разберём сборку глубже: как работает `wasm-bindgen-cli` изнутри, что делает `wasm-opt`, как настроить интеграцию с Vite для production, и как уменьшить размер `.wasm` до минимума.