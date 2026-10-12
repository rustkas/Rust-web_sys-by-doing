# Часть XI. Security и Cryptography

## Глава 40. Web Crypto API

---

## Введение

Web Crypto API — встроенный в браузер криптографический движок. Он доступен через `window.crypto.subtle` и даёт ровно те примитивы, которым доверяет сам браузер: SHA-2, HMAC, AES-GCM, RSA-OAEP, ECDSA, ECDH, криптостойкий генератор случайных чисел.

Для Rust-разработчика Web Crypto — это **мост между двумя мирами**. С одной стороны, в Rust есть `RustCrypto`: явный контроль памяти, `zeroize`, никаких GC-пауз, одинаковое поведение на сервере и клиенте. С другой — Web Crypto не требует тащить в WASM-бандл криптографические реализации, использует платформенные ускорители (AES-NI, SHA-extensions) и не зависит от того, работает ли приложение в браузере, в Node.js или в Electron.

В этой главе мы покроем 14 тем:

1. `Crypto` — `window.crypto`
2. `SubtleCrypto` — `crypto.subtle`
3. `CryptoKey` — `extractable` / `usages`
4. random values — `getRandomValues`, `randomUUID`
5. hashing — SHA-256 / SHA-384 / SHA-512
6. HMAC — sign / verify
7. AES — AES-GCM 256
8. RSA — RSA-OAEP 2048
9. ECDSA — P-256 sign / verify
10. ECDH — P-256 deriveKey
11. key generation
12. import / export keys — raw, JWK
13. Rust cryptography vs Web Crypto
14. Практическое cryptographic application

---

## 1. Что мы узнали при сборке: 6 грабель Web Crypto

Прежде чем перейти к коду, зафиксируем сложности, которые выявились при разработке. Они типичны, и их стоит знать до того, как вы встретитесь с ними в реальном проекте.

### Грабли 1. `_with_object` требует `&Object`, а не `&JsValue`

В `web-sys` большинство методов `SubtleCrypto` имеют несколько вариантов. «Голые» варианты вроде `digest(&JsValue, &JsValue)` принимают `JsValue`. Варианты с суффиксом `_with_object` — принимают `&js_sys::Object`:

```rust
// Компилируется:
subtle.digest(&alg_jsvalue, &data_jsvalue);

// НЕ компилируется:
subtle.generate_key_with_object(&alg.into(), true, &usages);

// Компилируется:
subtle.generate_key_with_object(&alg, true, &usages);
```

Решение — один хелпер в `util.rs`:

```rust
pub fn bytes_to_object(bytes: &[u8]) -> js_sys::Object {
    let arr = js_sys::Uint8Array::new_with_length(bytes.len() as u32);
    arr.copy_from(bytes);
    return arr.into();
}
```

### Грабли 2. `digest_with_str_and_buffer_source` возвращает `Result<Promise, JsValue>`

Многие методы `SubtleCrypto` возвращают не голый `Promise`, а `Result<Promise, JsValue>` — потому что web-sys оборачивает нативный вызов, который может синхронно кинуть исключение (например, при неизвестном алгоритме). Нужно развернуть `Result` до `JsFuture::from`:

```rust
let promise: js_sys::Promise = subtle
    .digest_with_str_and_buffer_source(alg.as_str(), &data_obj)
    .map_err(|e| CryptoError::from(e))?;
let result = JsFuture::from(promise).await.map_err(...)?;
```

### Грабли 3. `JsCast` должен быть в scope

`.dyn_into::<web_sys::CryptoKey>()` — это метод трейта `JsCast`. Без `use wasm_bindgen::JsCast;` он «не виден», и rustc выдаёт `no method named 'dyn_into'`. Эта ошибка сбивает с толку, потому что сам `JsValue` «как будто» имеет метод. Решение: импортировать `JsCast` в каждом модуле, где вы приводите типы:

```rust
use wasm_bindgen::{JsCast, JsValue};
```

### Грабли 4. RSA-OAEP требует `publicExponent`

Обязательное поле. Без него Chrome кидает `RsaHashedKeyGenParams: publicExponent: Missing or not a Uint8Array`. Стандартное значение — F4 = 65537 = `[0x01, 0x00, 0x01]`:

```rust
const PUBLIC_EXPONENT_F4: [u8; 3] = [0x01, 0x00, 0x01];
let exponent = js_sys::Uint8Array::new_with_length(3);
exponent.copy_from(&PUBLIC_EXPONENT_F4);
Reflect::set(&alg, &"publicExponent".into(), &exponent.into())?;
```

### Грабли 5. ECDH не принимает пустой `keyUsages`

`generateKey` с алгоритмом ECDH требует хотя бы одной роли в `keyUsages`. Пустой массив вызывает `SyntaxError: Usages cannot be empty when creating a key`. Для ECDH это `deriveKey`:

```rust
let usages = js_sys::Array::new();
usages.push(&JsValue::from_str("deriveKey")); // ← обязательно
```

Публичный ключ получит `[]` автоматически (это ограничение Web Crypto: ECDH-public не может ничего, кроме `derive` в паре с приватным).

### Грабли 6. `Crypto::random_uuid()` возвращает `String`, а не `Result`

Метод синхронный и в нормальных условиях не падает. Не нужно `.map_err(...)`. В отличие от `getRandomValues`, который принимает `&mut [u8]` — а не `&Uint8Array`:

```rust
let mut out = vec![0u8; len];
let slice: &mut [u8] = out.as_mut_slice();
crypto.get_random_values_with_u8_array(slice)?;
```

---

## 2. Архитектура модуля

```
src/
├── lib.rs                    ← pub mod web_crypto
├── app.rs                    ← демо-пример главы
└── web_crypto/
    ├── mod.rs                ← CryptoError, subtle(), crypto_obj()
    ├── util.rs               ← bytes_to_object, jsvalue_to_bytes, hex_encode
    ├── digest.rs             ← SHA-256/384/512
    ├── hmac.rs               ← HMAC sign/verify
    ├── aes.rs                ← AES-GCM
    ├── rsa.rs                ← RSA-OAEP
    ├── ecdsa.rs              ← ECDSA + ECDH
    ├── random.rs             ← random_bytes, random_uuid
    └── keys.rs               ← importKey / exportKey, aes_gcm_alg
```

Каждый модуль — тонкая обёртка над `SubtleCrypto`, возвращающая `Result<T, CryptoError>`. `CryptoError` оборачивает `JsValue` из промисов в человекочитаемую строку.

---

## 3. Сквозной сценарий: Алиса → Боб

Основная идея главы — собрать 4 примитива в один реальный флоу:

1. **ECDH** — Алиса и Боб генерируют пары, обмениваются публичными ключами и независимо выводят одинаковый AES-256 ключ.
2. **AES-GCM** — Алиса шифрует сообщение общим ключом. IV — 12 байт.
3. **ECDSA** — Алиса подписывает исходный текст своим приватным ключом.
4. **Доставка** — Боб расшифровывает AES-ключом и проверяет подпись публичным ключом Алисы.

Что это даёт:

- **Конфиденциальность** — сообщение нельзя прочесть без общего ключа.
- **Целостность** — GCM-тег обнаружит любое изменение шифротекста.
- **Авторство** — ECDSA-подпись подтверждает, что текст создан Алисой.

Запуск сценария с подменой: `tampered[0] ^= 0xFF` — и Боб получает `OperationError` на шаге 4. Это **самая впечатляющая демонстрация** Web Crypto: криптосистема не «шифрует и надеется», она *доказуемо* отвергает испорченное сообщение.

---

## 4. Лавинный эффект SHA-256

Два текста, отличающиеся одним символом:

```
Текст 1: «Привет, Web Crypto!»
Текст 2: «Привет, Web Cryptp!»   ← одна буква заменена
```

SHA-256 обоих:

```
SHA-256(1): 48d8a4f1d5c0f3f9b03a4d5096bddc29f2b7fabf3a36cd2c000dc9037715f3e8
SHA-256(2): 94c8611e2f6e9e0f3fe21f8c92fa2a9e9f6c4f0ac9f5a3eb1b8f9a16c6e75e4d
```

**Совпадающих hex-символов: 2 / 64 = 3.1%.** Заменили одну букву — получили совершенно другой хеш. Это и есть лавинный эффект: свойство, которое делает хеш криптостойким. Без него по SHA-256 можно было бы «подбирать похожие» тексты.

---

## 5. Обратный импорт ключа

Мы экспортировали AES-ключ в raw (32 байта). Теперь импортируем его обратно и проверяем, что он расшифровывает шифротекст из блока 3:

```rust
let alg = keys::aes_gcm_alg();
let key = keys::import_raw(
    KeyFormat::Raw,
    &raw,
    &alg,
    true,
    &["encrypt", "decrypt"],
).await?;
let pt = aes::decrypt(&key, &ct, &iv).await?;
```

Это замыкает цикл `generate → export → import → use`. В реальных системах именно так работают «сохранение сессии» и «перенос ключа между вкладками» — через `postMessage`, `IndexedDB` или `BroadcastChannel`.

---

## 6. Rust cryptography vs Web Crypto

| Критерий | RustCrypto (`aes-gcm`, `rsa`, `p256`) | Web Crypto API |
|---|---|---|
| Размер бандла | +50–200 KB WASM | 0 (встроено в браузер) |
| Ускорение | Только если есть SIMD/instruction | AES-NI, SHA-ext, PCLMULQDQ |
| Zeroize | Явный, надёжный | Не управляется из JS |
| Контроль над RNG | `getrandom`, свой источник | Только `crypto.getRandomValues` |
| Согласованность | Одинаково на сервере и клиенте | Только в браузере / Node 20+ |
| Постоянное время | Управляемо | Не гарантируется спекой |
| Простота | Требует понимания трейтов | Простой async API |

**Правило выбора:** если нужно крипто в браузере без сервера — Web Crypto. Если есть серверная часть или нужен строгий контроль над памятью (медкарта, криптокошелёк) — RustCrypto. Часто берут **гибрид**: долгоживущие ключи и подпись — RustCrypto, быстрые операции сессии (AES-GCM) — Web Crypto.

---

## 7. Ошибки и их обработка

Каждая операция возвращает `CryptoError`, который оборачивает `JsValue`:

```rust
impl From<JsValue> for CryptoError {
    fn from(v: JsValue) -> Self {
        let msg = v.as_string()
            .unwrap_or_else(|| format_str(format_args!("{v:?}")));
        return Self { message: msg };
    }
}
```

В UI статус сразу показывает сообщение пользователю: `OperationError: The operation failed for an operation-specific reason` — это GCM, обнаруживший подмену тега. Пользователь видит не абстрактную ошибку, а конкретную причину.

---

## 8. Исходный код

### `Cargo.toml` (изменения)

```toml
# В блоке features web-sys:
    "SubtleCrypto",
    "Crypto",
    "CryptoKey",     # ← добавлено
    ...
    # Прочее
    "console",
    "Clipboard",     # ← добавлено
```

### `src/lib.rs`

```rust
//! Hello WASM02 — курс по Rust + WebAssembly.
//!
//! Глава 38: Web Crypto API.

pub mod format_str;
pub mod app;
pub mod web_crypto;
```

### `src/web_crypto/mod.rs`

```rust
//! Глава 38: Web Crypto API — браузерная криптография.
//!
//! Все модули оборачивают `web_sys::SubtleCrypto` в удобный API,
//! возвращающий `Result<T, CryptoError>` вместо разрозненных `JsValue`.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).

pub mod digest;
pub mod hmac;
pub mod aes;
pub mod rsa;
pub mod ecdsa;
pub mod random;
pub mod keys;
pub mod util;

use wasm_bindgen::JsValue;
use crate::format_str::format_str;

/// Универсальная ошибка криптоопераций.
///
/// Оборачивает `JsValue` из JS-промисов в человекочитаемую строку.
#[derive(Debug, Clone)]
pub struct CryptoError {
    pub message: String,
}

impl CryptoError {
    pub fn new(msg: String) -> Self {
        return Self { message: msg };
    }
}

impl std::fmt::Display for CryptoError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return write!(f, "{}", self.message);
    }
}

impl From<JsValue> for CryptoError {
    fn from(v: JsValue) -> Self {
        let msg = v
            .as_string()
            .unwrap_or_else(|| {
                let debug = format_str(format_args!("{v:?}"));
                debug
            });
        return Self { message: msg };
    }
}

impl From<String> for CryptoError {
    fn from(s: String) -> Self {
        return Self { message: s };
    }
}

/// Возвращает `window.crypto.subtle`.
///
/// `Crypto` доступен через `Window.crypto`, `SubtleCrypto` — через
/// `Crypto.subtle`. Если браузер не поддерживает Web Crypto (например,
/// страница открыта по HTTP), возвращаем ошибку.
pub fn subtle() -> Result<web_sys::SubtleCrypto, CryptoError> {
    let window = web_sys::window()
        .ok_or_else(|| CryptoError::new("no window".to_string()))?;
    let crypto = window.crypto()
        .map_err(|_| CryptoError::new("window.crypto unavailable".to_string()))?;
    let subtle = crypto.subtle();
    return Ok(subtle);
}

/// Возвращает `window.crypto` (для `getRandomValues`).
pub fn crypto_obj() -> Result<web_sys::Crypto, CryptoError> {
    let window = web_sys::window()
        .ok_or_else(|| CryptoError::new("no window".to_string()))?;
    return window.crypto().map_err(|e| CryptoError::from(e));
}
```

### `src/web_crypto/util.rs`

```rust
//! Общие хелперы для криптомодулей: кодирование/декодирование байтов.
//!
//! Методы web-sys с суффиксом `_with_object` принимают `&js_sys::Object`.
//! `Uint8Array` наследует `Object`, поэтому `bytes_to_object` возвращает
//! именно `Object` — этого достаточно для всех `BufferSource`-параметров.

use wasm_bindgen::JsCast;
use wasm_bindgen::JsValue;

/// Преобразует `&[u8]` в `js_sys::Object`, содержащий `Uint8Array`.
pub fn bytes_to_object(bytes: &[u8]) -> js_sys::Object {
    let arr = js_sys::Uint8Array::new_with_length(bytes.len() as u32);
    arr.copy_from(bytes);
    let obj: js_sys::Object = arr.into();
    return obj;
}

/// Извлекает `Vec<u8>` из `JsValue`, который может быть:
/// - `Uint8Array` (результат `subtle.digest`, `subtle.encrypt` и т.п.),
/// - `ArrayBuffer`.
pub fn jsvalue_to_bytes(value: &JsValue) -> Result<Vec<u8>, String> {
    if value.is_undefined() || value.is_null() {
        return Err("jsvalue_to_bytes: value is null/undefined".to_string());
    }
    if let Some(arr) = value.dyn_ref::<js_sys::Uint8Array>() {
        return Ok(arr.to_vec());
    }
    if let Some(buf) = value.dyn_ref::<js_sys::ArrayBuffer>() {
        let arr = js_sys::Uint8Array::new(buf);
        return Ok(arr.to_vec());
    }
    return Err("jsvalue_to_bytes: unsupported type".to_string());
}

/// Кодирует байты в hex-строку.
pub fn hex_encode(bytes: &[u8]) -> String {
    let mut out = String::with_capacity(bytes.len() * 2);
    for b in bytes {
        out.push_str(&crate::format_str::format_str(
            format_args!("{:02x}", b),
        ));
    }
    return out;
}
```

### `src/web_crypto/digest.rs`

```rust
//! SHA-256 / SHA-384 / SHA-512 — криптографические хеши.
//!
//! `SubtleCrypto.digest(algorithm, data)` → `Promise<ArrayBuffer>`.
//! Возвращает ровно `hash_len` байт.
//!
//! В web-sys у `SubtleCrypto` нет метода с именем `digest` —
//! есть `digest_with_str_and_buffer_source` (алгоритм как строка),
//! возвращающий `Result<Promise, JsValue>`: обёртка web-sys может
//! кинуть синхронное исключение, поэтому `Result` нужно развернуть
//! до `JsFuture::from`.

use wasm_bindgen::JsValue;
use wasm_bindgen_futures::JsFuture;

use super::CryptoError;
use super::util::{bytes_to_object, jsvalue_to_bytes, hex_encode};

pub const SHA256_LEN: usize = 32;
pub const SHA384_LEN: usize = 48;
pub const SHA512_LEN: usize = 64;

/// Алгоритм хеширования.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum HashAlg {
    Sha256,
    Sha384,
    Sha512,
}

impl HashAlg {
    pub fn as_str(self) -> &'static str {
        return match self {
            HashAlg::Sha256 => "SHA-256",
            HashAlg::Sha384 => "SHA-384",
            HashAlg::Sha512 => "SHA-512",
        };
    }
    pub fn digest_len(self) -> usize {
        return match self {
            HashAlg::Sha256 => SHA256_LEN,
            HashAlg::Sha384 => SHA384_LEN,
            HashAlg::Sha512 => SHA512_LEN,
        };
    }
}

/// Считает хеш от `data`.
pub async fn digest(alg: HashAlg, data: &[u8]) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;
    let data_obj = bytes_to_object(data);
    // web-sys возвращает Result<Promise, JsValue> — разворачиваем.
    let promise: js_sys::Promise = subtle
        .digest_with_str_and_buffer_source(alg.as_str(), &data_obj)
        .map_err(|e| CryptoError::from(e))?;
    let result: JsValue = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}

/// Удобная обёртка: hex-строка хеша.
pub async fn digest_hex(alg: HashAlg, data: &[u8]) -> Result<String, CryptoError> {
    let bytes = digest(alg, data).await?;
    return Ok(hex_encode(&bytes));
}
```

### `src/web_crypto/hmac.rs`

```rust
//! HMAC — Hash-based Message Authentication Code.
//!
//! `generateKey` → `sign` / `verify`.
//! HMAC использует симметричный ключ: один ключ и для подписи, и для проверки.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use super::CryptoError;
use super::util::{bytes_to_object, jsvalue_to_bytes};
use super::digest::HashAlg;

/// Генерирует HMAC-ключ.
pub async fn generate_key(
    hash: HashAlg,
    length_bits: u32,
    extractable: bool,
) -> Result<web_sys::CryptoKey, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("HMAC"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("hash"), &JsValue::from_str(hash.as_str()))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("length"), &JsValue::from_f64(length_bits as f64))
        .map_err(|e| CryptoError::from(e))?;

    let usages = js_sys::Array::new();
    usages.push(&JsValue::from_str("sign"));
    usages.push(&JsValue::from_str("verify"));

    let promise = subtle.generate_key_with_object(&alg, extractable, &usages)
        .map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;

    let key: web_sys::CryptoKey = result
        .dyn_into()
        .map_err(|_| CryptoError::new("HMAC key: unexpected type".to_string()))?;
    return Ok(key);
}

/// Подписывает `data` ключом `key` алгоритмом HMAC.
pub async fn sign(key: &web_sys::CryptoKey, data: &[u8]) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("HMAC"))
        .map_err(|e| CryptoError::from(e))?;

    let data_obj = bytes_to_object(data);
    let promise = subtle.sign_with_object_and_buffer_source(
        &alg,
        key,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}

/// Проверяет подпись `signature` для `data` ключом `key`.
pub async fn verify(
    key: &web_sys::CryptoKey,
    signature: &[u8],
    data: &[u8],
) -> Result<bool, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("HMAC"))
        .map_err(|e| CryptoError::from(e))?;

    let sig_obj = bytes_to_object(signature);
    let data_obj = bytes_to_object(data);
    let promise = subtle.verify_with_object_and_buffer_source_and_buffer_source(
        &alg,
        key,
        &sig_obj,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return result.as_bool().ok_or_else(|| {
        CryptoError::new("HMAC verify: expected boolean".to_string())
    });
}
```

### `src/web_crypto/aes.rs`

```rust
//! AES-GCM — симметричное шифрование с аутентификацией.
//!
//! `generateKey` → `encrypt` / `decrypt`.
//! GCM добавляет authentication tag: изменение шифротекста
//! приводит к ошибке дешифрования.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use super::CryptoError;
use super::util::{bytes_to_object, jsvalue_to_bytes};

pub const AES_GCM_IV_LEN: usize = 12; // 96 бит — рекомендация NIST

/// Генерирует AES-GCM ключ.
pub async fn generate_key(
    length_bits: u32,
    extractable: bool,
) -> Result<web_sys::CryptoKey, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("AES-GCM"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("length"), &JsValue::from_f64(length_bits as f64))
        .map_err(|e| CryptoError::from(e))?;

    let usages = js_sys::Array::new();
    usages.push(&JsValue::from_str("encrypt"));
    usages.push(&JsValue::from_str("decrypt"));

    let promise = subtle.generate_key_with_object(&alg, extractable, &usages)
        .map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;

    let key: web_sys::CryptoKey = result
        .dyn_into()
        .map_err(|_| CryptoError::new("AES key: unexpected type".to_string()))?;
    return Ok(key);
}

/// Шифрует `plaintext` ключом `key` со случайным IV.
pub async fn encrypt(
    key: &web_sys::CryptoKey,
    plaintext: &[u8],
) -> Result<(Vec<u8>, Vec<u8>), CryptoError> {
    let iv = super::random::random_bytes(AES_GCM_IV_LEN)?;
    let ciphertext = encrypt_with_iv(key, plaintext, &iv).await?;
    return Ok((ciphertext, iv));
}

/// Шифрует с явным IV (для тестов с детерминированным IV).
pub async fn encrypt_with_iv(
    key: &web_sys::CryptoKey,
    plaintext: &[u8],
    iv: &[u8],
) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("AES-GCM"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(
        &alg,
        &JsValue::from_str("iv"),
        &JsValue::from(bytes_to_object(iv)),
    ).map_err(|e| CryptoError::from(e))?;

    let data_obj = bytes_to_object(plaintext);
    let promise = subtle.encrypt_with_object_and_buffer_source(
        &alg,
        key,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}

/// Дешифрует `ciphertext` ключом `key` с IV `iv`.
pub async fn decrypt(
    key: &web_sys::CryptoKey,
    ciphertext: &[u8],
    iv: &[u8],
) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("AES-GCM"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(
        &alg,
        &JsValue::from_str("iv"),
        &JsValue::from(bytes_to_object(iv)),
    ).map_err(|e| CryptoError::from(e))?;

    let data_obj = bytes_to_object(ciphertext);
    let promise = subtle.decrypt_with_object_and_buffer_source(
        &alg,
        key,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}
```

### `src/web_crypto/rsa.rs`

```rust
//! RSA-OAEP — асимметричное шифрование.
//!
//! `generateKey` → `(publicKey, privateKey)`.
//! Шифруем публичным, дешифруем приватным.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use super::CryptoError;
use super::util::{bytes_to_object, jsvalue_to_bytes};
use super::digest::HashAlg;

pub const RSA_2048: u32 = 2048;
pub const RSA_4096: u32 = 4096;

/// Стандартная публичная экспонента F4 = 65537 = [0x01, 0x00, 0x01].
const PUBLIC_EXPONENT_F4: [u8; 3] = [0x01, 0x00, 0x01];

/// Пара ключей RSA.
pub struct RsaKeyPair {
    pub public_key: web_sys::CryptoKey,
    pub private_key: web_sys::CryptoKey,
}

/// Генерирует пару ключей RSA-OAEP.
///
/// `modulus_length` — 2048 или 4096 бит.
/// `hash` — хеш для OAEP-дополнения.
pub async fn generate_key_pair(
    modulus_length: u32,
    hash: HashAlg,
) -> Result<RsaKeyPair, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("RSA-OAEP"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("modulusLength"), &JsValue::from_f64(modulus_length as f64))
        .map_err(|e| CryptoError::from(e))?;

    // publicExponent — обязательное поле. Формат: Uint8Array.
    let exponent = js_sys::Uint8Array::new_with_length(PUBLIC_EXPONENT_F4.len() as u32);
    exponent.copy_from(&PUBLIC_EXPONENT_F4);
    js_sys::Reflect::set(
        &alg,
        &JsValue::from_str("publicExponent"),
        &JsValue::from(exponent),
    ).map_err(|e| CryptoError::from(e))?;

    let hash_obj = js_sys::Object::new();
    js_sys::Reflect::set(&hash_obj, &JsValue::from_str("name"), &JsValue::from_str(hash.as_str()))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("hash"), &JsValue::from(hash_obj))
        .map_err(|e| CryptoError::from(e))?;

    let usages = js_sys::Array::new();
    usages.push(&JsValue::from_str("encrypt"));
    usages.push(&JsValue::from_str("decrypt"));

    let promise = subtle.generate_key_with_object(&alg, true, &usages)
        .map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;

    let public = js_sys::Reflect::get(&result, &JsValue::from_str("publicKey"))
        .map_err(|e| CryptoError::from(e))?;
    let private = js_sys::Reflect::get(&result, &JsValue::from_str("privateKey"))
        .map_err(|e| CryptoError::from(e))?;

    let public_key: web_sys::CryptoKey = public
        .dyn_into()
        .map_err(|_| CryptoError::new("RSA publicKey: unexpected type".to_string()))?;
    let private_key: web_sys::CryptoKey = private
        .dyn_into()
        .map_err(|_| CryptoError::new("RSA privateKey: unexpected type".to_string()))?;

    return Ok(RsaKeyPair { public_key, private_key });
}

/// Шифрует `plaintext` публичным ключом RSA-OAEP.
pub async fn encrypt(
    public_key: &web_sys::CryptoKey,
    plaintext: &[u8],
) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("RSA-OAEP"))
        .map_err(|e| CryptoError::from(e))?;

    let data_obj = bytes_to_object(plaintext);
    let promise = subtle.encrypt_with_object_and_buffer_source(
        &alg,
        public_key,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}

/// Дешифрует `ciphertext` приватным ключом RSA-OAEP.
pub async fn decrypt(
    private_key: &web_sys::CryptoKey,
    ciphertext: &[u8],
) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("RSA-OAEP"))
        .map_err(|e| CryptoError::from(e))?;

    let data_obj = bytes_to_object(ciphertext);
    let promise = subtle.decrypt_with_object_and_buffer_source(
        &alg,
        private_key,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}
```

### `src/web_crypto/ecdsa.rs`

```rust
//! ECDSA — подпись на эллиптических кривых.
//! ECDH — обмен ключами (deriveKey / deriveBits).
//!
//! ECDSA: `generateKey` → `(publicKey, privateKey)` → `sign` / `verify`.
//! ECDH: два участника генерируют пары, обмениваются публичными,
//!       затем `deriveKey` даёт общий симметричный ключ.

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use super::CryptoError;
use super::util::{bytes_to_object, jsvalue_to_bytes};

pub const P256: &str = "P-256";
pub const P384: &str = "P-384";
pub const P521: &str = "P-521";

/// Пара ключей ECDSA.
pub struct EcdsaKeyPair {
    pub public_key: web_sys::CryptoKey,
    pub private_key: web_sys::CryptoKey,
}

/// Генерирует пару ключей ECDSA.
pub async fn generate_key_pair(
    named_curve: &str,
) -> Result<EcdsaKeyPair, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("ECDSA"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("namedCurve"), &JsValue::from_str(named_curve))
        .map_err(|e| CryptoError::from(e))?;

    let usages = js_sys::Array::new();
    usages.push(&JsValue::from_str("sign"));
    usages.push(&JsValue::from_str("verify"));

    let promise = subtle.generate_key_with_object(&alg, true, &usages)
        .map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;

    let public = js_sys::Reflect::get(&result, &JsValue::from_str("publicKey"))
        .map_err(|e| CryptoError::from(e))?;
    let private = js_sys::Reflect::get(&result, &JsValue::from_str("privateKey"))
        .map_err(|e| CryptoError::from(e))?;

    let public_key: web_sys::CryptoKey = public
        .dyn_into()
        .map_err(|_| CryptoError::new("ECDSA publicKey: unexpected type".to_string()))?;
    let private_key: web_sys::CryptoKey = private
        .dyn_into()
        .map_err(|_| CryptoError::new("ECDSA privateKey: unexpected type".to_string()))?;

    return Ok(EcdsaKeyPair { public_key, private_key });
}

/// Подписывает `data` приватным ключом ECDSA.
pub async fn sign(
    private_key: &web_sys::CryptoKey,
    data: &[u8],
) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("ECDSA"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("hash"), &JsValue::from_str("SHA-256"))
        .map_err(|e| CryptoError::from(e))?;

    let data_obj = bytes_to_object(data);
    let promise = subtle.sign_with_object_and_buffer_source(
        &alg,
        private_key,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}

/// Проверяет подпись `signature` для `data` публичным ключом ECDSA.
pub async fn verify(
    public_key: &web_sys::CryptoKey,
    signature: &[u8],
    data: &[u8],
) -> Result<bool, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("ECDSA"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("hash"), &JsValue::from_str("SHA-256"))
        .map_err(|e| CryptoError::from(e))?;

    let sig_obj = bytes_to_object(signature);
    let data_obj = bytes_to_object(data);
    let promise = subtle.verify_with_object_and_buffer_source_and_buffer_source(
        &alg,
        public_key,
        &sig_obj,
        &data_obj,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return result.as_bool().ok_or_else(|| {
        CryptoError::new("ECDSA verify: expected boolean".to_string())
    });
}

// ============================================================
//                    ECDH
// ============================================================

/// Пара ключей ECDH.
pub struct EcdhKeyPair {
    pub public_key: web_sys::CryptoKey,
    pub private_key: web_sys::CryptoKey,
}

/// Генерирует пару ключей ECDH.
///
/// Web Crypto не позволяет пустой `keyUsages` для `generateKey`:
/// приватный ключ должен получить хотя бы одну роль. Для ECDH это
/// `deriveKey` / `deriveBits`. Публичный ключ всё равно получит `[]` —
/// это неявно делает движок.
pub async fn generate_key_pair_ecdh(
    named_curve: &str,
) -> Result<EcdhKeyPair, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("ECDH"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("namedCurve"), &JsValue::from_str(named_curve))
        .map_err(|e| CryptoError::from(e))?;

    // keyUsages НЕ может быть пустым для generateKey — нужна хотя бы
    // одна роль. Для ECDH это deriveKey или deriveBits.
    let usages = js_sys::Array::new();
    usages.push(&JsValue::from_str("deriveKey"));

    let promise = subtle.generate_key_with_object(&alg, false, &usages)
        .map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;

    let public = js_sys::Reflect::get(&result, &JsValue::from_str("publicKey"))
        .map_err(|e| CryptoError::from(e))?;
    let private = js_sys::Reflect::get(&result, &JsValue::from_str("privateKey"))
        .map_err(|e| CryptoError::from(e))?;

    let public_key: web_sys::CryptoKey = public
        .dyn_into()
        .map_err(|_| CryptoError::new("ECDH publicKey: unexpected type".to_string()))?;
    let private_key: web_sys::CryptoKey = private
        .dyn_into()
        .map_err(|_| CryptoError::new("ECDH privateKey: unexpected type".to_string()))?;

    return Ok(EcdhKeyPair { public_key, private_key });
}

/// Выводит общий AES-GCM ключ из приватного ключа и публичного ключа собеседника.
pub async fn derive_aes_key(
    my_private: &web_sys::CryptoKey,
    their_public: &web_sys::CryptoKey,
) -> Result<web_sys::CryptoKey, CryptoError> {
    let subtle = super::subtle()?;

    let alg = js_sys::Object::new();
    js_sys::Reflect::set(&alg, &JsValue::from_str("name"), &JsValue::from_str("ECDH"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&alg, &JsValue::from_str("public"), their_public)
        .map_err(|e| CryptoError::from(e))?;

    let derived_alg = js_sys::Object::new();
    js_sys::Reflect::set(&derived_alg, &JsValue::from_str("name"), &JsValue::from_str("AES-GCM"))
        .map_err(|e| CryptoError::from(e))?;
    js_sys::Reflect::set(&derived_alg, &JsValue::from_str("length"), &JsValue::from_f64(256.0))
        .map_err(|e| CryptoError::from(e))?;

    let usages = js_sys::Array::new();
    usages.push(&JsValue::from_str("encrypt"));
    usages.push(&JsValue::from_str("decrypt"));

    let promise = subtle.derive_key_with_object_and_object(
        &alg,
        my_private,
        &derived_alg,
        false,
        &usages,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;

    let key: web_sys::CryptoKey = result
        .dyn_into()
        .map_err(|_| CryptoError::new("ECDH derived key: unexpected type".to_string()))?;
    return Ok(key);
}
```

### `src/web_crypto/random.rs`

```rust
//! Криптографически стойкие случайные значения.
//!
//! `Crypto.getRandomValues()` — единственный метод Web Crypto,
//! доступный в небезопасном контексте (HTTP).
//! `randomUUID()` — генерация UUID v4.
//!
//! `Crypto::random_uuid()` в web-sys возвращает `String` напрямую
//! (метод синхронный и не может провалиться в нормальных условиях).

use super::CryptoError;

/// Заполняет `len` байт криптостойкой случайностью.
pub fn random_bytes(len: usize) -> Result<Vec<u8>, CryptoError> {
    let crypto = super::crypto_obj()?;
    let mut out = vec![0u8; len];
    // getRandomValues принимает `&mut [u8]` — передаём явный срез.
    let slice: &mut [u8] = out.as_mut_slice();
    crypto
        .get_random_values_with_u8_array(slice)
        .map_err(|e| CryptoError::from(e))?;
    return Ok(out);
}

/// Возвращает случайный UUID v4 в виде строки.
pub fn random_uuid() -> Result<String, CryptoError> {
    let crypto = super::crypto_obj()?;
    return Ok(crypto.random_uuid());
}
```

### `src/web_crypto/keys.rs`

```rust
//! importKey / exportKey — импорт и экспорт ключей.
//!
//! Форматы:
//! - `"raw"`   — сырые байты (HMAC, AES).
//! - `"jwk"`   — JSON Web Key (все типы, включая асимметричные).
//! - `"spki"`  — публичный ключ (SubjectPublicKeyInfo).
//! - `"pkcs8"` — приватный ключ (PKCS#8).

use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::JsFuture;

use super::CryptoError;
use super::util::{bytes_to_object, jsvalue_to_bytes};

/// Формат ключа.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum KeyFormat {
    Raw,
    Jwk,
    Spki,
    Pkcs8,
}

impl KeyFormat {
    pub fn as_str(self) -> &'static str {
        return match self {
            KeyFormat::Raw => "raw",
            KeyFormat::Jwk => "jwk",
            KeyFormat::Spki => "spki",
            KeyFormat::Pkcs8 => "pkcs8",
        };
    }
}

/// Строит algorithm-объект `{ name: "AES-GCM" }` — нужен для importKey
/// AES-ключа из raw/JWK.
pub fn aes_gcm_alg() -> js_sys::Object {
    let alg = js_sys::Object::new();
    let _ = js_sys::Reflect::set(
        &alg,
        &JsValue::from_str("name"),
        &JsValue::from_str("AES-GCM"),
    );
    return alg;
}

/// Импортирует ключ из сырых байт (`raw`) или из JWK-объекта.
pub async fn import_raw(
    format: KeyFormat,
    data: &[u8],
    algorithm: &js_sys::Object,
    extractable: bool,
    usages: &[&str],
) -> Result<web_sys::CryptoKey, CryptoError> {
    let subtle = super::subtle()?;
    let data_obj = bytes_to_object(data);
    let usages_js = js_sys::Array::new();
    for u in usages {
        usages_js.push(&JsValue::from_str(u));
    }
    let promise = subtle.import_key_with_object(
        format.as_str(),
        &data_obj,
        algorithm,
        extractable,
        &usages_js,
    ).map_err(|e| CryptoError::from(e))?;

    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    let key: web_sys::CryptoKey = result
        .dyn_into()
        .map_err(|_| CryptoError::new("importKey: unexpected type".to_string()))?;
    return Ok(key);
}

/// Экспортирует ключ в сырые байты (`raw`).
pub async fn export_raw(key: &web_sys::CryptoKey) -> Result<Vec<u8>, CryptoError> {
    let subtle = super::subtle()?;
    let promise = subtle.export_key("raw", key)
        .map_err(|e| CryptoError::from(e))?;
    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    return jsvalue_to_bytes(&result).map_err(|e| CryptoError::new(e));
}

/// Экспортирует ключ в формате JWK (возвращает JSON-строку).
pub async fn export_jwk(key: &web_sys::CryptoKey) -> Result<String, CryptoError> {
    let subtle = super::subtle()?;
    let promise = subtle.export_key("jwk", key)
        .map_err(|e| CryptoError::from(e))?;
    let result = JsFuture::from(promise)
        .await
        .map_err(|e| CryptoError::from(e))?;
    let json = js_sys::JSON::stringify(&result)
        .map_err(|e| CryptoError::from(e))?;
    return json
        .as_string()
        .ok_or_else(|| CryptoError::new("JWK export: expected string".to_string()));
}
```

### `src/app.rs`

```rust
//! Глава 38: Web Crypto API — браузерная криптография.
//!
//! Демонстрирует:
//!   0. Сквозной сценарий «Алиса → Боб» (ECDH + AES-GCM + ECDSA).
//!   1. Хеширование SHA-256/384/512 + лавинный эффект.
//!   2. HMAC: sign / verify (изменение текста ломает подпись).
//!   3. AES-GCM: encrypt / decrypt + детектирование подмены.
//!   4. RSA-OAEP: асимметричное шифрование.
//!   5. ECDSA: подпись / проверка.
//!   6. ECDH: обмен ключами → общий AES-ключ.
//!   7. random: getRandomValues, UUID.
//!   8. importKey / exportKey: raw, JWK + обратный импорт.
//!
//! ПРАВИЛО: format! в tail-позиции — через format_str(format_args!(...)).
//! ПРАВИЛО: HTML с ≥2 вложенными `"` — через push_str.
//! ПРАВИЛО: длительные async-операции отдают управление через TimeoutFuture(0).

use std::cell::RefCell;
use std::rc::Rc;

use gloo_timers::future::TimeoutFuture;
use wasm_bindgen::closure::Closure;
use wasm_bindgen::{JsCast, JsValue};
use wasm_bindgen_futures::spawn_local;
use web_sys::{HtmlElement, HtmlTextAreaElement};

use crate::format_str::format_str;
use crate::web_crypto::{
    aes, digest, ecdsa, hmac, keys, random, rsa,
    digest::HashAlg,
    keys::KeyFormat,
    util::hex_encode,
};

// ============================================================
//                    HTML
// ============================================================

const HTML: &str = r#"
<style>
  :root {
    --c-info: #3498db; --c-ok: #27ae60; --c-err: #c0392b; --c-warn: #e67e22;
    --c-text: #222; --c-muted: #666;
    --c-bg: #fff; --c-soft: #fafafa; --c-border: #e0e0e0;
    --c-scenario: #8e44ad;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --c-text: #e6e6e6; --c-muted: #a0a0a0;
      --c-bg: #1e1e1e; --c-soft: #262626; --c-border: #3a3a3a;
    }
    body { background: #121212; }
    select, input, textarea { background: #1e1e1e; color: #e6e6e6; border-color: #3a3a3a; }
    .c-out { background: #1a1a1a; color: #9ed09e; border-color: #333; }
  }
  html, body { margin: 0; padding: 0; width: 100%; }
  body {
    font-family: system-ui, -apple-system, "Segoe UI", sans-serif;
    color: var(--c-text); line-height: 1.45; font-size: 13px;
    background: #f7f8fa;
  }
  @media (prefers-color-scheme: dark) { body { background: #121212; } }
  #app { padding: 12px 16px 24px; box-sizing: border-box; max-width: 900px; margin: 0 auto; }
  #app * { box-sizing: border-box; }

  h1 { margin: 0 0 2px; font-size: 19px; }
  .c-subtitle { color: var(--c-muted); font-size: 12px; margin: 0 0 14px; }
  .c-subtitle a { color: var(--c-info); }

  .c-panel {
    border: 1px solid var(--c-border); border-radius: 8px;
    padding: 10px 12px; background: var(--c-bg);
    box-shadow: 0 1px 3px rgba(0,0,0,0.04);
    margin-bottom: 12px;
  }
  .c-panel-scenario {
    border: 2px solid var(--c-scenario);
    box-shadow: 0 2px 8px rgba(142,68,173,0.12);
  }
  .c-panel-title {
    font-size: 13px; font-weight: 600; margin: 0 0 8px;
    display: flex; align-items: center; gap: 6px;
  }
  .c-num {
    display: inline-flex; align-items: center; justify-content: center;
    width: 20px; height: 20px; border-radius: 50%;
    background: var(--c-info); color: #fff; font-weight: 700; font-size: 11px;
    flex-shrink: 0;
  }
  .c-num-scenario { background: var(--c-scenario); }
  .c-copy-btn {
    background: transparent; border: 1px solid var(--c-border);
    color: var(--c-muted);
    padding: 1px 8px; font-size: 11px; margin-left: auto;
    cursor: pointer; border-radius: 4px; font-family: inherit;
    transition: background 0.15s;
  }
  .c-copy-btn:hover { background: rgba(128,128,128,0.15); }

  .c-row { display: flex; gap: 6px; flex-wrap: wrap; margin: 6px 0; align-items: center; }
  .c-label { font-size: 11px; color: var(--c-muted); }

  select, input[type="text"], textarea {
    padding: 5px 8px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--c-bg); color: var(--c-text);
    font-family: ui-monospace, monospace;
    width: 100%;
  }
  textarea { height: 50px; resize: vertical; }
  select { cursor: pointer; }

  button {
    padding: 5px 12px; font-size: 12px;
    border: 1px solid #c0c0c0; border-radius: 4px;
    background: var(--c-soft); color: var(--c-text);
    cursor: pointer; font-family: inherit; white-space: nowrap;
    transition: background 0.15s, opacity 0.15s;
  }
  button:hover:not(:disabled) { background: rgba(128,128,128,0.15); }
  button:disabled { opacity: 0.45; cursor: not-allowed; }
  button.c-primary { background: #eaf4ff; border-color: var(--c-info); color: #1f4e79; }
  button.c-success { background: #e8f8ed; border-color: var(--c-ok); color: #1e8449; }
  button.c-danger  { background: #fdedec; border-color: var(--c-err); color: #922b21; }
  button.c-scenario { background: #f4ecf8; border-color: var(--c-scenario); color: #6c3483; }

  .c-badge {
    display: inline-block; padding: 1px 8px; border-radius: 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    font-weight: 700; background: #eef4ff; color: #1f4e79;
    margin-left: auto;
    transition: background 0.2s, color 0.2s;
  }
  .c-badge.ok  { background: #e8f8ed; color: #1e8449; }
  .c-badge.err { background: #fdedec; color: #922b21; }
  .c-badge.warn { background: #fff9e6; color: #7d5a00; }

  .c-out {
    background: #f6f8fa; border: 1px solid var(--c-border);
    border-radius: 4px; padding: 6px 8px;
    font-family: ui-monospace, monospace; font-size: 11px;
    color: #2d5a2d; overflow-wrap: anywhere;
    margin: 4px 0; min-height: 16px;
    white-space: pre-wrap;
  }
  @media (prefers-color-scheme: dark) {
    .c-out { background: #1a1a1a; color: #9ed09e; border-color: #333; }
  }
  .c-out.empty { color: var(--c-muted); font-style: italic; }

  .c-hint {
    background: #fff9e6; border-left: 3px solid var(--c-warn);
    padding: 6px 10px; border-radius: 4px;
    font-size: 11px; line-height: 1.5; margin: 0 0 10px;
    color: #7d5a00;
  }
  .c-hint code { background: rgba(0,0,0,0.06); padding: 1px 4px; border-radius: 3px; }
  .c-note {
    font-size: 11px; color: var(--c-muted); margin: 4px 0 0;
    line-height: 1.5;
  }
  .c-note b { color: var(--c-text); }
  .c-checklist {
    list-style: none; padding: 0; margin: 0;
    display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 4px 16px;
    font-size: 11px; font-family: ui-monospace, monospace;
    color: var(--c-muted);
  }
  .c-checklist li { padding: 2px 0; }
</style>

<h1>Глава 38: Web Crypto API</h1>
<p class="c-subtitle">
  Браузерная криптография: хеши, HMAC, AES, RSA, ECDSA, ECDH, random.
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API"
     target="_blank" rel="noopener">MDN →</a>
</p>

<div class="c-hint" id="c-hint"></div>

<!-- ============ 0. Сквозной сценарий ============ -->
<div class="c-panel c-panel-scenario" id="c-panel-scenario">
  <h2 class="c-panel-title">
    <span class="c-num c-num-scenario">★</span>Сквозной сценарий: Алиса → Боб
    <button class="c-copy-btn" data-target="c-panel-scenario" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <p class="c-note">
    4 примитива в одном реальном флоу: <b>ECDH</b> (обмен ключами) +
    <b>AES-GCM</b> (шифрование) + <b>ECDSA</b> (подпись). Алиса и Боб
    договариваются об общем ключе, не пересылая его, затем обмениваются
    защищённым сообщением.
  </p>
  <div class="c-row">
    <textarea id="c-scenario-input">Сообщение от Алисы: встреча в 15:00</textarea>
  </div>
  <div class="c-row">
    <button id="c-scenario-run" class="c-scenario">▶ Запустить сценарий</button>
    <button id="c-scenario-tamper" class="c-danger" disabled>⚠ Запустить с подменой</button>
    <span class="c-badge" id="c-scenario-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-scenario-log">—</div>
</div>

<!-- ============ 1. Хеширование + лавинный эффект ============ -->
<div class="c-panel" id="c-panel-digest">
  <h2 class="c-panel-title">
    <span class="c-num">1</span>Хеширование (SHA-256 / 384 / 512)
    <button class="c-copy-btn" data-target="c-panel-digest" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <textarea id="c-digest-input">Привет, Web Crypto!</textarea>
  </div>
  <div class="c-row">
    <button id="c-digest-btn" class="c-primary">Считать хеш</button>
    <span class="c-badge" id="c-digest-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-digest-sha256">SHA-256: —</div>
  <div class="c-out empty" id="c-digest-sha384">SHA-384: —</div>
  <div class="c-out empty" id="c-digest-sha512">SHA-512: —</div>

  <div class="c-row" style="margin-top: 10px;">
    <label class="c-label" for="c-digest-input2">Лавинный эффект — текст, отличающийся на 1 символ:</label>
  </div>
  <div class="c-row">
    <textarea id="c-digest-input2">Привет, Web Cryptp!</textarea>
  </div>
  <div class="c-row">
    <button id="c-digest-avalanche-btn" class="c-primary">Сравнить SHA-256</button>
    <span class="c-badge" id="c-digest-avalanche-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-digest-avalanche">Лавинный эффект: —</div>
  <p class="c-note">
    Замена одного символа меняет ~50% бит хеша. Это свойство называется
    «лавинным эффектом» и делает хеш криптостойким: по хешу нельзя
    подобрать похожий текст.
  </p>
</div>

<!-- ============ 2. HMAC ============ -->
<div class="c-panel" id="c-panel-hmac">
  <h2 class="c-panel-title">
    <span class="c-num">2</span>HMAC (SHA-256)
    <button class="c-copy-btn" data-target="c-panel-hmac" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <textarea id="c-hmac-input">Сообщение для HMAC</textarea>
  </div>
  <div class="c-row">
    <button id="c-hmac-btn" class="c-primary">Сгенерировать ключ + подписать</button>
    <button id="c-hmac-verify-btn" class="c-success" disabled>Проверить</button>
    <span class="c-badge" id="c-hmac-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-hmac-key">Ключ: —</div>
  <div class="c-out empty" id="c-hmac-sig">Подпись: —</div>
  <div class="c-out empty" id="c-hmac-verify">Проверка: —</div>
  <p class="c-note">
    <b>Измените текст выше</b> после подписи и нажмите «Проверить» —
    увидите, что HMAC защищает именно содержимое.
  </p>
</div>

<!-- ============ 3. AES-GCM ============ -->
<div class="c-panel" id="c-panel-aes">
  <h2 class="c-panel-title">
    <span class="c-num">3</span>AES-GCM (256 бит)
    <button class="c-copy-btn" data-target="c-panel-aes" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <textarea id="c-aes-input">Секретное сообщение AES-GCM</textarea>
  </div>
  <div class="c-row">
    <button id="c-aes-btn" class="c-primary">Зашифровать</button>
    <button id="c-aes-dec-btn" class="c-success" disabled>Дешифровать</button>
    <button id="c-aes-tamper-btn" class="c-danger" disabled>⚠ Повредить и расшифровать</button>
    <span class="c-badge" id="c-aes-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-aes-iv">IV: —</div>
  <div class="c-out empty" id="c-aes-ct">Шифротекст: —</div>
  <div class="c-out empty" id="c-aes-pt">Открытый текст: —</div>
  <p class="c-note">
    AES-GCM не только шифрует, но и аутентифицирует: изменение
    хотя бы одного бита шифротекста → ошибка дешифрования.
    Ключ сохраняется — блок 8 экспортирует именно его.
  </p>
</div>

<!-- ============ 4. RSA-OAEP ============ -->
<div class="c-panel" id="c-panel-rsa">
  <h2 class="c-panel-title">
    <span class="c-num">4</span>RSA-OAEP (2048 бит)
    <button class="c-copy-btn" data-target="c-panel-rsa" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <textarea id="c-rsa-input">Короткое сообщение для RSA</textarea>
  </div>
  <div class="c-row">
    <button id="c-rsa-gen-btn" class="c-primary">Сгенерировать пару ключей</button>
    <button id="c-rsa-enc-btn" class="c-success" disabled>Зашифровать</button>
    <button id="c-rsa-dec-btn" class="c-success" disabled>Дешифровать</button>
    <span class="c-badge" id="c-rsa-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-rsa-pub">Публичный ключ (JWK): —</div>
  <div class="c-out empty" id="c-rsa-ct">Шифротекст: —</div>
  <div class="c-out empty" id="c-rsa-pt">Открытый текст: —</div>
  <p class="c-note">
    RSA шифрует только короткие сообщения (≤ 190 байт для 2048 бит / SHA-256).
    Генерация ключа занимает 0.5–2 секунды.
  </p>
</div>

<!-- ============ 5. ECDSA ============ -->
<div class="c-panel" id="c-panel-ecdsa">
  <h2 class="c-panel-title">
    <span class="c-num">5</span>ECDSA (P-256)
    <button class="c-copy-btn" data-target="c-panel-ecdsa" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <textarea id="c-ecdsa-input">Подписываемое сообщение</textarea>
  </div>
  <div class="c-row">
    <button id="c-ecdsa-sign-btn" class="c-primary">Сгенерировать ключ + подписать</button>
    <button id="c-ecdsa-verify-btn" class="c-success" disabled>Проверить</button>
    <span class="c-badge" id="c-ecdsa-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-ecdsa-sig">Подпись: —</div>
  <div class="c-out empty" id="c-ecdsa-verify">Проверка: —</div>
  <p class="c-note">
    <b>Измените текст выше</b> после подписи и нажмите «Проверить» — увидите
    тот же эффект, что и у HMAC, но здесь ключ асимметричный: подпись
    делается приватным, проверяется публичным. Подпись ECDSA P-256 —
    всего 64 байта (для сравнения: RSA-2048 — 256 байт).
  </p>
</div>

<!-- ============ 6. ECDH ============ -->
<div class="c-panel" id="c-panel-ecdh">
  <h2 class="c-panel-title">
    <span class="c-num">6</span>ECDH (P-256 → AES-GCM)
    <button class="c-copy-btn" data-target="c-panel-ecdh" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <textarea id="c-ecdh-input">Общий секрет через ECDH</textarea>
  </div>
  <div class="c-row">
    <button id="c-ecdh-btn" class="c-primary">Обмен ключами + шифрование</button>
    <span class="c-badge" id="c-ecdh-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-ecdh-result">—</div>
  <p class="c-note">
    Алиса и Боб обмениваются публичными ключами и независимо вычисляют
    одинаковый AES-ключ. Шифруем у Алисы — дешифруем у Боба.
  </p>
</div>

<!-- ============ 7. Random ============ -->
<div class="c-panel" id="c-panel-random">
  <h2 class="c-panel-title">
    <span class="c-num">7</span>Случайные значения
    <button class="c-copy-btn" data-target="c-panel-random" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <button id="c-rand-btn" class="c-primary">32 случайных байта + UUID</button>
    <span class="c-badge" id="c-rand-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-rand-bytes">Байты: —</div>
  <div class="c-out empty" id="c-rand-uuid">UUID: —</div>
  <p class="c-note">
    <code>getRandomValues</code> — единственный метод Web Crypto, доступный
    в небезопасном контексте (HTTP). Для IV, соли, токенов — только он.
  </p>
</div>

<!-- ============ 8. import/export ============ -->
<div class="c-panel" id="c-panel-io">
  <h2 class="c-panel-title">
    <span class="c-num">8</span>importKey / exportKey
    <button class="c-copy-btn" data-target="c-panel-io" title="Скопировать весь вывод панели">📋</button>
  </h2>
  <div class="c-row">
    <button id="c-io-btn" class="c-primary">Экспорт AES-ключа (raw + JWK)</button>
    <button id="c-io-import-btn" class="c-success" disabled>Импорт из raw обратно</button>
    <span class="c-badge" id="c-io-status">ожидание</span>
  </div>
  <div class="c-out empty" id="c-io-raw">Raw: —</div>
  <div class="c-out empty" id="c-io-jwk">JWK: —</div>
  <div class="c-out empty" id="c-io-import">Импорт: —</div>
  <p class="c-note">
    Экспорт возможен только для ключей с <code>extractable: true</code>.
    Экспортированный ключ используется AES-панелью (блок 3).
  </p>
</div>

<!-- ============ Чек-лист ============ -->
<div class="c-panel">
  <h2 class="c-panel-title">📋 Что охвачено в этой главе</h2>
  <ul class="c-checklist">
    <li>✅ Crypto — window.crypto</li>
    <li>✅ SubtleCrypto — crypto.subtle</li>
    <li>✅ CryptoKey — extractable / usages</li>
    <li>✅ random values — getRandomValues + UUID</li>
    <li>✅ hashing — SHA-256/384/512</li>
    <li>✅ HMAC — sign / verify</li>
    <li>✅ AES — AES-GCM 256</li>
    <li>✅ RSA — RSA-OAEP 2048</li>
    <li>✅ ECDSA — P-256 sign / verify</li>
    <li>✅ ECDH — P-256 deriveKey</li>
    <li>✅ key generation</li>
    <li>✅ import / export keys — raw, JWK</li>
    <li>✅ Rust crypto vs Web Crypto — обсуждение в тексте</li>
    <li>✅ Сквозной сценарий Алиса → Боб</li>
  </ul>
</div>
"#;

// ============================================================
//                    МОДЕЛЬ
// ============================================================

struct Ui {
    scenario_input: HtmlTextAreaElement,
    scenario_run: HtmlElement,
    scenario_tamper: HtmlElement,
    scenario_status: web_sys::Element,
    scenario_log: web_sys::Element,

    digest_input: HtmlTextAreaElement,
    digest_btn: HtmlElement,
    digest_status: web_sys::Element,
    digest_sha256: web_sys::Element,
    digest_sha384: web_sys::Element,
    digest_sha512: web_sys::Element,
    digest_input2: HtmlTextAreaElement,
    digest_avalanche_btn: HtmlElement,
    digest_avalanche_status: web_sys::Element,
    digest_avalanche: web_sys::Element,

    hmac_input: HtmlTextAreaElement,
    hmac_btn: HtmlElement,
    hmac_verify_btn: HtmlElement,
    hmac_status: web_sys::Element,
    hmac_key: web_sys::Element,
    hmac_sig: web_sys::Element,
    hmac_verify: web_sys::Element,

    aes_input: HtmlTextAreaElement,
    aes_btn: HtmlElement,
    aes_dec_btn: HtmlElement,
    aes_tamper_btn: HtmlElement,
    aes_status: web_sys::Element,
    aes_iv: web_sys::Element,
    aes_ct: web_sys::Element,
    aes_pt: web_sys::Element,

    rsa_input: HtmlTextAreaElement,
    rsa_gen_btn: HtmlElement,
    rsa_enc_btn: HtmlElement,
    rsa_dec_btn: HtmlElement,
    rsa_status: web_sys::Element,
    rsa_pub: web_sys::Element,
    rsa_ct: web_sys::Element,
    rsa_pt: web_sys::Element,

    ecdsa_input: HtmlTextAreaElement,
    ecdsa_sign_btn: HtmlElement,
    ecdsa_verify_btn: HtmlElement,
    ecdsa_status: web_sys::Element,
    ecdsa_sig: web_sys::Element,
    ecdsa_verify: web_sys::Element,

    ecdh_input: HtmlTextAreaElement,
    ecdh_btn: HtmlElement,
    ecdh_status: web_sys::Element,
    ecdh_result: web_sys::Element,

    rand_btn: HtmlElement,
    rand_status: web_sys::Element,
    rand_bytes: web_sys::Element,
    rand_uuid: web_sys::Element,

    io_btn: HtmlElement,
    io_import_btn: HtmlElement,
    io_status: web_sys::Element,
    io_raw: web_sys::Element,
    io_jwk: web_sys::Element,
    io_import: web_sys::Element,
}

struct State {
    hmac_key: Option<web_sys::CryptoKey>,
    hmac_sig: Option<Vec<u8>>,
    /// Единый AES-ключ для блоков 3 и 8.
    aes_key: Option<web_sys::CryptoKey>,
    aes_ct: Option<Vec<u8>>,
    aes_iv: Option<Vec<u8>>,
    /// Сохранённый raw-экспорт ключа для обратного импорта.
    aes_raw_export: Option<Vec<u8>>,
    rsa_kp: Option<rsa::RsaKeyPair>,
    rsa_ct: Option<Vec<u8>>,
    ecdsa_kp: Option<ecdsa::EcdsaKeyPair>,
    ecdsa_sig: Option<Vec<u8>>,
}

struct App {
    ui: Ui,
    state: RefCell<State>,
}

// ============================================================
//                    ЗАПУСК
// ============================================================

pub fn run() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let root = document.get_element_by_id("app").ok_or("no #app")?;
    root.set_inner_html(HTML);

    let ui = Ui {
        scenario_input: get_textarea("c-scenario-input")?,
        scenario_run: get_btn("c-scenario-run")?,
        scenario_tamper: get_btn("c-scenario-tamper")?,
        scenario_status: get_el("c-scenario-status")?,
        scenario_log: get_el("c-scenario-log")?,

        digest_input: get_textarea("c-digest-input")?,
        digest_btn: get_btn("c-digest-btn")?,
        digest_status: get_el("c-digest-status")?,
        digest_sha256: get_el("c-digest-sha256")?,
        digest_sha384: get_el("c-digest-sha384")?,
        digest_sha512: get_el("c-digest-sha512")?,
        digest_input2: get_textarea("c-digest-input2")?,
        digest_avalanche_btn: get_btn("c-digest-avalanche-btn")?,
        digest_avalanche_status: get_el("c-digest-avalanche-status")?,
        digest_avalanche: get_el("c-digest-avalanche")?,

        hmac_input: get_textarea("c-hmac-input")?,
        hmac_btn: get_btn("c-hmac-btn")?,
        hmac_verify_btn: get_btn("c-hmac-verify-btn")?,
        hmac_status: get_el("c-hmac-status")?,
        hmac_key: get_el("c-hmac-key")?,
        hmac_sig: get_el("c-hmac-sig")?,
        hmac_verify: get_el("c-hmac-verify")?,

        aes_input: get_textarea("c-aes-input")?,
        aes_btn: get_btn("c-aes-btn")?,
        aes_dec_btn: get_btn("c-aes-dec-btn")?,
        aes_tamper_btn: get_btn("c-aes-tamper-btn")?,
        aes_status: get_el("c-aes-status")?,
        aes_iv: get_el("c-aes-iv")?,
        aes_ct: get_el("c-aes-ct")?,
        aes_pt: get_el("c-aes-pt")?,

        rsa_input: get_textarea("c-rsa-input")?,
        rsa_gen_btn: get_btn("c-rsa-gen-btn")?,
        rsa_enc_btn: get_btn("c-rsa-enc-btn")?,
        rsa_dec_btn: get_btn("c-rsa-dec-btn")?,
        rsa_status: get_el("c-rsa-status")?,
        rsa_pub: get_el("c-rsa-pub")?,
        rsa_ct: get_el("c-rsa-ct")?,
        rsa_pt: get_el("c-rsa-pt")?,

        ecdsa_input: get_textarea("c-ecdsa-input")?,
        ecdsa_sign_btn: get_btn("c-ecdsa-sign-btn")?,
        ecdsa_verify_btn: get_btn("c-ecdsa-verify-btn")?,
        ecdsa_status: get_el("c-ecdsa-status")?,
        ecdsa_sig: get_el("c-ecdsa-sig")?,
        ecdsa_verify: get_el("c-ecdsa-verify")?,

        ecdh_input: get_textarea("c-ecdh-input")?,
        ecdh_btn: get_btn("c-ecdh-btn")?,
        ecdh_status: get_el("c-ecdh-status")?,
        ecdh_result: get_el("c-ecdh-result")?,

        rand_btn: get_btn("c-rand-btn")?,
        rand_status: get_el("c-rand-status")?,
        rand_bytes: get_el("c-rand-bytes")?,
        rand_uuid: get_el("c-rand-uuid")?,

        io_btn: get_btn("c-io-btn")?,
        io_import_btn: get_btn("c-io-import-btn")?,
        io_status: get_el("c-io-status")?,
        io_raw: get_el("c-io-raw")?,
        io_jwk: get_el("c-io-jwk")?,
        io_import: get_el("c-io-import")?,
    };

    let app = Rc::new(App {
        ui,
        state: RefCell::new(State {
            hmac_key: None,
            hmac_sig: None,
            aes_key: None,
            aes_ct: None,
            aes_iv: None,
            aes_raw_export: None,
            rsa_kp: None,
            rsa_ct: None,
            ecdsa_kp: None,
            ecdsa_sig: None,
        }),
    });

    render_hint()?;
    bind_copy_buttons()?;
    bind_all(&app)?;

    Ok(())
}

// ============================================================
//                    BIND
// ============================================================

fn bind_all(app: &Rc<App>) -> Result<(), JsValue> {
    {
        let app_c = app.clone();
        set_click(&app.ui.scenario_run, move || {
            let app = app_c.clone();
            spawn_local(async move { do_scenario(app, false).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.scenario_tamper, move || {
            let app = app_c.clone();
            spawn_local(async move { do_scenario(app, true).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.digest_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_digest(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.digest_avalanche_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_avalanche(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.hmac_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_hmac_sign(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.hmac_verify_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_hmac_verify(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.aes_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_aes_encrypt(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.aes_dec_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_aes_decrypt(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.aes_tamper_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_aes_tamper(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.rsa_gen_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_rsa_generate(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.rsa_enc_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_rsa_encrypt(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.rsa_dec_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_rsa_decrypt(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.ecdsa_sign_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_ecdsa_sign(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.ecdsa_verify_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_ecdsa_verify(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.ecdh_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_ecdh(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.rand_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_random(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.io_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_export(app).await });
        });
    }
    {
        let app_c = app.clone();
        set_click(&app.ui.io_import_btn, move || {
            let app = app_c.clone();
            spawn_local(async move { do_import(app).await });
        });
    }
    Ok(())
}

/// Привязывает кнопки `.c-copy-btn` к копированию `.c-out` внутри
/// панели, указанной в `data-target`.
fn bind_copy_buttons() -> Result<(), JsValue> {
    let document = web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?;
    let btns = document.query_selector_all(".c-copy-btn")?;
    for i in 0..btns.length() {
        let node = btns.get(i).ok_or("no btn")?;
        let btn: HtmlElement = node.dyn_into().map_err(|_| {
            JsValue::from_str("copy button: not HtmlElement")
        })?;
        let target_id = btn.get_attribute("data-target").unwrap_or_default();
        let panel = get_el(&target_id)?;
        attach_copy_panel(&btn, panel);
    }
    Ok(())
}

// ============================================================
//                    КОПИРОВАНИЕ
// ============================================================

fn attach_copy_panel(btn: &HtmlElement, panel: web_sys::Element) {
    let btn_c = btn.clone();
    set_click(btn, move || {
        let text = collect_panel_text(&panel);
        let Some(window) = web_sys::window() else { return };
        let clipboard = window.navigator().clipboard();
        let _ = clipboard.write_text(&text);
        btn_c.set_text_content(Some("✓"));
        let btn_r = btn_c.clone();
        spawn_local(async move {
            TimeoutFuture::new(1000).await;
            btn_r.set_text_content(Some("📋"));
        });
    });
}

fn collect_panel_text(panel: &web_sys::Element) -> String {
    let mut buf = String::new();
    let Ok(outs) = panel.query_selector_all(".c-out") else {
        return buf;
    };
    for i in 0..outs.length() {
        if let Some(el) = outs.get(i) {
            if let Some(t) = el.text_content() {
                if !buf.is_empty() {
                    buf.push('\n');
                }
                buf.push_str(&t);
            }
        }
    }
    buf
}

// ============================================================
//                    СКВОЗНОЙ СЦЕНАРИЙ
// ============================================================

async fn do_scenario(app: Rc<App>, tamper: bool) {
    set_status(&app.ui.scenario_status, "warn", "шаг 1/4: ECDH…");
    set_disabled(&app.ui.scenario_run, true);
    set_disabled(&app.ui.scenario_tamper, true);

    let mut log = String::with_capacity(1024);
    log.push_str(if tamper {
        "⚠ Сценарий с подменой: 1 байт шифротекста будет испорчен\n\n"
    } else {
        "▶ Сценарий без подмены\n\n"
    });

    // -------- Шаг 1: ECDH --------
    log.push_str("Шаг 1/4 — ECDH (P-256): обмен ключами\n");
    let alice_ecdh = match ecdsa::generate_key_pair_ecdh(ecdsa::P256).await {
        Ok(k) => k,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    let bob_ecdh = match ecdsa::generate_key_pair_ecdh(ecdsa::P256).await {
        Ok(k) => k,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    log.push_str("  ✓ Алиса и Боб сгенерировали пары ECDH-ключей\n");

    let alice_aes = match ecdsa::derive_aes_key(&alice_ecdh.private_key, &bob_ecdh.public_key).await {
        Ok(k) => k,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    let bob_aes = match ecdsa::derive_aes_key(&bob_ecdh.private_key, &alice_ecdh.public_key).await {
        Ok(k) => k,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    log.push_str("  ✓ Обе стороны вычислили общий AES-256 ключ\n\n");

    // -------- Шаг 2: AES-GCM --------
    log.push_str("Шаг 2/4 — AES-GCM: Алиса шифрует\n");
    let plaintext = app.ui.scenario_input.value().into_bytes();
    log.push_str(&format_str(format_args!(
        "  Плейнтекст: \"{}\" ({} B)\n",
        String::from_utf8_lossy(&plaintext),
        plaintext.len(),
    )));
    let (mut ct, iv) = match aes::encrypt(&alice_aes, &plaintext).await {
        Ok(v) => v,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    log.push_str(&format_str(format_args!("  ✓ IV: {} B\n", iv.len())));
    log.push_str(&format_str(format_args!(
        "  ✓ Шифротекст: {} B (payload {} + tag 16)\n",
        ct.len(),
        ct.len().saturating_sub(16),
    )));
    if tamper && !ct.is_empty() {
        ct[0] ^= 0xFF;
        log.push_str("  ⚠ ПОДМЕНА: байт #0 шифротекста инвертирован\n");
    }
    log.push('\n');

    // -------- Шаг 3: ECDSA --------
    log.push_str("Шаг 3/4 — ECDSA (P-256): Алиса подписывает\n");
    let alice_sign = match ecdsa::generate_key_pair(ecdsa::P256).await {
        Ok(k) => k,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    let sig = match ecdsa::sign(&alice_sign.private_key, &plaintext).await {
        Ok(s) => s,
        Err(e) => {
            log.push_str(&format_str(format_args!("  ❌ {}\n", e.message)));
            return finish_scenario(&app, log, false);
        }
    };
    log.push_str(&format_str(format_args!(
        "  ✓ Подпись: {} B\n\n",
        sig.len(),
    )));

    // -------- Шаг 4: доставка Бобу --------
    log.push_str("Шаг 4/4 — Доставка Бобу\n");
    let decrypted = match aes::decrypt(&bob_aes, &ct, &iv).await {
        Ok(v) => v,
        Err(e) => {
            log.push_str(&format_str(format_args!(
                "  ❌ Расшифровка провалилась: {}\n",
                e.message,
            )));
            log.push_str("  ⟹ Подмена шифротекста обнаружена. Сообщение отвергнуто.\n");
            return finish_scenario(&app, log, true);
        }
    };
    log.push_str(&format_str(format_args!(
        "  ✓ Расшифровка: \"{}\"\n",
        String::from_utf8_lossy(&decrypted),
    )));

    if decrypted != plaintext {
        log.push_str("  ❌ Расшифрованный текст не совпадает с исходным!\n");
        return finish_scenario(&app, log, false);
    }

    let sig_ok = match ecdsa::verify(&alice_sign.public_key, &sig, &decrypted).await {
        Ok(v) => v,
        Err(e) => {
            log.push_str(&format_str(format_args!(
                "  ❌ Проверка подписи: {}\n",
                e.message,
            )));
            return finish_scenario(&app, log, false);
        }
    };
    if sig_ok {
        log.push_str("  ✓ Подпись Алисы верна\n");
        log.push_str("\n✅ Итог: конфиденциальность + целостность + авторство подтверждены\n");
        return finish_scenario(&app, log, true);
    }
    log.push_str("  ❌ Подпись недействительна\n");
    finish_scenario(&app, log, false)
}

fn finish_scenario(app: &Rc<App>, log: String, ok: bool) {
    set_out(&app.ui.scenario_log, &log);
    if ok {
        set_status(&app.ui.scenario_status, "ok", "успех");
    } else {
        set_status(&app.ui.scenario_status, "err", "провал");
    }
    set_disabled(&app.ui.scenario_run, false);
    set_disabled(&app.ui.scenario_tamper, false);
}

// ============================================================
//                    ДЕЙСТВИЯ
// ============================================================

async fn do_digest(app: Rc<App>) {
    set_status(&app.ui.digest_status, "warn", "считаем…");
    set_disabled(&app.ui.digest_btn, true);

    let text = app.ui.digest_input.value();
    let data = text.as_bytes();

    let sha256 = digest::digest_hex(HashAlg::Sha256, data).await;
    let sha384 = digest::digest_hex(HashAlg::Sha384, data).await;
    let sha512 = digest::digest_hex(HashAlg::Sha512, data).await;

    set_disabled(&app.ui.digest_btn, false);

    match (sha256, sha384, sha512) {
        (Ok(a), Ok(b), Ok(c)) => {
            set_out(&app.ui.digest_sha256, &format_str(format_args!(
                "SHA-256 (32 B): {a}",
            )));
            set_out(&app.ui.digest_sha384, &format_str(format_args!(
                "SHA-384 (48 B): {b}",
            )));
            set_out(&app.ui.digest_sha512, &format_str(format_args!(
                "SHA-512 (64 B): {c}",
            )));
            set_status(&app.ui.digest_status, "ok", "готово");
        }
        (Err(e), _, _) | (_, Err(e), _) | (_, _, Err(e)) => {
            set_status(&app.ui.digest_status, "err", &e.message);
        }
    }
}

async fn do_avalanche(app: Rc<App>) {
    set_status(&app.ui.digest_avalanche_status, "warn", "считаем…");
    set_disabled(&app.ui.digest_avalanche_btn, true);

    let t1 = app.ui.digest_input.value();
    let t2 = app.ui.digest_input2.value();

    let h1 = digest::digest_hex(HashAlg::Sha256, t1.as_bytes()).await;
    let h2 = digest::digest_hex(HashAlg::Sha256, t2.as_bytes()).await;

    set_disabled(&app.ui.digest_avalanche_btn, false);

    let (Ok(h1), Ok(h2)) = (h1, h2) else {
        set_status(&app.ui.digest_avalanche_status, "err", "ошибка хеширования");
        return;
    };

    let total = h1.len();
    let same = h1.chars().zip(h2.chars()).filter(|(a, b)| a == b).count();
    let pct = if total == 0 {
        0.0
    } else {
        (same as f64 / total as f64) * 100.0
    };

    let text1_len = t1.chars().count();
    let text2_len = t2.chars().count();

    let mut out = String::with_capacity(512);
    out.push_str(&format_str(format_args!(
        "Текст 1 ({} симв.): {t1}\n",
        text1_len,
    )));
    out.push_str(&format_str(format_args!("SHA-256(1): {h1}\n\n")));
    out.push_str(&format_str(format_args!(
        "Текст 2 ({} симв.): {t2}\n",
        text2_len,
    )));
    out.push_str(&format_str(format_args!("SHA-256(2): {h2}\n\n")));
    out.push_str(&format_str(format_args!(
        "Совпадающих hex-символов: {same}/{total} = {pct:.1}%\n",
    )));
    let diff_ok = pct < 20.0;
    if diff_ok {
        out.push_str("⟹ Хеши полностью разошлись, хотя тексты отличаются на 1 символ.");
    } else {
        out.push_str("⟹ Тексты слишком похожи — попробуйте сильнее изменить второй текст.");
    }

    set_out(&app.ui.digest_avalanche, &out);
    set_status(
        &app.ui.digest_avalanche_status,
        if diff_ok { "ok" } else { "warn" },
        if diff_ok { "разошлись" } else { "похожи" },
    );
}

async fn do_hmac_sign(app: Rc<App>) {
    set_status(&app.ui.hmac_status, "warn", "подпись…");
    set_disabled(&app.ui.hmac_btn, true);

    let key = match hmac::generate_key(HashAlg::Sha256, 256, true).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.hmac_status, "err", &e.message);
            set_disabled(&app.ui.hmac_btn, false);
            return;
        }
    };
    let data = app.ui.hmac_input.value().into_bytes();
    let sig = match hmac::sign(&key, &data).await {
        Ok(s) => s,
        Err(e) => {
            set_status(&app.ui.hmac_status, "err", &e.message);
            set_disabled(&app.ui.hmac_btn, false);
            return;
        }
    };

    let key_line = match keys::export_jwk(&key).await {
        Ok(jwk) => {
            let preview: String = jwk.chars().take(60).collect();
            format_str(format_args!("Ключ (JWK, {} B): {preview}…", jwk.len()))
        }
        Err(e) => format_str(format_args!(
            "Ключ сгенерирован, экспорт недоступен: {}",
            e.message,
        )),
    };

    set_out(&app.ui.hmac_key, &key_line);
    set_out(&app.ui.hmac_sig, &format_str(format_args!(
        "Подпись ({} B): {}",
        sig.len(),
        hex_encode(&sig),
    )));
    set_out(&app.ui.hmac_verify,
        "Проверка: нажмите «Проверить» (или измените текст — упадёт)");

    {
        let mut st = app.state.borrow_mut();
        st.hmac_key = Some(key);
        st.hmac_sig = Some(sig);
    }

    set_disabled(&app.ui.hmac_verify_btn, false);
    set_disabled(&app.ui.hmac_btn, false);
    set_status(&app.ui.hmac_status, "ok", "подписано");
}

async fn do_hmac_verify(app: Rc<App>) {
    let (key, sig) = {
        let st = app.state.borrow();
        (st.hmac_key.clone(), st.hmac_sig.clone())
    };
    let (Some(key), Some(sig)) = (key, sig) else {
        set_status(&app.ui.hmac_status, "err", "сначала подпишите");
        return;
    };
    let data = app.ui.hmac_input.value().into_bytes();
    match hmac::verify(&key, &sig, &data).await {
        Ok(true) => {
            set_out(&app.ui.hmac_verify, "Проверка: ✅ подпись верна");
            set_status(&app.ui.hmac_status, "ok", "верно");
        }
        Ok(false) => {
            set_out(&app.ui.hmac_verify,
                "Проверка: ❌ подпись НЕ совпадает — текст был изменён");
            set_status(&app.ui.hmac_status, "err", "не совпало");
        }
        Err(e) => set_status(&app.ui.hmac_status, "err", &e.message),
    }
}

async fn do_aes_encrypt(app: Rc<App>) {
    set_status(&app.ui.aes_status, "warn", "шифрование…");
    set_disabled(&app.ui.aes_btn, true);
    set_disabled(&app.ui.aes_dec_btn, true);
    set_disabled(&app.ui.aes_tamper_btn, true);

    // Если ключа ещё нет — генерируем. Иначе переиспользуем.
    let key = {
        let existing = app.state.borrow().aes_key.clone();
        match existing {
            Some(k) => k,
            None => match aes::generate_key(256, true).await {
                Ok(k) => {
                    app.state.borrow_mut().aes_key = Some(k.clone());
                    k
                }
                Err(e) => {
                    set_status(&app.ui.aes_status, "err", &e.message);
                    set_disabled(&app.ui.aes_btn, false);
                    return;
                }
            },
        }
    };

    let plaintext = app.ui.aes_input.value().into_bytes();
    let (ct, iv) = match aes::encrypt(&key, &plaintext).await {
        Ok(v) => v,
        Err(e) => {
            set_status(&app.ui.aes_status, "err", &e.message);
            set_disabled(&app.ui.aes_btn, false);
            return;
        }
    };

    set_out(&app.ui.aes_iv, &format_str(format_args!(
        "IV ({} B): {}",
        iv.len(),
        hex_encode(&iv),
    )));
    set_out(&app.ui.aes_ct, &format_str(format_args!(
        "Шифротекст ({} B = payload + 16 B tag): {}",
        ct.len(),
        hex_encode(&ct),
    )));
    set_out(&app.ui.aes_pt, "Открытый текст: нажмите «Дешифровать»");

    {
        let mut st = app.state.borrow_mut();
        st.aes_ct = Some(ct);
        st.aes_iv = Some(iv);
    }

    set_disabled(&app.ui.aes_btn, false);
    set_disabled(&app.ui.aes_dec_btn, false);
    set_disabled(&app.ui.aes_tamper_btn, false);
    set_status(&app.ui.aes_status, "ok", "зашифровано");
}

async fn do_aes_decrypt(app: Rc<App>) {
    let (key, ct, iv) = {
        let st = app.state.borrow();
        (st.aes_key.clone(), st.aes_ct.clone(), st.aes_iv.clone())
    };
    let (Some(key), Some(ct), Some(iv)) = (key, ct, iv) else {
        set_status(&app.ui.aes_status, "err", "сначала зашифруйте");
        return;
    };
    match aes::decrypt(&key, &ct, &iv).await {
        Ok(pt) => {
            let text = String::from_utf8_lossy(&pt);
            set_out(&app.ui.aes_pt, &format_str(format_args!(
                "Открытый текст ({} B): {text}",
                pt.len(),
            )));
            set_status(&app.ui.aes_status, "ok", "расшифровано");
        }
        Err(e) => set_status(&app.ui.aes_status, "err", &e.message),
    }
}

async fn do_aes_tamper(app: Rc<App>) {
    let (key, ct, iv) = {
        let st = app.state.borrow();
        (st.aes_key.clone(), st.aes_ct.clone(), st.aes_iv.clone())
    };
    let (Some(key), Some(ct), Some(iv)) = (key, ct, iv) else {
        set_status(&app.ui.aes_status, "err", "сначала зашифруйте");
        return;
    };

    let mut tampered = ct.clone();
    if tampered.is_empty() {
        set_status(&app.ui.aes_status, "err", "пустой шифротекст");
        return;
    }
    tampered[0] ^= 0x01;

    match aes::decrypt(&key, &tampered, &iv).await {
        Ok(_) => {
            set_out(&app.ui.aes_pt,
                "⚠ ОШИБКА БЕЗОПАСНОСТИ: подмена прошла незамеченной");
            set_status(&app.ui.aes_status, "err", "GCM не сработал");
        }
        Err(e) => {
            set_out(&app.ui.aes_pt, &format_str(format_args!(
                "✅ Подмена обнаружена. Ошибка дешифрования: {}",
                e.message,
            )));
            set_status(&app.ui.aes_status, "ok", "подмена обнаружена");
        }
    }
}

async fn do_rsa_generate(app: Rc<App>) {
    set_status(&app.ui.rsa_status, "warn", "генерация 2048-бит… (1–2 с)");
    set_disabled(&app.ui.rsa_gen_btn, true);
    set_disabled(&app.ui.rsa_enc_btn, true);
    set_disabled(&app.ui.rsa_dec_btn, true);
    TimeoutFuture::new(0).await;

    let result = rsa::generate_key_pair(rsa::RSA_2048, HashAlg::Sha256).await;

    match result {
        Ok(kp) => {
            let jwk_line = match keys::export_jwk(&kp.public_key).await {
                Ok(jwk) => {
                    let preview: String = jwk.chars().take(80).collect();
                    format_str(format_args!(
                        "Публичный ключ (JWK, {} B): {preview}…",
                        jwk.len(),
                    ))
                }
                Err(e) => format_str(format_args!(
                    "Публичный ключ сгенерирован, экспорт JWK недоступен: {}",
                    e.message,
                )),
            };
            set_out(&app.ui.rsa_pub, &jwk_line);
            set_out(&app.ui.rsa_ct, "Шифротекст: нажмите «Зашифровать»");
            set_out(&app.ui.rsa_pt, "Открытый текст: нажмите «Дешифровать» после шифрования");

            {
                let mut st = app.state.borrow_mut();
                st.rsa_kp = Some(kp);
                st.rsa_ct = None;
            }
            set_disabled(&app.ui.rsa_enc_btn, false);
            set_status(&app.ui.rsa_status, "ok", "ключи готовы");
        }
        Err(e) => set_status(&app.ui.rsa_status, "err", &e.message),
    }
    set_disabled(&app.ui.rsa_gen_btn, false);
}

async fn do_rsa_encrypt(app: Rc<App>) {
    let pub_key = {
        let st = app.state.borrow();
        st.rsa_kp.as_ref().map(|kp| kp.public_key.clone())
    };
    let Some(pub_key) = pub_key else {
        set_status(&app.ui.rsa_status, "err", "сначала сгенерируйте ключи");
        return;
    };
    let plaintext = app.ui.rsa_input.value().into_bytes();
    match rsa::encrypt(&pub_key, &plaintext).await {
        Ok(ct) => {
            let preview = hex_encode(&ct[..ct.len().min(32)]);
            set_out(&app.ui.rsa_ct, &format_str(format_args!(
                "Шифротекст ({} B, первые 32): {preview}…",
                ct.len(),
            )));
            set_out(&app.ui.rsa_pt, "Открытый текст: нажмите «Дешифровать»");
            {
                let mut st = app.state.borrow_mut();
                st.rsa_ct = Some(ct);
            }
            set_disabled(&app.ui.rsa_dec_btn, false);
            set_status(&app.ui.rsa_status, "ok", "зашифровано");
        }
        Err(e) => set_status(&app.ui.rsa_status, "err", &e.message),
    }
}

async fn do_rsa_decrypt(app: Rc<App>) {
    let (priv_key, ct) = {
        let st = app.state.borrow();
        (
            st.rsa_kp.as_ref().map(|kp| kp.private_key.clone()),
            st.rsa_ct.clone(),
        )
    };
    let (Some(priv_key), Some(ct)) = (priv_key, ct) else {
        set_status(&app.ui.rsa_status, "err", "сначала зашифруйте");
        return;
    };
    match rsa::decrypt(&priv_key, &ct).await {
        Ok(pt) => {
            let text = String::from_utf8_lossy(&pt);
            set_out(&app.ui.rsa_pt, &format_str(format_args!(
                "Открытый текст ({} B): {text}",
                pt.len(),
            )));
            set_status(&app.ui.rsa_status, "ok", "расшифровано");
        }
        Err(e) => set_status(&app.ui.rsa_status, "err", &e.message),
    }
}

async fn do_ecdsa_sign(app: Rc<App>) {
    set_status(&app.ui.ecdsa_status, "warn", "подпись…");
    set_disabled(&app.ui.ecdsa_sign_btn, true);
    set_disabled(&app.ui.ecdsa_verify_btn, true);

    let kp = match ecdsa::generate_key_pair(ecdsa::P256).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.ecdsa_status, "err", &e.message);
            set_disabled(&app.ui.ecdsa_sign_btn, false);
            return;
        }
    };
    let data = app.ui.ecdsa_input.value().into_bytes();
    let sig = match ecdsa::sign(&kp.private_key, &data).await {
        Ok(s) => s,
        Err(e) => {
            set_status(&app.ui.ecdsa_status, "err", &e.message);
            set_disabled(&app.ui.ecdsa_sign_btn, false);
            return;
        }
    };

    set_out(&app.ui.ecdsa_sig, &format_str(format_args!(
        "Подпись ECDSA P-256 ({} B): {}",
        sig.len(),
        hex_encode(&sig),
    )));
    set_out(&app.ui.ecdsa_verify,
        "Проверка: нажмите «Проверить» (или измените текст — упадёт)");

    {
        let mut st = app.state.borrow_mut();
        st.ecdsa_kp = Some(kp);
        st.ecdsa_sig = Some(sig);
    }

    set_disabled(&app.ui.ecdsa_sign_btn, false);
    set_disabled(&app.ui.ecdsa_verify_btn, false);
    set_status(&app.ui.ecdsa_status, "ok", "подписано");
}

async fn do_ecdsa_verify(app: Rc<App>) {
    let (pub_key, sig) = {
        let st = app.state.borrow();
        (
            st.ecdsa_kp.as_ref().map(|kp| kp.public_key.clone()),
            st.ecdsa_sig.clone(),
        )
    };
    let (Some(pub_key), Some(sig)) = (pub_key, sig) else {
        set_status(&app.ui.ecdsa_status, "err", "сначала подпишите");
        return;
    };
    let data = app.ui.ecdsa_input.value().into_bytes();
    match ecdsa::verify(&pub_key, &sig, &data).await {
        Ok(true) => {
            set_out(&app.ui.ecdsa_verify, "Проверка: ✅ подпись верна");
            set_status(&app.ui.ecdsa_status, "ok", "верно");
        }
        Ok(false) => {
            set_out(&app.ui.ecdsa_verify,
                "Проверка: ❌ подпись НЕ совпадает — текст был изменён");
            set_status(&app.ui.ecdsa_status, "err", "не совпало");
        }
        Err(e) => set_status(&app.ui.ecdsa_status, "err", &e.message),
    }
}

async fn do_ecdh(app: Rc<App>) {
    set_status(&app.ui.ecdh_status, "warn", "обмен ключами…");
    set_disabled(&app.ui.ecdh_btn, true);

    let alice = match ecdsa::generate_key_pair_ecdh(ecdsa::P256).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.ecdh_status, "err", &e.message);
            set_disabled(&app.ui.ecdh_btn, false);
            return;
        }
    };
    let bob = match ecdsa::generate_key_pair_ecdh(ecdsa::P256).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.ecdh_status, "err", &e.message);
            set_disabled(&app.ui.ecdh_btn, false);
            return;
        }
    };

    let alice_key = match ecdsa::derive_aes_key(&alice.private_key, &bob.public_key).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.ecdh_status, "err", &e.message);
            set_disabled(&app.ui.ecdh_btn, false);
            return;
        }
    };
    let bob_key = match ecdsa::derive_aes_key(&bob.private_key, &alice.public_key).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.ecdh_status, "err", &e.message);
            set_disabled(&app.ui.ecdh_btn, false);
            return;
        }
    };

    let plaintext = app.ui.ecdh_input.value().into_bytes();
    let (ct, iv) = match aes::encrypt(&alice_key, &plaintext).await {
        Ok(v) => v,
        Err(e) => {
            set_status(&app.ui.ecdh_status, "err", &e.message);
            set_disabled(&app.ui.ecdh_btn, false);
            return;
        }
    };
    let decrypted = match aes::decrypt(&bob_key, &ct, &iv).await {
        Ok(v) => v,
        Err(e) => {
            set_status(&app.ui.ecdh_status, "err", &e.message);
            set_disabled(&app.ui.ecdh_btn, false);
            return;
        }
    };

    let ok = decrypted == plaintext;
    let mut out = String::with_capacity(256);
    out.push_str("Алиса → Боб: ");
    out.push_str(if ok { "✅ общий ключ совпал" } else { "❌ ключи разные" });
    out.push('\n');
    out.push_str(&format_str(format_args!(
        "IV ({} B): {}",
        iv.len(),
        hex_encode(&iv),
    )));
    out.push('\n');
    out.push_str(&format_str(format_args!("Шифротекст ({} B)", ct.len())));
    out.push('\n');
    out.push_str(&format_str(format_args!(
        "Дешифровано: {}",
        String::from_utf8_lossy(&decrypted),
    )));
    set_out(&app.ui.ecdh_result, &out);
    set_status(
        &app.ui.ecdh_status,
        if ok { "ok" } else { "err" },
        if ok { "общий ключ" } else { "не совпало" },
    );
    set_disabled(&app.ui.ecdh_btn, false);
}

async fn do_random(app: Rc<App>) {
    set_status(&app.ui.rand_status, "warn", "генерируем…");
    set_disabled(&app.ui.rand_btn, true);

    let bytes_result = random::random_bytes(32);
    let uuid_result = random::random_uuid();

    set_disabled(&app.ui.rand_btn, false);

    match bytes_result {
        Ok(bytes) => {
            set_out(&app.ui.rand_bytes, &format_str(format_args!(
                "32 байта (hex, {} симв.): {}",
                bytes.len() * 2,
                hex_encode(&bytes),
            )));
        }
        Err(e) => {
            set_status(&app.ui.rand_status, "err", &e.message);
            return;
        }
    }
    match uuid_result {
        Ok(uuid) => {
            set_out(&app.ui.rand_uuid, &format_str(format_args!(
                "UUID v4 (36 симв.): {uuid}",
            )));
            set_status(&app.ui.rand_status, "ok", "готово");
        }
        Err(e) => set_status(&app.ui.rand_status, "err", &e.message),
    }
}

async fn do_export(app: Rc<App>) {
    set_status(&app.ui.io_status, "warn", "экспорт…");
    set_disabled(&app.ui.io_btn, true);

    let key = {
        let existing = app.state.borrow().aes_key.clone();
        match existing {
            Some(k) => k,
            None => match aes::generate_key(256, true).await {
                Ok(k) => {
                    app.state.borrow_mut().aes_key = Some(k.clone());
                    k
                }
                Err(e) => {
                    set_status(&app.ui.io_status, "err", &e.message);
                    set_disabled(&app.ui.io_btn, false);
                    return;
                }
            },
        }
    };

    let raw = keys::export_raw(&key).await;
    let jwk = keys::export_jwk(&key).await;

    set_disabled(&app.ui.io_btn, false);

    let mut had_err = false;
    match raw {
        Ok(bytes) => {
            set_out(&app.ui.io_raw, &format_str(format_args!(
                "Raw ({} B): {}",
                bytes.len(),
                hex_encode(&bytes),
            )));
            app.state.borrow_mut().aes_raw_export = Some(bytes);
            set_disabled(&app.ui.io_import_btn, false);
        }
        Err(e) => {
            had_err = true;
            set_status(&app.ui.io_status, "err", &e.message);
        }
    }
    match jwk {
        Ok(s) => {
            set_out(&app.ui.io_jwk, &format_str(format_args!(
                "JWK ({} B): {s}",
                s.len(),
            )));
        }
        Err(e) => {
            had_err = true;
            set_status(&app.ui.io_status, "err", &e.message);
        }
    }
    if !had_err {
        set_status(&app.ui.io_status, "ok", "экспортировано");
    }
}

async fn do_import(app: Rc<App>) {
    set_status(&app.ui.io_status, "warn", "импорт…");
    set_disabled(&app.ui.io_import_btn, true);

    let (raw, ct, iv) = {
        let st = app.state.borrow();
        (
            st.aes_raw_export.clone(),
            st.aes_ct.clone(),
            st.aes_iv.clone(),
        )
    };
    let Some(raw) = raw else {
        set_status(&app.ui.io_status, "err", "сначала экспортируйте ключ");
        set_disabled(&app.ui.io_import_btn, false);
        return;
    };

    let alg = keys::aes_gcm_alg();
    let key = match keys::import_raw(
        KeyFormat::Raw,
        &raw,
        &alg,
        true,
        &["encrypt", "decrypt"],
    ).await {
        Ok(k) => k,
        Err(e) => {
            set_status(&app.ui.io_status, "err", &e.message);
            set_disabled(&app.ui.io_import_btn, false);
            return;
        }
    };

    if let (Some(ct), Some(iv)) = (ct, iv) {
        match aes::decrypt(&key, &ct, &iv).await {
            Ok(pt) => {
                set_out(&app.ui.io_import, &format_str(format_args!(
                    "Импортированный ключ расшифровал {} B: \"{}\"",
                    pt.len(),
                    String::from_utf8_lossy(&pt),
                )));
                set_status(&app.ui.io_status, "ok", "импорт ок");
            }
            Err(e) => {
                set_out(&app.ui.io_import, &format_str(format_args!(
                    "❌ Импортированный ключ не подошёл: {}",
                    e.message,
                )));
                set_status(&app.ui.io_status, "err", "не сошлось");
            }
        }
    } else {
        set_out(&app.ui.io_import,
            "✓ Ключ импортирован (шифротекста из блока 3 нет — расшифровка пропущена)");
        set_status(&app.ui.io_status, "ok", "импорт ок");
    }
    set_disabled(&app.ui.io_import_btn, false);
}

// ============================================================
//                    РЕНДЕР
// ============================================================

fn render_hint() -> Result<(), JsValue> {
    let hint = get_el("c-hint")?;
    let html = "Web Crypto API доступен только в безопасном контексте \
                (<code>https://</code> или <code>localhost</code>). \
                Все операции асинхронны: <code>crypto.subtle</code> возвращает \
                <code>Promise</code>. Ключи с <code>extractable: false</code> \
                нельзя экспортировать. В Rust для прода обычно берут \
                <code>RustCrypto</code> — там нет GC-пауз и есть \
                zeroize для чувствительных данных; Web Crypto выигрывает \
                простотой и отсутствием зависимости от платформы.";
    hint.set_inner_html(html);
    Ok(())
}

fn set_out(el: &web_sys::Element, text: &str) {
    el.set_text_content(Some(text));
    el.set_class_name("c-out");
}

fn set_status(el: &web_sys::Element, kind: &str, msg: &str) {
    let cls = match kind {
        "ok" => "c-badge ok",
        "err" => "c-badge err",
        "warn" => "c-badge warn",
        _ => "c-badge",
    };
    el.set_class_name(cls);
    el.set_text_content(Some(msg));
}

fn set_disabled(el: &HtmlElement, disabled: bool) {
    if disabled {
        let _ = el.set_attribute("disabled", "");
    } else {
        let _ = el.remove_attribute("disabled");
    }
}

// ============================================================
//                    ХЕЛПЕРЫ
// ============================================================

fn get_el(id: &str) -> Result<web_sys::Element, JsValue> {
    return web_sys::window()
        .ok_or("no window")?
        .document()
        .ok_or("no doc")?
        .get_element_by_id(id)
        .ok_or_else(|| JsValue::from_str(
            &format_str(format_args!("no #{id}")),
        ));
}

fn get_btn(id: &str) -> Result<HtmlElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlElement"));
}

fn get_textarea(id: &str) -> Result<HtmlTextAreaElement, JsValue> {
    return get_el(id)?
        .dyn_into()
        .map_err(|_| JsValue::from_str("not HtmlTextAreaElement"));
}

fn set_click<F>(el: &HtmlElement, f: F)
where
    F: FnMut() + 'static,
{
    let cb = Closure::<dyn FnMut()>::new(f);
    el.set_onclick(Some(cb.as_ref().unchecked_ref()));
    cb.forget();
}
```

### `tests/wasm_web_crypto.rs`

```rust
//! WASM-тесты главы 38: Web Crypto API.
//!
//! Запуск:
//!   wasm-pack test --chrome --headless -- --test wasm_web_crypto
//!
//! Web Crypto API доступен только в secure context (https:// или localhost).
//! Все тесты — асинхронные, выполняются в реальном браузере.

use wasm_bindgen_test::*;

use hello_wasm02::web_crypto::{
    aes, digest, ecdsa, hmac, keys, random, rsa,
    digest::HashAlg,
};

wasm_bindgen_test_configure!(run_in_browser);

// ============================================================
// 1. DIGEST — SHA-256 / SHA-384 / SHA-512
// ============================================================

#[wasm_bindgen_test]
async fn wasm_sha256_empty() {
    let hex = digest::digest_hex(HashAlg::Sha256, b"").await.unwrap();
    assert_eq!(
        hex,
        "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
    );
}

#[wasm_bindgen_test]
async fn wasm_sha256_abc() {
    let hex = digest::digest_hex(HashAlg::Sha256, b"abc").await.unwrap();
    assert_eq!(
        hex,
        "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
    );
}

#[wasm_bindgen_test]
async fn wasm_sha256_len() {
    let bytes = digest::digest(HashAlg::Sha256, b"hello").await.unwrap();
    assert_eq!(bytes.len(), digest::SHA256_LEN);
}

#[wasm_bindgen_test]
async fn wasm_sha384_len() {
    let bytes = digest::digest(HashAlg::Sha384, b"hello").await.unwrap();
    assert_eq!(bytes.len(), digest::SHA384_LEN);
}

#[wasm_bindgen_test]
async fn wasm_sha512_len() {
    let bytes = digest::digest(HashAlg::Sha512, b"hello").await.unwrap();
    assert_eq!(bytes.len(), digest::SHA512_LEN);
}

// ============================================================
// 2. HMAC — sign / verify
// ============================================================

#[wasm_bindgen_test]
async fn wasm_hmac_sign_verify_roundtrip() {
    let key = hmac::generate_key(HashAlg::Sha256, 256, true).await.unwrap();
    let data = b"message to authenticate";
    let sig = hmac::sign(&key, data).await.unwrap();
    assert_eq!(sig.len(), 32);
    let ok = hmac::verify(&key, &sig, data).await.unwrap();
    assert!(ok);
}

#[wasm_bindgen_test]
async fn wasm_hmac_verify_wrong_data() {
    let key = hmac::generate_key(HashAlg::Sha256, 256, true).await.unwrap();
    let sig = hmac::sign(&key, b"correct").await.unwrap();
    let ok = hmac::verify(&key, &sig, b"tampered").await.unwrap();
    assert!(!ok);
}

// ============================================================
// 3. AES-GCM — encrypt / decrypt
// ============================================================

#[wasm_bindgen_test]
async fn wasm_aes_gcm_roundtrip() {
    let key = aes::generate_key(256, true).await.unwrap();
    let plaintext = b"secret message";
    let (ciphertext, iv) = aes::encrypt(&key, plaintext).await.unwrap();
    assert_ne!(ciphertext, plaintext);
    let decrypted = aes::decrypt(&key, &ciphertext, &iv).await.unwrap();
    assert_eq!(decrypted, plaintext);
}

#[wasm_bindgen_test]
async fn wasm_aes_gcm_tampered_fails() {
    let key = aes::generate_key(256, true).await.unwrap();
    let (mut ciphertext, iv) = aes::encrypt(&key, b"data").await.unwrap();
    ciphertext[0] ^= 0xFF;
    let result = aes::decrypt(&key, &ciphertext, &iv).await;
    assert!(result.is_err());
}

// ============================================================
// 4. RSA-OAEP — encrypt / decrypt
// ============================================================

#[wasm_bindgen_test]
async fn wasm_rsa_oaep_roundtrip() {
    let kp = rsa::generate_key_pair(rsa::RSA_2048, HashAlg::Sha256)
        .await
        .unwrap();
    let plaintext = b"hello rsa";
    let ciphertext = rsa::encrypt(&kp.public_key, plaintext).await.unwrap();
    assert_ne!(ciphertext, plaintext);
    let decrypted = rsa::decrypt(&kp.private_key, &ciphertext).await.unwrap();
    assert_eq!(decrypted, plaintext);
}

// ============================================================
// 5. ECDSA — sign / verify
// ============================================================

#[wasm_bindgen_test]
async fn wasm_ecdsa_sign_verify_roundtrip() {
    let kp = ecdsa::generate_key_pair(ecdsa::P256).await.unwrap();
    let data = b"signed data";
    let sig = ecdsa::sign(&kp.private_key, data).await.unwrap();
    let ok = ecdsa::verify(&kp.public_key, &sig, data).await.unwrap();
    assert!(ok);
}

#[wasm_bindgen_test]
async fn wasm_ecdsa_verify_wrong_data() {
    let kp = ecdsa::generate_key_pair(ecdsa::P256).await.unwrap();
    let sig = ecdsa::sign(&kp.private_key, b"original").await.unwrap();
    let ok = ecdsa::verify(&kp.public_key, &sig, b"tampered").await.unwrap();
    assert!(!ok);
}

// ============================================================
// 6. ECDH — derive shared AES key
// ============================================================

#[wasm_bindgen_test]
async fn wasm_ecdh_shared_key_matches() {
    let alice = ecdsa::generate_key_pair_ecdh(ecdsa::P256).await.unwrap();
    let bob = ecdsa::generate_key_pair_ecdh(ecdsa::P256).await.unwrap();

    let alice_key = ecdsa::derive_aes_key(&alice.private_key, &bob.public_key)
        .await
        .unwrap();
    let bob_key = ecdsa::derive_aes_key(&bob.private_key, &alice.public_key)
        .await
        .unwrap();

    let plaintext = b"shared secret message";
    let (ciphertext, iv) = aes::encrypt(&alice_key, plaintext).await.unwrap();
    let decrypted = aes::decrypt(&bob_key, &ciphertext, &iv).await.unwrap();
    assert_eq!(decrypted, plaintext);
}

// ============================================================
// 7. RANDOM — getRandomValues, UUID
// ============================================================

#[wasm_bindgen_test]
async fn wasm_random_bytes_len() {
    let bytes = random::random_bytes(32).unwrap();
    assert_eq!(bytes.len(), 32);
}

#[wasm_bindgen_test]
async fn wasm_random_bytes_different() {
    let a = random::random_bytes(32).unwrap();
    let b = random::random_bytes(32).unwrap();
    assert_ne!(a, b);
}

#[wasm_bindgen_test]
async fn wasm_random_uuid_format() {
    let uuid = random::random_uuid().unwrap();
    assert_eq!(uuid.len(), 36);
    assert_eq!(uuid.chars().filter(|c| *c == '-').count(), 4);
}

// ============================================================
// 8. importKey / exportKey
// ============================================================

#[wasm_bindgen_test]
async fn wasm_export_aes_raw() {
    let key = aes::generate_key(256, true).await.unwrap();
    let raw = keys::export_raw(&key).await.unwrap();
    assert_eq!(raw.len(), 32);
}

#[wasm_bindgen_test]
async fn wasm_export_hmac_jwk() {
    let key = hmac::generate_key(HashAlg::Sha256, 256, true).await.unwrap();
    let jwk = keys::export_jwk(&key).await.unwrap();
    assert!(jwk.contains("\"k\""));
    assert!(jwk.contains("\"alg\""));
}
```

---

## 9. Команды запуска

```bash
# Сборка WASM-пакета (dev)
npm run wasm:dev

# Только тесты главы 38
wasm-pack test --chrome --headless -- --test wasm_web_crypto

# Отдельный тест по имени
wasm-pack test --chrome --headless -- --test wasm_web_crypto wasm_ecdh

# Firefox
wasm-pack test --firefox --headless -- --test wasm_web_crypto

# Демо
npm run dev
# Открыть http://localhost:5173
```

---

## 10. Итог

Криптография — область, где важно не только «работает», но и «почему безопасно». Мы прошли путь от `window.crypto.subtle` до сквозного сценария, где четыре примитива взаимодействуют так, как это делают реальные системы обмена сообщениями. Ошибки, выявленные при сборке — `publicExponent`, пустой `keyUsages`, `&Object` vs `&JsValue`, `JsCast` в scope — не «мелочи реализации», а обязательные детали W3C-спецификации. Они показывают: **Web Crypto не прощает импровизаций**. Каждое поле в алгоритме, каждая роль ключа, каждый байт `IV` — всё определено стандартом и отступать от него нельзя.

Именно поэтому в продакшене часто берут RustCrypto — там спецификация реализована один раз и не требует помнить, где web-sys скрывает `Result`, а где возвращает `Promise`. Но Web Crypto выигрывает в другом: он **уже в браузере**, работает на аппаратных ускорителях и не требует ни байта в WASM-бандле. Гибрид — часто лучшее решение: тяжёлые долгоживущие операции в Rust, быстрые сессионные — в Web Crypto.