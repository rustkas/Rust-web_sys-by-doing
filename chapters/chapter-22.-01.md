# Глава 22.1. OffscreenCanvas: canvas вне DOM

## 22.1.0. Обновлённая структура

```
src/
  offscreen_canvas.rs             ← НОВЫЙ модуль 22.1
  offscreen_worker.rs             ← из главы 22 (не трогаем)
  canvas_dpi.rs … canvas_image_pipeline.rs
tests/
  wasm_offscreen_canvas.rs        ← НОВЫЙ тест
src/lib.rs                        ← +1 строка
src/app.rs                        ← демо 22.1 (заменяется целиком)
web/public/workers/
  oc_circle_worker.js             ← НОВЫЙ JS-воркер
Cargo.toml                        ← без правок (фичи OffscreenCanvas уже есть)
web/styles/main.css               ← без правок
```

**Комментарий.** Модуль `offscreen_canvas.rs` — про сам объект `OffscreenCanvas`: два способа создания, получение 2D-контекста, размеры буфера, одноразовость передачи, опциональные методы (`convertToBlob`, `transferToImageBitmap`). Префикс CSS — `oc-` (offscreen canvas), не пересекается с `ow-` из главы 22. Фокус подглавы — **сам canvas вне DOM**, без обмена сообщениями и transferable-данных: это отдельная тема. Демо сравнивает два холста — рисование из main thread и рисование из Worker'а — и показывает, что происходит с каждым, когда main thread занят.

**Ссылки для детального изучения:**

- MDN: [`OffscreenCanvas`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
- MDN: [`OffscreenCanvas()` constructor](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/OffscreenCanvas)
- MDN: [`HTMLCanvasElement.transferControlToOffscreen`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/transferControlToOffscreen)
- MDN: [`OffscreenCanvas.getContext`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/getContext)
- MDN: [`OffscreenCanvas.convertToBlob`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/convertToBlob)
- MDN: [`OffscreenCanvas.transferToImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas/transferToImageBitmap)
- web.dev: [OffscreenCanvas — Speed up your canvas operations](https://web.dev/articles/offscreen-canvas)

---

## 22.1.1. О чём эта подглава

Всё, что мы рисовали до сих пор — с 21.1 по 21.7 — рисовалось в **main thread**. `getContext("2d")` на `<canvas>`, цикл `for` по пикселям, `putImageData`, `requestAnimationFrame`. Кнопки, скролл, ввод — всё живёт в том же потоке. Пока задача маленькая, разница незаметна. Когда задача большая — пользователь видит «зависшую страницу».

Решение — **Web Worker**: отдельный поток со своим event loop. Но у Worker'а нет DOM, значит, нет `<canvas>` — а без canvas рисовать некуда. Именно эту проблему решает `OffscreenCanvas`.

`OffscreenCanvas` — это canvas **без DOM**. Его можно создать двумя способами:

- `new OffscreenCanvas(w, h)` — «виртуальный» холст, который живёт только в памяти. Полезен для пиксельной обработки без показа (например, генерация превью, рисование на скрытом буфере).
- `canvas.transferControlToOffscreen()` — отбирает управление у существующего DOM-`<canvas>`. После этого содержимым холста управляет не main thread, а тот, кому передан `OffscreenCanvas` (обычно Worker).

Ключевые свойства:

- `width` / `height` — это **размеры буфера**, а не CSS. CSS-размер задаётся отдельно и не влияет на разрешение рисования.
- `getContext("2d")` работает так же, как на обычном canvas.
- `convertToBlob()` и `transferToImageBitmap()` — дополнительные методы, которых нет у DOM-canvas.
- `transferControlToOffscreen()` — **одноразовая** операция. Второй вызов на том же элементе выбросит `InvalidStateError`.

Подглава отвечает на три вопроса:

**Первый — зачем вообще OffscreenCanvas.** Чтобы отдать рисование Worker'у и не блокировать main thread.

**Второй — в чём разница между `new` и `transferControlToOffscreen`.** Первый — «мой собственный холст, я его никому не показываю». Второй — «отдаю управление моим холстом, но он остаётся видимым на странице».

**Третий — как понять, что main thread освободился.** Демо сравнивает FPS на двух холстах: один рисует main thread, другой — Worker через OffscreenCanvas. Кнопка «занять main thread на 2 секунды» блокирует его; левый холст замирает, правый продолжает рисовать.

---

## 22.1.2. Что уже есть и что появится

К концу 21.7 умеем:
- рисовать на `<canvas>` в main thread;
- применять Rust-фильтры к `ImageData`;
- загружать изображения через `ImageBitmap`;
- организовывать длительные операции с yield через `TimeoutFuture`.

К концу 22.1 будем уметь:
- создавать `OffscreenCanvas` двумя способами;
- получать на нём 2D-контекст;
- передавать DOM-canvas Worker'у через `transferControlToOffscreen`;
- видеть разницу «main thread рисует» vs «Worker рисует»;
- понимать, почему `transferControlToOffscreen` одноразовый.

---

## 22.1.3. Два способа создать OffscreenCanvas

**`new OffscreenCanvas(w, h)`** — конструктор. Возвращает объект, который не привязан к DOM. Ни `.style`, ни `.parentNode`, ни `clientWidth`. Всё, что есть — `width`, `height`, `getContext`, `convertToBlob`, `transferToImageBitmap`.

```js
const off = new OffscreenCanvas(400, 300);
const ctx = off.getContext("2d");
ctx.fillStyle = "#58a6ff";
ctx.fillRect(0, 0, 400, 300);
// Холст не виден ни на одной странице.
```

Используется, когда результат не нужно показывать пользователю прямо сейчас: подготовка изображения перед отправкой, генерация текстуры, промежуточный буфер.

**`canvas.transferControlToOffscreen()`** — вызывается на существующем DOM-`<canvas>`:

```js
const canvas = document.querySelector("#my-canvas");
const off = canvas.transferControlToOffscreen();
// off.width === canvas.width, off.height === canvas.height
// canvas.getContext("2d") теперь выбрасывает исключение
```

Возвращает `OffscreenCanvas`, который **делит буфер** с DOM-элементом. Всё, что рисуется на `off`, немедленно появляется в `<canvas>` на странице. Но main thread больше не может вызывать `getContext` на оригинальном элементе — управление передано.

---

## 22.1.4. width / height — это буфер, а не CSS

Размеры canvas двусмысленны, и это частая ошибка.

```html
<canvas id="c" width="400" height="300" style="width: 800px;"></canvas>
```

Здесь:
- `canvas.width === 400` — размер **буфера** в пикселях. Это разрешение рисования.
- `getBoundingClientRect().width === 800` — CSS-размер отображения.
- Браузер растянет буфер 400 px до 800 CSS px. Пиксели будут размытыми.

У `OffscreenCanvas` есть только `width`/`height`. Он не отображается сам по себе — либо передаётся Worker'у (тогда наследует буфер DOM-элемента), либо живёт в памяти.

Правило: **для резких картинок на retina-экране** буфер должен быть больше CSS-размера. Например, CSS 400 px, буфер 800 px, `ctx.scale(2, 2)`. Это ровно то, чем занимается модуль `canvas_dpi.rs` из 21.1.

---

## 22.1.5. Одноразовость transferControlToOffscreen

Ключевая деталь: `transferControlToOffscreen` можно вызвать на элементе **ровно один раз**.

```js
const canvas = document.querySelector("#c");
const off1 = canvas.transferControlToOffscreen();  // ok
const off2 = canvas.transferControlToOffscreen();  // InvalidStateError
```

Причина: после передачи canvas у main thread'а больше нет контроля над буфером. Он «принадлежит» тому, кто получил `OffscreenCanvas`. Пока Worker жив, никто другой не может вклиниться.

Практическое следствие: **передавайте canvas только тогда, когда Worker готов его принять.** Если создать Worker, передать canvas, а потом Worker упал — вернуть управление main thread'у уже нельзя. Придётся создавать новый `<canvas>`.

Исключение: `Worker.terminate()` уничтожает Worker, но `OffscreenCanvas` уходит вместе с ним — DOM-элемент становится бесполезным (в нём застыл последний кадр, `getContext` всё ещё бросает исключение).

---

## 22.1.6. convertToBlob и transferToImageBitmap

Два дополнительных метода `OffscreenCanvas`, которых нет у DOM-версии.

**`convertToBlob(options?)`** — асинхронно превращает содержимое холста в `Blob`. Аргументы: `type` (`"image/png"`, `"image/jpeg"`, `"image/webp"`), `quality` (0…1). Возвращает `Promise<Blob>`.

```js
const blob = await off.convertToBlob({ type: "image/png" });
const url = URL.createObjectURL(blob);
```

Применение: экспорт кадра в файл без показа canvas пользователю.

**`transferToImageBitmap()`** — синхронно возвращает `ImageBitmap` с текущим содержимым. Буфер холста очищается. `ImageBitmap` — transferable: его можно отправить в Worker без копирования.

```js
const bitmap = off.transferToImageBitmap();
worker.postMessage({ bitmap }, [bitmap]);  // zero-copy
```

Применение: быстрая передача кадра между Worker'ами в конвейере рендера.

Оба метода — опциональные (не все браузеры). Их наличие проверяется feature detection (см. модуль `offscreen_canvas.rs`).

---

## 22.1.7. Что показывает демо

**Layout:**

- Два холста рядом: **левый** — рисует main thread через `requestAnimationFrame`; **правый** — передан Worker'у через `transferControlToOffscreen`, рисует Worker.
- Оба холста показывают одинаковую анимацию: пять пульсирующих кругов, движущихся по круговой орбите.
- Кнопка **«Занять main thread на 2 секунды»** — запускает busy-loop в main thread.
- Панель статистики: FPS main thread'а, FPS Worker'а, статус передачи.

**Сценарий:**

1. При загрузке страницы оба холста начинают анимацию. Оба показывают стабильные 60 FPS.
2. Пользователь нажимает кнопку. Main thread блокируется на 2 секунды: **левый холст замирает**, CSS-анимации страницы тоже замирают, **правый продолжает рисовать круги** — его rAF в Worker'е не зависит от main thread.
3. Через 2 секунды main thread освобождается, левый холст «оживает». Оба FPS снова 60.

**Почему это наглядно.**

- **Первый факт.** Правый холст действительно рисует Worker. Его rAF живёт в отдельном event loop.
- **Второй факт.** Main thread действительно блокируется. Левый холст и любой CSS-индикатор на странице останавливаются.
- **Третий факт.** `transferControlToOffscreen` действительно отдаёт управление. `getContext("2d")` на правом `<canvas>` из main thread'а больше не работает — что и делает Worker единственным владельцем буфера.

**Что осознанно не реализовано.**

- **Обмен командами через `postMessage`.** Это тема 22.2–22.3, здесь — только передача canvas'а.
- **Transferable-данные.** В 22.3.
- **WASM в Worker'е.** В 22.4.

Подглава намеренно узкая: **сам OffscreenCanvas и его отличие от DOM-canvas**.

---

## 22.1.8. Wow-момент

**Первый.** Нажмите «Занять main thread». Левый холст мгновенно замирает — застывший кадр с кругами. Правый продолжает анимацию, круги двигаются. Один поток — два независимых источника кадров.

**Второй.** Откройте DevTools → Performance. Запустите запись, нажмите кнопку, подождите. Профиль main thread'а: 2-секундный плоский красный блок. Профиль Worker'а: непрерывные маленькие зелёные всплески на 60 FPS. Задача анимации **не в main thread** — она в Worker'е.

**Третий.** В консоли DevTools выполните `document.getElementById("oc-worker-canvas").getContext("2d")`. Получите `InvalidStateError`. Управление передано. Это не свойство DOM-элемента, а состояние, зафиксированное в движке.

**Четвёртый.** Остановите Worker через DevTools (`Worker.terminate()` в sources panel). Правый холст замирает навсегда. `transferControlToOffscreen` необратим.

---

## 22.1.9. Чему это учит

**Первое.** `OffscreenCanvas` — это **полноценный canvas**, просто без DOM. Все операции рисования работают так же. Разница только в способе отображения.

**Второе.** `transferControlToOffscreen` — **передача владения**, а не копирование. Буфер один, владелец один. DOM-элемент остаётся видимым, но перестаёт быть «программируемым» из main thread'а.

**Третье.** `width`/`height` — это размеры **буфера**. CSS-размер — отдельная величина. Смешивать их — источник размытых картинок.

**Четвёртое.** Одноразовость передачи — **осознанное ограничение**, а не баг. Без него несколько потоков могли бы писать в один буфер, что породило бы гонки.

**Пятое.** Опциональные методы (`convertToBlob`, `transferToImageBitmap`) нужно проверять feature detection. Их нет во всех браузерах.

---

## 22.1.10. Исходные файлы

### `src/offscreen_canvas.rs`

```rust
//! 22.1. OffscreenCanvas: canvas вне DOM.
//!
//! Обёртки вокруг `OffscreenCanvas` и его API:
//!
//! - **feature detection** — поддерживает ли браузер `OffscreenCanvas`
//!   и `HtmlCanvasElement::transferControlToOffscreen`;
//! - **два способа создания** — `new OffscreenCanvas(w, h)` и
//!   `canvas.transferControlToOffscreen()`;
//! - **получение 2D-контекста** — через `getContext("2d")`;
//! - **размеры буфера** — `width`/`height` в пикселях, не CSS;
//! - **опциональные методы** — `convertToBlob`, `transferToImageBitmap`.
//!
//! ## Утилита `bool_of`
//!
//! rust-analyzer стабильно теряет тип `matches!` в tail-позиции
//! функции: `expected bool, found ()`. Обёртка `bool_of(matches!(…))`
//! с явной сигнатурой возвращает типу конкретность. На кодогенерацию
//! не влияет — `#[inline]` + identity-функция компилируются в ничто.

use js_sys::{Function, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    HtmlCanvasElement, OffscreenCanvas, OffscreenCanvasRenderingContext2d,
};

// =============================================================================
// Утилита для обхода бага rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `matches!` для tail-позиций.
///
/// rust-analyzer теряет тип `matches!` в хвостовой позиции функции
/// и выдаёт `expected bool, found ()`. Обёртка с явной сигнатурой
/// возвращает типу конкретность и снимает ошибку E0308.
#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Feature detection
// =============================================================================

/// Глобальный конструктор `OffscreenCanvas` доступен.
pub fn is_offscreen_canvas_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("OffscreenCanvas")),
        Ok(v) if v.is_function()
    ))
}

/// `HtmlCanvasElement::transferControlToOffscreen` доступен.
pub fn is_transfer_control_supported() -> bool {
    let global = js_sys::global();
    let Ok(document) = Reflect::get(&global, &JsValue::from_str("document")) else {
        return false;
    };
    let Ok(create) = Reflect::get(&document, &JsValue::from_str("createElement")) else {
        return false;
    };
    let Ok(create) = create.dyn_into::<Function>() else {
        return false;
    };
    let Ok(canvas) = create.call1(&document, &JsValue::from_str("canvas")) else {
        return false;
    };
    bool_of(matches!(
        Reflect::get(&canvas, &JsValue::from_str("transferControlToOffscreen")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Два способа создания
// =============================================================================

/// Создаёт `OffscreenCanvas` напрямую, без DOM-элемента.
///
/// Используется, когда холст нужен только для пиксельной обработки
/// и не показывается пользователю.
pub fn new_offscreen(width: u32, height: u32) -> Result<OffscreenCanvas, JsValue> {
    OffscreenCanvas::new(width, height)
}

/// Отзывает управление у DOM-canvas'а и возвращает `OffscreenCanvas`.
///
/// Одноразовая операция: повторный вызов на том же элементе даёт
/// `InvalidStateError`.
pub fn transfer_control(canvas: &HtmlCanvasElement) -> Result<OffscreenCanvas, JsValue> {
    canvas.transfer_control_to_offscreen()
}

// =============================================================================
// Контекст
// =============================================================================

/// Получает 2D-контекст у `OffscreenCanvas`.
///
/// `OffscreenCanvas::get_context` возвращает `Object` — конкретный тип
/// контекста зависит от строки. Приводим через `dyn_into`.
pub fn get_2d(
    canvas: &OffscreenCanvas,
) -> Result<OffscreenCanvasRenderingContext2d, JsValue> {
    let ctx = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("2d context unavailable"))?;
    ctx.dyn_into::<OffscreenCanvasRenderingContext2d>()
        .map_err(|_| JsValue::from_str("not an OffscreenCanvasRenderingContext2d"))
}

// =============================================================================
// Размеры буфера
// =============================================================================

/// Ширина буфера в пикселях. **Не** CSS-размер.
pub fn width_of(canvas: &OffscreenCanvas) -> u32 {
    canvas.width()
}

/// Высота буфера в пикселях. **Не** CSS-размер.
pub fn height_of(canvas: &OffscreenCanvas) -> u32 {
    canvas.height()
}

// =============================================================================
// Опциональные методы
// =============================================================================

/// `convertToBlob` доступен на этом `OffscreenCanvas`.
///
/// Метод не во всех браузерах (Safari добавил позже), поэтому
/// наличие проверяется через Reflect.
pub fn has_convert_to_blob(canvas: &OffscreenCanvas) -> bool {
    bool_of(matches!(
        Reflect::get(canvas.as_ref(), &JsValue::from_str("convertToBlob")),
        Ok(v) if v.is_function()
    ))
}

/// `transferToImageBitmap` доступен на этом `OffscreenCanvas`.
pub fn has_transfer_to_image_bitmap(canvas: &OffscreenCanvas) -> bool {
    bool_of(matches!(
        Reflect::get(canvas.as_ref(), &JsValue::from_str("transferToImageBitmap")),
        Ok(v) if v.is_function()
    ))
}
```

### `src/lib.rs` — добавление

Добавьте одну строку рядом с `offscreen_worker`:

```rust
pub mod offscreen_worker;   // 22
pub mod offscreen_canvas;   // ← 22.1
```

### `tests/wasm_offscreen_canvas.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::HtmlCanvasElement;

use hello_wasm02::offscreen_canvas::{
    get_2d, has_convert_to_blob, has_transfer_to_image_bitmap, height_of,
    is_offscreen_canvas_supported, is_transfer_control_supported, new_offscreen,
    transfer_control, width_of,
};

wasm_bindgen_test_configure!(run_in_browser);

fn make_canvas() -> HtmlCanvasElement {
    let doc = web_sys::window().unwrap().document().unwrap();
    doc.create_element("canvas")
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()
        .unwrap()
}

#[wasm_bindgen_test]
fn offscreen_canvas_is_supported_in_browser() {
    assert!(
        is_offscreen_canvas_supported(),
        "OffscreenCanvas should be available"
    );
}

#[wasm_bindgen_test]
fn transfer_control_is_supported_in_browser() {
    assert!(
        is_transfer_control_supported(),
        "transferControlToOffscreen should be available"
    );
}

#[wasm_bindgen_test]
fn new_offscreen_has_correct_dimensions() {
    let off = new_offscreen(100, 50).expect("new OffscreenCanvas");
    assert_eq!(width_of(&off), 100);
    assert_eq!(height_of(&off), 50);
}

#[wasm_bindgen_test]
fn transfer_control_returns_same_dimensions() {
    let canvas = make_canvas();
    canvas.set_width(320);
    canvas.set_height(240);
    let off = transfer_control(&canvas).expect("transfer");
    assert_eq!(width_of(&off), 320);
    assert_eq!(height_of(&off), 240);
}

#[wasm_bindgen_test]
fn second_transfer_control_fails() {
    let canvas = make_canvas();
    canvas.set_width(64);
    canvas.set_height(64);
    let _ = transfer_control(&canvas).expect("first transfer");
    let second = transfer_control(&canvas);
    assert!(second.is_err(), "second transfer must fail");
}

#[wasm_bindgen_test]
fn get_2d_on_transferred_works() {
    let canvas = make_canvas();
    canvas.set_width(64);
    canvas.set_height(64);
    let off = transfer_control(&canvas).expect("transfer");
    let _ctx = get_2d(&off).expect("2d context on transferred");
}

#[wasm_bindgen_test]
fn get_2d_on_new_works() {
    let off = new_offscreen(64, 64).expect("new");
    let _ctx = get_2d(&off).expect("2d context on new");
}

#[wasm_bindgen_test]
fn optional_apis_are_detected() {
    let off = new_offscreen(32, 32).expect("new");
    // Наличие методов зависит от браузера; тест проверяет только то,
    // что detection не паникует и возвращает bool.
    let _ = has_convert_to_blob(&off);
    let _ = has_transfer_to_image_bitmap(&off);
}
```

### `web/public/workers/oc_circle_worker.js`

```javascript
// 22.1. Circle worker.
//
// Worker владеет OffscreenCanvas, переданным через
// transferControlToOffscreen. Рисует ту же анимацию пульсирующих
// кругов, что и main thread, и отправляет FPS в main thread.
//
// Никаких команд, кроме init: подглава про сам OffscreenCanvas,
// а не про обмен сообщениями.

let ctx = null;
let width = 0;
let height = 0;
let frames = 0;
let lastFpsAt = 0;

function draw(now) {
    requestAnimationFrame(draw);
    if (!ctx) return;

    ctx.fillStyle = "#0d1117";
    ctx.fillRect(0, 0, width, height);
    ctx.lineWidth = 4;

    const cx = width / 2;
    const cy = height / 2;
    const t = now / 1000;

    for (let i = 0; i < 5; i++) {
        const offset = i * 0.5;
        const r = 20 + Math.abs(Math.sin(t + offset)) * 60;
        const angle = t * 0.7 + offset;
        const px = cx + Math.cos(angle) * 70;
        const py = cy + Math.sin(angle) * 70;
        const hue = (i * 60 + (t * 40) % 360) % 360;

        ctx.beginPath();
        ctx.arc(px, py, r, 0, Math.PI * 2);
        ctx.strokeStyle = `hsl(${hue.toFixed(0)}, 70%, 60%)`;
        ctx.stroke();
    }

    frames++;
    if (!lastFpsAt) lastFpsAt = now;
    if (now - lastFpsAt >= 500) {
        const fps = (frames * 1000) / (now - lastFpsAt);
        self.postMessage({ type: "fps", fps });
        frames = 0;
        lastFpsAt = now;
    }
}

self.onmessage = (e) => {
    const msg = e.data;
    if (msg.type === "init") {
        width = msg.width;
        height = msg.height;
        ctx = msg.canvas.getContext("2d");
        requestAnimationFrame(draw);
    }
};
```

### Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Wasm-тесты только для этой подглавы
wasm-pack test --headless --chrome --test wasm_offscreen_canvas

# 3. Демо
npm run wasm:dev && npm start
```

Убедитесь, что файл `web/public/workers/oc_circle_worker.js` доступен по адресу `/workers/oc_circle_worker.js`. Vite монтирует папку `public` в корень автоматически.

---

## 22.1.11. `src/app.rs` — автономный демо-пример

```rust
//! Глава 22.1 — OffscreenCanvas: canvas вне DOM.
//!
//! Демо сравнивает два холста рядом:
//!
//! - **левый** — рисует main thread через `requestAnimationFrame`;
//! - **правый** — передан Worker'у через `transferControlToOffscreen`,
//!   Worker сам управляет анимацией.
//!
//! Кнопка «Занять main thread на 2 секунды» блокирует main thread.
//! Левый холст замирает, правый продолжает рисовать. FPS-панель
//! показывает числа обоих потоков.
//!
//! ## Что демонстрируется
//!
//! - `canvas.transferControlToOffscreen()` — передача управления
//!   DOM-элементом Worker'у.
//! - Одноразовость передачи — после неё `getContext` на оригинале
//!   бросает `InvalidStateError`.
//! - Разница поведения: main thread vs Worker.
//! - `width`/`height` = размеры буфера, не CSS.
//!
//! ## Правило Rc<RefCell<State>>
//!
//! `borrow()` и `borrow_mut()` не пересекаются. Каждый borrow —
//! в узком scope, `Ref`/`RefMut` drop'аются до следующего borrow.
//!
//! **Файл автономен.** Стили `.oc-*` инжектируются в документ.
//! Wow: нажмите «Занять main thread на 2 секунды» — левый холст
//! застынет, правый продолжит рисовать.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use js_sys::{Array, Function, Object, Reflect};
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    CanvasRenderingContext2d, Document, HtmlCanvasElement, HtmlElement,
    OffscreenCanvas, Worker,
};

// =============================================================================
// Утилиты для обхода багов rust-analyzer в tail-позиции
// =============================================================================

/// Обёртка над `format!` для tail-позиций. См. `format_str.rs`.
#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

/// Обёртка над `matches!` для tail-позиций.
///
/// rust-analyzer теряет тип `matches!` в хвостовой позиции функции
/// и выдаёт `expected bool, found ()`. Обёртка с явной сигнатурой
/// возвращает типу конкретность и снимает ошибку E0308.
/// На кодогенерацию не влияет — `#[inline]` + identity-функция.
#[inline]
fn bool_of(value: bool) -> bool {
    value
}

// =============================================================================
// Инжекция стилей
// =============================================================================

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "oc-injected-style";
    if let Some(old) = document.get_element_by_id(id) {
        if let Some(parent) = old.parent_node() {
            parent.remove_child(&old)?;
        }
    }
    let style = document.create_element("style")?;
    style.set_id(id);
    style.set_text_content(Some(STYLE_CSS));
    let head = document
        .head()
        .ok_or_else(|| JsValue::from_str("no <head>"))?;
    head.append_child(&style)?;
    Ok(())
}

const STYLE_CSS: &str = r#"
.oc-desc {
    max-width: 900px;
    margin: 0 0 20px 0;
    padding: 12px 16px;
    background: #161b22;
    border-left: 3px solid #58a6ff;
    border-radius: 4px;
    color: #c9d1d9;
    font-size: 14px;
    line-height: 1.55;
}
.oc-desc-line { margin: 6px 0; }

.oc-row {
    display: flex;
    gap: 16px;
    flex-wrap: wrap;
    margin-bottom: 16px;
    max-width: 900px;
}
.oc-col {
    flex: 1 1 320px;
    min-width: 280px;
    display: flex;
    flex-direction: column;
}
.oc-caption {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    margin-bottom: 6px;
}
.oc-canvas {
    display: block;
    width: 100%;
    aspect-ratio: 4 / 3;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 8px;
}

.oc-block {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
}

.oc-btn {
    padding: 8px 14px;
    border-radius: 6px;
    border: 1px solid #30363d;
    background: #21262d;
    color: #c9d1d9;
    cursor: pointer;
    font-size: 12px;
    font-weight: 500;
    transition: background 0.15s ease;
}
.oc-btn:hover:not([disabled]) { background: #30363d; }
.oc-btn:active:not([disabled]) { transform: translateY(1px); }
.oc-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.oc-btn-warn {
    background: #21262d;
    border-color: #30363d;
    color: #f0883e;
}
.oc-btn-warn:hover:not([disabled]) { background: #30363d; }

.oc-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
}

.oc-stats {
    max-width: 900px;
    padding: 10px 14px;
    background: #0d1117;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    line-height: 1.7;
    margin-bottom: 16px;
}
.oc-stats .label { color: #8b949e; }
.oc-stats .value { color: #79c0ff; }
.oc-stats .ok { color: #3fb950; }
.oc-stats .warn { color: #f0883e; }

.oc-footnote {
    max-width: 900px;
    padding: 10px 14px;
    background: #161b22;
    border-left: 3px solid #f0883e;
    border-radius: 4px;
    color: #8b949e;
    font-size: 12px;
    line-height: 1.55;
}
"#;

// =============================================================================
// Константы
// =============================================================================

const WORKER_URL: &str = "/workers/oc_circle_worker.js";
const CANVAS_W: u32 = 480;
const CANVAS_H: u32 = 360;
const BUSY_MS: f64 = 2000.0;

// =============================================================================
// Состояние демо
// =============================================================================

struct DemoState {
    worker: Option<Worker>,
    main_frames: u32,
    main_last_fps_at: f64,
    main_fps: f64,
    worker_fps: f64,
}

impl DemoState {
    fn new() -> Self {
        Self {
            worker: None,
            main_frames: 0,
            main_last_fps_at: 0.0,
            main_fps: 0.0,
            worker_fps: 0.0,
        }
    }
}

// =============================================================================
// Feature detection
// =============================================================================

fn is_offscreen_canvas_supported() -> bool {
    let global = js_sys::global();
    bool_of(matches!(
        Reflect::get(&global, &JsValue::from_str("OffscreenCanvas")),
        Ok(v) if v.is_function()
    ))
}

fn is_transfer_control_supported() -> bool {
    let global = js_sys::global();
    let Ok(document) = Reflect::get(&global, &JsValue::from_str("document")) else {
        return false;
    };
    let Ok(create) = Reflect::get(&document, &JsValue::from_str("createElement")) else {
        return false;
    };
    let Ok(create) = create.dyn_into::<Function>() else {
        return false;
    };
    let Ok(canvas) = create.call1(&document, &JsValue::from_str("canvas")) else {
        return false;
    };
    bool_of(matches!(
        Reflect::get(&canvas, &JsValue::from_str("transferControlToOffscreen")),
        Ok(v) if v.is_function()
    ))
}

// =============================================================================
// Worker
// =============================================================================

fn create_worker(url: &str) -> Result<Worker, JsValue> {
    Worker::new(url)
}

fn transfer_control(canvas: &HtmlCanvasElement) -> Result<OffscreenCanvas, JsValue> {
    canvas.transfer_control_to_offscreen()
}

fn post_init(
    worker: &Worker,
    canvas: &OffscreenCanvas,
    width: u32,
    height: u32,
) -> Result<(), JsValue> {
    let msg = Object::new();
    Reflect::set(&msg, &JsValue::from_str("type"), &JsValue::from_str("init"))?;
    Reflect::set(&msg, &JsValue::from_str("canvas"), canvas.as_ref())?;
    Reflect::set(
        &msg,
        &JsValue::from_str("width"),
        &JsValue::from_f64(width as f64),
    )?;
    Reflect::set(
        &msg,
        &JsValue::from_str("height"),
        &JsValue::from_f64(height as f64),
    )?;

    let transfer = Array::new();
    transfer.push(canvas.as_ref());
    worker.post_message_with_transfer(&msg, &transfer)
}

// =============================================================================
// Рисование
// =============================================================================

fn set_prop(target: &JsValue, key: &str, value: &JsValue) {
    let _ = Reflect::set(target, &JsValue::from_str(key), value);
}

/// Одна и та же анимация пульсирующих кругов для обоих холстов.
/// `now` — миллисекунды (как из `requestAnimationFrame`).
fn draw_circles(ctx: &CanvasRenderingContext2d, w: u32, h: u32, now: f64) {
    let ctx_js: &JsValue = ctx.as_ref();

    set_prop(ctx_js, "fillStyle", &JsValue::from_str("#0d1117"));
    ctx.fill_rect(0.0, 0.0, w as f64, h as f64);

    set_prop(ctx_js, "lineWidth", &JsValue::from_f64(4.0));

    let cx = w as f64 / 2.0;
    let cy = h as f64 / 2.0;
    let t = now / 1000.0;

    for i in 0..5 {
        let offset = i as f64 * 0.5;
        let r = 20.0 + (t + offset).sin().abs() * 60.0;
        let angle = t * 0.7 + offset;
        let px = cx + angle.cos() * 70.0;
        let py = cy + angle.sin() * 70.0;
        let hue = (i as f64 * 60.0 + (t * 40.0) % 360.0) % 360.0;
        let color = format_str(format_args!("hsl({hue:.0}, 70%, 60%)"));

        ctx.begin_path();
        let _ = ctx.arc(px, py, r, 0.0, std::f64::consts::PI * 2.0);
        set_prop(ctx_js, "strokeStyle", &JsValue::from_str(&color));
        ctx.stroke();
    }
}

// =============================================================================
// Busy-loop: блокирует main thread
// =============================================================================

fn busy_wait_ms(ms: f64) {
    let start = js_sys::Date::now();
    let mut acc = 0.0f64;
    while js_sys::Date::now() - start < ms {
        acc = (acc + 1.0).sqrt();
        if acc > 1.0e6 {
            acc = 0.0;
        }
    }
    // Принудительное чтение, чтобы цикл не выбросили как no-op.
    std::hint::black_box(acc);
}

// =============================================================================
// Главный цикл main thread'а
// =============================================================================

fn start_main_loop(state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    let canvas = document
        .get_element_by_id("oc-main-canvas")
        .ok_or_else(|| JsValue::from_str("no oc-main-canvas"))?
        .dyn_into::<HtmlCanvasElement>()?;

    let ctx = canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("main canvas has no 2d context"))?
        .dyn_into::<CanvasRenderingContext2d>()?;

    let cell: Rc<RefCell<Option<wasm_bindgen::closure::Closure<dyn FnMut(f64)>>>> =
        Rc::new(RefCell::new(None));
    let cell2 = cell.clone();

    *cell.borrow_mut() = Some(wasm_bindgen::closure::Closure::wrap(Box::new(
        move |now: f64| {
            draw_circles(&ctx, CANVAS_W, CANVAS_H, now);

            let need_stats = {
                let mut st = state.borrow_mut();
                st.main_frames += 1;
                if st.main_last_fps_at == 0.0 {
                    st.main_last_fps_at = now;
                    false
                } else if now - st.main_last_fps_at >= 500.0 {
                    st.main_fps =
                        st.main_frames as f64 * 1000.0 / (now - st.main_last_fps_at);
                    st.main_frames = 0;
                    st.main_last_fps_at = now;
                    true
                } else {
                    false
                }
            };

            if need_stats {
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
            }

            // Планируем следующий кадр. cell2 — self-referential Rc,
            // замыкание держит само себя живым.
            if let Some(w) = web_sys::window() {
                if let Some(cb) = cell2.borrow().as_ref() {
                    let _ = w.request_animation_frame(cb.as_ref().unchecked_ref());
                }
            }
        },
    ) as Box<dyn FnMut(f64)>));

    let cb = cell.borrow();
    window.request_animation_frame(cb.as_ref().unwrap().as_ref().unchecked_ref())?;
    // Ссылочный цикл cell → closure → cell2 → cell удерживает объект
    // живым на всё время страницы. Явный forget не нужен.

    Ok(())
}

// =============================================================================
// Обработчики
// =============================================================================

fn wire_worker_messages(
    worker: &Worker,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let closure = wasm_bindgen::closure::Closure::<dyn FnMut(web_sys::MessageEvent)>::new(
        move |ev: web_sys::MessageEvent| {
            let data = ev.data();
            let Ok(kind) = Reflect::get(&data, &JsValue::from_str("type")) else {
                return;
            };
            let Some(kind_str) = kind.as_string() else {
                return;
            };

            if kind_str == "fps" {
                let fps = Reflect::get(&data, &JsValue::from_str("fps"))
                    .ok()
                    .and_then(|v| v.as_f64())
                    .unwrap_or(0.0);
                {
                    let mut st = state.borrow_mut();
                    st.worker_fps = fps;
                }
                if let Some(doc) = web_sys::window().and_then(|w| w.document()) {
                    let st = state.borrow();
                    update_stats(&doc, &st);
                }
            }
        },
    );

    worker.set_onmessage(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    Ok(())
}

fn wire_busy_button(document: &Document) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("oc-busy")
        .ok_or_else(|| JsValue::from_str("no oc-busy"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();

    let closure = wasm_bindgen::closure::Closure::<dyn FnMut()>::new(move || {
        if let Some(s) = doc.get_element_by_id("oc-status") {
            s.set_text_content(Some(
                "Main thread занят: 2 секунды. Левый холст замирает, правый продолжает.",
            ));
        }

        // Даём браузеру отрисовать статус перед блокировкой.
        let doc2 = doc.clone();
        let timeout = gloo_timers::callback::Timeout::new(60, move || {
            busy_wait_ms(BUSY_MS);
            if let Some(s) = doc2.get_element_by_id("oc-status") {
                s.set_text_content(Some("Main thread свободен."));
            }
        });
        timeout.forget();
    });

    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// =============================================================================
// UI
// =============================================================================

fn update_stats(document: &Document, st: &DemoState) {
    let Some(el) = document.get_element_by_id("oc-stats") else {
        return;
    };
    let transfer_str = if st.worker.is_some() {
        "выполнен"
    } else {
        "не выполнен"
    };
    let html = format_str(format_args!(
        r#"<div><span class="label">FPS main thread: </span><span class="value">{mfps:.1}</span></div>
           <div><span class="label">FPS worker:      </span><span class="ok">{wfps:.1}</span></div>
           <div><span class="label">transferControlToOffscreen: </span><span class="value">{ts}</span></div>"#,
        mfps = st.main_fps,
        wfps = st.worker_fps,
        ts = transfer_str,
    ));
    el.set_inner_html(&html);
}

const DESC_LINES: &[&str] = &[
    "Два холста рядом. <strong>Левый</strong> — рисует main thread через \
     <code>requestAnimationFrame</code>. <strong>Правый</strong> — передан \
     Worker'у через <code>transferControlToOffscreen()</code>, рисует Worker \
     в своём event loop.",
    "Нажмите «Занять main thread на 2 секунды»: main thread блокируется, \
     левый холст замирает, правый продолжает анимацию. Это и есть \
     <strong>изоляция</strong> — задача, требующая 60 FPS, вынесена из \
     критического пути UI.",
    "После <code>transferControlToOffscreen()</code> вызов \
     <code>getContext(\"2d\")</code> на правом <code>&lt;canvas&gt;</code> \
     из main thread'а бросает <code>InvalidStateError</code>. Управление \
     буфером передано один раз и навсегда.",
];

fn build_description(document: &Document) -> Result<web_sys::Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("oc-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("oc-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// =============================================================================
// Точка входа
// =============================================================================

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("22.1 — OffscreenCanvas: canvas вне DOM"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    if !is_offscreen_canvas_supported() || !is_transfer_control_supported() {
        let warn = document.create_element("p")?;
        warn.set_class_name("oc-footnote");
        warn.set_text_content(Some(
            "Браузер не поддерживает OffscreenCanvas или \
             transferControlToOffscreen. Демо не может быть запущено.",
        ));
        root.append_child(&warn)?;
        return Ok(());
    }

    // Два холста рядом.
    let row = document.create_element("div")?;
    row.set_class_name("oc-row");
    row.set_inner_html(&format_str(format_args!(
        r#"<div class="oc-col">
               <div class="oc-caption">Main thread · requestAnimationFrame</div>
               <canvas id="oc-main-canvas" class="oc-canvas" width="{w}" height="{h}"></canvas>
           </div>
           <div class="oc-col">
               <div class="oc-caption">Worker · transferControlToOffscreen</div>
               <canvas id="oc-worker-canvas" class="oc-canvas" width="{w}" height="{h}"></canvas>
           </div>"#,
        w = CANVAS_W,
        h = CANVAS_H,
    )));
    root.append_child(&row)?;

    // Кнопка + статус.
    let controls = document.create_element("div")?;
    controls.set_class_name("oc-block");
    controls.set_inner_html(
        r#"<button class="oc-btn oc-btn-warn" id="oc-busy">Занять main thread на 2 секунды</button>
           <span class="oc-label" id="oc-status">Main thread свободен.</span>"#,
    );
    root.append_child(&controls)?;

    // Статистика.
    let stats = document.create_element("div")?;
    stats.set_class_name("oc-stats");
    stats.set_id("oc-stats");
    stats.set_inner_html("…");
    root.append_child(&stats)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("oc-footnote");
    footnote.set_text_content(Some(
        "Одноразовость transferControlToOffscreen: после передачи \
         правого холста Worker'у main thread теряет над ним управление. \
         Это не ошибка, а свойство API: у буфера один владелец. \
         Остановить Worker можно, но вернуть холст main thread'у — нельзя.",
    ));
    root.append_child(&footnote)?;

    // Передача правого canvas'а Worker'у.
    let worker_canvas = document
        .get_element_by_id("oc-worker-canvas")
        .ok_or_else(|| JsValue::from_str("no oc-worker-canvas"))?
        .dyn_into::<HtmlCanvasElement>()?;

    let offscreen = transfer_control(&worker_canvas)?;
    let worker = create_worker(WORKER_URL)?;
    post_init(&worker, &offscreen, CANVAS_W, CANVAS_H)?;

    // Состояние.
    let state = Rc::new(RefCell::new(DemoState::new()));
    state.borrow_mut().worker = Some(worker.clone());

    wire_worker_messages(&worker, state.clone())?;
    start_main_loop(state.clone())?;
    wire_busy_button(&document)?;

    Ok(())
}
```

---

## 22.1.12. Анализ решения

**Какую ценность даёт демо.**

Подглава показывает **одну концепцию на одном экране**: OffscreenCanvas — это canvas вне DOM. Всё остальное (Worker, postMessage, transferable, WASM) сюда не подмешивается.

- Левый холст — main thread. Правый — Worker. **Оба одинаковые визуально**, пока main thread свободен. Это подтверждает: OffscreenCanvas — не «другой canvas», а тот же, только без DOM.
- Кнопка «занять main thread» даёт **прямое** различие. Не «у нас две скорости», а «левый застыл, правый продолжает». Это wow-момент, который не требует объяснений.
- FPS-панель даёт **числа**: 60 → 0 → 60 на main, 60 → 60 → 60 на Worker.

**Скрытие технических сложностей.**

- Пользователь видит два холста и одну кнопку. Внутри — `transferControlToOffscreen`, `Worker`, `postMessage`, `requestAnimationFrame`, но всё это инкапсулировано.
- Статус «Main thread занят / свободен» — предметное сообщение, а не «блокировка event loop на 2 секунды».

**Что осознанно не реализовано.**

- **`new OffscreenCanvas(w, h)` в демо.** Оба холста созданы из DOM-элементов. `new OffscreenCanvas` покрыт тестами и упомянут в тексте, но не показан — он не так нагляден. Сценарий «виртуальный холст для превью без показа» — это тема 21.4 (ImageData) или будущей подглавы про `convertToBlob`/`transferToImageBitmap`.
- **`convertToBlob` / `transferToImageBitmap`.** Покрыты feature detection в модуле и тестами.
- **Отправка данных в Worker.** Не тема 22.1.
- **WASM в Worker.** Не тема 22.1.

**Правила проекта, применённые в этой подглаве.**

1. **Autonomous app.rs.** Всё inline: `format_str`, `bool_of`, feature detection, transfer_control, worker, drawing.
2. **`format_str(format_args!(...))` в tail-позициях.** В `draw_circles`, `update_stats`, `run`.
3. **`bool_of(matches!(...))` в tail-позициях.** В `is_offscreen_canvas_supported`, `is_transfer_control_supported` — и в модуле, и в `app.rs`. Снимает E0308 rust-analyzer'а.
4. **RefMut-дисциплина.** Каждый `borrow_mut` в узком scope, `borrow` — после drop.
5. **Feature detection перед использованием API.**
6. **`is_function()` перед вызовом через Reflect.**
7. **Стили инжектируются в документ.** Правок в `main.css` не требуется.
8. **Уникальный префикс `oc-`.** Не пересекается с `ow-` (глава 22), `canvas-` (глава 21).
9. **Строки — через одну функцию `format_str`.** Никаких `format!` в tail-позиции.
10. **Никаких неиспользуемых импортов.** `Event` удалён из `web_sys` — не использовался.

**Проверки корректности.**

- `Closure` для busy-wait переносится в `gloo_timers::callback::Timeout::forget()` — не утечка, а сознательное удержание до вызова.
- Rc-цикл в `start_main_loop` (cell → closure → cell2 → cell) удерживает анимацию живой всё время страницы. Явный `mem::forget` не нужен — Rc сам себе держит ссылку.
- `busy_wait_ms` использует `std::hint::black_box` — компилятор не выбросит цикл.
- `setTimeout(60 ms)` перед блокировкой даёт браузеру отрисовать статус «Main thread занят…» до того, как поток встанет.
- Обработчик `wire_busy_button` не берёт `state` — состояние тут не нужно.

---

## 22.1.13. Список доработок после первой итерации

1. **`bool_of` в feature detection.** `matches!(…)` в tail-позиции даёт E0308 от rust-analyzer. Утилита `bool_of` (identity-функция с `#[inline]`) применяется как в `offscreen_canvas.rs`, так и в `app.rs`. Это уже принятый в проекте приём (см. главу 22).
2. **Убран `Event` из импортов `app.rs`.** В демо нет обработчиков, принимающих `Event` — клики обрабатываются через `Closure<dyn FnMut()>` без аргумента.
3. **Rc-цикл в `start_main_loop`.** `std::mem::forget(cell)` убран: `closure` держит `cell2` (клон того же Rc), а `cell` держит `closure`. Цикл сам себя поддерживает.
4. **`busy_wait_ms` с `black_box`.** Первая версия использовала наивный цикл, который rustc мог соптимизировать. Добавлен `std::hint::black_box(acc)` в конце.
5. **Задержка перед блокировкой.** Первая версия вызывала `busy_wait_ms` прямо в обработчике клика — статус «Main thread занят» не успевал отрисоваться. Разделено через `Timeout::new(60, ...)`.
6. **Reflect-сеттеры для canvas-свойств.** `ctx.set_fill_style_str(...)` в некоторых версиях web-sys возвращает `Result`. Через `Reflect::set` — всегда работает, без зависимости от версии.
7. **Feature detection до создания Worker'а.** Если браузер не поддерживает — теряется `<canvas>`. Теперь feature detection **до** создания row с холстами.

---

## 22.1.14. Итог

`OffscreenCanvas` — это canvas, у которого нет DOM. Его можно создать двумя способами: `new OffscreenCanvas(w, h)` — «виртуальный» холст в памяти; `canvas.transferControlToOffscreen()` — отобрать управление у DOM-элемента.

`transferControlToOffscreen` — одноразовая операция. После неё main thread теряет доступ к буферу: `getContext("2d")` на оригинальном элементе бросает `InvalidStateError`. Это не баг, а свойство модели владения: у буфера один хозяин.

`width`/`height` у `OffscreenCanvas` — размеры **буфера**, а не CSS. Их путают, и это источник размытых картинок на retina-экранах.

Демо показывает разницу «main thread рисует» vs «Worker рисует»: два холста рядом, одинаковые визуально, пока main thread свободен. Кнопка блокировки main thread'а заставляет левый холст замереть, а правый — продолжить. Это и есть цель подглавы: показать, что OffscreenCanvas — не «ещё один canvas», а **полноценный объект рисования, не привязанный к DOM**.

Подглава закрывает тему `OffscreenCanvas` **как объекта**. Обмен данными с Worker'ом, transferable-объекты, WASM в Worker'е, пулы и SharedArrayBuffer — следующие подглавы (22.2–22.6).