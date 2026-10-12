# Глава 21.3. Состояние, трансформации, композиция

## 21.3.1. О чём эта подглава

Две предыдущие подглавы научили нас настраивать холст (21.1) и рисовать на нём фигуры, текст и градиенты (21.2). К этому моменту мы умеем создать прямоугольник, окружность, кривую, вывести текст. Но все эти операции в текущем виде **плоские**: они рисуются на одном уровне, поверх фона, без взаимодействия друг с другом. Как только появляется второй слой, третий слой, поворот или прозрачность — начинаются вопросы.

Эта подглава отвечает на три из них: **порядок**, **координаты**, **наложение**.

**Стек состояний.** Что произойдёт, если я установлю `lineWidth = 10`, нарисую одну фигуру, а потом захочу вернуть `lineWidth = 2` для следующей? Без стека придётся вручную помнить все установленные значения. Со стеком — `save()` перед группой, `restore()` после, и контекст вернётся к предыдущему состоянию. Это то же самое, что transaction в базах данных: сохранить точку, сделать группу операций, откатиться.

**Трансформации.** Как нарисовать квадрат, повёрнутый на 45°? Или сдвинуть всю сцену на 100 пикселей вправо? Ответ — матрица трансформации. И важное: она применяется **к системе координат**, а не к фигуре. После `translate(100, 0)` все последующие операции отсчитывают координаты от новой точки. Это сбивает с толку, пока не привыкнешь, — и поэтому нужен наглядный пример.

**Композиция.** Как нарисовать прозрачный слой поверх другого? Как смешать цвета в режиме `multiply`? Как ограничить рисование круглой областью? Всё это — `globalAlpha`, `globalCompositeOperation`, `clip()`.

Подглава — про **порядок и взаимодействие**. Она не вводит новых фигур (это было в 21.2) и не углубляется в пиксели (это будет в 21.4). Она показывает, как управлять уже нарисованным и как выстраивать сцены из нескольких элементов.

## 21.3.2. Что уже есть и что появится

К концу 21.2 мы умеем:
- рисовать прямоугольники, окружности, кривые, текст, градиенты;
- задавать стиль кисти — толщина, цвет, закругление;
- перерисовывать всю сцену при каждом изменении.

К концу 21.3 будем уметь:
- **сохранять и восстанавливать** состояние контекста через `save()` / `restore()`;
- **сдвигать, масштабировать и поворачивать** систему координат;
- понимать разницу между `transform`, `setTransform` и `resetTransform`;
- **смешивать слои** через `globalAlpha` и `globalCompositeOperation`;
- **ограничивать рисование** произвольной областью через `clip()`;
- читать текущую матрицу трансформации через `getTransform()`.

Тема отвечает на ключевой вопрос: что происходит, когда фигур становится **много**.

## 21.3.3. Стек состояний: save и restore

Контекст canvas — это не «набор параметров вызова», а **состояние**, накапливаемое от операции к операции. Установили `fillStyle = "red"` — все последующие `fill()` будут красными, пока не измените. Установили `lineWidth = 10` — все обводки будут толщиной 10. Это уже разбиралось в 21.2, но тогда мы просто выставляли стиль перед каждой группой фигур вручную.

Сложные сцены так делать неудобно. Представьте: рисуете заголовок крупным шрифтом, потом — подпись мелким, потом — снова заголовок. Приходится постоянно возвращать прежние значения: `ctx.font = "24px"`, `ctx.fillStyle = "#fff"`, `ctx.textAlign = "center"`, и через десяток строк всё это откатывать. Забыли один параметр — сцена поехала.

Решение — **стек состояний**. Два метода:

- `ctx.save()` — положить текущее состояние на вершину стека;
- `ctx.restore()` — снять верхнее состояние и применить его.

Что именно входит в состояние? Спецификация перечисляет: `strokeStyle`, `fillStyle`, `globalAlpha`, `lineWidth`, `lineCap`, `lineJoin`, `miterLimit`, `lineDashOffset`, `shadowOffsetX`, `shadowOffsetY`, `shadowBlur`, `shadowColor`, `globalCompositeOperation`, `font`, `textAlign`, `textBaseline`, `direction`, `imageSmoothingEnabled`, текущую **матрицу трансформации**, текущий **clip-путь**, а также image smoothing quality. Практически — **всё, что можно установить на контексте и что влияет на рисование**.

Стек не ограничен глубиной (кроме памяти). `save()` вызывается сколько угодно раз, `restore()` снимает по одному. Если стек пуст, `restore()` — no-op (не ошибка). Асимметрия (пять `save()` и три `restore()`) допустима, но оставшиеся два состояния «зависнут» в стеке — вреда нет, но и пользы тоже.

**Классическая ошибка.** Забыть `restore()` после группы операций. Тогда все стили, установленные внутри группы, «протекут» в следующие. В демо есть специальная галочка `save()` — она включает и выключает `save`/`restore` вокруг второй фигуры, и наглядно показывает, что происходит без них.

## 21.3.4. Матрица трансформации

Каждый рисунок на canvas проходит через **аффинное преобразование** — матрицу 2×3:

```
[ a c e ]
[ b d f ]
```

`a, b, c, d` — линейная часть (масштаб, поворот, скос), `e, f` — сдвиг. Точка `(x, y)` превращается в `(a·x + c·y + e, b·x + d·y + f)`.

По умолчанию матрица — **единичная**: `a=1, b=0, c=0, d=1, e=0, f=0`. Точка остаётся собой.

Три базовых метода изменяют матрицу:

- `translate(dx, dy)` — сдвигает. Изменяет `e` и `f`.
- `scale(sx, sy)` — масштабирует. Изменяет `a` и `d`.
- `rotate(angle)` — поворачивает. Изменяет все четыре линейных компонента.

Все три **умножают** текущую матрицу на новую. Это ключевая деталь: если вызвать `translate(10, 0)` дважды, сдвиг будет на 20, а не на 10. Трансформации **накапливаются**.

Отсюда правило: любая группа операций с трансформацией должна быть обёрнута в `save` / `restore`. Без этого следующая группа получит накопленный сдвиг, масштаб или поворот.

Дополнительные методы:

- `transform(a, b, c, d, e, f)` — умножить матрицу на заданную (накопление).
- `setTransform(a, b, c, d, e, f)` — **заменить** матрицу целиком (сброс).
- `resetTransform()` — установить единичную матрицу.

`setTransform` — способ «обнулиться» без `save`/`restore`. Полезен, когда вы точно знаете, что никакое предыдущее состояние не нужно. `resetTransform` — то же самое, но без аргументов.

**Важная тонкость.** Трансформации применяются к **системе координат**, а не к уже нарисованному. После `translate(100, 0)` все `fillRect(0, 0, 10, 10)` рисуют квадрат в физической точке `(100, 0)`. Это как переставить карандаш, а не сдвинуть лист бумаги.

В демо правый холст показывает это вживую: три квадрата, второй смещён через `translate`, третий — с `restore` или без. Без `restore` третий квадрат уезжает вслед за вторым, потому что трансформация накопилась.

## 21.3.5. Композиция: globalAlpha и globalCompositeOperation

Когда вы рисуете что-то поверх уже нарисованного, по умолчанию новое **заменяет** старое (в области, где они пересекаются). Это режим `source-over` — «источник поверх назначения». Но существуют и другие режимы: смешивание цветов, инверсия, осветление. Управляются они двумя свойствами:

**`globalAlpha`** — глобальная прозрачность. Значение от 0 до 1. Устанавливается **один раз**, действует на все последующие операции рисования. При `globalAlpha = 0.5` новый слой смешивается со старым в пропорции 50/50: результат = `0.5·новый + 0.5·старый`. В отличие от `fillStyle = "rgba(..., 0.5)"`, который задаёт альфу для конкретной заливки, `globalAlpha` умножается **на альфу источника**. Два эффекта складываются: `rgba(255, 0, 0, 0.5)` при `globalAlpha = 0.5` даёт итоговую прозрачность 0.25.

**`globalCompositeOperation`** — режим смешивания. Строка, значения: `source-over` (по умолчанию), `source-in`, `source-out`, `source-atop`, `destination-over`, `destination-in`, `destination-out`, `destination-atop`, `lighter`, `copy`, `xor`, `multiply`, `screen`, `overlay`, `darken`, `lighten`, `color-dodge`, `color-burn`, `hard-light`, `soft-light`, `difference`, `exclusion`, `hue`, `saturation`, `color`, `luminosity`.

Полный список — 26 значений. Практически используются пять-шесть:

- **`source-over`** — обычное наложение. То, что рисуем сейчас, кладётся поверх.
- **`multiply`** — умножение. Результат = `source · destination`. Белый `source` не меняет `destination`, чёрный — делает чёрным. Затемняет.
- **`screen`** — инверсия умножения. `1 - (1 - s)·(1 - d)`. Осветляет.
- **`overlay`** — комбинация multiply и screen в зависимости от `destination`. Увеличивает контраст.
- **`difference`** — модуль разности. Чёрный `source` не меняет `destination`, белый — инвертирует.
- **`lighter`** — сложение с клиппингом к 1.0. Быстро даёт пересветы.

В демо — шесть режимов в выпадашке. Пользователь переключает, видит, как меняется пересечение слоёв. Разница в цветах в пересечении — и есть режим смешивания в действии.

**Зачем это нужно.** В графических редакторах слои смешиваются именно так — Photoshop, Figma, GIMP используют те же формулы. Режимы смешивания canvas совпадают с W3C Compositing and Blending Level 1, из которого они и пришли.

## 21.3.6. Clipping: clip()

`clip()` — метод, который ограничивает область рисования **текущим путём**. После вызова всё, что рисуется на контексте, обрезается по этому пути. За пределами пути пиксели не меняются.

```js
ctx.save();
ctx.beginPath();
ctx.arc(cx, cy, r, 0, Math.PI * 2);
ctx.clip();               // теперь всё рисуется внутри круга
ctx.fillStyle = "red";
ctx.fillRect(0, 0, w, h); // закрасит только круг
ctx.restore();            // снимает clip
```

**Ключевые детали:**

- Clip — **не маска изображения**, а **состояние**. Он действует на все последующие операции, пока не будет снят через `restore()`.
- Clip **пересекается** с уже установленным clip. Если внутри существующей клип-области установить новый путь и вызвать `clip()` ещё раз, результирующая область — пересечение.
- Путь, использованный для clip, **не удаляется** — его можно залить или обвести отдельно.
- Clip снимается только через `restore()`. Нельзя «вернуть всё как было» без стека состояний.

В демо clip включается одной галочкой: верхний слой рисуется внутри круга, за пределами круга — не рисуется. Классическая задача — «показать картинку внутри круглой рамки», «обрезать график по оси», «сделать маску для текста».

## 21.3.7. Чтение матрицы: getTransform

`ctx.getTransform()` возвращает текущую матрицу как объект `DomMatrix`. У него шесть свойств: `a, b, c, d, e, f`. Это те же компоненты, которые принимает `setTransform`.

Зачем это нужно в коде:

- **В тестах.** Проверка «после `save` + `translate` + `restore` матрица вернулась к исходной» возможна только через `getTransform`. Пиксели тут не помогут — они не различают «сдвинуто ровно так же» от «сдвинуто вообще никак».
- **Для отладки.** Когда сцена «уехала», посмотреть `getTransform()` — первое, что стоит сделать. Видно накопленный сдвиг, масштаб, поворот.
- **Для синхронизации.** Если два canvas'а должны иметь одинаковое преобразование (например, основной и мини-превью), можно копировать матрицу туда-обратно.

В `web-sys` этот метод возвращает `Result<DomMatrix, JsValue>` — на случай, если состояние контекста повреждено. В тестах мы используем `read_transform` как обёртку, которая возвращает кортеж из шести `f64` и не падает.

## 21.3.8. Что показывает демо

Демо — **сцена композиции**. Два холста рядом и четыре контрола.

**Левый холст — «Слои и композиция».** Три прямоугольника: синий, зелёный со смещением, красный — верхний. Три контрола:

1. **Слайдер Alpha** — `globalAlpha` верхнего слоя, от 0 до 1. Крутите — красный прямоугольник плавно становится прозрачнее или плотнее, смешиваясь с зелёным и синим под ним.
2. **Выпадашка Blend** — `globalCompositeOperation`. Шесть режимов: `source-over`, `multiply`, `screen`, `overlay`, `difference`, `lighter`. Каждый даёт визуально разный результат в пересечении.
3. **Чекбокс Clipping** — включает `clip()` вокруг верхнего слоя. Красный прямоугольник обрезается по кругу в правой части холста. За пределами круга — пусто (видны нижние слои).

**Правый холст — «save() vs без save».**

1. **Чекбокс save** — переключает использование `save`/`restore` вокруг второй фигуры.

При включённом save три квадрата: синий в (20, 20), зелёный в (120, 60), оранжевый в (20, 20). Все три — на разных местах, никакого накопления.

При выключенном save: синий в (20, 20), зелёный в (120, 60), оранжевый — **в (120, 60)**. Потому что после второго квадрата `translate(100, 40)` накопился, и третий рисуется уже со сдвигом. Пользователь видит это глазами: одно нажатие галочки — и оранжевый квадрат «перескакивает» из угла в сдвинутое положение.

**Внизу — footnote.** Объясняет ловушку: без `restore` трансформации накапливаются.

## 21.3.9. Wow-момент

**Слева.** Двигаем alpha от 0.8 к 0.2 — красный слой становится прозрачнее, виден зелёный и синий под ним. Переключаем blend на `multiply` — пересечение становится тёмным. На `screen` — светлым. На `difference` — яркими становятся только края пересечений. Включаем clip — красный слой обрезается по кругу, прямоугольный угол исчезает.

**Справа.** Снимаем галочку save — оранжевый квадрат прыгает вправо-вниз. Возвращаем — он прыгает обратно. Тридцать секунд кликов — и понятно, зачем нужен стек состояний.

Это не «показываем числа», а **показываем разницу на экране**. Студент видит, как выбор режима меняет картинку — так же, как в Photoshop. А то, что стоит за этим всего одна строка кода (`set_global_composite_operation("multiply")`), — делает тему доступной.

## 21.3.10. Чему это учит

**Первое.** Стек состояний — не «удобство», а **фундамент**. Любая нетривиальная сцена без `save`/`restore` рано или поздно сломается. Забыли вернуть `lineWidth` — сломается вёрстка следующего элемента. Забыли сбросить `globalAlpha` — следующий слой получит не ту прозрачность. Все ошибки этого класса решаются одной привычкой: **группа операций обёрнута в save/restore**.

**Второе.** Трансформации **накапливаются**, а не устанавливаются. `translate(10, 0)` дважды = `translate(20, 0)`. Это отличается от `setTransform`, который заменяет матрицу целиком. Забыть про накопление — классическая ошибка. В демо есть `set_transform` в начале и конце функции — это «страховка» от накопления между вызовами.

**Третье.** Композиция — **математика над пикселями**, а не магия. `multiply` даёт `s·d`, `screen` — `1-(1-s)·(1-d)`, `overlay` — комбинацию по значению `d`. Формулы — из спецификации. Понимание формул позволяет предсказать результат до того, как он появится на экране.

**Четвёртое.** Clip — **состояние**, а не разовая операция. Он действует до `restore`, а не до конца текущего «рисования». Это даёт больше гибкости (можно нарисовать несколько фигур внутри clip-области), но требует симметричного `save`/`restore`. Не забывайте `restore` после clip.

**Пятое.** Матрица доступна для чтения. `getTransform` в тестах и в отладке — незаменимо. Без него проверить «трансформация восстановилась» можно только через пиксели, а пиксели не различают «сдвиг был и вернулся» от «сдвига не было».

## 21.3.11. Ссылки

- MDN: [`save()`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/save)
- MDN: [`restore()`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/restore)
- MDN: [Transformations](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Transformations)
- MDN: [`globalCompositeOperation`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation)
- MDN: [`globalAlpha`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalAlpha)
- MDN: [`clip()`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/clip)
- MDN: [`getTransform()`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/getTransform)
- W3C: [Compositing and Blending Level 1](https://www.w3.org/TR/compositing-1/)
- WHATWG Spec: [Transformations](https://html.spec.whatwg.org/multipage/canvas.html#transformations)

## 21.3.12. Исходные файлы

### `src/canvas_composition.rs`

```rust
//! 21.3. Состояние, трансформации, композиция.
//!
//! Три группы операций над контекстом:
//!
//! 1. **Стек состояний** — `save()` / `restore()`. Запоминает и восстанавливает
//!    всё, что входит в состояние: стили, матрицу трансформации, clip,
//!    `globalAlpha`, `globalCompositeOperation`, `font`, `textAlign` и др.
//!
//! 2. **Матрица трансформации** — `translate`, `rotate`, `scale`,
//!    `setTransform`, `resetTransform`. Трансформации применяются к
//!    системе координат, а не к уже нарисованному.
//!
//! 3. **Композиция** — `globalAlpha` (прозрачность) и
//!    `globalCompositeOperation` (режим смешивания). Плюс `clip()` —
//!    маска, ограничивающая область рисования.
//!
//! Модуль не зависит от предыдущих подглав: только `web_sys` и стандартная
//! библиотека.

use wasm_bindgen::prelude::*;
use web_sys::{CanvasRenderingContext2d, DomMatrix};

// =============================================================================
// Композиция слоёв
// =============================================================================

/// Описание прямоугольного слоя.
#[derive(Clone, Copy, Debug)]
pub struct Layer {
    pub color: &'static str,
    pub x: f64,
    pub y: f64,
    pub w: f64,
    pub h: f64,
}

/// Рисует три наложенных прямоугольника. Верхний слой — красный —
/// рисуется с заданными `alpha` и `blend_mode`; при `use_clip = true`
/// его область ограничена кругом в правой части холста.
///
/// Слои нарисованы так, чтобы пересечения были видны глазами:
/// - синий и зелёный перекрываются слегка;
/// - красный перекрывает оба.
pub fn draw_layers(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
    top_alpha: f64,
    blend_mode: &str,
    use_clip: bool,
) -> Result<(), JsValue> {
    // Фон.
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);

    // Слой 1 — синий.
    let l1 = Layer {
        color: "#1f6feb",
        x: 30.0,
        y: 30.0,
        w: 140.0,
        h: 140.0,
    };
    ctx.set_fill_style_str(l1.color);
    ctx.fill_rect(l1.x, l1.y, l1.w, l1.h);

    // Слой 2 — зелёный.
    let l2 = Layer {
        color: "#3fb950",
        x: 120.0,
        y: 80.0,
        w: 140.0,
        h: 140.0,
    };
    ctx.set_fill_style_str(l2.color);
    ctx.fill_rect(l2.x, l2.y, l2.w, l2.h);

    // Слой 3 — красный с alpha/blend/clip.
    if use_clip {
        ctx.save();
        // Круговой clip: центр вокруг правой части холста.
        ctx.begin_path();
        let cx = w * 0.72;
        let cy = h * 0.5;
        let r = h * 0.42;
        ctx.arc(cx, cy, r, 0.0, std::f64::consts::TAU)?;
        ctx.clip();
    }

    ctx.set_global_alpha(top_alpha.clamp(0.0, 1.0));
    ctx.set_global_composite_operation(blend_mode)?;
    let l3 = Layer {
        color: "#f85149",
        x: 210.0,
        y: 30.0,
        w: 140.0,
        h: 140.0,
    };
    ctx.set_fill_style_str(l3.color);
    ctx.fill_rect(l3.x, l3.y, l3.w, l3.h);

    if use_clip {
        ctx.restore();
    }

    // Восстановление чистого состояния для последующих операций.
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;
    Ok(())
}

// =============================================================================
// Стек состояний: save / restore
// =============================================================================

/// Рисует три квадрата. Второй смещён через `translate(100, 40)`.
/// Третий рисуется **после** второго, но с `restore()` или без —
/// в зависимости от флага `with_restore`.
///
/// `with_restore = true`:  третий квадрат снова в исходной точке (20, 20).
/// `with_restore = false`: трансформация накопилась, третий квадрат
///                         смещён на (120, 60).
pub fn draw_transform_demo(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
    with_restore: bool,
) -> Result<(), JsValue> {
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;

    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);

    // Квадрат 1 — без трансформации.
    ctx.set_fill_style_str("#58a6ff");
    ctx.fill_rect(20.0, 20.0, 60.0, 60.0);

    // Квадрат 2 — со сдвигом.
    if with_restore {
        ctx.save();
    }
    ctx.translate(100.0, 40.0)?;
    ctx.set_fill_style_str("#3fb950");
    ctx.fill_rect(20.0, 20.0, 60.0, 60.0);
    if with_restore {
        ctx.restore();
    }

    // Квадрат 3 — после сдвига.
    ctx.set_fill_style_str("#f0883e");
    ctx.fill_rect(20.0, 20.0, 60.0, 60.0);

    // Гарантируем чистое состояние на выходе.
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    Ok(())
}

// =============================================================================
// Clipping
// =============================================================================

/// Заполняет весь холст чёрным, затем ограничивает рисование кругом
/// и заливает белым. Результат: круг белый, остальное — чёрное.
///
/// Тесты читают пиксели в центре (должно быть белым) и в углу
/// (должно остаться чёрным) — так проверяется, что clip действительно
/// ограничивает область.
pub fn draw_clip_demo(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
) -> Result<(), JsValue> {
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;

    // Чёрный фон.
    ctx.set_fill_style_str("#000000");
    ctx.fill_rect(0.0, 0.0, w, h);

    // Клип: круг в центре.
    ctx.save();
    ctx.begin_path();
    ctx.arc(w * 0.5, h * 0.5, h * 0.4, 0.0, std::f64::consts::TAU)?;
    ctx.clip();

    // Белая заливка — попадёт только в круг.
    ctx.set_fill_style_str("#ffffff");
    ctx.fill_rect(0.0, 0.0, w, h);
    ctx.restore();

    Ok(())
}

// =============================================================================
// Чтение матрицы трансформации
// =============================================================================

/// Возвращает `(a, b, c, d, e, f)` — компоненты текущей матрицы 2D-аффинного
/// преобразования. Матрица хранится как `[a c e; b d f]`.
///
/// Используется в тестах для проверки, что `save()` / `restore()`
/// действительно восстанавливают трансформацию.
pub fn read_transform(
    ctx: &CanvasRenderingContext2d,
) -> (f64, f64, f64, f64, f64, f64) {
    match ctx.get_transform() {
        Ok(m) => matrix_components(&m),
        Err(_) => (1.0, 0.0, 0.0, 1.0, 0.0, 0.0),
    }
}

fn matrix_components(m: &DomMatrix) -> (f64, f64, f64, f64, f64, f64) {
    (m.a(), m.b(), m.c(), m.d(), m.e(), m.f())
}
```

### `src/lib.rs` — добавление

```rust
pub mod canvas_dpi;           // 21.1
pub mod canvas_drawing;       // 21.2
pub mod canvas_composition;   // ← 21.3
```

### `Cargo.toml` — дополнение к `[dependencies.web-sys] features`

```toml
    "DomMatrix",   # для get_transform() / read_transform
```

### `tests/wasm_canvas_composition.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::{CanvasRenderingContext2d, Document, HtmlCanvasElement};

use hello_wasm02::canvas_composition::{
    draw_clip_demo, draw_layers, draw_transform_demo, read_transform,
};

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

/// Читает RGBA в точке `(x, y)`. Скрывает разницу между `f64`- и
/// `i32`-версиями `getImageData` (зависит от `web_sys_unstable_apis`).
fn read_pixel(ctx: &CanvasRenderingContext2d, x: f64, y: f64) -> Vec<u8> {
    #[cfg(web_sys_unstable_apis)]
    let data = ctx.get_image_data(x as i32, y as i32, 1, 1).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    let data = ctx.get_image_data(x, y, 1.0, 1.0).unwrap();

    data.data().0
}

// ---------- save / restore ----------

#[wasm_bindgen_test]
fn save_restore_restores_identity_transform() {
    let (_c, ctx) = make_canvas(100, 100);
    let before = read_transform(&ctx);
    ctx.save();
    ctx.translate(10.0, 20.0).expect("translate");
    ctx.scale(2.0, 2.0).expect("scale");
    ctx.rotate(std::f64::consts::FRAC_PI_4).expect("rotate");
    ctx.restore();
    let after = read_transform(&ctx);
    assert!((before.0 - after.0).abs() < 1e-9, "a: {before:?} vs {after:?}");
    assert!((before.4 - after.4).abs() < 1e-9, "e: {before:?} vs {after:?}");
    assert!((before.5 - after.5).abs() < 1e-9, "f: {before:?} vs {after:?}");
}

#[wasm_bindgen_test]
fn without_restore_transform_stays() {
    let (_c, ctx) = make_canvas(100, 100);
    ctx.translate(30.0, 5.0).expect("translate");
    let t = read_transform(&ctx);
    assert!((t.4 - 30.0).abs() < 1e-9, "e should be 30, got {}", t.4);
    assert!((t.5 - 5.0).abs() < 1e-9, "f should be 5, got {}", t.5);
}

#[wasm_bindgen_test]
fn transform_demo_with_save_puts_third_square_at_origin() {
    let (_c, ctx) = make_canvas(300, 200);
    draw_transform_demo(&ctx, 300.0, 200.0, true).unwrap();

    // С restore третий квадрат — в (20, 20). Пиксель в (50, 50) оранжевый.
    let px = read_pixel(&ctx, 50.0, 50.0);
    assert!(px[0] > 200, "expected orange, got rgb({},{},{})", px[0], px[1], px[2]);
    assert!(px[1] > 100, "orange has significant green");
}

#[wasm_bindgen_test]
fn transform_demo_without_save_puts_third_square_at_shift() {
    let (_c, ctx) = make_canvas(300, 200);
    draw_transform_demo(&ctx, 300.0, 200.0, false).unwrap();

    // Без restore третий квадрат — в (120, 60). Пиксель в (150, 90) оранжевый.
    let px = read_pixel(&ctx, 150.0, 90.0);
    assert!(px[0] > 200, "expected orange at shifted position, got rgb({},{},{})",
        px[0], px[1], px[2]);
}

// ---------- Композиция ----------

#[wasm_bindgen_test]
fn draw_layers_completes_for_all_modes() {
    let (_c, ctx) = make_canvas(400, 220);
    for mode in [
        "source-over", "multiply", "screen", "overlay", "difference", "lighter",
    ] {
        draw_layers(&ctx, 400.0, 220.0, 0.8, mode, false).expect(mode);
    }
}

#[wasm_bindgen_test]
fn multiply_darkens_intersection() {
    let (_c, ctx) = make_canvas(200, 200);

    // Красный прямоугольник.
    ctx.set_global_composite_operation("source-over").unwrap();
    ctx.set_global_alpha(1.0);
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(0.0, 0.0, 100.0, 100.0);

    // Зелёный прямоугольник со смещением, режим multiply.
    ctx.set_global_composite_operation("multiply").unwrap();
    ctx.set_fill_style_str("#00ff00");
    ctx.fill_rect(50.0, 50.0, 100.0, 100.0);

    // Пересечение (75, 75): multiply даёт (0, 0, 0) — black.
    let cross = read_pixel(&ctx, 75.0, 75.0);
    assert!(cross[0] < 40, "red channel should be near 0, got {}", cross[0]);
    assert!(cross[1] < 40, "green channel should be near 0, got {}", cross[1]);
    assert!(cross[2] < 40, "blue channel should be near 0, got {}", cross[2]);

    // Красная зона вне пересечения (25, 25): (255, 0, 0).
    let red_only = read_pixel(&ctx, 25.0, 25.0);
    assert!(red_only[0] > 200, "red zone should stay red");
    assert!(red_only[1] < 40, "red zone should have no green");
}

#[wasm_bindgen_test]
fn global_alpha_affects_result_color() {
    let (_c, ctx) = make_canvas(50, 50);

    // Красный фон.
    ctx.set_global_composite_operation("source-over").unwrap();
    ctx.set_global_alpha(1.0);
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(0.0, 0.0, 50.0, 50.0);

    // Зелёный с alpha = 0.5.
    ctx.set_global_alpha(0.5);
    ctx.set_fill_style_str("#00ff00");
    ctx.fill_rect(0.0, 0.0, 50.0, 50.0);

    // Результат: смесь 50/50 — примерно (127, 127, 0).
    let px = read_pixel(&ctx, 25.0, 25.0);
    assert!(px[0] > 100 && px[0] < 160, "red should be around 127, got {}", px[0]);
    assert!(px[1] > 100 && px[1] < 160, "green should be around 127, got {}", px[1]);
}

// ---------- Clip ----------

#[wasm_bindgen_test]
fn clip_limits_drawing_area() {
    let (_c, ctx) = make_canvas(200, 200);
    draw_clip_demo(&ctx, 200.0, 200.0).unwrap();

    // Центр — внутри круга, белый.
    let inside = read_pixel(&ctx, 100.0, 100.0);
    assert!(inside[0] > 200, "inside clip should be white, got {}", inside[0]);

    // Угол — вне круга, остался чёрным.
    let outside = read_pixel(&ctx, 5.0, 5.0);
    assert!(outside[0] < 40, "outside clip should be black, got {}", outside[0]);
}

#[wasm_bindgen_test]
fn clip_does_not_persist_after_restore() {
    let (_c, ctx) = make_canvas(200, 200);
    draw_clip_demo(&ctx, 200.0, 200.0).unwrap();

    // После restore клип снят — можно рисовать в углу.
    ctx.set_fill_style_str("#ffff00");
    ctx.fill_rect(0.0, 0.0, 20.0, 20.0);

    let px = read_pixel(&ctx, 5.0, 5.0);
    assert!(px[0] > 200 && px[1] > 200, "yellow after restore, got rgb({},{},{})",
        px[0], px[1], px[2]);
}
```

### `src/app.rs` — автономный демо-пример

```rust
//! Глава 21.3 — Состояние, трансформации, композиция.
//!
//! «Сцена композиции»: два холста рядом.
//! - Слева — слои с управлением: прозрачность верхнего слоя,
//!   режим смешивания, включение clipping.
//! - Справа — наглядная разница между `save()` и без него:
//!   чекбокс переключает, накапливаются ли трансформации.
//!
//! **Файл автономен.** Стили `.co-*` инжектируются в документ через
//! `inject_styles()` — правок в `main.css` не требуется.
//!
//! Wow: снимите галочку «save()» — третий квадрат «уедет» вправо-вниз,
//! потому что трансформация от второго квадрата накопилась. Одна галочка
//! показывает, зачем нужен стек состояний.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{
    CanvasRenderingContext2d, Document, Event, HtmlCanvasElement, HtmlElement,
    HtmlInputElement, HtmlSelectElement, Node,
};

// =============================================================================
// Утилита обхода бага rust-analyzer: `format!` в tail-позиции теряет тип
// =============================================================================

#[inline]
fn format_str(args: Arguments<'_>) -> String {
    args.to_string()
}

// =============================================================================
// Инжекция стилей
// =============================================================================

/// Инжектирует `<style>` с правилами `.co-*` в `<head>`. Идемпотентно:
/// если стиль уже есть — удаляется и добавляется заново, чтобы правки
/// в этой функции всегда применялись без перезагрузки.
fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "co-injected-style";
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
.co-desc {
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
.co-desc-line { margin: 6px 0; }

.co-controls {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px 24px;
}
@media (max-width: 720px) {
    .co-controls { grid-template-columns: 1fr; }
}
.co-control-row {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
}
.co-label {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    min-width: 120px;
}
.co-slider {
    flex: 1;
    min-width: 140px;
    max-width: 220px;
    accent-color: #58a6ff;
}
.co-value {
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #79c0ff;
    min-width: 60px;
}
.co-select {
    padding: 4px 8px;
    background: #0d1117;
    color: #c9d1d9;
    border: 1px solid #30363d;
    border-radius: 4px;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    flex: 1;
    max-width: 220px;
}
.co-check-label {
    font-size: 12px;
    color: #c9d1d9;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    cursor: pointer;
}
.co-checkbox {
    accent-color: #58a6ff;
    width: 16px;
    height: 16px;
    cursor: pointer;
}

.co-canvases {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 16px;
    margin-bottom: 20px;
}
@media (max-width: 800px) {
    .co-canvases { grid-template-columns: 1fr; }
}
.co-panel {
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 8px;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 10px;
}
.co-panel-title {
    font-size: 12px;
    font-weight: 600;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
}
.co-canvas {
    display: block;
    width: 400px;
    height: 220px;
    background: #0d1117;
    border-radius: 4px;
    max-width: 100%;
}

.co-footnote {
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
// Рисование
// =============================================================================

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

/// Рисует три наложенных прямоугольника с alpha/blend/clip на верхнем слое.
///
/// 1. Фон.
/// 2. Синий слой.
/// 3. Зелёный слой со смещением.
/// 4. Красный (верхний) — с заданными `alpha` и `blend_mode`. При
///    `use_clip = true` рисуется внутри круга в правой части холста.
///
/// После рисования состояние восстанавливается: `global_alpha = 1`,
/// `global_composite_operation = "source-over"`. Следующий вызов
/// получит чистый контекст.
fn draw_layers(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
    top_alpha: f64,
    blend_mode: &str,
    use_clip: bool,
) -> Result<(), JsValue> {
    // Фон.
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;
    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);

    // Слой 1 — синий.
    ctx.set_fill_style_str("#1f6feb");
    ctx.fill_rect(30.0, 30.0, 140.0, 140.0);

    // Слой 2 — зелёный.
    ctx.set_fill_style_str("#3fb950");
    ctx.fill_rect(120.0, 80.0, 140.0, 140.0);

    // Слой 3 — красный с alpha/blend/clip.
    if use_clip {
        ctx.save();
        ctx.begin_path();
        let cx = w * 0.72;
        let cy = h * 0.5;
        let r = h * 0.42;
        ctx.arc(cx, cy, r, 0.0, std::f64::consts::TAU)?;
        ctx.clip();
    }

    ctx.set_global_alpha(top_alpha.clamp(0.0, 1.0));
    ctx.set_global_composite_operation(blend_mode)?;
    ctx.set_fill_style_str("#f85149");
    ctx.fill_rect(210.0, 30.0, 140.0, 140.0);

    if use_clip {
        ctx.restore();
    }

    // Восстановление чистого состояния для последующих операций.
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;
    Ok(())
}

/// Рисует три квадрата, где второй смещён. Третий — с restore или без.
///
/// - `with_restore = true`:  третий квадрат снова в исходной точке (20, 20).
/// - `with_restore = false`: трансформация накопилась, третий квадрат
///                           смещён на (120, 60).
///
/// В начале и конце функции трансформация явно сбрасывается через
/// `set_transform` — чтобы гарантировать чистое состояние независимо
/// от предыдущих вызовов.
fn draw_transform_demo(
    ctx: &CanvasRenderingContext2d,
    w: f64,
    h: f64,
    with_restore: bool,
) -> Result<(), JsValue> {
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    ctx.set_global_alpha(1.0);
    ctx.set_global_composite_operation("source-over")?;

    ctx.set_fill_style_str("#0d1117");
    ctx.fill_rect(0.0, 0.0, w, h);

    // Квадрат 1.
    ctx.set_fill_style_str("#58a6ff");
    ctx.fill_rect(20.0, 20.0, 60.0, 60.0);

    // Квадрат 2 — со сдвигом.
    if with_restore {
        ctx.save();
    }
    ctx.translate(100.0, 40.0)?;
    ctx.set_fill_style_str("#3fb950");
    ctx.fill_rect(20.0, 20.0, 60.0, 60.0);
    if with_restore {
        ctx.restore();
    }

    // Квадрат 3.
    ctx.set_fill_style_str("#f0883e");
    ctx.fill_rect(20.0, 20.0, 60.0, 60.0);

    // Возврат к identity-матрице на выходе.
    ctx.set_transform(1.0, 0.0, 0.0, 1.0, 0.0, 0.0)?;
    Ok(())
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "<code>save()</code> и <code>restore()</code> — стек состояния. В состояние входит \
     всё: <code>fillStyle</code>, <code>strokeStyle</code>, <code>lineWidth</code>, \
     матрица трансформации, clipping, <code>globalAlpha</code>, \
     <code>globalCompositeOperation</code>, <code>font</code> и прочее.",
    "Трансформации применяются к <strong>системе координат</strong>, а не к нарисованному. \
     <code>translate(100, 0)</code> сдвигает «карандаш» — все последующие операции \
     отсчитывают координаты от новой точки. Если не вызвать <code>restore()</code>, \
     трансформация накапливается.",
    "Композиция — <code>globalAlpha</code> (прозрачность слоя) и \
     <code>globalCompositeOperation</code> (режим смешивания: multiply, screen, \
     difference и др.). <code>clip()</code> ограничивает область рисования \
     текущим путём — за его пределами пиксели не меняются.",
];

const CANVAS_W: u32 = 400;
const CANVAS_H: u32 = 220;

struct DemoState {
    top_alpha: f64,
    blend_mode: String,
    use_clip: bool,
    with_restore: bool,
}

impl Default for DemoState {
    fn default() -> Self {
        Self {
            top_alpha: 0.8,
            blend_mode: String::from("source-over"),
            use_clip: false,
            with_restore: true,
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
    h2.set_text_content(Some("21.3 — Состояние, трансформации, композиция"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Панель управления: слева — композиция, справа — save/restore.
    let controls = document.create_element("div")?;
    controls.set_class_name("co-controls");
    controls.set_inner_html(
        r#"<div class="co-control-row">
               <label class="co-label" for="co-alpha">Alpha верхнего слоя:</label>
               <input type="range" id="co-alpha" class="co-slider"
                      min="0" max="1" step="0.05" value="0.8">
               <span id="co-alpha-val" class="co-value">0.80</span>
           </div>
           <div class="co-control-row">
               <label class="co-label" for="co-mode">Смешивание:</label>
               <select id="co-mode" class="co-select">
                   <option value="source-over">source-over</option>
                   <option value="multiply">multiply</option>
                   <option value="screen">screen</option>
                   <option value="overlay">overlay</option>
                   <option value="difference">difference</option>
                   <option value="lighter">lighter</option>
               </select>
           </div>
           <div class="co-control-row">
               <label class="co-label">Clipping:</label>
               <label class="co-check-label">
                   <input type="checkbox" id="co-clip" class="co-checkbox">
                   ограничить верхний слой кругом
               </label>
           </div>
           <div class="co-control-row">
               <label class="co-label">save/restore:</label>
               <label class="co-check-label">
                   <input type="checkbox" id="co-save" class="co-checkbox" checked>
                   использовать save() между квадратами
               </label>
           </div>"#,
    );
    root.append_child(&controls)?;

    // Два холста.
    let canvases = document.create_element("div")?;
    canvases.set_class_name("co-canvases");
    canvases.set_inner_html(
        r#"<div class="co-panel">
               <div class="co-panel-title">Слои и композиция</div>
               <canvas id="co-canvas-layers" class="co-canvas"
                       width="400" height="220"></canvas>
           </div>
           <div class="co-panel">
               <div class="co-panel-title">save() vs без save</div>
               <canvas id="co-canvas-save" class="co-canvas"
                       width="400" height="220"></canvas>
           </div>"#,
    );
    root.append_child(&canvases)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("co-footnote");
    footnote.set_text_content(Some(
        "Классическая ловушка canvas: рисование без save/restore накапливает \
         трансформации. Каждый translate / rotate / scale модифицирует текущую \
         матрицу, и она остаётся до следующего setTransform или restore. Правый \
         холст это показывает вживую — переключите чекбокс и посмотрите, куда \
         «уезжает» третий квадрат.",
    ));
    root.append_child(&footnote)?;

    // Начальное состояние.
    let state = Rc::new(RefCell::new(DemoState::default()));
    {
        let st = state.borrow();
        render_all(&document, &st)?;
    }

    // Обработчики — четыре независимых контрола.
    wire_alpha(&document, state.clone())?;
    wire_blend(&document, state.clone())?;
    wire_clip(&document, state.clone())?;
    wire_save(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_alpha(document: &Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let slider = get_input(document, "co-alpha")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("co-alpha") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        let Ok(v) = s.value().parse::<f64>() else {
            return;
        };
        state.borrow_mut().top_alpha = v;
        if let Some(el) = doc.get_element_by_id("co-alpha-val") {
            el.set_text_content(Some(&format_str(format_args!("{v:.2}"))));
        }
        let st = state.borrow();
        let _ = render_layers(&doc, &st);
    });
    slider.add_event_listener_with_callback("input", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_blend(document: &Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("co-mode")
        .ok_or_else(|| JsValue::from_str("no co-mode"))?;
    let select: HtmlSelectElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("co-mode") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlSelectElement>() else {
            return;
        };
        state.borrow_mut().blend_mode = s.value();
        let st = state.borrow();
        let _ = render_layers(&doc, &st);
    });
    select.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_clip(document: &Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let el = get_input(document, "co-clip")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("co-clip") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        state.borrow_mut().use_clip = s.checked();
        let st = state.borrow();
        let _ = render_layers(&doc, &st);
    });
    el.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_save(document: &Document, state: Rc<RefCell<DemoState>>) -> Result<(), JsValue> {
    let el = get_input(document, "co-save")?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |_ev: Event| {
        let Some(s) = doc.get_element_by_id("co-save") else {
            return;
        };
        let Ok(s) = s.dyn_into::<HtmlInputElement>() else {
            return;
        };
        state.borrow_mut().with_restore = s.checked();
        let st = state.borrow();
        let _ = render_save_demo(&doc, &st);
    });
    el.add_event_listener_with_callback("change", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Рендер ----------

fn render_all(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    render_layers(document, state)?;
    render_save_demo(document, state)?;
    Ok(())
}

fn render_layers(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("co-canvas-layers")
        .ok_or_else(|| JsValue::from_str("no layers canvas"))?
        .dyn_into()?;
    let ctx = get_2d_context(&canvas)?;
    draw_layers(
        &ctx,
        CANVAS_W as f64,
        CANVAS_H as f64,
        state.top_alpha,
        &state.blend_mode,
        state.use_clip,
    )
}

fn render_save_demo(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("co-canvas-save")
        .ok_or_else(|| JsValue::from_str("no save canvas"))?
        .dyn_into()?;
    let ctx = get_2d_context(&canvas)?;
    draw_transform_demo(&ctx, CANVAS_W as f64, CANVAS_H as f64, state.with_restore)
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("co-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("co-desc-line");
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

## 21.3.13. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. wasm-тесты
wasm-pack test --headless --chrome --test wasm_canvas_composition

# 3. Полный набор
npm test

# 4. Демо
npm run wasm:dev && npm start
```

## 21.3.14. Итог

Состояние, трансформации, композиция — три механизма, которые управляют **порядком** и **взаимодействием** фигур. Стек состояний — основа; без него любая сложная сцена накапливает трансформации и залипает в непредсказуемом положении. Трансформации — не «изменение координат фигуры», а «изменение системы координат карандаша». Композиция — не «наложение картинок», а математическая операция над пикселями, у которой есть конкретные режимы.

Демо даёт две галочки, слайдер и выпадашку — и показывает всё это на одном экране. Снял галочку «save()» — третий квадрат «уехал». Переключил «multiply» — пересечение затемнилось. Включил clip — верхний слой обрезался. За тридцать секунд — три концепции.

Код модуля автономен, тесты покрывают и пиксели, и матрицы, а `app.rs` инжектирует свои стили `.co-*` без правок в `main.css`. Правила проекта (`format_str`, отсутствие dead_code, идемпотентность обработчиков, `Result` для методов, возвращающих ошибку) применены последовательно.

Следующая подглава **21.4** — про `ImageData`: пиксельный буфер, точку входа в Rust-обработку. Мы умеем рисовать фигуры, теперь заглянем под них — в байты, из которых они состоят.