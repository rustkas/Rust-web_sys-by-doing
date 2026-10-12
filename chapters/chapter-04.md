# Глава 4. Сборка, оптимизация и деплой

## 1. `wasm-bindgen-cli` vs `wasm-pack`

В главе 3 мы использовали `wasm-pack` как основной инструмент сборки. Но под ним работает другой инструмент — `wasm-bindgen-cli`, и понимание разницы между ними важно для контроля над процессом.

**`wasm-bindgen-cli`** — это низкоуровневый инструмент, который выполняет **постобработку** `.wasm`-модуля. Он читает скомпилированный `rustc` файл, анализирует экспортируемые и импортируемые функции и генерирует JavaScript-клей для взаимодействия с ними. Это инструмент, который знает, как превратить «сырой» Wasm-модуль в модуль, пригодный для использования из JS.

Установка:

```bash
cargo install wasm-bindgen-cli
```

Использование напрямую:

```bash
wasm-bindgen --target web --out-dir pkg target/wasm32-unknown-unknown/release/hello_wasm02.wasm
```

**`wasm-pack`** — это **оркестратор** всего процесса. Он делает всё, что делает `wasm-bindgen-cli`, но дополнительно:

- компилирует Rust-код в Wasm (вызывает `cargo build --target wasm32-unknown-unknown`);
- запускает `wasm-bindgen` с правильными флагами;
- генерирует `package.json` для npm;
- копирует `README.md` и `LICENSE` из исходного crate'а;
- генерирует TypeScript-определения;
- упаковывает результат в папку `pkg/`.

Именно `wasm-pack` является рекомендуемым инструментом, потому что он автоматизирует рутинные шаги и не требует писать скрипты для bootstrapping Wasm-модуля. Прямое использование `wasm-bindgen-cli` оправдано только в случаях, когда нужен тонкий контроль над процессом (например, в кастомных CI-пайплайнах с собственным bundler'ом).

## 2. `wasm-opt` и оптимизация размера

`wasm-opt` — это инструмент из проекта **Binaryen**, который оптимизирует `.wasm`-модуль **на уровне байткода**. Он работает **после** компиляции `rustc` и `wasm-bindgen` и применяет к модулю серию оптимизационных проходов: удаление мёртвого кода, свёртку констант, инлайнинг, сокращение размера стековых операций.

Установка:

```bash
# macOS
brew install binaryen

# Ubuntu/Debian
sudo apt-get install binaryen

# Или из исходников
git clone https://github.com/WebAssembly/binaryen
cd binaryen && cmake . && make
```

Использование:

```bash
wasm-opt -Oz -o output.wasm input.wasm
```

Флаг `-Oz` означает агрессивную оптимизацию по размеру. Флаг `-O3` — оптимизацию по скорости. `wasm-pack` **автоматически запускает `wasm-opt`** для release-сборок, если `wasm-opt` установлен в системе. Типичное уменьшение размера — **15–30%**.

Если вы хотите отключить `wasm-opt` (например, если он вызывает проблемы совместимости с определённым браузером), добавьте в `Cargo.toml`:

```toml
[package.metadata.wasm-pack.profile.release]
wasm-opt = false
```

## 3. `lto`, `opt-level`, `codegen-units`

Rust-компилятор предоставляет несколько параметров профиля, которые напрямую влияют на размер и производительность Wasm-модуля. Вот оптимальная конфигурация для production:

```toml
[profile.release]
opt-level = "s"        # "z" для минимального размера, "3" для максимальной скорости
lto = true             # Link Time Optimization
codegen-units = 1      # Лучшая оптимизация за счёт времени компиляции
panic = "abort"        # Убирает разворачивание стека при панике
strip = true           # Удаляет отладочные секции
```

Разберём каждый параметр.

**`opt-level`** — уровень оптимизации. Значение `"s"` оптимизирует по размеру, `"z"` — агрессивно по размеру, `"3"` — по скорости. Для WebAssembly размер критичен, потому что модуль загружается по сети. Начните с `"s"`, а если размер всё ещё велик — переключитесь на `"z"`.

**`lto`** — Link Time Optimization. Позволяет оптимизировать код на уровне всей программы, а не отдельных модулей. Это может существенно уменьшить размер за счёт удаления неиспользуемых функций.

**`codegen-units`** — количество единиц компиляции. `1` означает, что весь crate компилируется как единое целое, что даёт лучшую оптимизацию, но увеличивает время компиляции.

**`panic = "abort"`** — при панике Wasm-модуль не разворачивает стек, а немедленно прерывает выполнение. Это удаляет всю машинерию для разворачивания стека и может уменьшить размер на **~30%**. Важно: при `panic = "abort"` вы не можете ловить паники через `catch_unwind`.

**`strip = true`** — удаляет отладочные секции из `.wasm`-файла. В release-сборке они не нужны, а размер увеличивают.

## 4. Размер бандла: что реально влияет

Размер `.wasm`-файла — критический показатель для WebAssembly-приложения. Вот факторы, которые влияют на него больше всего:

**Debug vs Release.** Debug-сборка может быть в **10–20 раз больше** release-сборки. Простой проект с DOM-доступом: ~1–2 МБ в debug, ~50–100 КБ в release. Всегда проверяйте release.

**Cargo features.** Каждая включённая feature `web_sys` добавляет сгенерированный код. Если вы включили `HtmlCanvasElement`, но не используете Canvas — код всё равно попадёт в бинарник, потому что `wasm-bindgen` не выполняет tree-shaking на уровне features. Отсюда правило: включайте только те features, которые реально используются.

**Сериализация.** `serde` и `serde_json` добавляют значительный объём кода. Если вы используете их только для передачи данных через границу JS↔Rust, подумайте, можно ли обойтись `JsValue` или прямым чтением полей.

**Зависимости.** Каждый crate в dependency tree добавляет код. Периодически проверяйте `cargo tree` и удаляйте лишние зависимости. `wasm-pack` не удаляет неиспользуемый код автоматически.

**Компрессия.** `.wasm`-файлы сжимаются очень хорошо: типичный коэффициент — **до 50% от исходного размера** при Brotli или gzip. Убедитесь, что ваш сервер отдаёт `.wasm` с правильными заголовками `Content-Encoding`. Brotli эффективнее gzip для Wasm.

**`wasm-opt`.** Как уже упоминалось, даёт 15–30% уменьшения.

Целевые ориентиры: браузерные модули — **< 500 КБ несжатыми, < 200 КБ сжатыми**.

## 5. Интеграция с Vite / webpack / Rollup

Каждый bundler требует своего способа интеграции с Wasm. Разберём основные сценарии.

**Vite.** Самый простой путь. После `wasm-pack build --target web --out-dir pkg` вы импортируете сгенерированный модуль как обычный ES-модуль:

```javascript
import init from './pkg/hello_wasm02.js';

await init();
```

Vite автоматически обработает `.wasm`-файл и настроит правильную загрузку. Для более продвинутой интеграции есть плагины, например `unplugin-rust-wasm-pack`, который запускает `wasm-pack` автоматически и генерирует TypeScript-шимы.

**webpack.** Требует `wasm-pack-plugin` или ручной настройки `experiments.asyncWebAssembly = true`. После сборки с `--target bundler` импорт работает как обычный модуль:

```javascript
import { greet } from './pkg/hello_wasm02';

greet('World');
```

**Rollup.** Требует плагин `@rollup/plugin-wasm` или аналогичный. С `--target bundler` импорт работает так же, как в webpack.

Общее правило: если вы используете современный bundler (Vite, webpack 5, Rollup), собирайте с `--target bundler`. Если вы используете нативные ES-модули в браузере без bundler'а — `--target web`. Если вам нужен глобальный объект для `<script>` — `--target no-modules`.

## 6. Bundler vs `--target web` / `--target no-modules`

`wasm-pack` поддерживает пять target-режимов:

| Target | Модульная система | Для чего |
|---|---|---|
| `bundler` | ES Modules | webpack, Rollup, Parcel — default |
| `web` | ES Modules | Нативные ES-модули в браузере |
| `nodejs` | CommonJS | Node.js через `require()` |
| `no-modules` | IIFE (глобальный объект) | Legacy-браузеры без модулей |
| `deno` | ES Modules | Deno runtime |

**`bundler`** — режим по умолчанию. Генерирует `package.json` с `"type": "module"` и `sideEffects` для tree-shaking. Импорт работает через `import` в bundler'е.

**`web`** — генерирует ES-модуль, который можно загрузить напрямую в браузере. Требует **ручного вызова `init()`** перед использованием экспортированных функций. Это то, что мы использовали в главе 3.

**`no-modules`** — генерирует IIFE, который создаёт глобальный объект. Используется, когда нет возможности использовать ES-модули (например, очень старые браузеры или простые `<script>`-теги).

Практическое правило: **используйте `bundler` для production**, если у вас есть bundler. `web` — для прототипов и простых проектов без bundler'а. `no-modules` — только если вы поддерживаете legacy-браузеры.

## 7. Структура production-сборки

После `wasm-pack build --release --target web` в директории `pkg/` появляются следующие файлы:

```
pkg/
├── hello_wasm02.js          # JS-клей (ES-модуль)
├── hello_wasm02_bg.wasm     # Сам Wasm-модуль
├── hello_wasm02.d.ts        # TypeScript-определения
├── hello_wasm02_bg.wasm.d.ts # TypeScript-определения для Wasm
├── package.json             # npm-манифест
├── README.md                # Копия README из crate
└── LICENSE                  # Копия LICENSE
```

Для production-развёртывания вам нужны **только `hello_wasm02.js` и `hello_wasm02_bg.wasm`** (плюс `.d.ts`, если вы используете TypeScript). Остальные файлы — для npm-публикации.

Структура production-бандла после сборки через Vite:

```
dist/
├── index.html
├── assets/
│   ├── index-abc123.js      # Ваш JS + сгенерированный клей
│   └── hello_wasm02-xyz789.wasm  # Wasm-модуль
```

Vite автоматически хэширует имена файлов для cache busting.

## 8. Кэширование и версионирование `.wasm`

`.wasm`-файлы **должны кэшироваться агрессивно**, потому что они редко меняются между деплоями. Но когда они меняются, браузер должен получить новую версию.

**Стратегия 1: Хэширование имён файлов.** Это то, что делает Vite по умолчанию. `hello_wasm02-xyz789.wasm` — при изменении содержимого меняется хэш, и браузер загружает новый файл. Старый файл можно кэшировать «навсегда» (`Cache-Control: public, max-age=31536000, immutable`).

**Стратегия 2: Query string версионирование.** Добавьте версию к URL:

```html
<script type="module" src="/pkg/hello_wasm02.js?v=1.2.3"></script>
```

Это менее эффективно, чем хэширование, но проще в реализации, если вы не используете bundler с автоматическим хэшированием.

**Стратегия 3: Service Worker.** Если вы используете offline-first архитектуру (часть VIII книги), Service Worker может управлять кэшированием `.wasm` и обновлять его по своему усмотрению.

**Важно:** `index.html` **не должен** кэшироваться агрессивно. Он должен перезагружаться при каждом визите (`Cache-Control: no-cache` или короткий `max-age`), чтобы браузер увидел новые хэшированные имена файлов.

## 9. CI/CD для WASM-проектов

Настроим GitHub Actions для сборки и деплоя Wasm-проекта. Вот базовый workflow:

```yaml
name: Build and Deploy WASM

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: wasm32-unknown-unknown

      - name: Install wasm-pack
        run: cargo install wasm-pack --locked

      - name: Install wasm-opt
        run: sudo apt-get install binaryen

      - name: Build WASM
        run: wasm-pack build --release --target web --out-dir pkg

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Build frontend
        run: npm run build

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
```

Ключевые моменты:

- **`targets: wasm32-unknown-unknown`** — устанавливает Wasm-target через `rust-toolchain` action.
- **`cargo install wasm-pack --locked`** — устанавливает `wasm-pack`. Альтернатива: `curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh`.
- **`sudo apt-get install binaryen`** — устанавливает `wasm-opt` (опционально, но рекомендуется).
- **`--release`** — обязательно для production-сборки.

Для кэширования зависимостей можно добавить `actions/cache` для `~/.cargo` и `node_modules`.

## 10. Публикация и хостинг

**Публикация в npm.** `wasm-pack` умеет публиковать сгенерированный пакет в npm-реестр:

```bash
wasm-pack login
wasm-pack publish
```

`wasm-pack login` аутентифицирует вас в npm, `wasm-pack publish` создаёт tarball и публикует его. Перед публикацией убедитесь, что в `Cargo.toml` указаны корректные `description`, `license`, `repository` — они попадут в `package.json`.

**Хостинг.** Поскольку Wasm-приложение — это набор статических файлов (HTML, JS, `.wasm`), его можно хостить где угодно:

- **GitHub Pages** — бесплатно, отлично подходит для демо и документации.
- **Vercel / Netlify** — автоматический деплой из Git, поддержка preview-окружений.
- **Cloudflare Pages** — быстрый CDN, автоматическая Brotli-компрессия.
- **AWS S3 + CloudFront** — для production-нагрузок.
- **Nginx** — если вы хостите на своём сервере.

**Важно для хостинга:** сервер должен отдавать `.wasm` с правильным MIME-типом: `application/wasm`. Без этого браузер не сможет использовать `WebAssembly.instantiateStreaming()` и будет вынужден загружать модуль целиком перед компиляцией. Большинство современных хостингов (Vercel, Netlify, Cloudflare Pages) уже настроены правильно. Для Nginx добавьте в конфигурацию:

```nginx
types {
    application/wasm wasm;
}
```

Также убедитесь, что включена Brotli или gzip-компрессия для `.wasm`-файлов. Это уменьшит размер передачи в 2 раза.

---

**Итог главы:**

- `wasm-pack` — оркестратор, `wasm-bindgen-cli` — низкоуровневый инструмент постобработки.
- `wasm-opt` даёт 15–30% уменьшения размера; `wasm-pack` запускает его автоматически.
- Оптимальный release-профиль: `opt-level = "s"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, `strip = true`.
- На размер влияют: debug vs release, Cargo features, зависимости, сериализация.
- Для Vite/webpack/Rollup используйте `--target bundler`; для нативных ES-модулей — `--target web`.
- Хэширование имён файлов — лучшая стратегия cache busting.
- CI/CD на GitHub Actions: установить target, `wasm-pack`, `wasm-opt`, собрать и задеплоить.
- Публикация в npm — через `wasm-pack publish`.
- Хостинг: любой статический хостинг; убедитесь в MIME-типе `application/wasm` и компрессии.

---

**Что дальше.** В главе 5 мы разберём отладку Rust/WASM в браузере: `console_error_panic_hook`, source maps, чтение стеков вызовов Wasm в DevTools и типичные ошибки, которые возникают на границе Rust↔JavaScript.