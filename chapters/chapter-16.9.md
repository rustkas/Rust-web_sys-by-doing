# Глава 16.9. Типы ответов и binary data

## Обновлённая структура проекта

```
.
├── auth-server/
│   └── src/
│       ├── main.rs                 ← ОБНОВЛЁН: +/api/image, /api/binary
│       └── handlers.rs             ← ОБНОВЛЁН: +image, binary
│
├── src/
│   ├── fetch.rs                    ← ОБНОВЛЁН: +get_array_buffer, get_bytes, get_blob,
│   │                                  get_form_data, download_url, parse_data_url,
│   │                                  download_data_url
│   ├── app.rs                      ← ОБНОВЛЁН: секция 9 «Типы ответов»
│   └── (остальные без изменений)
│
├── tests/
│   ├── wasm_response_types.rs      ← НОВЫЙ: 12 тестов типов ответов
│   └── (остальные без изменений)
│
├── Cargo.toml                      ← ОБНОВЛЁН: +base64, +фичи web-sys
└── (корневые файлы без изменений)
```

**Краткий комментарий.** До сих пор мы работали с JSON и текстом. Но `fetch` умеет гораздо больше: `arrayBuffer()` даёт доступ к сырым байтам, `blob()` — к файлам и изображениям, `formData()` — к multipart-данным. В этой главе мы добавим функции для каждого из этих типов, научимся превращать `ArrayBuffer` в `Vec<u8>` через `Uint8Array`, а `Blob` — в ссылку для скачивания. Также разберём два особых случая: **blob-URL из чужого origin** (который браузер запрещает скачивать) и **data-URL** (в котором данные встроены прямо в строку и доступны всегда).

## Ссылки для детального изучения

- **MDN: `Response.arrayBuffer()`** — получение сырых байтов: [developer.mozilla.org/ru/docs/Web/API/Response/arrayBuffer](https://developer.mozilla.org/ru/docs/Web/API/Response/arrayBuffer)
- **MDN: `Response.blob()`** — получение `Blob` (файлы, изображения): [developer.mozilla.org/ru/docs/Web/API/Response/blob](https://developer.mozilla.org/ru/docs/Web/API/Response/blob)
- **MDN: `Response.formData()`** — получение `FormData`: [developer.mozilla.org/ru/docs/Web/API/Response/formData](https://developer.mozilla.org/ru/docs/Web/API/Response/formData)
- **MDN: `Blob`** — интерфейс бинарного объекта: [developer.mozilla.org/ru/docs/Web/API/Blob](https://developer.mozilla.org/ru/docs/Web/API/Blob)
- **MDN: `URL.createObjectURL()`** — создание ссылки на `Blob`: [developer.mozilla.org/ru/docs/Web/API/URL/createObjectURL](https://developer.mozilla.org/ru/docs/Web/API/URL/createObjectURL)
- **MDN: `data:` URL** — встраивание данных в URL: [developer.mozilla.org/ru/docs/Web/HTTP/Basics_of_HTTP/Data_URLs](https://developer.mozilla.org/ru/docs/Web/HTTP/Basics_of_HTTP/Data_URLs)
- **MDN: `MediaSource`** — почему blob-URL может быть потоком, а не файлом: [developer.mozilla.org/ru/docs/Web/API/MediaSource](https://developer.mozilla.org/ru/docs/Web/API/MediaSource)
- **`js_sys::Uint8Array`** — Rust-биндинг для типизированного массива: [docs.rs/js-sys/latest/js_sys/struct.Uint8Array.html](https://docs.rs/js-sys/latest/js_sys/struct.Uint8Array.html)
- **`web_sys::Blob`** — Rust-биндинг для Blob: [docs.rs/web-sys/latest/web_sys/struct.Blob.html](https://docs.rs/web-sys/latest/web_sys/struct.Blob.html)

## Новые и обновлённые файлы

### `Cargo.toml` (корневой, обновлён)

Добавить в `[dependencies]`:

```toml
base64 = "0.22"
```

Добавить в `features` для `web-sys`:

```toml
"Blob",
"BlobPropertyBag",
"FormData",
"Url",
"HtmlAnchorElement",
```

`ArrayBuffer` и `Uint8Array` живут в `js-sys`, их в `web-sys` добавлять не нужно — `js-sys` уже подключён.

### `auth-server/src/handlers.rs` (обновлён — добавить в конец)

```rust
use axum::response::IntoResponse;
use axum::http::header;

/// Возвращает простое изображение PNG (1x1 пиксель).
///
/// Используется для демонстрации `response.blob()`.
pub async fn image() -> impl IntoResponse {
    // Минимальный PNG 1x1 прозрачный.
    let png: Vec<u8> = vec![
        0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A, // PNG signature
        0x00, 0x00, 0x00, 0x0D, 0x49, 0x48, 0x44, 0x52, // IHDR chunk
        0x00, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00, 0x01, // width=1, height=1
        0x08, 0x06, 0x00, 0x00, 0x00, 0x1F, 0x15, 0xC4,
        0x89, 0x00, 0x00, 0x00, 0x0A, 0x49, 0x44, 0x41, // IDAT chunk
        0x54, 0x78, 0x9C, 0x63, 0x00, 0x01, 0x00, 0x00,
        0x05, 0x00, 0x01, 0x0D, 0x0A, 0x2D, 0xB4, 0x00, // IEND chunk
        0x00, 0x00, 0x00, 0x49, 0x45, 0x4E, 0x44, 0xAE,
        0x42, 0x60, 0x82,
    ];

    (
        [(header::CONTENT_TYPE, "image/png")],
        png,
    )
}

/// Возвращает бинарные данные фиксированной длины.
///
/// Используется для демонстрации `response.arrayBuffer()` и `get_bytes`.
pub async fn binary() -> impl IntoResponse {
    let bytes: Vec<u8> = (0u8..=255).collect(); // 256 байт
    (
        [(header::CONTENT_TYPE, "application/octet-stream")],
        bytes,
    )
}
```

### `auth-server/src/main.rs` (обновлён — добавить маршруты)

Внутри `Router::new()`:

```rust
.route("/api/image", get(handlers::image))
.route("/api/binary", get(handlers::binary))
```

### `src/fetch.rs` (обновлён — добавить в конец)

В начало файла добавьте импорты:

```rust
use js_sys::Promise;
use futures::stream::{self, StreamExt};
use wasm_bindgen::closure::Closure;
```

Функции для работы с типами ответов:

```rust
// ============================================================
// Типы ответов и binary data
// ============================================================

/// GET-запрос, возвращает `ArrayBuffer` (сырые байты).
///
/// `ArrayBuffer` — это низкоуровневый буфер, с которым работает
/// JavaScript. Чтобы прочитать байты в Rust, оберните его в
/// `Uint8Array` и вызовите `to_vec()`.
pub async fn get_array_buffer(url: &str) -> Result<js_sys::ArrayBuffer, FetchError> {
    let resp = send_request_classified(url).await?;
    let promise = resp.array_buffer().map_err(classify_error)?;
    let value = JsFuture::from(promise).await.map_err(classify_error)?;

    let buffer: js_sys::ArrayBuffer = value.dyn_into().map_err(|_| {
        FetchError::Parse("array_buffer result is not an ArrayBuffer".into())
    })?;

    Ok(buffer)
}

/// GET-запрос, возвращает `Vec<u8>` (сырые байты в Rust).
///
/// Внутри: `Response.arrayBuffer()` → `JsValue` → `Uint8Array` → `Vec<u8>`.
/// Это стандартный способ получить бинарные данные в Rust + WASM.
pub async fn get_bytes(url: &str) -> Result<Vec<u8>, FetchError> {
    let buffer = get_array_buffer(url).await?;
    let uint8 = js_sys::Uint8Array::new(&buffer);
    Ok(uint8.to_vec())
}

/// GET-запрос, возвращает `Blob`.
///
/// `Blob` — это объект, представляющий файл или изображение.
/// Его можно отдать в `<img>`, `<video>`, или создать ссылку
/// для скачивания через `URL.createObjectURL`.
pub async fn get_blob(url: &str) -> Result<web_sys::Blob, FetchError> {
    let resp = send_request_classified(url).await?;
    let promise = resp.blob().map_err(classify_error)?;
    let value = JsFuture::from(promise).await.map_err(classify_error)?;

    let blob: web_sys::Blob = value.dyn_into().map_err(|_| {
        FetchError::Parse("blob result is not a Blob".into())
    })?;

    Ok(blob)
}

/// GET-запрос, возвращает `FormData`.
///
/// `FormData` обычно используется для отправки данных, но `fetch`
/// умеет и читать его из ответа — если сервер вернул `multipart/form-data`.
pub async fn get_form_data(url: &str) -> Result<web_sys::FormData, FetchError> {
    let resp = send_request_classified(url).await?;
    let promise = resp.form_data().map_err(classify_error)?;
    let value = JsFuture::from(promise).await.map_err(classify_error)?;

    let form: web_sys::FormData = value.dyn_into().map_err(|_| {
        FetchError::Parse("form_data result is not a FormData".into())
    })?;

    Ok(form)
}
```

### `src/fetch.rs` — секция скачивания blob

```rust
// ============================================================
// Скачивание blob-ресурса
// ============================================================

/// Скачивает ресурс по URL и сохраняет его как файл.
///
/// Работает для blob-URL, созданных **в том же origin**, что и
/// приложение. Для blob-URL из другого origin браузер отклонит
/// запрос — это ограничение безопасности, а не ошибка кода.
///
/// **Почему blob-URL из чужого origin не скачивается.** Blob-URL —
/// это не сетевой адрес, а ссылка на объект в памяти конкретного
/// окна. Спецификация File API требует, чтобы `fetch` по blob-URL
/// выполнялся из того же origin, где blob был создан. Если blob
/// живёт в iframe `api.ortified.ws`, а мы работаем на
/// `localhost:5173` — браузер отклонит запрос.
///
/// Механика:
/// 1. `fetch` получает `Response`.
/// 2. `response.blob()` превращает тело в `Blob`.
/// 3. `URL.createObjectURL` создаёт временную ссылку на этот blob.
/// 4. Программный клик по `<a download>` запускает скачивание.
/// 5. `URL.revokeObjectURL` освобождает ссылку.
pub async fn download_url(
    url: &str,
    filename: &str,
) -> Result<(), FetchError> {
    let resp = send_request_classified(url).await?;
    let promise = resp.blob().map_err(classify_error)?;
    let value = JsFuture::from(promise).await.map_err(classify_error)?;

    let blob: web_sys::Blob = value.dyn_into().map_err(|_| {
        FetchError::Parse("blob result is not a Blob".into())
    })?;

    let object_url = web_sys::Url::create_object_url_with_blob(&blob)
        .map_err(classify_error)?;

    let document = web_sys::window()
        .and_then(|w| w.document())
        .ok_or_else(|| FetchError::Network("no document".into()))?;

    let anchor: web_sys::HtmlAnchorElement = document
        .create_element("a")
        .map_err(classify_error)?
        .dyn_into()
        .map_err(|_| FetchError::Parse("failed to cast to HtmlAnchorElement".into()))?;

    anchor.set_href(&object_url);
    anchor.set_download(filename);
    // Атрибут `style` вместо `anchor.style()` — метод `style()` доступен
    // у `HtmlElement`, а не у `HtmlAnchorElement` напрямую. Через
    // set_attribute работает надёжнее и без приведения типов.
    anchor.set_attribute("style", "display:none").ok();

    let body = document.body().ok_or_else(|| {
        FetchError::Network("no document.body".into())
    })?;
    body.append_child(&anchor).map_err(classify_error)?;

    anchor.click();

    body.remove_child(&anchor).map_err(classify_error)?;
    web_sys::Url::revoke_object_url(&object_url).map_err(classify_error)?;

    Ok(())
}
```

### `src/fetch.rs` — секция Data URL

```rust
// ============================================================
// Data URL
// ============================================================

/// Разбирает `data:`-URL и возвращает `(mime_type, bytes)`.
///
/// Формат: `data:<mime>;base64,<data>` или `data:<mime>,<data>`.
/// Поддерживаются оба варианта. Base64 декодируется крейтом `base64`,
/// URL-encoded — через простой percent-decode.
///
/// **Почему не через `fetch`.** `data:`-URL не требует сетевого
/// запроса, все данные уже в строке. `fetch` тоже сработает, но
/// тогда мы получим `Response` и будем читать его через `blob()`,
/// что избыточно.
///
/// **Отличие от blob-URL.** `data:`-URL не привязан к origin:
/// строка содержит данные целиком, а не ссылку на объект в памяти
/// другого окна. Поэтому скачать `data:`-URL можно откуда угодно.
pub fn parse_data_url(url: &str) -> Result<(String, Vec<u8>), FetchError> {
    let rest = url.strip_prefix("data:").ok_or_else(|| {
        FetchError::Parse("not a data URL".into())
    })?;

    // Разделяем метаданные и тело по первой запятой.
    let comma = rest.find(',').ok_or_else(|| {
        FetchError::Parse("data URL has no comma".into())
    })?;

    let (meta, payload) = rest.split_at(comma);
    let payload = &payload[1..]; // убираем саму запятую

    let mut parts = meta.split(';');
    let mime = parts.next().unwrap_or("text/plain").to_string();
    let is_base64 = parts.any(|p| p.trim() == "base64");

    if is_base64 {
        use base64::{engine::general_purpose::STANDARD, Engine};
        let bytes = STANDARD.decode(payload).map_err(|e| {
            FetchError::Parse(format!("base64 decode error: {e}"))
        })?;
        Ok((mime, bytes))
    } else {
        let bytes = percent_decode(payload.as_bytes());
        Ok((mime, bytes))
    }
}

/// Простой percent-decode для `data:`-URL без base64.
fn percent_decode(input: &[u8]) -> Vec<u8> {
    let mut out = Vec::with_capacity(input.len());
    let mut i = 0;
    while i < input.len() {
        if input[i] == b'%' && i + 2 < input.len() {
            let hi = (input[i + 1] as char).to_digit(16);
            let lo = (input[i + 2] as char).to_digit(16);
            if let (Some(h), Some(l)) = (hi, lo) {
                out.push((h * 16 + l) as u8);
                i += 3;
                continue;
            }
        }
        out.push(input[i]);
        i += 1;
    }
    out
}

/// Скачивает `data:`-URL как файл.
///
/// В отличие от `download_url`, здесь нет сетевого запроса:
/// данные уже в строке. Мы декодируем base64, упаковываем байты
/// в `Blob` и запускаем скачивание через `<a download>`.
///
/// Возвращает размер скачанного файла в байтах.
pub async fn download_data_url(
    url: &str,
    filename: &str,
) -> Result<usize, FetchError> {
    let (mime, bytes) = parse_data_url(url)?;

    // Создаём Uint8Array из байтов и оборачиваем в Blob.
    let uint8 = js_sys::Uint8Array::from(bytes.as_slice());
    let array = js_sys::Array::new();
    array.push(&uint8.buffer());

    let options = web_sys::BlobPropertyBag::new();
    options.set_type(&mime);

    let blob = web_sys::Blob::new_with_u8_array_sequence_and_options(
        &array,
        &options,
    )
    .map_err(classify_error)?;

    let size = bytes.len();
    let object_url = web_sys::Url::create_object_url_with_blob(&blob)
        .map_err(classify_error)?;

    let document = web_sys::window()
        .and_then(|w| w.document())
        .ok_or_else(|| FetchError::Network("no document".into()))?;

    let anchor: web_sys::HtmlAnchorElement = document
        .create_element("a")
        .map_err(classify_error)?
        .dyn_into()
        .map_err(|_| FetchError::Parse("failed to cast to HtmlAnchorElement".into()))?;

    anchor.set_href(&object_url);
    anchor.set_download(filename);
    anchor.set_attribute("style", "display:none").ok();

    let body = document.body().ok_or_else(|| {
        FetchError::Network("no document.body".into())
    })?;
    body.append_child(&anchor).map_err(classify_error)?;
    anchor.click();
    body.remove_child(&anchor).map_err(classify_error)?;
    web_sys::Url::revoke_object_url(&object_url).map_err(classify_error)?;

    Ok(size)
}
```

### `src/app.rs` — добавить секцию 9

После секции 8, перед `log!("Rust API Client — часть 2 готова")`:

```rust
    // ============================================================
    // СЕКЦИЯ 9: Типы ответов
    // ============================================================

    let section9 = build_section(&container, "9. Типы ответов")?;

    let intro9 = dom::create_paragraph(
        "Fetch умеет читать тело ответа разными способами. \
         json() — для JSON, text() — для текста, \
         arrayBuffer() — для сырых байтов, \
         blob() — для файлов и изображений, \
         formData() — для multipart. \
         ArrayBuffer превращается в Vec<u8> через Uint8Array. \
         Blob можно скачать через URL.createObjectURL. \
         data: URL — данные встроены в строку, скачивается \
         откуда угодно. blob: URL из чужого origin скачать нельзя.",
    )?;
    dom::add_class(&dom::as_html(&intro9), "hint")?;
    dom::append(&section9, &intro9)?;

    let json_btn = dom::create_element("button")?;
    json_btn.set_text_content(Some("Загрузить JSON"));
    let text_btn = dom::create_element("button")?;
    text_btn.set_text_content(Some("Загрузить текст"));
    let blob_btn = dom::create_element("button")?;
    blob_btn.set_text_content(Some("Загрузить изображение (Blob)"));
    let bytes_btn = dom::create_element("button")?;
    bytes_btn.set_text_content(Some("Загрузить байты"));
    let download_blob_btn = dom::create_element("button")?;
    download_blob_btn.set_text_content(Some("Скачать blob-ресурс"));
    let download_data_btn = dom::create_element("button")?;
    download_data_btn.set_text_content(Some("Скачать data: URL (WebM)"));

    dom::append(&section9, &json_btn)?;
    dom::append(&section9, &text_btn)?;
    dom::append(&section9, &blob_btn)?;
    dom::append(&section9, &bytes_btn)?;
    dom::append(&section9, &download_blob_btn)?;
    dom::append(&section9, &download_data_btn)?;

    let result9 = dom::create_element("div")?;
    dom::add_class(&dom::as_html(&result9), "api-result")?;
    result9.set_text_content(Some("(нажмите кнопку)"));
    dom::append(&section9, &result9)?;

    // ---- JSON ----
    {
        let result9 = result9.clone();
        callbacks::listen(
            &json_btn,
            "click",
            move |_| {
                let result9 = result9.clone();
                result9.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/1";
                    match fetch::send_request_classified(url).await {
                        Ok(resp) => {
                            let json: serde_json::Value =
                                fetch::response_json(&resp).await.unwrap();
                            let text = format!(
                                "✅ response.json()\n\n\
                                 Получен объект:\n{}",
                                json
                            );
                            result9.set_text_content(Some(&text));
                        }
                        Err(e) => result9.set_text_content(Some(&format!("❌ {:?}", e))),
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Текст ----
    {
        let result9 = result9.clone();
        callbacks::listen(
            &text_btn,
            "click",
            move |_| {
                let result9 = result9.clone();
                result9.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "https://jsonplaceholder.typicode.com/posts/1";
                    match fetch::send_request_classified(url).await {
                        Ok(resp) => {
                            let promise = resp.text().unwrap();
                            let value = wasm_bindgen_futures::JsFuture::from(promise)
                                .await
                                .unwrap();
                            let text = value.as_string().unwrap_or_default();

                            let char_count = text.chars().count();
                            let preview = truncate_preview(&text, 200);

                            let output = format!(
                                "✅ response.text()\n\n\
                                 Длина: {} символов\n\
                                 Начало:\n{}",
                                char_count, preview
                            );
                            result9.set_text_content(Some(&output));
                        }
                        Err(e) => result9.set_text_content(Some(&format!("❌ {:?}", e))),
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Blob (изображение) ----
    {
        let result9 = result9.clone();
        callbacks::listen(
            &blob_btn,
            "click",
            move |_| {
                let result9 = result9.clone();
                result9.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "http://127.0.0.1:3000/api/image";
                    match fetch::get_blob(url).await {
                        Ok(blob) => {
                            let size = blob.size();
                            let mime = blob.type_();

                            let object_url =
                                web_sys::Url::create_object_url_with_blob(&blob)
                                    .unwrap_or_default();

                            let text = format!(
                                "✅ response.blob()\n\n\
                                 Размер: {} байт\n\
                                 MIME-тип: {}\n\
                                 Object URL: {}",
                                size, mime, object_url
                            );
                            result9.set_text_content(Some(&text));
                        }
                        Err(e) => result9.set_text_content(Some(&format!("❌ {:?}", e))),
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Байты ----
    {
        let result9 = result9.clone();
        callbacks::listen(
            &bytes_btn,
            "click",
            move |_| {
                let result9 = result9.clone();
                result9.set_text_content(Some("Запрос..."));
                wasm_bindgen_futures::spawn_local(async move {
                    let url = "http://127.0.0.1:3000/api/binary";
                    match fetch::get_bytes(url).await {
                        Ok(bytes) => {
                            let preview: Vec<String> = bytes
                                .iter()
                                .take(16)
                                .map(|b| format!("{:02X}", b))
                                .collect();
                            let text = format!(
                                "✅ response.arrayBuffer() → Uint8Array → Vec<u8>\n\n\
                                 Длина: {} байт\n\
                                 Первые 16 байт (hex):\n{}",
                                bytes.len(),
                                preview.join(" ")
                            );
                            result9.set_text_content(Some(&text));
                        }
                        Err(e) => result9.set_text_content(Some(&format!("❌ {:?}", e))),
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Скачивание blob-ресурса ----
    {
        let result9 = result9.clone();
        callbacks::listen(
            &download_blob_btn,
            "click",
            move |_| {
                let result9 = result9.clone();
                result9.set_text_content(Some("Скачивание..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // ВАЖНО: если этот URL создан в origin `api.ortified.ws`,
                    // а наше приложение работает на `localhost:5173`,
                    // браузер отклонит запрос: blob-URL доступен только
                    // из того origin, который его создал. Это ожидаемое
                    // поведение, а не ошибка кода.
                    //
                    // Для рабочего примера используем blob из нашего
                    // же origin — сгенерированный из image-эндпоинта.
                    let url = "http://127.0.0.1:3000/api/image";

                    match fetch::download_url(url, "image.png").await {
                        Ok(()) => {
                            result9.set_text_content(Some(
                                "✅ Скачивание запущено. Проверьте папку загрузок.",
                            ));
                        }
                        Err(e) => {
                            result9.set_text_content(Some(&format!(
                                "❌ {:?}\n\n\
                                 Если URL — blob: из чужого origin, \
                                 ошибка ожидаема: браузер запрещает \
                                 fetch к blob-URL из другого origin.",
                                e
                            )));
                        }
                    }
                });
            },
            &closures,
        )?;
    }

    // ---- Скачивание data: URL ----
    {
        let result9 = result9.clone();
        callbacks::listen(
            &download_data_btn,
            "click",
            move |_| {
                let result9 = result9.clone();
                result9.set_text_content(Some("Декодирование..."));
                wasm_bindgen_futures::spawn_local(async move {
                    // data: URL — все данные встроены в строку.
                    // Никаких сетевых запросов, никаких origin-ограничений.
                    let url = "data:video/webm;base64,GkXfo0AgQoaBAUL3gQFC8oEEQvOBCEKCQAR3ZWJtQoeBAkKFgQIYU4BnQI0VSalmQCgq17FAAw9CQE2AQAZ3aGFtbXlXQUAGd2hhbW15RIlACECPQAAAAAAAFlSua0AxrkAu14EBY8WBAZyBACK1nEADdW5khkAFVl9WUDglhohAA1ZQOIOBAeBABrCBCLqBCB9DtnVAIueBAKNAHIEAAIAwAQCdASoIAAgAAUAmJaQAA3AA/vz0AAA=";

                    match fetch::download_data_url(url, "video.webm").await {
                        Ok(size) => {
                            result9.set_text_content(Some(&format!(
                                "✅ Скачивание запущено.\n\
                                 Размер: {} байт\n\
                                 Файл: video.webm\n\n\
                                 data: URL не требует сети — \
                                 все данные были в строке, \
                                 мы их декодировали из base64 \
                                 и упаковали в Blob.",
                                size
                            )));
                        }
                        Err(e) => {
                            result9.set_text_content(Some(&format!("❌ {:?}", e)));
                        }
                    }
                });
            },
            &closures,
        )?;
    }
```

### `tests/wasm_response_types.rs` (новый)

```rust
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

/// JSON-ответ десериализуется в serde_json::Value.
#[wasm_bindgen_test]
async fn test_get_json() {
    let url = "https://jsonplaceholder.typicode.com/posts/1";
    let resp = hello_wasm02::fetch::send_request_classified(url)
        .await
        .unwrap();
    let json: serde_json::Value = hello_wasm02::fetch::response_json(&resp)
        .await
        .unwrap();
    assert_eq!(json["id"], 1);
    assert!(json["title"].is_string());
}

/// Текстовый ответ читается как String.
#[wasm_bindgen_test]
async fn test_get_text() {
    let url = "https://jsonplaceholder.typicode.com/posts/1";
    let resp = hello_wasm02::fetch::send_request_classified(url)
        .await
        .unwrap();
    let promise = resp.text().unwrap();
    let value = wasm_bindgen_futures::JsFuture::from(promise)
        .await
        .unwrap();
    let text = value.as_string().unwrap();
    assert!(!text.is_empty());
    assert!(text.contains("userId"));
}

/// ArrayBuffer возвращается корректно.
#[wasm_bindgen_test]
async fn test_get_array_buffer() {
    let url = "http://127.0.0.1:3000/api/binary";
    let buffer = hello_wasm02::fetch::get_array_buffer(url).await.unwrap();
    assert!(buffer.byte_length() > 0);
}

/// ArrayBuffer превращается в Vec<u8>.
#[wasm_bindgen_test]
async fn test_get_bytes() {
    let url = "http://127.0.0.1:3000/api/binary";
    let bytes = hello_wasm02::fetch::get_bytes(url).await.unwrap();
    assert_eq!(bytes.len(), 256);
    assert_eq!(bytes[0], 0);
    assert_eq!(bytes[255], 255);
}

/// Blob возвращается корректно.
#[wasm_bindgen_test]
async fn test_get_blob() {
    let url = "http://127.0.0.1:3000/api/image";
    let blob = hello_wasm02::fetch::get_blob(url).await.unwrap();
    assert!(blob.size() > 0.0);
}

/// ArrayBuffer имеет правильную длину.
#[wasm_bindgen_test]
async fn test_array_buffer_length() {
    let url = "http://127.0.0.1:3000/api/binary";
    let buffer = hello_wasm02::fetch::get_array_buffer(url).await.unwrap();
    assert_eq!(buffer.byte_length(), 256);
}

/// Uint8Array создаётся из ArrayBuffer.
#[wasm_bindgen_test]
async fn test_uint8_array_from_buffer() {
    let url = "http://127.0.0.1:3000/api/binary";
    let buffer = hello_wasm02::fetch::get_array_buffer(url).await.unwrap();
    let uint8 = js_sys::Uint8Array::new(&buffer);
    assert_eq!(uint8.length(), 256);
    assert_eq!(uint8.get_index(0), 0);
    assert_eq!(uint8.get_index(255), 255);
}

/// Blob имеет правильный размер.
#[wasm_bindgen_test]
async fn test_blob_size() {
    let url = "http://127.0.0.1:3000/api/image";
    let blob = hello_wasm02::fetch::get_blob(url).await.unwrap();
    // PNG 1x1 прозрачный — примерно 70 байт.
    assert!(blob.size() > 50.0);
    assert!(blob.size() < 200.0);
}

/// Blob имеет правильный MIME-тип.
#[wasm_bindgen_test]
async fn test_blob_type() {
    let url = "http://127.0.0.1:3000/api/image";
    let blob = hello_wasm02::fetch::get_blob(url).await.unwrap();
    assert_eq!(blob.type_(), "image/png");
}

/// JSON с unicode корректно десериализуется.
#[wasm_bindgen_test]
async fn test_json_with_unicode() {
    let url = "http://127.0.0.1:3000/api/delay/0";
    let resp = hello_wasm02::fetch::send_request_classified(url)
        .await
        .unwrap();
    let json: serde_json::Value = hello_wasm02::fetch::response_json(&resp)
        .await
        .unwrap();
    let message = json["message"].as_str().unwrap();
    assert!(message.contains("секунд"));
}

/// data: URL с base64 декодируется в байты.
#[wasm_bindgen_test]
async fn test_parse_data_url_base64() {
    let url = "data:video/webm;base64,GkXfo0AgQoaBAUL3gQFC8oEEQvOBCEKCQAR3ZWJtQoeBAkKFgQIYU4BnQI0VSalmQCgq17FAAw9CQE2AQAZ3aGFtbXlXQUAGd2hhbW15RIlACECPQAAAAAAAFlSua0AxrkAu14EBY8WBAZyBACK1nEADdW5khkAFVl9WUDglhohAA1ZQOIOBAeBABrCBCLqBCB9DtnVAIueBAKNAHIEAAIAwAQCdASoIAAgAAUAmJaQAA3AA/vz0AAA=";
    let (mime, bytes) = hello_wasm02::fetch::parse_data_url(url).unwrap();
    assert_eq!(mime, "video/webm");
    assert!(bytes.len() > 100, "должно быть больше 100 байт, получено {}", bytes.len());
    // WebM начинается с сигнатуры EBML: 1A 45 DF A3.
    assert_eq!(bytes[0], 0x1A);
    assert_eq!(bytes[1], 0x45);
    assert_eq!(bytes[2], 0xDF);
    assert_eq!(bytes[3], 0xA3);
}

/// data: URL без base64 (текстовый) декодируется из percent-encoding.
#[wasm_bindgen_test]
async fn test_parse_data_url_plain() {
    let url = "data:text/plain,Hello%2C%20world!";
    let (mime, bytes) = hello_wasm02::fetch::parse_data_url(url).unwrap();
    assert_eq!(mime, "text/plain");
    assert_eq!(String::from_utf8(bytes).unwrap(), "Hello, world!");
}
```

## Команды запуска

### Разработка (три терминала)

**Терминал 1 — сервер:**

```bash
npm run server
```

После правок в `handlers.rs` и `main.rs` сервер нужно перезапустить.

**Терминал 2 — сборка WASM и dev-сервер Vite:**

```bash
npm run wasm:dev
npm run dev
```

**Терминал 3 — тесты:**

```bash
npm run test:chrome
```

Ожидаемый результат: `48 passed; 0 failed` (36 из глав 16.4–16.8 + 12 новых).

### Проверка только новых тестов

```bash
wasm-pack test --headless --chrome --test wasm_response_types
```

### Проверка компиляции

```bash
cargo check --target wasm32-unknown-unknown
cargo check --manifest-path auth-server/Cargo.toml
```

### Ручная проверка сервера

```bash
curl -i http://127.0.0.1:3000/api/image
curl -i http://127.0.0.1:3000/api/binary
```

### Ручная проверка демо

Откройте `http://localhost:5173`, прокрутите до секции «9. Типы ответов»:

| Кнопка | Ожидаемый результат |
|---|---|
| «Загрузить JSON» | Объект с полями `id`, `title`, `body`, `userId` |
| «Загрузить текст» | Длина в символах, первые 200 символов |
| «Загрузить изображение (Blob)» | Размер ~70 байт, MIME `image/png`, Object URL |
| «Загрузить байты» | 256 байт, первые 16 в hex: `00 01 02 ... 0F` |
| «Скачать blob-ресурс» | Скачивается `image.png` |
| «Скачать data: URL (WebM)» | Скачивается `video.webm` (~180 байт) |

## Про blob-URL из чужого origin: почему не скачивается

Отдельно стоит сказать о ситуации, с которой вы столкнётесь, если попробуете скачать `blob:https://api.ortified.ws/...` из своего приложения. Это **не сработает**, и вот почему.

**Blob-URL — это не файл.** Это ссылка на объект в памяти конкретного окна браузера. Спецификация File API требует, чтобы запрос по blob-URL выполнялся из того же origin, где blob был создан. Если плеер живёт в iframe `api.ortified.ws`, а ваше приложение работает на `localhost:5173`, браузер отклонит `fetch` с ошибкой CORS или `TypeError: Failed to fetch`.

Есть исключения: если blob создан внутри `<video>` через **MediaSource Extensions**, то `fetch` по этой ссылке вернёт не файл, а поток. В лучшем случае вы получите неполный blob, в худшем — пустой. Единственный способ скачать такое видео — через `MediaRecorder`, который запишет воспроизводимый поток в реальном времени, или через поиск оригинальных сегментов (`.m3u8`, `.ts`) в DevTools → Network.

**Как проверить свой blob-URL в консоли.** Откройте DevTools, переключитесь в контекст iframe с плеером, вставьте:

```js
fetch('blob:https://api.ortified.ws/...')
  .then(r => r.blob())
  .then(b => {
    console.log(`${b.size} байт, тип: ${b.type}`);
    if (b.size === 0) console.warn('Пустой blob — вероятно, это MediaSource');
  })
  .catch(e => console.error(e));
```

Если `size > 0` — blob содержит данные, и код из `download_url` сработает (при запуске в том же origin). Если `size === 0` — это MediaSource, и нужен `MediaRecorder`.

## Итог главы

Мы научились читать тело ответа в разных форматах:

1. **`response.json()`** — JSON → `JsValue` → `serde_json::Value`. Работает через `serde-wasm-bindgen`.
2. **`response.text()`** — текст → `String`. Используется, когда ответ не JSON.
3. **`response.arrayBuffer()`** — сырые байты в `js_sys::ArrayBuffer`.
4. **`response.blob()`** — `web_sys::Blob` для файлов и изображений.
5. **`response.formData()`** — `web_sys::FormData` для multipart-ответов.

Плюс две функции для скачивания:

6. **`download_url`** — для blob-URL из того же origin. Из чужого origin браузер отклонит запрос.
7. **`download_data_url`** — для data-URL. Работает всегда, потому что данные встроены в строку.

Ключевые уроки:

- **`ArrayBuffer` — низкоуровневый буфер.** Через `Uint8Array::new(&buffer).to_vec()` он превращается в `Vec<u8>` одной строкой.
- **`Blob` — это файл.** У него есть `size()` и `type_()`. Через `Url::create_object_url_with_blob(&blob)` получается ссылка для `<a download>` или `<img src>`.
- **`response.body()` — это `ReadableStream`.** Мы его не трогали, потому что он заслуживает отдельной главы. В 16.10 разберём, как читать поток по частям.
- **Один `Response` — одно чтение тела.** После `resp.json()` или `resp.text()` тело помечено использованным, повторное чтение падает с ошибкой.
- **blob-URL привязан к origin.** Скачать blob из чужого origin нельзя — это ограничение безопасности, а не баг.
- **data-URL работает везде.** Данные встроены в строку, а не в память окна.
- **`&text[..200]` паникует на UTF-8.** Используйте `chars()` и `truncate_preview`, а не байтовые срезы.
- **`style()` есть у `HtmlElement`, а не у `HtmlAnchorElement`.** Используйте `set_attribute("style", ...)` — работает надёжнее.

В следующей главе — **`ReadableStream`**, **чтение по частям**, **прогресс загрузки** и **обработка больших файлов без загрузки в память целиком**.