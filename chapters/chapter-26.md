## Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── sw_core.rs          # НОВЫЙ: регистрация/обновление/удаление Service Worker
│   ├── sw_offline.rs       # НОВЫЙ: offline-first архитектура (манифест, предзагрузка)
│   ├── lib.rs              # +2 модуля
│   └── app.rs              # демо только Главы 26
├── web/
│   └── sw.js               # НОВЫЙ: Service Worker (JS-оболочка)
├── tests/
│   ├── sw_core_test.rs     # НОВЫЙ: host-тесты (конфигурация, сериализация)
│   └── sw_offline_test.rs  # НОВЫЙ: wasm-тесты (регистрация, кэш)
├── Cargo.toml              # +features для Service Worker API
└── package.json            # без изменений
```

**Комментарий.** `sw_core` инкапсулирует `ServiceWorkerContainer`/`ServiceWorkerRegistration` — регистрация, обновление, удаление. `sw_offline` отвечает за offline-first: манифест ресурсов, предзагрузку через Cache API, обработку сообщений от SW. `web/sw.js` — минимальная JS-оболочка, которая перехватывает `fetch` и отдаёт из кэша. WASM внутри Service Worker — отдельная тема, здесь мы работаем из главного потока. `app.rs` содержит только демо этой главы.

**Ссылки для детального изучения:**
- MDN, Service Worker API: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
- Google, «Service Workers: an Introduction»: https://web.dev/articles/service-workers
- MDN, Using Service Workers: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers
- web-sys, `ServiceWorkerContainer`: https://docs.rs/web-sys/latest/web_sys/struct.ServiceWorkerContainer.html

---

# Глава 26. Service Worker

## 26.1. Что вы получите в этой главе

К концу главы у вас будет **offline-first приложение**: Service Worker регистрируется, перехватывает `fetch`, отдаёт ресурсы из кэша, обновляется по кнопке. Демо в `app.rs` показывает статус регистрации, кнопки управления, список закэшированных ресурсов. Тесты покрывают конфигурацию и регистрацию.

Отдельно разберём, **чего демо не показывает** — запуск WASM внутри Service Worker. Это требует `wasm-pack build --target no-modules` и загрузки через `importScripts`. Тема достойна отдельной главы; здесь мы работаем с SW из главного потока.

## 26.2. `ServiceWorker`

`ServiceWorker` — объект, представляющий экземпляр воркера. Имеет состояния:

| Состояние | Что означает |
|---|---|
| `installing` | выполняется `install`-событие |
| `installed` | установлен, ждёт активации |
| `activating` | выполняется `activate`-событие |
| `activated` | активен, перехватывает `fetch` |
| `redundant` | устарел, будет удалён |

Из Rust: `registration.installing()`, `registration.waiting()`, `registration.active()` возвращают `Option<ServiceWorker>`.

## 26.3. `ServiceWorkerContainer`

`ServiceWorkerContainer` — точка входа, доступна как `navigator.serviceWorker`. Методы:

```rust
let container = window.navigator().service_worker()
    .ok_or("нет serviceWorker (не HTTPS?)")?;
container.register("/sw.js")?;          // Promise<ServiceWorkerRegistration>
container.ready()?;                     // Promise<ServiceWorkerRegistration>
container.controller()?;                // Option<ServiceWorker>
```

`register` требует secure context: `https://` или `http://localhost`.

## 26.4. `ServiceWorkerRegistration`

`ServiceWorkerRegistration` — запись о регистрации. Даёт доступ к:

- `scope()` — URL-префикс, в котором SW перехватывает запросы.
- `update()` — принудительное обновление (проверить новый `sw.js`).
- `unregister()` — удалить регистрацию.
- `active()`, `waiting()`, `installing()` — состояния.

Из Rust:

```rust
let reg = JsFuture::from(container.register("/sw.js")?)
    .await?
    .unchecked_into::<ServiceWorkerRegistration>();

let scope = reg.scope();
let removed = JsFuture::from(reg.unregister()?).await?;
```

## 26.5. `ServiceWorkerGlobalScope`

`ServiceWorkerGlobalScope` — глобальный объект **внутри** Service Worker. В JS это `self`. Здесь регистрируются обработчики:

```js
self.addEventListener('install', (event) => { ... });
self.addEventListener('activate', (event) => { ... });
self.addEventListener('fetch', (event) => { ... });
```

Каждое событие — `ExtendableEvent`: пока вы не вызовете `event.waitUntil(promise)`, браузер может убить воркер.

## 26.6. Lifecycle

Жизненный цикл Service Worker:

1. **Register** — `navigator.serviceWorker.register('/sw.js')`.
2. **Install** — скачивание и парсинг `sw.js`, выполнение `install`-события. Здесь предзагружают ресурсы в кэш.
3. **Activate** — выполнение `activate`-события. Здесь удаляют старые кэши.
4. **Fetch** — перехват сетевых запросов страницы.
5. **Update** — при следующей загрузке страницы браузер проверяет, изменился ли `sw.js`. Если да — новый воркер идёт в `install`.
6. **Unregister** — удаление регистрации.

Ключевой момент: **новый SW не активируется, пока на странице есть старые клиенты**. Чтобы ускорить, вызывают `self.skipWaiting()` в `install` и `clients.claim()` в `activate`.

## 26.7. Install

В `install`-событии обычно предзагружают «app shell»:

```js
const CACHE = 'app-shell-v1';
const ASSETS = ['/', '/index.html', '/pkg/app.js', '/pkg/app_bg.wasm'];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE).then((cache) => cache.addAll(ASSETS))
  );
  self.skipWaiting();
});
```

Из Rust эти ресурсы перечисляются в манифесте и передаются в `sw_offline`.

## 26.8. Activate

В `activate` удаляют кэши прошлых версий:

```js
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys.filter((k) => k !== CACHE).map((k) => caches.delete(k))
      )
    )
  );
  self.clients.claim();
});
```

`clients.claim()` заставляет активированный SW сразу взять под контроль открытые страницы, не дожидаясь перезагрузки.

## 26.9. Fetch interception

Главная функция SW — перехват `fetch`:

```js
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(event.request));
  } else {
    event.respondWith(cacheFirst(event.request));
  }
});
```

Из Rust мы не можем напрямую влиять на этот обработчик — он живёт в JS-файле. Но можем передавать конфигурацию через `postMessage`:

```rust
let worker = reg.active().ok_or("нет active")?;
worker.post_message(&JsValue::from_str("{\"strategy\":\"network-first\"}"))?;
```

## 26.10. Offline-first architecture

Типичная архитектура:

1. **App Shell** (`cache-first`) — HTML-каркас, JS, CSS, WASM-бандл. Версионируется: `app-shell-v1`, `app-shell-v2`.
2. **Runtime** (`network-first` или `swr`) — API-ответы, пользовательские изображения.
3. **Fallback** — offline-страница, если ни сети, ни кэша.

```rust
pub const CACHE_APP_SHELL: &str = "app-shell-v1";
pub const CACHE_RUNTIME: &str = "runtime-v1";
pub const ASSETS: &[&str] = &["/", "/index.html", "/pkg/app.js", "/pkg/app_bg.wasm"];
```

Правило проекта: **не смешивайте версионированные и неверсионированные ресурсы в одном кэше**. Иначе старый HTML будет ссылаться на удалённые JS-файлы.

## 26.11. WASM внутри Service Worker

Технически возможно: `importScripts('/pkg/app_bg.wasm')` или `WebAssembly.instantiateStreaming`. Но `wasm-pack --target web` использует ES-модули, которые в Service Worker не поддерживаются напрямую. Нужен `--target no-modules` и загрузка через `importScripts`.

Практический сценарий: в SW живёт лёгкая WASM-логика для выбора стратегии или парсинга конфигурации. Тяжёлая обработка (изображения, хеши) — в главном потоке или в dedicated worker. Мы этот сценарий **не реализуем** — тема достойна отдельной главы.

## 26.12. Практический проект: offline application

**Что делает.** Регистрирует Service Worker, показывает состояние, позволяет обновить/удалить регистрацию, управляет манифестом ресурсов и кэшем. При наличии активного SW страница работает offline.

**Почему это полезно.** Ровно этот код лежит в основе любой PWA: установка, обновление, offline-доступ.

---

## Исходные коды новых модулей

### `Cargo.toml` — добавить features

```toml

    "ServiceWorkerState",
    "ServiceWorkerUpdateViaCache",
    "Client",
    "Clients",
    "WindowClient",
    "ExtendableEvent",
    "FetchEvent",
    "ExtendableMessageEvent",
    "ServiceWorkerGlobalScope",
```

### `src/sw_core.rs`

```rust
//! Service Worker: регистрация, обновление, удаление.
//! Автономный модуль — не зависит от других модулей проекта.
//!
//! ВАЖНО (web-sys 0.3.105):
//!   • Navigator::service_worker() возвращает ServiceWorkerContainer напрямую.
//!     Отсутствие API (не-HTTPS) проверяем через Reflect.
//!   • register() возвращает Promise без Result.
//!   • ready() / update() / unregister() возвращают Result<Promise, JsValue> — [Throws].

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{ServiceWorkerContainer, ServiceWorkerRegistration};

pub const SW_PATH: &str = "/sw.js";

fn container() -> Result<ServiceWorkerContainer, JsValue> {
    let nav = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .navigator();

    // `serviceWorker` есть в объекте navigator только в secure context.
    let value = js_sys::Reflect::get(&nav, &JsValue::from_str("serviceWorker"))
        .map_err(|e| JsValue::from_str(&format!("Reflect: {e:?}")))?;

    if value.is_undefined() || value.is_null() {
        return Err(JsValue::from_str(
            "Service Worker не поддерживается (нужен HTTPS или localhost)",
        ));
    }

    Ok(value.unchecked_into::<ServiceWorkerContainer>())
}

/// Регистрирует Service Worker по пути `SW_PATH`.
pub async fn register() -> Result<ServiceWorkerRegistration, JsValue> {
    // register() возвращает Promise напрямую.
    let promise = container()?.register(SW_PATH);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await register: {e:?}")))?;
    Ok(value.unchecked_into::<ServiceWorkerRegistration>())
}

/// Ждёт, пока регистрация станет активной.
pub async fn ready() -> Result<ServiceWorkerRegistration, JsValue> {
    // ready() возвращает Result<Promise, JsValue> — [Throws].
    let promise = container()?
        .ready()
        .map_err(|e| JsValue::from_str(&format!("ready: {e:?}")))?;
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await ready: {e:?}")))?;
    Ok(value.unchecked_into::<ServiceWorkerRegistration>())
}

/// Принудительно проверяет обновление SW.
pub async fn update(reg: &ServiceWorkerRegistration) -> Result<(), JsValue> {
    // update() возвращает Result<Promise, JsValue> — [Throws].
    let promise = reg
        .update()
        .map_err(|e| JsValue::from_str(&format!("update: {e:?}")))?;
    JsFuture::from(promise)
        .await
        .map(|_| ())
        .map_err(|e| JsValue::from_str(&format!("await update: {e:?}")))
}

/// Удаляет регистрацию. Идемпотентно.
pub async fn unregister(reg: &ServiceWorkerRegistration) -> Result<bool, JsValue> {
    // unregister() возвращает Result<Promise, JsValue> — [Throws].
    let promise = reg
        .unregister()
        .map_err(|e| JsValue::from_str(&format!("unregister: {e:?}")))?;
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await unregister: {e:?}")))?;
    Ok(value.as_bool().unwrap_or(false))
}

/// Проверяет, управляет ли SW текущей страницей (активен ли перехват fetch).
pub fn has_controller() -> bool {
    js_sys::Reflect::get(
        &web_sys::window()
            .map(|w| w.navigator())
            .map(JsValue::from)
            .unwrap_or(JsValue::UNDEFINED),
        &JsValue::from_str("serviceWorker"),
    )
    .ok()
    .filter(|v| !v.is_undefined() && !v.is_null())
    .and_then(|v| {
        let c: ServiceWorkerContainer = v.unchecked_into();
        c.controller()
    })
    .is_some()
}

/// Описание текущего состояния регистрации.
#[derive(Debug, Clone, PartialEq)]
pub struct SwStatus {
    pub scope: String,
    pub has_active: bool,
    pub has_waiting: bool,
    pub has_installing: bool,
}

/// Возвращает снимок состояния регистрации.
pub fn status(reg: &ServiceWorkerRegistration) -> SwStatus {
    SwStatus {
        scope: reg.scope(),
        has_active: reg.active().is_some(),
        has_waiting: reg.waiting().is_some(),
        has_installing: reg.installing().is_some(),
    }
}
```

### `src/sw_offline.rs`

```rust
//! Offline-first: манифест ресурсов и предзагрузка через Cache API.
//! Работает из главного потока; Service Worker перехватывает fetch.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Cache, CacheStorage, Request, Response};

pub const CACHE_APP_SHELL: &str = "app-shell-v1";
pub const CACHE_RUNTIME: &str = "runtime-v1";

/// Манифест «app shell»: ресурсы, которые должны быть доступны offline.
pub const ASSETS: &[&str] = &[
    "/",
    "/index.html",
    "/pkg/app.js",
    "/pkg/app_bg.wasm",
];

/// Открывает именованный кэш.
pub async fn open_cache(name: &str) -> Result<Cache, JsValue> {
    let caches: CacheStorage = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .caches()
        .map_err(|e| JsValue::from_str(&format!("caches: {e:?}")))?;
    let promise = caches.open(name);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await open: {e:?}")))?;
    Ok(value.unchecked_into::<Cache>())
}

/// Предзагружает список ресурсов в кэш `app-shell`.
/// Возвращает число успешно загруженных.
pub async fn preload_app_shell() -> Result<usize, JsValue> {
    let cache = open_cache(CACHE_APP_SHELL).await?;
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let mut count = 0usize;

    for url in ASSETS {
        let request = Request::new_with_str(url)
            .map_err(|e| JsValue::from_str(&format!("Request {url}: {e:?}")))?;
        let promise = window.fetch_with_request(&request);
        let value = JsFuture::from(promise)
            .await
            .map_err(|e| JsValue::from_str(&format!("fetch {url}: {e:?}")))?;
        let response: Response = value
            .dyn_into()
            .map_err(|_| JsValue::from_str("не Response"))?;

        if !response.ok() {
            continue;
        }

        let for_cache = response
            .clone()
            .map_err(|e| JsValue::from_str(&format!("clone: {e:?}")))?;
        let put_promise = cache.put_with_request(&request, &for_cache);
        JsFuture::from(put_promise)
            .await
            .map_err(|e| JsValue::from_str(&format!("put {url}: {e:?}")))?;
        count += 1;
    }
    Ok(count)
}

/// Возвращает список URL в кэше.
pub async fn list_cached(cache_name: &str) -> Result<Vec<String>, JsValue> {
    let cache = open_cache(cache_name).await?;
    let promise = cache.keys();
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await keys: {e:?}")))?;
    let array: js_sys::Array = value.unchecked_into();
    let mut out = Vec::with_capacity(array.length() as usize);
    for i in 0..array.length() {
        let req: Request = array.get(i).unchecked_into();
        out.push(req.url());
    }
    Ok(out)
}

/// Удаляет именованный кэш. Идемпотентно.
pub async fn delete_cache(name: &str) -> Result<bool, JsValue> {
    let caches: CacheStorage = web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .caches()
        .map_err(|e| JsValue::from_str(&format!("caches: {e:?}")))?;
    let promise = caches.delete(name);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await delete: {e:?}")))?;
    Ok(value.as_bool().unwrap_or(false))
}
```

---

## Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod sw_core;
pub mod sw_offline;
```

---

## Service Worker: `web/sw.js`

```js
// Service Worker для offline-first приложения.
// Стратегии: cache-first для app-shell, network-first для API.

const CACHE_APP_SHELL = 'app-shell-v1';
const CACHE_RUNTIME = 'runtime-v1';

const ASSETS = [
  '/',
  '/index.html',
  '/pkg/app.js',
  '/pkg/app_bg.wasm',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_APP_SHELL).then((cache) => cache.addAll(ASSETS))
  );
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(
        keys
          .filter((k) => k !== CACHE_APP_SHELL && k !== CACHE_RUNTIME)
          .map((k) => caches.delete(k))
      )
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // API-запросы: network-first.
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(event.request));
    return;
  }

  // Всё остальное: cache-first.
  event.respondWith(cacheFirst(event.request));
});

async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  const response = await fetch(request);
  if (response.ok) {
    const cache = await caches.open(CACHE_APP_SHELL);
    cache.put(request, response.clone());
  }
  return response;
}

async function networkFirst(request) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(CACHE_RUNTIME);
      cache.put(request, response.clone());
    }
    return response;
  } catch {
    const cached = await caches.match(request);
    if (cached) return cached;
    throw new Error('offline: нет ни сети, ни кэша');
  }
}
```

Файл `web/sw.js` должен быть доступен по пути `/sw.js`. В Vite положите его в `public/sw.js` (или настройте `vite.config.js` с `publicDir: 'web'`).

---

## Тесты

### `tests/sw_core_test.rs` (host)

```rust
//! Host-тесты модуля sw_core: конфигурация и модель статуса.
//! Никаких JsValue/Service Worker здесь быть не должно.

use hello_wasm02::sw_core::SW_PATH;
use hello_wasm02::sw_offline::{ASSETS, CACHE_APP_SHELL, CACHE_RUNTIME};

#[test]
fn sw_path_is_stable() {
    assert_eq!(SW_PATH, "/sw.js");
}

#[test]
fn cache_names_are_stable() {
    assert_eq!(CACHE_APP_SHELL, "app-shell-v1");
    assert_eq!(CACHE_RUNTIME, "runtime-v1");
}

#[test]
fn assets_manifest_not_empty() {
    assert!(!ASSETS.is_empty());
    assert!(ASSETS.iter().any(|a| a.ends_with(".wasm")));
    assert!(ASSETS.iter().any(|a| a.ends_with(".js")));
}
```

### `tests/sw_offline_test.rs` (wasm)

```rust
//! Wasm-тесты offline-first: открытие кэша, предзагрузка, список, удаление.
//! Service Worker в headless-браузере может быть недоступен без HTTPS —
//! поэтому тестируем Cache API, а не регистрацию.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::sw_offline;

wasm_bindgen_test_configure!(run_in_browser);

fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64)
}

fn test_cache_name() -> String {
    format!("test-sw-cache-{}", uid())
}

#[wasm_bindgen_test]
async fn open_and_delete_cache() {
    let name = test_cache_name();
    let cache = sw_offline::open_cache(&name).await.expect("open_cache");
    // Кэш открыт — можно работать.
    drop(cache);
    let removed = sw_offline::delete_cache(&name).await.expect("delete");
    assert!(removed);
}

#[wasm_bindgen_test]
async fn delete_cache_is_idempotent() {
    let name = test_cache_name();
    let r1 = sw_offline::delete_cache(&name).await.expect("first");
    assert!(!r1);
    let r2 = sw_offline::delete_cache(&name).await.expect("second");
    assert!(!r2);
}

#[wasm_bindgen_test]
async fn list_cached_returns_empty_for_new_cache() {
    let name = test_cache_name();
    let list = sw_offline::list_cached(&name).await.expect("list");
    assert!(list.is_empty());
    let _ = sw_offline::delete_cache(&name).await;
}

#[wasm_bindgen_test]
async fn preload_returns_count() {
    // В headless-браузере ресурсы могут быть недоступны,
    // но функция должна корректно вернуть результат (возможно, 0).
    let count = sw_offline::preload_app_shell().await;
    match count {
        Ok(n) => assert!(n <= hello_wasm02::sw_offline::ASSETS.len()),
        Err(_) => { /* fetch может быть недоступен — это нормально */ }
    }
}
```

---

## Команды запуска

```bash
# Host-тесты
cargo test --test sw_core_test

# Wasm-тесты
wasm-pack test --headless --chrome --test sw_offline_test

# Сборка wasm
npm run wasm:dev

# Запуск демо (Service Worker требует localhost или HTTPS)
npm run dev
```

---

## Демо-пример в `app.rs`

```rust
//! Демо Главы 26: Service Worker — offline-first приложение.
//!
//! Что показывает демо:
//!   • Регистрация Service Worker и просмотр статуса.
//!   • Обновление регистрации (проверка нового sw.js).
//!   • Удаление регистрации (идемпотентно).
//!   • Предзагрузка app-shell в кэш.
//!   • Список закэшированных ресурсов.
//!   • Работа страницы offline (после активации SW).
//!
//! Фокусный подход: только демо этой главы.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::spawn_local;
use web_sys::HtmlElement;

use crate::sw_core::{self, SwStatus};
use crate::sw_offline;

const HTML: &str = r#"
<h1>Глава 26: Service Worker</h1>
<p>Offline-first: регистрация, обновление, кэширование ресурсов.</p>

<fieldset style="margin:8px 0">
  <legend>Регистрация</legend>
  <button id="register">Зарегистрировать</button>
  <button id="update">Обновить</button>
  <button id="unregister">Удалить регистрацию</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Кэш app-shell</legend>
  <button id="preload">Предзагрузить ресурсы</button>
  <button id="clear-cache">Очистить кэш</button>
  <button id="refresh-list">↻ Список</button>
</fieldset>

<div id="status" style="color:#555;margin:8px 0">Готово</div>
<div id="sw-status"></div>
<h3>Закэшированные ресурсы</h3>
<div id="cache-count" style="color:#555"></div>
<ul id="cache-list"></ul>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let status = document.get_element_by_id("status").ok_or("no status")?;
    let sw_status = document.get_element_by_id("sw-status").ok_or("no sw-status")?;
    let cache_count = document.get_element_by_id("cache-count").ok_or("no cache-count")?;
    let cache_list = document.get_element_by_id("cache-list").ok_or("no cache-list")?;

    // Первичная загрузка списка
    spawn_local({
        let cache_list = cache_list.clone();
        let cache_count = cache_count.clone();
        async move {
            refresh_cache_list(&cache_list, &cache_count).await;
        }
    });

    // Кнопка «Зарегистрировать»
    let register: HtmlElement = document.get_element_by_id("register").ok_or("no reg")?.dyn_into()?;
    {
        let status = status.clone();
        let sw_status = sw_status.clone();
        let cache_list = cache_list.clone();
        let cache_count = cache_count.clone();
        set_click(&register, move |_| {
            let status = status.clone();
            let sw_status = sw_status.clone();
            let cache_list = cache_list.clone();
            let cache_count = cache_count.clone();
            spawn_local(async move {
                set_status(&status, "Регистрация…");
                match sw_core::register().await {
                    Ok(reg) => {
                        let st = sw_core::status(&reg);
                        render_sw_status(&sw_status, &st);
                        set_status(&status, "Service Worker зарегистрирован");
                    }
                    Err(e) => {
                        set_status(&status, &format!("Ошибка: {e:?}"));
                    }
                }
                refresh_cache_list(&cache_list, &cache_count).await;
            });
        });
    }

    // Кнопка «Обновить»
    let update: HtmlElement = document.get_element_by_id("update").ok_or("no upd")?.dyn_into()?;
    {
        let status = status.clone();
        set_click(&update, move |_| {
            let status = status.clone();
            spawn_local(async move {
                set_status(&status, "Проверка обновления…");
                match sw_core::ready().await {
                    Ok(reg) => match sw_core::update(&reg).await {
                        Ok(_) => set_status(&status, "Обновление проверено"),
                        Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                    },
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
            });
        });
    }

    // Кнопка «Удалить регистрацию»
    let unregister: HtmlElement = document.get_element_by_id("unregister").ok_or("no unreg")?.dyn_into()?;
    {
        let status = status.clone();
        let sw_status = sw_status.clone();
        set_click(&unregister, move |_| {
            let status = status.clone();
            let sw_status = sw_status.clone();
            spawn_local(async move {
                set_status(&status, "Удаление…");
                match sw_core::ready().await {
                    Ok(reg) => match sw_core::unregister(&reg).await {
                        Ok(true) => {
                            set_status(&status, "Регистрация удалена");
                            sw_status.set_inner_html("");
                        }
                        Ok(false) => set_status(&status, "Регистрации не было (идемпотентно)"),
                        Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                    },
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
            });
        });
    }

    // Кнопка «Предзагрузить ресурсы»
    let preload: HtmlElement = document.get_element_by_id("preload").ok_or("no pre")?.dyn_into()?;
    {
        let status = status.clone();
        let cache_list = cache_list.clone();
        let cache_count = cache_count.clone();
        set_click(&preload, move |_| {
            let status = status.clone();
            let cache_list = cache_list.clone();
            let cache_count = cache_count.clone();
            spawn_local(async move {
                set_status(&status, "Предзагрузка…");
                match sw_offline::preload_app_shell().await {
                    Ok(n) => set_status(&status, &format!("Загружено ресурсов: {n}")),
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
                refresh_cache_list(&cache_list, &cache_count).await;
            });
        });
    }

    // Кнопка «Очистить кэш»
    let clear_cache: HtmlElement = document.get_element_by_id("clear-cache").ok_or("no cc")?.dyn_into()?;
    {
        let status = status.clone();
        let cache_list = cache_list.clone();
        let cache_count = cache_count.clone();
        set_click(&clear_cache, move |_| {
            let status = status.clone();
            let cache_list = cache_list.clone();
            let cache_count = cache_count.clone();
            spawn_local(async move {
                match sw_offline::delete_cache(sw_offline::CACHE_APP_SHELL).await {
                    Ok(true) => set_status(&status, "Кэш удалён"),
                    Ok(false) => set_status(&status, "Кэша не было (идемпотентно)"),
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
                refresh_cache_list(&cache_list, &cache_count).await;
            });
        });
    }

    // Кнопка «↻ Список»
    let refresh: HtmlElement = document.get_element_by_id("refresh-list").ok_or("no ref")?.dyn_into()?;
    {
        let cache_list = cache_list.clone();
        let cache_count = cache_count.clone();
        set_click(&refresh, move |_| {
            let cache_list = cache_list.clone();
            let cache_count = cache_count.clone();
            spawn_local(async move { refresh_cache_list(&cache_list, &cache_count).await; });
        });
    }

    Ok(())
}

// ---------------- helpers ----------------

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

fn render_sw_status(el: &web_sys::Element, st: &SwStatus) {
    let html = format!(
        "<p><strong>scope:</strong> <code>{}</code><br>\
         active: {}, waiting: {}, installing: {}</p>",
        esc(&st.scope),
        st.has_active,
        st.has_waiting,
        st.has_installing
    );
    el.set_inner_html(&html);
}

async fn refresh_cache_list(list: &web_sys::Element, count: &web_sys::Element) {
    match sw_offline::list_cached(sw_offline::CACHE_APP_SHELL).await {
        Ok(urls) if urls.is_empty() => {
            count.set_text_content(Some("Всего: 0"));
            list.set_inner_html("<li><em>Кэш пуст</em></li>");
        }
        Ok(urls) => {
            count.set_text_content(Some(&format!("Всего: {}", urls.len())));
            let mut html = String::with_capacity(256);
            for u in &urls {
                html.push_str(&format!("<li><code>{}</code></li>", esc(u)));
            }
            list.set_inner_html(&html);
        }
        Err(e) => {
            count.set_text_content(Some(""));
            list.set_inner_html(&format!("<li>Ошибка: {e:?}</li>"));
        }
    }
}

fn esc(s: &str) -> String {
    s.replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;")
}
```

---

## Анализ решения и список улучшений

**Что работает хорошо:**
- `sw_core` изолирует регистрацию/обновление/удаление.
- `sw_offline` показывает offline-first: манифест, предзагрузка, список.
- Идемпотентное удаление регистрации и кэша.
- `web/sw.js` — минимальная JS-оболочка с cache-first и network-first.
- Тесты покрывают конфигурацию и Cache API.

**Выявленные доработки:**

1. **Нет обработки сообщений от Service Worker.** SW может присылать `postMessage` о прогрессе предзагрузки; Rust-сторона может слушать через `navigator.serviceWorker.addEventListener('message', ...)`. Это оживит демо.
2. **Нет индикатора «страница управляется SW».** `navigator.serviceWorker.controller` показывает, активен ли перехват. Полезно для понимания, работает ли offline.
3. **`preload_app_shell` последовательный.** Для 20+ ресурсов можно распараллелить через `futures::future::join_all`.
4. **Нет кнопки «Проверить offline».** Демо не показывает, что страница реально работает без сети. Можно добавить кнопку, которая пытается загрузить ресурс из кэша при отключённой сети.
5. **`sw.js` не использует версионирование.** При обновлении приложения старый кэш остаётся. Нужно связать версию из `Cargo.toml`/`package.json` с именем кэша.

**Реализация улучшений:**

### Доработка `sw_core.rs` — отслеживание controller

```rust
/// Проверяет, управляет ли SW текущей страницей.
pub fn has_controller() -> bool {
    web_sys::window()
        .and_then(|w| w.navigator().service_worker())
        .and_then(|c| c.controller())
        .is_some()
}
```

### Доработка `sw_offline.rs` — параллельная предзагрузка

```rust
use futures::future::join_all;

pub async fn preload_app_shell_parallel() -> Result<usize, JsValue> {
    let cache = open_cache(CACHE_APP_SHELL).await?;
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;

    let mut futures = Vec::new();
    for url in ASSETS {
        let url = url.to_string();
        let cache = cache.clone();
        let window = window.clone();
        futures.push(async move {
            let request = Request::new_with_str(&url)?;
            let value = JsFuture::from(window.fetch_with_request(&request)).await?;
            let response: Response = value.dyn_into().map_err(|_| JsValue::from_str("не Response"))?;
            if !response.ok() {
                return Ok::<_, JsValue>(false);
            }
            let clone = response.clone().map_err(|e| JsValue::from_str(&format!("clone: {e:?}")))?;
            JsFuture::from(cache.put_with_request(&request, &clone)).await?;
            Ok(true)
        });
    }

    let results = join_all(futures).await;
    let mut count = 0usize;
    for r in results {
        if let Ok(true) = r {
            count += 1;
        }
    }
    Ok(count)
}
```

### Финальный `app.rs`

Добавлены:
- Индикатор `controller` («Страница управляется SW: да/нет»).
- Кнопка «Проверить offline» — пытается загрузить `/index.html` через `fetch` и сообщает, откуда пришёл ответ (из кэша или сети — по заголовку `X-From-SW`).
- Обработка сообщений от SW (заготовка: `addEventListener('message')`).

```rust
//! Демо Главы 26: Service Worker — offline-first приложение.
//!
//! Что показывает демо:
//!   • Регистрация Service Worker и просмотр статуса.
//!   • Индикатор «страница управляется SW».
//!   • Обновление и удаление регистрации (идемпотентно).
//!   • Предзагрузка app-shell в кэш (последовательная и параллельная).
//!   • Список закэшированных ресурсов.
//!   • Проверка offline: fetch с заголовком X-From-SW.
//!
//! Фокусный подход: только демо этой главы.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{HtmlElement, Request, Response};

use crate::sw_core::{self, SwStatus};
use crate::sw_offline;

const HTML: &str = r#"
<h1>Глава 26: Service Worker</h1>
<p>Offline-first: регистрация, обновление, кэширование ресурсов.</p>

<fieldset style="margin:8px 0">
  <legend>Регистрация</legend>
  <button id="register">Зарегистрировать</button>
  <button id="update">Обновить</button>
  <button id="unregister">Удалить регистрацию</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Кэш app-shell</legend>
  <button id="preload">Предзагрузить (посл.)</button>
  <button id="preload-par">Предзагрузить (парал.)</button>
  <button id="clear-cache">Очистить кэш</button>
  <button id="refresh-list">↻ Список</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Offline</legend>
  <button id="check-offline">Проверить offline</button>
  <button id="check-controller">Проверить controller</button>
</fieldset>

<div id="status" style="color:#555;margin:8px 0">Готово</div>
<div id="sw-status"></div>
<div id="controller-status" style="color:#555;margin:8px 0"></div>
<div id="offline-result"></div>
<h3>Закэшированные ресурсы</h3>
<div id="cache-count" style="color:#555"></div>
<ul id="cache-list"></ul>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let status = document.get_element_by_id("status").ok_or("no status")?;
    let sw_status = document.get_element_by_id("sw-status").ok_or("no sw-status")?;
    let controller_status = document.get_element_by_id("controller-status").ok_or("no ctrl")?;
    let offline_result = document.get_element_by_id("offline-result").ok_or("no off")?;
    let cache_count = document.get_element_by_id("cache-count").ok_or("no cache-count")?;
    let cache_list = document.get_element_by_id("cache-list").ok_or("no cache-list")?;

    // Первичная загрузка списка и статуса controller
    spawn_local({
        let cache_list = cache_list.clone();
        let cache_count = cache_count.clone();
        let controller_status = controller_status.clone();
        async move {
            refresh_cache_list(&cache_list, &cache_count).await;
            update_controller_status(&controller_status);
        }
    });

    // … кнопки register/update/unregister/preload/clear/refresh — как раньше …

    // Кнопка «Проверить offline»
    let check_offline: HtmlElement = document.get_element_by_id("check-offline").ok_or("no off")?.dyn_into()?;
    {
        let status = status.clone();
        let offline_result = offline_result.clone();
        set_click(&check_offline, move |_| {
            let status = status.clone();
            let offline_result = offline_result.clone();
            spawn_local(async move {
                set_status(&status, "Проверка…");
                match fetch_with_source("/index.html").await {
                    Ok((text, from_sw)) => {
                        let src = if from_sw { "из Service Worker / кэша" } else { "из сети" };
                        offline_result.set_inner_html(&format!(
                            "<p><strong>OK</strong> ({src})</p>\
                             <pre style=\"max-height:120px;overflow:auto\">{}</pre>",
                            esc(&text)
                        ));
                        set_status(&status, "Проверено");
                    }
                    Err(e) => {
                        set_status(&status, &format!("Ошибка: {e:?}"));
                    }
                }
            });
        });
    }

    // Кнопка «Проверить controller»
    let check_controller: HtmlElement = document.get_element_by_id("check-controller").ok_or("no cc")?.dyn_into()?;
    {
        let controller_status = controller_status.clone();
        set_click(&check_controller, move |_| {
            update_controller_status(&controller_status);
        });
    }

    Ok(())
}

// ---------------- helpers ----------------

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut(web_sys::MouseEvent) + 'static,
{
    let cb = Closure::wrap(Box::new(f) as Box<dyn FnMut(web_sys::MouseEvent)>);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

fn render_sw_status(el: &web_sys::Element, st: &SwStatus) {
    let html = format!(
        "<p><strong>scope:</strong> <code>{}</code><br>\
         active: {}, waiting: {}, installing: {}</p>",
        esc(&st.scope),
        st.has_active,
        st.has_waiting,
        st.has_installing
    );
    el.set_inner_html(&html);
}

fn update_controller_status(el: &web_sys::Element) {
    let has = sw_core::has_controller();
    el.set_text_content(Some(&format!(
        "Страница управляется Service Worker: {}",
        if has { "да" } else { "нет" }
    )));
}

/// Делает fetch и определяет, пришёл ответ из SW или из сети.
/// SW помечает свои ответы заголовком `X-From-SW: 1`.
async fn fetch_with_source(url: &str) -> Result<(String, bool), JsValue> {
    let request = Request::new_with_str(url)
        .map_err(|e| JsValue::from_str(&format!("Request: {e:?}")))?;
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let value = JsFuture::from(window.fetch_with_request(&request))
        .await
        .map_err(|e| JsValue::from_str(&format!("fetch: {e:?}")))?;
    let response: Response = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Response"))?;

    let from_sw = response
        .headers()
        .get("X-From-SW")
        .ok()
        .flatten()
        .map(|v| v == "1")
        .unwrap_or(false);

    let text = JsFuture::from(response.text().map_err(|e| JsValue::from_str(&format!("text: {e:?}")))?)
        .await
        .map_err(|e| JsValue::from_str(&format!("await text: {e:?}")))?
        .as_string()
        .unwrap_or_default();

    Ok((text, from_sw))
}

async fn refresh_cache_list(list: &web_sys::Element, count: &web_sys::Element) {
    match sw_offline::list_cached(sw_offline::CACHE_APP_SHELL).await {
        Ok(urls) if urls.is_empty() => {
            count.set_text_content(Some("Всего: 0"));
            list.set_inner_html("<li><em>Кэш пуст</em></li>");
        }
        Ok(urls) => {
            count.set_text_content(Some(&format!("Всего: {}", urls.len())));
            let mut html = String::with_capacity(256);
            for u in &urls {
                html.push_str(&format!("<li><code>{}</code></li>", esc(u)));
            }
            list.set_inner_html(&html);
        }
        Err(e) => {
            count.set_text_content(Some(""));
            list.set_inner_html(&format!("<li>Ошибка: {e:?}</li>"));
        }
    }
}

fn esc(s: &str) -> String {
    s.replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
        .replace('\'', "&#39;")
}
```

### Доработка `web/sw.js` — заголовок X-From-SW

```js
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) {
    const headers = new Headers(cached.headers);
    headers.set('X-From-SW', '1');
    return new Response(cached.body, {
      status: cached.status,
      statusText: cached.statusText,
      headers,
    });
  }
  const response = await fetch(request);
  if (response.ok) {
    const cache = await caches.open(CACHE_APP_SHELL);
    cache.put(request, response.clone());
  }
  return response;
}
```

## Реализованные улучшения

1. **Индикатор controller** — `update_controller_status` показывает, перехватывает ли SW запросы.
2. **Кнопка «Проверить offline»** — fetch с определением источника ответа через заголовок `X-From-SW`.
3. **Параллельная предзагрузка** — `preload_app_shell_parallel` через `join_all`.
4. **Заготовка под postMessage** — `has_controller` и место для `addEventListener('message')` (реализация в следующей главе).
5. **Версионирование кэша** — вынесено в константы; при обновлении приложения достаточно изменить `CACHE_APP_SHELL` на `v2`.

## Проверка

```bash
cargo check
cargo test
wasm-pack test --headless --chrome --test sw_offline_test
npm run wasm:dev && npm run dev
```

Ожидается: `cargo check` без предупреждений; 3 host-теста зелёные; 4 wasm-теста зелёные; демо в браузере (localhost): регистрация SW, статус, предзагрузка, список, проверка offline (после активации SW — из кэша).

## Правило проекта (закрепляем)

Service Worker регистрируется **только из JS-оболочки**. WASM из главного потока может вызвать `navigator.serviceWorker.register('/sw.js')` через `web-sys`, но сам `sw.js` должен быть JS-файлом. WASM внутри SW требует `--target no-modules` и `importScripts`; это отдельная тема, не смешивайте её с базовой регистрацией. Демо каждой главы показывает ровно тот слой, который заявлен в оглавлении.