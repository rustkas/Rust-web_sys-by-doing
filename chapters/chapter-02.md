# Глава 2. Как устроен `web_sys`

## 1. WebIDL как источник API

Каждый браузерный API — `Window`, `Document`, `fetch`, `CanvasRenderingContext2d` — описан в спецификациях W3C и WHATWG. Но спецификации пишутся на естественном языке с вкраплениями формального синтаксиса. Чтобы из них можно было генерировать код, существует **WebIDL** (Web Interface Definition Language) — язык описания интерфейсов, на котором фиксируются сигнатуры методов, типы аргументов, наследование и поведение браузерных API.

Пример фрагмента WebIDL для `Window`:

```webidl
[Exposed=Window]
interface Window : EventTarget {
  readonly attribute Document document;
  undefined resizeTo(long width, long height);
  Window? open(optional USVString url = "", optional DOMString target = "_blank");
};
```

Здесь всё строго: `document` — атрибут только для чтения типа `Document`; `resizeTo` принимает два `long` и ничего не возвращает; `open` имеет необязательные параметры с значениями по умолчанию. Именно из таких описаний `web_sys` генерирует Rust-код.

WebIDL — не «документация для людей», а **машиночитаемый контракт**. Спецификация WebIDL определяет грамматику, а браузеры (и инструменты вроде `wasm-bindgen`) используют её для генерации привязок. Это ключевой момент: `web_sys` не «написан вручную», он **порождён** из тех же спецификаций, которым следуют браузеры.

## 2. Почему crate генерируется автоматически

Web Platform содержит сотни интерфейсов, тысячи методов и десятки тысяч сигнатур. Писать привязки к ним вручную — не просто трудоёмко, а **невозможно поддерживать**: спецификации меняются, браузеры добавляют новые API, старые помечаются deprecated.

`web_sys` решает эту проблему радикально: crate генерируется **полностью механически** из WebIDL-файлов. Вот как это устроено:

```
crates/web-sys/
├── build.rs              ← запускает генерацию
├── Cargo.toml            ← содержит список features
├── src/
│   └── lib.rs            ← включает сгенерированный код
└── webidls/
    └── enabled/          ← WebIDL-файлы из спецификаций
        ├── Window.webidl
        ├── Document.webidl
        ├── HtmlElement.webidl
        └── ...
```

`build.rs` вызывает WebIDL-фронтенд `wasm-bindgen`, который парсит все `.webidl`-файлы и генерирует Rust-привязки в `OUT_DIR`. `src/lib.rs` — это буквально несколько строк, которые включают сгенерированный код:

```rust
// src/lib.rs (упрощённо)
mod features;
pub use features::*;
pub use js_sys;
pub use wasm_bindgen;
```

`build.rs` — это **процесс сборки**, который превращает формальные спецификации в типизированный Rust-код. Благодаря этому `web_sys` всегда соответствует актуальным спецификациям: достаточно обновить WebIDL-файлы и перегенерировать crate.

## 3. Почему `web_sys` содержит огромное количество типов

Откройте документацию `web_sys` на docs.rs — и вы увидите тысячи элементов: `AbortController`, `AesCbcParams`, `AnalyserNode`, `AnimationEvent`… Список начинается с `A` и заканчивается на `XrSession`. Это не потому, что crate «раздут». Это потому, что **сам Web Platform огромен** — и `web_sys` отражает его один в один.

Каждый браузерный интерфейс, каждая словарная структура (dictionary), каждый enum из WebIDL превращаются в отдельный Rust-тип. Более того, поскольку WebIDL поддерживает **частичные определения** (partial interfaces, mixins), которые могут быть разбросаны по нескольким спецификациям, система генерации собирает их в единое целое.

Важно понимать: **`web_sys` не пытается быть удобным**. Он не группирует API по «модулям» вроде `dom`, `net`, `media`. Он даёт прямой, плоский, механический доступ ко всему, что есть в Web Platform. Удобство — задача других слоёв (например, `Emerge Core` из части XVIII).

## 4. Classes, dictionaries и enums

WebIDL различает три категории типов, и `web_sys` отражает это различие.

**Classes (интерфейсы)** — это объекты, у которых есть методы и атрибуты. В Rust они представлены как структуры, обёрнутые в `#[wasm_bindgen]`. Пример: `Window`, `Document`, `HtmlElement`, `Blob`, `File`. Это те типы, с которыми вы работаете напрямую.

**Dictionaries** — это конфигурационные объекты, которые передаются в методы, но не имеют собственного поведения. Например, `RequestInit` (параметры для `fetch`), `AddEventListenerOptions`, `ShadowRootInit`. В Rust они представлены как структуры с публичными полями и обычно реэкспортируются из `js_sys` для полей типа `JsValue`.

```rust
// Пример dictionary
#[wasm_bindgen]
pub struct RequestInit {
    pub method: Option<String>,
    pub headers: Option<js_sys::Object>,
    pub body: Option<JsValue>,
    // ...
}
```

**Enums** — это перечисления, которые WebIDL определяет как набор именованных значений. Например, `BinaryType` (для `WebSocket`), `ShadowRootMode`, `GpuFeatureName`. В Rust они представлены как `#[non_exhaustive]` enum'ы.

```rust
#[non_exhaustive]
pub enum ShadowRootMode {
    Open = 0,
    Closed = 1,
}
```

Различие важно на практике: **classes вы вызываете**, **dictionaries вы конструируете**, **enums вы передаёте**. Документация `web_sys` всегда помечает, к какой категории относится тип.

## 5. Re-exports `wasm_bindgen` и `js_sys`

`web_sys` не изолирован. В `src/lib.rs` есть две важные строки:

```rust
pub use js_sys;
pub use wasm_bindgen;
```

Это **реэкспорты**. Они означают, что если ваш crate зависит от `web-sys`, вы можете использовать `web_sys::wasm_bindgen` и `web_sys::js_sys` без добавления их в `Cargo.toml`. Это не просто удобство — это гарантия того, что версии будут совместимы.

Зачем это нужно? Потому что `web_sys` сильно зависит от `wasm_bindgen` (для генерации привязок) и от `js_sys` (для типов вроде `Array`, `Object`, `Promise`). Если бы вы добавляли `wasm-bindgen` вручную, вы могли бы случайно указать несовместимую версию. Реэкспорт решает эту проблему: используйте `web_sys::wasm_bindgen`, и версия всегда будет правильной.

На практике это означает, что типичный `use`-блок выглядит так:

```rust
use web_sys::{Window, Document, HtmlElement};
use web_sys::wasm_bindgen::prelude::*;
use web_sys::js_sys::{Array, Promise};
```

## 6. Cargo features

`web_sys` использует **Cargo features** для управления тем, какие API доступны. Правило простое: **каждый тип имеет свою feature, названную в честь этого типа**。

Например:

| Feature | Что включает |
|---|---|
| `Window` | Тип `Window` и связанные функции |
| `Document` | Тип `Document` |
| `HtmlElement` | Тип `HtmlElement` |
| `Element` | Тип `Element` |
| `Node` | Тип `Node` |
| `EventTarget` | Тип `EventTarget` |
| `Blob` | Тип `Blob` |
| `File` | Тип `File` |

Features **наследуются**. Например, feature `HtmlElement` зависит от `Element`, которая зависит от `Node`, которая зависит от `EventTarget`. Если вы включите `HtmlElement`, Cargo автоматически включит всю цепочку.

Чтобы использовать метод, вам нужно включить features **всех типов**, которые в нём участвуют. Например, `Window::resize_to` требует feature `Window`. А `Document::create_element` требует `Document` и `Element` (поскольку возвращает `Result<Element, JsValue>`).

## 7. Почему API отключены по умолчанию

`web_sys` по умолчанию **почти пуст**. Это не случайность и не ошибка конфигурации — это осознанное решение.

Причина первая: **время компиляции**. Web Platform содержит тысячи типов. Если бы все они компилировались по умолчанию, каждая сборка проекта занимала бы минуты, даже если вы используете только `Window` и `Document`.

Причина вторая: **размер бинарника**. Wasm-модуль, который включает привязки ко всему Web Platform, будет содержать код, который никогда не вызывается. Dead code elimination в Wasm не всегда может убрать неиспользуемые привязки, потому что они являются частью публичного API crate'а.

Причина третья: **стабильность**. Некоторые API (`web_sys_unstable_apis`) помечены как нестабильные и требуют явного `--cfg=web_sys_unstable_apis`. Включать их по умолчанию было бы опасно.

Именно поэтому в `Cargo.toml` вы явно перечисляете то, что используете:

```toml
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

Это не «неудобство», а **инструмент контроля**. Вы точно знаете, какие API использует ваш проект, и сборка остаётся быстрой.

## 8. Как читать документацию `web_sys`

Документация `web_sys` на docs.rs — это основной инструмент работы. Она устроена следующим образом.

**Каждый тип** имеет страницу со списком методов, ассоциированных функций и трейтов. **Каждый метод** помечен блоком, который указывает, какие Cargo features нужно включить:

> *This API requires the following crate features to be activated: `Document`, `Element`, `Node`*

Это самый важный элемент документации. Если вы видите ошибку компиляции «no method named `create_element` found», почти наверняка вы забыли включить feature `Document` или `Element`.

**Навигация по типам** — через поиск. Введите в поиске docs.rs `resizeTo` — и вы найдёте `Window::resize_to`. Введите `createElement` — найдёте `Document::create_element`.

**Структура страницы типа** следует WebIDL:

- **Methods** — методы, которые вы вызываете у объекта
- **Trait Implementations** — `JsCast`, `AsRef`, `From`, `Into` и другие
- **Deref Target** — на какой тип этот тип дереференсится (например, `HtmlElement` дереференсится на `Element`)

Практический совет: **не пытайтесь запомнить features**. Открывайте документацию для каждого метода, который используете, и копируйте список required features в `Cargo.toml`.

## 9. Как определить необходимые features

Есть несколько стратегий определения features.

**Стратегия «от метода».** Вы знаете, что хотите вызвать `document.getElementById()`. Ищете `getElementById` в docs.rs. Находите `Document::get_element_by_id`. В документации указано: features `Document`, `Element`. Добавляете их.

**Стратегия «от типа».** Вы знаете, что работаете с `HtmlInputElement`. Открываете его страницу. В документации указано, что для него нужна feature `HtmlInputElement`, а также `HtmlElement`, `Element`, `Node`, `EventTarget` (цепочка наследования). Добавляете все.

**Стратегия «от компилятора».** Пишете код, компилируете, получаете ошибку. В сообщении об ошибке Cargo часто указывает, какой feature отсутствует. Это итеративный, но надёжный подход для сложных случаев.

**Стратегия «из Crates.io».** Если вы используете чужой проект, откройте его `Cargo.toml` и посмотрите, какие features `web-sys` он включает. Это отправная точка, но не гарантия, что вам нужны все они.

## 10. Feature unification, workspace features и как не превратить `Cargo.toml` в свалку

Здесь начинается самое интересное — и самое опасное.

**Feature unification** — это механизм Cargo, который объединяет features всех crates в dependency graph. Если crate A включает `web-sys` с feature `Window`, а crate B — с feature `Document`, то итоговый бинарник получит **оба** feature. Это работает и в обратную сторону: если вы не хотите, чтобы какая-то feature была включена, вы не можете её «отключить», если её требует другой crate.

Практическое следствие: **не полагайтесь на то, что features изолированы**. Если вы пишете библиотеку, которая использует `web-sys`, и ваш пользователь тоже использует `web-sys`, features объединятся.

**Unstable APIs.** Некоторые API (`WebGPU`, `WebCodecs`, экспериментальные части `File System Access`) скрыты за `#[cfg(web_sys_unstable_apis)]`. Чтобы их включить, нужно добавить в `.cargo/config.toml`:

```toml
[build]
rustflags = ["--cfg=web_sys_unstable_apis"]
```

Это не Cargo feature, а **cfg-флаг компилятора**. Он не объединяется с другими features и не виден в `Cargo.toml`. Это важно: если ваша библиотека использует unstable API, пользователь должен сам добавить этот флаг — иначе код не скомпилируется.

**Как не превратить `Cargo.toml` в свалку.** Несколько практических правил:

1. **Группируйте features по назначению.** Не сваливайте всё в один список. Разделите на секции: DOM, Events, Network, Storage.

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    # DOM
    "Window", "Document", "Element", "HtmlElement", "Node",
    # Events
    "Event", "EventTarget", "MouseEvent", "KeyboardEvent",
    # Network
    "Request", "Response", "Headers", "WebSocket",
]
```

2. **Используйте `default-features = false`.** Это отключает features по умолчанию, которые `web_sys` всё равно почти не имеет, но полезно как явное указание.

3. **Выносите features в отдельные секции для dev и prod.** Например, `WebSocket` может быть нужен только в production, а `console` — только в dev.

4. **Не добавляйте features «на всякий случай».** Каждая feature увеличивает время компиляции и потенциально — размер бинарника. Если метод не используется, feature не нужна.

5. **Проверяйте с `cargo tree`.** Если вы не уверены, какая feature откуда пришла, используйте `cargo tree -e features` — он покажет, какой crate включает какую feature.

**Workspace features.** Если вы работаете в Cargo workspace, features можно определять на уровне workspace и наследовать в дочерних crates. Это удобно для больших проектов, но требует дисциплины: убедитесь, что каждый crate включает только те features, которые ему действительно нужны.

---

**Итог главы:**

- `web_sys` генерируется из **WebIDL** — формального языка описания браузерных API.
- Генерация **автоматическая**, потому что Web Platform слишком велик для ручного поддержания.
- Типы делятся на **classes** (объекты с методами), **dictionaries** (конфигурации) и **enums** (перечисления).
- `web_sys` **реэкспортирует** `wasm_bindgen` и `js_sys`, чтобы гарантировать совместимость версий.
- **Cargo features** — основной механизм управления API: каждый тип имеет свою feature.
- API **отключены по умолчанию**, чтобы не тратить время компиляции и размер бинарника.
- Документация docs.rs **указывает required features** для каждого метода — это главный инструмент.
- **Feature unification** может неожиданно включать features, которые вы не просили. Планируйте `Cargo.toml` осознанно.
- **Unstable APIs** требуют `--cfg=web_sys_unstable_apis` — это не Cargo feature, а флаг компилятора.

В следующей главе мы создадим первый проект с `web_sys` и пройдём весь путь от `Cargo.toml` до работающего Wasm-модуля в браузере.