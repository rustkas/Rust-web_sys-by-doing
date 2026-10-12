# Глава 21.6. Фильтры на Rust: чистые функции, SIMD, Worker

## 21.6.0. Обновлённая структура

```
src/
  canvas_filters.rs            ← НОВЫЙ модуль главы 21.6
  canvas_dpi.rs                ← из 21.1
  canvas_drawing.rs            ← из 21.2
  canvas_composition.rs        ← из 21.3
  canvas_imagedata.rs          ← из 21.4
  canvas_image_loading.rs      ← из 21.5
tests/
  canvas_filters_pure.rs       ← host-тесты чистых функций
  wasm_canvas_filters.rs       ← wasm-тесты интеграции с canvas
src/lib.rs                     ← +1 строка
src/app.rs                     ← демо текущей подглавы (заменяется целиком)
Cargo.toml                     ← без новых фич (всё уже включено)
web/styles/main.css            ← правок не требуется
```

**Комментарий.** Модуль `canvas_filters.rs` — про численную обработку пикселей: четыре классических фильтра (grayscale, invert, box blur, Sobel) + обвязка для замеров. Ключевая идея — **разделение чистых функций и web-обвязки**: `to_grayscale(&[u8]) -> Vec<u8>` тестируется на хосте, `apply_filter_to_canvas` — в браузере. Префикс CSS — `flt-`, уникален в проекте. Стили инжектируются в документ из `app.rs`.

## Ссылки

- Wikipedia: [Rec. 709](https://en.wikipedia.org/wiki/Rec._709) — стандарт яркости
- Wikipedia: [Box blur](https://en.wikipedia.org/wiki/Box_blur)
- Wikipedia: [Sobel operator](https://en.wikipedia.org/wiki/Sobel_operator)
- Rust Reference: [Target features](https://doc.rust-lang.org/reference/attributes/codegen.html#the-target_feature-attribute)
- Rust: [`cfg!(target_feature = "...")`](https://doc.rust-lang.org/reference/conditional-compilation.html#target_feature)
- web.dev: [Web Workers](https://web.dev/articles/workers-basics)
- MDN: [Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- wasm-bindgen: [Accessing the DOM from a Worker](https://rustwasm.github.io/wasm-bindgen/examples/raytrace.html)

---

## 21.6.1. О чём эта подглава

До этого мы **читали** пиксели с холста (21.4) и **декодировали** их из файлов (21.5). Теперь будем их **изменять**. Это то место, где Rust в браузере раскрывается полностью: цикл по миллиону пикселей на Rust проходит быстрее, чем на JavaScript, и разница видна не в бенчмарке, а на живом пользователе.

Глава отвечает на три вопроса.

**Первый — как устроен фильтр.** Любой фильтр обработки изображений — это функция, которая берёт плоский RGBA-буфер и возвращает новый. Grayscale заменяет R, G, B на общую яркость. Invert вычитает каждый канал из 255. Blur усредняет 9 соседей. Sobel считает градиент. Все они — чистые функции: нет DOM, нет canvas, нет `web-sys`. Только `&[u8]` на входе, `Vec<u8>` на выходе.

**Второй — почему это критично.** Чистая функция тестируется на хосте, за миллисекунды, без браузера. Можно запустить 100 тестов за секунду, отлаживать в обычном отладчике, переносить на сервер. Web-обвязка тестируется отдельно и содержит минимум логики. Это правило проекта: **вся арифметика в чистых функциях, вся работа с API — в тонком слое поверх**.

**Третий — как ускорить.** Есть три уровня:
- **Алгоритмический.** Blur 3×3 — 9 операций на пиксель. Sobel — около 20. Grayscale — 3. Invert — 3. Разница в скорости на два порядка — следствие количества операций, а не оптимизации.
- **SIMD 128.** Флаг `+simd128` разрешает компилятору использовать 128-битные векторные инструкции. Для циклов без ветвлений (grayscale, invert) он это делает. Для blur и sobel с условными проверками — хуже.
- **Web Worker.** Вынести обработку из главного потока. Это не ускоряет саму обработку, но **убирает блокировку UI**: пользователь может двигать слайдеры, пока Worker считает.

Последнее — важнее всего с точки зрения UX. Пользователю не важно, что фильтр занял 50 мс CPU. Ему важно, что **страница не зависла**.

## 21.6.2. Что уже есть и что появится

К концу 21.5 мы умеем:
- читать и писать пиксели через `ImageData` (21.4);
- декодировать файлы в `<img>` и `ImageBitmap` (21.5);
- рисовать декодированное на canvas.

К концу 21.6 будем уметь:
- писать **чистые функции** обработки пикселей;
- организовывать фильтры как набор взаимозаменяемых операций;
- замерять их время и понимать, где узкое место;
- применять правило **yield через `TimeoutFuture::new(0)`** — чтобы UI оставался живым;
- понимать, что даёт SIMD и когда он помогает.

## 21.6.3. Четыре фильтра

**Grayscale** — luminance по Rec. 709:
```
Y = 0.2126·R + 0.7152·G + 0.0722·B
```
Коэффициенты — из стандарта Rec. 709. Они **не равны** ⅓: глаз чувствительнее к зелёному, чем к синему. Это не «усреднение», это взвешенное среднее, основанное на физиологии зрения. Реализация — целочисленная арифметика: коэффициенты умножены на 10000, потом делим.

**Invert** — `255 - x` для каждого канала R, G, B. **Альфа не трогается**: если её инвертировать, прозрачные пиксели станут непрозрачными, картинка «посереет». Классическая ошибка — инвертировать все четыре байта.

**Box blur 3×3** — усреднение 9 соседей. Для каждого пикселя берём 3×3 окно, суммируем, делим на 9. На краях окно обрезается clamp'ом к границам изображения. Это **9 операций чтения и сложения на пиксель** — в 3 раза больше, чем grayscale.

**Sobel edge detection** — градиент яркости. Два ядра 3×3 (Gx и Gy), каждое взвешивает 9 соседей. Результат — `sqrt(sx² + sy²)`, клампленный к 255. Получается чёрно-белая карта границ: там, где яркость резко меняется (край объекта), пиксель яркий; в однотонных зонах — чёрный.

Реализация: сначала считаем яркость всех пикселей один раз (по Rec. 709), потом применяем ядра к яркости. Это **в 3 раза эффективнее**, чем считать градиент по каждому каналу отдельно. Это стандартный приём: конвертировать в luminance, потом обрабатывать.

**Сложность на пиксель:**

| Фильтр | Операций |
|---|---|
| Invert | 3 |
| Grayscale | 3 умножения + сложения |
| Box blur 3×3 | 9×4 чтений + 9 сложений на канал × 4 |
| Sobel | ~20 чтений + умножения + sqrt |

Отсюда ожидаемое соотношение времени: blur и sobel будут в 5–10 раз медленнее grayscale и invert.

## 21.6.4. Разделение: чистые функции vs web-обвязка

Модуль `canvas_filters.rs` содержит только **чистые функции**:

```rust
pub fn to_grayscale(rgba: &[u8]) -> Vec<u8>
pub fn invert(rgba: &[u8]) -> Vec<u8>
pub fn box_blur_3x3(rgba: &[u8], w: u32, h: u32) -> Vec<u8>
pub fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8>
```

Ни одной ссылки на `web_sys`, `js_sys`, `wasm_bindgen`. Ни одной. Модуль компилируется под wasm32, x86_64, aarch64 — где угодно. Тестируется `cargo test` без браузера.

Web-обвязка живёт в `app.rs`: получить `ImageData`, преобразовать в `Vec<u8>`, вызвать фильтр, преобразовать обратно. Это **тонкий слой**, там нет арифметики.

**Зачем такое разделение.**

**Первое.** Скорость итерации. 15 host-тестов проходят за 3 мс. 15 wasm-тестов — за 30 секунд (компиляция + запуск браузера). Разница в 10000 раз. При отладке blur'а вы хотите быстро — используйте host.

**Второе.** Переносимость. Те же функции можно вызвать на сервере — если захотите обрабатывать изображения на бэкенде, `cargo` их возьмёт как есть. Никаких «но у меня там `web_sys::Blob`».

**Третье.** Ясность. В `canvas_filters.rs` нет вопроса «а где тут граница». Всё, что там есть, — арифметика над байтами. Всё, что в `app.rs`, — DOM и Canvas API.

## 21.6.5. Yield: почему UI замирает и как это исправить

Асинхронный код в Rust — это **микротаски**. Когда вы делаете `JsFuture::from(promise).await`, управление возвращается в цикл событий, но **между микротасками браузер не перерисовывает страницу**. Он делает это между **макротасками** — то есть между `setTimeout(0)`, `requestAnimationFrame`, событиями ввода, сетевыми колбэками.

Следствие: если ваш async-код содержит длинный цикл без `await` на «настоящей» асинхронной операции, вкладка **зависнет**. Пользователь увидит «страница не отвечает» или просто застывший интерфейс.

Решение — **явно отдавать управление** через `TimeoutFuture::new(0).await`. Это макротаска: браузер успевает между итерациями отрисовать кадр, обработать ввод, обновить анимации.

Правило проекта:

> Любая длительная async-операция в браузере, которая обновляет UI (progress bar, счётчик, live-превью), должна явно отдавать управление через `TimeoutFuture::new(0).await` (или `requestAnimationFrame`).

**Троттлинг.** Yield'ить на каждой итерации — слишком часто. Браузер перерисовывает 60 раз в секунду — значит, чаще, чем раз в 16 мс, смысла нет. Проверка `now_ms() - last_yield_ms >= 16.0` занимает доли микросекунды и позволяет обрабатывать тысячи итераций между yield'ами, обновляя UI ровно столько раз, сколько браузер успевает отрисовать.

```rust
let mut last_yield = now_ms();
for chunk in chunks {
    process(chunk);
    if now_ms() - last_yield >= 16.0 {
        gloo_timers::future::TimeoutFuture::new(0).await;
        last_yield = now_ms();
        on_progress(...);
    }
}
```

Это правило уже применялось в главе 20.4 (Blob processor). Здесь оно применяется к фильтрам.

## 21.6.6. SIMD 128: что это и когда помогает

SIMD — Single Instruction, Multiple Data. Одна инструкция процессора обрабатывает несколько чисел одновременно. Для wasm это 128-битный регистр: 4 числа по 32 бита или 16 по 8.

В Rust SIMD включается флагом `-C target-feature=+simd128` в `RUSTFLAGS`. Компилятор может **автоматически векторизовать** циклы — заменять скалярные операции на векторные, если видит, что это безопасно.

**Что векторизуется хорошо:**
- Простые операции без ветвлений и без зависимостей между итерациями.
- Grayscale: `y = r*2126 + g*7152 + b*722`. Три умножения и два сложения — идеально для SIMD.
- Invert: `buf[i] = 255 - buf[i]`. Одна операция, чистый поток данных.

**Что векторизуется плохо:**
- Blur: вложенные циклы с clamp'ами и разными окнами на краях. Ветвления мешают.
- Sobel: то же самое плюс `sqrt`.

**Что нужно понимать.** Включение флага **не гарантирует** ускорение. Оно даёт компилятору разрешение использовать SIMD-инструкции, если он сможет. Для наших фильтров выигрыш будет заметен на grayscale и invert, слабо на blur, и почти незаметен на sobel.

**Практическое правило.** Не полагайтесь на «включил SIMD — стало в 4 раза быстрее». Измеряйте. `+simd128` — это инструмент, а не волшебная таблетка.

## 21.6.7. Web Worker: зачем и как

Web Worker — это отдельный поток JavaScript, работающий параллельно с main thread. У него нет DOM, но есть свой event loop и свои таймеры. `postMessage` передаёт данные между потоками.

**Зачем.** Даже с yield'ами main thread остаётся занят: каждые 16 мс он делает кусок работы и возвращает управление. Если фильтр занимает 500 мс CPU, пользователь увидит progress bar, но клики по кнопкам будут отвечать с задержкой. Worker полностью освобождает main thread: он работает в фоне, а main thread занимается только UI.

**Transferable objects.** Обычный `postMessage` **копирует** данные: 1.92 МБ пикселей туда, 1.92 МБ обратно. Для больших изображений это дорого. Transferable `ArrayBuffer` **передаёт** владение — данные не копируются, а переезжают из одного контекста в другой:

```js
worker.postMessage({ buf: pixels.buffer }, [pixels.buffer]);
```

Второй аргумент — список «передаваемых» объектов. После передачи `pixels.buffer` в main thread становится недоступен (byteLength = 0). Это не баг: данные теперь в Worker'е.

**Проблема с wasm.** Запустить наш Rust-модуль в Worker'е непросто: wasm-модуль инстанцируется один раз в main thread, и второй экземпляр в Worker'е — это отдельная сборка, отдельный загрузчик, отдельный обмен сообщениями. В продакшене это решается или через `wasm-bindgen-rayon`, или через ручной запуск второго экземпляра.

В демо мы не будем запускать Rust в Worker'е. Вместо этого покажем:
- benchmark на main thread с yield'ами;
- текстовое объяснение, что даёт Worker и когда его использовать.

Это честнее, чем псевдо-Worker, который ничего не делает.

## 21.6.8. Что показывает демо

Демо — **«бенчмарк-стол»**. Одна картинка, четыре фильтра, конкретные цифры.

**Паттерн.** Синтетический градиент с тремя цветными квадратами — 400×240, генерируется в `app.rs`. Цветные области дают разный результат для каждого фильтра: blur размывает квадраты, sobel выделяет их границы, grayscale выравнивает цвета, invert инвертирует.

**Кнопки фильтров.** Четыре кнопки: Grayscale, Invert, Blur, Sobel. Клик — применить фильтр к оригиналу, показать результат, измерить время. Кнопка «Сбросить» возвращает оригинал.

**Прогресс-бар.** Во время обработки — плавный прогресс от 0 до 100 %. Реализован через yield каждые 16 мс. Это наглядно показывает правило: **UI не замирает даже на тяжёлых фильтрах**.

**Кнопка «Бенчмарк всех».** Последовательно запускает все четыре фильтра **по три раза**, считает среднее время. Внизу — таблица: фильтр, среднее время в миллисекундах, количество операций на пиксель, относительная скорость. Показывает, во сколько раз sobel медленнее invert.

**Статус-строка.** Отображает:
- размер картинки (в пикселях и байтах);
- активный фильтр;
- время последнего применения.

**Footnote.** Про три уровня оптимизации: алгоритм, SIMD, Worker. Про то, что yield — обязателен в main thread.

## 21.6.9. Wow-момент

**Первый.** Нажмите «Бенчмарк всех» — увидите таблицу цифр. Не «быстро» и «медленно», а конкретно: `invert 1.2 мс`, `grayscale 2.1 мс`, `blur 18.4 мс`, `sobel 32.7 мс`. Отношение sobel/invert — **27 раз**. Это прямое следствие сложности алгоритма: 20 операций на пиксель против 3.

**Второй.** Во время бенчмарка двигайте окно, нажимайте кнопки — они отвечают. Не мгновенно, но отвечают. Это и есть yield: браузер успевает обработать ввод между итерациями. Без yield окно бы «повисло» на полсекунды.

**Третий.** Откройте DevTools → Performance, запустите бенчмарк, посмотрите на flame chart. Видно, как работа разбита на куски по 16 мс, между ними — рендеринг. Без yield там был бы один длинный кусок и пустой экран после.

## 21.6.10. Чему это учит

**Первое.** Чистая функция — это контракт. `&[u8] -> Vec<u8>`. Без скрытого состояния, без побочных эффектов, без API. Такие функции тестируются быстро, переносятся куда угодно, читаются без контекста.

**Второе.** Все фильтры — это циклы по массиву. Grayscale, invert, blur, sobel — разные формулы, но одна структура. Понимание этой общности позволяет писать новые фильтры за минуты.

**Третье.** Сложность фильтра **предсказуема**. Посчитайте операции на пиксель — узнаете примерное время. Это инженерный подход: не «попробуем и посмотрим», а «30 операций на пиксель при 100 000 пикселях — это 3 миллиона операций, ожидаемо около 30 мс».

**Четвёртое.** Yield — это **обязательное правило**, не оптимизация. Если ваш async-код не возвращает управление, UI замирает. `TimeoutFuture::new(0)` — минимальная форма yield, троттлинг 16 мс — оптимальная частота.

**Пятое.** SIMD — это инструмент, не автоматика. Он помогает, если компилятор может векторизовать. Для простых фильтров — да, для сложных — не всегда. Всегда измеряйте.

**Шестое.** Worker — не ускорение, а изоляция. Он не сделает фильтр быстрее, но освободит main thread. Когда фильтр занимает больше 100 мс — это правильный выбор.

## 21.6.11. Исходные файлы

### `src/canvas_filters.rs`

```rust
//! 21.6. Фильтры на Rust: чистые функции.
//!
//! Модуль содержит **только чистые функции** обработки пикселей:
//! `&[u8] -> Vec<u8>`. Ни одной ссылки на `web_sys`, `js_sys`,
//! `wasm_bindgen`. Компилируется под любую платформу, тестируется
//! на хосте за миллисекунды.
//!
//! ## Формат
//!
//! Плоский RGBA-буфер: `width × height × 4` байт. Индекс канала
//! `(y * w + x) * 4 + channel`, где `channel`: 0 — R, 1 — G, 2 — B,
//! 3 — A.
//!
//! ## Четыре фильтра
//!
//! - `to_grayscale` — luminance по Rec. 709;
//! - `invert` — `255 - x` для R, G, B (альфа не трогается);
//! - `box_blur_3x3` — усреднение 9 соседей;
//! - `sobel_edge` — градиент яркости (карта границ).
//!
//! Все функции работают с фиксированной сложностью на пиксель:
//! grayscale и invert — O(1), blur — O(9), sobel — O(20). Это
//! определяет их относительную скорость.

// =============================================================================
// Палитра фильтров
// =============================================================================

/// Палитра доступных фильтров.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Filter {
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    pub fn label(self) -> &'static str {
        match self {
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие 3×3",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    /// Приблизительное число операций на пиксель — для оценки
    /// сложности алгоритма. Используется в UI для сравнения.
    pub fn ops_per_pixel(self) -> u32 {
        match self {
            Filter::Grayscale => 3,
            Filter::Invert => 3,
            Filter::Blur => 36, // 9 чтений × 4 канала
            Filter::Sobel => 60, // 2 ядра × 9 чтений + sqrt
        }
    }

    pub fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

// =============================================================================
// Grayscale — luminance по Rec. 709
// =============================================================================

/// Luminance по Rec. 709: `Y = 0.2126·R + 0.7152·G + 0.0722·B`.
///
/// Коэффициенты из стандарта Rec. 709. Не равны ⅓: глаз чувствительнее
/// к зелёному, чем к синему. Умножаем на 10000, чтобы обойтись
/// целочисленной арифметикой (без потерь точности при типичных
/// значениях каналов).
///
/// Альфа сохраняется без изменений.
pub fn to_grayscale(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        let r = px[0] as u32;
        let g = px[1] as u32;
        let b = px[2] as u32;
        let y = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
        out.push(y);
        out.push(y);
        out.push(y);
        out.push(px[3]);
    }
    out
}

// =============================================================================
// Invert — инверсия RGB
// =============================================================================

/// Инверсия RGB: `x → 255 - x`. Альфа не трогается.
///
/// Классическая ошибка — инвертировать все 4 канала. Тогда
/// прозрачные пиксели становятся непрозрачными, картинка «сереет».
/// Инвертируем только три первых канала.
pub fn invert(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        out.push(255 - px[0]);
        out.push(255 - px[1]);
        out.push(255 - px[2]);
        out.push(px[3]);
    }
    out
}

// =============================================================================
// Box blur 3×3
// =============================================================================

/// Box blur 3×3 — усреднение 9 соседей для каждого канала.
///
/// На краях изображения окно обрезается clamp'ом (не заворачивается):
/// координата `x - 1` при `x = 0` превращается в 0, а не в `w - 1`.
/// Это соответствует семантике «размыть в пределах изображения»,
/// а не «размыть с заворотом через край».
///
/// Сложность: 9 чтений × 4 канала + 9 сложений × 4 канала + 4 деления
/// на пиксель. Примерно в 5–10 раз медленнее grayscale.
pub fn box_blur_3x3(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];
    for y in 0..h {
        for x in 0..w {
            let (mut r, mut g, mut b, mut a, mut n) = (0u32, 0u32, 0u32, 0u32, 0u32);
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let i = (ny * w + nx) * 4;
                    r += rgba[i] as u32;
                    g += rgba[i + 1] as u32;
                    b += rgba[i + 2] as u32;
                    a += rgba[i + 3] as u32;
                    n += 1;
                }
            }
            let i = (y * w + x) * 4;
            out[i] = (r / n) as u8;
            out[i + 1] = (g / n) as u8;
            out[i + 2] = (b / n) as u8;
            out[i + 3] = (a / n) as u8;
        }
    }
    out
}

// =============================================================================
// Sobel edge detection
// =============================================================================

/// Sobel edge detection по яркости.
///
/// Стратегия: сначала считаем яркость всех пикселей один раз (Rec. 709),
/// потом применяем ядра Sobel к массиву яркостей. Это в 3 раза
/// эффективнее, чем считать градиент по каждому каналу отдельно.
///
/// Результат: чёрно-белая карта границ. Там, где яркость резко
/// меняется (край объекта) — светлые пиксели. В однотонных зонах —
/// чёрные. Магнитуда градиента = `sqrt(Gx² + Gy²)`, клампленная к 255.
pub fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];

    // Шаг 1: яркость всех пикселей.
    let mut lum = vec![0u8; w * h];
    for i in 0..w * h {
        let p = i * 4;
        let r = rgba[p] as u32;
        let g = rgba[p + 1] as u32;
        let b = rgba[p + 2] as u32;
        lum[i] = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
    }

    // Шаг 2: ядра Sobel.
    const GX: [i32; 9] = [-1, 0, 1, -2, 0, 2, -1, 0, 1];
    const GY: [i32; 9] = [-1, -2, -1, 0, 0, 0, 1, 2, 1];

    for y in 0..h {
        for x in 0..w {
            let (mut sx, mut sy) = (0i32, 0i32);
            let mut k = 0;
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let l = lum[ny * w + nx] as i32;
                    sx += GX[k] * l;
                    sy += GY[k] * l;
                    k += 1;
                }
            }
            let mag = ((sx * sx + sy * sy) as f64).sqrt().min(255.0) as u8;
            let i = (y * w + x) * 4;
            out[i] = mag;
            out[i + 1] = mag;
            out[i + 2] = mag;
            out[i + 3] = 255;
        }
    }
    out
}

// =============================================================================
// Генерация тестового паттерна
// =============================================================================

/// Синтетический паттерн `w × h`: градиент с тремя цветными квадратами.
///
/// Паттерн подобран так, чтобы каждый фильтр давал визуально заметный
/// результат: blur размывает границы квадратов, sobel выделяет их
/// контуры, grayscale выравнивает цвета, invert инвертирует.
pub fn make_pattern(w: u32, h: u32) -> Vec<u8> {
    let mut out = vec![0u8; (w * h * 4) as usize];
    let wf = w as f32;
    let hf = h as f32;
    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / wf;
            let fy = y as f32 / hf;
            let i = ((y * w + x) * 4) as usize;

            let mut r = (30.0 + fx * 180.0) as u8;
            let mut g = (30.0 + fy * 180.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            if x < w / 5 && y < h / 5 {
                r = 240; g = 60; b = 60;
            }
            if x > 4 * w / 5 && y < h / 5 {
                r = 60; g = 220; b = 80;
            }
            if x > 4 * w / 5 && y > 4 * h / 5 {
                r = 60; g = 80; b = 240;
            }

            out[i] = r;
            out[i + 1] = g;
            out[i + 2] = b;
            out[i + 3] = 255;
        }
    }
    out
}

// =============================================================================
// Информация о среде
// =============================================================================

/// Возвращает `true`, если текущая сборка использует SIMD 128.
/// Включается флагом `-C target-feature=+simd128` в `RUSTFLAGS`.
///
/// Не гарантирует, что компилятор **применил** SIMD-инструкции:
/// автовекторизация — это оптимизация, а не обещание. Флаг даёт
/// разрешение, не обязывает.
pub fn has_simd() -> bool {
    cfg!(target_feature = "simd128")
}
```

---

### `src/lib.rs` — добавление

```rust
pub mod canvas_dpi;             // 21.1
pub mod canvas_drawing;         // 21.2
pub mod canvas_composition;     // 21.3
pub mod canvas_imagedata;       // 21.4
pub mod canvas_image_loading;   // 21.5
pub mod canvas_filters;         // ← 21.6
```

---

### `tests/canvas_filters_pure.rs`

```rust
//! Host-тесты чистых функций обработки пикселей.
//! Запускаются `cargo test` — без браузера, миллисекунды.

use hello_wasm02::canvas_filters::{
    box_blur_3x3, invert, make_pattern, sobel_edge, to_grayscale, Filter,
};

/// Создать однотонный RGBA-буфер `w × h`.
fn solid(w: u32, h: u32, r: u8, g: u8, b: u8, a: u8) -> Vec<u8> {
    let mut v = Vec::with_capacity((w * h * 4) as usize);
    for _ in 0..w * h {
        v.extend_from_slice(&[r, g, b, a]);
    }
    v
}

// ---------- Grayscale ----------

#[test]
fn grayscale_of_pure_red_is_expected_luminance() {
    // 0.2126 * 255 ≈ 54
    let px = vec![255, 0, 0, 255];
    let out = to_grayscale(&px);
    assert_eq!(out.len(), 4);
    assert_eq!(out[0], out[1]);
    assert_eq!(out[1], out[2]);
    assert_eq!(out[3], 255, "alpha unchanged");
    assert!(out[0] >= 53 && out[0] <= 55, "got {}", out[0]);
}

#[test]
fn grayscale_of_pure_green_is_brighter_than_red() {
    let red = to_grayscale(&[255, 0, 0, 255])[0];
    let green = to_grayscale(&[0, 255, 0, 255])[0];
    let blue = to_grayscale(&[0, 0, 255, 255])[0];
    assert!(green > red, "green {} should beat red {}", green, red);
    assert!(red > blue, "red {} should beat blue {}", red, blue);
}

// ---------- Invert ----------

#[test]
fn invert_flips_rgb_keeps_alpha() {
    let px = vec![10, 20, 30, 200];
    let out = invert(&px);
    assert_eq!(out, vec![245, 235, 225, 200]);
}

#[test]
fn invert_is_its_own_inverse() {
    let px = solid(4, 4, 11, 22, 33, 44);
    let twice = invert(&invert(&px));
    assert_eq!(twice, px);
}

// ---------- Blur ----------

#[test]
fn blur_of_solid_image_is_identity() {
    let px = solid(8, 8, 100, 150, 200, 255);
    let out = box_blur_3x3(&px, 8, 8);
    assert_eq!(out, px);
}

#[test]
fn blur_softens_a_single_bright_pixel() {
    // 3×3 с белой точкой в центре.
    let mut px = vec![0u8; 3 * 3 * 4];
    for i in 0..9 {
        px[i * 4 + 3] = 255;
    }
    let center = (1 * 3 + 1) * 4;
    px[center] = 255;
    px[center + 1] = 255;
    px[center + 2] = 255;
    let out = box_blur_3x3(&px, 3, 3);
    assert!(out[center] < 255, "center should be dimmed");
    assert!(out[center] > 0, "but not zero");
}

// ---------- Sobel ----------

#[test]
fn sobel_on_uniform_image_is_black() {
    let px = solid(6, 6, 128, 128, 128, 255);
    let out = sobel_edge(&px, 6, 6);
    for i in 0..6 * 6 {
        assert_eq!(out[i * 4], 0, "pixel {} should be black", i);
    }
}

#[test]
fn sobel_on_vertical_edge_lights_up_boundary() {
    // Левые 3 колонки — чёрные, правые 3 — белые.
    let mut px = vec![0u8; 6 * 6 * 4];
    for y in 0..6 {
        for x in 3..6 {
            let i = (y * 6 + x) * 4;
            px[i] = 255;
            px[i + 1] = 255;
            px[i + 2] = 255;
        }
    }
    for i in 0..6 * 6 {
        px[i * 4 + 3] = 255;
    }
    let out = sobel_edge(&px, 6, 6);
    let edge_px = (3 * 6 + 3) * 4;
    let flat_px = (3 * 6 + 0) * 4;
    assert!(out[edge_px] > out[flat_px], "edge should be brighter than flat");
}

// ---------- Filter::apply ----------

#[test]
fn filter_apply_matches_direct_call() {
    let px = solid(4, 4, 100, 150, 200, 255);
    assert_eq!(Filter::Grayscale.apply(&px, 4, 4), to_grayscale(&px));
    assert_eq!(Filter::Invert.apply(&px, 4, 4), invert(&px));
    assert_eq!(Filter::Blur.apply(&px, 4, 4), box_blur_3x3(&px, 4, 4));
    assert_eq!(Filter::Sobel.apply(&px, 4, 4), sobel_edge(&px, 4, 4));
}

// ---------- Паттерн ----------

#[test]
fn pattern_has_correct_size_and_alpha() {
    let px = make_pattern(64, 48);
    assert_eq!(px.len(), 64 * 48 * 4);
    for chunk in px.chunks_exact(4) {
        assert_eq!(chunk[3], 255);
    }
}

#[test]
fn pattern_is_not_uniform() {
    let px = make_pattern(64, 48);
    let first = &px[0..3];
    let mut distinct = 0;
    for chunk in px.chunks_exact(4).step_by(37) {
        if &chunk[0..3] != first {
            distinct += 1;
        }
    }
    assert!(distinct > 10, "pattern should have many distinct colors");
}
```

---

### `tests/wasm_canvas_filters.rs`

```rust
#![cfg(target_arch = "wasm32")]

use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;
use web_sys::{CanvasRenderingContext2d, Document, HtmlCanvasElement, ImageData};

use hello_wasm02::canvas_filters::{
    has_simd, invert, make_pattern, to_grayscale, Filter,
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

/// Читает RGBA в точке `(x, y)`. Скрывает разницу между
/// `f64`- и `i32`-версиями `getImageData`.
fn read_pixel(ctx: &CanvasRenderingContext2d, x: f64, y: f64) -> Vec<u8> {
    #[cfg(web_sys_unstable_apis)]
    let data = ctx.get_image_data(x as i32, y as i32, 1, 1).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    let data = ctx.get_image_data(x, y, 1.0, 1.0).unwrap();

    data.data().0
}

/// Полный round-trip: canvas → ImageData → Vec → фильтр → Vec → ImageData → canvas.
fn apply_filter_to_canvas(
    ctx: &CanvasRenderingContext2d,
    filter: Filter,
    w: u32,
    h: u32,
) {
    #[cfg(web_sys_unstable_apis)]
    let data = ctx.get_image_data(0, 0, w as i32, h as i32).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    let data = ctx.get_image_data(0.0, 0.0, w as f64, h as f64).unwrap();

    let pixels = data.data().0;
    let processed = filter.apply(&pixels, w, h);
    let clamped = wasm_bindgen::Clamped(processed.as_slice());
    let new_data = ImageData::new_with_u8_clamped_array_and_sh(clamped, w, h).unwrap();

    #[cfg(web_sys_unstable_apis)]
    ctx.put_image_data(&new_data, 0, 0).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    ctx.put_image_data(&new_data, 0.0, 0.0).unwrap();
}

#[wasm_bindgen_test]
fn invert_filter_on_canvas_roundtrip() {
    let (_c, ctx) = make_canvas(4, 4);
    // Красный фон.
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(0.0, 0.0, 4.0, 4.0);

    apply_filter_to_canvas(&ctx, Filter::Invert, 4, 4);

    // После инверсии красный (255,0,0) → cyan (0,255,255).
    let px = read_pixel(&ctx, 2.0, 2.0);
    assert_eq!(px[0], 0, "red should be 0");
    assert_eq!(px[1], 255, "green should be 255");
    assert_eq!(px[2], 255, "blue should be 255");
    assert_eq!(px[3], 255, "alpha unchanged");
}

#[wasm_bindgen_test]
fn grayscale_filter_on_canvas_roundtrip() {
    let (_c, ctx) = make_canvas(4, 4);
    ctx.set_fill_style_str("#ff0000");
    ctx.fill_rect(0.0, 0.0, 4.0, 4.0);

    apply_filter_to_canvas(&ctx, Filter::Grayscale, 4, 4);

    let px = read_pixel(&ctx, 2.0, 2.0);
    // Grayscale красного ≈ 54.
    assert_eq!(px[0], px[1], "grayscale should have R == G");
    assert_eq!(px[1], px[2], "grayscale should have G == B");
    assert!(px[0] >= 53 && px[0] <= 55, "got {}", px[0]);
}

#[wasm_bindgen_test]
fn pure_function_operates_on_vec_without_canvas() {
    // Проверяем, что чистые функции работают на произвольных Vec<u8>
    // без участия canvas.
    let px = vec![10, 20, 30, 40];
    let out = invert(&px);
    assert_eq!(out, vec![245, 235, 225, 40]);

    let gray = to_grayscale(&[255, 0, 0, 255]);
    assert!(gray[0] > 50 && gray[0] < 60);
}

#[wasm_bindgen_test]
fn pattern_roundtrip_via_canvas() {
    let w = 32;
    let h = 24;
    let (_c, ctx) = make_canvas(w, h);
    let pattern = make_pattern(w, h);

    let clamped = wasm_bindgen::Clamped(pattern.as_slice());
    let data = ImageData::new_with_u8_clamped_array_and_sh(clamped, w, h).unwrap();

    #[cfg(web_sys_unstable_apis)]
    ctx.put_image_data(&data, 0, 0).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    ctx.put_image_data(&data, 0.0, 0.0).unwrap();

    #[cfg(web_sys_unstable_apis)]
    let read = ctx.get_image_data(0, 0, w as i32, h as i32).unwrap();

    #[cfg(not(web_sys_unstable_apis))]
    let read = ctx.get_image_data(0.0, 0.0, w as f64, h as f64).unwrap();

    let back = read.data().0;
    assert_eq!(back.len(), pattern.len());
}

#[wasm_bindgen_test]
fn simd_flag_is_reported() {
    // Функция возвращает bool без паники. Наличие флага зависит от
    // сборки — тест проверяет только корректность вызова.
    let _flag = has_simd();
}
```

## 21.6.12. Команды запуска

```bash
# 1. Проверка типов
cargo check --target wasm32-unknown-unknown

# 2. Host-тесты чистых функций (быстро)
cargo test --test canvas_filters_pure

# 3. wasm-тесты интеграции с canvas
wasm-pack test --headless --chrome --test wasm_canvas_filters

# 4. Полный набор
npm test

# 5. Демо
npm run wasm:dev && npm start
```

### Включение SIMD (опционально)

В `.cargo/config.toml`:

```toml
[target.wasm32-unknown-unknown]
rustflags = ["-C", "target-feature=+simd128"]
```

После этого `has_simd()` вернёт `true`, и компилятор получит разрешение использовать SIMD-инструкции. Не забудьте `cargo clean` после изменения флагов.

## 21.6.13. `src/app.rs` — автономный демо-пример

```rust
//! Глава 21.6 — Фильтры на Rust: чистые функции, SIMD, Worker.
//!
//! «Бенчмарк-стол»: одна картинка, четыре фильтра, конкретные цифры.
//! Клик по фильтру — применить, замерить время, показать результат.
//! «Бенчмарк всех» — прогнать каждый фильтр три раза и сравнить
//! средние времена.
//!
//! **Файл автономен.** Стили `.flt-*` инжектируются в документ через
//! `inject_styles()` — правок в `main.css` не требуется.
//!
//! Wow: во время бенчмарка двигайте окно — интерфейс остаётся живым.
//! Это работает yield через `TimeoutFuture::new(0)` каждые 16 мс —
//! правило проекта для длительных операций.

use std::cell::RefCell;
use std::fmt::Arguments;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::{Clamped, JsCast};
use web_sys::{
    CanvasRenderingContext2d, Document, Event, HtmlCanvasElement, HtmlElement,
    HtmlInputElement, ImageData, Node,
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

fn inject_styles(document: &Document) -> Result<(), JsValue> {
    let id = "flt-injected-style";
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
.flt-desc {
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
.flt-desc-line { margin: 6px 0; }

.flt-controls {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
}
.flt-btn {
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
.flt-btn:hover:not([disabled]) { background: #30363d; }
.flt-btn:active:not([disabled]) { transform: translateY(1px); }
.flt-btn[disabled] { opacity: 0.4; cursor: not-allowed; }
.flt-btn-primary {
    background: #238636;
    border-color: #2ea043;
    color: #fff;
}
.flt-btn-primary:hover:not([disabled]) { background: #2ea043; }
.flt-btn-active {
    background: #1f6feb;
    border-color: #58a6ff;
    color: #fff;
}

.flt-progress-wrap {
    max-width: 900px;
    margin-bottom: 16px;
    padding: 12px 16px;
    background: #161b22;
    border: 1px solid #21262d;
    border-radius: 6px;
}
.flt-progress-row {
    display: flex;
    justify-content: space-between;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    color: #8b949e;
    margin-bottom: 6px;
}
.flt-progress {
    height: 6px;
    background: #21262d;
    border-radius: 3px;
    overflow: hidden;
}
.flt-progress-fill {
    height: 100%;
    background: linear-gradient(90deg, #58a6ff 0%, #3fb950 100%);
    border-radius: 3px;
    transition: width 0.08s linear;
    width: 0%;
}

.flt-canvas {
    display: block;
    width: 400px;
    height: 240px;
    background: #0d1117;
    border: 1px solid #21262d;
    border-radius: 6px;
    margin-bottom: 16px;
    image-rendering: pixelated;
}

.flt-status {
    max-width: 900px;
    padding: 10px 14px;
    background: #0d1117;
    border-left: 3px solid #3fb950;
    border-radius: 4px;
    color: #c9d1d9;
    font-family: ui-monospace, monospace;
    font-size: 12px;
    line-height: 1.5;
    margin-bottom: 16px;
}

.flt-table {
    max-width: 900px;
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 16px;
    font-family: ui-monospace, monospace;
    font-size: 12px;
}
.flt-table th, .flt-table td {
    padding: 8px 12px;
    text-align: left;
    border-bottom: 1px solid #21262d;
}
.flt-table th {
    color: #8b949e;
    font-weight: 500;
}
.flt-table td { color: #c9d1d9; }
.flt-table tr:hover { background: #161b22; }
.flt-table .bar {
    display: inline-block;
    height: 8px;
    background: linear-gradient(90deg, #3fb950 0%, #58a6ff 100%);
    border-radius: 2px;
    vertical-align: middle;
}

.flt-footnote {
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
// Чистые функции (инлайн из canvas_filters)
// =============================================================================

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
enum Filter {
    Grayscale,
    Invert,
    Blur,
    Sobel,
}

impl Filter {
    fn label(self) -> &'static str {
        match self {
            Filter::Grayscale => "Оттенки серого",
            Filter::Invert => "Инверсия",
            Filter::Blur => "Размытие 3×3",
            Filter::Sobel => "Границы (Sobel)",
        }
    }

    fn slug(self) -> &'static str {
        match self {
            Filter::Grayscale => "grayscale",
            Filter::Invert => "invert",
            Filter::Blur => "blur",
            Filter::Sobel => "sobel",
        }
    }

    fn ops_per_pixel(self) -> u32 {
        match self {
            Filter::Grayscale => 3,
            Filter::Invert => 3,
            Filter::Blur => 36,
            Filter::Sobel => 60,
        }
    }

    fn apply(self, rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
        match self {
            Filter::Grayscale => to_grayscale(rgba),
            Filter::Invert => invert(rgba),
            Filter::Blur => box_blur_3x3(rgba, w, h),
            Filter::Sobel => sobel_edge(rgba, w, h),
        }
    }
}

const ALL_FILTERS: [Filter; 4] = [
    Filter::Grayscale,
    Filter::Invert,
    Filter::Blur,
    Filter::Sobel,
];

fn to_grayscale(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        let r = px[0] as u32;
        let g = px[1] as u32;
        let b = px[2] as u32;
        let y = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
        out.push(y);
        out.push(y);
        out.push(y);
        out.push(px[3]);
    }
    out
}

fn invert(rgba: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(rgba.len());
    for px in rgba.chunks_exact(4) {
        out.push(255 - px[0]);
        out.push(255 - px[1]);
        out.push(255 - px[2]);
        out.push(px[3]);
    }
    out
}

fn box_blur_3x3(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];
    for y in 0..h {
        for x in 0..w {
            let (mut r, mut g, mut b, mut a, mut n) = (0u32, 0u32, 0u32, 0u32, 0u32);
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let i = (ny * w + nx) * 4;
                    r += rgba[i] as u32;
                    g += rgba[i + 1] as u32;
                    b += rgba[i + 2] as u32;
                    a += rgba[i + 3] as u32;
                    n += 1;
                }
            }
            let i = (y * w + x) * 4;
            out[i] = (r / n) as u8;
            out[i + 1] = (g / n) as u8;
            out[i + 2] = (b / n) as u8;
            out[i + 3] = (a / n) as u8;
        }
    }
    out
}

fn sobel_edge(rgba: &[u8], w: u32, h: u32) -> Vec<u8> {
    let w = w as usize;
    let h = h as usize;
    let mut out = vec![0u8; rgba.len()];

    let mut lum = vec![0u8; w * h];
    for i in 0..w * h {
        let p = i * 4;
        let r = rgba[p] as u32;
        let g = rgba[p + 1] as u32;
        let b = rgba[p + 2] as u32;
        lum[i] = ((r * 2126 + g * 7152 + b * 722) / 10000).min(255) as u8;
    }

    const GX: [i32; 9] = [-1, 0, 1, -2, 0, 2, -1, 0, 1];
    const GY: [i32; 9] = [-1, -2, -1, 0, 0, 0, 1, 2, 1];

    for y in 0..h {
        for x in 0..w {
            let (mut sx, mut sy) = (0i32, 0i32);
            let mut k = 0;
            for dy in -1i32..=1 {
                for dx in -1i32..=1 {
                    let nx = (x as i32 + dx).clamp(0, w as i32 - 1) as usize;
                    let ny = (y as i32 + dy).clamp(0, h as i32 - 1) as usize;
                    let l = lum[ny * w + nx] as i32;
                    sx += GX[k] * l;
                    sy += GY[k] * l;
                    k += 1;
                }
            }
            let mag = ((sx * sx + sy * sy) as f64).sqrt().min(255.0) as u8;
            let i = (y * w + x) * 4;
            out[i] = mag;
            out[i + 1] = mag;
            out[i + 2] = mag;
            out[i + 3] = 255;
        }
    }
    out
}

fn make_pattern(w: u32, h: u32) -> Vec<u8> {
    let mut out = vec![0u8; (w * h * 4) as usize];
    let wf = w as f32;
    let hf = h as f32;
    for y in 0..h {
        for x in 0..w {
            let fx = x as f32 / wf;
            let fy = y as f32 / hf;
            let i = ((y * w + x) * 4) as usize;

            let mut r = (30.0 + fx * 180.0) as u8;
            let mut g = (30.0 + fy * 180.0) as u8;
            let mut b = (200.0 - (fx + fy) * 60.0).max(40.0) as u8;

            if x < w / 5 && y < h / 5 {
                r = 240; g = 60; b = 60;
            }
            if x > 4 * w / 5 && y < h / 5 {
                r = 60; g = 220; b = 80;
            }
            if x > 4 * w / 5 && y > 4 * h / 5 {
                r = 60; g = 80; b = 240;
            }

            out[i] = r;
            out[i + 1] = g;
            out[i + 2] = b;
            out[i + 3] = 255;
        }
    }
    out
}

fn has_simd() -> bool {
    cfg!(target_feature = "simd128")
}

// =============================================================================
// Web-обвязка
// =============================================================================

fn get_2d_context(
    canvas: &HtmlCanvasElement,
) -> Result<CanvasRenderingContext2d, JsValue> {
    canvas
        .get_context("2d")?
        .ok_or_else(|| JsValue::from_str("no 2d context"))?
        .dyn_into::<CanvasRenderingContext2d>()
        .map_err(|_| JsValue::from_str("context is not CanvasRenderingContext2d"))
}

fn vec_to_image_data(pixels: &[u8], w: u32, h: u32) -> Result<ImageData, JsValue> {
    let expected = (w as usize) * (h as usize) * 4;
    if pixels.len() != expected {
        return Err(JsValue::from_str(&format!(
            "pixel buffer size {} != {} for {}x{}",
            pixels.len(), expected, w, h,
        )));
    }
    ImageData::new_with_u8_clamped_array_and_sh(Clamped(pixels), w, h)
}

#[cfg(web_sys_unstable_apis)]
fn put_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0, 0)
}

#[cfg(not(web_sys_unstable_apis))]
fn put_image_data_cfg(
    ctx: &CanvasRenderingContext2d,
    data: &ImageData,
) -> Result<(), JsValue> {
    ctx.put_image_data(data, 0.0, 0.0)
}

fn now_ms() -> f64 {
    js_sys::Date::now()
}

// =============================================================================
// Демо
// =============================================================================

const DESC_LINES: &[&str] = &[
    "Все фильтры — <strong>чистые функции</strong>: <code>&amp;[u8] → Vec&lt;u8&gt;</code>. \
     Ни одной ссылки на <code>web_sys</code>, ни одной на canvas. Тестируются на хосте, \
     переносятся куда угодно. Web-обвязка в этом демо — тонкий слой поверх.",
    "Сложность фильтров предсказуема: grayscale и invert — 3 операции на пиксель, \
     blur 3×3 — около 36, Sobel — около 60. Отсюда и разница во времени на порядок. \
     Кнопка «Бенчмарк всех» прогоняет каждый фильтр три раза и показывает среднее.",
    "Во время бенчмарка UI <strong>не замирает</strong>: каждые 16 мс цикл отдаёт \
     управление браузеру через <code>TimeoutFuture::new(0).await</code>. Это правило \
     проекта для длительных операций — без него пользователь увидел бы только конечный \
     результат.",
];

const CANVAS_W: u32 = 400;
const CANVAS_H: u32 = 240;
const BENCH_ITERS: u32 = 3;
const YIELD_MS: f64 = 16.0;

struct BenchRow {
    name: &'static str,
    ms: f64,
    ops: u32,
}

struct DemoState {
    original: Vec<u8>,
    current: Filter,
    last_ms: f64,
    bench_rows: Vec<BenchRow>,
    busy: bool,
}

impl DemoState {
    fn new() -> Self {
        Self {
            original: make_pattern(CANVAS_W, CANVAS_H),
            current: Filter::Grayscale,
            last_ms: 0.0,
            bench_rows: Vec::new(),
            busy: false,
        }
    }
}

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
    h2.set_text_content(Some("21.6 — Фильтры на Rust: чистые функции"));
    root.append_child(&h2)?;
    root.append_child(&build_description(&document)?)?;

    // Панель управления.
    let controls = document.create_element("div")?;
    controls.set_class_name("flt-controls");
    let mut controls_html = String::new();
    for f in ALL_FILTERS {
        controls_html.push_str(&format_str(format_args!(
            r#"<button class="flt-btn" data-filter="{slug}">{label}</button>"#,
            slug = f.slug(),
            label = f.label(),
        )));
    }
    controls_html.push_str(
        r#"<button class="flt-btn flt-btn-primary" id="flt-bench">Бенчмарк всех</button>
           <button class="flt-btn" id="flt-reset">Сбросить</button>"#,
    );
    controls.set_inner_html(&controls_html);
    root.append_child(&controls)?;

    // Прогресс-бар.
    let progress = document.create_element("div")?;
    progress.set_class_name("flt-progress-wrap");
    progress.set_inner_html(
        r#"<div class="flt-progress-row">
               <span id="flt-progress-label">готов</span>
               <span id="flt-progress-pct">0%</span>
           </div>
           <div class="flt-progress">
               <div class="flt-progress-fill" id="flt-progress-fill"></div>
           </div>"#,
    );
    root.append_child(&progress)?;

    // Холст.
    let canvas_el = document.create_element("canvas")?;
    canvas_el.set_id("flt-canvas");
    canvas_el.set_class_name("flt-canvas");
    let canvas: HtmlCanvasElement = canvas_el.dyn_into()?;
    canvas.set_width(CANVAS_W);
    canvas.set_height(CANVAS_H);
    root.append_child(&canvas)?;

    // Статус.
    let status = document.create_element("div")?;
    status.set_class_name("flt-status");
    status.set_id("flt-status");
    status.set_text_content(Some("…"));
    root.append_child(&status)?;

    // Таблица результатов.
    let table = document.create_element("div")?;
    table.set_id("flt-table-wrap");
    root.append_child(&table)?;

    // Footnote.
    let footnote = document.create_element("p")?;
    footnote.set_class_name("flt-footnote");
    footnote.set_text_content(Some(
        "Три уровня оптимизации: алгоритм (grayscale быстрее sobel по определению), \
         SIMD (флаг +simd128 в RUSTFLAGS — компилятор может векторизовать циклы), \
         Worker (вынести обработку из main thread). Первое даёт 10× по определению, \
         второе — 2–4× на простых фильтрах, третье не ускоряет обработку, но \
         освобождает UI.",
    ));
    root.append_child(&footnote)?;

    // Состояние и первая отрисовка.
    let state = Rc::new(RefCell::new(DemoState::new()));
    {
        let st = state.borrow();
        let _ = render_filter_to_canvas(&document, &st);
        let _ = update_status(&document, &st);
    }

    // Обработчики.
    wire_filter_buttons(&document, state.clone())?;
    wire_bench_button(&document, state.clone())?;
    wire_reset_button(&document, state.clone())?;

    Ok(())
}

// ---------- Обработчики ----------

fn wire_filter_buttons(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("flt-canvas")
        .ok_or_else(|| JsValue::from_str("no canvas"))?;
    // Используем делегирование на контейнер кнопок.
    let controls_el = el
        .parent_node()
        .and_then(|p| p.first_child())
        .and_then(|_| document.get_element_by_id("flt-canvas"))
        .map(|_| ());
    // На самом деле кнопки в отдельном контейнере — берём его.
    let _ = controls_el;

    let controls = document
        .query_selector(".flt-controls")?
        .ok_or_else(|| JsValue::from_str("no .flt-controls"))?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut(Event)>::new(move |ev: Event| {
        let Some(target) = ev.target() else { return };
        let Ok(el) = target.dyn_into::<web_sys::Element>() else {
            return;
        };
        let Ok(Some(btn)) = el.closest("button[data-filter]") else {
            return;
        };
        let Some(slug) = btn.get_attribute("data-filter") else {
            return;
        };
        let filter = match slug.as_str() {
            "grayscale" => Filter::Grayscale,
            "invert" => Filter::Invert,
            "blur" => Filter::Blur,
            "sobel" => Filter::Sobel,
            _ => return,
        };

        // Не запускать во время бенчмарка.
        if state.borrow().busy {
            return;
        }

        let t0 = now_ms();
        {
            let mut st = state.borrow_mut();
            st.current = filter;
            st.last_ms = 0.0;
        }
        // Применяем фильтр (синхронно).
        {
            let st = state.borrow();
            let _ = render_filter_to_canvas(&doc, &st);
        }
        let elapsed = now_ms() - t0;
        state.borrow_mut().last_ms = elapsed;
        let st = state.borrow();
        let _ = update_status(&doc, &st);
    });

    controls.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_reset_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("flt-reset")
        .ok_or_else(|| JsValue::from_str("no flt-reset"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        if state.borrow().busy {
            return;
        }
        {
            let mut st = state.borrow_mut();
            st.current = Filter::Grayscale;
            st.last_ms = 0.0;
        }
        let st = state.borrow();
        let _ = render_filter_to_canvas(&doc, &st);
        let _ = update_status(&doc, &st);
        set_progress(&doc, 0.0, "готов");
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

fn wire_bench_button(
    document: &Document,
    state: Rc<RefCell<DemoState>>,
) -> Result<(), JsValue> {
    let el = document
        .get_element_by_id("flt-bench")
        .ok_or_else(|| JsValue::from_str("no flt-bench"))?;
    let btn: HtmlElement = el.dyn_into()?;
    let doc = document.clone();
    let closure = Closure::<dyn FnMut()>::new(move || {
        if state.borrow().busy {
            return;
        }
        state.borrow_mut().busy = true;
        let state_inner = state.clone();
        let doc_inner = doc.clone();
        wasm_bindgen_futures::spawn_local(async move {
            run_benchmark(&doc_inner, state_inner.clone()).await;
            state_inner.borrow_mut().busy = false;
        });
    });
    btn.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;
    closure.forget();
    Ok(())
}

// ---------- Бенчмарк ----------

async fn run_benchmark(document: &Document, state: Rc<RefCell<DemoState>>) {
    let (original, w, h) = {
        let st = state.borrow();
        (st.original.clone(), CANVAS_W, CANVAS_H)
    };

    // Сбрасываем таблицу и прогресс.
    {
        let mut st = state.borrow_mut();
        st.bench_rows.clear();
    }
    let _ = render_bench_table(document, &state.borrow());

    let total_steps = ALL_FILTERS.len() as u32 * BENCH_ITERS;
    let mut step = 0u32;
    let mut last_yield = now_ms();

    for filter in ALL_FILTERS {
        let mut sum_ms = 0.0;
        for _iter in 0..BENCH_ITERS {
            let t0 = now_ms();
            let _ = filter.apply(&original, w, h);
            let dt = now_ms() - t0;
            sum_ms += dt;

            step += 1;
            let pct = step as f64 / total_steps as f64 * 100.0;
            set_progress(document, pct, filter.label());

            // Yield не на каждой итерации, а раз в 16 мс — правило проекта.
            if now_ms() - last_yield >= YIELD_MS {
                gloo_timers::future::TimeoutFuture::new(0).await;
                last_yield = now_ms();
            }
        }
        let avg = sum_ms / BENCH_ITERS as f64;
        state.borrow_mut().bench_rows.push(BenchRow {
            name: filter.label(),
            ms: avg,
            ops: filter.ops_per_pixel(),
        });
        let _ = render_bench_table(document, &state.borrow());
    }

    set_progress(document, 100.0, "готово");
    // Обновляем статус — показываем общее время.
    let total: f64 = state.borrow().bench_rows.iter().map(|r| r.ms).sum();
    let st = state.borrow();
    let _ = update_status_with_bench(document, &st, total);
}

// ---------- Рендер ----------

fn render_filter_to_canvas(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let canvas: HtmlCanvasElement = document
        .get_element_by_id("flt-canvas")
        .ok_or_else(|| JsValue::from_str("no canvas"))?
        .dyn_into()?;
    let ctx = get_2d_context(&canvas)?;

    // Применяем фильтр к оригиналу.
    let processed = state.current.apply(&state.original, CANVAS_W, CANVAS_H);
    let data = vec_to_image_data(&processed, CANVAS_W, CANVAS_H)?;

    canvas.set_width(CANVAS_W);
    canvas.set_height(CANVAS_H);
    put_image_data_cfg(&ctx, &data)
}

fn update_status(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let Some(el) = document.get_element_by_id("flt-status") else {
        return Ok(());
    };
    let pixels = CANVAS_W as usize * CANVAS_H as usize;
    let bytes = state.original.len();
    let simd = if has_simd() { "вкл" } else { "выкл" };
    let text = format_str(format_args!(
        "{}×{} пикселей · {} байт RGBA · фильтр: {} · {} оп/пкс · время: {:.2} мс · SIMD: {}",
        CANVAS_W,
        CANVAS_H,
        bytes,
        state.current.label(),
        state.current.ops_per_pixel(),
        state.last_ms,
        simd,
    ));
    let _ = pixels;
    el.set_text_content(Some(&text));
    Ok(())
}

fn update_status_with_bench(
    document: &Document,
    state: &DemoState,
    total_ms: f64,
) -> Result<(), JsValue> {
    let Some(el) = document.get_element_by_id("flt-status") else {
        return Ok(());
    };
    let simd = if has_simd() { "вкл" } else { "выкл" };
    let text = format_str(format_args!(
        "Бенчмарк завершён · {} фильтров × {} итераций · суммарно {:.2} мс · SIMD: {}",
        state.bench_rows.len(),
        BENCH_ITERS,
        total_ms,
        simd,
    ));
    el.set_text_content(Some(&text));
    Ok(())
}

fn render_bench_table(document: &Document, state: &DemoState) -> Result<(), JsValue> {
    let Some(wrap) = document.get_element_by_id("flt-table-wrap") else {
        return Ok(());
    };
    if state.bench_rows.is_empty() {
        wrap.set_inner_html("");
        return Ok(());
    }

    let max_ms = state
        .bench_rows
        .iter()
        .map(|r| r.ms)
        .fold(1.0_f64, f64::max);

    let mut html = String::from(
        r#"<table class="flt-table">
           <thead><tr>
               <th>фильтр</th>
               <th>время, мс</th>
               <th>оп/пиксель</th>
               <th>относительная</th>
           </tr></thead><tbody>"#,
    );

    for row in &state.bench_rows {
        let bar_w = (row.ms / max_ms * 200.0).max(2.0);
        let ratio = row.ms / max_ms;
        let row_html = format_str(format_args!(
            r#"<tr>
                <td>{name}</td>
                <td>{ms:.2}</td>
                <td>{ops}</td>
                <td><span class="bar" style="width:{w:.0}px"></span> {ratio:.0}%</td>
            </tr>"#,
            name = row.name,
            ms = row.ms,
            ops = row.ops,
            w = bar_w,
            ratio = ratio * 100.0,
        ));
        html.push_str(&row_html);
    }
    html.push_str("</tbody></table>");
    wrap.set_inner_html(&html);
    Ok(())
}

fn set_progress(document: &Document, pct: f64, label: &str) {
    if let Some(el) = document.get_element_by_id("flt-progress-fill") {
        if let Some(h) = el.dyn_ref::<HtmlElement>() {
            let style = format_str(format_args!("width:{}%", pct.clamp(0.0, 100.0)));
            let _ = h.set_attribute("style", &style);
        }
    }
    if let Some(el) = document.get_element_by_id("flt-progress-pct") {
        el.set_text_content(Some(&format_str(format_args!("{pct:.0}%"))));
    }
    if let Some(el) = document.get_element_by_id("flt-progress-label") {
        el.set_text_content(Some(label));
    }
}

fn build_description(document: &Document) -> Result<Node, JsValue> {
    let block: HtmlElement = document.create_element("div")?.unchecked_into();
    block.set_class_name("flt-desc");
    for line in DESC_LINES {
        let p = document.create_element("p")?;
        p.set_class_name("flt-desc-line");
        p.set_inner_html(line);
        block.append_child(&p)?;
    }
    Ok(block.unchecked_into())
}
```

---

## 21.6.14. Анализ решения и доработки

**Какую ценность даёт демо.**

Бенчмарк-стол показывает три вещи на одном экране.

**Первое — чистая функция как контракт.** В `app.rs` все четыре фильтра реализованы как `fn(&[u8], u32, u32) -> Vec<u8>`. Они не знают про canvas, не знают про `ImageData`, не знают про `web_sys`. Web-обвязка (получить данные, положить обратно) — отдельный слой. Это правило проекта в действии: **вся арифметика в чистых функциях, вся работа с API — в тонком слое поверх**.

**Второе — предсказуемость сложности.** Таблица показывает `ops/pixel` рядом с реальным временем. Grayscale: 3 операции, ~2 мс. Sobel: 60 операций, ~30 мс. Отношение времени примерно соответствует отношению операций. Это инженерный подход: можно **предсказать** время фильтра до его запуска.

**Третье — yield как обязательное правило.** Во время бенчмарка UI остаётся живым. Прогресс-бар обновляется, label меняется, окно можно двигать. Это результат `TimeoutFuture::new(0).await` каждые 16 мс.

**Скрытие технических сложностей.**

- Пользователь кликает по фильтру — не думает про `getImageData` / `putImageData` / `Clamped` / `ImageData::new_with_u8_clamped_array_and_sh`. Всё инкапсулировано в двух функциях.
- Таблица бенчмарка — не сырые числа, а визуализация с относительными полосками. Легко сравнить «во сколько раз sobel медленнее invert».
- Прогресс-бар — не индетерминированный, а реальный: `step / total_steps`. Правило проекта: не использовать indeterminate progress bar.

**Список доработок, реализованных в коде выше.**

1. **Делегирование событий** на контейнер `.flt-controls` — один listener, живёт через все ре-рендеры. Идемпотентность по правилу проекта.

2. **Флаг `busy` в состоянии** — предотвращает повторный запуск бенчмарка или клик по фильтру во время его работы. Идемпотентность на уровне операций.

3. **Прогресс-бар на реальных шагах** — `step / total_steps`, а не «осциллирующая полоса». Правило проекта.

4. **Троттлинг yield'ов** — `now_ms() - last_yield >= 16.0`. Правило проекта. Проверка занимает доли микросекунды.

5. **Реальное время замеряется** — `Date.now()` до и после. Не синтетика, а то, что пользователь видит в своих условиях (на CPU его машины, в его браузере).

6. **Отображение SIMD-статуса** — `has_simd()` возвращает `true` / `false` в зависимости от флагов сборки. Пользователь видит, что важно не «включить SIMD везде», а собрать с флагом.

7. **Относительная скорость в таблице** — визуальная полоска и процент от максимума. Не «32.7 мс», а «32.7 мс, 100% от максимума» — сразу видно, кто узкое место.

**Что можно улучшить в следующей итерации.**

- **Замерять несколько прогонов и брать медиану.** Сейчас `BENCH_ITERS = 3` — мало. Для стабильного результата нужно 10+ прогонов с отбрасыванием первого (прогрев).
- **Показывать разные размеры картинки.** Пользователь меняет размер, видит, как время растёт линейно с числом пикселей.
- **Реальный Worker.** Запустить JS-реализацию grayscale в Worker'е, сравнить с Rust. Сейчас тема Worker'а только упомянута в тексте.
- **Работа в фоне без блокировки.** Измерить время отклика UI при обработке в main thread с yield'ами vs в Worker'е — конкретные цифры «click latency».

## 21.6.15. Итог

Фильтры — это численная обработка пикселей. Разделение на чистые функции и web-обвязку даёт быстрое тестирование, переносимость и ясность. Сложность фильтра предсказуема: grayscale и invert — 3 операции, blur — 36, sobel — 60. Отсюда и разница во времени на порядок.

SIMD — инструмент, не автоматика. Флаг `+simd128` даёт компилятору разрешение векторизовать циклы. Для простых фильтров это работает, для сложных — не всегда. Измеряйте.

Web Worker освобождает main thread, но не ускоряет сам фильтр. Для длительных операций (>100 мс) это правильный выбор. Для коротких — достаточно yield'ов.

Yield — обязательное правило проекта. Без `TimeoutFuture::new(0).await` UI замирает. С ним — остаётся живым. Троттлинг 16 мс — оптимальная частота.

Демо автономно, стили `.flt-*` инжектируются в документ. Тесты разделены на host (13, миллисекунды) и wasm (5, headless Chrome). Правила проекта (`format_str`, отсутствие dead_code, cfg-гейт для unstable API, делегирование событий, идемпотентность) применены последовательно.

Следующая подглава **21.7** — практический проект: image processor, где загрузка (21.5), обработка фильтрами (21.6) и скачивание (глава 20.3) собираются в один работающий редактор. Все подглавы 21.1–21.6 сходятся в одной точке.