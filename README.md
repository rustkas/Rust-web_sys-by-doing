# Rust web_sys by doing
### Практическое руководство по работе с библиотекой `web_sys` в Rust

---

## О книге

Эта книга — **практическое руководство** по созданию полноценных веб-приложений на Rust с использованием библиотеки **`web_sys`** — основного инструмента для доступа к браузерному API из WASM-кода.

Вместо сухого пересказа документации мы строим **настоящее приложение** и последовательно добавляем в него новые возможности: работу с DOM, обработку событий, сетевые запросы, файловую систему, графику, аудио и видео, параллельные вычисления и многое другое. Каждая глава — это законченный, работающий пример, который можно сразу запустить и применить в своём проекте.

**Принцип книги — «learning by doing»**: минимум теории, максимум кода, который решает реальные задачи.

### Кому подойдёт книга

- Разработчикам, уже знакомым с основами Rust и желающим выйти в веб;
- Frontend-инженерам, которые хотят перенести тяжёлые части приложения на Rust/WASM;
- Тем, кто интересуется WebAssembly и хочет понять, как выглядит «настоящая» веб-разработка на Rust;
- Всем, кто предпочитает учиться на примерах, а не на абстрактных описаниях.

### Что рассматривается в книге

- Настройка окружения: `wasm-pack`, `wasm-bindgen`, `trunk`, интеграция с `web_sys` и `js-sys`;
- DOM, типизированные элементы, динамическое создание и модификация узлов;
- События: мышь, клавиатура, формы, делегирование, `JsValue` как граница Rust ↔ JavaScript;
- Сетевые запросы через `fetch`, `Request`, `Response`, `JsFuture`, retry, timeout, CORS;
- Streams: `ReadableStream`, `WritableStream`, `TransformStream`, backpressure;
- Файлы: `File API`, `File System Access API`, OPFS, `Blob`, Object URL, скачивание;
- Графика: `Canvas`, `ImageData`, `OffscreenCanvas`, фильтры на Rust;
- Медиа: `WebCodecs`, `AudioWorklet`, `MediaStream`, MP4/WebM, demuxing/muxing;
- Хранение данных: `localStorage`, `IndexedDB`, `Cache API`;
- Service Worker и офлайн-архитектура;
- Параллелизм: `Web Workers`, `SharedArrayBuffer`, `Typed Arrays`, `Scheduling`;
- API браузера: `Clipboard`, `Notifications`, `Geolocation`, `MediaDevices`;
- Сеть в реальном времени: `WebSocket`, `WebTransport`, `WebRTC`;
- GPU: `WebGL`, `WebGPU`, compute shaders;
- Безопасность и криптография: `WebAuthn`, Web Crypto;
- Обработка ошибок, паника, восстановление после сбоев.

---

## Сопутствующий проект

Все примеры из книги собираются в **единое работающее приложение**, которое развивается от главы к главе:

> 🔗 **Репозиторий проекта: [https://github.com/rustkas/hello-wasm02](https://github.com/rustkas/hello-wasm02)**

Рекомендуется параллельно читать главу и открывать соответствующий коммит в проекте — так вы увидите не только отдельный пример, но и то, как он встраивается в реальную кодовую базу.

---

## Структура репозитория

- Все главы книги находятся в папке [`chapters`](https://github.com/rustkas/Rust-web_sys-by-doing/tree/main/chapters);
- Формат имени файла: `chapter-<номер>.md` — глава, `chapter-<номер>.<подномер>.md` — подглава;
- Часть подглав использует отрицательные номера (`chapter-22.-01.md`) — это сохраняет порядок вывода в оглавлении;
- Оглавление в этом README генерируется автоматически из заголовков `# ...` в начале каждого файла.

---

## Содержание

- [Глава 1. Rust получает доступ к Web Platform](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-01.md)
- [Глава 2. Как устроен `web_sys`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-02.md)
- [Глава 3. Первый Rust + `web_sys` проект](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-03.md)
- [Глава 4. Сборка, оптимизация и деплой](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-04.md)
- [chapter-05](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-05.md)
- [Глава 6. Тестирование Rust/WASM](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-06.md)
- [Глава 7. `Window`, `Document` и DOM](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-07.md)
- [Глава 8. Типизированный DOM](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-08.md)
- [Глава 9. События](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-09.md)
- [Глава 10. `JsValue` как универсальный тип границы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-10.md)
- [Глава 11. `js_sys` и JavaScript objects](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-11.md)
- [Глава 12. JavaScript callbacks из Rust](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-12.md)
- [Глава 13. Async/await на границе Rust ↔ JavaScript](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-13.md)
- [Глава 14. Browser navigation и URL](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-14.md)
- [Глава 15. Storage](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.md)
  - [15.1. Знакомство со Storage](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.1.md)
  - [15.2. Полный API Storage](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.2.md)
  - [15.3. `localStorage` vs `sessionStorage`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.3.md)
  - [15.4. Доступность Storage: private mode, исключения](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.4.md)
  - [15.5. Сериализация: строки, числа, булевы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.5.md)
  - [15.6. JSON: структуры в Storage](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.6.md)
  - [15.7. `StorageEvent` — синхронизация вкладок](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.7.md)
  - [15.8. Ограничения Storage и синхронный API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.8.md)
  - [15.9. Практический проект: панель управления Storage](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-15.9.md)
- [Глава 16. Fetch](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.md)
  - [16.1. Первый GET: Request, Response, JsFuture](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.1.md)
  - [16.2. HTTP-методы и тело запроса](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.2.md)
  - [16.3. Заголовки: чтение, запись, case-insensitivity](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.3.md)
  - [Глава 16.4. Аутентификация и авторизация](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.4.md)
  - [Глава 16.5. Ошибки: различение и классификация](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.5.md)
  - [Глава 16.6. Retry и восстановление](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.6.md)
  - [Глава 16.7. Timeout и отмена](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.7.md)
  - [Глава 16.8. Параллельные запросы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.8.md)
  - [Глава 16.9. Типы ответов и binary data](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.9.md)
  - [Глава 16.11. CORS и cross-origin](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.10.md)
  - [Глава 16.11. CORS и cross-origin](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.11.md)
  - [Глава 16.12. API client: паттерны и архитектура](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.12.md)
  - [Глава 16.13. Fetch в Worker и Service Worker](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.13.md)
  - [Глава 16.14. Практический проект: полноценный Rust API client](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-16.14.md)
- [Глава 17. Streams: ручная реализация и `wasm-streams`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.md)
  - [17.1. `ReadableStream`: источник данных](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.1.md)
  - [17.2. `WritableStream`: приёмник данных](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.2.md)
  - [17.3. `TransformStream`: конвейер преобразования](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.3.md)
  - [17.4. Backpressure и очереди](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.4.md)
  - [17.5. Rust ↔ JS: конвертация потоков](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.5.md)
  - [17.6. Byte streams и BYOB](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.6.md)
  - [17.6. Byte streams и BYOB](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.7.md)
  - [17.8. Streams в Service Worker](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-17.8.md)
- [Глава 18. File API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.md)
  - [Глава 18.1. Базовые типы: `File`, `Blob`, `FileList`, метаданные](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.1.md)
  - [Глава 18.2. Чтение текста: `FileReader`, `FileReaderSync`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.2.md)
  - [Глава 18.3. Бинарные данные: `ArrayBuffer`, `Uint8Array`, `Vec<u8>`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.3.md)
  - [Глава 18.4. Object URL: превью и скачивание](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.4.md)
  - [Глава 18.5. Data URL, base64 и crate `base64`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.5.md)
  - [Глава 18.6. Drag & Drop: файлы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.6.md)
  - [Глава 18.7. Drag & Drop папок: `webkitGetAsEntry`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.7.md)
  - [Глава 18.8. Валидация файлов: MIME, magic-байты, размер](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.8.md)
  - [Глава 18.9. `Blob.slice`: диапазоны и большие файлы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.9.md)
  - [Глава 18.10. `Blob.stream()` и потоковое чтение](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.10.md)
  - [Глава 18.11. Web Workers для файловой обработки](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.11.md)
  - [Глава 18.12. Хеширование файлов: `sha2`, `SubtleCrypto` и адаптивный пул воркеров](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.12.md)
  - [Глава 18.13. Загрузка на сервер: `FormData` + `fetch`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.13.md)
  - [Глава 18.14. Прогресс, отмена и интеллектуальная отправка](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.14.md)
  - [Глава 18.15. File System Access API: открытие, сохранение, папки](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.15.md)
  - [Глава 18.16. Web Share API: поделиться файлом](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.16.md)
  - [Глава 18.17. Хранение: Cache API, IndexedDB, OPFS](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.17.md)
  - [Глава 18.18. Практический проект: конвертер файлов](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-18.18.md)
- [Глава 19. File System Access API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.md)
  - [Глава 19.1. `FileSystemHandle` и `FileSystemFileHandle` — базовые типы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.01.md)
  - [Глава 19.2. Три picker'а: open / save / directory](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.02.md)
  - [Глава 19.3. Permissions и user gesture](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.03.md)
  - [Глава 19.4. Чтение файлов: `text`, `arrayBuffer`, `slice`, `stream`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.04.md)
  - [Глава 19.5. Запись файлов: `FileSystemWritableFileStream`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.05.md)
  - [Глава 19.6. Большие файлы: `adaptive chunk`, backpressure, копирование](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.06.md)
  - [19.7. `FileSystemDirectoryHandle`: обход, `resolve`, идемпотентность](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.07.md)
  - [19.8. OPFS: приватная файловая система браузера](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.08.md)
  - [19.9. Квоты и `StorageManager`](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.09.md)
  - [19.10. `FileSystemObserver`: реакция на изменения](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.10.md)
  - [19.11. Интеграция с WASM-библиотеками](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.11.md)
  - [19.12. Практический проект: локальный CSV-редактор](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-19.12.md)
- [Глава 20. Blob, Object URL и скачивание](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-20.md)
  - [20.1. Blob как контейнер бинарных данных](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-20.01.md)
  - [Глава 20.2. Object URL: жизненный цикл и утечки](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-20.02.md)
  - [Глава 20.4. Практический проект: Blob → PNG + JSON](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-20.04.md)
- [Глава 21. Canvas и изображения](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.md)
  - [Глава 21.1. HtmlCanvasElement: размеры и HiDPI](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.01.md)
  - [Глава 21.2. Drawing API: фигуры, пути, текст](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.02.md)
  - [Глава 21.3. Состояние, трансформации, композиция](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.03.md)
  - [Глава 21.4. ImageData: пиксельный буфер и его границы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.04.md)
  - [Глава 21.5. Загрузка изображений: `<img>`, `ImageBitmap`, EXIF](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.05.md)
  - [Глава 21.6. Фильтры на Rust: чистые функции, SIMD, Worker](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.06.md)
  - [Глава 21.7. Image Processor: полный workflow](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-21.07.md)
- [Глава 22. OffscreenCanvas и Workers](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.md)
  - [Глава 22.6. Практический проект: Mandelbrot explorer](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.-06.md)
  - [Глава 22.5. Worker pools и SharedArrayBuffer](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.-05.md)
  - [Глава 22.4. WASM в Worker: Rust в параллель](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.-04.md)
  - [Глава 22.3. Transferable objects: zero-copy между потоками](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.-03.md)
  - [Глава 22.2. Dedicated Worker: базовые примитивы](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.-02.md)
  - [Глава 22.1. OffscreenCanvas: canvas вне DOM](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-22.-01.md)
- [Глава 23. WebCodecs](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.md)
  - [Глава 23.1. VideoFrame и EncodedVideoChunk: два представления кадра](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.01.md)
  - [Часть II. Кодеки](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.02.md)
  - [Глава 23.3. VideoEncoder: распакованное → сжатое](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.03.md)
  - [Глава 23.4. Codec negotiation и тюнинг VideoEncoder (side-by-side)](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.04.md)
  - [Глава 23.5. AudioCodecs: AudioDecoder, AudioEncoder, AudioData](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.05.md)
  - [Глава 23.6. ImageDecoder: статика и animated GIF](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.06.md)
  - [Глава 23.7. WebCodecs + WASM/GPU: обработка пикселей](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.07.md)
  - [Глава 23.8. MediaStream: real-time camera pipeline](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.08.md)
  - [Глава 23.9. Структура контейнеров: MP4 и WebM](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.09.md)
  - [Глава 23.10. Demuxing: MP4 → EncodedVideoChunk](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.10.md)
  - [Глава 23.11. Muxing: EncodedVideoChunk → MP4](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.11.md)
  - [Глава 23.12. Практический проект: MP4 → обработка → MP4](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.12.md)
  - [Глава 23.13. AudioWorklet: real-time audio pipeline](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.13.md)
  - [Глава 23.14. Temporal scalability: L1T2, L1T3](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-23.14.md)
- [Глава 24. IndexedDB](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.md)
  - [Глава 24.1. Почему `localStorage` недостаточно](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.01.md)
  - [Глава 24.2. `IdbFactory` — точка входа](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.02.md)
  - [Глава 24.3. IdbDatabase — соединение с базой](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.03.md)
  - [Глава 24.4. `IdbDatabase` — схема как контракт](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.04.md)
  - [Глава 24.5. `IdbObjectStore` — записи, курсоры, диапазоны, связи](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.05.md)
  - [Глава 24.6. `add` vs `put` — семантика вставки](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-24.06.md)
- [Глава 25. Cache API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-25.md)
- [Глава 26. Service Worker](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-26.md)
- [Глава 27. Offline Application Architecture](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-27.md)
- [Глава 28. Worker и параллельное выполнение](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-28.md)
- [Глава 29. SharedArrayBuffer и Typed Arrays](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-29.md)
- [Глава 30. Scheduling](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-30.md)
- [Глава 31. Performance API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-31.md)
- [Глава 32. Clipboard API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-32.md)
  - [Глава 32.01. `ClipboardEvent` — paste без разрешений](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-32.01.md)
  - [Глава 32.2. Асинхронный `ClipboardItem` — что реально работает](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-32.02.md)
- [Глава 33. Notifications API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-33.md)
- [Глава 34. Geolocation](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-34.md)
- [Глава 35. MediaDevices: камера и микрофон](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-35.md)
- [Глава 36. WebSocket](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-36.md)
- [Глава 37. WebTransport](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-37.md)
- [Глава 45. WebRTC](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-38.md)
- [Глава 39. Real-Time Browser Architecture](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-39.md)
- [Глава 40. Web Crypto API](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-40.md)
- [Глава 41 Web Authentication](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-41.md)
  - [Глава 41.1 Web Authentication. Setup web-server](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-41.01.md)
  - [Глава 41.2 Production-grade WebAuthn-сервер](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-41.02.md)
- [Глава 42. WebGL](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-42.md)
- [Глава 43. WebGL 2](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-43.md)
- [Глава 44. WebGPU: compute shaders в браузере](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-44.md)
- [Глава 45. Web Bluetooth — переработанная версия](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-45.md)
- [Глава 46. Где должен находиться Rust](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-46.md)
- [Глава 47. Обработка ошибок](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-47.md)
- [Глава 48. Panic и восстановление после сбоев](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-48.md)
- [Глава 49. Rust/WASM File Converter](https://github.com/rustkas/Rust-web_sys-by-doing/blob/main/chapters/chapter-49.md)

---

## Как читать книгу

Каждую главу можно изучать независимо, но для последовательного освоения материала рекомендуется идти по порядку — от первой главы к последней. Подглавы внутри одной главы логически дополняют друг друга и раскрывают тему шаг за шагом.

Оптимальный режим работы:

1. Прочитать главу;
2. Открыть соответствующий коммит в [сопутствующем проекте](https://github.com/rustkas/hello-wasm02);
3. Повторить код у себя, внося собственные изменения;
4. Использовать пример как основу для своих задач.

---

## Как внести вклад

Если вы нашли ошибку, опечатку или хотите предложить улучшение — откройте Issue или пришлите Pull Request. Приветствуются:

- исправления опечаток и неточностей;
- уточнения в примерах кода;
- предложения по новым темам;
- переводы глав.

---

## Лицензия

Материалы книги распространяются под лицензией MIT. Подробности — в файле [LICENSE](LICENSE).
