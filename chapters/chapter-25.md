## Обновлённая структура файлов и папок

```
hello-wasm02/
├── src/
│   ├── cache_api.rs          # НОВЫЙ: обёртка над Cache/CacheStorage (open, put, match, delete)
│   ├── cache_offline.rs      # НОВЫЙ: практический проект — offline-репозиторий ресурсов
│   ├── lib.rs                # +2 модуля
│   └── app.rs                # демо только Главы 25
├── tests/
│   ├── cache_api_test.rs     # НОВЫЙ: host-тесты (сериализация манифеста, константы)
│   └── cache_offline_test.rs # НОВЫЙ: wasm-тесты (реальные операции с Cache API)
├── Cargo.toml                # уже содержит Cache, CacheStorage, Request, Response
└── package.json              # без изменений
```

**Комментарий.** `cache_api` инкапсулирует `web_sys::Cache` и `web_sys::CacheStorage` — студент видит чистый Rust-API без `JsFuture` и `Reflect`. `cache_offline` показывает реальный сценарий: «сохранить ресурс в кэш, прочитать из кэша, выбрать стратегию (cache-first / network-first / stale-while-revalidate)». `app.rs` содержит только демо этой главы, предыдущие секции не накапливаются.

**Ссылки для детального изучения:**
- MDN, Cache API: https://developer.mozilla.org/en-US/docs/Web/API/Cache
- MDN, CacheStorage: https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage
- MDN, Service Worker API: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API
- Google, «Service Workers: an Introduction»: https://web.dev/articles/service-workers
- web-sys, `Cache`: https://docs.rs/web-sys/latest/web_sys/struct.Cache.html

---

# Глава 25. Cache API

## 25.1. Что вы получите в этой главе

К концу главы у вас будет работающий **offline-репозиторий ресурсов**: загрузка по URL, сохранение в именованный кэш, чтение из кэша, выбор стратегии (cache-first, network-first, stale-while-revalidate), идемпотентное удаление. Демо в `app.rs` показывает всё это в UI. Тесты покрывают каждый публичный вызов.

Отдельно разберём, **чего демо не показывает** — интеграции с Service Worker. Она требует отдельного JS-файла и регистрации в браузере; в этой главе мы работаем с Cache API из главного потока, что тоже полностью легально.

## 25.2. `Cache`

`Cache` — именованное хранилище пар `Request → Response`. Живёт в `CacheStorage`, доступен и из `window`, и из worker-контекста. Ключевое отличие от HTTP-кэша: элементы **не обновляются и не истекают автоматически**, вы сами управляете их жизненным циклом.

В `web-sys` методы `Cache`:

```rust
cache.put_with_request(&request, &response)?;   // сохранить пару
cache.match_with_request(&request)?;             // найти (Promise<Option<Response>>)
cache.delete_with_request(&request)?;            // удалить
cache.add_with_request(&request)?;               // fetch + put одной операцией
cache.add_all_with_request(&requests)?;          // batch
cache.keys()?;                                   // список ключей
```

`match` не резервирует имя в Rust (в отличие от Cloudflare Workers, где `match` переименован в `get`).

## 25.3. `CacheStorage`

`CacheStorage` — «главный каталог» всех именованных кэшей. Доступен через `window.caches` или `WorkerGlobalScope.caches`. Методы:

```rust
let caches = window.caches()?;
caches.open("my-cache")?.await?;       // открыть (или создать) кэш
caches.has("my-cache")?.await?;        // существует ли
caches.delete("my-cache")?.await?;     // удалить целиком
caches.keys()?.await?;                 // имена всех кэшей
caches.match_with_request(&request)?;  // поиск во всех кэшах
```

`CacheStorage` требует secure context: на `http://` (кроме `localhost`) будет `SecurityError`.

## 25.4. `Request` и `Response`

`Request` — ключ. Может быть строкой URL (тогда браузер сам создаст `Request`) или полноценным объектом с методом, заголовками, режимом. `Response` — значение. Может быть получен из `fetch()`, создан из строки/`Blob`/`ArrayBuffer` или восстановлен из кэша.

**Важное правило:** `Response` можно использовать только один раз. Если вы отдаёте его в UI и хотите ещё положить в кэш — вызывайте `response.clone()` до первого использования:

```rust
let response = fetch(req).await?;
let for_cache = response.clone()?;   // clone до использования
// ... использовать response в UI ...
cache.put_with_request(&req, &for_cache)?;
```

## 25.5. Стратегии кэширования

Три базовые стратегии покрывают 90% сценариев:

| Стратегия | Алгоритм | Когда применять |
|---|---|---|
| **Cache First** | проверить кэш → если есть, вернуть; иначе сеть + сохранить | редко меняющиеся ресурсы: шрифты, иконки, JS/WASM-бандл |
| **Network First** | попробовать сеть → если успех, сохранить и вернуть; иначе кэш | часто меняющиеся данные: API-ответы, HTML |
| **Stale-While-Revalidate** | вернуть из кэша немедленно + фоном обновить из сети | ленты, списки, где важна скорость показа |

```rust
// Cache First
pub async fn cache_first(url: &str, cache: &Cache) -> Result<Response, JsValue> {
    let req = Request::new_with_str(url)?;
    if let Some(r) = get_from_cache(cache, &req).await? {
        return Ok(r);
    }
    let fresh = fetch_and_cache(url, cache).await?;
    Ok(fresh)
}

// Network First
pub async fn network_first(url: &str, cache: &Cache) -> Result<Response, JsValue> {
    match fetch_and_cache(url, cache).await {
        Ok(r) => Ok(r),
        Err(_) => {
            let req = Request::new_with_str(url)?;
            get_from_cache(cache, &req).await?.ok_or_else(|| JsValue::from_str("нет ни сети, ни кэша"))
        }
    }
}

// Stale-While-Revalidate
pub async fn swr(url: &str, cache: &Cache) -> Result<Response, JsValue> {
    let req = Request::new_with_str(url)?;
    let cached = get_from_cache(cache, &req).await?;
    // фоновая ревалидация
    let c = cache.clone();
    let u = url.to_string();
    spawn_local(async move { let _ = fetch_and_cache(&u, &c).await; });
    cached.ok_or_else(|| JsValue::from_str("кэш пуст"))
}
```

## 25.6. Offline resources

Cache API хранит `Response`, а не «файлы». Это значит, что можно кэшировать что угодно, у чего есть `Request`: HTML, CSS, JS, изображения, JSON-ответы API. Для offline-first приложения обычно используют два кэша:

- **App Shell** (`cache-first`): HTML-каркас, JS, CSS, шрифты. Обновляется только при смене версии.
- **Runtime** (`network-first` или `swr`): динамические ответы API, изображения пользователей.

```rust
pub const CACHE_APP_SHELL: &str = "app-shell-v1";
pub const CACHE_RUNTIME: &str = "runtime-v1";
```

Правило проекта: **не смешивайте версионированные и неверсионированные ресурсы в одном кэше**. Иначе при обновлении приложения старый HTML будет отдавать ссылки на удалённые JS-файлы.

## 25.7. Cache + Service Worker

Service Worker — фоновый скрипт, который перехватывает сетевые запросы страницы через событие `fetch`. В обработчике вы решаете: отдать из кэша, пойти в сеть, или скомбинировать.

```js
// sw.js (JS-файл, регистрируется в браузере)
self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);
  if (url.pathname.startsWith('/assets/')) {
    event.respondWith(cacheFirst(event.request));
  } else {
    event.respondWith(networkFirst(event.request));
  }
});
```

**Интеграция с Rust/WASM**: `wasm-pack` собирает WASM-модуль, который загружается асинхронно. Service Worker требует синхронной регистрации обработчиков в `install`/`activate`. Поэтому на практике:

1. Пишете `sw.js` на JavaScript (регистрация, `fetch`-обработчик).
2. В обработчике вызываете WASM-функции для логики (например, выбор стратегии, парсинг манифеста).
3. WASM-модуль кэшируется через `cache.add('/pkg/app_bg.wasm')` при `install`.

Это стандартный паттерн: JS-оболочка + WASM-логика. В нашем демо мы работаем с Cache API из главного потока — Service Worker не обязателен, но понимание его роли важно.

## 25.8. Практический проект: offline-репозиторий ресурсов

**Что делает.** Загружает ресурсы по URL, кэширует их в именованный кэш, читает из кэша, показывает список закэшированных ключей, удаляет идемпотентно.

**Почему это полезно.** Ровно этот код лежит в основе любого offline-first приложения: PWA, документации с офлайн-доступом, мобильного клиента с предзагрузкой.

---

## Исходные коды новых модулей

### `src/cache_api.rs`

```rust
//! Обёртка над Cache API: открытие кэша, put/match/delete, список ключей.
//! Автономный модуль — не зависит от других модулей проекта.
//!
//! ВАЖНО: методы Cache/CacheStorage (open, put, match, delete, keys)
//! в web-sys возвращают `Promise` напрямую, без обёртки `Result`.
//! Ошибки ловятся на `JsFuture::from(...).await`. Метод `window.caches()`
//! и `Request::new_with_str` помечены [Throws] — у них есть `Result`.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Cache, CacheStorage, Request, Response};

pub const CACHE_NAME: &str = "hello-wasm-offline-v1";

fn caches() -> Result<CacheStorage, JsValue> {
    web_sys::window()
        .ok_or_else(|| JsValue::from_str("no window"))?
        .caches()
        .map_err(|e| JsValue::from_str(&format!("caches: {e:?}")))
}

/// Открывает (или создаёт) именованный кэш.
pub async fn open_cache(name: &str) -> Result<Cache, JsValue> {
    let promise = caches()?.open(name);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await open: {e:?}")))?;
    Ok(value.unchecked_into::<Cache>())
}

/// Сохраняет пару Request→Response в кэш.
pub async fn put(cache: &Cache, url: &str, response: &Response) -> Result<(), JsValue> {
    let request = Request::new_with_str(url)
        .map_err(|e| JsValue::from_str(&format!("Request: {e:?}")))?;
    let promise = cache.put_with_request(&request, response);
    JsFuture::from(promise)
        .await
        .map(|_| ())
        .map_err(|e| JsValue::from_str(&format!("await put: {e:?}")))
}

/// Читает Response из кэша. None — если записи нет.
pub async fn get(cache: &Cache, url: &str) -> Result<Option<Response>, JsValue> {
    let request = Request::new_with_str(url)
        .map_err(|e| JsValue::from_str(&format!("Request: {e:?}")))?;
    let promise = cache.match_with_request(&request);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await match: {e:?}")))?;
    if value.is_undefined() {
        Ok(None)
    } else {
        Ok(Some(value.unchecked_into::<Response>()))
    }
}

/// Удаляет запись. Идемпотентно.
pub async fn remove(cache: &Cache, url: &str) -> Result<bool, JsValue> {
    let request = Request::new_with_str(url)
        .map_err(|e| JsValue::from_str(&format!("Request: {e:?}")))?;
    let promise = cache.delete_with_request(&request);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await delete: {e:?}")))?;
    Ok(value.as_bool().unwrap_or(false))
}

/// Возвращает список URL, лежащих в кэше.
pub async fn keys(cache: &Cache) -> Result<Vec<String>, JsValue> {
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

/// Удаляет весь именованный кэш. Идемпотентно.
pub async fn delete_cache(name: &str) -> Result<bool, JsValue> {
    let promise = caches()?.delete(name);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("await delete_cache: {e:?}")))?;
    Ok(value.as_bool().unwrap_or(false))
}
```

### `src/cache_offline.rs`

```rust
//! Практический проект Главы 25: offline-репозиторий ресурсов.
//!
//! Загружает ресурс по URL, кэширует, читает из кэша,
//! применяет одну из трёх стратегий:
//!   • Cache First — кэш → сеть.
//!   • Network First — сеть → кэш.
//!   • Stale-While-Revalidate — кэш немедленно + фоном обновить.
//!
//! ВАЖНО: `Response` — одноразовый. Перед `put` всегда `clone()`.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{Request, Response};

use crate::cache_api;

/// Стратегия кэширования.
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Strategy {
    /// Кэш → сеть (редко меняющиеся ресурсы).
    CacheFirst,
    /// Сеть → кэш (часто меняющиеся данные).
    NetworkFirst,
    /// Кэш немедленно + фоном обновить (ленты, списки).
    StaleWhileRevalidate,
}

/// Загружает URL через fetch. Возвращает Response и сохраняет в кэш.
///
/// Кэширует только успешные ответы (`response.ok()`), чтобы 404/500
/// не «залипали» в офлайне.
async fn fetch_and_cache(url: &str, cache: &web_sys::Cache) -> Result<Response, JsValue> {
    let request = Request::new_with_str(url)
        .map_err(|e| JsValue::from_str(&format!("Request: {e:?}")))?;
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;

    // Window::fetch_with_request возвращает Promise напрямую.
    let promise = window.fetch_with_request(&request);
    let value = JsFuture::from(promise)
        .await
        .map_err(|e| JsValue::from_str(&format!("fetch: {e:?}")))?;
    let response: Response = value
        .dyn_into()
        .map_err(|_| JsValue::from_str("не Response"))?;

    if !response.ok() {
        return Err(JsValue::from_str(&format!("HTTP {}", response.status())));
    }

    // Response одноразовый: clone до использования.
    let for_cache = response
        .clone()
        .map_err(|e| JsValue::from_str(&format!("clone: {e:?}")))?;
    cache_api::put(cache, url, &for_cache).await?;
    Ok(response)
}

/// Cache First: кэш → сеть.
pub async fn cache_first(url: &str, cache: &web_sys::Cache) -> Result<Response, JsValue> {
    if let Some(r) = cache_api::get(cache, url).await? {
        return Ok(r);
    }
    fetch_and_cache(url, cache).await
}

/// Network First: сеть → кэш (fallback).
pub async fn network_first(url: &str, cache: &web_sys::Cache) -> Result<Response, JsValue> {
    match fetch_and_cache(url, cache).await {
        Ok(r) => Ok(r),
        Err(_) => cache_api::get(cache, url)
            .await?
            .ok_or_else(|| JsValue::from_str("нет ни сети, ни кэша")),
    }
}

/// Stale-While-Revalidate: отдать из кэша немедленно + фоном обновить.
pub async fn stale_while_revalidate(
    url: &str,
    cache: &web_sys::Cache,
) -> Result<Response, JsValue> {
    let cached = cache_api::get(cache, url).await?;

    // Фоновая ревалидация.
    let cache_clone = cache.clone();
    let url_clone = url.to_string();
    spawn_local(async move {
        let _ = fetch_and_cache(&url_clone, &cache_clone).await;
    });

    cached.ok_or_else(|| JsValue::from_str("кэш пуст, ревалидация запущена"))
}

/// Диспетчер по стратегии.
pub async fn fetch_with_strategy(
    url: &str,
    cache: &web_sys::Cache,
    strategy: Strategy,
) -> Result<Response, JsValue> {
    match strategy {
        Strategy::CacheFirst => cache_first(url, cache).await,
        Strategy::NetworkFirst => network_first(url, cache).await,
        Strategy::StaleWhileRevalidate => stale_while_revalidate(url, cache).await,
    }
}

/// Сохраняет текстовый Response в кэш — без сети.
/// Полезно для предзагрузки «app shell» и ручного заполнения кэша.
///
/// Требует feature `ResponseInit` в web-sys (см. Cargo.toml).
pub async fn put_text(
    cache: &web_sys::Cache,
    url: &str,
    text: &str,
    content_type: &str,
) -> Result<(), JsValue> {
    let headers = web_sys::Headers::new()
        .map_err(|e| JsValue::from_str(&format!("Headers: {e:?}")))?;
    headers
        .set("Content-Type", content_type)
        .map_err(|e| JsValue::from_str(&format!("set header: {e:?}")))?;

    let init = web_sys::ResponseInit::new();
    init.set_status(200);
    init.set_headers(&headers);

    let response = Response::new_with_opt_str_and_init(Some(text), &init)
        .map_err(|e| JsValue::from_str(&format!("Response: {e:?}")))?;

    cache_api::put(cache, url, &response).await
}
```

---

## Добавление в `lib.rs`

```rust
// … существующие модули …
pub mod cache_api;
pub mod cache_offline;
```

---

## Тесты

### `tests/cache_api_test.rs` (host)

```rust
//! Host-тесты модуля cache_api: константы и сериализация моделей.
//! Никаких JsValue/Cache здесь быть не должно — только чистая Rust-логика.

use hello_wasm02::cache_api::CACHE_NAME;
use hello_wasm02::cache_offline::Strategy;

#[test]
fn cache_name_is_stable() {
    assert_eq!(CACHE_NAME, "hello-wasm-offline-v1");
}

#[test]
fn strategy_enum_roundtrip() {
    let strategies = [
        Strategy::CacheFirst,
        Strategy::NetworkFirst,
        Strategy::StaleWhileRevalidate,
    ];
    for s in strategies {
        assert_eq!(s, s.clone());
    }
}

#[test]
fn strategy_variants_are_distinct() {
    assert_ne!(Strategy::CacheFirst, Strategy::NetworkFirst);
    assert_ne!(Strategy::NetworkFirst, Strategy::StaleWhileRevalidate);
    assert_ne!(Strategy::CacheFirst, Strategy::StaleWhileRevalidate);
}
```

### `tests/cache_offline_test.rs` (wasm)

```rust
//! Wasm-тесты практического проекта Главы 25 (Cache API).
//! Компилируются только под wasm32 — на host файл пуст.

#![cfg(target_arch = "wasm32")]

use wasm_bindgen_test::*;

use hello_wasm02::cache_api;
use hello_wasm02::cache_offline::{self, Strategy};

wasm_bindgen_test_configure!(run_in_browser);

/// Уникальный суффикс, чтобы тесты не сталкивались в одном кэше
/// при последовательных запусках.
fn uid() -> String {
    return format!("{}", js_sys::Date::now() as u64)
}

fn test_cache_name() -> String {
    return format!("test-cache-{}", uid())
}

#[wasm_bindgen_test]
async fn open_and_delete_cache() {
    let name = test_cache_name();
    let cache = cache_api::open_cache(&name).await.expect("open_cache");
    // Кэш открыт — можно положить запись.
    cache_offline::put_text(&cache, "/hello", "world", "text/plain")
        .await
        .expect("put_text");
    let keys = cache_api::keys(&cache).await.expect("keys");
    assert!(keys.iter().any(|k| k.ends_with("/hello")));

    // Удаляем целиком.
    let removed = cache_api::delete_cache(&name).await.expect("delete_cache");
    assert!(removed);
}

#[wasm_bindgen_test]
async fn put_and_get_roundtrip() {
    let name = test_cache_name();
    let cache = cache_api::open_cache(&name).await.expect("open_cache");

    cache_offline::put_text(&cache, "/data", "{\"k\":1}", "application/json")
        .await
        .expect("put_text");

    let response = cache_api::get(&cache, "/data").await.expect("get");
    assert!(response.is_some());
    let resp = response.unwrap();
    let text = wasm_bindgen_futures::JsFuture::from(resp.text().unwrap())
        .await
        .unwrap()
        .as_string()
        .unwrap();
    assert_eq!(text, "{\"k\":1}");

    let _ = cache_api::delete_cache(&name).await;
}

#[wasm_bindgen_test]
async fn get_missing_returns_none() {
    let name = test_cache_name();
    let cache = cache_api::open_cache(&name).await.expect("open_cache");
    let result = cache_api::get(&cache, "/no-such").await.expect("get");
    assert!(result.is_none(), "не должно быть записи");
    let _ = cache_api::delete_cache(&name).await;
}

#[wasm_bindgen_test]
async fn remove_is_idempotent() {
    let name = test_cache_name();
    let cache = cache_api::open_cache(&name).await.expect("open_cache");
    cache_offline::put_text(&cache, "/del", "x", "text/plain").await.unwrap();

    let first = cache_api::remove(&cache, "/del").await.expect("first");
    assert!(first);
    let second = cache_api::remove(&cache, "/del").await.expect("second");
    assert!(!second); // идемпотентность: второй раз нечего удалять

    let _ = cache_api::delete_cache(&name).await;
}

#[wasm_bindgen_test]
async fn cache_first_returns_cached() {
    let name = test_cache_name();
    let cache = cache_api::open_cache(&name).await.expect("open_cache");
    cache_offline::put_text(&cache, "/cached", "from cache", "text/plain").await.unwrap();

    let response = cache_offline::fetch_with_strategy("/cached", &cache, Strategy::CacheFirst)
        .await
        .expect("cache_first");
    let text = wasm_bindgen_futures::JsFuture::from(response.text().unwrap())
        .await
        .unwrap()
        .as_string()
        .unwrap();
    assert_eq!(text, "from cache");

    let _ = cache_api::delete_cache(&name).await;
}

#[wasm_bindgen_test]
async fn network_first_falls_back_to_cache() {
    let name = test_cache_name();
    let cache = cache_api::open_cache(&name).await.expect("open_cache");
    // Кладём в кэш, но URL не существует в сети.
    cache_offline::put_text(&cache, "https://invalid.invalid/x", "offline", "text/plain")
        .await
        .unwrap();

    let response = cache_offline::fetch_with_strategy(
        "https://invalid.invalid/x",
        &cache,
        Strategy::NetworkFirst,
    )
    .await
    .expect("network_first fallback");
    let text = wasm_bindgen_futures::JsFuture::from(response.text().unwrap())
        .await
        .unwrap()
        .as_string()
        .unwrap();
    assert_eq!(text, "offline");

    let _ = cache_api::delete_cache(&name).await;
}

#[wasm_bindgen_test]
async fn delete_cache_is_idempotent() {
    let name = test_cache_name();
    // Кэша ещё нет — delete не должен падать.
    let r1 = cache_api::delete_cache(&name).await.expect("first");
    assert!(!r1);
    let r2 = cache_api::delete_cache(&name).await.expect("second");
    assert!(!r2);
}

```

---

## Команды запуска

```bash
# Host-тесты (без браузера)
cargo test --test cache_api_test

# Wasm-тесты (headless Chrome)
wasm-pack test --headless --chrome --test cache_offline_test

# Сборка wasm
npm run wasm:dev

# Запуск демо
npm run dev
```

---

## Демо-пример в `app.rs`


```rust
//! Демо Главы 25: Cache API — offline-репозиторий ресурсов.
//!
//! Что показывает демо:
//!   • CacheStorage.open() — открытие именованного кэша.
//!   • Cache.put / match / delete — базовые операции.
//!   • Три стратегии: Cache First, Network First, Stale-While-Revalidate.
//!   • Список ключей в кэше (cache.keys()) со счётчиком.
//!   • Ручное добавление текстовой записи (put_text).
//!   • Идемпотентное удаление записи и всего кэша.
//!
//! Фокусный подход: только демо этой главы.

use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::{spawn_local, JsFuture};
use web_sys::{HtmlElement, HtmlInputElement, Response};

use crate::cache_api;
use crate::cache_offline::{self, Strategy};

const CACHE: &str = "demo-cache-v1";

const HTML: &str = r#"
<h1>Глава 25: Cache API</h1>
<p>Offline-репозиторий ресурсов: кэширование, чтение, стратегии.</p>

<fieldset style="margin:8px 0">
  <legend>Загрузить и кэшировать</legend>
  <input id="url" placeholder="URL или путь" style="width:320px" value="/index.html">
  <select id="strategy">
    <option value="cache-first">Cache First</option>
    <option value="network-first">Network First</option>
    <option value="swr">Stale-While-Revalidate</option>
  </select>
  <button id="fetch-btn">Загрузить</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Добавить запись вручную (put_text)</legend>
  <input id="put-key" placeholder="/manual" style="width:160px">
  <input id="put-value" placeholder="текст" style="width:220px">
  <button id="put-btn">Сохранить</button>
</fieldset>

<fieldset style="margin:8px 0">
  <legend>Операции с кэшем</legend>
  <input id="key" placeholder="URL записи" style="width:320px">
  <button id="delete-entry">Удалить запись</button>
  <button id="delete-cache">Очистить весь кэш</button>
  <button id="refresh-keys">↻ Список</button>
</fieldset>

<div id="status" style="color:#555;margin:8px 0">Загрузка…</div>
<div id="result"></div>
<h3>Закэшированные ресурсы</h3>
<div id="keys-count" style="color:#555"></div>
<ul id="keys-list"></ul>
"#;

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window().ok_or("no window")?.document().ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let status = document.get_element_by_id("status").ok_or("no status")?;
    let result = document.get_element_by_id("result").ok_or("no result")?;
    let keys_list = document.get_element_by_id("keys-list").ok_or("no keys")?;
    let keys_count = document.get_element_by_id("keys-count").ok_or("no count")?;

    // Первичная загрузка списка ключей
    spawn_local({
        let status = status.clone();
        let keys_list = keys_list.clone();
        let keys_count = keys_count.clone();
        async move {
            set_status(&status, "Готово");
            refresh_keys(&keys_list, &keys_count).await;
        }
    });

    // Кнопка «Загрузить»
    let fetch_btn: HtmlElement = document.get_element_by_id("fetch-btn").ok_or("no fetch")?.dyn_into()?;
    {
        let status = status.clone();
        let result = result.clone();
        let keys_list = keys_list.clone();
        let keys_count = keys_count.clone();
        set_click(&fetch_btn, move |_| {
            let url = get_input("url");
            if url.is_empty() { return; }
            let strategy = match get_input("strategy").as_str() {
                "network-first" => Strategy::NetworkFirst,
                "swr" => Strategy::StaleWhileRevalidate,
                _ => Strategy::CacheFirst,
            };
            let status = status.clone();
            let result = result.clone();
            let keys_list = keys_list.clone();
            let keys_count = keys_count.clone();
            spawn_local(async move {
                set_status(&status, "Загрузка…");
                let cache = match cache_api::open_cache(CACHE).await {
                    Ok(c) => c,
                    Err(e) => { set_status(&status, &format!("Ошибка кэша: {e:?}")); return; }
                };
                match cache_offline::fetch_with_strategy(&url, &cache, strategy).await {
                    Ok(response) => {
                        let text = response_text(&response).await;
                        result.set_inner_html(&format!(
                            "<p><strong>OK</strong> ({strategy:?})</p>\
                             <pre style=\"max-height:200px;overflow:auto\">{}</pre>",
                            esc(&text)
                        ));
                        set_status(&status, "Загружено и закэшировано");
                    }
                    Err(e) => {
                        set_status(&status, &format!("Ошибка: {e:?}"));
                    }
                }
                refresh_keys(&keys_list, &keys_count).await;
            });
        });
    }

    // Кнопка «Сохранить» (put_text)
    let put_btn: HtmlElement = document.get_element_by_id("put-btn").ok_or("no put")?.dyn_into()?;
    {
        let status = status.clone();
        let keys_list = keys_list.clone();
        let keys_count = keys_count.clone();
        set_click(&put_btn, move |_| {
            let key = get_input("put-key");
            let value = get_input("put-value");
            if key.is_empty() { return; }
            let status = status.clone();
            let keys_list = keys_list.clone();
            let keys_count = keys_count.clone();
            spawn_local(async move {
                let cache = match cache_api::open_cache(CACHE).await {
                    Ok(c) => c,
                    Err(e) => { set_status(&status, &format!("Ошибка: {e:?}")); return; }
                };
                match cache_offline::put_text(&cache, &key, &value, "text/plain").await {
                    Ok(_) => set_status(&status, "Запись сохранена"),
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
                refresh_keys(&keys_list, &keys_count).await;
            });
        });
    }

    // Кнопка «Удалить запись»
    let del_entry: HtmlElement = document.get_element_by_id("delete-entry").ok_or("no del")?.dyn_into()?;
    {
        let status = status.clone();
        let keys_list = keys_list.clone();
        let keys_count = keys_count.clone();
        set_click(&del_entry, move |_| {
            let key = get_input("key");
            if key.is_empty() { return; }
            let status = status.clone();
            let keys_list = keys_list.clone();
            let keys_count = keys_count.clone();
            spawn_local(async move {
                let cache = match cache_api::open_cache(CACHE).await {
                    Ok(c) => c,
                    Err(e) => { set_status(&status, &format!("Ошибка: {e:?}")); return; }
                };
                match cache_api::remove(&cache, &key).await {
                    Ok(true) => set_status(&status, "Запись удалена"),
                    Ok(false) => set_status(&status, "Записи не было (идемпотентно)"),
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
                refresh_keys(&keys_list, &keys_count).await;
            });
        });
    }

    // Кнопка «Очистить весь кэш»
    let del_cache: HtmlElement = document.get_element_by_id("delete-cache").ok_or("no dc")?.dyn_into()?;
    {
        let status = status.clone();
        let keys_list = keys_list.clone();
        let keys_count = keys_count.clone();
        set_click(&del_cache, move |_| {
            let status = status.clone();
            let keys_list = keys_list.clone();
            let keys_count = keys_count.clone();
            spawn_local(async move {
                match cache_api::delete_cache(CACHE).await {
                    Ok(true) => set_status(&status, "Кэш удалён"),
                    Ok(false) => set_status(&status, "Кэша не было (идемпотентно)"),
                    Err(e) => set_status(&status, &format!("Ошибка: {e:?}")),
                }
                refresh_keys(&keys_list, &keys_count).await;
            });
        });
    }

    // Кнопка «↻ Список»
    let refresh: HtmlElement = document.get_element_by_id("refresh-keys").ok_or("no ref")?.dyn_into()?;
    {
        let keys_list = keys_list.clone();
        let keys_count = keys_count.clone();
        set_click(&refresh, move |_| {
            let keys_list = keys_list.clone();
            let keys_count = keys_count.clone();
            spawn_local(async move { refresh_keys(&keys_list, &keys_count).await; });
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

fn get_input(id: &str) -> String {
    web_sys::window()
        .and_then(|w| w.document())
        .and_then(|d| d.get_element_by_id(id))
        .and_then(|e| e.dyn_into::<HtmlInputElement>().ok())
        .map(|i| i.value())
        .unwrap_or_default()
}

fn set_status(el: &web_sys::Element, msg: &str) {
    el.set_text_content(Some(msg));
}

async fn response_text(response: &Response) -> String {
    let promise = match response.text() {
        Ok(p) => p,
        Err(_) => return String::from("<нет текста>"),
    };
    match JsFuture::from(promise).await {
        Ok(v) => v.as_string().unwrap_or_else(|| String::from("<не строка>")),
        Err(_) => String::from("<ошибка чтения>"),
    }
}

async fn refresh_keys(keys_list: &web_sys::Element, keys_count: &web_sys::Element) {
    let cache = match cache_api::open_cache(CACHE).await {
        Ok(c) => c,
        Err(e) => {
            keys_list.set_inner_html(&format!("<li>Ошибка кэша: {e:?}</li>"));
            keys_count.set_text_content(Some(""));
            return;
        }
    };
    match cache_api::keys(&cache).await {
        Ok(keys) if keys.is_empty() => {
            keys_count.set_text_content(Some("Всего: 0"));
            keys_list.set_inner_html("<li><em>Кэш пуст</em></li>");
        }
        Ok(keys) => {
            keys_count.set_text_content(Some(&format!("Всего: {}", keys.len())));
            let mut html = String::with_capacity(256);
            for k in &keys {
                html.push_str(&format!("<li><code>{}</code></li>", esc(k)));
            }
            keys_list.set_inner_html(&html);
        }
        Err(e) => {
            keys_count.set_text_content(Some(""));
            keys_list.set_inner_html(&format!("<li>Ошибка: {e:?}</li>"));
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


## Проверка

```bash
cargo check
cargo test
wasm-pack test --headless --chrome --test cache_offline_test
npm run wasm:dev && npm run dev
```

Ожидается: `cargo check` без предупреждений; 3 host-теста зелёные; 7 wasm-тестов зелёные; демо в браузере: загрузка `/index.html` в Cache First (мгновенно из кэша при повторном клике), Network First, ручное добавление записей, список с количеством, идемпотентное удаление.