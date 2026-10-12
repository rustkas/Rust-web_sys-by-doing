# Глава 21.2. Drawing API: фигуры, пути, текст

## 21.2.1. О чём эта подглава

Предыдущая подглава разобралась с размерами: сколько пикселей в буфере, как сделать холст резким на Retina, почему `style.width` и `canvas.width` — разные вещи. Холст правильно настроен. Но он **пустой**. Мы умеем его создавать, не умеем на нём рисовать.

Эта подглава — про то, **что вообще можно изобразить**. Прямоугольники, линии, круги, ломаные, кривые Безье, текст, градиенты. Всё это — Drawing API, набор методов `CanvasRenderingContext2d`, с которыми мы уже встречались в 21.1 (`fillRect` для фона), но не разбирали систематически.

Ключевая идея главы: **всё, кроме прямоугольников, рисуется через пути**. Прямоугольники — четыре метода (`fillRect`, `strokeRect`, `clearRect`, `rect`). Всё остальное: начни путь, добавь сегменты, закончи путь, залей или обведи. Дуга, круг, кривая Безье — все они сводятся к одному механизму.

Это то, что отличает canvas от SVG. В SVG каждая фигура — отдельный элемент DOM с атрибутами. В canvas фигура — след, оставленный кистью. Никакой структуры, никакой объектной модели, только пиксели.

## 21.2.2. Что уже есть и что появится

К концу 21.1 мы умеем:
- создавать `<canvas>` программно и в разметке;
- настраивать буфер с учётом DPR;
- получать `CanvasRenderingContext2d`;
- очищать холст цветом через `fillRect`.

Этого достаточно, чтобы показать фон. Всё остальное — новое.

К концу 21.2 будем уметь:
- рисовать **прямоугольники** во всех вариантах;
- строить **произвольные пути** из прямых и кривых сегментов;
- рисовать **дуги и окружности**;
- **кривые Безье** — кубические и квадратичные;
- выводить **текст** и замерять его размеры;
- заливать фигуры **градиентами**;
- управлять **стилем кисти** — толщина, цвет, закругление.

## 21.2.3. Прямоугольники

Простейший случай — четыре метода:

- `fillRect(x, y, w, h)` — залить прямоугольник текущим `fillStyle`;
- `strokeRect(x, y, w, h)` — обвести текущим `strokeStyle` и `lineWidth`;
- `clearRect(x, y, w, h)` — стереть область (сделать прозрачной);
- `rect(x, y, w, h)` — добавить прямоугольник **в текущий путь**.

Первые три рисуют немедленно. Четвёртый — откладывает рисование до `fill()` или `stroke()`, чтобы можно было комбинировать с другими сегментами. Это тонкое, но важное различие: `fillRect` — «кисть коснулась холста», `rect` — «положил в очередь».

В демо прямоугольник рисуется двумя вызовами: `fillRect` для заливки, `strokeRect` для обводки. Через путь было бы длиннее, результат тот же.

## 21.2.4. Пути — основной механизм

Путь — последовательность сегментов, которую можно залить или обвести один раз. Строится так:

```js
ctx.beginPath();          // начать новый путь, сбросить предыдущий
ctx.moveTo(x0, y0);       // поднять кисть и переставить в точку
ctx.lineTo(x1, y1);       // провести линию от текущей точки
ctx.lineTo(x2, y2);       // ещё одну
ctx.closePath();          // замкнуть контур
ctx.fill();               // залить область внутри пути
ctx.stroke();             // обвести контур пути
```

**`beginPath()` обязателен.** Без него новые сегменты добавятся к предыдущему пути — получится путаница из старых и новых фигур. Это ловушка №1 для начинающих: нарисовали треугольник, потом круг, а `stroke()` обводит и то, и другое.

**`moveTo` разрывает путь.** После `moveTo` кисть ставится в новую точку, но линия не проводится. Это позволяет в одном пути иметь несколько несвязанных ломаных — все они будут залиты или обведены как один контур.

**`closePath` не то же самое, что `lineTo` в начальную точку.** `closePath` создаёт замкнутый контур: углы между последним сегментом и первым сглаживаются через `lineJoin`. `lineTo` такой связи не делает. Разница видна на больших толщинах — с `closePath` стык аккуратный.

## 21.2.5. Дуги и окружности

`ctx.arc(cx, cy, radius, startAngle, endAngle)` — дуга. Углы в радианах, отсчёт от положительной оси X по часовой стрелке. Полная окружность — от `0` до `2π`.

```js
ctx.beginPath();
ctx.arc(200, 55, 35, 0, Math.PI * 2);
ctx.fill();
ctx.stroke();
```

Пятый аргумент `counterclockwise` (по умолчанию `false`) переключает направление.

**`arc` автоматически проводит линию** от текущей точки пути к началу дуги. Если это не нужно — вызывайте `moveTo` перед `arc`. В нашем демо окружность — отдельный путь, `beginPath` сбрасывает состояние, `arc` начинает фигуру с нуля.

Есть ещё `arcTo(x1, y1, x2, y2, radius)` — дуга между двумя лучами, и `ellipse(cx, cy, rx, ry, rotation, startAngle, endAngle)` — эллипс. В демо их не показываем: `arc` покрывает 95 % случаев.

## 21.2.6. Кривые Безье

Два метода, оба через контрольные точки:

- **Квадратичная** `quadraticCurveTo(cpx, cpy, x, y)` — одна контрольная точка, конечная точка.
- **Кубическая** `bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x, y)` — две контрольные точки, конечная точка.

Как это работает: кривая **не проходит** через контрольные точки. Она к ним стремится, но обходит по дуге. Чем дальше контрольная точка от прямой между началом и концом, тем сильнее изгиб.

В демо две кривые:
- **Волна** через кубическую — плавный S-образный изгиб;
- **Горка** через квадратичную — симметричная дуга.

Обе без `closePath`, только `stroke()`. Показывают разницу между двумя методами на одном экране.

## 21.2.7. Текст

Три метода:

- `fillText(text, x, y)` — залить текст;
- `strokeText(text, x, y)` — обвести контур букв;
- `measureText(text)` — вернуть метрики строки.

Стиль текста задаётся через `font` — CSS-подобная строка: `"16px monospace"`, `"bold 24px sans-serif"`, `"italic 12px Georgia"`. Синтаксис тот же, что в CSS, потому что canvas использует тот же парсер.

`measureText` возвращает объект с метриками: `width`, `actualBoundingBoxAscent`, `actualBoundingBoxDescent`. `width` — ширина строки в CSS-пикселях. Нужна, чтобы центрировать текст, автоподобрать размер шрифта, проверить, поместится ли длинная строка в область.

В демо три размера текста: 16px, 12px, 9px. В статус-строке выводится `measureText("Large 16px")` — конкретное число, зависящее от текущего шрифта. Это маленький, но важный штрих: показать, что метрики не магические, их можно посчитать и использовать.

## 21.2.8. Стиль кисти

Ключевое свойство контекста:

- `fillStyle` — цвет или градиент для заливки;
- `strokeStyle` — цвет или градиент для обводки;
- `lineWidth` — толщина линии в CSS-пикселях;
- `lineCap` — как обрабатываются концы линий: `butt`, `round`, `square`;
- `lineJoin` — как обрабатываются стыки между сегментами: `miter`, `round`, `bevel`.

**Контекст — не «параметры вызова», а состояние.** Установили `lineWidth = 2` — все последующие линии рисуются толщиной 2, пока не измените. Это относится к любому стилю. Отсюда правило: если рисунок сложный, ставьте стили перед каждой группой фигур.

В демо четыре слайдера:
- **Толщина линии** — `lineWidth` от 0.5 до 10;
- **Цвет обводки** — 5 вариантов `strokeStyle`;
- **Цвет заливки** — 5 вариантов `fillStyle`;
- **Соединение** — `lineJoin` из трёх вариантов: miter, round, bevel.

Крутите слайдеры — сцена перерисовывается мгновенно. Это и есть смысл: все фигуры рисуются одной функцией `draw_scene`, состояние контекста настраивается перед ней, вся сцена обновляется как единое целое.

## 21.2.9. Градиенты

`fillStyle` и `strokeStyle` принимают не только цвет, но и объект градиента. Два вида:

- **Линейный** `createLinearGradient(x0, y0, x1, y1)` — цвет плавно переходит от точки к точке;
- **Радиальный** `createRadialGradient(x0, y0, r0, x1, y1, r1)` — между двумя окружностями.

У градиента добавляются **цветовые стопы** через `addColorStop(offset, color)`, где `offset` от 0 до 1.

```js
const grad = ctx.createLinearGradient(220, 160, 380, 210);
grad.addColorStop(0, "#58a6ff");
grad.addColorStop(1, "#3fb950");
ctx.fillStyle = grad;
ctx.fillRect(220, 160, 160, 50);
```

В демо — линейный градиент, который меняет цвета вместе со слайдером. Крутите «цвет обводки» — градиент тоже меняется, потому что первый стоп берёт текущий `strokeStyle`. Это тонкий момент: градиент **создаётся заново** при каждой перерисовке с текущими параметрами кисти.

## 21.2.10. Что показывает демо

Демо — **инспектор кисти**. Слева-сверху панель с четырьмя слайдерами. Под ней холст 420×240 с фиксированной сценой. Внизу — статус-строка.

Сцена:

1. **Прямоугольник** — заливка + обводка.
2. **Окружность** — `arc` + `fill` + `stroke`.
3. **Ломаная** — `moveTo` + четыре `lineTo`, только `stroke`.
4. **Кубическая кривая Безье** — волна.
5. **Квадратичная кривая** — горка.
6. **Три размера текста** — 16px, 12px, 9px.
7. **Линейный градиент** — плавный переход между двумя цветами.

Слайдеры управляют `lineWidth`, `strokeStyle`, `fillStyle`, `lineJoin`. При изменении любого — вся сцена перерисовывается целиком. Никаких инкрементальных обновлений, никакого кэша.

В статус-строке — три числа: `lineWidth = 2.0 · lineJoin = round · measureText("Large 16px") = 112.3px`. Показывает, как `measureText` возвращает метрики, зависящие от текущего шрифта.

## 21.2.11. Wow-момент

Откройте демо. Сцена нарисована целиком, не постепенно. Крутите слайдер толщины от 0.5 до 10 — линии становятся толще, стыки ломаной сглаживаются, окружность «наливается» краской. Крутите цвет обводки — меняются линии и градиент. Крутите цвет заливки — только прямоугольник и круг.

Три секунды слайдеров — и понятно, что `lineWidth`, `strokeStyle`, `fillStyle` — это **состояние**, а не аргументы. Всё, что после установки, использует их. Три секунды — закрепили то, что обычно требует чтения документации.

Мелкий текст при толщине 0.5 становится полупрозрачным (субпиксельные линии); при толщине 10 — жирным. Кривые Безье при толщине 2 ещё «стройные», при толщине 8 — «плюшевые». Всё это наблюдения одного слайдера.

Переключите `lineJoin` с `round` на `miter` и обратно — увидите, как меняются стыки ломаной. `miter` даёт острые углы, `round` — сглаженные, `bevel` — «срезанные». Разница заметна только на больших толщинах — но именно там она и важна.

## 21.2.12. Чему это учит

**Canvas императивен.** Нет объектов, есть пиксели. Нарисовали — они там, забыли. Никакого `getRectById`, никакого `removeShape`. Чтобы «убрать» фигуру — нарисуйте поверх или очистите холст и нарисуйте заново. Демо перерисовывает **всю сцену** при каждом изменении: другого способа нет.

**Пути — универсальный механизм.** `fillRect` — исключение, «простейший случай». Всё остальное — построение пути и его заливка или обводка. `arc`, `bezierCurveTo`, `lineTo`, `rect` — все они добавляют сегменты к одному пути.

**`beginPath` нельзя забывать.** Без него новый сегмент добавляется к старому пути — и `fill()` заливает **всю историю** рисования. Это самая частая ошибка после «why is my canvas blurry». В тестах есть отдельная проверка `begin_path_resets_previous_segments` — она ловит именно эту ошибку.

**Стиль — состояние, а не параметр.** `fillStyle` не передаётся в `fill()`, он берётся из контекста. Это отличается от SVG, где `<rect fill="#fff"/>` содержит всё нужное. В canvas: «поставь стиль — нарисуй — поставь другой стиль — нарисуй другое».

**Метрики — не побочный эффект.** `measureText` возвращает объект, из которого можно получить ширину, высоту подъёма и спуска. Без этого невозможно центрировать текст по ширине: `fillText(text, x, y)` рисует **от левого края**, а не от центра. Хотите центрировать — `x = cx - measureText(text).width / 2`.

**Градиент создаётся для каждой перерисовки.** Градиент — это функция параметра: между какими точками и с какими стопами. Он не «сохраняется» в контексте — сохраняется только текущая ссылка на объект градиента. Перерисовали сцену с новым цветом — создали новый градиент.

## 21.2.13. Как устроен код

Модуль `canvas_drawing.rs` — тонкая обёртка над `CanvasRenderingContext2d`:

- `Brush` — структура с полями `stroke`, `fill`, `line_width`, `line_cap`, `line_join`. Компактное представление состояния;
- `draw_scene(ctx, w, h, brush)` — рисует весь набор фигур;
- `measure(ctx, text)` — обёртка над `measureText`, возвращает `(width, ascent, descent)`;
- `clear(ctx, w, h, bg)` — очистка.

Демо `app.rs` **автономен**: содержит собственные копии всех нужных функций. Ничего не импортирует из других модулей. Стили `.dr-*` инжектируются в документ через функцию `inject_styles` — правок в `main.css` не требуется. Это правило проекта: демо-пример для каждой подглавы — самостоятельная единица, читатель открывает файл и видит ровно то, что относится к теме.

Тесты — wasm-only, потому что Drawing API целиком в браузере. Пять тестов проверяют **выход** (пиксели), а не вызовы: цвет заливки, толщину обводки, метрики текста, градиент, изоляцию путей. Единственный способ проверить императивный API — посмотреть, что оказалось в буфере.

Отдельная деталь: в тестах хелпер `read_pixel` скрывает разницу между двумя сигнатурами `getImageData` (`f64` vs `i32`), которая зависит от флага `web_sys_unstable_apis`. Один cfg-гейт — тесты собираются и с флагом, и без.

## 21.2.14. Ссылки

- MDN: [`CanvasRenderingContext2D`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D)
- MDN: [Drawing shapes with canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_shapes)
- MDN: [Drawing text](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Drawing_text)
- MDN: [`measureText`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/measureText)
- MDN: [`createLinearGradient`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/createLinearGradient)
- web.dev: [Learn HTML canvas](https://web.dev/learn/canvas)
- WHATWG Spec: [Canvas drawing model](https://html.spec.whatwg.org/multipage/canvas.html#drawing-model)

## 21.2.15. Итог

Drawing API — механизм превращения абстрактных координат в пиксели. Прямоугольники — исключение, всё остальное через пути. Путь строится от `beginPath` до `fill` или `stroke`. Стиль — состояние контекста, а не аргумент вызова. Кривые Безье дают плавные изгибы, дуги — округлые формы, текст — строки, метрики — ширины.

Демо-инспектор показывает это на четырёх слайдерах и одной сцене. Сдвинул ползунок — вся сцена перерисовалась. За три секунды студент понимает, что стиль — не «параметр вызова», а «настройка кисти». Это то, что обычно требует чтения документации; здесь — на живом примере.

Следующая подглава **21.3** — про состояние, трансформации и композицию: как слои накладываются друг на друга, зачем `save/restore`, как работают `translate` и `globalCompositeOperation`. Тема тесно связана с этой: рисовать мы умеем, теперь учимся управлять **порядком** и **взаимодействием** фигур.

**21.4** откроет пиксельный буфер — что происходит под всеми этими фигурами. **21.5** покажет загрузку изображений. **21.6** — фильтры на Rust. **21.7** — практический проект.

Но первым шагом нужно понять, что такое «путь» и «стиль». Без этого всё остальное — синтаксис без смысла.

# Глава 21.2 — Исходные коды доработанных файлов

## Структура

```
src/
  canvas_drawing.rs              ← модуль главы 21.2
  app.rs                         ← демо, автономный, стили инжектируются
tests/
  wasm_canvas_drawing.rs         ← wasm-тесты с cfg-гейтом для getImageData
src/lib.rs                       ← +1 строка
Cargo.toml                       ← +2 фичи (CanvasGradient, TextMetrics)
```

Правок в `web/styles/main.css` не требуется — все стили `.dr-*` живут внутри `app.rs`.

---

## 1. `src/canvas_drawing.rs`

```rust
//! 21.2. Drawing API: фигуры, пути, текст.
//!
//! Всё рисование идёт через `CanvasRenderingContext2d`. Прямоугольники —
//! четыре простых метода (`fillRect`, `strokeRect`, `clearRect`, `rect`).
//! Всё остальное — **пути**: `beginPath` → набор сегментов (`lineTo`,
//! `arc`, `bezierCurveTo`, `quadraticCurveTo`) → `fill()` или `stroke()`.
//!
//! Стиль — **состояние контекста**: `fillStyle`, `strokeStyle`, `lineWidth`,
//! `lineCap`, `lineJoin`, `font`. Устанавливается один раз, действует на
//! все последующие операции, пока не изменится.
//!
//! Модуль держит `Brush` — компактное представление этого состояния —
//! и одну функцию `draw_scene`, которая рисует демонстрационный набор фигур.

use wasm_bindgen::prelude::*;
use web_sys::{CanvasGradient, CanvasRenderingContext2d, TextMetrics};

// =============================================================================
// Кисть — состояние контекста
// =============================================================================

/// Параметры кисти. Все поля — `&'static str`, потому что в демо это
/// фиксированные палитры из константных массивов.
#[derive(Clone, Copy, Debug)]
pub struct Brush {
    pub stroke: &'static str,
    pub fill: &'static str,
    pub line_width: f64,
    pub line_cap: &'static str,
    pub line_join: &'static str,
}

impl Default for Brush {
    fn default() -> Self {
        Self {
            stroke: "#58a6ff",
            fill: "rgba(88, 166, 255, 0.25)",
            line_width: 2.0,
            line_cap: "round",
            line_join: "round",
        }
    }
}

// =============================================================================
// Рисование
// =============================================================================

/// Очищает холст фоновым цветом.
pub fn clear(ctx: &CanvasRenderingContext2d, w: f64, h: f64, bg: &str) {
    ctx.set_fill_style_str(bg);
    ctx.fill_rect(0.0, 0.0, w, h);
}

/// Устанавливает стиль кисти на контекст.
fn apply_brush(ctx: &CanvasRenderingContext2d, brush: Brush) {
    ctx.set_line_width(brush.line_width);
    ctx.set_line_cap(brush.line_cap);
    ctx.set_line_join(brush.line_join);
    ctx.set_stroke_style_str(brush.stroke);
    ctx.set_fill_style_str(brush.fill);
}

/// Создаёт линейный градиент между двумя точками с двумя стопами.
/// Возвращает `None`, если браузер не дал объект или стоп не принялся.
fn make_gradient(
    ctx: &CanvasRenderingContext2d,
    x0: f64,
    y0: f64,
    x1: f64,
    y1: f64,
    c0: &str,
    c1: &str,
) -> Option<CanvasGradient> {
    let g = ctx.create_linear_gradient(x0, y0, x1, y1);
    g.add_color_stop(0.0, c0).ok()?;
    g.add_color_stop(1.0, c1).ok()?;
    Some(g)
}

/// Рисует демонстрационный набор фигур в **CSS-координатах** `(0..w, 0..h)`.
///
/// Состав сцены:
/// 1. прямоугольник — заливка + обводка (`fillRect` + `strokeRect`);
/// 2. окружность — путь `arc` + `fill` + `stroke`;
/// 3. ломаная — путь `moveTo` + четыре `lineTo`, только `stroke`;
/// 4. кубическая кривая Безье — `bezierCurveTo`, только `stroke`;
/// 5. квадратичная кривая — `quadraticCurveTo`, только `stroke`;
/// 6. три размера текста — 16px, 12px, 9px (`fillText` + `font`);
/// 7. прямоугольник с линейным градиентом (`createLinearGradient`).
pub fn draw_scene(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
    brush: Brush,
) -> Result<(), JsValue> {
    clear(ctx, w, h, "#0d1117");
    apply_brush(ctx, brush);

    // 1. Прямоугольник — заливка и обводка.
    ctx.fill_rect(20.0, 20.0, 100.0, 70.0);
    ctx.stroke_rect(20.0, 20.0, 100.0, 70.0);

    // 2. Окружность — путь + fill + stroke.
    ctx.begin_path();
    ctx.arc(200.0, 55.0, 35.0, 0.0, std::f64::consts::TAU)?;
    ctx.fill();
    ctx.stroke();

    // 3. Ломаная — пять точек, только обводка.
    ctx.begin_path();
    ctx.move_to(260.0, 90.0);
    ctx.line_to(290.0, 30.0);
    ctx.line_to(320.0, 80.0);
    ctx.line_to(350.0, 25.0);
    ctx.line_to(380.0, 70.0);
    ctx.stroke();

    // 4. Кубическая кривая Безье — «волна».
    ctx.begin_path();
    ctx.move_to(20.0, 140.0);
    ctx.bezier_curve_to(90.0, 100.0, 170.0, 180.0, 240.0, 140.0);
    ctx.stroke();

    // 5. Квадратичная кривая — «горка».
    ctx.begin_path();
    ctx.move_to(260.0, 140.0);
    ctx.quadratic_curve_to(320.0, 90.0, 380.0, 140.0);
    ctx.stroke();

    // 6. Текст — три размера, для наглядности разницы.
    ctx.set_fill_style_str("#c9d1d9");
    ctx.set_font("16px monospace");
    ctx.fill_text("Large 16px", 20.0, 180.0)?;
    ctx.set_font("12px monospace");
    ctx.fill_text("Medium 12px", 20.0, 200.0)?;
    ctx.set_font("9px monospace");
    ctx.fill_text("Small 9px", 20.0, 216.0)?;

    // 7. Градиент — линейный переход между цветом кисти и зелёным.
    if let Some(grad) = make_gradient(
        ctx, 220.0, 160.0, 380.0, 210.0, brush.stroke, "#3fb950",
    ) {
        ctx.set_fill_style_canvas_gradient(&grad);
        ctx.fill_rect(220.0, 160.0, 160.0, 50.0);
    }

    Ok(())
}

/// Возвращает `(width, ascent, descent)` для строки при текущем `font`.
/// Использует `measureText`, нужен для центрирования и подбора размера.
pub fn measure(
    ctx: &CanvasRenderingContext2d,
    text: &str,
) -> Result<(f64, f64, f64), JsValue> {
    let m: TextMetrics = ctx.measure_text(text)?;
    Ok((
        m.width(),
        m.actual_bounding_box_ascent(),
        m.actual_bounding_box_descent(),
    ))
}
```

---

## 2. `tests/wasm_canvas_drawing.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::{CanvasRenderingContext2d, Document, HtmlCanvasElement};

use hello_wasm02::canvas_drawing::{draw_scene, measure, Brush};

wasm_bindgen_test_configure!(run_in_browser);

fn make_canvas(w: u32, h: u32) -> (HtmlCanvasElement, CanvasRenderingContext2d) {
    let doc: Document = web_sys::window().unwrap().document().unwrap();
    let c: HtmlCanvasElement = doc.create_element("canvas").unwrap().dyn_into().unwrap();
    c.set_width(w);
    c.set_height(h);
    let ctx: CanvasRenderingContext2d = c
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into()
        .unwrap();
    (c, ctx)
}

/// Читает RGBA в точке `(x, y)`.
///
/// `getImageData` в web-sys 0.3.105 имеет две сигнатуры, переключаемые
/// cfg-флагом `web_sys_unstable_apis`:
/// - stable: `f64, f64, f64, f64`;
/// - unstable: `i32, i32, i32, i32`.
///
/// Хелпер скрывает это различие. Внешний интерфейс — `f64`-координаты.
/// Остальные тесты не думают про cfg.
fn read_pixel(ctx: &CanvasRenderingContext2d, x: f64, y: f64) -> Vec<u8> {
    #[cfg(web_sys_unstable_apis)]
    let data = ctx.get_image_data(x as i32, y as i32, 1, 1).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    let data = ctx.get_image_data(x, y, 1.0, 1.0).unwrap();

    // `ImageData::data()` в 0.3.105 возвращает `Clamped<Vec<u8>>` — берём `.0`.
    data.data().0
}

#[wasm_bindgen_test]
fn draw_scene_completes_without_panic() {
    let (_c, ctx) = make_canvas(400, 240);
    draw_scene(&ctx, 400.0, 240.0, Brush::default()).expect("draw");
}

#[wasm_bindgen_test]
fn filled_rect_has_expected_color() {
    let (_c, ctx) = make_canvas(50, 50);
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(10.0, 10.0, 30.0, 30.0);

    let px = read_pixel(&ctx, 20.0, 20.0);
    assert_eq!(px[0], 255, "red");
    assert_eq!(px[1], 0, "green");
    assert_eq!(px[2], 0, "blue");
}

#[wasm_bindgen_test]
fn stroke_thickness_affects_pixels() {
    // Тонкая линия vs толстая — толстая должна закрасить больше.
    let (_c1, ctx1) = make_canvas(100, 100);
    ctx1.set_stroke_style_str("#ffffff");
    ctx1.set_line_width(1.0);
    ctx1.begin_path();
    ctx1.move_to(10.0, 50.0);
    ctx1.line_to(90.0, 50.0);
    ctx1.stroke();

    let (_c2, ctx2) = make_canvas(100, 100);
    ctx2.set_stroke_style_str("#ffffff");
    ctx2.set_line_width(10.0);
    ctx2.begin_path();
    ctx2.move_to(10.0, 50.0);
    ctx2.line_to(90.0, 50.0);
    ctx2.stroke();

    // Читаем пиксель на расстоянии 4 px от центра линии.
    // Тонкая линия его не задела, толстая — задела.
    let thin = read_pixel(&ctx1, 50.0, 54.0);
    let thick = read_pixel(&ctx2, 50.0, 54.0);
    assert!(thin[0] < 40, "thin stroke should not reach far");
    assert!(thick[0] > 200, "thick stroke should cover y+4");
}

#[wasm_bindgen_test]
fn measure_text_grows_with_font_size() {
    let (_c, ctx) = make_canvas(200, 50);
    ctx.set_font("10px monospace");
    let w_small = measure(&ctx, "Sample").unwrap().0;
    ctx.set_font("30px monospace");
    let w_large = measure(&ctx, "Sample").unwrap().0;
    assert!(w_large > w_small, "large {w_large} should beat small {w_small}");
    assert!(w_small > 0.0, "width should be positive");
}

#[wasm_bindgen_test]
fn gradient_has_distinct_colors_at_ends() {
    let (_c, ctx) = make_canvas(100, 50);
    let grad = ctx.create_linear_gradient(0.0, 0.0, 100.0, 0.0);
    grad.add_color_stop(0.0, "#000000").unwrap();
    grad.add_color_stop(1.0, "#ffffff").unwrap();
    ctx.set_fill_style_canvas_gradient(&grad);
    ctx.fill_rect(0.0, 0.0, 100.0, 50.0);

    let left = read_pixel(&ctx, 2.0, 25.0);
    let right = read_pixel(&ctx, 97.0, 25.0);
    assert!(left[0] < 40, "left should be dark, got {}", left[0]);
    assert!(right[0] > 215, "right should be bright, got {}", right[0]);
}

#[wasm_bindgen_test]
fn begin_path_resets_previous_segments() {
    // Ключевая проверка: без beginPath второй stroke() обвёл бы
    // и первую линию тоже. С beginPath — только вторую.
    let (_c, ctx) = make_canvas(100, 100);

    // Первая линия — по диагонали (20,20)→(80,80).
    ctx.set_stroke_style_str("#ff0000");
    ctx.set_line_width(2.0);
    ctx.begin_path();
    ctx.move_to(20.0, 20.0);
    ctx.line_to(80.0, 80.0);
    ctx.stroke();

    // Вторая — горизонтальная на y=10. Начинаем с beginPath.
    ctx.set_stroke_style_str("#00ff00");
    ctx.begin_path();
    ctx.move_to(20.0, 10.0);
    ctx.line_to(80.0, 10.0);
    ctx.stroke();

    // Пиксель в центре диагонали должен остаться красным —
    // вторая линия прошла по y=10 и не пересеклась с ним.
    let diag = read_pixel(&ctx, 50.0, 50.0);
    assert!(diag[0] > 200, "diagonal should still be red");
    assert!(diag[1] < 40, "diagonal should not be green");
}
```

---

## 3. `src/app.rs`

```rust
//! Глава 21.2 — Drawing API: фигуры, пути, текст.
//!
//! «Инспектор кисти»: сцена из фигур обновляется при движении
//! слайдеров. Слайдеры управляют состоянием контекста (lineWidth,
//! strokeStyle, fillStyle, lineJoin) — и вся сцена перерисовывается
//! целиком, потому что canvas императивен: чтобы «убрать» фигуру,
//! её надо перерисовать заново.
//!
//! **Файл автономен.** Ничего не импортируется из других модулей проекта.
//! Все стили `.dr-*` инжектируются в документ через `inject_styles()`
//! — правок в `main.css` не требуется.
//!
//! Wow: сдвиньте «Толщина линии» до 10 — увидите, как стыки ломаной
//! сглаживаются, окружность «наливается» краской, тонкий текст
//! превращается в жирный. Три секунды слайдеров показывают, что
//! стиль — это состояние контекста, а не аргумент вызова.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    CanvasGradient, CanvasRenderingContext2d, Document, Event, HtmlCanvasElement,
    HtmlElement, HtmlInputElement, Node,
};

// =============================================================================
// Утилита обхода бага rust-analyzer: `format!` в tail-позиции теряет тип
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Стили, инжектируемые в документ
// =============================================================================

/// Инжектирует `<style>` с правилами `.dr-*` в `<head>`. Идемпотентно:
/// если стиль уже есть — удаляется и добавляется заново, чтобы правки
/// в этой функции всегда применялись без перезагрузки.
fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "dr-injected-style";
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

/// Все CSS-правила демо. Один блок — проще проверять и править.
const STYLE_CSS: &str = r#"
.dr-desc {
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
.dr-desc-line { margin: 6px 0; }

.dr-controls {
    max-width: 720px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: flex;
    flex-direction: column;
    gap: 10px;
}
.dr-control-row {
    display: flex;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
}
.dr-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    min-width: 100px;
}
.dr-slider {
    flex: 1;
    min-width: 180px;
    max-width: 320px;
    accent-color: #58a6ff;
}
.dr-value {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #79c0ff;
    min-width: 120px;
}

.dr-canvas {
    display: block;
    width: 420px;
    height: 240px;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 6px;
    margin-bottom: 16px;
}

.dr-status {
    max-width: 720px;
    padding: 10px 14px;
    background: #0d1117;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    line-height: 1.5;
    margin-bottom: 20px;
}

.dr-footnote {
    max-width: 720px;
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
// Кисть и рисование
// =============================================================================

#[derive(Clone, Copy, Debug)]
struct Brush {
    stroke: &'static str,
    fill: &'static str,
    line_width: f64,
    line_join: &'static str,
}

impl Brush {
    fn from_state(state: &DemoState) -> Self {
        Self {
            stroke: STROKES[state.stroke_idx],
            fill: FILLS[state.fill_idx],
            line_width: state.line_width,
            line_join: LINE_JOINS[state.line_join_idx],
        }
    }
}

const STROKES: [&str; 5] = ["#58a6ff", "#3fb950", "#f0883e", "#d2a8ff", "#f85149"];
const FILLS: [&str; 5] = [
    "rgba(88, 166, 255, 0.25)",
    "rgba(63, 185, 80, 0.25)",
    "rgba(240, 136, 62, 0.25)",
    "rgba(210, 168, 255, 0.25)",
    "rgba(248, 81, 73, 0.25)",
];
/// lineJoin — три варианта. Slug и label совпадают, поэтому один массив.
const LINE_JOINS: [&str; 3] = ["miter", "round", "bevel"];

/// Достаёт 2d-контекст как `CanvasRenderingContext2d`.
/// `dyn_into` возвращает `Result<_, Object>` — нужен `.map_err`.
fn get_2d_context(
    canvas: &HtmlCanvasElement,
) -> Result<CanvasRenderingContext2d, JsValue> {
    canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?
        .dyn_into::<CanvasRenderingContext2d>()
        .map_err(|_| JsValue::from_str("context is not CanvasRenderingContext2d"))
}

/// Очищает холст фоновым цветом.
fn clear(ctx: &CanvasRenderingContext2d, w: f64, h: f64) {
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);
}

/// Создаёт линейный градиент между двумя точками с двумя стопами.
fn make_gradient(
    ctx: &CanvasRenderingContext2d,
    x0: f64,
    y0: f64,
    x1: f64,
    y1: f64,
    c0: &str,
    c1: &str,
) -> Option<CanvasGradient> {
    let g = ctx.create_linear_gradient(x0, y0, x1, y1);
    g.add_color_stop(0.0, c0).ok()?;
    g.add_color_stop(1.0, c1).ok()?;
    Some(g)
}

/// Рисует демонстрационный набор фигур в **CSS-координатах** `(0..w, 0..h)`.
///
/// Состав сцены:
/// 1. прямоугольник — заливка + обводка (`fillRect` + `strokeRect`);
/// 2. окружность — путь `arc` + `fill` + `stroke`;
/// 3. ломаная — путь `moveTo` + четыре `lineTo`, только `stroke`;
/// 4. кубическая кривая Безье — `bezierCurveTo`, только `stroke`;
/// 5. квадратичная кривая — `quadraticCurveTo`, только `stroke`;
/// 6. три размера текста — 16px, 12px, 9px (`fillText` + `font`);
/// 7. прямоугольник с линейным градиентом (`createLinearGradient`).
fn draw_scene(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
    brush: Brush,
) -> Result<(), JsValue> {
    clear(ctx, w, h);

    // Установка стиля кисти — состояние контекста.
    ctx.set_line_width(brush.line_width);
    ctx.set_line_cap("round");
    ctx.set_line_join(brush.line_join);
    ctx.set_stroke_style_str(brush.stroke);
    ctx.set_fill_style_str(brush.fill);

    // 1. Прямоугольник — заливка + обводка.
    ctx.fill_rect(20.0, 20.0, 100.0, 70.0);
    ctx.stroke_rect(20.0, 20.0, 100.0, 70.0);

    // 2. Окружность — путь + fill + stroke.
    ctx.begin_path();
    ctx.arc(200.0, 55.0, 35.0, 0.0, std::f64::consts::TAU)?;
    ctx.fill();
    ctx.stroke();

    // 3. Ломаная — пять точек, только обводка.
    ctx.begin_path();
    ctx.move_to(260.0, 90.0);
    ctx.line_to(290.0, 30.0);
    ctx.line_to(320.0, 80.0);
    ctx.line_to(350.0, 25.0);
    ctx.line_to(380.0, 70.0);
    ctx.stroke();

    // 4. Кубическая кривая Безье — «волна».
    ctx.begin_path();
    ctx.move_to(20.0, 140.0);
    ctx.bezier_curve_to(90.0, 100.0, 170.0, 180.0, 240.0, 140.0);
    ctx.stroke();

    // 5. Квадратичная кривая — «горка».
    ctx.begin_path();
    ctx.move_to(260.0, 140.0);
    ctx.quadratic_curve_to(320.0, 90.0, 380.0, 140.0);
    ctx.stroke();

    // 6. Текст — три размера.
    ctx.set_fill_style_str("#c9d1d9");
    ctx.set_font("16px monospace");
    ctx.fill_text("Large 16px", 20.0, 180.0)?;
    ctx.set_font("12px monospace");
    ctx.fill_text("Medium 12px", 20.0, 200.0)?;
    ctx.set_font("9px monospace");
    ctx.fill_text("Small 9px", 20.0, 216.0)?;

    // 7. Градиент — линейный переход.
    if let Some(grad) = make_gradient(
        ctx, 220.0, 160.0, 380.0, 210.0, brush.stroke, "#3fb950",
    ) {
        ctx.set_fill_style_canvas_gradient(&grad);
        ctx.fill_rect(220.0, 160.0, 160.0, 50.0);
    }

    Ok(())
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Прямоугольники — четыре простых метода (<code>fillRect</code>, \
     <code>strokeRect</code>, <code>clearRect</code>, <code>rect</code>). \
     Всё остальное — <strong>пути</strong>: <code>beginPath</code>, \
     <code>moveTo</code>, <code>lineTo</code>, <code>arc</code>, \
     <code>bezierCurveTo</code>, потом <code>fill()</code> или <code>stroke()</code>.",
    "Стиль — <strong>состояние контекста</strong>: установили <code>lineWidth = 2</code> — \
     все последующие линии толщиной 2, пока не измените. Это не параметр вызова, \
     а свойство «кисти», с которой рисуете.",
    "Крутите слайдеры ниже. Сцена перерисовывается целиком при каждом изменении — \
     потому что canvas императивен: чтобы «убрать» фигуру, её надо перерисовать \
     заново, никаких <code>removeShape</code> тут нет.",
];

const CANVAS_W: u32 = 420;
const CANVAS_H: u32 = 240;

struct DemoState {
    line_width: f64,
    stroke_idx: usize,
    fill_idx: usize,
    line_join_idx: usize,
}

impl Default for DemoState {
    fn default() -> Self {
        Self {
            line_width: 2.0,
            stroke_idx: 0,
            fill_idx: 0,
            line_join_idx: 1, // round
        }
    }
}

pub fn run() -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("no document"))?;

    // Стили — до всего остального. Правок в main.css не требуется.
    inject_styles(&document)?;

    let root = document
        .get_element_by_id("app")
        .ok_or_else(|| JsValue::from_str("no #app"))?;
    root.set_inner_html("");

    let h2 = document.create_element("h2")?;
    h2.set_text_content(Some("21.2 — Drawing API: фигуры, пути, текст"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Панель управления
    let controls = document.create_element("div")?;
    controls.set_class_name("dr-controls");
    controls.set_inner_html(
        r#"<div class="dr-control-row">
               <label class="dr-label" for="dr-width">Толщина:</label>
               <input type="range" id="dr-width" class="dr-slider"
                      min="0.5" max="10" step="0.5" value="2">
               <span id="dr-width-val" class="dr-value">2.0px</span>
           </div>
           <div class="dr-control-row">
               <label class="dr-label" for="dr-stroke">Обводка:</label>
               <input type="range" id="dr-stroke" class="dr-slider"
                      min="0" max="4" step="1" value="0">
               <span id="dr-stroke-val" class="dr-value">#58a6ff</span>
           </div>
           <div class="dr-control-row">
               <label class="dr-label" for="dr-fill">Заливка:</label>
               <input type="range" id="dr-fill" class="dr-slider"
                      min="0" max="4" step="1" value="0">
               <span id="dr-fill-val" class="dr-value">rgba(88,166,255,0.25)</span>
           </div>
           <div class="dr-control-row">
               <label class="dr-label" for="dr-join">Соединение:</label>
               <input type="range" id="dr-join" class="dr-slider"
                      min="0" max="2" step="1" value="1">
               <span id="dr-join-val" class="dr-value">round</span>
           </div>"#,
    );
    root.append_child(&controls)?;

    // Холст. Размеры задаются CSS-классом `.dr-canvas`.
    let canvas_el = document.create_element("canvas")?;
    canvas_el.set_id("dr-canvas");
    canvas_el.set_class_name("dr-canvas");
    let canvas: HtmlCanvasElement = canvas_el.dyn_into()?;
    canvas.set_width(CANVAS_W);
    canvas.set_height(CANVAS_H);
    root.append_child(&canvas)?;

    // Статус
    let status = document.create_element("div")?;
    status.set_class_name("dr-status");
    status.set_id("dr-status");
    status.set_text_content(Some("…"));
    root.append_child(&status)?;

    // Footnote
    let footnote = document.create_element("p")?;
    footnote.set_class_name("dr-footnote");
    footnote.set_text_content(Some(
        "Обратите внимание: measureText возвращает ширину строки в CSS-пикселях. \
         Это нужно для центрирования текста, автоподбора размера шрифта и проверки, \
         что длинная строка поместится в область. Без него fillText рисует текст \
         от левого края, а не от центра.",
    ));
    root.append_child(&footnote)?;

    // Начальное состояние и первая отрисовка.
    let state = Rc::new(RefCell::new(DemoState::default()));
    {
        let st = state.borrow();
        render_scene(&document, &st)?;
        update_slider_labels(&document, &st)?;
    }

    // Обработчики — четыре слайдера, каждый обновляет своё поле
    // и перерисовывает сцену.
    wire_line_width(&document, state.clone())?;
    wire_stroke(&document, state.clone())?;
    wire_fill(&document, state.clone())?;
    wire_line_join(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_line_width(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let slider = get_input(document, "dr-width")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("dr-width") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(v) = s.value().parse::<f64>() else {
            return;
        };
        state.borrow_mut().line_width = v;
        let st = state.borrow();
        let _ = render_scene(&doc, &st);
        let _ = update_slider_labels(&doc, &st);
    });
    slider.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_stroke(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let slider = get_input(document, "dr-stroke")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("dr-stroke") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(v) = s.value().parse::<usize>() else {
            return;
        };
        state.borrow_mut().stroke_idx = v.min(STROKES.len() - 1);
        let st = state.borrow();
        let _ = render_scene(&doc, &st);
        let _ = update_slider_labels(&doc, &st);
    });
    slider.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_fill(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let slider = get_input(document, "dr-fill")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("dr-fill") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(v) = s.value().parse::<usize>() else {
            return;
        };
        state.borrow_mut().fill_idx = v.min(FILLS.len() - 1);
        let st = state.borrow();
        let _ = render_scene(&doc, &st);
        let _ = update_slider_labels(&doc, &st);
    });
    slider.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_line_join(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let slider = get_input(document, "dr-join")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("dr-join") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(v) = s.value().parse::<usize>() else {
            return;
        };
        state.borrow_mut().line_join_idx = v.min(LINE_JOINS.len() - 1);
        let st = state.borrow();
        let _ = render_scene(&doc, &st);
        let _ = update_slider_labels(&doc, &st);
    });
    slider.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Рендер ----------

fn render_scene(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("dr-canvas")
        .ok_or_else(|| JsValue::from_str("no canvas"))?
        .dyn_into()?;
    let ctx = get_2d_context(&canvas)?;
    let brush = Brush::from_state(state);
    draw_scene(&ctx, CANVAS_W as f64, CANVAS_H as f64, brush)?;

    // Обновляем статус: lineWidth, lineJoin и measureText.
    ctx.set_font("16px monospace");
    let width = ctx
        .measure_text("Large 16px")
        .map(|m| m.width())
        .unwrap_or(0.0);

    if let Some(el) = document.get_element_by_id("dr-status") {
        let text = format_str(format_args!(
            "lineWidth = {:.1} · lineJoin = {} · measureText(\"Large 16px\") = {:.1}px",
            state.line_width, LINE_JOINS[state.line_join_idx], width,
        ));
        el.set_text_content(Some(&text));
    }
    Ok(())
}

fn update_slider_labels(
    document: &Document,
    state: &DemoState,
) -> Result<(), JsValue> {
    if let Some(el) = document.get_element_by_id("dr-width-val") {
        el.set_text_content(Some(&format_str(format_args!("{:.1}px", state.line_width))));
    }
    if let Some(el) = document.get_element_by_id("dr-stroke-val") {
        el.set_text_content(Some(STROKES[state.stroke_idx]));
        if let Some(h) = el.dyn_ref::<HtmlElement>() {
            let style = format_str(format_args!("color:{}", STROKES[state.stroke_idx]));
            let _ = h.set_attribute("style", &style);
        }
    }
    if let Some(el) = document.get_element_by_id("dr-fill-val") {
        el.set_text_content(Some(FILLS[state.fill_idx]));
    }
    if let Some(el) = document.get_element_by_id("dr-join-val") {
        el.set_text_content(Some(LINE_JOINS[state.line_join_idx]));
    }
    Ok(())
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("dr-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("dr-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}

// ---------- Мелочи ----------

fn get_input(document: &Document, id: &str) -> Result<HtmlInputElement, JsValue> {
    document
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(&format!("no input #{id}")))?
        .dyn_into::<HtmlInputElement>()
        .map_err(|_| JsValue::from_str(&format!("#{id} is not input")))
}
```

---

## 4. `src/lib.rs` — добавление

```rust
pub mod canvas_dpi;       // 21.1
pub mod canvas_drawing;   // ← 21.2
```

---

## 5. `Cargo.toml` — дополнение к `[dependencies.web-sys] features`

```toml
    "CanvasRenderingContext2d",   # уже было
    "HtmlCanvasElement",          # уже было
    "HtmlElement",                # из 21.1
    "CssStyleDeclaration",        # из 21.1
    "CanvasGradient",             # НОВОЕ — для createLinearGradient
    "TextMetrics",                # НОВОЕ — для measureText
```

---

## Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. wasm-тесты Drawing API
wasm-pack test --headless --chrome --test wasm_canvas_drawing

# 3. Полный набор
npm test

# 4. Демо
npm run wasm:dev && npm start
```

## Что нужно помнить

**`inject_styles` идемпотентна.** Повторный вызов снимает старый `<style id="dr-injected-style">` и добавляет новый — правки в коде всегда применяются без перезагрузки. Это правило проекта: любая операция должна быть безопасной при повторе.

**`read_pixel` в тестах — cfg-гейт.** Две ветки: под `web_sys_unstable_apis` (`i32`) и без флага (`f64`). Внешний интерфейс один — `f64`-координаты. Тесты собираются в обоих режимах сборки без правок.

**Ни одного `canvas.style()`.** Размер холста задаётся CSS-классом `.dr-canvas { width: 420px; height: 240px; }`. Никаких upcast'ов к `HtmlElement` — это осталось только в модуле `canvas_dpi` из 21.1, где `style()` нужен для динамического размера.

**Правок в `web/styles/main.css` не требуется.** Если в предыдущей итерации вы добавляли туда `.dr-*` — удалите. Теперь они живут только в `app.rs`.